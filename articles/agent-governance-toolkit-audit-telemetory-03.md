---
title: "Agent Governance Toolkit の監査とテレメトリ - 本番環境への道 - .NET での実装入門"
emoji: "🔐"
type: "tech"
topics: ["azure", "dotnet", "ai", "agent", "security"]
published: true
publication_name: "microsoft"
---

# Agent Governance Toolkit の監査とテレメトリ - 本番環境への道 - .NET での実装入門

## はじめに

前回の記事では、Agent Governance Toolkit (AGT) の **Agent Trust** 機能を使った信頼スコアベースのアクセス制御について解説しました。今回は、その次のステップとして、**本番環境を見据えた監査ログ収集** と **テレメトリ統合** について、実装例を交えて解説します。

AI エージェントが重要な判断やツール実行を行う環境では、「何をしたのか？」「いつしたのか？」という監査証跡の記録が重要になります。AGT は、ガバナンスイベントを外部システムに出力し、Azure Blob Storage や Application Insights などの本番環境インフラとシームレスに統合できます。

## 本番環境での監査とテレメトリの重要性

### 規制要件への対応

金融機関やヘルスケア業界では、AI システムの決定プロセスや実行内容の監査証跡が法的に求められます。

### 異常検知と事後対応

監査ログを分析することで、異常なエージェント動作の兆候を検出し、迅速に対応できます。

### パフォーマンス分析

テレメトリデータを収集することで、エージェント実行の遅延ボトルネックや失敗パターンを特定できます。

### コンプライアンスレポート

監査証跡は、セキュリティ監査やコンプライアンスレビューの証拠として使用できます。

## AGTAuditBlobTelemetryApp01 プロジェクトの概要

このサンプルプロジェクトは、前のサンプル (AGTIdentityWithMAFApp02) をベースに、以下を追加実装しています：

| 機能 | 説明 |
|------|------|
| **Blob監査**: `BlobAuditSink` | すべてのガバナンスイベントを Azure Blob Storage の Append Blob に追記ロック式で保存。 |
| **テレメトリ統合** | `Azure.Monitor.OpenTelemetry.Exporter` を使用してメトリクス、トレース、ログを Application Insights にエクスポート |
| **Direct Governance Evaluation** | ツール呼び出しの許可/拒否判定をポリシーに基づいて直接評価するデモ |
| **スレッドセーフ** | `SemaphoreSlim` を使用して複数スレッドからの同時アクセスを保護 |

### プロジェクト構成

```
AGTAuditBlobTelemetryApp01/
├── Program.cs                      # メインプログラム
├── BlobAuditSink.cs               # Blob Storage への監査出力
├── AGTAuditBlobTelemetryApp01.csproj
├── policies/
│   └── default.yaml               # ガバナンスポリシー
└── README.md
```

## アーキテクチャ概要

```
┌─────────────────────────────────────┐
│  Microsoft Agent Framework (MAF)    │
│  + Governance Kernel                │
└────────────┬────────────────────────┘
             │
             ├─→ ① Governance Events
             │
             ├─────────────────────────────────────────┐
             │                                         │
             ▼                                         ▼
    ┌─────────────────────┐          ┌────────────────────────┐
    │ Blob Storage        │          │ Application Insights   │
    │ (Audit Logs)        │          │ (Metrics/Traces/Logs)  │
    │ default.yaml        │          │                        │
    │ via BlobAuditSink   │          │ via Azure Monitor      │
    │                     │          │ OpenTelemetry Exporter │
    └─────────────────────┘          └────────────────────────┘
```

### データフロー

1. **Microsoft Agent Framework エージェント実行**: ツール呼び出しが発生
2. **ガバナンス判定**: GovernanceKernel がポリシーに基づいて許可/拒否を判定
3. **イベント発火**: `GovernanceEvent` がメモリに生成
4. **監査出力（① 経路）**: `BlobAuditSink` が JSON 形式でログを Blob に追記
5. **テレメトリ出力（② 経路）**: `ActivitySource` でスパンを記録し、Application Insights にエクスポート

## 主要コンポーネント解説

### 1. BlobAuditSink クラス

監査ログを Azure Blob Storage の **Append Blob** に記録するコンポーネントです。

```csharp
internal sealed class BlobAuditSink : IDisposable
{
    private readonly AppendBlobClient _appendBlobClient;
    private readonly SemaphoreSlim _gate = new(1, 1);
    private bool _initialized;

    // TODO: 本番環境では Managed Identity を利用すること
    public BlobAuditSink(string storageAccountUri, AzureCliCredential credential, 
                         string containerName, string blobName)
    {
        // Blob コンテナ・クライアントの初期化
        var serviceClient = new BlobServiceClient(new Uri(storageAccountUri), credential);
        var containerClient = serviceClient.GetBlobContainerClient(containerName);
        containerClient.CreateIfNotExists();
        _appendBlobClient = containerClient.GetAppendBlobClient(blobName);
    }

    public void Append(GovernanceEvent governanceEvent)
    {
        // ガバナンスイベントを JSON 形式で Blob に追記
        var record = new GovernanceAuditRecord(
            governanceEvent.EventId,
            governanceEvent.Timestamp,
            governanceEvent.Type.ToString(),
            governanceEvent.AgentId,
            governanceEvent.SessionId,
            governanceEvent.PolicyName,
            new Dictionary<string, object>(governanceEvent.Data));

        var payload = JsonSerializer.Serialize(record) + Environment.NewLine;
        var bytes = Encoding.UTF8.GetBytes(payload);

        _gate.Wait();  // スレッドセーフ: 1 つのスレッドのみが Blob に追記可能
        try
        {
            EnsureBlobExists();
            using var stream = new MemoryStream(bytes, writable: false);
            _appendBlobClient.AppendBlock(stream);
        }
        finally
        {
            _gate.Release();
        }
    }
}
```

#### ポイント

- **SemaphoreSlim**: 複数スレッドからの同時アクセスを制御。Append Blob の追記操作を 1 つのスレッドに限定
- **AppendBlobClient**: 従来の Upload/Download ではなく、常に末尾に追記。ログファイルに最適
- **JSONL 形式**: 1 行 = 1 レコード。ストリーミング処理やスプレッドシートへの取り込みに適しています。こちらを Microsoft Fabric 等のデータ基盤で取り込み処理可能です。
- **⚠️ セキュリティ**: ツール引数に API キーや個人情報が混入する可能性があります。詳しくは後述の「[セキュリティ考慮事項：監査ログのサニタイズ](#セキュリティ考慮事項監査ログのサニタイズ)」を参照してください。

### 2. OpenTelemetry の統合

```csharp
using var meterProvider = Sdk.CreateMeterProviderBuilder()
    .SetResourceBuilder(resourceBuilder)
    .AddMeter(GovernanceMetrics.MeterName)
    .AddAzureMonitorMetricExporter(options =>
    {
        options.ConnectionString = applicationInsightsConnectionString;
    })
    .Build();

using var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .SetResourceBuilder(resourceBuilder)
    .AddSource(GovernanceMetrics.MeterName)
    .AddSource(ActivitySource.Name)
    .AddAzureMonitorTraceExporter(options =>
    {
        options.ConnectionString = applicationInsightsConnectionString;
    })
    .Build();
```

#### ポイント

- **MeterProvider**: `GovernanceMetrics.MeterName` (= `"agent_governance"`) からのメトリクスを Azure Monitor に送信
- **TracerProvider**: `ActivitySource` で生成されたスパンを Application Insights に記録
- **ResourceBuilder**: サービス名とバージョンを付与し、複数エージェント間で識別可能に

### 3. ガバナンスイベントの監査フック

```csharp
kernel.OnAllEvents(evt =>
{
    // スパン生成: トレース情報として Application Insights に記録
    using var activity = ActivitySource.StartActivity($"GovernanceEvent.{evt.Type}", 
                                                      ActivityKind.Internal);
    activity?.SetTag("event.type", evt.Type.ToString());
    activity?.SetTag("agent.id", evt.AgentId);
    activity?.SetTag("policy.name", evt.PolicyName ?? "(none)");
    activity?.SetTag("tool.name", evt.Data.GetValueOrDefault("tool_name", "(unknown)"));

    // Blob Storage に監査ログを追記
    auditSink.Append(evt);
    
    // ログ出力
    logger.LogInformation(
        "[Audit -> Blob] {EventType} policy={PolicyName} tool={ToolName} agentId={AgentId}",
        evt.Type,
        evt.PolicyName ?? "(none)",
        evt.Data.GetValueOrDefault("tool_name", "(unknown)"),
        evt.AgentId);
});
```

#### ポイント

- **OnAllEvents コールバック**: すべてのガバナンスイベント（PolicyCheck, PolicyViolation, ToolCallBlocked など）をキャッチ
- **二重出力**: Blob Storage（監査証跡）と Application Insights（分析用テレメトリ）に同時出力
- **スパンタグ**: 後から Application Insights でフィルタリングやグループ化が可能

### 4. Direct Governance Evaluation

ツール呼び出しを実際に実行せず、ポリシーに基づいて許可/拒否だけを判定するデモです。

```csharp
// 許可例
var allowed = kernel.EvaluateToolCall(
    agent.Did,
    "GetLocation",
    new Dictionary<string, object>
    {
        ["city"] = "London"
    });

Console.WriteLine($"GetLocation call allowed? {allowed.Allowed}");
Console.WriteLine($"GetLocation call reason: {allowed.Reason}");

// 拒否例
var blocked = kernel.EvaluateToolCall(
    agent.Did,
    "execute_shell",
    new Dictionary<string, object>
    {
        ["cmd"] = "rm -rf /"
    });

Console.WriteLine($"Blocked call allowed? {blocked.Allowed}");
Console.WriteLine($"Blocked call reason: {blocked.Reason}");
```

#### ポイント

- **事前判定**: 実際にツールを実行する前に判定可能。ログ処理などに用いると効率的
- **詳細な理由**: `allowed.Reason` でポリシーのどのルールで判定されたかを確認可能

## ガバナンスポリシー解説

```yaml
apiVersion: governance.toolkit/v1
version: "1.0"
name: audit-blob-telemetry-policy
default_action: deny

rules:
  - name: allow-weather
    condition: "tool_name == 'GetWeather'"
    action: allow
    priority: 10

  - name: allow-time
    condition: "tool_name == 'GetTime'"
    action: allow
    priority: 10

  - name: allow-location
    condition: "tool_name == 'GetLocation'"
    action: allow
    priority: 10

  - name: block-shell
    condition: "tool_name == 'execute_shell'"
    action: deny
    priority: 100
```

### ポリシー戦略

- **デフォルトアクション: deny**: 明示的に許可されたツール以外は拒否（ホワイトリスト方式）
- **優先度ルール**: 優先度が高いほど先に評価。`block-shell` は優先度 100 で、明示的に拒否

## 実行方法

### 前提条件

以下の環境変数を設定します：

```powershell
$env:AZURE_OPENAI_ENDPOINT = "https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_DEPLOYMENT = "gpt-4o-mini"
$env:APPLICATIONINSIGHTS_CONNECTION_STRING = "InstrumentationKey=...;..."
$env:AZURE_STORAGE_ACCOUNT_NAME = "yourstorageaccount"
$env:AUDIT_STORAGE_CONTAINER = "agt-audit"
```

### 実行

```powershell
cd AGTAuditBlobTelemetryApp01
dotnet run
```

### 実行結果

```
==================================================
  AGT Audit to Blob + Telemetry to Application Insights
==================================================

Agent DID: did:mesh:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Agent Status: Active
Telemetry: Application Insights via AzureMonitorExporterOptions
Audit: Azure Blob Storage container 'agt-audit'

✓ OpenTelemetry providers initialized
  - Metrics: agent_governance
  - Traces: AgentGovernance.Examples.AuditBlobTelemetry
  - Logs: AgentGovernance.Examples

=== Agent Run 1 ===
[Audit -> Blob] PolicyCheck policy= tool=GetWeather agentId=did:mesh:...
  Role: assistant
    FunctionCall: GetWeather({"city":"Seattle"})
  Role: tool
    FunctionResult: call_xxx = Seattle is sunny, 22°C
  Role: assistant
    Text: The weather in Seattle is sunny with a temperature of 22°C.

=== Agent Run 2 ===
[Audit -> Blob] PolicyCheck policy= tool=GetTime agentId=did:mesh:...
  Role: assistant
    FunctionCall: GetTime({"timezone":"Tokyo"})
  Role: tool
    FunctionResult: call_yyy = Current time in Tokyo: 09:30:00 UTC
  Role: assistant
    Text: The current time in Tokyo is 09:30:00 UTC.

=== Direct Governance Demo - Allow Example ===
GetLocation call allowed? True
GetLocation call reason: Matches allow-location rule

=== Direct Governance Demo - Deny Example ===
Blocked call allowed? False
Blocked call reason: Matches block-shell rule (deny)

=== Flushing Telemetry to Application Insights ===
Flushing metrics...
Flushing traces...
Flushing logs...
Waiting 5 seconds for data transmission...
✓ Telemetry flush complete.
```

## セキュリティ考慮事項：監査ログのサニタイズ

本番環境では、ツール引数に **API キー、パスワード、個人識別情報（PII）** が混入する可能性があります。そのため、Blob に記録する前にマスキング処理を実施することが重要です。

**マスキング戦略:**

| 対象キー | マスキング例 | 説明 |
|---------|-----------|-------|
| `api_key`, `token` | `sk***...d7e2` | 最初と最後の2文字のみ表示 |
| `password`, `secret` | `***` | 完全に隠蔽 |
| `email` | `us***...com` | メールアドレスの中間をマスク |
| `cmd` (シェルコマンド) | 実行の可否判定が重要なため、マスク対象外 | ただし引数が機密を含む場合は別処理 |

## Blob Storage 監査ログの確認

Azure Portal で Blob Storage アカウントを開き、`agt-audit` コンテナの `governance-audit-YYYYMMDD.jsonl` ファイルを確認します。

内容例：

```json
{"EventId":"evt-42a6f9d598c44d56b15152699ac8eafa","Timestamp":"2026-07-02T01:47:32.0371686+00:00","Type":"PolicyViolation","AgentId":"did:agentmesh:weatheragent","SessionId":"maf-e2ed4a320f944bbf","PolicyName":null,"Data":{"message":"What is the weather in Seattle?","allowed":false,"action":"deny","reason":"No matching rules; default action is deny.","evaluation_ms":37.2383}}
{"EventId":"evt-7ff65a1c8bd84195a26722adc20ce2bf","Timestamp":"2026-07-02T01:47:41.1239754+00:00","Type":"PolicyViolation","AgentId":"did:agentmesh:weatheragent","SessionId":"maf-9f949b3838054324","PolicyName":null,"Data":{"message":"What time is it in Tokyo?","allowed":false,"action":"deny","reason":"No matching rules; default action is deny.","evaluation_ms":0.1392}}
{"EventId":"evt-baf03404cd8a477eaaa5e0af93413a74","Timestamp":"2026-07-02T01:47:48.501694+00:00","Type":"PolicyViolation","AgentId":"did:agentmesh:weatheragent","SessionId":"maf-e88d7382ed3b49a0","PolicyName":null,"Data":{"message":"Where is Paris located?","allowed":false,"action":"deny","reason":"No matching rules; default action is deny.","evaluation_ms":0.0919}}
{"EventId":"evt-107e6c8e18e14fdcb8272844e57c45b1","Timestamp":"2026-07-02T01:47:53.9755476+00:00","Type":"PolicyCheck","AgentId":"did:mesh:4c3ca93fec8b9d78fd05839694ed7df8","SessionId":"session-775002743f0f4d28","PolicyName":"allow-location","Data":{"tool_name":"GetLocation","allowed":true,"action":"allow","reason":"Matched rule \u0027allow-location\u0027 with action \u0027Allow\u0027.","evaluation_ms":4.3211,"arguments":{"city":"London"}}}
{"EventId":"evt-6bc7e6b8632a41cb885fc5e87097c6ac","Timestamp":"2026-07-02T01:48:02.4498334+00:00","Type":"ToolCallBlocked","AgentId":"did:mesh:4c3ca93fec8b9d78fd05839694ed7df8","SessionId":"session-bfda81e9ce30429a","PolicyName":"block-shell","Data":{"tool_name":"execute_shell","allowed":false,"action":"deny","reason":"Matched rule \u0027block-shell\u0027 with action \u0027Deny\u0027.","evaluation_ms":0.065,"arguments":{"cmd":"rm -rf /"}}}
```

### 活用方法

**KQL クエリ - Kusto Query Language**を使用してログを分析：

```kusto
//agent_governance. 配下メトリクスの許可/拒否件数集計
AppMetrics
| where Name startswith "agent_governance."
| where TimeGenerated > ago(24h)
| extend Decision = tostring(Properties.decision)
| summarize Count = sum(Sum) by Name, Decision
| order by Name asc, Decision asc
```

## Application Insights での確認

Azure Portal → Application Insights → **パフォーマンス** → **ロール** タブで、エージェントの実行性能を確認できます。

| メトリクス | 説明 |
|-----------|------|
| `agent_governance.tool_calls` | ツール呼び出し総数 |
| `agent_governance.policy_checks` | ポリシーチェック回数 |
| `agent_governance.tool_calls_allowed` | 許可されたツール呼び出し |
| `agent_governance.tool_calls_denied` | 拒否されたツール呼び出し |

## 実運用での活用シナリオ

### シナリオ 1: 異常アクセスパターンの検知

```csharp
// 特定のエージェントが短時間に多くのツール拒否を受けた場合、アラート発行
// 注意: PolicyName が null になりうるため、Type ベースと null-safe チェックを組み合わせ
var recentDenials = auditLogs
    .Where(l => l.Timestamp > DateTime.UtcNow.AddMinutes(-5))
    .Where(l => l.Type is "PolicyViolation" or "ToolCallBlocked"
             || (l.PolicyName?.Contains("deny") ?? false)
             || (l.PolicyName?.Contains("block") ?? false))
    .GroupBy(l => l.AgentId)
    .Where(g => g.Count() > 10);

foreach (var agentGroup in recentDenials)
{
    NotifySecurityTeam($"Agent {agentGroup.Key} attempted " +
        $"{agentGroup.Count()} forbidden operations in the last 5 minutes");
}
```

**注意点:**
- **Type ベースの判定**: JSONLサンプルを見ると、デフォルト拒否による `PolicyViolation` は `PolicyName` が `null` です。型ベースの判定を主軸にすることで、null チェックなしでの例外を防げます
- **null-safe オペレータ**: `PolicyName?.Contains()` と `?? false` を組み合わせることで、null 参照例外を回避しながら、ポリシー名ベースのフィルタリングも可能にしています

### シナリオ 2: エージェント動作の監視ダッシュボード

Application Insights のカスタムダッシュボードを作成し、リアルタイムで以下を表示：

- **エージェント数**: 現在活動中のエージェント数
- **拒否率**: 過去 1 時間のポリシー拒否の割合
- **平均応答時間**: ツール呼び出しの平均レイテンシ
- **エラー率**: 例外が発生したツール呼び出しの割合

## 本番環境での注意点

### 1. スケーラビリティ

大規模なエージェント群では、Blob Storage への書き込みがボトルネックになる可能性があります。

**対策**:
- Azure Data Lake Storage (ADLS) の階層的パーティショニング
- イベントハブ経由でのバッチ処理
- 複数の Append Blob に負荷分散

### 2. コスト最適化

Blob Storage に大量のログを保存すると、ストレージコストが増加します。

**対策**:
- 古いログの自動削除（ライフサイクルポリシー）
- ホットティア → クールティア → アーカイブへの段階的移行
- 必要な情報のみを保存（フィルタリング）

### 3. パフォーマンス

Application Insights へのテレメトリ出力が遅延を増加させる可能性があります。

**対策**:
- バッチ処理モード（デフォルト）を使用
- トレース数の制限（サンプリング）
- 非同期出力

## 次のステップ

本シリーズの続きでは、以下のトピックを予定しています：

1. **エージェント監査ログの分析**: KQL と Power BI を使用した高度なレポート
2. **インシデント対応**: 異常検知時の自動化されたレスポンス
3. **マルチエージェント管理**: 複数エージェントの一元管理とガバナンス

## まとめ

AGT の監査とテレメトリ統合により、以下を実現できます：

1. **完全な監査証跡**: すべてのポリシー判定と実行結果を記録
2. **リアルタイム監視**: Application Insights でエージェント動作を可視化
3. **異常検知**: パターン認識による早期警告
4. **本番環境対応**: スケーラブルで堅牢なアーキテクチャ

AI エージェントが重要な役割を担う本番環境では、**透明性** と **追跡可能性** が不可欠です。AGT の監査・テレメトリ機能を活用することで、安全かつ信頼性の高いエージェント運用を実現できます。

## 参考リンク

- [Agent Governance Toolkit - GitHub](https://github.com/microsoft/agent-governance-toolkit)
- [前回の記事: Agent Governance Toolkit の Agent Trust によるエージェント信頼性管理](https://zenn.dev/microsoft/articles/agent-governance-toolkit-trust-02)
- [Azure Blob Storage - Append Blob](https://learn.microsoft.com/ja-jp/azure/storage/blobs/storage-blobs-append)
- [Azure Monitor OpenTelemetry Exporter](https://learn.microsoft.com/ja-jp/azure/azure-monitor/app/opentelemetry-enable)
- [Application Insights - KQL クエリ](https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/kql-quick-reference)
- [サンプルコードリポジトリ](https://github.com/normalian/MyAGTSamples)

---

この記事が、AI エージェントの本番運用における監査・テレメトリの実装にお役に立つことを願っています。質問やご指摘があれば、ぜひコメント欄でお知らせください！
