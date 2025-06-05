---
title: Custom migrations
id: custom-migrations
---

:::info[支持代码库]

本节描述的变更可在配套代码库的
[PR #7](https://github.com/rotemtam/ent-versioned-migrations-demo/pull/7/files)
中查看。

:::

## 自定义迁移

某些情况下，您可能需要编写Atlas无法自动生成的定制化迁移脚本。这在需要执行Ent当前不支持的数据库变更，或需要预置初始化数据时特别有用。

本节我们将学习如何为项目添加自定义迁移。假设我们需要为用户表预置一些初始数据作为示例场景。

## 创建自定义迁移

首先在项目中添加新的迁移文件：

```shell
atlas migrate new seed_users --dir file://ent/migrate/migrations
```

可以看到`ent/migrate/migrations`目录下新增了名为`20221115102552_seed_users.sql`的文件。

打开该文件并添加以下SQL语句：

```sql
INSERT INTO `users` (`name`, `email`, `title`)
VALUES ('Jerry Seinfeld', 'jerry@seinfeld.io', 'Mr.'),
       ('George Costanza', 'george@costanza.io', 'Mr.')
```

## 重新计算校验文件

尝试运行新的自定义迁移：

```shell
atlas migrate apply --dir file://ent/migrate/migrations --url mysql://root:pass@localhost:3306/db
```

Atlas报错如下：

```text
You have a checksum error in your migration directory.
This happens if you manually create or edit a migration file.
Please check your migration files and run

'atlas migrate hash'

to re-hash the contents and resolve the error

Error: checksum mismatch
```

Atlas通过[迁移目录完整性](https://atlasgo.io/concepts/migration-directory-integrity)机制确保线性迁移历史。这能防止多名开发者在并行开发时产生迁移历史冲突。

我们需要重新计算迁移目录的哈希值来解决此错误：

```shell
atlas migrate hash --dir file://ent/migrate/migrations
```

再次运行`atlas migrate apply`，迁移将被成功应用：

```text
atlas migrate apply --dir file://ent/migrate/migrations --url mysql://root:pass@localhost:3306/db
```

Atlas输出如下：

```text
Migrating to version 20221115102552 from 20221115101649 (1 migrations in total):

  -- migrating version 20221115102552
    -> INSERT INTO `users` (`name`, `email`, `title`)
VALUES ('Jerry Seinfeld', 'jerry@seinfeld.io', 'Mr.'),
       ('George Costanza', 'george@costanza.io', 'Mr.')
  -- ok (9.077102ms)

  -------------------------
  -- 19.857555ms
  -- 1 migrations 
  -- 1 sql statements
```

下一节我们将学习如何使用Atlas的[代码检查](https://atlasgo.io/versioned/lint)功能自动验证迁移安全性。