# Apache Spark 5.0.0-SNAPSHOT コールドリーディング

> 調査日: 2026-07-29
> 対象: master ブランチ (commit `04d4c266592`)
> Scala 2.13.18 / Java 17 / Python 3.10+

## ドキュメント一覧

| ファイル | 内容 |
|---------|------|
| [00-overview.md](00-overview.md) | 本ファイル。概要とインデックス |
| [01-build-and-modules.md](01-build-and-modules.md) | ビルドシステム、モジュール構成、依存関係 |
| [02-core.md](02-core.md) | Spark Core: SparkContext, RDD, Scheduler, Shuffle, Storage |
| [03-sql-catalyst.md](03-sql-catalyst.md) | Spark SQL / Catalyst: パイプライン、最適化、AQE、CodeGen |
| [04-structured-streaming.md](04-structured-streaming.md) | Structured Streaming: 実行エンジン、State管理 |
| [05-spark-connect.md](05-spark-connect.md) | Spark Connect: gRPC アーキテクチャ、Protocol |
| [06-pyspark.md](06-pyspark.md) | PySpark: Dispatch パターン、UDF、Arrow 統合 |
| [07-connectors-and-resource-managers.md](07-connectors-and-resource-managers.md) | Connectors、Kubernetes/YARN 統合 |
| [08-ecosystem.md](08-ecosystem.md) | MLlib、GraphX、Common モジュール |
| [09-spark5-new-features.md](09-spark5-new-features.md) | Spark 5.0 の新機能・変更点 |
| [10-design-patterns.md](10-design-patterns.md) | アーキテクチャ上の設計パターン |

## 規模感

- Maven モジュール数: **51** (profile によるオプション含む)
- 主要言語: Scala (2.13), Java (17), Python (3.10+)
- 主要依存ライブラリバージョン: Hadoop 3.5.0, Kafka 3.9.2, Arrow 19.0.0, Protobuf 4.33.5, gRPC 1.76.0
- 2025年1月〜7月の commit 数: **1,866** (SQL 724, Python 370, Connect 287 が上位)

## アーキテクチャ概観

```
User API: PySpark | Scala/Java | Connect Client | SQL CLI | spark-pipelines
            │
            ▼
SQL Engine: Parser → Analyzer → Optimizer → Planner → Preparations
            │        (60+ rules)  (47 files)  (16 strategies)  (AQE, WSCG)
            ▼
Core:       DAGScheduler → TaskScheduler → SchedulerBackend
            │                                (Local/Standalone/K8s/YARN)
            ▼
Runtime:    Executor ← BlockManager ← UnifiedMemoryManager
                       ShuffleManager (Sort + Pipelined)
```
