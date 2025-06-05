---
title: Auto generate REST CRUD with Ent and ogen 
author: MasseElch 
authorURL: "https://github.com/masseelch"
authorImageURL: "https://avatars.githubusercontent.com/u/12862103?v=4"
image: "https://entgo.io/images/assets/ogent/1.png"
---

2021年底我们曾宣布[Ent](https://entgo.io)获得了一个新的官方扩展，用于生成完全符合[OpenAPI规范](https://swagger.io/resources/open-api/)的文档：[`entoas`](https://github.com/ent/contrib/tree/master/entoas)。

今天我们非常高兴地宣布，专为与`entoas`协同工作的新扩展[`ogent`](https://github.com/ariga/ogent)正式发布。该扩展利用[`ogen`](https://github.com/ogen-go/ogen)（[官网](https://ogen.dev/docs/intro/)）的强大能力，为`entoas`生成的OpenAPI规范文档提供类型安全且无反射的实现方案。

`ogen`是一个基于OpenAPI规范v3文档的强约束Go代码生成器，能够同时生成服务端和客户端实现。开发者只需实现数据层访问接口即可。`ogen`具备诸多优秀特性，包括与[OpenTelemetry](https://opentelemetry.io/)的集成，强烈建议您体验并支持该项目。

本文介绍的扩展作为Ent与[`ogen`](https://github.com/ogen-go/ogen)生成代码之间的桥梁，通过`entoas`的配置来补全`ogen`代码的缺失部分。

下图展示了Ent如何与`entoas`和`ogent`扩展协同工作，以及`ogen`在其中的作用关系。

若您是Ent的新用户，想了解更多关于数据库连接、迁移操作或实体处理的内容，请参阅[入门教程](https://entgo.io/docs/tutorial-setup/)。

本文涉及的完整代码可在[示例模块](https://github.com/ariga/ogent/tree/main/example/todo)中获取。

### 快速开始

:::note[]
注意：Ent虽然支持Go 1.16+版本，但`ogen`要求至少使用Go 1.17版本。
:::

要使用`ogent`扩展，请按照[文档说明](https://entgo.io/docs/code-gen#use-entc-as-a-package)通过`entc`包进行代码生成。首先在Go模块中安装`entoas`和`ogent`扩展：

```shell
go get ariga.io/ogent@main
```

接下来通过两个步骤启用扩展并配置Ent：

1\. 新建`ent/entc.go`文件并写入以下内容：

```go title="ent/entc.go"
//go:build ignore

package main

import (
	"log"

	"ariga.io/ogent"
	"entgo.io/contrib/entoas"
	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
	"github.com/ogen-go/ogen"
)

func main() {
	spec := new(ogen.Spec)
	oas, err := entoas.NewExtension(entoas.Spec(spec))
	if err != nil {
		log.Fatalf("creating entoas extension: %v", err)
	}
	ogent, err := ogent.NewExtension(spec)
	if err != nil {
		log.Fatalf("creating ogent extension: %v", err)
	}
	err = entc.Generate("./schema", &gen.Config{}, entc.Extensions(ogent, oas))
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

2\. 修改`ent/generate.go`文件以执行`ent/entc.go`：

```go title="ent/generate.go"
package ent

//go:generate go run -mod=mod entc.go
```

完成上述配置后，即可根据Schema生成OAS文档并实现服务端代码！

### 生成CRUD HTTP API服务

构建HTTP API服务的第一步是创建Ent Schema图。以下是一个简明的示例Schema：

```go title="ent/schema/todo.go"
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
)

// Todo holds the schema definition for the Todo entity.
type Todo struct {
	ent.Schema
}

// Fields of the Todo.
func (Todo) Fields() []ent.Field {
	return []ent.Field{
		field.String("title"),
		field.Bool("done"),
	}
}
```

上述代码展示了Ent定义Schema图的典型方式，本例中我们创建了一个待办事项实体。

现在运行代码生成器：

```shell
go generate ./...
```

您将看到Ent生成的大量文件。其中`ent/openapi.json`由`entoas`扩展生成，以下是其内容片段：

```json title="ent/openapi.json"
{
  "info": {
    "title": "Ent Schema API",
    "description": "This is an auto generated API description made out of an Ent schema definition",
    "termsOfService": "",
    "contact": {},
    "license": {
      "name": ""
    },
    "version": "0.0.0"
  },
  "paths": {
    "/todos": {
      "get": {
    [...]
```

不过本文重点在于服务端实现部分，因此我们主要关注名为`ent/ogent`的目录。所有以`_gen.go`结尾的文件均由`ogen`生成，其中`oas_server_gen.go`文件包含了运行服务器时需要实现的接口定义。

```go title="ent/ogent/oas_server_gen.go"
// Handler handles operations described by OpenAPI v3 specification.
type Handler interface {
	// CreateTodo implements createTodo operation.
	//
	// Creates a new Todo and persists it to storage.
	//
	// POST /todos
	CreateTodo(ctx context.Context, req CreateTodoReq) (CreateTodoRes, error)
	// DeleteTodo implements deleteTodo operation.
	//
	// Deletes the Todo with the requested ID.
	//
	// DELETE /todos/{id}
	DeleteTodo(ctx context.Context, params DeleteTodoParams) (DeleteTodoRes, error)
	// ListTodo implements listTodo operation.
	//
	// List Todos.
	//
	// GET /todos
	ListTodo(ctx context.Context, params ListTodoParams) (ListTodoRes, error)
	// ReadTodo implements readTodo operation.
	//
	// Finds the Todo with the requested ID and returns it.
	//
	// GET /todos/{id}
	ReadTodo(ctx context.Context, params ReadTodoParams) (ReadTodoRes, error)
	// UpdateTodo implements updateTodo operation.
	//
	// Updates a Todo and persists changes to storage.
	//
	// PATCH /todos/{id}
	UpdateTodo(ctx context.Context, req UpdateTodoReq, params UpdateTodoParams) (UpdateTodoRes, error)
}
```

`ogent`扩展会在`ogent.go`文件中为该处理器提供默认实现。关于如何定义生成路由规则及预加载边关系的配置方法，请参阅`entoas`的[官方文档](https://github.com/ent/contrib/entoas)。

以下是自动生成的READ路由示例：

```go
// ReadTodo handles GET /todos/{id} requests.
func (h *OgentHandler) ReadTodo(ctx context.Context, params ReadTodoParams) (ReadTodoRes, error) {
	q := h.client.Todo.Query().Where(todo.IDEQ(params.ID))
	e, err := q.Only(ctx)
	if err != nil {
		switch {
		case ent.IsNotFound(err):
			return &R404{
				Code:   http.StatusNotFound,
				Status: http.StatusText(http.StatusNotFound),
				Errors: rawError(err),
			}, nil
		case ent.IsNotSingular(err):
			return &R409{
				Code:   http.StatusConflict,
				Status: http.StatusText(http.StatusConflict),
				Errors: rawError(err),
			}, nil
		default:
			// Let the server handle the error.
			return nil, err
		}
	}
	return NewTodoRead(e), nil
}
```

### 运行服务器

接下来创建`main.go`文件并完成应用服务器的组装，用于提供Todo-API服务。下面的main函数会初始化SQLite内存数据库，执行迁移创建所需数据表，并按照`ent/openapi.json`定义的API规范在`localhost:8080`启动服务：

```go title="main.go"
package main

import (
	"context"
	"log"
	"net/http"

	"entgo.io/ent/dialect"
	"entgo.io/ent/dialect/sql/schema"
	"<your-project>/ent"
	"<your-project>/ent/ogent"
	_ "github.com/mattn/go-sqlite3"
)

func main() {
	// Create ent client.
	client, err := ent.Open(dialect.SQLite, "file:ent?mode=memory&cache=shared&_fk=1")
	if err != nil {
		log.Fatal(err)
	}
	// Run the migrations.
	if err := client.Schema.Create(context.Background(), schema.WithAtlas(true)); err != nil {
		log.Fatal(err)
	}
	// Start listening.
	srv, err := ogent.NewServer(ogent.NewOgentHandler(client))
	if err != nil {
		log.Fatal(err)
	}
	if err := http.ListenAndServe(":8080", srv); err != nil {
		log.Fatal(err)
	}
}
```

通过`go run -mod=mod main.go`启动服务器后即可开始调用API。

首先尝试创建新Todo（演示时不发送请求体）：

```shell
↪ curl -X POST -H "Content-Type: application/json" localhost:8080/todos
{
  "error_message": "body required"
}
```

可见`ogen`已自动处理该异常——因为`entoas`在生成时将创建资源的请求体标记为必填项。现在带上请求体重试：

```shell
↪ curl -X POST -H "Content-Type: application/json" -d '{"title":"Give ogen and ogent a Star on GitHub"}'  localhost:8080/todos
{
  "error_message": "decode CreateTodo:application/json request: invalid: done (field required)"
}
```

出错了？`ogen`会明确提示问题所在：`done`字段是必填项。修改schema定义将该字段改为可选即可修复：

```go {18} title="ent/schema/todo.go"
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
)

// Todo holds the schema definition for the Todo entity.
type Todo struct {
	ent.Schema
}

// Fields of the Todo.
func (Todo) Fields() []ent.Field {
	return []ent.Field{
		field.String("title"),
		field.Bool("done").
		    Optional(),
	}
}
```

配置变更后需要重新执行代码生成并重启服务：

```shell
go generate ./...
go run -mod=mod main.go
```

现在再次尝试创建Todo：

```shell
↪ curl -X POST -H "Content-Type: application/json" -d '{"title":"Give ogen and ogent a Star on GitHub"}'  localhost:8080/todos
{
  "id": 1,
  "title": "Give ogen and ogent a Star on GitHub",
  "done": false
}
```

成功！数据库里新增了一条Todo记录。

假设您已完成Todo事项并为[`ogen`](https://github.com/ogen-go/ogen)和[`ogent`](https://github.com/ariga/ogent)点了赞（强烈推荐！），可通过PATCH请求标记完成状态：

```shell
↪ curl -X PATCH -H "Content-Type: application/json" -d '{"done":true}'  localhost:8080/todos/1
{
  "id": 1,
  "title": "Give ogen and ogent a Star on GitHub",
  "done": true
}
```

### 添加自定义端点

虽然当前已能标记完成状态，但若能添加专属路由`PATCH todos/:id/done`会更直观。这需要完成两个步骤：在OAS文档中声明新路由，并实现路由逻辑。首先通过`entoas`的mutation构建器编辑`ent/entc.go`文件添加路由描述：

```go {17-37} title="ent/entc.go"
//go:build ignore

package main

import (
	"log"

	"entgo.io/contrib/entoas"
	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
	"github.com/ariga/ogent"
	"github.com/ogen-go/ogen"
)

func main() {
	spec := new(ogen.Spec)
	oas, err := entoas.NewExtension(
		entoas.Spec(spec),
		entoas.Mutations(func(_ *gen.Graph, spec *ogen.Spec) error {
			spec.AddPathItem("/todos/{id}/done", ogen.NewPathItem().
				SetDescription("Mark an item as done").
				SetPatch(ogen.NewOperation().
					SetOperationID("markDone").
					SetSummary("Marks a todo item as done.").
					AddTags("Todo").
					AddResponse("204", ogen.NewResponse().SetDescription("Item marked as done")),
				).
				AddParameters(ogen.NewParameter().
					InPath().
					SetName("id").
					SetRequired(true).
					SetSchema(ogen.Int()),
				),
			)
			return nil
		}),
	)
	if err != nil {
		log.Fatalf("creating entoas extension: %v", err)
	}
	ogent, err := ogent.NewExtension(spec)
	if err != nil {
		log.Fatalf("creating ogent extension: %v", err)
	}
	err = entc.Generate("./schema", &gen.Config{}, entc.Extensions(ogent, oas))
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

执行代码生成命令(`go generate ./...`)后，`ent/openapi.json`文件中会出现新路由条目：

```json
"/todos/{id}/done": {
  "description": "Mark an item as done",
  "patch": {
    "tags": [
      "Todo"    
    ],
    "summary": "Marks a todo item as done.",
    "operationId": "markDone",
    "responses": {
      "204": {
        "description": "Item marked as done"
      }
    }
  },
  "parameters": [
    {
      "name": "id",
      "in": "path",
      "schema": {
        "type": "integer"
      },
      "required": true
    }
  ]
}
```

由`ogen`生成的`ent/ogent/oas_server_gen.go`文件也会同步更新：

```go {21-24} title="ent/ogent/oas_server_gen.go"
// Handler handles operations described by OpenAPI v3 specification.
type Handler interface {
	// CreateTodo implements createTodo operation.
	//
	// Creates a new Todo and persists it to storage.
	//
	// POST /todos
	CreateTodo(ctx context.Context, req CreateTodoReq) (CreateTodoRes, error)
	// DeleteTodo implements deleteTodo operation.
	//
	// Deletes the Todo with the requested ID.
	//
	// DELETE /todos/{id}
	DeleteTodo(ctx context.Context, params DeleteTodoParams) (DeleteTodoRes, error)
	// ListTodo implements listTodo operation.
	//
	// List Todos.
	//
	// GET /todos
	ListTodo(ctx context.Context, params ListTodoParams) (ListTodoRes, error)
	// MarkDone implements markDone operation.
	//
	// PATCH /todos/{id}/done
	MarkDone(ctx context.Context, params MarkDoneParams) (MarkDoneNoContent, error)
	// ReadTodo implements readTodo operation.
	//
	// Finds the Todo with the requested ID and returns it.
	//
	// GET /todos/{id}
	ReadTodo(ctx context.Context, params ReadTodoParams) (ReadTodoRes, error)
	// UpdateTodo implements updateTodo operation.
	//
	// Updates a Todo and persists changes to storage.
	//
	// PATCH /todos/{id}
	UpdateTodo(ctx context.Context, req UpdateTodoReq, params UpdateTodoParams) (UpdateTodoRes, error)
}
```

若此时运行服务，Go编译器会报错——因为`ogent`代码生成器尚未实现新路由逻辑。需要手动修改`main.go`文件来实现新增方法：

```go {15-22,34-38,40} title="main.go"
package main

import (
	"context"
	"log"
	"net/http"

	"entgo.io/ent/dialect"
	"entgo.io/ent/dialect/sql/schema"
	"github.com/ariga/ogent/example/todo/ent"
	"github.com/ariga/ogent/example/todo/ent/ogent"
	_ "github.com/mattn/go-sqlite3"
)

type handler struct {
	*ogent.OgentHandler
	client *ent.Client
}

func (h handler) MarkDone(ctx context.Context, params ogent.MarkDoneParams) (ogent.MarkDoneNoContent, error) {
	return ogent.MarkDoneNoContent{}, h.client.Todo.UpdateOneID(params.ID).SetDone(true).Exec(ctx)
}

func main() {
	// Create ent client.
	client, err := ent.Open(dialect.SQLite, "file:ent?mode=memory&cache=shared&_fk=1")
	if err != nil {
		log.Fatal(err)
	}
	// Run the migrations.
	if err := client.Schema.Create(context.Background(), schema.WithAtlas(true)); err != nil {
		log.Fatal(err)
	}
	// Create the handler.
	h := handler{
		OgentHandler: ogent.NewOgentHandler(client),
		client:       client,
	}
	// Start listening.
	srv := ogent.NewServer(h)
	if err := http.ListenAndServe(":8180", srv); err != nil {
		log.Fatal(err)
	}
}

```

重启服务后即可通过以下请求标记Todo事项为完成状态：

```shell
↪ curl -X PATCH localhost:8180/todos/1/done
```

### 未来计划

`ogent` 还有一些改进计划，最值得注意的是为 LIST 路由添加代码生成、类型安全的过滤功能。我们希望能先听取您的反馈。

### 总结

本文我们发布了 `ogent`，这是为 `entoas` 生成的 OpenAPI 规范文档提供的官方实现生成器。该扩展利用 [`ogen`](https://github.com/ogen-go/ogen) 的强大功能（一个功能丰富、支持 OpenAPI v3 文档的 Go 代码生成器），提供了一个开箱即用、可扩展的 RESTful HTTP API 服务器实现。

请注意，`ogen` 和 `entoas`/`ogent` 尚未发布首个主要版本，仍在开发中。不过，其 API 可视为稳定的。

有问题吗？需要入门帮助？欢迎加入我们的 [Discord 服务器](https://discord.gg/qZmPgTE6RX) 或 [Slack 频道](https://entgo.io/docs/slack/)。

:::note[获取更多 Ent 资讯和更新：]

- 订阅我们的 [新闻通讯](https://entgo.substack.com/)
- 在 [Twitter](https://twitter.com/entgo_io) 上关注我们
- 加入 [Gophers Slack](https://entgo.io/docs/slack) 的 #ent 频道
- 加入 [Ent Discord 服务器](https://discord.gg/qZmPgTE6RX)

:::