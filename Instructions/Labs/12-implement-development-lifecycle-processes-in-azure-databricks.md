---
lab:
  index: 12
  title: Azure Databricks で開発ライフサイクルのプロセスを実装する
  module: Implement Development Lifecycle Processes in Azure Databricks
  module-url: 'https://learn.microsoft.com/training/wwl-databricks/implement-development-lifecycle-processes-in-azure-databricks/'
  notebook: 'https://github.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/blob/main/Allfiles/12-implement-development-lifecycle-processes-in-azure-databricks.ipynb'
  description: このラボでは、pytest を使ってデータ変換パイプラインのテスト戦略を実装した後、Databricks CLI を使ってパイプラインを宣言型オートメーション バンドルとしてパッケージ化してデプロイします。
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
- 環境間で一貫してデプロイできるようにするための、**宣言型オートメーション バンドル (DAB)** としてのパイプラインのパッケージ化
- バンドルを検証、プレビュー、デプロイするための、**Databricks CLI** の使用

このラボは、次の 3 つの部分で構成されています。

| 部分 | トピック | 作業に使用する機能 |
|------|-------|--------|
| **パート 1** | pytest を使って単体テストを実装する | Azure Databricks ワークスペース内のサーバーレス コンピューティングで実行されているノートブック内 |
| **パート 2** | 宣言型オートメーション バンドルを構成する | **ローカル マシンの PowerShell ターミナル** 内 (Databricks CLI をローカルで実行) |
| **パート 3** | Databricks CLI を使ってバンドルをデプロイして検証する | 同じ **ローカルの PowerShell ターミナル** 内。その後、Databricks ワークスペース UI で検証 |

> **重要事項:** パート 2 および 3 では、Databricks ワークスペース内のターミナルではなく、**お使いのコンピューターまたはラボ仮想マシンにインストールされた Databricks CLI** を使用します。 それらのパートのローカル PowerShell ウィンドウを開き、作業中は開いたままにしておきます。

---

## 🤖 このラボ全体で Genie Code を使用する

すべての演習において、**Genie Code** を使用することが想定されており、そうすることを**おすすめします**。 

Genie Code を開くには、次を選択します ![アシスタント アイコン](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/media/genie-code.svg) をノートブック セルの右側で選択するか、キーボード ショートカットを使用します。

これは次の目的で使用されます。
- pytest フィクスチャを生成して関数をテストする
- エラー メッセージを理解して失敗したテストを修正する
- YAML のバンドル構成のドラフトを作成する
- CLI コマンド構文を検索する

---
- 'An **Azure Databricks Premium workspace** provisioned using [Lab 00': 'Set up your Azure Databricks environment](00-setup.md).'
- You have permission to create jobs in the workspace (required for Part 3).
- Basic familiarity with Python and pytest.
---

## パート 1: テスト戦略を実装する (ノートブック)

### ノートブックをインポートする

1. Databricks ワークスペースで、左側のサイド バーにある **[ワークスペース]** をクリックします。
2. このラボを保存するフォルダーに移動するか、そのフォルダーを作成します。
3. **[⋮]** (ケバブ メニュー) をクリックするか、フォルダーを右クリックして、**[インポート]** を選択します。
4. **[URL]** を選択し、URL (`https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/12-implement-development-lifecycle-processes-in-azure-databricks.ipynb`) を入力し、**[インポート]** をクリックします。
5. インポートしたノートブックを開き、上部にあるコンピューティング セレクターで **[サーバーレス]** コンピューティングを選択します。

### ノートブックの作業を行う

このノートブックには、次の 3 つの演習が含まれています。

- **演習 1**: pytest をインストールし、テスト対象として提供される **transforms.py** モジュールを確認します。
- **演習 2**: pytest フィクスチャを使って、各変換関数の単体テストを記述します。
- **演習 3**: Spark セッションで作成されたデータに対して完全なパイプラインを実行する統合テストを作成します。

すべての演習を終えてから、パート 2 に進みます。

---

## パート 2: 宣言型オートメーション バンドルを構成する

宣言型オートメーション バンドル (DAB) を使うと、YAML 構成ファイルで**コードとしてのインフラストラクチャ**として、Databricks のリソース (ジョブ、パイプライン、ノートブック) を定義できます。 これにより、デプロイの繰り返しと監査が可能になります。

このパートでは、ローカル コンピューターで **Databricks CLI** を使って注文処理ジョブのバンドルを作成します。

> **作業に使用する機能:** パート 2 および 3 のすべてのコマンドは、**お使いのコンピューターやラボ仮想マシン上の PowerShell ターミナル**で実行されます。 今すぐ **Windows ターミナル**または **PowerShell** を開き、このラボの残りの作業を行う間、開いたままにしておいてください。 Databricks CLI はネットワーク経由でワークスペースと通信します。これらのコマンドは、Databricks の Web UI 内で実行するものでは**ありません**。

### Databricks CLI をインストールして構成する

1. ローカルの PowerShell ウィンドウで、Databricks CLI をインストールします。

   ```powershell
   winget install Databricks.DatabricksCLI
   ```

   次に、PowerShell を**閉じてから再度開き**、新しい `databricks` コマンドが PATH に含まれるようにし、インストールを確認します。

   ```powershell
   databricks --version
   ```

   "予想される結果:" コマンドはバージョン番号 (例: `Databricks CLI v0.x.x`) を出力します。** コマンドが認識されないと報告された場合は、PowerShell を再度開いて、もう一度試してください。

2. Azure Databricks ワークスペースに対する CLI の認証を行います。

   ```powershell
   databricks auth login --host https://<your-workspace-url>
   ```

   **<your-workspace-url>** は、ご自分のワークスペースの URL に置き換えます (例: https://adb-1234567890123456.7.azuredatabricks.net))。 これは、ワークスペースが開いているときにブラウザーのアドレス バーからコピーできます。 **プロジェクト名**の入力を求められたら、Enter キーを押して既定値をそのまま使用します。 ブラウザー ウィンドウが開きます。サインインして、要求を承認します。

   "予想される結果:" PowerShell によって `Profile DEFAULT was successfully saved` が表示され、CLI がワークスペースに対して認証されたことを確認できます。**

3. 新しいプロジェクト ディレクトリと、バンドルに必要な 2 つのサブフォルダーを作成し、プロジェクト ディレクトリに移動します。

   ```powershell
   mkdir ~/order-pipeline-bundle; cd ~/order-pipeline-bundle
   mkdir notebooks, resources
   ```

   "予想される結果:" PowerShell プロンプトが `order-pipeline-bundle` で終了し、フォルダーには空の `notebooks` と `resources` サブフォルダーが含まれるようになります。** `ls` を実行して確定します。

### バンドル構成ファイルを作成する

**databricks.yml** ファイルは宣言型自動化バンドルの中心であり、展開するリソースを記述します。 このステップでは、`order-pipeline-bundle` フォルダーのルートにそのファイルを作成します。 あなたが担う作業は、次の要件を満たす databricks.yml ファイルを作成することです。

- バンドル名: `order-pipeline-bundle`
- 次のものを含む変数セクション:
  - `environment` 変数 (既定値: `development`)
  - 説明を含み、**空の文字列 (`""`) を既定値とする** `cluster_policy_id` 変数。 クラスター ポリシー ID により、ジョブ クラスターに構成規則が適用され、運用バンドルでは、それらのクラスターでこの値を参照します。 このラボをシンプルに保つために (ジョブではサーバーレス コンピューティングを使用します)、空の文字列を既定値とし、この変数を**省略できる**ようにしているため、デプロイ時にこの変数を指定する必要はありません。 実際の値を使用した方がよい場合は、**[コンピューティング]**  >  **[ポリシー]** でワークスペースのポリシー ID をコピーします (ポリシーを開き、その詳細または URL に表示されている ID を使用します)。
- 次のような order-pipeline-job という名前のジョブを定義する resources セクション:
  - 環境変数を使用する表示名を持ちます: ${var.environment}-order-pipeline
  - 次の 2 つのノートブック タスクがあります:
    - validate-data — ./notebooks/validate.py を実行します
    - transform-data — validate-data に依存し、./notebooks/transform.py を実行します
- 次のものを含む targets セクション:
  - dev ターゲット (既定値、開発モード、環境 = 開発)
  - prod ターゲット (運用モード、独自のワークスペース ホスト、環境 = 運用)

このファイルはコマンド ラインからではなくプレーンテキスト エディターを使用して作成するため、YAML の読み取りと編集が容易になります。

1. テキスト エディターで `order-pipeline-bundle` フォルダーを開きます。

   - **VS Code** (推奨): PowerShell ウィンドウで、`code .` を実行して現在のフォルダーを開きます。 `code` コマンドを使用できない場合は、VS Code を手動で開き、**[ファイル]**  >  **[フォルダーを開く**] の順に選択し、`order-pipeline-bundle` フォルダーを選択します。
   - **メモ帳**: `notepad databricks.yml` を実行します。 ファイルがまだ存在しないため、作成するかどうかをメモ帳から尋ねられます。**[はい]** を選択します。

2. `order-pipeline-bundle` フォルダーのルートに **databricks.yml** という名前の新しいファイルを作成します (VS Code で、**[新しいファイル]** を選択し、`databricks.yml` という名前を付けます)。

3. 次のスターター構成をファイルにコピーします。 バンドル名、`environment` 変数、最初のタスク、`dev` ターゲットが既に含まれています。 `# TODO` のマークが付いた 3 つのセクションを完成します。

   ```yaml
   bundle:
     name: order-pipeline-bundle

   variables:
     environment:
       description: The deployment environment name
       default: development
     # TODO: Add a variable named 'cluster_policy_id'.
     # Give it a 'description' and a 'default' value of "" (an empty string),
     # which makes the variable optional so you don't have to supply it at deploy time.
     # (A real cluster policy ID comes from Compute > Policies in your workspace,
     # but you don't need one for this lab.)

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
           # Refer to the 'depends_on' key in the Declarative Automation Bundle schema.

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
   ```

   > **ヒント:** YAML ではインデントが区別されるため、各レベルで **2 つのスペース** を使用し、タブは使用しないでください。 各 `# TODO` ブロックを周囲の行の例と揃えてください。

4. ファイルを **[保存]** します (VS Code では **Ctrl + S** キーを押し、メモ帳では **[ファイル]**  >  **[保存]** を選択します)。 ノートブックまたはリソース内ではなく、order-pipeline-bundle フォルダー内に `databricks.yml` として直接保存されたことを確認してください。

> 🤖 **Genie Code のヒント:** "2 つのターゲット、ジョブ タスク、カスタム変数を含む完全な宣言型オートメーション バンドルの databricks.yml の例を示してください" と指示し、調整できる完全な参照構成を取得します。**

> **解答が必要な場合** すべての `# TODO` セクションが入力された完全な参照ファイルは、[Allfiles/answers/12-databricks-bundle-with-answers.yml](https://github.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/blob/main/Allfiles/answers/12-databricks-bundle-with-answers.yml) にあります。 まず、自分で `# TODO` セクションを完成してから、解答と照らし合わせてみてください。

"予想される結果:" databricks.yml ファイルは order-pipeline-bundle のルートに存在し、cluster_policy_id 変数、depends_on を含む 2 つ目の transform-data タスク、追加した prod ターゲットが含まれます。** これは、PowerShell で `Get-Content databricks.yml` を使用して確認できます。

### プレースホルダー ノートブックを作成する (検証に必要)

バンドルの検証では、`databricks.yml` で参照されているすべてのノートブックがディスク上に実際に存在し、**かつ Databricks ノートブック**であることを確認します。 プレーンな `.py` ファイルでは不十分です。ソース形式の Databricks ノートブックは特別な 1 行目 `# Databricks notebook source` で始まる必要があり、かつファイルを **バイト順マーク (BOM) なしの UTF-8** として保存する必要があります。 ファイルに BOM が含まれているか、または別のエンコードが使用されている場合、Databricks CLI はヘッダーを検出できず、エディター上ではテキストが正しく見える場合でも、検証は、"… はノートブックではありません" というエラーで失敗します。**

ローカルの PowerShell ウィンドウ (引き続き `order-pipeline-bundle` フォルダー内) で次のコマンドを実行して、両方のプレースホルダー ノートブックを BOM なしの UTF-8 として作成します。

```powershell
[System.IO.File]::WriteAllLines("$PWD\notebooks\validate.py",  @("# Databricks notebook source", "# validate"))
[System.IO.File]::WriteAllLines("$PWD\notebooks\transform.py", @("# Databricks notebook source", "# transform"))
```

> **`Set-Content` ではない理由** PowerShell のバージョンによっては、`Set-Content` (または `Out-File`) の前に BOM を付加したり、UTF-16 を使用したりすることができますが、これがまさに "ノートブックではありません" というエラーの原因となります。** `[System.IO.File]::WriteAllLines` は常に BOM **なし**の UTF-8 を作成するため、マジック ヘッダーがファイルの先頭に配置されます。

"予想される結果:" `notebooks` フォルダーには、それぞれ `# Databricks notebook source`で始まる `validate.py` と `transform.py` が含まれます。** `Get-Content notebooks/validate.py` を実行して、ヘッダーが 1 行目にあることを確認します。 `databricks bundle validate` を再実行すると、"ノートブックではありません" というエラーは表示されなくなります。**

---

## パート 3: Databricks CLI を使ってバンドルをデプロイして検証する

バンドルを構成したら、**Databricks CLI** を使って、その検証、プレビュー、ワークスペースへのデプロイを行います。

> **作業に使用する機能:** このパートのコマンドはすべて、**ローカルの PowerShell ウィンドウ**で、`order-pipeline-bundle` フォルダー内から実行します。 PowerShell を閉じてしまった場合は、もう一度開き、最初に `cd ~/order-pipeline-bundle` を実行します。

### ステップ 1 — バンドルを検証する

order-pipeline-bundle ディレクトリ内から次のコマンドを実行します。 これにより、databricks.yml の構文が正しく、有効なリソースを参照していることが確認されます。

```powershell
databricks bundle validate
```

"予想される結果:" このコマンドを実行すると、バンドルの **名前** (`order-pipeline-bundle`)、**[ターゲット]** (`dev`)、**[ワークスペース]** のパスを含む要約が出力され、続いて `Validation OK!` と表示されます。** 代わりにエラーが表示された場合は、メッセージを確認し (通常、問題のある行やキーが示されます)、YAML を修正してからコマンドをもう一度実行します。

> 🤖 **ヒント:** 検証エラー メッセージをコピーして Genie Code に貼り付け、説明と推奨される修正を取得します。

### ステップ 2 — デプロイ計画をプレビューする

ワークスペースの変更を行う前に、デプロイで作成または更新される内容をプレビューします。

```powershell
databricks bundle plan
```

出力結果を確認します。 "予想される結果:" プランから、order-pipeline-job を**作成する**ことが報告されます (まだ存在しないため)。この段階では、リソースは変更されません。** 特定のターゲットに対してプランを実行するには、そのターゲットを明示的に指定します。

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

"予想される結果:" コマンドが `Deployment complete!` などのメッセージで終了し、エラーは発生しません。**

### ステップ 4 — デプロイされたリソースを検証する

デプロイが成功したことを確認します。

```powershell
databricks bundle summary
```

"予想される結果:" 出力に、デプロイされた **order-pipeline-job** とそれへの直接 **URL** が一覧表示されます。** ブラウザーでその URL を開き (またはワークスペースで **[ジョブとパイプライン]** に移動して)、次の点を確認します。
- ジョブは、前に `[dev <your-username>]` が付いた名前で表示される。
- ジョブを開くと、**validate-data** と **transform-data** の両方のタスクが表示される (**transform-data** はタスク グラフの **validate-data** によって異なる)。

> 🤖 **ヒント:** "宣言型オートメーション バンドルの開発モードでは、ジョブ名とスケジュールに対して何が行われますか?" と質問して、** ジョブにプレフィックスとしてユーザー名が付いている理由を理解します。

### ステップ 5 — クリーンアップする (省略可能)

デプロイされたリソースをワークスペースから削除するには、`order-pipeline-bundle` フォルダーから次の手順を実行します。

```powershell
databricks bundle destroy -t dev
```

メッセージが表示されたら、作成されたジョブを削除することを確認します。 "予想される結果:" リソースが破壊されたことが CLI から報告され、ジョブは **[ジョブとパイプライン]** に表示されなくなります。**

---

## まとめ

このラボでは:

- pytest フィクスチャを使って、個々の変換関数を検証する**単体テスト**を実装しました
- 変数、ジョブ リソース、複数環境のターゲットを含む**宣言型オートメーション バンドル**を構成しました
- **Databricks CLI** を使って、バンドルのデプロイの検証、計画、デプロイ、確認を行いました
