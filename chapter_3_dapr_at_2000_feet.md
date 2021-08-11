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

