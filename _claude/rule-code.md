# Claudeコード動作規定

> **このファイルはClaudeコード専用です。Claudeチャット環境では読み込み禁止。**
> CLAUDE.mdの環境判定で `CLAUDE_ENV=chat` が検出されなかった場合にのみ読み込むこと。

## ファイル操作

- gitコマンドを使用してファイルの読み書き・commit & pushを行う
- commit & push の前に必ず `git pull` を実行し、最新状態を取得すること
- commit メッセージの本文に session URL を含める（取得可能な場合）:
  ```
  - session: https://claude.ai/code/session_XXXXX
  ```

## CC Web（ブラウザ版 Claude Code）のpush制約

CC Webにはプラットフォームレベルのpush制約がある:

- `claude/` プレフィックス付きブランチにしかpushできない（`git push origin main` は不可）
- ブランチ名はCC Webが自動生成するものをそのまま使用すること
- PR作成はCC Web UIの「PR作成」ボタンで行う（gh CLIは使用不可）
- マージ済みブランチに追加コミットしないこと

### OfficeWorkリポでの作業

OfficeWorkリポ自体のファイル（MDファイル等）を編集する場合:

- ブランチを作成し、編集内容をcommit & push → PR作成ボタンでPR作成
- マージ・ブランチ削除はClaude Chat側が実施する

### ソースコード用リポでの作業

ソースコード用リポ（_context.mdに記載）で作業する場合:

- **コード変更**: ブランチ作成 → commit & push → PR作成ボタンでPR作成
- **タスクファイル操作（_tasks/）**: `git checkout main && git pull` → mainに直接commit & push
- コード変更とタスクファイル操作は別々のcommitで行うこと
- マージ・ブランチ削除・デプロイはClaude Chat側が実施する

## 一時ファイル

- Pythonスクリプト等の一時ファイルは作業フォルダ内の `_temp/` に作成する
- `_temp/` は `.gitignore` で除外し、リポジトリには commit しない

## VM環境

### 特性

- クラウドVMはセッションごとに使い捨て。予告なく消滅しうる
- commit & push していないファイルはVM消滅とともに失われる。常にpush済み状態を維持すること
- ツールのインストール状態もセッションをまたいで保持されない

### ツールの自動導入

`.devcontainer/devcontainer.json` が存在する場合、VM起動時にそれに基づき必要なツールが自動インストールされる。

セッション中に新たなツールが必要になった場合:
1. VM上に手動でインストールして作業を続行する
2. 今後も継続的に必要であれば、devcontainer.json への追加（未作成の場合は新規作成）をユーザーに提案する

## UIツール

- ユーザーへの確認が必要な場面では **AskUserQuestionツール** を使うこと（テキストの番号リストで代替しない）
- Agentツール呼び出し時は `description` を日本語で記述し、プロンプトにも「日本語で回答せよ」を含めること（UI表示の日本語化）

## DOCX編集の注意

- DOCX（OOXML）のXML編集にPython ElementTreeを使ってはならない。ElementTreeはXMLを書き直す際に名前空間宣言を変更し、Word XMLが無効になる。必ず文字列置換（`str.replace` / `re.sub`）で行うこと

## project.md の自動取得

プレースホルダが残っている場合、`git remote -v` を実行しリモートURLからユーザー名とリポジトリ名を抽出する。URL形式: `https://github.com/{ユーザー名}/{リポジトリ名}.git`

## セッション開始フロー

CLAUDE.md のステップ完了後、`aipatto_todo_list` で自分宛の未実行依頼（user_todo）を把握する（三層モデル・各ツールの一般仕様は directives_session「作業管理」節を参照）。

**未実行の依頼がある場合:**

1. ウェルカムメッセージに続けて、未完了 todo の一覧を表示する（`aipatto_todo_list`。必要に応じ所属 user_task を併記）。
2. 最優先（または最古）の 1 件を `aipatto_todo_get` で読み込み、内容をユーザーに要約し「この依頼を実行してよいですか？」と確認する。
3. **ユーザーの承認なしに実行しない。**
4. 承認されたら `aipatto_perform_create` で実行インスタンス（user_todo_perform）を開始し実行する。途中の確認が要る場合は `aipatto_perform_respond`（qa）で質問返しする。
5. 実行の結果・成果物・問題は `aipatto_perform_update`（result / status）に記録する（ファイルの「実行結果」セクションに代えて perform に記録）。
6. 完了したら perform の status を「レビュー待ち」相当にし、ユーザーに「Claudeチャットに結果を確認してもらってください」と伝える。
7. 次の依頼があれば「次の依頼があります。続けますか？」と確認する。

**未実行の依頼がない場合:** ウェルカムメッセージを表示し、ユーザーの指示を待つ。

> 移行期フォールバック（非推奨）: `_claude/tasks/pending/` の指示ファイル確認・`review/` 移動は当面残置するが、依頼把握・実行記録の正は DB ツールとする。ソースコード用リポ（aipatto-server 等）の `_tasks/` 運用は dev 固有として据え置く（下記）。

## 完了報告の書式（2026-04-28 ルール追加）

セッション内のあらゆる作業完了報告（指示実行完了、レビュー応答、調査報告等）の末尾に、以下の改行型 3 行を必ず付与する。

書式:

```
✅ <タスクファイル名> INSTR-N 完了
branch: <ブランチ名> @ <最新commit SHA短縮8桁>
次の指示をどうぞ。
```

例:

```
✅ task-FIX-159_signup-rename.md INSTR-3 完了
branch: claude/signup-rename-phase-4-wZezd @ 32dd0415
次の指示をどうぞ。
```

- タスクファイル名は**ファイル名のみ**（`task-FIX-159_signup-rename.md`）。フルパスは不要
- INSTR-N は受領した指示番号（Chat 側 rule-chat.md「指示ファイルプロンプトの書式」と対応）
- ブランチ情報は `<ブランチ名> @ <SHA8桁>` 形式
- main 直 commit の場合は `branch: main @ <SHA>`
- ブランチ作成のみで commit 未実施の段階では `branch: <ブランチ名> @ (未push)` と記載
- 改行型 3 行を厳守（1 行集約は不可）

**目的**: Chat 側で main マージ判断・diff 確認・PR 作成が即時可能になる。複数タスク並行や時間経過後の追跡性も確保する。

タスクと無関係な対話（雑談、書式確認等）には付与不要。指示・調査・実装・レビューに関する報告では原則として付与する。
