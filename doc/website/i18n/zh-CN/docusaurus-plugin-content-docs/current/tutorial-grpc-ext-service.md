---
id: grpc-external-service
title: Working with External gRPC Services
sidebar_label: External gRPC Services
---

通常，您会希望在gRPC服务器中包含那些无法从Ent模式自动生成的方法。为实现这一目标，可以在`entpb`目录下的额外`.proto`文件中定义这些方法，并将其作为附加服务声明。

:::info[]

本节描述的变更可参考[此拉取请求](https://github.com/rotemtam/ent-grpc-example/pull/7/files)。

:::

例如，假设需要添加名为`TopUser`的方法来返回ID号最大的用户。为此，在`entpb`目录下创建新的`.proto`文件并定义服务：

```protobuf title="ent/proto/entpb/ext.proto"
syntax = "proto3";

package entpb;

option go_package = "github.com/rotemtam/ent-grpc-example/ent/proto/entpb";

import "entpb/entpb.proto";

import "google/protobuf/empty.proto";


service ExtService {
  rpc TopUser ( google.protobuf.Empty ) returns ( User );
}
```

接着更新`entpb/generate.go`文件，将新文件加入`protoc`命令的输入参数：

```diff title="ent/proto/entpb/generate.go"
- //go:generate protoc -I=.. --go_out=.. --go-grpc_out=.. --go_opt=paths=source_relative --go-grpc_opt=paths=source_relative --entgrpc_out=.. --entgrpc_opt=paths=source_relative,schema_path=../../schema entpb/entpb.proto 
+ //go:generate protoc -I=.. --go_out=.. --go-grpc_out=.. --go_opt=paths=source_relative --go-grpc_opt=paths=source_relative --entgrpc_out=.. --entgrpc_opt=paths=source_relative,schema_path=../../schema entpb/entpb.proto entpb/ext.proto
```

重新运行代码生成命令：

```shell
go generate ./...
```

观察`ent/proto/entpb`目录下生成的新文件：

```shell
tree
.
|-- entpb.pb.go
|-- entpb.proto
|-- entpb_grpc.pb.go
|-- entpb_user_service.go
// highlight-start
|-- ext.pb.go
|-- ext.proto
|-- ext_grpc.pb.go
// highlight-end
`-- generate.go

0 directories, 9 files
```

现在可以在`ent/proto/entpb/ext.go`中实现`TopUser`方法：

```go title="ent/proto/entpb/ext.go"
package entpb

import (
	"context"

	"github.com/rotemtam/ent-grpc-example/ent"
	"github.com/rotemtam/ent-grpc-example/ent/user"
	"google.golang.org/protobuf/types/known/emptypb"
)

// ExtService implements ExtServiceServer.
type ExtService struct {
	client *ent.Client
	UnimplementedExtServiceServer
}

// TopUser returns the user with the highest ID.
func (s *ExtService) TopUser(ctx context.Context, _ *emptypb.Empty) (*User, error) {
	id := s.client.User.Query().Aggregate(ent.Max(user.FieldID)).IntX(ctx)
	user := s.client.User.GetX(ctx, id)
	return toProtoUser(user)
}

// NewExtService returns a new ExtService.
func NewExtService(client *ent.Client) *ExtService {
	return &ExtService{
		client: client,
	}
}

```

### 将新服务添加至gRPC服务器

最后更新`cmd/server.go`以包含新服务：

```go title="cmd/server.go"
package main

import (
	"context"
	"log"
	"net"

	_ "github.com/mattn/go-sqlite3"
	"github.com/rotemtam/ent-grpc-example/ent"
	"github.com/rotemtam/ent-grpc-example/ent/proto/entpb"
	"google.golang.org/grpc"
)

func main() {
	// Initialize an ent client.
	client, err := ent.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	if err != nil {
		log.Fatalf("failed opening connection to sqlite: %v", err)
	}
	defer client.Close()

	// Run the migration tool (creating tables, etc).
	if err := client.Schema.Create(context.Background()); err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}

	// Initialize the generated User service.
	svc := entpb.NewUserService(client)

	// Create a new gRPC server (you can wire multiple services to a single server).
	server := grpc.NewServer()

    // highlight-start
	// Register the User service with the server.
	entpb.RegisterUserServiceServer(server, svc)
	// highlight-end

	// Register the external ExtService service with the server.
	entpb.RegisterExtServiceServer(server, entpb.NewExtService(client))

	// Open port 5000 for listening to traffic.
	lis, err := net.Listen("tcp", ":5000")
	if err != nil {
		log.Fatalf("failed listening: %s", err)
	}

	// Listen for traffic indefinitely.
	if err := server.Serve(lis); err != nil {
		log.Fatalf("server ended: %s", err)
	}
}

```