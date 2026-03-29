# アーキテクチャ設計書

> **プロジェクト**: Astro × Azure Static Web Apps ハンズオン  
> **最終更新**: 2026-03-29  
> **ステータス**: 設計確定

---

## 1. アーキテクチャ概要図

![サービスアーキテクチャ](architecture.svg)

---

## 2. アーキテクチャの全体像

本アーキテクチャは、Astro (SSG) で生成した静的サイトを Azure Static Web Apps にホスティングし、GitHub Actions による CI/CD パイプラインで自動デプロイする構成である。セキュリティは Managed Identity と OIDC Federated Credential を軸に、パスワードレス・トークンレスな運用を実現する。

### 2.1 構成要素一覧

| コンポーネント | サービス / ツール | 役割 |
|---|---|---|
| 開発環境 | WSL (Linux) + VS Code | ローカル開発・テスト |
| フレームワーク | Astro (SSG) | 静的サイトのビルド |
| ローカルエミュレータ | SWA CLI | 認証・API を含むローカルテスト |
| ソースコード管理 | GitHub | リポジトリ・PR 管理 |
| CI/CD | GitHub Actions | 自動ビルド・デプロイ |
| ホスティング | Azure Static Web Apps (Standard) | 静的サイト配信・認証・API ルーティング |
| ID 基盤 | Microsoft Entra ID | App Registration・Federated Credential |
| シークレット管理 | Azure Key Vault | シークレットの安全な格納 |
| ID 連携 | Managed Identity (System-assigned) | SWA → Key Vault のパスワードレスアクセス |

---

## 3. データフロー

### 3.1 CI/CD デプロイフロー

```
開発者 (WSL/VS Code)
  │
  │  git push (main ブランチ)
  ▼
GitHub Repository
  │
  │  on: push トリガー
  ▼
GitHub Actions ワークフロー
  │
  ├─ 1. ソースコードをチェックアウト
  ├─ 2. Node.js 22 セットアップ
  ├─ 3. OIDC ID Token を取得
  ├─ 4. Entra ID (Federated Credential) でトークン検証
  └─ 5. Azure/static-web-apps-deploy@v1 でデプロイ
         │
         ▼
  Azure Static Web Apps
    └─ Astro ビルド成果物 (dist/) を配信
```

**設計ポイント**:
- **OIDC Identity Token 方式**を採用し、長期間有効なデプロイトークンをシークレットとして保持するリスクを排除
- GitHub Actions の `id-token: write` パーミッションで短命なトークンを動的に生成し、Entra ID の Federated Credential で検証
- トークンのローテーション管理が不要

### 3.2 シークレット管理フロー

```
Azure Static Web Apps
  │
  │  System-assigned Managed Identity
  │  (自動生成される資格情報 — 開発者に公開されない)
  ▼
Azure Key Vault
  │
  │  "Key Vault Secrets User" ロール (RBAC)
  │  GET 操作のみ許可
  ▼
シークレット値を取得
  │
  │  @Microsoft.KeyVault(VaultName=...;SecretName=...)
  ▼
SWA Application Settings に自動解決
```

**設計ポイント**:
- **Managed Identity (System-assigned)** を使用することで、SWA がパスワードやクライアントシークレットなしで Key Vault にアクセス
- Key Vault の RBAC で **最小権限の原則**を適用（`Key Vault Secrets User` = 読み取りのみ）
- Application Settings の `@Microsoft.KeyVault(...)` 構文で、シークレット値がコードや構成ファイルに直書きされることを防止
- シークレットのローテーション時も SWA 側の変更は不要（Key Vault の最新バージョンを自動取得）

### 3.3 認証・認可フロー

```
エンドユーザー (ブラウザ)
  │
  │  https://example.com/login
  │  → /.auth/login/github にリダイレクト
  ▼
GitHub OAuth (または Entra ID)
  │
  │  認証成功後コールバック
  ▼
Azure Static Web Apps
  │
  ├─ セッション Cookie を発行
  ├─ /.auth/me でユーザー情報 JSON を返却
  └─ staticwebapp.config.json のルールに基づきアクセス制御
       │
       ├─ /admin/*  → "authenticated" ロール必須
       ├─ /login    → /.auth/login/github にリダイレクト
       └─ 401       → ログインページに自動リダイレクト
```

**設計ポイント**:
- SWA の**組み込み認証**を利用し、認証基盤を自前構築しない（OAuth フローは SWA が自動処理）
- `staticwebapp.config.json` でルートベースのロール制御（コードレスな認可）
- `responseOverrides` による 401 時の自動リダイレクトで、ユーザーフレンドリーなフロー
- プリコンフィギュアされたプロバイダ（GitHub / Entra ID）は追加設定なしで利用可能

---

## 4. セキュリティ設計

### 4.1 ゼロトラスト原則の適用

本アーキテクチャは以下のゼロトラスト原則に基づいて設計されている。

| 原則 | 適用箇所 |
|---|---|
| **明示的な検証** | 全リクエストで認証状態を検証（SWA 組み込み認証） |
| **最小権限アクセス** | Key Vault は `Secrets User`（読み取り専用）、Managed Identity は対象リソース限定 |
| **侵害を想定** | 長期トークン不使用（OIDC 短命トークン）、シークレットの直書き禁止 |

### 4.2 シークレットの管理方針

| 対象 | 管理方法 | 保管場所 |
|---|---|---|
| デプロイ認証 | OIDC Federated Credential | なし (都度発行される短命トークン) |
| アプリケーションシークレット | Key Vault 参照 | Azure Key Vault |
| 認証プロバイダ設定 | SWA 組み込み | Azure SWA (マネージド) |
| GitHub Actions 変数 | GitHub Secrets | GitHub (暗号化ストレージ) |

### 4.3 ネットワークセキュリティ

| 対策 | 実装 |
|---|---|
| HTTPS 強制 | SWA がデフォルトで HTTP → HTTPS リダイレクト |
| SSL 証明書 | Let's Encrypt 自動管理 (カスタムドメイン対応) |
| CDN | Azure グローバルエッジネットワーク上で配信 |
| DDoS 防御 | Azure プラットフォームの標準 DDoS 保護 |

---

## 5. Azure リソース構成

### 5.1 リソースグループ

| リソースグループ | リージョン | 用途 |
|---|---|---|
| `rg-astro-swa` | Japan East | 全リソースの論理コンテナ |

### 5.2 リソース一覧

| リソース | リソース名 | SKU / プラン | 用途 |
|---|---|---|---|
| Static Web Apps | `swa-astro-site` | Standard | 静的サイトホスティング |
| Key Vault | `kv-astro-swa` | Standard | シークレット管理 |

### 5.3 命名規則

| プレフィックス | リソースタイプ |
|---|---|
| `rg-` | リソースグループ |
| `swa-` | Static Web Apps |
| `kv-` | Key Vault |

---

## 6. GitHub 構成

### 6.1 リポジトリ構造

```
astro_azure_swa/
├── .github/
│   └── workflows/
│       └── azure-static-web-apps.yml    # CI/CD ワークフロー
├── docs/
│   ├── architecture.md                  # 本ドキュメント
│   ├── architecture.svg                 # アーキテクチャ図
│   ├── phase-0-prerequisites.md         # Phase 0
│   ├── phase-1-astro-project.md         # Phase 1
│   ├── phase-2-github-setup.md          # Phase 2
│   ├── phase-3-azure-resources.md       # Phase 3
│   ├── phase-4-security.md              # Phase 4
│   ├── phase-5-cicd.md                  # Phase 5
│   ├── phase-6-deploy-verify.md         # Phase 6
│   └── phase-7-operations.md            # Phase 7
├── src/                                 # Astro ソースコード
├── public/                              # 静的アセット
├── staticwebapp.config.json             # SWA 認証・ルーティング設定
├── astro.config.mjs                     # Astro 設定
├── package.json
├── concept.md                           # コンテンツ設計書
└── README.md
```

### 6.2 ブランチ戦略

| ブランチ | デプロイ先 | トリガー |
|---|---|---|
| `main` | 本番環境 (Production) | push 時に自動デプロイ |
| PR (→ main) | ステージング環境 | PR オープン時に自動作成、マージ/クローズ時に自動削除 |
| `dev` | — | 開発用 (main への PR 元) |
| `feature/*` | — | 機能開発用の一時ブランチ |

### 6.3 GitHub Actions Secrets

| シークレット名 | 用途 | 備考 |
|---|---|---|
| `AZURE_STATIC_WEB_APPS_API_TOKEN` | SWA デプロイトークン | OIDC 方式でも必要 |
| `AZURE_CLIENT_ID` | Entra ID アプリ ID | OIDC 用 |
| `AZURE_TENANT_ID` | Entra ID テナント ID | OIDC 用 |
| `AZURE_SUBSCRIPTION_ID` | Azure サブスクリプション ID | OIDC 用 |

---

## 7. 開発環境構成

### 7.1 WSL + VS Code 構成

```
Windows
  └── WSL (Ubuntu)
        ├── Node.js v22.12+
        ├── npm
        ├── Azure CLI
        ├── SWA CLI
        ├── GitHub CLI
        └── Git
             │
             └── VS Code (Windows) ← Remote - WSL 拡張で接続
                   ├── Astro 拡張 (astro-build.astro-vscode)
                   ├── Azure Static Web Apps 拡張
                   ├── Azure Account 拡張
                   ├── Azure Resources 拡張
                   ├── GitHub Actions 拡張
                   └── GitHub Pull Requests 拡張
```

### 7.2 ローカル開発フロー

| コマンド | 用途 | ポート |
|---|---|---|
| `npm run dev` | Astro 開発サーバー (HMR) | 4321 |
| `npm run build` | 本番ビルド (dist/ 生成) | — |
| `swa start dist` | SWA エミュレータ (認証込み) | 4280 |

---

## 8. 非機能要件

### 8.1 可用性

| 項目 | 仕様 |
|---|---|
| SLA | 99.95% (Standard プラン) |
| CDN | Azure グローバルエッジネットワーク |
| リージョン | Japan East (東日本) |
| ステージング | PR ごとの自動ステージング環境 |

### 8.2 拡張性

本アーキテクチャは以下の拡張が可能である（本ハンズオンのスコープ外）。

| 拡張項目 | 実装方法 |
|---|---|
| API バックエンド | Azure Functions (マネージド) or Bring Your Own API |
| SSR 対応 | Astro SSR アダプタ + Azure Functions |
| IaC | Bicep / Terraform によるリソース管理 |
| 監視 | Application Insights 統合 |
| マルチリージョン | Azure Front Door + 複数 SWA インスタンス |
