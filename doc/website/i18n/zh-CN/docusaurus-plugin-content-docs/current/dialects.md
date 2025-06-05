---
id: dialects
title: Supported Dialects
---

## MySQL

MySQL 支持 [迁移](migrate.md) 章节中提到的所有功能特性，并持续在以下三个版本进行测试：`5.6.35`、`5.7.26` 和 `8`。

## MariaDB

MariaDB 支持 [迁移](migrate.md) 章节中提到的所有功能特性，并持续在以下三个版本进行测试：`10.2`、`10.3` 和最新版本。

## PostgreSQL

PostgreSQL 支持 [迁移](migrate.md) 章节中提到的所有功能特性，并持续在以下五个版本进行测试：`11`、`12`、`13`、`14` 和 `15`。

## CockroachDB **(<ins>预览版</ins>)**

CockroachDB 支持目前处于预览阶段，需使用 [Atlas 迁移引擎](migrate.md#atlas-integration)。  
当前集成测试基于 CRDB 版本 `v21.2.11`。

## SQLite

通过 [Atlas](https://github.com/ariga/atlas) 驱动，SQLite 支持 [迁移](migrate.md) 章节中提到的所有功能特性。需注意部分变更（如列修改）会通过临时表实现，具体操作流程遵循 [SQLite 官方文档](https://www.sqlite.org/lang_altertable.html#otheralter) 描述的步骤。

## Gremlin

Gremlin 不支持迁移及索引功能，**<ins>目前视为实验性功能</ins>**。

## TiDB **(<ins>预览版</ins>)**

TiDB 支持目前处于预览阶段，需使用 [Atlas 迁移引擎](migrate.md#atlas-integration)。  
TiDB 与 MySQL 兼容，因此 MySQL 支持的各项功能理论上也适用于 TiDB。  
已知兼容性问题列表请访问：https://docs.pingcap.com/tidb/stable/mysql-compatibility  
当前集成测试基于 TiDB 版本 `5.4.0`、`6.0.0`。