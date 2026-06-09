# Claudeチャット動作規定

> **このファイルはClaudeチャット専用です。Claudeコード環境では読み込み禁止。**
> プロジェクト指示に `CLAUDE_ENV=chat` が含まれている場合にのみ読み込むこと。

## ファイル操作

- GitHub API を使用してファイルの読み書き・commitを行う
- プロジェクト指示（システムプロンプト）内の Personal Access Token を認証に使用する
- API経由のcommitでは `git pull` は不要（APIは常に最新のリポジトリ状態に対して操作する）

## UIツール

- セッション中にユーザーへ選択を求める場面では、チャット環境で利用可能な選択肢提示機能を使ってよい
- ただしセッション開始時の状況把握（`aipatto_folder_list` によるダッシュボード表示）後は選択UIを使わず、ユーザーの自然な入力を待つ

## project.md の自動取得

プレースホルダが残っている場合、プロジェクト指示（システムプロンプト）から以下を自動取得する:
- 「対象リポジトリ:」行からユーザー名とリポジトリ名を抽出する。形式: `- {ユーザー名}/{リポジトリ名}（Private、本番）`
- 「プロジェクト名:」行からプロジェクト名を抽出する。形式: `プロジェクト名: {プロジェクト名}`

## セッション開始フロー

CLAUDE.md のステップ完了後、AiPatto MCP の DB ツールで作業状況を把握しダッシュボードを表示する。状況の正は D1（user_folder / user_task / user_subtask / user_exec）であり、四層モデルと各ツールの一般仕様は directives_session「作業管理」節を参照する。

### 1. 状況の取得

- `aipatto_folder_list`（必要に応じ `include_closed`）で作業フォルダ（user_folder）一覧と各 folder 配下の task 件数を取得する。
- 個別 folder の中身（配下 task、さらに subtask の状態）が要るときは `aipatto_folder_get` / `aipatto_task_get` で取得する（＝DB 版ダッシュボード）。

### 2. ダッシュボード表示

ウェルカムメッセージに続けて取得結果を MD テーブルで表示する。folder が無ければ「作業フォルダはまだありません」と表示する。状態は subtask の is_draft/is_close と最新 exec の status から導出する（⏳ 未着手＝exec 無し、📝 レビュー待ち＝exec 完了・subtask 未クローズ）。

（例）

📂 作業フォルダ（user_folder）:

| フォルダ | 未完了 subtask | 概要 |
|---|---|---|
| FlexRoute | 3 | フルフローテスト一式 |

📋 未完了の依頼（user_subtask）:

| subtask | 所属 task | 状態 | 概要 |
|---|---|---|---|
| #12 | フルフローテスト | ⏳ 未着手 | フルフローテスト一式 |
| #13 | バグ修正 | 📝 レビュー待ち | バグ修正の確認 |

### 3. ユーザーの指示を待つ

ダッシュボード表示後、ユーザーの自然な入力を待つ。

- **作業フォルダ（folder）を指定された場合**: `aipatto_folder_get` で当該 user_folder と配下 task / subtask を読み前回状態を把握する。body 等にソースコード用リポが記載されていれば、そのリポの CLAUDE.md・仕様書 MD を全文読み最重要資料とする。
- **新規作業フォルダの作成を指示された場合**: `aipatto_folder_create`（title / purpose / body）で user_folder を作成する。詳細ヒアリングは作成後に行う。

> 移行期フォールバック（非推奨）: ファイルベースの `_claude/tasks/` 収集・各フォルダ `_context.md` 走査・`aipatto_dashboard_get` は当面残置するが、状況把握の正は DB ツールとする。

## タスクキュー（作成・レビュー）

依頼と実行は DB の四層で扱う（一般仕様は directives_session「作業管理」節を参照）。本節は Chat 側の dev 固有運用に限る。

### 依頼の起票（Chat → Code）

ユーザーの要望が Code で実行すべき作業と判断した場合:

1. `aipatto_subtask_create` で user_subtask を作成する（description に成果物の内容を確定して記載＝委任時の指示品質）。厳格ツリーゆえ先に `aipatto_folder_create` で user_folder、その配下に `aipatto_task_create` で user_task を作り、`user_task_seq` で subtask を所属させる（各親必須）。
2. Code への受け渡しは、対象 user_subtask を指す最小限のプロンプトをユーザーに提示する（directives_session のハンドオフ運用に従い、本文に指示を書かずレコードを指す）。

### レビュー（Code の実行後）

レビューはセッション開始時に自動実行せず、ユーザーの指示時に実施する。

1. 対象 user_subtask の最新 `user_exec`（result / status / qa）を `aipatto_subtask_get` で読む。
2. **現物確認**: perform に記録された PR / commit の diff を GitHub API で実取得し変更ファイルと突合する（報告だけで完結させない）。
3. 問題なければ `aipatto_subtask_complete` でクローズする（is_close）。ソースコード用リポの PR マージ・ブランチ削除・デプロイは Chat が実施する（dev 固有・据え置き）。
4. 問題があれば修正依頼を新たな user_subtask として起票する。

> 移行期フォールバック（非推奨）: `_claude/tasks/{pending,review,done}` の指示ファイル ライフサイクルおよび `ref/.ref.md` は当面残置するが、起票・レビューの正は DB ツールとする。



## 指示ファイルプロンプトの書式（2026-04-28 ルール追加）

ソースコード用リポへの指示プロンプトをユーザーへ提示する際、冒頭の「## 指示概要」行には**タスクファイル名と指示番号を併記**する。

書式:

```
## 指示概要(指示番号：<タスクファイル名> INSTR-N)
- セッション種別: 新セッション
- リポジトリ: tatugmad/<repo>
- タスクファイル: <フルパス>
```

例:

```
## 指示概要(指示番号：task-FIX-159_signup-rename.md INSTR-2)
- セッション種別: 新セッション
- リポジトリ: tatugmad/aipatto-server
- タスクファイル: _tasks/pending/task-FIX-159_signup-rename.md
```

- タスクファイル名はフルパスではなく**ファイル名のみ**（`task-FIX-159_signup-rename.md`）
- INSTR-N の N は同一タスク内の指示通し番号（INSTR-1 から開始）
- 「タスクファイル:」行はフルパスを記載

**目的**: 複数タスク並行や時間経過後に、指示番号だけでどのタスクのどの指示か追跡可能にする。Code 完了報告（rule-code.md）と書式が対応する。
