---
title: OpenTelemetry的一些随笔(1)
description: OpenTelemetry的一些随笔
date: 2020-12-10
tags:
	- opentelemetry
layout: layouts/post.njk
---

中文网站上面关于 OpenTelemetry 的介绍非常之少，仅有的也只是稍微介绍一下发展历史，并没有深入到它的架构、里面所用到的一些基础概念。

当然在开始之前，还是需要稍微提及一下 OpenTelemetry 的概念。

## Observability

---

最近呈冉冉向上之势的 CNCF，在前段时间出了个[Trail Map](https://github.com/cncf/trailmap)，上面洋洋洒洒写了十大项。大体意思就是呢，你要搞 Cloud Native 是不是，那你对着这张图看看你到底有没有脸说。

这十大项是

- Containerized 容器化 - 现在的语境下面基本上就是 docker 化
- CI/CD 持续集成/持续交付 - 这个应该也不用解释了
- Orchestration & Application Definition 容器编排 - 当前语境下就是有没有上 k8s
- Observability & Analysis 可观察 & 分析 - 这就是本文要讲的部分
- Service Proxy, Discovery & Mesh 服务发现/代理/网络 - 也是为了治理 micro service 而引入的 buzzwords
- Networking, Policy & Security 网络政策 - 在云原生下的网络管理
- Distributed Database & Storage 分布式数据库 & 存储 - 这个就不用说了
- Streaming & Messaging 消息化 - 很多公司以为就是用上 kafka 或者任何 mq 就消息化了，实际上关于这一块有非常多可以讨论的方向。很多大厂的人都对此一窍不通，毕竟异步架构比同步架构复杂性增加无数倍。
- Software Distribution 软件分发 - 这块不是很懂，大概是安全性/P2P 分发之类的，对基础设备非常看重的大厂才会用吧。

这十项中，很多公司都有在做，但本文的主题 observability 这个在我看来是属于跟 CI/CD 一样的基础设施，但很少有公司从一开始就考虑。

在这个言必称 Cloud Native 的时代，张口 docker（哪怕 dockerfile 写得跟 xml 一样又臭又长），闭口 kubernetes（哪怕完全不理解 chart 只会东抄一点西抄一点），service mesh、distribution 这些 buzzword 满大街飞，observability 被提及的概率实在是太小了点。

为什么？因为实在不够高大上。

observability 说到底，只是用来看看整个系统运行状态，用于报警或者 debug 用的。整个系统需要记录的三样东西也非常没有技术含量：logs/metrics/traces。

- logs：就是日志，毫无技术含量。CNCF 项目中有 fluentd。
- metrics：就是系统在运行时的一些数据，比如 count（数数，只增不减）、gauge（也是数数，可增可减，反应一段时间内的变化）等。CNCF 项目中有 prometheus。
- traces：记录某个 request 或者某条 event 在它的生命周期内，在不同的 service 留下的痕迹。简单来说，可以理解为集中式日志，而不用在不同 service log 之间跳来跳去查找问题。CNCF 项目中有 jaeger 和 opentracing（opentracing 只是一个标准）。

可以看得到 CNCF 在这三块都有孵化项目，但是都是各司一职。OpenTelemetry 应运而生，目标是整合这三大块的标准，并且可以利用现有的项目：metrics 可以输出到 prometheus，tracing 可以输出到 jaeger。logs 这一块 OpenTelemetry 还没有动手，但也已经在时间表上了。

关于 OpenTelemetry 的历史，大家可以去网上查查看，在此就只是给大家一个概念。下面我们要结合一个具体的例子来说明 OpenTelemetry 的基础概念，以及它的实际用处。

## 实践

---

### 安装 Jaeger

```shell
# 请先确保你机器上有docker
$ docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 14250:14250 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.21
```

打开浏览器 http://localhost:16686/ 就可以看到 Jaeger UI。

### 安装Go
如果你已经安装有Go可以跳进这一步。如果没有的话，可以参考[GVM](https://github.com/moovweb/gvm)。

### 开始
```go
package main

// initTracer creates a new trace provider instance and registers it as global trace provider.
func initTracerProvider() func() {
	// Create and install Jaeger export pipeline.
	flush, err := jaeger.InstallNewPipeline(
		jaeger.WithCollectorEndpoint("http://localhost:14268/api/traces"),
		jaeger.WithProcess(jaeger.Process{
			ServiceName: "trace-demo",
			Tags: []label.KeyValue{
				label.String("exporter", "jaeger"),
			},
		}),
		jaeger.WithSDK(&sdktrace.Config{DefaultSampler: sdktrace.AlwaysSample()}),
	)
	if err != nil {
		log.Fatal(err)
	}
	return flush
}

func main() {
	ctx := context.Background()

	flush := initTracerProvider()
	defer flush()

	tr := otel.Tracer("component-main")
	ctx, span := tr.Start(ctx, "foo")
	defer span.End()

	bar(ctx)
}

func bar(ctx context.Context) {
	tr := otel.Tracer("component-bar")
	_, span := tr.Start(ctx, "bar")
	defer span.End()

	// Do bar...
}
```
尝试运行一下这个例子
```shell
$ go run main.go
```
然后到http://localhost:16686，就能看到数据就已经进去（虽然这时候并不知道这些东西都是什么）。

下面我要开始解耦整个OpenTelemetry了（用最简单但并不一定准确的语言，因为我们需要先架构一个大蓝图）

## 《史记》
---
如果我们要给一个人写传记，大体上应该会照着几个步骤：
- 开头先写一下这个人姓甚名甚，家庭情况，出生年月等。
- 中间像记流水帐一样，何年何月在何地做过何事，重要的事就讲详细点，不重要的事就粗略一点。
- 最后说明一下此人在何时何月何地去世，盖棺论定。

这一人的一生，就可以理解为OpenTelemetry的`tracer`。而TA所做的每一件事，都可以理解为`span`。而记录TA这一生故事的纸，可以理解为`tracer provider`，在上面的例子中，`tracer provider`就是`jaeger`。

所以，在上面的例子中，我们实际上做了这些事情：
- 在main函数中，生成一个tracer。
- 调用这个tracer.Start，得到一个context，以及一个span（这个span的名字叫做foo）。
- 调用bar方法，将context做为参数传入。
- 在bar方法中，再生成一个tracer。
- 调用该tracer.Start方法，将上面传入的context作为参数，得到context和一个新的span（这个span名字叫做bar）。
- 调用在defer中的flush()，将trace和span信息一并保存到`trace provider` `jaeger`中。

在这里可能有点混淆，看`jaeger`里面的意思，这两个名字分别是`foo`和`bar`是属于同一个`trace`的，但是为什么在这里我们却是两个`tracer`？

答案很简单，因为`trace`的信息并不保存在`tracer`里面，而是保存在`context`里面。

前面有提到，`context`里面实际保存了`span` foo的完整信息，`span` foo中有一个`SpanContext`对象，`SpanContext`对象其中保存着`TraceID`。当用非空的`context`作为参数调用tracer.Start时，新生成的`span` bar将用同样的`TraceID`。所以在`jaeger`里面这两个自然会被当成同一个`trace`。

这一段代码里在`opentelemetry-go/sdk/trace/span.go`的`startSpanInternal`中
```go
func startSpanInternal(ctx context.Context, tr *tracer, name string, parent trace.SpanContext, remoteParent bool, o *trace.SpanConfig) *span {
	span := &span{}
	span.spanContext = parent

	cfg := tr.provider.config.Load().(*Config)

	if hasEmptySpanContext(parent) {
		// Generate both TraceID and SpanID
		span.spanContext.TraceID, span.spanContext.SpanID = cfg.IDGenerator.NewIDs(ctx)
	} else {
		// TraceID already exists, just generate a SpanID
		span.spanContext.SpanID = cfg.IDGenerator.NewSpanID(ctx, parent.TraceID)
	}

...
}
```
可以在上面例子中尝试打印`span.SpanContext().TraceID`，会发现两者是一样的。
```shell
$ go run main.go
foo span trace id 16749d09802cf5e0c8299edb59a6b6ed
bar span trace id 16749d09802cf5e0c8299edb59a6b6ed
```

现在相信大家已经明白如何在同一个服务中，利用jaeger进行tracing，在下一篇里面会讲解open-telemetry真正的作用，如何在不同的服务中进行tracing。