BigQueryとData Studioで<br>野球の分析レポートを作ろう
===
GTUG Girls #35  
Created by Yui Takeuchi ( [@omohayui](https://twitter.com/omohayui) )

---

# 自己紹介てきな何か

Note:
TODO

---

# BigQuery とは？

- [Google Cloud Platform](https://cloud.google.com/?hl=ja) で提供されるビッグデータ解析用のフルマネージドサービス
- 1PB（ペタバイト）とか10億行といった膨大なデータに対して、集計・分析処理を"極めて"高速に実行できる

Note:
ディスクや仮想マシンなどのリソースを用意する必要ないよ！  
1PB・・・1024TB

---

## なぜ高速なの？
![](https://cloud.google.com/blog/big-data/2016/05/images/146299369373262/BigQuery-shines-1.png)
<small>https://cloudplatform-jp.googleblog.com/2016/05/bigquery-dataproc.html</small>

<small>時間があれば後で ▼</small>

Note:
25分で 10億行をロード  
クエリは 2秒以下

--

### 理由その１：カラム型データストア
![](http://www.publickey1.jp/blog/12/bigquerywp01.jpg)
<small>http://www.publickey1.jp/blog/12/bigquery_1.html</small>  

--

#### カラム型データストアとは
通常のデータベースでは1行ごとにまとめて格納するデータを、列ごとにまとめて格納する方式  
↓
- トラフィックの最小化
- 高い圧縮率

Note:
トラフィックの最小化  
クエリ対象となる列のデータだけにしかアクセスしないため、トラフィックが最小化できる  
高い圧縮率  
同じ列に含まれるデータは類似性が高く、カーディナリティ（いわゆるデータのばらつき）が小さいため

--

### 理由その２：ツリーアーキテクチャ
![](http://www.publickey1.jp/blog/12/bigquerywp02.jpg)

--

#### ツリーアーキテクチャとは
クライアントからクエリを受け取るroot serverから、  
クエリ処理を実行する多数のleaf serversに対して、  
クエリがツリー構造で広がっていくことで、  
大規模分散処理を実現

---

## ハンズオンで使う用語など
![BigQueryまわりの用語](./img/bq_image.jpg)
<small>https://cloud.google.com/bigquery/what-is-bigquery</small>

Note:

プロジェクトはGCPのトップレベルのコンテナ
データセットはテーブルへのアクセス権限を制御できるグループ
テーブルは実際にデータを入れるもの
ジョブはユーザーが作成するアクション

--

|||
|:---|:---|
|プロジェクト|GCP のトップレベルのコンテナ|
|データセット|テーブルへのアクセス権限を<br>制御できるグループ|
|テーブル|実際にデータを入れるもの|
|ジョブ|ユーザーが作成するアクション|

---

## BigQueryの料金体系について
* データのインポート、コピー、エクスポートが無料
* ストレージは毎月 10GB まで無料
* クエリは毎月 1TB まで無料 (超えたら1TBごとに$5)

---

## 運用するとき気をつけること
  * SLECT * は避けよう
  * テーブルを分割しよう
  * 不安なときは dryrun でデータ処理量を確認しよう
  * コストが高くなりそうなクエリにはクエリ単位で上限を設定しよう
  * プロジェクト単位で[カスタム割り当て](https://cloud.google.com/bigquery/cost-controls#controlling_query_costs_using_bigquery_custom_quotas)を設定することも

<small>https://cloud.google.com/bigquery/pricing?hl=ja#free</small>

Note: データを突っ込む分には無料だけど、むやみやたらに重いクエリを投げないように注意。executeする前にdryrunを使ったりするのがおすすめ。
クエリ単位で上限 (Maximum Bytes Billed) を設定したり

---

# Google DataStudio とは
[Google DataStudio](https://datastudio.google.com) は読み込んだデータを分かりやすく可視化してビジネスの判断材料に使ったりできる <strong>無料</strong> のツール

---

## Googleサービスの外部データとの<br>連携がとても簡単
* Adwords
* BigQuery
* CloudSQL
* Googleスプレッドシート
* Googleアナリティクス
* etc...

---

## ただし
Beta版なので色々ともっとこうできたら・・・  
的なものはある

今後の機能拡張に期待・・・！

---

# 今回扱う野球データについて
NPB(日本野球機構)のデータはあれやこれやの問題で使わせてもらえない

個人で頑張って集めている人のデータもありますが  
勝手には使えません。

(パッカソンみたいな公式主催のイベントでは  
使わせて貰えたんだけど）

---

なので・・・

<strong>MLBのデータを使います！</strong>

MLBはオープンライセンスで一般ユーザーも使うことができる

---

### The Lahman Baseball Database
<small>http://www.seanlahman.com/baseball-archive/statistics/</small>
* 情報量は少ないが、csvデータを整形処理なしでそのまま取り込んで使える
  * 初めての野球データ分析に最適！
* 毎年シーズン終了後に更新される
* 今回は最新のデータ (2016年シーズン) のcsvファイルを使う
  * [2016 - comma-delimited version](http://seanlahman.com/files/database/baseballdatabank-2017.1.zip) をダウンロード

