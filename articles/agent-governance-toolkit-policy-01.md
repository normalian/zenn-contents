---
title: "AIエージェントに「門番」を置く — Agent Governance Toolkit 入門（.NET編）"
emoji: "🛡️"
type: "tech"
topics: ["dotnet", "azure", "ai", "agentgovernancetoolkit", "microsoft"]
published: true
publication_name: "microsoft"
---

## はじめに

AIエージェントが自律的にツールを呼び出す時代になりました。ファイルを読み書きし、APIを叩き、データベースを参照する——便利な反面、**誰が・何を・どんな条件で呼び出してよいか**を制御する仕組みがなければ、本番環境には置けません。

2026年4月にMicrosoftがオープンソースとして公開した **Agent Governance Toolkit（AGT）** は、まさにその「門番」をAIエージェントに組み込むためのツールキットです。

この記事では、以下のサンプルリポジトリをベースに、.NET（C#）での AGT 入門を解説します。

https://github.com/normalian/MyAGTSamples

---

## Agent Governance Toolkit とは

AGT は **AIエージェントのランタイムセキュリティ**を実現するオープンソースプロジェクトです（MIT ライセンス）。

主な機能は次のとおりです（他にもいくつもありあｍす）。

- **Policy Engine** — YAML/JSON で記述したルールをもとにツール呼び出しを allow/deny する
- **実行リング** — エージェントの信頼スコアに応じて権限を段階的に制限する
- **プロンプトインジェクション検知** — ツール入出力に埋め込まれた不正命令を検出する
- **サーキットブレーカー** — 連続失敗時に自動的に保護状態へ移行する
- **レート制限** — ツールごとの呼び出し頻度上限を設定する
- **Zero-Trust ID** — DIDベースのエージェントアイデンティティ管理
- **OpenTelemetry対応** — Azure Monitor/Prometheus/Datadog へのメトリクス出力

Python・TypeScript・Rust・Go・**.NET** の 5言語に対応しており、NuGet パッケージ `Microsoft.AgentGovernance` 一つで導入できます。

---

## サンプル構成

リポジトリには 2 つのサンプルが含まれています。

```
MyAGTSamples/
├── AGTPolicyApp01/          # Step 1: ポリシー単体の評価
│   ├── Program.cs
│   └── policies/
│       └── default.yaml
└── AGTPolicyWithMAFApp02/   # Step 2: Microsoft Agent Framework との統合
    ├── Program.cs
    └── policies/
        └── default.yaml
```

本記事では、この 2 ステップを順番に解説します。

---

## Step 1: ポリシーエンジン単体を試す（AGTPolicyApp01）

### 環境準備

```bash
# .NET 9 SDK が必要
dotnet --version

# パッケージのインストール
dotnet add package Microsoft.AgentGovernance
```

### ポリシーファイルを作成する

`policies/default.yaml` を以下の内容で作成します。

```yaml
apiVersion: governance.toolkit/v1
version: "1.0"
name: default-governance-policy

# デフォルトは deny（明示的に許可されていないものはすべて拒否）
default_action: deny

rules:
  # allowed_tools に含まれるツールは許可
  - name: allow-safe-tools
    condition: "tool_name in allowed_tools"
    action: allow
    priority: 10

  # blocked_tools に含まれるツールは拒否
  - name: block-dangerous-tools
    condition: "tool_name in blocked_tools"
    action: deny
    priority: 100

  # http_request は 100回/分 のレート制限
  - name: rate-limit-http
    condition: "tool_name == 'http_request'"
    action: rate_limit
    limit: "100/minute"
```

ポイントは `default_action: deny` です。**明示的に許可されていないツールはすべてブロック**される、ホワイトリスト方式です。

### Program.cs — ポリシー評価の基本

```csharp
using AgentGovernance;
using AgentGovernance.Policy;

// GovernanceKernel を初期化
var kernel = new GovernanceKernel(new GovernanceOptions
{
    PolicyPaths = new() { "policies/default.yaml" },
    ConflictStrategy = ConflictResolutionStrategy.DenyOverrides,
    EnableRings = true,                    // 実行リングを有効化
    EnablePromptInjectionDetection = true, // プロンプトインジェクション検知
    EnableCircuitBreaker = true,           // サーキットブレーカー
});

// ツール呼び出しを評価する
var result = kernel.EvaluateToolCall(
    agentId: "did:mesh:analyst-001",
    toolName: "file_write",
    args: new() { ["path"] = "/etc/config" }
);

if (!result.Allowed)
{
    Console.WriteLine($"❌ Blocked: {result.Reason}");
    return;
}

// 許可された場合のみツールを実行
Console.WriteLine("✅ Allowed — proceeding with tool execution");
```

### 実行してみる

```bash
dotnet run --project AGTPolicyApp01/AGTPolicyApp01.csproj
```

`file_write` は `allowed_tools` に含まれていないため、デフォルトポリシーに従ってブロックされます。

```
❌ Blocked: tool 'file_write' not in allowed_tools (default_action: deny)
```

`file_read` などを `allowed_tools` に追加すれば許可できます。このように、**エージェントを実行する前にポリシーを評価してから**ツールを呼び出す、というパターンが AGT の基本です。

---

## Step 2: Microsoft Agent Framework との統合（AGTPolicyWithMAFApp02）

Step 1 ではポリシー評価を手動で呼び出しましたが、実際のエージェントフレームワークと統合する場合は `.WithGovernance()` 拡張メソッドを使うと、ツール呼び出しへのポリシー評価が**自動的に差し込まれます**。

### 追加パッケージのインストール

```bash
dotnet add package Microsoft.AgentGovernance
dotnet add package Microsoft.AgentGovernance.Extensions.Microsoft.Agents
dotnet add package Microsoft.Agents.AI
dotnet add package Azure.AI.OpenAI
dotnet add package Azure.Identity
```

### 事前準備：環境変数

Azure OpenAI の接続情報を環境変数に設定します。

```bash
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_DEPLOYMENT_NAME="gpt-4.1-mini"
```

認証は `AzureCliCredential` を使うため、`az login` 済みであれば追加設定不要です。

### Program.cs — MAF エージェントへのガバナンス統合

```csharp
using AgentGovernance;
using AgentGovernance.Extensions.Microsoft.Agents;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using System.ComponentModel;

namespace AGTPolicyWithMAFApp02
{
    class Program
    {
        // エージェントが呼び出せるツールを定義
        [Description("Get weather information for a location.")]
        static string GetWeather(
            [Description("Target city name")] string city)
        {
            return $"{city} is sunny, 22°C";
        }

        static async Task Main(string[] args)
        {
            string agentName = "myagent";

            var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
                ?? throw new InvalidOperationException("AZURE_OPENAI_ENDPOINT is not set.");
            var deploymentName =
                Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME")
                ?? "gpt-4.1-mini";

            // ① GovernanceKernel を構成（Step 1 と同じ）
            var kernel = new GovernanceKernel(new GovernanceOptions
            {
                PolicyPaths = new() { "policies/default.yaml" },
                EnablePromptInjectionDetection = true,
            });

            // ② ガバナンスイベントを購読（デバッグ・監査用）
            kernel.OnAllEvents(evt =>
            {
                Console.WriteLine(
                    $"[Governance Event] Type: {evt.Type}, " +
                    $"Tool: {evt.PolicyName}, Agent: {evt.AgentId}");
            });

            // ③ MAF エージェントを構築し、.WithGovernance() でガバナンスを注入
            AIAgent agent = new AzureOpenAIClient(
                    new Uri(endpoint), new AzureCliCredential())
                .GetChatClient(deploymentName)
                .AsIChatClient()
                .AsAIAgent(
                    name: agentName,
                    instructions: "You are a helpful assistant.",
                    tools: [
                        AIFunctionFactory.Create(GetWeather, name: "GetWeather")
                    ]
                )
                .WithGovernance(
                    kernel,
                    new AgentFrameworkGovernanceOptions
                    {
                        DefaultAgentId = agentName,
                        EnableFunctionMiddleware = true,

                        // ツールがブロックされたときの応答をカスタマイズ
                        BlockedToolResultFactory = toolResult =>
                        {
                            Console.WriteLine(
                                $"[BLOCKED TOOL] {toolResult.AuditEntry.PolicyName}: " +
                                $"{toolResult.Reason}");
                            return $"'{toolResult.AuditEntry.AgentId}' " +
                                   $"was blocked for tool calling";
                        }
                    });

            // ④ エージェントを実行
            // GetWeather はポリシーでブロックされているため、ツール呼び出しが阻止される
            var response1 = await agent.RunAsync("What is the weather in Seattle?");
            Console.WriteLine($"Agent response1: {response1}");

            // ツール呼び出しが不要な質問は通常通り回答される
            var response2 = await agent.RunAsync("Who are you?");
            Console.WriteLine($"Agent response2: {response2}");
        }
    }
}
```

### 実行してみる

```bash
dotnet run --project AGTPolicyWithMAFApp02/AGTPolicyWithMAFApp02.csproj
```

`GetWeather` が `allowed_tools` に含まれていなければ、ポリシーに従ってブロックされます。

```
[Governance Event] Type: ToolCallEvaluated, Tool: default-governance-policy, Agent: myagent
[BLOCKED TOOL] default-governance-policy: tool 'GetWeather' not in allowed_tools
Agent response1: 'myagent' was blocked for tool calling
Agent response2: I am an AI assistant. How can I help you?
```

ポリシーに `GetWeather` を追加すれば、ツール呼び出しが許可されて天気情報が返ります。

---

## コードの重要ポイント解説

### ① `GovernanceKernel` — ガバナンスの中心

```csharp
var kernel = new GovernanceKernel(new GovernanceOptions
{
    PolicyPaths = new() { "policies/default.yaml" }, // ポリシーファイルのパス
    EnablePromptInjectionDetection = true,           // 注入攻撃の検知
    EnableRings = true,                              // 信頼スコアに基づく実行制限
    EnableCircuitBreaker = true,                     // 連続失敗時の自動保護
});
```

`GovernanceOptions` で有効化できる主な機能です。まず `PolicyPaths` と `EnablePromptInjectionDetection` だけ有効にして始めるのが手軽です。

### ② `kernel.OnAllEvents()` — オブザーバビリティ

```csharp
kernel.OnAllEvents(evt =>
{
    Console.WriteLine($"[Governance Event] Type: {evt.Type}, ...");
});
```

すべてのガバナンス判断がイベントとして流れてきます。本番環境では Application Insights や OpenTelemetry に繋ぐことで、「どのエージェントが・何を・なぜブロックされたか」を可視化できます。

### ③ `.WithGovernance()` — フレームワーク統合の要

```csharp
agent.WithGovernance(kernel, new AgentFrameworkGovernanceOptions
{
    DefaultAgentId = agentName,
    EnableFunctionMiddleware = true,     // ツール呼び出しを自動インターセプト
    BlockedToolResultFactory = result => // ブロック時の応答を差し替え
    {
        return "ポリシーにより実行できませんでした";
    }
});
```

`.WithGovernance()` は MAF エージェントのツール呼び出しパイプラインに自動で割り込みます。エージェントコードを書き換えることなく、後からガバナンスを追加できるのが大きな利点です。

### ④ ポリシーで挙動をコントロール

`GetWeather` を許可したい場合は、ポリシーの `allowed_tools` に追加するだけです。

```yaml
rules:
  - name: allow-safe-tools
    condition: "tool_name in allowed_tools"
    action: allow
    priority: 10

allowed_tools:
  - GetWeather   # ← これを追加するだけでツールが使えるようになる
  - file_read
```

コードを変更せず、YAMLだけで制御が変わります。これが「ポリシー駆動のガバナンス」の本質です。

---

## AGT が解決するセキュリティリスク

AGT は OWASP Agentic AI Top 10 に対応しています。本サンプルで体験できるのは以下のリスクへの対応です。

| リスク | AGT の対応 | 本記事での該当箇所 |
|---|---|---|
| ツールの悪用 | ホワイトリストポリシーによるブロック | `default_action: deny` |
| プロンプトインジェクション | 入力のスキャンと検知 | `EnablePromptInjectionDetection` |
| 過剰な呼び出し | レート制限 | `limit: "100/minute"` |
| 不正なエージェント | DIDベースID + 信頼スコア | `agentId` パラメータ |

---

## まとめ

Agent Governance Toolkit を使うと、AIエージェントへのガバナンス追加は非常にシンプルです。

1. **ポリシーYAMLを書く** — `default_action: deny` から始めて、許可するものだけ列挙する
2. **`GovernanceKernel` を初期化する** — オプションで必要な機能を有効化する
3. **`.WithGovernance()` でエージェントに注入する** — エージェントコードは触らなくてよい

「AIエージェントを本番に出したいが、何をどこまで実行させていいか不安」という場面で、AGT は即戦力になります。

サンプルリポジトリは以下をご参照ください。

https://github.com/normalian/MyAGTSamples

---

## 参考リンク

- [microsoft/agent-governance-toolkit（公式リポジトリ）](https://github.com/microsoft/agent-governance-toolkit)
- [Introducing the Agent Governance Toolkit（Microsoft Open Source Blog）](https://opensource.microsoft.com/blog/2026/04/02/introducing-the-agent-governance-toolkit-open-source-runtime-security-for-ai-agents/)
- [Governing MCP tool calls in .NET with the AGT（.NET Blog）](https://devblogs.microsoft.com/dotnet/governing-mcp-tool-calls-in-dotnet-with-the-agent-governance-toolkit/)
- [Tutorial 19 — .NET SDK（公式チュートリアル）](https://github.com/microsoft/agent-governance-toolkit/blob/main/docs/tutorials/19-dotnet-sdk.md)
