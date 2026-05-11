---
lab:
  index: 7
  title: Unity Catalog にデータを取り込む
  module: Ingest data into Unity Catalog
  module-url: 'https://learn.microsoft.com/training/wwl-databricks/ingest-data-into-unity-catalog/'
  notebook: 'https://github.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/blob/main/Allfiles/07-ingest-data-into-unity-catalog.ipynb'
  description: このラボでは、Azure Databricks で使用できるコア データ インジェスト手法を練習します。 PySpark DataFrames、SQL COPY INTO、CREATE TABLE AS SELECT を使って、Unity Catalog の管理ボリュームから Delta テーブルに CSV ファイルを読み込みます。 また、クラウド ストレージから新しいファイルを自動的に検出して処理するように自動ローダーを構成し、継続的に届くデータの厳密に 1 回のインジェストを実際に行ってみます。
  duration: 45 minutes
  level: 300
  islab: true
  primarytopics:
    - Azure Databricks
---

# ラボ 07: Unity Catalog にデータを取り込む

## シナリオ

あなたは、スペインとドイツでの太陽光発電所の運営とオランダとデンマークでの風力タービンの設置を行っている再生可能エネルギー企業である **Solaris Energy** のデータ エンジニアです。 チームは、エネルギー生成の測定値、タービン メンテナンス イベント、すべてのサイトからのグリッド テレメトリを統合するために、Azure Databricks で集中型データ プラットフォームを構築しています。

このラボでは、Azure Databricks で使用できるコア データ インジェスト手法を練習します。PySpark DataFrames でバッチ インジェストを行い、**COPY INTO** で SQL ベースのファイルを読み込み、**CTAS** で集計テーブルを作成し、**自動ローダー**で継続的にファイルを検出します。

このラボを終えると、次のことができるようになります。

- 取り込まれたデータを格納するための Unity Catalog 階層 (カタログ、スキーマ、ボリューム) を作成する
- PySpark DataFrames を使って管理ボリュームから Delta テーブルに CSV データを読み込む
- COPY INTO と組み込みの重複除去を使ってファイルを増分的に読み込む
- CREATE TABLE AS SELECT を使って集計テーブルを作成する
- クラウド ストレージから新しいファイルを自動的に検出して処理するように自動ローダーを構成する

---

## 🤖 Genie Code - このラボ全体で使用してください。

すべての演習において、**Genie Code** を使用することが想定されており、そうすることをおすすめします。 Genie Code は、ノートブック セルのツール バーで直接使用できます。 これは次の目的で使用されます。

- SQL と PySpark の操作に関する構文を検索する
- エラー メッセージを説明する
- 定型コードを生成して適合させる
- 機能のしくみに関するフォローアップの質問をする

Genie Code を開くには、次を選択します ![アシスタント アイコン](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/media/genie-code.svg) をノートブック セルの右側で選択するか、キーボード ショートカットを使用します。

💡 **演習の各セルには、Genie Code に直接コピーして開始できる、推奨プロンプトが含まれています。**

---
- 'An **Azure Databricks Premium workspace** provisioned using [Lab 00': 'Set up your Azure Databricks environment](00-setup.md).'
- 'You have the **Data Engineer** or equivalent role in the workspace, with permission to create catalogs and volumes'
- Basic familiarity with SQL and Python
---

## ノートブック以外の探索: Lakeflow Connect (省略可能)

**Lakeflow Connect** では、カスタム コードを記述することなく、SQL Server、Salesforce、SharePoint などの外部ソースから Unity Catalog のテーブルにデータを直接取り込むための、グラフィカルでロー コードのアプローチが提供されます。

ワークスペースで Lakeflow Connect を調べるには:

1. Databricks ワークスペースのサイド バーで、**[Data Engineering]** → **[データ インジェスト]** をクリックします。
2. 使用できるコネクタを参照します。 カテゴリに注意してください: データベース コネクタ、SaaS コネクタ、ファイルベースのインジェスト。
3. **[SQL Server]** をクリックして、接続を構成し、取り込むテーブルを選んで、格納先のカタログとスキーマを設定する方法を確認します。
4. **SCD Type 1** (上書き) と **SCD Type 2** (履歴追跡)、および**完全更新**と**増分**抽出に関するオプションを確認します。

> 💡 このラボの一部として、Lakeflow Connect の設定を完了する必要はありません。構成済みの SQL Server ソースは提供されていません。 上記の探索は、知識を得るためだけのものです。

---

## ノートブック以外の探索: Lakeflow Spark Declarative Pipelines (省略可能)

**Lakeflow Spark Declarative Pipelines** (旧称 Delta Live Tables) は、自動オーケストレーション、スキーマ管理、厳密に 1 回の保証を備えた、運用グレードのインジェスト パイプラインを構築するための、おすすめする方法です。

パイプライン エディターを調べるには:

1. サイド バーで、**[Data Engineering]** → **[パイプライン]** をクリックします。
2. **[パイプラインの作成]** をクリックして、**[ETL パイプライン]** を選びます。
3. ソース ノートブックまたは SQL ファイルの指定、パイプラインのカタログとスキーマの名前の設定、クラスターの種類の選択に関するオプションを確認します。
4. **[キャンセル]** をクリックします。パイプラインを作成または実行する必要はありません。

> 💡 **Auto CDC API** (create_auto_cdc_flow/AUTO CDC INTO) は、Lakeflow 宣言パイプライン内で変更データ キャプチャ (CDC) のフィードを処理するための、おすすめする方法です。 重複除去、順不同のイベント、SCD Type 1 または Type 2 のパターンが、自動的に処理されます。 パイプライン ノートブックまたは SQL ファイルで定義した後、Pipelines UI からそれを実行します。

---

## ノートブックのインポート

ラボ ノートブックを Databricks ワークスペースにインポートするには、次の手順のようにします。

1. Databricks ワークスペースで、左サイド バーの **[ワークスペース]** をクリックします。
2. ラボを格納するフォルダー (Users/<自分のメール アドレス>/Labs など) に移動するか、作成します。
3. **⋮** (ケバブ) メニューをクリックするか、フォルダーを右クリックして、**[インポート]** を選択します。
4. **[URL]** を選択し、次の URL を入力して **[インポート]** をクリックします。`https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/07-ingest-data-into-unity-catalog.ipynb`
5. インポートしたノートブックを開き、上部のコンピューティング セレクターで **[サーバーレス]** コンピューティングを選択します。

---

## ラボの概要

ノートブックは、次の 4 つの演習で構成されています。

| 演習 | トピック | 手法 |
|---|---|---|
| 1 | カタログ階層を設定する | SQL DDL — CREATE CATALOG、CREATE SCHEMA、CREATE VOLUME |
| 2 | DataFrames を使用したバッチ インジェスト | PySpark spark.read/df.write |
| 3 | SQL ベースのファイル インジェスト | COPY INTO、CREATE TABLE AS SELECT |
| 4 | 自動ローダー | spark.readStream を使用した cloudFiles 形式 |

演習を順番に行います。 各演習は、前の演習で作成したカタログとデータに基づいています。
