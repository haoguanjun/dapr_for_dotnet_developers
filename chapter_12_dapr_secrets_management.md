

# 第 12 章 Dapr 密钥管理构建块

https://docs.microsoft.com/zh-cn/dotnet/architecture/dapr-for-net-developers/secrets-management

企业应用需要使用密钥，常见的场景包括：

* 数据库连接串中包含用户名和密码
* 调用外部 API 所使用的 API Key
* 用于外部系统认证的客户端证书

密钥必须小心地进行管理，它们永远也不应该暴露到应用程序之外。

在不久之前，流行的做法是将应用程序密钥保存狂应用程序代码中的配置文件中。对于 .NET 开发者来说，会温暖地回忆起 _web.config_ 文件。虽然实现起来简单，但将密钥与代码集成在一起却远不安全。常见的错误是，在推送代码到 Git 仓储的时候包含了该文件，将密钥暴露出来。

广泛可接受的构建分布式应用程序的方法论是 [12-因素 应用](https://12factor.net/)。它描述了一组特性和最佳实践。其中第 3 个因素定义了 _配置和密钥与代码分离_

对于这个问题，.NET 平台包含了 [Secret Manager](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets#secret-manager)  功能，它可以将敏感数据存储在项目之外的物理文件夹中。由于密钥处于源代码控制之外，该功能并没有加密数据。它被设计为仅用于**开发目的**。

更为现代也更为安全的方式是将密钥隔离出来，保存到密钥管理工具中，例如 **Hashicorp Vault** 或者 **Azure Key Vault** 中。这些工具支持你外部存储密钥，跨环境的多种凭据，以及在应用程序代码中使用它们。不过，每种工具都有其复杂性和学习曲线。

Dapr 提供了简化管理密钥的构建块。

## 解决何种问题

Dapr 的 [secrets management building block](https://docs.dapr.io/developing-applications/building-blocks/secrets/secrets-overview/) 抽象了密钥和密钥管理的复杂性。

* 隐藏了底层不一致接口带来的复杂性
* 支持多种可插拔的密钥存储组件，可以在开发和产品环境之间工作。
* 应用程序不需要直接依赖密钥存储库
* 开发人员不需要针对每种存储库的详细知识

对密钥的访问通过认证和授权进行。只有具有适当权限的应用程序可以访问密钥。运行在 Kubernetes 环境中的应用程序还可以使用内置的密钥管理机制。

## 它是如何工作的

应用程序通过两种途径使用密钥管理构建块

* 直接通过密钥管理构建块提取密钥
* 通过 Dapr 组件配置间接引用密钥

我们首先介绍直接提取密钥。通过 Dapr 组件配置文件来引用密钥将在下一节说明。

在应用程序通过密钥管理构建块与 Dapr sidecar 交互的时候。sidecar 暴露一组密钥管理 API。该 API 既可以使用 HTTP 也可以使用 gRPC 来调用。使用如下的 URL 访问 HTTP API

```http
http://localhost:<dapr-port>/v1.0/secrets/<store-name>/<name>?<metadata>
```

该 URL 包含如下部分：

* `<dapr-port>` 指定 Dapr sidecar 所监听的端口号
* `<store-name>` 指定 Dapr 密钥存储的名称
* `<name>` 指定提取的密钥名称
* `<metadata>` 提供附加的密钥信息。该部分是可选的，并且对于不同的密钥存储有着不同的 metadata 属性。关于详细的 metadata 属性，请参考 [Secrets API reference | Dapr Docs](https://docs.dapr.io/reference/api/secrets_api/)

> ⚠️
>
> 上述的 URL 表示原生的 Dapr API 调用，对于任何支持 HTTP 或者 gRPC 的开发平台可用。流行的平台，例如 .NET、Java 和 Go 有自己特定的 API

JSON 格式的响应中包含密钥的键和值。

图 11-1 展示了 Dapr 是如何处理密钥 API 的。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/secrets/secrets-flow.png)

图 11-1 使用 Dapr 密钥 API 提取密钥

1. 服务使用密钥存储和密钥的名称调用 Dapr secrets API。
2. Dapr sidecar 从密钥存储中提取指定的密钥
3. Dapr sidecar 将密钥信息返回给服务

有些密钥存储支持在单个密钥中存储多个键值对。对于该场景，响应中将在单个 JSON 响应中包含多组键值对，如下所示：

```http
GET http://localhost:3500/v1.0/secrets/secret-store/interestRates?metadata.version_id=3
```

响应内容

```json
{
  "tier1-percentage": "2.5",
  "tier2-percentage": "3.8",
  "tier3-percentage": "5.1"
}
```

Dapr secrets API 也提供提取应用程序可以访问的所有密钥的操作。

```http
http://localhost:<dapr-port>/v1.0/secrets/<store-name>/bulk
```

## 使用 Dapr .NET SDK

对于 .NET 开发者，Dapr 的 .NET SDK 简化了 Dapr 密钥管理的使用。考虑 `DaprClient.GetSecretAsync()` 方法。它支持使用最小的代价来从任何 Dapr 密钥存储中直接提取密钥。这是一个提取访问 SQL Server 数据库的连接串密钥的示例。

```csharp
var metadata = new Dictionary<string, string> { ["version_id"] = "3" };
Dictionary<string, string> secrets 
  = await daprClient.GetSecretAsync("secret-store", "eshopsecrets", metadata);
string connectionString = secrets["customerdb"];
```

方法 `GetSecretAsync()` 的参数包括：

* Dapr 密钥存储组件的名称 ( 'secret-store' )
* 提取的密钥名称 ( 'eshopsecrets' )
* 可选的 metadata 键值对 ( 'version_id=3' )

由于密钥可以包含多组键值对，所以方法返回一个字典对象。在上面的示例中，密钥名称 `customerdb` 从集合中返回连接串。

Dapr .NET SDK 还提供了一个 .NET 配置提供器。它可以加载指定的密钥到底层的 .NET 配置 API 中。运行中的应用程序可以通过 `IConfiguration` 字典来引用密钥，该 `IConfiguration` 已经被注册到 ASP.NET Core 的依赖注入中。

密钥配置提供器可以通过 NuGet 包 [Dapr.Extensions.Configuration](https://www.nuget.org/packages/Dapr.Extensions.Configuration) 来使用。该提供器可以在 ASP.NET Web API 应用程序的 Program.cs 中注册。

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.WebHost.ConfigureAppConfiguration(config =>
{
    var daprClient = new DaprClientBuilder().Build();
    var secretDescriptors = new List<DaprSecretDescriptor>
    {
        new DaprSecretDescriptor("eshopsecrets")
    };
    config.AddDaprSecretStore("secret-store", secretDescriptors, daprClient);
});
```

上面的示例在应用程序启动的时候，将 `eshopsecrets` 密钥集合加载到 .NET 配置系统中。注册该提供器需要一个 `DaprClient` 的实例来调用位于 Dapr sidecar 中的 secrets API。其它参数包括密钥存储的名称和一个 `DaprSecretDescriptior` 对象来提供密钥的名称。

一旦加载之后，你就可以直接从应用程序代码中提取密钥。

```csharp
public void GetCustomer(IConfiguration config)
{
    var connectionString = config["eshopsecrets"]["customerdb"];
}
```

## 密钥存储组件

密钥管理构建块支持多种密钥存储组件。在本文编写的时候，下列密钥存储是可用的：

* Environment Variables
* Local file
* Kubernetes secrets
* AWS Secrets Manager
* Azure Key Vault
* GCP Secret Manager
* HashiCorp Vault

> 重要
>
> 环境变量和本地文件组件被设计仅用于开发

下面介绍如何配置密钥存储。

### 配置

你通过 Dapr 组件配置文件配置密钥存储。典型的文件结构如下所示：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: [component name]
  namespace: [namespace]
spec:
  type: secretstores.[secret store type]
  version: [secret store version]
  metadata:
  - name: [property name]
    value: [property value]
```

所有的 Dapr 组件配置文件要求一个 `name`，伴随一个可选的 `namespace` 值。另外，在 `spec` 配置节中的  `type` 字段指定密钥存储组件的类型。位于 `metadata` 节中的属性对于每种密钥存储各不相同。

### 间接使用 Dapr 密钥

如本章前面所述，应用程序可以通过在组件配置文件中引用来使用密钥。可以考虑某个状态管理组键使用 Redis Cache 用于存储状态：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: eshop-basket-statestore
  namespace: eshop
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    value: e$h0p0nD@pr
```

上述的配置文件中包含了明文来连接到 Redis 服务器。**硬编码** 的口令永远不是一个好主意。将该配置文件保存到公共存储中将会暴露口令。将口令保存到密钥存储中将显著改进该场景。

下面的示例展示各种不同的密钥存储。

### 本地文件

本地文件组键用于开发场景。它将密钥存储到本地文件系统的一个 JSON 文件中。这里是名为 `eshop-secrets.json` 的示例。其中包含单个的密钥 - Redis 的口令。

```json
{
  "eShopRedisPassword": "e$h0p0nD@pr"
}
```

将该文件保存到运行 Dapr 应用程序的 `components` 文件夹中。

下面的密钥存储配置文件使用该 JSON 文件作为密钥存储。它也保存在 `components` 文件夹中。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: eshop-local-secret-store
  namespace: eshop
spec:
  type: secretstores.local.file
  version: v1
  metadata:
  - name: secretsFile
    value: ./components/eshop-secrets.json
  - name: nestedSeparator
    value: ":"
```

该组件的类型为：`secretstore.local.file`。而 `secretsFile` 元数据指定密钥文件的路径。

> 重要
>
> 密钥文件的路径既可以是绝对路径，也可以是相对路径。相对路径基于应用程序启动的文件夹。在该示例中，`components` 文件夹是包含 .NET 应用程序的子文件夹。

从应用程序文件夹中，启动 Dapr 应用程序，通过命令行参数指定 `components` 路径。

```bash
dapr run --app-id basket-api --components-path ./components dotnet run
```

> 注意
>
> 上述示例使用了 Dapr 自寄宿模式运行。对于 Kubernetes 模式，考虑使用卷挂载。

Dapr 配置文件中的 `nettedSeparator` 指定了用来平面化 JSON 结构的分隔符。考虑下面的配置片段：

```json
{
    "redisPassword": "some password",
    "connectionStrings": {
        "customerdb": "some connection string",
        "productdb": "some connection string"
    }
}
```

使用冒号 (:) 作为分隔符，你可以使用键 `connectionStrings:customerdb` 来提取  `customerdb` 连接串。

> 注意
>
> 冒号 (:) 是默认的分隔符。

在下面的示例中，状态管理配置文件引用本地密钥存储组件来获取访问 Redis 服务器的口令。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: eshop-basket-statestore
  namespace: eshop
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    secretKeyRef:
      name: eShopRedisPassword
      key: eShopRedisPassword
auth:
  secretStore: eshop-local-secret-store
```

`secretKeyRef` 元素引用包含口令的密钥。它替换了以前的明文口令。密钥的名称和键的名称，`eShopRedisPassword` 引用密钥。密钥管理组件的名称 `eshop-local-secret-store` 在 `auth` 元数据元素中找到。

你可能奇怪为什么 `eShopRedisPassword` 既用来识别密钥的名称，也用来识别键。在本地密钥存储中，密钥不是通过分离的名称来识别。在随后的 Kubernetes 中将会不同。

### Kubernetes 密钥

第 2 个示例关注于运行在 Kubernetes 中的 Dapr 应用程序。它使用 Kubernetes 提供的标准的密钥机制。使用 Kubernetes CLI ( `kubectl` ) 来创建名为 `eshop-redis-secret` 的密钥，其中包含口令：

```bash
kubectl create secret generic eshopsecrets --from-literal=redisPassword=e$h0p0nD@pr -n eshop
```

一旦创建之后，你就可以在状态管理的组件配置文件中引用该密钥：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: eshop-basket-statestore
  namespace: eshop
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis:6379
  - name: redisPassword
    secretKeyRef:
      name: eshopsecrets
      key: redisPassword
auth:
  secretStore: kubernetes
```

该 `secretKeyRef`  元素指定 Kubernetes 密钥的名称和密钥的键，通过  `eshopsecrets`, 和  `redisPassword` 表示。`auth` 元数据节指示 Dapr 使用 Kubernetes 密钥管理组件。

> 注意
>
> Auth 默认使用 Kubernetes 密钥，此场景下可以忽略

在生产环境下，密钥通常作为自动化的 CI/CD 管线的一部分创建。此时需要确认只有拥有适当权限的人员可以访问和改变密钥。开发人员创建配置文件，不需要知道实际的密钥值。

### Auzre 密钥

下一个示例继续真实世界的生产场景。它使用 **Azure Key Vault** 作为密钥存储。Azure Key Vault 是托管的 Azure 服务，支持密钥安全地存储在云端。

为了该示例可以使用，下面的准备需要预先完成：

* 你拥有安全访问 Azure 订阅的管理访问
* 预先准备名为 `eshopkv` 的 Azure Key Vault，其中保存了名为 `redisPassword` 的密钥，包含了连接 Redis 服务器的密码。
* 已经在 Azure Active Directory 中创建了 [service principal](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal) 
* 为该 [service principal](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal) 在本地文件系统中安装了 X509 证书 ( 包含公钥和私钥 ) 

> 注意
>
> [service principal](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal) 是可以由应用程序认证 Azure Service 的标识。该 service principal 使用 X509 证书。应用程序使用该证书作为凭据来认证自己。

在  [Dapr Azure Key Vault secret store documentation](https://docs.dapr.io/operations/components/setup-secret-store/supported-secret-stores/azure-keyvault/) 中提供了创建并配置 Key Vault 环境的详细说明。

#### 在自寄宿模式使用 Key Vault

在 Dapr 自寄宿模式下使用 Azure Key Vault 需要如下组件配置文件：

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: eshop-azurekv-secret-store
  namespace: eshop
spec:
  type: secretstores.azure.keyvault
  version: v1
  metadata:
  - name: vaultName
    value: eshopkv
  - name: spnTenantId
    value: "619926af-a7c3-4e95-93ed-4ecc4e3e652b"
  - name: spnClientId
    value: "6cf48032-6c38-43be-9d6f-2a43ce736b09"
  - name: spnCertificateFile
    value : "azurekv-spn-cert.pfx"
```

密钥存储类型为 `secretstores.azure.keyvault`。`metadata` 元素使用如下属性提供对 Key Vault 的访问：

* `vaultName` 包含 Azure Key Vault 的名称
* `spnTenantId` 包含用于认证 Key Vault 的 _tenant ID_
* `spnClientId` 包含用于认证 Key Vault 的 service principla 的 app ID
* `spnCertificateFile` 包含 service principla 用来认证 Key Vault 的证书文件路径

> Tip
>
> 你可以通过 Azure portal 或者 Azure CLI 复制 service principal 信息。

现在应用程序可以从 Azure Key Vault 中提取 Redis 口令了。

#### 在 Kubernetes 中使用 Key Vault

与 Kubernetes 一起使用 Azure Key Vault 也需要 service principal 来认证 Azure Key Vault。

首先，使用 kubectl CLI 工具创建一个 _Kubernetes secret_ ，其中包含证书文件。

```bash
kubectl create secret generic [k8s_spn_secret_name] --from-file=[pfx_certificate_file_local_path] -n eshop
```

此命令需要 2 个命令行参数：

1. `[k8s_spn_secret_name]` 是在 Kubernetes 密钥存储中的名称
2. `[pfx_certificate_file_local_path]` 是 X509 证书文件的路径

创建之后，你可以在密钥存储组件配置文件中引用该 Kubernetes 密钥。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: eshop-azurekv-secret-store
  namespace: eshop
spec:
  type: secretstores.azure.keyvault
  version: v1
  metadata:
  - name: vaultName
    value: [your_keyvault_name]
  - name: spnTenantId
    value: "619926af-a7c3-4e95-93ed-4ecc4e3e652b"
  - name: spnClientId
    value: "6cf48032-6c38-43be-9d6f-2a43ce736b09"
  - name: spnCertificate
    secretKeyRef:
      name: [k8s_spn_secret_name]
      key: [pfx_certificate_file_local_name]
auth:
    secretStore: kubernetes
```

此时，运行在 Kubernetes 中的应用程序可以从 Azure Key Vault 中提取 Redis 口令了。

> 重要提示
>
> 重要的是保证用于 service principal 的 X509 证书文件保存在安全的位置。最佳的位置是在源代码仓库之外的一个众所周知的文件夹中。该配置文件可以引用该文件夹来使用证书文件。在本地开发机上，你自己负责将证书文件复制到该文件夹中。对于自动部署环境，管线将自动复制该证书到应用程序部署的位置。最佳实践是针对每个环境使用不同的 service principal。这样可以防止来自 **开发环境** 的 service principal 访问 **生产环境** 的密钥。

当运行在 Auzre Kubernetes Service ( AKS ) 中的时候，倾向于使用  [Azure Managed Identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview) 实现基于 Auzre Key Vault 的认证。托管的标识不在本书讨论范围之内，它在  [Azure Key Vault with managed identities](https://docs.dapr.io/operations/components/setup-secret-store/supported-secret-stores/azure-keyvault-managed-identity/) 中说明。

### Scope secrets

Secret scopes 支持你控制你的应用程序可以访问哪个密钥。你可以在 Dapr sidecar 配置文件中配置 Scopes。在  [Dapr configuration documentation](https://docs.dapr.io/operations/configuration/configuration-overview/)  中提供关于 scoping secrets 的说明。

这里是包含了 secret scopes 的 Dapr sidecar 配置文件的示例。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: dapr-config
  namespace: eshop
spec:
  tracing:
    samplingRate: "1"
  secrets:
    scopes:
      - storeName: eshop-azurekv-secret-store
        defaultAccess: allow
        deniedSecrets: ["redisPassword", "apiKey"]
```

针对每个密钥存储指定 scopes。在上面的示例中，密钥存储被命名为：`eshop-azurekv-secret-store`。使用如下属性配置对密钥的访问：

| 属性             | 值                  | 说明                                                         |
| ---------------- | ------------------- | ------------------------------------------------------------ |
| `defaultAccess`  | `allow` 或者 `deny` | 允许或者禁止访问指定密钥存储中的 _所有_ 密钥。该属性可选，默认为 `allow` |
| `allowedSecrets` | 密钥键列表          | 数组中指定的密钥可访问，该属性可选                           |
| `deniedSecrets`  | 密钥键列表          | 数组中指定的密钥 _不_ 可访问，该属性可选                     |

`allowedSecrets` 和 `deniedSecrets` 属性优先于 `defaultAccess` 属性。考虑指定了 `defaultAccess: allowed` 和一个 `allowedSecrets` 列表，在此场景下，只有在 `allowedSecrets` 列表中的密钥可以被应用程序访问。

## 示例应用：Dapr Traffic Control

在 Dapr 交通管理示例应用程序中，密钥管理构建块用于多个位置。密钥从代码中和 Dapr 组件配置文件中的引用中被使用。图 10-2 展示了Dapr 交通管理示例应用的概念架构。Dapr 密钥管理构建块被用于标记为数字 6 的位置。

![](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/media/secrets/dapr-solution-secrets-management.png)

图 10-2 Dapr 交通管理示例应用概念架构

罚款服务使用 SMTP 输出绑定来发送电子邮件 ( 见 绑定 一章 )。电子邮件组件文件使用密钥管理构建块来获得连接到 SMTP 邮件服务器的凭据。为了计算超速罚款，该服务使用虚拟的罚款计算组件，它需要一个 license key。它也是从密钥管理构建块中获取的。

交通管理服务将汽车信息存储在 Redis 状态存储中 ( 见 状态管理 一章)。它使用密钥管理构建块来获取连接到 Redis 服务器的凭据。

由于该交通管理示例应用程序可以运行在自托管模式下，或者 Kubernetes 模式下，有两种方式提供密钥：

* 本地的 JSON 文件
* Kubernetes 密钥

### 密钥

考虑 dapr/components 文件夹中的 secrets-file.yaml 组件配置文件。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: trafficcontrol-secrets
  namespace: dapr-trafficcontrol
spec:
  type: secretstores.local.file
  version: v1
  metadata:
  - name: secretsFile
    value: ../dapr/components/secrets.json
  - name: nestedSeparator
    value: "."
scopes:
  - trafficcontrolservice
  - finecollectionservice
```

该文件描述了名为 `trafficcontrol-secrets` 的密钥管理组件。其中 `type` 元素设置为 `local.file`，`secretsFile` 设置为 `../dapr/components/secrets.json`。对于自托管模式，使用  [Local file](https://docs.microsoft.com/en-us/dotnet/architecture/dapr-for-net-developers/secrets-management#local-file) 组件。路径必须相对于服务启动的文件夹。JSON 格式的密钥文件包含密钥。

```json
{
    "state":{
        "redisPassword": ""
    },
    "smtp":{
        "user": "_username",
        "password": "_password"
    },
    "finecalculator":{
        "licensekey": "HX783-K2L7V-CRJ4A-5PN1G"
    }
}
```

在示例应用程序中，Redis 服务器使用没有口令的方式。为了连接到 SMTP 服务器，需要使用 _username 和 _password 凭据。而用于罚款计算的 license key 则是一个随机生成的字符串。

虽然密钥存储在内嵌的级别，密钥管理构建块在读取的时候会平铺该层次结构。它使用一个符号作为级别之间的分隔符 ( 由组件配置文件中的 `nestedSeparator` 字段指定 )。该结构使得你可以使用平铺后的名称来引用密钥，例如 `smtp.user`。

当运行在 Kubernetes 中时，密钥使用内置的 Kubernetes 密钥存储指定。考虑下面的位于 k8s 文件夹中的 `secrets.yaml` Kubernetes 描述文件。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: trafficcontrol-secrets
  namespace: dapr-trafficcontrol
type: Opaque
data:
  smtp.user: X3VzZXJuYW1l
  smtp.password: X3Bhc3N3b3Jk
  finecalculator.licensekey: SFg3ODMtSzJMN1YtQ1JKNEEtNVBOMUc=
```

该组件也命名为 `trafficcontrol-secrets`。密钥以 Base64 编码形式存储。

> 重要提示
>
> Base64 表示一种编码方式，它并不加密数据。生产环境下，Base64 并不安全。

后面的段落将说明在交通管理示例应用程序中是如何使用密钥的。

### SMTP 服务器凭据

考虑位于 `dapr/components` 文件夹中的 `email.yaml` 组件配置文件。

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

`auth` 配置节引用了名为 `trafficcontrol-secrets` 的密钥管理组件。其中的 `user` 和 `password` 部分引用密钥 `smtp.user` 和 `smtp.password` 部分。

当运行在 Kubernetes 中时，内置的 Kubernetes 密钥存储被使用。存储在 `k8s` 文件夹中的  `email.yaml` 引用 Kubernetes 密钥来获取连接到 SMTP 服务器的凭据。

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
    value: mailserver
  - name: port
    value: 25
  - name: user
    secretKeyRef:
      name: trafficcontrol-secrets
      key: smtp.user
  - name: password
    secretKeyRef:
      name: trafficcontrol-secrets
      key: smtp.password
  - name: skipTLSVerify
    value: true
scopes:
  - finecollectionservice
```

与本地密钥存储不同，Kubernetes 存储不使用 `auth` 配置节显式指定密钥存储组件。相反，默认就是使用内置的 Kubernetes 密钥存储。

### Redis 服务器凭据

接着下一个，考虑位于 `dapr/components` 文件夹中的  `statestore.yaml` 组件配置文件。

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: dapr-trafficcontrol
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    secretKeyRef:
      name: state.redisPassword
      key: state.redisPassword
  - name: actorStateStore
    value: "true"
auth:
  secretStore: trafficcontrol-secrets
scopes:
  - trafficcontrolservice
```

一旦开始，`auth` 配置节引用名为 `trafficcontrol-secrets` 密钥管理组件。在绑定元数据中的`redisPassword` 引用 `state.redisPassword` 绑定。

### FineCalculator 组件 license key

罚款计算服务基于超速信息计算罚款。该组件实现为领域服务，抽象为接口 `IFineCalculator`

```csharp
public interface IFineCalculator
{
    public int CalculateFine(string licenseKey, int violationInKmh);
}
```

`CalculateFine()` 方法接受一个包含 `licenseKey` 的字符串作为第一个参数。该 Key 解锁用来实现的第三方组件。为了保持示例的简化，该实现由一系列硬编码的 `if` 语句来实现。可以在 `DomainsServices` 文件夹中的 `HardCodedFineCalculator` 中找到实现内容。

```csharp
public class HardCodedFineCalculator : IFineCalculator
    {
        public int CalculateFine(string licenseKey, int violationInKmh)
        {
            if (licenseKey != "HX783-K2L7V-CRJ4A-5PN1G")
            {
                throw new InvalidOperationException("Invalid license-key specified.");
            }

            int fine = 9; // default administration fee
            if (violationInKmh < 5 )
            {
                fine += 18;
            }
            else if (violationInKmh >= 5 && violationInKmh < 10 )
            {
                fine += 31;
            }

            // ...

            else if (violationInKmh == 35)
            {
                fine += 372;
            }
            else
            {
                // violation above 35 KMh will be determined by the prosecutor
                return 0;
            }

            return fine;
        }
    }
```

该实现模拟检查传入的 `licenseKey`。在调用 `CalculateFine()` 方法的时候，罚款服务的 `CollectionController`  必须传入正确的 license key 参数。它从 Dapr 密钥管理构建块中获取该 license key，这是通过由 Dapr .NET SDK 提供的 Dapr 客户端来完成。如果你检查 `CollectionController` 的构造函数，就可以看到：

```csharp
// set finecalculator component license-key
if (_fineCalculatorLicenseKey == null)
{
    bool runningInK8s = Convert.ToBoolean(Environment.GetEnvironmentVariable("DOTNET_RUNNING_IN_CONTAINER") ?? "false");
    var metadata = new Dictionary<string, string> { { "namespace", "dapr-trafficcontrol" } };
    if (runningInK8s)
    {
        var k8sSecrets = daprClient.GetSecretAsync(
            "kubernetes", "trafficcontrol-secrets", metadata).Result;
        _fineCalculatorLicenseKey = k8sSecrets["finecalculator.licensekey"];
    }
    else
    {
        var secrets = daprClient.GetSecretAsync(
            "trafficcontrol-secrets", "finecalculator.licensekey", metadata).Result;
        _fineCalculatorLicenseKey = secrets["finecalculator.licensekey"];
    }
}
```

代码首先检查服务是运行在 Kubernetes 环境下还是自托管环境下。该检测是必须的，因为不同的密钥管理组件被用于不同的场景下。`GetSecretAsync()` 方法的第一个参数是 Dapr 组件的名称。第二个参数是密钥的名称。`metadata` 作为第三个参数指定包含密钥的命名空间。密钥 `finecalculator.licensekey` 的值存储在私有字段中以备后继使用。

使用 Dapr 密钥管理提供如下优势：

1. 没有敏感信息存储在代码中或者应用程序配置文件中
2. 不需要学习任何新的与密钥存储交互的 API 

## 总结

Dapr 密钥管理构建块提供了存储和提取敏感配置设置的能力，例如口令和连接串。它保证了密钥私有和防止意外的暴露。

构建块支持多种不同的密钥存储，并使用 Dapr 密钥 API 隐藏了其复杂性。

Dapr .NET SDK 提供了 `DaprClient` 对象来获取密钥。它也包括来一个 .NET 配置提供器来将密钥添加到 .NET 配置系统中。一旦配置加载之后，你就可以在 .NET 代码中使用这些密钥。

可以使用密钥 Scope 来控制访问指定的密钥。





