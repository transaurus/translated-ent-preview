---
id: generating-ent-schemas 
title: Generating Schemas
---

## 简介

为便于开发能编程式生成 `ent.Schema` 的工具链，`ent` 支持通过 `entgo.io/contrib/schemast` 包操作 `schema/` 目录。

## API接口

### 加载

要操作现有模式目录，需先将其加载到 `schemast.Context` 对象中：

```go
package main

import (
	"fmt"
	"log"

	"entgo.io/contrib/schemast"
)

func main() {
	ctx, err := schemast.Load("./ent/schema")
	if err != nil {
		log.Fatalf("failed: %v", err)
	}
	if ctx.HasType("user") {
		fmt.Println("schema directory contains a schema named User!")
	}
}
```

### 输出

使用 `schemast.Print` 将上下文打印到目标目录：

```go
package main

import (
	"log"

	"entgo.io/contrib/schemast"
)

func main() {
	ctx, err := schemast.Load("./ent/schema")
	if err != nil {
		log.Fatalf("failed: %v", err)
	}
	// A no-op since we did not manipulate the Context at all.
	if err := schemast.Print("./ent/schema"); err != nil {
		log.Fatalf("failed: %v", err)
	}
}
```

### 变更器

要修改 `ent/schema` 目录，可使用 `schemast.Mutate` 方法，该方法接收一组应用于上下文的 `schemast.Mutator`：

```go
package schemast

// Mutator changes a Context.
type Mutator interface {
	Mutate(ctx *Context) error
}
```

当前仅实现了一种 `schemast.Mutator` 类型，即 `UpsertSchema`：

```go
package schemast

// UpsertSchema implements Mutator. UpsertSchema will add to the Context the type named
// Name if not present and rewrite the type's Fields, Edges, Indexes and Annotations methods.
type UpsertSchema struct {
	Name        string
	Fields      []ent.Field
	Edges       []ent.Edge
	Indexes     []ent.Index
	Annotations []schema.Annotation
}
```

使用方法：

```go
package main

import (
	"log"

	"entgo.io/contrib/schemast"
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
)

func main() {
	ctx, err := schemast.Load("./ent/schema")
	if err != nil {
		log.Fatalf("failed: %v", err)
	}
	mutations := []schemast.Mutator{
		&schemast.UpsertSchema{
			Name: "User",
			Fields: []ent.Field{
				field.String("name"),
			},
		},
		&schemast.UpsertSchema{
			Name: "Team",
			Fields: []ent.Field{
				field.String("name"),
			},
		},
	}
	err = schemast.Mutate(ctx, mutations...)
	if err := ctx.Print("./ent/schema"); err != nil {
		log.Fatalf("failed: %v", err)
	}
}
```

运行程序后，可观察到模式目录中新增两个文件：`user.go` 和 `team.go`：

```go
// user.go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema"
	"entgo.io/ent/schema/field"
)

type User struct {
	ent.Schema
}

func (User) Fields() []ent.Field {
	return []ent.Field{field.String("name")}
}
func (User) Edges() []ent.Edge {
	return nil
}
func (User) Annotations() []schema.Annotation {
	return nil
}
```

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema"
	"entgo.io/ent/schema/field"
)

type Team struct {
	ent.Schema
}

func (Team) Fields() []ent.Field {
	return []ent.Field{field.String("name")}
}
func (Team) Edges() []ent.Edge {
	return nil
}
func (Team) Annotations() []schema.Annotation {
	return nil
}
```

### 处理边关系

`ent` 中边的定义方式如下：

```go
edge.To("edge_name", OtherSchema.Type)
```

此语法依赖于定义边时 `OtherSchema` 结构体已存在，因此可引用其 `Type` 方法。当以编程方式生成模式时，显然需要在类型定义存在前向代码生成器描述边关系。可通过如下方式实现：

```go
type placeholder struct {
    ent.Schema
}

func withType(e ent.Edge, typeName string) ent.Edge {
    e.Descriptor().Type = typeName
    return e
}

func newEdgeTo(edgeName, otherType string) ent.Edge {
    // we pass a placeholder type to the edge constructor:
    e := edge.To(edgeName, placeholder.Type)
    // then we override the other type's name directly on the edge descriptor: 
    return withType(e, otherType)
}
```

## 示例

`protoc-gen-ent` ([文档](https://github.com/ent/contrib/tree/master/entproto/cmd/protoc-gen-ent)) 是一个 protoc 插件，它从 .proto 文件编程式生成 `ent.Schema`，使用 `schemast` 操作目标 `schema` 目录。具体实现可[查阅源码](https://github.com/ent/contrib/blob/master/entproto/cmd/protoc-gen-ent/main.go#L34)。

## 注意事项

`schemast` 仍处于实验阶段，API 未来可能变更。此外，当前暂不支持部分 `ent.Field` 定义 API 功能，完整特性支持情况请参见[源码](https://github.com/ent/contrib/blob/aed7a43a3e54550c1dd9a1a066ce1236b4bae56c/schemast/field.go#L158)。