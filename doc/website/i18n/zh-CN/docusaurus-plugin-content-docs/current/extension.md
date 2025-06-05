---
id: extensions
title: Extensions
---

### 简介

Ent的[扩展API](https://pkg.go.dev/entgo.io/ent/entc#Extension)提供了创建代码生成扩展的能力，这些扩展将[代码生成钩子](code-gen.md#code-generation-hooks)、[模板](templates.md)和[注解](templates.md#annotations)捆绑在一起，形成可复用的组件，为Ent核心功能增添新的丰富特性。例如，Ent的[entgql插件](https://pkg.go.dev/entgo.io/contrib/entgql#Extension)就提供了一个`Extension`，能够从Ent模式自动生成GraphQL服务器。

### 定义新扩展

所有扩展都必须实现[Extension](https://pkg.go.dev/entgo.io/ent/entc#Extension)接口：

```go
type Extension interface {
	// Hooks holds an optional list of Hooks to apply
	// on the graph before/after the code-generation.
	Hooks() []gen.Hook

	// Annotations injects global annotations to the gen.Config object that
	// can be accessed globally in all templates. Unlike schema annotations,
	// being serializable to JSON raw value is not mandatory.
	//
	//	{{- with $.Config.Annotations.GQL }}
	//		{{/* Annotation usage goes here. */}}
	//	{{- end }}
	//
	Annotations() []Annotation

	// Templates specifies a list of alternative templates
	// to execute or to override the default.
	Templates() []*gen.Template

	// Options specifies a list of entc.Options to evaluate on
	// the gen.Config before executing the code generation.
	Options() []Option
}
```

为了简化新扩展的开发，开发者可以嵌入[entc.DefaultExtension](https://pkg.go.dev/entgo.io/ent/entc#DefaultExtension)，这样无需实现所有方法即可创建扩展：

```go
package hello

// GreetExtension implements entc.Extension.
type GreetExtension struct {
	entc.DefaultExtension
}
```

### 添加模板

Ent支持添加[外部模板](templates.md)，这些模板将在代码生成过程中被渲染。要将此类外部模板捆绑到扩展中，需实现`Templates`方法：

```gotemplate title="templates/greet.tmpl"
{{/* Tell Intellij/GoLand to enable the autocompletion based on the *gen.Graph type. */}}
{{/* gotype: entgo.io/ent/entc/gen.Graph */}}

{{ define "greet" }}

{{/* Add the base header for the generated file */}}
{{ $pkg := base $.Config.Package }}
{{ template "header" $ }}

{{/* Loop over all nodes and add the Greet method */}}
{{ range $n := $.Nodes }}
    {{ $receiver := $n.Receiver }}
    func ({{ $receiver }} *{{ $n.Name }}) Greet() string {
		return "Hello, {{ $n.Name }}"
    }
{{ end }}

{{ end }}
```

```go
func (*GreetExtension) Templates() []*gen.Template {
	return []*gen.Template{
		gen.MustParse(gen.NewTemplate("greet").ParseFiles("templates/greet.tmpl")),
	}
}
```

### 添加全局注解

注解是一种便捷的方式，可以为扩展的用户提供API，以修改代码生成的行为。要向扩展添加注解，需实现`Annotations`方法。假设在我们的`GreetExtension`中，我们希望为用户提供配置生成代码中问候语的能力：

```go
// GreetingWord implements entc.Annotation.
type GreetingWord string

// Name of the annotation. Used by the codegen templates.
func (GreetingWord) Name() string {
	return "GreetingWord"
}
```

然后将其添加到`GreetExtension`结构体中：

```go
type GreetExtension struct {
	entc.DefaultExtension
	word GreetingWord
}
```

接下来，实现`Annotations`方法：

```go
func (s *GreetExtension) Annotations() []entc.Annotation {
	return []entc.Annotation{
		s.word,
	}
}
```

现在，你可以在模板中访问`GreetingWord`注解：

```gotemplate
func ({{ $receiver }} *{{ $n.Name }}) Greet() string {
    return "{{ $.Annotations.GreetingWord }}, {{ $n.Name }}"
}
```

### 添加钩子

entc包提供了向代码生成阶段添加一系列[钩子](code-gen.md#code-generation-hooks)（中间件）的选项。这一选项非常适合为模式添加自定义验证器，或使用图模式生成额外资源。要将代码生成钩子捆绑到扩展中，需实现`Hooks`方法：

```go
func (s *GreetExtension) Hooks() []gen.Hook {
    return []gen.Hook{
        DisallowTypeName("Shalom"),
    }
}

// DisallowTypeName ensures there is no ent.Schema with the given name in the graph.
func DisallowTypeName(name string) gen.Hook {
	return func(next gen.Generator) gen.Generator {
		return gen.GenerateFunc(func(g *gen.Graph) error {
			for _, node := range g.Nodes {
				if node.Name == name {
					return fmt.Errorf("entc: validation failed, type named %q not allowed", name)
				}
			}
			return next.Generate(g)
		})
	}
}
```

### 在代码生成中使用扩展

要在代码生成配置中使用扩展，可使用`entc.Extensions`，这是一个辅助方法，返回一个`entc.Option`，用于应用我们选择的扩展：

```go title="ent/entc.go"
//+build ignore

package main

import (
	"fmt"
	"log"

	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
)

func main() {
	err := entc.Generate("./schema",
		&gen.Config{},
		entc.Extensions(&GreetExtension{
			word: GreetingWord("Shalom"),
		}),
	)
	if err != nil {
		log.Fatal("running ent codegen:", err)
	}
}
```

### 社区扩展

- **[entoas](https://github.com/ent/contrib/tree/master/entoas)**
  `entoas` 是一个源自 `elk` 项目的扩展，现已独立成为官方 OpenAPI 规范文档生成器。通过该扩展可快速开发并生成符合 RESTful 规范的 HTTP 服务接口文档。未来将发布配套扩展，基于 `entoas` 生成的文档自动实现服务端集成。

- **[entrest](https://github.com/lrstanley/entrest)**
  `entrest` 是 `entoas`（及 `ogent`）和已停更 `elk` 的替代方案。它能从 Ent 模式生成符合标准的 OpenAPI 规范及功能完备的 RESTful API 服务实现，核心特性包括：可开关的分页功能、高级过滤/查询能力、跨关联排序、边数据预加载等。

- **[entgql](https://github.com/ent/contrib/tree/master/entgql)**  
  该扩展可将 Ent 模式转换为 [GraphQL](https://graphql.org/) 服务，深度集成流行框架 [gqlgen](https://github.com/99designs/gqlgen)。其特色功能包括自动生成类型安全的 GraphQL 过滤器，实现 GraphQL 查询到 Ent 查询的无缝映射。  
  参考[本教程](https://entgo.io/docs/tutorial-todo-gql)快速入门。

- **[entproto](https://github.com/ent/contrib/tree/master/entproto)**  
  `entproto` 能从 Ent 模式生成 Protobuf 消息定义和 gRPC 服务定义。配套工具 `protoc-gen-entgrpc` 作为 Protobuf 编译器插件，可自动生成 gRPC 服务的完整实现，开发者仅需定义 Ent 模式即可构建 gRPC 服务。  
  学习使用方式请参阅[基础教程](https://entgo.io/docs/grpc-intro)，更多背景可阅读[功能原理介绍](https://entgo.io/blog/2021/03/18/generating-a-grpc-server-withent)或[进阶特性解析](https://entgo.io/blog/2021/06/28/gprc-ready-for-use/)。

- **[elk (已停更)](https://github.com/masseelch/elk)**  
  该扩展可根据 Ent 模式生成 RESTful API 端点，包含 HTTP CRUD 处理器和 OpenAPI JSON 文件。**注意：本项目已停止维护，由 `entoas` 替代**，配套实现生成器正在开发中。  
  历史技术细节可参考[CRUD接口实现指南](https://entgo.io/blog/2021/07/29/generate-a-fully-working-go-crud-http-api-with-ent)和[OpenAPI生成原理](https://entgo.io/blog/2021/09/10/openapi-generator)。

- **[entviz (已停更)](https://github.com/hedwigz/entviz)**  
  该扩展可将 Ent 模式可视化为浏览器交互式图表，并随代码变更自动更新。通过配置可实现模式重新生成时自动刷新图表，实时呈现数据结构变化。  
  集成方法详见[可视化方案介绍](https://entgo.io/blog/2021/08/26/visualizing-your-data-graph-using-entviz)。**维护者已于2023-09-16归档此项目**。