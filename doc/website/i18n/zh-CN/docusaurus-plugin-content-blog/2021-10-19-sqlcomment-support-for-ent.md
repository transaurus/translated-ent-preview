---
title: Introducing sqlcomment - Database Performance Analysis with Ent and Google's Sqlcommenter
author: Amit Shani
authorURL: "https://github.com/hedwigz"
authorImageURL: "https://avatars.githubusercontent.com/u/8277210?v=4"
authorTwitter: itsamitush
image: https://entgo.io/images/assets/sqlcomment/share.png
---

Ent 是一个强大的实体框架，可帮助开发者编写简洁的代码，这些代码会被转换为（可能是复杂的）数据库查询。随着应用程序使用量的增长，很快就会遇到数据库性能问题。排查数据库性能问题 notoriously 困难，尤其是在没有合适工具的情况下。

以下示例展示了 Ent 查询代码如何转换为 SQL 查询。

传统上，将性能不佳的数据库查询与生成它们的应用程序代码关联起来非常困难。数据库性能分析工具可以通过分析数据库服务器日志来指出慢查询，但如何将它们追溯到应用程序呢？

### Sqlcommenter

今年早些时候，[Google 推出了](https://cloud.google.com/blog/topics/developers-practitioners/introducing-sqlcommenter-open-source-orm-auto-instrumentation-library) Sqlcommenter。Sqlcommenter 是

> <em>一个开源库，旨在弥合 ORM 库与理解数据库性能之间的差距。Sqlcommenter 让应用程序开发者能够看到哪些应用程序代码生成了慢查询，并将应用程序跟踪映射到数据库查询计划</em>

换句话说，Sqlcommenter 向 SQL 查询添加了应用程序上下文元数据。这些信息可用于提供有意义的洞察。它通过在查询中添加 [SQL 注释](https://en.wikipedia.org/wiki/SQL_syntax#Comments)来实现这一点，这些注释携带元数据但在查询执行时被数据库忽略。
例如，以下查询包含一个注释，该注释携带了发出查询的应用程序信息（`users-mgr`）、触发查询的控制器和路由（`users` 和 `user_rename`），以及使用的数据库驱动（`ent:v0.9.1`）：

```sql
update users set username = ‘hedwigz’ where id = 88
/*application='users-mgr',controller='users',route='user_rename',db_driver='ent:v0.9.1'*/
```

为了了解从 Sqlcommenter 元数据中收集的分析如何帮助我们更好地理解应用程序的性能问题，请考虑以下示例：Google Cloud 最近推出了 [Cloud SQL Insights](https://cloud.google.com/blog/products/databases/get-ahead-of-database-performance-issues-with-cloud-sql-insights)，这是一个基于云的 SQL 性能分析产品。在下图中，我们看到 Cloud SQL Insights 仪表板的截图，显示 HTTP 路由 'api/users' 导致数据库上的许多锁。我们还可以看到，该查询在过去 6 小时内被调用了 16,067 次。

这就是 SQL 标签的强大之处——它们提供了应用程序级信息与数据库监控之间的关联。

### sqlcomment

[sqlcomm**ent**](https://github.com/ariga/sqlcomment) 是一个 Ent 驱动，它遵循 [sqlcommenter 规范](https://google.github.io/sqlcommenter/spec/) 使用注释向 SQL 查询添加元数据。通过用 `sqlcomment` 包装现有的 Ent 驱动，用户可以利用任何支持该标准的工具来排查查询性能问题。
事不宜迟，让我们看看 `sqlcomment` 的实际应用。

首先，安装 sqlcomment 运行：

```bash
go get ariga.io/sqlcomment
```

`sqlcomment` 包装了底层的 SQL 驱动，因此我们需要使用 ent 的 `sql` 模块而不是 Ent 的常用辅助函数 `ent.Open` 来打开 SQL 连接。

:::info
确保在以下代码片段中导入 `entgo.io/ent/dialect/sql`
:::

```go
// Create db driver.
db, err := sql.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
if err != nil {
	log.Fatalf("Failed to connect to database: %v", err)
}

// Create sqlcomment driver which wraps sqlite driver.
drv := sqlcomment.NewDriver(db,
	sqlcomment.WithDriverVerTag(),
	sqlcomment.WithTags(sqlcomment.Tags{
		sqlcomment.KeyApplication: "my-app",
		sqlcomment.KeyFramework:   "net/http",
	}),
)

// Create and configure ent client.
client := ent.NewClient(ent.Driver(drv))
```

现在，每当我们执行查询时，`sqlcomment` 都会将我们设置的标签附加到 SQL 查询中。如果我们运行以下查询：

```go
client.User.
	Update().
	Where(
		user.Or(
			user.AgeGT(30),
			user.Name("bar"),
		),
		user.HasFollowers(),
	).
	SetName("foo").
	Save()
```

Ent 将输出以下带注释的 SQL 查询：

```sql
UPDATE `users`
SET `name` = ?
WHERE (
    `users`.`age` > ?
    OR `users`.`name` = ?
  )
  AND `users`.`id` IN (
    SELECT `user_following`.`follower_id`
    FROM `user_following`
  )
  /*application='my-app',db_driver='ent:v0.9.1',framework='net%2Fhttp'*/
```

如你所见，Ent 输出了一个带有注释的 SQL 查询，注释包含了与该查询相关的所有信息。

sqlcomm**ent** 支持更多标签，并与 [OpenTelemetry](https://opentelemetry.io) 和 [OpenCensus](https://opencensus.io) 集成。
如需查看更多示例和场景，请访问 [GitHub 仓库](https://github.com/ariga/sqlcomment)。

### 总结

本文展示了如何通过 SQL 注释为查询添加元数据，从而关联源代码与数据库查询。接着介绍了 `sqlcomment`——一个能为所有查询添加 SQL 标签的 Ent 驱动。最后通过安装和配置 `sqlcomment` 与 Ent 的集成，实际演示了其运作方式。如果您对代码感兴趣或想参与贡献，欢迎访问 [GitHub 项目](https://github.com/ariga/sqlcomment)。

有问题需要帮助？欢迎加入我们的 [Discord 服务器](https://discord.gg/qZmPgTE6RX) 或 [Slack 频道](https://entgo.io/docs/slack/)。

:::note[获取更多 Ent 资讯：]

- 订阅我们的 [新闻通讯](https://entgo.substack.com/)
- 关注 [Twitter](https://twitter.com/entgo_io)
- 加入 [Gophers Slack](https://entgo.io/docs/slack) 的 #ent 频道
- 参与 [Ent Discord 服务器](https://discord.gg/qZmPgTE6RX)

:::