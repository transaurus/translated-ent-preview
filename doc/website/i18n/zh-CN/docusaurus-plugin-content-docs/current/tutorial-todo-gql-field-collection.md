---
id: tutorial-todo-gql-field-collection
title: GraphQL Field Collection
sidebar_label: Field Collection
---

本节我们将延续[GraphQL示例](tutorial-todo-gql.mdx)，阐述Ent如何为GraphQL模式实现[字段收集机制](https://spec.graphql.org/June2018/#sec-Field-Collection)，并解决解析器中的"N+1问题"。

#### 克隆代码（可选）

本教程代码库位于[github.com/a8m/ent-graphql-example](https://github.com/a8m/ent-graphql-example)，每个步骤都有对应的Git标签。若想跳过基础配置直接使用GraphQL服务器的初始版本，可通过以下命令克隆仓库：

```console
git clone git@github.com:a8m/ent-graphql-example.git
cd ent-graphql-example 
go run ./cmd/todo/
```

## 问题描述

GraphQL中的*"N+1问题"*指服务器为获取节点关联（即边关系）执行了不必要的数据库查询。潜在查询数量（N+1）与根查询返回的节点数、其关联节点数等呈递归级数关系，实际可能远超N+1量级。

以下列查询为例说明：

```graphql
query {
    users(first: 50) {
        edges {
            node {
                photos {
                    link
                }
                posts {
                    content
                    comments {
                        content
                    }
                }
            }
        }
    }
}
```

该查询试图获取前50个用户及其照片、帖子（含评论）。

**在初级解决方案中**（问题场景），服务器会先执行1次查询获取50个用户，随后为每个用户分别执行：获取照片（50次）、获取帖子（50次）。假设每个用户有10篇帖子，则每篇帖子需再执行1次评论查询（共500次）。最终将产生`1+50+50+500=601`次查询。

![gql-请求树](https://entgo.io/images/assets/request-tree.png)

## Ent解决方案

Ent的字段收集扩展通过[预加载机制](eager-load.mdx)为关联关系（即边）实现自动化的[GraphQL字段收集](https://spec.graphql.org/June2018/#sec-Field-Collection)。当查询请求节点及其边关系时，`entgql`会自动为根查询添加[`With<E>`](eager-load.mdx)步骤，最终客户端只需执行固定次数的数据库查询——该机制支持递归处理。

针对上述GraphQL查询，客户端仅需执行：1次用户查询、1次照片查询、2次帖子及评论查询（总计4次！）。此逻辑同时适用于根查询/解析器与节点API。

## 示例演示

为演示效果，我们**暂时禁用自动字段收集**，将`ent.Client`切换为调试模式后重启GraphQL服务器：

```diff title="ent.resolvers.go"
func (r *queryResolver) Todos(ctx context.Context, after *ent.Cursor, first *int, before *ent.Cursor, last *int, orderBy *ent.TodoOrder) (*ent.TodoConnection, error) {
-	return r.client.Todo.Query().
+	return r.client.Debug().Todo.Query().
		Paginate(ctx, after, first, before, last,
			ent.WithTodoOrder(orderBy),
		)
}
```

执行[分页教程](tutorial-todo-gql-paginate.md)中的GraphQL查询，并在结果中添加`parent`边关系：

```graphql
query {
    todos(last: 10, orderBy: {direction: DESC, field: TEXT}) {
        edges {
            node {
                id
                text
                parent {
                    id
                }
            }
            cursor
        }
    }
}
```

观察进程输出可见：服务器执行了11次查询（1次获取最近10条待办事项，10次获取每条事项的父项）：

```sql
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` ORDER BY `id` ASC LIMIT 11
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` JOIN (SELECT `todo_parent` FROM `todos` WHERE `id` = ?) AS `t1` ON `todos`.`id` = `t1`.`todo_parent` LIMIT 2
```

让我们看Ent如何自动解决问题：当定义Ent边时，`entgql`会将其自动绑定到GraphQL中的使用场景，并在`gql_edge.go`文件中生成节点解析器：

```go title="ent/gql_edge.go"
func (t *Todo) Children(ctx context.Context) ([]*Todo, error) {
	if fc := graphql.GetFieldContext(ctx); fc != nil && fc.Field.Alias != "" {
		result, err = t.NamedChildren(graphql.GetFieldContext(ctx).Field.Alias)
	} else {
		result, err = t.Edges.ChildrenOrErr()
	}
	if IsNotLoaded(err) {
		result, err = t.QueryChildren().All(ctx)
	}
	return result, err
}
```

若在**不禁用字段收集**的情况下再次检查进程输出，我们会发现这次服务器仅执行了两个数据库查询：一个用于获取最后10条待办事项，另一个用于获取第一条查询返回的每个待办事项的父项。

```sql
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority`, `todos`.`todo_parent` FROM `todos` ORDER BY `id` DESC LIMIT 11
SELECT DISTINCT `todos`.`id`, `todos`.`text`, `todos`.`created_at`, `todos`.`status`, `todos`.`priority` FROM `todos` WHERE `todos`.`id` IN (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
```

如果运行此示例时遇到问题，请跳转到[初始章节](#clone-the-code-optional)，克隆代码并运行示例。

## 字段映射

[`entgql.MapsTo`](https://pkg.go.dev/entgo.io/contrib/entgql#MapsTo) 允许您在Ent模式与GraphQL模式之间添加自定义字段/边映射。当您需要在GraphQL模式中以不同名称暴露字段或边时，此功能非常实用。例如：

```go
// One to one mapping.
field.Int("priority").
	Annotations(
		entgql.OrderField("PRIORITY_ORDER"),
		entgql.MapsTo("priorityOrder"),
	)

// Multiple GraphQL fields can map to the same Ent field.
field.Int("category_id").
	Annotations(
		entgql.MapsTo("categoryID", "category_id", "categoryX"),
	)
```

---

完成！通过对Ent模式定义启用自动字段收集，我们显著提升了应用中GraphQL查询的效率。下一节中，我们将学习如何使GraphQL变更操作具备事务性。