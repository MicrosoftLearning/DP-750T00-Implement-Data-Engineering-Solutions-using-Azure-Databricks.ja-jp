---
lab:
  index: 0
  title: Azure Databricks 環境をクリーンアップする
  module: Clean up your Azure Databricks environment
  description: Azure Cloud Shell を使って Azure 内のリソースをクリーンアップします。
  duration: 5 minutes
  level: 100
  islab: false
---

# Azure Databricks 環境をクリーンアップする

このコースのすべてのラボを終えたら、Azure で不要な料金がかからないよう、作成したリソースを削除する必要があります。 このクリーンアップ ラボでは、リソース グループとその中のすべてのリソースを削除します。

このクリーンアップを終えるには、約 **5 分**かかります。

---

## リソース グループとすべてのリソースを削除する

リソース グループを削除すると、Azure Databricks ワークスペースと、ラボの間にその中に作成された他のすべてのリソースが削除されます。

### タスク 1: Azure Cloud Shell を開く

1. 提供された資格情報を使用して、Azure portal (`https://portal.azure.com`) にサインインします。

2. Azure portal の上部にあるツール バーで、**Cloud Shell** ボタン (**[>_]**) を選択します。 メッセージが表示されたら、シェルの種類として **[Bash]** を選択します。

3. Cloud Shell プロンプトが表示されるまで待ちます。

### タスク 2: リソース グループを削除する

1. Cloud Shell で次のコマンドを実行し、クリーンアップ スクリプトをダウンロードして実行します。

    ```bash
    curl -sL https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Instructions/Labs/99-cleanup.sh | bash
    ```

2. このスクリプトは、リソース グループを非同期的に削除します。 コマンドを実行したら、Cloud Shell を閉じてかまいません。

> [!NOTE]
> これにより、**rg-dp750** リソース グループと、**adb-dp750** Azure Databricks ワークスペースおよびラボの間にグループに追加した他のすべてのリソースが、完全に削除されます。 この操作を元に戻すことはできません。

