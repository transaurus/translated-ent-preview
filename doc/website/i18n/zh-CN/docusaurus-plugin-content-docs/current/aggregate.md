---
id: aggregate
title: Aggregation
---

## 聚合

`Aggregate` 选项允许添加一个或多个聚合函数。

```go
package main

import (
	"context"

	"<project>/ent"
	"<project>/ent/payment"
	"<project>/ent/pet"
)

func Do(ctx context.Context, client *ent.Client) {
	// Aggregate one field.
	sum, err := client.Payment.Query().
		Aggregate(
			ent.Sum(payment.Amount),
		).
		Int(ctx)

	// Aggregate multiple fields.
	var v []struct {
		Sum, Min, Max, Count int
	}
	err := client.Pet.Query().
		Aggregate(
			ent.Sum(pet.FieldAge),
			ent.Min(pet.FieldAge),
			ent.Max(pet.FieldAge),
			ent.Count(),
		).
		Scan(ctx, &v)
}
```

## 分组查询

按所有用户的 `name` 和 `age` 字段分组，并计算其年龄总和。

```go
package main

import (
	"context"
	
	"<project>/ent"
	"<project>/ent/user"
)

func Do(ctx context.Context, client *ent.Client) {
	var v []struct {
		Name  string `json:"name"`
		Age   int    `json:"age"`
		Sum   int    `json:"sum"`
		Count int    `json:"count"`
	}
	err := client.User.Query().
		GroupBy(user.FieldName, user.FieldAge).
		Aggregate(ent.Count(), ent.Sum(user.FieldAge)).
		Scan(ctx, &v)
}
```

按单个字段分组。

```go
package main

import (
	"context"
	
	"<project>/ent"
	"<project>/ent/user"
)

func Do(ctx context.Context, client *ent.Client) {
	names, err := client.User.
		Query().
		GroupBy(user.FieldName).
		Strings(ctx)
}
```

## 按关联边分组

自定义聚合函数在需要编写特定存储逻辑时非常有用。

以下示例展示如何按所有用户的 `id` 和 `name` 分组，并计算其宠物年龄的平均值。

```go
package main

import (
	"context"
	"log"

	"<project>/ent"
	"<project>/ent/pet"
	"<project>/ent/user"
)

func Do(ctx context.Context, client *ent.Client) {
	var users []struct {
		ID      int
		Name    string
		Average float64
	}
	err := client.User.Query().
		GroupBy(user.FieldID, user.FieldName).
		Aggregate(func(s *sql.Selector) string {
			t := sql.Table(pet.Table)
			s.Join(t).On(s.C(user.FieldID), t.C(pet.OwnerColumn))
			return sql.As(sql.Avg(t.C(pet.FieldAge)), "average")
		}).
		Scan(ctx, &users)
}
```

## Having 条件与分组查询

[自定义 SQL 修饰符](https://entgo.io/docs/feature-flags/#custom-sql-modifiers) 可用于全面控制查询各部分。以下示例展示如何检索每个角色中最年长的用户。

```go
package main

import (
	"context"
	"log"

	"entgo.io/ent/dialect/sql"
	"<project>/ent"
	"<project>/ent/user"
)

func Do(ctx context.Context, client *ent.Client) {
	var users []struct {
		Id    	Int
		Age     Int
		Role    string
	}
	err := client.User.Query().
		Modify(func(s *sql.Selector) {
			s.GroupBy(user.Role)
			s.Having(
				sql.EQ(
					user.FieldAge,
					sql.Raw(sql.Max(user.FieldAge)),
				),
			)
		}).
		ScanX(ctx, &users)
}

```

**注意：** `sql.Raw` 至关重要，它向谓词表明 `sql.Max` 不是参数。

上述代码实际生成的 SQL 查询如下：

```sql
SELECT * FROM user GROUP BY user.role HAVING user.age = MAX(user.age)
```