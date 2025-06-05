---
title: Announcing the "Schema Import Initiative" and protoc-gen-ent
author: Rotem Tamir
authorURL: "https://github.com/rotemtam"
authorImageURL: "https://s.gravatar.com/avatar/36b3739951a27d2e37251867b7d44b1a?s=80"
authorTwitter: _rtam
---

迁移到新的ORM框架并非易事，其转换成本对许多组织而言可能高不可攀。尽管开发者们总被"闪亮的新事物"吸引，但现实是我们很少有机会参与真正的"绿地"项目。在职业生涯的大部分时间里，我们都在技术债务和业务约束（即遗留系统）构成的语境中工作，这些因素限制了我们向前演进的选择空间。想要成功的新技术必须提供互操作能力和迁移路径，帮助组织无缝过渡到解决现有问题的新方案。

为降低迁移至Ent（或进行技术验证）的成本，我们启动了"**Schema导入计划**"，支持从外部资源生成Ent Schema的多种用例。该计划的核心是`schemast`包（[源代码](https://github.com/ent/contrib/tree/master/schemast)，[文档](https://entgo.io/docs/generating-ent-schemas)），开发者可通过高级API编写生成和操作Ent Schema的程序，无需关注代码解析和AST操作等底层细节。

### Protobuf导入支持

首个基于该API的项目是`protoc-gen-ent`——一个从`.proto`文件生成Ent Schema的protoc插件（[文档](https://github.com/ent/contrib/tree/master/entproto/cmd/protoc-gen-ent)）。已有Protobuf定义的组织可通过此工具自动生成Ent代码。例如给定简单消息定义：

```protobuf
syntax = "proto3";

package entpb;

option go_package = "github.com/yourorg/project/ent/proto/entpb";

message User {
  string name = 1;
  string email_address = 2;
}
```

当设置`ent.schema.gen`选项为true时：

```diff
syntax = "proto3";

package entpb;

+import "options/opts.proto";
 
option go_package = "github.com/yourorg/project/ent/proto/entpb";  

message User {
+  option (ent.schema).gen = true; // <-- tell protoc-gen-ent you want to generate a schema from this message
  string name = 1;
  string email_address = 2;
}
```

开发者可通过标准protoc命令调用该插件：

```shell
protoc -I=proto/ --ent_out=. --ent_opt=schemadir=./schema proto/entpb/user.proto
```

即可生成对应的Ent Schema：

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
)

type User struct {
	ent.Schema
}

func (User) Fields() []ent.Field {
	return []ent.Field{field.String("name"), field.String("email_address")}
}
func (User) Edges() []ent.Edge {
	return nil
}
```

立即使用`protoc-gen-ent`并了解所有配置选项，请参阅[文档](https://github.com/ent/contrib/tree/master/entproto/cmd/protoc-gen-ent)！

### 加入Schema导入计划

您是否有需要自动导入Ent的其他Schema定义？借助`schemast`包，编写所需工具比以往更加简单。不确定如何开始？希望与社区协作规划实现您的想法？欢迎通过[Discord服务器](https://discord.gg/qZmPgTE6RX)、[Slack频道](https://app.slack.com/client/T029RQSE6/C01FMSQDT53)或[GitHub讨论区](https://github.com/ent/ent/discussions)联系我们！

:::note[获取更多Ent资讯：]
- 订阅[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://app.slack.com/client/T029RQSE6/C01FMSQDT53)的#ent频道
- 加入[Ent Discord服务器](https://discord.gg/qZmPgTE6RX)