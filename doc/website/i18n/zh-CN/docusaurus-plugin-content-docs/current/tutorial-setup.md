---
id: tutorial-setup
title: Setting Up
sidebar_label: Setting Up
---

本指南面向首次使用者，提供从零搭建Ent项目的详细说明。开始前请确保已安装以下环境依赖。

## 环境准备

- [Go](https://go.dev/doc/install)
- [Docker](https://docs.docker.com/get-docker) (可选)

安装完依赖后，为项目创建目录并初始化Go模块：

```console
mkdir todo
cd $_
go mod init todo
```

## 安装步骤

执行以下Go命令安装Ent，并初始化项目结构及`Todo`数据模型：

```console
go get entgo.io/ent/cmd/ent
```

```console
go run -mod=mod entgo.io/ent/cmd/ent new Todo
```

完成Ent安装并运行`ent new`后，项目目录结构应如下所示：

```console
.
├── ent
│   ├── generate.go
│   └── schema
│       └── todo.go
├── go.mod
└── go.sum
```

`ent`目录存放生成的代码资产（详见下节），`ent/schema`目录则包含实体数据模型定义。

## 代码生成

前文执行`ent new Todo`时，已在`todo/ent/schema/`目录下创建了名为`Todo`的模型文件`todo.go`：

```go
package schema

import "entgo.io/ent"

// Todo holds the schema definition for the Todo entity.
type Todo struct {
	ent.Schema
}

// Fields of the Todo.
func (Todo) Fields() []ent.Field {
	return nil
}

// Edges of the Todo.
func (Todo) Edges() []ent.Edge {
	return nil
}
```

可见初始模型未定义任何字段或关联关系。现在运行以下命令生成与`Todo`实体交互的代码资产：

```console
go generate ./ent
```

## 创建测试用例

执行`go generate ./ent`会触发Ent的自动代码生成工具，该工具根据`schema`包中的模型定义生成可操作数据库的Go代码。此时在`./ent/client.go`中可找到用于查询和修改`Todo`实体的客户端代码。我们将创建一个[可测试示例](https://go.dev/blog/examples)进行验证，测试中使用[SQLite](https://github.com/mattn/go-sqlite3)作为数据库驱动。

```console
go get github.com/mattn/go-sqlite3
touch example_test.go
```

将以下代码粘贴至`example_test.go`，该代码会实例化`ent.Client`并自动在数据库中创建所有模型资源（表、列等）：

```go
package todo

import (
	"context"
	"log"
	"todo/ent"

	"entgo.io/ent/dialect"
	_ "github.com/mattn/go-sqlite3"
)

func Example_Todo() {
	// Create an ent.Client with in-memory SQLite database.
	client, err := ent.Open(dialect.SQLite, "file:ent?mode=memory&cache=shared&_fk=1")
	if err != nil {
		log.Fatalf("failed opening connection to sqlite: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	// Run the automatic migration tool to create all schema resources.
	if err := client.Schema.Create(ctx); err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
	// Output:
}
```

运行`go test`验证功能是否正常。

```console
go test
```

完成项目初始化后，即可开始创建待办事项列表。