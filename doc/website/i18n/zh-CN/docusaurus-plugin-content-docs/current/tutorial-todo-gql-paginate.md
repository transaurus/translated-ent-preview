---
id: tutorial-todo-gql-paginate
title: Relay Cursor Connections (Pagination)
sidebar_label: Relay Cursor Connections
---

本节我们将延续[GraphQL示例](tutorial-todo-gql.mdx)，讲解如何实现[Relay游标连接规范](https://relay.dev/graphql/connections.htm)。若您不熟悉游标连接接口，请阅读摘自[relay.dev](https://relay.dev/graphql/connections.htm#sel-DABDDDAADFA0E3kM)的说明：

> 在查询中，连接模型为结果集的分片与分页提供了标准化机制。
> 
> 在响应中，连接模型通过标准化方式提供游标标识，并告知客户端是否还有更多结果。
> 
> 以下查询示例展示了这四个特性：
> ```graphql
> {
>   user {
>     id
>     name
>     friends(first: 10, after: "opaqueCursor") {
>       edges {
>         cursor
>         node {
>           id
>           name
>         }
>       }
>       pageInfo {
>         hasNextPage
>       }
>     }
>   }
> }
> ```

#### 克隆代码（可选）

本教程代码托管于[github.com/a8m/ent-graphql-example](https://github.com/a8m/ent-graphql-example)，每个步骤都有对应的Git标签。若想跳过基础配置直接使用GraphQL服务器的初始版本，可通过以下命令克隆仓库：

```console
git clone git@github.com:a8m/ent-graphql-example.git
cd ent-graphql-example 
go run ./cmd/todo/
```

## 为Schema添加注解

通过在任意可比较的Ent字段上添加`entgql.Annotation`注解即可定义排序规则。注意指定的`OrderField`名称必须大写，且需与GraphQL schema中的枚举值匹配。

```go title="ent/schema/todo.go"
func (Todo) Fields() []ent.Field {
    return []ent.Field{
		field.Text("text").
			NotEmpty().
			Annotations(
				entgql.OrderField("TEXT"),
			),
		field.Time("created_at").
			Default(time.Now).
			Immutable().
			Annotations(
				entgql.OrderField("CREATED_AT"),
			),
		field.Enum("status").
			NamedValues(
				"InProgress", "IN_PROGRESS",
				"Completed", "COMPLETED",
			).
			Default("IN_PROGRESS").
			Annotations(
				entgql.OrderField("STATUS"),
			),
		field.Int("priority").
			Default(0).
			Annotations(
				entgql.OrderField("PRIORITY"),
			),
    }
}
```

## 多字段排序

默认情况下，`orderBy`参数仅接受单个`<T>Order`值。要启用多字段排序，只需在目标schema中添加`entgql.MultiOrder()`注解。

```go title="ent/schema/todo.go"
func (Todo) Annotations() []schema.Annotation {
    return []schema.Annotation{
        //highlight-next-line
        entgql.MultiOrder(),
    }
}
```

当该注解被添加到`Todo` schema后，`orderBy`参数类型将从`TodoOrder`变更为`[TodoOrder!]`。

## 按关联边数量排序

非唯一关联边可通过`OrderField`注解实现基于特定边类型计数的节点排序。

```go title="ent/schema/todo/go"
func (Todo) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("children", Todo.Type).
			Annotations(
				entgql.RelayConnection(),
				// highlight-next-line
				entgql.OrderField("CHILDREN_COUNT"),
			).
			From("parent").
			Unique(),
	}
}
```

:::info
此排序项的命名规则为：`UPPER(<边名称>)_COUNT`。例如`CHILDREN_COUNT`或`POSTS_COUNT`。
:::

## 按关联边字段排序

唯一关联边可通过`OrderField`注解实现基于关联边字段的节点排序（例如：按作者姓名排序文章，或根据父级优先级排序待办事项）。注意：要按边字段排序，该字段必须在被引用类型中标注为`OrderField`。

此排序项的命名规则为：`UPPER(<边名称>)_<边字段>`。例如`PARENT_PRIORITY`。

```go title="ent/schema/todo.go"
// Fields returns todo fields.
func (Todo) Fields() []ent.Field {
	return []ent.Field{
		// ...
		field.Int("priority").
			Default(0).
			Annotations(
				// highlight-next-line
				entgql.OrderField("PRIORITY"),
			),
    }
}

// Edges returns todo edges.
func (Todo) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("children", Todo.Type).
			From("parent").
			Annotations(
				// highlight-next-line
				entgql.OrderField("PARENT_PRIORITY"),
			).
			Unique(),
    }
}
```

:::info
此排序项的命名规则为：`UPPER(<边名称>)_<边字段>`。例如`PARENT_PRIORITY`或`AUTHOR_NAME`。
:::

## 为查询添加分页支持

1\. 启用分页的下一步是声明`Todo`类型为Relay Connection。

```go title="ent/schema/todo.go"
func (Todo) Annotations() []schema.Annotation {
	return []schema.Annotation{
        //highlight-next-line
		entgql.RelayConnection(),
		entgql.QueryField(),
		entgql.Mutations(entgql.MutationCreate()),
	}
}
```

2\. 运行`go generate .`后，您会注意到`ent.resolvers.go`文件已变更。转到`Todos`解析器，修改其实现以将分页参数传递给`.Paginate()`：

```go title="ent.resolvers.go" {2-5}
func (r *queryResolver) Todos(ctx context.Context, after *ent.Cursor, first *int, before *ent.Cursor, last *int, orderBy *ent.TodoOrder) (*ent.TodoConnection, error) {
	return r.client.Todo.Query().
		Paginate(ctx, after, first, before, last,
			ent.WithTodoOrder(orderBy),
		)
}
```

:::info[Relay连接配置]

`entgql.RelayConnection()` 函数表示该节点或边应支持分页功能，
因此返回结果将是Relay连接类型而非节点列表（`[T!]!` => `<T>Connection!`）。

在schema `T`（位于ent/schema中）设置此注解，即可为该节点启用分页功能，Ent将自动生成该schema的所有Relay类型，
包括：`<T>Edge`、`<T>Connection`和`PageInfo`。例如：

```go
func (Todo) Annotations() []schema.Annotation {
	return []schema.Annotation{
		entgql.RelayConnection(),
		entgql.QueryField(),
	}
}
```

在边上设置此注解表示该GraphQL字段应支持嵌套分页，返回类型为Relay连接。例如：

```go
func (Todo) Edges() []ent.Edge {
	return []ent.Edge{
			edge.To("parent", Todo.Type).
				Unique().
				From("children").
				Annotations(entgql.RelayConnection()),
	}
}
```

生成的GraphQL schema将变为：

```diff
-children: [Todo!]!
+children(first: Int, last: Int, after: Cursor, before: Cursor): TodoConnection!
```

:::

## 分页功能使用

现在可以测试新的GraphQL解析器了。首先通过多次运行以下查询创建若干待办事项（修改变量可选）：

```graphql
mutation CreateTodo($input: CreateTodoInput!) {
    createTodo(input: $input) {
        id
        text
        createdAt
        priority
        parent {
            id
        }
    }
}

# Query Variables: { "input": { "text": "Create GraphQL Example", "status": "IN_PROGRESS", "priority": 1 } }
# Output: { "data": { "createTodo": { "id": "2", "text": "Create GraphQL Example", "createdAt": "2021-03-10T15:02:18+02:00", "priority": 1, "parent": null } } }
```

然后使用分页API查询待办列表：

```graphql
query {
    todos(first: 3, orderBy: {direction: DESC, field: TEXT}) {
        edges {
            node {
                id
                text
            }
            cursor
        }
    }
}

# Output: { "data": { "todos": { "edges": [ { "node": { "id": "16", "text": "Create GraphQL Example" }, "cursor": "gqFpEKF2tkNyZWF0ZSBHcmFwaFFMIEV4YW1wbGU" }, { "node": { "id": "15", "text": "Create GraphQL Example" }, "cursor": "gqFpD6F2tkNyZWF0ZSBHcmFwaFFMIEV4YW1wbGU" }, { "node": { "id": "14", "text": "Create GraphQL Example" }, "cursor": "gqFpDqF2tkNyZWF0ZSBHcmFwaFFMIEV4YW1wbGU" } ] } } }
```

还可以使用上条查询获取的游标来获取其后所有项目：

```graphql
query {
    todos(first: 3, after:"gqFpEKF2tkNyZWF0ZSBHcmFwaFFMIEV4YW1wbGU", orderBy: {direction: DESC, field: TEXT}) {
        edges {
            node {
                id
                text
            }
            cursor
        }
    }
}

# Output: { "data": { "todos": { "edges": [ { "node": { "id": "15", "text": "Create GraphQL Example" }, "cursor": "gqFpD6F2tkNyZWF0ZSBHcmFwaFFMIEV4YW1wbGU" }, { "node": { "id": "14", "text": "Create GraphQL Example" }, "cursor": "gqFpDqF2tkNyZWF0ZSBHcmFwaFFMIEV4YW1wbGU" }, { "node": { "id": "13", "text": "Create GraphQL Example" }, "cursor": "gqFpDaF2tkNyZWF0ZSBHcmFwaFFMIEV4YW1wbGU" } ] } } }
```

---

很好！通过简单修改，我们的应用已支持分页功能。请继续阅读下一节，我们将讲解如何实现GraphQL字段集合，
并了解Ent如何解决GraphQL解析器中的*"N+1问题"*。