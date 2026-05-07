---
lab:
  index: 6
  title: Azure Databricks を使用したデータ モデリングの設計と実装
  module: Design and implement data modeling with Azure Databricks
  module-url: 'https://learn.microsoft.com/training/wwl-databricks/design-implement-data-modeling-unity-catalog/'
  notebook: 'https://github.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/blob/main/Allfiles/06-design-implement-data-modeling-unity-catalog.ipynb'
  description: このラボでは、Unity Catalog でリテール バンキング シナリオ用の Delta Lake データ モデルを設計して実装し、SCD Type 2 履歴追跡を使用して顧客ディメンションを構築し、リキッド クラスタリングを使用したトランザクション ファクト テーブルを作成します。 変更データ フィードを適用して、クエリ可能な FCA コンプライアンス監査証跡を作成します。また、Delta Lake のタイム トラベル機能を使用して以前のテーブル バージョンを検査し、復元します。
  duration: 45 minutes
  level: 300
  islab: true
  primarytopics:
    - Azure Databricks
---

# ラボ 06: Azure Databricks を使用したデータ モデリングの設計と実装

## シナリオ

あなたは、イギリス全土に事業を展開しているリテール バンクである **Northbank Financial** のデータ エンジニアです。 データ エンジニアリング チームは、顧客分析、規制レポート、不正行為の検出を強化するために、Azure Databricks で最新のレイクハウス プラットフォームの構築を進めています。

チームは次の要件に対応する必要があります。

- **顧客ディメンション管理:** 市区町村、アカウント セグメント、アカウントの種類などの顧客属性は、時間の経過に伴って変化します。 規制レポートには、顧客の現在の状態だけでなく、**トランザクションの時点でのプロファイル**が反映されている必要があります。
- **トランザクション ファクト ストレージ:** プラットフォームでは、何百万もの毎日の支払いトランザクションを効率的に格納してクエリを実行できる必要があります。 クエリ パターンは特定の日付範囲を対象とするため、物理データ組織は高速なフィルター処理をサポートする必要があります。
- **コンプライアンスの監査証跡:** Basel III および FCA 規制では、トランザクション レコードの変更が完全に追跡可能であることが必要とされます。 チームは、トランザクション データに対するすべての変更のクエリ可能なログを必要としています。
- **履歴データの回復:** バックアップから復元することなく、偶発的なデータ変更を回復できる必要があります。 このソリューションでは、ポイントインタイム リストアをレイクハウス内で直接サポートする必要があります。

このラボでは、データ モデリングの概念を適用してこれらの要件を設計および実装します。そのために、**Delta Lake**、**Unity Catalog**、**SCD Type 2**、**変更データ フィード**、**Delta Lake タイム トラベル**を使用します。

---

## 目標

このラボを終えるまでに、次のことを行います。

- マネージド Delta Lake テーブルを使用して Unity Catalog データ モデルを作成する。
- **リキッド クラスタリング**を適用して、トランザクション テーブルでのクエリのパフォーマンスを最適化する。
- **MERGE** を使用して顧客ディメンションに **SCD Type 2** を実装する。
- ポイントインタイム フィルターを使用して顧客の履歴レコードのクエリを実行する。
- **変更データ フィード**を有効にし、**table_changes()** を使用して監査証跡のクエリを実行する。
- **Delta Lake タイム トラベル**を使用して以前のテーブル バージョンを検査および復元する。

このラボは完了するまで、約 **45** 分かかります。

---

## 🤖 このラボ全体で Databricks アシスタントを使用する

このラボでは、**Databricks アシスタントを常に**使用することが**想定されており、そうすることをお勧めします**。 ノートブック内のすべての演習セルには、アシスタント パネルに直接貼り付けることができる推奨プロンプトが含まれています。

Databricks アシスタントを開くには、 ![アシスタント アイコン](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/media/databricks-assistant.svg) をノートブック セルの右側でクリックするか、ツール バーに表示されているキーボード ショートカットを押します。

> 💡 **ヒント:** アシスタントの出力を無条件にコピーして貼り付けることはしないでください。 それをよく読んで理解し、目の前の作業に合わせて調整してください。 アシスタントは思考を加速させるためのツールであり、それを置き換えるものではありません。

---
'Before starting this lab, ensure you have':
  - Access to an **Azure Databricks Premium workspace** (already provisioned for you).
  - An active **Unity Catalog metastore** attached to the workspace.
  - The **CREATE CATALOG** privilege on the metastore.
  - 'Familiarity with basic SQL (CREATE TABLE, SELECT, MERGE) and Python/PySpark.'
---

## ラボのノートブックをインポートする

1. Azure Databricks ワークスペースの左側のサイド バーで **[ワークスペース]** をクリックします。
2. このラボを保存するフォルダーに移動するか、そのフォルダーを作成します。
3. フォルダーの横にある **[⋮]** (ケバブ) メニューをクリックし、**[インポート]** を選択します。
4. **[URL]** を選択し、URL (`https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/06-design-implement-data-modeling-unity-catalog.ipynb`) を入力し、**[インポート]** をクリックします。
5. インポートしたノートブックを開き、上部にあるコンピューティング セレクターで **[サーバーレス]** コンピューティングを選択します。

---

## ノートブックを開く前に: カタログ エクスプローラーでマネージド テーブルと外部テーブルを探索する

このタスクは、ノートブック コードを必要とせず、**Databricks UI** を使用して完了する必要があります。

### マネージド テーブルと外部テーブルを比較する

ノートブックで**演習 1** を完了 (カタログとテーブルを作成) したら、ここに戻って次の手順を実行します。

> ⚠️ 最初にノートブックの演習 1 を完了してから、ここに戻ります。

1. Databricks ワークスペースにサインインし、左側のサイド バーの **[カタログ]** をクリックして**カタログ エクスプローラー**を開きます。
2. **banking_lab** カタログを展開し、次に **silver** スキーマを展開します。
3. **dim_customer** テーブルをクリックして詳細パネルを開きます。
4. **[詳細]** タブで、**[保存場所]** フィールドを見つけます。
   - ストレージのパスは Unity Catalog で管理されていることに注意してください。テーブル作成時に [保存先] を指定しませんでした。**
   - これは**マネージド テーブル**です。メタデータと基になるデータ ファイルの両方が Unity Catalog によって制御されます。
5. **fact_transactions** テーブルに対して同じ検査を繰り返します。
6. **fact_transactions** の**クラスタリング**に関する情報に注意してください。これにより、リキッド クラスタリングがアクティブであることがわかります。

### 覚えておくべき主な違い

| | マネージド テーブル | 外部テーブル |
|---|---|---|
| **Metadata** | Unity Catalog | Unity Catalog |
| **データ ファイル** | Unity Catalog によって管理されている場所 | ユーザー指定の場所 |
| **DROP TABLE の動作** | 8 日経過後ファイルを削除する | ファイルは残る |
| **自動最適化** | 予測最適化が利用可能 | 使用不可 |

Northbank の分析プラットフォームにはマネージド テーブルが最適な選択肢であり、自動メンテナンスとシンプルなガバナンスというメリットを享受できます。

---

では、ノートブックに戻り、**演習 2** に進みましょう。
