---
id: traversals
title: Graph Traversal
---

出于示例目的，我们将生成以下关系图：

![er-traversal-graph](https://entgo.io/images/assets/er_traversal_graph.png)

第一步是生成三个模式：`Pet`、`User` 和 `Group`。

```console
go run -mod=mod entgo.io/ent/cmd/ent new Pet User Group
```

为这些模式添加必要的字段和边：

`ent/schema/pet.go`

```go
// Pet holds the schema definition for the Pet entity.
type Pet struct {
	ent.Schema
}

// Fields of the Pet.
func (Pet) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
	}
}

// Edges of the Pet.
func (Pet) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("friends", Pet.Type),
		edge.From("owner", User.Type).
			Ref("pets").
			Unique(),
	}
}
``` 

`ent/schema/user.go`

```go
// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("age"),
		field.String("name"),
	}
}

// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("pets", Pet.Type),
		edge.To("friends", User.Type),
		edge.From("groups", Group.Type).
			Ref("users"),
		edge.From("manage", Group.Type).
			Ref("admin"),
	}
}
``` 

`ent/schema/group.go`

```go
// Group holds the schema definition for the Group entity.
type Group struct {
	ent.Schema
}

// Fields of the Group.
func (Group) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
	}
}

// Edges of the Group.
func (Group) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("users", User.Type),
		edge.To("admin", User.Type).
			Unique(),
	}
}
``` 

接下来编写填充顶点和边的代码：

```go
func Gen(ctx context.Context, client *ent.Client) error {
	hub, err := client.Group.
		Create().
		SetName("Github").
		Save(ctx)
	if err != nil {
		return fmt.Errorf("failed creating the group: %w", err)
	}
	// Create the admin of the group.
	// Unlike `Save`, `SaveX` panics if an error occurs.
	dan := client.User.
		Create().
		SetAge(29).
		SetName("Dan").
		AddManage(hub).
		SaveX(ctx)

	// Create "Ariel" and its pets.
	a8m := client.User.
		Create().
		SetAge(30).
		SetName("Ariel").
		AddGroups(hub).
		AddFriends(dan).
		SaveX(ctx)
	pedro := client.Pet.
		Create().
		SetName("Pedro").
		SetOwner(a8m).
		SaveX(ctx)
	xabi := client.Pet.
		Create().
		SetName("Xabi").
		SetOwner(a8m).
		SaveX(ctx)

	// Create "Alex" and its pets.
	alex := client.User.
		Create().
		SetAge(37).
		SetName("Alex").
		SaveX(ctx)
	coco := client.Pet.
		Create().
		SetName("Coco").
		SetOwner(alex).
		AddFriends(pedro).
		SaveX(ctx)

	fmt.Println("Pets created:", pedro, xabi, coco)
	// Output:
	// Pets created: Pet(id=1, name=Pedro) Pet(id=2, name=Xabi) Pet(id=3, name=Coco)
	return nil
}
```

让我们通过几个遍历示例来演示具体实现：

![er-traversal-graph-gopher](https://entgo.io/images/assets/er_traversal_graph_gopher.png)

上述遍历从 `Group` 实体开始，通过其 `admin`（边）继续遍历，接着通过 `friends`（边）遍历，获取它们的 `pets`（边），再获取每只宠物的 `friends`（边），最终请求这些宠物的主人。

```go
func Traverse(ctx context.Context, client *ent.Client) error {
	owner, err := client.Group.			// GroupClient.
		Query().                     	// Query builder.
		Where(group.Name("Github")). 	// Filter only Github group (only 1).
		QueryAdmin().                	// Getting Dan.
		QueryFriends().              	// Getting Dan's friends: [Ariel].
		QueryPets().                 	// Their pets: [Pedro, Xabi].
		QueryFriends().              	// Pedro's friends: [Coco], Xabi's friends: [].
		QueryOwner().                	// Coco's owner: Alex.
		Only(ctx)                    	// Expect only one entity to return in the query.
	if err != nil {
		return fmt.Errorf("failed querying the owner: %w", err)
	}
	fmt.Println(owner)
	// Output:
	// User(id=3, age=37, name=Alex)
	return nil
}
```

那么以下遍历又该如何实现？

![er-traversal-graph-gopher-query](https://entgo.io/images/assets/er_traversal_graph_gopher_query.png)

我们需要获取所有满足以下条件的宠物（实体）：其 `owner`（边）是某个群组 `admin`（边）的 `friend`（边）。

```go
func Traverse(ctx context.Context, client *ent.Client) error {
	pets, err := client.Pet.
		Query().
		Where(
			pet.HasOwnerWith(
				user.HasFriendsWith(
					user.HasManage(),
				),
			),
		).
		All(ctx)
	if err != nil {
		return fmt.Errorf("failed querying the pets: %w", err)
	}
	fmt.Println(pets)
	// Output:
	// [Pet(id=1, name=Pedro) Pet(id=2, name=Xabi)]
	return nil
}
```

完整示例可在 [GitHub](https://github.com/ent/ent/tree/master/examples/traversal) 查看。