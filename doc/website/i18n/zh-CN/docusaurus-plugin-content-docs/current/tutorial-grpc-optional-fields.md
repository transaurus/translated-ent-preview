---
id: grpc-optional-fields
title: Optional Fields
sidebar_label: Optional Fields
---

Protobuf 的一个常见问题是其空值表示方式：原始类型字段的零值不会被编码到二进制表示中，这意味着应用程序无法区分原始字段的零值与未设置状态。

为了解决这个问题，Protobuf 项目支持称为"包装类型"的[已知类型](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf)。例如，布尔值的包装类型名为`google.protobuf.BoolValue`，其[定义如下](https://github.com/protocolbuffers/protobuf/blob/991bcada050d7e9919503adef5b52547ec249d35/src/google/protobuf/wrappers.proto#L103-L107)：

```protobuf title="ent/proto/entpb/entpb.proto"
// Wrapper message for `bool`.
//
// The JSON representation for `BoolValue` is JSON `true` and `false`.
message BoolValue {
  // The bool value.
  bool value = 1;
}
```

当`entproto`生成 Protobuf 消息定义时，会使用这些包装类型来表示 Ent 中的"可选"字段。

让我们通过修改 Ent 模式添加可选字段来观察这一机制：

```go title="ent/schema/user.go" {14-16}
// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			Unique().
			Annotations(
				entproto.Field(2),
			),
		field.String("email_address").
			Unique().
			Annotations(
				entproto.Field(3),
			),
		field.String("alias").
			Optional().
			Annotations(entproto.Field(4)),
	}
}
```

重新运行`go generate ./...`后，可以看到生成的`User` Protobuf 定义变为：

```protobuf title="ent/proto/entpb/entpb.proto" {8}
message User {
  int32 id = 1;

  string name = 2;

  string email_address = 3;

  google.protobuf.StringValue alias = 4; // <-- this is new 

  repeated Category administered = 5;
}
```

生成的服务实现同样会利用这个字段。观察`entpb_user_service.go`文件：

```go title="ent/proto/entpb/entpb_user_service.go" {3-6}
func (svc *UserService) createBuilder(user *User) (*ent.UserCreate, error) {
	m := svc.client.User.Create()
	if user.GetAlias() != nil {
		userAlias := user.GetAlias().GetValue()
		m.SetAlias(userAlias)
	}
	userEmailAddress := user.GetEmailAddress()
	m.SetEmailAddress(userEmailAddress)
	userName := user.GetName()
	m.SetName(userName)
	for _, item := range user.GetAdministered() {
		administered := int(item.GetId())
		m.AddAdministeredIDs(administered)
	}
	return m, nil
}
```

在客户端代码中使用包装类型时，可以通过[wrapperspb](https://github.com/protocolbuffers/protobuf-go/blob/3f51f05e40d61e930a5416f1ed7092cef14cc058/types/known/wrapperspb/wrappers.pb.go#L458-L460)包提供的辅助方法轻松构建这些类型的实例。例如在`cmd/client/main.go`中：

```go {5}
func randomUser() *entpb.User {
	return &entpb.User{
		Name:         fmt.Sprintf("user_%d", rand.Int()),
		EmailAddress: fmt.Sprintf("user_%d@example.com", rand.Int()),
		Alias:        wrapperspb.String("John Doe"),
	}
}
```