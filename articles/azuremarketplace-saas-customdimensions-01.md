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
本 offer type は他のものとは一風をかくしまして、既存サービスが存在することが前提になります。既存サービスに対する Azure Marketplace からの導線として、Entra ID を利用した認証・認可の導線となるイベントを受け取る Web アプリケーションを自作成してサービスの購入・課金・解約等を制御する流れになります。
当たり前ですが、そんな Web アプリケーションを一から自作するのは現実的ではありません。そのために Microsoft は以下の SaaS Accelerator と呼ばれるテンプレートを用意しています。
https://github.com/Azure/Commercial-Marketplace-SaaS-Accelerator

上記をデプロイすると Azure Marketplace からリダイレクトされる先の Web ポータル、Azure Marketplace からのイベントを受け取るための Web サービス、ユーザの購入情報等が格納される SQL DB 等がデプロイされます。
こいつら自身は何のサービスも提供しないので、実際にサービスを提供する際は上記の SaaS Accelerator のアプリを少しカスタマイズしてリダイレクト処理を入れるといった流れが必要だと思います。

## 本題の custom dimensions とは？
SaaS type offer には標準的な課金方式があります。以下の様に「月 or 年額いくら」や「ユーザ単位でいくら」という課金単位なら比較的簡単です。
![](/images/azuremarketplace-saas-customdimensions-01/standard-billing-01.png) 
ポイントは上記に該当しない場合でしょう。分かりやすいのが SendGrid さん等で「メールを何通送ったら幾ら」というのは上記の課金方式に該当しません。

これを解決するために存在するのが Custom Dimensions です。以下に公式サイトがあります。
https://learn.microsoft.com/en-us/partner-center/marketplace-offers/saas-metered-billing

こちらを利用することで「こういう項目に対し、この数量に対して、これだけ課金して」というのを Partner Center Portal 上で定義することで利用可能になります。実際の利用の流れは以下です。
- Partner Center Portal 上で meter 定義
- Service Principal 等の設定（ Commercial-Marketplace-SaaS-Accelerator を利用している場合、そちらが作成した Service Principal を使う）
- 自身の pro-code で課金 API 呼ぶ

最後の「自身の pro-code で課金 API 呼ぶ」はハードルの高い方も多いと思います。そうです。課金を細かくカスタマイズする場合、自分で項目的義をした後はいつ課金するか、どう課金するかは自分で API を呼んで制御する必要があるのです（まぁ、全てのビジネスシナリオにプラットフォーム側としても対応できないので、折衷案なのかもしれませんが）。

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

実行後は以下の様な結果となります。
![](/images/azuremarketplace-saas-customdimensions-01/program-result-01.png) 

上記で Accepted が返ってきているので、Azure Marketplace 側で課金を受け付けたことが分かります。本当は「API でエラー帰ってきたらどうするの？」や API が一時間ごとの課金なので、短時間で送りまくるとだめだったりします。さらに、課金が実際に表示されるのは翌日以降だったりと色々とあるのですが、とりあえずまずは基本のノウハウです。

