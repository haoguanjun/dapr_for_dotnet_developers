# 第 9 章 Dapr 执行组件构建块

https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/actors

执行组件模型源自 1973 年。 这是由 Carl Hewitt 的建议，作为并发计算的概念模型，这种形式的计算会同时执行多个计算。 高度并行的计算机当时还没有出现，但多核 Cpu 和分布式系统的最新进步使得执行组件模型成为了最新的模型。

在执行组件模型中，_执行_ 组件是计算和状态的独立单元。 参与者完全彼此隔离，它们永远不会共享内存。 参与者使用消息相互通信。 当执行组件收到消息时，它可以更改其内部状态，并将消息发送到其他 (可能是新的) 执行组件。

执行组件模型使编写并发系统变得更简单的原因是，它提供了基于轮换的 (或单线程) 访问模型。 多个执行组件可以同时运行，但每个执行组件一次只处理一个接收的消息。 这意味着，在任何时候，都可以确保在执行组件中最多有一个线程处于活动状态。 这使得编写正确的并发系统和并行系统变得更加容易。

## 9.1 解决何种问题

执行组件模型实现通常绑定到特定语言或平台。 不过，使用 Dapr 执行组件构建块可以从任何语言或平台利用执行组件模型。

Dapr 的实现基于 [virtual actor pattern introduced by Project "Orleans"](https://www.microsoft.com/research/project/orleans-virtual-actors/)。 对于虚拟执行组件模式，无需显式创建参与者。 第一次将消息发送到执行组件时，参与者将被隐式激活并放置在群集中的节点上。 当不执行操作时，执行组件会以静默方式从内存中卸载。 如果某个节点出现故障，Dapr 会自动将激活的执行组件移到正常的节点。 除了在参与者之间发送消息以外，Dapr 执行组件模型还支持使用计时器和提醒计划将来的工作。

虽然执行组件模型可以提供很大的优势，但务必仔细考虑执行组件设计。 例如，如果多个客户端调用相同的执行组件，则会导致性能不佳，因为执行组件操作会按顺序执行。 下面是检查方案是否适用于 Dapr 参与者的一些标准：

* 问题空间涉及并发性。 如果没有执行组件，则需要在代码中引入显式锁定机制。
* 可以将问题空间分区为小、独立和隔离的状态和逻辑单元。
* 不需要对执行组件状态进行低延迟的读取。 不能保证低延迟读取，因为执行组件操作按顺序执行。
* 不需要在一组参与者之间查询状态。 跨执行组件的查询效率低下，因为每个执行组件的状态都需要单独读取，并且可能会导致不可预测的延迟。

满足这些条件的一种设计模式非常好，就是 [orchestration-based saga](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga) 或 流程管理器 设计模式。 Saga 管理必须执行的一系列步骤才能达到某些结果。 Saga (或进程管理器) 维护序列的当前状态，并触发下一步。 如果步骤失败，saga 可以执行补偿操作。 利用参与者，可以轻松处理 saga 中的并发，并跟踪当前状态。 EShopOnDapr 参考应用程序使用 saga 模式和 Dapr 执行组件来实现排序过程。

## 9.2 它是如何工作的

Dapr sidecar 提供了 HTTP/gRPC API 来调用执行者。这是基础的 HTTP API 的 URL：

```http
http://localhost:<daprPort>/v1.0/actors/<actorType>/<actorId>/
```
 
* \<daprPort>： Dapr 侦听的 HTTP 端口。
* \<actorType>：执行组件类型。
* \<actorId>：要调用的特定参与者的 ID。

Sidecar 管理每个执行组件的运行时间和位置，以及在参与者之间路由消息的方式。 如果一段时间未使用某个执行组件，则运行时将停用该执行组件，并将其从内存中删除。 执行组件管理的任何状态都将被保留，并在执行组件重新激活时可用。 Dapr 使用空闲计时器来确定何时可以停用参与者。 当在执行组件上调用操作时 (通过方法调用或提醒触发) ，会重置空闲计时器，并保持激活执行组件实例。

Sidecar API 只是公式的一部分。 服务本身还需要实现 API 规范，因为你为参与者编写的实际代码将在服务本身内运行。 图 11-1 显示了服务和它的挎斗之间的各种 API 调用：

![](https://docs.microsoft.com/zh-cn/dotnet/architecture/dapr-for-net-developers/media/actors/sidecar-communication.png)

图 11-1 执行组件服务和 Dapr sidecar 之间的 API 调用

为了提供可伸缩性和可靠性，将在执行组件服务的所有实例中对参与者进行分区。 Dapr 放置服务负责跟踪分区信息。 启动执行组件服务的新实例时，挎斗会将支持的执行组件类型注册到放置服务。 放置服务计算给定参与者类型的更新分区信息，并将其广播给所有实例。 图11-2 显示了将服务扩展到第二个副本时发生的情况：

![](https://docs.microsoft.com/zh-cn/dotnet/architecture/dapr-for-net-developers/media/actors/placement.png)

图 11-2 执行组件放置服务

1. 启动时，Sidecar 调用执行组件服务以获取注册的执行组件类型和执行组件的配置设置。
2. Sidecar 将注册的执行组件类型的列表发送到放置服务。
3. 放置服务会将更新的分区信息广播到所有执行组件服务实例。 每个实例都将保留分区信息的缓存副本，并使用它来调用执行组件。

> 重要
> 由于执行组件是在各服务实例间随机分发的，因此应应执行组件操作始终需要调用网络中的其他节点。

下图显示了在 Pod 1 中运行的排序服务实例调用 ship OrderActor ID 为的实例的方法 3 。 由于 ID 的执行组件 3 放在不同的实例中，因此将导致调用群集中的不同节点：

![](https://docs.microsoft.com/zh-cn/dotnet/architecture/dapr-for-net-developers/media/actors/invoke-actor-method.png)

图 11-3 调用执行组件的方法

1. 服务在 Sidecar 上调用执行组件 API。 请求正文中的 JSON 有效负载包含要发送到执行组件的数据。
2. Sidecar 使用位置服务中的本地缓存的分区信息来确定哪个执行组件服务实例 (分区，) 负责托管 ID 为的执行组件 3 。 在此示例中，它是 pod 2 中的服务实例。 调用将转发到相应的 Sidecar。
3. Pod 2 中的 Sidecar 实例调用服务实例以调用执行组件。 如果执行组件尚未) 并执行执行组件方法，则该服务实例将激活该执行组件

## 基于轮转的访问模型

基于轮转的访问模型可确保在一个执行组件实例内最多只有一个线程处于活动状态。 若要了解此操作的原因，请考虑以下用于递增计数器值的方法示例：

```csharp
public int Increment()
{
    var currentValue = GetValue();
    var newValue = currentValue + 1;

    SaveValue(newValue);

    return newValue;
}
```

假设方法返回的当前值 GetValue 为 1 。 当两个线程同时调用方法时，它们会在调用方法 Increment GetValue 之前调用方法 SaveValue 。 这会导致两个线程以相同初始值开始 (1) 。 然后，线程递增值并将 2 其返回给调用方。 现在，两次调用后的结果值是， 2 而不是它的值 3 。 这是一个简单的示例，说明了在使用多个线程时可能会滑入代码的问题种类，并且很容易解决。 但在实际应用程序中，并发和并行方案可能会变得非常复杂。

在传统编程模型中，可以通过引入锁定机制来解决此问题。 例如：

```csharp
public int Increment()
{
    int newValue;

    lock (_lockObject)
    {
        var currentValue = GetValue();
        newValue = currentValue + 1;

        SaveValue(newValue);
    }

    return newValue;
}
```

遗憾的是，使用显式锁定机制容易出错。 它们很容易导致死锁，并可能对性能产生严重影响。

由于使用的是基于 轮转 的访问模型，因此，您无需担心使用参与者的多个线程，使编写并发系统变得更加容易。 下面的执行组件示例将对上一个示例中的代码进行密切镜像，但不需要任何锁定机制是正确的：

```csharp
public async Task<int> IncrementAsync()
{
    var counterValue = await StateManager.TryGetStateAsync<int>("counter");

    var currentValue = counterValue.HasValue ? counterValue.Value : 0;
    var newValue = currentValue + 1;

    await StateManager.SetStateAsync("counter", newValue);

    return newValue;
}
```

## 计时器和提醒

参与者可以使用计时器和提醒来计划自身的调用。 这两个概念都支持配置截止时间。 不同之处在于回调注册的生存期：

* 只要激活执行组件，计时器就会保持活动状态。 计时器 不会 重置空闲计时器，因此它们不能使执行组件处于活动状态。
* 提醒长于执行组件激活。 如果停用了某个执行组件，则会重新激活该执行组件。 提醒 将 重置空闲计时器。

计时器是通过调用执行组件 API 来注册的。 在下面的示例中，在时间为0的情况下注册计时器，时间为10秒。

```bash
curl -X POST http://localhost:3500/v1.0/actors/<actorType>/<actorId>/timers/<name> \
  -H "Content-Type: application/json" \
  -d '{
        "dueTime": "0h0m0s0ms",
        "period": "0h0m10s0ms"
      }'
```

由于截止时间为0，因此将立即触发计时器。 计时器回调完成后，计时器将等待10秒，然后再次触发。

提醒注册方式类似。 下面的示例演示了一个提醒注册，该注册的截止时间为5分钟，空时间为空：

```bash
curl -X POST http://localhost:3500/v1.0/actors/<actorType>/<actorId>/reminders/<name> \
  -H "Content-Type: application/json" \
  -d '{
        "dueTime": "0h5m0s0ms",
        "period": ""
      }'
```

此提醒将在5分钟后激发。 由于给定时间段为空，这将为一次性提醒。

> 备注
> 计时器和提醒均遵循基于 轮转 的访问模型。 当计时器或提醒触发时，直到任何其他方法调用或计时器/提醒回调完成后才会执行回调。

## 状态持久性

使用 Dapr 状态管理构建块保存执行组件状态。 由于执行组件可以一轮执行多个状态操作，因此状态存储组件必须支持多项事务。 撰写本文时，以下状态存储支持多项事务：

* Azure Cosmos DB
* MongoDB
* MySQL
* PostgreSQL
* Redis
* RethinkDB
* SQL Server

若要配置要与执行组件一起使用的状态存储组件，需要将以下元数据附加到状态存储配置：

```yaml
- name: actorStateStore
  value: "true"
```

下面是 Redis 状态存储的完整示例：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    value: ""
  - name: actorStateStore
    value: "true"
```

## 使用 Dapr .NET SDK

只能使用 HTTP/gRPC 调用创建执行组件模型实现。 但是，更方便的方法是使用特定于语言的 Dapr Sdk。 撰写本文时，.NET、Java 和 Python Sdk 都为使用参与者提供了广泛的支持。

若要开始使用 .NET Dapr 执行组件 SDK，你可以将对的包引用添加到 Dapr.Actors 你的服务项目。 创建实际参与者的第一步是定义从派生的接口 IActor 。 客户端使用接口调用执行组件上的操作。 下面是一个简单的参与者界面示例，用于保持分数：

```csharp
public interface IScoreActor : IActor
{
    Task<int> IncrementScoreAsync();

    Task<int> GetScoreAsync();
}
```

> 重要
> 执行组件方法的返回类型必须为 Task 或 Task\<T> 。 此外，执行组件方法最多只能有一个参数。 返回类型和参数都必须可 System.Text.Json 序列化。

接下来，通过从派生类来实现参与者 ScoreActor Actor 。 ScoreActor类还必须实现 IScoreActor 接口：

```csharp
public class ScoreActor : Actor, IScoreActor
{
    public ScoreActor(ActorHost host) : base(host)
    {
    }

    // TODO Implement interface methods.
}
```

上面代码段中的构造函数采用 host 类型的参数 ActorHost 。 ActorHost类表示执行组件运行时中的参与者类型的宿主。 需要将此参数传递给基类的构造函数 Actor 。 参与者还支持依赖项注入。 使用 .NET 依赖关系注入容器来解析添加到执行组件构造函数的任何其他参数。

现在，让我们实现 IncrementScoreAsync 接口的方法：

```csharp
public Task<int> IncrementScoreAsync()
{
    return StateManager.AddOrUpdateStateAsync(
        "score",
        1,
        (key, currentScore) => currentScore + 1
    );
}
```

在上面的代码片段中，对方法的一次调用 StateManager.AddOrUpdateStateAsync 提供了完整的 IncrementScoreAsync 方法实现。 AddOrUpdateStateAsync方法采用三个参数：

要更新的状态的键。
如果尚未将评分存储在状态存储中，则为要写入的值。
在 Func 状态存储中已有分数存储时要调用的。 它将使用状态键和当前评分，并返回更新后的分数以写回到状态存储区。
GetScoreAsync实现读取状态存储中的当前评分，并将其返回给客户端：

```csharp
public async Task<int> GetScoreAsync()
{
    var scoreValue = await StateManager.TryGetStateAsync<int>("score");
    if (scoreValue.HasValue)
    {
        return scoreValue.Value;
    }

    return 0;
}
```

若要在 ASP.NET Core 服务中承载执行组件，您必须添加对包的引用 Dapr.Actors.AspNetCore 并在文件中进行一些更改 Program 。 在下面的示例中，调用 MapActorsHandlers 在 ASP.NET Core 路由中注册 Dapr 参与者终结点：

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
// Actors building block does not support HTTPS redirection.
//app.UseHttpsRedirection();
app.MapControllers();
// Add actor endpoints.
app.MapActorsHandlers();
```

执行组件终结点是必需的，因为 Dapr 挎斗调用应用程序来承载和与执行组件实例进行交互。

> 重要
> 请确保你的 Startup 类不包含 app.UseHttpsRedirection 将客户端重定向到 HTTPS 终结点的调用。 这不适用于执行组件。 按照设计，默认情况下，Dapr 挎斗通过未加密的 HTTP 发送请求。 启用后，HTTPS 中间件会阻止这些请求。

Program文件也是注册特定执行组件类型的位置。 下面的示例 ScoreActor 使用 AddActors 扩展方法注册：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddActors(options =>
{
    options.Actors.RegisterActor<ScoreActor>();
});
```

此时，ASP.NET Core 服务已准备好承载 ScoreActor 和接受传入的请求。 客户端应用程序使用执行组件代理来调用执行组件上的操作。 下面的示例演示了控制台客户端应用程序如何 IncrementScoreAsync 对实例调用操作 ScoreActor ：

```csharp
var actorId = new ActorId("scoreActor1");

var proxy = ActorProxy.Create<IScoreActor>(actorId, "ScoreActor");

var score = await proxy.IncrementScoreAsync();

Console.WriteLine($"Current score: {score}");
```

上面的示例使用 Dapr.Actors 包来调用执行组件服务。 若要在执行组件上调用操作，需要能够解决该操作。 此操作需要两个部分：

1. 执行组件 类型 在整个应用程序中唯一标识执行组件实现。 默认情况下，执行组件类型是 (没有命名空间) 的实现类的名称。 可以通过将添加 ActorAttribute 到实现类并设置其属性，自定义参与者类型 TypeName 。
2. ActorId 唯一标识执行组件类型的实例。 还可以通过调用来使用此类生成随机执行组件 id ActorId.CreateRandom 。

该示例使用 ActorProxy.Create 为创建代理实例 ScoreActor 。 Create方法采用两个参数：标识特定执行组件和执行组件 ActorId 类型。 它还具有一个泛型类型参数，用于指定执行组件类型所实现的执行组件接口。 由于服务器和客户端应用程序都需要使用执行组件接口，它们通常存储在单独的共享项目中。

该示例中的最后一个步骤调用 IncrementScoreAsync 参与者上的方法并输出结果。 请记住，Dapr 放置服务跨 Dapr 分支分发执行组件实例。 因此，需要将执行组件调用作为对另一个节点的网络调用。

### 从 ASP.NET Core 客户端调用参与者

上一部分中的控制台客户端示例直接使用静态 ActorProxy.Create 方法获取执行组件代理实例。 如果客户端应用程序是 ASP.NET Core 应用程序，则应使用 IActorProxyFactory 接口创建执行组件代理。 主要优点是它允许您在一个位置管理配置。 AddActors上的扩展方法 IServiceCollection 带有一个委托，该委托使你可以指定执行组件运行时选项，如 Dapr 挎斗的 HTTP 终结点。 下面的示例指定了用于执行组件 JsonSerializerOptions 状态持久性和消息反序列化的自定义：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddActors(options =>
{
    var jsonSerializerOptions = new JsonSerializerOptions()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        PropertyNameCaseInsensitive = true
    };

    options.JsonSerializerOptions = jsonSerializerOptions;
    options.Actors.RegisterActor<ScoreActor>();
});
```

为 AddActors 注册 IActorProxyFactory .net 依赖项注入而调用的。 这允许 ASP.NET Core 将实例注入 IActorProxyFactory 控制器类。 下面的示例从 ASP.NET Core 控制器类调用执行组件方法：

```csharp
[ApiController]
[Route("[controller]")]
public class ScoreController : ControllerBase
{
    private readonly IActorProxyFactory _actorProxyFactory;

    public ScoreController(IActorProxyFactory actorProxyFactory)
    {
        _actorProxyFactory = actorProxyFactory;
    }

    [HttpPut("{scoreId}")]
    public Task<int> IncrementAsync(string scoreId)
    {
        var scoreActor = _actorProxyFactory.CreateActorProxy<IScoreActor>(
            new ActorId(scoreId),
            "ScoreActor");

        return scoreActor.IncrementScoreAsync();
    }
}
```

参与者还可以直接调用其他参与者。 Actor基类 IActorProxyFactory 通过属性公开类 ProxyFactory 。 若要从执行组件中创建执行组件代理，请使用 ProxyFactory 基类的属性 Actor 。 下面的示例演示一个 OrderActor ，它对两个其他参与者调用操作：

```csharp
public class OrderActor : Actor, IOrderActor
{
    public OrderActor(ActorHost host) : base(host)
    {
    }

    public async Task ProcessOrderAsync(Order order)
    {
        var stockActor = ProxyFactory.CreateActorProxy<IStockActor>(
            new ActorId(order.OrderNumber),
            "StockActor");

        await stockActor.ReserveStockAsync(order.OrderLines);

        var paymentActor = ProxyFactory.CreateActorProxy<IPaymentActor>(
            new ActorId(order.OrderNumber),
            "PaymentActor");

        await paymentActor.ProcessPaymentAsync(order.PaymentDetails);
    }
}
```

> 备注
> 默认情况下，Dapr 执行组件不可重入。 这意味着不能在同一链中多次调用 Dapr 执行组件。 例如， Actor A -> Actor B -> Actor A 不允许使用调用链。 撰写本文时，有一个预览功能可用于支持重入。 但是，尚无 SDK 支持。 有关更多详细信息，请参阅 官方文档。

### 调用非 .NET 执行组件

到目前为止，这些示例使用基于 .NET 接口的强类型执行组件代理来说明执行组件调用。 当执行组件主机和客户端都是 .NET 应用程序时，这非常有效。 但是，如果执行组件主机不是 .NET 应用程序，则没有创建强类型代理的执行组件接口。 在这些情况下，可以使用弱类型代理。

创建弱类型代理的方式与强类型代理类似。 需要将执行组件方法名称作为字符串传递，而不是依赖于 .NET 接口。

```csharp
[HttpPut("{scoreId}")]
public Task<int> IncrementAsync(string scoreId)
{
    var scoreActor = _actorProxyFactory.CreateActorProxy(
        new ActorId(scoreId),
        "ScoreActor");

    return scoreActor("IncrementScoreAsync");
}
```

### 计时器和提醒

使用 RegisterTimerAsync 基类的 Actor 方法计划执行组件计时器。 在下面的示例中， TimerActor 公开 StartTimerAsync 方法。 客户端可以调用 方法来启动一个计时器，该计时器将给定的文本重复写入日志输出。

```csharp
public class TimerActor : Actor, ITimerActor
{
    public TimerActor(ActorHost host) : base(host)
    {
    }

    public Task StartTimerAsync(string name, string text)
    {
        return RegisterTimerAsync(
            name,
            nameof(TimerCallback),
            Encoding.UTF8.GetBytes(text),
            TimeSpan.Zero,
            TimeSpan.FromSeconds(3));
    }

    public Task TimerCallbackAsync(byte[] state)
    {
        var text = Encoding.UTF8.GetString(state);

        Logger.LogInformation($"Timer fired: {text}");

        return Task.CompletedTask;
    }
}
```

StartTimerAsync方法调用 RegisterTimerAsync 来计划计时器。 RegisterTimerAsync 采用五个参数：

1. 计时器的名称。
2. 触发计时器时要调用的方法的名称。
3. 要传递给回调方法的状态。
4. 首次调用回调方法之前要等待的时间。
5. 回调方法调用之间的时间间隔。 可以指定 以 TimeSpan.FromMilliseconds(-1) 禁用定期信号。

`TimerCallbackAsync` 方法以二进制形式接收用户状态。 在示例中，回调在将状态写入日志之前将状态 string 解码回 。

可以通过调用 来停止计时器 `UnregisterTimerAsync` ：

```csharp
public class TimerActor : Actor, ITimerActor
{
    // ...

    public Task StopTimerAsync(string name)
    {
        return UnregisterTimerAsync(name);
    }
}
```

请记住，计时器不会重置执行组件空闲计时器。 当执行组件上未进行其他调用时，可能会停用该执行组件，并且计时器将自动停止。 若要计划重置空闲计时器的工作，请使用我们接下来将查看的提醒。

若要在执行组件中使用提醒，执行组件类必须实现 IRemindable 接口：

```csharp
public interface IRemindable
{
    Task ReceiveReminderAsync(
        string reminderName, byte[] state,
        TimeSpan dueTime, TimeSpan period);
}
```

ReceiveReminderAsync触发提醒时调用 方法。 它采用 4 个参数：

1. 提醒的名称。
2. 注册期间提供的用户状态。
3. 注册期间提供的调用到期时间。
4. 注册期间提供的调用周期。

若要注册提醒，请使用 RegisterReminderAsync 执行组件基类的 方法。 以下示例设置一个提醒，以在到期时间为 3 分钟时发送一次。

```csharp
public class ReminderActor : Actor, IReminderActor, IRemindable
{
    public ReminderActor(ActorHost host) : base(host)
    {
    }

    public Task SetReminderAsync(string text)
    {
        return RegisterReminderAsync(
            "DoNotForget",
            Encoding.UTF8.GetBytes(text),
            TimeSpan.FromSeconds(3),
            TimeSpan.FromMilliseconds(-1));
    }

    public Task ReceiveReminderAsync(
        string reminderName, byte[] state,
        TimeSpan dueTime, TimeSpan period)
    {
        if (reminderName == "DoNotForget")
        {
            var text = Encoding.UTF8.GetString(state);

            Logger.LogInformation($"Don't forget: {text}");
        }

        return Task.CompletedTask;
    }
}
```

RegisterReminderAsync方法类似于 RegisterTimerAsync ，但不必显式指定回调方法。 如上面的示例所示，实现 IRemindable.ReceiveReminderAsync 以处理触发的提醒。

提醒同时重置空闲计时器和持久性。 即使执行组件已停用，也会在触发提醒时重新激活。 若要停止触发提醒，请调用 UnregisterReminderAsync 。

## 示例应用程序：Dapr 流量控制

Dapr 流量控制的默认版本不使用执行组件模型。 但是，它确实包含可以启用的 TrafficControl 服务的基于执行组件的替代实现。 若要使用 TrafficControl 服务中的执行组件，请打开 文件并取消注释文件 src/TrafficControlService/Controllers/TrafficController.cs 顶部的 USE_ACTORMODEL 语句：

```csharp
#define USE_ACTORMODEL
```

启用执行组件模型后，应用程序将使用执行组件来表示车辆。 可在车辆执行组件上调用的操作在接口中 IVehicleActor 定义：

```csharp
public interface IVehicleActor : IActor
{
    Task RegisterEntryAsync(VehicleRegistered msg);
    Task RegisterExitAsync(VehicleRegistered msg);
}
```

首次在 (检测到) 车辆时，模拟入口相机会调用 RegisterEntryAsync 方法。 此方法的唯一责任是在执行组件状态中存储条目时间戳：

```csharp
var vehicleState = new VehicleState
{
    LicenseNumber = msg.LicenseNumber,
    EntryTimestamp = msg.Timestamp
};
await StateManager.SetStateAsync("VehicleState", vehicleState);
```

当车辆到达速度相机区域末尾时，退出摄像头调用 RegisterExitAsync 方法。 RegisterExitAsync方法首先获取当前状态并更新它以包括退出时间戳：

```csharp
var vehicleState = await StateManager.GetStateAsync<VehicleState>("VehicleState");
vehicleState.ExitTimestamp = msg.Timestamp;
```

> 备注
> 上面的代码当前假定 实例 VehicleState 已由 方法 RegisterEntryAsync 保存。 可以通过首先检查 以确保状态存在来改进代码。 得益于基于轮次的访问模型，代码中不需要显式锁。

状态更新后， RegisterExitAsync 方法会检查车辆是否驾驶速度过快。 如果是，则执行组件将消息发布到 collectfine pub/sub 主题：

```csharp
int violation = _speedingViolationCalculator.DetermineSpeedingViolationInKmh(
    vehicleState.EntryTimestamp, vehicleState.ExitTimestamp);

if (violation > 0)
{
    var speedingViolation = new SpeedingViolation
    {
        VehicleId = msg.LicenseNumber,
        RoadId = _roadId,
        ViolationInKmh = violation,
        Timestamp = msg.Timestamp
    };

    await _daprClient.PublishEventAsync("pubsub", "collectfine", speedingViolation);
}
```

上面的代码使用两个外部依赖项。 封装用于确定车辆是否驾驶速度过快 _speedingViolationCalculator 的业务逻辑。 允许 _daprClient 执行组件使用 Dapr pub/sub 构建基块发布消息。

这两个依赖项在 类 Startup 中注册，并且使用构造函数依赖项注入注入到执行组件中：

```csharp
private readonly DaprClient _daprClient;
private readonly ISpeedingViolationCalculator _speedingViolationCalculator;
private readonly string _roadId;

public VehicleActor(
    ActorHost host, DaprClient daprClient,
    ISpeedingViolationCalculator speedingViolationCalculator)
    : base(host)
{
    _daprClient = daprClient;
    _speedingViolationCalculator = speedingViolationCalculator;
    _roadId = _speedingViolationCalculator.GetRoadId();
}
```

基于执行组件的实施不再直接使用 Dapr 状态管理构建基块。 而是在执行每个操作后自动保留状态。

## 总结

Dapr 执行组件构建基块可以更轻松地编写正确的并发系统。 执行组件是状态和逻辑的小单元。 它们使用基于轮次的访问模型，无需使用锁定机制编写线程安全代码。 执行组件是隐式创建的，在未执行任何操作时以无提示方式从内存中卸载。 重新激活执行组件时，自动持久保存并加载执行组件中存储的任何状态。 执行组件模型实现通常是为特定语言或平台创建的。 但是，借助 Dapr 执行组件构建基块，可以从任何语言或平台利用执行组件模型。

执行组件支持计时器和提醒来计划将来的工作。 计时器不会重置空闲计时器，并且允许执行组件在未执行其他操作时停用。 提醒会重置空闲计时器，并且也会自动保留。 计时器和提醒都遵守基于轮次的访问模型，确保在处理计时器/提醒事件时无法执行任何其他操作。

使用 Dapr 状态管理构建基块 持久保存执行组件状态。 支持多项事务的任何状态存储都可用于存储执行组件状态。


