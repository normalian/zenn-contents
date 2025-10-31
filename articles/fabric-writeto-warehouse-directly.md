---
title: "Microsoft Fabric 上で Notebook から直接 Warehouse とやり取りする"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["microsoftfabric"]
published: false
published: true
publication_name: "microsoft"
---

皆様 Microsoft Fabric を利用されていますでしょうか？Microsoft がデータプラットフォームとして強く推しているサービスで、Power BI を筆頭にありとあらゆるサービスがどんどん参加に入って急成長しています。Power BI 以外にも Data Factory はもちろん、RDB 的なデータを格納する Data Warehouse 機能、Python Notebook が使える Data Science 機能なども提供されています。
非常に重厚長大かつ強力な機能を提供している Microsoft Fabric ですが、OneLake という機能を使用してデータ ストレージを一元化しようとしているところに肝があります。

ここでちょくちょく聞かれるのが「Python Notebook から Data Warehouse のデータを直接処理したい」という要望です。言われてみれば当たり前の機能ですが、上記の「OneLake という機能を使用してデータ ストレージを一元化しようとしている」で課題が発生します。
Notebook は以下の様に Lakehouse をアタッチするのですが、Lakehouse とは OneLake上に構築される論理的なデータ管理・分析プラットフォームであり、Warehouse を直接アタッチすることはできません。
![](/images/fabric-writeto-warehouse-directly/fabric-writeto-warehouse-directly-01.png) 

つまり Warehouse に対するデータ処理をする場合は以下の様に Data Pipeline 等を用いてやり取りする流れになります。
- Notebook <--> Lakehouse <--> Data Pipeline 等 <--> Warehouse

## Notebook から直接 Warehouse のデータを読み込む
話がこれで終わると面白くないので、書き込みはさておき読み込みは逃げ道があったので今回はこちらを紹介しようと思います。その答えは以下の記事にあります。
- [Fabric Python Notebook を使用して接続する](https://learn.microsoft.com/ja-jp/fabric/data-warehouse/how-to-connect#connect-using-fabric-python-notebook)

上記の記事に記載がありますが、T-SQL 実行セル上で -bind オプションを利用することで python 変数に SQL 文の実行結果が格納可能なことが分かります。以下に "warehouse01" という名前の Warehouse 上に dbo.table1 というテーブルがあることを想定したクエリ例を記載しています。

```sql
# T-SQL を実行し -bind で Python 変数の格納
%%tsql -artifact warehouse01 -type Warehouse -bind _sqldf
SELECT * FROM dbo.table1
```

上記のクエリで _sqldf 変数に実行結果が格納されます。pandas DataFrame 形式で格納されるので、以下の様に実行することで表示可能です。

```python
# Python セルで変数を表示
print(_sqldf.head())
```

以下にスクリーンショット例を記載します。
![](/images/fabric-writeto-warehouse-directly/fabric-writeto-warehouse-directly-02.png) 
