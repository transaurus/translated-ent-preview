---
id: tutorial-todo-gql-schema-generator
title: Schema Generator
sidebar_label: Schema Generator
---

本节我们将延续[GraphQL示例](tutorial-todo-gql.mdx)，讲解如何从`ent/schema`生成类型安全的GraphQL模式。

### 配置Ent

打开`ent/entc.go`文件，添加高亮行（扩展选项）：

```go {5} title="ent/entc.go"
func main() {
	ex, err := entgql.NewExtension(
		entgql.WithWhereInputs(true),
		entgql.WithConfigPath("../gqlgen.yml"),
		entgql.WithSchemaGenerator(),
		entgql.WithSchemaPath("../ent.graphql"),
	)
	if err != nil {
		log.Fatalf("creating entgql extension: %v", err)
	}
	opts := []entc.Option{
		entc.Extensions(ex),
		entc.TemplateDir("./template"),
	}
	if err := entc.Generate("./schema", &gen.Config{}, opts...); err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
} 
```

`WithSchemaGenerator`选项用于启用GraphQL模式生成功能。

### 为`Todo`模式添加注解

`entgql.RelayConnection()`注解用于为`Todo`类型生成Relay的`<T>Edge`、`<T>Connection`和`PageInfo`类型。

`entgql.QueryField()`注解用于在`Query`类型中生成`todos`字段。

```go {13,14} title="ent/schema/todo.go"
// Edges of the Todo.
func (Todo) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("parent", Todo.Type).
			Unique().
			From("children").
	}
}

// Annotations of the Todo.
func (Todo) Annotations() []schema.Annotation {
	return []schema.Annotation{
		entgql.RelayConnection(),
		entgql.QueryField(),
	}
}
```

`entgql.RelayConnection()`注解也可用于边字段，以生成first/last/after/before等参数，并将字段类型改为`<T>Connection!`。例如将`children`字段从`children: [Todo!]!`改为`children(first: Int, last: Int, after: Cursor, before: Cursor): TodoConnection!`。您可以为边字段添加`entgql.RelayConnection()`注解：

```go {7} title="ent/schema/todo.go"
// Edges of the Todo.
func (Todo) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("parent", Todo.Type).
			Unique().
			From("children").
			Annotations(entgql.RelayConnection()),
	}
}
```

### 清理手写模式

请从`todo.graphql`中移除以下类型，避免与EntGQL在`ent.graphql`文件中生成的类型冲突。

```diff title="todo.graphql"
-interface Node {
-  id: ID!
-}

"""Maps a Time GraphQL scalar to a Go time.Time struct."""
scalar Time

-"""
-Define a Relay Cursor type:
-https://relay.dev/graphql/connections.htm#sec-Cursor
-"""
-scalar Cursor

-"""
-Define an enumeration type and map it later to Ent enum (Go type).
-https://graphql.org/learn/schema/#enumeration-types
-"""
-enum Status {
-  IN_PROGRESS
-  COMPLETED
-}
-
-type PageInfo {
-  hasNextPage: Boolean!
-  hasPreviousPage: Boolean!
-  startCursor: Cursor
-  endCursor: Cursor
-}

-type TodoConnection {
-  totalCount: Int!
-  pageInfo: PageInfo!
-  edges: [TodoEdge]
-}

-type TodoEdge {
-  node: Todo
-  cursor: Cursor!
-}

-"""The following enums match the entgql annotations in the ent/schema."""
-enum TodoOrderField {
-  CREATED_AT
-  PRIORITY
-  STATUS
-  TEXT
-}

-enum OrderDirection {
-  ASC
-  DESC
-}

input TodoOrder {
  direction: OrderDirection!
  field: TodoOrderField
}

-"""
-Define an object type and map it later to the generated Ent model.
-https://graphql.org/learn/schema/#object-types-and-fields
-"""
-type Todo implements Node {
-  id: ID!
-  createdAt: Time
-  status: Status!
-  priority: Int!
-  text: String!
-  parent: Todo
-  children: [Todo!]
-}

"""
Define an input type for the mutation below.
https://graphql.org/learn/schema/#input-types
Note that this type is mapped to the generated
input type in mutation_input.go.
"""
input CreateTodoInput {
  status: Status! = IN_PROGRESS
  priority: Int
  text: String
  parentID: ID
  ChildIDs: [ID!]
}

"""
Define an input type for the mutation below.
https://graphql.org/learn/schema/#input-types
Note that this type is mapped to the generated
input type in mutation_input.go.
"""
input UpdateTodoInput {
  status: Status
  priority: Int
  text: String
  parentID: ID
  clearParent: Boolean
  addChildIDs: [ID!]
  removeChildIDs: [ID!]
}

"""
Define a mutation for creating todos.
https://graphql.org/learn/queries/#mutations
"""
type Mutation {
  createTodo(input: CreateTodoInput!): Todo!
  updateTodo(id: ID!, input: UpdateTodoInput!): Todo!
  updateTodos(ids: [ID!]!, input: UpdateTodoInput!): [Todo!]!
}

-"""Define a query for getting all todos and support the Node interface."""
-type Query {
-  todos(after: Cursor, first: Int, before: Cursor, last: Int, orderBy: TodoOrder, where: TodoWhereInput): TodoConnection
-  node(id: ID!): Node
-  nodes(ids: [ID!]!): [Node]!
-}
```

### 确保Ent与GQLGen的执行顺序

我们还需要修改`generate.go`文件以确保Ent和GQLGen的执行顺序。这是为了保证GQLGen能正确识别Ent生成的对象并执行代码生成器。

首先删除`ent/generate.go`文件，然后更新`ent/entc.go`文件中的路径配置，因为Ent代码生成将从项目根目录运行。

```diff title="ent/entc.go"
func main() {
	ex, err := entgql.NewExtension(
		entgql.WithWhereInputs(true),
-		entgql.WithConfigPath("../gqlgen.yml"),
+		entgql.WithConfigPath("./gqlgen.yml"),
		entgql.WithSchemaGenerator(),
-		entgql.WithSchemaPath("../ent.graphql"),
+		entgql.WithSchemaPath("./ent.graphql"),
	)
	if err != nil {
		log.Fatalf("creating entgql extension: %v", err)
	}
	opts := []entc.Option{
		entc.Extensions(ex),
-		entc.TemplateDir("./template"),
+		entc.TemplateDir("./ent/template"),
	}
-	if err := entc.Generate("./schema", &gen.Config{}, opts...); err != nil {
+	if err := entc.Generate("./ent/schema", &gen.Config{}, opts...); err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
} 
```

更新`generate.go`以包含ent代码生成。

```go {3} title="generate.go"
package todo

//go:generate go run -mod=mod ./ent/entc.go
//go:generate go run -mod=mod github.com/99designs/gqlgen
```

修改`generate.go`文件后，即可按以下方式执行代码生成：

```console
go generate ./...
```

您将看到`ent.graphql`文件会更新为EntGQL模式生成器产生的新内容。

### 扩展Ent生成的类型

您会注意到生成的类型将包含已定义的`Query`类型对象及其字段：

```graphql
type Query {
  """Fetches an object given its ID."""
  node(
    """ID of the object."""
    id: ID!
  ): Node
  """Lookup nodes by a list of IDs."""
  nodes(
    """The list of node IDs."""
    ids: [ID!]!
  ): [Node]!
  todos(
    """Returns the elements in the list that come after the specified cursor."""
    after: Cursor

    """Returns the first _n_ elements from the list."""
    first: Int

    """Returns the elements in the list that come before the specified cursor."""
    before: Cursor

    """Returns the last _n_ elements from the list."""
    last: Int

    """Ordering options for Todos returned from the connection."""
    orderBy: TodoOrder

    """Filtering options for Todos returned from the connection."""
    where: TodoWhereInput
  ): TodoConnection!
}
```

要为`Query`类型添加新字段，可执行以下操作：

```graphql title="todo.graphql"
extend type Query {
	"""Returns the literal string 'pong'."""
	ping: String!
}
```

您可以扩展任何由Ent生成的类型。若要跳过某个字段，可在该字段或边上使用`entgql.Skip()`注解。

---

完成！如您所见，在适配Schema Generator功能后，我们不再需要手动编写GQL模式。如有疑问或需要入门帮助，欢迎加入我们的[Discord服务器](https://discord.gg/qZmPgTE6RX)或[Slack频道](https://entgo.io/docs/slack)。