---
lab:
  index: 0
  title: Azure Databricks 環境を設定する
  module: Set up your Azure Databricks environment
  description: Azure Cloud Shell を使用して、Azure サブスクリプションで Azure Databricks Premium ワークスペースをプロビジョニングします。
  duration: 15 minutes
  level: 100
  islab: false
---

# Azure Databricks 環境を設定する

このコースのラボを開始する前に、**Azure Databricks Premium ワークスペース**をプロビジョニングする必要があります。 このセットアップ ラボでは、**Azure Cloud Shell** を使用してそのプロセスを実行する手順について説明します。そのため、ツールをローカルにインストールする必要はありません。

このラボは完了するまで、約 **15 分**かかります。

---

## Azure Databricks Premium ワークスペースをプロビジョニングする

Cloud Shell で 1 つの Azure CLI スクリプトを使用して、ランダムに選択された Azure リージョンにリソース グループと Azure Databricks Premium ワークスペースを作成します。

### タスク 1: Azure Cloud Shell を開く

1. 提供された資格情報を使用して、Azure portal (`https://portal.azure.com`) にサインインします。

2. Azure portal の上部にあるツール バーで、**Cloud Shell** ボタン (**[>_]**) を選択します。 メッセージが表示されたら、シェルの種類として **[Bash]** を選択します。

    > [!NOTE]
    > Cloud Shell ボタンが表示されない場合は、ブラウザー ウィンドウの幅が狭すぎる可能性があります。 ウィンドウを拡大するか、`https://shell.azure.com` に直接移動して、完全なブラウザー タブで Cloud Shell を開いてみてください。
    > 
    > ![[Cloud Shell] アイコン](Media/cloud-shell.png)
    
3. Cloud Shell を初めて使用する場合は、ストレージ アカウントを作成するように求められます。 **[ストレージ アカウントは必要ありません]** を選択し、ご自分のサブスクリプションを選択し、**[適用]** を選択します。

4. Cloud Shell プロンプトが表示されるまで待ちます。 次のようになります。

    ```
    yourname@Azure:~$
    ```

### タスク 2: プロビジョニング スクリプトを実行する

1. Cloud Shell で次のコマンドを実行して、セットアップ スクリプトをダウンロードして実行します。

    ```bash
    curl -sL https://raw.githubusercontent.com/MicrosoftLearning/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/refs/heads/main/Instructions/Labs/00-setup.sh | bash
    ```

2. デプロイが完了するまで待ちます。 これには約 **5 分**かかります。 

> [!NOTE]
> リージョンは、サポートされているパブリック Azure リージョンの一覧からランダムに選択されます。 ワークスペース名とリソース グループ名は固定されているため、以降のラボで簡単に見つけることができます。

### タスク 3: Azure Databricks ワークスペースを開く

1. Azure portal の上部にある検索バーで、**Azure Databricks** を検索します。

2. 一覧から **adb-dp750** ワークスペースを選択します。

3. ワークスペースの概要ページで、**[ワークスペースの起動]** を選択します。 Azure Databricks UI が新しいブラウザー タブで開きます。

4. Azure Databricks のホーム ページが表示されることを確認します。 これで、コースのラボを開始する準備ができました。

> [!IMPORTANT]
> **rg-dp750**リソース グループ名をメモしておいてください。 コース後リソースをクリーンアップする場合に必要になります。