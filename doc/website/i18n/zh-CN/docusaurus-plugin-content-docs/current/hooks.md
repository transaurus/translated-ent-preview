---
id: hooks
title: Hooks
---

`Hooks`选项允许在执行图数据变更操作前后添加自定义逻辑。

## 数据变更

变更操作是指会修改数据库状态的操作。例如向图中添加新节点、删除两个节点间的边，或批量删除节点等。

共有5种变更类型：

- `Create` - 在图中创建节点
- `UpdateOne` - 更新图中某个节点（例如递增其字段值）
- `Update` - 批量更新符合谓词条件的节点
- `DeleteOne` - 从图中删除单个节点
- `Delete` - 删除所有符合谓词条件的节点

每种生成的节点类型都有对应的变更类型。例如所有[`User`构建器](crud.mdx#create-an-entity)共享同一个生成的`UserMutation`对象，但所有构建器类型都实现了通用的<a target="_blank" href="https://pkg.go.dev/entgo.io/ent?tab=doc#Mutation">`ent.Mutation`</a>接口。

:::info[数据库触发器支持]
与数据库触发器不同，钩子是在应用层而非数据库层执行。如需在数据库层执行特定逻辑，请参考[模式迁移指南](/docs/migration/triggers)使用数据库触发器。
:::

## 钩子

钩子是接收<a target="_blank" href="https://pkg.go.dev/entgo.io/ent?tab=doc#Mutator">`ent.Mutator`</a>并返回变更器的函数，作为变更器之间的中间件运行，类似于常见的HTTP中间件模式。

```go
type (
	// Mutator is the interface that wraps the Mutate method.
	Mutator interface {
		// Mutate apply the given mutation on the graph.
		Mutate(context.Context, Mutation) (Value, error)
	}

	// Hook defines the "mutation middleware". A function that gets a Mutator
	// and returns a Mutator. For example:
	//
	//	hook := func(next ent.Mutator) ent.Mutator {
	//		return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
	//			fmt.Printf("Type: %s, Operation: %s, ConcreteType: %T\n", m.Type(), m.Op(), m)
	//			return next.Mutate(ctx, m)
	//		})
	//	}
	//
	Hook func(Mutator) Mutator
)
```

存在两种变更钩子——**模式钩子**和**运行时钩子**。**模式钩子**主要用于在模式中定义自定义变更逻辑，**运行时钩子**则用于添加日志、指标、追踪等功能。下面分别说明：

## 运行时钩子

首先看一个记录所有类型变更操作的简单示例：

```go
func main() {
	client, err := ent.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	if err != nil {
		log.Fatalf("failed opening connection to sqlite: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	// Run the auto migration tool.
	if err := client.Schema.Create(ctx); err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
    // Add a global hook that runs on all types and all operations.
	client.Use(func(next ent.Mutator) ent.Mutator {
		return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
			start := time.Now()
			defer func() {
				log.Printf("Op=%s\tType=%s\tTime=%s\tConcreteType=%T\n", m.Op(), m.Type(), time.Since(start), m)
			}()
			return next.Mutate(ctx, m)
		})
	})
    client.User.Create().SetName("a8m").SaveX(ctx)
    // Output:
    // 2020/03/21 10:59:10 Op=Create	Type=User	Time=46.23µs	ConcreteType=*ent.UserMutation
}
```

全局钩子适用于添加追踪、指标、日志等功能。但有时需要更细粒度的控制：

```go
func main() {
    // <client was defined in the previous block>

    // Add a hook only on user mutations.
	client.User.Use(func(next ent.Mutator) ent.Mutator {
        // Use the "<project>/ent/hook" to get the concrete type of the mutation.
		return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
			return next.Mutate(ctx, m)
		})
	})
    
    // Add a hook only on update operations.
    client.Use(hook.On(Logger(), ent.OpUpdate|ent.OpUpdateOne))
    
    // Reject delete operations.
    client.Use(hook.Reject(ent.OpDelete|ent.OpDeleteOne))
}
```

假设需要在多个类型（如`Group`和`User`）间共享字段变更钩子，主要有两种实现方式：

```go
// Option 1: use type assertion.
client.Use(func(next ent.Mutator) ent.Mutator {
    type NameSetter interface {
        SetName(value string)
    }
    return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
        // A schema with a "name" field must implement the NameSetter interface. 
        if ns, ok := m.(NameSetter); ok {
            ns.SetName("Ariel Mashraki")
        }
        return next.Mutate(ctx, m)
    })
})

// Option 2: use the generic ent.Mutation interface.
client.Use(func(next ent.Mutator) ent.Mutator {
	return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
        if err := m.SetField("name", "Ariel Mashraki"); err != nil {
            // An error is returned, if the field is not defined in
			// the schema, or if the type mismatch the field type.
        }
        return next.Mutate(ctx, m)
    })
})
```

## 模式钩子

模式钩子定义在类型模式中，仅作用于匹配该模式的变更操作。将钩子定义在模式中的动机是将节点类型的所有逻辑集中存放在模式这一处。

```go
package schema

import (
	"context"
	"fmt"

    gen "<project>/ent"
    "<project>/ent/hook"

	"entgo.io/ent"
)

// Card holds the schema definition for the CreditCard entity.
type Card struct {
	ent.Schema
}

// Hooks of the Card.
func (Card) Hooks() []ent.Hook {
	return []ent.Hook{
		// First hook.
		hook.On(
			func(next ent.Mutator) ent.Mutator {
				return hook.CardFunc(func(ctx context.Context, m *gen.CardMutation) (ent.Value, error) {
					if num, ok := m.Number(); ok && len(num) < 10 {
						return nil, fmt.Errorf("card number is too short")
					}
					return next.Mutate(ctx, m)
				})
			},
			// Limit the hook only for these operations.
			ent.OpCreate|ent.OpUpdate|ent.OpUpdateOne,
		),
		// Second hook.
		func(next ent.Mutator) ent.Mutator {
			return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
				if s, ok := m.(interface{ SetName(string) }); ok {
					s.SetName("Boring")
				}
				v, err := next.Mutate(ctx, m)
				// Post mutation action.
				fmt.Println("new value:", v)
				return v, err
			})
		},
	}
}
```

## 钩子注册

使用[**模式钩子**](#schema-hooks)时，可能出现模式包与生成ent包之间的循环导入。为避免这种情况，ent会生成`ent/runtime`包来负责在运行时注册模式钩子。

:::important
用户**必须**导入`ent/runtime`才能注册模式钩子。该包可导入在`main`包（靠近数据库驱动导入位置），或创建`ent.Client`的包中：

```go
import _ "<project>/ent/runtime"
```
:::

#### 循环导入错误

首次在项目中设置模式钩子时，可能会遇到如下错误：

```text
entc/load: parse schema dir: import cycle not allowed: [ent/schema ent/hook ent/ ent/schema]
To resolve this issue, move the custom types used by the generated code to a separate package: "Type1", "Type2"
```

该错误可能由于生成的代码依赖`ent/schema`包中定义的自定义类型，但该包又导入了`ent/hook`包。这种对`ent`包的间接导入形成了循环依赖，从而导致错误发生。请按以下步骤解决：

- 首先，注释掉`ent/schema`中所有钩子、隐私策略或拦截器的使用代码
- 将`ent/schema`中定义的自定义类型移至新包，例如`ent/schema/schematype`
- 运行`go generate ./...`更新生成的`ent`包指向新包（例如`schema.T`变为`schematype.T`）
- 取消注释钩子、隐私策略或拦截器代码，再次运行`go generate ./...`，此时代码生成应能正常通过

## 执行顺序

钩子按照注册到客户端的顺序依次调用。因此`client.Use(f, g, h)`会在变更操作时执行`f(g(h(...)))`的链式调用。

需注意**运行时钩子**会先于**模式钩子**执行。例如当`g`和`h`在模式中定义，而`f`通过`client.Use(...)`注册时，实际执行顺序为：`f(g(h(...)))`。

## 钩子辅助工具

生成的hooks包提供了若干辅助函数，可用于控制钩子的执行时机。

```go
package schema

import (
	"context"
	"fmt"

	"<project>/ent/hook"

	"entgo.io/ent"
	"entgo.io/ent/schema/mixin"
)


type SomeMixin struct {
	mixin.Schema
}

func (SomeMixin) Hooks() []ent.Hook {
    return []ent.Hook{
        // Execute "HookA" only for the UpdateOne and DeleteOne operations.
        hook.On(HookA(), ent.OpUpdateOne|ent.OpDeleteOne),

        // Don't execute "HookB" on Create operation.
        hook.Unless(HookB(), ent.OpCreate),

        // Execute "HookC" only if the ent.Mutation is changing the "status" field,
        // and clearing the "dirty" field.
        hook.If(HookC(), hook.And(hook.HasFields("status"), hook.HasClearedFields("dirty"))),

        // Disallow changing the "password" field on Update (many) operation.
        hook.If(
            hook.FixedError(errors.New("password cannot be edited on update many")),
            hook.And(
                hook.HasOp(ent.OpUpdate),
                hook.Or(
                	hook.HasFields("password"),
                	hook.HasClearedFields("password"),
                ),
            ),
        ),
    }
}
```

## 事务钩子

钩子也可注册在活动事务上，将在`Tx.Commit`或`Tx.Rollback`时执行。更多信息请参阅[事务文档](transactions.md#hooks)。

## 代码生成钩子

`entc`包支持为代码生成阶段添加钩子（中间件）列表。详细信息请参考[代码生成文档](code-gen.md#code-generation-hooks)。