---
id: migrate
title: Automatic Migration
---

`ent`的迁移功能支持将数据库模式与项目根目录下`ent/migrate/schema.go`中定义的模式对象保持同步。

## 自动迁移

在应用程序初始化时运行自动迁移逻辑：

```go
if err := client.Schema.Create(ctx); err != nil {
	log.Fatalf("failed creating schema resources: %v", err)
}
```

`Create`会为`ent`项目创建所有必需的数据库资源。默认情况下，`Create`工作在*"仅追加"*模式下；这意味着它只会创建新表和索引、向表追加列或扩展列类型。例如将`int`改为`bigint`。

那么如何删除列或索引呢？

## 删除资源

`WithDropIndex`和`WithDropColumn`是用于删除表列和索引的两个选项。

```go
package main

import (
	"context"
	"log"
	
	"<project>/ent"
	"<project>/ent/migrate"
)

func main() {
	client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
	if err != nil {
		log.Fatalf("failed connecting to mysql: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	// Run migration.
	err = client.Schema.Create(
		ctx, 
		migrate.WithDropIndex(true),
		migrate.WithDropColumn(true), 
	)
	if err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
}
```

若需在调试模式下运行迁移（打印所有SQL查询），执行：

```go
err := client.Debug().Schema.Create(
	ctx, 
	migrate.WithDropIndex(true),
	migrate.WithDropColumn(true),
)
if err != nil {
	log.Fatalf("failed creating schema resources: %v", err)
}
```

## 全局唯一ID

默认情况下，SQL主键从每个表的1开始；这意味着不同类型的多个实体可以共享相同的ID。与AWS Neptune不同，后者的节点ID是UUID。[阅读此文](features.md#globally-unique-id)了解如何在使用Ent与SQL数据库时启用全局唯一ID。

## 离线模式

**随着Atlas即将成为默认迁移引擎，离线迁移将被[版本化迁移](versioned-migrations.mdx)取代。**

离线模式允许您在将模式更改执行到数据库之前，将其写入`io.Writer`。这对于在数据库上执行SQL命令前进行验证，或获取要手动运行的SQL脚本非常有用。

**打印变更**

```go
package main

import (
	"context"
	"log"
	"os"
	
	"<project>/ent"
	"<project>/ent/migrate"
)

func main() {
	client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
	if err != nil {
		log.Fatalf("failed connecting to mysql: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	// Dump migration changes to stdout.
	if err := client.Schema.WriteTo(ctx, os.Stdout); err != nil {
		log.Fatalf("failed printing schema changes: %v", err)
	}
}
```

**将变更写入文件**

```go
package main

import (
	"context"
	"log"
	"os"
	
	"<project>/ent"
	"<project>/ent/migrate"
)

func main() {
	client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
	if err != nil {
		log.Fatalf("failed connecting to mysql: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	// Dump migration changes to an SQL script.
	f, err := os.Create("migrate.sql")
	if err != nil {
		log.Fatalf("create migrate file: %v", err)
	}
	defer f.Close()
	if err := client.Schema.WriteTo(ctx, f); err != nil {
		log.Fatalf("failed printing schema changes: %v", err)
	}
}
```

## 外键

默认情况下，`ent`在定义关系（边）时使用外键来确保数据库端的正确性和一致性。

但`ent`也提供了通过`WithForeignKeys`选项禁用此功能的选项。需注意将此选项设为`false`时，迁移将不会在模式DDL中创建外键，边验证和清理必须由开发人员手动处理。

我们预计在不久的将来会提供一组钩子，用于在应用层实现外键约束。

```go
package main

import (
    "context"
    "log"

    "<project>/ent"
    "<project>/ent/migrate"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err = client.Schema.Create(
        ctx,
        migrate.WithForeignKeys(false), // Disable foreign keys.
    )
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

## 迁移钩子

该框架提供了向迁移阶段添加钩子（中间件）的选项。此选项非常适合修改或过滤迁移正在处理的表，或在数据库中创建自定义资源。

```go
package main

import (
    "context"
    "log"

    "<project>/ent"
    "<project>/ent/migrate"

    "entgo.io/ent/dialect/sql/schema"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err = client.Schema.Create(
        ctx,
        schema.WithHooks(func(next schema.Creator) schema.Creator {
            return schema.CreateFunc(func(ctx context.Context, tables ...*schema.Table) error {
                // Run custom code here.
                return next.Create(ctx, tables...)
            })
        }),
    )
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

## Atlas集成

从v0.10开始，Ent支持使用[Atlas](https://atlasgo.io)运行迁移，这是一个更强大的迁移框架，涵盖当前Ent迁移包不支持的许多功能。要使用Atlas引擎执行迁移，请使用`WithAtlas(true)`选项。

```go {21}
package main

import (
    "context"
    "log"

    "<project>/ent"
    "<project>/ent/migrate"

    "entgo.io/ent/dialect/sql/schema"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err = client.Schema.Create(ctx, schema.WithAtlas(true))
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

除了标准选项（如`WithDropColumn`、`WithGlobalUniqueID`），Atlas集成还提供了用于挂钩到模式迁移步骤的额外选项。

![atlas-migration-process](https://entgo.io/images/assets/migrate-atlas-process.png)

#### Atlas `Diff`和`Apply`钩子

以下是两个展示如何挂钩到Atlas `Diff`和`Apply`步骤的示例。

```go
package main

import (
    "context"
    "log"

    "<project>/ent"
    "<project>/ent/migrate"

	"ariga.io/atlas/sql/migrate"
	atlas "ariga.io/atlas/sql/schema"
	"entgo.io/ent/dialect"
	"entgo.io/ent/dialect/sql/schema"
)

func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err := 	client.Schema.Create(
		ctx,
		// Hook into Atlas Diff process.
		schema.WithDiffHook(func(next schema.Differ) schema.Differ {
			return schema.DiffFunc(func(current, desired *atlas.Schema) ([]atlas.Change, error) {
				// Before calculating changes.
				changes, err := next.Diff(current, desired)
				if err != nil {
					return nil, err
				}
				// After diff, you can filter
				// changes or return new ones.
				return changes, nil
			})
		}),
		// Hook into Atlas Apply process.
		schema.WithApplyHook(func(next schema.Applier) schema.Applier {
			return schema.ApplyFunc(func(ctx context.Context, conn dialect.ExecQuerier, plan *migrate.Plan) error {
				// Example to hook into the apply process, or implement
				// a custom applier. For example, write to a file.
				//
				//	for _, c := range plan.Changes {
				//		fmt.Printf("%s: %s", c.Comment, c.Cmd)
				//		if err := conn.Exec(ctx, c.Cmd, c.Args, nil); err != nil {
				//			return err
				//		}
				//	}
				//
				return next.Apply(ctx, conn, plan)
			})
		}),
	)
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

#### `Diff`钩子示例

如果在 `ent/schema` 中重命名字段，Ent 不会将此变更识别为重命名操作，而是在差异比对阶段生成 `DropColumn` 和 `AddColumn` 变更。一种解决方案是在字段上使用 [StorageKey](schema-fields.mdx#storage-key) 选项，以保留数据库表中的旧列名。不过，通过 Atlas 的 `Diff` 钩子可以更优雅地将 `DropColumn` 和 `AddColumn` 变更替换为 `RenameColumn` 变更。

```go
func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    // ...
    if err := client.Schema.Create(ctx, schema.WithDiffHook(renameColumnHook)); err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}

func renameColumnHook(next schema.Differ) schema.Differ {
    return schema.DiffFunc(func(current, desired *atlas.Schema) ([]atlas.Change, error) {
        changes, err := next.Diff(current, desired)
        if err != nil {
            return nil, err
        }
        for _, c := range changes {
            m, ok := c.(*atlas.ModifyTable)
            // Skip if the change is not a ModifyTable,
            // or if the table is not the "users" table.
            if !ok || m.T.Name != user.Table {
                continue
            }
            changes := atlas.Changes(m.Changes)
            switch i, j := changes.IndexDropColumn("old_name"), changes.IndexAddColumn("new_name"); {
            case i != -1 && j != -1:
                // Append a new renaming change.
                changes = append(changes, &atlas.RenameColumn{
                    From: changes[i].(*atlas.DropColumn).C,
                    To: changes[j].(*atlas.AddColumn).C,
                })
                // Remove the drop and add changes.
                changes.RemoveIndex(i, j)
                m.Changes = changes
            case i != -1 || j != -1:
                return nil, errors.New("old_name and new_name must be present or absent")
            }
        }
        return changes, nil
    })
}
```

#### `Apply` 钩子示例

`Apply` 钩子不仅能访问和修改迁移计划及其原始变更（SQL语句），还特别适用于在计划执行前后运行自定义SQL语句。例如，默认情况下不允许将可为空的列直接改为非空且无默认值的列。但我们可以通过 `Apply` 钩子先执行 `UPDATE` 语句，将该列中所有 `NULL` 值更新为非空值，从而绕过此限制：

```go
func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    // ...
    if err := client.Schema.Create(ctx, schema.WithApplyHook(fillNulls)); err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}

func fillNulls(next schema.Applier) schema.Applier {
	return schema.ApplyFunc(func(ctx context.Context, conn dialect.ExecQuerier, plan *migrate.Plan) error {
		// There are three ways to UPDATE the NULL values to "Unknown" in this stage.
		// Append a custom migrate.Change to the plan, execute an SQL statement directly
		// on the dialect.ExecQuerier, or use the ent.Client used by the project.

		// Execute a custom SQL statement.
		query, args := sql.Dialect(dialect.MySQL).
			Update(user.Table).
			Set(user.FieldDropOptional, "Unknown").
			Where(sql.IsNull(user.FieldDropOptional)).
			Query()
		if err := conn.Exec(ctx, query, args, nil); err != nil {
			return err
		}

		// Append a custom statement to migrate.Plan.
		//
		//  plan.Changes = append([]*migrate.Change{
		//	    {
		//		    Cmd: fmt.Sprintf("UPDATE users SET %[1]s = '%[2]s' WHERE %[1]s IS NULL", user.FieldDropOptional, "Unknown"),
		//	    },
		//  }, plan.Changes...)

		// Use the ent.Client used by the project.
		//
		//  drv := sql.NewDriver(dialect.MySQL, sql.Conn{ExecQuerier: conn.(*sql.Tx)})
		//  if err := ent.NewClient(ent.Driver(drv)).
		//  	User.
		//  	Update().
		//  	SetDropOptional("Unknown").
		//  	Where(/* Add predicate to filter only rows with NULL values */).
		//  	Exec(ctx); err != nil {
		//  	return fmt.Errorf("fix default values to uppercase: %w", err)
		//  }

		return next.Apply(ctx, conn, plan)
	})
}
```