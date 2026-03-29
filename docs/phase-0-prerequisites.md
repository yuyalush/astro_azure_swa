# Phase 0: 前提条件・環境準備

> **目的**: 開発に必要なツール・アカウント・VS Code 拡張機能をすべてセットアップし、以降の Phase にスムーズに進められる状態にする。  
> **所要時間目安**: 30〜60 分  
> **前提**: なし（本ハンズオンの最初のステップ）

---

## 目次

1. [アカウントの準備](#1-アカウントの準備)
2. [WSL 環境の確認](#2-wsl-環境の確認)
3. [Node.js のインストール](#3-nodejs-のインストール)
4. [CLI ツールのインストール](#4-cli-ツールのインストール)
5. [VS Code と拡張機能のセットアップ](#5-vs-code-と拡張機能のセットアップ)
6. [動作確認](#6-動作確認)
7. [完了チェックリスト](#7-完了チェックリスト)

---

## 1. アカウントの準備

以下のアカウントが必要です。未作成の場合はこのステップで作成してください。

### 1.1 Azure アカウント

Azure サブスクリプションが必要です。

- **無料アカウント**: [https://azure.microsoft.com/free](https://azure.microsoft.com/free)
  - 12 か月の無料サービス + $200 のクレジット（30 日間）
- **注意**: Phase 4 の Managed Identity 利用には Static Web Apps **Standard プラン**が必要です（月額約 $9/アプリ）

### 1.2 GitHub アカウント

ソースコード管理と CI/CD に使用します。

- **アカウント作成**: [https://github.com/signup](https://github.com/signup)
- GitHub Actions は公開リポジトリなら無料、プライベートリポジトリは月 2,000 分の無料枠あり

---

## 2. WSL 環境の確認

本ハンズオンは WSL (Windows Subsystem for Linux) 上で開発を行います。

### 2.1 WSL のインストール確認

ターミナル (PowerShell) で以下を実行します。

```bash
wsl --version
```

WSL がインストールされていない場合:

```bash
wsl --install
```

> 再起動が必要な場合があります。

### 2.2 Linux ディストリビューションの確認

```bash
wsl -l -v
```

Ubuntu がインストールされていることを確認します。未インストールの場合:

```bash
wsl --install -d Ubuntu
```

### 2.3 WSL に入る

以降のコマンドはすべて WSL (Linux) 内で実行します。

```bash
wsl
```

---

## 3. Node.js のインストール

Astro は Node.js v22.12.0 以上を必要とします。nvm (Node Version Manager) を使ったインストールを推奨します。

### 3.1 nvm のインストール

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

シェルを再起動するか、以下を実行:

```bash
source ~/.bashrc
```

### 3.2 Node.js のインストール

```bash
# LTS の最新版をインストール
nvm install --lts

# バージョンを確認 (v22.12.0 以上であること)
node -v
npm -v
```

> **なぜ v22.12.0 以上？**  
> Azure Static Web Apps の GitHub Actions ビルド環境がデフォルトで古い Node.js を使用するため、`package.json` の `engines` フィールドで最低バージョンを指定する必要があります。ローカル環境もこのバージョンに合わせておくことで、ビルドの不整合を防ぎます。

---

## 4. CLI ツールのインストール

### 4.1 Azure CLI

```bash
# Microsoft のリポジトリからインストール
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# バージョン確認
az version
```

### 4.2 Azure Static Web Apps CLI

```bash
npm install -g @azure/static-web-apps-cli

# バージョン確認
swa --version
```

### 4.3 GitHub CLI（推奨）

```bash
# Ubuntu / Debian
(type -p wget >/dev/null || (sudo apt update && sudo apt-get install wget -y)) \
  && sudo mkdir -p -m 755 /etc/apt/keyrings \
  && out=$(mktemp) && wget -nv -O$out https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  && cat $out | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
  && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
  && sudo apt update \
  && sudo apt install gh -y

# バージョン確認
gh --version
```

### 4.4 Git（確認）

WSL の Ubuntu にはデフォルトでインストールされています。

```bash
git --version

# 未インストールの場合
sudo apt update && sudo apt install git -y
```

Git のグローバル設定:

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

---

## 5. VS Code と拡張機能のセットアップ

### 5.1 VS Code のインストール

Windows 側に VS Code をインストールします（WSL 側ではなく Windows 側）。

- **ダウンロード**: [https://code.visualstudio.com/](https://code.visualstudio.com/)

### 5.2 WSL 拡張機能

VS Code から WSL 内のファイルを直接編集するために必要です。

```bash
# VS Code の拡張機能をコマンドラインからインストール (Windows 側のターミナルで実行)
code --install-extension ms-vscode-remote.remote-wsl
```

WSL 内から VS Code を起動する方法:

```bash
# WSL 内で、プロジェクトディレクトリに移動してから
code .
```

### 5.3 推奨拡張機能のインストール

以下の拡張機能をインストールします。VS Code のコマンドパレット (`Ctrl+Shift+P`) → `Extensions: Install Extension` でも検索可能です。

```bash
# Astro — .astro ファイルのシンタックスハイライト、IntelliSense、フォーマット
code --install-extension astro-build.astro-vscode

# Azure Static Web Apps — SWA リソースの作成・管理
code --install-extension ms-azuretools.vscode-azurestaticwebapps

# Azure Account — Azure へのサインインとサブスクリプション管理
code --install-extension ms-vscode.azure-account

# Azure Resources — Azure リソースの一覧表示・管理
code --install-extension ms-azuretools.vscode-azureresourcegroups

# GitHub Actions — ワークフローの構文チェック・実行状況の確認
code --install-extension github.vscode-github-actions

# GitHub Pull Requests — VS Code 内での PR 作成・レビュー
code --install-extension github.vscode-pull-request-github
```

### 5.4 各拡張機能の用途

| 拡張機能 | ID | 使用する Phase | 主な用途 |
|---|---|---|---|
| **WSL** | `ms-vscode-remote.remote-wsl` | 全 Phase | WSL 内のファイルを VS Code で編集 |
| **Astro** | `astro-build.astro-vscode` | Phase 1〜 | `.astro` ファイルの編集支援 |
| **Azure Static Web Apps** | `ms-azuretools.vscode-azurestaticwebapps` | Phase 3〜 | SWA リソースの GUI 操作 |
| **Azure Account** | `ms-vscode.azure-account` | Phase 3〜 | Azure サインイン管理 |
| **Azure Resources** | `ms-azuretools.vscode-azureresourcegroups` | Phase 3〜 | リソース一覧の確認 |
| **GitHub Actions** | `github.vscode-github-actions` | Phase 5〜 | ワークフロー実行状況の確認 |
| **GitHub Pull Requests** | `github.vscode-pull-request-github` | Phase 6〜 | PR の作成・管理 |

---

## 6. 動作確認

すべてのインストールが完了したら、以下のコマンドで動作を確認します。

### 6.1 ツールのバージョン確認

```bash
echo "=== Node.js ===" && node -v
echo "=== npm ===" && npm -v
echo "=== Azure CLI ===" && az version --query '"azure-cli"' -o tsv
echo "=== SWA CLI ===" && swa --version
echo "=== GitHub CLI ===" && gh --version
echo "=== Git ===" && git --version
```

### 6.2 Azure CLI のサインイン

```bash
az login
```

ブラウザが開き、Azure アカウントでサインインします。

```bash
# サインイン確認
az account show --query '{name:name, id:id, tenantId:tenantId}' -o table
```

> **WSL からブラウザが開かない場合**:  
> `az login --use-device-code` を使用してください。表示されるコードを https://microsoft.com/devicelogin に入力してサインインできます。

### 6.3 GitHub CLI のサインイン

```bash
gh auth login
```

プロンプトに従って認証します（GitHub.com → HTTPS → ブラウザ認証を推奨）。

```bash
# サインイン確認
gh auth status
```

---

## 7. 完了チェックリスト

以下がすべて ✅ になっていれば、Phase 0 は完了です。

| # | チェック項目 | 確認コマンド |
|---|---|---|
| 1 | WSL (Ubuntu) が動作している | `wsl -l -v` |
| 2 | Node.js v22.12.0 以上がインストールされている | `node -v` |
| 3 | npm がインストールされている | `npm -v` |
| 4 | Azure CLI がインストールされている | `az version` |
| 5 | Azure CLI でサインイン済み | `az account show` |
| 6 | SWA CLI がインストールされている | `swa --version` |
| 7 | GitHub CLI がインストールされている | `gh --version` |
| 8 | GitHub CLI でサインイン済み | `gh auth status` |
| 9 | Git が設定済み | `git config --global --list` |
| 10 | VS Code に推奨拡張機能がインストールされている | VS Code の拡張機能パネルで確認 |

---

**次のステップ**: [Phase 1: Astro プロジェクトの作成とローカル開発](phase-1-astro-project.md)
