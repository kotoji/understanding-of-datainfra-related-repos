# Apache Iceberg 1.12.0-SNAPSHOT コードリーディング

> 調査日: 2026-07-29 (集計値は 2026-08-12 に再取得)
> 対象コミット: `089adca50` = `apache-iceberg-1.11.0-490-g089adca50` (main ブランチ)
> Java 17/21 / Gradle 8.14.5
>
> 本ドキュメント中の集計値 (モジュール数、ファイル数、コミット数等) はすべて上記コミット時点のもの。
> コミット数・コントリビュータ数は `HEAD` の履歴のみを対象とする (`git rev-list --count HEAD` / `git shortlog -sn HEAD`)。フェッチ済みのリリースブランチを含む `--all` はより大きい値になる。
>
> ソースの位置は `module/.../ClassName.java` 形式で示す。`...` は `src/main/java/org/apache/iceberg` の省略 (例: `core/.../TableMetadata.java` = `core/src/main/java/org/apache/iceberg/TableMetadata.java`)。

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
| [11-advanced-features.md](11-advanced-features.md) | Row Lineage、Puffin、View |
| [12-architecture-summary.md](12-architecture-summary.md) | 設計原則の総括、各要素の噛み合い方 |

## 規模感

- Gradle モジュール数: **25** (無条件 include) + Flink / Kafka Connect / 各エンジンのバージョン別サブプロジェクト (条件付き include)
- 主要言語: Java (17/21), Scala (Spark/Flink 連携)
- Java ファイル数 (テストコード含む): api 422, core 963, spark 1652, flink 1639
- 総コミット数: 9,034 (2017-12-18 〜 2026-08-11)
- 主要コントリビュータ: Ryan Blue (759), Anton Okolnychyi (496), Fokko Driesprong (460), Eduard Tudenhoefner (368)。なお最多は dependabot[bot] (1,038) で、依存バージョン更新の自動コミット

## アーキテクチャ概観

縦の `▼` は層をまたぐ関係を表すが、層ごとに意味が異なるため矢印の横に明記した。横の `→` は同じ層の中での「インターフェース → 具象実装」または「包含関係」を表す。

```
Engine Integration: Spark | Flink | Kafka Connect | MapReduce              [10]
                      │
                      ▼ エンジンが Iceberg の書き込み API を呼ぶ
Operations:         SnapshotProducer → AppendFiles / OverwriteFiles /      [08]
                      │                RowDelta / RewriteFiles
                      │                (OCC: compare-and-swap + retry)
                      ▼ 新しい TableMetadata の commit を SPI 経由で委譲
Catalog:            Catalog (interface) → REST / Hive / Glue / JDBC /      [06]
                      │                   Hadoop / Nessie
                      │                 TableOperations (SPI): atomic CAS
                      ▼ Catalog が指し示す「テーブルの現在の状態」
Metadata:           TableMetadata → Snapshot → ManifestList → Manifest     [05]
                      │             → DataFile / DeleteFile
                      │             Schema [03], PartitionSpec [04],
                      │             Expression [09]
                      ▼ メタデータ中のパスを実際に読み書きする手段
I/O:                FileIO (interface) → S3FileIO / GCSFileIO /            [07]
                      │                  ADLSFileIO / HadoopFileIO
                      ▼ FileIO が返す InputFile/OutputFile を各フォーマットが受け取る
File Format:        Parquet / ORC / Arrow (Builder pattern)                [07]
```

`[NN]` は詳細を扱うドキュメント番号。

## 設計目標

- Serializable isolation
- O(1) スキャンプランニング
- クライアント側ジョブプランニング (メタストアの中央ボトルネック回避)
- 完全なスキーマ・パーティション進化
- ストレージ分離 (パーティショニングはテーブル設定であり、ディレクトリ構造ではない)
- 複数ファイルフォーマット・エンジンのサポート
