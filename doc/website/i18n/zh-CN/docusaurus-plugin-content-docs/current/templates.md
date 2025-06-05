---
id: templates
title: External Templates
---

`ent` 通过 `--template` 标志支持执行外部 [Go 模板](https://golang.org/pkg/text/template)。若模板名称已被 `ent` 预定义，则会覆盖现有模板；否则会将执行输出写入与模板同名的文件。例如：

`stringer.tmpl` - 此模板示例将被写入名为 `ent/stringer.go` 的文件。

```gotemplate
{{/* The line below tells Intellij/GoLand to enable the autocompletion based on the *gen.Graph type. */}}
{{/* gotype: entgo.io/ent/entc/gen.Graph */}}

{{ define "stringer" }}

{{/* Add the base header for the generated file */}}
{{ $pkg := base $.Config.Package }}
{{ template "header" $ }}

{{/* Loop over all nodes and implement the "GoStringer" interface */}}
{{ range $n := $.Nodes }}
	{{ $receiver := $n.Receiver }}
	func ({{ $receiver }} *{{ $n.Name }}) GoString() string {
		if {{ $receiver }} == nil {
			return fmt.Sprintf("{{ $n.Name }}(nil)")
		}
		return {{ $receiver }}.String()
	}
{{ end }}

{{ end }}
```

`debug.tmpl` - 此模板示例将被写入名为 `ent/debug.go` 的文件。

```gotemplate
{{ define "debug" }}

{{/* A template that adds the functionality for running each client <T> in debug mode */}}

{{/* Add the base header for the generated file */}}
{{ $pkg := base $.Config.Package }}
{{ template "header" $ }}

{{/* Loop over all nodes and add option the "Debug" method */}}
{{ range $n := $.Nodes }}
	{{ $client := print $n.Name "Client" }}
	func (c *{{ $client }}) Debug() *{{ $client }} {
		if c.debug {
			return c
		}
		cfg := config{driver: dialect.Debug(c.driver, c.log), log: c.log, debug: true, hooks: c.hooks}
		return &{{ $client }}{config: cfg}
	}
{{ end }}

{{ end }}
```

若要覆盖现有模板，需使用其预定义名称。例如：

```gotemplate
{{/* A template for adding additional fields to specific types. */}}
{{ define "model/fields/additional" }}
	{{- /* Add static fields to the "Card" entity. */}}
	{{- if eq $.Name "Card" }}
		// StaticField defined by templates.
		StaticField string `json:"static_field,omitempty"`
	{{- end }}
{{ end }}
```

## 辅助模板

如前所述，`ent` 会将每个模板的执行输出写入与模板同名的文件。例如，定义为 `{{ define "stringer" }}` 的模板输出将写入 `ent/stringer.go` 文件。

默认情况下，`ent` 会将所有通过 `{{ define "<name>" }}` 声明的模板写入独立文件。但有时需要定义辅助模板——这些模板不会被直接调用，而是由其他模板执行。为此，`ent` 支持两种命名格式来标识辅助模板：

1\. `{{ define "helper/.+" }}` 用于全局辅助模板。例如：

```gotemplate
{{ define "helper/foo" }}
    {{/* Logic goes here. */}}
{{ end }}

{{ define "helper/bar/baz" }}
    {{/* Logic goes here. */}}
{{ end }}
```

2\. `{{ define "<root-template>/helper/.+" }}` 用于局部辅助模板。若模板的执行输出会被写入文件，则视为"根模板"。例如：

```gotemplate
{{/* A root template that is executed on the `gen.Graph` and will be written to a file named: `ent/http.go`.*/}}
{{ define "http" }}
    {{ range $n := $.Nodes }}
        {{ template "http/helper/get" $n }}
        {{ template "http/helper/post" $n }}
    {{ end }}
{{ end }}

{{/* A helper template that is executed on `gen.Type` */}}
{{ define "http/helper/get" }}
    {{/* Logic goes here. */}}
{{ end }}

{{/* A helper template that is executed on `gen.Type` */}}
{{ define "http/helper/post" }}
    {{/* Logic goes here. */}}
{{ end }}
```

## 注解

模式注解允许将元数据附加到字段和边，并将其注入外部模板。注解必须是可序列化为 JSON 原始值的 Go 类型（如结构体、映射或切片），并实现 [Annotation](https://pkg.go.dev/entgo.io/ent/schema?tab=doc#Annotation) 接口。

以下为注解在模式和模板中的使用示例：

1\. 注解定义：

```go
package entgql

// Annotation annotates fields with metadata for templates.
type Annotation struct {
	// OrderField is the ordering field as defined in graphql schema.
	OrderField string
}

// Name implements ent.Annotation interface.
func (Annotation) Name() string {
	return "EntGQL"
}
```

2\. 在 ent/schema 中使用注解：

```go
// User schema.
type User struct {
	ent.Schema
}

// Fields of the user.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Time("creation_date").
			Annotations(entgql.Annotation{
				OrderField: "CREATED_AT",
			}),
	}
}
```

3\. 在外部模板中使用注解：

```gotemplate
{{ range $node := $.Nodes }}
	{{ range $f := $node.Fields }}
		{{/* Get the annotation by its name. See: Annotation.Name */}}
		{{ if $annotation := $f.Annotations.EntGQL }}
			{{/* Get the field from the annotation. */}}
			{{ $orderField := $annotation.OrderField }}
		{{ end }}
	{{ end }}
{{ end }}
```

## 全局注解

全局注解是一种注入到 `gen.Config` 对象中的注解类型，可在所有模板中全局访问。例如，持有配置文件信息（如 `gqlgen.yml` 或 `swagger.yml`）的注解可被所有模板访问：

1\. 注解定义：

```go
package gqlconfig

import (
	"entgo.io/ent/schema"
	"github.com/99designs/gqlgen/codegen/config"
)

// Annotation defines a custom annotation
// to be inject globally to all templates.
type Annotation struct {
    Config *config.Config
}

func (Annotation) Name() string {
    return "GQL"
}

var _ schema.Annotation = (*Annotation)(nil)
```

2\. 在 `ent/entc.go` 中使用注解：

```go
func main() {
	cfg, err := config.LoadConfig("<path to gqlgen.yml>")
	if err != nil {
		log.Fatalf("loading gqlgen config: %v", err)
	}
	opts := []entc.Option{
		entc.TemplateDir("./template"),
		entc.Annotations(gqlconfig.Annotation{Config: cfg}),
	}
	err = entc.Generate("./schema", &gen.Config{
		Templates: entgql.AllTemplates,
	}, opts...)
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

3\. 在外部模板中使用注解：

```gotemplate
{{- with $.Annotations.GQL.Config.StructTag }}
    {{/* Access the GQL configuration on *gen.Graph */}}
{{- end }}

{{ range $node := $.Nodes }}
    {{- with $node.Config.Annotations.GQL.Config.StructTag }}
        {{/* Access the GQL configuration on *gen.Type */}}
    {{- end }}
{{ end }}
```

## 示例

- 为 GraphQL 实现 `Node` API 的自定义模板 - 
[Github](https://github.com/ent/ent/blob/master/entc/integration/template/ent/template/node.tmpl)。

- 使用自定义函数执行外部模板的示例。参见 [配置](https://github.com/ent/ent/blob/master/examples/entcpkg/ent/entc.go) 及其 
[README](https://github.com/ent/ent/blob/master/examples/entcpkg) 文件。

## 文档

模板可在特定节点类型或整个模式图上执行。API文档请参阅<a target="_blank" href="https://pkg.go.dev/entgo.io/ent/entc/gen?tab=doc">GoDoc</a>。

## 自动补全

JetBrains用户可通过添加以下模板注解来启用模板中的自动补全功能：

```gotemplate
{{/* The line below tells Intellij/GoLand to enable the autocompletion based on the *gen.Graph type. */}}
{{/* gotype: entgo.io/ent/entc/gen.Graph */}}

{{ define "template" }}
    {{/* ... */}}
{{ end }}
```

实际效果如下：

![template-autocomplete](https://entgo.io/images/assets/template-autocomplete.gif)