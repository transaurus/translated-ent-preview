---
title: Serverless GraphQL using with AWS and ent
author: Bodo Kaiser
authorURL: "https://github.com/bodokaiser"
authorImageURL: "https://avatars.githubusercontent.com/u/1780466?v=4"
image: https://entgo.io/images/assets/appsync/share.png
---

[GraphQL][1] 是一种用于 HTTP API 的查询语言，通过静态类型接口便捷地呈现当今复杂的数据层级结构。使用 GraphQL 的一种方式是导入实现 GraphQL 服务器的库，并注册自定义解析器来实现数据库接口。另一种方式是使用 GraphQL 云服务来实现 GraphQL 服务器，并将无服务器云函数注册为解析器。云服务的众多优势中，最显著的实践价值在于解析器的独立性与可组合性——例如可以编写一个解析器连接关系型数据库，另一个连接搜索数据库。

下文我们将基于 [亚马逊云服务(AWS)][2] 构建此类架构，具体采用 [AWS AppSync][3] 作为 GraphQL 云服务，使用 [AWS Lambda][4] 运行关系型数据库解析器，该解析器采用 [Go][5] 语言开发并以 [Ent][6] 作为实体框架。相较于 AWS Lambda 最流行的运行时 Nodejs，Go 具备更快的启动速度、更高的性能表现，以及（个人认为）更优的开发体验。而 Ent 作为补充，通过类型安全的方式访问关系型数据库的创新方案，在 Go 生态中堪称独树一帜。综上所述，结合 Ent 与 AWS Lambda 作为 AWS AppSync 解析器的架构，能完美应对当今严苛的 API 需求。

后续章节我们将依次完成：在 AWS AppSync 中配置 GraphQL、部署运行 Ent 的 AWS Lambda 函数、提出整合 Ent 与 AWS Lambda 事件处理器的 Go 实现方案、对 Ent 函数进行快速测试，最终将其注册为 AWS AppSync API 的数据源并配置解析器（用于定义 GraphQL 请求到 AWS Lambda 事件的映射）。请注意本教程需要 AWS 账户及**可公开访问的 Postgres 数据库 URL**，可能会产生费用。

### 配置 AWS AppSync 架构

在 AWS AppSync 中配置 GraphQL 架构时，请登录 AWS 账户并通过导航栏选择 AppSync 服务。在 AppSync 服务的落地页点击"Create API"按钮，进入"Getting Started"页面：

在标有"Customize your API or import from Amazon DynamoDB"的面板顶部选择"Build from scratch"选项，点击该面板的"Start"按钮。此时您将看到可输入 API 名称的表单。本教程中输入"Todo"（如下图所示），点击"Create"按钮。

创建 AppSync API 后，您将看到包含三个面板的落地页：架构定义面板、API 查询面板、以及 AppSync 应用集成面板（如下图所示）。

点击第一个面板中的"Edit Schema"按钮，将原有架构替换为以下 GraphQL 架构：

```graphql
input AddTodoInput {
	title: String!
}

type AddTodoOutput {
	todo: Todo!
}

type Mutation {
	addTodo(input: AddTodoInput!): AddTodoOutput!
	removeTodo(input: RemoveTodoInput!): RemoveTodoOutput!
}

type Query {
	todos: [Todo!]!
	todo(id: ID!): Todo
}

input RemoveTodoInput {
	todoId: ID!
}

type RemoveTodoOutput {
	todo: Todo!
}

type Todo {
	id: ID!
	title: String!
}

schema {
	query: Query
	mutation: Mutation
}
```

替换架构后，系统会进行简短验证，此时您应能点击右上角的"Save Schema"按钮，并看到如下界面：

若此时向 AppSync API 发送 GraphQL 请求，由于未配置解析器，API 将返回错误。我们将在通过 AWS Lambda 部署 Ent 函数后配置解析器。

详细解释此 GraphQL 架构已超出本教程范围。简而言之，该架构通过 `Query.todos` 实现待办事项列表查询，通过 `Query.todo` 实现单项查询，通过 `Mutation.createTodo` 实现创建事项，通过 `Mutation.deleteTodo` 实现删除操作。该 GraphQL API 类似于简单的 REST API 设计中的 `/todos` 资源端点（对应 `GET /todos`、`GET /todos/:id`、`POST /todos` 和 `DELETE /todos/:id`）。关于 GraphQL 架构设计的细节（如 `Query` 和 `Mutation` 对象的参数与返回值），我遵循 [GitHub GraphQL API](https://docs.github.com/en/graphql/reference/queries) 的实践方案。

### 配置 AWS Lambda

在完成AppSync API配置后，下一步是创建运行Ent的AWS Lambda函数。通过导航栏进入AWS Lambda服务，即可看到列出所有函数的服务首页：

点击右上角的"创建函数"按钮，在上方面板选择"从头开始编写"。将函数命名为"ent"，运行时选择"Go 1.x"，然后点击底部的"创建函数"按钮。随后将进入"ent"函数的详情页面：

在审查Go代码和上传编译好的二进制文件前，我们需要调整"ent"函数的一些默认设置。首先将默认处理程序名称从`hello`改为`main`，这与编译后的Go二进制文件名一致：

其次添加环境变量`DATABASE_URL`，用于配置数据库网络参数和凭据：

要建立数据库连接，需传入[数据源名称(DSN)](https://en.wikipedia.org/wiki/Data_source_name)，例如`postgres://用户名:密码@主机名/数据库名`。AWS Lambda默认会加密环境变量，这是提供数据库连接参数既快速又安全的方式。另一种方案是使用AWS Secretsmanager服务在Lambda函数冷启动时动态获取凭据，这种方式支持凭据轮换等功能。第三种选择是通过AWS IAM处理数据库授权。

如果在AWS RDS中创建Postgres数据库，默认用户名和数据库名均为`postgres`。可通过修改RDS实例来重置密码。

### 配置Ent并部署AWS Lambda

现在我们将审查、编译数据库Go二进制文件并部署到"ent"函数。完整源代码可参考[bodokaiser/entgo-aws-appsync](https://github.com/bodokaiser/entgo-aws-appsync)仓库。

首先创建一个空目录并进入：

```console
mkdir entgo-aws-appsync
cd entgo-aws-appsync
```

接着初始化一个新的Go模块来管理项目：

```console
go mod init entgo-aws-appsync
```

然后创建`Todo`模型并引入ent依赖：

```console
go run -mod=mod entgo.io/ent/cmd/ent new Todo
```

并添加`title`字段：

```go {15-17} title="ent/schema/todo.go"
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
)

// Todo holds the schema definition for the Todo entity.
type Todo struct {
	ent.Schema
}

// Fields of the Todo.
func (Todo) Fields() []ent.Field {
	return []ent.Field{
		field.String("title"),
	}
}

// Edges of the Todo.
func (Todo) Edges() []ent.Edge {
	return nil
}
```

最后执行Ent代码生成：

```console
go generate ./ent
```

使用Ent编写一组解析器函数，实现待办事项的创建、读取和删除操作：

```go title="internal/handler/resolver.go"
package resolver

import (
	"context"
	"fmt"
	"strconv"

	"entgo-aws-appsync/ent"
	"entgo-aws-appsync/ent/todo"
)

// TodosInput is the input to the Todos query.
type TodosInput struct{}

// Todos queries all todos.
func Todos(ctx context.Context, client *ent.Client, input TodosInput) ([]*ent.Todo, error) {
	return client.Todo.
		Query().
		All(ctx)
}

// TodoByIDInput is the input to the TodoByID query.
type TodoByIDInput struct {
	ID string `json:"id"`
}

// TodoByID queries a single todo by its id.
func TodoByID(ctx context.Context, client *ent.Client, input TodoByIDInput) (*ent.Todo, error) {
	tid, err := strconv.Atoi(input.ID)
	if err != nil {
		return nil, fmt.Errorf("failed parsing todo id: %w", err)
	}
	return client.Todo.
		Query().
		Where(todo.ID(tid)).
		Only(ctx)
}

// AddTodoInput is the input to the AddTodo mutation.
type AddTodoInput struct {
	Title string `json:"title"`
}

// AddTodoOutput is the output to the AddTodo mutation.
type AddTodoOutput struct {
	Todo *ent.Todo `json:"todo"`
}

// AddTodo adds a todo and returns it.
func AddTodo(ctx context.Context, client *ent.Client, input AddTodoInput) (*AddTodoOutput, error) {
	t, err := client.Todo.
		Create().
		SetTitle(input.Title).
		Save(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed creating todo: %w", err)
	}
	return &AddTodoOutput{Todo: t}, nil
}

// RemoveTodoInput is the input to the RemoveTodo mutation.
type RemoveTodoInput struct {
	TodoID string `json:"todoId"`
}

// RemoveTodoOutput is the output to the RemoveTodo mutation.
type RemoveTodoOutput struct {
	Todo *ent.Todo `json:"todo"`
}

// RemoveTodo removes a todo and returns it.
func RemoveTodo(ctx context.Context, client *ent.Client, input RemoveTodoInput) (*RemoveTodoOutput, error) {
	t, err := TodoByID(ctx, client, TodoByIDInput{ID: input.TodoID})
	if err != nil {
		return nil, fmt.Errorf("failed querying todo with id %q: %w", input.TodoID, err)
	}
	err = client.Todo.
		DeleteOne(t).
		Exec(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed deleting todo with id %q: %w", input.TodoID, err)
	}
	return &RemoveTodoOutput{Todo: t}, nil
}
```

解析器函数使用输入结构体来映射GraphQL请求参数，输出结构体则支持返回多个对象以处理复杂操作。

为了将Lambda事件映射到解析器函数，我们实现了一个根据事件中`action`字段进行路由的处理器：

```go title="internal/handler/handler.go"
package handler

import (
	"context"
	"encoding/json"
	"fmt"
	"log"

	"entgo-aws-appsync/ent"
	"entgo-aws-appsync/internal/resolver"
)

// Action specifies the event type.
type Action string

// List of supported event actions.
const (
	ActionMigrate Action = "migrate"

	ActionTodos      = "todos"
	ActionTodoByID   = "todoById"
	ActionAddTodo    = "addTodo"
	ActionRemoveTodo = "removeTodo"
)

// Event is the argument of the event handler.
type Event struct {
	Action Action          `json:"action"`
	Input  json.RawMessage `json:"input"`
}

// Handler handles supported events.
type Handler struct {
	client *ent.Client
}

// Returns a new event handler.
func New(c *ent.Client) *Handler {
	return &Handler{
		client: c,
	}
}

// Handle implements the event handling by action.
func (h *Handler) Handle(ctx context.Context, e Event) (interface{}, error) {
	log.Printf("action %s with payload %s\n", e.Action, e.Input)

	switch e.Action {
	case ActionMigrate:
		return nil, h.client.Schema.Create(ctx)
	case ActionTodos:
		var input resolver.TodosInput
		return resolver.Todos(ctx, h.client, input)
	case ActionTodoByID:
		var input resolver.TodoByIDInput
		if err := json.Unmarshal(e.Input, &input); err != nil {
			return nil, fmt.Errorf("failed parsing %s params: %w", ActionTodoByID, err)
		}
		return resolver.TodoByID(ctx, h.client, input)
	case ActionAddTodo:
		var input resolver.AddTodoInput
		if err := json.Unmarshal(e.Input, &input); err != nil {
			return nil, fmt.Errorf("failed parsing %s params: %w", ActionAddTodo, err)
		}
		return resolver.AddTodo(ctx, h.client, input)
	case ActionRemoveTodo:
		var input resolver.RemoveTodoInput
		if err := json.Unmarshal(e.Input, &input); err != nil {
			return nil, fmt.Errorf("failed parsing %s params: %w", ActionRemoveTodo, err)
		}
		return resolver.RemoveTodo(ctx, h.client, input)
	}

	return nil, fmt.Errorf("invalid action %q", e.Action)
}
```

除了解析器操作外，我们还添加了迁移操作，这是暴露数据库迁移的便捷方式。

最后需要将`Handler`类型的实例注册到AWS Lambda库：

```go title="lambda/main.go"
package main

import (
	"database/sql"
	"log"
	"os"

	"entgo.io/ent/dialect"
	entsql "entgo.io/ent/dialect/sql"

	"github.com/aws/aws-lambda-go/lambda"
	_ "github.com/jackc/pgx/v4/stdlib"

	"entgo-aws-appsync/ent"
	"entgo-aws-appsync/internal/handler"
)

func main() {
	// open the database connection using the pgx driver
	db, err := sql.Open("pgx", os.Getenv("DATABASE_URL"))
	if err != nil {
		log.Fatalf("failed opening database connection: %v", err)
	}

	// initiate the ent database client for the Postgres database
	client := ent.NewClient(ent.Driver(entsql.OpenDB(dialect.Postgres, db)))
	defer client.Close()

	// register our event handler to listen on Lambda events
	lambda.Start(handler.New(client).Handle)
}
```

`main`的函数体会在AWS Lambda冷启动时执行。冷启动后Lambda函数进入"热"状态，此时仅执行事件处理代码，这使得Lambda执行非常高效。

运行以下命令编译并部署Go代码：

```console
GOOS=linux go build -o main ./lambda
zip function.zip main
aws lambda update-function-code --function-name ent --zip-file fileb://function.zip
```

第一条命令生成名为`main`的编译二进制文件。第二条命令将二进制文件压缩为ZIP归档，这是AWS Lambda要求的格式。第三条命令用新的ZIP归档替换名为`ent`的AWS Lambda函数代码。若使用多个AWS账户，可通过`--profile <你的aws配置>`参数指定。

成功部署AWS Lambda后，在网页控制台中打开"ent"函数的"测试"选项卡，使用"migrate"操作进行调用：

若操作成功，您将看到绿色反馈框，此时可测试"todos"操作的结果：

如果测试执行失败，很可能是数据库连接存在问题。

### 配置AWS AppSync解析器

成功部署"ent"函数后，我们需要将其注册为AppSync API的数据源，并配置模式解析器以将AppSync请求映射到Lambda事件。首先在网页控制台中打开AWS AppSync API，左侧导航面板进入"数据源"。

点击右上角的"创建数据源"按钮，开始注册"ent"函数作为数据源：

现在打开AppSync API的GraphQL模式，在右侧边栏中找到`Query`类型。点击`Query.Todos`类型旁的"附加"按钮：

在`Query.todos`的解析器视图中，选择Lambda函数作为数据源，启用请求映射模板选项，

并复制以下模板：

```vtl title="Query.todos"
{
  "version" : "2017-02-28",
  "operation": "Invoke",
  "payload": {
    "action": "todos"
  }
}
```

对剩余的`Query`和`Mutation`类型重复相同流程：

```vtl title="Query.todo"
{
  "version" : "2017-02-28",
  "operation": "Invoke",
  "payload": {
    "action": "todo",
    "input": $util.toJson($context.args.input)
  }
}
```

```vtl title="Mutation.addTodo"
{
  "version" : "2017-02-28",
  "operation": "Invoke",
  "payload": {
    "action": "addTodo",
    "input": $util.toJson($context.args.input)
  }
}
```

```vtl title="Mutation.removeTodo"
{
  "version" : "2017-02-28",
  "operation": "Invoke",
  "payload": {
    "action": "removeTodo",
    "input": $util.toJson($context.args.input)
  }
}
```

请求映射模板让我们能构造调用Lambda函数的事件对象。通过`$context`对象，我们可以访问GraphQL请求和认证会话。此外，还可以顺序排列多个解析器，并通过`$context`对象引用各自的输出。原则上也可以定义响应映射模板，但多数情况下直接返回响应对象即可。

### 使用查询浏览器测试AppSync

最简单的测试方式是使用AWS AppSync中的查询浏览器。另一种方法是在AppSync API设置中注册API密钥，使用任何标准GraphQL客户端。

首先创建标题为`foo`的待办事项：

```graphql
mutation MyMutation {
  addTodo(input: {title: "foo"}) {
    todo {
      id
      title
    }
  }
}
```

请求待办事项列表应返回一个标题为`foo`的待办事项：

```graphql
query MyQuery {
  todos {
    title
    id
  }
}
```

通过ID查询`foo`待办事项也应成功：

```graphql
query MyQuery {
  todo(id: "1") {
    title
    id
  }
}
```

### 总结

我们成功部署了用于管理简单待办事项的无服务器GraphQL API，使用AWS AppSync、AWS Lambda和Ent框架。特别提供了通过网页控制台配置AWS AppSync和AWS Lambda的分步指南，并讨论了Go代码结构的设计方案。

本文未涵盖测试和AWS数据库基础设施搭建。这些方面在无服务器范式下比传统模式更具挑战性。例如当多个Lambda函数并行冷启动时，数据库连接池会快速耗尽，需要数据库代理。此外由于难以隔离运行云服务，我们只能进行本地和端到端测试。

尽管如此，所提出的GraphQL服务器方案能良好扩展到实际应用的复杂需求，既能受益于无服务器基础设施，又能享受Ent框架带来的愉悦开发体验。

有问题需要帮助？欢迎加入我们的[Discord服务器](https://discord.gg/qZmPgTE6RX)或[Slack频道](https://entgo.io/docs/slack/)。

:::note[获取更多 Ent 资讯与更新：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 在[Twitter](https://twitter.com/entgo_io)上关注我们
- 加入[Gophers Slack](https://entgo.io/docs/slack)的 #ent 频道
- 加入[Ent Discord 服务器](https://discord.gg/qZmPgTE6RX)

:::

[1]: https://graphql.org

[2]: https://aws.amazon.com

[3]: https://aws.amazon.com/appsync/

[4]: https://aws.amazon.com/lambda/

[5]: https://go.dev

[6]: https://entgo.io