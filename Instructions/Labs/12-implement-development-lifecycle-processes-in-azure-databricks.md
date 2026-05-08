---
lab:
  index: 12
  title: Azure Databricks で開発ライフサイクルのプロセスを実装する
  module: Implement Development Lifecycle Processes in Azure Databricks
  module-url: 'https://learn.microsoft.com/training/wwl-databricks/implement-development-lifecycle-processes-in-azure-databricks/'
  notebook: 'https://github.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/blob/main/Allfiles/12-implement-development-lifecycle-processes-in-azure-databricks.ipynb'
  description: このラボでは、pytest を使ってデータ変換パイプラインのテスト戦略を実装した後、Databricks CLI を使ってパイプラインを Databricks アセット バンドルとしてパッケージ化してデプロイします。
  duration: 45 minutes
  level: 300
  islab: true
  primarytopics:
    - Azure Databricks
---

# ラボ 12: Azure Databricks で開発ライフサイクルのプロセスを実装する

## はじめに

あなたは、倉庫業務チームが使用する**注文処理パイプライン**のメンテナンスを行うデータ エンジニアです。 このパイプラインは、生の注文データを読み取り、無効なレコードを削除し、状態コードを正規化して、税込みの合計を計算します。

パイプラインが運用環境に移行するのに伴い、チームは適切な**ソフトウェア開発ライフサイクル (SDLC) プラクティス**を導入することにしました。 これは次のことを意味します。

- デプロイ前にバグを捕捉するための、**テスト戦略**の実装
- 環境間で一貫してデプロイできるようにするための、**Databricks アセット バンドル (DAB)** としてのパイプラインのパッケージ化
- バンドルを検証、プレビュー、デプロイするための、**Databricks CLI** の使用

このラボは、次の 3 つの部分で構成されています。

| 部分 | トピック | 場所 |
|------|-------|--------|
| **パート 1** | pytest を使って単体テストを実装する | ノートブック |
| **パート 2** | Databricks アセット バンドルを構成する | ワークスペース ターミナル |
| **パート 3** | Databricks CLI を使ってバンドルをデプロイして検証する | ワークスペース ターミナル |

---

## 🤖 このラボ全体で Databricks アシスタントを使用します

すべての演習において、**Databricks アシスタント**を使用することが想定されており、そうすることを**お勧めします**。 

Databricks アシスタントを開くには、 ![アシスタント アイコン](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/media/databricks-assistant.svg) をノートブック セルの右側で選択するか、キーボード ショートカットを使用します。

これは次の目的で使用されます。
- pytest フィクスチャを生成して関数をテストする
- エラー メッセージを理解して失敗したテストを修正する
- YAML のバンドル構成のドラフトを作成する
- CLI コマンド構文を検索する

---

## 前提条件

- **Azure Databricks Premium ワークスペース**が既にプロビジョニングされていて、それにアクセスできる。
- ワークスペースでジョブを作成するためのアクセス許可を持っている (パート 3 に必要)。
- Python と pytest に関する基本的な知識。

---

## パート 1: テスト戦略を実装する (ノートブック)

### ノートブックをインポートする

1. Databricks ワークスペースで、左サイド バーの **[ワークスペース]** をクリックします。
2. このラボを保存するフォルダーに移動するか、作成します。
3. **⋮** (ケバブ メニュー) をクリックするか、フォルダーを右クリックして、**[インポート]** を選択します。
4. **[URL]** を選択し、次の URL を入力して **[インポート]** をクリックします。`https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/12-implement-development-lifecycle-processes-in-azure-databricks.ipynb`
5. インポートしたノートブックを開き、上部にあるコンピューティング セレクターで **[サーバーレス]** コンピューティングを選択します。

### ノートブックの作業を行う

このノートブックには、次の 3 つの演習が含まれています。

- **演習 1**: pytest をインストールし、テスト対象として提供される **transforms.py** モジュールを確認します。
- **演習 2**: pytest フィクスチャを使って、各変換関数の単体テストを記述します。
- **演習 3**: Spark セッションで作成されたデータに対して完全なパイプラインを実行する統合テストを作成します。

すべての演習を終えてから、パート 2 に進みます。

---

## パート 2: Databricks アセット バンドルを構成する

Databricks アセット バンドル (DAB) を使うと、YAML 構成ファイルで**コードとしてのインフラストラクチャ**として、Databricks のリソース (ジョブ、パイプライン、ノートブック) を定義できます。 これにより、デプロイの繰り返しと監査が可能になります。

このパートでは、ローカル コンピューターで **Databricks CLI** を使って注文処理ジョブのバンドルを作成します。

### Databricks CLI をインストールして構成する

1. PowerShell を使って Databricks CLI をインストールします。

   ```powershell
   winget install Databricks.DatabricksCLI
   ```

   次のようにしてインストールを検証します。

   ```powershell
   databricks --version
   ```

2. Azure Databricks ワークスペースに対する CLI の認証を行います。

   ```powershell
   databricks auth login --host https://<your-workspace-url>
   ```

   **<your-workspace-url>** は、ご自分のワークスペースの URL に置き換えます (例: https://adb-1234567890123456.7.azuredatabricks.net))。 ブラウザーの指示に従って認証を完了します。

3. 新しいプロジェクト ディレクトリを作成し、そこに移動します。

   ```powershell
   mkdir ~/order-pipeline-bundle; cd ~/order-pipeline-bundle
   mkdir notebooks, resources
   ```

### バンドル構成ファイルを作成する

あなたのタスクは、次の要件を含む databricks.yml ファイルを作成することです。

- バンドル名: `order-pipeline-bundle`
- 次のものを含む変数セクション:
  - 環境変数 (既定値: `development`)
  - 説明を含み、既定値を持たない cluster_policy_id 変数
- 次のような order-pipeline-job という名前のジョブを定義する resources セクション:
  - 環境変数を使用する表示名を持ちます: ${var.environment}-order-pipeline
  - 次の 2 つのノートブック タスクがあります:
    - validate-data — ./notebooks/validate.py を実行します
    - transform-data — validate-data に依存し、./notebooks/transform.py を実行します
- 次のものを含む targets セクション:
  - dev ターゲット (既定値、開発モード、環境 = 開発)
  - prod ターゲット (運用モード、独自のワークスペース ホスト、環境 = 運用)

次の PowerShell スニペットを**開始点**として使用し、`# TODO` でマークされたセクションを入力します。

```powershell
@'
bundle:
  name: order-pipeline-bundle

variables:
  environment:
    description: The deployment environment name
    default: development
  # TODO: Add a variable named 'cluster_policy_id'
  # It should have a description and no default value.

resources:
  jobs:
    order-pipeline-job:
      name: ${var.environment}-order-pipeline
      tasks:
        - task_key: validate-data
          notebook_task:
            notebook_path: ./notebooks/validate.py
        # TODO: Add a second task named 'transform-data'
        # It should depend on 'validate-data' and run ./notebooks/transform.py
        # Refer to the 'depends_on' key in the Databricks Asset Bundle schema.

targets:
  dev:
    default: true
    mode: development
    variables:
      environment: development
  # TODO: Add a 'prod' target that:
  # - sets mode to production
  # - sets a workspace host (use a placeholder URL for now)
  # - overrides the environment variable to 'production'
'@ | Set-Content databricks.yml
```

> 🤖 **Databricks アシスタントのヒント:** "2 つのターゲット、ジョブ タスク、カスタム変数を含む完全な Databricks アセット バンドルの databricks.yml の例を示してください" と指示し、調整できる完全な参照構成を取得します。**

### プレースホルダー ノートブックを作成する (検証に必要)

バンドル検証では、参照先のノートブックが存在することが確認されます。 2 つのプレースホルダー ノートブック ファイルを作成します。

```powershell
"# validate" | Set-Content notebooks/validate.py
"# transform" | Set-Content notebooks/transform.py
```

---

## パート 3: Databricks CLI を使ってバンドルをデプロイして検証する

バンドルを構成したら、**Databricks CLI** を使って、その検証、プレビュー、ワークスペースへのデプロイを行います。

### ステップ 1 — バンドルを検証する

order-pipeline-bundle ディレクトリ内から次のコマンドを実行します。 これにより、databricks.yml の構文が正しく、有効なリソースを参照していることが確認されます。

```powershell
databricks bundle validate
```

検証が成功すると、バンドル名、ターゲット環境、ワークスペース パスを示す概要が表示されます。 エラーがある場合は、出力を確認し、YAML を修正してから続けます。

> 🤖 **ヒント:** 検証エラー メッセージをコピーして Databricks アシスタントに貼り付け、説明と推奨される修正を取得します。

### ステップ 2 — デプロイ計画をプレビューする

ワークスペースの変更を行う前に、デプロイで作成または更新される内容をプレビューします。

```powershell
databricks bundle plan
```

出力結果を確認します。 order-pipeline-job が**作成される**ことがわかるはずです (まだ存在しないため)。 既定以外のターゲットの場合は、明示的に指定します。

```powershell
databricks bundle plan -t dev
```

### ステップ 3 — バンドルをデプロイする

バンドルを `dev` ターゲットにデプロイします。

```powershell
databricks bundle deploy -t dev
```

デプロイの間に、CLI は次のことを行います。
- ノートブック ファイルをワークスペースにアップロードします
- ワークスペースの **[ジョブとパイプライン]** セクションに order-pipeline-job を作成します (開発モードがアクティブであるため、`[dev <username>]` というプレフィックスが付きます)

### ステップ 4 — デプロイされたリソースを検証する

デプロイが成功したことを確認します。

```powershell
databricks bundle summary
```

出力には、作成されたジョブを直接指す URL が含まれます。 ブラウザーでその URL を開き、ジョブがワークスペースに表示され、両方のタスク (validate-data と transform-data) が正しく構成されていることを確認します。

> 🤖 **ヒント:** "Databricks アセット バンドルの開発モードでは、ジョブ名とスケジュールに対して何が行われますか?" と質問して、** ジョブにプレフィックスとしてユーザー名が付いている理由を理解します。

### ステップ 5 — クリーンアップする (省略可能)

デプロイされたリソースをワークスペースから削除するには:

```powershell
databricks bundle destroy -t dev
```

作成されたジョブの削除を求められたら確認します。

---

## まとめ

このラボでは:

- pytest フィクスチャを使って、個々の変換関数を検証する**単体テスト**を実装しました
- 変数、ジョブ リソース、複数環境のターゲットを含む **Databricks アセット バンドル**を構成しました
- **Databricks CLI** を使って、バンドルのデプロイの検証、計画、デプロイ、確認を行いました
