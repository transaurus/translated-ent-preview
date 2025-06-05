---
id: transactions
title: Transactions
---

## 开启事务

```go
// GenTx generates group of entities in a transaction.
func GenTx(ctx context.Context, client *ent.Client) error {
	tx, err := client.Tx(ctx)
	if err != nil {
		return fmt.Errorf("starting a transaction: %w", err)
	}
	hub, err := tx.Group.
		Create().
		SetName("Github").
		Save(ctx)
	if err != nil {
		return rollback(tx, fmt.Errorf("failed creating the group: %w", err))
	}
	// Create the admin of the group.
	dan, err := tx.User.
		Create().
		SetAge(29).
		SetName("Dan").
		AddManage(hub).
		Save(ctx)
	if err != nil {
		return rollback(tx, err)
	}
	// Create user "Ariel".
	a8m, err := tx.User.
		Create().
		SetAge(30).
		SetName("Ariel").
		AddGroups(hub).
		AddFriends(dan).
		Save(ctx)
	if err != nil {
		return rollback(tx, err)
	}
	fmt.Println(a8m)
	// Output:
	// User(id=2, age=30, name=Ariel)
	
	// Commit the transaction.
	return tx.Commit()
}

// rollback calls to tx.Rollback and wraps the given error
// with the rollback error if occurred.
func rollback(tx *ent.Tx, err error) error {
	if rerr := tx.Rollback(); rerr != nil {
		err = fmt.Errorf("%w: %v", err, rerr)
	}
	return err
}
```

若需在事务成功后查询已创建实体的关联边（例如 `a8m.QueryGroups()`），必须调用 `Unwrap()` 方法。该方法会将实体内部封装的底层客户端状态恢复为非事务模式。

:::warning[注意]
对非事务性实体（即事务已提交或回滚后）调用 `Unwrap()` 将引发 panic。
:::

完整示例参见 [GitHub](https://github.com/ent/ent/tree/master/examples/traversal)。

## 事务型客户端

当现有代码已基于 `*ent.Client` 实现，且需要改造（或封装）为支持事务交互时，可使用事务型客户端。该客户端可通过现有事务获取，本质上仍是 `*ent.Client`。

```go
// WrapGen wraps the existing "Gen" function in a transaction.
func WrapGen(ctx context.Context, client *ent.Client) error {
	tx, err := client.Tx(ctx)
	if err != nil {
		return err
	}
	txClient := tx.Client()
	// Use the "Gen" below, but give it the transactional client; no code changes to "Gen".
	if err := Gen(ctx, txClient); err != nil {
		return rollback(tx, err)
	}
	return tx.Commit()
}

// Gen generates a group of entities.
func Gen(ctx context.Context, client *ent.Client) error {
	// ...
	return nil
}
```

完整示例参见 [GitHub](https://github.com/ent/ent/tree/master/examples/traversal)。

## 最佳实践

封装可复用的事务回调函数：

```go
func WithTx(ctx context.Context, client *ent.Client, fn func(tx *ent.Tx) error) error {
	tx, err := client.Tx(ctx)
	if err != nil {
		return err
	}
	defer func() {
		if v := recover(); v != nil {
			tx.Rollback()
			panic(v)
		}
	}()
	if err := fn(tx); err != nil {
		if rerr := tx.Rollback(); rerr != nil {
			err = fmt.Errorf("%w: rolling back transaction: %v", err, rerr)
		}
		return err
	}
	if err := tx.Commit(); err != nil {
		return fmt.Errorf("committing transaction: %w", err)
	}
	return nil
}
```

调用示例：

```go
func Do(ctx context.Context, client *ent.Client) {
	// WithTx helper.
	if err := WithTx(ctx, client, func(tx *ent.Tx) error {
		return Gen(ctx, tx.Client())
	}); err != nil {
		log.Fatal(err)
	}
}
```

## 钩子

与[模式钩子](hooks.md#schema-hooks)和[运行时钩子](hooks.md#runtime-hooks)类似，可在活跃事务上注册钩子，这些钩子将在执行 `Tx.Commit` 或 `Tx.Rollback` 时触发：

```go
func Do(ctx context.Context, client *ent.Client) error {
    tx, err := client.Tx(ctx)
    if err != nil {
        return err
    }
    // Add a hook on Tx.Commit.
    tx.OnCommit(func(next ent.Committer) ent.Committer {
        return ent.CommitFunc(func(ctx context.Context, tx *ent.Tx) error {
            // Code before the actual commit.
            err := next.Commit(ctx, tx)
            // Code after the transaction was committed.
            return err
        })
    })
    // Add a hook on Tx.Rollback.
    tx.OnRollback(func(next ent.Rollbacker) ent.Rollbacker {
        return ent.RollbackFunc(func(ctx context.Context, tx *ent.Tx) error {
            // Code before the actual rollback.
            err := next.Rollback(ctx, tx)
            // Code after the transaction was rolled back.
            return err
        })
    })
    //
    // <Code goes here>
    //
    return err
}
```

## 隔离级别

部分驱动支持调整事务隔离级别。例如使用 [sql](sql-integration.md) 驱动时，可通过 `BeginTx` 方法实现。

```go
tx, err := client.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelRepeatableRead})
```