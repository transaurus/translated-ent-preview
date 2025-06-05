---
title: Announcing Edge-field Support in v0.7.0
author: Rotem Tamir
authorURL: "https://github.com/rotemtam"
authorImageURL: "https://s.gravatar.com/avatar/36b3739951a27d2e37251867b7d44b1a?s=80"
authorTwitter: _rtam
---

过去几个月，Ent项目的[issues](https://github.com/ent/ent/issues)中有大量关于为一对一或一对多边关系实体检索添加外键字段支持的讨论。我们很高兴地宣布，从[v0.7.0](https://github.com/ent/ent/releases/tag/v0.7.0)版本开始，ent已支持该功能。

### 支持边字段之前

在合并该分支前，用户若想获取实体的外键字段，必须使用预加载机制。假设我们的模式如下：

```go
// ent/schema/user.go:

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			Unique().
			NotEmpty(),
	}
}

// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("pets", Pet.Type).
			Ref("owner"),
	}
}

// ent/schema/pet.go

// Pet holds the schema definition for the Pet entity.
type Pet struct {
	ent.Schema
}

// Fields of the Pet.
func (Pet) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			NotEmpty(),
	}
}

// Edges of the Pet.
func (Pet) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("owner", User.Type).
			Unique().
			Required(),
	}
}
```

该模式描述了两个关联实体：`User`和`Pet`，它们之间存在一对多边关系：一个用户可以拥有多只宠物，而一只宠物只能有一个主人。

从数据存储中检索宠物时，开发者通常需要访问宠物上的外键字段。但由于该字段是由`owner`边隐式创建的，检索实体时无法直接获取。开发者需要通过如下方式从存储中获取：

```go
func Test(t *testing.T) {
    ctx := context.Background()
	c := enttest.Open(t, dialect.SQLite, "file:ent?mode=memory&cache=shared&_fk=1")
	defer c.Close()
	
	// Create the User
	u := c.User.Create().
		SetUserName("rotem").
		SaveX(ctx)

	// Create the Pet
	p := c.Pet.
		Create().
		SetOwner(u). // Associate with the user
		SetName("donut").
		SaveX(ctx)

	petWithOwnerId := c.Pet.Query().
		Where(pet.ID(p.ID)).
		WithOwner(func(query *ent.UserQuery) {
			query.Select(user.FieldID)
		}).
		OnlyX(ctx)
	fmt.Println(petWithOwnerId.Edges.Owner.ID)
	// Output: 1
}
```

这种方式不仅冗长，而且从数据库查询效率来看也较低效。若使用`.Debug()`执行查询，可以看到ent为满足该调用生成的数据库查询：

```sql
SELECT DISTINCT `pets`.`id`, `pets`.`name`, `pets`.`pet_owner` FROM `pets` WHERE `pets`.`id` = ? LIMIT 2 
SELECT DISTINCT `users`.`id` FROM `users` WHERE `users`.`id` IN (?)
```

此例中，Ent首先获取ID为`1`的Pet，然后冗余地从`users`表中再次获取ID为`1`用户的`id`字段。

### 边字段支持方案

[Edge-field support](https://entgo.io/docs/schema-edges/#edge-field) greatly simplifies and improves the efficiency of this flow. With this feature, developers can define the foreign key field as part of the schemas `Fields()`, and by using the `.Field(..)` modifier on the edge definition instruct Ent to expose and map the foreign column to this field.  So, in our example schema, we would modify it to be:

```go
// user.go stays the same

// pet.go
// Fields of the Pet.
func (Pet) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			NotEmpty(),
		field.Int("owner_id"), // <-- explicitly add the field we want to contain the FK
	}
}

// Edges of the Pet.
func (Pet) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("owner", User.Type).
			Field("owner_id"). // <-- tell ent which field holds the reference to the owner
			Unique().
			Required(),
	}
}
```

更新客户端代码前需重新运行代码生成：

```sql
go generate ./...
```

现在我们可以将查询简化为：

```go
func Test(t *testing.T) {
	ctx := context.Background()
	c := enttest.Open(t, dialect.SQLite, "file:ent?mode=memory&cache=shared&_fk=1")
	defer c.Close()

	u := c.User.Create().
		SetUserName("rotem").
		SaveX(ctx)

	p := c.Pet.Create().
		SetOwner(u).
		SetName("donut").
		SaveX(ctx)

	petWithOwnerId := c.Pet.GetX(ctx, p.ID) // <-- Simply retrieve the Pet

	fmt.Println(petWithOwnerId.OwnerID)
	// Output: 1
}
```

使用`.Debug()`修饰符运行时，可见数据库查询逻辑现已合理：

```sql
SELECT DISTINCT `pets`.`id`, `pets`.`name`, `pets`.`owner_id` FROM `pets` WHERE `pets`.`id` = ? LIMIT 2
```

太棒了 🎉！

### 现有模式迁移至边字段

若您已在现有模式中使用Ent，可能已存在外键列的一对多关系。根据模式配置方式，这些列可能以不同于新添加字段的名称存储。例如您想创建`owner_id`字段，但Ent自动生成的外键列名为`pet_owner`。

可通过查看`./ent/migrate/schema.go`文件确认Ent使用的列名：

```go
PetsColumns = []*schema.Column{
	{Name: "id", Type: field.TypeInt, Increment: true},
	{Name: "name", Type: field.TypeString},
	{Name: "pet_owner", Type: field.TypeInt, Nullable: true}, // <-- this is our FK
}
```

为实现平滑迁移，必须显式告知Ent继续使用现有列名。可通过在字段或边上使用`StorageKey`修饰符实现，例如：

```go
// In schema/pet.go:

// Fields of the Pet.
func (Pet) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			NotEmpty(),
		field.Int("owner_id").
			StorageKey("pet_owner"), // <-- explicitly set the column name
	}
}
```

近期我们计划实现模式版本控制功能，将模式变更历史与代码共同存储。该信息将使ent能以自动化且可预测的方式支持此类迁移。

### 总结

边字段支持功能已正式发布，可通过`go get -u entgo.io/ent@v0.7.0`安装使用。

衷心感谢以下各位抽出时间提供反馈并协助完善此功能设计： [Alex Snast](https://github.com/alexsn)、[Ruben de Vries](https://github.com/rubensayshi)、[Marwan Sulaiman](https://github.com/marwan-at-work)、[Andy Day](https://github.com/adayNU)、[Sebastian Fekete](https://github.com/aight8) 以及 [Joe Harvey](https://github.com/errorhandler)。🙏

### 获取更多Ent资讯与更新：

- 关注我们的Twitter账号 [twitter.com/entgo_io](https://twitter.com/entgo_io)
- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 加入[Gophers Slack](https://app.slack.com/client/T029RQSE6/C01FMSQDT53)的#ent频道
- 参与[Ent Discord社区](https://discord.gg/qZmPgTE6RX)