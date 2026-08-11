# ビルドシステムとモジュール構成

## ビルドシステム

- **ビルドツール**: Gradle 8.14.5 (wrapper)
- **Java 要件**: **Java 17 or 21 のみ** (他バージョンは `GradleException`)
- **バージョンカタログ**: `gradle/libs.versions.toml` (232 行)
- **並列・キャッシュ**: `org.gradle.parallel=true`, `org.gradle.caching=true`

## モジュール一覧

`settings.gradle` の `include` は 2 種類ある:

- **無条件 include (25 モジュール)**: 以下の一覧のうち `flink` / `kafka-connect` を除く全部 + `bom` (Maven BOM 生成用の空モジュール、以下の一覧では省略)
- **条件付き include**: `flink` (`flinkVersions` が非空のとき)、`kafka-connect` (`kafkaVersions` に `3` を含むとき)、および各エンジンのバージョン別サブプロジェクト (`iceberg-spark-3.5_2.12` など)

以下のカテゴリ分けは説明のためのもので、Gradle 上の階層ではない。

### コアライブラリ (4)
| Module | Artifact | Java Files | 役割 |
|--------|----------|------------|------|
| `api` | `iceberg-api` | 422 | 公開 API (インターフェース、型) |
| `common` | `iceberg-common` | 5 | 共有ユーティリティ |
| `core` | `iceberg-core` | 963 | テーブル仕様実装、REST Catalog クライアント |
| `data` | `iceberg-data` | 67 | エンジン非依存データ層 |

### ファイルフォーマット (3)
| Module | Artifact | Java Files | 役割 |
|--------|----------|------------|------|
| `parquet` | `iceberg-parquet` | 81 | Parquet サポート |
| `orc` | `iceberg-orc` | 40 | ORC サポート |
| `arrow` | `iceberg-arrow` | 38 | Arrow サポート |

### Catalog 実装 (5 + core 内蔵)
| Module | 役割 |
|--------|------|
| `hive-metastore` | Hive Metastore Catalog |
| `nessie` | Nessie (Git-like バージョニング) Catalog |
| `snowflake` | Snowflake Catalog (読み取り専用) |
| `bigquery` | BigQuery Metastore Catalog |
| `dell` | Dell ECS 向け Catalog (`EcsCatalog`) + FileIO (`EcsFileIO`) |
| (core 内蔵) | Hadoop, JDBC, REST Catalog |

### クラウドプロバイダ (7)

`*-bundle` は依存関係を shade した fat JAR で、実装は本体モジュール側にある。

| Module | 役割 |
|--------|------|
| `aws` / `aws-bundle` | S3, Glue, DynamoDB, Lake Formation |
| `azure` / `azure-bundle` | Azure Data Lake Storage |
| `gcp` / `gcp-bundle` | Google Cloud Storage, KMS |
| `aliyun` | Alibaba Cloud OSS |

### エンジン連携 (4 + バージョン別サブプロジェクト)

`spark` / `flink` / `kafka-connect` はサブプロジェクトを束ねるコンテナで、実装はバージョン別サブプロジェクト (`spark/v3.5/spark` 等) にある。

| Module | バージョン | include | 役割 |
|--------|------------|---------|------|
| `spark` | 3.5, 4.0, 4.1 | 無条件 | Spark DataSource V2 |
| `flink` | 1.20, 2.0, 2.1 | 条件付き | Flink Table API |
| `mr` | - | 無条件 | MapReduce / Hive InputFormat |
| `kafka-connect` | 3 | 条件付き | Kafka Connect Sink |

### その他 (3)
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
| Caffeine | 3.2.4 | キャッシング |
| AWS SDK | 2.50.2 (BOM) | AWS 連携 |
| Hadoop | 3.4.3 | ファイルシステム抽象 (compileOnly) |
