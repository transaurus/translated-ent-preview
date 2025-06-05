---
title: Announcing Versioned Migrations Authoring
author: MasseElch
authorURL: "https://github.com/masseelch"
authorImageURL: "https://avatars.githubusercontent.com/u/12862103?v=4"
image: "https://entgo.io/images/assets/migrate/versioned-share.png"
---

当[Ariel](https://github.com/a8m)在一月底发布Ent v0.10.0时，他[介绍了](2022-01-20-announcing-new-migration-engine.md)一个基于开源项目[Atlas](https://github.com/ariga/atlas)的新迁移引擎。

最初，Atlas支持一种称为"声明式迁移"的数据库管理模式。通过声明式迁移，数据库模式的期望状态作为输入提供给迁移引擎，引擎会规划并执行一系列操作将数据库变更至目标状态。这种方法在云原生基础设施领域已被Kubernetes和Terraform等项目广泛采用。它在许多场景下表现优异，事实上过去几年也很好地服务了Ent框架。但数据库迁移是个极其敏感的领域，许多项目需要更可控的方式。

因此，大多数行业标准解决方案如[Flyway](https://flywaydb.org/)、[Liquibase](https://liquibase.org/)或Go生态常见的[golang-migrate/migrate](https://github.com/golang-migrate/migrate)都支持称为"版本化迁移"的工作流。

版本化迁移（有时称为"变更基础迁移"）不是描述期望状态（"数据库应该是什么样"），而是描述变更本身（"如何达到该状态"）。通常通过创建一组包含所需SQL语句的文件来实现。每个文件都有唯一版本号和变更描述。上述工具能够解析这些迁移文件并按正确顺序应用（部分）文件，从而实现数据库结构的变更。

本文将展示Atlas和Ent新增的一种迁移工作流，我们称之为"版本化迁移编写"。它尝试将声明式方法的简洁表达与版本化迁移的安全性、明确性相结合。用户仍声明期望状态，但Atlas引擎会将从当前状态到新状态的迁移方案写入文件，这些文件可提交至代码仓库、人工微调并通过常规代码审查流程。

我将以`golang-migrate/migrate`为例演示该工作流。

### 开始使用

首先需要确保使用最新版Ent：

```shell
go get -u entgo.io/ent@master
```

Ent生成迁移文件有两种方式：使用实例化Ent客户端，或从解析的模式图中生成变更。本文采用第二种方式，第一种方式可参考[文档](./docs/versioned-migrations#from-client)。

### 生成版本化迁移文件

现在已启用版本化迁移功能，让我们创建一个小型模式并生成初始迁移文件。考虑以下新Ent项目的模式：

```go title="ent/schema/user.go"
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("username"),
	}
}

// Indexes of the User.
func (User) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("username").Unique(),
	}
}

```

如前所述，我们将使用解析的模式图来计算模式与连接数据库间的差异。以下是一个可跟随操作的（半）持久化MySQL Docker容器示例：

```shell
docker run --rm --name ent-versioned-migrations --detach --env MYSQL_ROOT_PASSWORD=pass --env MYSQL_DATABASE=ent -p 3306:3306 mysql
```

操作完成后，可通过`docker stop ent-versioned-migrations`停止并移除容器所有资源。

现在创建加载模式图并生成迁移文件的小函数。新建名为`main.go`的Go文件并复制以下内容：

```go title="main.go"
package main

import (
	"context"
	"log"
	"os"

	"ariga.io/atlas/sql/migrate"
	"entgo.io/ent/dialect/sql"
	"entgo.io/ent/dialect/sql/schema"
	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	// We need a name for the new migration file.
	if len(os.Args) < 2 {
		log.Fatalln("no name given")
	}
	// Create a local migration directory.
	dir, err := migrate.NewLocalDir("migrations")
	if err != nil {
		log.Fatalln(err)
	}
	// Load the graph.
	graph, err := entc.LoadGraph("./ent/schema", &gen.Config{})
	if err != nil {
		log.Fatalln(err)
	}
	tbls, err := graph.Tables()
	if err != nil {
		log.Fatalln(err)
	}
	// Open connection to the database.
	drv, err := sql.Open("mysql", "root:pass@tcp(localhost:3306)/ent")
	if err != nil {
		log.Fatalln(err)
	}
	// Inspect the current database state and compare it with the graph.
	m, err := schema.NewMigrate(drv, schema.WithDir(dir))
	if err != nil {
		log.Fatalln(err)
	}
	if err := m.NamedDiff(context.Background(), os.Args[1], tbls...); err != nil {
		log.Fatalln(err)
	}
}
```

最后只需创建迁移目录并执行上述Go文件：

```shell
mkdir migrations
go run -mod=mod main.go initial
```

此时在`migrations`目录下会生成两个新文件：`<timestamp>_initial.down.sql`和`<timestamp>_initial.up.sql`。其中`x.up.sql`文件用于创建数据库版本`x`，而`x.down.sql`则用于回滚至上一版本。

```sql title="<timestamp>_initial.up.sql"
CREATE TABLE `users` (`id` bigint NOT NULL AUTO_INCREMENT, `username` varchar(191) NOT NULL, PRIMARY KEY (`id`), UNIQUE INDEX `user_username` (`username`)) CHARSET utf8mb4 COLLATE utf8mb4_bin;
```

```sql title="<timestamp>_initial.down.sql"
DROP TABLE `users`;
```

### 执行迁移

要执行这些迁移，请先按照`golang-migrate/migrate`工具的[README说明](https://github.com/golang-migrate/migrate/blob/master/cmd/migrate/README.md)进行安装。然后运行以下命令验证配置是否正确。

```shell
migrate -help
```

```text
Usage: migrate OPTIONS COMMAND [arg...]
       migrate [ -version | -help ]

Options:
  -source          Location of the migrations (driver://url)
  -path            Shorthand for -source=file://path
  -database        Run migrations against this database (driver://url)
  -prefetch N      Number of migrations to load in advance before executing (default 10)
  -lock-timeout N  Allow N seconds to acquire database lock (default 15)
  -verbose         Print verbose logging
  -version         Print version
  -help            Print usage

Commands:
  create [-ext E] [-dir D] [-seq] [-digits N] [-format] NAME
               Create a set of timestamped up/down migrations titled NAME, in directory D with extension E.
               Use -seq option to generate sequential up/down migrations with N digits.
               Use -format option to specify a Go time format string.
  goto V       Migrate to version V
  up [N]       Apply all or N up migrations
  down [N]     Apply all or N down migrations
  drop         Drop everything inside database
  force V      Set version V but don't run migration (ignores dirty state)
  version      Print current migration version
```

现在可以执行初始迁移，将数据库与我们的模式同步：

```shell
migrate -source 'file://migrations' -database 'mysql://root:pass@tcp(localhost:3306)/ent' up
```

```text
<timestamp>/u initial (349.256951ms)
```

### 工作流程

我们将通过两个场景演示版本化迁移的标准工作流：1) 编辑模式图并生成迁移变更；2) 手动创建迁移文件来初始化数据。首先添加Group模式及与User的多对多关系，然后创建包含管理员用户的管理员组。请进行以下修改：

```go title="ent/schema/user.go" {22-28}
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("username"),
	}
}

// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("groups", Group.Type).
			Ref("users"),
	}
}

// Indexes of the User.
func (User) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("username").Unique(),
	}
}
```

```go title="ent/schema/group.go"
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

// Group holds the schema definition for the Group entity.
type Group struct {
	ent.Schema
}

// Fields of the Group.
func (Group) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
	}
}

// Edges of the Group.
func (Group) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("users", User.Type),
	}
}

// Indexes of the Group.
func (Group) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("name").Unique(),
	}
}
```

模式更新后，创建新的迁移文件集。

```shell
go run -mod=mod main.go add_group_schema
```

迁移目录中会再次生成两个新文件：`<timestamp>_add_group_schema.down.sql`和`<timestamp>_add_group_schema.up.sql`。

```sql title="<timestamp>_add_group_schema.up.sql"
CREATE TABLE `groups` (`id` bigint NOT NULL AUTO_INCREMENT, `name` varchar(191) NOT NULL, PRIMARY KEY (`id`), UNIQUE INDEX `group_name` (`name`)) CHARSET utf8mb4 COLLATE utf8mb4_bin;
CREATE TABLE `group_users` (`group_id` bigint NOT NULL, `user_id` bigint NOT NULL, PRIMARY KEY (`group_id`, `user_id`), CONSTRAINT `group_users_group_id` FOREIGN KEY (`group_id`) REFERENCES `groups` (`id`) ON DELETE CASCADE, CONSTRAINT `group_users_user_id` FOREIGN KEY (`user_id`) REFERENCES `users` (`id`) ON DELETE CASCADE) CHARSET utf8mb4 COLLATE utf8mb4_bin;
```

```sql title="<timestamp>_add_group_schema.down.sql"
DROP TABLE `group_users`;
DROP TABLE `groups`;
```

现在你可以选择编辑生成的文件来添加种子数据，或新建专用文件。这里选择后者：

```shell
migrate create -format unix -ext sql -dir migrations seed_admin
```

```text
[...]/ent-versioned-migrations/migrations/<timestamp>_seed_admin.up.sql
[...]/ent-versioned-migrations/migrations/<timestamp>_seed_admin.down.sql
```

此时可编辑这些文件，添加创建管理员组和用户的SQL语句。

```sql title="migrations/<timestamp>_seed_admin.up.sql"
INSERT INTO `groups` (`id`, `name`) VALUES (1, 'Admins');
INSERT INTO `users` (`id`, `username`) VALUES (1, 'admin');
INSERT INTO `group_users` (`group_id`, `user_id`) VALUES (1, 1);
```

```sql title="migrations/<timestamp>_seed_admin.down.sql"
DELETE FROM `group_users` where `group_id` = 1 and `user_id` = 1;
DELETE FROM `groups` where id = 1;
DELETE FROM `users` where id = 1;
```

再次应用迁移即可完成：

```shell
migrate -source file://migrations -database 'mysql://root:pass@tcp(localhost:3306)/ent' up
```

```text
<timestamp>/u add_group_schema (417.434415ms)
<timestamp>/u seed_admin (674.189872ms)
```

### 总结

本文演示了使用Ent版本化迁移与`golang-migate/migrate`的标准工作流。我们创建了示例模式，生成迁移文件并学习如何应用它们，同时掌握了添加自定义迁移文件的方法。

若有疑问或需要入门帮助？欢迎加入我们的[Discord服务器](https://discord.gg/qZmPgTE6RX)或[Slack频道](https://entgo.io/docs/slack/)。

:::note[获取更多Ent资讯：]

- 订阅[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord服务器](https://discord.gg/qZmPgTE6RX)

:::