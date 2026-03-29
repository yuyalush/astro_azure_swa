# Astro × Azure Static Web Apps ハンズオンコンテンツ設計書

## コンテンツ概要

Astro フレームワークを使用した静的サイトを Azure Static Web Apps (SWA) にデプロイするまでの一連の流れを、ステップバイステップで実施できるハンズオンコンテンツ。Managed Identity を活用したセキュアな構成、GitHub Actions による CI/CD パイプラインの構築を含む。

### 対象読者

- Web 開発の基礎知識がある開発者
- Azure の基本的な概念を理解している方
- GitHub の基本操作（リポジトリ作成、push、PR）ができる方

### 最終ゴール

- Astro で構築した静的サイトが Azure Static Web Apps 上で稼働している
- GitHub への push をトリガーに自動デプロイされる CI/CD パイプラインが構築されている
- Managed Identity による Key Vault 連携でシークレットがセキュアに管理されている
- 認証・認可が `staticwebapp.config.json` で適切に設定されている

### 技術スタック

| カテゴリ | 技術 |
|---|---|
| フレームワーク | Astro (SSG モード) |
| ホスティング | Azure Static Web Apps (Standard プラン) |
| CI/CD | GitHub Actions |
| 認証 | SWA 組み込み認証 (GitHub / Microsoft Entra ID) |
| シークレット管理 | Azure Key Vault + Managed Identity |
| デプロイ認証 | OIDC (Identity Token) 方式 |
| ソースコード管理 | GitHub |
| エディタ | Visual Studio Code |

---

## Phase 構成

各 Phase は Agent が独立して実行できる粒度で分割されている。Phase 間には依存関係があり、原則として番号順に実施する。

```
Phase 0: 前提条件・環境準備
    ↓
Phase 1: Astro プロジェクトの作成とローカル開発
    ↓
Phase 2: GitHub リポジトリのセットアップ
    ↓
Phase 3: Azure リソースの作成
    ↓
Phase 4: セキュリティ設定 (Managed Identity)
    ↓
Phase 5: GitHub Actions CI/CD パイプラインの構成
    ↓
Phase 6: デプロイと動作検証
    ↓
Phase 7: 運用・メンテナンスガイド
```

---

## Phase 0: 前提条件・環境準備

### 目的

開発に必要なツール・アカウント・VS Code 拡張機能をすべてセットアップし、以降の Phase にスムーズに進められる状態にする。

### 前提条件

- **Azure サブスクリプション**: [無料アカウント](https://azure.microsoft.com/free) でも開始可能（ただし Phase 4 の Managed Identity 利用には Standard プランが必要）
- **GitHub アカウント**: リポジトリの作成・管理に使用
- **Node.js**: v22.12.0 以上（Astro ビルドの互換性のため）
- **VS Code**: 最新版

### 推奨 VS Code 拡張機能

以下の拡張機能をインストールすることで、開発・デプロイの効率が大幅に向上する。

| 拡張機能 | ID | 用途 |
|---|---|---|
| **Astro** | `astro-build.astro-vscode` | `.astro` ファイルのシンタックスハイライト、IntelliSense、フォーマット |
| **Azure Static Web Apps** | `ms-azuretools.vscode-azurestaticwebapps` | SWA リソースの作成・管理、ローカルエミュレーション連携 |
| **Azure Account** | `ms-vscode.azure-account` | Azure へのサインインとサブスクリプション管理 |
| **Azure Resources** | `ms-azuretools.vscode-azureresourcegroups` | Azure リソースの一覧表示・管理 |
| **GitHub Actions** | `github.vscode-github-actions` | ワークフローの構文チェック、実行状況の確認 |
| **GitHub Pull Requests** | `github.vscode-pull-request-github` | VS Code 内で PR の作成・レビュー |

### CLI ツールのインストール

```bash
# Azure CLI
# Windows: winget install -e --id Microsoft.AzureCLI
# macOS: brew install azure-cli

# Azure Static Web Apps CLI
npm install -g @azure/static-web-apps-cli

# GitHub CLI (オプション)
# Windows: winget install -e --id GitHub.cli
# macOS: brew install gh
```

### 完了条件

- [ ] Node.js v22.12.0 以上がインストールされている (`node -v` で確認)
- [ ] VS Code に推奨拡張機能がインストールされている
- [ ] Azure CLI でサインイン済み (`az login` → `az account show`)
- [ ] SWA CLI がインストールされている (`swa --version`)
- [ ] GitHub アカウントにサインイン済み

---

## Phase 1: Astro プロジェクトの作成とローカル開発

### 目的

Astro プロジェクトをスキャフォールドし、ローカル環境での開発・プレビューができる状態にする。

### 依存関係

- Phase 0 が完了していること

### 手順

#### 1.1 Astro プロジェクトの作成

```bash
npm create astro@latest astro_azure_swa
```

ウィザードの推奨選択:
- テンプレート: `basics` または `empty`
- TypeScript: `Yes`（推奨）
- 依存関係のインストール: `Yes`

#### 1.2 プロジェクト構成の確認

```
astro_azure_swa/
├── src/
│   ├── pages/          # ページコンポーネント (ファイルベースルーティング)
│   ├── layouts/        # レイアウトコンポーネント
│   └── components/     # 再利用可能なコンポーネント
├── public/             # 静的アセット (そのまま配信される)
├── astro.config.mjs    # Astro 設定ファイル
├── package.json
└── tsconfig.json
```

#### 1.3 package.json の engines フィールド設定

Azure SWA のビルド環境との互換性のため、`package.json` に以下を追加する。

```json
{
  "engines": {
    "node": ">=22.12.0"
  }
}
```

> **注意**: この設定がないと、SWA の GitHub Actions ビルドが古い Node.js バージョンで実行され、Astro ビルドが失敗する既知の問題がある。

#### 1.4 ローカル開発サーバーの起動

```bash
npm run dev
```

ブラウザで `http://localhost:4321` にアクセスし、動作確認。

#### 1.5 ビルドの確認

```bash
npm run build
```

`dist/` ディレクトリに静的ファイルが生成されることを確認。

#### 1.6 SWA CLI によるローカルエミュレーション

```bash
swa start dist
```

SWA の認証エンドポイント (`/.auth/login/github` 等) を含むローカルエミュレーション環境で動作確認できる。

### 完了条件

- [ ] Astro プロジェクトが作成されている
- [ ] `npm run dev` でローカル開発サーバーが起動する
- [ ] `npm run build` で `dist/` にビルド成果物が生成される
- [ ] `package.json` に `engines` フィールドが設定されている
- [ ] `swa start dist` でローカルエミュレーションが動作する

---

## Phase 2: GitHub リポジトリのセットアップ

### 目的

プロジェクトのソースコードを GitHub リポジトリで管理し、CI/CD の基盤を整える。

### 依存関係

- Phase 1 が完了していること

### 手順

#### 2.1 .gitignore の確認

Astro のスキャフォールドで生成される `.gitignore` に以下が含まれていることを確認する。

```
node_modules/
dist/
.astro/
.env
.env.*
```

#### 2.2 GitHub リポジトリの作成

```bash
# GitHub CLI を使用する場合
gh repo create astro_azure_swa --public --source=. --remote=origin

# または GitHub Web UI でリポジトリを作成し、手動で remote 追加
git remote add origin https://github.com/<YOUR_USERNAME>/astro_azure_swa.git
```

#### 2.3 初回コミットと push

```bash
git init
git add .
git commit -m "Initial commit: Astro project scaffold"
git branch -M main
git push -u origin main
```

#### 2.4 ブランチ戦略

| ブランチ | 用途 |
|---|---|
| `main` | 本番環境にデプロイされるブランチ |
| `dev` | 開発用ブランチ（PR を経由して main にマージ） |
| `feature/*` | 機能開発用の一時ブランチ |

```bash
git checkout -b dev
git push -u origin dev
```

### 完了条件

- [ ] GitHub リポジトリが作成されている
- [ ] `main` ブランチにソースコードが push されている
- [ ] `.gitignore` が適切に設定されている
- [ ] `dev` ブランチが作成されている

---

## Phase 3: Azure リソースの作成

### 目的

Azure 上に Static Web Apps リソースを作成し、GitHub リポジトリと接続する。

### 依存関係

- Phase 2 が完了していること

### 手順

#### 3.1 リソースグループの作成

```bash
az group create \
  --name rg-astro-swa \
  --location japaneast
```

#### 3.2 Azure Static Web Apps リソースの作成

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

> **プラン選択の判断基準**:
> | プラン | 特徴 |
> |---|---|
> | **Free** | 無料。Managed Identity・Key Vault 連携・カスタム認証は利用不可 |
> | **Standard** | Managed Identity、Key Vault 連携、Bring Your Own API、SLA あり。本コンテンツでは **Standard を推奨** |

#### 3.3 デプロイトークンの取得（後続 Phase で使用）

```bash
az staticwebapp secrets list \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --query "properties.apiKey" -o tsv
```

> GitHub Actions で OIDC 方式を使用する場合、このトークンは参照用として取得するのみ。

#### 3.4 VS Code から確認

Azure Static Web Apps 拡張機能のサイドバーから、作成したリソースが表示されることを確認する。

### 完了条件

- [ ] リソースグループ `rg-astro-swa` が作成されている
- [ ] Static Web Apps リソース `swa-astro-site` が Standard プランで作成されている
- [ ] GitHub リポジトリと接続されている
- [ ] VS Code の Azure 拡張機能からリソースが確認できる

---

## Phase 4: セキュリティ設定 (Managed Identity)

### 目的

Managed Identity を活用して、シークレットの安全な管理と、GitHub Actions からのセキュアなデプロイを実現する。

### 依存関係

- Phase 3 が完了していること
- Static Web Apps が Standard プランであること

### 手順

#### 4.1 System-assigned Managed Identity の有効化

Azure Portal:
1. Static Web Apps リソースを開く
2. **設定** → **ID (Identity)** を選択
3. **システム割り当て済み** タブで **状態** を **オン** に変更
4. **保存** をクリック

Azure CLI:
```bash
az staticwebapp identity assign \
  --name swa-astro-site \
  --resource-group rg-astro-swa
```

#### 4.2 Azure Key Vault の作成

```bash
az keyvault create \
  --name kv-astro-swa \
  --resource-group rg-astro-swa \
  --location japaneast \
  --sku standard
```

#### 4.3 Key Vault にシークレットを登録

```bash
az keyvault secret set \
  --vault-name kv-astro-swa \
  --name "ExampleSecret" \
  --value "your-secret-value"
```

#### 4.4 Key Vault アクセスポリシーの設定

SWA の Managed Identity に Key Vault のシークレット読み取り権限を付与する。

```bash
# SWA の Managed Identity の Principal ID を取得
PRINCIPAL_ID=$(az staticwebapp identity show \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --query "principalId" -o tsv)

# Key Vault の RBAC でシークレット読み取り権限を付与
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee "$PRINCIPAL_ID" \
  --scope $(az keyvault show --name kv-astro-swa --query id -o tsv)
```

#### 4.5 SWA Application Settings での Key Vault 参照

Azure Portal:
1. Static Web Apps リソースの **設定** → **構成** を開く
2. **アプリケーション設定** で **追加** をクリック
3. 値に Key Vault 参照を設定:

```
@Microsoft.KeyVault(VaultName=kv-astro-swa;SecretName=ExampleSecret)
```

#### 4.6 GitHub Actions OIDC 連携の設定

デプロイ時にデプロイトークンではなく、OIDC (Federated Identity Credential) を使用してよりセキュアにデプロイする。

##### 4.6.1 Microsoft Entra ID でアプリ登録

```bash
# アプリケーション登録
az ad app create --display-name "github-actions-astro-swa"

# アプリケーション ID を取得
APP_ID=$(az ad app list --display-name "github-actions-astro-swa" --query "[0].appId" -o tsv)

# サービスプリンシパルを作成
az ad sp create --id "$APP_ID"
```

##### 4.6.2 Federated Credential の作成

```bash
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "github-actions-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<YOUR_USERNAME>/astro_azure_swa:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

##### 4.6.3 GitHub リポジトリにシークレットを登録

GitHub リポジトリの **Settings** → **Secrets and variables** → **Actions** に以下を登録:

| シークレット名 | 値 |
|---|---|
| `AZURE_CLIENT_ID` | アプリケーション (クライアント) ID |
| `AZURE_TENANT_ID` | ディレクトリ (テナント) ID |
| `AZURE_SUBSCRIPTION_ID` | Azure サブスクリプション ID |

#### 4.7 staticwebapp.config.json の作成

プロジェクトルートに `staticwebapp.config.json` を作成し、認証・認可ルールを設定する。

```json
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
```

### 完了条件

- [ ] SWA で System-assigned Managed Identity が有効化されている
- [ ] Key Vault が作成され、シークレットが登録されている
- [ ] Key Vault の RBAC で SWA の MI にシークレット読み取り権限が付与されている
- [ ] SWA の Application Settings で Key Vault 参照が設定されている
- [ ] Microsoft Entra ID でアプリ登録と Federated Credential が構成されている
- [ ] GitHub リポジトリにOIDC 用のシークレットが登録されている
- [ ] `staticwebapp.config.json` が作成されている

---

## Phase 5: GitHub Actions CI/CD パイプラインの構成

### 目的

GitHub Actions ワークフローを作成し、main ブランチへの push で自動デプロイ、PR でステージング環境が自動作成される CI/CD パイプラインを構築する。

### 依存関係

- Phase 3, Phase 4 が完了していること

### 手順

#### 5.1 ワークフロー YAML の作成

`.github/workflows/azure-static-web-apps.yml` を作成する。

```yaml
name: Azure Static Web Apps CI/CD

on:
  push:
    branches:
      - main
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches:
      - main

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  build_and_deploy:
    if: github.event_name == 'push' || (github.event_name == 'pull_request' && github.event.action != 'closed')
    runs-on: ubuntu-latest
    name: Build and Deploy
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true
          lfs: false

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install OIDC Client
        run: npm install @actions/core@1.6.0 @actions/http-client

      - name: Get Id Token
        uses: actions/github-script@v7
        id: idtoken
        with:
          script: |
            const coredemo = require('@actions/core')
            return await coredemo.getIDToken()
          result-encoding: string

      - name: Build And Deploy
        id: builddeploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          action: "upload"
          app_location: "/"
          output_location: "dist"
          github_id_token: ${{ steps.idtoken.outputs.result }}

  close_pull_request:
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    runs-on: ubuntu-latest
    name: Close Pull Request
    steps:
      - name: Close Pull Request
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          action: "close"
```

#### 5.2 ワークフロー設定のポイント

| 設定項目 | 値 | 説明 |
|---|---|---|
| `app_location` | `"/"` | Astro プロジェクトのルートディレクトリ |
| `output_location` | `"dist"` | Astro のビルド出力ディレクトリ |
| `id-token: write` | — | OIDC トークン取得に必要なパーミッション |
| `github_id_token` | — | OIDC 方式でのデプロイ認証に使用 |

#### 5.3 PR ステージング環境

- PR を作成すると、自動でステージング環境がデプロイされる
- PR がクローズ/マージされると、`close_pull_request` ジョブでステージング環境が自動削除される
- ステージング URL は PR のコメントに自動投稿される

#### 5.4 GitHub リポジトリへのシークレット登録

```bash
# デプロイトークンを GitHub Secrets に登録
gh secret set AZURE_STATIC_WEB_APPS_API_TOKEN \
  --body "$(az staticwebapp secrets list --name swa-astro-site --resource-group rg-astro-swa --query 'properties.apiKey' -o tsv)"
```

### 完了条件

- [ ] `.github/workflows/azure-static-web-apps.yml` が作成されている
- [ ] ワークフロー内で OIDC (Identity Token) 方式が設定されている
- [ ] GitHub リポジトリに `AZURE_STATIC_WEB_APPS_API_TOKEN` シークレットが登録されている
- [ ] PR 時のステージング環境の自動作成・削除が設定されている

---

## Phase 6: デプロイと動作検証

### 目的

実際にデプロイを実行し、サイトの動作・認証フロー・Key Vault 連携を検証する。

### 依存関係

- Phase 5 が完了していること

### 手順

#### 6.1 デプロイの実行

```bash
git add .
git commit -m "Add GitHub Actions workflow and SWA configuration"
git push origin main
```

#### 6.2 GitHub Actions の実行確認

1. GitHub リポジトリの **Actions** タブを開く
2. ワークフロー `Azure Static Web Apps CI/CD` が実行中であることを確認
3. `Build and Deploy` ジョブが成功することを確認

VS Code の GitHub Actions 拡張機能からも実行状況を確認できる。

#### 6.3 デプロイされたサイトの確認

```bash
# SWA のデフォルト URL を取得
az staticwebapp show \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --query "defaultHostname" -o tsv
```

ブラウザで `https://<defaultHostname>` にアクセスし、サイトが表示されることを確認。

#### 6.4 認証フローの検証

1. `https://<defaultHostname>/login` にアクセス → GitHub ログイン画面にリダイレクトされる
2. ログイン後、サイトに戻される
3. `https://<defaultHostname>/.auth/me` にアクセスし、ユーザー情報が返されることを確認
4. `https://<defaultHostname>/logout` でログアウトできることを確認

#### 6.5 Key Vault 連携の検証

SWA の Application Settings に設定した Key Vault 参照が正しく解決されていることを確認する。

```bash
az staticwebapp appsettings list \
  --name swa-astro-site \
  --resource-group rg-astro-swa
```

#### 6.6 PR ステージング環境の検証

1. `dev` ブランチで変更を加えてコミット
2. `main` ブランチに対して PR を作成
3. GitHub Actions が実行され、ステージング URL が PR コメントに表示されることを確認
4. ステージング URL にアクセスし、変更が反映されていることを確認
5. PR をマージし、ステージング環境が削除されることを確認

### 完了条件

- [ ] GitHub Actions ワークフローが成功している
- [ ] デプロイされたサイトにブラウザからアクセスできる
- [ ] 認証フロー (ログイン / ログアウト / ユーザー情報取得) が動作する
- [ ] Key Vault のシークレット参照が正しく解決されている
- [ ] PR 作成時にステージング環境が自動デプロイされる
- [ ] PR マージ時にステージング環境が削除される

---

## Phase 7: 運用・メンテナンスガイド

### 目的

本番運用に向けた追加設定と、トラブルシューティングの手引きを提供する。

### 依存関係

- Phase 6 が完了していること

### 手順

#### 7.1 カスタムドメインの設定

```bash
az staticwebapp hostname set \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --hostname www.example.com
```

- DNS に CNAME レコードを追加（`www` → SWA のデフォルトホスト名）
- SSL 証明書は SWA が自動管理（Let's Encrypt）

#### 7.2 Application Insights の統合（オプション）

Application Insights を統合して、サイトのパフォーマンスとエラーを監視する。

1. Application Insights リソースを作成
2. `staticwebapp.config.json` に接続文字列を設定
3. フロントエンドに Application Insights SDK を追加（オプション）

#### 7.3 よくあるトラブルシューティング

| 問題 | 原因 | 対処法 |
|---|---|---|
| GitHub Actions ビルドが失敗する | Node.js バージョンの不一致 | `package.json` の `engines` フィールドを確認 |
| デプロイ後に 404 エラー | `output_location` の設定ミス | ワークフローの `output_location` が `"dist"` であることを確認 |
| 認証が動作しない | `staticwebapp.config.json` の構文エラー | JSON の構文を検証し、ルート設定を確認 |
| Key Vault 参照が解決されない | Managed Identity の権限不足 | Key Vault の RBAC で `Key Vault Secrets User` ロールが付与されているか確認 |
| OIDC デプロイが失敗する | Federated Credential の `subject` 不一致 | `subject` の `repo:` と `ref:` が正しいか確認 |
| SWA CLI のローカルエミュレーションが起動しない | ポート競合 | `swa start dist --port 4280` でポートを明示的に指定 |

#### 7.4 リソースのクリーンアップ

ハンズオン終了後、不要なリソースを削除する場合:

```bash
# リソースグループごと削除（配下のリソースもすべて削除される）
az group delete --name rg-astro-swa --yes --no-wait

# GitHub Actions のシークレットを削除
gh secret delete AZURE_STATIC_WEB_APPS_API_TOKEN

# Microsoft Entra ID のアプリ登録を削除
az ad app delete --id "$APP_ID"
```

### 完了条件

- [ ] カスタムドメインが設定されている（実施した場合）
- [ ] トラブルシューティング手順を把握している
- [ ] クリーンアップ手順を把握している

---

## Agent 実行ガイド

各 Phase は以下のパターンで Agent に実行を指示できる。

### 指示テンプレート

```
Phase <N> を実行してください。
前提条件: Phase <N-1> の完了条件がすべて満たされていること。
```

### Phase 間の依存関係マトリクス

| Phase | 依存する Phase | 主な成果物 |
|---|---|---|
| 0 | なし | 開発環境のセットアップ |
| 1 | 0 | Astro プロジェクト、ローカル動作確認 |
| 2 | 1 | GitHub リポジトリ、ブランチ戦略 |
| 3 | 2 | Azure リソース (SWA, リソースグループ) |
| 4 | 3 | Managed Identity, Key Vault, OIDC, 認証設定 |
| 5 | 3, 4 | GitHub Actions ワークフロー |
| 6 | 5 | デプロイ検証、動作確認 |
| 7 | 6 | 運用設定、トラブルシューティング |

### スコープ外（今後の拡張候補）

- SSR (Server-Side Rendering) モードの対応
- Terraform / Bicep による Infrastructure as Code
- Azure Functions による API バックエンドの追加
- 複数環境 (staging / production) の構成
- Azure Front Door / CDN との統合
