# 交通管制示例应用

[Introduction to the Traffic Control sample application | Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/sample-application)

在前面的章节种，你已经学习了基本的 Dapr 概念。见识了 Dapr 如何帮助你和你的团队在构建分布式应用的时候，降低架构和操作的复杂性。本章介绍你可以用来探索 Dapr 构建块的示例应用。

> 注意：
>
> 从 [Dapr Traffic Control Github repo](https://github.com/EdwinVW/dapr-traffic-control) 下载示例应用。该仓库包含如何在你的机器上运行该示例应用的详细说明。

交通管制示例应用模拟高速公路的交通管制系统。它的目的是检测超速的汽车并给违章司机开罚款单。这类应用在现实中存在，这里是它是如何工作的。一组摄像头 (每条车道一个) 安装在一段没有出入口的高速公路的起始和结束点 (例如 10 公里) 。在汽车通过摄像头下方的时候，对该汽车拍照。使用光学字符识别 ( OCR ) 系统软件，从照片中提取汽车牌照号码。如果平均速度高于高速公路的限速，系统提取驾驶员信息，并自动开发罚单。

尽管该仿真系统很简单。系统的职责可以分为多个微服务。图 4-1 展示了应用程序中每个服务的概览。



![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/sample-application/services.png)

图 4-1 示例应用中的服务

- **摄像头仿真** 是一个 .NET Core 的控制台应用程序，模拟汽车并发送消息到流量管控服务。每辆仿真的汽车都会调用进入和离开端点服务两者。
- **交通管控服务** 是一个 ASP.NET Core Web API 应用程序，提供 `/entrycam` 和 `/exitcam` 两个端点。调用其中的一个端点服务可以模拟汽车分别通过旗下的入口或出口。请求消息中简单地包含了汽车牌照 ( 没有实现实际的 OCR )
- **FineCollection 服务** 是一个 ASP.NET Core Web API 应用程序，只提供一个端点服务 `/collectfine`。调用此端点将发送罚款提示给超速汽车的驾驶员。请求的内容包含超速违章的所有信息
- **VehicleREgistration 服务** 是一个 ASP.NET Core Web API 应用程序，提供一个端点 `/vehicleinfo/{licensenumber}`。通过在 URL 中提供的汽车牌照号码，获取超速汽车和汽车所有者的信息。

图 4-2 中的序列图展示了仿真的流程。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/sample-application/sequence.png)

图 4-2 仿真流的序列图

服务之间的通讯通过直接调用其它的 API 实现。该设计可以使用，但是存在一些缺陷。

最大的问题是，如果调用链中的服务之一离线，那么整个调用链将中断。通过将解耦的服务之间的直接调用替换为异步消息可以解决该问题。异步消息通常使用消息中间件来实现，比如：RabbitMQ 或者 Azure Service Bus。

还有其它的缺陷，在流量管控系统中，汽车的状态信息是存储在内存中。当服务升级或者崩溃，而导致重新启动的时候，状态将会丢失。状态应当存储在服务之外。

## 使用 Dapr 构建块

Dapr 的一个目标就是对于微服务应用程序提供云原生的能力。该流量管控应用程序使用 Dapr 构建块来增强鲁棒性，并降低前面段落中提到的设计缺陷。图 4-3 展示了一个启用 Dapr 版本的流量管控应用程序。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/sample-application/dapr-solution.png)

图 4-3 使用 Dapr 构建块的流量管控应用该程序

1. **Service invocation** Dapr 服务调用构建块处理 FineCollectionService 与 VehicleRegistrationService 之间的请求/响应通讯。由于该调用是一个查询，用来提取必须的数据来完成操作，同步调用是可以接受的。服务调用构建块提供服务发现。FineCollection 服务不再需要知道 VehicleRegistration 服务位于何处，它也实现了在 VehicleRegistration 服务离线时的自动重试。
2. **Publish & subscrit** 发布和订阅构建块处理异步消息，用于从 TrafficControlService 发送超速违章到 FineCollectionService 服务。如果 FineCollectionService 服务暂时失效，数据将会累计在队列中，并在随后的事件重新恢复处理。RabbitMQ 作为当前的消息中间件，将消息从生产者传递给消费者。由于 Dapr 的 pub/sub 构建块抽象了消息中间件，开发人员不需要学习 RabbitMQ 客户端库的内部细节。切换到其它的消息中间件也不需要代码变更，只需要修改配置。
3. **State management** TrafficControl 服务使用状态管理构建块在服务之外的 Redis 缓存中持久化汽车状态。与 pub/sub 类似，开发人员也不需要学习 Redis 特定的 API，切换到其它数据存储也不需要修改代码。
4. **Output binding** FineCollection 服务通过电子邮件将罚单发送给超速汽车的所有者。Dapr SMTP 输出绑定抽象了基于 SMTP 协议的电子邮件传输细节。
5. **Input binding** CameraSimulation 使用 MQTT 协议将模拟的汽车消息发送给 TrafficControll 服务。它使用  .NET MQT 库发送消息到 Mosquito - 这是一个轻量级的 MQTT 中间件。TrafficControl 服务使用 Dapr 的 MQTT 绑定来订阅 MQTT 中间件，并接收消息。
6. **Secrets management** FineCollectionService 需要认证才能连接到 SMTP 服务器，而且罚款计算组件也需要一个内容的授权码。这里使用密钥管理构建块来获取认证凭据和授权码。
7. **Actors** TrafficControlService 有一个基于 Dapr 执行人的替换实现。在该实现中，TrafficControl 服务针对每个入口摄像机的企业创建一个新的执行人。汽车的牌照号码赋予 Actor ID。执行人封装了该汽车的状态。它被持久在 Redis 缓存中。当汽车被离开摄像头登记的时候，调用该执行人。执行人然后计算出平均速度，并可能开出超速罚单。

图 4-4 展示了所有 Dapr 构建块的执行序列图

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/sample-application/sequence-dapr.png#lightbox)

图 4-4 Dapr 构建块的执行序列图

本书后面的每一章将专门针对一个 Dapr 构建块。每章将深入说明该构建块是如何工作、如何配置、以及如何使用它。每章说明在流量管控应用程序中是如何使用该构建块的。

## 托管

流量管控应用程序既可以运行在自寄宿模式，也可以运行在 Kubernetes 中。

### 自寄宿模式

在示例的仓储中，包含了在你的本地机器上，以 Docker 容器的方式，启动基础服务架构的 PowerShell 脚本 (Redis, RabbitMQ, and Mosquitto)。脚本位于 `src/Infrastructure` 文件夹中。对于应用程序中的每个应用程序服务，在仓储中对应一个独立的文件夹，每个文件夹中包含一个名为 `start-selfhosted.ps1` 的 PowerShell 脚本，用来启动 Dapr 服务。

### Kubernetes 模式

示例仓储中的 `src/k8s` 文件夹中包含了在 Kubernetes 中使用 Dapr 运行应用程序的 Kubernetes 说明文件 ( 包括基础架构服务 )。 该文件夹中还包括了用来在 Kubernetes 中启动和停止解决方案的  `start.ps1` 和 `stop.ps1` PowerShell 脚本。所有的服务都运行在 `dapr-trafficcontrol` 命名空间中。

## 总结

流量管控示例应用程序时一个微服务应用程序，用来模拟高速公路的超速管理。

该应用程序使用多个 Dapr 构建块来变得更稳健和云原生。示例尽可能简化以主要关注于 Dapr。

后继的章节将使用该示例应用程序来介绍 Dapr 构建块。

## 参考

- [Dapr Traffic Control Sample](https://github.com/EdwinVW/dapr-traffic-control)

