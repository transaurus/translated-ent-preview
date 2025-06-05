---
title: Generating Ent Schemas from Existing SQL Databases 
author: Zeev Manilovich
authorURL: "https://github.com/zeevmoney"
authorImageURL: "https://avatars.githubusercontent.com/u/7361100?v=4"
---

数月前，Ent项目发布了[Schema Import Initiative](https://entgo.io/blog/2021/05/04/announcing-schema-imports)，旨在支持从外部资源生成Ent模式的各种用例。今天，我很高兴分享我一直在开发的项目：**entimport**——一个双关语命名（importent）的命令行工具，用于从现有SQL数据库创建Ent模式。这是社区期待已久的功能，希望能帮助更多人。它可以简化从其他语言或ORM迁移到Ent的过程，也能实现跨平台数据自动同步等场景。  
当前版本支持MySQL和PostgreSQL数据库（存在部分限制），对其他关系型数据库如SQLite的支持正在开发中。

## 快速开始

为演示`entimport`的工作原理，我将通过MySQL数据库展示端到端的使用流程。整体步骤如下：

1. 创建数据库和模式 - 首先构建一个现有数据库环境，定义可供导入Ent的数据表结构  
2. 初始化Ent项目 - 使用Ent CLI创建目录结构和模式生成脚本  
3. 安装`entimport`  
4. 对演示数据库运行`entimport` - 将已创建的数据库模式导入Ent项目  
5. 讲解如何配合生成的模式使用Ent

现在开始逐步操作。

### 创建数据库

我们将使用[Docker](https://docs.docker.com/get-docker/)容器创建数据库环境。通过`docker-compose`文件自动配置MySQL容器参数。  

在新建的`entimport-example`目录中创建`docker-compose.yaml`文件，内容如下：

Start the project in a new directory called `entimport-example`. Create a file named `docker-compose.yaml` and paste the
following content inside:

```yaml
version: "3.7"

services:

  mysql8:
    platform: linux/amd64
    image: mysql
    environment:
      MYSQL_DATABASE: entimport
      MYSQL_ROOT_PASSWORD: pass
    healthcheck:
      test: mysqladmin ping -ppass
    ports:
      - "3306:3306"
```

该文件包含MySQL容器的服务配置，通过以下命令启动：

```shell
docker-compose up -d
```

接下来创建包含两个实体关系的简单模式：

- 用户(User)  
- 汽车(Car)

使用MySQL shell连接数据库（确保在项目根目录执行）：

> 注意需在项目根目录运行

```shell
docker-compose exec mysql8 mysql --database=entimport -ppass
```

```sql
create table users
(
    id        bigint auto_increment primary key,
    age       bigint       not null,
    name      varchar(255) not null,
    last_name varchar(255) null comment 'surname'
);

create table cars
(
    id          bigint auto_increment primary key,
    model       varchar(255) not null,
    color       varchar(255) not null,
    engine_size mediumint    not null,
    user_id     bigint       null,
    constraint cars_owners foreign key (user_id) references users (id) on delete set null
);
```

验证是否成功创建表结构，在MySQL shell中执行：

```sql
show tables;
+---------------------+
| Tables_in_entimport |
+---------------------+
| cars                |
| users               |
+---------------------+
```

应看到`users`和`cars`两张表。

### 初始化Ent项目

完成数据库准备后，我们需要创建包含Ent的[Go](https://golang.org/doc/install)项目。此阶段将建立Ent目录结构以适配后续导入的模式。  

在`entimport-example`目录初始化Go项目：

Initialize a new Go project inside a directory called `entimport-example`

```shell
go mod init entimport-example
```

执行Ent初始化命令：

```shell
go run -mod=mod entgo.io/ent/cmd/ent new 
```

项目结构应如下所示：

```
├── docker-compose.yaml
├── ent
│   ├── generate.go
│   └── schema
└── go.mod
```

### 安装entimport

好了，现在有趣的部分开始了！我们终于可以安装 `entimport` 并见证它的实际运作了。  
首先运行 `entimport`：

```shell
go run -mod=mod ariga.io/entimport/cmd/entimport -h
```

`entimport` 将被下载，命令行会输出：

```
Usage of entimport:
  -dialect string
        database dialect (default "mysql")
  -dsn string
        data source name (connection information)
  -schema-path string
        output path for ent schema (default "./ent/schema")
  -tables value
        comma-separated list of tables to inspect (all if empty)
```

### 运行 entimport

现在我们可以将 MySQL 数据库模式导入到 Ent 了！

使用以下命令完成导入：

> 该命令会导入模式中的所有表，也可以通过 `-tables` 标志指定特定表。

```shell
go run ariga.io/entimport/cmd/entimport -dialect mysql -dsn "root:pass@tcp(localhost:3306)/entimport"
```

与许多 Unix 工具类似，`entimport` 在成功运行时不会输出任何信息。要验证其是否正常运行，我们需要检查文件系统，特别是 `ent/schema` 目录。

```console {5-6}
├── docker-compose.yaml
├── ent
│   ├── generate.go
│   └── schema
│       ├── car.go
│       └── user.go
├── go.mod
└── go.sum
```

让我们看看结果——记得我们有两个模式：`users` 模式和 `cars` 模式，它们之间存在一对多关系。现在来看看 `entimport` 的表现。

```go title="entimport-example/ent/schema/user.go"
type User struct {
	ent.Schema
}

func (User) Fields() []ent.Field {
	return []ent.Field{field.Int("id"), field.Int("age"), field.String("name"), field.String("last_name").Optional().Comment("surname")}
}
func (User) Edges() []ent.Edge {
	return []ent.Edge{edge.To("cars", Car.Type)}
}
func (User) Annotations() []schema.Annotation {
	return nil
}
```

```go title="entimport-example/ent/schema/car.go"
type Car struct {
	ent.Schema
}

func (Car) Fields() []ent.Field {
	return []ent.Field{field.Int("id"), field.String("model"), field.String("color"), field.Int32("engine_size"), field.Int("user_id").Optional()}
}
func (Car) Edges() []ent.Edge {
	return []ent.Edge{edge.From("user", User.Type).Ref("cars").Unique().Field("user_id")}
}
func (Car) Annotations() []schema.Annotation {
	return nil
}
```

> **`entimport` 成功创建了实体及其关系！**

目前看来一切顺利，现在让我们实际试用一下。首先需要生成 Ent 模式。之所以这样做，是因为 Ent 是一个**模式优先**的 ORM，它会为不同数据库[生成](https://entgo.io/docs/code-gen)用于交互的 Go 代码。

运行 Ent 代码生成：

```shell
go generate ./ent
```

查看 `ent` 目录：

```
...
├── ent
│   ├── car
│   │   ├── car.go
│   │   └── where.go
...
│   ├── schema
│   │   ├── car.go
│   │   └── user.go
...
│   ├── user
│   │   ├── user.go
│   │   └── where.go
...
```

### Ent 示例

快速运行一个示例来验证我们的模式是否正常工作：

在项目根目录创建名为 `example.go` 的文件，内容如下：

> 这部分示例可在[此处](https://github.com/zeevmoney/entimport-example/blob/master/part1/example.go)找到

```go title="entimport-example/example.go"
package main

import (
	"context"
	"fmt"
	"log"

	"entimport-example/ent"

	"entgo.io/ent/dialect"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	client, err := ent.Open(dialect.MySQL, "root:pass@tcp(localhost:3306)/entimport?parseTime=True")
	if err != nil {
		log.Fatalf("failed opening connection to mysql: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	example(ctx, client)
}
```

尝试添加用户，在文件末尾写入以下代码：

```go title="entimport-example/example.go"
func example(ctx context.Context, client *ent.Client) {
	// Create a User.
	zeev := client.User.
		Create().
		SetAge(33).
		SetName("Zeev").
		SetLastName("Manilovich").
		SaveX(ctx)
	fmt.Println("User created:", zeev)
}
```

然后运行：

```shell
go run example.go
```

输出应为：

`# 用户已创建: User(id=1, age=33, name=Zeev, last_name=Manilovich)`

检查数据库确认用户是否真的被添加

```sql
SELECT *
FROM users
WHERE name = 'Zeev';

+--+---+----+----------+
|id|age|name|last_name |
+--+---+----+----------+
|1 |33 |Zeev|Manilovich|
+--+---+----+----------+
```

很好！现在让我们进一步使用 Ent 并添加一些关系，在 `example()` 函数末尾添加以下代码：

> 确保在 import() 声明中添加 `"entimport-example/ent/user"`

```go title="entimport-example/example.go"
// Create Car.
vw := client.Car.
    Create().
    SetModel("volkswagen").
    SetColor("blue").
    SetEngineSize(1400).
    SaveX(ctx)
fmt.Println("First car created:", vw)

// Update the user - add the car relation.
client.User.Update().Where(user.ID(zeev.ID)).AddCars(vw).SaveX(ctx)

// Query all cars that belong to the user.
cars := zeev.QueryCars().AllX(ctx)
fmt.Println("User cars:", cars)

// Create a second Car.
delorean := client.Car.
    Create().
    SetModel("delorean").
    SetColor("silver").
    SetEngineSize(9999).
    SaveX(ctx)
fmt.Println("Second car created:", delorean)

// Update the user - add another car relation.
client.User.Update().Where(user.ID(zeev.ID)).AddCars(delorean).SaveX(ctx)

// Traverse the sub-graph.
cars = delorean.
    QueryUser().
    QueryCars().
    AllX(ctx)
fmt.Println("User cars:", cars)
```

> 这部分示例可在[此处](https://github.com/zeevmoney/entimport-example/blob/master/part2/example.go)找到

现在执行：`go run example.go`。  
运行上述代码后，数据库中将存在一个用户与两辆汽车的 O2M（一对多）关系。

```sql
SELECT *
FROM users;

+--+---+----+----------+
|id|age|name|last_name |
+--+---+----+----------+
|1 |33 |Zeev|Manilovich|
+--+---+----+----------+

SELECT *
FROM cars;

+--+----------+------+-----------+-------+
|id|model     |color |engine_size|user_id|
+--+----------+------+-----------+-------+
|1 |volkswagen|blue  |1400       |1      |
|2 |delorean  |silver|9999       |1      |
+--+----------+------+-----------+-------+
```

### 同步数据库变更

为了保持数据库同步，我们需要让 `entimport` 能够在数据库变更后更新模式。下面演示其工作方式。

运行以下 SQL 代码，向 `users` 表添加带有 `unique` 索引的 `phone` 列：

```sql
alter table users
    add phone varchar(255) null;

create unique index users_phone_uindex
    on users (phone);
```

表结构应变为：

```sql
describe users;
+-----------+--------------+------+-----+---------+----------------+
| Field     | Type         | Null | Key | Default | Extra          |
+-----------+--------------+------+-----+---------+----------------+
| id        | bigint       | NO   | PRI | NULL    | auto_increment |
| age       | bigint       | NO   |     | NULL    |                |
| name      | varchar(255) | NO   |     | NULL    |                |
| last_name | varchar(255) | YES  |     | NULL    |                |
| phone     | varchar(255) | YES  | UNI | NULL    |                |
+-----------+--------------+------+-----+---------+----------------+
```

现在让我们再次运行`entimport`以从数据库获取最新架构：

```shell
go run -mod=mod ariga.io/entimport/cmd/entimport -dialect mysql -dsn "root:pass@tcp(localhost:3306)/entimport"
```

可以看到`user.go`文件已被修改：

```go title="entimport-example/ent/schema/user.go"
func (User) Fields() []ent.Field {
	return []ent.Field{field.Int("id"), ..., field.String("phone").Optional().Unique()}
}
```

现在我们可以再次运行`go generate ./ent`，并使用新架构为User实体添加`phone`字段。

## 未来计划

如上所述，初始版本支持MySQL和PostgreSQL数据库。  
同时支持所有类型的SQL关系。我计划进一步升级该工具，添加缺失的PostgreSQL字段、默认值等功能。

## 总结

本文介绍了Ent社区期待已久的`entimport`工具，并通过示例展示了如何将其与Ent结合使用。该工具是Ent架构导入工具集的又一补充，旨在简化Ent的集成流程。如需讨论或支持，请[提交issue](https://github.com/ariga/entimport/issues/new)。完整示例可[在此处获取](https://github.com/zeevmoney/entimport-example)。希望本文对您有所帮助！

:::note[获取更多Ent资讯：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter账号](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord社区](https://discord.gg/qZmPgTE6RX)

:::