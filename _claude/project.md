<!-- schema-version: 1.0.0 -->
<!-- schema-start -->
## ファイル定義
このファイルはプロジェクト固有の環境情報を記録する。初回セットアップ時にプレースホルダを置換して確定する。

### 必須セクション
- アカウント: GitHubアカウント名、リポジトリ名
- 環境: PC構成、Claudeプラン、実行環境
- ダウンロードURL基底パス: GitHub raw URLのベース
- プロジェクト固有の注意事項: 環境固有の留意点
<!-- schema-end -->

# プロジェクト固有情報

## アカウント

- GitHubアカウント: {{GITHUB_USER}}
- リポジトリ: `{{GITHUB_USER}}/{{REPO_NAME}}`（Private）

## 環境

- PC: （初回セットアップで記録）
- Claude: 有料プラン（ブラウザ版 claude.ai を使用）
- 実行環境: claude.ai/code のクラウドVM（セッションごとに起動・消滅）

## ダウンロードURL基底パス

```
https://github.com/{{GITHUB_USER}}/{{REPO_NAME}}/raw/main/
```

## プロジェクト固有の注意事項

（必要に応じて追記）
