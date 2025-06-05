---
title: Building Observable Ent Applications with Prometheus
author: Yoni Davidson
authorURL: "https://github.com/yonidavidson"
authorImageURL: "https://avatars0.githubusercontent.com/u/5472778"
authorTwitter: yonidavidson
---

可观测性是指系统内部状态能够被外部监测的程度。当计算机程序演变为成熟的生产系统时，这一特性变得尤为重要。提升软件系统可观测性的方法之一是导出指标，即通过某种外部可见的方式报告运行系统状态的量化描述。例如，暴露一个HTTP端点来展示进程启动以来发生的错误次数。本文将探讨如何利用Prometheus构建更具可观测性的Ent应用。

### 什么是Ent？

[Ent](https://entgo.io/docs/getting-started/)是一个简洁而强大的Go语言实体框架，可轻松构建和维护具有大型数据模型的应用。

### 什么是Prometheus？

[Prometheus](https://prometheus.io/)是由SoundCloud工程师于2012年开发的开源监控系统，包含嵌入式时间序列数据库和多种第三方系统集成。Prometheus客户端通过HTTP端点（通常为`/metrics`）暴露进程指标，该端点由Prometheus采集器发现并按固定间隔（通常30秒）轮询，将数据写入时间序列数据库。

Prometheus只是指标收集后端的一个示例。业界广泛使用的同类系统还包括AWS CloudWatch、InfluxDB等。文章最后我们将讨论与这类后端统一标准集成的可能路径。

### 使用Prometheus

要通过Prometheus暴露应用指标，需要创建Prometheus[采集器](https://prometheus.io/docs/introduction/glossary/#collector)，该组件负责从服务器收集一组指标。

本示例将使用Prometheus支持的[两种指标类型](https://prometheus.io/docs/concepts/metric_types/)：计数器和直方图。计数器是单调递增的累积指标，用于记录事件发生次数（如服务器处理的请求数或错误发生次数）。直方图将观测值采样到可配置大小的桶中，通常用于表示延迟分布（例如5ms、10ms、100ms、1s内返回的请求数）。此外，Prometheus允许通过标签细分指标，例如按端点名称分类统计请求数。

我们通过[官方Go客户端](https://github.com/prometheus/client_golang)创建采集器，使用其中的[promauto](https://pkg.go.dev/github.com/prometheus/client_golang@v1.11.0/prometheus/promauto)包简化采集器创建过程。以下是统计请求总数或错误数的采集器简单示例：

```go
package example

import (
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

var (
	// List of dynamic labels
	labelNames = []string{"endpoint", "error_code"}

	// Create a counter collector
	exampleCollector = promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "endpoint_errors",
			Help: "Number of errors in endpoints",
		},
		labelNames,
	)
)

// When using you set the values of the dynamic labels and then increment the counter
func incrementError() {
	exampleCollector.WithLabelValues("/create-user", "400").Inc()
}
```

### Ent钩子

[钩子](https://entgo.io/docs/hooks)是Ent的特性，允许在数据实体变更操作前后添加自定义逻辑。

变更操作是指会修改数据库内容的操作，共有五种类型：

1. 创建(Create)
2. 单条更新(UpdateOne)
3. 批量更新(Update)
4. 单条删除(DeleteOne)
5. 批量删除(Delete)

钩子函数接收[ent.Mutator](https://pkg.go.dev/entgo.io/ent#Mutator)并返回新的mutator，其工作原理类似于常见的[HTTP中间件模式](https://github.com/go-chi/chi#middleware-handlers)。

```go
package example

import (
	"context"

	"entgo.io/ent"
)

func exampleHook() ent.Hook {
	//use this to init your hook
	return func(next ent.Mutator) ent.Mutator {
		return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
			// Do something before mutation.
			v, err := next.Mutate(ctx, m)
			if err != nil {
				// Do something if error after mutation.
			}
			// Do something after mutation.
			return v, err
		})
	}
}
```

在Ent中，存在两种类型的变更钩子——模式钩子（schema hooks）和运行时钩子（runtime hooks）。模式钩子主要用于在特定实体类型上定义自定义变更逻辑，例如将实体创建同步到其他系统。而运行时钩子则用于定义更全局的逻辑，例如添加日志记录、指标收集、追踪等功能。

对于我们的使用场景，显然应该采用运行时钩子，因为为了确保价值，我们需要对所有实体类型的所有操作都导出指标：

```go
package example

import (
	"entprom/ent"
	"entprom/ent/hook"
)

func main() {
	client, _ := ent.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")

	// Add a hook only on user mutations.
	client.User.Use(exampleHook())

	// Add a hook only on update operations.
	client.Use(hook.On(exampleHook(), ent.OpUpdate|ent.OpUpdateOne))
}
```

### 为Ent应用导出Prometheus指标

完成所有背景介绍后，让我们直切主题，展示如何结合使用Prometheus和Ent钩子来构建可观测性应用。本示例的目标是通过钩子导出以下指标：

| Metric Name                    | Description                              |
|--------------------------------|------------------------------------------|
| ent_operation_total            | Number of ent mutation operations        |
| ent_operation_error            | Number of failed ent mutation operations |
| ent_operation_duration_seconds | Time in seconds per operation            |

每个指标将通过标签细分为两个维度：

* `mutation_type`：被变更的实体类型（User、BlogPost、Account等）
* `mutation_op`：执行的操作类型（Create、Delete等）

首先定义我们的收集器：

```go
//Ent dynamic dimensions
const (
	mutationType = "mutation_type"
	mutationOp   = "mutation_op"
)

var entLabels = []string{mutationType, mutationOp}

// Create a collector for total operations counter
func initOpsProcessedTotal() *prometheus.CounterVec {
	return promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "ent_operation_total",
			Help: "Number of ent mutation operations",
		},
		entLabels,
	)
}

// Create a collector for error counter
func initOpsProcessedError() *prometheus.CounterVec {
	return promauto.NewCounterVec(
		prometheus.CounterOpts{
			Name: "ent_operation_error",
			Help: "Number of failed ent mutation operations",
		},
		entLabels,
	)
}

// Create a collector for duration histogram collector
func initOpsDuration() *prometheus.HistogramVec {
	return promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name: "ent_operation_duration_seconds",
			Help: "Time in seconds per operation",
		},
		entLabels,
	)
}
```

接着定义新的钩子：

```go
// Hook init collectors, count total at beginning error on mutation error and duration also after.
func Hook() ent.Hook {
	opsProcessedTotal := initOpsProcessedTotal()
	opsProcessedError := initOpsProcessedError()
	opsDuration := initOpsDuration()
	return func(next ent.Mutator) ent.Mutator {
		return ent.MutateFunc(func(ctx context.Context, m ent.Mutation) (ent.Value, error) {
			// Before mutation, start measuring time.
			start := time.Now()
			// Extract dynamic labels from mutation.
			labels := prometheus.Labels{mutationType: m.Type(), mutationOp: m.Op().String()}
			// Increment total ops counter.
			opsProcessedTotal.With(labels).Inc()
			// Execute mutation.
			v, err := next.Mutate(ctx, m)
			if err != nil {
				// In case of error increment error counter.
				opsProcessedError.With(labels).Inc()
			}
			// Stop time measure.
			duration := time.Since(start)
			// Record duration in seconds.
			opsDuration.With(labels).Observe(duration.Seconds())
			return v, err
		})
	}
}
```

### 将Prometheus收集器接入服务

定义完钩子后，接下来展示如何将其接入应用，并使用Prometheus提供暴露收集器指标的端点：

```go
package main

import (
	"context"
	"log"
	"net/http"

	"entprom"
	"entprom/ent"

	_ "github.com/mattn/go-sqlite3"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

func createClient() *ent.Client {
	c, err := ent.Open("sqlite3", "file:ent?mode=memory&cache=shared&_fk=1")
	if err != nil {
		log.Fatalf("failed opening connection to sqlite: %v", err)
	}
	ctx := context.Background()
	// Run the auto migration tool.
	if err := c.Schema.Create(ctx); err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
	return c
}

func handler(client *ent.Client) func(w http.ResponseWriter, r *http.Request) {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := context.Background()
		// Run operations.
		_, err := client.User.Create().SetName("a8m").Save(ctx)
		if err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}
	}
}

func main() {
	// Create Ent client and migrate
	client := createClient()
	// Use the hook
	client.Use(entprom.Hook())
	// Simple handler to run actions on our DB.
	http.HandleFunc("/", handler(client))
	// This endpoint sends metrics to the prometheus to collect
	http.Handle("/metrics", promhttp.Handler())
	log.Println("server starting on port 8080")
	// Run the server
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

通过curl或浏览器多次访问服务器`/`路径后，访问`/metrics`端点即可看到Prometheus客户端的输出：

```
# HELP ent_operation_duration_seconds Time in seconds per operation
# TYPE ent_operation_duration_seconds histogram
ent_operation_duration_seconds_bucket{mutation_op="OpCreate",mutation_type="User",le="0.005"} 2
ent_operation_duration_seconds_bucket{mutation_op="OpCreate",mutation_type="User",le="0.01"} 2
ent_operation_duration_seconds_bucket{mutation_op="OpCreate",mutation_type="User",le="0.025"} 2
ent_operation_duration_seconds_bucket{mutation_op="OpCreate",mutation_type="User",le="0.05"} 2
ent_operation_duration_seconds_bucket{mutation_op="OpCreate",mutation_type="User",le="0.1"} 2
ent_operation_duration_seconds_bucket{mutation_op="OpCreate",mutation_type="User",le="0.25"} 2
ent_operation_duration_seconds_bucket{mutation_op="OpCreate",mutation_type="User",le="0.5"} 2
ent_operation_duration_seconds_bucket{mutation_op="OpCreate",mutation_type="User",le="1"} 2
ent_operation_duration_seconds_bucket{mutation_op="OpCreate",mutation_type="User",le="2.5"} 2
ent_operation_duration_seconds_bucket{mutation_op="OpCreate",mutation_type="User",le="5"} 2
ent_operation_duration_seconds_bucket{mutation_op="OpCreate",mutation_type="User",le="10"} 2
ent_operation_duration_seconds_bucket{mutation_op="OpCreate",mutation_type="User",le="+Inf"} 2
ent_operation_duration_seconds_sum{mutation_op="OpCreate",mutation_type="User"} 0.000265669
ent_operation_duration_seconds_count{mutation_op="OpCreate",mutation_type="User"} 2
# HELP ent_operation_error Number of failed ent mutation operations
# TYPE ent_operation_error counter
ent_operation_error{mutation_op="OpCreate",mutation_type="User"} 1
# HELP ent_operation_total Number of ent mutation operations
# TYPE ent_operation_total counter
ent_operation_total{mutation_op="OpCreate",mutation_type="User"} 2
```

顶部显示的是直方图计算结果，统计了每个"桶"中的操作数量。随后可以看到总操作数和错误数。每个指标都附有描述信息，这些描述会在通过Prometheus仪表板查询时显示。

Prometheus客户端只是整个架构中的一个组件。要运行包含抓取器（定期轮询端点）、存储指标并能响应查询的Prometheus服务，以及交互式UI的完整系统，建议阅读官方文档或使用示例仓库中的docker-compose.yaml文件。

### Ent可观测性未来工作

如前所述，当前存在大量指标收集后端方案，Prometheus只是众多成功项目中的一个。虽然这些解决方案在多个维度存在差异（自托管与SaaS、不同存储引擎及查询语言等），但从指标上报客户端角度看，它们本质上是相通的。

此类情况下，优秀的软件工程实践建议通过接口抽象具体后端实现。该接口可由不同后端实现，使得客户端应用能轻松切换实现方案。近年来行业正发生此类变革，例如开放容器计划（OCI）和服务网格接口（SMI）都致力于为标准问题域定义接口规范。在可观测性领域，OpenCensus和OpenTracing正在合并为OpenTelemetry也体现了同样的趋势。

虽然发布类似本文的Ent+Prometheus扩展件很有吸引力，但我们坚信应该采用基于标准的方法解决可观测性问题。诚邀所有人参与讨论如何为Ent实现这一目标的最佳实践。

### 总结

我们在本文开篇介绍了Prometheus——一个流行的开源监控解决方案。接着探讨了Ent框架中的"Hooks"功能，该功能允许在数据实体变更操作前后添加自定义逻辑。随后我们演示了如何将两者集成，利用Ent构建可观测性应用。最后讨论了Ent在可观测性领域的未来发展方向，并邀请所有人参与[讨论共同塑造其未来](https://github.com/ent/ent/discussions/1819)。

有任何疑问或需要入门帮助？欢迎加入我们的[Discord服务器](https://discord.gg/qZmPgTE6RX)或[Slack频道](https://entgo.io/docs/slack/)。

:::note[获取更多Ent资讯：]

- 订阅我们的[新闻通讯](https://entgo.substack.com/)
- 关注[Twitter账号](https://twitter.com/entgo_io)
- 加入[Gophers Slack](https://entgo.io/docs/slack)的#ent频道
- 参与[Ent Discord社区](https://discord.gg/qZmPgTE6RX)

:::