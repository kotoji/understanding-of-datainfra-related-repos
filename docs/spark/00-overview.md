# Apache Spark 5.0.0-SNAPSHOT コードリーディング

> 調査日: 2026-07-29
> 対象: master ブランチ (commit `04d4c266592`)
> Scala 2.13.18 / Java 17 (ビルド baseline。実行は 17/21/25 対応) / Python 3.11+
>
> 本ドキュメント中の集計値 (モジュール数、ファイルサイズ、ルール数等) はすべて上記コミット時点のもの。

## ドキュメント一覧

| ファイル                                                                         | 内容                                                       |
| -------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| [00-overview.md](00-overview.md)                                                 | 本ファイル。概要とインデックス                             |
| [01-build-and-modules.md](01-build-and-modules.md)                               | ビルドシステム、モジュール構成、依存関係                   |
| [02-core.md](02-core.md)                                                         | Spark Core: SparkContext, RDD, Scheduler, Shuffle, Storage |
| [03-sql-catalyst.md](03-sql-catalyst.md)                                         | Spark SQL / Catalyst: パイプライン、最適化、AQE、CodeGen   |
| [04-structured-streaming.md](04-structured-streaming.md)                         | Structured Streaming: 実行エンジン、State管理              |
| [05-spark-connect.md](05-spark-connect.md)                                       | Spark Connect: gRPC アーキテクチャ、Protocol               |
| [06-pyspark.md](06-pyspark.md)                                                   | PySpark: Dispatch パターン、UDF、Arrow 統合                |
| [07-connectors-and-resource-managers.md](07-connectors-and-resource-managers.md) | Connectors、Kubernetes/YARN 統合                           |
| [08-ecosystem.md](08-ecosystem.md)                                               | MLlib、GraphX、Common モジュール                           |
| [09-spark5-new-features.md](09-spark5-new-features.md)                           | Spark 5.0 の新機能・変更点                                 |
| [10-design-patterns.md](10-design-patterns.md)                                   | アーキテクチャ上の設計パターン                             |
| [11-job-execution-flow.md](11-job-execution-flow.md)                             | ジョブ実行フロー深掘り: SparkContext 初期化 → 結果返却     |

## 規模感

- Maven モジュール数: **50** (profile によるオプション含む)
- 主要言語: Scala (2.13), Java (17), Python (3.11+)
- 主要依存ライブラリバージョン: Hadoop 3.5.0, Kafka 3.9.2, Arrow 19.0.0, Protobuf 4.33.5, gRPC 1.76.0
- 2025年1月〜7月の commit 数: **1,866** (SQL 724, Python 370, Connect 287 が上位)

### 主要ファイルサイズ

| ファイル                                            | サイズ | 内容                            |
| --------------------------------------------------- | ------ | ------------------------------- |
| `catalyst/analysis/Analyzer.scala`                  | 219KB  | 解析器 (60+ ルール)             |
| `catalyst/optimizer/Optimizer.scala`                | 136KB  | 最適化器 (47 ルールファイル)    |
| `project/SparkBuild.scala`                          | 96KB   | SBT ビルド定義                  |
| `sql/core/.../adaptive/AdaptiveSparkPlanExec.scala` | 44KB   | AQE 実行エンジン                |
| `core/SparkContext.scala`                           | 145KB  | SparkContext (3,617行)          |
| `core/scheduler/DAGScheduler.scala`                 | 212KB  | DAG スケジューラ (4,464行)      |
| `pom.xml` (root)                                    | 138KB  | Maven 設定 (3,755行)            |
| `python/pyspark/pandas/frame.py`                    | 521KB  | Pandas API DataFrame (14,525行) |

## アーキテクチャ概観

矢印の意味: `→` はリクエスト/データの受け渡し (フロー)、`←` はコンポーネントが下位のコンポーネントを利用する関係 (依存) を表す。

```
User API: PySpark | Scala/Java | Connect Client | SQL CLI | spark-pipelines
            │
            ▼ (クエリ/ジョブ投入)
SQL Engine: Parser → Analyzer → Optimizer → Planner → Preparations
            │        (60+ rules)  (47 files)  (17 strategies)  (AQE, WSCG)
            ▼ (物理実行プランを RDD 操作に変換)
Core:       DAGScheduler → TaskScheduler → SchedulerBackend
            │                                (Local/Standalone/K8s/YARN)
            ▼ (Task を Executor で実行)
Runtime:    Executor は BlockManager (ストレージ) と UnifiedMemoryManager (メモリ確保)
            を利用しつつ、ShuffleManager (Sort + Pipelined) で shuffle データを読み書きする
```
