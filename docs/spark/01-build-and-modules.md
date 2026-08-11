# ビルドシステムとモジュール構成

## Maven + SBT のデュアルビルド

| 項目 | Maven | SBT |
|------|-------|-----|
| Wrapper | `build/mvn` (Maven 3.9.16 自動 DL) | `build/sbt` (SBT 1.12.13) |
| 用途 | CI / リリースビルド | 開発時のインクリメンタルコンパイル |
| 設定 | `pom.xml` (3,755行) | `project/SparkBuild.scala` (POM 読み込み) |

SBT は `sbt-pom-reader` (`SparkBuild` オブジェクトが `PomBuild` を extends) 経由で Maven POM をそのまま読み込む。依存関係のバージョンを SBT 側で別途管理しないことで、CI/リリースで使う Maven ビルドと開発時の SBT ビルドの間で依存バージョンが分岐するのを防いでいる。

## 主要 Maven Profile

| Profile | 内容 |
|---------|------|
| `yarn` | YARN Resource Manager 統合 |
| `kubernetes` | Kubernetes 統合 |
| `hive-thriftserver` | JDBC/ODBC サーバ |
| `hadoop-cloud` | S3/Azure/GCS cloud storage |
| `hadoop-provided` / `hive-provided` 等 | 依存を provided にする |

## 主要スクリプト

| スクリプト | 用途 |
|-----------|------|
| `bin/spark-submit` | アプリケーション submit |
| `bin/spark-shell` | Scala REPL |
| `bin/pyspark` | Python REPL |
| `bin/spark-sql` | SQL CLI |
| `bin/spark-connect-shell` | Connect 有効 REPL |
| `bin/spark-pipelines` | Declarative Pipelines CLI |
| `sbin/start-master.sh` / `start-worker.sh` | Standalone クラスタ起動 |
| `dev/make-distribution.sh` | バイナリ配布物作成 |
| `dev/run-tests.py` | CI テストランナー |

## モジュール依存グラフ (簡略版)

```
spark-tags (基底)
  └─ spark-launcher
  └─ spark-common-utils / spark-common-utils-java
  └─ spark-unsafe (off-heap メモリ)
  └─ spark-sketch (Count-Min Sketch 等)
  └─ spark-kvstore (組込み KV ストア。RocksDB / LevelDB / InMemory)
  └─ spark-network-common → spark-network-shuffle
       └─ spark-core ← 全モジュールの基盤
            ├─ spark-sql-api (公開 API 型定義)
            │    └─ spark-catalyst (クエリコンパイラ)
            │         └─ spark-sql (実行エンジン)
            │              └─ spark-hive (Hive 連携)
            ├─ spark-streaming (DStream, deprecated)
            ├─ spark-mllib-local → spark-graphx
            │                    → spark-mllib
            ├─ spark-repl (Scala REPL)
            └─ spark-connect-server (gRPC サーバ)
```

## 全モジュール一覧 (root `pom.xml` の `<module>` 全 50 件)

**`common/` (11):** sketch, kvstore, network-common, network-shuffle, network-yarn, unsafe, utils, utils-java, variant, tags, config

**Core 系 (6):** core, launcher, tools, repl, examples, assembly

**`sql/` (6):** api, catalyst, core, hive, hive-thriftserver, pipelines

**`sql/connect/` (5):** shims, common, server, client/jvm, client/jdbc

**`udf/worker/` (3):** proto, core, grpc

**`connector/` (11):** avro, protobuf, kafka-0-10, kafka-0-10-sql, kafka-0-10-assembly, kafka-0-10-token-provider, spark-ganglia-lgpl, kinesis-asl, kinesis-asl-assembly, profiler, docker-integration-tests

**`resource-managers/` (3):** yarn, kubernetes/core, kubernetes/integration-tests

**その他 (5):** streaming, mllib, mllib-local, graphx, hadoop-cloud
