---
lab:
  index: 1
  title: Azure Databricks を探索する
  module: Explore Azure Databricks
  module-url: 'https://learn.microsoft.com/training/modules/explore-azure-databricks/'
  notebook: 'https://github.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/blob/main/Allfiles/01-explore-azure-databricks.ipynb'
  description: このラボでは、Azure Databricks ワークスペースの UI を調べて、サンプル データセットを Unity Catalog ボリュームにアップロードし、Python、SQL マジック コマンド、Markdown などのノートブック機能を操作します。 全体を通して Databricks アシスタントを使用し、架空の公共交通機関である CityMoves Transit のコンテキストでコードの生成と調整を行います。
  duration: 30 minutes
  level: 200
  islab: true
  primarytopics:
    - Azure Databricks
---

# ラボ 01: Azure Databricks を調べる

このラボでは、大都市圏全体でバス、トラム、列車のサービスを管理する地域の公共交通機関である **CityMoves Transit** のコンテキストで、Azure Databricks に関する最初の手順を行います。 新しくオンボードされたデータ エンジニアであるあなたの目標は、後続のラボでデータ パイプラインと分析について詳しく学ぶ前に、Azure Databricks ワークスペースを理解することです。

このラボを終えるまでに、次のことを行います。

- Azure Databricks ワークスペースの UI の主要な領域を見て回ります。
- データ インジェスト インターフェイスを使ってサンプル データセットをアップロードします。
- Python コード セル、SQL マジック コマンド、Markdown ドキュメントなどのノートブック機能を調べます。

このラボは完了するまで、約 **45** 分かかります。

---
'Before starting this lab, ensure you have':
  - Access to an **Azure Databricks Premium workspace** (already provisioned for you).
  - Familiarity with basic Azure portal navigation.
  - No prior Databricks experience is required.
---

## 演習 1: Azure Databricks ワークスペースの UI を見て回る

コードを記述する前に、Azure Databricks の環境を調べてみましょう。 UI に慣れておくと、このコース全体の作業をいっそう効率よく行うのに役立ちます。

### タスク 1: ワークスペースのサイド バーを調べる

1. Azure Databricks ワークスペースで、**左側のサイド バー**に注目します。 次のセクションを確認してください。
   - **ワークスペース**: ノートブックとファイルの個人用フォルダーと共有フォルダー。
   - **最近使用したもの**: 最近開いたオブジェクト。
   - **カタログ**: Unity Catalog のデータ資産 (テーブル、ボリューム、スキーマ)。
   - **ジョブとパイプライン**: 自動化されたワークフロー。
   - **コンピューティング**: クラスターと SQL ウェアハウスの管理。
   - **マーケットプレース**: パートナーのデータとソリューション。

2. **[+ 新規]** (サイド バーの上部) をクリックし、作成できるオブジェクトの種類 (ノートブック、クエリ、クラスター、ダッシュボードなど) を確認します。 **まだ何も作成しないでください**。ここでは調べるだけです。

3. 上部の **[検索]** バーを使って `routes` を検索します。 まだ何も表示されませんが、後でインジェストが済んだらこれを使ってデータ資産を検索します。

### タスク 2: Databricks アシスタントを調べる

**Databricks アシスタント**は、Azure Databricks に直接組み込まれている、AI を搭載したペアのプログラマーです。 コードの生成、エラーの説明、改善点の提案、質問への回答のすべてを、ユーザー インターフェイスを離れることなく行えます。 このラボと移行のすべてのラボを通して、それを使うことが想定されており、そうすることをお勧めします。

1. Azure Databricks のホーム ページで、ページの右上隅にある **[Databricks アシスタント]** アイコン (![assistant-icon](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/media/databricks-assistant.svg)) をクリックして、アシスタント パネルを開きます。

2. 次のプロンプトを入力し、応答を確認します。

    ```
    What can I do with Azure Databricks as a data engineer?
    ```

3. 回答を確認します。 アシスタントがコンテキストに応じたワークスペース対応のガイダンスをどのように提供するのかに注目してください。

> 💡 **これ以降、コードまたは SQL の記述を求められたら常に、Databricks アシスタントをお使いください。何が必要かを平易な言葉で説明し、提案を調整して実行します。**

### タスク 3: サンプルの輸送データセットをアップロードする

CityMoves Transit では、ルート情報が CSV ファイルで提供されています。 あなたのタスクは、後のラボで使えるように、データ インジェスト UI を使ってそれをワークスペースにアップロードすることです。

アップロードする前に、ファイルを格納するための Unity Catalog **ボリューム**が必要です。 次の手順に従って作成します。

1. Databricks ワークスペースのサイド バーで、**[カタログ]** をクリックします。
2. カタログ エクスプローラーで、**adb-dp750** カタログを展開してから、**default** スキーマを展開します。
3. **default** の横にある **⋮** メニューをクリックした後、**[作成]** > **[ボリューム]** を選びます。
4. ボリューム名に「`lab_data`」と入力し、種類は **[マネージド]** のままにして、**[作成]** をクリックします。

次に、データ ファイルをアップロードします。

5. Databricks ワークスペースのサイド バーで、**[+ 新規]** をクリックして、**[データの追加またはアップロード]** を選びます。

6. データ アップロード インターフェイスで、**[ファイルのアップロード]** を選びます。

7. 次の URL からファイルをダウンロードした後、**[参照]** をクリックしてそれを選びます: `https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/data/routes.csv`

8. 格納先を求められたら、先ほど作成したボリューム **adb-dp750** > **default** > **lab_data** を選びます。

9. アップロードが完了したら、左側のサイド バーで **[カタログ]** に移動し、アップロードしたファイルを見つけます。 カタログ階層 (**adb-dp750** > **default** > **lab_data**) を展開して、routes.csv が表示されることを確認します。

    > **注意**: このラボでは、データのクエリや読み込みを行う必要はありません。 目的は、単にアップロード ワークフローについて理解することです。 このデータは、後のラボで使用します。

---

## ラボのノートブックをインポートする

データ ファイルをアップロードしたので、ラボのノートブックを Databricks ワークスペースにインポートします。

1. Azure Databricks ワークスペースの左側のサイド バーで **[ワークスペース]** をクリックします。

2. ラボを格納するフォルダー (Labs/01-explore-azure-databricks など) に移動するか、作成します。

3. フォルダーの横にある **⋮** (ケバブ) メニューをクリックするか、フォルダーを右クリックして、**[インポート]** を選びます。

4. **[URL]** を選択し、次の URL を入力して **[インポート]** をクリックします。`https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/01-explore-azure-databricks.ipynb`

5. インポートしたノートブックを開きます。 ノートブックの上部にあるコンピューティング セレクターで、**[サーバーレス]** コンピューティングを選びます。

---

## ノートブックで続ける

UI ベースの演習はこれで終わりです。 次に、インポートしたノートブック 01-explore-azure-databricks.ipynb を開き、実践的なコーディング演習を続けます。

セルを実行する前に、**[サーバーレス]** コンピューティングが選ばれていることを確認します。
