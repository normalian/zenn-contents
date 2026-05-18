# Agent Governance Toolkit の監査とテレメトリ - 本番環境への道 - .NET 9 での実装入門

## はじめに

前回の記事では、Agent Governance Toolkit (AGT) の **Agent Trust** 機能を使った信頼スコアベースのアクセス制御について解説しました。今回は、その次のステップとして、**本番環境での監査ログ収集** と **テレメトリ統合** について、実装例を交えて解説します。

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
| **Blob監査**: `BlobAuditSink` | すべてのガバナンスイベントを Azure Blob Storage の Append Blob に追記ロック式で保存 |
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

1. **MAF エージェント実行**: ツール呼び出しが発生
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
- **JSONL 形式**: 1 行 = 1 レコード。ストリーミング処理やスプレッドシートへの取り込みに適している

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

- **OnAllEvents コールバック**: すべてのガバナンスイベント（PolicyCheck, ToolCallAllowed, ToolCallDenied など）をキャッチ
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

## Blob Storage 監査ログの確認

Azure Portal で Blob Storage アカウントを開き、`agt-audit` コンテナの `governance-audit-YYYYMMDD.jsonl` ファイルを確認します。

内容例：

```json
{"EventId":"evt-001","Timestamp":"2025-05-17T10:30:00Z","Type":"PolicyCheck","AgentId":"did:mesh:...","SessionId":"sess-001","PolicyName":"allow-weather","Data":{"tool_name":"GetWeather","city":"Seattle"}}
{"EventId":"evt-002","Timestamp":"2025-05-17T10:30:01Z","Type":"PolicyCheck","AgentId":"did:mesh:...","SessionId":"sess-001","PolicyName":"allow-time","Data":{"tool_name":"GetTime","timezone":"Tokyo"}}
{"EventId":"evt-003","Timestamp":"2025-05-17T10:30:02Z","Type":"PolicyCheck","AgentId":"did:mesh:...","SessionId":"sess-002","PolicyName":"block-shell","Data":{"tool_name":"execute_shell","cmd":"rm -rf /"}}
```

### 活用方法

**KQL クエリ（Kusto Query Language）**を使用してログを分析：

```kusto
// 過去 24 時間のポリシー拒否事例
customEvents
| where name == "GovernanceEvent.PolicyDenied"
| where timestamp > ago(24h)
| project timestamp, user_Id, tostring(customDimensions["policy.name"]), tostring(customDimensions["tool.name"])
| summarize count() by tostring(customDimensions["policy.name"])
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

### シナリオ 1: 定期的なコンプライアンスレポート

```csharp
// 監査ログから過去 7 日間の統計を抽出
var auditBlob = containerClient.GetBlobClient($"governance-audit-{DateTime.UtcNow.AddDays(-7):yyyyMMdd}.jsonl");
using (var download = await auditBlob.DownloadAsync())
{
    using var reader = new StreamReader(download.Value.Content);
    string line;
    int allowCount = 0, denyCount = 0;
    
    while ((line = await reader.ReadLineAsync()) != null)
    {
        var record = JsonSerializer.Deserialize<GovernanceAuditRecord>(line);
        if (record.Type == "PolicyCheck")
        {
            if (record.PolicyName.StartsWith("allow-"))
                allowCount++;
            else if (record.PolicyName.StartsWith("deny-") || record.PolicyName.StartsWith("block-"))
                denyCount++;
        }
    }
    
    Console.WriteLine($"Compliance Report - Last 7 Days");
    Console.WriteLine($"  Allowed Operations: {allowCount}");
    Console.WriteLine($"  Denied Operations: {denyCount}");
    Console.WriteLine($"  Denial Rate: {(double)denyCount / (allowCount + denyCount) * 100:F2}%");
}
```

### シナリオ 2: 異常アクセスパターンの検知

```csharp
// 特定のエージェントが短時間に多くのツール拒否を受けた場合、アラート発行
var recentDenials = auditLogs
    .Where(l => l.Timestamp > DateTime.UtcNow.AddMinutes(-5))
    .Where(l => l.PolicyName.Contains("deny") || l.PolicyName.Contains("block"))
    .GroupBy(l => l.AgentId)
    .Where(g => g.Count() > 10);

foreach (var agentGroup in recentDenials)
{
    NotifySecurityTeam($"Agent {agentGroup.Key} attempted " +
        $"{agentGroup.Count()} forbidden operations in the last 5 minutes");
}
```

### シナリオ 3: エージェント動作の監視ダッシュボード

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
3. **コンプライアンス対応**: 規制要件に対応した詳細なレポート生成
4. **異常検知**: パターン認識による早期警告
5. **本番環境対応**: スケーラブルで堅牢なアーキテクチャ

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
