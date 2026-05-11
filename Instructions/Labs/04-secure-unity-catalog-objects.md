---
lab:
  index: 4
  title: Unity カタログ オブジェクトをセキュリティで保護する
  module: Secure Unity Catalog objects
  module-url: 'https://learn.microsoft.com/training/wwl-databricks/secure-unity-catalog-objects/'
  notebook: 'https://github.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/blob/main/Allfiles/04-secure-unity-catalog-objects.ipynb'
  description: このラボでは、Azure Databricks の Unity カタログ オブジェクトをセキュリティで保護するために、Databricks グループにきめ細かなアクセス制御を付与し、行フィルターを適用して地域別に顧客データを制限し、列マスク関数を使用して PII メール アドレスをマスクします。 また、Azure Key Vault でサポートされるシークレット スコープを作成し、ノートブック内でシークレットを安全に取得します。これにより、機密性の高い資格情報がコードで公開されることはありません。
  duration: 45 minutes
  level: 300
  islab: true
  primarytopics:
    - Azure Databricks
---

# ラボ 04: Unity カタログ オブジェクトをセキュリティで保護する

## シナリオ

あなたは **NorthMart Retail** のデータ エンジニアです。同社は、北部、南部、東部、西部の 4 つの地域で運営されている全国的なスーパーマーケット チェーンです。 チームは Azure Databricks の一元化されたデータ プラットフォームを管理しており、これには顧客データ、ロイヤルティ プログラム レコード、地域の販売トランザクションが含まれています。

セキュリティ チームから、いくつかの懸案が提起されています。

- 地域アナリストは、自分の担当地域のデータのみを閲覧でき、他の地域の顧客レコードを閲覧できないようにする必要がある。
- 顧客のメール アドレスは個人を特定できる情報 (PII) であり、ほとんどのユーザーに対してマスクする必要がある。
- サードパーティのロイヤルティ プラットフォームを統合するには API キーが必要であるが、そのキーをノートブックやコード内に保存してはならない。

このラボでは、**アクセス制御**、**行のフィルター処理**、**列のマスキング**、**Azure Key Vault でサポートされるシークレット** を Azure Databricks Unity Catalog に実装して、これら 3 つの懸案を解決します。

## 目標

このラボを終えるまでに、次のことを行います。

- SQL を使用して、Databricks グループにスキーマ レベルのアクセス許可を付与して検証する。
- 行フィルター関数を適用して、顧客レコードを地域別に制限する。
- 列マスクを適用して PII メール データを保護する。
- Azure Key Vault を作成してシークレットを保存する。
- Azure Databricks で、Azure Key Vault でサポートされるシークレット スコープを作成する。
- ノートブック内でシークレットを安全に取得する。

このラボは完了するまで、約 **35 から 40 分**かかります。

---

## 🤖 このラボ全体で Genie Code を使用する

このラボでは、**Genie Code** を常に使用することが**想定されており、そうすることをおすすめします**。 ノートブック内のすべての演習セルには、Genie Code パネルに直接貼り付けることができる推奨プロンプトが含まれています。

Genie Code を開くには、次を選択します ![アシスタント アイコン](https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/media/genie-code.svg) をノートブック セルの右側でクリックするか、ツール バーに表示されているキーボード ショートカットを押します。

> 💡**ヒント:** Genie Code の出力を無条件にコピーして貼り付けることはしないでください。 それを読み、理解し、タスクの要件に合わせて調整してください。 Genie Code は思考を加速させるためのツールであり、あなたの考えに置き換わるものではありません。

---
'Before starting this lab, ensure you have':
  - 'An **Azure Databricks Premium workspace** provisioned using [Lab 00': 'Set up your Azure Databricks environment](00-setup.md).'
  - An active **Unity Catalog metastore** attached to the workspace.
  - The **CREATE CATALOG** privilege on the metastore.
  - An **Azure subscription** where you can create a Key Vault.
  - 'Familiarity with basic SQL (CREATE TABLE, SELECT, GRANT).'
---

## ラボのノートブックをインポートする

1. Azure Databricks ワークスペースで、左側のサイド バーにある **[ワークスペース]** を選択します。
2. このラボを保存するフォルダーに移動するか、そのフォルダーを作成します。
3. フォルダーの横にある **[⋮]** (ケバブ) メニューを選択し、**[インポート]** を選択します。
4. **[URL]** を選択し、URL (`https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Allfiles/04-secure-unity-catalog-objects.ipynb`) を入力し、**[インポート]** を選択します。
5. インポートしたノートブックを開き、上部にあるコンピューティング セレクターで **[サーバーレス]** コンピューティングを選択します。

---

## ノートブックを開く前: Databricks グループを作成する

演習 1 で Databricks グループにアクセス許可を付与します。 その演習に取り掛かるときにすぐに使用できるように、ここでグループを作成しておきます。

1. Databricks ワークスペースで、**ユーザー名** (右上) → **[設定]** の順に選択します。
2. **[ID およびアクセス管理]** → **[グループ]** の順に移動し、**[管理]** → **[グループの追加]** の順に選択します。
3. グループに `retail-analysts` という名前を付け、**[保存]** を選択します。
4. グループが作成されたら、メンバーとして自分のユーザー アカウントを追加します。

> ℹ️ このグループは、NorthMart Retail の地域アナリスト チームを表します。 演習 1 で、このチームにラボのスキーマへのアクセス権を付与します。

---

## 演習 4 の前: Azure Key Vault を作成する

演習 4 では、事前に作成されたシークレットを含む Azure Key Vault が必要です。 ノートブックで演習 4 に取り掛かる前に Azure portal で以下の手順を完了するか、並行して準備します。

### 手順 1: Key Vault を作成する

1. [Azure portal](https://portal.azure.com) を開き、**[リソースの作成]** を選択します。
2. **[Key Vault]** を検索し、**[作成]** を選択します。
3. Key Vault を次のように構成します。
   - **リソース グループ**: ラボのリソース グループを使用します。
   - **キー コンテナー名**: `kv-northmart-<your-initials>` (グローバルに一意である必要があります)。
   - **地域**: Databricks ワークスペースと同じリージョン。
   - **価格レベル**: Standard。
4. **[アクセス構成]** タブで、**[アクセス許可モデル]** を **[コンテナーのアクセス ポリシー]** に変更します。
5. **[確認および作成]**、**[作成]** の順に選択します。

### 手順 2: ユーザーのアクセス ポリシーを追加する

1. Key Vault がデプロイされたら、ポータルで開きます。
2. **[アクセス ポリシー]** → **[作成]** の順に選択します。
3. **シークレット アクセス許可**で、 **取得** と **リスト**を選択します。
4. **[プリンシパル]** で、自分の Azure ユーザー アカウントを検索して選択します。
5. **[作成]** を選択してポリシーを保存します。

### 手順 3: シークレットを追加する

1. Key Vault で、**[シークレット]** → **[生成/インポート]** の順に選択します。
2. 次のように設定します。
   - **名前**: `loyalty-api-key`
   - **値**: `NORTHMART-LOYALTY-2026-SECURE`
3. **［作成］** を選択します

### 手順 4: Key Vault の詳細をメモする

Key Vault を終了する前に、**[プロパティ]** に移動し、次の情報をコピーします。
- **コンテナー URI** (DNS 名)。例: *https://kv-northmart-abc.vault.azure.net/*
- **リソース ID**。例: */subscriptions/xxxxxxxx/resourceGroups/rg-lab/providers/Microsoft.KeyVault/vaults/kv-northmart-abc*

演習 4 で Databricks シークレット スコープを作成する際に、両方の値が必要になります。

### ステップ 5: Databricks のシークレット スコープを作成する

1. ブラウザーで次の URL に移動します。

    ```
    https://<your-databricks-workspace-url>#secrets/createScope
    ```

    > ⚠️ **createScope** の **S** は大文字にする必要があります。 `<your-databricks-workspace-url>` を実際のワークスペース URL に置き換えてください (末尾の '/' は省略します)。

2. スコープを構成します:
   - **スコープ名**: `retail-kv-scope`
   - **プリンシパルの管理**: `All workspace users`
   - **DNS 名**: 手順 4 のコンテナー URI を貼り付けます。
   - **リソース ID**: 手順 4 のリソース ID を貼り付けます。
3. **［作成］** を選択します

> ✅**予想される結果:** スコープが作成されたことを示す確認メッセージが表示されます。 これでスコープが Azure Key Vault にリンクされ、そこに追加するすべてのシークレットに Databricks ノートブックからアクセスできます。

---

これで、ノートブックを開いて演習を行う準備ができました。
