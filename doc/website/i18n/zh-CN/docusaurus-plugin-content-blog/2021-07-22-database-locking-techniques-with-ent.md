---
title: Database Locking Techniques with Ent
author: Rotem Tamir
authorURL: "https://github.com/rotemtam"
authorImageURL: "https://s.gravatar.com/avatar/36b3739951a27d2e37251867b7d44b1a?s=80"
authorTwitter: _rtam
---

锁是任何并发计算机程序的基础构建模块之一。当多个操作同时发生时，程序员会借助锁来确保对资源的并发访问具有互斥性。锁（及其他互斥原语）存在于技术栈的各个层级——从底层的CPU指令到应用层API（如Go语言中的`sync.Mutex`）。

在使用关系型数据库时，应用开发者的常见需求之一是对记录加锁。假设有一个电商网站的`inventory`库存表，其中包含表示商品状态的`state`列（可能值为`available`可购买或`purchased`已售出）。为避免两个用户误认为同时成功购买了同一商品，应用程序必须阻止两个操作同时将商品状态从"可购买"修改为"已售出"。

如何实现这种保证？仅让服务器在更新前检查商品是否处于`available`状态是不够的。设想两个用户同时购买同一商品：两个请求会几乎同时到达应用服务器，两者查询数据库时都会看到商品状态为"可购买"，继而各自发出将状态改为"已售出"并设置`buyer_id`的`UPDATE`语句。虽然两个查询都会执行成功，但最终只有最后执行更新操作的买家会被记录为商品所有者。

多年来，开发者逐渐形成了多种技术方案来满足这类需求。有些方案依赖数据库提供的显式锁机制，有些则利用数据库ACID特性实现互斥。本文将探讨如何通过Ent框架实现其中两种技术方案。

### 乐观锁（Optimistic Locking）

乐观锁（有时也称乐观并发控制）是一种无需显式加锁即可实现锁定行为的技术。

其核心工作原理如下：

- 每条记录分配一个数值版本号（必须单调递增，通常使用最后更新的Unix时间戳）
- 事务读取记录时获取当前版本号
- 执行更新操作时：
    - UPDATE语句必须包含版本号未变更的条件（如`WHERE id=<id> AND version=<previous version>`）
    - 必须递增版本号（可通过+1或设置为当前时间戳实现）
- 数据库返回受影响行数。若为0，表示读取记录后、更新前该记录已被修改，此时事务应回滚并重试

乐观锁通常适用于"低竞争"环境（即事务冲突概率较低），且要求所有写操作都必须遵循版本控制逻辑。如果存在不遵守此逻辑的写入方，该技术将失效。

下面演示如何通过Ent实现乐观锁：

首先定义包含`online`布尔字段（用户在线状态）和`int64`版本号字段的`User`实体模式：

```go
// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Bool("online"),
		field.Int64("version").
			DefaultFunc(func() int64 {
				return time.Now().UnixNano()
			}).
			Comment("Unix time of when the latest update occurred")
	}
}
```

接着实现基于乐观锁的`online`字段更新逻辑：

```go
func optimisticUpdate(tx *ent.Tx, prev *ent.User, online bool) error {
	// The next version number for the record must monotonically increase
	// using the current timestamp is a common technique to achieve this. 
	nextVer := time.Now().UnixNano()

	// We begin the update operation:
	n := tx.User.Update().

		// We limit our update to only work on the correct record and version:
		Where(user.ID(prev.ID), user.Version(prev.Version)).

		// We set the next version:
		SetVersion(nextVer).

		// We set the value we were passed by the user:
		SetOnline(online).
		SaveX(context.Background())

	// SaveX returns the number of affected records. If this value is 
	// different from 1 the record must have been changed by another
	// process.
	if n != 1 {
		return fmt.Errorf("update failed: user id=%d updated by another process", prev.ID)
	}
	return nil
}
```

接下来，我们编写一个测试用例来验证当两个进程尝试编辑同一条记录时，只有其中一个会成功：

```go
func TestOCC(t *testing.T) {
	client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	ctx := context.Background()

	// Create the user for the first time.
	orig := client.User.Create().SetOnline(true).SaveX(ctx)

	// Read another copy of the same user.
	userCopy := client.User.GetX(ctx, orig.ID)

	// Open a new transaction:
	tx, err := client.Tx(ctx)
	if err != nil {
		log.Fatalf("failed creating transaction: %v", err)
	}

	// Try to update the record once. This should succeed.
	if err := optimisticUpdate(tx, userCopy, false); err != nil {
		tx.Rollback()
		log.Fatal("unexpected failure:", err)
	}

	// Try to update the record a second time. This should fail.
	err = optimisticUpdate(tx, orig, false)
	if err == nil {
		log.Fatal("expected second update to fail")
	}
	fmt.Println(err)
}
```

运行测试：

```go
=== RUN   TestOCC
update failed: user id=1 updated by another process
--- PASS: Test (0.00s)
```

完美！通过乐观锁机制，我们可以避免两个进程相互干扰！

### 悲观锁

如前所述，乐观锁并非适用于所有场景。对于更倾向于将锁完整性维护职责委托给数据库的用例，部分数据库引擎（如MySQL、PostgreSQL和MariaDB，但SQLite不支持）提供了悲观锁功能。这些数据库支持在`SELECT`语句中添加`SELECT ... FOR UPDATE`修饰符。MySQL官方文档如此[解释](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-reads.html)：

> `SELECT ... FOR UPDATE`会读取最新的可用数据，并对读取的每行记录设置排他锁。其锁机制与SQL更新语句对行记录的锁定方式相同。

用户也可以使用`SELECT ... FOR SHARE`语句，文档中说明该语句：

> 对读取的所有行设置共享模式锁。其他会话可以读取这些行，但在您的事务提交前无法修改。如果这些行被其他尚未提交的事务修改，您的查询将等待该事务结束后使用最新值。

Ent近期通过名为`sql/lock`的特性标志增加了对`FOR SHARE`/`FOR UPDATE`语句的支持。使用时需在`generate.go`文件中添加`--feature sql/lock`参数：

```go
//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate --feature sql/lock ./schema 
```

下面我们实现一个使用悲观锁确保单进程独占更新`User`对象`online`字段的函数：

```go
func pessimisticUpdate(tx *ent.Tx, id int, online bool) (*ent.User, error) {
	ctx := context.Background()

	// On our active transaction, we begin a query against the user table
	u, err := tx.User.Query().

		// We add a predicate limiting the lock to the user we want to update.
		Where(user.ID(id)).

		// We use the ForUpdate method to tell ent to ask our DB to lock
		// the returned records for update.
		ForUpdate(
			// We specify that the query should not wait for the lock to be
			// released and instead fail immediately if the record is locked.
			sql.WithLockAction(sql.NoWait),
		).
		Only(ctx)
	
	// If we failed to acquire the lock we do not proceed to update the record.
	if err != nil {
		return nil, err
	}
	
	// Finally, we set the online field to the desired value. 
	return u.Update().SetOnline(online).Save(ctx)
}
```

现在编写测试用例验证当两个进程尝试编辑同一条记录时的独占性：

```go
func TestPessimistic(t *testing.T) {
	ctx := context.Background()
	client := enttest.Open(t, dialect.MySQL, "root:pass@tcp(localhost:3306)/test?parseTime=True")

	// Create the user for the first time.
	orig := client.User.Create().SetOnline(true).SaveX(ctx)

	// Open a new transaction. This transaction will acquire the lock on our user record.
	tx, err := client.Tx(ctx)
	if err != nil {
		log.Fatalf("failed creating transaction: %v", err)
	}
	defer tx.Commit()
	
	// Open a second transaction. This transaction is expected to fail at 
	// acquiring the lock on our user record. 
	tx2, err := client.Tx(ctx)
	if err != nil {
		log.Fatalf("failed creating transaction: %v", err)
	}
	defer tx.Commit()
	
	// The first update is expected to succeed.
	if _, err := pessimisticUpdate(tx, orig.ID, true); err != nil {
		log.Fatalf("unexpected error: %s", err)
	}
	
	// Because we did not run tx.Commit yet, the row is still locked when
	// we try to update it a second time. This operation is expected to 
	// fail. 
	_, err = pessimisticUpdate(tx2, orig.ID, true)
	if err == nil {
		log.Fatal("expected second update to fail")
	}
	fmt.Println(err)
}
```

此示例中有几点值得注意：

- 由于SQLite不支持`SELECT .. FOR UPDATE`，我们使用真实MySQL实例运行测试
- 为简化示例，采用`sql.NoWait`选项使数据库在无法获取锁时立即返回错误。这意味着调用应用需在收到错误后重试写入操作。若不指定该选项，可实现应用阻塞至锁释放后继续执行的流程（虽不总是理想方案，但提供了有趣的设计可能性）
- 必须显式提交事务。忘记提交可能导致严重问题——只要锁保持，其他操作都无法读取或更新该记录

运行测试：

```go
=== RUN   TestPessimistic
Error 3572: Statement aborted because lock(s) could not be acquired immediately and NOWAIT is set.
--- PASS: TestPessimistic (0.08s)
```

成功！我们利用MySQL的"锁定读取"功能及Ent的新特性支持，实现了具有真正互斥保证的锁机制。

### 结论

本文开篇阐述了促使开发者使用数据库锁技术的业务需求场景，继而介绍了实现记录更新互斥的两种不同方案，并演示了如何通过Ent应用这些技术。

如有疑问或需要入门帮助？欢迎加入我们的[Discord服务器](https://discord.gg/qZmPgTE6RX)或[Slack频道](https://entgo.io/docs/slack)。

:::note[获取更多 Ent 资讯与更新：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 在[Twitter](https://twitter.com/entgo_io)上关注我们
- 加入 [Gophers Slack](https://entgo.io/docs/slack) 的 #ent 频道
- 加入 [Ent Discord 服务器](https://discord.gg/qZmPgTE6RX)

:::