---
title: "Microsoft Purview の Data Map 機能を利用する"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["microsoftpurview", "AI", "Security"]
published: true
publication_name: "microsoft"
---

今回は黙々と Microsoft Purview の紹介をしたいと思います。古くは Azure Purview としてリリースされたこともありましたが、今は Microsoft Purview としてデータガバナンス・データセキュリティの根幹を担うソリューションとして発展しました。しかし、発展しすぎたゆえに多機能な挙句にライセンス的にも複雑なこともあり、とっつきにくさも手伝いあまり深く触っている情報が目にしにくいという印象を受けています（エンプラ機能あるある）。

全体像を理解するという意味では以下の「ひと目でわかる（といいな） Purview」や「Tech Briefing :Azure Purview による企業内データガバナンスの実現とその概要」が分かりやすい認識です。
https://www.youtube.com/watch?v=nqWsqO9MS58
https://www.youtube.com/watch?v=ZEAMM9qIn9k

Insider Risk management や Compliance Manager 等の機能を混ぜて解説すると話がややこしくなるので、まずはとっつき部分であるまずはデータガバナンスについてのテーマを深堀したいと思いますが、特にファーストステップである Data Map で Azure Storage をスキャンすることから始めたいと思います。
今回はスキャンのみですが、個人情報が含むデータの発見、自身で作成したルールで特定の情報検出する方法、検出したデータを共有する方法等は今回以外で別途紹介しようと思います。


## Microsoft Purview アカウントの作成

まずは Azure ポータルを開き、以下の様に Microsoft Purview アカウントを作成します。アカウント名とリージョンを選んで作成します。
![](/images/mspurview-gettingstarted-01/purview-article-image01.png) 

以下の記事に記載がありますが、Microsoft Purview アカウントのリージョンは自身の Entra ID のリージョン範囲に限定されます。そのリージョンにメタデータ等が収集されます。
https://learn.microsoft.com/en-us/purview/legacy/create-microsoft-purview-portal

他の画面では VNET 統合や Kafka 統合の設定があるのですが、今回はスキップします。リソース作成後は overview メニューの画面に Microsoft Purview Governance ポータルのリンクがあるので、こちらを選んで Microsoft Purview の操作を開始します。
![](/images/mspurview-gettingstarted-01/purview-article-image02.png) 


## Data Map で Azure Storage を追加する

Microsoft Purview Governance ポータルの画面は以下の様になります。最初に tour 系のメッセージがでるので、必要の場合はそちらも閲覧してください。以下の画面左メニューでも最も使うのは Solution だと思います。こちらをクリックすると今回利用する Data Map も含め、10個程度のメニュー（というより、各項目がソリューションと言えるくらいのボリューム）が表示されます。

Solution -> Data Map を選択するか、トップ画面下部の Data Map メニューを選んで次の操作を行いましょう。
![](/images/mspurview-gettingstarted-01/purview-article-image03.png) 

Data Map の画面で最初に行うことはドメインの作成です。データドメインの作成なしには何のデータも登録できません、以下の画面にしたがって New domain のボタンを押してドメインを作成しましょう。
![](/images/mspurview-gettingstarted-01/purview-article-image04.png) 

次にドメインの Display name に加えてドメイン管理者を Entra ID ユーザから選択します。
![](/images/mspurview-gettingstarted-01/purview-article-image05.png) 

ドメインの作成が完了すると、以下の様にドメイン一覧に追加されます。
![](/images/mspurview-gettingstarted-01/purview-article-image06.png) 

ドメインが作成されたので、次に Data sources メニューを選択することで以下の画面が表示されます。私は既に Cosmos DB や AWS S3 等のリソースを追加済なドメインが存在しますが、作ったばかりのドメインは以下の様に空っぽの状態です。画面の Register ボタンを押してリソースを追加していきましょう。
![](/images/mspurview-gettingstarted-01/purview-article-image07.png) 

登録できるデータソースは以下の様に Azure 以外も当然含まれます。今回は Azure Blob Storage を選択します。
![](/images/mspurview-gettingstarted-01/purview-article-image08.png) 

登録できるデータソースの一覧は以下で参照できます。
https://learn.microsoft.com/en-us/purview/data-map-data-sources


今回は Azure Blob Storage を選択後、以下のメニューが表示されるので、自身の Azure Storage アカウント情報を入力して登録します。
![](/images/mspurview-gettingstarted-01/purview-article-image09.png) 

以下の様にデータソースに Azure Blob Storage が追加されました。
![](/images/mspurview-gettingstarted-01/purview-article-image10.png) 

** Microsoft Purview の Managed Identity に当該 Azure Storage アカウントの Storage Blob Data Reader ロールを追加する
カンの良い Azure ユーザなら気づいたと思いますが、Microsoft Purview アカウントには何の権限も割り振っていないのでこのままの操作では Azure Storage にアクセスできずに UnauthorizedAccess が発生します。
ではどのアカウント・サービスプリンシパル等に権限を追加するのでしょうか？一度  Data sources 画面の Azure Blog Storage リソースにある以下の Scan アイコンをクリックすると Scan 画面が表示されますが、ここの Managed Identity object id を控えておきましょう。
![](/images/mspurview-gettingstarted-01/purview-article-image11.png) 

次に当該 Azure Storage アカウントの IAM メニューから add role assignment を選択し、以下の Storage Blob Data Reader ロールを選択します。
![](/images/mspurview-gettingstarted-01/purview-article-image12.png) 

対象として Managed Identity を選択し、Microsoft Purview アカウントの Managed Identity を選択します。
![](/images/mspurview-gettingstarted-01/purview-article-image13.png) 


## Data Map から Azure Storage アカウントをスキャンする
権限付与は完了しているので、先ほどと同様に以下の data sources 画面から以下のアイコンを押しスキャンを実施します。
![](/images/mspurview-gettingstarted-01/purview-article-image14.png) 

次に以下の画面が表示されますが、特に変更は不要だと思います。
![](/images/mspurview-gettingstarted-01/purview-article-image15.png) 

スキャンする対象が選択可能ですが、今回はそのまますべてを選択します。
![](/images/mspurview-gettingstarted-01/purview-article-image16.png) 

更にスキャンルールを選択します。今回はデフォルトのものを選択しますが、自身のルールを組み込んだルールを利用したスキャンも可能です。
![](/images/mspurview-gettingstarted-01/purview-article-image17.png) 

最後にトリガーを選択します。定期実行も選択可能ですが、今回は once を選択します。次に確認画面が表示されますが、そのまま実行します。
![](/images/mspurview-gettingstarted-01/purview-article-image18.png) 


## スキャン結果を確認する

スキャンが開始されたので、次は以下の画面を参考に Monitoring メニューを表示します。実行結果のサマリが表示されていますが、さらに View details をクリックします。
![](/images/mspurview-gettingstarted-01/purview-article-image19.png) 

以下の様に実行結果の詳細が表示されます。正しく終了していれば Succeeded となっているはずですが、Managed Identity の付与をし忘れた場合は以下の様に Completed with exceptions というエラーになるので注意してください。
![](/images/mspurview-gettingstarted-01/purview-article-image20.png) 

スキャンが正常に終わった後はどの様なデータに分類するかの操作が必要となります。今回は深堀しませんが、以下の様に Unified Catalog で様々な操作が可能であり、データスキャンの結果としていくつかの個人情報がラベル付されているのが分かると思います。
![](/images/mspurview-gettingstarted-01/purview-article-image21.png) 


次回以降では Unified Catalog でのデータ分類、データ操作に加え、データの共有等も紹介していきたいと思います。


## References 
- https://learn.microsoft.com/en-us/purview/data-map-data-sources
- https://learn.microsoft.com/en-us/purview/data-governance-roles-permissions
- https://learn.microsoft.com/en-us/purview/legacy/create-microsoft-purview-portal