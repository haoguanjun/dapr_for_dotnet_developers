# 第 6 章 Dapr 服务调用构建块

https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/service-invocation

纵观分布式系统，服务经常需要与其它服务通讯以完成业务操作。 [Dapr service invocation building block](https://docs.dapr.io/developing-applications/building-blocks/service-invocation/service-invocation-overview/) 可以帮助简化服务之间的通讯。

## 6.1 解决何种问题

在分布式系统中的服务间调用，看起来似乎很简单，但是涉及到各种挑战。例如：
* 如何定位其它服务
* 如何安全调用其它服务，特定的服务地址
* 当短暂的瞬时错误发生的时候，如何处理重试

最后，由于分布式应用程序由各种不同的服务聚合而成，洞察服务之间的调用图谱是诊断产品问题的关键。

服务调用构建块通过使用 Dapr sidecar 作为 [反向代理](https://kemptechnologies.com/reverse-proxy/reverse-proxy/) 来处理此类挑战。

## 6.2 它是如何工作的

让我们从一个示例开始。考虑有两个服务，"服务 A" 和 "服务 B"。

服务 A 需要调用服务 B 的 `catalog/items` API。尽管服务 A 可以依赖于服务 B，并直接调用它，但是，服务 A 却调用 Dapr sidecar 的服务调用 API。图 6-1 展示了操作的实现。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/service-invocation/service-invocation-flow.png)

图 6-1 Dapr 服务调用是如何工作的

注意上图中调用的步骤：

1. 服务 A 通过调用服务 A 的 sidecar 的服务调用 API 来调用服务 B 的 `catalog/items` API
   > 注意
   > sidecar 使用可插拔的命名解析组件来解析服务 B 的地址。在自托管模式下，Dapr 使用 mDNS 来进行查询。当运行在 Kubernetes 模式下，Kubernetes DNS 服务决定该地址。
2. 服务 A 的 sidecar 将请求转发给服务 B 的 sidecar
3. 服务 B 的 sidecar 发出对服务 B 的 `catalog/items` API 的实际调用
4. 服务 B 执行请求处理，将响应返回给它的 sidecar
5. 服务 B 的 sidecar 将响应转发回服务 A 的 sidecar
6. 服务 A 的 sidecar 将响应返回给服务 A

由于调用需要通过 sidecar，Dapr 可以注入一些有用的横切行为：
* 对于失效调用的自动重试
* 对于服务之间的调用使用 mutual (mTLS) 认证，包括自动证书翻转
* 使用访问控制策略来控制客户端可以进行何种操作
* 对于服务间的所有调用，捕获 traces 和 metrics 信息来提供洞察和诊断

任何应用程序都可以使用 Dapr 内置的 invoke API 来调用 Dapr sidecar。该 API 既可以使用 HTTP 又可以使用 gRPC 方式。使用如下的 URL 调用 HTTP API：

```http
http://localhost:<dapr-port>/v1.0/invoke/<application-id>/method/<method-name>
```

* <dapr-port> Dapr 监听的端口号
* <application-id> 服务调用的 application ID
* <method-name> 调用远程服务的方法名称

在随后的示例中，示例的 _curl_ 调用访问了服务 B 的 `catalog/items` GET 端点。

```bash
curl http://localhost:3500/v1.0/invoke/serviceb/method/catalog/items
```

> 注意
> Dapr API 支持任何可以支持 HTTP 或者 gRPC 的应用程序使用 Dapr 构建块。进而，服务调用构建块可以作为协议之间的桥梁。服务之间可以使用 HTTP、gRPC 或者两者同时来进行通讯。

在下一节中，您将学到如何使用 .NET SDK 来简化服务调用。

## 6.2 使用 Dapr .NET SDK

Dapr .NET SDK 提供给 .NET 开发人员直觉的和语言特定的方式来与 Dapr 交互。该 SDK 提供给开发者 3 种方式来调用远程服务：
1. 使用 HttpClient 调用远程 HTTP 服务
2. 使用 DaprClient 调用远程 HTTP 服务
3. 使用 DaprClient 调用远程 gRPC 服务

### 6.2.1 使用 HttpClient 调用远程 HTTP 服务

调用 HTTP 端点推荐的方式是使用 Dapr 提供的加强版 `HttpClient`。下面的示例通过调用 `orderservice` 的 `submit()` 方法来提交订单：

```csharp
var httpClient = DaprClient.CreateInvokeHttpClient();
await httpClient.PostAsJsonAsync("http://orderservice/submit", order);
```

在该示例中，`DaprClient.CreateInvokeHttpClient()` 方法返回一个 `HttpClient` 实例，用来执行 Dapr 的服务调用。该返回的 `HttpClient` 使用特制的 Dapr 消息处理器来重写对外调用请求的 URL。主机的名称被解释为被调用服务的 application ID。重写之后的请求实际上如下调用：

```http
http://127.0.0.1:3500/v1/invoke/orderservice/method/submit
```

该示例使用 Dapr HTTP 端点的默认值，形式为 `http://127.0.0.1:<dapr-http-port>/`。`dapr-http-port` 的实际值通过环境变量 `DAPR_HTTP_PORT` 获得。如果没有设置，默认的端口号将使用 3500.

另外，你可以在对 `DaprClient.CreateInvokeHttpClient()` 方法的调用中配置定制端点：

```csharp
var httpClient = DaprClient.CreateInvokeHttpClient(
        daprEndpoint = "localhost:4000");
```

你还可以通过指定 application ID 来之间设置基础地址。这样可以在调用的时候使用相对地址：

```csharp
var httpClient = DaprClient.CreateInvokeHttpClient("orderservice");
await httpClient.PostAsJsonAsync("/submit");
```

该 `HttpClient` 对象倾向于长生命周期。单个的 `HttpClient` 实例可以在整个应用程序生命周期中被重用。下面的场景演示来 `OrderServiceClient` 如何重用 Dapr 的 `HttpClient` 实例。

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<IOrderServiceClient, OrderServiceClient>(
    _ => new OrderServiceClient(DaprClient.CreateInvokeHttpClient("orderservice")));
```

在上面的代码片段中，`OrderServiceClient` 被注册为 ASP.NET Core 依赖注入系统的单例。在实现的工厂中，通过调用 `DaprClient.CreateInvokeHttpClient()` 方法创建了新的 `HttpClient` 对象实例。通过将 `OrderServiceClient` 注册为单例，它将在整个应用程序生命周期中被重用。

`OrderServiceClient` 本身没有 Dapr 特定的代码。尽管 Dapr 服务调用在底层被使用，你可以将 Httpclient 看作与其它 HttpClient 等同。

```csharp
public class OrderServiceClient : IOrderServiceClient
{
    private readonly HttpClient _httpClient;

    public OrderServiceClient(HttpClient httpClient)
    {
        _httpClient = httpClient ?? throw new ArgumentNullException(nameof(httpClient));
    }

    public async Task SubmitOrder(Order order)
    {
        var response = await _httpClient.PostAsJsonAsync("submit", order);
        response.EnsureSuccessStatusCode();
    }
}
```

对 Dapr 服务调用使用 HttpClient 类有多种优势：

* HttpClient 是众所周知的类，多数开发人员已经在代码中使用过。对 Dapr 服务调用使用 HttpClient 使得开发人员可以重用现有的知识技能。
* HttpClient 支持高级场景，例如自定义请求头，以及完全控制请求和响应消息
* 在 .NET 5 中，HttpClient 支持使用 System.Text.Json 进行自动的序列化与反序列化
* HttpClient 集成了在现存的多种框架和库中，例如 Refit，RestSharp 和 Polly

### 6.2.2 使用 DaprClient 调用 HTTP 服务

尽管 HttpClient 是使用 HTTP 语义调用服务的推荐方式，你还可以使用 `DaprClient.InvokeMethodAsync()` 家族的方法。下面的示例通过调用 `orderservice` 服务的 `sumit()` 方法来提交订单：

```csharp
var daprClient = new DaprClientBuilder().Build();
try
{
    var confirmation =
        await daprClient.InvokeMethodAsync<Order, OrderConfirmation>(
            "orderservice", "submit", order);
}
catch (InvocationException ex)
{
    // Handle error
}
```

第 3 个参数，`order` 是内部序列化之后的 ( 使用 `System.Text.JsonSerializer` )，然后作为请求的内容发送。.NET SDK 负责调用 sidecar。它也会反序列化响应内容为 `OrderConfirmation` 对象。因为没有指定 HTTP 方法，请求将使用 HTTP POST 来执行。

下面的示例展示如何通过指定 `HttpMethod` 来发出 HTTP GET 请求：

```csharp
var catalogItems = await daprClient.InvokeMethodAsync<IEnumerable<CatalogItem>>(HttpMethod.Get, "catalogservice", "items");
```

对于有些场景，你可能需要更多对于请求消息的控制。例如，当你需要特定的请求头的时候，或者你希望使用自定义的序列化器来处理请求内容。`DaprClient.CreateInvokeMethodRequest()` 方法创建一个 `HttpRequestMessage` 对象。下面的示例代码展示如何添加一个 HTTP 认证头到请求消息中：

```csharp
var request = daprClient.CreateInvokeMethodRequest("orderservice", "submit", order);
request.Headers.Authorization = new AuthenticationHeaderValue("bearer", token);
```

现在的 `HttpRequestMessage` 设置了如下属性：
* Url = `http://127.0.0.1:3500/v1.0/invoke/orderservice/method/submit`
* HttpMethod = POST
* Content = `JsonContent` 对象包含 JSON 序列化的 `order`
* Headers.Authorization = "bearer \<token>"

一旦你设置对 Request 对象的设置，使用 `DaprClient.InvokeMethodAsync()` 来发送它：

```csharp
var orderConfirmation = await daprClient.InvokeMethodAsync<OrderConfirmation>(request);
```

如果请求成功处理，`DaprClient.InvokeMethodAsync` 将响应内容反序列化为 `OrderConfirmation` 对象实例。另外，你可以使用 `DaprClient.InvokeMethodWithResponseAsync` 来获得底层的 `HttpResponseMessage` 的完全控制：

```csharp
var response = await daprClient.InvokeMethodWithResponseAsync(request);
response.EnsureSuccessStatusCode();

var orderConfirmation = response.Content.ReadFromJsonAsync<OrderConfirmation>();
```

> 注意
> 对于使用 HTTP 进行服务调用，值的考虑使用前一节中介绍的使用 Dapr HttpClient 方式。使用 HttpClient 可以提供附加的优势，例如与现有框架和库集成。

### 6.2.3 使用 DaprClient 调用 gRPC 服务

DaprClient 提供了一套 `InvokeMethodGrpcAsync()` 方法用于调用 gRPC 端点。与 HTTP 方法主要的不同在于使用了 Protobuf 协议序列化器而不是 JSON。下面的示例通过 gRPC 调用 `orderservice` 的 `submitOrder` 方法。

```csharp
var daprClient = new DaprClientBuilder().Build();
try
{
    var confirmation = await daprClient.InvokeMethodGrpcAsync<Order, OrderConfirmation>("orderservice", "submitOrder", order);
}
catch (InvocationException ex)
{
    // Handle error
}
```

在上面的示例中，DaprClient 使用 Protobuf 序列化给定的 `order` 对象，并将结果用于 gRPC 请求体中。同样，响应内容也使用 Protobuf 反序列化然后返回给调用方。Protobuf 特别的提供了比用于 HTTP 服务调用方式的 JSON 方式更好的性能。

## 6.3 命名解析组件

在本书编写时，Dapr 提供支持下面的命名解析组件：
* mDNS ( 当运行在自托管模式下时默认使用 )
* Kubernetes Name Resolution ( 当运行在 Kubernetes 模式下默认使用 )
* HashiCorp Consul

### 6.3.1 配置

如果使用非默认的命名解析组件，添加 `nameResolution` 规范到应用程序的 Dapr 配置文件中。下面示例的 Dapr 配置文件支持使用 HashiCorp Consul 命名解析：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: dapr-config
spec:
  nameResolution:
    component: "consul"
    configuration:
      selfRegister: true
```


## 6.4 示例应用：Dapr 交通管理

在 Dapr 交通管理示例应用中，Finecollection 罚款服务使用 Dapr 服务调用构建块通过 VehicleRegistration 车辆注册服务来获得车辆和车主信息。图 6-2 展示了 Dapr 交通管理示例应用程序的概念架构。Dapr 服务调用构建块用于流程图中标注数字 1 的部分。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/service-invocation/dapr-solution-service-invocation.png)

图 6-2 Dapr 交通管理示例应用概念架构

信息通过 `FineCollection` 罚款服务中的 ASP.NET  `CollectionController` 来获得。`CollectFine` 方法期望一个进入的 `SpeedingViolation` 参数。它调用 Dapr 服务调用构建块来调用 VehicleRegistration 服务。代码片段如下所示：

```csharp
[Topic("pubsub", "speedingviolations")]
[Route("collectfine")]
[HttpPost]
public async Task<ActionResult> CollectFine(SpeedingViolation speedingViolation, [FromServices] DaprClient daprClient)
{
   // ...

   // get owner info (Dapr service invocation)
   var vehicleInfo = _vehicleRegistrationService.GetVehicleInfo(speedingViolation.VehicleId).Result;

   // ...
}
```

代码使用 `VehicleRegistrationService` 代理类型来调用 VehicleRegistration 服务。ASP.NET Core 使用构造函数注入来注入服务代理的实例：

```csharp
public CollectionController(
    ILogger<CollectionController> logger,
    IFineCalculator fineCalculator,
    VehicleRegistrationService vehicleRegistrationService,
    DaprClient daprClient)
{
    // ...
}
```

`VehicleRegistrationService` 类中包含单个的方法 `GetVehicleInfo()`。它使用 ASP.NET Core 的 `HttpClient` 来调用 VehicleRegistration 服务：

```csharp
public class VehicleRegistrationService
{
    private HttpClient _httpClient;
    public VehicleRegistrationService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<VehicleInfo> GetVehicleInfo(string licenseNumber)
    {
        return await _httpClient.GetFromJsonAsync<VehicleInfo>(
            $"vehicleinfo/{licenseNumber}");
    }
}
```

代码没有直接依赖任何 Dapr 类。相反，它借助于本模块前面部分中 使用 HttpClient 调用 HTTP 服务 部分中，集成在 ASP.NET Core 的集成。下面的代码来自 `Startup` 类中的 `ConfigureService()` 方法，注册了 `VehicleRegistrationService` 代理：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<VehicleRegistrationService>(_ =>
    new VehicleRegistrationService(DaprClient.CreateInvokeHttpClient(
        "vehicleregistrationservice", $"http://localhost:{daprHttpPort}"
    )));
```

`DaprClient.CreateInvokeHttpClient()` 方法创建 `HttpClient` 实例，在底层使用服务调用构建块来调用 VehicleRegistration 服务。它需要 Dapr 的目标服务的 app-id 和 Dapr sidecar 的 URL。在启动时，`daprHttpPort` 参数包含了用于与 Dapr sidecar 通讯的端口号。

在交通管理示例应用中使用 Dapr 服务调用提供了如下优势：
1. 解藕了目标服务的地址
2. 增加了自动重试特性的弹性
3. 基于代理重用现有的 `HttpClient` 的能力 ( 通过 ASP.NET Core 集成提供 )

## 6.5 总结

在本章中，你学习了服务调用构建块。看到了如何通过直接的 HTTP 调用到 Dapr sidecar 方式，和使用 Dapr .NET SDK 方式。

Dapr .NET SDK 提供了多种方式来调用远程方法。HttpClient 对于希望重用现有的技能的开发人员非常有价值，它也兼容多种现有的框架和库。DaprClient 提供直接使用 HTTP 或者 gRPC 语义的 Dapr 服务调用。
