---
id: grpc-service-generation-options
title: Configuring Service Method Generation
sidebar_label: Service Generation Options
---

默认情况下，entproto会为带有`ent.Service()`注解的`ent.Schema`生成若干服务方法。通过在`entproto.Service()`注解中包含`entproto.Methods()`参数，可以自定义方法生成。`entproto.Methods()`接受位标志参数来决定应生成哪些服务方法。可用标志包括：

```go
// Generates a Create gRPC service method for the entproto.Service.
entproto.MethodCreate

// Generates a Get gRPC service method for the entproto.Service.
entproto.MethodGet

// Generates an Update gRPC service method for the entproto.Service.
entproto.MethodUpdate

// Generates a Delete gRPC service method for the entproto.Service.
entproto.MethodDelete

// Generates a List gRPC service method for the entproto.Service.
entproto.MethodList

// Generates a Batch Create gRPC service method for the entproto.Service.
entproto.MethodBatchCreate

// Generates all service methods for the entproto.Service.
// This is the same behavior as not including entproto.Methods.
entproto.MethodAll
```

如需生成包含多个方法的服务，可通过位或运算组合标志。

我们通过修改ent模式来实践这一功能。假设需要禁止gRPC客户端执行数据变更操作，可以修改`ent/schema/user.go`文件：

```go title="ent/schema/user.go" {5}
func (User) Annotations() []schema.Annotation {
	return []schema.Annotation{
		entproto.Message(),
		entproto.Service(
			entproto.Methods(entproto.MethodCreate | entproto.MethodGet | entproto.MethodList | entproto.MethodBatchCreate),
        ),
	}
}
```

重新运行`go generate ./...`后，将在`entpb.proto`中生成如下服务定义：

```protobuf title="ent/proto/entpb/entpb.proto"
service UserService {
  rpc Create ( CreateUserRequest ) returns ( User );

  rpc Get ( GetUserRequest ) returns ( User );

  rpc List ( ListUserRequest ) returns ( ListUserResponse );

  rpc BatchCreate ( BatchCreateUsersRequest ) returns ( BatchCreateUsersResponse );
}
```

可以看到服务定义中已移除`Update`和`Delete`方法。完美符合需求！