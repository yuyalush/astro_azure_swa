# Phase 7: 運用・メンテナンスガイド

> **目的**: 本番運用に向けた追加設定と、日常的な運用・トラブルシューティングの手引きを提供する。  
> **所要時間目安**: 30〜60 分（設定項目によって異なる）  
> **前提**: [Phase 6](phase-6-deploy-verify.md) が完了していること

---

## 目次

1. [カスタムドメインの設定](#1-カスタムドメインの設定)
2. [Application Insights の統合（オプション）](#2-application-insights-の統合オプション)
3. [セキュリティの強化](#3-セキュリティの強化)
4. [日常的な運用タスク](#4-日常的な運用タスク)
5. [よくあるトラブルシューティング](#5-よくあるトラブルシューティング)
6. [リソースのクリーンアップ](#6-リソースのクリーンアップ)
7. [今後の拡張](#7-今後の拡張)

---

## 1. カスタムドメインの設定

### 1.1 前提条件

- 独自ドメインを所有していること（お名前.com、Cloudflare、Route 53 など）
- ドメインの DNS レコードを管理できること

### 1.2 カスタムドメインの追加

```bash
# カスタムドメインを SWA に登録
az staticwebapp hostname set \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --hostname www.example.com
```

### 1.3 DNS レコードの設定

ドメインレジストラの DNS 管理画面で以下のレコードを追加します。

#### www サブドメインの場合（CNAME）

| タイプ | ホスト | 値 |
|---|---|---|
| CNAME | `www` | `<DEFAULT_HOST>.azurestaticapps.net` |

```bash
# デフォルトホスト名の確認
az staticwebapp show \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --query "defaultHostname" -o tsv
```

#### ルートドメインの場合 (ALIAS / ANAME)

ルートドメイン (`example.com`) を使用する場合、DNS プロバイダが ALIAS / ANAME レコードをサポートしている必要があります。

| タイプ | ホスト | 値 |
|---|---|---|
| ALIAS | `@` | `<DEFAULT_HOST>.azurestaticapps.net` |

### 1.4 SSL 証明書

- SWA は**カスタムドメインに対しても自動で SSL 証明書**を発行・管理します (Let's Encrypt)
- 手動での証明書設定は不要
- 証明書の自動更新も SWA が処理

### 1.5 検証

```bash
# カスタムドメインの設定確認
az staticwebapp hostname list \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  -o table
```

ドメインの DNS 伝播には最大 48 時間かかる場合がありますが、通常は数分〜数時間で完了します。

---

## 2. Application Insights の統合（オプション）

サイトのパフォーマンス監視とエラートラッキングを設定します。

### 2.1 Application Insights リソースの作成

```bash
# Application Insights リソースの作成
az monitor app-insights component create \
  --app appi-astro-swa \
  --location japaneast \
  --resource-group rg-astro-swa \
  --application-type web
```

### 2.2 接続文字列の取得

```bash
# Instrumentation Key の取得
INSTRUMENTATION_KEY=$(az monitor app-insights component show \
  --app appi-astro-swa \
  --resource-group rg-astro-swa \
  --query "instrumentationKey" -o tsv)

echo "Instrumentation Key: $INSTRUMENTATION_KEY"
```

### 2.3 Key Vault にインストルメンテーションキーを格納

```bash
az keyvault secret set \
  --vault-name kv-astro-swa \
  --name "AppInsightsKey" \
  --value "$INSTRUMENTATION_KEY"

# SWA の App Settings に Key Vault 参照を追加
az staticwebapp appsettings set \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --setting-names \
    "APPINSIGHTS_INSTRUMENTATIONKEY=@Microsoft.KeyVault(VaultName=kv-astro-swa;SecretName=AppInsightsKey)"
```

### 2.4 フロントエンドへの組み込み（オプション）

クライアントサイドのトラッキングを追加する場合、Astro のレイアウトに Application Insights SDK を追加します。

```bash
npm install @microsoft/applicationinsights-web
```

`src/layouts/Layout.astro` のヘッダーにスクリプトを追加:

```html
<script>
  import { ApplicationInsights } from '@microsoft/applicationinsights-web';

  const appInsights = new ApplicationInsights({
    config: {
      instrumentationKey: import.meta.env.PUBLIC_APPINSIGHTS_KEY || ''
    }
  });
  appInsights.loadAppInsights();
  appInsights.trackPageView();
</script>
```

> **注意**: クライアントサイドに公開される環境変数は `PUBLIC_` プレフィックスが必要です (Astro の仕様)。Instrumentation Key はクライアントサイドで使用されるもので、シークレットではありません。

---

## 3. セキュリティの強化

### 3.1 セキュリティヘッダーの追加

`staticwebapp.config.json` にセキュリティヘッダーを追加します。

```json
{
  "globalHeaders": {
    "X-Content-Type-Options": "nosniff",
    "X-Frame-Options": "DENY",
    "X-XSS-Protection": "1; mode=block",
    "Referrer-Policy": "strict-origin-when-cross-origin",
    "Permissions-Policy": "camera=(), microphone=(), geolocation=()",
    "Content-Security-Policy": "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'"
  }
}
```

> **CSP (Content-Security-Policy)** は必要に応じて調整してください。外部スクリプトや CDN を使用する場合、対応するドメインを追加する必要があります。

### 3.2 不要な認証プロバイダのブロック

GitHub 認証のみを使用し、他のプロバイダをブロックする場合:

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

### 3.3 Key Vault シークレットのローテーション

シークレットを定期的にローテーションすることを推奨します。

```bash
# 新しいシークレット値を設定 (新しいバージョンが自動作成される)
az keyvault secret set \
  --vault-name kv-astro-swa \
  --name "ExampleSecret" \
  --value "new-secret-value-67890"
```

> SWA の Application Settings で Key Vault 参照を使用している場合、シークレットのローテーション時に SWA 側の設定変更は不要です（最新バージョンが自動取得されます）。

---

## 4. 日常的な運用タスク

### 4.1 デプロイ履歴の確認

```bash
# GitHub Actions のデプロイ履歴
gh run list --workflow="azure-static-web-apps.yml" --limit 10

# 特定の実行の詳細
gh run view <RUN_ID>
```

### 4.2 SWA の環境一覧

```bash
az staticwebapp environment list \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  -o table
```

### 4.3 Application Settings の管理

```bash
# 設定の一覧
az staticwebapp appsettings list \
  --name swa-astro-site \
  --resource-group rg-astro-swa

# 設定の追加・更新
az staticwebapp appsettings set \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --setting-names "NEW_SETTING=value"

# 設定の削除
az staticwebapp appsettings delete \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --setting-names "OLD_SETTING"
```

### 4.4 ログの確認

Azure Portal:

1. SWA リソースを開く
2. **監視** → **ログ** をクリック
3. KQL クエリでログを検索

---

## 5. よくあるトラブルシューティング

### 一覧表

| # | 問題 | 原因 | 対処法 |
|---|---|---|---|
| 1 | GitHub Actions ビルドが失敗する | Node.js バージョンの不一致 | `package.json` の `engines` フィールドを確認 |
| 2 | デプロイ後に 404 エラー | `output_location` の設定ミス | ワークフローの `output_location` が `"dist"` か確認 |
| 3 | 認証が動作しない | `staticwebapp.config.json` の構文エラー | `python3 -m json.tool staticwebapp.config.json` で検証 |
| 4 | Key Vault 参照が解決されない | Managed Identity の権限不足 | `Key Vault Secrets User` ロールが付与されているか確認 |
| 5 | OIDC デプロイが失敗する | Federated Credential の `subject` 不一致 | `subject` のリポジトリ名・ブランチ名を確認 |
| 6 | SWA CLI のローカルエミュレーションが起動しない | ポート競合 | `swa start dist --port 4280` でポート明示 |
| 7 | カスタムドメインに SSL 証明書が発行されない | DNS 伝播未完了 | 最大 48 時間待機、DNS 設定を再確認 |
| 8 | ステージング環境が作成されない | PR の対象ブランチが `main` でない | ワークフローの `branches` 設定を確認 |

### 5.1 詳細: ビルドエラーの調査

```bash
# 最新の失敗したワークフローログを表示
gh run list --status failure --limit 1
gh run view <RUN_ID> --log-failed
```

### 5.2 詳細: ローカルでの再現テスト

```bash
# ローカルでビルドが通るか確認
npm run build

# SWA エミュレータで動作確認
swa start dist

# 認証のテスト
# http://localhost:4280/.auth/login/github にアクセス
```

### 5.3 詳細: Azure リソースの状態確認

```bash
# SWA リソースの状態
az staticwebapp show \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --query '{name:name, sku:sku.name, state:state, url:defaultHostname}' \
  -o table

# Managed Identity の状態
az staticwebapp identity show \
  --name swa-astro-site \
  --resource-group rg-astro-swa

# Key Vault の状態
az keyvault show \
  --name kv-astro-swa \
  --query '{name:name, state:properties.provisioningState, rbac:properties.enableRbacAuthorization}' \
  -o table
```

---

## 6. リソースのクリーンアップ

ハンズオン終了後、不要なリソースを削除してコストを抑えます。

> **⚠️ 注意**: 以下の操作は元に戻せません。本番環境で使用している場合は実行しないでください。

### 6.1 Azure リソースの削除

```bash
# リソースグループごと削除 (配下のリソースもすべて削除される)
# SWA, Key Vault, Application Insights がすべて削除されます
az group delete \
  --name rg-astro-swa \
  --yes \
  --no-wait

echo "リソースグループの削除をリクエストしました。完全に削除されるまで数分かかります。"
```

### 6.2 Entra ID のアプリ登録の削除

```bash
# アプリケーション ID を取得
APP_ID=$(az ad app list \
  --display-name "github-actions-astro-swa" \
  --query "[0].appId" -o tsv)

# アプリ登録を削除
az ad app delete --id "$APP_ID"

echo "Entra ID のアプリ登録を削除しました。"
```

### 6.3 GitHub Secrets の削除

```bash
gh secret delete AZURE_STATIC_WEB_APPS_API_TOKEN
gh secret delete AZURE_CLIENT_ID
gh secret delete AZURE_TENANT_ID
gh secret delete AZURE_SUBSCRIPTION_ID

echo "GitHub Secrets を削除しました。"
```

### 6.4 GitHub リポジトリの保持

リポジトリ自体はコードの履歴として保持することを推奨しますが、削除する場合:

```bash
# リポジトリを削除（確認プロンプトあり）
gh repo delete <YOUR_USERNAME>/astro_azure_swa --yes
```

### 6.5 削除確認

```bash
# Azure リソースが削除されたか確認
az group exists --name rg-astro-swa
# false が返ることを確認

# Entra ID アプリが削除されたか確認
az ad app list --display-name "github-actions-astro-swa" --query "length(@)"
# 0 が返ることを確認
```

---

## 7. 今後の拡張

本ハンズオンで構築した環境をベースに、以下の拡張が可能です。

| 拡張項目 | 概要 | 参考リソース |
|---|---|---|
| **API バックエンド** | Azure Functions をマネージド API として追加 | [SWA API ドキュメント](https://learn.microsoft.com/azure/static-web-apps/add-api) |
| **SSR 対応** | Astro SSR アダプタを使用してサーバーサイドレンダリングを追加 | [Astro SSR ガイド](https://docs.astro.build/guides/server-side-rendering/) |
| **IaC (Infrastructure as Code)** | Bicep / Terraform でリソース管理を自動化 | [SWA Bicep テンプレート](https://learn.microsoft.com/azure/templates/microsoft.web/staticsites) |
| **マルチ環境** | staging / production の環境を明確に分離 | SWA のステージング環境機能 |
| **CMS 統合** | Headless CMS (Contentful, Strapi 等) を組み合わせ | [Astro CMS ガイド](https://docs.astro.build/guides/cms/) |
| **Cosmos DB 連携** | グローバル分散データベースをバックエンドに追加 | [Cosmos DB ドキュメント](https://learn.microsoft.com/azure/cosmos-db/) |
| **Azure Front Door** | CDN + WAF + カスタムルーティングで更にセキュアに | [Azure Front Door ドキュメント](https://learn.microsoft.com/azure/frontdoor/) |

---

**前のステップ**: [Phase 6: デプロイと動作検証](phase-6-deploy-verify.md)  
**最初に戻る**: [Phase 0: 前提条件・環境準備](phase-0-prerequisites.md)  
**設計書**: [アーキテクチャ設計書](architecture.md)
