---
title: "Visual Studio Code から Entra ID 認証設定をした Azure Function 上の MCP サーバへの接続"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MCP", "AI", "Security"]
published: true
publication_name: "microsoft"
---

# はじめに
既に機能がリリースされて大分経った機能となりますが、以下の Visual Studio Code から Entra ID 認証設定をした Azure Function 上の MCP サーバへの接続を試したので記事にしてみました。ほとんどは以下のしばやんの記事と同じなので自分への備忘録としての色の方が強いですが（汗。
https://blog.shibayan.jp/entry/20251121/1763700606

# Azure Function のリソース作成と設定
ここでは二つの設定をします。Azure Function の「Entra ID 認証設定」と「OAuth 2.0 PRM 設定」です。

## Entra ID 認証設定
Azure Function 自体のリソース作成については特に言及しませんが、以下の記事を参考にして認証設定をしていきます。
https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-provider-aad?tabs=workforce-configuration

設定後の情報となりますが、以下の様に Microsoft 認証を選ぶ点、Additional checks にある Allowed client applications にに Visual Studio Code の Client Id である aebc6443-996d-45c2-90f0-388ff96faa56 を追加する点の二点に注意が必要です。
![](/images/function-mcp-auth-01/image01.png) 

Visual Studio Code 以外のアプリケーションを利用する場合、以下の記事から選択することでどのアプリケーションからの認証を受け付けるのか選択できます。
https://learn.microsoft.com/en-us/power-platform/admin/apps-to-allow


## OAuth 2.0 PRM 設定
次に App Service 認証 から MCP クライアントに ID プロバイダーによる認証を要求するための OAuth 2.0 Protected Resource Metadata 設定を追加します。Environment - App settings から WEBSITE_AUTH_PRM_DEFAULT_WITH_SCOPES 環境変数に api://"your-client-id"/user_impersonation の値を以下の様に追加します。
![](/images/function-mcp-auth-01/image02.png) 


# Azure Function に乗せる .NET アプリケーション
今回のサンプルは Agent Framework を利用しています。本記事で実際に動かしたサンプルは以下に配置しているので、こちらを利用して動かしてみると理解が深まるのではと思います。
https://github.com/normalian/MyAFSamples/tree/main/Sample08MCPServerFunctionApp

## host.json の設定
今回は Entra ID の認証を使いたいだけなのでアクセスキーでの認証をオフにします。これを忘れると WebJobsAuthLevel was forbidden エラーが発生します。これは Azure Function リソース自体の Auth 認証・Azure Function へのキー認証と二重の認証になっているからです。今回は Entra ID 認証のみを使いたいので、以下の様に記載します。

```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      },
      "enableLiveMetricsFilters": true
    }
  },
  "extensions": {
    "mcp": {
      "system": {
        "webhookAuthorizationLevel": "Anonymous"
      }
    }
  }
}
```


## C# アプリケーションを作成

以下の様な単純なアプリケーションを作成します。特に記載はしませんが、Azure Function 上で DefaultAzureCredential を利用して Azure OpenAI のリソースにアクセスする際は Managed Identity に Cognitive Services OpenAI User ロールの付与を忘れるとエラーになるのでご注意を。 

```csharp
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using Microsoft.Agents.AI.Hosting.AzureFunctions;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

class Program
{
    public static async Task Main(string[] args)
    {
        var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT") ?? throw new InvalidOperationException("AZURE_OPENAI_ENDPOINT is not set.");
        var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4.1-mini";
        AzureOpenAIClient client = new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential());

        // Put following mcp.json file in the root of your function app to enable MCP server. The url should be your function app url plus "/runtime/webhooks/mcp/sse".
        // {
        //        "servers": {
        //        "mine-mcp-server-local": {
        //          "type": "sse",
        //              "url": "https://your-appservice-name.azurewebsites.net/runtime/webhooks/mcp/sse"
        //         }
        //    }
        // }

        AIAgent agent = client.GetChatClient(deploymentName).AsIChatClient().AsAIAgent(
            instructions: @"In this Normalian project, to facilitate management, class names must follow naming conventions based on business categories.
            Class names according to business categories are provided by the `GenerateClassNamingConvention` tool.
            Please define classes using the names provided here. Do not define classes with any other names.

            If you are unsure about the business category from the class name, use the `DetermineBusinessCategory` tool to obtain the business category.",
            name: "NormalianMyProjectAdvisor",
            description: "This tool describes naming rule overview of the Normalian project.");

        using IHost app = FunctionsApplication
            .CreateBuilder(args)
            .ConfigureFunctionsWebApplication()
            .ConfigureDurableAgents(options =>
            {
                options.AddAIAgent(agent, enableHttpTrigger: false, enableMcpToolTrigger: true);
            })
            .Build();
        app.Run();
    }
}
```

# Visual Studio Code 上から呼び出す

以下の設定ファイルを .vscode/mcp.json として配置し、Visual Studio Code を起動し、設定した MCP サーバへ接続します。接続時は popup windows が表示されるので、認証情報を入力してください。 

```json
{
  "servers": {
    "mine-mcp-server-local": {
      "type": "sse",
      "url": "https://sample08mcpserverfunctionapp202603.azurewebsites.net/runtime/webhooks/mcp/sse"
    }
  }
}
```

認証が完了後、以下の様に GitHub Copilot を開いて会話することで MCP サーバを利用した会話が可能です。
![](/images/function-mcp-auth-01/image04.png) 

App Service の Log Stream 側で動作内容も確認が可能です。
![](/images/function-mcp-auth-01/image03.png) 

