---
title: How to implement the Twitter edit button with Ent
author: Amit Shani
authorURL: "https://github.com/hedwigz"
authorImageURL: "https://avatars.githubusercontent.com/u/8277210?v=4"
authorTwitter: itsamitush
image: "https://entgo.io/images/assets/enthistory/share.png"
---

随着埃隆·马斯克发起投票询问用户是否需要"编辑按钮"功能，Twitter的这一特性迅速成为头条新闻。

[![埃隆的推文](https://entgo.io/images/assets/enthistory/enthistory2.webp)](https://twitter.com/elonmusk/status/1511143607385874434)

毫无疑问，这是Twitter用户最期待的功能之一。

作为一名软件开发人员，我立即开始思考如何自行实现这个功能。变更追踪/审计问题在各类应用中非常普遍。假设你有一个实体（例如`Tweet`），需要追踪其某个字段（例如`content`字段）的变更，存在多种通用解决方案。某些数据库甚至提供专有方案，如微软的变更追踪和MariaDB的系统版本表。但在大多数场景下，开发者需要自行"缝合"这些功能。幸运的是，Ent提供的模块化扩展系统让你仅需几行代码就能接入此类功能。

![Twitter+编辑按钮](https://entgo.io/images/assets/enthistory/enthistory3.gif)

### Ent框架简介

Ent是一个让大型Go应用开发变得轻松的实体框架，开箱即用提供以下强大特性：

* 类型安全的[CRUD API](https://entgo.io/docs/crud)生成
* 复杂的[图遍历](https://entgo.io/docs/traversals)（轻松实现SQL连接）
* [分页](https://entgo.io/docs/paging)
* [隐私](https://entgo.io/docs/privacy)控制
* 安全的数据库[迁移](https://entgo.io/blog/2022/03/14/announcing-versioned-migrations)

借助Ent的代码生成引擎和高级[扩展系统](https://entgo.io/blog/2021/09/02/ent-extension-api/)，你可以轻松模块化Ent客户端，集成那些通常需要手动耗时实现的高级功能，例如：

* 生成[REST](https://entgo.io/blog/2022/02/15/generate-rest-crud-with-ent-and-ogen)、[gRPC](https://entgo.io/docs/grpc-intro)和[GraphQL](https://entgo.io/docs/graphql)服务端
* [缓存](http://entgo.io/blog/2021/10/14/introducing-entcache)
* 通过[sqlcommenter](https://entgo.io/blog/2021/10/19/sqlcomment-support-for-ent)实现监控

### Enthistory扩展

当我们想为某个Web服务添加"活动与历史"面板时，开发了`enthistory`扩展。该面板用于展示"谁在何时修改了什么"（即审计功能）。在[Atlas](https://atlasgo.io/)——这个使用声明式HCL文件管理数据库的工具中，我们有个名为"schema"的实体，本质上是大型文本块。所有模式变更都会被记录，并可在"活动与历史"面板中查看。

![活动与历史](https://entgo.io/images/assets/enthistory/enthistory1.gif)

这类功能非常普遍，可见于Google文档、GitHub PR和Facebook帖子等应用中，但在广受欢迎的Twitter上却遗憾缺失。

已有超过300万人投票支持Twitter添加"编辑按钮"，下面让我展示Twitter如何能毫不费力地满足用户期待！

使用Enthistory时，你只需像这样在Ent模式中添加注解：

```go
func (Tweet) Fields() []ent.Field {
	return []ent.Field{
		field.String("content").
			Annotations(enthistory.TrackField()),
		field.Time("created").
			Default(time.Now),
	}
}
```

Enthistory会挂钩到Ent客户端，确保所有对"Tweet"的CRUD操作都被记录到"tweets_history"表中，无需修改代码，并提供API来消费这些记录：

```go
// Creating a new Tweet doesn't change. enthistory automatically modifies
// your transaction on the fly to record this event in the history table
client.Tweet.Create().SetContent("hello world!").SaveX(ctx)

// Querying history changes is as easy as querying any other entity's edge.
t, _ := client.Tweet.Get(ctx, id)
hs := client.Tweet.QueryHistory(t).WithChanges().AllX(ctx)
```

如果不使用Enthistory，你需要这样做：以类似Twitter的应用为例，它有一个名为"tweets"的表，其中一列存储推文内容。

| id      | content | created_at | author_id |
| ----------- | ----------- | ----------- | ----------- |
| 1      | Hello Twitter!       | 2022-04-06T13:45:34+00:00       | 123       |
| 2      | Hello Gophers!       | 2022-04-06T14:03:54+00:00       | 456       |

现在假设我们想允许用户编辑内容，同时在前端展示修改记录。解决这个问题有几种常见方案，各有利弊（我们将在另一篇技术文章中深入探讨）。目前可行的方案是创建"tweets_history"表来记录推文变更：

| id      | tweet_id | timestamp | event | content |
| ----------- | ----------- | ----------- | ----------- | ----------- |
| 1      | 1       | 2022-04-06T12:30:00+00:00       | CREATED       | hello world!       |
| 2      | 2       | 2022-04-06T13:45:34+00:00       | UPDATED       | hello Twitter!       |

通过类似上表的结构，我们可以记录原始推文"1"的变更。当需要时，可以显示该推文最初发布于12:30:00，内容为"hello world!"，后在13:45:34修改为"hello Twitter!"。

要实现这一点，我们需要修改所有针对"tweets"的`UPDATE`语句，使其同时向"tweets_history"执行`INSERT`。为确保数据一致性，必须将这两个语句包裹在事务中，防止第一条成功而第二条失败导致历史记录损坏。同时还需确保每次对"tweets"的`INSERT`操作都伴随对"tweets_history"的插入。

```diff
# INSERT is logged as "CREATE" history event
- INSERT INTO tweets (`content`) VALUES ('Hello World!');
+BEGIN;
+INSERT INTO tweets (`content`) VALUES ('Hello World!');
+INSERT INTO tweets_history (`content`, `timestamp`, `record_id`, `event`)
+VALUES ('Hello World!', NOW(), 1, 'CREATE');
+COMMIT;

# UPDATE is logged as "UPDATE" history event
- UPDATE tweets SET `content` = 'Hello World!' WHERE id = 1;
+BEGIN;
+UPDATE tweets SET `content` = 'Hello World!' WHERE id = 1;
+INSERT INTO tweets_history (`content`, `timestamp`, `record_id`, `event`)
+VALUES ('Hello World!', NOW(), 1, 'UPDATE');
+COMMIT;
```

这种方法虽然可行，但你需要为不同实体（如"comment_history"、"settings_history"）创建单独的历史表。为避免这种情况，Enthistory创建了统一的"history"和"changes"表，将所有追踪字段集中记录，且无需添加额外列就能支持多种字段类型。

### 预发布阶段

Enthistory仍处于早期设计阶段，正在进行内部测试，因此尚未开源发布，但我们计划很快开放。如果你想体验预发布版本，我编写了一个结合GraphQL+Enthistory的简单React应用来演示推文编辑功能，代码库见[这里](https://github.com/hedwigz/edit-twitter-example-app)，欢迎反馈意见。

### 总结

我们看到了Ent的模块化扩展系统如何让你像安装软件包一样轻松实现高级功能。[开发自己的扩展](https://entgo.io/blog/2021/12/09/contributing-my-first-feature-to-ent-grpc-plugin)既有趣又富有教育意义！未来Enthistory将支持追踪边（外键关联表）的变更，与OpenAPI和GraphQL扩展集成，并提供更多底层实现方案。

:::note[获取更多Ent资讯：]

- 订阅[新闻简报](https://entgo.substack.com/)
- 关注[Twitter账号](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord社区](https://discord.gg/qZmPgTE6RX)

:::