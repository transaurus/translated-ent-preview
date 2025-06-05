---
id: schema-mixin
title: Mixin
---

`Mixin` 允许您创建可复用的 `ent.Schema` 代码片段，这些片段可以通过组合方式注入到其他模式中。

`ent.Mixin` 接口定义如下：

```go
type Mixin interface {
	// Fields returns a slice of fields to add to the schema.
	Fields() []Field
	// Edges returns a slice of edges to add to the schema.
	Edges() []Edge
	// Indexes returns a slice of indexes to add to the schema.
	Indexes() []Index
	// Hooks returns a slice of hooks to add to the schema.
	// Note that mixin hooks are executed before schema hooks.
	Hooks() []Hook
	// Policy returns a privacy policy to add to the schema.
	// Note that mixin policy are executed before schema policy.
	Policy() Policy
	// Annotations returns a list of schema annotations to add
	// to the schema annotations.
	Annotations() []schema.Annotation
}
```

## 示例

`Mixin` 的一个常见用例是将公共字段列表混入到您的模式中。

```go
package schema

import (
	"time"

	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/mixin"
)

// -------------------------------------------------
// Mixin definition

// TimeMixin implements the ent.Mixin for sharing
// time fields with package schemas.
type TimeMixin struct{
	// We embed the `mixin.Schema` to avoid
	// implementing the rest of the methods.
	mixin.Schema
}

func (TimeMixin) Fields() []ent.Field {
	return []ent.Field{
		field.Time("created_at").
			Immutable().
			Default(time.Now),
		field.Time("updated_at").
			Default(time.Now).
			UpdateDefault(time.Now),
	}
}

// DetailsMixin implements the ent.Mixin for sharing
// entity details fields with package schemas.
type DetailsMixin struct{
	// We embed the `mixin.Schema` to avoid
	// implementing the rest of the methods.
	mixin.Schema
}

func (DetailsMixin) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age").
			Positive(),
		field.String("name").
			NotEmpty(),
	}
}

// -------------------------------------------------
// Schema definition

// User schema mixed-in the TimeMixin and DetailsMixin fields and therefore
// has 5 fields: `created_at`, `updated_at`, `age`, `name` and `nickname`.
type User struct {
	ent.Schema
}

func (User) Mixin() []ent.Mixin {
	return []ent.Mixin{
		TimeMixin{},
		DetailsMixin{},
	}
}

func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("nickname").
			Unique(),
	}
}

// Pet schema mixed-in the DetailsMixin fields and therefore
// has 3 fields: `age`, `name` and `weight`.
type Pet struct {
	ent.Schema
}

func (Pet) Mixin() []ent.Mixin {
	return []ent.Mixin{
		DetailsMixin{},
	}
}

func (Pet) Fields() []ent.Field {
	return []ent.Field{
		field.Float("weight"),
	}
}
```

## 内置 Mixin

`mixin` 包提供了一些内置的 mixin，可用于向模式添加 `create_time` 和 `update_time` 字段。

要使用它们，请按如下方式将 `mixin.Time` mixin 添加到您的模式中：

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/mixin"
)

type Pet struct {
	ent.Schema
}

func (Pet) Mixin() []ent.Mixin {
	return []ent.Mixin{
		mixin.Time{},
		// Or, mixin.CreateTime only for create_time
		// and mixin.UpdateTime only for update_time.
	}
}
```