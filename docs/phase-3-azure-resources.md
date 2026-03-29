# Phase 3: Azure リソースの作成

> **目的**: Azure 上に Static Web Apps リソースを作成し、GitHub リポジトリと接続する。  
> **所要時間目安**: 20〜30 分  
> **前提**: [Phase 2](phase-2-github-setup.md) が完了していること

---

## 目次

1. [Azure サブスクリプションの確認](#1-azure-サブスクリプションの確認)
2. [リソースグループの作成](#2-リソースグループの作成)
3. [Azure Static Web Apps リソースの作成](#3-azure-static-web-apps-リソースの作成)
4. [リソースの確認](#4-リソースの確認)
5. [デプロイトークンの取得](#5-デプロイトークンの取得)
6. [VS Code からの確認](#6-vs-code-からの確認)
7. [完了チェックリスト](#7-完了チェックリスト)

---

## 1. Azure サブスクリプションの確認

### 1.1 サインイン状態の確認

```bash
az account show --query '{name:name, id:id, state:state}' -o table
```

サインインしていない場合:

```bash
# WSL の場合はデバイスコードフローが便利
az login --use-device-code
```

### 1.2 サブスクリプションの選択

複数のサブスクリプションがある場合、使用するサブスクリプションを選択します。

```bash
# サブスクリプション一覧
az account list --query '[].{name:name, id:id, isDefault:isDefault}' -o table

# 使用するサブスクリプションを設定
az account set --subscription "<SUBSCRIPTION_ID>"
```

---

## 2. リソースグループの作成

Azure リソースを論理的にまとめるリソースグループを作成します。

```bash
az group create \
  --name rg-astro-swa \
  --location japaneast
```

出力例:
```json
{
  "id": "/subscriptions/xxxx/resourceGroups/rg-astro-swa",
  "location": "japaneast",
  "name": "rg-astro-swa",
  "properties": {
    "provisioningState": "Succeeded"
  }
}
```

> **リージョンの選択**: 本ハンズオンでは `japaneast` (東日本) を使用しますが、SWA は以下のリージョンで利用可能です:
> - `centralus`, `eastus2`, `eastasia`, `westeurope`, `westus2` など
> - `japaneast` は SWA のホスティングに対応しているリージョンです

---

## 3. Azure Static Web Apps リソースの作成

### 3.1 プランの選択

| プラン | 月額コスト | Managed Identity | Key Vault 連携 | Bring Your Own API | SLA |
|---|---|---|---|---|---|
| **Free** | 無料 | ❌ | ❌ | ❌ | なし |
| **Standard** | 約 $9/アプリ | ✅ | ✅ | ✅ | 99.95% |

> **本ハンズオンでは Standard プランを使用します。** Phase 4 の Managed Identity と Key Vault 連携は Standard プランでのみ利用可能です。

### 3.2 SWA リソースの作成

```bash
az staticwebapp create \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --location japaneast \
  --sku Standard \
  --source https://github.com/<YOUR_USERNAME>/astro_azure_swa \
  --branch main \
  --app-location "/" \
  --output-location "dist" \
  --login-with-github
```

> **`<YOUR_USERNAME>`** を自分の GitHub ユーザー名に置き換えてください。

各パラメータの意味:

| パラメータ | 値 | 説明 |
|---|---|---|
| `--name` | `swa-astro-site` | SWA リソースの名前 (Azure 内で一意) |
| `--resource-group` | `rg-astro-swa` | リソースグループ名 |
| `--location` | `japaneast` | リソースのリージョン |
| `--sku` | `Standard` | プラン (Free or Standard) |
| `--source` | GitHub リポジトリ URL | ソースコードの場所 |
| `--branch` | `main` | デプロイ対象のブランチ |
| `--app-location` | `"/"` | Astro プロジェクトのルート |
| `--output-location` | `"dist"` | Astro のビルド出力ディレクトリ |
| `--login-with-github` | — | GitHub OAuth でリポジトリアクセスを認可 |

### 3.3 GitHub 認可のフロー

`--login-with-github` を指定すると:

1. ブラウザが開き、GitHub OAuth の認可画面が表示される
2. Azure Static Web Apps に対してリポジトリへのアクセスを許可する
3. 認可後、以下が自動的に行われる:
   - GitHub リポジトリに `AZURE_STATIC_WEB_APPS_API_TOKEN` シークレットが自動登録される
   - `.github/workflows/` に GitHub Actions ワークフローファイルが自動生成される

> **WSL で ブラウザが開かない場合**: 表示された URL を手動でブラウザに貼り付けてください。

### 3.4 自動生成されたワークフローの確認

SWA 作成時に GitHub リポジトリに自動的に追加されたワークフローファイルを確認します。

```bash
# リモートの変更を pull
git pull origin main

# 生成されたワークフローファイルを確認
ls .github/workflows/
cat .github/workflows/azure-static-web-apps-*.yml
```

> **注意**: 自動生成されたワークフローファイルは Phase 5 でカスタマイズします。現時点ではそのままで構いません。

---

## 4. リソースの確認

### 4.1 Azure CLI で確認

```bash
# SWA リソースの詳細
az staticwebapp show \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --query '{name:name, sku:sku.name, defaultHostname:defaultHostname, state:state}' \
  -o table
```

出力例:
```
Name             Sku       DefaultHostname                             State
---------------  --------  ------------------------------------------  -------
swa-astro-site   Standard  gentle-water-0a1b2c3d4.japaneast.azurestaticapps.net  Ready
```

### 4.2 デフォルト URL にアクセス

```bash
# デフォルト URL を取得
DEFAULT_URL=$(az staticwebapp show \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --query "defaultHostname" -o tsv)

echo "https://$DEFAULT_URL"
```

> GitHub Actions のワークフローが初回実行されていれば、この URL でサイトにアクセスできます。まだの場合は、次の push でデプロイされます。

### 4.3 リソースグループ内のリソース一覧

```bash
az resource list \
  --resource-group rg-astro-swa \
  --query '[].{name:name, type:type, location:location}' \
  -o table
```

---

## 5. デプロイトークンの取得

GitHub Actions で使用するデプロイトークンを取得します。

```bash
az staticwebapp secrets list \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --query "properties.apiKey" -o tsv
```

> **注意**: このトークンは Phase 5 で OIDC 方式と併用します。SWA 作成時に `--login-with-github` を使用した場合、GitHub Secrets には自動登録済みです。

### 5.1 GitHub Secrets の確認

```bash
# GitHub CLI で Secrets の一覧を確認 (値は表示されない)
gh secret list
```

`AZURE_STATIC_WEB_APPS_API_TOKEN` が登録されていることを確認します。

---

## 6. VS Code からの確認

### 6.1 Azure 拡張機能でサインイン

1. VS Code のアクティビティバー（左サイドバー）で **Azure** アイコンをクリック
2. **Sign in to Azure** をクリック
3. ブラウザで Azure アカウントにサインイン

### 6.2 SWA リソースの確認

1. Azure サイドバーで **Static Web Apps** セクションを展開
2. サブスクリプションを展開
3. `swa-astro-site` が表示されることを確認

VS Code の Azure Static Web Apps 拡張機能からは以下の操作が可能です:
- リソースの詳細表示
- 環境変数の確認・設定
- ログの確認
- ブラウザでサイトを開く

---

## 7. 完了チェックリスト

以下がすべて ✅ になっていれば、Phase 3 は完了です。

| # | チェック項目 | 確認方法 |
|---|---|---|
| 1 | リソースグループ `rg-astro-swa` が作成されている | `az group show --name rg-astro-swa` |
| 2 | SWA リソース `swa-astro-site` が作成されている | `az staticwebapp show --name swa-astro-site -g rg-astro-swa` |
| 3 | SKU が Standard である | 上記コマンドの `sku.name` が `Standard` |
| 4 | GitHub リポジトリと接続されている | `az staticwebapp show` で `repositoryUrl` を確認 |
| 5 | GitHub Actions ワークフローが自動生成されている | `ls .github/workflows/` |
| 6 | GitHub Secrets にデプロイトークンが登録されている | `gh secret list` |
| 7 | VS Code の Azure 拡張機能から SWA リソースが確認できる | Azure サイドバーで確認 |
| 8 | デフォルト URL にアクセスできる（初回デプロイ完了後） | ブラウザでアクセス |

---

**前のステップ**: [Phase 2: GitHub リポジトリのセットアップ](phase-2-github-setup.md)  
**次のステップ**: [Phase 4: セキュリティ設定 (Managed Identity)](phase-4-security.md)
