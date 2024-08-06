---
title: "AI はアーキテクチャ設計ができるのか？アーキテクチャ図を出力したり、月額利用料を見積もらせる ～GTP-4o 編～"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Azure OpenAI", "AI", "gpt-4o"]
published: true
publication_name: "microsoft"
---

OpenAI の発表以来、日進月歩各モデルの進化は進んでいます。つい先日に Azure OpenAI でも gpt-4o が利用可能になり既に様々な用途で利用している方が多いと思います。画像認識に関しても様々な用途が試されています。特に以下の記事を拝見して「なるほどー、アーキテクチャ図を読み込ませるという手があったな」というアイディアが得られました。
- [GPT-4oによる画像解析](https://zenn.dev/yyteam/articles/aoai-gpt4o-sample)

上記の記事ではアーキテクチャ図の単体を読ませて終わっていますが、今回はアーキテクチャ図を入力させることに加え、ビジネス・システム要件を入力した際にどの程度の回答をしてくれるのかも含めて試してみたいと思います。

** 前提条件
今回は gpt-4o を利用しますが、現時点で同モデルは Japan region で利用することができません。以下のサイトを参考に gpt-4o のモデルが利用できるリージョンを選択してください。また、今回の環境はすべて Azure OpenAI Studio で試します。
- [Azure OpenAI Service models](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models)
また、当然ながら本記事を利用して見積もりやアーキテクチャ図を作成したい際は必ず自身でもレビュー・修正を行ってください。本記事を踏襲した結果での問題や障害については免責とさせて頂ければ幸いです。

** 現実の案件に即したビジネス要件・課題、システム要件を入力してみる
では、前置きはさておきで、まずは実際の案件に近い要件を入力してみましょう。ビジネス要件は以下を想定します。

- 1982 年に設立された Contoso 社は、世界規模でゴルフ製品（ゴルフクラブ、ゴルフボール、ゴルフ向けの付属製品）を販売する巨大メーカーに成長した
- リ○マ○ショックの景気後退により、東京近辺にデータセンタを集約したものの、500万円/月程度のデータセンタ費用が発生している
- 更に、日本各所でいくつかのサーバを利用しており、コストがかかっている

システム課題は以下を想定します。
- データセンタ内に数十の Web サーバが存在し、顧客データが複数の場所に分散して存在している
-ハードウェア更新時期を迎え、今後のハードウェア更新リスクを回避するめに Contoso 社はシステムをどこかに移行したい

次にシステム要件として以下を想定します。
- 「製品カタログ」「在庫」「部門」の三つの WEB アプリケーションが存在する
- Web アプリは C# で開発されている
- 全世界に販売しているため、日本以外の国からのレスポンスも改善が必要
- 製品カタログのデータは SQL Server に格納されており、製品カタログはアジア（日本含む）、アメリカ東海岸、西ヨーロッパからアクセスされる
- 製品カタログのデータは日本でしか更新されず、夜間にデータバックアップが取得されている
- 製品カタログのデータベースサイズは 50GB で、年間 5% 程度データ量が増加する
- 製品カタログは主に読み込み用途で利用される
- 外部アクセスからのセキュリティ担保は SSL 暗号化通信を想定しているが、内部ユーザ（カタログ更新）は SSL に加えたオンプレミスからの閉域ネットワークを想定

では実際に以下のプロンプトを読み込ませてみましょう。system message は以下とします。

```txt
You are an expert of Microsoft Azure. You assist Microsoft Azure architecture design.
```

次に実際 入力した文章は以下となります。

```txt
以下のシステムアーキテクチャ設計を Microsoft Azure を利用して行ってください。
=============== business requirements ========================
- 1982 年に設立された Contoso 社は、世界規模でゴルフ製品（ゴルフクラブ、ゴルフボール、ゴルフ向けの付属製品）を販売する巨大メーカーに成長した
- リ○マ○ショックの景気後退により、東京近辺にデータセンタを集約したものの、500万円/月程度のデータセンタ費用が発生している
- 更に、日本各所でいくつかのサーバを利用しており、コストがかかっている
=============== current system issues  ========================
- データセンタ内に数十の Web サーバが存在し、顧客データが複数の場所に分散して存在している
-ハードウェア更新時期を迎え、今後のハードウェア更新リスクを回避するめに Contoso 社はシステムをどこかに移行したい
=============== system requirements ========================
- 「製品カタログ」「在庫」「部門」の三つの WEB アプリケーションが存在する
- Web アプリは C# で開発されている
- 全世界に販売しているため、日本以外の国からのレスポンスも改善が必要
- 製品カタログのデータは SQL Server に格納されており、製品カタログはアジア（日本含む）、アメリカ東海岸、西ヨーロッパからアクセスされる
- 製品カタログのデータは日本でしか更新されず、夜間にデータバックアップが取得されている
- 製品カタログのデータベースサイズは 50GB で、年間 5% 程度データ量が増加する
- 製品カタログは主に読み込み用途で利用される
- 外部アクセスからのセキュリティ担保は SSL 暗号化通信を想定しているが、内部ユーザ（カタログ更新）は SSL に加えたオンプレミスからの閉域ネットワークを想定```

帰ってきた結果は以下になります。

```txt
Contoso社の要件に基づいて、Microsoft Azureを利用したシステムアーキテクチャを設計します。以下に詳細なアーキテクチャを示します。
1. Azure Infrastructure Setup
1.1. Azure Regions
Primary Region: Japan East
Secondary Regions for High Availability and Geographical Redundancy: East US, West Europe
2. Web Applications
2.1. App Service Plan
Service: Azure App Service
Plan: Standard or Premium plan depending on performance requirements
Scaling: Auto-scaling enabled based on traffic
2.2. Deployment Slots
Use deployment slots for staging and production to facilitate smooth updates.
3. Database
3.1. SQL Server
Service: Azure SQL Database
Tier: Business Critical or Premium tier for high performance and availability
Geo-Replication: Set up Active Geo-Replication to East US and West Europe for read scalability and high availability
3.2. Data Sync
Service: Azure SQL Data Sync to synchronize updates from the primary (Japan) to secondary regions (East US, West Europe)
4. Security
4.1. SSL Encryption
Service: Azure Application Gateway or Azure Front Door to manage SSL certificates and provide global load balancing
Certificates: Manage SSL certificates via Azure Key Vault
4.2. Virtual Network (VNet) and Subnets
Service: Azure VNet
Configuration: Create subnets for web apps, database, and other services
4.3. VPN Gateway
Service: Azure VPN Gateway
Configuration: Establish a site-to-site VPN connection for secure communication between on-premises data center and Azure VNet
5. Backup and Disaster Recovery
5.1. Azure Backup
Service: Azure Backup
Configuration: Schedule nightly backups of the SQL database
5.2. Disaster Recovery
Service: Azure Site Recovery
Configuration: Set up Azure Site Recovery for VM replication and failover
6. Monitoring and Management
6.1. Azure Monitor
Service: Azure Monitor
Configuration: Set up monitoring for performance, availability, and security alerts
6.2. Log Analytics
Service: Azure Log Analytics
Configuration: Collect and analyze logs from all Azure resources
7. Cost Management
7.1. Azure Cost Management and Billing
Service: Azure Cost Management
Configuration: Set budgets and alerts to monitor and control costs
8. Architecture Diagram
 

8.1. Components
Azure App Service for Web Apps
Azure SQL Database with Geo-Replication
Azure VNet with subnets for web apps, databases, and other services
Azure VPN Gateway for secure on-premises connectivity
Azure Application Gateway or Azure Front Door for SSL and load balancing
Azure Backup for database backups
Azure Monitor and Log Analytics for monitoring and logging
Conclusion
 
この設計により、Contoso社は以下の利点を得ることができます：

高可用性と地理的冗長性
自動スケーリングによるコスト効率
SSL暗号化によるセキュリティ向上
簡単な管理と監視

このアーキテクチャにより、現在のデータセンタコストを削減し、ハードウェア更新リスクを回避
```

system message が英語だからか英語と日本語が入り混じっています。これだけだと構成が良くわからないので、次はアーキテクチャ図を出してもらいましょう。

** アーキテクチャ図を出してもらう
アーキテクチャ図を出力してもらうため、以下を OpenAI に入力します。
```txt
Contoso社の要件に基づき、アーキテクチャ図を出力してください。
```

結果は以下になります。テキストも後述しますが、表示が崩れるのでまずはスクリーンショットを以下に示します。
![](/images/aoai-architecture-01/image01.png) 

テキストは以下になります。各 Azure の機能を用いた構成がどの様に要件に合致しているかが示されています。しかし、一部が最後まで出ていないことが分かります。

```txt
以下は、Contoso社の要件に基づくAzureアーキテクチャ図のテキスト説明です。図を手動で描画する際の参考にしてください。
アーキテクチャ図の構成要素
 
1. オンプレミス環境
  - オンプレミスネットワーク
    内部ユーザー（カタログ更新）
    VPNデバイス（ルーターやファイアウォール）
3. Azure環境
   - Azure Virtual Network (VNet)
     サブネット
       Webサブネット
       データベースサブネット
       ゲートウェイサブネット
   - Azure VPN Gateway
     Azure VPN GatewayとオンプレミスのVPNデバイスを接続
   - Azure Application Gateway または Azure Front Door
     グローバルなロードバランシングとSSL管理
   - Azure App Service
     製品カタログ Web アプリ
     在庫 Web アプリ
     部門 Web アプリ
   - Azure SQL Database
     製品カタログデータベース（日本リージョンを主、他リージョンにレプリケーション）
   - Azure Key Vault
     SSL証明書管理
   - Azure Backup
     SQLデータベースのバックアップ
   - Azure Monitor & Log Analytics
     パフォーマンス監視とログ管理


アーキテクチャ図のテキスト説明 

オンプレミス環境  
┌─────────────────────┐  
│オンプレミスネットワーク          │  
│  ┌───────────────────┐ │  
│  │内部ユーザー（カタログ更新） │ │  
│  └───────────────────┘ │  
│  ┌───────────────────┐ │  
│  │VPNデバイス（ルーター/ファイアウォール）│ │  
│  └───────────────────┘ │  
└─────────────────────┘  
        │  
        │（VPN接続）  
        │  
        ▼  
Azure環境  
┌─────────────────────────────┐  
│Azure Virtual Network (VNet)                  │  
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐ │  
│  │Webサブネット   │   │データベースサブネット │   │ゲートウェイサブネット │ │  
│  └─────────────┘   └─────────────┘   └─────────────┘ │  
│        │                 │                │             │  
│        ▼                 ▼                ▼             │  
│  ┌─────────────┐   ┌─────────────┐    ┌─────────────┐ │  
│  │Azure App Service │   │Azure SQL Database │    │Azure VPN Gateway │ │  
│  │（製品カタログWebアプリ│   │（Geo-Replication）│    │                       │ │  
│  │ 在庫Webアプリ            │   └─────────────┘    └─────────────┘ │  
│  │ 部門Webアプリ            │                           │  
│  └─────────────┘                           │  
│        │                                      │  
│        ▼                                      │  
│  ┌─────────────┐                           │  
│  │Azure Application  │                           │  
│  │Gateway/Front Door │                           │  
│  └─────────────┘                           │  
│       
```

次に、プロンプトを変更してアーキテクチャ図がすべて表示されるように以下を入力します。

```txt
Contoso社の要件に基づき、アーキテクチャ図を出力してください。トークン数に収まるよう、アーキテクチャの図のみを出力して下さい。
```

結果は以下になります。

```txt
以下は、Contoso社の要件に基づくAzureアーキテクチャ図のテキスト表現です。

オンプレミス環境  
┌─────────────────────┐  
│オンプレミスネットワーク          │  
│  ┌───────────────────┐ │  
│  │内部ユーザー（カタログ更新） │ │  
│  └───────────────────┘ │  
│  ┌───────────────────┐ │  
│  │VPNデバイス（ルーター/ファイアウォール）│ │  
│  └───────────────────┘ │  
└─────────────────────┘  
        │  
        │（VPN接続）  
        │  
        ▼  
Azure環境  
┌─────────────────────────────┐  
│Azure Virtual Network (VNet)                  │  
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐ │  
│  │Webサブネット   │   │データベースサブネット │   │ゲートウェイサブネット │ │  
│  └─────────────┘   └─────────────┘   └─────────────┘ │  
│        │                 │                │             │  
│        ▼                 ▼                ▼             │  
│  ┌─────────────┐   ┌─────────────┐    ┌─────────────┐ │  
│  │Azure App Service │   │Azure SQL Database │    │Azure VPN Gateway │ │  
│  │（製品カタログWebアプリ│   │（Geo-Replication）│    │                       │ │  
│  │ 在庫Webアプリ            │   └─────────────┘    └─────────────┘ │  
│  │ 部門Webアプリ            │                           │  
│  └─────────────┘                           │  
│        │                                      │  
│        ▼                                      │  
│  ┌─────────────┐                           │  
│  │Azure Application  │                           │  
│  │Gateway/Front Door │                           │  
│  └─────────────┘                           │  
│        │                                      │  
│        ▼                                      │  
│  ┌─────────────┐                           │  
│  │Azure Key Vault │                           │  
│  └─────────────┘                           │  
│        │                                      │  
│        ▼                                      │  
│  ┌─────────────┐                           │  
│  │Azure Backup      │                           │  
│  └─────────────┘                           │  
│        │                                      │  
│        ▼                                      │  
│  ┌─────────────┐                           │  
│  │Azure Monitor  │                           │  
│  │& Log Analytics │                           │  
│  └─────────────┘                           │  
└─────────────────────────────┘  
 
この図は、オンプレミス環境とAzure環境の主要なコンポーネントおよびそれらの接続を示しています。実際の図を描画する際は、このテキスト表現を元にしてください。
```

かなりいい感じです。しかし　Azure App Service の後段に Azure Application Gateway や Azure Front Door があることに加え、なぜかその先に Azure Key Vault / Azure Backup / Azure Monitor 等があるのはやや不可解です。この辺りはユーザと管理者のユーザシナリオを追加すればより良い結果が出そうです。

しかし、もともとの要件では「 製品カタログのデータは SQL Server に格納されており、製品カタログはアジア（日本含む）、アメリカ東海岸、西ヨーロッパからアクセスされる」がありましたが、単一のリージョンの図になっているように見受けられます。試しにアメリカ東海岸のリージョンについてのアーキテクチャ図を出力させてみましょう。入力した文字は以下となります。

```txt
Contoso社の要件に基づき、アーキテクチャ図を出力してください。トークン数に収まるよう、アメリカ東海岸のリージョンのアーキテクチャの図のみを出力して下さい。
```

結果は以下になります。

```txt
以下は、Contoso社の要件に基づくアメリカ東海岸リージョンのAzureアーキテクチャ図のテキスト表現です。

アメリカ東海岸リージョン  
┌─────────────────────────────┐  
│Azure Virtual Network (VNet)                  │  
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐ │  
│  │Webサブネット   │   │データベースサブネット │   │ゲートウェイサブネット │ │  
│  └─────────────┘   └─────────────┘   └─────────────┘ │  
│        │                 │                │             │  
│        ▼                 ▼                ▼             │  
│  ┌─────────────┐   ┌─────────────┐    ┌─────────────┐ │  
│  │Azure App Service │   │Azure SQL Database │    │Azure VPN Gateway │ │  
│  │（製品カタログWebアプリ│   │（Geo-Replication）│    │                       │ │  
│  │ 在庫Webアプリ            │   └─────────────┘    └─────────────┘ │  
│  │ 部門Webアプリ            │                           │  
│  └─────────────┘                           │  
│        │                                      │  
│        ▼                                      │  
│  ┌─────────────┐                           │  
│  │Azure Application  │                           │  
│  │Gateway/Front Door │                           │  
│  └─────────────┘                           │  
│        │                                      │  
│        ▼                                      │  
│  ┌─────────────┐                           │  
│  │Azure Key Vault │                           │  
│  └─────────────┘                           │  
│        │                                      │  
│        ▼                                      │  
│  ┌─────────────┐                           │  
│  │Azure Backup      │                           │  
│  └─────────────┘                           │  
│        │                                      │  
│        ▼                                      │  
│  ┌─────────────┐                           │  
│  │Azure Monitor  │                           │  
│  │& Log Analytics │                           │  
│  └─────────────┘                           │  
└─────────────────────────────┘  
 
この図は、アメリカ東海岸リージョンにおける主要なコンポーネントおよびそれらの接続を示しています。実際の図を描画する際は、このテキスト表現を元にしてください。
```

Azure SQL Database 部分が Geo-Replication にもなっていていい感じです。では東日本リージョンではどうなるか、以下を入力して試してみましょう。

```txt
Contoso社の要件に基づき、アーキテクチャ図を出力してください。トークン数に収まるよう、東日本のリージョンのアーキテクチャの図のみを出力して下さい。
```

結果は以下になります。

```txt
以下は、Contoso社の要件に基づく東日本リージョンのAzureアーキテクチャ図のテキスト表現です。

東日本リージョン  
┌─────────────────────────────┐  
│Azure Virtual Network (VNet)                  │  
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐ │  
│  │Webサブネット   │   │データベースサブネット │   │ゲートウェイサブネット │ │  
│  └─────────────┘   └─────────────┘   └─────────────┘ │  
│        │                 │                │             │  
│        ▼                 ▼                ▼             │  
│  ┌─────────────┐   ┌─────────────┐    ┌─────────────┐ │  
│  │Azure App Service │   │Azure SQL Database │    │Azure VPN Gateway │ │  
│  │（製品カタログWebアプリ│   │（Primary）        │    │                       │ │  
│  │ 在庫Webアプリ            │   └─────────────┘    └─────────────┘ │  
│  │ 部門Webアプリ            │                           │  
│  └─────────────┘                           │  
│        │                                      │  
│        ▼                                      │  
│  ┌─────────────┐                           │  
│  │Azure Application  │                           │  
│  │Gateway/Front Door │                           │  
│  └─────────────┘                           │  
│        │                                      │  
│        ▼                                      │  
│  ┌─────────────┐                           │  
│  │Azure Key Vault │                           │  
│  └─────────────┘                           │  
│        │                                      │  
│        ▼                                      │  
│  ┌─────────────┐                           │  
│  │Azure Backup      │                           │  
│  └─────────────┘                           │  
│        │                                      │  
│        ▼                                      │  
│  ┌─────────────┐                           │  
│  │Azure Monitor  │                           │  
│  │& Log Analytics │                           │  
│  └─────────────┘                           │  
└─────────────────────────────┘  
 
この図は、東日本リージョンにおける主要なコンポーネントおよびそれらの接続を示しています。実際の図を描画する際は、このテキスト表現を元にしてください。
```

Azure SQL Database 部分が Primary になっています。この辺りも吸収してくれるのはなかなかです。こんな感じで壁打ちをしながらのアーキテクチャ設計をすれば従来よりは相当負荷軽減になると思います。

** 月額利用料を見積もってもらう

これまたよくある話なので、月額利用料の見積もりもしてみましょう。実際によく聞かれるような雑な質問からふってみます。

```txt
Contoso社の要件に基づき、月額利用料を出力してください。
```

結果は以下になります。

```txt
月額利用料を算出するには、具体的な要件やリソースの詳細（たとえば、使用するリソースの種類、数、リージョン、スケール、使用量など）を知る必要があります。しかし、一般的なAzureリソースの月額利用料を概算するためのサンプルを提供します。このサンプルには、主要なコンポーネントの基本的な使用量を含めています。

仮定された構成要素と使用量
 - Azure Virtual Network (VNet): 無料
 - Azure VPN Gateway: Standard SKU
 - Azure Application Gateway: Standard_v2
 - Azure App Service: S1 Standard Plan (3つのWebアプリ)
 - Azure SQL Database: Business Critical, 4 vCores
 - Azure Key Vault: Standard Tier
 - Azure Backup: 500 GB
 - Azure Monitor & Log Analytics: 100 GBのログデータ

概算月額利用料
以下の価格は2023年の一般的な価格を基にしていますが、最新の価格はAzureの公式サイトで確認してください。
 - Azure VPN Gateway (Standard SKU)
   約 $125/月
 - Azure Application Gateway (Standard_v2)
   約 $150/月
 - Azure App Service (S1 Standard Plan)
   約 $75/月 x 3 = $225/月
 - Azure SQL Database (Business Critical, 4 vCores)
   約 $750/月
 - Azure Key Vault (Standard Tier)
   約 $4/月
 - Azure Backup (500 GB)
   約 $25/月
 - Azure Monitor & Log Analytics (100 GB)
   約 $20/月

合計
  Azure VPN Gateway:            $125  
  Azure Application Gateway:    $150  
  Azure App Service:            $225  
  Azure SQL Database:           $750  
  Azure Key Vault:              $4  
  Azure Backup:                 $25  
  Azure Monitor & Log Analytics: $20  
-------------------------------------  
合計:                        約 $1,299/月  
 
この計算はあくまでも概算であり、実際の使用量や構成により変動します。詳細な見積もりを行うためには、Azure Pricing Calculator（https://azure.microsoft.com/ja-jp/pricing/calculator/）を使用することをお勧めします。このツールでは、リソースの詳細な設定や地域ごとの価格を指定して、より正確な見積もりを得ることができます。
```

複数リージョンの話やデータ通信料が含まれていないことが分かります。そこを踏まえてもう少しプロンプトを変えてみます。

```txt
Contoso社の要件に基づき、東日本、アメリカ東海岸、西ヨーロッパ毎の月額利用料、全体での月額利用料をそれぞれ出力してください。内部ユーザは 500 人、外部ユーザは 100,000 人を想定してデータ通信料も可能な限り見積もりに含んでください。
```

```txt
以下は、Contoso社の要件に基づく東日本、アメリカ東海岸、西ヨーロッパ各リージョンの月額利用料と全体の月額利用料の概算です。内部ユーザ500人、外部ユーザ100,000人を想定し、データ通信料も含めています。

リージョンごとの主要なリソースと使用量
- Azure Virtual Network (VNet)
- Azure VPN Gateway: Standard SKU
- Azure Application Gateway: Standard_v2
- Azure App Service: P1v2 Premium Plan (3つのWebアプリ)
- Azure SQL Database: Business Critical, 8 vCores
- Azure Key Vault: Standard Tier
- Azure Backup: 500 GB
- Azure Monitor & Log Analytics: 200 GBのログデータ
- データ通信料: 500人の内部ユーザと100,000人の外部ユーザのデータ通信量を考慮

リージョンごとの月額利用料
 
東日本リージョン
- Azure VPN Gateway (Standard SKU): 約 $125/月
- Azure Application Gateway (Standard_v2): 約 $150/月
- Azure App Service (P1v2 Premium Plan): 約 $230/月 x 3 = $690/月
- Azure SQL Database (Business Critical, 8 vCores): 約 $1,500/月
- Azure Key Vault (Standard Tier): 約 $4/月
- Azure Backup (500 GB): 約 $25/月
- Azure Monitor & Log Analytics (200 GB): 約 $40/月
- データ通信料: 約 $200/月
  合計: 約 $2,734/月

アメリカ東海岸リージョン
- Azure VPN Gateway (Standard SKU): 約 $125/月
- Azure Application Gateway (Standard_v2): 約 $150/月
- Azure App Service (P1v2 Premium Plan): 約 $230/月 x 3 = $690/月
- Azure SQL Database (Business Critical, 8 vCores): 約 $1,500/月
- Azure Key Vault (Standard Tier): 約 $4/月
- Azure Backup (500 GB): 約 $25/月
- Azure Monitor & Log Analytics (200 GB): 約 $40/月
- データ通信料: 約 $200/月
  合計: 約 $2,734/月

西ヨーロッパリージョン
- Azure VPN Gateway (Standard SKU): 約 $125/月
- Azure Application Gateway (Standard_v2): 約 $150/月
- Azure App Service (P1v2 Premium Plan): 約 $230/月 x 3 = $690/月
- Azure SQL Database (Business Critical, 8 vCores): 約 $1,500/月
- Azure Key Vault (Standard Tier): 約 $4/月
- Azure Backup (500 GB): 約 $25/月
- Azure Monitor & Log Analytics (200 GB): 約 $40/月
  データ通信料: 約 $200/月
  合計: 約 $2,734/月

全体の月額利用料
- 東日本リージョン:        約 $2,734/月  
- アメリカ東海岸リージョン:  約 $2,734/月
```

途中でメッセージが途切れていますが、かなり良い情報が出ています。実際のデータ通信料や利用料は当然微調整を行う必要はあるでしょうが、概算で出すものとしてはまずますという認識です。

** 結論
ご覧の通り、かなりの情報が出力されます。当然ながら突っ込みどころもかなりあるので、最終チェックにそれなりの経験がある人間が関与しないといけないことは間違いありません。一方で、どういった Azure の機能を使うかのめぼしをつけたり、月額利用料等の概算でも良いので無理くり出したいようなパターンとしてはかなり優秀です。
加えて、これは OpenAI を使う場合の大原則だと思いますが「あいまいな質問をするとあいまいな答えしか返ってこない」という点です。上記でもあれこれ試行錯誤しましたが、入力する情報量を増やせば増やすほどに OpenAI 側の返す情報の確度が上がっていることが分かります。

こちらの記事が皆様の参考になれば幸いです。
