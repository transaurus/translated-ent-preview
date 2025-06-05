---
title: Announcing "entoas" - An Extension to Automatically Generate OpenAPI Specification Documents from Ent Schemas 
author: MasseElch 
authorURL: "https://github.com/masseelch"
authorImageURL: "https://avatars.githubusercontent.com/u/12862103?v=4"
image: https://entgo.io/images/assets/elkopa/entoas-code.png
---

OpenAPI规范（OAS，前身为Swagger规范）是一种技术规范，为REST API定义了标准且与语言无关的接口描述。这使得人类和自动化工具无需查看源代码或额外文档即可理解所描述的服务。结合[Swagger工具链](https://swagger.io/)，仅需传入OAS文档，您就能为20多种语言生成服务端和客户端样板代码。

在[之前的博客文章](https://entgo.io/blog/2021/09/10/openapi-generator)中，我们向您展示了Ent扩展[`elk`](https://github.com/masseelch/elk)的一项新功能：一个完全兼容的[OpenAPI规范](https://swagger.io/resources/open-api/)文档生成器。

今天，我们非常高兴地宣布，该规范生成器现已成为Ent项目的官方扩展，并已迁移至[`ent/contrib`](https://github.com/ent/contrib/tree/master/entoas)代码库。此外，我们听取了社区的反馈，对生成器进行了一些改进，希望您会喜欢。

### 快速开始

要使用`entoas`扩展，请按照[此处](https://entgo.io/docs/code-gen#use-entc-as-a-package)所述，使用`entc`（Ent代码生成）包。首先将扩展安装到您的Go模块中：

```shell
go get entgo.io/contrib/entoas
```

接下来按照以下两个步骤启用扩展并配置Ent以配合`entoas`工作：

1\. 新建一个名为`ent/entc.go`的Go文件，并粘贴以下内容：

```go
// +build ignore

package main

import (
	"log"

	"entgo.io/contrib/entoas"
	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
)

func main() {
	ex, err := entoas.NewExtension()
	if err != nil {
		log.Fatalf("creating entoas extension: %v", err)
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

完成这些步骤后，即可从您的模式生成OAS文档！如果您是Ent的新手，想了解更多关于如何连接不同类型的数据库、运行迁移或操作实体的信息，请前往[设置教程](https://entgo.io/docs/tutorial-setup/)。

### 生成OAS文档

生成OAS文档的第一步是创建Ent模式图。为简洁起见，这里提供一个示例模式：

```go title="ent/schema/schema.go"
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

上述代码是Ent描述模式图的方式。在本例中，我们创建了三个实体：Fridge（冰箱）、Compartment（隔间）和Item（物品）。此外，我们还向图中添加了一些边：一个Fridge可以拥有多个Compartment，一个Compartment可以包含多个Item。

现在运行代码生成器：

```shell
go generate ./...
```

除了Ent通常生成的文件外，还会创建一个名为`ent/openapi.json`的新文件。以下是该文件的预览：

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
    "/compartments": {
      "get": {
    [...]
```

如果您有兴趣，可以将其内容复制并粘贴到[Swagger编辑器](https://editor.swagger.io/)中。效果应如下所示：

### 基础配置

我们的API描述尚未反映其功能，但`entoas`允许您修改这一点！打开`ent/entc.go`文件，传入更新后的Fridge API标题和描述：

```go {16-18} title="ent/entc.go"
//go:build ignore
// +build ignore

package main

import (
	"log"

	"entgo.io/contrib/entoas"
	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
)

func main() {
	ex, err := entoas.NewExtension(
		entoas.SpecTitle("Fridge CMS"),
		entoas.SpecDescription("API to manage fridges and their cooled contents. **ICY!**"), 
		entoas.SpecVersion("0.0.1"),
	)
	if err != nil {
		log.Fatalf("creating entoas extension: %v", err)
	}
	err = entc.Generate("./schema", &gen.Config{}, entc.Extensions(ex))
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

重新运行代码生成器将创建更新后的OAS文档。

```json {3-4,10} title="ent/openapi.json"
{
  "info": {
    "title": "Fridge CMS",
    "description": "API to manage fridges and their cooled contents. **ICY!**",
    "termsOfService": "",
    "contact": {},
    "license": {
      "name": ""
    },
    "version": "0.0.1"
  },
  "paths": {
    "/compartments": {
      "get": {
    [...]
```

### 操作配置

有时您不希望为每个节点的每个操作都生成端点。幸运的是，`entoas`允许我们配置要生成和忽略的端点。`entoas`的默认策略是暴露所有路由。您可以将此行为更改为仅暴露明确请求的路由，或者通过使用`entoas.Annotation`告诉`entoas`排除特定操作。策略还可用于启用/禁用子资源操作的生成：

```go {5-10,14-20} title="ent/schema/fridge.go"
// Edges of the Fridge.
func (Fridge) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("compartments", Compartment.Type).
			// Do not generate an endpoint for POST /fridges/{id}/compartments
			Annotations(
				entoas.CreateOperation(
					entoas.OperationPolicy(entoas.PolicyExclude),
				),
			), 
	}
}

// Annotations of the Fridge.
func (Fridge) Annotations() []schema.Annotation {
	return []schema.Annotation{
		// Do not generate an endpoint for DELETE /fridges/{id}
		entoas.DeleteOperation(entoas.OperationPolicy(entoas.PolicyExclude)),
	}
}
```

瞧！这些操作已被移除。

有关`entoas`策略工作原理及其功能的更多信息，请参阅[godoc文档](https://pkg.go.dev/entgo.io/contrib/entoas#Config)。

### 简单模型

默认情况下，`entoas`会为每个端点生成一个响应模式。命名策略详情请参阅[godoc文档](https://pkg.go.dev/entgo.io/contrib/entoas#Config)。

许多用户要求改变这一行为，直接将Ent模式映射到OAS文档。现在您可以通过配置`entoas`实现：

```go {5}
ex, err := entoas.NewExtension(
    entoas.SpecTitle("Fridge CMS"),
    entoas.SpecDescription("API to manage fridges and their cooled contents. **ICY!**"),
    entoas.SpecVersion("0.0.1"),
    entoas.SimpleModels(),
) 
```

### 总结

本文我们宣布了`entoas`——原`elk`的OpenAPI规范生成功能已正式集成至Ent。该特性将Ent的代码生成能力与OpenAPI/Swagger丰富的工具生态系统连接起来。

有问题？需要入门帮助？欢迎加入我们的[Discord服务器](https://discord.gg/qZmPgTE6RX)或[Slack频道](https://entgo.io/docs/slack/)。

:::note[获取更多Ent资讯：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter账号](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 加入[Ent Discord服务器](https://discord.gg/qZmPgTE6RX)

:::