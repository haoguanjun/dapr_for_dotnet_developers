# 第 6 章 Dapr 状态管理构建块

[The Dapr state management building block | Microsoft Docs](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/state-management)

分布式应用程序由一组独立的服务构成。尽管每个服务应当是无状态的，有些服务必须却必须跟踪状态的变化以生成业务操作。例如对于电子商务系统中的购物车服务，如果该服务不能跟踪状态，那么在离开站点的时候，顾客就会丢失购物车的内容，导致销售机会的丧失和顾客体验的下降。对于这些场景，状态需要被持久化到分布式的状态存储中。Dapr 状态管理构建块简化了状态跟踪处理，并提供跨多种数据存储的高级功能。

尝试状态管理构建块的话，可以看一下第 3 章中的计数器示例应用程序。

## 处理何种问题

在分布式应用程序中跟踪状态是一种挑战，例如：

* 应用程序可能需要不同类型的数据存储
* 对访问数据和更新数据的不同一致性级别
* 在同一时间可能由多个用户更新数据，需要冲突处理方案
* 在与数据存储交互的时候，出现暂时的瞬时错误必须能够进行重试

Dapr 状态管理构建块定位于处理这些挑战，在不需要依赖，或者学习第 3 方存储 SDK 学习曲线的情况下，Dapr 平滑地处理了状态跟踪问题。

> 重要：
>
> Dapr 状态管理提供 key/value API。该功能不支持关系存储，以及图数据存储。

## 如何工作

应用程序与 Dapr 的 sidebar 交互来存储并提供 key/value 数据。在该机制的背后，sidebar API 基于可配置的状态存储组件来持久化数据。开发人员可以从众多支持的状态存储中选择，包括 Azure Cosmos DB，SQL Server，以及 Cassandra。

该 API 即可以使用 HTTP 也可以使用 gRPC 进行调用，使用如下所示的 URL 调用 HTTP API

```http
http://localhost:<dapr-port>/v1.0/state/<store-name>/
```

* <dapr-port>: Dapr 监听的 HTTP 端口
* <store-name> : 状态存储组件使用的名称

图 5-1 展示了使用 Dapr 的购物车服务，使用名为 `statestore` 的 Dapr 存储组件存储键值对。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/state-management/state-management-flow.png)

 图 5-1 在 Dapr 状态存储中存储 key/value

注意前图中的步骤：

1. 购物车服务调用 Dapr sidebar 的状态管理 API。请求的 body 部分封装了多个 key/value 对。
2. Dapr sidebar 基于组件配置文件检查状态存储，在本例中，使用 Redis 缓存状态存储。
3. sidecar 持久化数据到 Redis cache 中

获取存储的数据通过简单的 API 调用完成。在下面的示例中，通过 _curl_ 命令调用 Dapr sidecar 的 API 来提取数据。

```bash
curl http://localhost:3500/v1.0/state/statestore/basket1
```

该命令的响应体中包含了存储的数据。

```json
{
  "items": [
    {
      "itemId": "DaprHoodie",
      "quantity": 1
    }
  ],
  "customerId": 1
}
```

下面开始介绍如何使用更高级状态管理构建块特性。

## 一致性

CAP 理论是一套应用于存储状态的分布式系统的理论。图 5-2 展示了 CAP 理论的 3 个属性。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/state-management/cap-theorem.png)

图 5-2 CAP 理论

CAP 理论指出分布式系统在一致性、可用性和分区容差之间的权衡。进而，任何数据分布式数据存储只能保证三种属性中的两种。

* 一致性（C）。群集中的每个节点都会使用最新的数据进行响应，即使系统必须在所有副本更新之前阻止请求。如果查询当前正在更新的项目的“一致系统”，则在所有副本成功更新之前，不会得到响应。但是，您将始终收到最新的数据。
* 可用性（A）。每个节点都会立即返回响应，即使该响应不是最新的数据。如果您在“可用系统”中查询正在更新的项目，您将得到该服务在此时可能提供的最佳答案。
* 分区公差（P）。保证即使复制数据节点出现故障或与其他复制数据节点失去连接，系统仍能继续运行。

分布式应用程序必须处理 P 属性。当服务通过网络呼叫相互通信时，将发生网络中断（P）。考虑到这一点，分布式应用程序必须是 AP 或 CP。

AP：应用程序选择可用性而不是一致性。Dapr 通过其 **eventual consistency** 最终一致性策略支持这一选择。考虑底层数据存储，例如Azure CosmosDB，它在多个副本上存储冗余数据。为了最终的一致性，状态存储将更新写入一个副本，并与客户端一起完成写入请求。在此之后，存储将异步更新其副本。读取请求可以从任何副本返回数据，包括尚未收到最新更新的副本。
CP：应用程序选择一致性而不是可用性。Dapr 以其 **strong consistency** 强一致性策略支持这一选择。在这种情况下，状态存储将在完成写入请求之前同步更新所有（或在某些情况下，一组）所需的副本。读取操作将在多个副本中一致地返回最新数据。
状态操作的一致性级别是通过向操作附加 _一致性提示_ 来指定的。以下 curl 命令使用强一致性提示将 Hello=World键/值对写入状态存储：

```bash
curl -X POST http://localhost:3500/v1.0/state/<store-name> \
  -H "Content-Type: application/json" \
  -d '[
        {
          "key": "Hello",
          "value": "World",
          "options": {
            "consistency": "strong"
          }
        }
      ]'
```

> 重要
>
> 实际是由底层的 Dapr 存储组件完成该操作的一致性提示。不是所有的数据存储都支持所有这两种一致性级别。如果没有设置一致性提示，默认的行为为 **eventual**

## 并发性

在多用户应用程序中，多个用户有可能同时更新相同的数据。Dapr 支持乐观并发控制（OCC）来管理冲突。OCC基于这样一种假设：由于用户处理数据的不同部分，所以更新冲突不常见。更有效的方法是假设更新会成功，如果不成功则重试。另一种方法是实现悲观锁定，这会影响性能，因为长时间运行的锁定会导致数据争用。
Dapr 支持使用 ETag 的乐观并发控制（OCC）。ETag 是与存储的密钥/值对的特定版本关联的值。每次密钥/值对更新时，ETag 值也会更新。当客户端检索密钥/值对时，响应包括当前 ETag 值。当客户端更新或删除密钥/值对时，它必须将该 ETag 值发送回请求正文。如果另一个客户端同时更新了数据，ETag 将不匹配，请求将失败。此时，客户端必须检索更新的数据，再次进行更改，然后重新提交更新。这种策略称为 **first-write-wins**。
Dapr 还支持 **last-write-wins** 策略。使用这种方法，客户机不会将 ETag 附加到写请求。状态存储组件将始终允许更新，即使在会话期间基础值已更改。last-write-wins 对于低数据争用的高吞吐量写入场景非常有用。此外，可以容忍覆盖偶尔的用户更新。

## 事务

Dapr可以将多项更改作为一个事务实现的单个操作写入数据存储。此功能仅适用于支持 [ACID 事务](https://en.wikipedia.org/wiki/ACID> 的数据存储。在撰写本文时，这些存储包括Redis、MongoDB、PostgreSQL、SQL Server和Azure CosmosDB。
在下面的示例中，多项目操作在单个事务中发送到状态存储。所有操作都必须成功，事务才能提交。如果一个或多个操作失败，则整个事务回滚。

```bash
curl -X POST http://localhost:3500/v1.0/state/<store-name>/transaction \
  -H "Content-Type: application/json" \
  -d '{
        "operations": [
          {
            "operation": "upsert",
            "request": { "key": "Key1", "value": "Value1"
            }
          },
          {
            "operation": "delete",
            "request": { "key": "Key2" }
          }
        ]
      }'
```

对于不支持事务的数据存储，多组键值仍然可以作为单个请求发送。下面的示例展示了 bulk 写操作。

```bash
curl -X POST http://localhost:3500/v1.0/state/<store-name> \
  -H "Content-Type: application/json" \
  -d '[
        { "key": "Key1", "value": "Value1" },
        { "key": "Key2", "value": "Value2" }
      ]'
```

对于批量操作，Dapr 将提交每单组的键值更新作为独立的请求到数据存储中。

## 使用 Dapr .NET SDK

Dapr .NET SDK 对 .NET 平台提供特定语言的支持。开发者可以使用在第 3 章中介绍的 `DaprClient` 类来读取和写入数据。下面的示例展示了如何使用 `DaprClient.GetStateAsync<TVale>` 方法从数据存储中读取数据。该方法需要使用存储的名称，`statestore` 和键值来作为参数：

```csharp
var weatherForecast 
    = await daprClient.GetStateAsync<WeatherForecast>("statestore", "AMS");
```

如果存储中没有关于键值 `AWS` 的数据，返回结果为 `default(WeatherForecast)`。

写入数据到数据存储中，使用 `DaprClient.SaveStateAsync<TValue>` 方法。

```csharp
daprClient.SaveStateAsync("statestore", "AMS", weatherForecast);
```

示例使用 **last-write-wins** 策略，因为没有提供 ETag 到存储组件中。为了使用乐观并发控制 (OCC) 的 **first-write-wins** 策略，首先使用 `DaprClient。GetStateAndEtagAsync` 方法获取当前的 ETag 值，然后带有获取的 Etag 来写入更新之后的值，使用 `DaprClient.TrySaveStateAsync` 方法。

```csharp
var (weatherForecast, etag) = await daprClient.GetStateAndETagAsync<WeatherForecast>("statestore", city);

// ... make some changes to the retrieved weather forecast

var result = await daprClient.TrySaveStateAsync("statestore", city, weatherForecast, etag);
```

在数据读取之后，当数据 ( 关联的 ETag ) 在存储中变化之后， `DaprClient.TrySaveStateAsync` 方法将会失败。该方法返回一个 boolean 值来指示该调用是否成功。处理失败的一种策略是简单地从存储中重新加载数据，重新变更，最后重新提交更新。

如果你总是期望写入成功，而不管对数据的其它变更，使用 **last-write-wins** 策略。

该 SDK 提供了其它方法来批量获取数据，删除数据，以及执行事务。更多信息，请参考：[Dapr .NET SDK repository](https://github.com/dapr/dotnet-sdk)

### ASP.NET Core 集成

Dapr 也支持 ASP.NET Core，一种跨平台的构建现代云基础的 Web 应用程序的框架。Dapr SDK 提供了直接集成状态管理能力到 [ASP.NET Core 模型绑定](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/model-binding) 的能力。通过在 `IMVCBuilder.AddDapr` 后面追加 `.AddDapr` 扩展方法到你的 `Startup.cs` 类中进行简单的配置。如下所示：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddControllers().AddDapr();
}
```

一旦配置完成，Dapr 可以通过 ASP.NET Core 的 `FromState` 特性，直接注入 key/value 对到控制器的 Action 方法中。不再需要引用 `DaprClient` 对象。下面的示例展示了对给定城市返回天气预报的一个 Web API。

```csharp
[HttpGet("{city}")]
public ActionResult<WeatherForecast> Get([FromState("statestore", "city")] StateEntry<WeatherForecast> forecast)
{
    if (forecast.Value == null)
    {
      return NotFound();
    }

    return forecast.Value;
}
```

在该示例中，控制器使用 `FromState` 特性加载天气预报。第一个特性参数是状态存储，`statestore` 。第二个特性参数是 `city`，它是路由模板变量的名称，用来获取状态的键值。如果你忽略掉第二个参数，绑定方法的参数 `forecast` 将被用于查找路由模板变量。

类 `StateEntry`  包含提取的键值对的所有信息的属性：`StoreNmae`、`Key`、 `Value` 以及 `Etag`。ETag 用于实现乐观并发控制 (OCC)。该类还提供用于删除和更新获取的键值对而不需要 `DaprClient` 实例。在下一个示例中，`TrySaveAsync` 方法用来使用 OCC 更新获取的天气预报。

```csharp
[HttpPut("{city}")]
public async Task Put(WeatherForecast updatedForecast, [FromState("statestore", "city")] StateEntry<WeatherForecast> currentForecast)
{
    // update cached current forecast with updated forecast passed into service endpoint
    currentForecast.Value = updatedForecast;

    // update state store
    var success = await currentForecast.TrySaveAsync();

    // ... check result
}
```



## 状态存储组件

到本书编写时为止，Dapr 提供如下的事务状态存储：

* Azure CosmosDB
* Azure SQL Server
* MongoDB
* PostgreSQL
* Redis

Dapr 还包括对支持 CRUD，但是没有事务支持的状态存储：

- Aerospike
- Azure Blob Storage
- Azure Table Storage
- Cassandra
- Cloudstate
- Couchbase
- etcd
- Google Cloud Firestore
- Hashicorp Consul
- Hazelcast
- Memcached
- Zookeeper

### 配置

当初始化本地、自寄宿的开发时，Dapr 注册 Redis 作为默认的状态存储。下面是默认的状态存储配置示例。注意默认的名称：`statestore`

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

> 注意
>
> 可以对单个应用程序使用不同的名称注册多个状态存储。

Redis 状态存储要求 `redisHost` 和 `redisPassword` 元数据来连接到 Redis 实例。在上面的示例中，Redis 口令 (默认是空字符串) 作为明文存储。最佳实践是避免明文字符串，总是使用引用密钥。关于密钥存储，参见第 10 章。

其它的元数据字段 `actorStateStore`，指示状态存储如何被 actors 构建块所消费。

### 键前缀策略

在底层存储中，状态管理组件使用不同的策略来管理存储的 key/value 对。回顾签名的示例

```bash
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[{
        "key": "basket1",
        "value": {
          "customerId": 1,
          "items": [
            { "itemId": "DaprHoodie", "quantity": 1 }
          ]
        }
     }]'
```

使用 Redis 控制台工具，可以查看内部的 Redis 缓存来了解 Redis 状态存储是如何持久化数据的。

```bash
127.0.0.1:6379> KEYS *
1) "basketservice||basket1"

127.0.0.1:6379> HGETALL basketservice||basket1
1) "data"
2) "{\"items\":[{\"itemId\":\"DaprHoodie\",\"quantity\":1}],\"customerId\":1}"
3) "version"
4) "1"
```

输出显示了完整的 Redis 用于数据的 Key 是 `basketservice||basket1`。默认情况下，Dapr 使用 Dapr 实例的应用程序标识 (application_id) 作为键的前缀。这种命名约定使得多个 Dapr 实例可以共享同一个数据存储而不会出现键名称冲突。对于开发者来说，关键是在运行使用 Dapr 的应用程序时，总是指定相同的 `application_id`。如果忽略了，Dapr 将生成一个唯一的 application ID。而如果 `application id` 变化了，应用程序就不能访问使用以前的键保存的状态了。

也就是说，在



## 示例应用：Dapr 流量管控

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/state-management/dapr-solution-state-management.png)

## 总结

