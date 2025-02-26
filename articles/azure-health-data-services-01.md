---
title: "Azure Health Data Service のことはじめ"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Azure Health Data Service", "csharp"]
published: true
publication_name: "microsoft"
---

この記事を読んでいる方の中で Azure Health Data Services を聞いたことがある方はどの程度いらっしゃるでしょうか？正直言って Azure Health Data Services を実際に触ったことのある方はごく一部なのではないかと思ったのでこの記事をまとめみました。
[What is Azure Health Data Services?](https://learn.microsoft.com/en-us/azure/healthcare-apis/healthcare-apis-overview)

どの様なサービスかというと、業界標準で決まっている Healthcare 系のデータを格納できるようになっています。Azure Health Data Services の格納できるデータ形式で以下の様に FHIR と DICOM がありますが、Microsoft や Azure 縛りでなく、業界標準で定められているフォーマットになり、こちらを利用できるのが Azure Health Data Services という位置づけです。
- FHIR https://www.hl7.org/fhir/overview.html
- DICOM https://www.dicomstandard.org/

ちなみに元々は Azure API for FHIR というサービスが存在したのですが、同サービスは 2026年9月30日に retire 予定です。新規にサービス構築する際は今回紹介する Azure Health Data Services FHIR service を利用しましょう。
[Frequently asked questions about FHIR service](https://learn.microsoft.com/en-us/azure/healthcare-apis/fhir/fhir-faq)

## Azure Health Data Services の FHIR リソース作成

Azure Portal 上からは以下の様に Health Data で検索すると文字が出てきます。 
![](/images/azure-health-data-services-01/image01.png) 

作成時に workspace 名が求められるだけなので、特に困るところはないと思います。次の Networking タブで Private Endpoint を含む閉域設定が可能なので、Healthcare の様な求められるコンプライアンスやルールが多い業態では重要なポイントとなりますが、今回紹介する内容とはやや主軸がずれるので割愛します。日本のリージョンでは Japan East のみ選択可能です。
![](/images/azure-health-data-services-01/image02.png) 

Azure Health Data Services の workspace 作成後、リソースを参照すると以下の画面となります。
![](/images/azure-health-data-services-01/image03.png) 

Overview 画面下部にも存在しますが、FHIR、DICOM、MedTech というそれぞれ異なるサービスがぶら下がっていることが分かると思います。Azure Health Data Services の Workspace 作成後に実際に利用するサービスを更に作成する流れです。

今回は FHIR サービスを利用してみましょう。左メニューから FHIR service を選択して新規に作成しましょう。以下の様に R4 と STU 3 のバージョンを選択できますが、今回は最新版である R4 を選択します。
![](/images/azure-health-data-services-01/image04.png) 

リソース作成後は Properties タブに移動し、そこで　Audience の URL をコピーします。これで作業準備は完了です。

![](/images/azure-health-data-services-01/image05.png) 


## Azure Health Data Services の FHIR にレコードを登録する

次に Visual Studio を起動して .NET Core Console アプリを新規に作成します。次に NuGet で以下のライブラリを取得します。
- Hl7.Fhir.R4
- Azure.Identity

名前の通り、Hl7.Fhir.R4 は業界標準の OSS ライブラリを使用します。今回は自分の組織アカウントを利用して認証していますが、実際のシステムでは Service Principal の利用を検討ください。今回のサンプルでは患者のレコード作成後に取得しているだけですが、参考になるかと。

API 仕様については以下を参照ください。
[REST API capabilities in the FHIR service in Azure Health Data Services](https://learn.microsoft.com/en-us/azure/healthcare-apis/fhir/rest-api-capabilities)


```csharp

using System;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Threading.Tasks;
using Azure.Core;
using Azure.Identity;
using Hl7.Fhir.Model;
using Hl7.Fhir.Serialization;

class Program
{
    private static readonly string fhirBaseUrl = "https://your-azure-health-data-services-fhir-url.azurehealthcareapis.com";

    static async System.Threading.Tasks.Task Main()
    {
        var httpClient = new HttpClient();

        // Authenticate using DefaultAzureCredential
        var token = await GetAccessTokenAsync();
        httpClient.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);

        // Create a Patient resource using FHIR SDK
        var patient = new Patient
        {
            Name = new[] { new HumanName { Family = "Test", Given = new[] { "Taro" } } }.ToList(),
            Gender = AdministrativeGender.Male,
            BirthDate = "2006-06-15"
        };

        // Serialize to JSON using FHIR Serializer
        var fhirSerializer = new FhirJsonSerializer();
        var patientJson = fhirSerializer.SerializeToString(patient);

        var patientId = await CreatePatient(httpClient, patientJson);
        Console.WriteLine($"Patient Created with ID: {patientId}");

        // Read the created patient
        await GetPatient(httpClient, patientId);
    }

    static async System.Threading.Tasks.Task<string> GetAccessTokenAsync()
    {
        var option = new DefaultAzureCredentialOptions()
        {
            ExcludeEnvironmentCredential = true,
            ExcludeWorkloadIdentityCredential = true,
            ExcludeManagedIdentityCredential = true,
            ExcludeSharedTokenCacheCredential = true,
            ExcludeVisualStudioCredential = true,
            ExcludeVisualStudioCodeCredential = true,
            TenantId = tenantId,
        };
        var credential = new DefaultAzureCredential(option);
        var tokenRequestContext = new TokenRequestContext(new[] { fhirBaseUrl + "/.default" });
        var tokenRequest = await credential.GetTokenAsync(tokenRequestContext);
        return tokenRequest.Token;
    }

    static async System.Threading.Tasks.Task<string> CreatePatient(HttpClient httpClient, string patientJson)
    {
        var response = await httpClient.PostAsync($"{fhirBaseUrl}/Patient", new StringContent(patientJson, System.Text.Encoding.UTF8, "application/fhir+json"));
        response.EnsureSuccessStatusCode();

        var responseBody = await response.Content.ReadAsStringAsync();
        Console.WriteLine(responseBody);

        var fhirParser = new FhirJsonParser();
        var patientResponse = fhirParser.Parse<Patient>(responseBody);
        return patientResponse.Id;
    }

    static async System.Threading.Tasks.Task GetPatient(HttpClient httpClient, string patientId)
    {
        var response = await httpClient.GetAsync($"{fhirBaseUrl}/Patient/{patientId}");
        response.EnsureSuccessStatusCode();

        var responseBody = await response.Content.ReadAsStringAsync();
        var fhirParser = new FhirJsonParser();
        var patient = fhirParser.Parse<Patient>(responseBody);

        Console.WriteLine($"Patient Retrieved: {patient.Name.ElementAtOrDefault(0).Given.FirstOrDefault()} {patient.Name.ElementAtOrDefault(0).Family}");
    }
}


```

これを実行すると以下の様になります。

![](/images/azure-health-data-services-01/image06.png) 

あんまりにも日本語の記事を見かけなかったので、誰かの参考になれば幸いです。
