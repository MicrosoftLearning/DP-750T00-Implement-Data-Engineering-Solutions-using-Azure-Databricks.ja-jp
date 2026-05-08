---
lab:
  index: 09
  title: Unity Catalog でのデータ品質制約の実装と管理
  module: Implement and manage data quality constraints in Unity Catalog
  module-url: 'https://learn.microsoft.com/training/wwl-databricks/implement-manage-data-quality-constraints-unity-catalog/'
  notebook: 'https://github.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/blob/main/Allfiles/09-implement-manage-data-quality-constraints-unity-catalog.ipynb'
  description: このラボでは、ClearCover Insurance 向けに、生の請求データにデータ品質制約を適用する Lakeflow Spark 宣言型パイプラインを構築します。 パイプラインの期待値を使用して NULL 値の許容と範囲チェックを実装し、col().cast() を使用してデータ型を検証したうえで、自動ローダーのレスキューされたデータ列を使用してスキーマ ドリフトを処理します。 その後、Databricks UI でパイプラインを作成して実行し、データ品質メトリックを監視します。
  duration: 45 minutes
  level: 300
  islab: true
  primarytopics:
    - Azure Databricks
---

# ラボ 09: Unity Catalog でのデータ品質制約の実装と管理

## はじめに

あなたは、架空の保険会社である **ClearCover Insurance** のデータ エンジニアです。 毎日、支社やパートナーのブローカーから生の請求データが届きます。 残念ながら、データには一貫性がありません。一部のレコードには必要な識別子が欠落しています。請求金額が文字列として書式設定されている場合や、負の値を含む場合があります。日付の形式が正しくない場合もあります。さらに、時間経過に伴い、ソース スキーマに新しい列が確認なしで付加される可能性があります。

あなたの任務は、パイプラインのすべてのレイヤーでデータ品質制約を適用する **Lakeflow Spark 宣言型パイプライン**を構築し、不良レコードが保険数理モデルやレポート ダッシュボードに到達する前に捕捉することです。

次の演習を行います。

| 演習   | トピック                                                    |
| ---------- | -------------------------------------------------------- |
| 演習 1 | ClearCover Insurance Data Platform (ノートブック) を設定する |
| 演習 2 | カタログ エクスプローラーでデータ品質の問題を調べる          |
| 演習 3 | NULL 値の許容と状態の検証を実装する              |
| 演習 4 | col().cast() を使用してデータ型チェックを追加する                  |
| 演習 5 | レスキューされたデータを使用してスキーマ ドリフトを処理する                    |
| 演習 6 | パイプラインを実行して監視する                             |

---

## 🤖 Databricks アシスタント - 常に使用する

このラボのすべての演習において、**Databricks アシスタントを使用することが想定されており、そうすることをお勧めします**。 すべての演習には、作業を開始するための推奨プロンプトが含まれています。 アシスタントはペア プログラマーです。コードの生成、エラーの解釈、代替手段の探索に使用します。

Databricks アシスタントを開くには、 ![アシスタント アイコン](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/media/databricks-assistant.svg) をノートブック セルの右側で選択するか、キーボード ショートカットを使用します。
---

## 前提条件

- **Azure Databricks Premium ワークスペース**が既にプロビジョニングされていて、使用可能である。
- Python と PySpark の基本的な概念について理解している。
- このラーニング パスの以前のラボを完了している (または Unity Catalog の基本に慣れている)。

---

## セットアップ ノートブックをインポートする

1. Databricks ワークスペースの左側のサイド バーで **[ワークスペース]** をクリックします。
2. このラボを保存するフォルダーに移動するか、そのフォルダーを作成します。
3. **[⋮]** (ケバブ) メニューをクリックするか、フォルダーを右クリックして、**[インポート]** を選択します。
4. **[URL]** を選択し、URL (`https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/09-implement-manage-data-quality-constraints-unity-catalog.ipynb`) を入力し、**[インポート]** をクリックします。
5. インポートしたノートブックを開き、上部のコンピューティング セレクターで **[サーバーレス]** コンピューティングを選択します。

---

## 演習 1: ClearCover Insurance Data Platform を設定する

セットアップ ノートブック **09-implement-manage-data-quality-constraints-unity-catalog** 内のすべてのセルを上から下に実行します。

このノートブックによって、次のオブジェクトが作成されます。

| Object                                | 説明                                                  |
| ------------------------------------- | ------------------------------------------------------------ |
| insurance_lab カタログ                 | ClearCover Insurance Platform の最上位の名前空間    |
| insurance_lab.bronze スキーマ           | ソース システムから受信した生で未処理の請求データ |
| insurance_lab.silver スキーマ           | 検証済みのタイプ セーフなレコード                              |
| insurance_lab.gold スキーマ             | 集計されたレポート データ                                    |
| insurance_lab.bronze.raw_files ボリューム | 生の請求 CSV ファイルのランディング ゾーン                         |
| insurance_lab.bronze.claims_raw テーブル | 生の請求レコード 20 個を含む Delta テーブル                       |

ノートブックが完了したら、続行する前に、**カタログ エクスプローラー**でオブジェクトを確認します。

---

## 演習 2: データ品質の問題を調べる

パイプライン コードを記述する前に、生データを調べて、修正する必要がある品質の問題を把握します。

### タスク 2.1: 生の請求テーブルのクエリを実行する

新しい SQL クエリ エディター (またはノートブック セル) を開き、以下を実行します。

```sql
SELECT *
FROM insurance_lab.bronze.claims_raw
ORDER BY claim_id NULLS LAST;
```

結果を確認し、次の問題ごとに少なくとも 1 つの行を見つけます。

| 問題点                      | チェックする列                     |
| -------------------------- | -------------------------------------- |
| プライマリ識別子の欠落 | claim_id または customer_id が NULL        |
| 解析不可能な日付           | claim_date に日付以外の文字列が含まれている  |
| 解析不可能な金額         | claim_amount が N/A を含んでいるか、空である  |
| 負の金額            | claim_amount が負の数値である      |
| 無効な状態             | 状態が OPEN、PENDING、CLOSED のいずれでもない |

### タスク 2.2: スキーマを検査する

以下を実行して、claim_date と claim_amount が STRING として格納されていることを確認します。

```sql
DESCRIBE TABLE insurance_lab.bronze.claims_raw;
```

これらの列は、ブロンズ レイヤーでは意図的に文字列になっています。 パイプラインの演習では、シルバーへのインジェスト時に正しい型が適用されます。

---

## 演習 3: NULL 値の許容と状態の検証

### タスク 3.0: ETL パイプラインを作成してパイプライン ファイルをインポートする

パイプライン コードを記述する前に、Databricks にスターター パイプライン ファイルをインポートし、Lakeflow Spark 宣言型パイプラインを作成します。

**パイプライン ファイルをインポートします。**

1. Databricks ワークスペースの左側のサイド バーで **[ワークスペース]** をクリックします。
2. ラボのノートブックを保存したフォルダーに移動します。
3. **[⋮]** (ケバブ) メニューをクリックするか、フォルダーを右クリックして、**[インポート]** を選択します。
4. **[URL]** を選択し、URL (`https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/09-implement-manage-data-quality-constraints-unity-catalog.py`) を入力し、**[インポート]** をクリックします。
5. ファイルは Python ソース ファイルとしてワークスペースに表示されます。次のステップ用にパスを書き留めてください。

**パイプラインを作成します。**

1. Databricks ワークスペースの左側のサイド バーで **[ジョブとパイプライン]** をクリックします。
2. **[ETL パイプラインの作成]** (Python) をクリックします。
3. 次の設定でパイプラインを構成します。

   | 設定        | Value                                                                                              |
   | -------------- | -------------------------------------------------------------------------------------------------- |
   | パイプライン名  | **ClearCover Claims Quality Pipeline**                                                             |
   | パイプライン モード  | **トリガー**                                                                                      |
   | ソース コード    | インポートした `09-implement-manage-data-quality-constraints-unity-catalog.py` ファイルを参照      |
   | ターゲット カタログ | **insurance_lab**、スキーマ **silver**                                                               |
   | Compute        | **サーバーレス**                                                                                     |

4. **Create** をクリックしてください。

インポートしたパイプライン ファイルを開きます。これは演習 3 - 5 で開いたままにします。 次に、それを編集してデータ品質制約を追加します。

### タスク 3.1: claims_validated() に NULL 値の許容と状態の期待値を追加する

09-implement-manage-data-quality-constraints-unity-catalog.py を開き、*claims_validated()* 関数に次の期待値を追加します。 `@dp.table(...)` と `def claims_validated():` の間にすべてのデコレーターを配置します。

| 想定される名前  | 条件                                 | アクション        |
| ----------------- | ----------------------------------------- | ------------- |
| valid_claim_id    | `claim_id IS NOT NULL`                    | Drop          |
| valid_customer_id | `customer_id IS NOT NULL`                 | Drop          |
| valid_status      | `status IN ('OPEN', 'PENDING', 'CLOSED')` | 警告 (保持)   |
| valid_coverage    | `coverage_amount > 0`                     | Fail パイプライン |

違反している行を削除するには `@dp.expect_or_drop`、削除せずに警告を表示するには `@dp.expect`、違反時にパイプラインを停止するには `@dp.expect_or_fail` を使用します。

> 🤖 **Databricks アシスタントに質問する:**
>  *"Lakeflow Spark 宣言型パイプラインの Python 関数で expect_or_drop、expect、expect_or_fail デコレーターを使用する方法を教えてください"*

---

## 演習 4: データ型チェック

claim_date 列と claim_amount 列は文字列として受信されます。 col().cast() で値の解析ができない場合、エラーが発生するのではなく NULL が返されます。 この動作を使用して、無効なレコードを特定して削除できます。

### タスク 4.1: claims_validated() 内に col().cast() を適用する

claims_validated() 関数本文の内部で、**return ステートメントの前**に 2 つの withColumn 呼び出しを追加します。

1. `col('claim_date').cast('date')` を使用して、claim_date を STRING から DATE に変換する
2. `col('claim_amount').cast('decimal(12,2)')` を使用して、claim_amount を STRING から DECIMAL(12,2) に変換する

変換された列が元の列に置き換わり、ダウンストリームの期待値とコンシューマーには型指定された値が表示されます。

> 🤖 **Databricks アシスタントに質問する:**
>  *"PySpark で、withColumn と col().cast() を使用してストリーミング データフレーム列を STRING から DATE 型に変換し、もう 1 つの列を STRING から DECIMAL(12,2) に変換します。完全な withColumn 構文を教えてください。"*

### タスク 4.2: 解析不可能な日付のレコードを削除する

タスク 4.1 でのキャスト後、claim_date がまだ NULL である行には、無効な元の値が含まれています。 `@dp.expect_or_drop` デコレーターを追加して次の行を削除します。

```
expectation name: valid_claim_date
condition:        claim_date IS NOT NULL
```

### タスク 4.3: 解析不可能または金額が欠落しているレコードを削除する

同様に、claim_amount をキャストできなかった行を削除します。

```
expectation name: valid_claim_amount
condition:        claim_amount IS NOT NULL
```

### タスク 4.4: 請求金額が負のレコードを削除する

負の請求金額は、どの保険のコンテキストにおいても無効です。 次の行を削除します。

```
expectation name: non_negative_amount
condition:        claim_amount >= 0
```

> 💡**ヒント:** すべての期待値デコレーターを @dp.table(...) と def claims_validated(): の間に配置します。 それらの順序は結果に影響しません。すべての期待値が各行で評価されます。

> 🤖 **Databricks アシスタントに質問する:**
>  *"Python で Lakeflow Spark 宣言型パイプラインを使用しています。col().cast() を適用して列を STRING から DATE に変換した後、キャストが失敗した行を削除するために使用する期待値の条件はどれですか?"*

---

## 演習 5: レスキューされたデータを使用してスキーマ ドリフトを処理する

ClearCover は複数のパートナー ブローカーから請求ファイルを受信します。 ブローカーは、事前の通知なしに追加の列 (broker_reference や fraud_score など) を追加することがあります。 これが発生したときにパイプラインをクラッシュさせるのではなく、予期しないデータを調査のために別の列に取得する必要があります。

### タスク 5.1: 自動ローダーにレスキューのスキーマ進化モードを実装する

09-implement-manage-data-quality-constraints-unity-catalog.py 内の claims_rescued() 関数を完成させます。

自動ローダーで spark.readStream (cloudFiles 形式) を使用して、次から CSV ファイルを読み取ります。

```
/Volumes/insurance_lab/bronze/raw_files/
```

これを以下のオプションを使用して構成します。

| オプション                         | Value                                           |
| ------------------------------ | ----------------------------------------------- |
| cloudFiles.format              | csv                                             |
| ヘッダー                         | true                                            |
| cloudFiles.schemaLocation      | /Volumes/insurance_lab/bronze/raw_files/_schema |
| cloudFiles.schemaEvolutionMode | 救助                                          |
| rescuedDataColumn              | _rescued_data                                   |
| cloudFiles.inferColumnTypes    | true                                            |

**pass** ステートメントを削除し、構成した readStream を返します。

> 🤖 ** Databricks アシスタントに質問する:**
>  *"schemaEvolutionMode のレスキューと _rescued_data 列を含む cloudFiles 形式の CSV を使用して、完全な PySpark 自動ローダーの readStream ブロックを記述してください。各オプションの機能について説明してください。"*

> 💡**ヒント:** ソース ファイルが予想されるスキーマと一致する場合、_rescued_data は行ごとに NULL になります。 今後のファイルで新しい列 (fraud_score など) が追加された場合、その値はパイプラインを中断するのではなく、_rescued_data に JSON として取得されます。

---

## 演習 6: パイプラインを実行して監視する

パイプライン コードが完成したら、演習 3 で作成したパイプラインを実行します。

### タスク 6.1: パイプライン ファイルを保存する

続行する前に、ワークスペース エディターで 09-implement-manage-data-quality-constraints-unity-catalog.py に対するすべての変更を保存済みであることを確認します。

### タスク 6.2: パイプラインを実行する

**[開始]** をクリックして、完全なパイプラインの実行をトリガーします。 実行が完了するのを待ちます。

グラフ ビューでパイプライン DAG を観察します。 3 つのデータセット ノードが表示されます。
- silver.claims_validated
- silver.claims_rescued
- gold.claims_summary

### タスク 6.3: データ品質メトリックを監視する

1. パイプライン グラフで、**claims_validated** データセット ノードをクリックします。
2. 右側のパネルで、**[データ クエリ]** タブを開きます。
3. 期待される結果を確認し、次の点に答えます。
   - どの期待値でレコードが**削除**され、それは何個あったか?
   - どの期待値で**警告**が発行されたか (レコードは保持されているが違反をログに記録)?
   - valid_coverage は **fail** をトリガーしたか? 該当する場合、これはソース内の行に `coverage_amount <= 0` が含まれていることを示します。その原因となった行を調査します。

> 💡**ヒント:** valid_coverage がパイプラインで失敗した場合は、insurance_lab.bronze.claims_raw を調べて、coverage_amount が 0 または NULL の行を探します。 パイプライン イベント ログのエラー メッセージにも、違反レコードが表示されます。

### タスク 6.4: 出力テーブルのクエリを実行する

次のクエリを実行して、パイプラインの出力を確認します。

```sql
-- How many claims made it through all validations?
SELECT COUNT(*) AS valid_claim_count
FROM insurance_lab.silver.claims_validated;

-- What types and statuses appear in the validated silver layer?
SELECT claim_type, status, COUNT(*) AS count
FROM insurance_lab.silver.claims_validated
GROUP BY claim_type, status
ORDER BY claim_type, status;

-- Review the gold summary
SELECT *
FROM insurance_lab.gold.claims_summary
ORDER BY claim_type, status;

-- Did Auto Loader capture any rescued data?
SELECT claim_id, _rescued_data
FROM insurance_lab.silver.claims_rescued
WHERE _rescued_data IS NOT NULL;
```

> 🤖 ** Databricks アシスタントに質問する:**
>  *"insurance_lab.bronze.claims_raw と insurance_lab.silver.claims_validated のカウントを調べ、各行の削減によって、ブロンズ データのデータ品質の問題について説明してください。"*

---

## クリーンアップ (オプション)

完了したら、すべてのラボ リソースを削除するには:

```sql
DROP CATALOG IF EXISTS insurance_lab CASCADE;
```
