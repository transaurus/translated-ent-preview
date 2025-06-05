---
id: grpc-server-and-client
title: Creating the Server and Client
sidebar_label: Server and Client
---

自动生成gRPC服务定义固然很酷，但我们仍需将其注册到具体的gRPC服务器——该服务器需要监听某个TCP端口以接收请求并响应RPC调用。

我们决定不自动生成这部分代码，因为这通常涉及团队/组织特定的行为（例如集成不同的中间件）。未来可能会改变这一设计。目前本节将描述如何创建一个简单的gRPC服务器来承载我们的服务代码。

### 创建服务器

新建文件`cmd/server/main.go`并写入：

```go
package main

import (
	"context"
	"log"
	"net"

	_ "github.com/mattn/go-sqlite3"
	"ent-grpc-example/ent"
	"ent-grpc-example/ent/proto/entpb"
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

	// Register the User service with the server.
	entpb.RegisterUserServiceServer(server, svc)

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

注意我们添加了`github.com/mattn/go-sqlite3`的导入，因此需要将其加入模块：

```console
go get -u github.com/mattn/go-sqlite3
```

接着让我们运行服务器，同时编写与之通信的客户端：

```console
go run -mod=mod ./cmd/server
```

### 创建客户端

让我们创建一个调用服务器的简易客户端。新建文件`cmd/client/main.go`并写入：

```go
package main

import (
	"context"
	"fmt"
	"log"
	"math/rand"
	"time"

	"ent-grpc-example/ent/proto/entpb"
	"google.golang.org/grpc"
	"google.golang.org/grpc/status"
)

func main() {
	rand.Seed(time.Now().UnixNano())

	// Open a connection to the server.
	conn, err := grpc.Dial(":5000", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("failed connecting to server: %s", err)
	}
	defer conn.Close()

	// Create a User service Client on the connection.
	client := entpb.NewUserServiceClient(conn)

	// Ask the server to create a random User.
	ctx := context.Background()
	user := randomUser()
	created, err := client.Create(ctx, &entpb.CreateUserRequest{
		User: user,
	})
	if err != nil {
		se, _ := status.FromError(err)
		log.Fatalf("failed creating user: status=%s message=%s", se.Code(), se.Message())
	}
	log.Printf("user created with id: %d", created.Id)

	// On a separate RPC invocation, retrieve the user we saved previously.
	get, err := client.Get(ctx, &entpb.GetUserRequest{
		Id: created.Id,
	})
	if err != nil {
		se, _ := status.FromError(err)
		log.Fatalf("failed retrieving user: status=%s message=%s", se.Code(), se.Message())
	}
	log.Printf("retrieved user with id=%d: %v", get.Id, get)
}

func randomUser() *entpb.User {
	return &entpb.User{
		Name:         fmt.Sprintf("user_%d", rand.Int()),
		EmailAddress: fmt.Sprintf("user_%d@example.com", rand.Int()),
	}
}
```

客户端首先建立与服务器监听端口5000的连接，然后发起`Create`请求创建新用户，再通过`Get`请求从数据库检索该用户。运行客户端代码：

```console
go run ./cmd/client
```

观察输出结果：

```console
2021/03/18 10:42:58 user created with id: 1
2021/03/18 10:42:58 retrieved user with id=1: id:1 name:"user_730811260095307266" email_address:"user_7338662242574055998@example.com"
```

太棒了！我们成功创建了能与真实gRPC服务器通信的客户端！后续章节将探讨ent/gRPC集成如何处理更高级的ent模式定义。