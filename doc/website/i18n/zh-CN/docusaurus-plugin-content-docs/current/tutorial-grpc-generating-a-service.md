---
id: grpc-generating-a-service
title: Generating a gRPC Service
sidebar_label: Generating a Service
---

从 `ent.Schema` 生成 Protobuf 结构体固然有用，但我们真正的目标是获得一个能够对数据库实体进行增删改查的实际服务端。为此，只需修改一行代码！当我们为 schema 添加 `entproto.Service` 注解时，即告知 `entproto` 代码生成器我们需要生成 gRPC 服务定义，随后 `protoc-gen-entgrpc` 将读取该定义并生成服务实现。编辑 `ent/schema/user.go` 并修改 schema 的 `Annotations`：

```go title="ent/schema/user.go" {4}
func (User) Annotations() []schema.Annotation {
	return []schema.Annotation{
		entproto.Message(),
		entproto.Service(), // <-- add this
	}
}
```

现在重新运行代码生成：

```console
go generate ./...
```

观察 `ent/proto/entpb` 目录下出现的重要变化：

```console
ent/proto/entpb
├── entpb.pb.go
├── entpb.proto
├── entpb_grpc.pb.go
├── entpb_user_service.go
└── generate.go
```

首先，`entproto` 向 `entpb.proto` 添加了服务定义：

```protobuf title="ent/proto/entpb/entpb.proto"
service UserService {
  rpc Create ( CreateUserRequest ) returns ( User );

  rpc Get ( GetUserRequest ) returns ( User );

  rpc Update ( UpdateUserRequest ) returns ( User );

  rpc Delete ( DeleteUserRequest ) returns ( google.protobuf.Empty );

  rpc List ( ListUserRequest ) returns ( ListUserResponse );

  rpc BatchCreate ( BatchCreateUsersRequest ) returns ( BatchCreateUsersResponse );
}
```

此外还新增了两个文件。第一个文件 `entpb_grpc.pb.go` 包含 gRPC 客户端存根和接口定义。打开该文件可以看到（除其他内容外）：

```go title="ent/proto/entpb/entpb_grpc.pb.go"
// UserServiceClient is the client API for UserService service.
//
// For semantics around ctx use and closing/ending streaming RPCs, please
// refer to https://pkg.go.dev/google.golang.org/grpc/?tab=doc#ClientConn.NewStream.
type UserServiceClient interface {
	Create(ctx context.Context, in *CreateUserRequest, opts ...grpc.CallOption) (*User, error)
	Get(ctx context.Context, in *GetUserRequest, opts ...grpc.CallOption) (*User, error)
	Update(ctx context.Context, in *UpdateUserRequest, opts ...grpc.CallOption) (*User, error)
	Delete(ctx context.Context, in *DeleteUserRequest, opts ...grpc.CallOption) (*emptypb.Empty, error)
	List(ctx context.Context, in *ListUserRequest, opts ...grpc.CallOption) (*ListUserResponse, error)
	BatchCreate(ctx context.Context, in *BatchCreateUsersRequest, opts ...grpc.CallOption) (*BatchCreateUsersResponse, error)
}
```

第二个文件 `entpub_user_service.go` 包含该接口的生成实现。例如针对 `Get` 方法的实现：

```go title="ent/proto/entpb/entpb_user_service.go"
// Get implements UserServiceServer.Get
func (svc *UserService) Get(ctx context.Context, req *GetUserRequest) (*User, error) {
	var (
		err error
		get *ent.User
	)
	id := int(req.GetId())
	switch req.GetView() {
	case GetUserRequest_VIEW_UNSPECIFIED, GetUserRequest_BASIC:
		get, err = svc.client.User.Get(ctx, id)
	case GetUserRequest_WITH_EDGE_IDS:
		get, err = svc.client.User.Query().
			Where(user.ID(id)).
			Only(ctx)
	default:
		return nil, status.Error(codes.InvalidArgument, "invalid argument: unknown view")
	}
	switch {
	case err == nil:
		return toProtoUser(get)
	case ent.IsNotFound(err):
		return nil, status.Errorf(codes.NotFound, "not found: %s", err)
	default:
		return nil, status.Errorf(codes.Internal, "internal error: %s", err)
	}
}

```

效果不错！接下来让我们创建能够响应服务请求的 gRPC 服务器。