---
id: faq
title: Frequently Asked Questions (FAQ)
sidebar_label: FAQ
---

## 常见问题

[如何从结构体 `T` 创建实体？](#how-to-create-an-entity-from-a-struct-t)  
[如何创建结构体（或变更）级别的验证器？](#how-to-create-a-mutation-level-validator)  
[如何编写审计日志扩展？](#how-to-write-an-audit-log-extension)  
[如何编写自定义谓词？](#how-to-write-custom-predicates)  
[如何将自定义谓词添加到代码生成资源中？](#how-to-add-custom-predicates-to-the-codegen-assets)  
[如何在 PostgreSQL 中定义网络地址字段？](#how-to-define-a-network-address-field-in-postgresql)  
[如何在 MySQL 中将时间字段自定义为 `DATETIME` 类型？](#how-to-customize-time-fields-to-type-datetime-in-mysql)  
[如何使用自定义 ID 生成器？](#how-to-use-a-custom-generator-of-ids)  
[如何使用全局唯一的自定义 XID？](#how-to-use-a-custom-xid-globally-unique-id)  
[如何在 MySQL 中定义空间数据类型字段？](#how-to-define-a-spatial-data-type-field-in-mysql)  
[如何扩展生成的模型？](#how-to-extend-the-generated-models)  
[如何扩展生成的构建器？](#how-to-extend-the-generated-builders)   
[如何在 BLOB 列中存储 Protobuf 对象？](#how-to-store-protobuf-objects-in-a-blob-column)  
[如何为表添加 `CHECK` 约束？](#how-to-add-check-constraints-to-table)  
[如何定义自定义精度的数字字段？](#how-to-define-a-custom-precision-numeric-field)  
[如何配置多个 `DB` 以分离读写？](#how-to-configure-two-or-more-db-to-separate-read-and-write)  
[如何配置 `json.Marshal` 将 `edges` 键内联到顶层对象中？](#how-to-configure-jsonmarshal-to-inline-the-edges-keys-in-the-top-level-object)

## 问题解答

#### 如何从结构体 `T` 创建实体？

不同的构建器不支持从给定结构体 `T` 设置实体字段（或边）的功能。原因在于更新数据库时无法区分零值/真实值（例如 `&ent.T{Age: 0, Name: ""}`）。设置这些值可能会在数据库中写入错误数据或更新不必要的列。

不过，[外部模板](templates.md)选项允许您通过添加自定义逻辑来扩展默认的代码生成资源。例如，要为每个创建构建器生成一个方法，该方法接受结构体作为输入并配置构建器，可使用以下模板：

```gotemplate
{{ range $n := $.Nodes }}
    {{ $builder := $n.CreateName }}
    {{ $receiver := $n.CreateReceiver }}

    func ({{ $receiver }} *{{ $builder }}) Set{{ $n.Name }}(input *{{ $n.Name }}) *{{ $builder }} {
        {{- range $f := $n.Fields }}
            {{- $setter := print "Set" $f.StructField }}
            {{ $receiver }}.{{ $setter }}(input.{{ $f.StructField }})
        {{- end }}
        return {{ $receiver }}
    }
{{ end }}
```

#### 如何创建变更级别的验证器？

要实现变更级别的验证器，您可以使用[模式钩子](hooks.md#schema-hooks)来验证应用于单一实体类型的变更，或使用[事务钩子](transactions.md#hooks)来验证应用于多个实体类型的变更（例如 GraphQL 变更）。示例如下：

```go
// A VersionHook is a dummy example for a hook that validates the "version" field
// is incremented by 1 on each update. Note that this is just a dummy example, and
// it doesn't promise consistency in the database.
func VersionHook() ent.Hook {
	type OldSetVersion interface {
		SetVersion(int)
		Version() (int, bool)
		OldVersion(context.Context) (int, error)
	}
	return func(next ent.Mutator) ent.Mutator {
		return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
			ver, ok := m.(OldSetVersion)
			if !ok {
				return next.Mutate(ctx, m)
			}
			oldV, err := ver.OldVersion(ctx)
			if err != nil {
				return nil, err
			}
			curV, exists := ver.Version()
			if !exists {
				return nil, fmt.Errorf("version field is required in update mutation")
			}
			if curV != oldV+1 {
				return nil, fmt.Errorf("version field must be incremented by 1")
			}
			// Add an SQL predicate that validates the "version" column is equal
			// to "oldV" (ensure it wasn't changed during the mutation by others).
			return next.Mutate(ctx, m)
		})
	}
}
```

#### 如何编写审计日志扩展？

编写此类扩展的首选方式是使用 [ent.Mixin](schema-mixin.md)。使用 `Fields` 选项设置所有导入混合模式的模式之间共享的字段，并使用 `Hooks` 选项为应用于这些模式的所有变更附加变更钩子。以下示例基于[仓库问题跟踪器](https://github.com/ent/ent/issues/830)中的讨论：

```go
// AuditMixin implements the ent.Mixin for sharing
// audit-log capabilities with package schemas.
type AuditMixin struct{
	mixin.Schema
}

// Fields of the AuditMixin.
func (AuditMixin) Fields() []ent.Field {
	return []ent.Field{
		field.Time("created_at").
			Immutable().
			Default(time.Now),
		field.Int("created_by").
			Optional(),
		field.Time("updated_at").
			Default(time.Now).
			UpdateDefault(time.Now),
		field.Int("updated_by").
			Optional(),
	}
}

// Hooks of the AuditMixin.
func (AuditMixin) Hooks() []ent.Hook {
	return []ent.Hook{
		hooks.AuditHook,
	}
}

// A AuditHook is an example for audit-log hook.
func AuditHook(next ent.Mutator) ent.Mutator {
	// AuditLogger wraps the methods that are shared between all mutations of
	// schemas that embed the AuditLog mixin. The variable "exists" is true, if
	// the field already exists in the mutation (e.g. was set by a different hook).
	type AuditLogger interface {
		SetCreatedAt(time.Time)
		CreatedAt() (value time.Time, exists bool)
		SetCreatedBy(int)
		CreatedBy() (id int, exists bool)
		SetUpdatedAt(time.Time)
		UpdatedAt() (value time.Time, exists bool)
		SetUpdatedBy(int)
		UpdatedBy() (id int, exists bool)
	}
	return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
		ml, ok := m.(AuditLogger)
		if !ok {
			return nil, fmt.Errorf("unexpected audit-log call from mutation type %T", m)
		}
		usr, err := viewer.UserFromContext(ctx)
		if err != nil {
			return nil, err
		}
		switch op := m.Op(); {
		case op.Is(ent.OpCreate):
			ml.SetCreatedAt(time.Now())
			if _, exists := ml.CreatedBy(); !exists {
				ml.SetCreatedBy(usr.ID)
			}
		case op.Is(ent.OpUpdateOne | ent.OpUpdate):
			ml.SetUpdatedAt(time.Now())
			if _, exists := ml.UpdatedBy(); !exists {
				ml.SetUpdatedBy(usr.ID)
			}
		}
		return next.Mutate(ctx, m)
	})
}
```

#### 如何编写自定义谓词？

用户可以在查询执行前提供自定义谓词来应用。例如：

```go
pets := client.Pet.
	Query().
	Where(predicate.Pet(func(s *sql.Selector) {
		s.Where(sql.InInts(pet.OwnerColumn, 1, 2, 3))
	})).
	AllX(ctx)

users := client.User.
	Query().
	Where(predicate.User(func(s *sql.Selector) {
		s.Where(sqljson.ValueContains(user.FieldTags, "tag"))
	})).
	AllX(ctx)
```

更多示例请参阅[谓词](predicates.md#custom-predicates)页面，或在仓库问题跟踪器中搜索更高级的示例，如[issue-842](https://github.com/ent/ent/issues/842#issuecomment-707896368)。

#### 如何将自定义谓词添加到代码生成资产中？

[模板](templates.md)选项支持扩展或覆盖默认的代码生成资产。要为[上述示例](#how-to-write-custom-predicates)生成类型安全的谓词，可按如下方式使用模板选项：

```gotemplate
{{/* A template that adds the "<F>Glob" predicate for all string fields. */}}
{{ define "where/additional/strings" }}
    {{ range $f := $.Fields }}
        {{ if $f.IsString }}
            {{ $func := print $f.StructField "Glob" }}
            // {{ $func }} applies the Glob predicate on the {{ quote $f.Name }} field.
            func {{ $func }}(pattern string) predicate.{{ $.Name }} {
                return predicate.{{ $.Name }}(func(s *sql.Selector) {
                    s.Where(sql.P(func(b *sql.Builder) {
                        b.Ident(s.C({{ $f.Constant }})).WriteString(" glob" ).Arg(pattern)
                    }))
                })
            }
        {{ end }}
    {{ end }}
{{ end }}
```

#### 如何在PostgreSQL中定义网络地址字段？

[GoType](schema-fields.mdx#go-type)和[SchemaType](schema-fields.mdx#database-type)选项允许用户定义数据库特定的字段。例如，要定义[`macaddr`](https://www.postgresql.org/docs/13/datatype-net-types.html#DATATYPE-MACADDR)字段，请使用以下配置：

```go
func (T) Fields() []ent.Field {
	return []ent.Field{
		field.String("mac").
			GoType(&MAC{}).
			SchemaType(map[string]string{
				dialect.Postgres: "macaddr",
			}).
			Validate(func(s string) error {
				_, err := net.ParseMAC(s)
				return err
			}),
	}
}

// MAC represents a physical hardware address.
type MAC struct {
	net.HardwareAddr
}

// Scan implements the Scanner interface.
func (m *MAC) Scan(value any) (err error) {
	switch v := value.(type) {
	case nil:
	case []byte:
		m.HardwareAddr, err = net.ParseMAC(string(v))
	case string:
		m.HardwareAddr, err = net.ParseMAC(v)
	default:
		err = fmt.Errorf("unexpected type %T", v)
	}
	return
}

// Value implements the driver Valuer interface.
func (m MAC) Value() (driver.Value, error) {
	return m.HardwareAddr.String(), nil
}
```

注意，如果数据库不支持`macaddr`类型（例如测试用的SQLite），该字段会回退到其原生类型（即`string`）。

`inet`示例：

```go
func (T) Fields() []ent.Field {
    return []ent.Field{
		field.String("ip").
			GoType(&Inet{}).
			SchemaType(map[string]string{
				dialect.Postgres: "inet",
			}).
			Validate(func(s string) error {
				if net.ParseIP(s) == nil {
					return fmt.Errorf("invalid value for ip %q", s)
				}
				return nil
			}),
    }
}

// Inet represents a single IP address
type Inet struct {
    net.IP
}

// Scan implements the Scanner interface
func (i *Inet) Scan(value any) (err error) {
    switch v := value.(type) {
    case nil:
    case []byte:
        if i.IP = net.ParseIP(string(v)); i.IP == nil {
            err = fmt.Errorf("invalid value for ip %q", v)
        }
    case string:
        if i.IP = net.ParseIP(v); i.IP == nil {
            err = fmt.Errorf("invalid value for ip %q", v)
        }
    default:
        err = fmt.Errorf("unexpected type %T", v)
    }
    return
}

// Value implements the driver Valuer interface
func (i Inet) Value() (driver.Value, error) {
    return i.IP.String(), nil
}
```

#### 如何在MySQL中将时间字段自定义为`DATETIME`类型？

默认情况下，`Time`字段在模式创建中使用MySQL的`TIMESTAMP`类型，该类型的范围为UTC时间'1970-01-01 00:00:01'到'2038-01-19 03:14:07'（参见[MySQL文档](https://dev.mysql.com/doc/refman/5.6/en/datetime.html)）。

要为更宽的时间范围自定义时间字段，请按如下方式使用MySQL的`DATETIME`：

```go
field.Time("birth_date").
	Optional().
	SchemaType(map[string]string{
		dialect.MySQL: "datetime",
	}),
```

#### 如何使用自定义ID生成器？

如果在数据库中使用自定义ID生成器而非自增ID（例如Twitter的[Snowflake](https://github.com/twitter-archive/snowflake/tree/snowflake-2010)），则需要编写一个自定义ID字段，在资源创建时自动调用生成器。

为此，可以根据用例选择使用`DefaultFunc`或模式钩子。如果生成器不返回错误，`DefaultFunc`更为简洁；而设置资源创建钩子则允许捕获错误。有关如何使用`DefaultFunc`的示例，请参阅[ID字段](schema-fields.mdx#id-field)部分。

以下是使用钩子与自定义生成器的示例，以[sonyflake](https://github.com/sony/sonyflake)为例。

```go
// BaseMixin to be shared will all different schemas.
type BaseMixin struct {
	mixin.Schema
}

// Fields of the Mixin.
func (BaseMixin) Fields() []ent.Field {
	return []ent.Field{
		field.Uint64("id"),
	}
}

// Hooks of the Mixin.
func (BaseMixin) Hooks() []ent.Hook {
	return []ent.Hook{
		hook.On(IDHook(), ent.OpCreate),
	}
}

func IDHook() ent.Hook {
    sf := sonyflake.NewSonyflake(sonyflake.Settings{})
	type IDSetter interface {
		SetID(uint64)
	}
	return func(next ent.Mutator) ent.Mutator {
		return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
			is, ok := m.(IDSetter)
			if !ok {
				return nil, fmt.Errorf("unexpected mutation %T", m)
			}
			id, err := sf.NextID()
			if err != nil {
				return nil, err
			}
			is.SetID(id)
			return next.Mutate(ctx, m)
		})
	}
}

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Mixin of the User.
func (User) Mixin() []ent.Mixin {
	return []ent.Mixin{
		// Embed the BaseMixin in the user schema.
		BaseMixin{},
	}
}
```

#### 如何使用自定义XID全局唯一ID？

包[xid](https://github.com/rs/xid)是一个全局唯一ID生成库，使用[Mongo Object ID](https://docs.mongodb.org/manual/reference/object-id/)算法生成12字节、20字符的ID，无需配置。xid包提供了Ent所需的[database/sql](https://pkg.go.dev/database/sql) `sql.Scanner`和`driver.Valuer`接口用于序列化。

要在任何字符串字段中存储XID，请使用[GoType](schema-fields.mdx#go-type)模式配置：

```go
// Fields of type T.
func (T) Fields() []ent.Field {
	return []ent.Field{
		field.String("id").
			GoType(xid.ID{}).
			DefaultFunc(xid.New),
	}
}
```

或作为跨多个模式可重用的[Mixin](schema-mixin.md)：

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/mixin"
	"github.com/rs/xid"
)

// BaseMixin to be shared will all different schemas.
type BaseMixin struct {
	mixin.Schema
}

// Fields of the User.
func (BaseMixin) Fields() []ent.Field {
	return []ent.Field{
		field.String("id").
			GoType(xid.ID{}).
			DefaultFunc(xid.New),
	}
}

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Mixin of the User.
func (User) Mixin() []ent.Mixin {
	return []ent.Mixin{
		// Embed the BaseMixin in the user schema.
		BaseMixin{},
	}
}
```

要在gqlgen中使用扩展标识符（XID），请按照[问题跟踪器](https://github.com/ent/ent/issues/1526#issuecomment-831034884)中提到的配置操作。

#### 如何在MySQL中定义空间数据类型字段？

通过 [GoType](schema-fields.mdx#go-type) 和 [SchemaType](schema-fields.mdx#database-type) 选项，用户可以定义数据库特定的字段类型。例如，要定义一个 [`POINT`](https://dev.mysql.com/doc/refman/8.0/en/spatial-type-overview.html) 类型的字段，可使用如下配置：

```go
// Fields of the Location.
func (Location) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
		field.Other("coords", &Point{}).
			SchemaType(Point{}.SchemaType()),
	}
}
```

```go
package schema

import (
	"database/sql/driver"
	"fmt"

	"entgo.io/ent/dialect"
	"entgo.io/ent/dialect/sql"
	"github.com/paulmach/orb"
	"github.com/paulmach/orb/encoding/wkb"
)

// A Point consists of (X,Y) or (Lat, Lon) coordinates
// and it is stored in MySQL the POINT spatial data type.
type Point [2]float64

// Scan implements the Scanner interface.
func (p *Point) Scan(value any) error {
	bin, ok := value.([]byte)
	if !ok {
		return fmt.Errorf("invalid binary value for point")
	}
	var op orb.Point
	if err := wkb.Scanner(&op).Scan(bin[4:]); err != nil {
		return err
	}
	p[0], p[1] = op.X(), op.Y()
	return nil
}

// Value implements the driver Valuer interface.
func (p Point) Value() (driver.Value, error) {
	op := orb.Point{p[0], p[1]}
	return wkb.Value(op).Value()
}

// FormatParam implements the sql.ParamFormatter interface to tell the SQL
// builder that the placeholder for a Point parameter needs to be formatted.
func (p Point) FormatParam(placeholder string, info *sql.StmtInfo) string {
	if info.Dialect == dialect.MySQL {
		return "ST_GeomFromWKB(" + placeholder + ")"
	}
	return placeholder
}

// SchemaType defines the schema-type of the Point object.
func (Point) SchemaType() map[string]string {
	return map[string]string{
		dialect.MySQL: "POINT",
	}
}
```

完整示例可参考 [示例仓库](https://github.com/a8m/entspatial)。

#### 如何扩展生成的模型？

Ent支持通过自定义模板扩展生成的类型（包括全局类型和模型）。例如，要为生成的模型添加额外结构体字段或方法，可以覆写 `model/fields/additional` 模板，具体可参考此[示例](https://github.com/ent/ent/blob/dd4792f5b30bdd2db0d9a593a977a54cb3f0c1ce/examples/entcpkg/ent/template/static.tmpl)。

若自定义字段/方法需要额外导入包，也可通过模板添加：

```gotemplate
{{- define "import/additional/field_types" -}}
    "github.com/path/to/your/custom/type"
{{- end -}}

{{- define "import/additional/client_dependencies" -}}
    "github.com/path/to/your/custom/type"
{{- end -}}
```

#### 如何扩展生成的构建器？

参阅 *[注入外部依赖](code-gen.md#external-dependencies)* 章节，或参考 [GitHub示例](https://github.com/ent/ent/tree/master/examples/entcpkg)。

#### 如何在BLOB列存储Protobuf对象？

假设已有Protobuf消息定义：

```protobuf
syntax = "proto3";

package pb;

option go_package = "project/pb";

message Hi {
  string Greeting = 1;
}
```

我们为生成的protobuf结构体添加接收方法，使其实现 [ValueScanner](https://pkg.go.dev/entgo.io/ent/schema/field#ValueScanner) 接口：

```go
func (x *Hi) Value() (driver.Value, error) {
	return proto.Marshal(x)
}

func (x *Hi) Scan(src any) error {
	if src == nil {
		return nil
	}
	if b, ok := src.([]byte); ok {
		if err := proto.Unmarshal(b, x); err != nil {
			return err
		}
		return nil
	}
	return fmt.Errorf("unexpected type %T", src)
}
```

在schema中添加 `field.Bytes` 类型字段，并将生成的protobuf结构体设为其底层 `GoType`：

```go
// Fields of the Message.
func (Message) Fields() []ent.Field {
	return []ent.Field{
		field.Bytes("hi").
			GoType(&pb.Hi{}),
	}
}
```

测试验证：

```go
package main

import (
	"context"
	"testing"

	"project/ent/enttest"
	"project/pb"

	_ "github.com/mattn/go-sqlite3"
	"github.com/stretchr/testify/require"
)

func TestMain(t *testing.T) {
	client := enttest.Open(t, "sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	defer client.Close()

	msg := client.Message.Create().
		SetHi(&pb.Hi{
			Greeting: "hello",
		}).
		SaveX(context.TODO())

	ret := client.Message.GetX(context.TODO(), msg.ID)
	require.Equal(t, "hello", ret.Hi.Greeting)
}
```

#### 如何为表添加 `CHECK` 约束？

通过 [`entsql.Annotation`](schema-annotations.md) 选项可在 `CREATE TABLE` 语句中添加自定义 `CHECK` 约束。具体配置示例如下：

```go
func (User) Annotations() []schema.Annotation {
	return []schema.Annotation{
		&entsql.Annotation{
			// The `Check` option allows adding an
			// unnamed CHECK constraint to table DDL.
			Check: "website <> 'entgo.io'",

			// The `Checks` option allows adding multiple CHECK constraints
			// to table creation. The keys are used as the constraint names.
			Checks: map[string]string{
				"valid_nickname":  "nickname <> firstname",
				"valid_firstname": "length(first_name) > 1",
			},
		},
	}
}
```

#### 如何定义自定义精度数值字段？

结合 [GoType](schema-fields.mdx#go-type) 和 [SchemaType](schema-fields.mdx#database-type) 可定义自定义精度数值字段，例如使用 [big.Int](https://pkg.go.dev/math/big) 的字段：

```go
func (T) Fields() []ent.Field {
	return []ent.Field{
		field.Int("precise").
			GoType(new(BigInt)).
			SchemaType(map[string]string{
				dialect.SQLite:   "numeric(78, 0)",
				dialect.Postgres: "numeric(78, 0)",
			}),
	}
}

type BigInt struct {
	big.Int
}

func (b *BigInt) Scan(src any) error {
	var i sql.NullString
	if err := i.Scan(src); err != nil {
		return err
	}
	if !i.Valid {
		return nil
	}
	if _, ok := b.Int.SetString(i.String, 10); ok {
		return nil
	}
	return fmt.Errorf("could not scan type %T with value %v into BigInt", src, src)
}

func (b *BigInt) Value() (driver.Value, error) {
	return b.String(), nil
}
```

#### 如何配置多个 `DB` 实现读写分离？

可通过封装 `dialect.Driver` 实现自定义驱动逻辑，例如：

可进一步扩展该方案，支持多读副本并添加负载均衡策略。

```go
func main() {
	// ...
	wd, err := sql.Open(dialect.MySQL, "root:pass@tcp(<addr>)/<database>?parseTime=True")
	if err != nil {
		log.Fatal(err)
	}
	rd, err := sql.Open(dialect.MySQL, "readonly:pass@tcp(<addr>)/<database>?parseTime=True")
	if err != nil {
		log.Fatal(err)
	}
	client := ent.NewClient(ent.Driver(&multiDriver{w: wd, r: rd}))
	defer client.Close()
	// Use the client here.
}

type multiDriver struct {
	r, w dialect.Driver
}

var _ dialect.Driver = (*multiDriver)(nil)

func (d *multiDriver) Query(ctx context.Context, query string, args, v any) error {
	e := d.r
	// Mutation statements that use the RETURNING clause.
	if ent.QueryFromContext(ctx) == nil {
		e = d.w
	}
	return e.Query(ctx, query, args, v)
}

func (d *multiDriver) Exec(ctx context.Context, query string, args, v any) error {
	return d.w.Exec(ctx, query, args, v)
}

func (d *multiDriver) Tx(ctx context.Context) (dialect.Tx, error) {
	return d.w.Tx(ctx)
}

func (d *multiDriver) BeginTx(ctx context.Context, opts *sql.TxOptions) (dialect.Tx, error) {
	return d.w.(interface {
		BeginTx(context.Context, *sql.TxOptions) (dialect.Tx, error)
	}).BeginTx(ctx, opts)
}

func (d *multiDriver) Close() error {
	rerr := d.r.Close()
	werr := d.w.Close()
	if rerr != nil {
		return rerr
	}
	if werr != nil {
		return werr
	}
	return nil
}

func (d *multiDriver) Dialect() string {
	return d.r.Dialect()
}
```

#### 如何配置 `json.Marshal` 将 `edges` 键内联到顶层对象？

实现无 `edges` 属性的实体编码需两步：

1. 移除Ent生成的默认 `edges` 标签
2. 通过自定义MarshalJSON方法扩展生成模型

可通过[代码生成扩展](extension.md)自动化实现，完整示例见 [examples/jsonencode](https://github.com/ent/ent/tree/master/examples/jsonencode) 目录。

```go title="ent/entc.go" {17,28}
//go:build ignore
// +build ignore

package main

import (
	"log"

	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
	"entgo.io/ent/schema/edge"
)

func main() {
	opts := []entc.Option{
		entc.Extensions{
			&EncodeExtension{},
		),
	}
	err := entc.Generate("./schema", &gen.Config{}, opts...)
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}

// EncodeExtension is an implementation of entc.Extension that adds a MarshalJSON
// method to each generated type <T> and inlines the Edges field to the top level JSON.
type EncodeExtension struct {
	entc.DefaultExtension
}

// Templates of the extension.
func (e *EncodeExtension) Templates() []*gen.Template {
	return []*gen.Template{
		gen.MustParse(gen.NewTemplate("model/additional/jsonencode").
			Parse(`
{{ if $.Edges }}
	// MarshalJSON implements the json.Marshaler interface.
	func ({{ $.Receiver }} *{{ $.Name }}) MarshalJSON() ([]byte, error) {
		type Alias {{ $.Name }}
		return json.Marshal(&struct {
			*Alias
			{{ $.Name }}Edges
		}{
			Alias: (*Alias)({{ $.Receiver }}),
			{{ $.Name }}Edges: {{ $.Receiver }}.Edges,
		})
	}
{{ end }}
`)),
	}
}

// Hooks of the extension.
func (e *EncodeExtension) Hooks() []gen.Hook {
	return []gen.Hook{
		func(next gen.Generator) gen.Generator {
			return gen.GenerateFunc(func(g *gen.Graph) error {
				tag := edge.Annotation{StructTag: `json:"-"`}
				for _, n := range g.Nodes {
					n.Annotations.Set(tag.Name(), tag)
				}
				return next.Generate(g)
			})
		},
	}
}
```