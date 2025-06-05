---
id: translations
title: Translations
---

## 简介

为了让Ent更易于其他母语者使用，我们启动了翻译计划。我们的目标是将本网站所有文档提供中文和日文版本。为简化翻译、审核和部署流程，我们已将本网站与[Crowdin](https://crowdin.com)集成。

## 参与贡献

1. 在我们的[Crowdin项目](https://crwd.in/ent)注册成为译者
2. 完成注册登录后，访问[项目主页](https://crowdin.com/project/ent)
3. 选择您要翻译的目标语言
4. 选择待翻译文档：
   * 所有文档位于`md/`目录下
   * 博客文章位于`website/blog`
   * 网站UI组件位于`website/i18n`
5. 有关Crowdin编辑器的使用说明，请阅读[在线编辑器文档](https://support.crowdin.com/online-editor/)
6. 您的翻译建议将由校对人员审核，通过后将随下次网站部署更新

## 申请成为校对员

若希望长期参与并担任某语言的校对工作，请通过Slack频道联系我们。

## 重要准则

- 多数文档首行为`id: <文档ID>`——该行绝对不可翻译，否则将导致网站构建失败
- 网站构建对HTML标签闭合敏感，若翻译需使用HTML标签，必须确保闭合或使用自闭合标签（例如勿使用未闭合的`<br>`，应使用`<br/>`）