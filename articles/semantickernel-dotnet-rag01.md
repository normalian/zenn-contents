---
title: "C# Semantic Kernel で簡単 RAG のサンプル ～VolatileMemoryStore編～"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["SemanticKernel", "AI", "csharp"]
published: true
publication_name: "microsoft"
---

皆様 Semantic Kernel を使ってますでしょうか？色んな所で話題は出るのですが、昨今というより OpenAI が出てからずっと流行りの RAG の簡単なサンプルはあまりころがっていないので、方々色々と確認したものをここに脳内ダンプしたいと思います。というのが preview や obstacle の機能が入り混じっており、いったいどう実装したら良いのか迷うのが多いというのが現状だと思っています（2025年5月現在）。
なので、本記事に関してはこれがベストプラクティスという様な考え方はせず「こういう風に動いた」という参考例として考えてもらえればと思います。

## 今回の環境

今回は C# で Semantic Kernel を触りましたが、利用したライブラリは以下となります。バージョンに関しては Connector は preview であり、Memory PlugIn に関しては alpha となっているので、中々に breaking change が怖いのはお察し頂けると思います。
- Microsoft.SemanticKernel: 1.51.0
- Microsoft.SemanticKernel.Connectors.AzureOpenAI: 1.51.0
- Microsoft.SemanticKernel.Connectors.InMemory: 1.51.0-preview
- Microsoft.SemanticKernel.Plugins.Memory: 1.51.0-alpha
- Microsoft.SemanticKernel.PromptTemplates.Handlebars: 1.51.0

## ベクトルストアの既存ドキュメントや参考情報を確認

そもそもの公式の GitHub を見ていると以下にサンプルが転がっていますが、RAG と言いつつ最初の Step1/Step2 に関しては Bing や Google サーチを対象としており、社畜御用達の社内ドキュメントをベクトル化したサンプルがイマイチ分かりにくい感じです。
[GitHub: semantic-kernel/dotnet/samples/GettingStartedWithTextSearch](https://github.com/microsoft/semantic-kernel/tree/main/dotnet/samples/GettingStartedWithTextSearch)

特に以下の Step4 に関してはベクトルストアを検索していますが、プロンプトで明確に Function Calling しており「あれ、 Function Calling の明記って必須だったっけ？」と頭を傾けてしまいました。
[GitHub: semantic-kernel/dotnet/samples/GettingStartedWithTextSearch/Step4_Search_With_VectorStore.cs](https://github.com/microsoft/semantic-kernel/blob/main/dotnet/samples/GettingStartedWithTextSearch/Step4_Search_With_VectorStore.cs)

次に GitHub ではない MS Learn 側のドキュメントを確認しましたが、こちらも「preview だから注意しろ」に加えてベクトルストアのクラスに対して直接検索をかけています。
[セマンティック カーネル ベクター ストア コネクタとは (プレビュー)](https://learn.microsoft.com/ja-jp/semantic-kernel/concepts/vector-store-connectors/)

念のためベクトル検索自身のページを参照してもやり方は一緒でした。
[セマンティック カーネル ベクター ストア コネクタを使用したベクター検索 (プレビュー)](https://learn.microsoft.com/ja-jp/semantic-kernel/concepts/vector-store-connectors/vector-search)

上記から分かったことはベクトルストアのクラスを Semantic Kernel の Kernel にコンポーネントとして登録し、プロンプトて function calling なりを利用して呼び出す必要がありそうということです。

## Semantic Kernel のコンポーネントに登録
上記で「Semantic Kernel の Kernel にコンポーネントとして登録」という記載をしましたが、ここにちょっと罠があります。以下に記載がありますが「実装がカーネルに登録されると、既定では、カーネルへのメソッド呼び出しによって、チャット完了またはテキスト生成サービスが使用されます。 サポートされている他のサービスは自動的に使用されません。」という点です。
[セマンティック カーネル コンポーネント](https://learn.microsoft.com/ja-jp/semantic-kernel/concepts/semantic-kernel-components)

すなわち、単に Kernel にコンポーネントを登録するだけだと自動では呼び出しをしないと言っています。これを飛び出させるには以下の実装を組み込む必要があります。こちらは以下にも記載があります。
[セマンティック カーネルの概要 - 9 計画](https://learn.microsoft.com/ja-jp/semantic-kernel/get-started/quick-start-guide?pivots=programming-language-csharp#9-planning)


```csharp
OpenAIPromptExecutionSettings openAIPromptExecutionSettings = new()
{
    FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
};
```

では上記を踏まえて実装をしてみましょう。

## Semantic Kernel のソースコードを実行

上記を踏まえ、Visual Studio を立ち上げて以下の C# のソースコード を作成してください。

```csharp

using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Connectors.AzureOpenAI;
using Microsoft.SemanticKernel.Memory;
using Microsoft.SemanticKernel.Plugins.Memory;
using Microsoft.SemanticKernel.PromptTemplates.Handlebars;
using Microsoft.SemanticKernel.Text;
using System;
using System.Threading.Tasks;

#pragma warning disable SKEXP0001, SKEXP0050
class Program
{
    static async Task Main()
    {
        string endpoint = "https://your-azureopenai-endpoint.openai.azure.com/";
        string apikey = "YOUR-AzureOpenAI-APIKEY";
        string collectionName = "sample-collection";
        string chatModelName = "gpt-4o";
        string embeddingModelName = "text-embedding-ada-002";

        string personName = "Kazuki";

        // initiate Semantic Kernel 
        var kernelBuilder = Kernel.CreateBuilder();
        kernelBuilder.AddAzureOpenAIChatCompletion(
            deploymentName: chatModelName,
            endpoint: endpoint,
            apiKey: apikey
        );
        var kernel = kernelBuilder.Build();
        var memoryStore = new VolatileMemoryStore();

        // I found "AzureOpenAITextEmbeddingGenerationService is obsolete: Use AzureOpenAIEmbeddingGenerator instead." error message, but I can't find AzureOpenAIEmbeddingGenerator class from assemblies
        var memory = new MemoryBuilder()
            .WithMemoryStore(memoryStore)
            .WithTextEmbeddingGeneration(new AzureOpenAITextEmbeddingGenerationService(
                embeddingModelName,
                endpoint,
                apikey))
            .Build();
        await memory.SaveInformationAsync(collectionName, id: "1", text: "Daichi Isami is based in Redmond, Washington. In his free time, Daichi enjoy traveling and exploring local food and drinks. Daichi have a fun hobby of playing chess.");
        await memory.SaveInformationAsync(collectionName, id: "2", text: "Kazuki Sato is based in Tokyo, Japan. He is great C# developer and works as Cloud Solution Architect in BigSoft. He was a former BigSoft MVP.");
        var textSearch = new TextMemoryPlugin(memory);
        kernel.ImportPluginFromObject(textSearch, "SearchPlugin");

        // It does not work if you put {{SearchPlugin.Recall input=input collection=collection}}
        //var promptTemplate = @"{{SearchPlugin-Recall input=input collection=collection}}
        //You are a helpful assistant. Search the memory and answer the question. Who is {{input}}?";

        var promptTemplate = @"You are a helpful assistant. Search the memory in {{collection}} and answer the question. Who is {{input}}?";

        // This prompt does not call SearchPlugin
        //var promptTemplate = @"You are a helpful assistant. Search the memory and answer the question. Who is {{input}}?";

        PromptExecutionSettings settings = new()
        {
            FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
        };
        KernelArguments arguments = new(settings)
        {
            ["input"] = personName,
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
#pragma warning restore SKEXP0001, SKEXP0050

```

実行結果は以下の様になります。メモリストア側のデータを参照しているのが分かると思います。
![](/images/semantickernel-dotnet-rag01/image01.png) 

この辺りのベクトルデータ参照周りはまだまだ過渡期なので、バージョンアップ時の breaking change に注意しながらご使用下さい。


## 注意点 1 - プロンプトの構文

今回のサンプルでは結局使わなくなりましたが、以下の様に "プラグイン名-関数名" で呼び出します。私はしばらく間違えて "プラグイン名.関数名" で記載していたので、ベクトルストア側にデータを読み込みに行かずにハマってしまいました。

```csharp
        var promptTemplate = @"{{SearchPlugin-Recall input=input collection=collection}}
        You are a helpful assistant. Search the memory and answer the question. Who is {{input}}?";
```

## 注意点 2 - FunctionChoiceBehavior.Auto を設定しない

注意点 1 の様に明示的にプロンプト内に Function Calling を設定すれば問題ありませんが、上記の例の様に指定しない場合は上記でも記載はありますが Semantic Kernel は他のサービスは自動的に使用しません。そのためには以下のコードを入れる必要がある点に注意してください。

```csharp
OpenAIPromptExecutionSettings openAIPromptExecutionSettings = new()
{
    FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
};
```

## 参考リンク
- [GitHub: semantic-kernel/dotnet/samples/GettingStartedWithTextSearch](https://github.com/microsoft/semantic-kernel/tree/main/dotnet/samples/GettingStartedWithTextSearch)
- [GitHub: semantic-kernel/dotnet/samples/GettingStartedWithTextSearch/Step4_Search_With_VectorStore.cs](https://github.com/microsoft/semantic-kernel/blob/main/dotnet/samples/GettingStartedWithTextSearch/Step4_Search_With_VectorStore.cs)
- [セマンティック カーネル ベクター ストア コネクタとは (プレビュー)](https://learn.microsoft.com/ja-jp/semantic-kernel/concepts/vector-store-connectors/)
- [セマンティック カーネル ベクター ストア コネクタを使用したベクター検索 (プレビュー)](https://learn.microsoft.com/ja-jp/semantic-kernel/concepts/vector-store-connectors/vector-search)
- [セマンティック カーネル コンポーネント](https://learn.microsoft.com/ja-jp/semantic-kernel/concepts/semantic-kernel-components)
- [セマンティック カーネルの概要 - 9 計画](https://learn.microsoft.com/ja-jp/semantic-kernel/get-started/quick-start-guide?pivots=programming-language-csharp#9-planning)
