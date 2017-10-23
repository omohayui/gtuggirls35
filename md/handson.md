# つくってみよう

---

## GCPアカウント作成
事前準備で登録してもらってる？

YES ▶  
NO  ▼

--

### 簡易版GCPアカウント作成
<a href="https://gist.github.com/omohayui/457d9f90af7260ebe93d3dd61c3647a5#アカウントの作成" target="_blank">こちらを参照</a>

---

## GCPのプロジェクトを作成する
<small>https://console.cloud.google.com/project</small>

プロジェクト名：「Analytics for MLB」  
プロジェクトID ：「analytics-for-mlb」  
<small>※ プロジェクトID は自動で作成されるが、編集することも可能</small>  
![](./img/handson_01.jpg)

---

## BigQuery API を有効にする
<small>https://console.cloud.google.com/flows/enableapi?apiid=bigquery</small>

* 先ほど作成したプロジェクトを選択して「続行」で有効にする
* 「認証情報に進む」⇒ 今回は不要
<small>※ 別プロジェクトからデータにアクセスする場合は設定</small>

![](./img/handson_02.jpg)

---

## BigQueryにデータをインポートする
サイドメニューから BigQuery へ 
![](./img/handson_03.jpg)

---

### Datasetの作成
<small>https://bigquery.cloud.google.com/queries/analytics-for-mlb</small>

* BigQueryのコンソール上で作成したプロジェクト名の右の三角をクリック

![](./img/handson_04.jpg)

---

* 「Create New Dataset」
  * Dataset ID：mlb_data
  * Data location：US
  * Data expiration：Never

![](./img/handson_05.jpg)

---

### Tableの作成
* 「＋」or「Create New Table」

![](./img/handson_06.jpg)

---

![](./img/handson_07.jpg)

---

* 「Source Data」：Create from Source
* 「Location」：File upload
  * 「Choose file」： <small>[アップロードするファイル]参照</small>
* 「File format」：CSV
* 「Table name」： <small>アップロードするファイル名に合わせる</small>
* 「Schema」：Automatically detect にチェック
* 「Header rows to skip」：1
  * <small>※ CSVファイルの1行目がカラム名としてLoadされる</small>
* 「Create Table」

---

#### 複数テーブル作成時のTips
2つ目移行のテーブルを作成するときは、  
「Repeat job」：Select Previous Job をクリックすると  
前回の設定がセットされるので、  
Choose File と Table Name だけ変更して作成できる

---

#### アップロードするファイル
1. baseballdatabank-2017.1.zip を解凍
2. 解凍したファイル内の core フォルダ内のCSVから
  - Master.csv - 選手のマスタ情報
  - Batting.csv - 各年の打撃成績
  - Pitching.csv - 各年の投手成績
  - Teams.csv - 各年のチーム成績

---

### Recent Jobs
Loadが完了すると 「mlb_data」dataset 下に  
Table が表示される

![](./img/handson_08.jpg)

Masterテーブルを確認してみよう

---

### Table Schema
- 「STRING」や「INTEGER」が「Automatically detect」で選ばれた field の型

![](./img/handson_09.jpg)

---

## SQLについて
- Legacy SQL
  - 従来からあるSQL構文
- Standard SQL
  - [Standard SQLの利点](https://cloud.google.com/bigquery/docs/reference/standard-sql/migrating-from-legacy-sql?hl=ja#advantages_of_standard_sql)

Note:
WITH句でViewで作れたりユーザー定義関数が使えるようになった、一般的なSQL構文で実行できるようになった、DML文が使える。
今から使うならStandardSQLの方が絶対良い。

---

### Legacy SQL で投げてみる
* Masterテーブルを選択した状態で「Query Table」をクリック
  * SELECT の後ろに表示したい fileld名 を入力
    * <small>Tips：Schema の field名をクリックするだけで入力してくれる</small>
  * playerID, nameFirst, nameLast を field に指定して「RUN QUERY」

---

#### Query例
```sql
SELECT playerID, nameFirst, nameLast 
FROM [analytics-for-mlb:mlb_data.Master] LIMIT 1000
```

---

### Standard SQL で投げてみる
1. 「Show options」
2. 「SQL Dialect」：Use Legacy SQL のチェックを外す
  * <small>Tips：SQLの冒頭に "#standardSQL" を付けるだけでもOK</small>
3. Table の書式を変える

---

#### Query例
```sql
#StandardSQL
SELECT playerID, nameFirst, nameLast 
FROM `analytics-for-mlb.mlb_data.Master` LIMIT 1000
```

---

### Batting Table から 2010-2016年毎の<br>「チーム毎のホームラン合計数」の集計データを作ってみよう
* 難易度 ★
* クエリリファレンス
  * https://cloud.google.com/bigquery/query-reference?hl=ja
* 「COMPOSE QUERY」から Query を作成

---

#### Query例
```sql
#standardSQL
SELECT yearID, lgID, teamID, SUM(HR) as totalHR 
FROM `analytics-for-mlb.mlb_data.Batting`
WHERE yearID BETWEEN 2010 AND 2016
GROUP BY yearID, teamID
ORDER BY yearID, lgID, totalHR
```

Note:
分析や機械学習でデータを扱う時 2010〜みたいに古いデータを切り捨てるのはなぜか？
未来の傾向を予測するには新しいデータ重要視すべき、ただデータ量も大事なのである程度古いデータも扱う。
---

### 2010-2016年毎の K/BB が<br>10以上の選手リストを出してみよう
* 難易度 ★★
* 参照するテーブル：Pitching, Master
* [K/BB](https://ja.wikipedia.org/wiki/K/BB) とは？
  * K/BB ＝ 奪三振÷与四球
  * 投手の制球力を示す指標の1つ
  * 3.5を超えると優秀！と言われる

---

#### Query例
```sql
#standardSQL
-- K/BB
SELECT
  yearID,
  CONCAT(nameFirst, " ", nameLast) AS name,
  CASE WHEN BB = 0 THEN 0 ELSE SO/BB END AS kbb 
FROM `analytics-for-mlb.mlb_data.Pitching` AS p
LEFT OUTER JOIN(
  SELECT
    playerID, nameFirst, nameLast
  FROM `analytics-for-mlb.mlb_data.Master` 
) AS m
ON p.playerID = m.playerID
WHERE
  yearID BETWEEN 2010 AND 2016
  AND  CASE WHEN BB = 0 THEN 0 ELSE SO/BB END >= 10
ORDER BY yearID, kbb DESC
```

---

#### CSVでダウンロードしてみよう
* 「Download as CSV」をクリック

集計結果を見るとあの日本人選手が・・・？？

---

### 2008-2016年の「チーム毎の30歳以下の選手のヒット合計数」と「チーム順位」を出してみよう
* 難易度 ★★★
* 参照するテーブル：Batting, Master, Team
  * 選手の生年月日はMasterテーブル
  * チーム名やチームの順位はTeamテーブル
  * チームの順位は地区ごとに分かれていることに注意

Note: アメリカン・リーグ、ナショナル・リーグそれぞれに東地区、中地区、西地区
---

#### Query例
```sql
SELECT
  bm.yearID AS year, t.lgID AS league, t.divID AS division, t.name AS team, bm.totalHit AS totalHitU30, t.rank AS rank
FROM(
  (
    SELECT
      yearID, teamID, SUM(H) AS totalHit 
    FROM `analytics-for-mlb.mlb_data.Batting` as b
    JOIN(
      SELECT
        birthYear, playerID
      FROM `analytics-for-mlb.mlb_data.Master`
    ) AS m
    ON b.playerID = m.playerID
    WHERE
      birthYear >= yearID - 30
    AND
      yearID BETWEEN 2008 AND 2016
    GROUP BY yearID, teamID
  ) AS bm
  LEFT OUTER JOIN(
    SELECT
      name, rank, lgID, divID, teamID, yearID
    FROM `analytics-for-mlb.mlb_data.Teams`
  ) AS t
  ON t.teamID = bm.teamID AND t.yearID = bm.yearID
)
ORDER BY year, league, division, rank
```
* 「Download as CSV」 で結果を確認

---

### Viewを作る
* 先程作った Query 結果から View を作成する
* 「Save View」
* Table ID：total_hit_u30

![](./img/handson_10.jpg)

---

## Google Data Studio
https://datastudio.google.com/
* 同意を求められたら
  * 「同意」：はい
  * 「お知らせ」：はい or いいえ

---

### データソースの追加
* 「新しいデータソースの作成」or「＋」

![](./img/handson_11.jpg)

---

* データソースの名前：TotalHits U30 Group By Team
* 「コネクタ」>「BIgQuery」>「マイプロジェクト」>「Analytics for MLB」>「mlb_data」>「total_hit_u30」（先程作ったView）を選択
* 「接続」
* BigQuery プロジェクトへのアクセス権を求められたら、アカウント選んで 許可

--

![](./img/handson_12.jpg)

---

* フィールドの「タイプ」と「集計方法」を修正

![](./img/handson_13.jpg)

---

* データソース一覧に表示されるのを確認
  * https://datastudio.google.com/navigation/datasources

---

### レポートの作成
* https://datastudio.google.com/navigation/reporting
* レポート一覧から「新しいレポートの開始」or「＋」
* レポートに名前をつける：TotalHits U30 By Team

---

#### データソースを追加
* データソースの選択：TotalHits U30 Group By Team
* 「レポートに追加」
* スコープのリクエストが求められたら：ok

![](./img/handson_14.jpg)

---

#### チーム毎の30歳以下のヒット数のグラフを追加
* グラフの種類「期間」を追加
  * 「期間のプロパティ」
    * データソース：TotalHits U30 Group By Team
    * ディメンション
      * 時間ディメンション：year
      * 内訳ディメンション：team
    * 指標：totalHitU30
    * 内訳ディメンションの並べ替え：totalHitU30, 降順

--

![](./img/handson_15.jpg)

---

#### チーム順位のグラフを追加
* グラフの種類「期間」を追加
  * 「期間のプロパティ」
    * データソース：TotalHits U30 Group By Team
    * ディメンション
      * 時間ディメンション：year
      * 内訳ディメンション：team
    * 指標：rank
    * 内訳ディメンションの並べ替え：rank, 昇順

--

![](./img/handson_16.jpg)

---

### フィルタオプションを追加する
* 「挿入」>「フィルタオプション」
  * league と division それぞれフィルタを追加する

![](./img/handson_17.jpg)

---

#### league フィルタオプションのプロパティ
![](./img/handson_18.jpg)

---

#### division フィルタオプションのプロパティ
![](./img/handson_19.jpg)

---

### ビューでレポートが作れたか確認
* 絞り込んでないのに全チーム表示されない？
  * ディメンションは上位10個までしか表示されない(仕様です)
* [複合グラフ](https://support.google.com/datastudio/answer/7398001?hl=ja#combo-charts-in-data-studio)なら１つのグラフで表示できるのでは？
  * ディメンションが2つのグラフでは指標は1つに制限される(仕様です)
  * つまり１チームごとのグラフなら、ヒット数と順位の指標を１つのグラフで表示できる

---

#### フィルタで絞り込んで AL × 東地区のデータを見てみよう
* ヤンキース
  * 2009〜2012年まで 1位, 2位, 1位, 1位 と好調
  * しかし若手ヒット数は2015年まで減少傾向になった
* オリオールズ
  * 2011年まで最下位が続いていたが2014年には優勝
  * 2011年から若手ヒット数が好調

--

![](./img/handson_20.jpg)

---

#### 何を調べたかったのか
ドラフト制度の違いがデータとして現れるのか

* MLB
  * 前シーズン最下位チームから順に選手を指名していくシステム (ウェーバー方式)
* NPB
  * 1巡目は入札抽選 (重複したらくじ引き！)
  * ２巡目以降はウェーバー方式と逆ウェーバー方式

MLBは最下位チームが優先的に良い選手を獲得できる
順位が変動しやすい
