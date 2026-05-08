---
lab:
  index: 08
  title: データのクレンジングと変換を行って Unity Catalog に読み込む
  module: 'Cleanse, transform, and load data into Unity Catalog'
  module-url: 'https://learn.microsoft.com/training/wwl-databricks/cleanse-transform-load-data-into-unity-catalog/'
  notebook: 'https://github.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/blob/main/Allfiles/08-cleanse-transform-load-data-into-unity-catalog.ipynb'
  description: このラボでは、Azure Databricks 内の生の不動産データをクリーンアップして整形します。 価格とタイムスタンプに適したデータ型を選び、PySpark を使って重複する一覧項目の削除と欠損値の入力を行い、内部結合と左結合を使ってテーブル間でデータを結合します。 また、SQL PIVOT と UNPIVOT を使って、傾向分析のための市場統計を再構築します。
  duration: 45 minutes
  level: 300
  islab: true
  primarytopics:
    - Azure Databricks
---

# ラボ 08: データのクレンジングと変換を行って Unity Catalog に読み込む

## はじめに

このラボでは、あなたはオランダの主要都市で事業を行う不動産仲介業者である **Pristine Properties** のデータ エンジニアの役割を担います。 同社は複数の代理店やサテライト データベースから、生の不動産物件一覧項目を取り込んでいますが、データは乱雑です。価格は正確でなく、一覧項目は更新バージョンと重複しており、キー フィールドには null が含まれ、市場統計は傾向分析を困難にするワイド列形式で提供されます。

あなたの仕事は、価格ダッシュボードと市場分析モデルに確実にフィードできるよう、データのクリーニング、型チェック、整形を行うことです。

次の演習を行います。

| 演習 | トピック |
|---|---|
| 演習 1 | Pristine Properties プラットフォームを設定する |
| 演習 2 | 一覧項目のデータをプロファイルする |
| 演習 3 | 適切なデータ型を選択する |
| 演習 4 | 重複と欠損値を処理する |
| 演習 5 | 一覧項目と代理店および販売データを結合する |
| 演習 6 | 市場統計のピボットとピボット解除を行う |

---

## 🤖 Databricks アシスタント - 常に使用してください

このラボのすべての演習において、**Databricks アシスタントを使用することが想定されており、そうすることをお勧めします**。 すべてのノートブックのセルには、作業を始めるためのおすすめするプロンプトが含まれています。 アシスタントはあなたとペアを組むプログラマーです。コードの生成、エラー メッセージの説明、代替手段の調査、アプローチの検証にそれをお使いください。

Databricks アシスタントを開くには、 ![アシスタント アイコン](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/media/databricks-assistant.svg) をノートブック セルの右側で選択するか、キーボード ショートカットを使用します。

---

## 前提条件

- **Azure Databricks Premium ワークスペース**が既にプロビジョニングされていて、使用可能である。
- SQL と Python/PySpark の基本的な概念について理解している。

---

## ノートブックのインポート

1. Databricks ワークスペースで、左サイド バーの **[ワークスペース]** をクリックします。

2. このラボを保存するフォルダーに移動するか、作成します。

3. **⋮** (ケバブ) メニューをクリックするか、フォルダーを右クリックして、**[インポート]** を選択します。

4. **[URL]** を選択し、次の URL を入力して **[インポート]** をクリックします。`https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/08-cleanse-transform-load-data-into-unity-catalog.ipynb`

5. インポートしたノートブックを開き、上部のコンピューティング セレクターで **[サーバーレス]** コンピューティングを選択します。

---

## ノートブックを使わないタスク: カタログ エクスプローラーでデータ プロファイルを作成する

演習 1 (環境のセットアップ) を完了した後、Unity Catalog UI を使って、**realestate_lab.bronze.listings** テーブルのデータ プロファイルを作成できます。 これは UI ベースのタスクであり、コードは必要ありません。

1. 左側のナビゲーション ウィンドウから**カタログ エクスプローラー**を開きます。
2. **realestate_lab** カタログ → **bronze** スキーマ → **listings** テーブルに移動します。
3. **[品質]** タブを選びます。
4. **[構成]** をクリックして、データ プロファイルを有効にします。
5. プロファイルの種類として **[スナップショット]** を選びます。これは、listings のような汎用テーブルに適しています。
6. **[保存して実行]** をクリックして、最初のプロファイルを生成します。
7. プロファイルが完了したら、生成されたメトリックを調べます。 次を確認します。
   - **null の数**: 欠損値が最も多い列はどれですか?
   - **個別の数**: 予期しない値が含まれる列はありますか?
   - **値の分布**: *list_price* の値の分散はどのようになっていますか?

これにより、プログラムでデータのクリーニングを始める前に、データの視覚的および統計的な概要がわかります。
