# Phase 5: GitHub Actions CI/CD パイプラインの構成

> **目的**: GitHub Actions ワークフローを作成し、main ブランチへの push で自動デプロイ、PR でステージング環境が自動作成される CI/CD パイプラインを構築する。  
> **所要時間目安**: 20〜30 分  
> **前提**: [Phase 3](phase-3-azure-resources.md) および [Phase 4](phase-4-security.md) が完了していること

---

## 目次

1. [CI/CD パイプラインの設計](#1-cicd-パイプラインの設計)
2. [既存ワークフローの整理](#2-既存ワークフローの整理)
3. [ワークフロー YAML の作成](#3-ワークフロー-yaml-の作成)
4. [ワークフロー設定の詳細解説](#4-ワークフロー設定の詳細解説)
5. [GitHub Secrets の確認](#5-github-secrets-の確認)
6. [ワークフローのコミット](#6-ワークフローのコミット)
7. [完了チェックリスト](#7-完了チェックリスト)

---

## 1. CI/CD パイプラインの設計

### 1.1 パイプラインの全体像

```
┌─────────────┐    push     ┌──────────────────┐   OIDC    ┌─────────┐
│  開発者      │ ──────────→ │  GitHub Actions   │ ────────→ │  Azure  │
│  (WSL)      │             │  ワークフロー      │           │  SWA    │
└─────────────┘             └──────────────────┘           └─────────┘
                                    │
                              ┌─────┴─────┐
                              │           │
                         push to main   PR open
                              │           │
                         本番デプロイ  ステージングデプロイ
                              │           │
                         Production   Preview URL
                         Environment  (PR ごとに独立)
```

### 1.2 トリガー条件

| イベント | 条件 | アクション |
|---|---|---|
| `push` | `main` ブランチ | 本番環境にデプロイ |
| `pull_request` opened | `main` ブランチが対象 | ステージング環境を作成・デプロイ |
| `pull_request` synchronize | `main` ブランチが対象 | ステージング環境を更新 |
| `pull_request` closed | `main` ブランチが対象 | ステージング環境を削除 |

---

## 2. 既存ワークフローの整理

Phase 3 で SWA リソースを作成した際、GitHub Actions ワークフローが自動生成されています。これをカスタマイズします。

### 2.1 自動生成されたワークフローの確認

```bash
ls .github/workflows/
```

`azure-static-web-apps-*.yml` というファイルが存在するはずです。

### 2.2 自動生成ファイルの削除

カスタムのワークフローに置き換えるため、自動生成されたファイルを削除します。

```bash
# 自動生成されたワークフローファイルを削除
rm .github/workflows/azure-static-web-apps-*.yml
```

---

## 3. ワークフロー YAML の作成

### 3.1 ディレクトリの作成

```bash
mkdir -p .github/workflows
```

### 3.2 ワークフローファイルの作成

```bash
cat > .github/workflows/azure-static-web-apps.yml << 'WORKFLOW'
name: Azure Static Web Apps CI/CD

on:
  push:
    branches:
      - main
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches:
      - main

# OIDC 認証に必要なパーミッション
permissions:
  id-token: write       # OIDC トークンの取得に必要
  contents: read         # リポジトリのチェックアウトに必要
  pull-requests: write   # PR コメント (ステージング URL 通知) に必要

jobs:
  # ========================================
  # ビルド & デプロイジョブ
  # ========================================
  build_and_deploy:
    if: github.event_name == 'push' || (github.event_name == 'pull_request' && github.event.action != 'closed')
    runs-on: ubuntu-latest
    name: Build and Deploy
    steps:
      # ソースコードのチェックアウト
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: true
          lfs: false

      # Node.js のセットアップ (Astro が要求するバージョン)
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      # OIDC 認証: ID Token の取得
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

      # Azure Static Web Apps へのデプロイ
      - name: Build And Deploy
        id: builddeploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          action: "upload"
          # === Astro 固有の設定 ===
          app_location: "/"          # Astro プロジェクトのルート
          output_location: "dist"    # Astro の ビルド出力ディレクトリ
          # === OIDC 認証 ===
          github_id_token: ${{ steps.idtoken.outputs.result }}

  # ========================================
  # PR クローズ時のステージング環境削除ジョブ
  # ========================================
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
WORKFLOW
```

### 3.3 ファイルの確認

```bash
cat .github/workflows/azure-static-web-apps.yml
```

---

## 4. ワークフロー設定の詳細解説

### 4.1 パーミッション設定

```yaml
permissions:
  id-token: write       # OIDC トークンの取得に必要
  contents: read         # リポジトリのチェックアウトに必要
  pull-requests: write   # PR コメント (ステージング URL 通知) に必要
```

| パーミッション | 理由 |
|---|---|
| `id-token: write` | GitHub が OIDC ID Token を発行するために必須 |
| `contents: read` | `actions/checkout@v4` でソースコードを読むために必須 |
| `pull-requests: write` | SWA がステージング URL を PR コメントに書き込むために必要 |

### 4.2 Astro 固有の設定

| 設定 | 値 | 説明 |
|---|---|---|
| `app_location` | `"/"` | `package.json` が存在するディレクトリ。Astro プロジェクトのルート |
| `output_location` | `"dist"` | `astro build` の出力先ディレクトリ |
| `node-version` | `'22'` | Astro v5 が要求する Node.js バージョン |

> **注意**: SWA の組み込みテンプレートには Astro が含まれていないため、`app_location` と `output_location` を手動で指定する必要があります。

### 4.3 OIDC 認証の仕組み

ワークフロー内の OIDC 認証は以下の流れで動作します:

1. **`Install OIDC Client`**: `@actions/core` パッケージをインストール
2. **`Get Id Token`**: `getIDToken()` で GitHub OIDC プロバイダから短命な ID Token を取得
3. **`Build And Deploy`**: `github_id_token` パラメータで ID Token を SWA デプロイアクションに渡す
4. SWA デプロイアクションが Entra ID の Federated Credential で Token を検証
5. 検証成功後、デプロイが実行される

### 4.4 ステージング環境

PR を作成すると:

1. `build_and_deploy` ジョブが実行される
2. PR 固有のステージング URL が自動生成される（例: `https://gentle-water-0a1b2c3d4-5.japaneast.azurestaticapps.net`）
3. ステージング URL が PR のコメントに自動投稿される
4. PR が更新（新しいコミットが push）されるたびに、ステージング環境も更新される
5. PR がクローズ/マージされると、`close_pull_request` ジョブでステージング環境が自動削除される

---

## 5. GitHub Secrets の確認

ワークフローで使用するすべてのシークレットが設定されていることを確認します。

```bash
gh secret list
```

必要なシークレット:

| シークレット名 | 設定元 | 用途 |
|---|---|---|
| `AZURE_STATIC_WEB_APPS_API_TOKEN` | Phase 3 (自動設定) | SWA デプロイトークン |
| `AZURE_CLIENT_ID` | Phase 4 (手動設定) | OIDC - アプリケーション ID |
| `AZURE_TENANT_ID` | Phase 4 (手動設定) | OIDC - テナント ID |
| `AZURE_SUBSCRIPTION_ID` | Phase 4 (手動設定) | OIDC - サブスクリプション ID |

不足がある場合は [Phase 4 のセクション 6.5](phase-4-security.md#65-github-リポジトリにシークレットを登録) を参照してください。

---

## 6. ワークフローのコミット

### 6.1 すべての変更をまとめてコミット

Phase 4 と Phase 5 の変更をまとめてコミットします。

```bash
# 変更状況を確認
git status

# 変更をステージング
git add .github/workflows/azure-static-web-apps.yml
git add staticwebapp.config.json

# コミット
git commit -m "Add CI/CD workflow and SWA config

- Add GitHub Actions workflow with OIDC authentication
- Add staticwebapp.config.json for auth and routing
- Configure Astro build settings (app_location, output_location)"
```

> **まだ push しないでください。** Phase 6 でまとめて push し、デプロイの動作確認を行います。

---

## 7. 完了チェックリスト

以下がすべて ✅ になっていれば、Phase 5 は完了です。

| # | チェック項目 | 確認方法 |
|---|---|---|
| 1 | 自動生成されたワークフローが削除されている | `ls .github/workflows/` に `azure-static-web-apps-*.yml` がない |
| 2 | カスタムワークフローが作成されている | `cat .github/workflows/azure-static-web-apps.yml` |
| 3 | OIDC (Identity Token) 方式が設定されている | ワークフロー内に `id-token: write` と `github_id_token` |
| 4 | `app_location` が `"/"` に設定されている | ワークフロー内で確認 |
| 5 | `output_location` が `"dist"` に設定されている | ワークフロー内で確認 |
| 6 | PR 時のステージング環境削除ジョブがある | `close_pull_request` ジョブの存在確認 |
| 7 | GitHub Secrets に 4 つのシークレットが登録されている | `gh secret list` |
| 8 | ワークフローと config がコミット済み | `git log --oneline -1` |

---

**前のステップ**: [Phase 4: セキュリティ設定 (Managed Identity)](phase-4-security.md)  
**次のステップ**: [Phase 6: デプロイと動作検証](phase-6-deploy-verify.md)
