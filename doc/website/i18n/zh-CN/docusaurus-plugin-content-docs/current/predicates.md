---
id: predicates
title: Predicates
---

## 字段谓词

- **布尔类型**:
  - =, !=
- **数值类型**:
  - =, !=, >, <, >=, <=,
  - IN, NOT IN
- **时间类型**:
  - =, !=, >, <, >=, <=
  - IN, NOT IN
- **字符串类型**:
  - =, !=, >, <, >=, <=
  - IN, NOT IN
  - Contains（包含）, HasPrefix（前缀匹配）, HasSuffix（后缀匹配）
  - ContainsFold（不区分大小写包含）, EqualFold（不区分大小写相等，**SQL**特有）
- **JSON类型**
  - =, !=
  - 对嵌套值使用=, !=, >, <, >=, <=（JSON路径）
  - 对嵌套值使用Contains（JSON路径）
  - HasKey（检查键是否存在）, Len&lt;P>（检查数组长度）
  - 对嵌套值检查`null`字面量（JSON路径）
- **可选字段**:
  - IsNil（为空）, NotNil（非空）

## 边谓词

- **HasEdge**。例如，对于名为`owner`且类型为`Pet`的边，使用：

  ```go
   client.Pet.
		Query().
		Where(pet.HasOwner()).
		All(ctx)
  ```

- **HasEdgeWith**。为边谓词添加谓词列表。

  ```go
   client.Pet.
		Query().
		Where(pet.HasOwnerWith(user.Name("a8m"))).
		All(ctx)
  ```

## 否定（NOT）

```go
client.Pet.
	Query().
	Where(pet.Not(pet.NameHasPrefix("Ari"))).
	All(ctx)
```

## 析取（OR）

```go
client.Pet.
	Query().
	Where(
		pet.Or(
			pet.HasOwner(),
			pet.Not(pet.HasFriends()),
		)
	).
	All(ctx)
```

## 合取（AND）

```go
client.Pet.
	Query().
	Where(
		pet.And(
			pet.HasOwner(),
			pet.Not(pet.HasFriends()),
		)
	).
	All(ctx)
```

## 自定义谓词

当需要编写特定于数据库方言的逻辑或控制执行的查询时，自定义谓词会非常有用。

#### 获取用户1、2和3的所有宠物

```go
pets := client.Pet.
	Query().
	Where(func(s *sql.Selector) {
		s.Where(sql.InInts(pet.FieldOwnerID, 1, 2, 3))
	}).
	AllX(ctx)
```

上述代码将生成以下SQL查询：

```sql
SELECT DISTINCT `pets`.`id`, `pets`.`owner_id` FROM `pets` WHERE `owner_id` IN (1, 2, 3)
```

#### 统计JSON字段`URL`包含`Scheme`键的用户数量

```go
count := client.User.
	Query().
	Where(func(s *sql.Selector) {
		s.Where(sqljson.HasKey(user.FieldURL, sqljson.Path("Scheme")))
	}).
	CountX(ctx)
```

上述代码将生成以下SQL查询：

```sql
-- PostgreSQL
SELECT COUNT(DISTINCT "users"."id") FROM "users" WHERE "url"->'Scheme' IS NOT NULL

-- SQLite and MySQL
SELECT COUNT(DISTINCT `users`.`id`) FROM `users` WHERE JSON_EXTRACT(`url`, "$.Scheme") IS NOT NULL
```

#### 获取所有拥有`"Tesla"`汽车的用户

考虑如下ent查询：

```go
users := client.User.Query().
	Where(user.HasCarWith(car.Model("Tesla"))).
	AllX(ctx)
```

该查询可以改写为三种形式：`IN`、`EXISTS`和`JOIN`。

```go
// `IN` version.
users := client.User.Query().
	Where(func(s *sql.Selector) {
		t := sql.Table(car.Table)
        s.Where(
            sql.In(
                s.C(user.FieldID),
                sql.Select(t.C(user.FieldID)).From(t).Where(sql.EQ(t.C(car.FieldModel), "Tesla")),
            ),
        )
	}).
	AllX(ctx)

// `JOIN` version.
users := client.User.Query().
	Where(func(s *sql.Selector) {
		t := sql.Table(car.Table)
		s.Join(t).On(s.C(user.FieldID), t.C(car.FieldOwnerID))
		s.Where(sql.EQ(t.C(car.FieldModel), "Tesla"))
	}).
	AllX(ctx)

// `EXISTS` version.
users := client.User.Query().
	Where(func(s *sql.Selector) {
		t := sql.Table(car.Table)
		p := sql.And(
            sql.EQ(t.C(car.FieldModel), "Tesla"),
			sql.ColumnsEQ(s.C(user.FieldID), t.C(car.FieldOwnerID)),
		)
		s.Where(sql.Exists(sql.Select().From(t).Where(p)))
	}).
	AllX(ctx)
```

上述代码将生成以下SQL查询：

```sql
-- `IN` version.
SELECT DISTINCT `users`.`id`, `users`.`age`, `users`.`name` FROM `users` WHERE `users`.`id` IN (SELECT `cars`.`owner_id` FROM `cars` WHERE `cars`.`model` = 'Tesla')

-- `JOIN` version.
SELECT DISTINCT `users`.`id`, `users`.`age`, `users`.`name` FROM `users` JOIN `cars` ON `users`.`id` = `cars`.`owner_id` WHERE `cars`.`model` = 'Tesla'

-- `EXISTS` version.
SELECT DISTINCT `users`.`id`, `users`.`age`, `users`.`name` FROM `users` WHERE EXISTS (SELECT * FROM `cars` WHERE `cars`.`model` = 'Tesla' AND `users`.`id` = `cars`.`owner_id`)
```

#### 获取名称包含特定模式的宠物

生成的代码提供了`HasPrefix`、`HasSuffix`、`Contains`和`ContainsFold`谓词用于模式匹配。
但若需使用自定义模式的`LIKE`运算符，请参考以下示例。

```go
pets := client.Pet.Query().
	Where(func(s *sql.Selector){
		s.Where(sql.Like(pet.Name,"_B%"))
	}).
	AllX(ctx)
```

上述代码将生成以下SQL查询：

```sql
SELECT DISTINCT `pets`.`id`, `pets`.`owner_id`, `pets`.`name`, `pets`.`age`, `pets`.`species` FROM `pets` WHERE `name` LIKE '_B%'
```

#### 自定义SQL函数

要使用内置SQL函数如`DATE()`，可选择以下方式之一：

1\. 通过`sql.P`选项传递方言感知的谓词函数：

```go
users := client.User.Query().
	Select(user.FieldID).
	Where(func(s *sql.Selector) {
		s.Where(sql.P(func(b *sql.Builder) {
			b.WriteString("DATE(").Ident("last_login_at").WriteByte(')').WriteOp(OpGTE).Arg(value)
		}))
	}).
	AllX(ctx)
```

上述代码将生成以下SQL查询：

```sql
SELECT `id` FROM `users` WHERE DATE(`last_login_at`) >= ?
```

2\. 使用`ExprP()`选项内联谓词表达式：

```go
users := client.User.Query().
	Select(user.FieldID).
	Where(func(s *sql.Selector) {
		s.Where(sql.ExprP("DATE(last_login_at) >= ?", value))
	}).
	AllX(ctx)
```

上述代码将生成相同的SQL查询：

```sql
SELECT `id` FROM `users` WHERE DATE(`last_login_at`) >= ?
```

## JSON谓词

JSON谓词默认不会作为代码生成的一部分自动生成。但ent官方提供了[`sqljson`](https://pkg.go.dev/entgo.io/ent/dialect/sql/sqljson)包，可通过[自定义谓词选项](#custom-predicates)对JSON列应用谓词操作。

#### 比较JSON值

```go
sqljson.ValueEQ(user.FieldData, data)

sqljson.ValueEQ(user.FieldURL, "https", sqljson.Path("Scheme"))

sqljson.ValueNEQ(user.FieldData, content, sqljson.DotPath("attributes[1].body.content"))

sqljson.ValueGTE(user.FieldData, status.StatusBadRequest, sqljson.Path("response", "status"))
```

#### 检查JSON键是否存在

```go
sqljson.HasKey(user.FieldData, sqljson.Path("attributes", "[1]", "body"))

sqljson.HasKey(user.FieldData, sqljson.DotPath("attributes[1].body"))
```

注意：值为`null`字面量的键也会匹配此操作。

#### 检查JSON `null`字面量

```go
sqljson.ValueIsNull(user.FieldData)

sqljson.ValueIsNull(user.FieldData, sqljson.Path("attributes"))

sqljson.ValueIsNull(user.FieldData, sqljson.DotPath("attributes[1].body"))
```

注意：`ValueIsNull`仅在值为JSON `null`时返回true，不适用于数据库`NULL`。

#### 比较JSON数组长度

```go
sqljson.LenEQ(user.FieldAttrs, 2)

sql.Or(
	sqljson.LenGT(user.FieldData, 10, sqljson.Path("attributes")),
	sqljson.LenLT(user.FieldData, 20, sqljson.Path("attributes")),
)
```

#### 检查JSON值是否包含另一值

```go
sqljson.ValueContains(user.FieldData, data)

sqljson.ValueContains(user.FieldData, attrs, sqljson.Path("attributes"))

sqljson.ValueContains(user.FieldData, code, sqljson.DotPath("attributes[0].status_code"))
```

#### 检查JSON字符串值是否包含子串/具有特定前缀或后缀

```go
sqljson.StringContains(user.FieldURL, "github", sqljson.Path("host"))

sqljson.StringHasSuffix(user.FieldURL, ".com", sqljson.Path("host"))

sqljson.StringHasPrefix(user.FieldData, "20", sqljson.DotPath("attributes[0].status_code"))
```

#### 检查JSON值是否等于列表中任一值

```go
sqljson.ValueIn(user.FieldURL, []any{"https", "ftp"}, sqljson.Path("Scheme"))

sqljson.ValueNotIn(user.FieldURL, []any{"github", "gitlab"}, sqljson.Path("Host"))
```

## 字段比较

`dialect/sql`包提供了一组比较函数，可用于查询中的字段比较。

```go
client.Order.Query().
	Where(
		sql.FieldsEQ(order.FieldTotal, order.FieldTax),
        sql.FieldsNEQ(order.FieldTotal, order.FieldDiscount),
	).
	All(ctx)

client.Order.Query().
	Where(
		order.Or(
			sql.FieldsGT(order.FieldTotal, order.FieldTax),
			sql.FieldsLT(order.FieldTotal, order.FieldDiscount),
		),
	).
	All(ctx)
```