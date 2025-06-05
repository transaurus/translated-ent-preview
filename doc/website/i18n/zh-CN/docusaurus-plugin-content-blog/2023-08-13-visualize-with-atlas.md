---
title: "Quickly Generate ERDs from your Ent Schemas (Updated)" 
author: Rotem Tamir
authorURL: "https://github.com/rotemtam"
authorImageURL: "https://s.gravatar.com/avatar/36b3739951a27d2e37251867b7d44b1a?s=80"
authorTwitter: _rtam
image: "https://atlasgo.io/uploads/ent/inspect/entviz.png"
---

### 内容提要

只需一条命令即可生成Ent架构的可视化图表：

```
atlas schema inspect \
  -u ent://ent/schema \
  --dev-url "sqlite://demo?mode=memory&_fk=1" \
  --visualize
```

![](https://entgo.io/images/assets/erd/edges-quick-summary.png)

大家好！

数月前我们发布了[entviz](/blog/2023/01/26/visualizing-with-entviz)——这个酷炫工具能帮助您可视化Ent数据模型。由于其出色的反响，我们决定将其直接集成至Ent使用的迁移引擎[Atlas](https://atlasgo.io)中。

自Atlas [v0.13.0](https://atlasgo.io/blog/2023/08/06/atlas-v-0-13)版本起，您无需安装额外工具即可直接通过Atlas可视化Ent数据模型。

### 私有与公开可视化

此前您只能将架构可视化图表分享至[Atlas公共沙盒](https://gh.atlasgo.cloud/explore)。虽然便于协作，但许多团队的敏感架构数据并不适合公开分享。

新版本支持将架构直接发布至[Atlas云平台](https://atlasgo.cloud)的私有工作区，确保仅您和团队成员可访问可视化图表。

### 使用Atlas可视化Ent架构

首先安装最新版Atlas：

```
curl -sSfL https://atlasgo.io/install.sh | sh
```

其他安装方式请参阅[Atlas安装文档](https://atlasgo.io/getting-started#installation)。

运行以下命令生成Ent架构可视化：

```
atlas schema inspect \
  -u ent://ent/schema \
  --dev-url "sqlite://demo?mode=memory&_fk=1" \
  --visualize
```

命令解析：

* `atlas schema inspect` - 该命令支持从多数据源检查架构并以多种格式输出，此处用于检查Ent架构
* `-u ent://ent/schema` - 指定本地Ent架构路径（`./ent/schema`目录）
* `--dev-url "sqlite://demo?mode=memory&_fk=1"` - Atlas需要借助[开发数据库](https://atlasgo.io/concepts/dev-database)进行架构标准化计算。本例使用内存SQLite数据库，若使用其他驱动可替换为`docker://mysql/8/dev`（MySQL）或`docker://postgres/15/?search_path=public`（PostgreSQL）

执行后将看到如下输出：

```text
Use the arrow keys to navigate: ↓ ↑ → ←
? Where would you like to share your schema visualization?:
  ▸ Publicly (gh.atlasgo.cloud)
    Your personal workspace (requires 'atlas login')
```

选择第一项可公开分享架构，选择第二项并通过`atlas login`登录（免费）Atlas账户可私有分享。

### 结语

本文演示了如何通过Atlas便捷可视化Ent架构。期待该功能为您带来便利，欢迎反馈使用体验！

:::note[获取更多Ent资讯：]

- 订阅[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord社区](https://discord.gg/qZmPgTE6RX)

:::