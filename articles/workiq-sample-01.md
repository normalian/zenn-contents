---
title: "WorkIQ API を利用して自分の活動について相談してみる"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["SemanticKernel", "AI", "Java"]
published: true
publication_name: "microsoft"
---

皆様、Work IQ は使っていますでしょうか？ Microsoft が推進する IQ 群（ Fabric IQ, Foundry ID, Work IQ, Web IQ）の中で、もっともよく利用しているはずなのに、もっとも意識して使っていない機能といっても過言ではないと思っています。Work IQ は Microsoft Office 365 上で個人が蓄積したデータを読み取った結果を RAG の様な形で参照可能人していますが、明示的に呼び出した記憶のある方は皆無なのではないでしょうか。
それもそのはず、普段の M365 Copilot 利用時には内部で勝手に呼んでくれています。Work IQ で蓄積されている個人の業務遂行時の情報を取得し、その結果をもって普段の Teams での会話や Outlook のメールなどを見て「お前、今週はこのフォローアップ忘れてんぞ」といったような会話がエージェントとして出来るわけです。

実は Work IQ は我々自身が明示的に呼び出すことが可能です。以下の記事にも記載がありますが、A2A/MCP/REST 形式での API 呼び出しが可能になっています。
- [Work IQ overview](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/)

加えて、既に C# 等でのアンプルアプリが以下の様に公開されています。こちらを利用することで、自身のアプリケーションから明示的に Work IQ を呼び出すことができるのが分かるでしょう。
- [GitHub - microsoft / work-iq-samples](https://github.com/microsoft/work-iq-samples/tree/main)

## 実業務で想定される利用法
Work IQ の根幹である「Microsoft Office 365 上で蓄積したデータを読み取る」という機能は非常に強力ですが、すなわち「本番環境で M365 が動いている Entra ID テナントの API を呼ばせて」と言うことになります。
実際には多くのプロジェクトを見ていると「Azure 環境」と「M365 環境」は Entra ID テナントを分けているので、この際に「Azure 環境の Entra ID テナントとは異なる Entra ID テナントの情報を読み取る」と言うことになります。これが実現できないのかと言われれば、当然実現は可能です。Entra ID を操作する際に利用する Service Principal を「M365 環境」側の Entra ID テナントで作成し、その情報を Azure 環境側に持ち込めば実現可能です。アーキテクチャ図的には以下になります。

![](/images/workiq-sample-01/workiq-sample-architecture-01.png) 

ただし、後述しますが Work IQ 従量課金分の Azure Subscription は「M365 環境」の Entra ID テナント配下に紐づける必要があります。「M365 環境」側で作成した Service Principal の情報を別環境から利用することで Web アプリ・クライアントアプリは勿論、Azure/On-premise/AWS/GCP を問わずに Work IQ を任意の場所で利用することができます。

## 実際に利用作成するサンプル
今回は以下の様に C# Console アプリから直接呼び出します。今回は MCP サーバ形式で呼び出しますが、もちろん REST/A2A での呼び出しも可能です。

```txt

C# Console App
  │
  ├─ Microsoft Entra ID
  │    └─ ユーザー対話サインイン
  │         Scope: WorkIQAgent.Ask
  │
  └─ Work IQ MCP
       https://workiq.svc.cloud.microsoft/mcp

```

上記のエンドポイントである workiq.svc.cloud.microsoft にアクセスすることで自身のアプリケーションから Work IQ を活用することが出来る様になっています。
次に、必要な設定について深堀していきたいと思います。

## Work IQ API Billing 設定の有効化

まず最初に行う必要があるのが admin.microsoft.com サイトでの課金の有効化です。Microsoft Admin ポータルを開き Home > Copilot > Cost management のメニューから課金を有効化し、Work IQ を利用した場合の課金対象となる Azure Subscription を選択する必要があります。この際に M365 環境側の Entra ID 配下の Azure Subscription しか選べないので、Azure Subscription に Entra ID テナントの変更等の処理が必要になるでしょう。
- [Understand usage-based billing and cost management for Copilot Credits](https://learn.microsoft.com/en-us/microsoft-365/copilot/usage-based-billing-overview-copilot-credits)
- [Optimize Copilot Credit costs with a pre-purchase plan](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/copilot-credit-p3)

実際のポータルで有効化に成功した画面は以下になります。

![](/images/workiq-sample-01/cost-billing-01.png) 

設定が完了し、Configuration タブに異動すると以下の様に芋付けされた Azure Subscription が表示され、Work IQ の従量課金はこちらに対して行われます。

![](/images/workiq-sample-01/cost-billing-02.png) 

こちらで Microsoft Admin ポータルでの設定は完了です。

## M365 環境の Entra ID 上に Service Principal を作成する

作成する C# で利用するための Service Principal を作成しますが、その前に課金とは別に Work IQ 自体を有効化しないといけません。[MS Learn - Enable your tenant for Work IQ](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/enable-work-iq?tabs=entra-admin) の記事を参考にして、Work IQ を使う Entra ID テナントにログインし、以下のコマンドを実行してください。

```bash

az ad sp create --id fdcc1f02-fc51-4226-8753-f668596af7f7

```

上記のコマンドで Work IQ 自体の Service Principal が作成され、自身のアプリでも利用できるようになります。この処理を行わないと、後述の自身の Service Principal に対する Work IQ の権限割り売り自体ができなくなります。


次に自身の C# ソースコード側で利用する Service Principal を作成します。以下の設定で作成します。
- Single-tenant App Registration を作成
- Mobile and desktop application を構成
- Redirect URI に http://localhost を設定
- Work IQ API の delegated permission を追加
- WorkIQAgent.Ask を選択
- Grant admin consent を実行

特に「Work IQ API の delegated permission を追加」する点が重要です。この設定をする場合、ユーザ側にポップアップ画面を表示させ、ユーザ側の認証情報を用いてアプリケーションを操作するという流れになります。


## C# アプリで実際に動かしてみる

上記で設定した Service Principal の情報を利用して、アプリケーションを実行します。質問として「７日以内の Teams やメール等の情報を見て終わってない様なタスクを挙げてほしい」な文章を入れています。現時点では SDK 等は無く、エンドポイントを直接叩く必要がありそうです。

```csharp

using Microsoft.Identity.Client;
using System.Net.Http.Headers;
using System.Text;

const string tenantId =
    "your-tenant-id";

const string clientId =
    "your-client-id";

string[] scopes =
{
    "api://workiq.svc.cloud.microsoft/WorkIQAgent.Ask"
};

Console.WriteLine("Signing in...");

var app =
    PublicClientApplicationBuilder
        .Create(clientId)
        .WithTenantId(tenantId)
        .WithRedirectUri("http://localhost")
        .Build();

var auth =
    await app
        .AcquireTokenInteractive(scopes)
        .ExecuteAsync();

Console.WriteLine($"User    : {auth.Account?.Username}");
Console.WriteLine($"Tenant  : {auth.TenantId}");
Console.WriteLine($"Expires : {auth.ExpiresOn}");
Console.WriteLine();

var httpClient = new HttpClient();

httpClient.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue(
        "Bearer",
        auth.AccessToken);

//
httpClient.DefaultRequestHeaders.Accept.Clear();

httpClient.DefaultRequestHeaders.Accept.Add(
    new MediaTypeWithQualityHeaderValue(
        "application/json"));

httpClient.DefaultRequestHeaders.Accept.Add(
    new MediaTypeWithQualityHeaderValue(
        "text/event-stream"));


//
// 1. tools/list
//
var toolsListRequest =
"""
{
  "jsonrpc":"2.0",
  "id":1,
  "method":"tools/list"
}
""";

Console.WriteLine();
Console.WriteLine("Calling tools/list...");

var toolsResponse =
    await httpClient.PostAsync(
        "https://workiq.svc.cloud.microsoft/mcp",
        new StringContent(
            toolsListRequest,
            Encoding.UTF8,
            "application/json"));

var toolsBody =
    await toolsResponse.Content.ReadAsStringAsync();

Console.WriteLine(toolsBody);

//
// 2. ask
//
var askRequest =
"""
{
  "jsonrpc":"2.0",
  "id":2,
  "method":"tools/call",
  "params":
  {
    "name":"ask",
    "arguments":
    {
      "question":"Review my last 7 days of work and identify forgotten or unfinished tasks. Include emails, meetings, Teams messages, documents, and Planner tasks. Return the top 10 items ordered by urgency."
    }
  }
}
""";

Console.WriteLine();
Console.WriteLine("Calling ask...");

var askResponse =
    await httpClient.PostAsync(
        "https://workiq.svc.cloud.microsoft/mcp",
        new StringContent(
            askRequest,
            Encoding.UTF8,
            "application/json"));

var askBody =
    await askResponse.Content.ReadAsStringAsync();

Console.WriteLine();
Console.WriteLine("===== ANSWER =====");
Console.WriteLine(askBody);


//
// 3. fetch
//
var fetchRequest = """
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params":
  {
    "name": "fetch",
    "arguments":
    {
      "entityUrls":
      [
        "/me/messages?$select=subject,receivedDateTime,from&$top=5"
      ]
    }
  }
}
""";

Console.WriteLine();
Console.WriteLine("Calling fetch...");

var fetchResponse = await httpClient.PostAsync(
    "https://workiq.svc.cloud.microsoft/mcp",
    new StringContent(fetchRequest, Encoding.UTF8, "application/json"));

Console.WriteLine($"HTTP {(int)fetchResponse.StatusCode}");

var fetchBody = await fetchResponse.Content.ReadAsStringAsync();

Console.WriteLine();
Console.WriteLine("==== FETCH RESULT ====");
Console.WriteLine(fetchBody);

Console.WriteLine();
Console.WriteLine("Done.");

```

実行結果は以下の様になります。


```txt

Calling tools/list...
event: message
data: {"result":{"tools":[{"name":"fetch_blob","description":"Fetch a binary file (document, image, photo) from a WorkIQ path \u2014 PDFs,Office files, profile photos, etc. Up to 4 MB for the raw file. Returns the file\u0027s bytes base64-encoded plus metadata.","inputSchema":{"type":"object","properties":{"path":{"description":"Relative WorkIQ path to the binary content, e.g. /me/photo/$value or /drives/{id}/items/{id}/content. Must be a relative path \u2014 do not include a base URL.","type":"string"},"format":{"description":"Optional. Forwarded as $format to convert the item on download (e.g. \u0022pdf\u0022); honored only on drive-content endpoints that accept it.","type":["string","null"],"default":null},"agentId":{"description":"Optional agent ID to target a specific agent.","type":["string","null"],"default":null}},"requ＜＜中略＞＞


Calling ask...

===== ANSWER =====
event: message
data: {"result":{"content":[{"type":"text","text":"I could not reliably retrieve recent meetings, Planner tasks, documents, or Teams activity from the last 7 days. The available grounding data primarily contains email activity, so the list below is limited to items that explicitly appear as open actions, unread messages, recommendations, or follow-up candidates in your mailbox. I am not inferring tasks that are not explicitly supported by the retrieved data.\n\n### Top 10 Potentially Forgotten or Unfinished Items (ordered by urgency)\n\n1. **Review and apply the Azure Logic App recommendation fo
＜＜中略＞＞

Calling fetch...
HTTP 200

==== FETCH RESULT ====
event: message
data: {"result":{"content":[],"structuredContent":{"results":[{"data":{"@odata.context":
＜＜中略＞＞

Done.

```

### ハマったエラー

以下の環境で

- Entra ID authentication は成功
- WorkIQAgent.Ask token も取得済み
- MCP initialize は成功
- tools/list も成功
- ask と fetch の実行時だけ billing error が発生

以下のエラーが発生しました。

```txt

caller tenant billing policy not configured for WorkIQ

```

確認の結果、アプリ認証や API permission の問題ではなく、Microsoft 365 tenant の usage-based billing policy が未構成だったので「Microsoft 365 Admin Center > Copilot > Cost management」で usage-based billing を有効化した。
