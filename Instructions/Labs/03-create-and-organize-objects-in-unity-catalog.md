---
lab:
  index: 3
  title: Unity Catalog でオブジェクトを作成および整理する
  module: Create and organize objects in Unity Catalog
  module-url: 'https://learn.microsoft.com/training/wwl-databricks/create-and-organize-objects-in-unity-catalog/'
  notebook: 'https://github.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/blob/main/Allfiles/03-create-and-organize-objects-in-unity-catalog.ipynb'
  description: このラボでは、大学のデータ プラットフォーム用の完全な Unity Catalog 名前空間を構築します。カタログ、メダリオン スキーマ、主キーと外部キーの制約を持つマネージド テーブル、ビュー、ボリューム、再利用可能な SQL 関数を作成します。 列の追加やガバナンス タグの適用などの DDL 操作を練習し、Unity Catalog がメダリオン アーキテクチャのすべてのレイヤーでどのように構造化データを整理および管理するかを調べます。 最終的に、実際の Data Engineering プラクティスを Azure Databricks に反映する、完全に構造化されたクエリ対応の環境を実現します。
  duration: 45 minutes
  level: 300
  islab: true
  primarytopics:
    - Azure Databricks
---

# ラボ 03: Unity Catalog でオブジェクトを作成および整理する

## シナリオ

あなたは、大学の業務のデジタル化を行っている中規模の機関である **Lakeside University** のデータ エンジニアです。 チームは、学生の記録、コース カタログ、登録データを管理するために、Azure Databricks で最新のデータ プラットフォームを構築する作業を任されています。

このラボでは、Lakeside University の開発環境用の完全な Unity Catalog 名前空間を設計して実装します。 カタログ、スキーマ、制約を持つテーブル、ビュー、ボリューム、再利用可能な SQL 関数のすべてを、組織の名前付け規則に従って作成します。

## 目標

このラボを終えるまでに、次のことを行います。

- Unity Catalog の名前付け規則に従って、カタログとメダリオン スキーマを作成します。
- 主キー制約と外部キー制約を備えたマネージド テーブルを作成します。
- 分析クエリに対応するための標準ビューとマテリアライズド ビューを構築します。
- マネージド ボリュームを作成し、そこに CSV ファイルを読み込みます。
- 成績分類用の再利用可能な SQL スカラー関数を記述します。
- **ALTER** ステートメントを使って、テーブルを拡張し、ガバナンス タグを適用します。

このラボは完了するまで、約 **45** 分かかります。

---

## 🤖 このラボ全体で Genie Code を使用する

このラボでは、**Genie Code** を常に使用することが想定されており、そうすることをおすすめします。 すべての演習には推奨プロンプトが含まれており、Genie Code パネルに直接貼り付けることができます。

Genie Code を開くには、次を選択します ![アシスタント アイコン](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/media/genie-code.svg) をノートブック セルの右側で選択するか、キーボード ショートカットを使用します。

> 💡**ヒント:** Genie Code の出力を無条件にコピーして貼り付けることはしないでください。 それを読み、理解し、各タスクの特定の要件に合わせて調整します。 Genie Code は思考を加速させるためのツールであり、あなたの考えに置き換わるものではありません。

---
'Before starting this lab, ensure you have':
  - 'An **Azure Databricks Premium workspace** provisioned using [Lab 00': 'Set up your Azure Databricks environment](00-setup.md).'
  - An active **Unity Catalog metastore** attached to the workspace.
  - The **CREATE CATALOG** privilege on the metastore (granted by your instructor or workspace admin).
  - 'Familiarity with basic SQL (CREATE TABLE, SELECT, JOIN).'
---

## ラボのノートブックをインポートする

1. Databricks ワークスペースで、左側のサイド バーの **[ワークスペース]** を選びます。
2. ラボのノートブックを格納するフォルダーに移動するか、作成します (ホーム フォルダーなど)。
3. **⋮** (ケバブ) メニューを選ぶか、フォルダーを右クリックして、**[インポート]** を選びます。
4. **[URL]** を選んで次の URL を入力し、**[インポート]** を選びます: `https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/03-create-and-organize-objects-in-unity-catalog.ipynb`
5. インポートしたノートブックを開きます。 ノートブックの上部にあるコンピューティング セレクターで、**[サーバーレス]** コンピューティングを選びます。

---

## ノートブックの作業を行う

インポートしたノートブックを開き、各セルの指示に従って演習 1 から 6 を行います。 ノートブックの最後にある **Next steps** セルまでいったら、ここに戻って、以下の演習を続けます。

---

## 演習: AI/BI Genie スペースを構成する (省略可能)

この省略可能なタスクに必要な Genie スペースは、ノートブックではなく Databricks UI を使ってすべて構成します。

ノートブックの演習を終えた後、興味があれば、Genie スペースを作成し、自然言語を使って Lakeside University のデータのクエリを実行してみることができます。

1. 左側のサイド バーで、**[+ 新規]** > **[Genie スペース]** を選びます。
2. **[データ]** で、次のテーブルを追加します。
   - `edu_dev.silver.students`
   - `edu_dev.silver.courses`
   - `edu_dev.silver.enrollments`
   - `edu_dev.silver.vw_student_enrollments`
   - `edu_dev.gold.vw_department_enrollment_stats`
3. スペースに `Lakeside University Analytics` という名前を付けます。
4. `enrollments.grade` 列の説明を `Numerical grade on a 0.0–10.0 scale where 8.5+ is an A, 7.0+ is a B, 5.5+ is a C, 4.0+ is a D, and below 4.0 is an F.` に更新します。
5. **[チャット]** タブに移動し、"*平均成績が最も高い学部はどこですか?*" と尋ねます。
6. 生成された SQL Genie を確認し、**vw_department_enrollment_stats** マテリアライズド ビューと比べます。

> 🤖 **Genie Code のヒント:** Genie スペース内から Genie Code に質問し、SQL の命令の記述や列のシノニムの定義に役立てることができます。

---

## クリーンアップする (省略可能)

このラボの間に作成されたリソースを削除したい場合は、ノートブックで次を実行します。

```sql
DROP CATALOG IF EXISTS edu_dev CASCADE;
```

> ⚠️ これを行うと、edu_dev の下に作成されたすべてのスキーマ、テーブル、ビュー、ボリューム、関数が完全に削除されます。 これらのオブジェクトがもう必要ないことが確実な場合にのみ、これを実行してください。
