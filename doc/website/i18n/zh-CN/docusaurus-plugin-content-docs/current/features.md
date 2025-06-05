---
id: feature-flags
title: Feature Flags
sidebar_label: Feature Flags
---

该框架提供了一系列可通过标志位添加或移除的代码生成特性。

## 使用方式

功能标志可通过CLI参数或作为`gen`包的参数提供。

#### 命令行

```console
go run -mod=mod entgo.io/ent/cmd/ent generate --feature privacy,entql ./ent/schema
```

#### Go代码

```go
// +build ignore

package main

import (
	"log"
	"text/template"

	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
)

func main() {
	err := entc.Generate("./schema", &gen.Config{
		Features: []gen.Feature{
			gen.FeaturePrivacy,
			gen.FeatureEntQL,
		},
		Templates: []*gen.Template{
			gen.MustParse(gen.NewTemplate("static").
				Funcs(template.FuncMap{"title": strings.ToTitle}).
				ParseFiles("template/static.tmpl")),
		},
	})
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

## 功能列表

### 自动解决合并冲突

`schema/snapshot`选项指示ent代码生成器(entc)将最新模式的快照存储到内部包中，并在用户模式无法构建时自动解决合并冲突。

可通过`--feature schema/snapshot`标志启用此功能，详情请参阅[ent/ent/issues/852](https://github.com/ent/ent/issues/852)。

### 隐私层

隐私层支持为数据库实体的查询和变更配置隐私策略。

通过`--feature privacy`标志启用，详见[隐私文档](privacy.mdx)。

### EntQL过滤

`entql`选项为不同查询构建器提供运行时通用动态过滤能力。

通过`--feature entql`标志启用，详见[隐私文档](privacy.mdx#multi-tenancy)。

### 命名边

`namedges`选项提供通过自定义名称预加载边的API。

通过`--feature namedges`标志启用，详见[预加载文档](eager-load.mdx)。

### 双向边引用

`bidiedges`选项指导Ent在预加载(O2M/O2O)边时设置双向引用。

通过`--feature bidiedges`标志启用。

:::note
使用标准encoding/json.MarshalJSON的用户应在调用`json.Marshal`前解除循环引用。
:::

### 模式配置

`sql/schemaconfig`选项允许为模型传递备用SQL数据库名称，适用于模型分散在不同数据库的场景。

通过`--feature sql/schemaconfig`标志启用后，可使用新选项如下：

```go
c, err := ent.Open(dialect, conn, ent.AlternateSchema(ent.SchemaConfig{
	User: "usersdb",
	Car: "carsdb",
}))
c.User.Query().All(ctx) // SELECT * FROM `usersdb`.`users`
c.Car.Query().All(ctx) 	// SELECT * FROM `carsdb`.`cars`
```

### 行级锁

`sql/lock`选项支持使用SQL`SELECT ... FOR {UPDATE | SHARE}`语法配置行级锁定。

通过`--feature sql/lock`标志启用。

```go
tx, err := client.Tx(ctx)
if err != nil {
	log.Fatal(err)
}

tx.Pet.Query().
	Where(pet.Name(name)).
	ForUpdate().
	Only(ctx)

tx.Pet.Query().
	Where(pet.ID(id)).
	ForShare(
		sql.WithLockTables(pet.Table),
		sql.WithLockAction(sql.NoWait),
	).
	Only(ctx)
```

### 自定义SQL修饰器

`sql/modifier`选项允许向构建器添加自定义SQL修饰器，在执行前修改语句。

通过`--feature sql/modifier`标志启用。

#### 修改示例1

```go
client.Pet.
	Query().
	Modify(func(s *sql.Selector) {
		s.Select("SUM(LENGTH(name))")
	}).
	IntX(ctx)
```

上述代码将生成如下SQL查询：

```sql
SELECT SUM(LENGTH(name)) FROM `pet`
```

#### 选择并扫描动态值

若需处理SQL修饰符并扫描Ent模式定义中未包含的动态值（如聚合或自定义排序），可对`sql.Selector`应用`AppendSelect`/`AppendSelectAs`方法，随后通过实体上的`Value`方法访问这些值：

```go {6,11}
const as = "name_length"

// Query the entity with the dynamic value.
p := client.Pet.Query().
	Modify(func(s *sql.Selector) {
		s.AppendSelectAs("LENGTH(name)", as)
	}).
	FirstX(ctx)

// Read the value from the entity.
n, err := p.Value(as)
if err != nil {
    log.Fatal(err)
}
fmt.Println("Name length: %d == %d", n, len(p.Name))
```

#### 修改示例2

```go
var p1 []struct {
	ent.Pet
	NameLength int `sql:"length"`
}

client.Pet.Query().
	Order(ent.Asc(pet.FieldID)).
	Modify(func(s *sql.Selector) {
		s.AppendSelect("LENGTH(name)")
	}).
	ScanX(ctx, &p1)
```

上述代码将生成如下SQL查询：

```sql
SELECT `pet`.*, LENGTH(name) FROM `pet` ORDER BY `pet`.`id` ASC
```

#### 修改示例3

```go
var v []struct {
	Count     int       `json:"count"`
	Price     int       `json:"price"`
	CreatedAt time.Time `json:"created_at"`
}

client.User.
	Query().
	Where(
        user.CreatedAtGT(x),
        user.CreatedAtLT(y),
	).
	Modify(func(s *sql.Selector) {
		s.Select(
			sql.As(sql.Count("*"), "count"),
			sql.As(sql.Sum("price"), "price"),
			sql.As("DATE(created_at)", "created_at"),
		).
		GroupBy("DATE(created_at)").
		OrderBy(sql.Desc("DATE(created_at)"))
	}).
	ScanX(ctx, &v)
```

上述代码将生成如下SQL查询：

```sql
SELECT
    COUNT(*) AS `count`,
    SUM(`price`) AS `price`,
    DATE(created_at) AS `created_at`
FROM
    `users`
WHERE
    `created_at` > x AND `created_at` < y
GROUP BY
    DATE(created_at)
ORDER BY
    DATE(created_at) DESC
```

#### 修改示例4

```go
var gs []struct {
	ent.Group
	UsersCount int `sql:"users_count"`
}

client.Group.Query().
	Order(ent.Asc(group.FieldID)).
	Modify(func(s *sql.Selector) {
		t := sql.Table(group.UsersTable)
		s.LeftJoin(t).
			On(
				s.C(group.FieldID),
				t.C(group.UsersPrimaryKey[1]),
			).
			// Append the "users_count" column to the selected columns.
			AppendSelect(
				sql.As(sql.Count(t.C(group.UsersPrimaryKey[1])), "users_count"),
			).
			GroupBy(s.C(group.FieldID))
	}).
	ScanX(ctx, &gs)
```

上述代码将生成如下SQL查询：

```sql
SELECT
    `groups`.*,
    COUNT(`t1`.`group_id`) AS `users_count`
FROM
    `groups` LEFT JOIN `user_groups` AS `t1`
ON
    `groups`.`id` = `t1`.`group_id`
GROUP BY
    `groups`.`id`
ORDER BY
    `groups`.`id` ASC
```

#### 修改示例5

```go
client.User.Update().
	Modify(func(s *sql.UpdateBuilder) {
		s.Set(user.FieldName, sql.Expr(fmt.Sprintf("UPPER(%s)", user.FieldName)))
	}).
	ExecX(ctx)
```

上述代码将生成如下SQL查询：

```sql
UPDATE `users` SET `name` = UPPER(`name`)
```

#### 修改示例6

```go
client.User.Update().
	Modify(func(u *sql.UpdateBuilder) {
		u.Set(user.FieldID, sql.ExprFunc(func(b *sql.Builder) {
			b.Ident(user.FieldID).WriteOp(sql.OpAdd).Arg(1)
		}))
		u.OrderBy(sql.Desc(user.FieldID))
	}).
	ExecX(ctx)
```

上述代码将生成如下SQL查询：

```sql
UPDATE `users` SET `id` = `id` + 1 ORDER BY `id` DESC
```

#### 修改示例7

向JSON列中的`values`数组追加元素：

```go
client.User.Update().
	Modify(func(u *sql.UpdateBuilder) {
        sqljson.Append(u, user.FieldTags, []string{"tag1", "tag2"}, sqljson.Path("values"))
	}).
	ExecX(ctx)
```

上述代码将生成如下SQL查询：

```sql
UPDATE `users` SET `tags` = CASE
    WHEN (JSON_TYPE(JSON_EXTRACT(`tags`, '$.values')) IS NULL OR JSON_TYPE(JSON_EXTRACT(`tags`, '$.values')) = 'NULL')
    THEN JSON_SET(`tags`, '$.values', JSON_ARRAY(?, ?))
    ELSE JSON_ARRAY_APPEND(`tags`, '$.values', ?, '$.values', ?) END
    WHERE `id` = ?
```

### SQL原生API

`sql/execquery`选项支持通过底层驱动的`ExecContext`/`QueryContext`方法执行语句。完整文档参见：[DB.ExecContext](https://pkg.go.dev/database/sql#DB.ExecContext)与[DB.QueryContext](https://pkg.go.dev/database/sql#DB.QueryContext)。

```go
// From ent.Client.
if _, err := client.ExecContext(ctx, "TRUNCATE t1"); err != nil {
	return err
}

// From ent.Tx.
tx, err := client.Tx(ctx)
if err != nil {
	return err
}
if err := tx.User.Create().Exec(ctx); err != nil {
	return err
}
if _, err := tx.ExecContext("SAVEPOINT user_created"); err != nil {
	return err
}
// ...
```

:::warning[注意]
通过`ExecContext`/`QueryContext`执行的语句不会经过Ent处理，可能绕过应用程序的基础层（如钩子、隐私鉴权、验证器等）。
:::

### 原子更新插入

`sql/upsert`选项支持使用SQL的`ON CONFLICT`/`ON DUPLICATE KEY`语法配置更新插入与批量更新插入逻辑。完整文档参见：[更新插入API](crud.mdx#upsert-one)。

可通过`--feature sql/upsert`标志为项目启用此功能。

```go
// Use the new values that were set on create.
id, err := client.User.
	Create().
	SetAge(30).
	SetName("Ariel").
	OnConflict().
	UpdateNewValues().
	ID(ctx)

// In PostgreSQL, the conflict target is required.
err := client.User.
	Create().
	SetAge(30).
	SetName("Ariel").
	OnConflictColumns(user.FieldName).
	UpdateNewValues().
	Exec(ctx)

// Bulk upsert is also supported.
client.User.
	CreateBulk(builders...).
	OnConflict(
		sql.ConflictWhere(...),
		sql.UpdateWhere(...),
	).
	UpdateNewValues().
	Exec(ctx)

// INSERT INTO "users" (...) VALUES ... ON CONFLICT WHERE ... DO UPDATE SET ... WHERE ...
```

### 全局唯一ID

默认情况下，SQL主键从每张表的1开始递增，这意味着不同类型的实体可能共享相同ID。这与AWS Neptune（节点ID采用UUID）不同。

此机制不适用于需要对象ID全局唯一的[GraphQL](https://graphql.org/learn/schema/#scalar-types)场景。

启用全局ID支持只需使用`--feature sql/globalid`标志。

:::warning[注意]
若曾使用过`migrate.WithGlobalUniqueID(true)`迁移选项，在切换至新globalid功能前请阅读[本指南](globalid-migrate)。
:::

**实现原理**：Ent迁移为每个实体类型（表）分配1<<32的ID区间，并将该信息存储在生成代码中（`internal/globalid.go`）。例如类型`A`的ID区间为`[1,4294967296)`，类型`B`为`[4294967296,8589934592)`，以此类推。

请注意，若启用此选项，最大支持的表数量为**65535**。