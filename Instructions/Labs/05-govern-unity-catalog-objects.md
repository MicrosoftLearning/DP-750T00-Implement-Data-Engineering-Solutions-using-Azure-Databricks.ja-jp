---
lab:
  index: 5
  title: Unity カタログ オブジェクトを管理する
  module: Govern Unity Catalog objects
  module-url: 'https://learn.microsoft.com/training/wwl-databricks/govern-unity-catalog-objects/'
  notebook: 'https://github.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/blob/main/Allfiles/05-govern-unity-catalog-objects.ipynb'
  description: このラボでは、Azure Databricks 上に構築した接続された車両データ プラットフォームに Unity Catalog ガバナンス コントロールを適用します。 SQL を使用して、テーブルと列に PII 分類のタグを付け、Delta Lake データ保持ポリシーを構成し、VACUUM を実行して削除されたデータを消去し、予測の最適化を有効にします。 次に、システム テーブルにクエリを実行して、データ系列をプログラムでトレースし、監査ログを分析して、だれがいつデータにアクセスしたかに関するコンプライアンスの質問に答えます。
  duration: 30 minutes
  level: 300
  islab: true
  primarytopics:
    - Azure Databricks
---

# ラボ 05: Unity Catalog オブジェクトを管理する

## シナリオ

あなたは、グローバルな自動車メーカーである **AutoSphere AG** のデータ エンジニアとして、Azure Databricks 上にコネクテッド カーのデータ プラットフォームを構築しています。 そのプラットフォームは、数百万台の車両からテレメトリを取り込み、顧客と車両の登録データを管理し、サービス レコードを追跡します。

データ ガバナンス チームは、次の懸念事項を提起しました。

- 車両テレメトリ データの書き込みパターンとして書き込みの頻度が高いことから、ストレージの増大を抑制し、GDPR のデータ最小化要件に準拠するために、明確に定義された**データ保持ポリシー**が必要です。
- データ チームは、テーブルがどのように派生したかを理解し、アップストリームでの変更の影響をトレースするために、**完全な系列の可視性**を必要としています。
- コンプライアンス チームは、手動のログ検索ではなくクエリ可能なログを使用して、**だれがいつどのデータにアクセスしたかを監査する**必要があります。

このラボでは、Azure Databricks Unity Catalog でタグ付け、データ保持ポリシー、系列クエリ、監査ログ分析を適用することで、これらすべての問題に対処します。

---

## 目標

このラボを終えるまでに、次のことを行います。

- テーブルと列にデータ検出のための説明的なコメントとタグを適用する。
- Delta Lake のデータ保持設定を構成し、VACUUM 操作を実行する。
- カタログ エクスプローラーでデータ系列を視覚的に表示する。
- 系列システム テーブルにプログラムでクエリを実行する。
- 監査ログ システム テーブルにクエリを実行して、データ アクセス パターンを調査する。

このラボは完了するまで、約 **30** 分かかります。

---

## 🤖 このラボ全体で Databricks アシスタントを使用する

このラボでは、**Databricks アシスタントを常に**使用することが**想定されており、そうすることをお勧めします**。 ノートブック内のすべての演習セルには、アシスタント パネルに直接貼り付けることができる推奨プロンプトが含まれています。

Databricks アシスタントを開くには、 ![アシスタント アイコン](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/media/databricks-assistant.svg) をノートブック セルの右側で選択するか、ツール バーに表示されているキーボード ショートカットを押します。

> 💡**ヒント:** アシスタントの出力を無条件にコピーして貼り付けることはしないでください。 それをよく読んで理解し、目の前の作業に合わせて調整してください。 アシスタントは思考を加速させるためのツールであり、それを置き換えるものではありません。

---
'Before starting this lab, ensure you have':
  - Access to an **Azure Databricks Premium workspace** (already provisioned for you).
  - An active **Unity Catalog metastore** attached to the workspace.
  - The **CREATE CATALOG** privilege on the metastore.
  - 'Familiarity with basic SQL (CREATE TABLE, SELECT, ALTER TABLE).'
---

## ラボのノートブックをインポートする

1. Azure Databricks ワークスペースで、左側のサイド バーにある **[ワークスペース]** を選択します。
2. このラボを保存するフォルダーに移動するか、そのフォルダーを作成します。
3. フォルダーの横にある **[⋮]** (ケバブ) メニューを選択し、**[インポート]** を選択します。
4. **[URL]** を選択し、URL (`https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/05-govern-unity-catalog-objects.ipynb`) を入力し、**[インポート]** を選択します。
5. インポートしたノートブックを開き、上部のコンピューティング セレクターで **[サーバーレス]** コンピューティングを選択します。

---

## ノートブックを開く前に: カタログ エクスプローラーでデータ系列を表示する

ノートブックで演習 1 (テーブルを作成する) を完了したら、ここに戻り、次の手順に従って**カタログ エクスプローラー**で系列を視覚的に調べます。 これは UI でのタスクであり、ノートブック コードは必要ありません。

> ⚠️ 最初にノートブックの演習 1 を完了してから、ここに戻ります。

### テーブル系列を表示する

1. Azure Databricks ワークスペースで、左側のサイド バーの **[カタログ]** をクリックしてカタログ エクスプローラーを開きます。
2. **automotive_catalog** > **governance_lab** に移動します。
3. **vehicle_telemetry** テーブルを選択します。
4. **[系列]** タブを選択します。
5. **[系列グラフの表示]** を選択して対話型の系列の視覚化を開きます。

アップストリームとダウンストリームの関係を観察します。 グラフに次のことが表示されていることに注意してください。
- どのノートブックまたはジョブがテーブルに書き込んだか。
- ダウンストリームのどのテーブルまたはビューがそれに依存しているか。

### 列レベルの系列を表示する

1. 引き続き **[系列]** タブで、**service_records** テーブル ノードを選択します。
2. 任意の列 (たとえば **vehicle_id**) を選択し、それがどのアップストリーム列にトレースバックされるかを調べます。

### テーブル履歴の表示

1. カタログ エクスプローラーで **vehicle_telemetry** テーブルに移動します。
2. **履歴**タブを選択します。
3. バージョン履歴を確認します。各行は 1 つの操作 (書き込み、更新、VACUUM など) を表します。

この履歴は、だれがいつテーブルを変更したかを理解するための監査証跡として利用できます。

---

> カタログ エクスプローラーでの系列の探索が完了したら、ノートブックの残りの演習に進みます。
