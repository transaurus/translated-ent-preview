---
title: Ent + gRPC is Ready for Usage
author: Rotem Tamir
authorURL: "https://github.com/rotemtam"
authorImageURL: "https://s.gravatar.com/avatar/36b3739951a27d2e37251867b7d44b1a?s=80"
authorTwitter: _rtam
---

数月前，我们宣布了实验性支持[从Ent模式定义生成gRPC服务](https://entgo.io/blog/2021/03/18/generating-a-grpc-server-with-ent)。当时实现尚未完善，但我们希望尽快发布供社区试用并收集反馈。

如今，在广泛采纳社区意见后，我们很高兴宣布[Ent](https://entgo.io)与[gRPC](https://grpc.io)的集成已"正式可用"，这意味着所有基础功能均已完备，大多数Ent应用均可使用该集成方案。

自首次发布以来我们新增了哪些功能？

- [可选字段支持](https://entgo.io/docs/grpc-optional-fields) - Protobuf的常见痛点在于零值表达机制：原始类型的零值不会被编码进二进制数据，导致应用无法区分零值与未设值。为此Protobuf项目提供了一套"[知名类型](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf)"作为"包装类型"，用结构体封装原始值。现在当`entproto`生成Protobuf消息定义时，会自动使用这些包装类型来表示Ent中的"可选"字段：
  ```protobuf {15}
  // 由entproto生成的代码，请勿编辑
  syntax = "proto3";
  
  package entpb;
  
  import "google/protobuf/wrappers.proto";
  
  message User {
    int32 id = 1;
  
    string name = 2;
  
    string email_address = 3;
  
    google.protobuf.StringValue alias = 4;
  }
  ```

- [多边关系支持](https://entgo.io/docs/grpc-edges) - 初始版本的`protoc-gen-entgrpc`仅支持生成"唯一"边(即最多引用单个实体)的gRPC服务实现。[最新版本](https://github.com/ent/contrib/commit/bf9430fbba45a808bc054144f9711833c76bf05c)已支持为O2M和M2M关系生成读写操作的gRPC方法。
- [部分响应](https://entgo.io/docs/grpc-edges#retrieving-edge-ids-for-entities) - 默认情况下服务的`Get`方法不会返回边信息，这是因为实体关联的边数量可能无限。为允许调用方控制是否返回边信息，生成的服务遵循[Google AIP-157](https://google.aip.dev/157)(部分响应规范)。简而言之，`Get<T>Request`消息包含名为View的枚举，调用方可通过该枚举控制是否从数据库检索边信息。
  
  ```protobuf {6-12}
  message GetUserRequest {
    int32 id = 1;
  
    View view = 2;
  
    enum View {
      VIEW_UNSPECIFIED = 0;
  
      BASIC = 1;
  
      WITH_EDGE_IDS = 2;
    }
  }
  ```

### 快速开始

- 我们发布了官方[Ent+gRPC教程](https://entgo.io/docs/grpc-intro)（及配套[GitHub仓库](https://github.com/rotemtam/ent-grpc-example)）帮助开发者入门
- 需要集成帮助或有其他问题？[加入Slack频道](https://entgo.io/docs/slack)或[Discord服务器](https://discord.gg/qZmPgTE6RX)

:::note[获取更多Ent资讯：]

- 订阅[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 加入[Ent Discord服务器](https://discord.gg/qZmPgTE6RX)

:::