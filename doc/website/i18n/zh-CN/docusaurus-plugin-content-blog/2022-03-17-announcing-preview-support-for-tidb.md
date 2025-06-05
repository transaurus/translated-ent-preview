---
title: Announcing preview support for TiDB
author: Amit Shani
authorURL: "https://github.com/hedwigz"
authorImageURL: "https://avatars.githubusercontent.com/u/8277210?v=4"
authorTwitter: itsamitush
---

我们[此前已宣布](2022-01-20-announcing-new-migration-engine.md)Ent的新迁移引擎Atlas。通过Atlas，为Ent添加对新数据库的支持变得前所未有的简单。今天，我很高兴地宣布，使用启用Atlas的最新版Ent，现已提供对[TiDB](https://en.pingcap.com/tidb/)的预览版支持。

Ent可用于访问多种类型数据库中的数据，包括图数据库和关系型数据库。最常见的是，用户一直在使用标准的开源关系型数据库，如MySQL、MariaDB和PostgreSQL。随着基于Ent构建的应用程序团队日益成功，需要处理更大规模的流量时，这些单节点数据库往往成为扩展的瓶颈。因此，Ent社区的许多成员要求支持[NewSQL](https://en.wikipedia.org/wiki/NewSQL)数据库，如TiDB。

### TiDB

[TiDB](https://en.pingcap.com/tidb/)是一个[开源](https://github.com/pingcap/tidb)的NewSQL数据库。它提供了许多传统数据库所不具备的特性，例如：

1. **水平扩展** - 多年来，软件架构师需要在关系型数据库提供的熟悉度和保证与_NoSQL_数据库（如MongoDB或Cassandra）的扩展能力之间做出选择。TiDB在保持与MySQL功能良好兼容性的同时支持水平扩展。
2. **HTAP（混合事务/分析处理）** - 此外，数据库传统上分为分析型（OLAP）和事务型（OLTP）数据库。TiDB打破了这种二分法，允许在同一数据库上同时进行分析和事务处理工作负载。
3. **预打包监控** w/ Prometheus+Grafana - TiDB从底层构建时就基于云原生范式，并原生支持标准的CNCF可观测性堆栈。

要了解更多信息，请查看官方的[TiDB简介](https://docs.pingcap.com/tidb/stable)。

### TiDB的Hello World

要快速体验Ent+TiDB的"Hello World"应用程序，请按照以下步骤操作：

1. 使用Docker启动一个本地TiDB服务器：

```shell
 docker run -p 4000:4000 pingcap/tidb
 ```

现在你应该有一个运行中的TiDB实例在端口4000上监听。

2. 克隆示例[`hello world`仓库](https://github.com/hedwigz/tidb-hello-world)：

```shell
 git clone https://github.com/hedwigz/tidb-hello-world.git
 ```

在这个示例仓库中，我们定义了一个简单的`User`模式：

```go title="ent/schema/user.go"
 func (User) Fields() []ent.Field {
 	return []ent.Field{
 		field.Time("created_at").
 			Default(time.Now),
 		field.String("name"),
 		field.Int("age"),
 	}
 }
 ```

然后，我们将Ent与TiDB连接：

```go title="main.go"
 client, err := ent.Open("mysql", "root@tcp(localhost:4000)/test?parseTime=true")
 if err != nil {
 	log.Fatalf("failed opening connection to tidb: %v", err)
 }
 defer client.Close()
 // Run the auto migration tool, with Atlas.
 if err := client.Schema.Create(context.Background(), schema.WithAtlas(true)); err != nil {
 	log.Fatalf("failed printing schema changes: %v", err)
 }
	```
 Note that in line `1` we connect to the TiDB server using a `mysql` dialect. This is possible due to the fact that TiDB is [MySQL compatible](https://docs.pingcap.com/tidb/stable/mysql-compatibility), and it does not require any special driver.  
 Having said that, there are some differences between TiDB and MySQL, especially when pertaining to schema migrations, such as information schema inspection and migration planning. For this reason, `Atlas` automatically detects if it is connected to `TiDB` and handles the migration accordingly.  
 In addition, note that in line `7` we used `schema.WithAtlas(true)`, which flags Ent to use `Atlas` as its 
 migration engine.  
   
 Finally, we create a user and save the record to TiDB to later be queried and printed.
 ```go title="main.go"
 client.User.Create().
		SetAge(30).
		SetName("hedwigz").
		SaveX(context.Background())
 user := client.User.Query().FirstX(context.Background())
 fmt.Printf("the user: %s is %d years old\n", user.Name, user.Age)
 ```

3. 运行示例程序：

```go
 $ go run main.go
 the user: hedwigz is 30 years old
 ```

哇！在这个快速演练中，我们成功实现了：

* 启动一个本地TiDB实例。
* 将Ent与TiDB连接。
* 使用Atlas迁移我们的Ent模式。
* 使用Ent向TiDB插入并查询数据。

### 预览版支持

Atlas与TiDB的集成已在TiDB版本`v5.4.0`（撰写本文时为`latest`）上进行了充分测试，未来我们将进一步扩展支持范围。如果你使用的是其他版本的TiDB或需要帮助，请随时[提交问题](https://github.com/ariga/atlas/issues)或加入我们的[Discord频道](https://discord.gg/zZ6sWVg6NT)。

:::note[获取更多Ent新闻和更新：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 在[Twitter](https://twitter.com/entgo_io)上关注我们
- 加入[Gophers Slack](https://entgo.io/docs/slack)上的#ent频道
- 加入[Ent Discord服务器](https://discord.gg/qZmPgTE6RX)

:::