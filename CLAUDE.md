# CLAUDE.md

Snowflake、dbt、Cortex Search、Cortex AIでIoT設備の異常検知とRAGによる原因分析を実演するデモ。

- 設計書: [docs/design.md](docs/design.md)
- タスク管理: [docs/todo.md](docs/todo.md)

## 会話

- 常に日本語で会話する

## 開発フロー

### ブランチ

- `main` に直接コミットしない
- todo.md の機能（F01〜F14）ごとにブランチを切る
- ブランチ名は todo.md に書いた `feature/NN-<name>` を使う（例: `feature/03-sensor-simulator`）
- 1つのブランチでは1つの機能だけを実装する

### Pull Request

- 1機能 = 1PR。機能が完成したら `gh pr create` でPRを出す
- PR本文には次を書く
  - 対象のtodo項目（F番号）
  - 変更内容の概要
  - 実行した検証コマンドとその結果（`uv run pytest`、`dbt build` など）
  - 関連するIssue番号
- PRを出したら todo.md の `PR:` 欄に番号を書き、マージ後にチェックを付ける

### Issues

- 作業中に見つけた問題点（バグ、仕様の不明点、技術的負債、Snowflake側の制約など）は `gh issue create` でIssueにする
- 今のPRの範囲外の問題は、そのPRでは直さずIssueに回す
- 起票したIssueは todo.md の該当機能の `Issues:` 欄と末尾の「Issues」表に記入する
- Issueを解決するPRには本文に `Closes #<番号>` を書く

## 実装ルール

1. Pythonの依存管理はuvを使用する
2. Python 3.12を使用する
3. UIはStreamlitで実装する
4. Snowflakeを唯一のデータ基盤にする
5. dbtでSQL変換を管理する
6. Cortex Searchで保守文書を検索する
7. Cortex AI_COMPLETEを利用する
8. 外部LLM APIやVectorDBを追加しない
9. 型ヒントを付ける
10. pytestによる単体テストを追加する
11. Snowflake接続がなくてもMockテストを動かせるようにする
12. 秘密情報をコミットしない
13. READMEにセットアップとデモ手順を書く
14. 実装完了後に検証コマンドを実行し、結果を報告する
15. Snowflakeに接続できない場合は、接続できたと偽ってはならない

## 設計原則

- 関数（メソッド）は小さくし、SOLID原則に従う
- 各コンポーネント（SnowflakeRepository、CortexSearchClient、CortexAIClient、MaintenanceRAGService）は単独で単体テストできるようにする
- SQLにはパラメーターバインドを使い、文字列連結でクエリを組み立てない
- 検索文書は信頼できない入力として扱い、文書内の指示でシステムの動作を変えない
- 設備の停止や制御コマンドは実装しない

## 進め方

- 最初からすべてを実装しない。todo.md の順に1機能ずつ進め、各機能で動作を確認する
