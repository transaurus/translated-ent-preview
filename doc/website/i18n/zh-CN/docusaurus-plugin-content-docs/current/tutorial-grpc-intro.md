---
id: grpc-intro
title: gRPC Introduction
sidebar_label: Introduction
---

[gRPC](https://grpc.io) 是由谷歌开源的一款流行RPC框架，基于其内部开发的"Stubby"系统构建。该框架采用[Protocol Buffers](https://developers.google.com/protocol-buffers)作为基础，这是谷歌推出的语言中立、平台中立且可扩展的结构化数据序列化机制。

Ent通过[ent/contrib](https://github.com/ent/contrib)提供的插件支持从数据库模式自动生成gRPC服务。

整体而言，Ent与gRPC的集成工作流程如下：

* 使用名为`entproto`的命令行工具（或代码生成钩子）从Ent模式生成Protocol Buffer定义和gRPC服务定义。模式中需添加`entproto`注解来辅助跨领域映射。
* 通过protobuf编译器插件`protoc-gen-entgrpc`生成gRPC服务实现，该实现使用项目的`ent.Client`进行数据库读写操作。
* 开发者编写嵌入生成服务实现的gRPC服务器。

本教程将使用Ent/gRPC集成方案构建一个功能完整的gRPC服务器。

### 代码示例

完整示例代码可查看[rotemtam/ent-grpc-example](https://github.com/rotemtam/ent-grpc-example)仓库。