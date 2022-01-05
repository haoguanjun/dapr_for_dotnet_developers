# 第 11 章 可观察性构建块

[The Dapr observability building block | Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/observability)



现代分布式系统是复杂的。从小的、松耦合的、无依赖的可部署服务开始，这些服务跨越进程和服务器边界。它们消费不同种类的基础架构后台服务 ( 数据库、消息中间件、密钥存储等等 )。最终，这些不同的片段组合在一起成为一个应用。

对于这些分离的，不断变化的部件，你怎么能知道正在发生什么呢？不幸的是，以前所遗留下来的监控方式是不够的。相反，系统必须是全程可见的。现代的 **可见性** 实践提供应用程序健康全时的可见和洞察。它使得你通过观察输出可以推断内部的状态。不仅仅是对分布式应用程序观察的强制监控和问题定位，它需要从头实施。

用来提取可见性系统性的被称为 **telemetry**。它可以分为 4 个类别

* **Distributed tracing**，提供对于分布式业务事务中服务间调用的流量洞察
* **Metrics**，提供服务和资源消费的性能洞察
* **Logging**，提供代码是如何执行，以及是否有错误发生的洞察
* **Health**，提供服务可用性洞察的端点

遥测的深度取决于应用平台提供的可观察性特性。以 Azure 为例，它提供了丰富的可观察性体验，包括了所有的遥测种类。只需要很少的配置，Azure IaaS 和 PaaS 服务就可以提供并发布遥测信息到 Azure Monitor 和 Azure Application insights services 上。应用程序洞察提供了系统日志、追踪以及问题领域的高度可视化的仪表板。甚至可以生成图表来基于服务之间的通讯来展示之间的依赖关系。

不过，如果应用程序不能使用 Azure 的 PaaS 和 IaaS 资源呢？还有可能获得关于应用程序洞察的丰富的遥测体验吗？答案是：是的。非 Azure 应用程序可以导入一些库，增加配置，并监测代码来生成遥测信息给 Azure application insights。但是，这种方式把应用程序与 Application Insights **紧耦合** 。将该应用移植到其它监控平台会导致高昂的重构。避免紧耦合，并在代码之外使用可观察性不更好吗？

使用 Dapr，你可以做到这一点。让我们看一下 Dapr 是如何增加可观察性到我们的分布式应用程序的。

## 1.1 解决何种问题

Dapr 可观察性构建块从应用程序中解耦了可观察性。它可以自动捕获由 Dapr sidecar 和 Dapr 本身的 Dapr 系统服务所生成的流量。该构建块关联分布于多个服务服务的单个操作的流量。它还提供关于性能测量、资源使用和系统健康的信息。Telemetry 被发布为开放标准格式，信息可以发送到你选择的后台监控平台。进而，这些信息可以被可视化、查询并分析。

由于 Dapr 抽象了遥测的实现细节，应用程序对可观察性是如何实现的是无感的。不需要在代码中引入外部库或者实现自定义的监测代码。Dapr 允许开发者专注于构建业务逻辑而不是基础细节。可观察性被配置为 Dapr 系统级别并一致地跨域所有的服务，甚至是由不同的团队开发，并使用不同的技术栈。

## 1.2 它是如何工作的

Dapr 的 sidecar 架构内置可观察性功能。 Dapr 的 sidecar 切入服务之间的通讯，并自动提取追踪、性能监测和日志信息。遥测数据被发布为开放标准格式。默认情况下，Dapr 支持 OpenTelemetry 和 Zipkin。

Dapr 提供一系列的收集器，它们可以发布遥测数据到不同的后台监控工具。这些工具基于 Dapr 的遥测进行分析和查询。图 10-1 展示了 Dapr 可观察性架构。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/observability/observability-architecture.png)

图 10-1 Dapr 可观察性架构

1. 服务 A 调用服务 B 上的一个操作。该调用由服务 A 的 sidecar 路由到服务 B 的 sidecar。
2. 当服务 B 完成操作之后，响应内容通过 sidecar 被路由回服务 A。sidecar 收集并发布所有请求和响应的遥测数据。
3. 配置的收集器收集遥测数据，被发送到后台的监测平台

作为开发人员，切记与配置其它的 Dapr 构建块不同，比如说 pub/sub 或者状态管理构建块等等。不是引用一个构建块，而是增加收集器和监控后台。图 10-1 展示了可以配置多个收集器并发送到不同的监控后台。

在本章的开始部分，我们识别 4 类遥测数据。下面的章节将提供每一类别的详细内容。将包含如何配置收集的指令，并与流行的监控后台进行集成。

## 10.3 分布式追踪

分布式追踪提供对分布式系统中跨越多个服务的流量追踪。对于问题调查而言，交换的请求和响应消息的日志是解决问题的宝贵信息来源。困难的是属于同一个业务事务的相关信息。

Dapr 使用  [W3C Trace Context](https://www.w3.org/TR/trace-context) 来关联这些消息。它对于同一个操作注入相同的上下文谢你到请求和响应中。图 10-2 展示了相关性工作。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/observability/w3c-trace-context.png)

> 注意
>
> _trace context_ 通常指在微服务场景下的相关性令牌。

图 10-2 W3C Trace Context 示例

1. 服务 A 调用 服务 B 上的一个操作。由于服务 A 开启调用，Dapr 创建一个车唯一的 trace 上下文，并注入到请求中。
2. 服务 B 收到请求并调用服务 C 上的操作。Dapr 检测到进入的请求包含 trace 上下文，提取它然后注入到针对服务 C 的调用中。
3. 服务 C 收到请求并处理。Dapr 检测到到来的请求中包含 trace context ，首先提取，然后注入到对服务 B 的响应中。
4. 服务 B 收到响应并处理。然后它创建响应，提取 trace 的上下文，并注入到对服务 A 的响应中

这些相关的所有请求和响应被称为一个 _trace_。图 10-3 展示了一个 trace:

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/observability/traces-spans.png)

图 10-3 Trace 与 Span

在图中，注意 trace 是如何表示单个的跨越众多服务的应用程序事务，Trace 是 Span 的集合。每个 Span 表示 Trace 内的单个操作或者工作单元。Span 是服务之间实现单个事务发送的请求和响应的总和。

### 10.3.1 使用 Zipkin 作为后端监控

Zipkin 是开源的分布式追踪系统。它可以接收并可视化遥测数据。Dapr 对 Zipkin 提供默认支持。下面的示例演示了如何配置 Zipkin 来可视化 Dapr 遥测。

#### 10.3.1.1 启用并配置追踪

在开始的时候，必须使用 Dapr 配置文件对 Dapr 运行时启用追踪。下面是名为 `dapr-config.yaml` 的启用追踪的配置文件：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: dapr-config
  namespace: default
spec:
  tracing:
    samplingRate: "1"
    zipkin:
      endpointAddress: "http://zipkin.default.svc.cluster.local:9411/api/v2/spans"
```

采样率 `samplingRate`  属性设置发布追踪信息的间隔。该值必须在 0 ( 禁用 )  到 1 ( 发布所有追踪 ) 之间。举例来说，使用 0.5，只发布一半的追踪，可以显著减少发布的流量。`endpointAddress` 执行运行在 Kubernetes 集群中的 Zipkin 服务器的端点地址。对于 Zipkin 来说，默认的端口是 9411。该配置必须使用 Kubernetes CLI 命令应用到 Kubernetes 集群中

```bash
kubectl apply -f dapr-config.yaml
```

#### 10.3.1.2 安装 Zipkin 服务器

在 Dapr 自托管模式下，Zipkin 服务器被自动安装，追踪也在默认的位于 `$HOME/.dapr/config.yaml` 或者 Windows 系统中的 `%USERPROFILE%\.dapr\config.yaml` 配置文件中被启用。

当运行在 Kubernetes 集群中时，Zipkin 必须手工部署。使用下面的名为 `zipkin.yaml` 的 Kubernetes 清单文件来部署标准的 Zipkin 服务器到 Kubernetes 集群中。

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: zipkin
  namespace: dapr-trafficcontrol
  labels:
    service: zipkin
spec:
  replicas: 1
  selector:
    matchLabels:
      service: zipkin
  template:
    metadata:
      labels:
        service: zipkin
    spec:
      containers:
        - name: zipkin
          image: openzipkin/zipkin-slim
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 9411
              protocol: TCP

---

kind: Service
apiVersion: v1
metadata:
  name: zipkin
  namespace: dapr-trafficcontrol
  labels:
    service: zipkin
spec:
  type: NodePort
  ports:
    - port: 9411
      targetPort: 9411
      nodePort: 32411
      protocol: TCP
      name: zipkin
  selector:
    service: zipkin
```

该部署使用标准的 `openzipkin/ziplin-slim` 容器镜像。Zipkin 服务暴露 Zipkin web 前端界面，你可以通过端口 `32411` 来查看遥测数据。使用 Kubernetes CLI 将 Zipkin 清单文件应用到 Kubernetes 集群中来部署 Zipkin 服务器

```bash
kubectl apply -f zipkin.yaml
```

#### 10.3.1.3 配置服务使用追踪配置

现在已经配置正确并开始发布遥测数据了。任何部署为应用程序一部分的 Dapr sidecar 必须要求在启动的时候生成遥测数据。为了这一目的，对每个部署的服务，添加 `dapr.io/config` 注解来引用 `dapr-config` 配置。下面是交通管理示例应用中罚款服务 FineCollection 的包含该注解的清单文件：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: finecollectionservice
  namespace: dapr-trafficcontrol
  labels:
    app: finecollectionservice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: finecollectionservice
  template:
    metadata:
      labels:
        app: finecollectionservice
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "finecollectionservice"
        dapr.io/app-port: "6001"
        dapr.io/config: "dapr-config"
    spec:
      containers:
      - name: finecollectionservice
        image: dapr-trafficcontrol/finecollectionservice:1.0
        ports:
        - containerPort: 6001
```

#### 10.3.1.4 在 Zipkin 中查看遥测数据

应用程序启动之后，Dapr sidecar 将生成遥测数据发送给 Zipkin 服务器。为了查看遥测数据，在浏览器中导航到 `http://localhost:32411`。你将可以看到 Zipkin 的前端界面：

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/observability/zipkin.png)

图 10-4 Zipkin 前端界面

在 _Find a trace_ 面板上，可以查询 traces。不指定任何限制就点击 _RUN QUERY_ 按钮可以看到所有的 traces。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/observability/zipkin-traces-overview.png)

图 10-5 Zipkin trace 概览

点击特定 trace 后面的 _SHOW_ 按钮，可以看到该 trace 的详细内容：

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/observability/zipkin-trace-details.png)

图 10-6 Zipkin trace 详情

在详情页面中的每个项目是一个 Span，表示选中 trace 的一个请求。

#### 10.1.3.5 检查服务之间的依赖关系

由于 Dapr 的 Sidecar 处理服务之间的流量。Zipkin 可以使用 trace 信息来检测出服务之间的依赖关系。要检查该依赖关系，转到 Zipkin 页面的 _Dependencies_ 面板，并选择，Zipkin 将展示这些服务和它们之间的依赖关系。



![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/observability/zipkin-dependencies.png)

图 10-7 Zipkin 的依赖关系

服务之间动画形式的点展示了从源头到目标的移动。红色的点表示失败的请求。

### 10.3.2 使用 Jaeger 或者 New Relic 监控后台

除了 Zipkin 之后，还有其它的后台软件也可以收集 Zipkin 格式的遥测数据。Jaeger 是另一个由 Uber 创建的开源追踪系统。它被用于在分布式的服务之间追踪事务，和在复杂的微服务环境下定位错误。New Relic 是一个全栈可观察性平台。它可以从分布式的应用中关联数据来提供你的系统的完整视图。为了尝试它们，可以将 Dapr 配置文件中的 `endpointAddress` 设置为 Jaeger 或者 New Relic 服务器。下面是配置 Dapr 发送遥测数据到 Jaeger 服务器的配置文件。用于 Jeager 的 URL 类似于 Zipkin，只有服务器运行的端口号不同：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: dapr-config
  namespace: default
spec:
  tracing:
    samplingRate: "1"
    zipkin:
      endpointAddress: "http://localhost:9415/api/v2/spans"
```

为了尝试 New Relic，将端点指定为 New Relic API 的端点。下面是用于 New Relic 的配置文件示例。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: dapr-config
  namespace: default
spec:
  tracing:
    samplingRate: "1"
    zipkin:
      endpointAddress: "https://trace-api.newrelic.com/trace/v1?Api-Key=<NR-API-KEY>&Data-Format=zipkin&Data-Format-Version=2"
```

对于如何使用这两个服务，请参阅 Jaeger 和 New Relic 站点。



## 10.4 Metrics

Metric 提供关于性能和资源使用的洞察。在系统底层，Dapr 生成众多的系统和运行时测量。Dapr 使用 [Prometheus](https://prometheus.io/) 作为测量标准。Dapr sidecar 和系统服务在端口 `9090` 暴露出测量端点。*Prometheus scraper*  使用预定义的间隔访问该端点来收集测量数据。_scraper_ 将测量数据发送给监控后台，图 10-8 展示了收集过程。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/observability/prometheus-scraper.png)

图 10-8 抓取 Prometheus 测量数据

每个 sidecar 和系统服务暴露监听在端口 `9090` 的测量端点。Prometheus 测量抓取器从每个端点捕获测量数据，并发布到后端的监控平台。

### 10.4.1 服务发现

你可能奇怪测量数据的抓取器是如何知道在哪里收集测量数据。Promethues 可以集成内置于发布环境的发现机制。例如，当运行在 Kubernetes 环境时，Prometheus 可以集成 Kebernetes API 来发现运行在该环境下的所有可用 Kebernetes 资源。

### 10.4.2 Metrics 列表

Dapr 从 Dapr 服务和运行时生成大量的测量数据。一些示例如下：

| Metric                                         |  来源   | 说明                                                         |
| :--------------------------------------------- | :-----: | :----------------------------------------------------------- |
| dapr_operator_service_created_total            | System  | The total number of Dapr services created by the Dapr Operator service. |
| dapr_injector_sidecar_injection/requests_total | System  | The total number of sidecar injection requests received by the Dapr Sidecar-Injector service. |
| dapr_placement_runtimes_total                  | System  | The total number of hosts reported to the Dapr Placement service. |
| dapr_sentry_cert_sign_request_received_total   | System  | The number of certificate signing requests (CRSs) received by the Dapr Sentry service. |
| dapr_runtime_component_loaded                  | Runtime | The number of successfully loaded Dapr components.           |
| dapr_grpc_io_server_completed_rpcs             | Runtime | Count of gRPC calls by method and status.                    |
| dapr_http_server_request_count                 | Runtime | Number of HTTP requests started in an HTTP server.           |
| dapr_http/client/sent_bytes                    | Runtime | Total bytes sent in request body (not including headers) by an HTTP client. |

更多关于可用的测量数据，请查阅 [Dapr metrics documentation](https://docs.dapr.io/operations/monitoring/metrics/).



### 10.4.3 配置 Dapr metrics

在运行时，可以在 Dapr 命令中使用 `--enable metrics=false` 来禁用 metric 收集端点。或者，使用 `--metrics-port` 参数来改变默认的端点。

可以使用 Dapr 配置文件来静态启用或者禁止运行时的 metric 收集。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: dapr-config
  namespace: dapr-trafficcontrol
spec:
  tracing:
    samplingRate: "1"
  metric:
    enabled: false
```



### 10.4.4 可视化 Dapr metrics

使用 Prometheus 抓取器收集和发布 metic 到监控后台，你又如何理解这些原始数据呢？一个流行的用来分析 Metic 的工具时 Grafana 。使用 Grafana，你可以从可用的 

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/observability/grafana-sample.png)

图 10-9 Grafana 仪表板

Dapr 文档中包含一个安装 Prometheus 和 Grafana 的教程。

## 10.5 Logging

日志提供服务在运行时刻正在发生何种情况的洞察。当运行应用程序的时候，Dapr 自动通过 sidecar 和 Dapr 系统服务生成日志。不过，通过你的应用程序代码生成的日志不会自动包括在其中。为了从应用程序代码生成日志，你可以特定的 SDK，例如 OpenTelemetry SDK for .NET。从应用程序代码生成日志在本章后面的 _使用 Dapr .NET SDK_ 部分说明。

### 10.5.1 Log 条目结构

Dapr 生成结构化的日志。每个日志条目包含如下内容。

| Field    | Description                                              | Example                             |
| :------- | :------------------------------------------------------- | :---------------------------------- |
| time     | ISO8601 formatted timestamp                              | `2021-01-10T14:19:31.000Z`          |
| level    | Level of the entry (`debug`, `info`, `warn`, or `error`) | `info`                              |
| type     | Log Type                                                 | `log`                               |
| msg      | Log Message                                              | `metrics server started on :62408/` |
| scope    | Logging Scope                                            | `dapr.runtime`                      |
| instance | Hostname where Dapr runs                                 | TSTSRV01                            |
| app_id   | Dapr App ID                                              | finecollectionservice               |
| ver      | Dapr Runtime Version                                     | `1.0`                               |

当在故障定位过程中搜索日志的时候，`time` 和 `level` 字段特别有帮助。`time` 字段可以排序日志条目，所以你可以准确定位特定的时间段。`debug` 级别的日志提供代码行为的详细说明。

### 10.5.2 纯文本 vs JSON 格式

在默认设置下，Dapr 生成纯文本格式的结构化日志。每条日志格式化为由 键值对 所组成的字符串。下面是纯文本日志的一组示例。

```
== DAPR == time="2021-01-12T16:11:39.4669323+01:00" level=info msg="starting Dapr Runtime -- version 1.0 -- commit 196483d" app_id=finecollectionservice instance=TSTSRV03 scope=dapr.runtime type=log ver=1.0
== DAPR == time="2021-01-12T16:11:39.467933+01:00" level=info msg="log level set to: info" app_id=finecollectionservice instance=TSTSRV03 scope=dapr.runtime type=log ver=1.0
== DAPR == time="2021-01-12T16:11:39.467933+01:00" level=info msg="metrics server started on :62408/" app_id=finecollectionservice instance=TSTSRV03 scope=dapr.metrics type=log ver=1.0
```

尽管看起来简单，但是难以解析。如果使用监控工具来查看日志的话，你会希望启用 JSON 格式的日志。使用 JSON 格式，监控工具可以索引和查询单个的字段。下面是上述日志的 JSON 格式的示例。

```json
{"app_id": "finecollectionservice", "instance": "TSTSRV03", "level": "info", "msg": "starting Dapr Runtime -- version 1.0 -- commit 196483d", "scope": "dapr.runtime", "time": "2021-01-12T16:11:39.4669323+01:00", "type": "log", "ver": "1.0"}
{"app_id": "finecollectionservice", "instance": "TSTSRV03", "level": "info", "msg": "log level set to: info", "scope": "dapr.runtime", "type": "log", "time": "2021-01-12T16:11:39.467933+01:00", "ver": "1.0"}
{"app_id": "finecollectionservice", "instance": "TSTSRV03", "level": "info", "msg": "metrics server started on :62408/", "scope": "dapr.metrics", "type": "log", "time": "2021-01-12T16:11:39.467933+01:00", "ver": "1.0"}
```

为了启用 JSON 格式化，需要配置每个 Dapr sidecar。在自托管模式下，你可以通过在命令行指定 `--log-as-json` 标志。

```bash
dapr run --app-id finecollectionservice --log-level info --log-as-json dotnet run
```

在 Kubernetes 环境下，可以对应用程序的每个部署增加一个 `dapr.io/log-as-json` 说明：

```yaml
annotations:
   dapr.io/enabled: "true"
   dapr.io/app-id: "finecollectionservice"
   dapr.io/app-port: "80"
   dapr.io/config: "dapr-config"
   dapr.io/log-as-json: "true"
```

当使用 Helm 将 Dapr 安装到 Kubernetes 时，可以对所有的 Dapr 系统服务启用 JSON 格式日志。

```bash
helm repo add dapr https://dapr.github.io/helm-charts/
helm repo update
kubectl create namespace dapr-system
helm install dapr dapr/dapr --namespace dapr-system --set global.logAsJson=true
```

### 10.5.3 收集日志

Dapr 生成的日志可以被收集到后台的监控系统用来分析。日志收集器从系统收集日志并发送到后端。流行的日志收集器是  [Fluentd](https://www.fluentd.org/)。请查阅 Dapr 文档中的  [How-To: Set up Fluentd, Elastic search and Kibana in Kubernetes](https://docs.dapr.io/operations/monitoring/logging/fluentd/) 。该文档包含设置 Fluentd 作为日志收集器，以及使用  [ELK Stack](https://www.elastic.co/elastic-stack) (Elastic Search and Kibana) 作为后端的说明 。

## 10.6 健康状况

服务的健康状态提供服务可用性的洞察。每个 Dapr sidecar 提供健康 API 以便托管环境用来检查 sidecar 的健康状态。该 API 只有一个操作：

```
GET http://localhost:3500/v1.0/healthz
```

该操作返回两种 HTTP 状态码：

* 204：sidecar 健康
* 500：sidecar 不健康

当运行在自托管模式下，健康 API 不会被自动调用。你可以通过应用程序代码或者健康监控工具来调用。

当运行在 Kubernetes 环境下，Dapr sidecar 注入器自动配置 Kubernetes 来使用该健康 API 执行 *liveness probes* 和 *readiness probes*。

Kubernetes 使用 liveness 探针来决定容器是否在正常运行。如果 liveness 探针返回失败码，Kubernetes 将认为该容器已经死掉，然后自动重新启动它。该功能增强了你的应用程序的可用性。

Kubernetes 使用 readiness 探针来决定容器是否已经就绪来处理流量。当 Pod 中所有的容器都就绪之后，Pod 被认为已经就绪。Readiness 决定了在负载均衡场景下， Kubernetes 服务是否可以导入流量到 Pod 中。没有准备就绪的 Pod 被自动从负载均衡中删除。

Liveness 和 readiness 探针有多个配置参数。它们都在 pod 的描述文件的指定配置节中进行配置。默认情况下，Dapr 对每个 sidecar 使用如下的配置：

```yaml
livenessProbe:
      httpGet:
        path: v1.0/healthz
        port: 3500
      initialDelaySeconds: 5
      periodSeconds: 10
      timeoutSeconds : 5
      failureThreshold : 3
readinessProbe:
      httpGet:
        path: v1.0/healthz
        port: 3500
      initialDelaySeconds: 5
      periodSeconds: 10
      timeoutSeconds : 5
      failureThreshold: 3
```

下面的参数可用于探针：

* `path` 指定 Dapr 健康 API 的端点
* `port` 指定 Dapr 健康 API 的端口
* `initialDelaySeconds` 指定 Kubernetes 第一次检查容器时等待的 秒 数
* `periodSeconds` 指定 Kebernetes 每两次检测之间的间隔 秒 数。
* `timeoutSeconds` 指定 Kubernetes 检测的响应超时时间 秒 数。超时被视为失效
* `failureThreshold` 指定 Kubernetes 认为容器不再存活或准备就绪时，所指定的失效次数。



## 10.7 Dapr 仪表板

Dapr 提供了一个仪表板来展示应用程序、组件和配置的状态信息。使用 Dapr CLI 来以 Web 应用程序的形式在本机的 8080 端口启动该仪表板。

```bash
dapr dashboard
```

对于 Kubernetes 模式下的 Dapr，使用如下命令：

```bash
dapr dashboard -k
```

该仪表板会展示应用程序中有 Dapr sidecar 的所有服务概览。下面的截屏展示了运行在 Kubernetes 中的交通管理示例应用程序的 Dapr 仪表板。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/observability/dapr-dashboard-overview.png)

图 10-10 Dapr 仪表板概览

当进行故障定位时，Dapr 仪表板独立于 Dapr 应用程序。它提供关于 Dapr sidecar 和服务的信息。你可以深入到每个服务的配置，包括日志。

该仪表板还展示应用程序中配置的组件 ( 以及其配置信息 )。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/observability/dapr-dashboard-components.png)

图 10-11 Dapr 仪表板组件

仪表板提供大量的信息。可以通过运行 Dapr 应用程序并浏览该仪表板来发现。

点击 Dapr 文档中的  [Dapr dashboard CLI command reference](https://docs.dapr.io/reference/cli/dapr-dashboard/) 来获取更多关于 Dapr 仪表板的命令。

## 10.8 使用 Dapr .NET SDK

Dapr 的 .NET SDK 不包含任何特定的可见性功能。所有可用的可见性功能在 Dapr 级别提供。

如果你希望从你的 .NET 应用程序代码中生成遥测数据，可以考虑使用 [OpenTelemetry SDK for .NET](https://opentelemetry.io/docs/net/)。Open Telemetry 项目时跨平台，开源供应商中立的。它提供生成、发送、收集、处理和导出遥测数据的端到端的实现。每种语言都有一个测量库来支持自动和手动测量。Telemetry 使用 Open Telemetry 标准发布遥测数据。该项目有广泛的工业支持和从云服务商、供应商及最终用户的适配。

## 10.9 示例应用：Dapr 交通管理

由于交通管理示例应用程序运行在 Dapr 之上。所有在本章中介绍的遥测都是可用的。如果你运行该应用程序并打开 Zipkin 的 Web 前端，你就可以看到端到端的追踪。如图 10-12 所示：

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/observability/traffic-control-zipkin.png)

图 10-12 Zipkin 端到端追踪示例

该追踪展示了当超速被检测到时，发生的通讯内容：

1. 出现的汽车触发 MQTT 输入绑定，发送包含汽车牌照号码、车道和时间戳的消息
2. MQTT 输入绑定使用该消息调用 TrafficControl 服务
3. TrafficControl 服务获取该车辆状态，添加到车辆信息中，并保存更新之后的车辆状态到状态存储中。
4. TrafficControl 服务使用 pub/sub 订阅发布该超速信息到 `speedingviolations` 主题上
5. FineCollection 服务使用在 `speedingviolations` 主题上的 pub/sub 订阅接收到超速信息
6. FineCollection 服务使用服务调用来调用 VehicleRegistration 服务上的 `vehicleinfo` 端点
7. FineCollection 服务调用输出绑定来发送邮件

点击任何 trace 行 ( span ) 来查看详细内容。如果你点击最后一行，你会看到 `sendmail` 绑定组件被调用，向车辆驾驶员发送违章提醒。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/observability/traffic-control-zipkin-details.png)

图 10-13 trace 的输出绑定详情

## 10.10 总结

详尽的可观察能力时在生产环境下运行分布式应用的关键。

Dapr 提供各种类型的遥测，包括分布式追踪，日志，测量和健康状态。

Dapr 仅仅通过 Dapr 的系统服务和 sidecar 来生成遥测数据。来自应用程序代码的遥测不会自动包括。不过你可以使用特定的 SDK，例如 OpenTelemetry SDK for .NET 从你的应用程序代码生成遥测。

Dapr 使用开放标准格式生成遥测数据，所以可以被集成到广泛的监控工具中。例如 Zipkin、Azure Application Insights、ELK 工具栈、New Relic 和 Grafana 等等。请查阅 Dapr 文档中的  [Monitor your application with Dapr](https://docs.dapr.io/operations/monitoring/) 来学习如何使用特定的监控工具来监控你的应用程序。

你需要遥测抓取器来抓取遥测并发布到监控后端。

Dapr 可以配置为生成结构化的日志。结构化日志更受欢迎因为它可以被后端工具索引。索引后的日志允许用户在日志上执行丰富的查询。

Dapr 提供了仪表板，它提供关于 Dapr 服务和其配置的信息。



## 10.11 参考资料

- [Azure Application Insights](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview/)
- [Open Telemetry](https://opentelemetry.io/)
- [Zipkin](https://zipkin.io/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [Jaeger](https://www.jaegertracing.io/)
- [New Relic](https://newrelic.com/)
- [Prometheus](https://prometheus.io/)
- [Grafana](https://grafana.com/grafana/)
- [Open Telemetry SDK for .NET](https://opentelemetry.io/docs/net/)
- [Fluentd](https://www.fluentd.org/)
- [ELK stack](https://www.elastic.co/elastic-stack)
- [Seq](https://datalust.co/seq)
- [Serilog](https://serilog.net/)



