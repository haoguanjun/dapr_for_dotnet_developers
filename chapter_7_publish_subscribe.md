# Dapr 发布 & 订阅构建块

https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/publish-subscribe

[Publish-Subscribe pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/publisher-subscriber) ( 常称为 pub/sub ) 是众所周知的广泛使用的消息模式。架构师通常在分布式应用程序中使用它。实际上，实现它的构建工作可能很复杂。不同的消息中间件产品之间通常存在细微的功能差异。Dapr 提供了构建块来显著简化发布/订阅功能的实现。

## 7.1 解决何种问题

发布/订阅模式主要的优势在于 **松耦合**，有时也称为 [时间解藕](https://docs.microsoft.com/en-us/azure/architecture/guide/technology-choices/messaging#decoupling)。该模式将服务解藕为消息发布方 ( publishers ) 和订阅方 ( subscribers )。发布方和订阅方都对对方无感 - 双方都依赖于中心化的 **消息代理** 来分发消息。

图 7-1 展示了 pub/sub 模式的高级概念架构

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/publish-subscribe/pub-sub-pattern.png)

图 7-1 pub/sub 架构

从前面的图中，注意到模式的步骤：
1. 发布方发送消息到消息代理
2. 订阅方通过订阅绑定到消息代理
3. 消息代理转发感兴趣的消息副本
4. 订阅方通过订阅消费消息

多数的消息代理封装队列机制，一旦接收到消息可以进行持久化。使用该机制，消息中间人通过存储消息来实现持久性。当发布方发布消息的时候，订阅方不需要立即可用，甚至不需要在线。一旦订阅方可用，订阅方接收并处理消息。对于消息分发，Dapr 保证 **At-Least-One** 语义。一旦消息发布，它将至少分发一次到任何感兴趣的订阅者。

> 提示
>
> 如果你的服务只能处理一条消息。你将需要提供一个 **幂等检查** 来确保相同的消息不会处理多次。虽然此类逻辑可以编码，有些消息中间人，例如 Azure Service Bus，提供了内置的 _duplicate detection_ 消息能力。

有多种消息中间人产品可用 - 包括商业和开源产品。每种有自己的优势和缺点。你的工作是匹配你的系统需求到适当的消息中间人。一旦选择，它就是将你的应用程序从消息中间件管道中解藕的最佳实践。你通过封装消息中间件成为内部的抽象来达到此功能。该抽象封装消息管道并为你的代码暴露通用的 pub/sub 操作。你的代码通过抽象进行通讯，而不是实际的消息中间件。通过聪明的决策，你将会编写和维护抽象以及它的底层实现。这种方式要求定制代码，它可能是复杂的、乏味的、可能包含了错误。

而 Dapr 的发布 & 订阅构建块提供了开箱即用的消息抽象和实现。你不得不编写的代码已经预先构建和封装在 Dapr 构建块中。你只需要绑定到它，消费它即可。而不是编写消息管道的代码。你和你的团队可以专注于业务功能来增加你的客户的价值。

## 7.2 它如何工作

Dapr 发布 & 订阅构建块提供平台无关的 API 框架来支持发送和接收消息。你的服务发布消息到命名的 topic 上。你的服务订阅到 topic 上来消费消息。

服务通过 Dapr sidecar 来调用 pub/sub API。sidecar 然后调用预先定义的 Dapr pub/sub 组件，其中封装了特定的消息中间件产品。图 7-2 展示了 Dapr pub/sub 消息栈。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/publish-subscribe/pub-sub-buildingblock.png)

图 7-2 Dapr pub/sub 栈

Dapr 发布 & 订阅构建块可以通过多种方式调用。

在最低级别，任何可以通过 HTTP 或者 gRPC 的编程平台使用 Dapr native API 调用。为了发布消息，可以使用如下的 API 调用。

```http
http://localhost:<dapr-port>/v1.0/publish/<pub-sub-name>/<topic>
```

在上面的调用中有多个 Dapr 特定的 URL 片断：
* <dapr-port> Dapr 监听的端口号
* <pub-sub-name> Dapr pub/sub 组件提供的名称
* <topic> 消息发布到的主题名称

使用下面的 curl 工具发布消息，你可以试用下面的命令

```bash
curl -X POST http://localhost:3500/v1.0/publish/pubsub/newOrder \
  -H "Content-Type: application/json" \
  -d '{ "orderId": "1234", "productId": "5678", "amount": 2 }'
```

通过订阅到主题来接收消息。在应用程序启动的时候，Dapr 运行时将调用应用程序众所周知的端点来识别并创建必要的订阅：

```http
http://localhost:<appPort>/dapr/subscribe
```

* \<appPort> 通知 Dapr sidecar  应用程序监听的端口

你可以自己实现该端点。但是 Dapr 提供了更符合直觉的实现方式。我们将在本章的后面处理该功能。

调用的响应包含应用程序可以订阅的主题的列表。每个主题包含一个当主题收到消息时用来调用的端点。下面是一个响应的示例：

```json
[
  {
    "pubsubname": "pubsub",
    "topic": "newOrder",
    "route": "/orders"
  },
  {
    "pubsubname": "pubsub",
    "topic": "newProduct",
    "route": "/productCatalog/products"
  }
]
```

在这个 JSON 响应中，你可以看到应用程序希望订阅的主题 `newOrder` 和 `newProduct`。它分别注册了端点 `/orders` 和 `/productCatalog/products`，对于这两个订阅，应用程序绑定到名为 `pubsub` 的 Dapr 组件。

图 7-3 提供了示例的流程

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/publish-subscribe/pub-sub-flow.png)

图 7-3 Dapr 的 Pub/Sub 流程

从上图中可以看到：

1. 服务 B ( 消费方 ) 调用的自己的 Dapr sidecar `/dapr/subscribe` 端点。服务响应它希望创建的订阅
2. 服务 B 的 Dapr sidecar 通过消息代理创建请求的订阅
3. 服务 A 在 Dapr 的服务 A sidecar `/v1.0/publish/<pub-sub-name>/<topic>` 端点发布消息
4. 服务 A sidecar 发布消息到消息代理
5. 消息代理将消息副本发送到服务 B 的 sidecar
6. 服务 B 的 sidecar 调用服务 B 订阅服务相关的端点 ( 此例中为 `/orders` )。服务响应 HTTP 状态码 200 OK，这样 sidecar 认为此消息已经被成功处理。

在此示例中，消息被成功处理。但是如果在服务 B 处理过程中出现某种问题，它可以使用响应来指出对于此消息发生的状况。当返回 HTTP 状态码 404 的时候，错误被记录，消息将被丢弃。对于 200 和 404 之外的状态码，将记录警告信息，消息将被重试。另外，服务 B 可以通过在响应体中包含 JSON 内容来显式指出发生的状况：

```json
{
  "status": "<status>"
}
```

下表展示了可用的状态值：

| 状态      | 操作 |
| ----------- | ----------- |
| SUCCESS      | 消息认为被成功处理并丢弃       |
| RETRY   | 消息被重试        |
| DROP   | 警告被记录，消息被丢弃        |
| 任何其它状态   | 消息被重试        |

### 7.2.1 相互竞争的消费者

当扩展订阅到某个主题的应用程序时，你不得不面对相互竞争的消费者。应该只有一个应用程序处理发布到主题的消息。幸运的是，Dapr 处理该问题。当单个服务的多个实例使用相同的 application-id 订阅一个主题时，Dapr 仅仅对其中之一分发消息。

## 7.3 使用 Dapr .NET SDK

对于 .NET 开发人员，Dapr .NET SDK 提供了更有生产力的方式来使用 Dapr。该 SDK 通过 DaprClient 类使你可以直接调用 Dapr 的功能。它直观且易于使用。

发布消息，DaprClient 暴露了 `PublishEventAsync()` 方法。

```csharp
var data = new OrderData
{
  orderId = "123456",
  productId = "67890",
  amount = 2
};

var daprClient = new DaprClientBuilder().Build();

await daprClient.PublishEventAsync<OrderData>("pubsub", "newOrder", data);
```

* 第一个参数 `pubsub` 是提供给消息代理实现的 Dapr 组件名称。我们将本章后面处理该组件。
* 第二个参数 `newOrder` 提供发送消息到的主题名称
* 第三个参数是消息的内容
* 可以通过范型指定消息的 .NET 类型

为了接收消息，通过对注册的主题绑定端点到订阅来完成。Dapr 的 AspNetCore 库使得它非常简单。假设，你有一个名为 `CreateOrder` 的 ASP.NET WebAPI 操作方法：

```csharp
[HttpPost("/orders")]
public async Task<ActionResult> CreateOrder(Order order)
```

> 重要
> 必须在项目中添加对 Dapr.AspNetCore NuGet 包的引用，以使用 Dapr 的 ASP.NET Core 集成。

绑定该方法到某个主题，通过使用 `Topic` 特性进行修饰完成：

```csharp
[Topic("pubsub", "newOrder")]
[HttpPost("/orders")]
public async Task<ActionResult> CreateOrder(Order order)
```

在该特性上，使用了两个关键元素：
* Dapr 的 pub/sub 组件目标 ( 这里是 `pubsub` )
* 订阅的主题 ( 这里是 `newOrder` )

一旦 Dapr 接收到该主题的消息，就会调用该 Action 方法。

你还需要启用 ASP.NET Core 的 Dapr 支持。Dapr .NET SDK 提供多种扩展方法可以用来完成该任务。

在 Program.cs 文件中，你必须在 `WebApplication` 的构建器上调用如下的扩展方法，以注册 
Dapr:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers().AddDapr();
```

追加在 `AddControllers()` 扩展方法之后的 `AddDapr()` 扩展方法注册必要的服务，来将 Dapr 集成到 MVC 管线中。它还注册了 `DaprClient` 实例到依赖注入容器中，它可以被注入到任何需要的服务中。

在 `WebApplication` 创建之后，你必须添加如下的中间件组件以支持 Dapr:

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
app.UseCloudEvents();
app.MapControllers();
app.MapSubscribeHandler();
```

对 `UseCloudEvents()` 方法的调用添加 `CloudEvents` 中间件到 ASP.NET Core 的中间件处理管道中。该中间件将使用 CloudEvents 结构格式来拆出请求，所以接收方法可以直接读取事件的内容。

> 注意
> [CloudEvents](https://cloudevents.io/) 是标准化的消息格式。提供通用的方式来描述跨平台的事件信息。Dapr 拥抱 CloudEvents。更多关于 CloudEvents 内容，请参考 cloudevents 规范。

在路由配置中对 `MapSubscribeHandler()` 的调用将增加 Dapr 订阅端点到应用程序中。该端点将负责对 `/dapr/subscribe` 的请求。当此端点被调用时，它将自动发现所有的装饰来 `[topic]` 特性的 WebAPI action 方法，并指导 Dapr 创建对它们的订阅。

## 7.4 Pub/sub 组件

Dapr pub/sub 组件处理实际的消息传输。有多种方案可用。每种封装了特定的消息代理产品来实现 pub/sub 功能。在本文撰写的时候，下述 pub/sub 组件可用：
* Apache Kafka
* Azure Event Hubs
* Azure Service Bus
* AWS SNS/SQS
* GCP Pub/Sub
* Hazelcast
* MQTT
* NATS
* Pulsar
* RabbitMQ
* Redis Streams

> 注意
> Azure cloud 技术栈有两种：消息功能 ( Azure Service Bus ) 和事件流 ( Azure Event Hub ) 可用。

有些社区创建的组件在 [components-contrib repository on GitHub](https://github.com/dapr/components-contrib/tree/master/pubsub)。如果它还不存在的话，鼓励你编写自己的消息代理 Dapr 组件。

### 7.4.1 配置

使用 Dapr 配置文件，你可以指定使用的 pub/sub 组件。该配置包含多个字段。其中 `name` 字段指定 你希望使用的 pub/sub 组件的名称。当发送或者接收消息的时候，你可以指定该名称 ( 如前面在 `PublishEventasync()` 方法中所见 )。

下面是一个 Dapr 配置的示例，配置使用 RabbitMQ 消息代理组件：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub-rq
spec:
  type: pubsub.rabbitmq
  version: v1
  metadata:
  - name: host
    value: "amqp://localhost:5672"
  - name: durable
    value: true
```

在该示例中，你可以看到可以在 `metadata` 块中可以指定任何特定的消息代理配置。在本示例中，配置了RabbitMQ 的持久队列。但是该 RabbitMQ 组件可以有更多配置选项。每种组件配置拥有自己特定的配置字段。可以阅读每种 pub/sub 组件的文件来了解何种字段可用。

下面使用编程方式通过代码来订阅主题。Dapr pub/sub 还提供了定义方式来订阅主题。该方式从应用程序代码中移除了对 Dapr 的依赖。进而，它还支持现存的应用程序订阅主题而不需要修改代码。下面的示例展示了用于配置订阅的 Dapr 配置文件。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: newOrder-subscription
spec:
  pubsubname: pubsub
  topic: newOrder
  route: /orders
scopes:
- ServiceB
- ServiceC
```

你需要针对每个订阅指定多个元素：
* 希望使用的 Dapr pub/sub 组件名称 ( 这里是 `pubsub` )
* 订阅的主题名称 ( 此例中为 `newOrder` )
* 此主题需要调用的 API 操作 ( 此例中为 `/orders` )
* `scope` 可以哪些服务可以发布和订阅到主题

## 7.5 示例应用：Dapr 交通管理

在 Dapr 交通管理示例应用中，TrafficControl 服务使用 Dapr 的 pub/sub 构建块来发送超速信息给 FineCollection 服务。图 7-4 展示了 Dapr 交通管理示例应用的概念架构。Dapr pub/sub 构建块被用于标记为数字 2 的部分：

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/publish-subscribe/dapr-solution-pub-sub.png)

图 7-4 Dapr 交通管理示例应用的概念架构

超速由 `CollectionController` 处理，它是一个普通的 ASP.NET Core 控制器。`CollectionController.CollectFine` 方法订阅并处理 `SpeedingViolation` 事件消息：

```csharp
[Topic("pubsub", "speedingviolations")]
[Route("collectfine")]
[HttpPost]
public async Task<ActionResult> CollectFine(
    SpeedingViolation speedingViolation, [FromServices] DaprClient daprClient)
{
    // ...
}
```

该方法使用 Dapr 的 `[Topic]` 特性装饰。指定使用名为 `pubsub` 的 pub/sub 组件将用来订阅到发送到 `speedingviolations` 主题的消息。

TrafficControl 服务发送超速消息。在 `TrafficController` 类中 `VehicleExit` 方法靠后的位置，`DaprClient` 对象使用 pub/sub 构建块用来发布 `SpeedingViolation` 消息

```csharp
/// ...

var speedingViolation = new SpeedingViolation
{
    VehicleId = msg.LicenseNumber,
    RoadId = _roadId,
    ViolationInKmh = violation,
    Timestamp = msg.Timestamp
};

// publish speedingviolation (Dapr publish / subscribe)
await daprClient.PublishEventAsync("pubsub", "speedingviolations", speedingViolation);

/// ...
```

注意 `DaprClient` 对象是如何精简到单行代码调用的。重复一下，绑定到 `speedingviolations` 主题和 `pubsub` Dapr 组件。

尽管交通管理应用使用 RabbitMQ 作为消息代理，它永远有不会直接引用 RabbitMQ。相反，在 `/dapr/components` 文件夹中使用的 Dapr 组件配置文件名为 `pubsub.yaml`，其中设置了消息代理：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: dapr-trafficcontrol
spec:
  type: pubsub.rabbitmq
  version: v1
  metadata:
  - name: host
    value: "amqp://localhost:5672"
  - name: durable
    value: "false"
  - name: deletedWhenUnused
    value: "false"
  - name: autoAck
    value: "false"
  - name: reconnectWait
    value: "0"
  - name: concurrency
    value: parallel
scopes:
  - trafficcontrolservice
  - finecollectionservice
```

配置中的 `type` 元素的值 `pubsub.rabbitmq` 定义了构建块使用 Dapr RabbitMQ 组件。

配置中的 `scopes` 元素 _包含_ 访问到 RabbitMQ 组件的应用。只有 TrafficControl 和 FineCollection 服务可以使用。

在交通管理示例应用中使用 Dapr pub/sub 提供了如下的优势：
1. 不需要维护消息代理抽象
2. 服务被解藕，增加了鲁棒性
3. 发布方和订阅方对彼此无感。这意味着附加的服务可以在未来被引入超速处理中，而不需要改变 TrafficControl 服务。

## 7.6 总结

发布/订阅模式有助于解藕在分布式应用程序中的服务。Dapr 发布/订阅构建块简化了在你的应用程序中实现该行为。

通过 Dapr pub/sub，你可以发布消息到指定的主题。同时，构建块将查询你的服务来决定订阅何种主题。

你可以直接基于 HTTP 使用 Dapr pub/sub，或者使用某种语言特定的 SDK ，例如 .NET SDK for Dapr 等等。.NET SDK 集成到 ASP.NET Core 平台。

使用 Dapr，你可以将受到支持的消息代理插入到你的应用程序中。你可以切换消息代理而不需要修改你的应用程序的代码。

