---
id: grpc-setting-up
title: Setting Up
sidebar_label: Setting Up 
---

首先，我们为项目初始化一个新的Go模块：

```console
mkdir ent-grpc-example
cd ent-grpc-example
go mod init ent-grpc-example
```

接着，使用`go run`调用ent代码生成器来初始化一个模式：

```console
go run -mod=mod entgo.io/ent/cmd/ent new User
```

此时项目目录结构应如下所示：

```console
.
├── ent
│   ├── generate.go
│   └── schema
│       └── user.go
├── go.mod
└── go.sum
```

接下来，将`entproto`包添加到项目中：

```console
go get -u entgo.io/contrib/entproto
```

现在我们将定义`User`实体的模式。打开`ent/schema/user.go`进行编辑：

```go title="ent/schema/user.go"
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
)

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			Unique(),
		field.String("email_address").
			Unique(),
	}
}
```

这一步中，我们为`User`实体添加了两个唯一字段：`name`和`email_address`。`ent.Schema`仅是模式的定义。要从中生成可用的生产代码，需要运行Ent的代码生成工具。执行：

```console
go generate ./...
```

注意此时已根据模式定义生成了新文件：

```console
├── ent
│   ├── client.go
│   ├── config.go
// .... many more
│   ├── user
│   ├── user.go
│   ├── user_create.go
│   ├── user_delete.go
│   ├── user_query.go
│   └── user_update.go
├── go.mod
└── go.sum
```

至此，我们可以连接数据库、运行迁移创建`users`表并开始读写数据。这部分内容在[设置教程](tutorial-setup.md)中有详细说明，现在让我们直接进入主题，学习如何从模式生成Protobuf定义和gRPC服务器。