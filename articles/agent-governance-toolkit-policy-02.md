---
title: "ファイル操作エージェントに「読み取り専用」の境界線を引く — Agent Framework Harness と Agent Governance Toolkit"
emoji: "🛡️"
type: "tech"
topics: ["dotnet", "azure", "ai", "agentgovernancetoolkit", "microsoft"]
published: false
---

## はじめに

[前回の記事](https://zenn.dev/microsoft/articles/agent-governance-toolkit-policy-01) では、Agent Governance Toolkit（AGT）のポリシーエンジンを単体で評価し、Microsoft Agent Framework のエージェントに `.WithGovernance()` で組み込む方法を紹介しました。

今回は一歩進めて、Microsoft Agent Framework の **Harness** と AGT を組み合わせます。題材は、エージェントがローカルの作業ディレクトリを操作するファイルアクセスエージェントです。

エージェントにファイルを扱わせると、次のような要件がすぐに発生します。

- ファイルの一覧表示・読み取り・検索は許可したい
- ファイルの作成・削除・置換は許可したくない
- その判断をエージェントのプロンプトだけに任せたくない
- 拒否されたツール呼び出しを監査ログで追跡したい

このような場合、Harness が「何を実行できるか」を提供し、AGT が「その実行を許可するか」を判断する分担が有効です。

本記事では、次のサンプルを参照します。

https://github.com/normalian/MyAGTSamples/tree/main/AGTPolicywithMAFApp03

## Harness と AGT の役割分担

まず、2 つのコンポーネントの役割を整理します。

| コンポーネント | 担当すること |
|---|---|
| Agent Framework Harness | ファイルアクセスなど、エージェントが利用できる機能とツールをまとめて提供する |
| Agent Governance Toolkit | ツール呼び出しをポリシーに照らして許可・拒否し、判断をイベントとして出力する |

Harness はエージェントの能力を構成する仕組みです。サンプルでは `AsHarnessAgent()` に `FileSystemAgentFileStore` を渡し、`file_access_ls`、`file_access_read`、`file_access_grep` などのツールをエージェントに公開しています。

一方、Harness がツールを公開したからといって、すべての呼び出しを実行してよいとは限りません。AGT の `.WithGovernance()` を後段に適用すると、エージェントがツールを呼び出す直前のパイプラインにガバナンスの判定を挿入できます。

つまり、プロンプトに「削除しないでください」と書くだけではなく、**削除ツールを呼び出しても実行層で拒否できる**ようになります。

## Microsoft Agent Framework Harness を詳しく理解する

ここで、今回利用している Harness そのものをもう少し詳しく見ておきます。

`Microsoft.Agents.AI.Harness` は、単にファイル操作ツールを追加するパッケージではありません。ツールを何度も呼び出しながら、長時間のタスクを進めるエージェントで必要になりやすい処理を、定型的なパイプラインとして組み立てるための拡張です。

Agent Framework の通常の `AIAgent` でも、ツール、会話履歴、実行ループ、コンテキスト管理を個別に構成できます。しかし、次の処理をアプリケーションごとに繰り返し実装すると、コードが複雑になります。

- モデルが返した function call を実行し、結果をモデルへ返すループ
- ツール実行を含む会話履歴の保存
- 長いタスクで膨らむコンテキストの圧縮
- Todo、ファイルアクセス、サブエージェントなどの補助機能
- ツール実行前の承認フロー

Harness は、このような長時間タスク向けの部品を `IChatClient` から `AIAgent` へまとめて接続します。

### `HarnessAgent` の内部で組み立てられるもの

参照記事では、Harness の中核を次の 3 つの層として説明しています。

```text
IChatClient
   │
   ├─ FunctionInvokingChatClient
   │    └─ function call と結果を往復させる自動実行ループ
   │
   ├─ PerServiceCallChatHistoryPersistingChatClient
   │    └─ サービス呼び出し単位で履歴を保存
   │
   └─ AIContextProviderChatClient
        └─ Context Provider とコンパクションによる文脈管理
```

これらを手作業で組み合わせる代わりに、次の拡張メソッドを呼び出します。

```csharp
AIAgent agent = chatClient.AsHarnessAgent(
    maxContextWindowTokens,
    maxOutputTokens,
    new HarnessAgentOptions
    {
        Name = "GovernedFileAccessAgent",
        ChatOptions = new ChatOptions
        {
            Instructions = "ファイルを調査するエージェントです。",
            Tools = [/* 独自ツール */],
        },
    });
```

`AsHarnessAgent()` は、`IChatClient` が手元にあれば、長時間タスクを想定した `HarnessAgent` を組み立てる入口になります。今回のサンプルでは、この呼び出しに `FileSystemAgentFileStore` と複数のオプションを追加しています。

> `Microsoft.Agents.AI.Harness` と関連する `AIContextProvider` は、参照記事の時点では実験的機能です。API やツール名、パッケージバージョンは変更される可能性があるため、導入時は使用するバージョンの公式ドキュメントとリリースノートを確認してください。

### コンテキストウィンドウの圧縮

長時間のエージェントでは、ユーザーの指示、モデルの応答、function call、function result が履歴に蓄積します。履歴がモデルのコンテキスト上限に近づくと、新しいツール呼び出しや結果を入れられなくなります。

Harness は `maxContextWindowTokens` と `maxOutputTokens` を基準に、入力に使えるバジェットを計算します。

```csharp
const int maxContextWindowTokens = 1_050_000;
const int maxOutputTokens = 128_000;

AIAgent agent = chatClient.AsHarnessAgent(
    maxContextWindowTokens,
    maxOutputTokens,
    new HarnessAgentOptions { /* ... */ });
```

概念的には、モデルのコンテキスト上限から次の応答用のトークンを差し引いた範囲が、会話履歴とツール結果に使える入力バジェットになります。履歴が膨らむと、Harness のコンパクション機構が古い履歴を圧縮し、以後の推論に必要な情報を残しながらコンテキストを小さくします。

このため、毎回 `messages` を自分で切り詰めたり、過去の tool result を手動で要約したりするコードを書く必要が減ります。ただし、コンパクションは「何でも完全に保持する」機能ではありません。重要な状態を失わせたくない場合は、Todo やファイルメモリなど、構造化された状態を使って明示的に保存する設計が重要です。

## サンプルの構成

サンプルの主要部分は次のとおりです。

```text
AGTPolicywithMAFApp03/
├── AGTPolicywithMAFApp03.csproj
├── Program.cs
├── policies/
│   └── default.yaml
└── working/
    ├── sample.txt
    └── notes/
        └── details.txt
```

`working` はエージェントがアクセスするファイルストアです。プロジェクトファイルでは、ポリシーとこのディレクトリをビルド出力へコピーしています。

```xml
<ItemGroup>
  <None Include="policies\default.yaml"
        CopyToOutputDirectory="PreserveNewest" />
  <None Include="working\**\*"
        CopyToOutputDirectory="PreserveNewest" />
</ItemGroup>
```

実行時は `AppContext.BaseDirectory` を基準にパスを組み立てるため、実行ディレクトリに依存しにくい構成です。

## 環境とパッケージ

サンプルは .NET 10 を対象にしています。使用している主なパッケージは次のとおりです（バージョンはサンプル作成時のものです）。

```xml
<PackageReference Include="Azure.AI.OpenAI" Version="2.9.0-beta.1" />
<PackageReference Include="Azure.Identity" Version="1.21.0" />
<PackageReference Include="Microsoft.AgentGovernance" Version="5.0.0" />
<PackageReference Include="Microsoft.AgentGovernance.Extensions.Microsoft.Agents"
                  Version="5.0.0" />
<PackageReference Include="Microsoft.Agents.AI" Version="1.17.0" />
<PackageReference Include="Microsoft.Agents.AI.Harness" Version="1.17.0" />
<PackageReference Include="Microsoft.Agents.AI.OpenAI" Version="1.17.0" />
```

Azure OpenAI のエンドポイントを設定し、Azure CLI で認証します。

```bash
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_DEPLOYMENT_NAME="gpt-5-mini"
az login
```

Windows PowerShell の場合は次のように設定できます。

```powershell
$env:AZURE_OPENAI_ENDPOINT = "https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_DEPLOYMENT_NAME = "gpt-5-mini"
az login
```

`AzureCliCredential` を使用するため、アプリケーションにキーを埋め込む必要はありません。

## ポリシーでファイル操作を制御する

`policies/default.yaml` では、読み取り系のツールを許可し、変更系のツールを拒否しています。

```yaml
apiVersion: governance.toolkit/v1
version: "1.0"
name: governed-file-access-policy

# このサンプルでは、明示的な deny 以外は許可する
default_action: allow

rules:
  - name: allow-file-access-read
    condition: "tool_name == 'file_access_read'"
    action: allow
    priority: 10

  - name: allow-file-access-ls
    condition: "tool_name == 'file_access_ls'"
    action: allow
    priority: 10

  - name: allow-file-access-grep
    condition: "tool_name == 'file_access_grep'"
    action: allow
    priority: 10

  - name: deny-file-access-write
    condition: "tool_name == 'file_access_write'"
    action: deny
    priority: 100

  - name: deny-file-access-replace
    condition: "tool_name == 'file_access_replace'"
    action: deny
    priority: 100

  - name: deny-file-access-replace-lines
    condition: "tool_name == 'file_access_replace_lines'"
    action: deny
    priority: 100

  - name: deny-file-access-delete
    condition: "tool_name == 'file_access_delete'"
    action: deny
    priority: 100
```

ここで重要なのは、`default_action` と `priority` の組み合わせです。

このサンプルは `default_action: allow` なので、ポリシーにないツールは基本的に許可されます。そのうえで、変更系ツールには優先度の高い `deny` を設定しています。`ConflictStrategy.DenyOverrides` も指定しているため、許可と拒否が競合した場合は拒否が優先されます。

より厳格なホワイトリスト方式にしたい場合は、`default_action: deny` に変更し、必要なツールを明示的に `allow` する設計にできます。本番環境では、Harness のバージョンアップで新しいツールが追加されても意図せず実行されないよう、ホワイトリスト方式を検討するとよいでしょう。

なお、`tool_name` は Harness が公開する function 名です。`file_access_read` と `file_access_Read` は別の名前なので、ポリシーの文字列は実際のツール名と完全に一致させる必要があります。

## Harness エージェントに AGT を組み込む

`Program.cs` の中心部分を抜き出すと、次のようになります。

```csharp
using AgentGovernance;
using AgentGovernance.Extensions.Microsoft.Agents;
using AgentGovernance.Policy;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException(
        "AZURE_OPENAI_ENDPOINT is not set.");
var deploymentName =
    Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME")
    ?? "gpt-5-mini";
var workingDirectory =
    Path.Combine(AppContext.BaseDirectory, "working");

var kernel = new GovernanceKernel(new GovernanceOptions
{
    PolicyPaths =
    [
        Path.Combine(AppContext.BaseDirectory, "policies", "default.yaml")
    ],
    ConflictStrategy = ConflictResolutionStrategy.DenyOverrides,
});

kernel.OnAllEvents(evt =>
{
    Console.WriteLine(
        $"[Governance] Type: {evt.Type}, " +
        $"Tool: {evt.PolicyName}, Agent: {evt.AgentId}");
});

AIAgent agent = new AzureOpenAIClient(
        new Uri(endpoint),
        new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsIChatClient()
    .AsHarnessAgent(new HarnessAgentOptions
    {
        Name = "GovernedFileAccessAgent",
        Description =
            "Demonstrates governed read-only access to sample files.",
        FileAccessStore =
            new FileSystemAgentFileStore(workingDirectory),
        FileAccessProviderOptions = new FileAccessProviderOptions
        {
            DisableReadOnlyToolApproval = true,
            DisableWriteToolApproval = true,
        },
        DisableTodoProvider = true,
        DisableAgentModeProvider = true,
        DisableAgentSkillsProvider = true,
        DisableFileMemory = true,
        DisableWebSearch = true,
        ChatOptions = new ChatOptions
        {
            Instructions =
                """
                You are a file access governance demonstration agent.
                Use the file_access_* tools to inspect the sample files.
                Read operations are allowed.
                Write, delete, and replace operations are denied by governance.
                """,
        },
    })
    .WithGovernance(
        kernel,
        new AgentFrameworkGovernanceOptions
        {
            DefaultAgentId = "governed-file-access-agent",
            EnableFunctionMiddleware = true,
            BlockedToolResultFactory = toolResult =>
            {
                Console.WriteLine(
                    $"[BLOCKED by Governance] " +
                    $"{toolResult.AuditEntry.PolicyName}: " +
                    $"{toolResult.Reason}");
                return
                    $"Tool call blocked by governance policy: " +
                    $"{toolResult.Reason}";
            },
        });
```

### `AsHarnessAgent()` が提供する機能

`AsHarnessAgent()` に `FileSystemAgentFileStore` を設定すると、エージェントはファイル操作用のツールを利用できるようになります。

このサンプルでは、ファイル操作以外の機能を無効にしています。

```csharp
DisableTodoProvider = true,
DisableAgentModeProvider = true,
DisableAgentSkillsProvider = true,
DisableFileMemory = true,
DisableWebSearch = true,
```

これはデモの対象をファイルアクセスに絞るためです。実際のアプリケーションでは、必要な機能だけを有効にし、それぞれのツールに対応するポリシーを用意します。機能を増やすほど、ポリシーの対象と監査項目も増える点に注意してください。

### `FileAccessProviderOptions` と AGT の違い

`DisableReadOnlyToolApproval` や `DisableWriteToolApproval` は Harness 側の承認フローに関する設定です。これらを無効にしても、AGT のポリシー評価が無効になるわけではありません。

Harness は「ツールをどう提供するか」、AGT は「提供されたツール呼び出しを実行してよいか」を担当します。承認 UI や人手による確認と、ポリシーによる強制的な拒否は、代替ではなく組み合わせて使える別の防御層です。

### `.WithGovernance()` が境界線になる

次の部分が統合の要です。

```csharp
.WithGovernance(
    kernel,
    new AgentFrameworkGovernanceOptions
    {
        DefaultAgentId = "governed-file-access-agent",
        EnableFunctionMiddleware = true,
    })
```

`EnableFunctionMiddleware = true` により、エージェントの function 呼び出しを AGT がインターセプトします。エージェントの推論結果が「書き込みを実行する」となっても、実際のツール実行前にポリシーへ照会され、deny なら実行されません。

`BlockedToolResultFactory` を指定すると、拒否時にエージェントへ返す結果をアプリケーション側で制御できます。ユーザー向けの説明を返したり、監査 ID を含めたりする場合に利用できます。

## 2 つの Agent ID に注意する

このサンプルには、意図的に異なる 2 つの識別子があります。

```csharp
Name = "GovernedFileAccessAgent"
```

これは Harness / Agent Framework 側のエージェント名です。

```csharp
DefaultAgentId = "governed-file-access-agent"
```

こちらは AGT がポリシー評価と監査イベントに使用するエージェント識別子です。

同じ値にする必要はありません。むしろ、フレームワーク上の表示名と、ガバナンス・監査用の安定した ID を分離しておくと、表示名を変更しても監査ログの追跡キーを維持できます。複数のエージェントを運用する場合は、テナントや環境を含めた一意な ID 体系を決めておくとよいでしょう。

## 実行して動作を確認する

サンプルは、読み取り、書き込み、削除の 3 つを順番に試します。

```csharp
Console.WriteLine("=== Allowed read ===");
await RunAndPrintAsync(
    agent,
    "Use file_access_ls and file_access_read to list " +
    "and read sample.txt. Do not modify any files.");

Console.WriteLine("\n=== Blocked write ===");
await RunAndPrintAsync(
    agent,
    "Use file_access_write to create blocked-write.txt " +
    "with the text 'this write must be denied'.");

Console.WriteLine("\n=== Blocked delete ===");
await RunAndPrintAsync(
    agent,
    "Use file_access_delete to delete sample.txt. " +
    "This operation must be denied by governance.");
```

実行コマンドは次のとおりです。

```bash
dotnet run --project AGTPolicywithMAFApp03/AGTPolicywithMAFApp03.csproj
```

期待する結果は次の状態です。

| 操作 | AGT の判断 | ファイルへの影響 |
|---|---|---|
| `file_access_ls` / `file_access_read` | allow | `sample.txt` を一覧・読み取りできる |
| `file_access_write` | deny | `blocked-write.txt` は作成されない |
| `file_access_delete` | deny | `sample.txt` は削除されない |

拒否された場合、`BlockedToolResultFactory` によって次のような結果がエージェントへ返されます。

```text
Tool call blocked by governance policy: ...
```

重要なのは、エージェントが拒否されたことを説明するだけではありません。**拒否されたツール自体が実行されていない**ことです。プロンプトインジェクションなどでエージェントの指示が変わっても、ポリシーが維持される限り、書き込みや削除の境界線は実行層に残ります。

## ガバナンスイベントを監査に利用する

サンプルでは、すべてのイベントをコンソールへ出力しています。

```csharp
kernel.OnAllEvents(evt =>
{
    Console.WriteLine(
        $"[Governance] Type: {evt.Type}, " +
        $"Tool: {evt.PolicyName}, Agent: {evt.AgentId}");
});
```

開発中は動作確認に使えますが、本番では Application Insights や OpenTelemetry などへ接続するのが実用的です。少なくとも次の情報を記録できるようにすると、調査しやすくなります。

- エージェント ID
- ツール名
- 適用されたポリシー名
- allow / deny の結果
- 拒否理由
- リクエストを識別する相関 ID

ログにはファイルの内容や機密性の高い引数をそのまま記録しない設計も必要です。監査のために必要な情報と、保存してはいけないデータを分けて考えましょう。

## `default_action: allow` と `deny` の使い分け

サンプルは、説明を分かりやすくするため `default_action: allow` を採用しています。Harness が提供する読み取り・書き込み・削除・置換のうち、危険なものだけを deny する構成です。

ただし、実運用では次のように使い分けます。

### deny リスト方式

```yaml
default_action: allow
```

既存のツールを広く利用しつつ、危険な操作だけを明示的に禁止する方式です。導入は簡単ですが、新しいツールが追加されたときに許可される可能性があります。

### allow リスト方式

```yaml
default_action: deny
```

必要なツールをすべて明示的に許可する方式です。初期設定は増えますが、未知のツールを安全側に倒せます。

ファイル操作を本番データに対して実行する場合は、`default_action: deny` を基本にし、読み取り対象のツールとパス条件を段階的に許可する設計が向いています。さらに、エージェントごとの ID、環境ごとのポリシー、レート制限、サーキットブレーカーなどを組み合わせると、より強固な境界を作れます。

## この組み合わせが有用な理由

Harness と AGT の組み合わせには、主に 4 つの利点があります。

1. **能力と権限を分離できる**  
   Harness の構成を変えずに、YAML のポリシーだけで実行可否を変更できます。

2. **プロンプトではなく実行層で拒否できる**  
   エージェントの指示や会話履歴が変化しても、deny ルールはツール呼び出しの境界で適用されます。

3. **監査可能な判断になる**  
   どのエージェントのどのツール呼び出しが、どのルールに一致して拒否されたかをイベントとして扱えます。

4. **承認フローと強制ポリシーを併用できる**  
   Harness の承認機能をユーザー体験のために使い、AGT を越えてはいけない安全境界として使えます。

これはファイルアクセスに限った話ではありません。データベース更新、外部 API の呼び出し、チケットのクローズ、メール送信など、取り消しにくい操作をエージェントに委ねる場合にも同じ考え方を適用できます。

## まとめ

今回のサンプルでは、次の流れで読み取り専用のファイルアクセスエージェントを構成しました。

1. Harness の `FileAccessProvider` でファイル操作ツールを公開する
2. AGT の YAML で読み取りを許可し、書き込み・削除・置換を拒否する
3. `.WithGovernance()` と Function Middleware でツール呼び出しにポリシーを挿入する
4. ガバナンスイベントとブロック時の結果を監査・ユーザー通知に利用する

Harness はエージェントに能力を与え、Agent Governance Toolkit はその能力に境界線を引きます。AI エージェントを本番環境へ近づけるには、便利なツールを追加することだけでなく、**そのツールをいつ、誰が、どの条件で実行できるかをコードの外側から制御できること**が重要です。

今回のサンプルコードはこちらです。

https://github.com/normalian/MyAGTSamples/tree/main/AGTPolicywithMAFApp03

## 参考リンク

- [前回の記事 — AIエージェントに「門番」を置く](https://zenn.dev/microsoft/articles/agent-governance-toolkit-policy-01)
- [MyAGTSamples — AGTPolicywithMAFApp03](https://github.com/normalian/MyAGTSamples/tree/main/AGTPolicywithMAFApp03)
- [Microsoft Agent Governance Toolkit](https://github.com/microsoft/agent-governance-toolkit)
- [Microsoft Agent Framework](https://github.com/microsoft/agent-framework)
- [Tutorial 43 — .NET MAF Hook Integration](https://microsoft.github.io/agent-governance-toolkit/tutorials/43-dotnet-maf-hook-integration/)
