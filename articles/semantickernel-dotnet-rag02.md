---
title: "C# Semantic Kernel で簡単 RAG のサンプル ～InMemoryVectorStore編～"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["SemanticKernel", "AI", "csharp"]
published: true
publication_name: "microsoft"
---

先日に[C# Semantic Kernel で簡単 RAG のサンプル ～VolatileMemoryStore編～](https://zenn.dev/microsoft/articles/semantickernel-dotnet-rag01)という記事を記載した後、すぐに Kazuki Ota さんが [.NET + Semantic Kernel でベクトル検索の RAG をする](https://zenn.dev/microsoft/articles/semantic-kernel-dotnet-rag01) という記事を記載してくれました。
Semantic Kernel で RAG を実装する場合のベクトルストア周りの実装はなかなか GA にならずにどうしたものかと思っていたら、MS Build 2025 にあわせて以下の記事が発表されていました。

@[card](https://devblogs.microsoft.com/semantic-kernel/vector-data-extensions-are-now-generally-available-ga/)

詳細な内容は記事自体を参照頂ければと思いますが、要は「ベクトルストアのインターフェース部分は GA したよ」と「AI Search 等の実装部分に関しては coming soon で GA だよ」だと思っています。今までは breaking change が怖くて使いにくかったですが、ある程度安定して来たのではないかと思います（2025年5月現在）。

## 今回の環境

今回は C# で Semantic Kernel を触りましたが、利用したライブラリは以下となります。Connector はまだ preview ですが、前回は InMemory 周りが alpha だったので、進捗があったのが理解できると思います。
- Microsoft.Extensions.AI.OpenAI: 9.5.0-preview.1.25265.7
- Microsoft.SemanticKernel: 1.54.0
- Microsoft.SemanticKernel.Connectors.AzureOpenAI: 1.54.0
- Microsoft.SemanticKernel.Connectors.InMemory: 1.54.0-preview
- Microsoft.SemanticKernel.PromptTemplates.Handlebars: 1.54.0

## ベクトルストアの既存ドキュメントや参考情報を確認

次に RAG 実装のためにプロンプトからベクトルデータを検索する方法について調べたいと思います。こちらに関しては以下が参考になります。
@[card](https://learn.microsoft.com/ja-jp/semantic-kernel/concepts/text-search/text-search-vector-stores?pivots=programming-language-csharp)

同記事にも記載がありますが、ベクトルストアのコレクションを VectorStoreTextSearch でラップしたり、更に Plugin 化したりする必要があったり、やや難解です。つまり、単にベクトルストアのコレクションのクラスをインスタンス化するだけでなく、一定のステップを踏む必要があるということです。

加えて、ベクトルストアのデータを保存するクラスを以下の属性を利用して作成する必要があります。以下は公式記事からの抜粋です。
- TextSearchResultValue - AI モデルが質問に回答するために使用するテキスト データなど、 TextSearchResultの値となるデータ モデルのプロパティにこの属性を追加します。
- TextSearchResultName - この属性を、 TextSearchResultの名前となるデータ モデルのプロパティに追加します。
- TextSearchResultLink - この属性を、 TextSearchResultへのリンクとなるデータ モデルのプロパティに追加します。

上記は複数のプロパティに付与してはいけない様です。データ構造を変換する場合にはカスタムマッパーと呼ばれるものを作成して変更できるようですが、今回は割愛します。詳細は当該リンク先を参照ください。

## Semantic Kernel の実行ソースコード

色々と上記で記載しましたが、以下に実装例を記載します。Visual Studio を立ち上げて以下の C# のソースコード を作成してください。

```csharp
using Azure.AI.OpenAI;
using Microsoft.Extensions.AI;
using Microsoft.Extensions.VectorData;
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Connectors.InMemory;
using Microsoft.SemanticKernel.Data;
using Microsoft.SemanticKernel.PromptTemplates.Handlebars;


#pragma warning disable SKEXP0001, SKEXP0050
class Program
{
    static async Task Main()
    {
        string endpoint = "https://your-azureopenai-endpoint.openai.azure.com/";
        string apikey = "YOUR-AzureOpenAI-APIKEY";
        string collectionName = "sample-collection";
        string chatModelName = "gpt-4o";
        string embeddingModelName = "text-embedding-3-large";

        string query = "Who is Daichi";

        // initiate Semantic Kernel 
        var kernelBuilder = Kernel.CreateBuilder();
        kernelBuilder.AddAzureOpenAIChatCompletion(
            deploymentName: chatModelName,
            endpoint: endpoint,
            apiKey: apikey
        );
        var kernel = kernelBuilder.Build();

        IEmbeddingGenerator<string, Embedding<float>> embeddingGenerator =
               new AzureOpenAIClient(new Uri(endpoint), new System.ClientModel.ApiKeyCredential(apikey))
               .GetEmbeddingClient(embeddingModelName)
               .AsIEmbeddingGenerator();

        VectorStore vectorStore = new InMemoryVectorStore( new InMemoryVectorStoreOptions
        {
            EmbeddingGenerator = embeddingGenerator
        });

        var vectorStoreCollection = vectorStore.GetCollection<string, Person>(collectionName);
        await vectorStoreCollection.EnsureCollectionExistsAsync();
        await vectorStoreCollection.UpsertAsync(new Person()
        {
            Id = 1.ToString(),
            Name = "Daichi Isami",
            Value = "Based in Redmond, Washington. Enjoys traveling and exploring local food and drinks.",
        });
        await vectorStoreCollection.UpsertAsync(new Person()
        {
            Id = 2.ToString(),
            Name = "Kazuki Sato",
            Value = "Based in Tokyo, Japan. A great C# developer and Cloud Solution Architect in BigSoft.",
        });

        var textSearch = new VectorStoreTextSearch<Person>(vectorStoreCollection);
        var searchPlugin = textSearch.CreateWithGetTextSearchResults("SearchPlugin");
        kernel.Plugins.Add(searchPlugin);

        KernelSearchResults<TextSearchResult> textResults = await textSearch.GetTextSearchResultsAsync(query, new() { Top = 2, Skip = 0 });
        Console.WriteLine("\n--- Text Search Results ---\n");

        await foreach (TextSearchResult ret in textResults.Results)
        {
            Console.WriteLine($"Name:  {ret.Name}");
            Console.WriteLine($"Value: {ret.Value}");
            Console.WriteLine($"Link:  {ret.Link}");
        }

        // Both prompt works
        // It does not work if you put {{SearchPlugin.Recall input=input collection=collection}}
        //var promptTemplate = @"{{SearchPlugin-GetTextSearchResults query=query}}
        //You are a helpful assistant. Search the memory and answer the question. {{query}}?";
        var promptTemplate = @"You are a helpful assistant. Use SearchPlugin and answer the question. {{query}}?";

        PromptExecutionSettings settings = new()
        {
            FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
        };
        KernelArguments arguments = new(settings)
        {
            ["query"] = query,
            ["collection"] = collectionName
        };
        HandlebarsPromptTemplateFactory promptTemplateFactory = new();

        var result = await kernel.InvokePromptAsync(
            promptTemplate,
            arguments,
            templateFormat: HandlebarsPromptTemplateFactory.HandlebarsTemplateFormat,
            promptTemplateFactory);

        Console.WriteLine("=== InvokePromptAsync ===");
        Console.WriteLine(result);

        Console.ReadLine();
    }
}

class Person
{
    [VectorStoreKey]
    public string Id { get; init; }

    [VectorStoreData]
    [TextSearchResultName]
    public string Name { get; init; }

    [VectorStoreData]
    [TextSearchResultValue]
    public string Value { get; init; }

    [VectorStoreData]
    [TextSearchResultLink]
    public string Link { get; init; }

    // The value of this property will automatically be converted to a vector on upsert.
    //[VectorStoreVector(Dimensions: 1536)]
    [VectorStoreVector(Dimensions: 1536)]
    public string? Embedding => this.Value;
}

#pragma warning restore SKEXP0001, SKEXP0050

```

実行結果は以下の様になります。メモリストア側のデータを参照しているのが分かると思います。
![](/images/semantickernel-dotnet-rag02/image01.png) 

この辺りのベクトルストアの実装周りはプレビューですが、もう大きな breaking change は起きないと予想しています。


## 注意点 1 - text-embedding-ada-002 が動かない？

今回のサンプルで当初は text-embedding-ada-002 が動かず、エラーメッセージ的には REST API が Dimensions（今回は 1536）を指定することに対応していなかったようです。VectorStoreVector(Dimensions: 1536) 属性は引数未指定が未実装だったので、今回は利用モデル自体を text-embedding-3-large に変えています。

## 注意点 2 - TextSearchResultValue, TextSearchResultName, TextSearchResultLink はモデルクラスに一つしか追加できない

上記に記載はしましたが、TextSearchResultValue, TextSearchResultName, TextSearchResultLink の3つの属性はクラス一つに対して各一つしか付与できません。複雑なデータ構造を操作する必要がある場合は以下の記事のカスタムマッパーを利用ください。

## 参考リンク

- [C# Semantic Kernel で簡単 RAG のサンプル ～VolatileMemoryStore編～](https://zenn.dev/microsoft/articles/semantickernel-dotnet-rag01)
- [.NET + Semantic Kernel でベクトル検索の RAG をする](https://zenn.dev/microsoft/articles/semantic-kernel-dotnet-rag01) 
- https://devblogs.microsoft.com/semantic-kernel/vector-data-extensions-are-now-generally-available-ga/
- https://learn.microsoft.com/ja-jp/semantic-kernel/concepts/text-search/text-search-vector-stores?pivots=programming-language-csharp
