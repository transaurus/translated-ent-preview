---
title: "Announcing v0.10: Ent gets a brand-new migration engine"
author: Ariel Mashraki
authorURL: https://github.com/a8m
authorImageURL: https://avatars0.githubusercontent.com/u/7413593
authorTwitter: arielmashraki
---

亲爱的社区成员们，

我非常高兴地宣布Ent下一个版本v0.10的发布。距离v0.9.1版本已有近六个月时间，因此这个版本自然包含了大量新功能。不过，我想特别花时间讨论我们过去几个月重点开发的一项重大改进：全新的迁移引擎。

### 隆重推出：[Atlas](https://github.com/ariga/atlas)

Ent当前的迁移引擎非常优秀，社区多年来在生产环境中使用它实现了许多巧妙的功能。但随着时间推移，现有架构无法解决的问题开始堆积。此外，我们认为现有的数据库迁移框架存在诸多不足。过去十年间，整个行业在安全管理生产系统变更方面积累了丰富经验（如基础设施即代码和声明式配置管理等原则），而这些理念在大多数迁移框架诞生时尚未形成。

鉴于这些问题具有普适性，且与应用程序使用的框架或编程语言无关，我们意识到可以将其作为通用基础设施来解决。因此，我们没有简单重写Ent的迁移引擎，而是决定将解决方案提取到一个新的开源项目——[Atlas](https://atlasgo.io)（[GitHub](https://ariga.io/atlas)）。

Atlas以CLI工具形式分发，采用基于HCL（类似Terraform）的[新DDL](https://atlasgo.io/ddl/intro)，也可作为[Go包](https://pkg.go.dev/ariga.io/atlas)使用。与Ent相同，Atlas采用[Apache License 2.0](https://github.com/ariga/atlas/blob/master/LICENSE)许可。

经过大量开发和测试，Atlas与Ent的集成终于可以投入使用。这对于提出[#1652](https://github.com/ent/ent/issues/1652)、[#1631](https://github.com/ent/ent/issues/1631)、[#1625](https://github.com/ent/ent/issues/1625)、[#1546](https://github.com/ent/ent/issues/1546)和[#1845](https://github.com/ent/ent/issues/1845)等问题的用户是个好消息——这些问题在原有迁移系统中难以妥善解决，现在通过Atlas引擎得以解决。

考虑到这是重大变更，目前Atlas迁移引擎采用自愿启用模式。未来我们将逐步转为自愿禁用模式，最终淘汰现有引擎。这个过渡过程会循序渐进，我们将根据社区反馈稳步推进。

### 开始使用Atlas迁移引擎

首先升级至Ent最新版本：

```shell
go get entgo.io/ent@v0.10.0
```

接着，使用`WithAtlas(true)`选项即可通过Atlas引擎执行迁移：

```go {17}
package main
import (
    "context"
    "log"
    "<project>/ent"
    "<project>/ent/migrate"
    "entgo.io/ent/dialect/sql/schema"
)
func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err = client.Schema.Create(ctx, schema.WithAtlas(true))
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

就这么简单！

Atlas引擎相比现有Ent代码的重大改进在于其分层架构，清晰分离了以下阶段：***检测***（理解数据库当前状态）、***差异分析***（计算当前状态与目标状态的差异）、***规划***（制定具体的差异修复方案）和***应用***。下图展示了Ent使用Atlas的工作流程：

![atlas-migration-process](https://entgo.io/images/assets/migrate-atlas-process.png)

除标准选项（如`WithDropColumn`、`WithGlobalUniqueID`）外，Atlas集成还提供了用于接入模式迁移步骤的额外选项。

以下两个示例展示了如何接入Atlas的`Diff`和`Apply`步骤：

```go
package main
import (
    "context"
    "log"
    "<project>/ent"
    "<project>/ent/migrate"
	"ariga.io/atlas/sql/migrate"
	atlas "ariga.io/atlas/sql/schema"
	"entgo.io/ent/dialect"
	"entgo.io/ent/dialect/sql/schema"
)
func main() {
    client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/test")
    if err != nil {
        log.Fatalf("failed connecting to mysql: %v", err)
    }
    defer client.Close()
    ctx := context.Background()
    // Run migration.
    err := 	client.Schema.Create(
		ctx,
		// Hook into Atlas Diff process.
		schema.WithDiffHook(func(next schema.Differ) schema.Differ {
			return schema.DiffFunc(func(current, desired *atlas.Schema) ([]atlas.Change, error) {
				// Before calculating changes.
				changes, err := next.Diff(current, desired)
				if err != nil {
					return nil, err
				}
				// After diff, you can filter
				// changes or return new ones.
				return changes, nil
			})
		}),
		// Hook into Atlas Apply process.
		schema.WithApplyHook(func(next schema.Applier) schema.Applier {
			return schema.ApplyFunc(func(ctx context.Context, conn dialect.ExecQuerier, plan *migrate.Plan) error {
				// Example to hook into the apply process, or implement
				// a custom applier. For example, write to a file.
				//
				//	for _, c := range plan.Changes {
				//		fmt.Printf("%s: %s", c.Comment, c.Cmd)
				//		if err := conn.Exec(ctx, c.Cmd, c.Args, nil); err != nil {
				//			return err
				//		}
				//	}
				//
				return next.Apply(ctx, conn, plan)
			})
		}),
	)
    if err != nil {
        log.Fatalf("failed creating schema resources: %v", err)
    }
}
```

### 下一步计划：v0.11版本

虽然本次版本发布周期较长，但下一个版本即将到来。以下是v0.11版本的主要规划：

* [支持边/关系模式](https://github.com/ent/ent/issues/1949) - 允许为关系附加元数据字段
* 重构GraphQL集成以完全兼容Relay规范，支持从Ent模式生成GraphQL资源（模式或完整服务端）
* 新增"迁移编写"功能：Atlas库提供创建"版本化"迁移目录的基础设施（类似Flyway/Liquibase/go-migrate等框架的常见做法）。许多用户已构建了与此类系统的集成方案，我们将基于Atlas为其提供更完善的基础支持
* 查询钩子（拦截器） - 当前钩子仅支持[变更操作](https://entgo.io/docs/hooks/#hooks)，大量用户要求扩展支持读取操作
* 多态边 - 关于多态支持的[议题已开放超一年](https://github.com/ent/ent/issues/1048)。随着Go 1.18泛型特性的落地，我们将重新讨论基于泛型的可能实现方案

### 总结

除了全新迁移引擎的激动消息外，本次发布在体量和内容上都堪称重磅，包含[42位贡献者提交的199个 commits](https://github.com/ent/ent/releases/tag/v0.10.0)。Ent作为社区协作的成果，正因为大家的参与而日益精进。在此向所有参与本次版本贡献的成员致以衷心感谢（按字母排序）：

[attackordie](https://github.com/attackordie),
[bbkane](https://github.com/bbkane),
[bodokaiser](https://github.com/bodokaiser),
[cjraa](https://github.com/cjraa),
[dakimura](https://github.com/dakimura),
[dependabot](https://github.com/dependabot),
[EndlessIdea](https://github.com/EndlessIdea),
[ernado](https://github.com/ernado),
[evanlurvey](https://github.com/evanlurvey),
[freb](https://github.com/freb),
[genevieve](https://github.com/genevieve),
[giautm](https://github.com/giautm),
[grevych](https://github.com/grevych),
[hedwigz](https://github.com/hedwigz),
[heliumbrain](https://github.com/heliumbrain),
[hilakashai](https://github.com/hilakashai),
[HurSungYun](https://github.com/HurSungYun),
[idc77](https://github.com/idc77),
[isoppp](https://github.com/isoppp),
[JeremyV2014](https://github.com/JeremyV2014),
[Laconty](https://github.com/Laconty),
[lenuse](https://github.com/lenuse),
[masseelch](https://github.com/masseelch),
[mattn](https://github.com/mattn),
[mookjp](https://github.com/mookjp),
[msal4](https://github.com/msal4),
[naormatania](https://github.com/naormatania),
[odeke-em](https://github.com/odeke-em),
[peanut-cc](https://github.com/peanut-cc),
[posener](https://github.com/posener),
[RiskyFeryansyahP](https://github.com/RiskyFeryansyahP),
[rotemtam](https://github.com/rotemtam),
[s-takehana](https://github.com/s-takehana),
[sadmansakib](https://github.com/sadmansakib),
[sashamelentyev](https://github.com/sashamelentyev),
[seiichi1101](https://github.com/seiichi1101),
[sivchari](https://github.com/sivchari),
[storyicon](https://github.com/storyicon),
[tarrencev](https://github.com/tarrencev),
[ThinkontrolSY](https://github.com/ThinkontrolSY),
[timoha](https://github.com/timoha),
[vecpeng](https://github.com/vecpeng),
[yonidavidson](https://github.com/yonidavidson), 以及
[zeevmoney](https://github.com/zeevmoney)。

此致，
Ariel

:::note[获取更多Ent资讯与更新：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 在[Twitter](https://twitter.com/entgo_io)上关注我们
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 加入[Ent Discord服务器](https://discord.gg/qZmPgTE6RX)

:::