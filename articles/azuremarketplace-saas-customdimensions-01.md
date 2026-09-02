---
title: "Azure Marketplace の SaaS type offer の課金で custom dimensions を試す"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AzureMarketplace", "csharp"]
published: true
publication_name: "microsoft"
---

皆さんは「Azure Marketplace から SendGrid を購入して使った」という様な Azure Marketplace のユーザであることは多いと思いますが、Azure Marketplace 上にサービスを公開した経験は絶無ではないかと想定しています。以下の「Mastering the Marketplace」は激熱な情報がそろっているのですが、英語のみの提供となり、日本語でこの手の話題を記載しているのをほぼ見ないからです。
https://microsoft.github.io/Mastering-the-Marketplace/

特に Azure Marketplace 上で課金を発生させる Transactable Offer と呼ばれるものは一定のお作法でサービスを構築する必要があります。Transactable Offer としてサービスを公開することで、Azure Portal や Marketplace Portal からの導線を得られるだけでなく、課金や領収書発行等を Azure Marketplace 側に任せることができます。特に有名な Transactable Offer は以下の４個です。
- Azure Virtual Machine
- Azure Container
- Azure Application（Managed Application というのは、この offer type のパッケージの固め方のお作法の一つ）
- SaaS https://learn.microsoft.com/en-us/marketplace/purchase-saas-offer-in-azure-portal

Azure Virtual Machine の offer type に関しては「F5 の様なネットワークアプライアンス」や「SQL Server AlwaysOn の可用性グループ」を想定してくれると分かりやすいと思います。同 offer type は「VM をデプロイリソース群に入れてね！」という制約はありますが、他のリソースも ARM テンプレートの範囲内でデプロイできるという柔軟性があります。Azure Container の offer type の詳細は避けますが「VM の代わりに AKS をデプロイリソース群に入れてね！」という offer type と言えば予想は着くと思います。

Azure Application は語りだすと止まらなくなりそうなので話題を避け、今回は SaaS について深堀したいと思います。

## SaaS type offer とは
本 offer type は他とは一線を画しまして、既存の自身で作成済のサービスが存在することが前提になります。既存サービスに対する Azure Marketplace からの導線として、Entra ID を利用した認証・認可のイベントを受け取る Web アプリケーションを自作し、サービスの購入・課金・解約等を制御する流れになります。
当たり前ですが、そんな Web アプリケーションを一から試行錯誤で自作するのは現実的ではありません。そのために Microsoft は以下の SaaS Accelerator と呼ばれるテンプレートを用意しています。
https://github.com/Azure/Commercial-Marketplace-SaaS-Accelerator

上記をデプロイすると Azure Marketplace からリダイレクトされる先の Web ポータル、Azure Marketplace からのイベントを受け取るための Web サービス、ユーザの購入情報等が格納される SQL DB 等がデプロイ・設定することができます。
こいつら自身は Azure Marketplace との連携機能以外は何の機能も提供しません。Azure Marketplace 上でサービスを購入後に自身のサービスを提供するサイトに誘導する場合、上記の SaaS Accelerator のアプリを少しカスタマイズしてリダイレクト処理を入れるといった流れが必要です。

## 本題の custom dimensions とは？
SaaS type offer には標準的な課金方式があります。以下の様に「月 or 年額いくら」や「ユーザ単位でいくら」という課金単位なら比較的簡単です。
![](/images/azuremarketplace-saas-customdimensions-01/standard-billing-01.png) 
ポイントは上記に該当しない場合でしょう。分かりやすいのが SendGrid さん等で「メールを何通送ったら幾ら」というもので、これは上記の課金方式に該当しません。

これを解決するために存在するのが Custom Dimensions です。以下に公式サイトがあります。
https://learn.microsoft.com/en-us/partner-center/marketplace-offers/saas-metered-billing

こちらを利用するには「こういう項目に対し、この数量に対して、これだけ課金して」というのを Partner Center Portal 上で定義することで利用可能になります。実際の利用の流れは以下です。
- Partner Center Portal 上で meter 定義
- Service Principal 等の設定（ Commercial-Marketplace-SaaS-Accelerator を利用している場合、そちらが作成した Service Principal を使う）
- 自身の pro-code で課金 API 呼ぶ

最後の「自身の pro-code で課金 API 呼ぶ」はハードルの高い方も多いと思います。そうです。課金を細かくカスタマイズする場合、自分で項目定義をした後は「いつ課金するか」や「どう課金するか」は自分で API を呼んで制御する必要があるのです（まぁ、プラットフォーム側としても全ての課金シナリオに対応できないので、折衷案なのかもしれませんが）。

## 実際に custom dimensions を使ってみる
上記で記載した通り、まずは Partner Center Portal 上で項目を定義します。以下は既に項目定義後に Offer を Publish しています。一度公開するとその後は同じ Plan 内では項目を編集できないので注意してください。
![](/images/azuremarketplace-saas-customdimensions-01/custom-dimensions-01.png) 

次に Publisher 側（サービス公開側）の Entra ID から以下の様に「XXXXXXXXXXx--FulfillmentAppReg」という App Registrations を発見できるはずです。ここから tenant id/client id/client secret を取得しましょう。
![](/images/azuremarketplace-saas-customdimensions-01/sp-01.png) 

さらに誰が課金するかを判定するためには marketplace-subscription-id が必要なのですが、実際の Azure Subscription ID ではありません。SaaS Accelerator で作成された DB に登録された ID になります。今回はサンプルなので、以下のポータルから直接取得します。
![](/images/azuremarketplace-saas-customdimensions-01/marketplace-subscription-id-01.png) 


では実際のソースコードです。今まで取得した値を以下で置き換えて利用します。

```csharp

using Azure.Core;
using Azure.Identity;
using System.Net.Http.Headers;
using System.Net.Http.Json;

internal static class Program
{
    private static async Task Main()
    {
        var tenantId = "<publisher-tenant-id>";
        var clientId = "<marketplace-aad-client-id>";
        var clientSecret = "<client-secret>";

        var subscriptionId =
            Guid.Parse("<marketplace-subscription-id>");

        var planId = "test_private_offer";
        var dimension = "qa_email_num";

        // 利用発生時間をUTCで指定
        var effectiveStartTimeUtc = DateTime.UtcNow;

        var requestBody = new
        {
            resourceId = subscriptionId,
            quantity = 1,
            dimension,
            effectiveStartTime = effectiveStartTimeUtc,
            planId
        };

        var credential = new ClientSecretCredential(
            tenantId,
            clientId,
            clientSecret);

        var token = await credential.GetTokenAsync(
            new TokenRequestContext(
            [
                "20e940b3-4c77-4b0b-9a53-9e16a1b010a7/.default"
            ]));

        using var httpClient = new HttpClient();

        httpClient.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token.Token);

        httpClient.DefaultRequestHeaders.Accept.Add(
            new MediaTypeWithQualityHeaderValue("application/json"));

        var url =
            "https://marketplaceapi.microsoft.com/api/usageEvent" +
            "?api-version=2018-08-31";

        using var response =
            await httpClient.PostAsJsonAsync(url, requestBody);

        var responseBody = await response.Content.ReadAsStringAsync();

        Console.WriteLine($"HTTP Status: {(int)response.StatusCode}");
        Console.WriteLine(responseBody);
    }
}


```

アプリケーションの実行後は以下の様な結果となります。
![](/images/azuremarketplace-saas-customdimensions-01/program-result-01.png) 

上記では API のレスポンスとして Accepted が返ってきているので、Azure Marketplace 側で課金を受け付けたことが分かります。

## 実行結果のチェック
上記を実行後、24時間以上経っても顧客側の Azure Portal/Partner Center Portal の課金メーターに反映されていませんでした。「どういうことだ？」と思い、以下のコマンドを実行しました。

```powershell
# Publisher Tenant
$tenantId     = "publisher-tenant-id"
$clientId     = "marketplace-aad-client-id"
$clientSecret = "client-secret"

# Marketplace API Resource
$scope = "20e940b3-4c77-4b0b-9a53-9e16a1b010a7/.default"

# AAD Token取得
$body = @{
    client_id     = $clientId
    client_secret = $clientSecret
    scope         = $scope
    grant_type    = "client_credentials"
}

$token = (
    Invoke-RestMethod `
        -Method POST `
        -Uri "https://login.microsoftonline.com/$tenantId/oauth2/v2.0/token" `
        -Body $body `
        -ContentType "application/x-www-form-urlencoded"
).access_token

# 過去24時間のUsage Event取得
$headers = @{
    Authorization = "Bearer $token"
}

$usageStartDate = [System.Web.HttpUtility]::UrlEncode((Get-Date).AddDays(-7).ToUniversalTime().ToString("o"))
 
$uri = "https://marketplaceapi.microsoft.com/api/usageEvents" +"?api-version=2018-08-31" +"&usageStartDate=$usageStartDate"

Invoke-RestMethod `
    -Method GET `
    -Uri $uri `
    -Headers $headers
```

実行した結果、以下の様に課金ステータスが submitted 状態で processed になっていないことが分かりました。どの程度で Partner Center Portal 側に具体的に反映されるか分かりませんが、途中経過として。
```text
usageDate           : 2026-08-31T00:00:00Z
usageResourceId     : 8cc9c3f2-a7a9-421b-c0fd-xxxxxxxxxxxxx
dimension           : qa_email_num
planId              : test_private_offer
planName            : 
offerId             : your-offer-id
offerName           : 
offerType           : SaaS
azureSubscriptionId : 6ad3de36-08ca-45ea-b674-xxxxxxxxxxxxx
reconStatus         : Submitted
submittedQuantity   : 1.0
processedQuantity   : 0.0
submittedCount      : 1
```

## まとめ

本当は「API でエラー帰ってきたらどうするの？」等を考慮する必要は当然ありますし、API が一時間ごとの課金なので、短時間で送りまくると上手く課金されなかったりします。現実的には個別のデータストアを用意し、そちらから１時間ごとに実行する様なバッチで実行するのが良いでしょう。とりあえずまずは基本のノウハウです。

## References
- https://learn.microsoft.com/en-us/partner-center/marketplace-offers/marketplace-metering-service-apis
- https://learn.microsoft.com/en-us/partner-center/marketplace-offers/pc-saas-fulfillment-subscription-api#get-list-of-all-subscriptions