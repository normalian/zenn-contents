---
title: "Semantic Kernel でバックエンドは Python & フロントエンドは C# を動かしてみる"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["SemanticKernel", "AI", "C#", "Python"]
published: true
publication_name: "microsoft"
---

私のポストでは何度か記載している Semantic Kernel ですが、似た用途で使われる LangChain との大きな差の一つとして C#, Python, Java で SDK を提供している点にあります。ここで試したくなったポイントが「フロントエンドは C# 側の Semantic Kernel で処理し、バックエンド側は Python 側で処理する」というユースケースです。現場での案件を見る限り、様々な形式のドキュメントを抽出・加工したうえで格納するまでの処理は Python で記載している例が多い一方、フロントエンド側は通常の業務アプリとして C# や Java で記載したいというニーズは良く聞きます。したがって今回はこの点について試してみたいと思います。

### 前提条件
以下のリソースは事前に作成してください。
- Azure OpenAI リソースに加え、gpt-4o モデルのデプロイメント、text-embedding-ada-002 モデルのデプロイメント
- Azure AI Search のリソース

### Python で Azure AI Search にデータを格納する
まずは Python 側のソースコードを見てみましょう。以下のように Normalian Co.,Ltd. という架空の会社に関する情報を格納しておきます。collection id はデフォルトでも generic ですが、変えたいことも多いと思いますので、サンプルとして明確に指定しています。Python 側の非

```python
import asyncio

from semantic_kernel.connectors.ai.open_ai.services.azure_text_embedding import AzureTextEmbedding
from semantic_kernel.core_plugins.text_memory_plugin import TextMemoryPlugin
from semantic_kernel.kernel import Kernel
from semantic_kernel.memory.semantic_text_memory import SemanticTextMemory
from semantic_kernel.connectors.memory.azure_cognitive_search import AzureCognitiveSearchMemoryStore

kernel = Kernel()

AZURE_OPENAI_ENDPOINT = "your_azureopenai_endpoint"
AZURE_OPENAI_API_KEY = "your_azureopenai_apikey"
AZURE_SEARCH_ENDPOINT = "your_aisearch_endpoint"
AZURE_SEARCH_API_KEY = "your_aisearch_apikey"

embedding_gen = AzureTextEmbedding(
    deployment_name = "text-embedding-ada-002",
    api_key = AZURE_OPENAI_API_KEY,
    endpoint = AZURE_OPENAI_ENDPOINT
)
kernel.add_service(embedding_gen)
aisearch_memory_store = AzureCognitiveSearchMemoryStore(
    vector_size=1536,
    search_endpoint=AZURE_SEARCH_ENDPOINT,
    admin_key=AZURE_SEARCH_API_KEY
)
memory = SemanticTextMemory(storage=aisearch_memory_store, embeddings_generator=embedding_gen)
kernel.add_plugin(TextMemoryPlugin(memory), "TextMemoryPluginACS")

collection_id = "generic"
async def populate_memory(memory: SemanticTextMemory) -> None:
    # Add some documents to the semantic memory
    await memory.save_information(collection=collection_id, id="qa1", text="Normalian is the most ordinary person in the world.")
    await memory.save_information(collection=collection_id, id="qa2", text="Normalian is CEO of Normalian Co.,Ltd.")
    await memory.save_information(collection=collection_id, id="qa3", text="Normalian Co.,Ltd. has two offices in Tokyo and Seattle.")

async def main():
    await populate_memory(memory)

if __name__ == "__main__":
    asyncio.run(main())
```

ちなみにコードを実行すると以下の様に非同期処理の connection close に失敗している的なエラーが出ていますが、データ自体は無事に登録されています(これの直し方が分かる方が居たら教えて頂けると...)。

```txt
C:\opt\workspace\python_aisearch_app01>python inputdata_to_aisearch.py
Unclosed client session
client_session: <aiohttp.client.ClientSession object at 0x000001C41B7F6060>
Unclosed connector
connections: ['[(<aiohttp.client_proto.ResponseHandler object at 0x000001C41B7DAE10>, 400692.656)]']
connector: <aiohttp.connector.TCPConnector object at 0x000001C41B7F60C0>

```

### C# で Azure AI Search に格納されたデータを利用する

次に Python 側で格納したデータを利用して返答を得てみましょう。サンプルコード自体は以下になります。今回は StepwisePlanner を利用してデータを得ているのですが、最初に HandlebarsPlanner を使った際は "The JSON value could not be converted to System.String. path:$ | LineNumber: 0 | BytePositionInLine: 1." というエラーが出てどうにもならなかったので、内部の非同期処理なりが悪さをしている気がします（未確認）。

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Connectors.AzureAISearch;
using Microsoft.SemanticKernel.Connectors.OpenAI;
using Microsoft.SemanticKernel.Memory;
using Microsoft.SemanticKernel.Planning;
using Microsoft.SemanticKernel.Plugins.Memory;
using System.Text;

Console.OutputEncoding = Encoding.GetEncoding("utf-8");
var configuration = new ConfigurationBuilder()
    .AddUserSecrets("a99df0ee-8e79-4b22-a983-efef33bfb063")
    .Build();

var chat_deployment = configuration["AzureOpenAI:Deployment"];
var aoai_endpoint = configuration["AzureOpenAI:Endpoint"];
var aoai_apiKey = configuration["AzureOpenAI:ApiKey"];
var aisearch_endpoint = configuration["AISearch:Endpoint"];
var aisearch_apiKey = configuration["AISearch:ApiKey"];

#pragma warning disable SKEXP0001, SKEXP0010, SKEXP0020, SKEXP0050, SKEXP0060
var builder = Kernel.CreateBuilder();

using ILoggerFactory loggerFactory = LoggerFactory.Create(builder =>
{
    builder.AddConsole();
    builder.SetMinimumLevel(LogLevel.Trace);
    //builder.SetMinimumLevel(LogLevel.Warning);
});
builder.Services.AddSingleton(loggerFactory);
builder.AddAzureOpenAIChatCompletion(
         chat_deployment,                 // Azure OpenAI Deployment Name
         aoai_endpoint,    // Azure OpenAI Endpoint
         aoai_apiKey);    // Azure OpenAI Key

var memory = new MemoryBuilder()
    .WithLoggerFactory(loggerFactory)
    .WithAzureOpenAITextEmbeddingGeneration("text-embedding-ada-002", aoai_endpoint, aoai_apiKey)
    .WithMemoryStore(new AzureAISearchMemoryStore(aisearch_endpoint, aisearch_apiKey))
    .Build();

builder.Plugins.AddFromObject(new TextMemoryPlugin(memory));

var kernel = builder.Build();

var ask = @"How many offices does Normalian Co.,Ltd. have?";

var options = new FunctionCallingStepwisePlannerOptions()
{
    MaxIterations = 5,
};

var planner = new FunctionCallingStepwisePlanner();
var result = await planner.ExecuteAsync(kernel, ask);

Console.WriteLine("============= The answer ======================="); 
Console.WriteLine(result.FinalAnswer);

#pragma warning restore SKEXP0001, SKEXP0010, SKEXP0020, SKEXP0050, SKEXP0060
```

一方で InvokePromptAsync を利用する場合、同メソッドはテンプレートの変数を展開して OpenAI に送付してくれるだけなので、事前の展開が必要になります。そのため InvokePromptAsync を利用する場合は以下の様にテンプレート内で TextMemoryPlugin を直接呼ぶ必要があります。ご参考まで。

```csharp
string skPrompt = """
ChatBot can have a conversation with you about any topic.
It can give explicit instructions or say 'I don't know' if it does not have an answer.

{{$input}}

## Your knowledge
{{TextMemoryPlugin.Recall $input}}
""";
``` 

上記の C# コードを実行すると以下の様な結果が帰ってきています。

```txt
============= The answer =======================
Normalian Co., Ltd. has two offices, which are located in Tokyo and Seattle.
```

上記のソースコードのうち builder.Plugins.AddFromObject(new TextMemoryPlugin(memory)) をコメントアウト（ Azure AI Search のデータを利用しない）の場合は以下のようになります。無事に RAG ができていることが分かります。

```txt
============= The answer =======================
I don't have the specific information about the number of offices Normalian Co., Ltd. has due to the limitations of my current environment. However, you can find this information by visiting the company's official website and checking their 'Contact Us' or 'About Us' sections. You can also look them up on business directories or contact the company directly for accurate details.
```

## 参考リンク
- [Building Semantic Memory with Embeddings](https://github.com/microsoft/semantic-kernel/blob/main/dotnet/notebooks/06-memory-and-embeddings.ipynb)
- [ベクトル データベースに対する App Service の認証と承認](https://learn.microsoft.com/ja-jp/dotnet/ai/how-to/app-service-db-auth?pivots=azure-portal)
