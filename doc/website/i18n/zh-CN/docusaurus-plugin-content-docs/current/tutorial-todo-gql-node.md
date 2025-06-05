---
id: tutorial-todo-gql-node
title: Relay Node Interface
sidebar_label: Relay Node Interface
---

本节我们将延续[GraphQL示例](tutorial-todo-gql.mdx)，讲解如何实现[Relay节点接口](https://relay.dev/graphql/objectidentification.htm)。若您不熟悉节点接口，请阅读以下摘自[relay.dev](https://relay.dev/graphql/objectidentification.htm#sel-DABDDBAADLA0Cl0c)的说明：

> 为便于GraphQL客户端优雅处理缓存和数据重取，GraphQL服务需以标准化方式暴露对象标识符。在查询中，模式应提供通过ID请求对象的标准机制；在响应中，模式需标准化提供这些ID。
>
> 我们将具有标识符的对象称为"节点"。以下查询展示了这两种情况：
>
>  ```graphql
>   {
>       node(id: "4") {
>           id
>          ... on User {
>               name
>           }
>       }
>   }
> ```

#### 克隆代码（可选）

本教程代码托管于[github.com/a8m/ent-graphql-example](https://github.com/a8m/ent-graphql-example)，每个步骤都对应Git标签。若想跳过基础配置直接使用GraphQL服务器的初始版本，可按如下方式克隆仓库：

```console
git clone git@github.com:a8m/ent-graphql-example.git
cd ent-graphql-example 
go run ./cmd/todo/
```

## 实现方案

Ent通过其GraphQL集成支持节点接口。只需几个简单步骤即可在应用中添加支持。首先我们通过编辑`gqlgen.yaml`文件告知`gqlgen`：Ent提供了`Node`接口：

```diff title="gqlgen.yml" {7-9}
# This section declares type mapping between the GraphQL and Go type systems.
models:
  # Defines the ID field as Go 'int'.
  ID:
    model:
      - github.com/99designs/gqlgen/graphql.IntID
  Node:
    model:
      - todo/ent.Noder
```

为应用这些变更，我们需要重新运行代码生成：

```console
go generate .
```

与之前类似，我们需要在`ent.resolvers.go`中实现GraphQL解析器。只需单行代码修改，即可用以下实现替换生成的`gqlgen`代码：

```diff title="ent.resolvers.go"
func (r *queryResolver) Node(ctx context.Context, id int) (ent.Noder, error) {
-	panic(fmt.Errorf("not implemented: Node - node"))
+	return r.client.Noder(ctx, id)
}

func (r *queryResolver) Nodes(ctx context.Context, ids []int) ([]ent.Noder, error) {
-	panic(fmt.Errorf("not implemented: Nodes - nodes"))
+	return r.client.Noders(ctx, ids)
}
```

## 查询节点

现在我们可以测试新的GraphQL解析器了。首先通过多次运行以下查询创建几个待办事项（修改变量可选）：

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

# Query Variables: { "input": { "text":"Create GraphQL Example", "status": "IN_PROGRESS", "priority": 1 } }
# Output: { "data": { "createTodo": { "id": "2", "text": "Create GraphQL Example", "createdAt": "2021-03-10T15:02:18+02:00", "priority": 1, "parent": null } } }
```

对某个待办事项运行**Node**接口将返回：

````graphql
query {
  node(id: 1) {
    id
    ... on Todo {
      text
    }
  }
}

# Output: { "data": { "node": { "id": "1", "text": "Create GraphQL Example" } } }
````

对某个待办事项运行**Nodes**接口将返回：

```graphql
query {
  nodes(ids: [1, 2]) {
    id
    ... on Todo {
      text
    }
  }
}

# Output: { "data": { "nodes": [ { "id": "1", "text": "Create GraphQL Example" }, { "id": "2", "text": "Create Tracing Example" } ] } }
```

---

完成！如您所见，通过少量代码修改，我们的应用现已实现Relay节点接口。下一节我们将展示如何使用Ent实现Relay游标连接规范，这对于支持查询结果的分片与分页非常有用。