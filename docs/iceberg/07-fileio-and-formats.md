# FileIO 抽象とファイルフォーマット統合

## FileIO インターフェース (`api/.../io/FileIO.java`)

`Serializable` (分散エンジンで転送可能) + `Closeable`:
- `newInputFile(path)` → `InputFile` (読み取り)
- `newOutputFile(path)` → `OutputFile` (書き込み)
- `deleteFile(path)` (削除)
- `initialize(properties)` (設定)

拡張インターフェース:
- `DelegateFileIO`: `FileIO + SupportsPrefixOperations + SupportsBulkOperations`
- `SupportsBulkOperations`: 一括削除
- `SupportsPrefixOperations`: プレフィックスによるファイルリスト
- `SupportsRecoveryOperations`: ファイル復旧
- `SupportsStorageCredentials`: 払い出しストレージ認証情報

### InputFile / OutputFile

- `InputFile`: `getLength()`, `newStream()` (→ `SeekableInputStream`), `location()`, `exists()`
- `OutputFile`: `create()` (既存時失敗), `createOrOverwrite()`, `location()`, `toInputFile()`

## FileIO 実装一覧

| 実装 | モジュール | スキーム | 特徴 |
|------|-----------|---------|------|
| `HadoopFileIO` | core | HDFS, local, S3A等 | Hadoop FileSystem API 使用 |
| `ResolvingFileIO` | core | 自動判別 | URI スキームで適切な FileIO に委譲。REST Catalog のデフォルト |
| `S3FileIO` | aws | `s3://`, `s3a://`, `s3n://`, `https://` | AWS SDK v2, バルク削除, credential vending |
| `GCSFileIO` | gcp | `gs://` | Google Cloud Storage |
| `ADLSFileIO` | azure | `abfss://` | Azure Data Lake Storage Gen2 |

## ファイルフォーマット統合

全フォーマットモジュールが Builder パターンのエントリポイントを提供し、`InputFile`/`OutputFile` (FileIO 抽象) を受け取る。特定のストレージバックエンドに依存しない。

### Parquet (`parquet/`)

エントリポイント: `Parquet.java`
- `Parquet.write(OutputFile)` → `WriteBuilder`
- `Parquet.read(InputFile)` → `ReadBuilder`

| クラス | 役割 |
|--------|------|
| `ParquetWriter` / `ParquetReader` | 低レベル読み書き |
| `VectorizedParquetReader` | ベクトル化/カラムナバッチ読み取り |
| `ParquetSchemaUtil` | Iceberg ↔ Parquet スキーマ変換 |
| `ParquetMetrics` | カラムレベルメトリクス収集 |
| `ParquetMetricsRowGroupFilter` | 統計ベースの行グループフィルタ |
| `ParquetDictionaryRowGroupFilter` | 辞書ベースフィルタ |
| `ParquetBloomRowGroupFilter` | Bloom フィルタ |
| `ParquetVariantReaders/Writers` | Variant 型 (shredded storage) |

### ORC (`orc/`)

エントリポイント: `ORC.java` (Parquet と同パターン)

| クラス | 役割 |
|--------|------|
| `OrcFileAppender` | ORC ファイル書き込み |
| `OrcIterable` | 反復読み取り |
| `OrcMetrics` | カラム統計収集 |
| `ExpressionToSearchArgument` | Iceberg 式 → ORC SearchArgument (述語プッシュダウン) |

### Arrow (`arrow/`)

| クラス | 役割 |
|--------|------|
| `ArrowReader` | Iceberg テーブル → Arrow `VectorSchemaRoot` |
| `VectorizedArrowReader` | Parquet → Arrow ベクトル |
| `ColumnarBatch` / `ColumnVector` | Arrow ベクトルをラップするバッチ抽象 |
| `ArrowSchemaUtil` | Iceberg → Arrow スキーマ変換 |
