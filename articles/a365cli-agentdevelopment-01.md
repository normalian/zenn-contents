---
title: "Agent365 CLI を利用したエージェント開発"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Agent365", "AI"]
published: true
publication_name: "microsoft"
---

# はじめに

本記事では、Microsoft が提供する Agent 365 CLI（a365 CLI）と Microsoft Agent Framework を利用して、Microsoft 365Teams と連携可能なエンタープライズ向け AI エージェントを Azure 上にデプロイ・公開するまでの手順を解説します。

生成 AI を活用したエージェントの開発は急速に進んでおり、「組織の ID・権限管理のもとで安全に Microsoft 365 と連携する」という観点では、単なる Web API ベースのチャットボットとは異なるガバナンスを利かせたアーキテクチャが求められます。Microsoft Agent 365（A365）は上記の AI エージェントを Microsoft Entra ID（旧 Azure AD）ベースのエンタープライズ ID として管理し、Teams や Outlook、Copilot などの Microsoft 365 ワークロードとセキュアに統合するためのコントロールプレーンであり、Agent 365 で管理・運用ができるかたちでエージェントを容易に公開できるところが Agent 365 CLI のメリットです。

Agent 365 CLI（a365 CLI）は Agent 365 に対応したエージェントの開発ライフサイクルを自動化するためのコマンドラインツールであり、以下のような処理を一貫して実行できます。

- エージェントの Blueprint（テンプレート）の作成
- Microsoft Entra ID 上の Agent Identity の作成
- Azure App Service などのインフラの自動プロビジョニング
- エージェントコードの Azure へのデプロイ
- Microsoft 365 管理センターへの公開用パッケージの生成
- Agent Instance（実体）の作成およびライセンス割り当て

本記事では、以下の公式サンプルアプリケーションをベースにエージェントの公開までを紹介します。

Microsoft Agent Framework Sample Agent (.NET)
https://github.com/microsoft/Agent365-Samples/tree/main/dotnet/agent-framework/sample-agent

このサンプルは、

- Microsoft Agent Framework をオーケストレーターとして利用し
- Microsoft Agent 365 SDK と Microsoft 365 Agents SDK を組み合わせて
- ASP.NET Core 上でエージェントをホストし
- `/api/messages` エンドポイントを通じて Microsoft 365 からのメッセージを受信する

最小構成のエンタープライズ対応 AI エージェントとなります。

---

## Agent 365 における主要リソース構成

Agent 365 では、一般的な「Bot アプリケーション」とは異なり エージェントは Microsoft Entra ID 上の専用リソースとして管理されます。分かりにくいですが Agent365 CLI によって主に作成されるリソースは以下の 3 層構造です。

### 1. Agent Blueprint

Agent Blueprint は エージェントの ID、権限、およびインフラ構成を定義するテンプレートであり、以下の情報を保持します。
- エージェント用のアプリケーション登録（App Registration）
- Microsoft Graph API などへのアクセス権限
- Messaging Bot API の権限
- MCP（Model Context Protocol）サーバーへのアクセス権限
- Agent Instance に継承されるポリシー設定

すべての Agent Instance は Blueprint を元に作成されます。

---

### 2. Agent Identity

Agent Identity は Microsoft Entra ID 上に作成されるサービスプリンシパルです。これは「人間のユーザー」と同様に、

- 固有の Object ID
- App ID
- API アクセス権限
- スポンサー（責任者）
- マネージャー情報

などを持ち Microsoft 365 ワークロードへのアクセス主体として動作します。

Agent Identity は Azure Bot Service の単なるバックエンドアプリとは異なり、組織のポリシー・監査・条件付きアクセスの対象となるエンタープライズ ID として扱われます。

### 3. Agent Instance

Agent Instance は Blueprint から生成される「実体」となるエージェントです。Agent365 CLI の `create-instance` コマンドにより

- Agent Identity に紐づく Agent User（Entra ID ユーザー）
- Microsoft 365 ライセンスの割り当て
- エージェントのスポンサー設定

などが自動的に構成されて Microsoft Teams や Copilot 上で @メンション可能なデジタルワーカーとして利用できる状態になります。

### 作成される Azure リソース

Agent365 CLI の `setup` コマンドを実行すると Microsoft Entra ID 上のリソースに加え、Azure 側にもエージェント実行環境が自動的にプロビジョニングされます。主に以下のリソースが作成されます。

- App Service Plan
- Azure Web App
- Managed Identity（Web App 用）
- Agent Blueprint に紐づくアプリ登録
- Microsoft Graph API 権限設定
- Messaging Bot API 権限設定

Azure Web App 上にデプロイされたエージェントは `/api/messages` エンドポイントを通し、Microsoft 365（Teams / Copilot）からのアクティビティを受信して Agent Framework による処理を実行します。

---

本記事では上記のリソースを a365 CLI でどのように構築し、Microsoft 365 と連携されるのか、実際のサンプルアプリケーションを用いて段階的に解説していきます。


## 前提条件

本記事の手順を実施するにあたり、以下の実施を行ってください。
- Microsoft Entra ID 上に Service Principal を作成
- Azure サブスクリプションの取得と Resource Group の作成
- Agent365 Cli のインストール
- 本記事で利用する公式サンプルアプリケーションを取得 https://github.com/microsoft/Agent365-Samples.git

以下のコマンドを利用して Agent365 Cli をインストールします。インストール後は -h コマンドで動作確認を行います。

```bash
dotnet tool install --global Microsoft.Agents.A365.DevTools.Cli --prerelease
a365 -h
```

# エージェントの作成と公開

## Service Principal の設定と エージェント設定ファイルの作成
Agent 365 CLI を利用してエージェントを Microsoft 365 上へ公開するまでの処理は、大きく分けて「Agetn365 Cli でのエージェント設定」「Agent Blueprint、Azure リソース等の環境セットアップ」「Agent の公開パッケージ作成」の 3 ステップで構成されます。まず以下のコマンドを実行し、Agent365 CLI の設定を初期化するのですが、まずは以下のコマンドを実行してみましょう。

```bash
a365 config init
```

すると以下の様に Service Principal の権限を確認されます。
![](/images/a365cli-agentdevelopment-01/sp-setting-01.png) 

既に権限設定済の場合は問題ありませんが、そうでない場合は Entra ID 側に移動して権限設定をしましょう。その場合、Microsoft Graph の権限を与えますが Application Permissions でなく Delegated Permissions な点に注意してください。
![](/images/a365cli-agentdevelopment-01/sp-setting-02.png) 

権限付与後、Grant Admin consent ボタンを押して権限を委譲します。これで Service Principal の設定は完了です。改めて a365 config init を実行し、当該 Service Principal の client id を入力すると Azure サブスクリプション、Resource Group、App Service Plan、デプロイリージョンなどを自動検出したうえで Agent365 CLI の構成ファイル（a365.config.json）を作成します。
![](/images/a365cli-agentdevelopment-01/a365-init-01.png) 

Microsoft Entra ID 上の Custom Client App Registration が存在するか、および必要な権限と Admin Consent が付与されているかの検証も同時に行われます。

## インフラリソースの作成

次に完了後に以下のコマンドを実行し、 Agent365 の実行環境を Azure および Microsoft Entra ID 上にプロビジョニングします。

```bash
a365 setup all
```

本コマンドは Agent365 の初期セットアップ処理を一括実行し、Azure 側では App Service、App Service Plan、Managed Identity 等のインフラを作成し、Microsoft Entra ID 側では Agent Blueprint の登録および Graph / Messaging Bot API などの継承権限設定を行います。またセットアップ結果として a365.generated.config.json が生成され、Blueprint ID や Messaging Endpoint などの各種設定値が保存されます。

![](/images/a365cli-agentdevelopment-01/a365-setupall-01.png) 

## Microsoft Admin Center へのパッケージ登録での Blueprint の作成

セットアップ完了後、Microsoft 365 管理センターへアップロードするための公開用パッケージを以下のコマンドで作成します。

```bash
a365 publish
```

![](/images/a365cli-agentdevelopment-01/a365-publish-01.png) 

上記のコマンドを実行すると Agent Blueprint ID を反映した manifest.json が更新され、アイコンファイルとともに manifest.zip が生成されます。
この manifest.zip を Microsoft 365 管理センター https://admin.microsoft.com/ の（Agents > All agents > Upload custom agent）からアップロードすることで、Microsoft Teams や Copilot 上で利用可能なエージェントとして組織内に公開することが可能となります。以下の画面の様に公開範囲のユーザも設定できます。
![](/images/a365cli-agentdevelopment-01/a365-admincenter-01.png) 
![](/images/a365cli-agentdevelopment-01/a365-admincenter-02.png) 

manifest.zip のアップロードが完了すると以下の様に Agent Blueprint が登録されます。

## Teams Developer Portal でのエンドポイント登録

次にエージェントが受け取るメッセージのアプリケーションのエンドポイントを Teams Dev Center から登録します。まずエンドポイントを確認するためにコマンドを実行します。

```bash
a365 config display -g
```

![](/images/a365cli-agentdevelopment-01/teams-devcenter-01.png) 

表示されている /api/messages の URL を保存し、Teams Developer Portal https://dev.teams.microsoft.com/home を開きます。Tools の中から自身の Agent Blueprint を選択し、以下の Configuration から Agent Type を API Based に選択後、Notification URL に保存した URL を入力して保存してください。

![](/images/a365cli-agentdevelopment-01/teams-devcenter-02.png) 


## エージェント instance & identity の作成

次にエージェントの実態を作成します。Blueprint はエージェントの設計図相当なので、ここで漸く実態の作成となります。

```bash
a365 create-instance
```

![](/images/a365cli-agentdevelopment-01/a365-createinstace-01.png) 

上記の実行が完了すると以下の画面の様に表示され、Agent の実態である instance id や agent id が表示されていることが分かると思います。これに加えて appsettings.json も更新され、client secret 等の情報がアップデートされます。

## ASP.NET Core アプリケーションを App Service にデプロイ

アプリケーションのデプロイ前に appsettings.json を更新します。以下に更新対象をそれぞれ記載しました。Service Principal の client id、Agent Blueprint ID、Entra ID のテナントID 等の Entra ID 周りの設定に加え、Azure OpenAI 関連の設定も必要になる点に注意してください。

```json
{
  "AgentApplication": {
    "StartTypingTimer": false,
    "RemoveRecipientMention": false,
    "NormalizeMentions": false,
    "AgenticAuthHandlerName": "agentic",
    "UserAuthorization": {
      "AutoSignin": false,
      "Handlers": {
        "agentic": {
          "Type": "AgenticUserAuthorization",
          "Settings": {
            "Scopes": [
              "https://graph.microsoft.com/.default"
            ],
            "AlternateBlueprintConnectionName": "ServiceConnection"
          }
        }
      }
    }
  },
  "TokenValidation": {
    "Audiences": [
      "<your-service-principal-client-id>"
    ],
    "Enabled": false,
    "TenantId": "<your-entraid-tenant-id>"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.Agents": "Warning",
      "Microsoft.Hosting.Lifetime": "Information",
      "Microsoft.SemanticKernel*": "Warning"
    }
  },
  "AllowedHosts": "*",
  "Connections": {
    "ServiceConnection": {
      "Settings": {
        "AuthType": "ClientSecret", // This AuthType is required to update from UserManagedIdentity to ClientSecret
        "AuthorityEndpoint": "https://login.microsoftonline.com/<your-entraid-tenant-id>",
        "ClientId": "<your-service-principal-client-id>",
        "Scopes": [
          "5a807f24-c9de-44ee-a3a7-329e88a00ffc/.default"
        ],
        "ClientSecret": "<your-clientid-but-this-id-should-be-generated>",
        "AgentId": "<your-agentid-but-this-id-should-be-generated>"
      }
    }
  },
  "ConnectionsMap": [
    {
      "ServiceUrl": "*",
      "Connection": "ServiceConnection"
    }
  ],
  "AIServices": {
    "AzureOpenAI": {
      "DeploymentName": "gpt-4.1-mini",
      "Endpoint": "<https://your-azure-openai-url.openai.azure.com/>",
      "ApiKey": "<your-azure-openai-api-key>"
    }
  },
  "OpenWeatherApiKey": "<your-api-key>"
}
```

設定ファイル編集後、以下のコマンドを実行してアプリケーションのデプロイを行います。

```bash
a365 deploy
```

コマンドが正常に実行されれば以下の様に表示されます。

![](/images/a365cli-agentdevelopment-01/a365-deploy-01.png) 


# 動作確認
一通りの設定が正常に完了している場合、Teams 側から「（自分のエージェント名） Agent User」のエージェントが作成されているので、Teams チャットから当該エージェントを探して話しかけると以下の様に返答が返ってきます。

![](/images/a365cli-agentdevelopment-01/a365-chatwithagent-01.png) 

# 参考リンク
- https://learn.microsoft.com/en-us/microsoft-agent-365/developer/agent-365-cli
- https://github.com/microsoft/Agent365-Samples
- https://dev.teams.microsoft.com/home
- https://admin.cloud.microsoft
- https://openweathermap.org/price