---
title: "What I learned contributing my first feature to Ent's gRPC plugin"
author: Jeremy Vesperman
authorURL: "https://github.com/jeremyv2014"
authorImageURL: "https://avatars.githubusercontent.com/u/9276415?v=4"
image: https://entgo.io/images/assets/grpc/ent_party.png
---

我从事软件开发多年，但直到最近才了解什么是ORM。在获得计算机工程学士学位的过程中，我学到了许多知识，但对象关系映射并不在其中——那时我过于专注用比特和字节构建底层系统，无暇顾及如此高层级的抽象。因此，当需要协助开发分布式Web应用时，我发现自己完全处于舒适区之外也就不足为奇了。

为他人开发软件的难点在于，你无法洞悉对方的思维。需求往往不够明确，而提问所能获取的理解又有限。有时你不得不构建原型并演示，才能获得有价值的反馈。

这种方法的问题在于，原型开发耗时且需要频繁调整方向。如果像我当初一样不了解ORM，就会在重复性劳动中浪费大量时间：

1. 根据客户反馈重新定义数据模型  
2. 重建测试数据库  
3. 重写数据库交互的SQL语句  
4. 重新定义前后端服务的gRPC接口  
5. 重构前端和网页界面  
6. 向客户演示并获取反馈  
7. 循环往复

耗费数百小时的工作成果可能面临全面重写。所以当资深开发者问我为何不使用Ent这样的ORM时，你可以想象我的解脱（以及尴尬）。

### 发现Ent

仅用一天时间，我们就用Ent重构了现有数据模型。难以置信竟有如此框架存在，而我一直在手工实现这些功能！通过entproto实现的gRPC集成更是锦上添花——只需在模式中添加注解就能实现基础CRUD操作，直接跳过了从数据模型定义到重构网页界面的所有中间步骤。但我的使用场景仍有一个问题：如果无法预知实体ID，如何通过gRPC接口获取实体详情？Ent支持查询全部记录，但entproto的`GetAll`方法在哪里？

### 成为开源贡献者

惊讶地发现这个功能并不存在！本可以在独立服务中实现该特性，但考虑到这是通用需求，我决定直接贡献代码。多年来我一直想寻找有意义的开源项目参与——这简直是完美契机！

经过凌晨时分的源码钻研，我成功实现了这个功能！怀着成就感提交PR后沉沉睡去，浑然不知即将迎来怎样的学习历程。

清晨醒来发现PR已被[Rotem](https://github.com/rotemtam)关闭，但获得了继续完善方案的邀请。关闭原因显而易见：我的`GetAll`实现存在风险——只有小数据量表才适合返回全量数据，大型表暴露此接口可能导致灾难性后果！

### 可选服务方法生成

我的解决方案是通过`entproto.Service()`参数使`GetAll`成为可选方法。但我们意识到应该将其设计为通用功能——为何仅因`GetAll`是最后添加的就给予特殊待遇？理想方案是让所有方法都可选生成，例如：

```go
entproto.Service(entproto.Methods(entproto.Create | entproto.Get))
```

然而，为了保持向后兼容性，空的 `entproto.Service()` 注解仍需生成所有方法。作为Go新手，我唯一能想到的实现方式是使用可变参数函数：

```go
func Service(methods ...Method)
```

这种方法的局限性在于只能有一个可变长度的参数类型。如果后续需要为服务注解添加更多选项怎么办？这时我接触到了强大的[函数式选项模式](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)：

```go
// ServiceOption configures the entproto.Service annotation.
type ServiceOption func(svc *service)

// Service annotates an ent.Schema to specify that protobuf service generation is required for it.
func Service(opts ...ServiceOption) schema.Annotation {
	s := service{
		Generate: true,
	}
	for _, apply := range opts {
		apply(&s)
	}
	// Default to generating all methods
	if s.Methods == 0 {
		s.Methods = MethodAll
	}
	return s
}
```

该模式通过接收可变数量的函数来设置结构体选项（本例中即服务注解）。基于此模式，我们除了`Methods`外还能实现任意数量的其他选项函数。非常巧妙！

### List：更优的GetAll方案

解决可选方法生成问题后，我们重新聚焦于安全实现`GetAll`。Rotem建议基于Google的API改进提案[AIP-132](https://google.aip.dev/132)实现List方法。该方案允许客户端分页获取所有实体，不仅比"GetAll"命名更优雅，还解决了安全性问题。

### List请求

该设计下的请求消息结构如下：

```protobuf
message ListUserRequest {
  int32 page_size = 1;

  string page_token = 2;

  View view = 3;

  enum View {
    VIEW_UNSPECIFIED = 0;

    BASIC = 1;

    WITH_EDGE_IDS = 2;
  }
}
```

#### 分页大小

`page_size`字段允许客户端指定单次响应返回的最大条目数（上限1000条），既避免了原始`GetAll`可能返回过量数据的问题，又通过限制最大页数防止服务器过载。

#### 分页令牌

`page_token`是Base64编码字符串，服务器据此确定下一页起始位置。空令牌表示获取第一页。

#### 视图模式

`view`字段用于指定响应是否返回实体关联的边缘ID。

### List响应

响应消息结构如下：

```protobuf
message ListUserResponse {
  repeated User user_list = 1;

  string next_page_token = 2;
}
```

#### 实体列表

`user_list`字段包含当前页的实体数据。

#### 下一页令牌

`next_page_token`是Base64编码字符串，可用于后续List请求获取下一页。空令牌表示当前页为最后一页。

### 分页实现

确定gRPC接口后，面临分页实现的技术选型。最基础的`LIMIT/OFFSET`分页需要[跳过已获取行](https://use-the-index-luke.com/no-offset)，存在严重性能缺陷——数据库必须读取所有跳过的行才能定位目标数据。

#### 键集分页

Rotem提出了更优的键集分页方案：通过唯一列（或列组合）排序，只需比较客户端提供的分页令牌与有序数据的键值关系（大于/小于等于），即可精准定位分页起始点。这种方案虽然实现稍复杂，但能避免无效数据读取，显著提升大表查询性能！

选定键集分页方案后，下一步需要确定实体排序方式。对Ent而言最直接的方法是使用`id`字段——所有模式都包含该字段且保证唯一性。我们选择将此作为初始实现方案。此外还需决定采用升序还是降序排列，初始版本选择了降序排序。

### 使用方式

下面我们来看看如何实际使用新的`List`功能：

```go
package main

import (
  "context"
  "log"

  "ent-grpc-example/ent/proto/entpb"
  "google.golang.org/grpc"
  "google.golang.org/grpc/status"
)

func main() {
  // Open a connection to the server.
  conn, err := grpc.Dial(":5000", grpc.WithInsecure())
  if err != nil {
    log.Fatalf("failed connecting to server: %s", err)
  }
  defer conn.Close()
  // Create a User service Client on the connection.
  client := entpb.NewUserServiceClient(conn)
  ctx := context.Background()
  // Initialize token for first page.
  pageToken := ""
  // Retrieve all pages of users.
  for {
    // Ask the server for the next page of users, limiting entries to 100.
    users, err := client.List(ctx, &entpb.ListUserRequest{
        PageSize:  100,
        PageToken: pageToken,
    })
    if err != nil {
        se, _ := status.FromError(err)
        log.Fatalf("failed retrieving user list: status=%s message=%s", se.Code(), se.Message())
    }
    // Check if we've reached the last page of users.
    if users.NextPageToken == "" {
        break
    }
    // Update token for next request.
    pageToken = users.NextPageToken
    log.Printf("users retrieved: %v", users)
  }
}
```

### 未来展望

当前`List`实现存在若干待改进的限制：首先排序仅支持`id`列，这虽然兼容所有模式但缺乏灵活性。理想情况下客户端应能指定排序列，或通过模式定义排序列。其次当前仅支持降序排列，未来可将其设为请求选项。最后目前仅支持`int32`、`uuid`和`string`类型的`id`字段，因为代码生成模板中需要为每种类型单独实现页码令牌的转换方法（毕竟开发资源有限）。

### 总结

最初决定为entproto贡献此功能时我十分忐忑，作为开源新手完全不知从何入手。但很高兴地告诉大家，参与Ent项目的过程充满乐趣！我有幸与优秀的开发者共事，同时为开源社区贡献力量。从函数式选项、键集分页到代码审查中的细节洞察，整个过程中我收获了丰富的Go语言和软件开发经验。强烈建议有意贡献的朋友勇敢尝试——你将从这段经历中获得超乎想象的成长！

若有疑问或需要入门帮助？欢迎加入我们的[Discord服务器](https://discord.gg/qZmPgTE6RX)或[Slack频道](https://entgo.io/docs/slack/)。

:::note[获取更多Ent资讯：]

- 订阅[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord社区](https://discord.gg/qZmPgTE6RX)

:::