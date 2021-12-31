# Dapr 绑定构建块

https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/bindings

基于云的无服务器产品，例如 Azure Functions 和 AWS Lambda，已经在整个分布式架构中广泛使用。在许多方面，它们使得微服务能够 _处理来自外部系统的事件，或者在外部系统中调用事件_ - 消除了底层复杂性和构建顾虑。有多种外部资源：包括数据存储，消息系统，以及 Web 资源，跨越各种不同的平台和供应商。Dapr 绑定构建块对你的 Dapr 应用程序带来了相同的资源绑定能力。

## 8.2 解决何种问题

Dapr 资源绑定使得你的服务可以与你的应用程序之外的外部资源集成业务操作。来自外部系统的事件可以触发你的服务中的操作，传递上下文信息。你的服务可以继续通过触发另一个外部系统的事件来继续操作。传递上下分内容信息。你的服务不需要与外部系统耦合或者感知。处理封装在内容预定义的 Dapr 组件中。Dapr 使用的组件可以在运行时很容易地切换而不需要改变代码。

例如，Twitter 帐号在用户推送某个关键字的推文的时候触发事件。你的服务暴露事件处理器来处理该推文。一旦完成之后，你的服务触发一个事件来调用外部的 Twilio 服务。Twilio 服务发送包含该推文的信息。图 8-1 展示了该操作的概念架构。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/bindings/bindings-architecture.png)

图 8-1 Dapr 资源绑定概念架构

乍一看，资源绑定行为与本书前面介绍的发布/订阅模式类似。尽管它们共享相似性，其实是不同的。发布/订阅关注于 Dapr 服务之间的异步通讯。而资源绑定有更为广泛的范围。它关注与跨软件平台的系统互操作。在独立的应用程序、数据存储和你的微服务应用之外的服务之间交互信息。

## 8.2 如何工作

Dapr 资源绑定从组件配置文件开始。该 YAML 文件描述了你将要绑定到的资源类型，以及其配置设置。配置完成之后，你的服务就可以从资源接收事件或者触发其上的事件。

> 注意
> 绑定配置将在后面的 组件 一节中详细说明

### 8.2.1 输入绑定

输入绑定从外部资源中使用输入事件触发你的代码。为了接收事件和数据，你需要注册一个来自你的成为 _事件处理器_ 的服务的公共端点。图 8-2 展示了处理流程：

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/bindings/input-binding-flow.png)

图 8-2 Dapr 输入绑定

图 8-2 说明了从外部的 Twitter 帐号接收事件的步骤：

1. Dapr sidecar 读取绑定配置文件，并订阅到该外部资源指定的事件。在示例中，事件源是一个 Twitter 帐号
2. 当匹配的 Tweet 发布在 Twitter 上的时候，运行在 Dapr sidecar 上的绑定组件提取并触发事件。
3. Dapr sidecar 调用配置的绑定端点 ( 也就是事件处理器 )，在该示例中，服务在 `/tweet` 端点的 6000 端口监听 HTTP POST 请求。因为是 HTTP POST 操作，在请求的内容中包含 JSON 格式内容。
4. 处理事件之后，服务返回 HTTP 状态码 200 OK

下面的 ASP.NET Core 控制器提供处理来自外部的 Twitter 绑定的触发事件：

```csharp
[ApiController]
public class SomeController : ControllerBase
{
    public class TwitterTweet
    {
        [JsonPropertyName("id_str")]
        public string ID {get; set; }

        [JsonPropertyName("text")]
        public string Text {get; set; }
    }

    [HttpPost("/tweet")]
    public ActionResult Post(TwitterTweet tweet)
    {
        // Handle tweet
        Console.WriteLine("Tweet received: {0}: {1}", tweet.ID, tweet.Text);

        // ...

        // Acknowledge message
        return Ok();
    }
}
```

如果操作可能出错，你应该返回适当的 400 或者 500 等 HTTP 状态码。对于绑定到 at-least-once 特性的保证，Dapr sidecar 将重试该触发。请查看 [Dapr documentation for resource bindings](https://docs.dapr.io/operations/components/setup-bindings/supported-bindings) 来了解 at-least-once 或者 exactly-once 分发保证。

### 8.2.2 输出绑定

Dapr 也包括了 _输出绑定_ 能力。它支持你的服务触发事件来调用外部资源。重复一下，通过配置 YAML 文件绑定配置开始，其描述了输出绑定。一旦就绪，你触发事件来调用你的应用程序的 Dapr sidecar 的绑定 API。图 8-3 展示了输出绑定流程。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/bindings/output-binding-flow.png)

图 8-3 Dapr 输出绑定流程

1. Dapr sidecar 读取绑定配置文件，其中包含了如何连接到外部资源的信息。在本例中，外部资源是 Twilio SMS 帐号
2. 你的应用程序调用 Dapr sidecar 的 `/v1.0/bindings/sms` 端点。在本例中，它使用 HTTP POST 来调用该 API。也可以使用 gRPC 
3. 运行在 Dapr sidecar 上的绑定组件调用外部消息系统来发送消息。消息将在 POST 主体中包含数据

作为示例，你可以使用 curl 调用 Dapr API 来调用输出绑定：

```bash
curl -X POST http://localhost:3500/v1.0/bindings/sms \
  -H "Content-Type: application/json" \
  -d '{
        "data": "Welcome to this awesome service",
        "metadata": {
          "toNumber": "555-3277"
        },
        "operation": "create"
      }'
```

注意该 HTTP 端口使用 Dapr sidecar 相同的端口 ( 在本例中，默认的 Dapr HTTP 端口是 3500 )

数据的结构 ( 也就是发送的消息 ) 基于绑定多种多样。在上面的示例中，数据消息中包含 `data` 元素。绑定到其它类型外部资源可以不同，特别是对于发送的 metadata。每种负荷必须还要包含一个 `operation` 字段，它定义了绑定将要执行的操作。在上面的示例中指定 `create` 操作，它创建 SMS 消息。常见的操作包括：
* create
* get
* delete
* list

这些决定于绑定支持的操作绑定的作者，每种绑定的文档说明可用的操作以及如何调用。

## 8.3 使用 Dapr .NET SDK

Dapr .NET SDK 为 .NET 开发人员提供语言特定的支持。在下面的示例中，对 `HttpClient.PostAsync()` 的调用被替换为 `DaprClient.InvokeBindingAsync()` 方法。该特定的方法简化了对配置的输出绑定的调用：

```csharp
private async Task SendSMSAsync([FromServices] DaprClient daprClient)
{
    var message = "Welcome to this awesome service";
    var metadata = new Dictionary<string, string>
    {
      { "toNumber", "555-3277" }
    };
    await daprClient.InvokeBindingAsync("sms", "create", message, metadata);
}
```

该方法使用 `metadata` 和 `message` 值。

当用来调用绑定的时候，`DaprClient` 使用 gRPC 来调用 Dapr sidecar 的 Dapr API。

## 绑定组件

在底层，资源绑定使用 Dapr 绑定组件实现。它们由社区贡献并使用 Go 编写。如果你需要集成还没有 Dapr 绑定存在的外部资源，可以自己创建，请检查 [Dapr components-contrib repo](https://github.com/dapr/components-contrib) 来检查如何贡献绑定。

> 注意
> Dapr 和所有它的组件都使用 Golang 开发。Go 是一种现代的，云原生的程序设计语言

使用 YAML 配置文件配置绑定。这是用于 Twitter 绑定的示例配置。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: twitter-mention
  namespace: default
spec:
  type: bindings.twitter
  version: v1
  metadata:
  - name: consumerKey
    value: "****" # twitter api consumer key, required
  - name: consumerSecret
    value: "****" # twitter api consumer secret, required
  - name: accessToken
    value: "****" # twitter api access token, required
  - name: accessSecret
    value: "****" # twitter api access secret, required
  - name: query
    value: "dapr" # your search query, required
```

每个绑定配置包含通用的 `metadata` 元素，包含 `name` 和 `namespace` 字段。Dapr 检查该端点基于配置的 `name` 字段来调用服务。在上述示例中，当事件发生的时候，Dapr 将调用你的服务的 `/twitter-mention` 方法。
在 `spec` 元素中，基于 `metadata` 指定 `type`。示例中指定使用该 API 访问 Twitter 帐号的凭据。对于不同的输入和输出绑定，metadata 是不同的。例如，使用 Twitter 作为输入绑定，你需要设置用来搜索文件的 `query` 字段。每次匹配的推文发送的时候，Dapr sidecar 将调用该服务的 `/twitter-mention` 端点。它也会发送推文的内容。

绑定可以用于配置输入、输出或者两者。有趣的是，绑定并不显式指定输出或者输出。相反，方向由绑定配置值的使用方式推断出来。

[ Dapr documentation for resource bindings](https://docs.dapr.io/operations/components/setup-bindings/supported-bindings/) 提供完整的可用绑定列表和它们特定的配置。

### Cron 绑定

请注意 Dapr 的 Cron 绑定。它不订阅外部系统，相反，该绑定使用可配置的定时调度来触发你的应用程序。绑定提供简化方式来实现后台任务使用常见的间隔来唤醒并执行任务。不需要实现可配置延迟的无限循环。这是一个 Cron 绑定配置的示例。

```csharp
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: checkOrderBacklog
  namespace: default
spec:
  type: bindings.cron
  version: v1
  metadata:
  - name: schedule
    value: "@every 30m"
```

在该示例中，Dapr 每 30 分钟通过调用 `/checkOrderBacklog` 端点来触发服务。有多种方式来设置 `schedule` 值。更多信息，请参考 [Cron binding documentation](https://docs.dapr.io/operations/components/setup-bindings/supported-bindings/cron/)。

## 示例应用：交通管理

在 Dapr 交通管理示例应用中，TrafficControl 服务使用 MQTT 输出绑定来接收来自 CameraSimulation 的消息。图 8-4 展示了 Dapr 交通管理示例应用的概念架构。Dapr 输入绑定用于标记为数字 5 的部分。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/bindings/dapr-solution-input-binding.png)

图 8-4 Dapr 交通管理示例应用的概念架构

### MQTT 输入绑定

MQTT 是轻量级的 pub/sub 消息协议，通常用手 IoT 场景。生产者发送 MQTT 消息到主题，订阅者从该主题接收消息。有多种 MQTT 消息代理可用。Traffic Control 示例应用使用的是 [Eclipse Mosquitto](https://mosquitto.org/)

CameraSimulation 不依赖于任何 Dapr 构建块。它使用 [System.Net.Mqtt](https://www.nuget.org/packages/System.Net.Mqtt) 库发送 MQTT 消息：

```csharp
// ...

// simulate entry
DateTime entryTimestamp = DateTime.Now;
var vehicleRegistered = new VehicleRegistered
{
    Lane = _camNumber,
    LicenseNumber = GenerateRandomLicenseNumber(),
    Timestamp = entryTimestamp
};
_trafficControlService.SendVehicleEntry(vehicleRegistered);

// ...
```

代码使用 ITrafficControlService 类型的代理来调用 TrafficControl 服务。.NET 使用构造函数注入 ITrafficControlService 接口的一个实现。

```csharp
public CameraSimulation(int camNumber, ITrafficControlService trafficControlService)
{
   _camNumber = camNumber;
   _trafficControlService = trafficControlService;
}
```

`MqttTrafficControlService` 类实现了接口 `ITrafficControlService`。它暴露量两个方法：
* SendVehicleEntryAsync
* SendVehicleExitAsync

它们都使用 MQTT 客户端发送消息到 `trafficcontrol/entrycam` 和 `trafficcontrol/exitcam` 主题：

```csharp
public async Task SendVehicleEntryAsync(VehicleRegistered vehicleRegistered)
{
    var eventJson = JsonSerializer.Serialize(vehicleRegistered);
    var message = new MqttApplicationMessage("trafficcontrol/entrycam", Encoding.UTF8.GetBytes(eventJson));
    await _client.PublishAsync(message, MqttQualityOfService.AtMostOnce);
}

public async Task SendVehicleExitAsync(VehicleRegistered vehicleRegistered)
{
    var eventJson = JsonSerializer.Serialize(vehicleRegistered);
    var message = new MqttApplicationMessage("trafficcontrol/exitcam", Encoding.UTF8.GetBytes(eventJson));
    await _client.PublishAsync(message, MqttQualityOfService.AtMostOnce);
}
```

构造函数设置 MQTT 客户端，以发送消息到 MQTT 代理的端口 1883。

在另一端，TrafficControl 服务使用 MQTT 输入绑定来接收由 CameraSimulation 发送的 `VehicleRegistered` 消息。对每个订阅的主题，在 `/dapr/components` 文件夹中，存在一个独立的组件配置文件。第一个是 `entrycam.yaml`:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: entrycam
  namespace: dapr-trafficcontrol
spec:
  type: bindings.mqtt
  version: v1
  metadata:
  - name: url
    value: mqtt://localhost:1883
  - name: topic
    value: trafficcontrol/entrycam
scopes:
  - trafficcontrolservice
```

配置中指定绑定的类型为：`bindings.mqtt`。还设置代理运行在 `localhost:1883`，这是标准的 Mosquitto 使用的端口。还暴露出 `topic`，`trafficcontrol/entrycam`。使用 `scopes`，该配置文件仅仅由 app-id 为 `trafficcontrolservice` 的服务访问绑定。

当 TrafficControl 服务启动的时候，Dapr sidecar 自动订阅到组件配置中的 `trafficcontrol/entrycam` MQTT 主题。当消息到达主题的时候，Dapr sidecar 调用你的服务的 HTTP 端点。sidecar 通过查询绑定配置中的 `metadata.name` 字段来决定 HTTP 端点的 URL。在上面的示例中，端点的 URL 是 `/entrycam`。在 TrafficControl 服务内，不需要代码来支持该端点。

```csharp
[HttpPost("entrycam")]
public async Task<ActionResult> VehicleEntry(VehicleRegistered msg)
{
    // ...
}
```

`exitcam.yaml` 组件配置文件配置 `exitcam` 端点的所有配置。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: exitcam
  namespace: dapr-trafficcontrol
spec:
  type: bindings.mqtt
  version: v1
  metadata:
  - name: url
    value: mqtt://localhost:1883
  - name: topic
    value: trafficcontrol/exitcam
scopes:
  - trafficcontrolservice
```

### SMTP 输出绑定

FineCollection 服务使用 Dapr SMTP 输出绑定来发送电子邮件。

图 8-5 展示了 Dapr 交通管理示例应用的概念架构。Dapr 输入绑定用于图中标记数字 4 的部分。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/bindings/dapr-solution-output-binding.png)

图 8-5 Dapr 交通管理示例应用的概念架构

在 FineCollection 服务中，CollectionController 控制器的 `CollectFine` 方法，包含了使用 Dapr client 来调用输出绑定的代码：

```csharp
// ...

// send fine by email (Dapr output binding)
var body = EmailUtils.CreateEmailBody(speedingViolation, vehicleInfo, fineString);
var metadata = new Dictionary<string, string>
{
    ["emailFrom"] = "noreply@cfca.gov",
    ["emailTo"] = vehicleInfo.OwnerEmail,
    ["subject"] = $"Speeding violation on the {speedingViolation.RoadId}"
};
await daprClient.InvokeBindingAsync("sendmail", "create", body, metadata);

// ...
```

代码使用简化的工具类来创建 HTML 邮件主体，包含了必要的信息。它还创建一个用于 SMTP 绑定需要的 metadata 字典。该绑定组件在调用的时候解释该 metadata。

在调用绑定是样本，下述参数是必须的：
* 绑定组件的名称，这里是 `sendmail`
* 绑定主要执行的操作，这里是 `create`
* 发送的消息主体，这里是 HTML 邮件主体
* 发送邮件的 metadata

在 `/dapr/components` 文件夹中的 `email.yaml` 组件配置文件中，Dapr 输出绑定名为 `sendmail`。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: sendmail
  namespace: dapr-trafficcontrol
spec:
  type: bindings.smtp
  version: v1
  metadata:
  - name: host
    value: localhost
  - name: port
    value: 4025
  - name: user
    secretKeyRef:
      name: smtp.user
      key: smtp.user
  - name: password
    secretKeyRef:
      name: smtp.password
      key: smtp.password
  - name: skipTLSVerify
    value: true
auth:
  secretStore: trafficcontrol-secrets
scopes:
  - finecollectionservice
```

配置设置的绑定类型为：`binding.smtp`

在 metadata 节中包含连接到 SMTP 服务器的信息。
请查看 [the binding's documentation](https://docs.dapr.io/reference/components-reference/supported-bindings/smtp/) 对于该绑定的特定 metadata 要求。连接到 SMTP 服务器的用户名和密码从密钥存储中提取。请参见 [Secrets management building block ](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/secrets-management) 一章。

`scopes` 元素指定仅仅 `finecollectionservice` 服务可以访问该绑定。

交通管理示例应用使用 [MailDev](https://github.com/maildev/maildev)。MailDev 是开发环境下的 SMTP 服务器，它并不真正发送电子邮件 ( 默认 )。相反，它收集电子邮件并在 Web 应用程序的收件箱中列出它们。MailDev 对于开发和测试环境特别有用。

在交通管理示例应用中使用 Dapr 绑定提供了如下的优势：
1. 使用 MQTT 消息和 SMTP 不需要学习该协议或者特定的 MQTT API
2. 使用 SMTP 发送电子邮件而不需要学习该协议或者特定的 SMTP API

## 总结

Dapr 资源绑定使得你可以集成外部资源和系统，而不需要依赖其库或者 SDK。这些外部系统不需要消息系统，例如消息总线或者消息代理。绑定还可用于数据存储和 Web 资源，例如 Twitter 或者 SendGrid。

输入绑定 ( 或 Trigger ) 响应外部系统的事件。它们调用你的应用程序预先配置的公共的 HTTP 端点。Dapr 使用配置文件中的绑定名称来决定调用你的应用程序的端点。

输出绑定将发送消息给外部系统。通过发出到 Dapr sidecar 的 HTTP POST 调用到 `/v1.0/bindings/<binding-name>` 来触发外部绑定。可以使用 gRPC 来调用绑定。.NET SDK 提供了 `InvokeBindingAsync()` 方法来使用 gRPC 调用 Dapr 绑定。

你使用 Dapr 组件实现绑定。这些组件由社区贡献。每种组件的配置通过 metadata 抽象外部系统的配置。另外，支持的命令和消息的结构对于每种绑定组件各不相同。


