---
title: Generating OpenAPI Specification with Ent 
author: MasseElch 
authorURL: "https://github.com/masseelch"
authorImageURL: "https://avatars.githubusercontent.com/u/12862103?v=4"
---

在[之前的博客文章](https://entgo.io/blog/2021/07/29/generate-a-fully-working-go-crud-http-api-with-ent)中，我们向您介绍了[`elk`](https://github.com/masseelch/elk)——一个Ent的[扩展](https://entgo.io/docs/extensions)，它能够从您的模式生成一个功能完整的Go CRUD HTTP API。在今天的文章中，我想向您介绍`elk`最近新增的一个闪亮功能：一个完全兼容的[OpenAPI规范（OAS）](https://swagger.io/resources/open-api/)生成器。

OAS（以前称为Swagger规范）是一种技术规范，定义了REST API的标准、语言无关的接口描述。这使得人类和自动化工具无需查看实际源代码或额外文档就能理解所描述的服务。结合[Swagger工具](https://swagger.io/)，您只需传入OAS文件，就能为超过20种语言生成服务器和客户端的样板代码。

### 入门指南

第一步是将`elk`包添加到您的项目中：

```shell
go get github.com/masseelch/elk@latest
```

`elk`使用Ent的[扩展API](https://entgo.io/docs/extensions)与Ent的代码生成集成。这要求我们使用`entc`（ent代码生成）包，如[此处](https://entgo.io/docs/code-gen#use-entc-as-a-package)所述，为我们的项目生成代码。按照以下两个步骤启用它，并配置Ent以与`elk`扩展一起工作：

1\. 创建一个名为`ent/entc.go`的新Go文件，并粘贴以下内容：

```go
// +build ignore

package main

import (
	"log"

	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
	"github.com/masseelch/elk"
)

func main() {
	ex, err := elk.NewExtension(
		elk.GenerateSpec("openapi.json"),
	)
	if err != nil {
		log.Fatalf("creating elk extension: %v", err)
	}
	err = entc.Generate("./schema", &gen.Config{}, entc.Extensions(ex))
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

2\. 编辑`ent/generate.go`文件以执行`ent/entc.go`文件：

```go
package ent

//go:generate go run -mod=mod entc.go
```

完成这些步骤后，一切就绪，可以从您的模式生成OAS文件了！如果您是Ent的新手，想了解更多关于如何连接到不同类型的数据库、运行迁移或处理实体的信息，请前往[设置教程](https://entgo.io/docs/tutorial-setup/)。

### 生成OAS文件

生成OAS文件的第一步是创建一个Ent模式图：

```shell
go run -mod=mod entgo.io/ent/cmd/ent new Fridge Compartment Item
```

为了演示`elk`的OAS生成能力，我们将一起构建一个示例应用程序。假设我有多个冰箱，每个冰箱有多个隔间，我和我的另一半希望随时了解其中的内容。为了提供这一极其有用的信息，我们将创建一个带有RESTful API的Go服务器。为了简化能够与我们的服务器通信的客户端应用程序的创建，我们将创建一个描述其API的OpenAPI规范文件。一旦有了这个文件，我们就可以使用Swagger Codegen以我们选择的语言构建一个前端来管理冰箱和内容！您可以找到一个使用docker生成客户端的示例[这里](https://github.com/masseelch/elk/blob/master/internal/openapi/ent/generate.go)。

让我们创建我们的模式：

```go title="ent/fridge.go"
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
)

// Fridge holds the schema definition for the Fridge entity.
type Fridge struct {
	ent.Schema
}

// Fields of the Fridge.
func (Fridge) Fields() []ent.Field {
	return []ent.Field{
		field.String("title"),
	}
}

// Edges of the Fridge.
func (Fridge) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("compartments", Compartment.Type),
	}
}
```

```go title="ent/compartment.go"
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
)

// Compartment holds the schema definition for the Compartment entity.
type Compartment struct {
	ent.Schema
}

// Fields of the Compartment.
func (Compartment) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
	}
}

// Edges of the Compartment.
func (Compartment) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("fridge", Fridge.Type).
			Ref("compartments").
			Unique(),
		edge.To("contents", Item.Type),
	}
}
```

```go title="ent/item.go"
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
)

// Item holds the schema definition for the Item entity.
type Item struct {
	ent.Schema
}

// Fields of the Item.
func (Item) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
	}
}

// Edges of the Item.
func (Item) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("compartment", Compartment.Type).
			Ref("contents").
			Unique(),
	}
}
```

现在，让我们生成Ent代码和OAS文件。

```shell
go generate ./...
```

除了Ent通常生成的文件外，还创建了一个名为`openapi.json`的文件。将其内容复制并粘贴到[Swagger Editor](https://editor.swagger.io/)中。您应该看到三个组：**Compartment**、**Item**和**Fridge**。

如果您碰巧打开了Fridge组中的POST操作选项卡，您会看到预期请求数据的描述和所有可能的响应。太棒了！

### 基本配置

当前API的描述尚未准确反映其功能，让我们来完善它！`elk`提供了易于使用的配置构建器来调整生成的OAS文件。打开`ent/entc.go`文件，传入更新后的冰箱API标题和描述：

```go title="ent/entc.go"
//go:build ignore
// +build ignore

package main

import (
	"log"

	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
	"github.com/masseelch/elk"
)

func main() {
	ex, err := elk.NewExtension(
		elk.GenerateSpec(
			"openapi.json",
			// It is a Content-Management-System ...
			elk.SpecTitle("Fridge CMS"), 
			// You can use CommonMark syntax (https://commonmark.org/).
			elk.SpecDescription("API to manage fridges and their cooled contents. **ICY!**"), 
			elk.SpecVersion("0.0.1"),
		),
	)
	if err != nil {
		log.Fatalf("creating elk extension: %v", err)
	}
	err = entc.Generate("./schema", &gen.Config{}, entc.Extensions(ex))
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

重新运行代码生成器将创建更新后的OAS文件，您可将其复制粘贴到Swagger Editor中。

### 操作配置

我们不希望暴露删除冰箱的端点（说真的，谁会需要这个功能？！）。幸运的是，`elk`允许我们配置要生成或忽略的端点。`elk`的默认策略是暴露所有路由。您可以将此行为更改为仅暴露显式请求的路由，或者直接通过`elk.SchemaAnnotation`告诉`elk`排除冰箱的DELETE操作：

```go title="ent/schema/fridge.go"
// Annotations of the Fridge.
func (Fridge) Annotations() []schema.Annotation {
	return []schema.Annotation{
		elk.DeletePolicy(elk.Exclude),
	}
}
```

瞧！DELETE操作已消失。

有关`elk`策略工作原理及更多功能的详细信息，请参阅[godoc文档](https://pkg.go.dev/github.com/masseelch/elk)。

### 扩展规范

在这个示例中，我最关心的应该是冰箱当前的内容。您可以使用[钩子](https://pkg.go.dev/github.com/masseelch/elk#Hook)将生成的OAS定制到任何程度。不过，这超出了本文的范围。关于如何向生成的OAS文件添加`fridges/{id}/contents`端点的示例，请参见[此处](https://github.com/masseelch/elk/tree/master/internal/fridge/ent/entc.go)。

### 生成符合OAS规范的服务器

我在开头承诺过我们将创建一个行为符合OAS描述的服务器。`elk`使这变得非常简单，您只需在配置扩展时调用`elk.GenerateHandlers()`：

```diff title="ent/entc.go"
[...]
func main() {
	ex, err := elk.NewExtension(
		elk.GenerateSpec(
			[...]
		),
+		elk.GenerateHandlers(),
	)
	[...]
}

```

接下来，重新运行代码生成：

```shell
go generate ./...
```

注意，此时会创建一个名为`ent/http`的新目录。

```shell
» tree ent/http
ent/http
├── create.go
├── delete.go
├── easyjson.go
├── handler.go
├── list.go
├── read.go
├── relations.go
├── request.go
├── response.go
└── update.go

0 directories, 10 files
```

您可以通过这个非常简单的`main.go`文件启动生成的服务器：

```go
package main

import (
	"context"
	"log"
	"net/http"

	"<your-project>/ent"
	elk "<your-project>/ent/http"

	_ "github.com/mattn/go-sqlite3"
	"go.uber.org/zap"
)

func main() {
	// Create the ent client.
	c, err := ent.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	if err != nil {
		log.Fatalf("failed opening connection to sqlite: %v", err)
	}
	defer c.Close()
	// Run the auto migration tool.
	if err := c.Schema.Create(context.Background()); err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
	// Start listen to incoming requests.
	if err := http.ListenAndServe(":8080", elk.NewHandler(c, zap.NewExample())); err != nil {
		log.Fatal(err)
	}
}
```

```shell
go run -mod=mod main.go
```

我们的冰箱API服务器已启动并运行。借助生成的OAS文件和Swagger工具链，您现在可以用任何支持的语言生成客户端存根，从此_永远_不必再手动编写RESTful客户端。

### 总结

本文介绍了`elk`的新功能——自动生成OpenAPI规范。该功能将Ent的代码生成能力与OpenAPI/Swagger丰富的工具生态系统连接起来。

有问题？需要入门帮助？欢迎加入我们的[Discord服务器](https://discord.gg/qZmPgTE6RX)或[Slack频道](https://entgo.io/docs/slack/)。

:::note[获取更多Ent资讯：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 关注我们的[Twitter](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)上的#ent频道
- 加入[Ent Discord服务器](https://discord.gg/qZmPgTE6RX)

:::