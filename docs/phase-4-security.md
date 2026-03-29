# Phase 4: セキュリティ設定 (Managed Identity)

> **目的**: Managed Identity を活用して、シークレットの安全な管理と、GitHub Actions からのセキュアなデプロイ (OIDC) を実現する。  
> **所要時間目安**: 40〜60 分  
> **前提**: [Phase 3](phase-3-azure-resources.md) が完了していること。SWA が Standard プランであること。

---

## 目次

1. [セキュリティ設計の概要](#1-セキュリティ設計の概要)
2. [System-assigned Managed Identity の有効化](#2-system-assigned-managed-identity-の有効化)
3. [Azure Key Vault の作成とシークレット登録](#3-azure-key-vault-の作成とシークレット登録)
4. [Key Vault アクセス権限の設定](#4-key-vault-アクセス権限の設定)
5. [SWA Application Settings での Key Vault 参照](#5-swa-application-settings-での-key-vault-参照)
6. [GitHub Actions OIDC 連携の設定](#6-github-actions-oidc-連携の設定)
7. [staticwebapp.config.json の作成](#7-staticwebappconfigjson-の作成)
8. [完了チェックリスト](#8-完了チェックリスト)

---

## 1. セキュリティ設計の概要

本 Phase で設定するセキュリティ構成は以下の 3 つの柱で構成されます。

### 1.1 全体像

```
┌─────────────────────────────────────────────────────┐
│                  セキュリティ設計                      │
├─────────────────┬─────────────────┬─────────────────┤
│  シークレット管理  │  デプロイ認証     │  ユーザー認証     │
│                 │                 │                 │
│  Key Vault      │  OIDC           │  SWA 組み込み    │
│  + Managed ID   │  + Federated    │  + config.json  │
│                 │    Credential   │                 │
│  パスワードレス   │  トークンレス     │  コードレス      │
│  アクセス        │  デプロイ        │  認可            │
└─────────────────┴─────────────────┴─────────────────┘
```

### 1.2 なぜ Managed Identity を使うのか

| 従来の方法 | 問題点 | Managed Identity の解決策 |
|---|---|---|
| 接続文字列を環境変数に直書き | シークレットの漏洩リスク | Key Vault 参照で値を隠蔽 |
| サービスプリンシパルの資格情報管理 | クレデンシャルのローテーション負荷 | Azure が自動管理、ローテーション不要 |
| Key Vault アクセス用のクライアントシークレット | "シークレットにアクセスするためのシークレット" 問題 | Managed Identity で直接認証 |

---

## 2. System-assigned Managed Identity の有効化

SWA リソースに System-assigned Managed Identity を有効化し、Azure リソースへのパスワードレスアクセスを可能にします。

### 2.1 Azure CLI で有効化

```bash
az staticwebapp identity assign \
  --name swa-astro-site \
  --resource-group rg-astro-swa
```

出力例:
```json
{
  "principalId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenantId": "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy",
  "type": "SystemAssigned"
}
```

### 2.2 Managed Identity の確認

```bash
az staticwebapp identity show \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --query '{principalId:principalId, tenantId:tenantId, type:type}' \
  -o table
```

> **`principalId`** の値をメモしてください。後続のステップで Key Vault のアクセス権限設定に使用します。

### 2.3 Azure Portal での有効化（代替手順）

1. [Azure Portal](https://portal.azure.com) で SWA リソース `swa-astro-site` を開く
2. 左メニューの **設定** → **ID (Identity)** をクリック
3. **システム割り当て済み** タブで **状態** を **オン** に変更
4. **保存** → 確認ダイアログで **はい** をクリック

---

## 3. Azure Key Vault の作成とシークレット登録

### 3.1 Key Vault の作成

```bash
az keyvault create \
  --name kv-astro-swa \
  --resource-group rg-astro-swa \
  --location japaneast \
  --sku standard \
  --enable-rbac-authorization true
```

> **`--enable-rbac-authorization true`**: Azure RBAC によるアクセス制御を有効にします。従来のアクセスポリシーよりも細かい権限管理が可能です。

> **名前の制約**: Key Vault 名は Azure 全体で一意である必要があります。`kv-astro-swa` が使用できない場合は、`kv-astro-swa-<ユニークな接尾辞>` のように変更してください。

### 3.2 自分自身に Key Vault 管理者権限を付与

Key Vault に RBAC 認可を有効にした場合、まず自分のアカウントにシークレット管理権限を付与します。

```bash
# 自分の Object ID を取得
MY_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)

# Key Vault の リソース ID を取得
KV_ID=$(az keyvault show --name kv-astro-swa --query id -o tsv)

# Key Vault Administrator ロールを付与
az role assignment create \
  --role "Key Vault Administrator" \
  --assignee "$MY_OBJECT_ID" \
  --scope "$KV_ID"
```

### 3.3 シークレットの登録

```bash
az keyvault secret set \
  --vault-name kv-astro-swa \
  --name "ExampleSecret" \
  --value "my-secret-value-12345"
```

> **実際のプロジェクトでは**: API キー、外部サービスの接続文字列、認証プロバイダのクライアントシークレットなどを格納します。

### 3.4 シークレットの確認

```bash
# シークレット一覧
az keyvault secret list \
  --vault-name kv-astro-swa \
  --query '[].{name:name}' -o table

# シークレット値の取得（確認用）
az keyvault secret show \
  --vault-name kv-astro-swa \
  --name "ExampleSecret" \
  --query "value" -o tsv
```

---

## 4. Key Vault アクセス権限の設定

SWA の Managed Identity に Key Vault のシークレット読み取り権限を付与します。

### 4.1 RBAC でアクセス権限を付与

```bash
# SWA の Managed Identity の Principal ID を取得
PRINCIPAL_ID=$(az staticwebapp identity show \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --query "principalId" -o tsv)

# Key Vault の リソース ID を取得
KV_ID=$(az keyvault show --name kv-astro-swa --query id -o tsv)

# Key Vault Secrets User ロールを付与 (読み取りのみ)
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee "$PRINCIPAL_ID" \
  --scope "$KV_ID"
```

### 4.2 ロール割り当ての確認

```bash
az role assignment list \
  --assignee "$PRINCIPAL_ID" \
  --scope "$KV_ID" \
  --query '[].{role:roleDefinitionName, scope:scope}' \
  -o table
```

出力例:
```
Role                    Scope
----------------------  -----------------------------------------------
Key Vault Secrets User  /subscriptions/.../resourceGroups/rg-astro-swa/providers/Microsoft.KeyVault/vaults/kv-astro-swa
```

### 4.3 最小権限の原則

| ロール | 権限 | 用途 |
|---|---|---|
| `Key Vault Secrets User` | シークレットの読み取りのみ | ✅ SWA に付与 (最小権限) |
| `Key Vault Secrets Officer` | シークレットの読み書き | ❌ SWA には不要 |
| `Key Vault Administrator` | 全操作 | 管理者のみ |

> **セキュリティのポイント**: SWA にはシークレットの読み取り権限 (`Key Vault Secrets User`) のみを付与します。書き込みや削除の権限は付与しません。

---

## 5. SWA Application Settings での Key Vault 参照

### 5.1 Key Vault 参照の構文

SWA の Application Settings で以下の構文を使うと、Key Vault のシークレットを安全に参照できます。

```
@Microsoft.KeyVault(VaultName=<VAULT_NAME>;SecretName=<SECRET_NAME>)
```

### 5.2 Azure CLI で Application Settings を設定

```bash
az staticwebapp appsettings set \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --setting-names \
    "EXAMPLE_SECRET=@Microsoft.KeyVault(VaultName=kv-astro-swa;SecretName=ExampleSecret)"
```

### 5.3 設定の確認

```bash
az staticwebapp appsettings list \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --query '{settings:properties}' -o json
```

> **注意**: Key Vault 参照が正しく解決されている場合、出力にはシークレットの実際の値は表示されません（参照文字列のみ表示されます）。

### 5.4 Azure Portal での設定（代替手順）

1. SWA リソース `swa-astro-site` を開く
2. **設定** → **構成** をクリック
3. **アプリケーション設定** の **追加** をクリック
4. 名前: `EXAMPLE_SECRET`
5. 値: `@Microsoft.KeyVault(VaultName=kv-astro-swa;SecretName=ExampleSecret)`
6. **OK** → **保存**

---

## 6. GitHub Actions OIDC 連携の設定

デプロイ時に長期間有効なデプロイトークンの代わりに、OIDC (OpenID Connect) Federated Credential を使用してよりセキュアにデプロイします。

### 6.1 OIDC の仕組み

```
GitHub Actions
  │
  │ 1. OIDC ID Token を発行
  │    (短命 — ジョブ実行中のみ有効)
  ▼
Microsoft Entra ID
  │
  │ 2. Federated Credential で
  │    GitHub のトークン発行者を信頼
  │    subject (リポジトリ・ブランチ) を検証
  ▼
Azure Static Web Apps
  │
  │ 3. 検証済みトークンでデプロイを許可
  ▼
デプロイ完了
```

### 6.2 Entra ID でアプリケーション登録

```bash
# アプリケーションを登録
az ad app create --display-name "github-actions-astro-swa"

# アプリケーション ID を取得
APP_ID=$(az ad app list \
  --display-name "github-actions-astro-swa" \
  --query "[0].appId" -o tsv)

echo "Application (Client) ID: $APP_ID"

# サービスプリンシパルの作成
az ad sp create --id "$APP_ID"
```

### 6.3 Federated Credential の作成

GitHub Actions がトークンを発行できるように、信頼関係を構成します。

```bash
# main ブランチ用の Federated Credential
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "github-actions-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<YOUR_USERNAME>/astro_azure_swa:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"],
    "description": "GitHub Actions - main branch deployment"
  }'
```

> **`<YOUR_USERNAME>`** を自分の GitHub ユーザー名に置き換えてください。

PR 用のステージング環境デプロイにも対応する場合:

```bash
# PR 用の Federated Credential
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "github-actions-pr",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<YOUR_USERNAME>/astro_azure_swa:pull_request",
    "audiences": ["api://AzureADTokenExchange"],
    "description": "GitHub Actions - pull request staging"
  }'
```

### 6.4 Federated Credential の確認

```bash
az ad app federated-credential list \
  --id "$APP_ID" \
  --query '[].{name:name, subject:subject, issuer:issuer}' \
  -o table
```

### 6.5 GitHub リポジトリにシークレットを登録

OIDC 認証に必要な情報を GitHub Secrets に登録します。

```bash
# テナント ID を取得
TENANT_ID=$(az account show --query tenantId -o tsv)

# サブスクリプション ID を取得
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# GitHub Secrets に登録
gh secret set AZURE_CLIENT_ID --body "$APP_ID"
gh secret set AZURE_TENANT_ID --body "$TENANT_ID"
gh secret set AZURE_SUBSCRIPTION_ID --body "$SUBSCRIPTION_ID"
```

### 6.6 登録されたシークレットの確認

```bash
gh secret list
```

出力例:
```
AZURE_CLIENT_ID                   Updated 2026-03-29
AZURE_STATIC_WEB_APPS_API_TOKEN   Updated 2026-03-29
AZURE_SUBSCRIPTION_ID             Updated 2026-03-29
AZURE_TENANT_ID                   Updated 2026-03-29
```

---

## 7. staticwebapp.config.json の作成

SWA の認証・認可・ルーティングを設定するファイルをプロジェクトルートに作成します。

### 7.1 設定ファイルの作成

```bash
cat > staticwebapp.config.json << 'EOF'
{
  "routes": [
    {
      "route": "/login",
      "redirect": "/.auth/login/github"
    },
    {
      "route": "/logout",
      "redirect": "/.auth/logout"
    },
    {
      "route": "/admin/*",
      "allowedRoles": ["authenticated"]
    }
  ],
  "responseOverrides": {
    "401": {
      "redirect": "/.auth/login/github?post_login_redirect_uri=.referrer",
      "statusCode": 302
    }
  },
  "navigationFallback": {
    "rewrite": "/index.html"
  }
}
EOF
```

### 7.2 設定項目の説明

#### ルート設定

| ルート | 動作 | 説明 |
|---|---|---|
| `/login` | `/.auth/login/github` にリダイレクト | ユーザーフレンドリーなログイン URL |
| `/logout` | `/.auth/logout` にリダイレクト | ログアウト |
| `/admin/*` | `authenticated` ロール必須 | 管理ページへのアクセス制限 |

#### レスポンスオーバーライド

| ステータス | 動作 | 説明 |
|---|---|---|
| `401` | GitHub ログインにリダイレクト | 未認証ユーザーのアクセス時、自動でログイン画面に誘導 |

> `post_login_redirect_uri=.referrer` により、ログイン後に元のページに戻ります。

#### ナビゲーションフォールバック

- SPA っぽい動作が必要な場合、存在しないパスを `index.html` に書き換え
- Astro の SSG では基本的に各ページに HTML が生成されるため、必須ではないがあると安心

### 7.3 不要な認証プロバイダをブロックする（オプション）

GitHub 認証のみを許可し、Entra ID (aad) をブロックする場合:

```json
{
  "routes": [
    {
      "route": "/.auth/login/aad",
      "statusCode": 404
    }
  ]
}
```

> 上記を `staticwebapp.config.json` の `routes` 配列に追加します。

### 7.4 変更をコミット

```bash
git add staticwebapp.config.json
git commit -m "Add staticwebapp.config.json for auth and routing"
```

> **まだ push しないでください。** Phase 5 で GitHub Actions ワークフローを構成してからまとめて push します。

---

## 8. 完了チェックリスト

以下がすべて ✅ になっていれば、Phase 4 は完了です。

| # | チェック項目 | 確認方法 |
|---|---|---|
| 1 | SWA で System-assigned Managed Identity が有効 | `az staticwebapp identity show --name swa-astro-site -g rg-astro-swa` |
| 2 | Key Vault `kv-astro-swa` が作成されている | `az keyvault show --name kv-astro-swa` |
| 3 | Key Vault にシークレットが登録されている | `az keyvault secret list --vault-name kv-astro-swa` |
| 4 | Key Vault の RBAC で SWA の MI に読み取り権限が付与されている | `az role assignment list --assignee <PRINCIPAL_ID>` |
| 5 | SWA の App Settings に Key Vault 参照が設定されている | `az staticwebapp appsettings list --name swa-astro-site -g rg-astro-swa` |
| 6 | Entra ID でアプリ登録が作成されている | `az ad app list --display-name github-actions-astro-swa` |
| 7 | Federated Credential が構成されている (main + PR) | `az ad app federated-credential list --id $APP_ID` |
| 8 | GitHub Secrets に OIDC 用の値が登録されている | `gh secret list` に 4 つのシークレット |
| 9 | `staticwebapp.config.json` が作成されている | `cat staticwebapp.config.json` |

---

**前のステップ**: [Phase 3: Azure リソースの作成](phase-3-azure-resources.md)  
**次のステップ**: [Phase 5: GitHub Actions CI/CD パイプラインの構成](phase-5-cicd.md)
