---
author: ggailey777
ms.service: azure-functions
ms.topic: include
ms.date: 09/04/2018
ms.author: glenga
ms.openlocfilehash: 629de079f7cc7d95d10f8ff951a47b8b8fc62dad
ms.sourcegitcommit: 2ec4b3d0bad7dc0071400c2a2264399e4fe34897
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 03/27/2020
ms.locfileid: "77474183"
---
Azure Functions で発生するエラーは、次のいずれかが元になっています。

- Azure Functions の組み込みの[トリガーとバインド](..\articles\azure-functions\functions-triggers-bindings.md)の使用
- 基になっている Azure サービスの API の呼び出し
- REST エンドポイントの呼び出し
- クライアント ライブラリ、パッケージ、またはサードパーティ API の呼び出し

データやメッセージが失われないようにするには、次の確実なエラー処理手法が重要です。 推奨されるエラー処理方法には、次のアクションが含まれます。

- [Application Insights を有効にする](../articles/azure-functions/functions-monitoring.md)
- [構造化エラー処理を使用する](#use-structured-error-handling)
- [べき等に設計する](../articles/azure-functions/functions-idempotent.md)
- [再試行ポリシーを実装する](../articles/azure-functions/functions-reliable-event-processing.md) (該当する場合)

### <a name="use-structured-error-handling"></a>構造化エラー処理を使用する

アプリケーションの正常性を監視するには、エラーをキャプチャして発行することが重要です。 関数コードの最上位レベルに、try/catch ブロックを含める必要があります。 catch ブロックでは、エラーをキャプチャして発行できます。

## <a name="retry-policies-preview"></a>再試行 ポリシー (プレビュー)

再試行 ポリシーは、関数アプリ内の任意のトリガー タイプの関数にて定義ができます。実行が成功するまで、または最大実行回数まで関数を再実行します。再試行 ポリシーは、関数アプリ内の全て関数もしくは個別の関数にて定義することができます。デフォルトでは、関数アプリはメッセージの再試行を実行しません。（[特定のトリガーソースに再試行 ポリシー](#using-retry-support-on-top-of-trigger-resilience)がある場合は除く）関数実行による例外処理が発生するたびに評価されます。ベスト プラクティスは、コード内のすべての例外をキャッチし、再試行につながるエラーを再実行する必要があります。Event Hub や Azure Cosmos DB チェックポイントは、再試行 ポリシーが完了するまで書き込まれず、そのパーテーションでの処理は、現在のバッチが完了するまで一時停止されます。

### Retry policy options

The following options are available for defining a retry policy.

**Max Retry Count** is the maximum number of times an execution is retried before eventual failure. A value of `-1` means to retry indefinitely. The current retry count is stored in memory of the instance. It's possible that an instance has a failure between retry attempts.  When an instance fails during a retry policy, the retry count is lost.  When there are instance failures, triggers like Event Hubs, Azure Cosmos DB, and Queue storage are able to resume processing and retry the batch on a new instance, with the retry count reset to zero.  Other triggers, like HTTP and timer, don't resume on a new instance.  This means that the max retry count is a best effort, and in some rare cases an execution could be retried more than the maximum, or for triggers like HTTP and timer be retried less than the maximum.

**Retry Strategy** controls how retries behave.  The following are two supported retry options:

| Option | Description|
|---|---|
|**`fixedDelay`**| A specified amount of time is allowed to elapse between each retry,|
| **`exponentialBackoff`**| The first retry waits for the minimum delay. On subsequent retries, time is added exponentially to the initial duration for each retry, until the maximum delay is reached.  Exponential back-off adds some small randomization to delays to stagger retries in high-throughput scenarios.|

#### App level configuration

A retry policy can be defined for all functions in an app [using the `host.json` file](../articles/azure-functions/functions-host-json.md#retry). 

#### Function level configuration

A retry policy can be defined for a specific function.  Function-specific configuration takes priority over app-level configuration.

#### Fixed delay retry

# [C#](#tab/csharp)

Retries require NuGet package [Microsoft.Azure.WebJobs](https://www.nuget.org/packages/Microsoft.Azure.WebJobs) >= 3.0.23

```csharp
[FunctionName("EventHubTrigger")]
[FixedDelayRetry(5, "00:00:10")]
public static async Task Run([EventHubTrigger("myHub", Connection = "EventHubConnection")] EventData[] events, ILogger log)
{
// ...
}
```

# [C# Script](#tab/csharp-script)

Here's the retry policy in the *function.json* file:

```json
{
    "disabled": false,
    "bindings": [
        {
            ....
        }
    ],
    "retry": {
        "strategy": "fixedDelay",
        "maxRetryCount": 4,
        "delayInterval": "00:00:10"
    }
}
```
# [JavaScript](#tab/javascript)

Here's the retry policy in the *function.json* file:


```json
{
    "disabled": false,
    "bindings": [
        {
            ....
        }
    ],
    "retry": {
        "strategy": "fixedDelay",
        "maxRetryCount": 4,
        "delayInterval": "00:00:10"
    }
}
```

# [Python](#tab/python)

Here's the retry policy in the *function.json* file:

```json
{
    "disabled": false,
    "bindings": [
        {
            ....
        }
    ],
    "retry": {
        "strategy": "fixedDelay",
        "maxRetryCount": 4,
        "delayInterval": "00:00:10"
    }
}
```

# [Java](#tab/java)

Here's the retry policy in the *function.json* file:


```json
{
    "disabled": false,
    "bindings": [
        {
            ....
        }
    ],
    "retry": {
        "strategy": "fixedDelay",
        "maxRetryCount": 4,
        "delayInterval": "00:00:10"
    }
}
```
---

#### Exponential backoff retry

# [C#](#tab/csharp)

Retries require NuGet package [Microsoft.Azure.WebJobs](https://www.nuget.org/packages/Microsoft.Azure.WebJobs) >= 3.0.23

```csharp
[FunctionName("EventHubTrigger")]
[ExponentialBackoffRetry(5, "00:00:04", "00:15:00")]
public static async Task Run([EventHubTrigger("myHub", Connection = "EventHubConnection")] EventData[] events, ILogger log)
{
// ...
}
```

# [C# Script](#tab/csharp-script)

Here's the retry policy in the *function.json* file:

```json
{
    "disabled": false,
    "bindings": [
        {
            ....
        }
    ],
    "retry": {
        "strategy": "exponentialBackoff",
        "maxRetryCount": 5,
        "minimumInterval": "00:00:10",
        "maximumInterval": "00:15:00"
    }
}
```

# [JavaScript](#tab/javascript)

Here's the retry policy in the *function.json* file:

```json
{
    "disabled": false,
    "bindings": [
        {
            ....
        }
    ],
    "retry": {
        "strategy": "exponentialBackoff",
        "maxRetryCount": 5,
        "minimumInterval": "00:00:10",
        "maximumInterval": "00:15:00"
    }
}
```

# [Python](#tab/python)

Here's the retry policy in the *function.json* file:

```json
{
    "disabled": false,
    "bindings": [
        {
            ....
        }
    ],
    "retry": {
        "strategy": "exponentialBackoff",
        "maxRetryCount": 5,
        "minimumInterval": "00:00:10",
        "maximumInterval": "00:15:00"
    }
}
```

# [Java](#tab/java)

Here's the retry policy in the *function.json* file:

```json
{
    "disabled": false,
    "bindings": [
        {
            ....
        }
    ],
    "retry": {
        "strategy": "exponentialBackoff",
        "maxRetryCount": 5,
        "minimumInterval": "00:00:10",
        "maximumInterval": "00:15:00"
    }
}
```
---

|function.json property  |Attribute Property | Description |
|---------|---------|---------| 
|strategy|n/a|Required. The retry strategy to use. Valid values are `fixedDelay` or `exponentialBackoff`.|
|maxRetryCount|n/a|Required. The maximum number of retries allowed per function execution. `-1` means to retry indefinitely.|
|delayInterval|n/a|The delay that will be used between retries when using `fixedDelay` strategy.|
|minimumInterval|n/a|The minimum retry delay when using `exponentialBackoff` strategy.|
|maximumInterval|n/a|The maximum retry delay when using `exponentialBackoff` strategy.| 

### Retry limitations during preview

- For .NET projects, you may need to manually pull in a version of [Microsoft.Azure.WebJobs](https://www.nuget.org/packages/Microsoft.Azure.WebJobs) >= 3.0.23.
- In the consumption plan, the app may be scaled down to zero while retrying the final messages in a queue.
- In the consumption plan, the app may be scaled down while performing retries.  For best results, choose a retry interval <= 00:01:00 and <= 5 retries.

## Using retry support on top of trigger resilience

The function app retry policy is independent of any retries or resiliency that the trigger provides.  The function retry policy will only layer on top of a trigger resilient retry.  For example, if using Azure Service Bus, by default queues have a message delivery count of 10.  The default delivery count means after 10 attempted deliveries of a queue message, Service Bus will dead-letter the message.  You can define a retry policy for a function that has a Service Bus trigger, but the retries will layer on top of the Service Bus delivery attempts.  

For instance, if you used the default Service Bus delivery count of 10, and defined a function retry policy of 5.  The message would first dequeue, incrementing the service bus delivery account to 1.  If every execution failed, after five attempts to trigger the same message, that message would be marked as abandoned.  Service Bus would immediately requeue the message, it would trigger the function and increment the delivery count to 2.  Finally, after 50 eventual attempts (10 service bus deliveries * five function retries per delivery), the message would be abandoned and trigger a dead-letter on service bus.

> [!WARNING]
> It is not recommended to set the delivery count for a trigger like Service Bus Queues to 1, meaning the message would be dead-lettered immediately after a single function retry cycle.  This is because triggers provide resiliency with retries, while the function retry policy is best effort and may result in less than the desired total number of retries.

### <a name="retry-support"></a>再試行のサポート

次のトリガーには、組み込みの再試行サポートがあります。

* [Azure BLOB Storage](../articles/azure-functions/functions-bindings-storage-blob.md)
* [Azure Queue Storage](../articles/azure-functions/functions-bindings-storage-queue.md)
* [Azure Service Bus (キュー/トピック)](../articles/azure-functions/functions-bindings-service-bus.md)

既定では、これらのトリガーにより要求が最大 5 回再試行されます。 5 回目の再試行後に、Azure Queue storage と Azure Service Bus の両方のトリガーによって[有害キュー](..\articles\azure-functions\functions-bindings-storage-queue-trigger.md#poison-messages)にメッセージが書き込まれます。

他のトリガーまたはバインドの種類については、再試行ポリシーを手動で実装する必要があります。 手動の実装には、[有害メッセージ キュー](..\articles\azure-functions\functions-bindings-storage-blob-trigger.md#poison-blobs)へのエラー情報の書き込みが含まれる場合があります。 有害キューに書き込むことによって、後で操作を再試行する機会ができます。 この方法は、Blob Storage トリガーによって使用されるものと同じです。
