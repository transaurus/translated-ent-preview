---
id: schema-indexes
title: Indexes
---

## 多字段索引

索引可配置在一个或多个字段上，以提升数据检索速度或定义唯一性约束。

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/index"
)

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

func (User) Indexes() []ent.Index {
	return []ent.Index{
        // non-unique index.
        index.Fields("field1", "field2"),
        // unique index.
        index.Fields("first_name", "last_name").
            Unique(),
	}
}
```

注意：若需将单个字段设为唯一，直接在字段构建器上调用`Unique`方法即可：

```go
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("phone").
			Unique(),
	}
}
```

## 关联边索引

索引可配置在字段与关联边的组合上，主要用途是在特定关系下设置字段的唯一性。例如：

![城市-街道ER图](https://entgo.io/images/assets/er_city_streets.png)

上例中，一个`City`拥有多条`Street`，我们需要确保街道名称在每个城市下具有唯一性。

`ent/schema/city.go`

```go
// City holds the schema definition for the City entity.
type City struct {
	ent.Schema
}

// Fields of the City.
func (City) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
	}
}

// Edges of the City.
func (City) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("streets", Street.Type),
	}
}
```

`ent/schema/street.go`

```go
// Street holds the schema definition for the Street entity.
type Street struct {
	ent.Schema
}

// Fields of the Street.
func (Street) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
	}
}

// Edges of the Street.
func (Street) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("city", City.Type).
			Ref("streets").
			Unique(),
	}
}

// Indexes of the Street.
func (Street) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("name").
			Edges("city").
			Unique(),
	}
}
```

`example.go`

```go
func Do(ctx context.Context, client *ent.Client) error {
	// Unlike `Save`, `SaveX` panics if an error occurs.
	tlv := client.City.
		Create().
		SetName("TLV").
		SaveX(ctx)
	nyc := client.City.
		Create().
		SetName("NYC").
		SaveX(ctx)
	// Add a street "ST" to "TLV".
	client.Street.
		Create().
		SetName("ST").
		SetCity(tlv).
		SaveX(ctx)
	// This operation fails because "ST"
	// was already created under "TLV".
	if err := client.Street.
		Create().
		SetName("ST").
		SetCity(tlv).
		Exec(ctx); err == nil {
		return fmt.Errorf("expecting creation to fail")
	}
	// Add a street "ST" to "NYC".
	client.Street.
		Create().
		SetName("ST").
		SetCity(nyc).
		SaveX(ctx)
	return nil
}
```

完整示例参见[GitHub仓库](https://github.com/ent/ent/tree/master/examples/edgeindex)。

## 关联边字段索引

当前`Edges`列总是排在`Fields`列之后。但某些索引需要这些列优先排序以实现特定优化。可通过[边字段](schema-edges.mdx#edge-field)功能解决此问题。

```go
// Card holds the schema definition for the Card entity.
type Card struct {
	ent.Schema
}
// Fields of the Card.
func (Card) Fields() []ent.Field {
	return []ent.Field{
		field.String("number").
			Optional(),
		field.Int("owner_id").
			Optional(),
	}
}
// Edges of the Card.
func (Card) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("owner", User.Type).
			Ref("card").
			Field("owner_id").
 			Unique(),
 	}
}
// Indexes of the Card.
func (Card) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("owner_id", "number"),
	}
}
```

## 方言支持

通过[注解](schema-annotations.md)可使用方言特定功能。例如在MySQL中使用[索引前缀](https://dev.mysql.com/doc/refman/8.0/en/column-indexes.html#column-indexes-prefix)需如下配置：

```go
// Indexes of the User.
func (User) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("description").
			Annotations(entsql.Prefix(128)),
		index.Fields("c1", "c2", "c3").
			Annotations(
				entsql.PrefixColumn("c1", 100),
				entsql.PrefixColumn("c2", 200),
			)
	}
}
```

上述代码会生成如下SQL语句：

```sql
CREATE INDEX `users_description` ON `users`(`description`(128))

CREATE INDEX `users_c1_c2_c3` ON `users`(`c1`(100), `c2`(200), `c3`)
```

## Atlas支持

自v0.10起，Ent使用[Atlas](https://github.com/ariga/atlas)执行迁移。该方案提供更精细的索引控制能力，如配置索引类型或定义逆序索引。

```go
func (User) Indexes() []ent.Index {
    return []ent.Index{
        index.Fields("c1").
            Annotations(entsql.Desc()),
        index.Fields("c1", "c2", "c3").
            Annotations(entsql.DescColumns("c1", "c2")),
        index.Fields("c4").
            Annotations(entsql.IndexType("HASH")),
        // Enable FULLTEXT search on MySQL,
        // and GIN on PostgreSQL.
        index.Fields("c5").
            Annotations(
                entsql.IndexTypes(map[string]string{
                    dialect.MySQL:    "FULLTEXT",
                    dialect.Postgres: "GIN",
                }),
            ),
		// For PostgreSQL, we can include in the index
		// non-key columns.
		index.Fields("workplace").
			Annotations(
				entsql.IncludeColumns("address"),
			),
		// Define a partial index on SQLite and PostgreSQL.
		index.Fields("nickname").
			Annotations(
				entsql.IndexWhere("active"),
			),	
		// Define a custom operator class.
		index.Fields("phone").
			Annotations(
				entsql.OpClass("bpchar_pattern_ops"),
			),
    }
}
```

上述代码会生成如下SQL语句：

```sql
CREATE INDEX `users_c1` ON `users` (`c1` DESC)

CREATE INDEX `users_c1_c2_c3` ON `users` (`c1` DESC, `c2` DESC, `c3`)

CREATE INDEX `users_c4` ON `users` USING HASH (`c4`)

-- MySQL only.
CREATE FULLTEXT INDEX `users_c5` ON `users` (`c5`)

-- PostgreSQL only.
CREATE INDEX "users_c5" ON "users" USING GIN ("c5")

-- Include index-only scan on PostgreSQL.
CREATE INDEX "users_workplace" ON "users" ("workplace") INCLUDE ("address")

-- Define partial index on SQLite and PostgreSQL.
CREATE INDEX "users_nickname" ON "users" ("nickname") WHERE "active"

-- PostgreSQL only.
CREATE INDEX "users_phone" ON "users" ("phone" bpchar_pattern_ops)
```

## 函数式索引

Ent模式支持在字段和关联边（外键）上定义索引，但未提供将索引部分定义为表达式（如函数调用）的API。若使用[Atlas](https://atlasgo.io/docs)管理模式迁移，可参考[本指南](/docs/migration/functional-indexes)定义函数式索引。

## 存储键名

与字段类似，可通过`StorageKey`方法配置自定义索引名称，该名称会映射为SQL方言中的索引名。

```go
func (User) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("field1", "field2").
			StorageKey("custom_index"),
	}
}
```