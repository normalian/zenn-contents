---
title: "Semantic Kernel の Java SDK を試す"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["SemanticKernel", "AI", "Java"]
published: false
---

前回の記事でも記載しましたが、Semantic Kernel は Microsoft が提供する OpenAI/Azure OpenAI/Hugging Face 等の LLM を統合する SDK となります。.NET/Python/Java で提供されているものの、Java 側のノウハウ公開が少ないなと思ったので今回はこちらを記載したいと思います。また、ご存じでない方もいらっしゃると思いますが、実は Java 版の Semantic Kernel は既に GA（一般リリース）されています。
https://devblogs.microsoft.com/semantic-kernel/announcing-the-general-availability-of-semantic-kernel-java/

今回のサンプルコードは以下に配置しているので、必要な場合は参照してください。サンプルには [Maven](https://maven.apache.org/) を利用していますが、その pom.xml を見ればわかるように、semantickernel-connectors-ai-openai, semantickernel-core, semantickernel-plugin-core, semantickernel-connectors-memory-azurecognitivesearch, semantickernel-gpt3-tokenizer といったモジュールは v1 未満で alpha バージョンとなっています。
https://github.com/normalian/normalian-sk-java-samples

最新版の jar バージョンを一通り確認しつつ作業を進めましたが、バージョンを２～３か月ほど古めに間違えて指定した際、クラス参照でエラーが発生したのでかなり頻繁なモジュール更新が入っていることがわかります。クラス参照等で問題があった際はつど jar ファイルを [JD-GUI](https://java-decompiler.github.io/) を使いつつ中身を見てモジュールの整合性を確認していました。

いろんな文章よりは動くコード。早速以下のソースコードで OpenAI を Semantic Kernel 経由でアクセスしてみましょう。事前に Azure OpenAI のリソースを作成し、環境変数に Azure OpenAI のエンドポイント、API Key、Model ID を指定することを忘れないようにしてください。

```java
package com.normalian.skjavasamples;

import static org.junit.Assert.assertTrue;

import org.junit.Test;

import com.azure.ai.openai.OpenAIAsyncClient;
import com.azure.ai.openai.OpenAIClientBuilder;
import com.azure.core.credential.AzureKeyCredential;
import com.microsoft.semantickernel.Kernel;
import com.microsoft.semantickernel.aiservices.openai.chatcompletion.OpenAIChatCompletion;
import com.microsoft.semantickernel.services.chatcompletion.ChatCompletionService;

public class ChatCompletionServiceTest {
    @Test
    public void simpleQA() {
        String endpoint = System.getenv("AZURE_OPENAI_ENDPOINT");
        String secretKey = System.getenv("AZURE_OPENAI_API_KEY");
        String modelID = System.getenv("AZURE_OPENAI_DEPLOYMENT_NAME");

        if (endpoint == null)
            throw new NullPointerException("AZURE_OPENAI_ENDPOINT is not defined on env variable");
        if (secretKey == null)
            throw new NullPointerException("AZURE_OPENAI_API_KEY is not defined on env variable");
        if (modelID == null)
            throw new NullPointerException("AZURE_OPENAI_DEPLOYMENT_NAME is not defined on env variable");

        // Create OpenAIClient instance
        OpenAIAsyncClient client = new OpenAIClientBuilder()
                .endpoint(endpoint)
                .credential(new AzureKeyCredential(secretKey))
                .buildAsyncClient();

        Kernel kernel = Kernel.builder()
                .withAIService(ChatCompletionService.class,
                        OpenAIChatCompletion.builder()
                                .withModelId(modelID)
                                .withOpenAIAsyncClient(client)
                                .build())
                .build();

        String request = "List the benefits of Java Semantic Kernel compared with Langchain.";
        String prompt = """
                Give me answer of following questions.

                =================== Questions ===================
                %s
                """.formatted(request);

        var result = kernel.invokePromptAsync(prompt).block().getResult();
        System.out.println(result);
        var str = result.toString();
        assertTrue(str != null && !str.trim().isEmpty());
    }

}
```

上記も Visual Studio Code 等で実行いただければ無事に返答が帰ってきます。

## 参考リンク
- [Announcing the General Availability of Semantic Kernel for Java](https://devblogs.microsoft.com/semantic-kernel/announcing-the-general-availability-of-semantic-kernel-java/)
- [AI tooling for Java developers with SK](https://devblogs.microsoft.com/semantic-kernel/ai-tooling-for-java-developers-with-sk/)
- [Semantic Kernel for Java](https://github.com/microsoft/semantic-kernel/blob/main/java/README.md)