# 第 3 章 从 20000 英尺之上俯瞰 Dapr

[Dapr at 20,000 feet | Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/dapr-at-20000-feet)

在第 1 章中，我们讨论了分布式微服务应用的吸引力。但是，我们也指出了它会急剧增加架构和操作的复杂性。记住这一点，这样问题就变成了你如何有你的蛋糕，与如何吃掉它。也就是说，你如何既能得到分布式架构敏捷的优势，又能最小化它的复杂性。

Dapr，或者 _分布式应用程序运行时_ ，就是一种构建现代分布式应用程序的方式。

从一个原型开始演进为高度成功的开源项目。它的赞助商 Microsoft，与客户和开源社区紧密合作进行设计和构建 Dapr。Dapr 项目带领开发者从头开始解决开发分布式应用程序的艰难挑战。

本书从 .NET 开发者的角度来看待 Dapr。在本章中，你将建立对 Dapr 的概念理解，以及它是如何工作的。随后，我们提供动手实验来指导你如何在你的应用程序中使用 Dapr。

想象一下，你正在从 20000 英尺高空的飞机上，从窗口往外巡视，广阔的大地展现在眼前。让我们对 Dapr 做同样的事情。将你自己带入 20000 英尺高度，你会看到什么？

## Dapr 和它解决的问题

Dapr 直面现代分布式应用程序的巨大挑战：**复杂性**

考虑一个可插拔组件的架构，Dapr 极大简化了分布式应用程序背后的管道。它提供了 **动态胶水** 从 Dapr 运行时将你的应用程序绑定到架构的功能上。

考虑需要将你的一个服务状态化？你需要设计什么。你可能需要编写定制化的代码来处理状态存储，例如 Redis 缓存。但是，Dapr 提供了开箱即用的状态管理功能。你的服务调用 Dapr 的状态管理 **构建块**，它就可以魔术般的通过 Dapr **组件配置** yaml 文件来绑定到状态存储 **组件**。Dapr 提供了一系列预构建的状态管理组件，包括 Redis。基于该模型，你的服务将状态管理代理给 Dapr 运行时。你的服务不需要 SDK 库，或者直接引用底层的组件。甚至你可以变更你的状态存储，例如从 Redis 变成 MySQL 或者 Cassandra，而不需要修改代码。

图 2-1 展示从 20000 英尺之上俯瞰的 Dapr

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/dapr-at-20000-feet/dapr-high-level.png)

图 2-1 

在图顶部的行中，注意 Dapr 对流行的开发平台提供了语言特定的 SDK。Dapr v1.0 支持包括：Go、Node.js、Python、.NET、Java 和 JavaScript。本书关注于 Dapr 的 .NET SDK，它也提供了对 ASP.NET Core 集成的直接支持。

虽然语言特定的 SDK 扩展了开发者的体验，Dapr 本身是平台无关的。在底层，Dapr 的编程模型通过标准的 HTTP/gRPC 通讯协议提供其能力。任何编程平台可以通过原生 HTTP 和 gRPC API 调用 Dapr 。

位于图中间的蓝色框中表示 Dapr 提供的构建块，其中的每个提供了你的应用程序可以使用的分布式应用程序的功能。

图的底部重点提示了 Dapr 的移植性，它可以跨越多种环境运行。

## Dapr 架构

此时，飞机盘旋在 Dapr 之上，降低高度，使你可以更近的查看 Dapr 是如何工作的。

### 构建块

从这个新的视角，你可以看到 Dapr 构建块的细节。

一个构建块封装了一个分布式架构的功能。可以通过 HTTP 或者 gRPC API 来访问该功能。图 2-2 展示了 Dapr v1.0 提供的构建块。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/dapr-at-20000-feet/building-blocks.png)

图 2-2 Dapr 构建块

下表中说明了每个构建块提供的基础架构服务

| 构建块                       | 说明                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| 状态管理 State Management    | 支持长时间运行的有状态服务的上下文信息                       |
| 服务调用 Service Invocation  | 直接调用，使用平台无关协议和众所周知的端点进行安全服务调用， |
| 发布于订阅 Publish/Subscribe | 在服务之间，实现安全、可扩展的发布/订阅消息                  |
| 绑定 Bindings                | 基于双向绑定通讯，从外部资源触发的事件来触发代码             |
| 可见性 Observability         | 监控和测量跨网络服务的消息调用                               |
| 密钥 Secrets                 | 安全访问外部密钥存储                                         |
| 执行人 Actor                 | 通过可重用的执行人对象封装逻辑与数据                         |

构建块从你的服务中抽象了实现分布式应用该程序功能的实现。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/dapr-at-20000-feet/building-blocks-integration.png)

构建块调用 Dapr 组件，Dapr 组件提供实例化每种资源的实现。你的服务中的代码只需要关心构建块。它不依赖外部的 SDK 或者库。 Dapr 为你处理管道问题。每个构建块都是独立的。你可以在你的应用程序中使用其中一个，或者某些，甚至全部。作为附加值，Dapr 构建块基于业界的最佳实践包括易于理解的可观察性。

在随后的章节中，对于每个构建块，我们提供详细说明和代码示例。此时，飞机会更加降低高度，从新的视角，现在你可以更近的观察 Dapr 组件层。

### 组件

尽管构建块提供了 API 来调用分布式应用程序功能，Dapr 组件提供实现该功能的具体实现。

考虑 **状态存储** 组件，它提供了统一的方式来管理 CRUD 操作，而不需要对你的服务代码进行任何修改。你可以在任何下列 Dapr 状态组件之间切换：

* AWS DynamoDB
* Aerospike
* Azure Blob Storage
* Azure CosmosDB
* Azure Table Storage
* Cassandra
* Cloud Firestore (Datastore mode)
* CloudState
* Couchbase
* Etcd
* HashiCorp Consul
* Hazelcast
* Memcached
* MongoDB
* PostgreSQL
* Redis
* RethinkDB
* SQL Server
* Zookeeper

每个组件通过统一的状态管理接口提供其实现：

```go
type Store interface {
   Init(metadata Metadata) error
   Delete(req *DeleteRequest) error
   BulkDelete(req []DeleteRequest) error
   Get(req *GetRequest) (*GetResponse, error)
   Set(req *SetRequest) error
   BulkSet(req []SetRequest) error
}
```

> 提示：
>
> Dapr 运行时，以及所有的 Dapr 组件都使用 Go 语言开发。Go 是开源社区的流行语言，验证了 Dapr 的跨平台承诺。

也许你使用 Azure Redis Cache 作为你的状态存储。使用如下配置进行设置：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: default
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: <HOST>
  - name: redisPassword
    value: <PASSWORD>
  - name: enableTLS
    value: <bool> # Optional. Allowed: true, false.
  - name: failover
    value: <bool> # Optional. Allowed: true, false.
```

在 spec 配置节，配置 Dapr 使用 Redis Cache 作为状态管理。该配置节也包含组件特定的元数据。在此例中，你还可以使用它来配置其它的 Redis 设置。

在随后的时间里，应用程序准备发布。对于产品观景，你可能希望将状态管理切换为 Azure STorage。Azure Table Storage 提供可负担和高持久的状态管理能力。

在本文撰写的时候，Dapr 提供了下列组件类型：

| 组件                       | 说明                                                         |
| -------------------------- | ------------------------------------------------------------ |
| 服务发现 Service discovery | 被服务调用构建块使用，以便与宿主环境集成，提供服务到服务的发现。 |
| 状态 State                 | 对众多状态存储实现提供统一的访问接口                         |
| 发布/订阅 Pub/Sub          | 对众多消息总线实现提供统一访问接口，                         |
| 绑定 Binding               | 基于双向绑定通讯，从外部资源触发的事件来触发代码             |
| 中间件 Middleware          | 支持自定义的中间件插入到请求处理管线中，在请求或者响应的中调用附加的操作。 |
| 密钥存储 Secret store      | 对外部密钥存储提供统一的访问接口，包括 Cloud、Edge、商业、开源服务等。 |

飞机已经完成了环绕 Dapr 的飞行，一再回顾，你可以看到它是如何连接在一起的。

### Sidebar 架构

Dapr 通过 [sidebar 架构]([Sidecar pattern - Cloud Design Patterns | Microsoft Docs](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar)) 来暴露出构建块和组件。Sidebar 使得 Dapr 伴随你的服务运行在独立的内存和进程中。Sidebar 不是你的服务的一部分，提供了隔离和封装性。但是可以连接到它。该分离使得每个组件可以有自己的运行环境，由不同的编程平台所构建。图 2-4 展示了 sidebar 模式。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/dapr-at-20000-feet/sidecar-generic.png)

图 2-4 Sidebar 模式

该模式被命名为 sidebar，中文称为挎斗，因为它就像三轮摩托的挎斗连接到摩托车上。在上面的图中，注意到 Dapr 是如何连接到你的服务以提供分布的应用能力的。

### 寄宿环境

Dapr 拥有跨平台的能力，可以运行在多种不同环境下。这些环境包括：Kubernates，一组虚拟机，或者边缘环境，例如 Azure IoT 等等。

对于本地环境，最容易的方式是使用自寄宿模型。在自寄宿模式下，微服务和 Sidebar 运行在分离的本地进程中，而不需要诸如 Kubernetes 容器调度器。更多信息，可以参考 [download and install the Dapr CLI](https://docs.dapr.io/getting-started/install-dapr/)。

图 2-5 展示了分别寄宿于独立的内存进程中，通过 HTTP 或者 gRPC 通讯的应用程序和 Dapr。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/dapr-at-20000-feet/self-hosted-dapr-sidecar.png)

图 2-5 自寄宿的 Dapr sidecar

默认情况下，Dapr 为 Redis 和 Zipkin 安装 Docker 容器，以提供默认的状态管理和可见性。如果你不希望在本地机器上安装 Docker，可以在自寄宿模型下，不需要任何 Docker 容器运行 Dapr。不过，你必须安装手动安装默认的组件，诸如用于状态管理的 Redis 和 Pub/Sub 组件。

Dapr 可以运行在容器化的环境下，例如 Kubernetes，图 2-6 展示了运行在独立的 side-car 容器中的 Dapr，伴随着应用程序的容器一起位于同一个 Kubernetes 的 Pod 中。

  ![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/dapr-at-20000-feet/kubernetes-hosted-dapr-sidecar.png)

图 2-6 寄宿于 Kubernetes 中的 Dapr sidebar

## Dapr 性能考虑

如你所见，Dapr 使用 Sidebar 模式来解耦于你的应用程序与分布式应用程序功能之间的连接。调用某个 Dapr 操作需要至少一次进程外的网络调用。图 2-7 展示了 Dapr 流量模式的示例。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/dapr-at-20000-feet/dapr-traffic-patterns.png)

图 2-7 Dapr 的流量模式

请看上图，一个问题是网络延迟和开销影响到每次调用。

Dapr 团队在性能方面投入大量资源。大量的工程工作已经投入到 Dapr 中使得 Dapr 高效。在 Dapr 之间的调用总是使用 gRPC，这提供了更高的性能和更小的二进制负载。在大多数情况下，额外的开销为小于毫秒。

为了增进性能，开发人也可以使用 gRPC 来访问 Dapr 构建块。

gRPC 是现代的，高性能框架，发展了古老的远程调用 RPC 协议。gRPC 使用 HTTP/2 作为传输协议，相比 HTTP RESTFul 提供了显著的性能增加，包括：

* 多路复用支持，用来在单个连接上发送多个并行的请求，HTTP 1.1 限制在同时只能处理单个请求/响应消息。
* 双向全双工通讯，同时发送客户端请求和服务器响应
* 内建的流支持，对于大量数据支持请求和响应异步流

更多内容，可以参考 [gRPC overview](https://docs.microsoft.com/en-us/dotnet/architecture/cloud-native/grpc#what-is-grpc) 

## Dapr 和服务聚合

对于分布式应用程序，服务聚合是另一个快速演进的技术。

服务聚合是一种可配置的基础架构层，内建处理服务到服务的通讯、弹性、负载均衡、侦测捕获。它将这些问题的处理从服务中移出到服务聚合层中。与 Dapr 类似，服务聚合也遵循 sidebar 架构。

图 2-8 展示了实现服务聚合技术的一个应用程序。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/dapr-at-20000-feet/service-mesh-with-side-car.png)

上图展示了消息是如何被 sidebar 代理拦截，这些代理伴随每个服务运行。每个代理可以针对服务配置流量规则。它理解消息，并可以在你的服务之间，以及与外部世界路由消息。

所以，问题变成了：Dapr 是服务聚合吗？

尽管它们都使用 sidebar 架构，但每种技术有着不同的目的。Dapr 提供分布式应用的功能。而服务聚合提供专门的网络基础架构层。

由于它们工作在不同的层级上，它们可以在同一个应用中一起工作。例如，服务聚合可以提供服务之间的网络通讯。Dapr 可以提供应用程序的服务，诸如状态管理或者工作者服务。

图 2-9 展示了同时实现 Dapr 和服务总线技术的一个应用程序

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/dapr-at-20000-feet/dapr-and-service-mesh.png)

图 2-9 Dapr 与服务聚合一起协同

[Dapr online documentation](https://docs.dapr.io/concepts/faq/#networking-and-service-meshes) 介绍了 Dapr 与服务聚合的集成。

## 总结

本章介绍了 Dapr，分布式的应用程序运行时。

Dapr 是由微软支持的开源项目，与客户和开源社区紧密协作。

本质上，Dapr 帮助减少分布式微服务应用程序固有的复杂性。它建立在构建块 API 的概念之上。Dapr 构建块暴露公共的分布式应用功能，诸如状态管理，服务到服务的调用，以及消息的发布/订阅等等。Dapr 组件位于构建块之下，为每个功能提供具体实现。应用程序通过配置文件绑定到各种组件。

在下一章，我们将提供练习，关于如何在你的应用程序中使用 Dapr 的动手指导。



