# Azureハンズオン: CICDやってみよう
2023年3月3日　Azureのリソースへの自動リリース
## 手順まとめ
### 1.「 カスタムテンプレートのデプロイ」からテンプレート作って指定したリソースグループ配下にリソースを作成

1. Azureの画面から「カスタムテンプレートのデプロイ」を検索する。
![image](https://user-images.githubusercontent.com/69332106/222651449-2b3a3caa-3465-4502-894a-834eacd84886.png)

2. 以下の値を設定。

  |  項目  |  値  |
  | ---- | ---- |   
  | サブスクリプション * | サブスクリプション指定 |
  | リージョン * | リージョン（Ex: Japan East） |
  | Resource Group Name * | 作成するリソースグループ名 |
  | Arc Name Prefix * | 一意になる文字列 (**文字数は 13 文字以内**) ※コンテナーレジストリの名前に用いられる |
  | Location | 変更不要 |

3. 進み、作成ボタン押すと、リソースグループ配下にリソースが作られる。
![image](https://user-images.githubusercontent.com/69332106/222656467-6aa405fb-1186-4f8f-920f-4db6893b1393.png)

### 2. .github/workflows/build-deploy.ymlを編集

```
name: Trigger auto deployment
on:
  # 手動実行（ビルド・デプロイ）用トリガー
  workflow_dispatch:      
# Add env
env:
  CONTAINER_REGISTRY: ryumanmf37qiiyxpvb4.azurecr.io # Azure Container Registryのドメイン(概要のログインサーバー)(~.azurecr.io) を設定。レジストリにdockerイメージを保管。
  # Add envs
  RESOURCE_GROUP_NAME: RS-Handson01-CICD # 準備タスクで作成されたリソースグループ名をここに記載
  CONTAINERAPP_NAME: ctapp-demo-api # 準備タスクで作成された API Container App 名をここに記載
jobs:
  build: # Githubアクションのbuild上
    runs-on: ubuntu-22.04
    steps: # Githubアクションのbuild上でのステップ
      - name: Checkout to the branch
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Set repository name to env
        run: | 
          echo "REPOSITORY_NAME=${GITHUB_REPOSITORY#${GITHUB_REPOSITORY_OWNER}/}" >> $GITHUB_ENV
      - name: Confirm if env have REPOSITORY_NAME
        run: |
          echo ${{ env.REPOSITORY_NAME }}
# コンテナーレジストリの接続・認証情報を追加
      - name: Log in to container registry
        # 認証処理を行うためのアクション
        uses: docker/login-action@v2
        with:
          registry: ${{ env.CONTAINER_REGISTRY }}
          username: ${{ secrets.CONTAINER_REGISTRY_USERNAME }}
          password: ${{ secrets.CONTAINER_REGISTRY_PASSWORD }}
# Add a step
      - name: Build and push container image to registry
        uses: docker/build-push-action@v3
        with:
          push: true
          tags: ${{ env.CONTAINER_REGISTRY }}/${{ env.REPOSITORY_NAME }}:${{ github.sha }}
          file: ./Dockerfile
          context: ./
# Add a job and steps
  deploy: # Githubアクションのdeploy上
    runs-on: ubuntu-22.04
    needs: build
  # Add a id-token permission
    permissions:
      id-token: write
    steps:
      - name: Set repository name to env
        run: | 
          echo "REPOSITORY_NAME=${GITHUB_REPOSITORY#${GITHUB_REPOSITORY_OWNER}/}" >> $GITHUB_ENV

      - name: Confirm if env have REPOSITORY_NAME
        run: |
          echo ${{ env.REPOSITORY_NAME }}
# Add a step
      - name: Azure Login using OIDC
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
# Add a step
      - name: Deploy to containerapp
        uses: azure/CLI@v1
        with:
          inlineScript: |
            az extension add --upgrade --name containerapp

            az containerapp registry set \
              --name ${{ env.CONTAINERAPP_NAME }} \
              --resource-group ${{ env.RESOURCE_GROUP_NAME }} \
              --server ${{ env.CONTAINER_REGISTRY }} \
              --username  ${{ secrets.CONTAINER_REGISTRY_USERNAME }} \
              --password ${{ secrets.CONTAINER_REGISTRY_PASSWORD }}

            az containerapp ingress enable \
              --name ${{ env.CONTAINERAPP_NAME }} \
              --resource-group ${{ env.RESOURCE_GROUP_NAME }} \
              --target-port 3500 \
              --type external

            container_name=$( \
              az containerapp show \
                --name ${{ env.CONTAINERAPP_NAME }} \
                --resource-group ${{ env.RESOURCE_GROUP_NAME }} \
                --query "properties.template.containers[0].name" \
                --output tsv
            )
            az containerapp update \
              --name ${{ env.CONTAINERAPP_NAME }} \
              --resource-group ${{ env.RESOURCE_GROUP_NAME }} \
              --container-name $container_name \
              --image ${{ env.CONTAINER_REGISTRY }}/${{ env.REPOSITORY_NAME }}:${{ github.sha }}
 ```

Githubアクションの見方
![image](https://user-images.githubusercontent.com/69332106/222660203-65be8bb6-d791-44d0-85b1-2c9d533eca28.png)
![image](https://user-images.githubusercontent.com/69332106/222660725-843b5859-f6d1-4e35-8528-fe0a023a9429.png)
![image](https://user-images.githubusercontent.com/69332106/222660797-81d2e46e-e217-4076-93e7-2df4def8113e.png)

# Azure Container Apps Album API

This is the companion repository for the [Azure Container Apps code-to-cloud quickstart]().

This backend Album API sample is available in other languages:

| [C#](https://github.com/azure-samples/containerapps-albumapi-csharp) | [Go](https://github.com/azure-samples/containerapps-albumapi-go) | [Python](https://github.com/azure-samples/containerapps-albumapi-python) |
| -------------------------------------------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------ |
