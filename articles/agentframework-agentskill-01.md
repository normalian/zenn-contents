---
title: Microsoft Agents Framework の Agent Skills 入門 - C# サンプルから学ぶ実装パターン
emoji: "🧠"
type: tech
topics: [dotnet, ai, azure, agent]
published: true
---

## はじめに

Microsoft Agents Framework でエージェントを作り始めると、すぐに出てくる課題が「どうやってドメイン知識を安全に、再利用しやすく渡すか」です。

今回の記事ではサンプルを題材に、Agent Skills を使った知識付与の実装を解説します。  
このサンプルは、次のパターンを 1 つのアプリで比較できるのがポイントです。

1. ファイルベースのスキル
2. コード内で定義するインラインスキル
3. 両者を組み合わせるミックス構成

参考にした記事のように、実装意図と運用目線をセットで読める形にまとめます。

---

## Agent Skills の概要

Agent Skills はエージェントに対して「どの知識を、どんな役割で使わせるか」を明示する仕組みです。  
単に長いプロンプトを渡すのではなく、スキル単位で責務を分けることで、回答品質と保守性を両立しやすくなります。

Microsoft Agents Framework では、主に次の 3 パターンでスキルを扱えます。

1. File-based skills  
SKILL.md と resources 配下のファイルを使って、知識をファイルとして管理する方式です。  
内容の更新をコード変更と分離しやすく、運用中の改善サイクルを回しやすいのが利点です。

2. Code-defined skills  
C# コード内でスキルを定義し、必要に応じて関数や外部 API 呼び出しを組み込む方式です。  
型安全に制御できるため、業務ロジックや動的データ連携を伴うケースに向いています。

3. Class-based skills  
スキルをクラスとして定義して再利用可能にする方式です。  
複数エージェントで同じスキル群を使い回したい場合や、DI・テスト・責務分離を明確にしたい大規模実装で効果を発揮します。

File-based skills で使う SKILL.md には、フロントマターの `name` と `description` が必須です。  
`name` にはスキルを一意に識別できる名前、`description` には「このスキルが何を提供し、どの質問で使うべきか」を簡潔に書きます。  
特に description はモデルのスキル選択に直結するため、対象領域・できること・想定質問を具体的に含めるのが重要です。

このように、静的知識中心なら File-based、実行ロジック重視なら Code-defined、再利用性と構造化を重視するなら Class-based、という基準で選ぶと整理しやすくなります。

今回は 1. File-based skills と 2. Code-defined skills の実装サンプルを例示します。

---

## Agent Skills を利用したサンプル

今回のサンプルは、Seattle 観光と天気情報を題材に、スキルを通じてエージェントの回答品質を高める構成です。

- モデル接続: Azure OpenAI
- 認証: Azure CLI ログイン済みユーザー
- フレームワーク: Microsoft.Agents.AI
- スキル構成: ファイル / インライン / ミックス

プロジェクトの見どころは、同じエージェント用途に対して、スキルの定義方法だけを差し替えながら挙動を比較できる点です。

---

## プロジェクト構成

~~~text
Sample12AgentSkillConsoleApp/
├─ Program.cs
├─ Sample12AgentSkillConsoleApp.csproj
└─ skills/
   ├─ seattle-tourism/
   │  ├─ SKILL.md
   │  └─ resources/
   │     ├─ attractions.md
   │     └─ restaurants.md
   └─ weather-info/
      ├─ SKILL.md
      └─ resources/
         └─ weather-data.md
~~~

csproj では skills 配下を出力ディレクトリへコピーする設定になっており、実行時にスキルファイルを読み込めます。

~~~xml
<ItemGroup>
  <Content Include="skills\**\*">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
</ItemGroup>
~~~

---

## 3 つの実装アプローチ

## 1. ファイルベーススキル

まずは UseFileSkills を使い、skills ディレクトリから SKILL.md を自動発見します。

~~~csharp
var fileSkillsProvider = new AgentSkillsProviderBuilder()
    .UseFileSkills([skillsDirectory])
    .Build();
~~~

この方式の利点は、スキルをコード変更なしで差し替えやすいことです。  
プロンプト設計担当と実装担当を分業しやすく、運用での更新にも向いています。

---

## 2. インラインスキル

次に AgentInlineSkill を使って、スキル定義・スクリプト・静的リソースをコードで組み立てます。

~~~csharp
var seattleTourismSkill = new AgentInlineSkill(
    name: "seattle-tourism",
    description: "Provides detailed tourism information about Seattle including attractions, restaurants, and events.",
    instructions: """
        You are a Seattle tourism expert. When asked about Seattle tourism,
        use the available scripts to look up attractions and restaurants.
        Always provide enthusiastic and helpful recommendations.
        """);

seattleTourismSkill.AddScript("get-attractions", GetSeattleAttractions,
    "Returns a list of popular Seattle attractions for the given category.");
~~~

この方式は、型安全にロジックを管理できるのが強みです。  
たとえば外部 API 呼び出しや社内ルール適用を、通常の C# と同じ感覚で実装できます。

---

## 3. ミックス構成

最後に、ファイルベースとインラインを同時に使います。

~~~csharp
var mixedSkillsProvider = new AgentSkillsProviderBuilder()
    .UseFileSkills([skillsDirectory])
    .UseSkills(transportSkill)
    .Build();
~~~

これは実運用で特に有効です。  
理由は単純で、変化しやすい知識はファイル、実行ロジックはコード、という役割分担がしやすいからです。

---

## SKILL.md の書き方

SKILL.md にはフロントマターで名前と説明を置き、本文にスキルの振る舞いを記述します。

~~~markdown
---
name: seattle-tourism
description: Provides detailed tourism information about Seattle including attractions, restaurants, and local events.
---

You are a Seattle tourism expert...
~~~

この記述をベースに、エージェントは利用可能なコンテキストとしてスキルを解釈します。  
resources 配下の markdown ファイルは、観光地一覧やレストラン情報のような参照知識として使えます。

---

## 実行方法

前提として .NET 9 SDK と Azure CLI ログインが必要です。

~~~powershell
az login
$env:AZURE_OPENAI_ENDPOINT = "https://your-openai-endpoint.openai.azure.com/"
$env:AZURE_OPENAI_DEPLOYMENT_NAME = "gpt-4.1-mini"
~~~

実行は次の通りです。

~~~powershell
cd Sample12AgentSkillConsoleApp
dotnet run
~~~

実行すると、以下の流れで結果が表示されます。

1. ファイルベーススキルで観光地と天気を問い合わせ
2. インラインスキルでレストランと天気を問い合わせ
3. ミックス構成で移動手段と観光を同時に問い合わせ

---

## 実装上のポイント

### 1. スキルの配置解決

Program では、実行パスに skills が見つからない場合、プロジェクト相対パスへフォールバックしています。  
これにより、bin 実行時と IDE 実行時の差分に強くなっています。

### 2. スクリプト関数の説明付与

Description 属性や AddScript の説明文で、LLM が関数を選択しやすくなります。  
関数名だけでなく、用途説明を丁寧に書くのが重要です。

### 3. ビルダーで段階的に合成

AgentSkillsProviderBuilder を軸に、スキルソースを後から追加できるため、段階的な拡張が容易です。  
PoC から本番まで同じ設計思想で育てられます。

---

## 本番運用での注意点

1. スキルの責務を分割する  
1 つの SKILL に何でも詰め込むより、業務単位で分ける方がメンテしやすいです。

2. 動的データはスクリプト経由に寄せる  
固定情報は resources、変動情報は AddScript 側に寄せると鮮度管理しやすくなります。

3. バージョン管理を明確にする  
SKILL.md と resources の更新履歴を追えるようにし、回答変化の原因を追跡できるようにしておきます。

---

## まとめ

Sample12AgentSkillConsoleApp は、Agent Skills の導入パターンを短時間で比較できる良い教材です。

- 手早く始めるならファイルベース
- 厳密に制御するならインライン
- 実運用ではミックス構成が現実的

特に、知識ファイルと実行ロジックを分けて設計できる点は、エージェント品質と保守性を両立するうえで大きな利点です。  
これから Agents Framework を実プロジェクトに入れるなら、まずこの Sample12 の 3 パターンを土台にすると進めやすいと思います。

---

## 参考リンク

- [MyAFSamples - Sample12AgentSkillConsoleApp](https://github.com/normalian/MyAFSamples/blob/main/Sample12AgentSkillConsoleApp/Program.cs)
- [Microsoft Agents Framework - C#](https://github.com/microsoft/agent-framework/tree/main/dotnet)
- [Agent Skills specification - C#](https://learn.microsoft.com/en-us/agent-framework/agents/skills?pivots=programming-language-csharp)
