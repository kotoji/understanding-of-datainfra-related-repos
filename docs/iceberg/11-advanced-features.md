# 高度な機能とアーキテクチャ総括

## Row Lineage (v3)

2つのメタデータカラム:
- `_row_id` (field id 2147483540): 行ごとの一意 long 識別子
- `_last_updated_sequence_number` (field id 2147483539): 最後に行を更新したコミットの sequence number

### 割り当てメカニズム (多段継承)

```
Table metadata → next-row-id
  └── Snapshot → first-row-id = table's next-row-id
        └── Manifest List → first_row_id (累積 added_rows_count + existing_rows_count)
              └── Manifest → DataFile → first_row_id (累積 record_count)
                    └── Row → _row_id = data_file.first_row_id + row_pos
```

この継承設計により、マニフェストとデータファイルをコミットシーケンス番号が判明する前に書き込める。楽観的並行制御でリトライ時に書き直し不要。

Equality delete で更新された行は追跡されない (完全な削除 + 新行挿入として扱う)。

## Puffin ファイル (統計)

仕様: `format/puffin-spec.md`

統計 blob を格納するバイナリフォーマット:
```
Magic (0x50 0x46 0x41 0x31) | Blob1 | ... | BlobN | Footer
```

サポート blob 種別:
- `apache-datasketches-theta-v1`: NDV (Number of Distinct Values) 推定用 Theta スケッチ
- `deletion-vector-v1`: 削除行位置の Roaring bitmap

圧縮: LZ4, Zstd, 非圧縮

テーブルメタデータの `statistics` フィールドで Puffin ファイルを参照。`partition-statistics` でパーティションごとの集計 (レコード数、ファイル数、削除数) を追跡。

## View サポート

仕様: `format/view-spec.md`

### テーブルとの違い

| 観点 | Table | View |
|------|-------|------|
| データファイル | あり | なし (SQL 定義のみ) |
| バージョニング | Snapshot | ViewVersion (SQL 定義の版) |
| マニフェスト | あり | なし |
| 操作 | データ操作多数 | `replaceVersion()`, `updateProperties()` のみ |
| スキーマ | 入力スキーマ | 出力スキーマ (クエリ結果の型) |
| フォーマットバージョン | v1-v4 | v1 |

### 主要クラス

- `View` (interface): `currentVersion()`, `versions()`, `history()`, `schema()`
- `ViewVersion`: SQL 表現を持つバージョン (複数 dialect 対応可能)
- `ViewCatalog`: `listViews()`, `loadView()`, `buildView()`, `dropView()`
- `ViewMetadata`: UUID, schemas, versions, history, properties (Immutables ベース)

Spark 連携: `SparkView` でラップ、`SparkCatalog` が Spark の `ViewCatalog` を実装

---

## アーキテクチャ総括

### レイヤード設計

```
┌─────────────────────────────────────────────────────┐
│  Engine Integration Layer                            │
│  (Spark / Flink / Kafka Connect / MapReduce)         │
│  エンジン固有の DataSource/Table/Sink 実装            │
├─────────────────────────────────────────────────────┤
│  Operations Layer                                    │
│  SnapshotProducer, MergingSnapshotProducer           │
│  AppendFiles, OverwriteFiles, RowDelta, etc.         │
│  コミットプロトコル (OCC + リトライ)                   │
├─────────────────────────────────────────────────────┤
│  Catalog Layer                                       │
│  Catalog (interface) → 具象実装 (REST, Hive, JDBC 等) │
│  TableOperations (SPI): アトミック CAS                │
│  CachingCatalog (decorator)                          │
├─────────────────────────────────────────────────────┤
│  Metadata Layer                                      │
│  TableMetadata (不変), Snapshot, ManifestFile          │
│  Schema, PartitionSpec, SortOrder                    │
│  Expression, Predicate, Evaluator                    │
├─────────────────────────────────────────────────────┤
│  I/O Layer                                           │
│  FileIO (interface) → S3FileIO, GCSFileIO, ADLS 等   │
│  InputFile / OutputFile                              │
├─────────────────────────────────────────────────────┤
│  File Format Layer                                   │
│  Parquet, ORC, Arrow (プラグイン方式)                  │
│  Builder パターンで InputFile/OutputFile を受取        │
└─────────────────────────────────────────────────────┘
```

### コア設計原則

1. **関心の分離**: 「メタデータの追跡場所」(Catalog) / 「メタデータのシリアライズ方法」(Parser) / 「ファイルの格納場所」(FileIO) / 「ファイルフォーマット」(Parquet/ORC/Arrow) が完全に独立
2. **不変性**: TableMetadata, Snapshot, Schema 等は不変オブジェクト。変更は常に新インスタンスを生成
3. **楽観的並行制御**: メタデータの compare-and-swap でアトミックコミット。リトライで競合解決
4. **フィールド ID ベース**: スキーマ進化はカラム名ではなく一意 ID で追跡。リネーム・並べ替えが安全
5. **マニフェスト再利用**: 変更のないマニフェストはスナップショット間で共有。O(1) スキャンプランニングの基盤
6. **隠れたパーティショニング**: ユーザーはパーティション式を意識せず `WHERE` を書ける。エンジンが自動射影
7. **プラグイン方式**: Catalog, FileIO, ファイルフォーマットすべて差し替え可能
