---
title: Announcing entcache - a Cache Driver for Ent
author: Ariel Mashraki
authorURL: "https://github.com/a8m"
authorImageURL: "https://avatars0.githubusercontent.com/u/7413593"
authorTwitter: arielmashraki
---

在开发[Ariga](https://ariga.io)的运营数据图谱查询引擎时，我们发现通过构建健壮的缓存库可以显著提升众多场景下的性能表现。作为Ent的重度使用者，我们很自然地将这一层实现为Ent的扩展。本文将简要介绍缓存的概念、其在软件架构中的作用，并展示`entcache`——一个专为Ent设计的缓存驱动。

缓存是提升应用性能的常用策略，其原理基于不同介质的数据读取速度可能存在数量级差异。[Jeff Dean](https://twitter.com/jeffdean?lang=en)在关于"[构建大规模分布式系统的软件工程建议](http://static.googleusercontent.com/media/research.google.com/en/us/people/jeff/stanford-295-talk.pdf)"的著名演讲中，曾给出以下经典数据：

![缓存性能对比](https://entgo.io/images/assets/entcache/cache-numbers.png)

这些数据印证了资深工程师的直觉认知：内存读取快于磁盘访问，同数据中心的数据获取快于互联网请求。我们还需补充的是，某些计算过程本身代价高昂，获取预计算结果往往比实时重复计算更为高效（且成本更低）。

[维基百科](https://en.wikipedia.org/wiki/Cache_(computing))将缓存定义为"存储数据以便未来请求能更快响应的硬件或软件组件"。换言之，若能将查询结果暂存于内存，其响应速度将远胜于：通过网络访问数据库→磁盘读取→执行计算→结果回传（仍需经过网络）的完整链路。

但作为工程师必须谨记：缓存设计是公认的复杂领域。正如早期网景工程师[Phil Karlton](https://martinfowler.com/bliki/TwoHardThings.html)的名言："计算机科学只有两件难事：缓存失效和命名"。例如在强一致性系统中，陈旧的缓存条目可能导致系统行为异常。因此，在架构设计中引入缓存时，必须审慎考虑每个细节。

### 介绍`entcache`

`entcache`包提供了一个可包装现有Ent SQL驱动的新驱动。其高层逻辑是对基础驱动的Query方法进行装饰，每次调用时：

1. 根据参数（语句和参数）生成缓存键（哈希值）

2. 检查缓存中是否已有该查询结果（缓存命中）。若存在则直接返回内存结果，跳过数据库查询

3. 若缓存未命中，则将查询传递给底层数据库执行

4. 查询执行后，驱动会记录返回行(`sql.Rows`)的原始值，并使用生成的缓存键将其存入缓存

该包提供多种配置选项：缓存条目TTL、哈希函数控制、自定义多级缓存存储、缓存淘汰与跳过机制等。完整文档参见[https://pkg.go.dev/ariga.io/entcache](https://pkg.go.dev/ariga.io/entcache)。

如前所述，正确配置应用缓存是精细工作，因此`entcache`为开发者提供了不同层级的缓存方案：

1. 基于 `context.Context` 的缓存。通常附加到单个请求中，不与其他缓存层级联动。用于消除同一请求内重复执行的查询。

2. 由 `ent.Client` 使用的驱动级缓存。由于应用通常为每个数据库创建独立驱动，因此我们将其视为进程级缓存。

3. 远程缓存。例如 Redis 数据库，为多个进程提供持久化缓存存储和共享能力。远程缓存层能抵御应用部署变更或故障，并减少不同进程对数据库执行的相同查询。

4. 缓存层次结构（多级缓存）允许以层级方式组织缓存。缓存存储的层级主要基于访问速度和缓存容量。例如：由应用内存中的 LRU 缓存和 Redis 支撑的远程缓存组成的两级缓存。

我们通过解释基于 `context.Context` 的缓存来演示这一机制。

### 上下文级缓存

`ContextLevel` 选项将驱动配置为使用上下文级缓存。上下文通常附加到请求（如 `*http.Request`）中，且不参与多级缓存模式。当采用此选项作为缓存存储时，附带的 `context.Context` 会携带一个 LRU 缓存（可配置其他策略），驱动在执行查询时会在该 LRU 缓存中存储和检索条目。

此选项非常适合需要强一致性、但仍希望避免同一请求内重复执行数据库查询的应用。例如以下 GraphQL 查询：

```graphql
query($ids: [ID!]!) {
    nodes(ids: $ids) {
        ... on User {
            id
            name
            todos {
                id
                owner {
                    id
                    name
                }
            }
        }
    }
}
```

原始方案解析该查询需要执行：1 次获取 N 个用户，N 次获取每个用户的待办事项，以及每个待办事项各 1 次获取其所有者（详细了解 [_N+1 问题_](https://entgo.io/docs/tutorial-todo-gql-field-collection/#problem)）。

但 Ent 提供了独特的解析方案（详见 [Ent 官网](https://entgo.io/docs/tutorial-todo-gql-field-collection)），最终只需执行 3 次查询：1 次获取 N 个用户，1 次获取**所有**用户的待办事项，1 次获取**所有**待办事项的所有者。

使用 `entcache` 后，查询次数可降至 2 次，因为首尾两次查询完全一致（参见 [代码示例](https://github.com/ariga/entcache/blob/master/internal/examples/ctxlevel/main_test.go)）。

![上下文级缓存示意图](https://entgo.io/images/assets/entcache/ctxlevel.png)

各缓存层级的详细说明请参阅仓库 [README](https://github.com/ariga/entcache/blob/master/README.md)。

### 快速开始

> 若您不熟悉如何新建 Ent 项目，请先完成 Ent [入门教程](https://entgo.io/docs/tutorial-setup)。

首先通过以下命令获取该包：

```shell
go get ariga.io/entcache
```

安装 `entcache` 后，您可以通过以下代码片段快速集成到项目中：

```go
// Open the database connection.
db, err := sql.Open(dialect.SQLite, "file:ent?mode=memory&cache=shared&_fk=1")
if err != nil {
	log.Fatal("opening database", err)
}
// Decorates the sql.Driver with entcache.Driver.
drv := entcache.NewDriver(db)
// Create an ent.Client.
client := ent.NewClient(ent.Driver(drv))

// Tell the entcache.Driver to skip the caching layer
// when running the schema migration.
if client.Schema.Create(entcache.Skip(ctx)); err != nil {
	log.Fatal("running schema migration", err)
}

// Run queries.
if u, err := client.User.Get(ctx, id); err != nil {
	log.Fatal("querying user", err)
}
// The query below is cached.
if u, err := client.User.Get(ctx, id); err != nil {
	log.Fatal("querying user", err)
}
```

更多高级示例请访问仓库的 [examples 目录](https://github.com/ariga/entcache/tree/master/internal/examples)。

### 总结

本文介绍了我在开发 [Ariga 运营数据图谱](https://ariga.io) 查询引擎时创建的 Ent 缓存驱动 "entcache"。我们首先讨论了在软件系统中引入缓存的动机，接着阐述了 `entcache` 的特性和能力，最后通过简单示例演示了如何将其集成到您的应用中。

我们正在开发一些功能，并希望继续推进，但需要社区帮助来完善设计（比如缓存失效问题，有人感兴趣吗？😉）。如果您有意参与贡献，请在Ent的Slack频道联系我。

:::note[获取更多Ent资讯：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 在[Twitter](https://twitter.com/entgo_io)关注我们
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord服务器](https://discord.gg/qZmPgTE6RX)

:::