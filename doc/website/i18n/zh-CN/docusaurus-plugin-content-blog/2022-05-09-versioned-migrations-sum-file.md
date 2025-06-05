---
title: Versioned Migrations Management and Migration Directory Integrity
author: Jannik Clausen (MasseElch)
authorURL: "https://github.com/masseelch"
authorImageURL: "https://avatars.githubusercontent.com/u/12862103?v=4"
image: "https://entgo.io/images/assets/migrate/atlas-validate.png"
---

五周前我们发布了Ent期待已久的功能：**版本化迁移**。在[发布博客](2022-03-14-announcing-versioned-migrations.md)中，我们简要介绍了声明式和基于变更的数据库模式同步方法及其缺陷，以及为什么[Atlas](https://atlasgo.io)（Ent底层的迁移引擎）尝试将两者的优势结合到一个工作流中是值得尝试的。我们称之为**版本化迁移编写**，如果你还没读过，现在正是时候！

通过版本化迁移编写，生成的迁移文件仍然是"基于变更的"，但由Atlas引擎安全规划。这意味着你仍然可以使用喜欢的迁移管理工具，如[Flyway](https://flywaydb.org/)、[Liquibase](https://liquibase.org/)、[golang-migrate/migrate](https://github.com/golang-migrate/migrate)或[pressly/goose](https://github.com/pressly/goose)来开发基于Ent的服务。

在这篇博客中，我想展示Atlas项目的另一个新功能**迁移目录完整性文件**（现已在Ent中支持），以及如何与你已经习惯并喜欢的任何迁移管理工具结合使用。

### 问题所在

使用版本化迁移时，开发者需要注意以下事项以避免破坏数据库：

1. 对已执行的迁移进行追溯性修改
2. 意外改变迁移文件的组织顺序
3. 提交语义错误的SQL脚本
理论上代码审查应该能防止团队合并存在这些问题的迁移。但根据我的经验，许多错误会逃过人眼审查，因此自动化预防机制更为可靠。

第一个问题（修改历史）大多数管理工具通过将已应用迁移文件的哈希值保存到数据库中进行比对来解决。如果不匹配则中止迁移。但这发生在开发周期非常晚的阶段（部署期间），如果能在更早阶段检测到将节省时间和资源。

对于第二个（和第三个）问题，考虑以下场景：

![atlas-versioned-migrations-no-conflict](https://entgo.io/images/assets/migrate/no-conflict-2.svg)

该图展示了两个未被检测到的错误。第一个是迁移文件的顺序问题。

团队A和团队B几乎同时从主分支切出特性分支。团队B生成时间戳为**x**的迁移文件并继续开发。团队A在较晚时间点生成迁移文件，因此获得时间戳**x+1**。团队A完成特性并合并到主分支，可能已自动部署包含版本**x+1**迁移的生产环境。目前没有问题。

现在团队B合并其包含时间戳**x**（早于已应用的**x+1**版本）的迁移文件。如果代码审查未发现这一点，迁移文件将进入生产环境，此时具体行为取决于所使用的迁移管理工具。

大多数工具有自己的解决方案，例如`pressly/goose`采用他们称为[混合版本控制](https://github.com/pressly/goose/issues/63#issuecomment-428681694)的方法。在介绍Atlas（Ent）的独特解决方案之前，我们先快速看下第三个问题：

如果团队A和团队B同时开发需要新增表或列的功能，且为它们指定了相同的名称（例如`users`），那么双方都可能生成创建该表的SQL语句。虽然先合并代码的团队能成功执行迁移，但后合并的团队会因表或列已存在而导致迁移失败。

### 解决方案

Atlas采用独特方式处理上述问题，其核心目标是尽早发现问题。我们认为最佳实践是在版本控制和持续集成(CI)环节进行拦截。Atlas的解决方案是引入名为**迁移目录完整性文件**的新机制，该文件命名为`atlas.sum`，与迁移文件共同存储，包含迁移目录的元数据。其格式借鉴了Go模块的`go.sum`文件，示例如下：

```text
h1:KRFsSi68ZOarsQAJZ1mfSiMSkIOZlMq4RzyF//Pwf8A=
20220318104614_team_A.sql h1:EGknG5Y6GQYrc4W8e/r3S61Aqx2p+NmQyVz/2m8ZNwA=
```

`atlas.sum`文件首行为整个目录的校验和，随后是每个迁移文件的哈希值（采用反向单分支Merkle哈希树实现）。我们将演示如何通过该文件在版本控制和CI中检测前文所述问题。其核心价值在于当多个团队同时添加迁移时，系统能主动提示需要人工检查合并冲突。

:::note
跟随以下命令快速搭建示例环境，这些命令将：

1. 创建Go模块并下载依赖
2. 创建基础User模型
3. 启用版本化迁移功能
4. 执行代码生成
5. 启动MySQL容器（使用`docker stop atlas-sum`停止）

```shell
mkdir ent-sum-file
cd ent-sum-file
go mod init ent-sum-file
go install entgo.io/ent/cmd/ent@master
go run entgo.io/ent/cmd/ent new User
sed -i -E 's|^//go(.*)$|//go\1 --feature sql/versioned-migration|' ent/generate.go
go generate ./...
docker run --rm --name atlas-sum --detach --env MYSQL_ROOT_PASSWORD=pass --env MYSQL_DATABASE=ent -p 3306:3306 mysql
```
:::

首先需通过`schema.WithSumFile()`选项启用`atlas.sum`文件管理功能。以下示例使用[实例化Ent客户端](/docs/versioned-migrations#from-client)生成迁移文件：

```go
package main

import (
	"context"
	"log"
	"os"

	"ent-sum-file/ent"

	"ariga.io/atlas/sql/migrate"
	"entgo.io/ent/dialect/sql/schema"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/ent")
	if err != nil {
		log.Fatalf("failed connecting to mysql: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	// Create a local migration directory.
	dir, err := migrate.NewLocalDir("migrations")
	if err != nil {
		log.Fatalf("failed creating atlas migration directory: %v", err)
	}
	// Write migration diff.
	// highlight-start
	err = client.Schema.NamedDiff(ctx, os.Args[1], schema.WithDir(dir), schema.WithSumFile())
	// highlight-end
	if err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
}
```

执行迁移目录生成命令后，您将看到符合`golang-migrate/migrate`规范的迁移文件，以及包含如下内容的`atlas.sum`文件：

```shell
mkdir migrations
go run -mod=mod main.go initial
```

```sql title="20220504114411_initial.up.sql"
-- create "users" table
CREATE TABLE `users` (`id` bigint NOT NULL AUTO_INCREMENT, PRIMARY KEY (`id`)) CHARSET utf8mb4 COLLATE utf8mb4_bin;

```

```sql title="20220504114411_initial.down.sql"
-- reverse: create "users" table
DROP TABLE `users`;

```

```text title="atlas.sum"
h1:SxbWjP6gufiBpBjOVtFXgXy7q3pq1X11XYUxvT4ErxM=
20220504114411_initial.down.sql h1:OllnelRaqecTrPbd2YpDbBEymCpY/l6ihbyd/tVDgeY=
20220504114411_initial.up.sql h1:o/6yOczGSNYQLlvALEU9lK2/L6/ws65FrHJkEk/tjBk=
```

可见`atlas.sum`为每个迁移文件都生成了对应条目。启用该功能后，当团队A和团队B各自生成迁移文件时，版本控制系统会在第二个团队尝试合并时主动引发冲突。

![atlas-versioned-migrations-no-conflict](https://entgo.io/images/assets/migrate/conflict-2.svg)

:::note
后续步骤中我们通过`go run -mod=mod ariga.io/atlas/cmd/atlas`调用Atlas CLI，您也可以参照[安装指南](https://atlasgo.io/cli/getting-started/setting-up#install-the-cli)全局安装CLI工具（安装后直接使用`atlas`命令调用）。
:::

您可通过以下命令随时校验`atlas.sum`文件与迁移目录的同步状态（当前应无错误输出）：

```shell
go run -mod=mod ariga.io/atlas/cmd/atlas migrate validate
```

然而，如果您手动修改了迁移文件（例如新增SQL语句、编辑现有语句或创建全新文件），`atlas.sum`文件将与迁移目录内容不同步。此时尝试为模式变更生成新迁移文件会被Atlas迁移引擎阻止。您可以通过创建空迁移文件并重新运行`main.go`来验证：

```shell
go run -mod=mod ariga.io/atlas/cmd/atlas migrate new migrations/manual_version.sql --format golang-migrate
go run -mod=mod main.go initial
# 2022/05/04 15:08:09 failed creating schema resources: validating migration directory: checksum mismatch
# exit status 1

```

执行`atlas migrate validate`命令会显示相同错误：

```shell
go run -mod=mod ariga.io/atlas/cmd/atlas migrate validate
# Error: checksum mismatch
# 
# You have a checksum error in your migration directory.
# This happens if you manually create or edit a migration file.
# Please check your migration files and run
# 
# 'atlas migrate hash --force'
# 
# to re-hash the contents and resolve the error.
# 
# exit status 1
```

要使`atlas.sum`文件重新与迁移目录同步，可再次使用Atlas CLI工具：

```shell
go run -mod=mod ariga.io/atlas/cmd/atlas migrate hash --force
```

出于安全考虑，Atlas CLI不会操作未与`atlas.sum`文件同步的迁移目录，因此需要添加`--force`参数强制执行。

针对开发者忘记在手动修改后更新`atlas.sum`文件的情况，建议在CI流程中添加`atlas migrate validate`检查。我们正在开发开箱即用的GitHub Action和CI解决方案，将自动处理此类问题及其他相关事项。

### 总结

本文简要介绍了基于变更SQL文件的模式迁移常见问题，并提出了基于Atlas项目的安全迁移解决方案。

如有疑问或需要入门帮助？欢迎加入我们的[Ent Discord社区](https://discord.gg/qZmPgTE6RX)。

:::note[获取更多Ent资讯：]

- 订阅[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord社区](https://discord.gg/qZmPgTE6RX)
:::