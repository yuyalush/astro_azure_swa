# Phase 2: GitHub リポジトリのセットアップ

> **目的**: プロジェクトのソースコードを GitHub リポジトリで管理し、CI/CD パイプラインの基盤を整える。  
> **所要時間目安**: 15〜20 分  
> **前提**: [Phase 1](phase-1-astro-project.md) が完了していること

---

## 目次

1. [Git リポジトリの初期化](#1-git-リポジトリの初期化)
2. [.gitignore の確認・設定](#2-gitignore-の確認設定)
3. [GitHub リポジトリの作成](#3-github-リポジトリの作成)
4. [初回コミットと push](#4-初回コミットと-push)
5. [ブランチ戦略の設定](#5-ブランチ戦略の設定)
6. [完了チェックリスト](#6-完了チェックリスト)

---

## 1. Git リポジトリの初期化

プロジェクトディレクトリで Git リポジトリを初期化します。

```bash
cd ~/projects/astro_azure_swa

# Git リポジトリを初期化
git init

# デフォルトブランチを main に設定
git branch -M main
```

---

## 2. .gitignore の確認・設定

Astro CLI で生成された `.gitignore` を確認し、必要に応じて追記します。

### 2.1 現在の .gitignore を確認

```bash
cat .gitignore
```

### 2.2 推奨 .gitignore

以下の内容が含まれていることを確認してください。不足している項目があれば追記します。

```gitignore
# Astro ビルド成果物
dist/
.astro/

# 依存関係
node_modules/

# 環境変数 (シークレットを含む可能性がある)
.env
.env.*
!.env.example

# OS ファイル
.DS_Store
Thumbs.db

# エディタ
.vscode/*
!.vscode/extensions.json
!.vscode/settings.json

# ログ
npm-debug.log*
```

### 2.3 .env.example の作成（オプション）

環境変数のテンプレートを用意しておくと、他の開発者が参加しやすくなります。

```bash
cat > .env.example << 'EOF'
# Azure Static Web Apps の設定例
# 実際の値は .env ファイルに記載 (.gitignore で除外済み)
PUBLIC_SITE_TITLE=Astro Azure SWA
EOF
```

---

## 3. GitHub リポジトリの作成

### 方法 A: GitHub CLI を使用する（推奨）

```bash
# パブリックリポジトリを作成して origin に設定
gh repo create astro_azure_swa --public --source=. --remote=origin
```

> **プライベートリポジトリにする場合**: `--public` を `--private` に変更。ただし、GitHub Actions の無料枠 (2,000 分/月) に注意。

出力例:
```
✓ Created repository <YOUR_USERNAME>/astro_azure_swa on GitHub
✓ Added remote https://github.com/<YOUR_USERNAME>/astro_azure_swa.git
```

### 方法 B: GitHub Web UI を使用する

1. [https://github.com/new](https://github.com/new) にアクセス
2. Repository name: `astro_azure_swa`
3. Public を選択
4. README / .gitignore / License は追加しない（ローカルに既にあるため）
5. **Create repository** をクリック

ローカルにリモートを追加:

```bash
git remote add origin https://github.com/<YOUR_USERNAME>/astro_azure_swa.git
```

### 3.1 リモートの確認

```bash
git remote -v
```

出力例:
```
origin  https://github.com/<YOUR_USERNAME>/astro_azure_swa.git (fetch)
origin  https://github.com/<YOUR_USERNAME>/astro_azure_swa.git (push)
```

---

## 4. 初回コミットと push

### 4.1 ステージングとコミット

```bash
# すべてのファイルをステージング
git add .

# コミット
git commit -m "Initial commit: Astro project scaffold"
```

### 4.2 コミット内容の確認

```bash
# コミットログを確認
git log --oneline

# 追跡されているファイルを確認
git ls-files
```

> **確認ポイント**: `node_modules/`、`dist/`、`.env` が含まれていないことを確認してください。

### 4.3 GitHub に push

```bash
git push -u origin main
```

### 4.4 GitHub 上で確認

```bash
# ブラウザでリポジトリを開く
gh repo view --web
```

リポジトリページでファイルが正しくアップロードされていることを確認します。

---

## 5. ブランチ戦略の設定

本ハンズオンでは以下のブランチ戦略を採用します。

### 5.1 ブランチの役割

```
main ─────────────────────────→ 本番 (Production)
  │                                ↑
  └── dev ─── feature/* ──── PR ──┘
       │        (機能開発)
       │
       └── 開発用統合ブランチ
```

| ブランチ | 用途 | デプロイ先 |
|---|---|---|
| `main` | 本番リリース用 | Azure SWA 本番環境 |
| `dev` | 開発統合ブランチ | — (PR 経由で main にマージ) |
| `feature/*` | 機能開発用 | — |

### 5.2 dev ブランチの作成

```bash
# dev ブランチを作成して push
git checkout -b dev
git push -u origin dev

# main ブランチに戻る
git checkout main
```

### 5.3 GitHub のブランチ保護ルール（推奨）

`main` ブランチへの直接 push を防止し、PR 経由でのマージを必須にします。

```bash
# GitHub CLI でブランチ保護ルールを設定
gh api repos/{owner}/{repo}/branches/main/protection \
  -X PUT \
  -f 'required_pull_request_reviews[dismiss_stale_reviews]=true' \
  -f 'required_pull_request_reviews[required_approving_review_count]=1' \
  -F 'enforce_admins=false' \
  -F 'required_status_checks=null' \
  -F 'restrictions=null'
```

> **注意**: ブランチ保護ルールの設定はオプションです。一人で進めるハンズオンの場合は、設定せずに `main` への直接 push で進めても問題ありません。

### 5.4 開発ワークフローの流れ

Phase 5 以降で使用する実際の開発フローです:

```bash
# 1. dev ブランチから feature ブランチを作成
git checkout dev
git checkout -b feature/add-about-page

# 2. 変更を加えてコミット
# ... ファイルを編集 ...
git add .
git commit -m "Add about page"

# 3. push して PR を作成
git push -u origin feature/add-about-page
gh pr create --base main --title "Add about page" --body "About ページを追加"

# 4. PR がマージされると、GitHub Actions が自動デプロイ
# 5. マージ後、feature ブランチを削除
git checkout dev
git branch -d feature/add-about-page
```

---

## 6. 完了チェックリスト

以下がすべて ✅ になっていれば、Phase 2 は完了です。

| # | チェック項目 | 確認方法 |
|---|---|---|
| 1 | Git リポジトリが初期化されている | `git status` |
| 2 | `.gitignore` が適切に設定されている | `cat .gitignore` |
| 3 | GitHub リポジトリが作成されている | `gh repo view` |
| 4 | `main` ブランチにソースコードが push されている | `gh repo view --web` |
| 5 | `dev` ブランチが作成されている | `git branch -a \| grep dev` |
| 6 | `node_modules/` や `dist/` がリポジトリに含まれていない | `git ls-files \| grep -E "node_modules\|dist"` (出力なし) |

---

**前のステップ**: [Phase 1: Astro プロジェクトの作成とローカル開発](phase-1-astro-project.md)  
**次のステップ**: [Phase 3: Azure リソースの作成](phase-3-azure-resources.md)
