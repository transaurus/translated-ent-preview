---
id: grpc-edges
title: Working with Edges
sidebar_label: Working with Edges
---

边（Edges）使我们能够在ent应用中表达不同实体之间的关系。让我们看看它们如何与生成的gRPC服务协同工作。

首先我们添加一个新实体`Category`，并创建与`User`类型关联的边：

```go title="ent/schema/category.go"
package schema

import (
	"entgo.io/contrib/entproto"
	"entgo.io/ent"
	"entgo.io/ent/schema"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
)

type Category struct {
	ent.Schema
}

func (Category) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			Annotations(entproto.Field(2)),
	}
}

func (Category) Annotations() []schema.Annotation {
	return []schema.Annotation{
		entproto.Message(),
	}
}

func (Category) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("admin", User.Type).
			Unique().
			Annotations(entproto.Field(3)),
	}
}
```

在`User`端创建反向关系：

```go title="ent/schema/user.go" {4-6}
// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("administered", Category.Type).
			Ref("admin").
			Annotations(entproto.Field(5)),
	}
}
```

注意以下几点：

* 我们的边也带有`entproto.Field`注解，稍后会解释原因
* 我们创建了一对多关系：一个`Category`有单个`admin`，而一个`User`可以管理多个分类

通过`go generate ./...`重新生成项目后，观察`.proto`文件的变化：

```protobuf title="ent/proto/entpb/entpb.proto" {1-7,18}
message Category {
  int64 id = 1;

  string name = 2;

  User admin = 3;
}

message User {
  int64 id = 1;

  string name = 2;

  string email_address = 3;

  google.protobuf.StringValue alias = 4;

  repeated Category administered = 5;
}
```

主要变更包括：

* 新增了`Category`消息类型，其中包含与模式中`admin`边对应的`admin`字段。由于该边被标记为`.Unique()`，这是一个非重复字段，其字段编号`3`对应边定义中的`entproto.Field`注解
* `User`消息定义中新增了`administered`字段。由于该方向未标记为`Unique`，这是一个重复字段，其字段编号`5`同样对应边上的注解

### 创建带边的实体

通过编写测试来演示如何创建带边的实体：

```go
package main

import (
	"context"
	"testing"

	_ "github.com/mattn/go-sqlite3"

	"ent-grpc-example/ent/category"
	"ent-grpc-example/ent/enttest"
	"ent-grpc-example/ent/proto/entpb"
	"ent-grpc-example/ent/user"
)

func TestServiceWithEdges(t *testing.T) {
	// start by initializing an ent client connected to an in memory sqlite instance
	ctx := context.Background()
	client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	defer client.Close()

	// next, initialize the UserService. Notice we won't be opening an actual port and
	// creating a gRPC server and instead we are just calling the library code directly. 
	svc := entpb.NewUserService(client)

	// next, we create a category directly using the ent client.
	// Notice we are initializing it with no relation to a User.
	cat := client.Category.Create().SetName("cat_1").SaveX(ctx)

	// next, we invoke the User service's `Create` method. Notice we are
	// passing a list of entpb.Category instances with only the ID set. 
	create, err := svc.Create(ctx, &entpb.CreateUserRequest{
		User: &entpb.User{
			Name:         "user",
			EmailAddress: "user@service.code",
			Administered: []*entpb.Category{
				{Id: int64(cat.ID)},
			},
		},
	})
	if err != nil {
		t.Fatal("failed creating user using UserService", err)
	}

	// to verify everything worked correctly, we query the category table to check
	// we have exactly one category which is administered by the created user.
	count, err := client.Category.
		Query().
		Where(
			category.HasAdminWith(
				user.ID(int(create.Id)),
			),
		).
		Count(ctx)
	if err != nil {
		t.Fatal("failed counting categories admin by created user", err)
	}
	if count != 1 {
		t.Fatal("expected exactly one group to managed by the created user")
	}
}
```

要从已创建的`User`关联到现有`Category`，我们不需要填充整个`Category`对象，只需填充`Id`字段即可，生成的服务代码会自动处理：

```go title="ent/proto/entpb/entpb_user_service.go" {3-6}
func (svc *UserService) createBuilder(user *User) (*ent.UserCreate, error) {
	  // truncated ...
	for _, item := range user.GetAdministered() {
		administered := int(item.GetId())
		m.AddAdministeredIDs(administered)
	}
	return m, nil
}
```

### 获取实体的边ID

我们已经了解如何创建实体间关系，但如何从生成的gRPC服务中获取这些数据？

参考以下测试示例：

```go
func TestGet(t *testing.T) {
	// start by initializing an ent client connected to an in memory sqlite instance
	ctx := context.Background()
	client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	defer client.Close()

	// next, initialize the UserService. Notice we won't be opening an actual port and
	// creating a gRPC server and instead we are just calling the library code directly.
	svc := entpb.NewUserService(client)

	// next, create a user, a category and set that user to be the admin of the category
	user := client.User.Create().
		SetName("rotemtam").
		SetEmailAddress("r@entgo.io").
		SaveX(ctx)

	client.Category.Create().
		SetName("category").
		SetAdmin(user).
		SaveX(ctx)

	// next, retrieve the user without edge information
	get, err := svc.Get(ctx, &entpb.GetUserRequest{
		Id: int64(user.ID),
	})
	if err != nil {
		t.Fatal("failed retrieving the created user", err)
	}
	if len(get.Administered) != 0 {
		t.Fatal("by default edge information is not supposed to be retrieved")
	}

	// next, retrieve the user *WITH* edge information
	get, err = svc.Get(ctx, &entpb.GetUserRequest{
		Id:   int64(user.ID),
		View: entpb.GetUserRequest_WITH_EDGE_IDS,
	})
	if err != nil {
		t.Fatal("failed retrieving the created user", err)
	}
	if len(get.Administered) != 1 {
		t.Fatal("using WITH_EDGE_IDS edges should be returned")
	}
}
```

如测试所示，默认情况下服务的`Get`方法不会返回边信息。这是有意为之，因为实体关联的其他实体数量可能是无限的。为了让调用方指定是否返回边信息，生成的服务遵循[AIP-157](https://google.aip.dev/157)（部分响应规范）。具体来说，`GetUserRequest`消息包含一个名为`View`的枚举：

```protobuf title="ent/proto/entpb/entpb.proto"
message GetUserRequest {
  int64 id = 1;

  View view = 2;

  enum View {
    VIEW_UNSPECIFIED = 0;

    BASIC = 1;

    WITH_EDGE_IDS = 2;
  }
}
```

观察生成的`Get`方法代码：

```go title="ent/proto/entpb/entpb_user_service.go"
// Get implements UserServiceServer.Get
func (svc *UserService) Get(ctx context.Context, req *GetUserRequest) (*User, error) {
	// .. truncated ..
	switch req.GetView() {
	case GetUserRequest_VIEW_UNSPECIFIED, GetUserRequest_BASIC:
		get, err = svc.client.User.Get(ctx, int(req.GetId()))
	case GetUserRequest_WITH_EDGE_IDS:
		get, err = svc.client.User.Query().
			Where(user.ID(int(req.GetId()))).
			WithAdministered(func(query *ent.CategoryQuery) {
				query.Select(category.FieldID)
			}).
			Only(ctx)
	default:
		return nil, status.Errorf(codes.InvalidArgument, "invalid argument: unknown view")
	}
// .. truncated ..
}
```

默认调用`client.User.Get`不会返回任何边ID信息，但如果传入`WITH_EDGE_IDS`，端点将获取通过`administered`边与用户关联的所有`Category`的`ID`字段。