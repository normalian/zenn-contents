---
title: "Azure Health Data Service の $convert-data endpoint を利用する"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Azure Health Data Service", "csharp"]
published: true
publication_name: "microsoft"
---

前回の記事では Azure Health Data Services の FHIR サービスに FHIR 形式のデータ（ JSON ぽいあれ ）を格納する手順を紹介しました。次は以下のデータ変換の API を呼ぶ手順を実例ベースで紹介したいと思います。
[Configure settings for $convert-data by using the Azure portal](https://learn.microsoft.com/en-us/azure/healthcare-apis/fhir/convert-data-configuration)

Azure Data Factory も以下の様にデータ変換のコンポーネントを持っている風ですが、実態は上記の $convert-data API を Azure Data Factory で呼んでいるだけなので、重要なのは API の使い方自体になるのは御推察頂けると思います。加えて、同 API 自体は Docker コンテナイメージとして公開しているのでオンプレでの利用も可能です。
https://learn.microsoft.com/en-us/azure/healthcare-apis/fhir/convert-data-azure-data-factory


GitHub で公開されている FHIR Converter

https://github.com/microsoft/fhir-converter?tab=readme-ov-file

を参照してもらえればと思いますが、以下の変換に対応しています。
- HL7v2 to FHIR
- C-CDA to FHIR	
- JSON to FHIR	
- FHIR STU3 to R4
- FHIR to HL7v2 (preview)

Azure Health Data Services では FHIR 形式、DICOM 形式、MedTech 形式のみ build-in での保存をサポートしているので、他のデータ形式（HL7v2 とか）の場合はこの API のお世話になる必要があるということです。

## HL7v2 から FHIR 形式の変換を試す

HL7v2 形式というデータ形式に覚えがない人も多いと思いますので、具体例として [$convert-data in the FHIR service](https://learn.microsoft.com/en-us/azure/healthcare-apis/fhir/convert-data-overview) の記事に以下の様なサンプルがあります。

```sql

MSH|^~\\&|SIMHOSP|SFAC|RAPP|RFAC|20200508131015||ADT^A01|517|T|2.3|||AL||44|ASCII\nEVN|A01|20200508131015|||C005^Whittingham^Sylvia^^^Dr^^^DRNBR^D^^^ORGDR|\nPID|1|3735064194^^^SIMULATOR MRN^MRN|3735064194^^^SIMULATOR MRN^MRN~2021051528^^^NHSNBR^NHSNMBR||Kinmonth^Joanna^Chelsea^^Ms^^D||19870624000000|F|||89 Transaction House^Handmaiden Street^Wembley^^FV75 4GJ^GBR^HOME||020 3614 5541^PRN|||||||||C^White - Other^^^||||||||\nPD1|||FAMILY PRACTICE^^12345|\nPV1|1|I|OtherWard^MainRoom^Bed 183^Simulated Hospital^^BED^Main Building^4|28b|||C005^Whittingham^Sylvia^^^Dr^^^DRNBR^D^^^ORGDR|||CAR|||||||||16094728916771313876^^^^visitid||||||||||||||||||||||ARRIVED|||20200508131015||

```

上記はのデータは HL7v2 の ADT（Admission, Discharge, and Transfer） メッセージの一種で、患者の新規登録（A01：患者入院（入院または登録））を表します。こちらも元記事の API に記載がありますが、そのためのパラメータである ADT_A01 も指定する必要があります。
では早速実際に処理を行うソースコードを例示します。元記事のサンプルの文字列を利用したらデータ形式がやや異なっていたようなので、ソースコード内に注釈をつけました。利用するライブラリは以下です。
- Hl7.Fhir.R4
- Azure.Identity
- Newtonsoft.Json

```csharp

using System;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;
using System.Threading.Tasks;
using Azure.Core;
using Azure.Identity;
using Hl7.Fhir.Model;
using Hl7.Fhir.Serialization;
using Newtonsoft.Json;

public class AHDSConvertRequest
{
    [JsonProperty("resourceType")]
    public string ResourceType { get; set; }

    [JsonProperty("parameter")]
    public List<Parameter> Parameter { get; set; }
}

public class Parameter
{
    [JsonProperty("name")]
    public string Name { get; set; }

    [JsonProperty("valueString")]
    public string ValueString { get; set; }
}
class Program
{
    private static readonly string tenantId = "your-entraid-tenant-id";
    private static readonly string fhirBaseUrl = "https://your-ahds-endpoint.fhir.azurehealthcareapis.com";
    private static readonly HttpClient client = new HttpClient();

    static async System.Threading.Tasks.Task Main()
    {
        var token = await GetAccessTokenAsync();
        await ConvertDataHL7v2ToFHIR(token);
    }
    static async System.Threading.Tasks.Task<string> GetAccessTokenAsync()
    {
        // 自分の Azure Cli でログインしているアカウントのテナント ID を使う
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
    static async System.Threading.Tasks.Task ConvertDataHL7v2ToFHIR(string accessToken)
    {
        // 元記事のデータは "gender": "196" という値になっていたので、male という値に修正
        var hl7v2Data = "MSH|^~\\&|SENDING_APPLICATION|SENDING_FACILITY|RECEIVING_APPLICATION|RECEIVING_FACILITY|20110613083637||ADT^A04|1817457|P|2.3.1\rEVN|A04|20110613083637|||^KOCH^LEONARD^J^^DR^^^L|||ADM|A\rPID|||56782445^^^2^ID 1|454721||DOE^JOHN^^^^||male";
        string fhirConverterUrl = $"{fhirBaseUrl.TrimEnd('/')}/$convert-data";

        var inputData = new AHDSConvertRequest
        {
            ResourceType = "Parameters",
            Parameter = new List<Parameter>
            {
                new Parameter{ Name = "inputData", ValueString = hl7v2Data },
                new Parameter{ Name = "inputDataType", ValueString = "Hl7v2" },
                new Parameter{ Name = "templateCollectionReference", ValueString = "microsofthealth/fhirconverter:default" },
                new Parameter{ Name = "rootTemplate", ValueString = "ADT_A01" }
            }
        };

        client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);
        client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));

        string jsonContent = JsonConvert.SerializeObject(inputData);
        var content = new StringContent(jsonContent, Encoding.UTF8, "application/json");

        HttpResponseMessage response = await client.PostAsync(fhirConverterUrl, content);

        if (response.IsSuccessStatusCode)
        {
            string result = await response.Content.ReadAsStringAsync();
            Console.WriteLine("FHIR Converted Data: ");
            Console.WriteLine(result);

            // 元記事のデータは "gender": "196" という値の場合、Hl7.Fhir.Model.Patient.Gender の AdministrativeGender 列挙型に適合しないので、
            // 厳密な型チェックを無効化する必要がある
            // var parserSettings = new PocoBuilderSettings { AllowUnrecognizedEnums = true };
            // var parser = new FhirJsonParser(new ParserSettings { AllowUnrecognizedEnums = true });
            var parser = new FhirJsonParser();

            Bundle bundle = parser.Parse<Bundle>(result);
            foreach (var entry in bundle.Entry)
            {
                var patient = entry.Resource as Patient;
                if (patient != null && patient.Name.Count > 0)
                {
                    Console.WriteLine($"Patient: {patient.Name[0].GivenElement[0].Value} {patient.Name[0].Family}");
                }
                var practitioner = entry.Resource as Practitioner;
                if (practitioner != null)
                {
                    Console.WriteLine($"Patient: {practitioner.Name[0].GivenElement[0].Value} {practitioner.Name[0].Family}");
                }

            }
        }
        else
        {
            Console.WriteLine($"Error: {response.StatusCode}");
            Console.WriteLine(await response.Content.ReadAsStringAsync());
        }
    }
}

```

こちらもやっぱり日本語の記事を見かけなかったので作成してみました。誰かの参考になれば幸いです。
