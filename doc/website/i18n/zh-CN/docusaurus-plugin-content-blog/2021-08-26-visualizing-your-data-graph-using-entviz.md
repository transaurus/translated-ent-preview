---
title: Visualizing your Data Graph Using entviz
author: Amit Shani
authorURL: "https://github.com/hedwigz"
authorImageURL: "https://avatars.githubusercontent.com/u/8277210?v=4"
authorTwitter: itsamitush
---

加入一个已有大型代码库的项目可能令人望而生畏。

理解应用程序的数据模型是开发者开始参与现有项目的关键。为帮助克服这一挑战，开发者常用工具之一是[ER（实体关系）图](https://en.wikipedia.org/wiki/Entity%E2%80%93relationship_model)，它能帮助开发者快速掌握应用的数据模型。

ER图以可视化形式呈现数据模型，并详细展示每个实体的字段。许多工具可辅助创建这类图表，例如Jetbrains DataGrip，它可以通过连接并检查现有数据库来生成ER图：

[Ent](https://entgo.io/docs/getting-started/)是一个简单而强大的Go语言实体框架，最初在Facebook内部开发，专门用于处理具有大型复杂数据模型的项目。
这正是Ent采用代码生成的原因——它能提供开箱即用的类型安全与代码补全，既有助于解释数据模型，又能提升开发效率。
除此之外，若能自动生成ER图，以视觉吸引力的方式维护数据模型的高层视图岂不更妙？（毕竟，谁不爱可视化呢？）

### 介绍entviz

[entviz](https://github.com/hedwigz/entviz)是一个Ent扩展，可自动生成静态HTML页面来可视化您的数据图谱。

若想了解entviz的实现原理，请查看[实现章节](#implementation)。

### 实际演示

首先，将entviz扩展添加到我们的entc.go文件中：

```bash
go get github.com/hedwigz/entviz
```

:::info
如果您不熟悉`entc`，欢迎阅读[entc文档](https://entgo.io/docs/code-gen#use-entc-as-a-package)了解更多。
:::

```go title="ent/entc.go"
import (
	"log"

	"entgo.io/ent/entc"
	"entgo.io/ent/entc/gen"
	"github.com/hedwigz/entviz"
)

func main() {
	err := entc.Generate("./schema", &gen.Config{}, entc.Extensions(entviz.Extension{}))
	if err != nil {
		log.Fatalf("running ent codegen: %v", err)
	}
}
```

假设我们有一个包含用户实体及若干字段的简单模式：

```go title="ent/schema/user.go"
// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name"),
		field.String("email"),
		field.Time("created").
			Default(time.Now),
	}
}
```

现在，每次运行以下命令时，entviz会自动生成我们的图谱可视化：

```bash
go generate ./...
```

您将在ent目录中看到一个名为`schema-viz.html`的新文件：

```bash
$ ll ./ent/schema-viz.html
-rw-r--r-- 1 hedwigz hedwigz 7.3K Aug 27 09:00 schema-viz.html
```

用您喜欢的浏览器打开该HTML文件即可查看可视化效果

![教程图片](https://entgo.io/images/assets/entviz/entviz-tutorial-1.png)

接下来，我们添加一个名为Post的实体，观察可视化如何变化：

```bash
ent new Post
```

```go title="ent/schema/post.go"
// Fields of the Post.
func (Post) Fields() []ent.Field {
	return []ent.Field{
		field.String("content"),
		field.Time("created").
			Default(time.Now),
	}
}
```

现在我们添加从User到Post的([O2M](https://entgo.io/docs/schema-edges/#o2m-two-types))边：

```go title="ent/schema/post.go"
// Edges of the User.
func (User) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("posts", Post.Type),
	}
}
```

最后重新生成代码：

```bash
go generate ./...
```

刷新浏览器即可查看更新后的结果！

![教程图片2](https://entgo.io/images/assets/entviz/entviz-tutorial-2.png)

### 实现

Entviz 是通过扩展 Ent 的[扩展 API](https://github.com/ent/ent/blob/1304dc3d795b3ea2de7101c7ca745918def668ef/entc/entc.go#L197) 实现的。  
Ent 扩展 API 允许您聚合多个[模板](https://entgo.io/docs/templates/)、[钩子](https://entgo.io/docs/hooks/)、[选项](https://entgo.io/docs/code-gen/#code-generation-options)和[注解](https://entgo.io/docs/templates/#annotations)。  
例如，entviz 使用模板添加了另一个 Go 文件 `entviz.go`，该文件公开了 `ServeEntviz` 方法，可用作 HTTP 处理器，如下所示：

```go
func main() {
	http.ListenAndServe("localhost:3002", ent.ServeEntviz())
}
```

We define an extension struct which embeds the default extension, and we export our template via the `Templates` method:

```go
//go:embed entviz.go.tmpl
var tmplfile string
 
type Extension struct {
	entc.DefaultExtension
}
 
func (Extension) Templates() []*gen.Template {
	return []*gen.Template{
		gen.MustParse(gen.NewTemplate("entviz").Parse(tmplfile)),
	}
}
```

The template file is the code that we want to generate:

```gotemplate
{{ define "entviz"}}
 
{{ $pkg := base $.Config.Package }}
{{ template "header" $ }}
import (
	_ "embed"
	"net/http"
	"strings"
	"time"
)

//go:embed schema-viz.html
var html string

func ServeEntviz() http.Handler {
	generateTime := time.Now()
	return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		http.ServeContent(w, req, "schema-viz.html", generateTime, strings.NewReader(html))
	})
}
{{ end }}
```

That's it! now we have a new method in ent package.  

### Wrapping-Up

We saw how ER diagrams help developers keep track of their data model. Next, we introduced entviz - an Ent extension that automatically generates an ER diagram for Ent schemas. We saw how entviz utilizes Ent's extension API to extend the code generation and add extra functionality. Finally, you got to see it in action by installing and use entviz in your own project. If you like the code and/or want to contribute - feel free to checkout the [project on github](https://github.com/hedwigz/entviz).

Have questions? Need help with getting started? Feel free to join our [Discord server](https://discord.gg/qZmPgTE6RX) or [Slack channel](https://entgo.io/docs/slack/).

:::note[For more Ent news and updates:]

- Subscribe to our [Newsletter](https://entgo.substack.com/)
- Follow us on [Twitter](https://twitter.com/entgo_io)
- Join us on #ent on the [Gophers Slack](https://entgo.io/docs/slack)
- Join us on the [Ent Discord Server](https://discord.gg/qZmPgTE6RX)

:::