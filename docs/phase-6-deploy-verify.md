# Phase 6: デプロイと動作検証

> **目的**: 実際にデプロイを実行し、サイトの動作・認証フロー・Key Vault 連携・ステージング環境を検証する。  
> **所要時間目安**: 30〜45 分  
> **前提**: [Phase 5](phase-5-cicd.md) が完了していること

---

## 目次

1. [本番デプロイの実行](#1-本番デプロイの実行)
2. [GitHub Actions の実行確認](#2-github-actions-の実行確認)
3. [デプロイされたサイトの確認](#3-デプロイされたサイトの確認)
4. [認証フローの検証](#4-認証フローの検証)
5. [Key Vault 連携の検証](#5-key-vault-連携の検証)
6. [PR ステージング環境の検証](#6-pr-ステージング環境の検証)
7. [トラブルシューティング](#7-トラブルシューティング)
8. [完了チェックリスト](#8-完了チェックリスト)

---

## 1. 本番デプロイの実行

Phase 4 と Phase 5 でコミットした変更を main ブランチに push し、GitHub Actions による自動デプロイを実行します。

### 1.1 push 前の確認

```bash
# 現在のブランチを確認
git branch --show-current
# main であることを確認

# コミット履歴を確認
git log --oneline -5

# push 対象の変更を確認
git diff origin/main --stat
```

### 1.2 main ブランチに push

```bash
git push origin main
```

> この push をトリガーに、GitHub Actions ワークフローが自動的に起動します。

---

## 2. GitHub Actions の実行確認

### 2.1 ブラウザで確認

```bash
# GitHub Actions のページを開く
gh run list --limit 1

# 最新の実行をブラウザで開く
gh run view --web
```

または、GitHub リポジトリの **Actions** タブを直接開きます:
```
https://github.com/<YOUR_USERNAME>/astro_azure_swa/actions
```

### 2.2 CLI で実行状況を監視

```bash
# 実行中のワークフローを監視 (完了するまで待機)
gh run watch
```

出力例:
```
✓ main Build and Deploy · 1234567890
Triggered via push about 2 minutes ago

JOBS
✓ Build and Deploy in 1m30s (ID 12345)

✓ Run Build and Deploy (1234567890) completed with 'success'
```

### 2.3 ジョブの詳細ログ

ビルドが失敗した場合、詳細ログを確認します:

```bash
# 最新の実行のログを表示
gh run view --log
```

### 2.4 VS Code からの確認

GitHub Actions 拡張機能がインストールされている場合:

1. VS Code のアクティビティバーで **GitHub Actions** アイコンをクリック
2. ワークフロー実行の一覧が表示される
3. `Azure Static Web Apps CI/CD` の最新実行をクリックして詳細を確認

---

## 3. デプロイされたサイトの確認

### 3.1 デフォルト URL の取得

```bash
# SWA のホスト名を取得
DEFAULT_HOST=$(az staticwebapp show \
  --name swa-astro-site \
  --resource-group rg-astro-swa \
  --query "defaultHostname" -o tsv)

echo "サイト URL: https://$DEFAULT_HOST"
```

### 3.2 ブラウザでアクセス

```bash
# ブラウザで開く (WSL の場合)
xdg-open "https://$DEFAULT_HOST" 2>/dev/null || echo "ブラウザで https://$DEFAULT_HOST にアクセスしてください"
```

### 3.3 確認ポイント

- [ ] サイトが正しく表示されるか
- [ ] CSS やアセットが正しく読み込まれているか
- [ ] ナビゲーションが動作するか
- [ ] HTTPS で接続されているか（ブラウザのアドレスバーに鍵アイコン）

### 3.4 curl での確認

```bash
# HTTP ステータスコードの確認
curl -s -o /dev/null -w "%{http_code}" "https://$DEFAULT_HOST"
# 200 が返ることを確認

# HTTPS リダイレクトの確認
curl -s -o /dev/null -w "%{http_code}" "http://$DEFAULT_HOST"
# 301 または 302 が返ることを確認 (HTTPS にリダイレクト)
```

---

## 4. 認証フローの検証

`staticwebapp.config.json` で設定した認証・認可のフローを検証します。

### 4.1 ログインフローの確認

1. ブラウザで `https://<DEFAULT_HOST>/login` にアクセス
2. GitHub の OAuth 認証画面にリダイレクトされることを確認
3. GitHub アカウントでログイン
4. サイトにリダイレクトバックされることを確認

### 4.2 ユーザー情報の確認

ログイン後、以下の URL にアクセスして認証情報を確認します:

```
https://<DEFAULT_HOST>/.auth/me
```

レスポンス例:
```json
{
  "clientPrincipal": {
    "identityProvider": "github",
    "userId": "xxxxxxxxxxxx",
    "userDetails": "your-github-username",
    "userRoles": ["anonymous", "authenticated"],
    "claims": [...]
  }
}
```

### 4.3 認可の確認

```bash
# 未認証状態で admin ページにアクセス (ログアウト後に実行)
curl -s -o /dev/null -w "%{http_code}" "https://$DEFAULT_HOST/admin/"
# 302 (ログインページにリダイレクト) が返ることを確認
```

### 4.4 ログアウトフローの確認

1. `https://<DEFAULT_HOST>/logout` にアクセス
2. ログアウトが実行される
3. `https://<DEFAULT_HOST>/.auth/me` にアクセスし、`clientPrincipal` が `null` であることを確認

---

## 5. Key Vault 連携の検証

### 5.1 Application Settings の確認

```bash
az staticwebapp appsettings list \
  --name swa-astro-site \
  --resource-group rg-astro-swa
```

`EXAMPLE_SECRET` が Key Vault 参照として設定されていることを確認します。

### 5.2 Key Vault 参照の解決状態

Azure Portal で確認するのが最も確実です:

1. SWA リソースを開く
2. **設定** → **構成** をクリック
3. `EXAMPLE_SECRET` の横に **緑のチェックマーク** が表示されていれば、Key Vault 参照が正常に解決されている

> **赤い × マーク** が表示されている場合は、以下を確認:
> - Managed Identity が有効化されているか
> - Key Vault の RBAC で `Key Vault Secrets User` ロールが付与されているか
> - Key Vault 名とシークレット名が正しいか

---

## 6. PR ステージング環境の検証

### 6.1 テスト用の変更を作成

```bash
# dev ブランチに切り替え
git checkout dev

# main の変更を取り込む
git merge main

# テスト用の変更を追加
echo '<!-- Staging test -->' >> src/pages/index.astro

# コミット
git add .
git commit -m "Test: staging environment verification"

# push
git push origin dev
```

### 6.2 PR を作成

```bash
gh pr create \
  --base main \
  --head dev \
  --title "Test: Staging environment verification" \
  --body "ステージング環境の動作確認テスト用 PR"
```

### 6.3 ステージング環境の確認

1. GitHub リポジトリの **Actions** タブで、PR トリガーのワークフローが実行されていることを確認
2. ワークフロー完了後、PR のコメントにステージング URL が投稿される

```bash
# PR の状態確認
gh pr view --web
```

3. ステージング URL にアクセスし、変更が反映されていることを確認

### 6.4 ステージング環境の削除確認

PR をマージまたはクローズして、ステージング環境が自動削除されることを確認します。

```bash
# PR をクローズ (テスト用なのでマージしない)
gh pr close
```

GitHub Actions の **Close Pull Request** ジョブが実行され、ステージング環境が削除されることを確認します。

### 6.5 テスト用変更のリセット

```bash
# dev ブランチの変更をリセット
git checkout dev
git reset --hard origin/main
git push --force-with-lease origin dev

# main ブランチに戻る
git checkout main
```

---

## 7. トラブルシューティング

### 7.1 GitHub Actions ビルドが失敗する

#### 症状: Node.js バージョンエラー

```
Error: The engine "node" is incompatible with this module.
```

**対処法**:
```bash
# package.json の engines フィールドを確認
cat package.json | grep -A 2 engines

# 設定されていない場合は追加
# "engines": { "node": ">=22.12.0" }
```

#### 症状: OIDC トークン取得の失敗

```
Error: Unable to get ACTIONS_ID_TOKEN_REQUEST_URL env variable
```

**対処法**:
- ワークフローの `permissions` に `id-token: write` が含まれているか確認
- Federated Credential の `subject` がリポジトリ名・ブランチ名と一致しているか確認

### 7.2 デプロイ後に 404 エラー

**対処法**:
```bash
# output_location が正しいか確認
grep "output_location" .github/workflows/azure-static-web-apps.yml
# "dist" であることを確認

# ローカルビルドで dist/ にファイルが生成されるか確認
npm run build && ls dist/
```

### 7.3 認証が動作しない

**対処法**:
```bash
# staticwebapp.config.json の JSON 構文を確認
python3 -m json.tool staticwebapp.config.json

# SWA エミュレータでローカルテスト
swa start dist
# http://localhost:4280/.auth/login/github にアクセス
```

### 7.4 Key Vault 参照が解決されない

**対処法**:
```bash
# Managed Identity の状態確認
az staticwebapp identity show \
  --name swa-astro-site \
  --resource-group rg-astro-swa

# Key Vault のロール割り当て確認
PRINCIPAL_ID=$(az staticwebapp identity show \
  --name swa-astro-site -g rg-astro-swa \
  --query "principalId" -o tsv)

az role assignment list \
  --assignee "$PRINCIPAL_ID" \
  --query '[].{role:roleDefinitionName}' -o table
# "Key Vault Secrets User" が含まれているか確認
```

---

## 8. 完了チェックリスト

以下がすべて ✅ になっていれば、Phase 6 は完了です。

| # | チェック項目 | 確認方法 |
|---|---|---|
| 1 | GitHub Actions ワークフローが成功している | `gh run list --limit 1` で `completed/success` |
| 2 | デプロイされたサイトにブラウザからアクセスできる | `https://<DEFAULT_HOST>` に HTTPS でアクセス |
| 3 | サイトの表示 (HTML/CSS/アセット) が正しい | ブラウザで目視確認 |
| 4 | `/login` → GitHub 認証画面にリダイレクトされる | ブラウザで確認 |
| 5 | `/.auth/me` でユーザー情報が返される | ログイン後にアクセス |
| 6 | `/admin/` が未認証ユーザーをリダイレクトする | ログアウト後にアクセス |
| 7 | `/logout` でログアウトできる | ブラウザで確認 |
| 8 | Key Vault 参照が正常に解決されている | Azure Portal の構成画面で ✅ マーク |
| 9 | PR 作成時にステージング環境が自動デプロイされる | PR 作成後に確認 |
| 10 | PR クローズ時にステージング環境が削除される | PR クローズ後に確認 |

---

**前のステップ**: [Phase 5: GitHub Actions CI/CD パイプラインの構成](phase-5-cicd.md)  
**次のステップ**: [Phase 7: 運用・メンテナンスガイド](phase-7-operations.md)
