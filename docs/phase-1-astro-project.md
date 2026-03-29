# Phase 1: Astro プロジェクトの作成とローカル開発

> **目的**: Astro プロジェクトをスキャフォールドし、ローカル環境での開発・プレビュー・SWA エミュレーションができる状態にする。  
> **所要時間目安**: 20〜30 分  
> **前提**: [Phase 0](phase-0-prerequisites.md) が完了していること

---

## 目次

1. [作業ディレクトリの準備](#1-作業ディレクトリの準備)
2. [Astro プロジェクトの作成](#2-astro-プロジェクトの作成)
3. [プロジェクト構成の理解](#3-プロジェクト構成の理解)
4. [package.json の設定](#4-packagejson-の設定)
5. [ローカル開発サーバーの起動](#5-ローカル開発サーバーの起動)
6. [ビルドの確認](#6-ビルドの確認)
7. [SWA CLI によるローカルエミュレーション](#7-swa-cli-によるローカルエミュレーション)
8. [完了チェックリスト](#8-完了チェックリスト)

---

## 1. 作業ディレクトリの準備

WSL に入り、プロジェクトを作成するディレクトリに移動します。

```bash
# WSL に入る (既に WSL 内であればスキップ)
wsl

# ホームディレクトリに移動
cd ~

# プロジェクト用のディレクトリを作成 (任意)
mkdir -p ~/projects
cd ~/projects
```

---

## 2. Astro プロジェクトの作成

### 2.1 Astro CLI でプロジェクトをスキャフォールド

```bash
npm create astro@latest astro_azure_swa
```

対話型ウィザードが起動します。以下の選択を推奨します:

| プロンプト | 推奨選択 | 理由 |
|---|---|---|
| How would you like to start? | `Use blog template` または `Include sample files` | サンプルページがあると動作確認しやすい |
| Do you plan to write TypeScript? | `Yes` | 型安全な開発のため |
| How strict should TypeScript be? | `Strict` | エラーの早期検出のため |
| Install dependencies? | `Yes` | 手動インストールの手間を省く |
| Initialize a new git repository? | `No` | Phase 2 で手動設定するため |

### 2.2 プロジェクトディレクトリに移動

```bash
cd astro_azure_swa
```

### 2.3 VS Code で開く

```bash
code .
```

> WSL から `code .` を実行すると、VS Code が Remote - WSL モードでプロジェクトを開きます。ステータスバーの左下に `WSL: Ubuntu` と表示されていれば正常です。

---

## 3. プロジェクト構成の理解

Astro CLI が生成するプロジェクトの構成を確認します。

```
astro_azure_swa/
├── src/
│   ├── pages/              # ページコンポーネント
│   │   └── index.astro     # トップページ (ファイルベースルーティング)
│   ├── layouts/             # レイアウトコンポーネント
│   │   └── Layout.astro    # ベースレイアウト
│   └── components/          # 再利用可能なコンポーネント
│       └── Card.astro      # サンプルコンポーネント
├── public/                  # 静的アセット (ビルド時にそのままコピーされる)
│   └── favicon.svg
├── astro.config.mjs         # Astro 設定ファイル
├── package.json             # 依存関係・スクリプト
└── tsconfig.json            # TypeScript 設定
```

### 主要ファイルの役割

#### `src/pages/`
- **ファイルベースルーティング**: このディレクトリ内の `.astro` / `.md` ファイルが URL パスに対応
- `src/pages/index.astro` → `/`
- `src/pages/about.astro` → `/about`
- `src/pages/blog/post-1.md` → `/blog/post-1`

#### `astro.config.mjs`
```javascript
import { defineConfig } from 'astro/config';

export default defineConfig({
  // デフォルトでは SSG (Static Site Generation) モードで動作
  // Azure SWA にデプロイする場合、特別な設定は不要
});
```

#### `public/`
- ビルド時に処理されず、そのまま `dist/` にコピーされるファイル
- favicon、robots.txt、OGP 画像などを配置

---

## 4. package.json の設定

### 4.1 engines フィールドの追加

Azure SWA の GitHub Actions ビルド環境との互換性のため、`package.json` に `engines` フィールドを追加します。

```bash
# package.json を開く
code package.json
```

以下のフィールドを追加してください:

```json
{
  "name": "astro_azure_swa",
  "type": "module",
  "version": "0.0.1",
  "engines": {
    "node": ">=22.12.0"
  },
  "scripts": {
    "dev": "astro dev",
    "build": "astro build",
    "preview": "astro preview",
    "astro": "astro"
  },
  "dependencies": {
    "astro": "^5.x"
  }
}
```

> **重要**: `engines` フィールドがないと、SWA のビルド環境がデフォルトの古い Node.js で Astro をビルドしようとし、失敗します。これは Astro + SWA の**既知の問題**です。

### 4.2 変更の確認

```bash
cat package.json | grep -A 2 engines
```

出力例:
```
  "engines": {
    "node": ">=22.12.0"
  },
```

---

## 5. ローカル開発サーバーの起動

### 5.1 開発サーバーの起動

```bash
npm run dev
```

以下のような出力が表示されます:

```
 astro  v5.x.x ready in XXms

┃ Local    http://localhost:4321/
┃ Network  use --host to expose

watching for file changes...
```

### 5.2 ブラウザで確認

ブラウザで `http://localhost:4321` にアクセスしてください。

> **WSL 環境での注意**: WSL 内で起動した開発サーバーには、Windows のブラウザから `localhost` でアクセスできます。

### 5.3 ホットリロード (HMR) の確認

1. VS Code で `src/pages/index.astro` を開く
2. テキストを変更して保存 (`Ctrl+S`)
3. ブラウザが自動的に更新されることを確認

### 5.4 開発サーバーの停止

`Ctrl+C` で停止します。

---

## 6. ビルドの確認

### 6.1 本番ビルドの実行

```bash
npm run build
```

以下のような出力が表示されます:

```
 astro  v5.x.x building for production...
...
 generating static routes
▶ src/pages/index.astro
  └─ /index.html (+XXms)
...
 build complete!
```

### 6.2 ビルド成果物の確認

```bash
# dist/ ディレクトリの内容を確認
ls -la dist/

# HTML ファイルが生成されていること
find dist/ -name "*.html"
```

`dist/` ディレクトリには以下が生成されます:
- HTML ファイル（各ページに対応）
- CSS ファイル（最適化済み）
- JavaScript ファイル（必要な場合のみ）
- `public/` からコピーされた静的アセット

> **ポイント**: この `dist/` ディレクトリの内容が、Azure SWA にデプロイされるファイルになります。GitHub Actions ワークフローの `output_location: "dist"` はこのディレクトリを指しています。

### 6.3 ビルドのプレビュー

```bash
npm run preview
```

ビルド済みのサイトを `http://localhost:4321` でプレビューできます。本番と同じ静的ファイルが配信されます。

---

## 7. SWA CLI によるローカルエミュレーション

SWA CLI を使うと、Azure SWA の認証機能を含むローカルエミュレーション環境でテストできます。

### 7.1 SWA エミュレータの起動

```bash
# まずビルドを実行
npm run build

# SWA エミュレータを起動
swa start dist
```

出力例:
```
Azure Static Web Apps emulator started at http://localhost:4280.
```

### 7.2 エミュレータの機能

SWA エミュレータでは以下の機能がローカルで利用可能です:

| エンドポイント | 機能 |
|---|---|
| `http://localhost:4280` | サイトのトップページ |
| `http://localhost:4280/.auth/login/github` | GitHub ログイン (モック) |
| `http://localhost:4280/.auth/login/aad` | Entra ID ログイン (モック) |
| `http://localhost:4280/.auth/me` | ログインユーザー情報 |
| `http://localhost:4280/.auth/logout` | ログアウト |

> **モック認証**: ローカルエミュレータでは実際の OAuth フローは実行されず、ダミーのユーザー情報でログインをシミュレートします。認証・認可のルーティングテストに便利です。

### 7.3 開発サーバーと組み合わせる

ビルドなしで、Astro の開発サーバー (HMR 付き) と SWA エミュレータを同時に使うこともできます:

```bash
# 別のターミナルで Astro 開発サーバーを起動
npm run dev

# SWA CLI から Astro 開発サーバーをプロキシ
swa start http://localhost:4321
```

この構成では:
- コード変更時に HMR が効く
- SWA の認証エンドポイントも利用可能
- API のテストも可能（API バックエンドを追加した場合）

### 7.4 エミュレータの停止

`Ctrl+C` で停止します。

---

## 8. 完了チェックリスト

以下がすべて ✅ になっていれば、Phase 1 は完了です。

| # | チェック項目 | 確認方法 |
|---|---|---|
| 1 | Astro プロジェクトが作成されている | `ls src/pages/index.astro` |
| 2 | `npm run dev` でローカル開発サーバーが起動する | `http://localhost:4321` にアクセス |
| 3 | HMR (ホットリロード) が動作する | ファイル保存でブラウザが自動更新 |
| 4 | `npm run build` で `dist/` にビルド成果物が生成される | `ls dist/index.html` |
| 5 | `package.json` に `engines` フィールドが設定されている | `cat package.json \| grep engines` |
| 6 | `swa start dist` でローカルエミュレーションが動作する | `http://localhost:4280` にアクセス |
| 7 | VS Code が WSL モードでプロジェクトを開いている | ステータスバーに `WSL: Ubuntu` |

---

**前のステップ**: [Phase 0: 前提条件・環境準備](phase-0-prerequisites.md)  
**次のステップ**: [Phase 2: GitHub リポジトリのセットアップ](phase-2-github-setup.md)
