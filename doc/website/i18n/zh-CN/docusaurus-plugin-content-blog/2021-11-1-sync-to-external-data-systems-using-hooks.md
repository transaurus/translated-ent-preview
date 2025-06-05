---
title: Sync Changes to External Data Systems using Ent Hooks
author: Ariel Mashraki
authorURL: https://github.com/a8m
authorImageURL: "https://avatars0.githubusercontent.com/u/7413593"
authorTwitter: arielmashraki
image: https://entgo.io/images/assets/sync-hook/share.png
---

Ent社区经常提出的一个常见问题是如何在Ent应用的后端数据库（如MySQL或PostgreSQL）与外部服务之间同步对象或引用。例如，用户希望在Ent中创建或删除用户记录时，从其CRM系统创建或删除对应记录；当实体更新时向[发布/订阅系统](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)发布消息；或验证对象存储（如AWS S3或Google云存储）中的blob引用。

确保两个独立数据系统之间的一致性并非易事。当我们希望将某个系统中的记录删除操作传播到另一个系统时，无法保证两个系统最终会达到同步状态，因为其中一方可能失败，或者网络连接可能延迟或中断。尽管如此，随着微服务架构的普及，这类问题愈发常见，分布式系统研究者提出了诸如[Saga模式](https://microservices.io/patterns/data/saga.html)等解决方案。

这些模式的应用通常复杂且困难，因此在许多情况下，架构师不会追求"完美"设计，而是采用更简单的解决方案——要么接受系统间存在一定程度的不一致性，要么通过后台协调程序进行处理。

本文不会讨论如何用Ent解决分布式事务或实现Saga模式，而是将范围限定在研究如何挂钩Ent的变更操作前后，并在其中运行自定义逻辑。

### 将变更传播至外部系统

在本示例中，我们将创建一个简单的`User`模式，包含两个不可变字符串字段`"name"`和`"avatar_url"`。首先运行`ent init`命令创建`User`模式骨架：

```shell
go run entgo.io/ent/cmd/ent new User
```

然后添加`name`和`avatar_url`字段，并运行`go generate`生成资源。

```go title="ent/schema/user.go"
type User struct {
	ent.Schema
}

func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").
			Immutable(),
		field.String("avatar_url").
			Immutable(),
	}
}
```

```shell
go generate ./ent
```

### 问题描述

`avatar_url`字段定义了指向对象存储桶（如AWS S3）中图像的URL。我们需要确保：

- 创建用户时，`"avatar_url"`存储的URL对应的图像必须存在于存储桶中
- 清理孤儿图像。即当用户从系统中删除时，其头像图像也应从存储桶中删除

我们将使用[`gocloud.dev/blob`](https://gocloud.dev/howto/blob)包进行blob操作。该包提供了读写、删除和列举桶中blob的抽象接口。类似于`database/sql`包，它支持通过配置驱动URL来以相同API操作多种对象存储，例如：

```go
// Open an in-memory bucket. 
if bucket, err := blob.OpenBucket(ctx, "mem://photos/"); err != nil {
	log.Fatal("failed opening in-memory bucket:", err)
}

// Open an S3 bucket named photos.
if bucket, err := blob.OpenBucket(ctx, "s3://photos"); err != nil {
	log.Fatal("failed opening s3 bucket:", err)
}

// Open a bucket named photos in Google Cloud Storage.
if bucket, err := blob.OpenBucket(ctx, "gs://my-bucket"); err != nil {
	log.Fatal("failed opening gs bucket:", err)
}
```

### 模式钩子

[钩子](https://entgo.io/docs/hooks)是Ent的强大功能，允许在图变更操作前后添加自定义逻辑。

钩子可以通过`client.Use`动态定义（称为"运行时钩子"），也可以在模式上显式定义（称为"模式钩子"）如下：

```go
// Hooks of the User.
func (User) Hooks() []ent.Hook {
	return []ent.Hook{
		EnsureImageExists(),
		DeleteOrphans(),
	}
}
```

如你所料，`EnsureImageExists`钩子负责确保创建用户时其头像URL存在于存储桶中，`DeleteOrphans`则确保删除孤儿图像。现在开始编写它们。

```go title="ent/schema/hooks.go"
func EnsureImageExists() ent.Hook {
	hk := func(next ent.Mutator) ent.Mutator {
		return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
			avatarURL, exists := m.AvatarURL()
			if !exists {
				return nil, errors.New("avatar field is missing")
			}
			// TODO:
			// 1. Verify that "avatarURL" points to a real object in the bucket.
			// 2. Otherwise, fail.
			return next.Mutate(ctx, m)
		})
	}
	// Limit the hook only to "Create" operations.
	return hook.On(hk, ent.OpCreate)
}

func DeleteOrphans() ent.Hook {
	hk := func(next ent.Mutator) ent.Mutator {
		return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
			id, exists := m.ID()
			if !exists {
				return nil, errors.New("id field is missing")
			}
			// TODO:
			// 1. Get the AvatarURL field of the deleted user.
			// 2. Cascade the deletion to object storage.
			return next.Mutate(ctx, m)
		})
	}
	// Limit the hook only to "DeleteOne" operations.
	return hook.On(hk, ent.OpDeleteOne)
}
```

此时你可能会问：_如何在变更钩子中访问blob客户端？_下一节将揭晓答案。

### 依赖注入

[entc.Dependency](https://entgo.io/docs/code-gen/#external-dependencies) 选项允许通过结构体字段的形式向生成的构建器注入外部依赖，并提供在客户端初始化时配置依赖项的机制。

为了在钩子函数中访问 `blob.Bucket`，我们可以参照[官网教程](https://entgo.io/docs/code-gen/#external-dependencies)中关于外部依赖的说明，将[`gocloud.dev/blob.Bucket`](https://pkg.go.dev/gocloud.dev/blob#Bucket)定义为依赖项。

```go title="ent/entc.go" {3-6}
func main() {
	opts := []entc.Option{
		entc.Dependency(
			entc.DependencyName("Bucket"),
			entc.DependencyType(&blob.Bucket{}),
		),
	}
	if err := entc.Generate("./schema", &gen.Config{}, opts...); err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

接着重新运行代码生成：

```shell
go generate ./ent
```

现在我们可以通过所有生成的构建器访问Bucket API。接下来完成上述钩子函数的实现。

```go title="ent/schema/hooks.go"
// EnsureImageExists ensures the avatar_url points
// to a real object in the bucket.
func EnsureImageExists() ent.Hook {
	hk := func(next ent.Mutator) ent.Mutator {
		return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
			avatarURL, exists := m.AvatarURL()
			if !exists {
				return nil, errors.New("avatar field is missing")
			}
			switch exists, err := m.Bucket.Exists(ctx, avatarURL); {
			case err != nil:
				return nil, fmt.Errorf("check key existence: %w", err)
			case !exists:
				return nil, fmt.Errorf("key %q does not exist in the bucket", avatarURL)
			default:
				return next.Mutate(ctx, m)
			}
		})
	}
	return hook.On(hk, ent.OpCreate)
}

// DeleteOrphans cascades the user deletion to the bucket.
// Hence, when a user is deleted, its avatar image is deleted
// as well.
func DeleteOrphans() ent.Hook {
	hk := func(next ent.Mutator) ent.Mutator {
		return hook.UserFunc(func(ctx context.Context, m *ent.UserMutation) (ent.Value, error) {
			id, exists := m.ID()
			if !exists {
				return nil, errors.New("id field is missing")
			}
			u, err := m.Client().User.Get(ctx, id)
			if err != nil {
				return nil, fmt.Errorf("getting deleted user: %w", err)
			}
			if err := m.Bucket.Delete(ctx, u.AvatarURL); err != nil {
				return nil, fmt.Errorf("deleting user avatar from bucket: %w", err)
			}
			return next.Mutate(ctx, m)
		})
	}
	return hook.On(hk, ent.OpDeleteOne)
}
```

现在来测试我们的钩子函数！编写一个可测试的示例来验证这两个钩子是否符合预期行为。

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/a8m/ent-sync-example/ent"
	_ "github.com/a8m/ent-sync-example/ent/runtime"

	"entgo.io/ent/dialect"
	_ "github.com/mattn/go-sqlite3"
	"gocloud.dev/blob"
	_ "gocloud.dev/blob/memblob"
)

func Example_SyncCreate() {
	ctx := context.Background()
	// Open an in-memory bucket.
	bucket, err := blob.OpenBucket(ctx, "mem://photos/")
	if err != nil {
		log.Fatal("failed opening bucket:", err)
	}
	client, err := ent.Open(
		dialect.SQLite,
		"file:ent?mode=memory&cache=shared&_fk=1",
		// Inject the blob.Bucket on client initialization.
		ent.Bucket(bucket),
	)
	if err != nil {
		log.Fatal("failed opening connection to sqlite:", err)
	}
	defer client.Close()
	if err := client.Schema.Create(ctx); err != nil {
		log.Fatal("failed creating schema resources:", err)
	}
	if err := client.User.Create().SetName("a8m").SetAvatarURL("a8m.png").Exec(ctx); err == nil {
		log.Fatal("expect user creation to fail because the image does not exist in the bucket")
	}
	if err := bucket.WriteAll(ctx, "a8m.png", []byte{255, 255, 255}, nil); err != nil {
		log.Fatalf("failed uploading image to the bucket: %v", err)
	}
	fmt.Printf("%q\n", keys(ctx, bucket))

	// User creation should pass as image was uploaded to the bucket.
	u := client.User.Create().SetName("a8m").SetAvatarURL("a8m.png").SaveX(ctx)

	// Deleting a user, should delete also its image from the bucket.
	client.User.DeleteOne(u).ExecX(ctx)
	fmt.Printf("%q\n", keys(ctx, bucket))

	// Output:
	// ["a8m.png"]
	// []
}
```

### 总结

太棒了！我们已成功配置Ent，通过[外部依赖注入](https://entgo.io/docs/code-gen#external-dependencies)机制将`blob.Bucket`集成到生成代码中。随后定义了两个变更钩子，并利用`blob.Bucket` API确保满足业务约束条件。

完整示例代码已托管在[github.com/a8m/ent-sync-example](https://github.com/a8m/ent-sync-example)。

:::note[获取更多Ent资讯：]

- 订阅[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter账号](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord社区](https://discord.gg/qZmPgTE6RX)

:::