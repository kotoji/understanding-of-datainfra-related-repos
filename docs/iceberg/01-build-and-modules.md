# ビルドシステムとモジュール構成

## ビルドシステム

- **ビルドツール**: Gradle 8.14.5 (wrapper)
- **Java 要件**: **Java 17 or 21 のみ** (他バージョンは `GradleException`)
- **バージョンカタログ**: `gradle/libs.versions.toml` (234 行)
- **並列・キャッシュ**: `org.gradle.parallel=true`, `org.gradle.caching=true`

## モジュール一覧

### コアライブラリ (4)
| Module | Artifact | Java Files | 役割 |
|--------|----------|------------|------|
| `api` | `iceberg-api` | 418 | 公開 API (インターフェース、型) |
| `common` | `iceberg-common` | 5 | 共有ユーティリティ |
| `core` | `iceberg-core` | 959 | テーブル仕様実装、REST Catalog クライアント |
| `data` | `iceberg-data` | 67 | エンジン非依存データ層 |

### ファイルフォーマット (3)
| Module | Artifact | Java Files | 役割 |
|--------|----------|------------|------|
| `parquet` | `iceberg-parquet` | 80 | Parquet サポート |
| `orc` | `iceberg-orc` | 40 | ORC サポート |
| `arrow` | `iceberg-arrow` | 38 | Arrow サポート |

### Catalog 実装 (6 + core 内蔵)
| Module | 役割 |
|--------|------|
| `hive-metastore` | Hive Metastore Catalog |
| `nessie` | Nessie (Git-like バージョニング) Catalog |
| `snowflake` | Snowflake Catalog |
| `bigquery` | BigQuery Metastore Catalog |
| `dell` | Dell ECS ストレージ |
| (core 内蔵) | Hadoop, JDBC, REST Catalog |

### クラウドプロバイダ (7)
| Module | 役割 |
|--------|------|
| `aws` / `aws-bundle` | S3, Glue, DynamoDB, Lake Formation |
| `azure` / `azure-bundle` | Azure Data Lake Storage |
| `gcp` / `gcp-bundle` | Google Cloud Storage, KMS |
| `aliyun` | Alibaba Cloud OSS |

### エンジン連携 (4 + バージョン別サブプロジェクト)
| Module | バージョン | 役割 |
|--------|------------|------|
| `spark` | 3.5, 4.0, 4.1 | Spark DataSource V2 |
| `flink` | 1.20, 2.0, 2.1 | Flink Table API |
| `mr` | - | MapReduce / Hive InputFormat |
| `kafka-connect` | 3 | Kafka Connect Sink |

### その他
| Module | 役割 |
|--------|------|
| `bundled-guava` | Shaded Guava (`org.apache.iceberg.relocated.com.google.common`) |
| `delta-lake` | Delta Lake → Iceberg 変換 |
| `open-api` | REST Catalog OpenAPI 仕様 + 互換性テストキット |

## モジュール依存関係グラフ

```
bundled-guava (shaded Guava)
  |
  +-- common
  |
  +-- api
  |
  +-- core (api [api], common [impl], bundled-guava [impl])
        |
        +-- parquet, orc (api [api], core [impl])
        |
        +-- arrow (api [api], core [impl], parquet [impl])
        |
        +-- data (api [api], core [impl]; compileOnly: parquet, orc)
        |
        +-- hive-metastore, aws, gcp, azure, aliyun, bigquery, nessie, dell, snowflake
        |
        +-- spark, flink, mr, kafka-connect (core + エンジン固有ライブラリ)
```

設計原則: ほぼ全モジュールが `bundled-guava` (shadow) と `api` (公開 API) に依存。`core` と `common` は implementation 依存。

## 主要ライブラリバージョン

| 依存 | バージョン | 用途 |
|------|-----------|------|
| Avro | 1.12.1 | メタデータシリアライゼーション |
| Parquet | 1.17.1 | カラムナファイルフォーマット |
| ORC | 1.9.9 | カラムナファイルフォーマット |
| Arrow | 15.0.2 | インメモリカラムナフォーマット |
| Jackson | 2.22.1 (BOM) | JSON 処理 |
| Guava | 33.6.0-jre | 内部 (shaded) |
| Caffeine | 2.9.3 | キャッシング |
| AWS SDK | 2.47.5 (BOM) | AWS 連携 |
| Hadoop | 3.4.3 | ファイルシステム抽象 (compileOnly) |
