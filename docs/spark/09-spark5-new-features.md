# Spark 5.0 の新機能・変更点

## Pipelined Shuffle (StreamingShuffleManager)

`PipelinedShuffleDependency` + `StreamingShuffleManager` で、producer stage 実行中に consumer stage が shuffle データの読み出しを開始できる。従来の `SortShuffleManager` と並存し、`SparkEnv.shuffleManagerFor` でルーティング。

なお設定キー `spark.shuffle.manager.incremental` のソース上の `.version()` 注釈は `4.3.0` になっているが、実装が入っているのは master (5.0.0-SNAPSHOT) のみ。

## Declarative Pipelines (SDP)

`sql/pipelines/` + `python/pyspark/pipelines/`:
- Python API (`pyspark/pipelines/api.py`): デコレータ `@table`, `@materialized_view`, `@temporary_view`, `@append_flow` と、関数 `create_streaming_table()`, `create_sink()`, `create_auto_cdc_flow()`
- Scala 側のグラフモデルでは `DatasetType` enum (`MATERIALIZED_VIEW` / `STREAMING_TABLE`) と `TemporaryView` / `Sink` (`sql/pipelines/.../graph/elements.scala`) として表現される
- Change Data Capture (Auto CDC / SCD Type 1 & 2 — `stored_as_scd_type`) サポート
- `spark-pipelines` CLI
- dbt / Delta Live Tables に対抗する高レベル宣言的パイプラインモデル

## 言語非依存 UDF Worker フレームワーク

`udf/worker/` (SPIP SPARK-55278):
- gRPC bidirectional streaming による UDF 実行プロトコル
- `WorkerDispatcher` → `WorkerSession` → `WorkerConnection`
- Wire protocol: `Init → PayloadChunk* → Finish/Cancel` (Engine→Worker), `InitResponse → DataResponse* → FinishResponse` (Worker→Engine)
- 任意の言語で UDF を実装可能にする

## Arrow UDF の全面展開

Arrow UDF (4.1.0~) が Scalar, Aggregation, Window に拡張。Arrow が全 UDF タイプのデフォルトデータ転送メカニズムになる方向。

## VARIANT 型の成熟

CSV, XML, Parquet で VARIANT サポート。`try_variant_set` 式、コロン演算子 (`:`) でフィールドアクセス。

## TIME 型

`make_time`, `to_time`, `current_time` 関数。Connect proto サポート。

## Geospatial 型 (初期段階)

`GeographyType` / `GeometryType` + `SpatialReferenceSystemCache`。Proto 番号予約済み。

## Python 最小バージョン引き上げ

Python 3.11+ (`python_requires=">=3.11"`), pandas >= 2.2.0, PyArrow >= 18.0.0

---

## 付録: リポジトリの開発プロセスに関する変化

以下は Spark 自体の製品機能ではなく、リポジトリの開発プロセス上の変化。

### AI 支援開発

`AGENTS.md` (265行、`CLAUDE.md` はそこへの symlink) で AI コーディングエージェント向けの包括的な指示を提供。テストフレームワーク階層、ビルドコマンド、PR ワークフロー、CI デバッグ手順を含む。
