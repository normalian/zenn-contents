---
title: "Network Manager を使って Entra ID テナント跨ぎの cross-tenant VNET 接続をする"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Network", "Azure"]
published: true
publication_name: "microsoft"
---

皆さんは Network Managers と呼ばれる機能をご存じでしょうか？Network Managers は VNET 間のネットワーク接続設定やルールを一元的に管理し、Azure Policy 等と組み合わせて中央管理するサービスです。VNET が増えてくると個別に VNET Peering や NSG の設定等を行うのが煩雑になるので、それを中央集権的に管理できるのが主な特徴です。

大半の方は「VNET 一個でもそこそこ使えるし、hub-spoke とかやりたかったら VNET Peering 使うからそれで十分じゃないの？」という感じだと思われます。そんなこんなで私もあんまり触らずにいたのですが、ふと見つけた以下の記事が目を引いたので、特にこちらにフォーカスしつつ Network Manager を紹介したいと思います。
[Azure Virtual Network Manager でのテナント間サポート](https://learn.microsoft.com/ja-jp/azure/virtual-network-manager/concept-cross-tenant) 

念のためですが、VNET Peering 自体も Entra ID cross-tenant 接続をサポートしています。本設定は VNET が少数の場合であれば問題ないのですが、VNET の数が増えた場合はやはり管理しやすさは Network Managers に軍配が上がると思います。
[仮想ネットワーク ピアリングの作成 - Resource Manager、さまざまなサブスクリプション、Microsoft Entra テナント](https://learn.microsoft.com/ja-jp/azure/virtual-network/create-peering-different-subscriptions?tabs=create-peering-portal)

Entra ID のテナントをまたぐので、当然ですがポータル上で Entra ID のテナント切り替えが処理必要になります。また、実施前に以下の点に注意が必要です。
- 接続元 VNET がある Entra ID テナント配下で Network Managers というリソースを作成する
- 接続先 VNET がある Entra ID テナント配下の Virtual Network Managers というメニュー配下の cross-tenant connections を作成する

これをアーキテクチャ図に表すと以下になります。アイコンは同じですが、Network Managers と Virtual Network Managers は異なるメニューです。
![](/images/network-manager-01/image-arch.png) 

Virtual Network Managers というメニュー配下に Network Managers と cross-tenant connections がありますが、 cross-tenant connections からは Virtual Network Managers から参照する必要があるのでこの流れになります。接続元と先では異なるリソースを作成する必要があり、特に接続先のサブスクリプションでは Network Managers リソースを作成する必要がない点に注意が必要です。（むしろ両方のサブスクリプションに作ると接続状態が Conflict とエラーになり接続できません）。

## Network Manager リソースの作成
まずは接続元の Entra ID 配下のサブスクリプションで Network Manager を作ってみましょう。ポータルの同リソース作成メニューから進むと以下の Basic タブが表示されます。今回は NET 同士を接続するために利用するので Connectivity にチェックをします。Network Security Group や User Defined Route 等を中央管理したい場合は他のメニューもチェックしますが、今回はスキップします。
![](/images/network-manager-01/image01.png) 

次の Management Scope タブで作成する Network Manager が管理するスコープを策定します。別 Entra ID テナントと接続したい VNET があるサブスクリプションを選択しましょう。
![](/images/network-manager-01/image02.png) 

この際、すでに作成済の Network Manager が存在する場合は注意が必要です。Management Scope が重複する Network Manager を作成しようとすると作成時に以下の様にエラーとなります。
![](/images/network-manager-01/image03.png) 

## Entra ID cross-tenant での VNET 参照手順
まずは接続元の Entra ID 配下の作成した Network Manager から Cross Tenant connections メニューを選択します。ここから + Create ボタンを選択し、以下の UI で接続先の Tenant ID と Subscription ID を入力してください（ Management group を指定した場合はそちらの ID を入力してください）。
![](/images/network-manager-01/image04.png) 

入力後は Pending のステータスとなりますので確認してください。Conflict 等のメッセージになっている場合は何かがおかしいので注意が必要です。次に Entra ID テナントを接続先に切り替え、接続先の Entra ID 配下の Virtual Network Managers メニュー配下の cross-tenant connections を参照します。
![](/images/network-manager-01/image05.png) 

以下の様に接続元の Entra ID テナント配下の Network Manager 名、リソースグループ名、サブスクリプション名を入力します。
![](/images/network-manager-01/image06.png) 

こちらの設定が完了すると以下の様にステータスが Connected になり接続が完了します。
![](/images/network-manager-01/image07.png) 

ここまでで Cross Tenant connections で必要な予備設定は終了で、これにより作成した Network Manager から別テナントの VNET が参照可能となりました。次に Network Manager を介して異なる VNET 同士を実際に接続していきます。

## Network Manager での接続設定＆設定のデプロイ
まずは Network Group を作成します。以下の様に Network Manager のリソースから Settings -> Network Groups 配下からリソースを作成します。詳細は割愛しますが、まずは名前を指定して空のリソースを作るだけなので特にはまりどころはないと思います。
![](/images/network-manager-01/image08.png) 

次に作成した Network Group 配下に接続元の仮想ネットワークを追加します。Manually add members メニューを選んで VNET を選択します。以下の画像では複数のリージョンをまたいでいますが、Network Manager でも VNET Peering でもリージョンをまたいだ接続は可能です。ただし、Network Manager の場合は後述の設定を確認する必要があります。ここまでは特にはまりどころはないでしょう。
![](/images/network-manager-01/image09.png) 

次に Entra ID テナントを跨いだ VNET を追加します。先ほどと同様に Manually add members を選択しますが、ここに「Tenant」というアイコンがあります。こちらを選択すると自分が選択できる Entra ID テナントが表示されるので、接続先の Entra ID テナントを選択します。
![](/images/network-manager-01/image10.png) 

追加で認証を求められるので、認証を完了すると別テナントの VNET が表示されるので、そちらを選択して追加すると以下の様になります。御覧の通り、別テナントの VNET はリンクになっていないことが分かります。
![](/images/network-manager-01/image11.png) 

これで接続対象の VNET を含めて Network Group に含めることができました。こちらの後は Network Manager 通常の操作と同様です。以下の記事をベースに紹介します。
[Azure Virtual Network Manager を使用してメッシュ ネットワーク トポロジを作成する](https://learn.microsoft.com/ja-jp/azure/virtual-network-manager/how-to-create-mesh-network)

Network Manager の Settings -> Configurations メニューから Create -> Connectivity configuration を選択し、今回は Mesh 構成を選択します。この際にリージョンをまたぐ場合は以下の Enable mesh connectivity across regions のチェックボックスを有効化するのを忘れないようにしてください。
![](/images/network-manager-01/image12.png) 

また、ここまで設定してきましたが、まだ実際の設定は自身の環境に反映されていません。最後の処理として Network Manager の Settings -> Deployments のメニューから構成を対象のリージョンにデプロイします。
![](/images/network-manager-01/image13.png) 

上記の設定がうまくいけばテナント跨ぎの VNET の疎通が可能になっているはずです。
