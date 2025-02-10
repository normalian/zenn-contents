---
title: "Azure AI Agent で簡単 RAG を実装する"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Azure AI Agent", "AI", "csharp"]
published: true
publication_name: "microsoft"
---

昨年末ごろにリリースされた Azure AI Agent service は皆様利用されているでしょうか？簡単に AI Agent が作れるという触れ込みで出たサービスですが、学習を兼ねてまとめたものをこちらに記載させて頂きます。

## Azure AI Agent service のリソース作成

まずはリソースを作製するので、以下の記事を参照に AI Foundry 経由でサービスを作成します。
[Learn / Azure / AI Services / Agents / Quickstart: Create a new agent](https://learn.microsoft.com/en-us/azure/ai-services/agents/quickstart?view=azure-dotnet-preview&pivots=ai-foundry)

まずは Azure AI Foundry のポータルである https://ai.azure.com/ にアクセスします。以下の様にプロジェクト一覧が表示されている場合はそのまま「+ Create Project」のボタンを押してください。
![](/images/aiagentservice-dotnet-01/image01.png) 

既に作業中の場合は別の画面が開かれていると思います。上記のトップ画面に遷移したい場合は以下の様にプロジェクトのメニューから「All projects」を選択するかそのまま「+ Create projects」ボタンを押せば OK です。
![](/images/aiagentservice-dotnet-01/image02.png) 

以下の様にウィザードを利用して Azure AI Foundry 上でプロジェクトを作製するのですが、様々なリソースを作製する必要があります。以下の画面を見て頂ければ分かると思いますが、リソースグループ自体に加えて Azure AI services, Azure AI hub, Azure AI project の 3 リソース名を入力する必要があります（今回は Azure AI Search は省略しています）。
![](/images/aiagentservice-dotnet-01/image03.png) 

ここで Create ボタンを押すと 4～6 分ほど待った後にプロジェクト画面に遷移します。念のため Azure Portal 側で同リソースグループを参照すると以下の様にストレージアカウントや Key Vault が作製されていることが分かります。
![](/images/aiagentservice-dotnet-01/image04.png) 

次に左メニューから Build and customize 配下にある Agents メニューを押して画面遷移します。プルダウンメニューから Azure Portal 側で確認した Azure AI Service リソースを選択し、Let's go ボタンをおして漸く Azure AI Agent service の利用が開始できます。
![](/images/aiagentservice-dotnet-01/image05.png) 

次に AI Agent で利用するモデルを選択します。gpt-4o のモデルが見切れていますが、他にどのモデルが現時点（2025年2月現在）で利用できるか分かると思います。今回は gpt-4o-mini を利用します。
![](/images/aiagentservice-dotnet-01/image06.png) 

AI Agent の作成が完了すると以下の様に作成された Agent が表示されます。こちらの ID は後で利用するのでコピーしておいてください。
![](/images/aiagentservice-dotnet-01/image07.png) 

次に RAG パターンを追加するために外部ファイルを追加していきます。当該 Agent を選択後、右メニューより Knowledge から「+Add」ボタンを押してファイルを追加していきましょう。
![](/images/aiagentservice-dotnet-01/image08.png) 

現時点（2025年2月現在）は gtp-4o-mini に対しては、以下の様にファイルか Azure AI Search の利用が可能です。
![](/images/aiagentservice-dotnet-01/image09.png) 

Files を選択すると以下の様に Vector ストアを作成しつつファイルをアップロードできるウィザードが表示されるので、ここで今回は WSUS の deprecation にアレコレ書いてある pptx ファイルをアップロードしたいと思います。
![](/images/aiagentservice-dotnet-01/image10.png) 

アップロードが上手く行けば以下の様にファイルが格納された Vector ストアが作成されます。
![](/images/aiagentservice-dotnet-01/image11.png) 

## Azure AI Agent service でのプログラミング

では C# で作成した AI Agent を利用してみましょう。参考にするのは以下の記事です。
[Learn / Azure / AI Services / Agents / Quickstart: Create a new agent - C#](https://learn.microsoft.com/en-us/azure/ai-services/agents/quickstart?view=azure-dotnet-preview&pivots=ai-foundry)

AI Agent 作成時にコピーしておいた Agent ID に加え、AI Project の Overview 画面より Project connection string の計二つを利用します。
![](/images/aiagentservice-dotnet-01/image12.png) 

.NET Core Console アプリケーションを作成し、Azure.AI.Projects と Azure.Identity 二つのライブラリを NuGet 経由で取得します。次に以下のソースコードを作成します。


```csharp

// See https://aka.ms/new-console-template for more information
using Azure.AI.Projects;
using Azure.Identity;
using Azure;

Console.WriteLine("Start application!");


var connectionString = @"your-ai-project-connection-string";

AgentsClient client = new AgentsClient(connectionString, new DefaultAzureCredential());

Response<Agent> agentResponse =  await client.GetAgentAsync("your-agent-id");
Agent agent = agentResponse.Value;

////// Step 2: Create a thread
Response<AgentThread> threadResponse = await client.CreateThreadAsync();
AgentThread thread = threadResponse.Value;

// If you have already created a thread, you can leverage the existing one.
//Response<AgentThread> threadResponse = await client.GetThreadAsync("your-thread-id");
//AgentThread thread = threadResponse.Value;

// Step 3: Add a message to a thread
Response<ThreadMessage> messageResponse = await client.CreateMessageAsync(
    thread.Id,
    MessageRole.User,
    "Which features do we use for WSUS deprecation?");
ThreadMessage message = messageResponse.Value;

Response<PageableList<ThreadMessage>> messagesListResponse = await client.GetMessagesAsync(thread.Id);
//Assert.That(messagesListResponse.Value.Data[0].Id == message.Id);

// Step 4: Run the agent
Response<ThreadRun> runResponse = await client.CreateRunAsync(
    thread.Id,
    agent.Id,
    additionalInstructions: "");
ThreadRun run = runResponse.Value;

do
{
    await Task.Delay(TimeSpan.FromMilliseconds(500));
    runResponse = await client.GetRunAsync(thread.Id, runResponse.Value.Id);
}
while (runResponse.Value.Status == RunStatus.Queued
    || runResponse.Value.Status == RunStatus.InProgress);

Response<PageableList<ThreadMessage>> afterRunMessagesResponse
    = await client.GetMessagesAsync(thread.Id);
IReadOnlyList<ThreadMessage> messages = afterRunMessagesResponse.Value.Data;

// Note: messages iterate from newest to oldest, with the messages[0] being the most recent
foreach (ThreadMessage threadMessage in messages)
{
    Console.Write($"{threadMessage.CreatedAt:yyyy-MM-dd HH:mm:ss} - {threadMessage.Role,10}: ");
    foreach (MessageContent contentItem in threadMessage.ContentItems)
    {
        if (contentItem is MessageTextContent textItem)
        {
            Console.Write(textItem.Text);
        }
        else if (contentItem is MessageImageFileContent imageFileItem)
        {
            Console.Write($"<image from ID: {imageFileItem.FileId}");
        }
        Console.WriteLine();
    }
}

Console.WriteLine("End Application");

```

作成したソースコードを実行すると以下の様になります。「WSUS が deprecation だけどうどうすんの？」問いに対して、赤枠で囲ったところを御覧いただければ RAG パターンで外部ファイルを参照していることが確認できます。
![](/images/aiagentservice-dotnet-01/image13.png) 

以上で簡単な AI Agent service の利用方法をまとめました。参考になれば幸いです。