# Apache Iceberg 1.12.0-SNAPSHOT コールドリーディング

> 調査日: 2026-07-29
> 対象: main ブランチ (tag `apache-iceberg-1.11.0` + 399 commits)
> Java 17/21 / Gradle 8.14.5

## ドキュメント一覧

| ファイル | 内容 |
|---------|------|
| [00-overview.md](00-overview.md) | 本ファイル。概要とインデックス |
| [01-build-and-modules.md](01-build-and-modules.md) | ビルドシステム、モジュール構成、依存関係グラフ |
| [02-format-spec.md](02-format-spec.md) | テーブルフォーマット仕様 v1-v4 |
| [03-type-system-and-schema.md](03-type-system-and-schema.md) | 型システム、Schema、スキーマ進化 |
| [04-partitioning-and-sorting.md](04-partitioning-and-sorting.md) | パーティショニング、Transform、パーティション進化、ソートオーダー |
| [05-metadata-and-snapshots.md](05-metadata-and-snapshots.md) | メタデータ構造、Snapshot、ManifestFile、ブランチ/タグ |
| [06-catalog-and-rest.md](06-catalog-and-rest.md) | Catalog 抽象、各実装のコミット方式、REST Catalog |
| [07-fileio-and-formats.md](07-fileio-and-formats.md) | FileIO 抽象、Parquet/ORC/Arrow 統合 |
| [08-commit-and-write-path.md](08-commit-and-write-path.md) | コミットプロトコル (OCC)、Write パス、削除メカニズム |
| [09-scan-planning.md](09-scan-planning.md) | 式・フィルタシステム、述語射影、スキャンプランニングパイプライン |
| [10-engine-integrations.md](10-engine-integrations.md) | Spark / Flink / Kafka Connect / Delta Lake マイグレーション |
| [11-advanced-features.md](11-advanced-features.md) | Row Lineage、Puffin、View、アーキテクチャ総括 |

## 規模感

- Gradle モジュール数: **24** (静的) + Spark/Flink/Kafka Connect バージョン別サブプロジェクト
- 主要言語: Java (17/21), Scala (Spark/Flink 連携)
- Java ファイル数: api 418, core 959, spark 1652, flink 1636
- 総コミット数: ~9,286 (2017-12-18 〜 2026-07-28)
- 主要コントリビュータ: Ryan Blue (782), Eduard Tudenhoefner (607), Anton Okolnychyi (507), Fokko Driesprong (492)

## アーキテクチャ概観

```
Engine Integration: Spark | Flink | Kafka Connect | MapReduce
                      │
                      ▼
Operations:         SnapshotProducer → AppendFiles / OverwriteFiles / RowDelta / RewriteFiles
                      │                 (OCC: compare-and-swap + retry)
                      ▼
Catalog:            Catalog (interface) → REST / Hive / Glue / JDBC / Hadoop / Nessie
                      │                   TableOperations (SPI): atomic CAS
                      ▼
Metadata:           TableMetadata → Snapshot → ManifestList → Manifest → DataFile/DeleteFile
                      │               Schema, PartitionSpec, SortOrder
                      ▼
I/O:                FileIO (interface) → S3FileIO / GCSFileIO / ADLSFileIO / HadoopFileIO
                      │
                      ▼
File Format:        Parquet / ORC / Arrow (Builder pattern, InputFile/OutputFile)
```

## 設計目標

- Serializable isolation
- O(1) スキャンプランニング
- クライアント側ジョブプランニング (メタストアの中央ボトルネック回避)
- 完全なスキーマ・パーティション進化
- ストレージ分離 (パーティショニングはテーブル設定であり、ディレクトリ構造ではない)
- 複数ファイルフォーマット・エンジンのサポート
