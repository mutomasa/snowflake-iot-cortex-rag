# TODO — Snowflake IoT Predictive Maintenance RAG

設計書: [design.md](./design.md)

## 運用ルール

- **1機能 = 1ブランチ = 1PR**。各機能の見出しにあるブランチ名で作業する
- PRを出したら `PR:` 欄に番号を記入し、マージ後にチェックを付ける
- 作業中に見つけた問題点はGitHub Issueを起票し、該当機能の `Issues:` 欄と末尾の「Issues」セクションに番号を記入する
- 各機能は記載の順に進める（後の機能は前の機能に依存する）

---

## Phase 1：環境構築・データ生成

### F01 Project Scaffold — branch: `feature/01-project-scaffold`

- [ ] `uv init` でプロジェクト作成（Python 3.12）
- [ ] `pyproject.toml` に依存関係を追加（snowflake-connector-python, snowflake-snowpark-python, snowflake (core), dbt-core, dbt-snowflake, streamlit, plotly, pydantic-settings, pytest）
- [ ] `src/iot_cortex/` パッケージ構成を作成
- [ ] `app/`, `dbt/`, `sql/`, `data/documents/`, `scripts/`, `tests/` ディレクトリを作成
- [ ] `.gitignore`（`.env`, 秘密鍵, `dbt/target`, `dbt/logs` など）
- [ ] `.env.example`（design.md §10）
- [ ] pytest設定（`pyproject.toml`）と空テストの実行確認
- [ ] README雛形
- 完了条件: `uv sync` と `uv run pytest` が成功する
- PR: (未作成) / Issues: -

### F02 Snowflake Connection — branch: `feature/02-snowflake-connection`

- [ ] `config.py`：pydantic-settingsで環境変数を読み込む
- [ ] `snowflake_client.py`：キーペア認証（SNOWFLAKE_JWT）による接続、Snowpark Session生成
- [ ] 秘密鍵・パスフレーズ・トークンをログ出力しない
- [ ] `sql/01_setup.sql`：DB `IOT_DEMO`、スキーマ（RAW / STAGING / INTERMEDIATE / MARTS / AI）、Warehouse `IOT_DEMO_WH`（小サイズ・AUTO_SUSPEND）、最小権限Role
- [ ] `sql/02_create_raw_tables.sql`：SENSOR_READINGS / EQUIPMENT_MASTER / MAINTENANCE_DOCUMENTS / INCIDENT_HISTORY
- [ ] 接続のMockテスト（Snowflakeなしで実行可能）
- 完了条件: Pythonから接続確認ができる（接続できない場合はその旨を報告）。Mockテストが通る
- PR: (未作成) / Issues: -

### F03 Sensor Simulator — branch: `feature/03-sensor-simulator`

- [ ] 設備定義：MTR-001（搬送用モーター）、MTR-002（冷却ファン）、PMP-001（循環ポンプ）
- [ ] FR-01のスキーマでテレメトリ生成（reading_id, equipment_id, timestamp(UTC), temperature_c, vibration_mm_s, current_a, rpm, operating_state）
- [ ] 生成間隔を設定可能に（初期値5秒）
- [ ] シナリオ注入：NORMAL / BEARING_WEAR / OVERHEAT / OVERLOAD / SENSOR_FAULT
- [ ] 設備ごとの閾値設定（デモ用であり実機の安全基準ではないことを明記）
- [ ] `tests/test_sensor_simulator.py`：正常データ生成、各異常シナリオ、データ範囲検証
- 完了条件: 全シナリオのデータが生成でき、単体テストが通る
- PR: (未作成) / Issues: -

### F04 RAW Ingestion — branch: `feature/04-raw-ingestion`

- [ ] `ingestion.py`：SnowflakeへのバッチINSERT（パラメーターバインド）
- [ ] 取り込み件数の表示
- [ ] reading_idによる再実行時の重複防止（MERGE等）
- [ ] 失敗時のエラーログ記録
- [ ] EQUIPMENT_MASTERの投入
- [ ] `scripts/generate_seed_data.py`, `scripts/run_simulator.py`
- [ ] `tests/test_ingestion.py`（Mock）
- 完了条件: RAWテーブルに3設備分のデータが格納され、Pythonから件数を確認できる（Phase 1完了条件）
- PR: (未作成) / Issues: -

---

## Phase 2：dbtパイプライン

### F05 dbt Staging — branch: `feature/05-dbt-staging`

- [ ] `dbt/dbt_project.yml` 作成
- [ ] 共通の接続情報からdbt用profilesを作る（環境変数参照、秘密情報はコミットしない）
- [ ] `models/sources.yml`
- [ ] `stg_sensor_readings.sql`：型変換、欠損チェック、重複排除、不正値の除外、Timestamp正規化、無効レコードを監査可能にする
- [ ] `stg_equipment.sql`
- [ ] dbt tests：reading_idのnot_null/unique、equipment_idのnot_null/relationships
- 完了条件: `dbt build --select staging` が成功する
- PR: (未作成) / Issues: -

### F06 dbt Intermediate — branch: `feature/06-dbt-intermediate`

- [ ] `int_sensor_5min_agg.sql`：5分単位の集計
- [ ] `int_sensor_features.sql`：temperature_avg_5min, vibration_avg_5min, vibration_stddev_5min, temperature_rate, vibration_rate, vibration_zscore
- [ ] Z-scoreの基準を正常運転データから算出し、正常データが不足する場合は閾値ベースにフォールバック
- [ ] `int_anomaly_detection.sql`：VIBRATION / TEMPERATURE / CURRENT_ANOMALY の閾値判定（閾値は設定可能に）
- [ ] 閾値・特徴量のマクロ化（`macros/`）
- 完了条件: `dbt build --select intermediate` が成功する
- PR: (未作成) / Issues: -

### F07 dbt Marts — branch: `feature/07-dbt-marts`

- [ ] `fct_equipment_anomalies.sql`：anomaly_id, equipment_id, started_at, ended_at, anomaly_type, severity, observed_values, threshold_values, detected_at
- [ ] 複数回実行しても異常イベントが重複しない（incremental + 決定的なanomaly_id）
- [ ] `fct_equipment_health.sql`：設備状態・設備別KPI・最新状態サマリー
- [ ] `rag_incident_context.sql`
- [ ] dbt tests：anomaly_typeのaccepted_values
- [ ] 合成データによる異常検知のdata tests（BEARING_WEARでVIBRATION異常が出る等）
- [ ] 増分処理の検証
- [ ] dbt docsの生成
- [ ] `scripts/run_dbt.py`
- 完了条件: `dbt build` が成功し、異常シナリオから期待する異常イベントが生成される（Phase 2完了条件）
- PR: (未作成) / Issues: -

---

## Phase 3：Cortex Search

### F08 Maintenance Documents — branch: `feature/08-maintenance-documents`

- [ ] `data/documents/` に合成文書を作成：motor_maintenance_manual, pump_maintenance_manual, bearing_failure_incident, overheating_incident, overload_incident, troubleshooting_guide
- [ ] 障害事例を10〜20件用意
- [ ] 各文書にdocument_id / document_title / equipment_type / document_type / contentを付与
- [ ] `document_loader.py`：Markdownの読み込みとチャンク分割（chunk_id付与）
- [ ] `sql/03_create_document_tables.sql`：AI.DOCUMENT_CHUNKS
- [ ] `scripts/load_documents.py`：Snowflakeへの投入
- [ ] チャンク化の単体テスト
- 完了条件: DOCUMENT_CHUNKSにチャンクが格納される
- PR: (未作成) / Issues: -

### F09 Cortex Search — branch: `feature/09-cortex-search`

- [ ] `sql/04_create_cortex_search.sql`：MAINTENANCE_SEARCH（ATTRIBUTES EQUIPMENT_TYPE, DOCUMENT_TYPE、TARGET_LAG '1 hour'）
- [ ] インデックス構築完了の確認手順
- [ ] `cortex_search.py`：`CortexSearchClient.search(query, equipment_type, top_k)`
- [ ] 属性フィルター、Top-K、document_id / document_title / chunk_idの返却
- [ ] `tests/test_cortex_search.py`：検索結果の整形（Mock）
- [ ] 検索品質の確認
- 完了条件: 「モーターの振動異常」でベアリング摩耗の文書が上位に返る（Phase 3完了条件）
- PR: (未作成) / Issues: -

---

## Phase 4：Cortex AI・RAG

### F10 Cortex AI RAG — branch: `feature/10-cortex-ai-rag`

- [ ] `SnowflakeRepository`：get_latest_sensor_data / get_anomaly_events（SQLで取得）
- [ ] `cortex_ai.py`：`CortexAIClient.analyze(prompt)`、AI_COMPLETEをパラメーターバインドで呼ぶ、モデル名は設定から変更可能
- [ ] 利用可能なモデルの確認手順
- [ ] `rag_service.py`：`MaintenanceRAGService.diagnose(equipment_id, question)`
- [ ] プロンプト生成（design.md §7.5）
- [ ] JSON Schemaによる構造化出力と検証
- [ ] 引用の検証（検索されていないdocument_idを拒否）
- [ ] 根拠不足時は「追加点検が必要」と返すフォールバック
- [ ] 検索文書を信頼できない入力として扱う（プロンプトインジェクション対策）
- [ ] 可観測性ログ（request_id, equipment_id, anomaly_id, search_query, retrieved_document_ids, cortex_model, latency_ms, execution_status）。全文・秘密情報はログに出さない
- [ ] `tests/test_rag_service.py`：プロンプト生成、JSONパース、引用ID検証、フォールバック（Mock）
- 完了条件: 選択した異常イベントについて、センサー値と保守文書を引用した日本語の診断支援回答を取得できる（Phase 4完了条件）
- PR: (未作成) / Issues: -

---

## Phase 5：Streamlit

### F11 Streamlit Overview & Sensors — branch: `feature/11-streamlit-overview-sensors`

- [ ] `app/main.py`：ナビゲーション、データ更新ボタン、自動リロードのON/OFF
- [ ] `components/metrics.py`, `components/charts.py`（Plotly）
- [ ] Page 1 Overview：設備一覧、正常・異常件数、最新センサー値、設備状態、最終更新時刻
- [ ] Page 2 Sensor Monitoring：温度・振動・電流の時系列、閾値ライン、設備・時間範囲フィルター
- 完了条件: Overviewとセンサー画面でSnowflakeのデータが表示される
- PR: (未作成) / Issues: -

### F12 Streamlit Anomalies & AI Assistant — branch: `feature/12-streamlit-anomalies-assistant`

- [ ] Page 3 Anomaly Detection：異常イベント一覧、種別、発生日時、判定根拠、関連センサーデータ
- [ ] Page 4 AI Maintenance Assistant：チャットUI、異常イベントを選んで質問、推定原因・点検提案、検索文書、引用元、SQLと文書の根拠表示
- [ ] 異常シナリオ注入UI（DEMO_MODE=trueのときだけ表示）
- [ ] AI_COMPLETEはユーザー操作時のみ実行
- [ ] エラー通知
- 完了条件: Streamlitの画面操作だけで一連のデモが実行できる（Phase 5完了条件）
- PR: (未作成) / Issues: -

---

## 評価・仕上げ

### F13 RAG Evaluation — branch: `feature/13-rag-evaluation`

- [ ] 評価シナリオを10件以上（調整用と最終検証用に分け、合成データであることを明示）
- [ ] Recall@5, Citation Validity, Groundedness, JSON Validity, Diagnosis Accuracyの計測
- [ ] `scripts/verify_pipeline.py`：パイプライン全体の検証
- 完了条件: 評価結果をレポートとして出力できる
- PR: (未作成) / Issues: -

### F14 Docs & Cleanup — branch: `feature/14-docs-and-cleanup`

- [ ] README：セットアップ手順、デモ手順（design.md §13）
- [ ] 課金リソース（Warehouse、Cortex Search Service）の停止・削除手順
- [ ] 最終検証（`uv run pytest`、`dbt build`、`verify_pipeline.py`）と結果の記録
- 完了条件: READMEの手順だけでデモを再現できる
- PR: (未作成) / Issues: -

---

## Issues（問題点）

見つけた問題はGitHub Issue（`gh issue create`）で管理する。ここには番号と概要だけを記入する。

| Issue | 概要 | 関連機能 | 状態 |
|---|---|---|---|
| - | - | - | - |

---

## スコープ外（将来の拡張）

- Phase 6：Streaming IoT（MQTT、Snowpipe Streaming）
- Phase 7：予測保全ML（Isolation Forest、XGBoost、RUL）
- Phase 8：Agentic Maintenance（人間の承認を必須とする）
