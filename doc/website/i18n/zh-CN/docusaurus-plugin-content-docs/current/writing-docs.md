---
id: writing-docs
title: Writing Docs
---

本文档包含为Ent文档网站贡献修改的指南。

Ent文档网站由项目主仓库[GitHub repo](https://github.com/ent/ent)生成。

请遵循以下简短指南贡献文档改进与新增内容：

### 环境配置

1\. [Fork并本地克隆](https://docs.github.com/en/github/getting-started-with-github/quickstart/fork-a-repo) 
[主仓库](https://github.com/ent/ent)。

2\. 文档站点使用[Docusaurus](https://docusaurus.io/)构建，运行需先安装[Node.js](https://nodejs.org/en/)。

3\. 安装依赖项：

```shell
cd doc/website && npm install
```

4\. 以开发模式运行网站：

```shell
cd doc/website && npm start
```

5\. 在浏览器中访问 [http://localhost:3000](http://localhost:3000)。

### 通用规范

* 文档文件存放于`doc/md`目录，采用[Markdown格式](https://en.wikipedia.org/wiki/Markdown)
  并包含顶部"front-matter"样式注解。详细了解Docusaurus的[文档格式](https://docusaurus.io/docs/docs-introduction)。
* Ent采用[Golang提交信息](https://github.com/golang/go/wiki/CommitMessage)格式保持仓库历史清晰可读。
  提交时请使用如下格式的提交信息：

```text
doc/md: adding a guide on contribution of docs to ent
```

### 添加新文档

1\. 在`doc/md`目录下新建Markdown文件，例如`doc/md/writing-docs.md`。

2\. 文件应按如下格式编写：

```markdown
---
id: writing-docs
title: Writing Docs
---
...
```

其中`id`应为文档唯一标识符，需与文件名（不含`.md`后缀）保持一致；`title`将作为页面标题显示于网站导航元素中。

3\. 若需页面显示在文档网站侧边栏，将其`id`添加至`website/sidebars.js`，例如：

```diff
{
      type: 'category',
      label: 'Misc',
      items: [
        'templates',
        'graphql',
        'sql-integration',
        'testing',
        'faq',
        'generating-ent-schemas',
        'feature-flags',
        'translations',
        'contributors',
+       'writing-docs',
        'slack'
      ],
      collapsed: false,
    },
```