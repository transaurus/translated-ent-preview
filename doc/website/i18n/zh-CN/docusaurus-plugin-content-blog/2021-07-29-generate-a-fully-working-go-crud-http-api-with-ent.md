---
title: Generate a fully-working Go CRUD HTTP API with Ent
author: MasseElch
authorURL: "https://github.com/masseelch"
authorImageURL: "https://avatars.githubusercontent.com/u/12862103?v=4"
---

当我们说Ent的核心原则之一是"Schema as Code"时，其含义远不止"Ent用于定义实体及其关系的DSL是通过常规Go代码实现的"。与其他许多ORM相比，Ent的独特之处在于将所有与实体相关的逻辑都以代码形式直接表达在模式定义中。

通过Ent，开发者可以直接在模式上编写所有授权逻辑（在Ent中称为"[隐私策略](https://entgo.io/docs/privacy)"）和所有变更副作用（在Ent中称为"[钩子](https://entgo.io/docs/hooks)"）。将所有内容集中在一处不仅非常方便，当与代码生成功能结合使用时，其真正威力才得以显现。

如果以这种方式定义模式，就有可能自动生成功能完整的生产级服务器代码。当我们将授权决策和自定义副作用的责任从RPC层转移到数据层时，基础CRUD（创建、读取、更新和删除）端点的实现就变得通用化，以至于可以被机器生成。这正是流行的GraphQL和gRPC Ent扩展背后的理念。

今天，我们很高兴推出名为`elk`的新Ent扩展，它能从Ent模式自动生成功能完备的RESTful API端点。`elk`致力于自动化处理为图中每个实体建立基础CRUD端点的所有繁琐工作，包括日志记录、请求体验证、关系预加载和序列化，同时避免使用反射并保持类型安全。

让我们开始吧！

### 快速开始

以下代码的最终版本可在[GitHub](https://github.com/masseelch/elk-example)上找到。

首先创建一个新的Go项目：

```shell
mkdir elk-example
cd elk-example
go mod init elk-example
```

调用ent代码生成器并创建两个模式：User和Pet：

```shell
go run -mod=mod entgo.io/ent/cmd/ent new Pet User
```

此时项目结构应如下所示：

```
.
├── ent
│   ├── generate.go
│   └── schema
│       ├── pet.go
│       └── user.go
├── go.mod
└── go.sum
```

接下来将`elk`包添加到项目中：

```shell
go get -u github.com/masseelch/elk
```

`elk`使用Ent的[扩展API](https://github.com/ent/ent/blob/a19a89a141cf1a5e1b38c93d7898f218a1f86c94/entc/entc.go#L197)与Ent的代码生成集成。这要求我们按照[此处](https://entgo.io/docs/code-gen#use-entc-as-a-package)所述使用`entc`（ent代码生成）包。按照以下三个步骤启用它并配置Ent以与`elk`扩展协同工作：

1\. 新建名为`ent/entc.go`的Go文件并粘贴以下内容：

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
		elk.GenerateHandlers(),
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

3\. `elk`在其生成的代码中使用了一些外部包。当前您需要在设置`elk`时手动获取这些包：

```shell
go get github.com/mailru/easyjson github.com/masseelch/render github.com/go-chi/chi/v5 go.uber.org/zap
```

完成这些步骤后，使用`elk`增强的ent就全部设置好了！要了解更多关于Ent的信息，如如何连接不同类型的数据库、运行迁移或操作实体，请参阅[设置教程](https://entgo.io/docs/tutorial-setup/)。

### 使用`elk`生成HTTP CRUD处理器

要生成功能完备的HTTP处理器，首先需要创建Ent模式定义。打开并编辑`ent/schema/pet.go`：

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
)

// Pet holds the schema definition for the Pet entity.
type Pet struct {
	ent.Schema
}

// Fields of the Pet.
func (Pet) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
		field.Int("age"),
	}
}

```

我们为`Pet`实体添加了两个字段：`name`和`age`。`ent.Schema`仅定义了实体的字段结构。要基于该模式生成可执行代码，请运行：

```shell
go generate ./...
```

注意，除了Ent常规生成的文件外，还会创建一个名为`ent/http`的新目录。这些文件由`elk`扩展生成，包含HTTP处理程序的实现代码。例如，以下是针对Pet实体读取操作的部分生成代码：

```go
const (
	PetCreate Routes = 1 << iota
	PetRead
	PetUpdate
	PetDelete
	PetList
	PetRoutes = 1<<iota - 1
)

// PetHandler handles http crud operations on ent.Pet.
type PetHandler struct {
	handler

	client *ent.Client
	log    *zap.Logger
}

func NewPetHandler(c *ent.Client, l *zap.Logger) *PetHandler {
	return &PetHandler{
		client: c,
		log:    l.With(zap.String("handler", "PetHandler")),
	}
}

// Read fetches the ent.Pet identified by a given url-parameter from the
// database and renders it to the client.
func (h *PetHandler) Read(w http.ResponseWriter, r *http.Request) {
	l := h.log.With(zap.String("method", "Read"))
	// ID is URL parameter.
	id, err := strconv.Atoi(chi.URLParam(r, "id"))
	if err != nil {
		l.Error("error getting id from url parameter", zap.String("id", chi.URLParam(r, "id")), zap.Error(err))
		render.BadRequest(w, r, "id must be an integer greater zero")
		return
	}
	// Create the query to fetch the Pet
	q := h.client.Pet.Query().Where(pet.ID(id))
	e, err := q.Only(r.Context())
	if err != nil {
		switch {
		case ent.IsNotFound(err):
			msg := stripEntError(err)
			l.Info(msg, zap.Error(err), zap.Int("id", id))
			render.NotFound(w, r, msg)
		case ent.IsNotSingular(err):
			msg := stripEntError(err)
			l.Error(msg, zap.Error(err), zap.Int("id", id))
			render.BadRequest(w, r, msg)
		default:
			l.Error("could not read pet", zap.Error(err), zap.Int("id", id))
			render.InternalServerError(w, r, nil)
		}
		return
	}
	l.Info("pet rendered", zap.Int("id", id))
	easyjson.MarshalToHTTPResponseWriter(NewPet2657988899View(e), w)
}

```

接下来，我们将创建一个实际的RESTful HTTP服务器来管理Pet实体。新建`main.go`文件并添加以下内容：

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"

	"elk-example/ent"
	elk "elk-example/ent/http"

	"github.com/go-chi/chi/v5"
	_ "github.com/mattn/go-sqlite3"
	"go.uber.org/zap"
)

func main() {
	// Create the ent client.
	c, err := ent.Open("sqlite3", "./ent.db?_fk=1")
	if err != nil {
		log.Fatalf("failed opening connection to sqlite: %v", err)
	}
	defer c.Close()
	// Run the auto migration tool.
	if err := c.Schema.Create(context.Background()); err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
	// Router and Logger.
	r, l := chi.NewRouter(), zap.NewExample()
	// Create the pet handler.
	r.Route("/pets", func(r chi.Router) {
		elk.NewPetHandler(c, l).Mount(r, elk.PetRoutes)
	})
	// Start listen to incoming requests.
	fmt.Println("Server running")
	defer fmt.Println("Server stopped")
	if err := http.ListenAndServe(":8080", r); err != nil {
		log.Fatal(err)
	}
}

```

启动服务器：

```shell
go run -mod=mod main.go
```

恭喜！现在您已拥有一个运行中的Pets API服务。虽然可以查询数据库中的所有宠物列表，但目前尚无数据。让我们先创建一只宠物：

```shell
curl -X 'POST' -H 'Content-Type: application/json' -d '{"name":"Kuro","age":3}' 'localhost:8080/pets'
```

您将收到如下响应：

```json
{
  "age": 3,
  "id": 1,
  "name": "Kuro"
}
```

在服务器运行终端中，您还能看到`elk`内置的日志输出：

```json
{
  "level": "info",
  "msg": "pet rendered",
  "handler": "PetHandler",
  "method": "Create",
  "id": 1
}
```

`elk`使用[zap](https://github.com/uber-go/zap)进行日志记录，更多信息请参阅其文档。

### 关联关系

为展示更多`elk`功能，让我们扩展数据图。编辑`ent/schema/user.go`和`ent/schema/pet.go`：

```go title="ent/schema/pet.go"
// Edges of the Pet.
func (Pet) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("owner", User.Type).
            Ref("pets").
            Unique(),
    }
}

```

```go title="ent/schema/user.go"
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
)

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
		field.Int("age"),
	}
}

// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("pets", Pet.Type),
	}
}

```

现在我们在Pet和User模式间建立了"一对多"关系：宠物属于用户，用户可拥有多只宠物。

重新运行代码生成器：

```shell
go generate ./...
```

别忘了在路由器上注册`UserHandler`。在`main.go`中添加以下代码：

```diff
[...]
    r.Route("/pets", func(r chi.Router) {
        elk.NewPetHandler(c, l, v).Mount(r, elk.PetRoutes)
    })
+    // Create the user handler.
+    r.Route("/users", func(r chi.Router) {
+        elk.NewUserHandler(c, l, v).Mount(r, elk.UserRoutes)
+    })
    // Start listen to incoming requests.
    fmt.Println("Server running")
[...]
```

重启服务器后，我们可以创建一个拥有先前所建宠物Kuro的`User`：

```shell
curl -X 'POST' -H 'Content-Type: application/json' -d '{"name":"Elk","age":30,"owner":1}' 'localhost:8080/users'
```

服务器返回如下响应：

```json
{
  "age": 30,
  "edges": {},
  "id": 1,
  "name": "Elk"
}
```

从输出可见用户已创建，但关联边为空。默认情况下`elk`不会在输出中包含边关系。您可以通过"序列化分组"功能配置边渲染。使用`elk.SchemaAnnotation`和`elk.Annotation`结构体注解模式，编辑`ent/schema/user.go`添加如下内容：

```go
// Edges of the User.
func (User) Edges() []ent.Edge {
    return []ent.Edge{
        edge.To("pets", Pet.Type).
            Annotations(elk.Groups("user")),
    }
}

// Annotations of the User.
func (User) Annotations() []schema.Annotation {
    return []schema.Annotation{elk.ReadGroups("user")}
}

```

字段和边上的`elk.Annotation`会指示elk进行预加载，并在请求"user"分组时将其加入响应负载。`elk.SchemaAnnotation`用于让`UserHandler`的读取操作请求"user"分组。注意：未附加序列化分组的字段默认会被包含，而边关系除非特别配置，否则默认排除。

再次重新生成代码并重启服务器。现在读取资源时，您应该能看到用户的宠物信息被渲染：

```shell
curl 'localhost:8080/users/1'
```

```json
{
  "age": 30,
  "edges": {
    "pets": [
      {
        "id": 1,
        "name": "Kuro",
        "age": 3,
        "edges": {}
      }
    ]
  },
  "id": 1,
  "name": "Elk"
}
```

### 请求验证

当前我们的模式允许为宠物或用户设置负年龄值，并且可以创建无主的宠物（如之前创建的Kuro）。Ent内置了对基础[验证](https://entgo.io/docs/schema-fields#validators)的支持。某些情况下，您可能希望在将API请求的有效载荷传递给Ent之前先进行验证。`elk`使用[此包](https://github.com/go-playground/validator)来定义验证规则并验证数据。我们可以通过`elk.Annotation`为创建和更新操作分别创建验证规则。在本例中，假设我们希望宠物模式仅允许年龄大于零，并禁止创建无主的宠物。编辑`ent/schema/pet.go`：

```go
// Fields of the Pet.
func (Pet) Fields() []ent.Field {
    return []ent.Field{
        field.String("name"),
        field.Int("age").
            Positive().
            Annotations(
                elk.CreateValidation("required,gt=0"),
                elk.UpdateValidation("gt=0"),
            ),
    }
}

// Edges of the Pet.
func (Pet) Edges() []ent.Edge {
    return []ent.Edge{
        edge.From("owner", User.Type).
            Ref("pets").
            Unique().
            Required().
            Annotations(elk.Validation("required")),
    }
}
```

接下来重新生成代码并重启服务器。为了测试新的验证规则，让我们尝试创建一个年龄无效且无主的宠物：

```shell
curl -X 'POST' -H 'Content-Type: application/json' -d '{"name":"Bob","age":-2}' 'localhost:8080/pets'
```

`elk`会返回一个详细的响应，其中包含验证失败的信息：

```json
{
  "code": 400,
  "status": "Bad Request",
  "errors": {
    "Age": "This value failed validation on 'gt:0'.",
    "Owner": "This value is required."
  }
}
```

注意字段名是大写的。验证器包使用结构体的字段名生成验证错误信息，但您可以轻松覆盖这一点，如[示例](https://github.com/go-playground/validator/blob/9a5bce32538f319bf69aebb3aca90d394bc6d0cb/_examples/struct-level/main.go#L37)所示。

如果您没有定义任何验证规则，`elk`将不会在其生成的代码中包含验证逻辑。`elk`的请求验证在需要进行跨字段验证时尤其有用。

### 即将推出的功能

我们希望您认同`elk`已经具备了一些实用的功能，但还有许多令人兴奋的功能即将到来。`elk`的下一个版本将包括：

- 功能完善的Flutter前端，用于管理您的节点
- 将Ent的验证集成到当前的请求验证器中
- 更多的传输格式（目前仅支持JSON）

### 结论

本文仅展示了`elk`功能的一小部分。要查看更多示例，请访问GitHub上的项目README。我们希望借助`elk`驱动的Ent，您和您的开发团队能够自动化构建RESTful API过程中的一些重复性任务，从而专注于更有意义的工作。

`elk`目前处于早期开发阶段，我们欢迎任何建议或反馈，如果您愿意提供帮助，我们将非常高兴。[GitHub Issues](https://github.com/masseelch/elk/issues)是您寻求帮助、反馈、建议和贡献的理想场所。

#### 关于作者

_MasseElch是来自德国北部多风平原地区的软件工程师。当不带着他的狗Kuro（它有自己的Instagram频道:scream:）徒步旅行或不与儿子玩捉迷藏时，他会喝咖啡并享受编程的乐趣。_