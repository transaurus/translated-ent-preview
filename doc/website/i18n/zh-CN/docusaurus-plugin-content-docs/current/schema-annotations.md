---
id: schema-annotations
title: Annotations
---

Schema注解允许将元数据附加到字段和边等Schema对象上，并将其注入到外部模板中。注解必须是可序列化为JSON原始值的Go类型（如结构体、映射或切片），并实现[Annotation](https://pkg.go.dev/entgo.io/ent/schema?tab=doc#Annotation)接口。

内置注解可用于配置不同存储驱动（如SQL）并控制代码生成输出。

## 自定义表名

可以通过`entsql`注解为类型指定自定义表名，示例如下：

```go title="ent/schema/user.go"
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/dialect/entsql"
	"entgo.io/ent/schema"
	"entgo.io/ent/schema/field"
)

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Annotations of the User.
func (User) Annotations() []schema.Annotation {
	return []schema.Annotation{
		entsql.Annotation{Table: "Users"},
	}
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age"),
		field.String("name"),
	}
}
```

## 自定义表Schema

使用[Atlas](https://atlasgo.io)迁移引擎时，Ent Schema可以跨多个数据库Schema进行定义和管理。更多信息请参阅[多Schema迁移文档](multischema-migrations.mdx)。

## 外键配置

Ent允许自定义外键创建行为，并为`ON DELETE`子句指定[引用操作](https://dev.mysql.com/doc/refman/8.0/en/create-table-foreign-keys.html#foreign-key-referential-actions)：

```go title="ent/schema/user.go" {27}
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/dialect/entsql"
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
		field.String("name").
			Default("Unknown"),
	}
}

// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("posts", Post.Type).
			Annotations(entsql.OnDelete(entsql.Cascade)),
	}
}
```

上例配置了级联删除外键，当父表记录被删除时，会自动删除子表中的匹配记录。

## 数据库注释

默认情况下，表和列注释不会存储到数据库中。但可以通过`WithComments(true)`注解启用此功能，例如：

```go title="ent/schema/user.go" {18-21,34-37}
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/dialect/entsql"
	"entgo.io/ent/schema"
	"entgo.io/ent/schema/field"
)

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Annotations of the User.
func (User) Annotations() []schema.Annotation {
	return []schema.Annotation{
		// Adding this annotation to the schema enables
		// comments for the table and all its fields.
		entsql.WithComments(true),
		schema.Comment("Comment that appears in both the schema and the generated code"),
	}
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			Comment("The user's name"),
		field.Int("age").
            Comment("The user's age"),
        field.String("skipped").
            Comment("This comment won't be stored in the database").
            // Explicitly disable comments for this field.
            Annotations(
                entsql.WithComments(false),
            ),
	}
}
```