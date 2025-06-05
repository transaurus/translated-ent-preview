---
id: schema-def
title: Introduction
---

## 快速概览

Schema（模式）用于描述图中某一实体类型的定义，例如 `User` 或 `Group`，
可包含以下配置项：

- 实体字段（或属性），例如：`User` 的姓名或年龄
- 实体边（或关联关系），例如：`User` 所属的群组或好友
- 数据库特定选项，例如：索引或唯一索引

以下是一个模式定义示例：

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/index"
)

type User struct {
	ent.Schema
}

func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age"),
		field.String("name"),
		field.String("nickname").
			Unique(),
	}
}

func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("groups", Group.Type),
		edge.To("friends", User.Type),
	}
}

func (User) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("age", "name").
			Unique(),
	}
}
```

实体模式通常存放在项目根目录下的 `ent/schema` 目录中，
可通过 `entc` 工具生成：

```console
go run -mod=mod entgo.io/ent/cmd/ent new User Group
```

:::note[]
请注意，部分模式名称（如 `Client`）因[内部使用](https://pkg.go.dev/entgo.io/ent/entc/gen#ValidSchemaName)而不可用。可通过[模式注解](schema-annotations.md#custom-table-name)中提到的注解方式规避保留名称。
:::

## 这只是另一种ORM

如果您习惯通过边（edges）来定义关联关系，这完全可行。
其建模方式与传统ORM相同。您可以用 `ent` 实现任何传统ORM能实现的模型。
本网站[边（Edges）](schema-edges.mdx)章节提供了大量示例帮助您快速入门。