---
id: code-gen
title: Introduction
---

## 安装

该项目附带一个名为`ent`的代码生成工具。要安装`ent`，请运行以下命令：

```bash
go get entgo.io/ent/cmd/ent
``` 

## 初始化新模型

要生成一个或多个模型模板，按如下方式运行`ent init`：

```bash
go run -mod=mod entgo.io/ent/cmd/ent new User Pet
```

`init`命令将在`ent/schema`目录下创建2个模型文件（`user.go`和`pet.go`）。如果`ent`目录不存在，该命令会自动创建。项目约定是在根目录下创建`ent`目录。

## 生成资源文件

添加若干[字段](schema-fields.mdx)和[边](schema-edges.mdx)后，您需要生成实体操作所需的资源文件。在项目根目录下运行`ent generate`，或使用`go generate`：

```bash
go generate ./ent
```

`generate`命令会为模型生成以下资源：

- 用于操作图的`Client`和`Tx`对象
- 每个模型类型的CRUD构建器，详见[CRUD](crud.mdx)
- 每个模型对应的实体对象（Go结构体）
- 包含构建器操作所需的常量和谓词的包
- 支持SQL方言的`migrate`包，详见[迁移](migrate.md)
- 用于添加变更中间件的`hook`包，详见[钩子](hooks.md)

## `entc`与`ent`的版本兼容性

在项目中使用`ent` CLI时，必须确保CLI使用的版本与项目中`ent`的版本完全一致。

实现方式之一是让`go generate`在运行`ent`时使用`go.mod`文件中指定的版本。如果项目未使用[Go modules](https://github.com/golang/go/wiki/Modules#quick-start)，请按以下方式初始化：

```console
go mod init <project>
```

然后重新运行以下命令将`ent`添加到`go.mod`文件：

```console
go get entgo.io/ent/cmd/ent
```

在`<project>/ent`目录下创建`generate.go`文件：

```go
package ent

//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate ./schema
```

最后，在项目根目录运行`go generate ./ent`即可对项目模型执行`ent`代码生成。

## 代码生成选项

运行`ent generate -h`查看代码生成选项详情：

```console
generate go code for the schema directory

Usage:
  ent generate [flags] path

Examples:
  ent generate ./ent/schema
  ent generate github.com/a8m/x

Flags:
      --feature strings       extend codegen with additional features
      --header string         override codegen header
  -h, --help                  help for generate
      --storage string        storage driver to support in codegen (default "sql")
      --target string         target directory for codegen
      --template strings      external templates to execute
```

## 存储选项

`ent`可为SQL和Gremlin方言生成资源文件，默认使用SQL方言。

## 外部模板

`ent`支持执行外部Go模板。若模板名称与`ent`内置模板重复，将覆盖原有模板；否则会将执行输出写入与模板同名的文件。标志格式支持`file`、`dir`和`glob`模式：

```console
go run -mod=mod entgo.io/ent/cmd/ent generate --template <dir-path> --template glob="path/to/*.tmpl" ./ent/schema
```

更多信息和示例请参阅[外部模板文档](templates.md)。

## 以包形式使用`entc`

另一种运行`ent`代码生成的方式是：先创建包含以下内容的`ent/entc.go`文件，再通过`ent/generate.go`文件执行：

```go title="ent/entc.go"
// +build ignore

package main

import (
	"log"

	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
	"entgo.io/ent/schema/field"
)

func main() {
	if err := entc.Generate("./schema", &gen.Config{}); err != nil {
		log.Fatal("running ent codegen:", err)
	}
}
```

```go title="ent/generate.go"
package ent

//go:generate go run -mod=mod entc.go
```

完整示例可在 [GitHub](https://github.com/ent/ent/tree/master/examples/entcpkg) 上查看。

## 模式描述

如需获取图模式的描述信息，请运行：

```bash
go run -mod=mod entgo.io/ent/cmd/ent describe ./ent/schema
```

输出示例如下：

```console
Pet:
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	| Field |  Type   | Unique | Optional | Nillable | Default | UpdateDefault | Immutable |       StructTag       | Validators |
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	| id    | int     | false  | false    | false    | false   | false         | false     | json:"id,omitempty"   |          0 |
	| name  | string  | false  | false    | false    | false   | false         | false     | json:"name,omitempty" |          0 |
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	+-------+------+---------+---------+----------+--------+----------+
	| Edge  | Type | Inverse | BackRef | Relation | Unique | Optional |
	+-------+------+---------+---------+----------+--------+----------+
	| owner | User | true    | pets    | M2O      | true   | true     |
	+-------+------+---------+---------+----------+--------+----------+
	
User:
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	| Field |  Type   | Unique | Optional | Nillable | Default | UpdateDefault | Immutable |       StructTag       | Validators |
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	| id    | int     | false  | false    | false    | false   | false         | false     | json:"id,omitempty"   |          0 |
	| age   | int     | false  | false    | false    | false   | false         | false     | json:"age,omitempty"  |          0 |
	| name  | string  | false  | false    | false    | false   | false         | false     | json:"name,omitempty" |          0 |
	+-------+---------+--------+----------+----------+---------+---------------+-----------+-----------------------+------------+
	+------+------+---------+---------+----------+--------+----------+
	| Edge | Type | Inverse | BackRef | Relation | Unique | Optional |
	+------+------+---------+---------+----------+--------+----------+
	| pets | Pet  | false   |         | O2M      | false  | true     |
	+------+------+---------+---------+----------+--------+----------+
```

## 代码生成钩子

`entc` 包支持通过钩子（中间件）列表来扩展代码生成阶段。此功能适用于为模式添加自定义验证器，或基于图模式生成额外资产。

```go
// +build ignore

package main

import (
	"fmt"
	"log"
	"reflect"

	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
)

func main() {
	err := entc.Generate("./schema", &gen.Config{
		Hooks: []gen.Hook{
			EnsureStructTag("json"),
		},
	})
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}

// EnsureStructTag ensures all fields in the graph have a specific tag name.
func EnsureStructTag(name string) gen.Hook {
	return func(next gen.Generator) gen.Generator {
		return gen.GenerateFunc(func(g *gen.Graph) error {
			for _, node := range g.Nodes {
				for _, field := range node.Fields {
					tag := reflect.StructTag(field.StructTag)
					if _, ok := tag.Lookup(name); !ok {
						return fmt.Errorf("struct tag %q is missing for field %s.%s", name, node.Name, field.Name)
					}
				}
			}
			return next.Generate(g)
		})
	}
}
```

## 外部依赖

若需扩展 `ent` 包下的生成客户端和构建器，并注入外部依赖作为结构体字段，请在 [`ent/entc.go`](#use-entc-as-a-package) 文件中使用 `entc.Dependency` 选项：

```go title="ent/entc.go" {3-12}
func main() {
	opts := []entc.Option{
		entc.Dependency(
			entc.DependencyType(&http.Client{}),
		),
		entc.Dependency(
			entc.DependencyName("Writer"),
			entc.DependencyTypeInfo(&field.TypeInfo{
				Ident:   "io.Writer",
				PkgPath: "io",
			}),
		),
	}
	if err := entc.Generate("./schema", &gen.Config{}, opts...); err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

随后在应用程序中使用：

```go title="example_test.go" {5-6,15-16}
func Example_Deps() {
	client, err := ent.Open(
		"sqlite3",
		"file:ent?mode=memory&cache=shared&_fk=1",
		ent.Writer(os.Stdout),
		ent.HTTPClient(http.DefaultClient),
	)
	if err != nil {
		log.Fatalf("failed opening connection to sqlite: %v", err)
	}
	defer client.Close()
	// An example for using the injected dependencies in the generated builders.
	client.User.Use(func(next ent.Mutator) ent.Mutator {
		return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
			_ = m.HTTPClient
			_ = m.Writer
			return next.Mutate(ctx, m)
		})
	})
	// ...
}
```

完整示例可在 [GitHub](https://github.com/ent/ent/tree/master/examples/entcpkg) 上查看。

## 特性开关

`entc` 包提供了一系列可通过标志位添加或移除的代码生成特性。

更多信息请参阅 [特性开关页面](features.md)。