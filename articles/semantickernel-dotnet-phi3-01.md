---
title: "C# Semantic Kernel と Ollama を使って Phi-3 を自端末で動かす"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["SemanticKernel", "AI", "csharp"]
published: true
publication_name: "microsoft"
---

※注 こちらの記事は [Run Phi-3 SLM on your machine with C# Semantic Kernel and Ollama](https://laurentkempe.com/2024/05/01/run-phi-3-slm-on-your-machine-with-csharp-semantic-kernel-and-ollama/) の内容を踏襲したものになります。既にご存じのかたも多いと思いますが、2024 年に Microsoft は Small-Language-Model(SLM) として [Phi-3](https://azure.microsoft.com/ja-jp/products/phi-3) を発表しています。
GPT シリーズに代表される Large-Language-Model(LLM) が台頭しましたが、すべての問題に単一のモデルに対応するのも限界が見えている気がしています。ユーザの間口は個別の問題は LLM で受けて個別の問題に分割し、それぞれを fine-tuning した SLM で解決するようなマルチモデルのエージェント方式がはやるのかなーとか思っていたりします（個人の主観です）。

さて、そんな中で Phi-3 を今更動かしてもしょうがないので「Semantic Kernel から呼べないかな～」とか思っていたら冒頭の記事を発見したのでためてみました。特筆すべきポイントはすべて自端末で完結することです。[Ollama](https://www.ollama.com/) を利用することで SLM/LLM を自端末で動かすことができ、OpenAI の Chat Completions API と互換機能を持つので容易に自身のコードに組み込むことが可能です。

## 前提条件
C# を利用して Semantic Kernel のアプリケーションを動かす前に Ollama をダウンロードし、Ollama 上に Phi-3 のモデルをダウンロードする必要があります。[Ollama - download](https://www.ollama.com/download) からインストールファイルを取得して設定してください。Windows であればインストーラを実行するだけで完了するはずです。インストール後、コマンドプロンプトを立ち上げて以下のコマンドを実行することで Phi-3 モデルが自端末上で実行されます。

```txt
ollama pull phi3:latest
```

コマンドを実行する際、2GB ほどのデータをダウンロードするので帯域に注意してください。
![](/images/semantickernel-dotnet-phi3-01/image01.png) 

## Semantic Kernel のソースコードを実行

Visual Studio を立ち上げ、以下の C# ソースコード を作成してください。以下に記載のある localhost のエンドポイントは Ollama が自端末で実行されているので、そちらにアクセスしています。Ollama 内ではデプロイ済の Phi-3 が公開されているので、モデルを指定してアクセスしています。


```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
using System.Text;

Console.WriteLine("Hello, World!");

#pragma warning disable SKEXP0010
var kernel = Kernel.CreateBuilder()
    .AddOpenAIChatCompletion(
        modelId: "phi3", 
        apiKey: null, 
        endpoint: new Uri("http://localhost:11434")).Build();

var chat = kernel.GetRequiredService<IChatCompletionService>();
ChatHistory chatHistory = new("You are an AI assistant to talk about Microsoft Azure history.");
var builder = new StringBuilder();

while (true)
{
    Console.Write("Questios: ");
    chatHistory.AddUserMessage(Console.ReadLine()!);
    builder.Clear();

    await foreach (var message in chat.GetStreamingChatMessageContentsAsync(chatHistory, new PromptExecutionSettings()))
    {
        Console.Write(message);
        builder.Append(message.Content);
    }
    Console.WriteLine();
    chatHistory.AddAssistantMessage(builder.ToString());

    Console.WriteLine();   
}
```

実行結果は以下の様になります。13th Gen Intel(R) Core(TM) i7-1365U 1.80 GHz, RAM 16.0 GB の ThinkPad では動きがやや重く、以下の様な簡単な質問に対する最初の一単語を返すのにも１分弱かかっています。
![](/images/semantickernel-dotnet-phi3-01/image02.png) 

また、IChatCompletionService.GetStreamingChatMessageContentsAsync メソッドは単語単語で都度返してくれるにも拘わらず上記の時間がかかりましたが、IChatCompletionService.GetChatMessageContentAsync メソッドを利用した際は 100 秒上限に引っかかってタイムアウトとなりました。この辺りはハードウェア側の問題になると思いますが、考慮が必要になる点だと思います。

## 参考リンク
- [Ollama](https://www.ollama.com/)
- [Ollama GitHub](https://github.com/ollama/ollama)
- [Run Phi-3 SLM on your machine with C# Semantic Kernel and Ollama](https://laurentkempe.com/2024/05/01/run-phi-3-slm-on-your-machine-with-csharp-semantic-kernel-and-ollama/)
