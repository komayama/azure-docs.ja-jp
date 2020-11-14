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

### <a name="retry-policy-options"></a>再試行 ポリシーオプション

再試行 ポリシー を定義するには、以下のオプションを使用することができます。

**最大試行回数** は最終的な失敗をする前に再試行される最大回数です。`-1` の値は、無制限に再試行することを示しております。現在の再試行回数はインスタンスのメモリに保存されます。再試行の間にインスタンスの障害が発生する可能性があります。再試行中にインスタンスの障害が発生すると再試行回数が失われます。インスタンスの障害が発生した場合は、トリガーである Event Hub, Azure Cosmos DB や Queue Storage は処理を再開し、再試行回数は 0 回にリセットされ、新しいインスタンス上にてバッチを再試行します。他のトリガーである HTTP やタイマーは、新しいインスタンス上では再試行しません。この最大再試行回数はベストエフォートであることを意味し、まれに最大値以上の再試行を実行をするもしくは HTTP やタイマーなどのトリガーである場合は最大値よりも少なく再試行される可能性があります。

**再試行方法(strategy)** 再試行の動作を制御します。再試行のオプションは以下の 2 つがサポートされています。

| オプション | 説明 |
|---|---|
|**`fixedDelay`**| 再試行の間に指定された時間が許可されます |
| **`exponentialBackoff`**| 最初の再試行は、最小限の遅延が発生します。その後の再試行では、最大限の遅延が発生するまで、各再試行の初期値に指数関数の時間が追加されます。指数バックオフは高いスループットのシナリオにて再試行時間をずらすために、遅延に小さいランダムな値を追加します。|

#### <a name="app-level-configuration"></a>アプリケーション レイヤーの構成

[`host.json` ファイルを使用して](../articles/azure-functions/functions-host-json.md#retry) アプリケーション内のすべての関数に対して、再試行 ポリシーを定義することができます。

#### <a name="function-level-configuration"></a>関数レベルの構成

再試行 ポリシーは特定の関数に定義することができます。特定の関数の構成は、アプリケーション レイヤーレベルの構成よりも優勢されます。

#### <a name="fixed-delay-retry"></a>Fixed delay 再試行

# [C#](#tab/csharp)

再試行には NuGet パッケージが必要です [Microsoft.Azure.WebJobs](https://www.nuget.org/packages/Microsoft.Azure.WebJobs) >= 3.0.23

```csharp
[FunctionName("EventHubTrigger")]
[FixedDelayRetry(5, "00:00:10")]
public static async Task Run([EventHubTrigger("myHub", Connection = "EventHubConnection")] EventData[] events, ILogger log)
{
// ...
}
```

# [C# Script](#tab/csharp-script)

*function.json* ファイルの再試行 ポリシーは以下の通りです:

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

*function.json* ファイルの再試行 ポリシーは以下の通りです:


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

*function.json* ファイルの再試行 ポリシーは以下の通りです:

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

*function.json* ファイルの再試行 ポリシーは以下の通りです:


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

#### <a name="exponential-backoff-retry"></a>Exponential backoff 再試行

# [C#](#tab/csharp)

再試行には NuGet パッケージが必要です [Microsoft.Azure.WebJobs](https://www.nuget.org/packages/Microsoft.Azure.WebJobs) >= 3.0.23

```csharp
[FunctionName("EventHubTrigger")]
[ExponentialBackoffRetry(5, "00:00:04", "00:15:00")]
public static async Task Run([EventHubTrigger("myHub", Connection = "EventHubConnection")] EventData[] events, ILogger log)
{
// ...
}
```

# [C# Script](#tab/csharp-script)

*function.json* ファイルの再試行 ポリシーは以下の通りです:

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

*function.json* ファイルの再試行 ポリシーは以下の通りです:

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

*function.json* ファイルの再試行 ポリシーは以下の通りです:

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

*function.json* ファイルの再試行 ポリシーは以下の通りです:

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

|function.json のプロパティ |属性 のプロパティ | 説明 |
|---------|---------|---------| 
|strategy|n/a|必須です。再試行の方法として使用されます。有効な値は、 `fixedDelay` もしくは `exponentialBackoff` です。|
|maxRetryCount|n/a|必須です。1 回の関数実行で許可される最大再試行回数です。 `-1` は無制限に再試行します。|
|delayInterval|n/a|`fixedDelay` strategy を使用した際に、再試行の間に使用される遅延です。|
|minimumInterval|n/a|`exponentialBackoff` strategy を使用した際に、最小の再試行遅延です。|
|maximumInterval|n/a|`exponentialBackoff` strategy を使用した際に、最大の再試行遅延です。| 

### <a name="retry-limitations-during-preview"></a>プレビュー中につき再試行の制限がございます

- .NET プロジェクトでは、[Microsoft.Azure.WebJobs](https://www.nuget.org/packages/Microsoft.Azure.WebJobs) >= 3.0.23 のバージョンを手動で取得する必要がございます。
- 従量課金プランでは、キュー内の最後のメッセージを再試行する間に、アプリケーションがスケールダウンし 0 になる可能性がございます。
- 従量課金プランでは、再試行中にアプリケーションがスケールダウンする可能性があります。最適な結果を得るには、再試行間隔 <= 00:01:00 と 再試行回数 <= 5 回です。

## <a name="using-retry-support-on-top-of-trigger-resilience"></a>トリガーの回復力に加えて再試行をサポートを使用する

関数アプリケーションの再試行ポリシーは複数回の再試行もしくはトリガーが提供する回復力とは無関係です。関数の再試行ポリシーは、トリガーの回復力のある再試行上にて階層化されます。例えば、もし Azure Service Bus を使用している場合、デフォルトではキューのメッセージ配信数は 10 になります。デフォルトの配信数は、キューメッセージの配信数が 10 であることを意味しており、Service Bus はメッセージを配信不能にします。Service Bus トリガーを持つ関数の再試行ポリシーを定義することができますが、再試行は、Service Bus 配信の試行の上に重なります。

たとえば、デフォルトの Service Bus 配信数を 10 使用し、関数の再試行ポリシーを 5 と定義したとします。メッセージは最初にデキューされ、Service Bus 配信アカウントを 1 に増加します。すべての実行が失敗した場合、同じメッセージを 5 回トリガーしようとした後、そのメッセージは破棄されたものとしてマークされます。Service Bus はすぐに再キューイングし、関数をトリガーされ配信数を 2 に増加します。最終的に 50 回の再試行後 ( 10 回の Service Bus の配信 * 配信ごとに 5 回の関数の再試行 )、メッセージは破棄され Service Bus にて デッド レター がトリガーされます。

> [!警告]
> Service Bus や キューなどのトリガーの配信数を 1 に設定することは推奨しておりません。これは、単一の関数の再試行サイクルの直後にメッセージが デッド レター になることを意味します。その理由としてトリガーは再試行による回復力を提供するためで、一方で関数の再試行ポリシーはベストエフォートであり、再試行の設定回数より少なくなる可能性がございます。

### <a name="retry-support"></a>再試行のサポート

次のトリガーには、組み込みの再試行サポートがあります。

* [Azure BLOB Storage](../articles/azure-functions/functions-bindings-storage-blob.md)
* [Azure Queue Storage](../articles/azure-functions/functions-bindings-storage-queue.md)
* [Azure Service Bus (キュー/トピック)](../articles/azure-functions/functions-bindings-service-bus.md)

既定では、これらのトリガーにより要求が最大 5 回再試行されます。 5 回目の再試行後に、Azure Queue storage と Azure Service Bus の両方のトリガーによって[有害キュー](..\articles\azure-functions\functions-bindings-storage-queue-trigger.md#poison-messages)にメッセージが書き込まれます。

他のトリガーまたはバインドの種類については、再試行ポリシーを手動で実装する必要があります。 手動の実装には、[有害メッセージ キュー](..\articles\azure-functions\functions-bindings-storage-blob-trigger.md#poison-blobs)へのエラー情報の書き込みが含まれる場合があります。 有害キューに書き込むことによって、後で操作を再試行する機会ができます。 この方法は、Blob Storage トリガーによって使用されるものと同じです。
