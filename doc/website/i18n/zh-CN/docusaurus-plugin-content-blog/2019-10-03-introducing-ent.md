---
title: Introducing ent
author: Ariel Mashraki
authorURL: "https://github.com/a8m"
authorImageURL: "https://avatars0.githubusercontent.com/u/7413593"
authorTwitter: arielmashraki
---

## Facebook Connectivity 特拉维夫团队的 Go 语言现状

20个月前，在积累了约5年Go语言开发经验并推动其在多家公司落地后，我加入了Facebook Connectivity（FBC）特拉维夫团队。  
当时团队正在启动新项目，需要选择编程语言。经过多语言对比评估，我们最终选择了Go。

此后，Go语言在FBC其他项目中持续扩散，仅特拉维夫办公室就已有约15名Go工程师，取得了显著成功。**新服务现在均采用Go编写**。

## 开发新型Go ORM的动机

在加入Facebook前的5年里，我的工作主要聚焦基础设施工具和微服务领域，较少涉及复杂数据模型开发。对于需要简单SQL数据库操作的服务，我们采用现有开源解决方案；而涉及复杂数据模型的服务则使用具备成熟ORM的其他语言实现，例如基于SQLAlchemy的Python。

在Facebook，我们习惯以图论思维构建数据模型，这种模式在内部实践中表现优异。  
由于Go生态缺乏完善的图式ORM，我们基于以下原则自主开发：

- **代码即Schema** - 类型定义、关系映射和约束条件应通过Go代码（而非结构体标签）实现，并通过CLI工具进行验证。Facebook内部类似工具的良好实践为此提供了参考。
- **静态类型化显式API** - 通过代码生成避免`interface{}`泛滥，这对提升开发效率（尤其对新成员）至关重要。
- **简化查询/聚合/图遍历** - 开发者不应被原始SQL语句或专业术语困扰。
- **谓词条件静态类型化** - 彻底杜绝字符串魔法。
- **完整支持context.Context** - 这对于实现全链路追踪、日志系统集成以及取消机制等特性不可或缺。
- **存储无关架构** - 通过代码生成模板保持存储层灵活性，项目初期基于Gremlin（AWS Neptune）开发，后期顺利切换至MySQL。

## 开源ent框架

**ent**是基于上述原则构建的Go实体框架（ORM）。通过**ent**，开发者可以便捷地用Go代码定义任意数据模型或图结构。Schema配置经**entc**（ent代码生成器）验证后，将生成符合Go语言习惯的静态类型化API，持续保障开发效率与工程愉悦度。当前支持MySQL、MariaDB、PostgreSQL、SQLite及基于Gremlin的图数据库。

今天我们正式开源**ent**，欢迎开始体验 → [entgo.io/docs/getting-started](/docs/getting-started)