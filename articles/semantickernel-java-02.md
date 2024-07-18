---
title: "Semantic Kernel で Text-To-Speech と Speech-To-Text を試す"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["SemanticKernel", "AI", "Java"]
published: true
publication_name: "microsoft"
---

前回に引き続いて Java 版の Semantic Kernel を試したいと思います。掲題の通りに Text-To-Speech と Speech-To-Text を試したいと思いますが、それぞれ個別の AI モデルを利用します。具体的には以下となります。
- Text-To-Speech: tts モデルを利用　https://learn.microsoft.com/ja-jp/azure/ai-services/openai/text-to-speech-quickstart?tabs=command-line
- Speech-To-Text: Whisper モデルを利用 https://learn.microsoft.com/ja-jp/azure/ai-services/openai/whisper-quickstart

どちらのモデルも全リージョンで利用できるわけではないので、以下の記事を参考に利用可能なリージョンを選択してください。2024年7月現在では northcentralus と swedencentral が音声系のモデル展開が早いようです。
https://learn.microsoft.com/ja-jp/azure/ai-services/openai/concepts/models#model-summary-table-and-region-availability

今回のソースコードの完全版は以下に配置してあるので、実際に動作させる際は参照してください。
https://github.com/normalian/normalian-sk-java-samples

### Text-To-Speech を試す
では Text-To-Speech の利用例を見てみましょう。動くコードは以下となります。事前に Azure OpenAI のリソースを作成し、tts モデルをデプロイするのを忘れないようにしてください。

```java
package com.normalian.skjavasamples;

import static org.junit.Assert.assertTrue;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.util.UUID;

import org.junit.Test;

import com.azure.ai.openai.OpenAIAsyncClient;
import com.azure.ai.openai.OpenAIClientBuilder;
import com.azure.core.credential.AzureKeyCredential;
import com.azure.core.util.BinaryData;
import com.microsoft.semantickernel.services.audio.AudioContent;
import com.microsoft.semantickernel.services.audio.AudioToTextExecutionSettings;
import com.microsoft.semantickernel.services.audio.AudioToTextService;
import com.microsoft.semantickernel.services.audio.TextToAudioExecutionSettings;
import com.microsoft.semantickernel.services.audio.TextToAudioService;

public class TextAudioTest {
    // @Test
    public void textToAudio01() throws IOException {

        File audioFile = new File("test-" + UUID.randomUUID() + ".mp3");

        String endpoint = System.getenv("AZURE_OPENAI_ENDPOINT");
        String secretKey = System.getenv("AZURE_OPENAI_API_KEY");
        String modelID = System.getenv("AZURE_OPENAI_TTS_DEPLOYMENT_NAME");

        if (endpoint == null)
            throw new NullPointerException("AZURE_OPENAI_ENDPOINT is not defined on env variable");
        if (secretKey == null)
            throw new NullPointerException("AZURE_OPENAI_API_KEY is not defined on env variable");
        if (modelID == null)
            throw new NullPointerException("AZURE_OPENAI_TTS_DEPLOYMENT_NAME is not defined on env variable");

        // Create OpenAIClient instance
        OpenAIAsyncClient client = new OpenAIClientBuilder()
                .endpoint(endpoint)
                .credential(new AzureKeyCredential(secretKey))
                .buildAsyncClient();

        // create TextToSpeech model
        var textToAudioService = TextToAudioService.builder()
                .withModelId(modelID)
                .withOpenAIAsyncClient(client)
                .build();

        String text = "Hi, I'm Daichi. Java is one of the most popular programming languages. Let's enjoy the world of AI with this historic language.";

        TextToAudioExecutionSettings executionSettings = TextToAudioExecutionSettings.builder()
                .withVoice("alloy")
                .withResponseFormat("mp3")
                .withSpeed(1.0)
                .build();

        AudioContent audioContent = textToAudioService.getAudioContentAsync(
                text,
                executionSettings)
                .block();

        // Save audio content to a file
        Files.write(audioFile.toPath(), audioContent.getData());

        // check the file exists
        assertTrue(audioFile.exists());
    }
```

上記でご覧になる通り、TTS 系のモデルを利用して音声ファイルを作成可能です。AudioContent クラス内にバイナリデータが配置されるので、そちらをファイルに書き出せば通常のサウンドプレイヤー等で再生可能な音声ファイルが生成されます。話者として alloy という音声を選んでいますが、全６種類の音声がから選べる（2024年7月現在）ので、必要な場合はパラメータを変えてください。


### Speech-To-Text を試す
では Speech-To-Text の利用例を見てみましょう。動くコードは以下となります。事前に Azure OpenAI のリソースを作成し、whisper モデルをデプロイするのを忘れないようにしてください。

```
package com.normalian.skjavasamples;

import static org.junit.Assert.assertTrue;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.util.UUID;

import org.junit.Test;

import com.azure.ai.openai.OpenAIAsyncClient;
import com.azure.ai.openai.OpenAIClientBuilder;
import com.azure.core.credential.AzureKeyCredential;
import com.azure.core.util.BinaryData;
import com.microsoft.semantickernel.services.audio.AudioContent;
import com.microsoft.semantickernel.services.audio.AudioToTextExecutionSettings;
import com.microsoft.semantickernel.services.audio.AudioToTextService;
import com.microsoft.semantickernel.services.audio.TextToAudioExecutionSettings;
import com.microsoft.semantickernel.services.audio.TextToAudioService;

public class TextAudioTest {
    @Test
    public void audioTotext01() throws IOException {

        File audioFile = new File("test01.mp3");

        String endpoint = System.getenv("AZURE_OPENAI_ENDPOINT");
        String secretKey = System.getenv("AZURE_OPENAI_API_KEY");
        String modelID = System.getenv("AZURE_OPENAI_WHISPER_DEPLOYMENT_NAME");

        if (endpoint == null)
            throw new NullPointerException("AZURE_OPENAI_ENDPOINT is not defined on env variable");
        if (secretKey == null)
            throw new NullPointerException("AZURE_OPENAI_API_KEY is not defined on env variable");
        if (modelID == null)
            throw new NullPointerException("AZURE_OPENAI_WHISPER_DEPLOYMENT_NAME is not defined on env variable");

        // Create OpenAIClient instance
        OpenAIAsyncClient client = new OpenAIClientBuilder()
                .endpoint(endpoint)
                .credential(new AzureKeyCredential(secretKey))
                .buildAsyncClient();

        // create TextToSpeech model
        var textToAudioService = AudioToTextService.builder()
                .withModelId(modelID)
                .withOpenAIAsyncClient(client)
                .build();

        byte[] data = BinaryData.fromFile(audioFile.toPath()).toBytes();

        AudioToTextExecutionSettings executionSettings = AudioToTextExecutionSettings.builder()
                .withLanguage("en")
                .withPrompt("sample prompt")
                .withResponseFormat("json")
                .withTemperature(0.3)
                .withFilename(audioFile.getName())
                .build();

        String text = textToAudioService.getTextContentsAsync(
                AudioContent.builder()
                        .withModelId(modelID) // this setting is mandatory even though this looks duplicated
                        .withData(data)
                        .build(),
                executionSettings).block();

        System.out.println(text);

        // Save audio content to a file
        assertTrue(text != null & text.length() != 0);
    }

```

上記で音声ファイルがテキストになって帰ってきます。なお、音声ファイルは 25MB 制限があるので注意してください。

## 参考リンク
- [クイック スタート: Azure OpenAI Service を使用したテキスト読み上げ](https://learn.microsoft.com/ja-jp/azure/ai-services/openai/text-to-speech-quickstart)
- [クイックスタート: Azure OpenAI の Whisper モデルを使った音声テキスト変換](https://learn.microsoft.com/ja-jp/azure/ai-services/openai/whisper-quickstart)
- [Semantic Kernel for Java](https://github.com/microsoft/semantic-kernel/blob/main/java/README.md)