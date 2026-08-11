# エンジン連携

## Spark 連携

### 対応バージョン
- Spark 3.5 (Scala 2.12)
- Spark 4.0 (Scala 2.13)
- Spark 4.1 (Scala 2.13)

### DataSource V2 統合

**エントリポイント**: `IcebergSource` (`DataSourceRegister` として `"iceberg"` で登録)

**SparkTable**: Spark の `Table` 実装。`SupportsRead`, `SupportsWrite`, `SupportsDeleteV2`, `SupportsRowLevelOperations`, `SupportsMetadataColumns` を実装。

### Catalog

| クラス | 説明 |
|--------|------|
| `SparkCatalog` | Iceberg `Catalog` をラップ。テーブル・ビュー・namespace 対応 |
| `SparkSessionCatalog` | 非 Iceberg 操作をセッションカタログに委譲。Hive/Iceberg 混在環境用 |

### Read パス

```
SparkScanBuilder (フィルタ変換、カラムプルーニング、集約プッシュダウン、LIMIT プッシュダウン)
  → SparkBatchQueryScan / SparkPartitioningAwareScan
    → SparkBatch → BatchDataReader / RowDataReader
```

ストリーミング: `SparkMicroBatchStream` (Structured Streaming micro-batch source)
CDC: `SparkChangelogScan` / `SparkChangelogTable`

### Write パス

```
SparkWriteBuilder (append/dynamic overwrite/filter overwrite/COW/file rewrite を選択)
  → SparkWrite
      +-- BatchAppend → table.newAppend()
      +-- DynamicOverwrite → table.newReplacePartitions()
      +-- OverwriteByFilter → table.newOverwrite().overwriteByRowFilter(expr)
      +-- CopyOnWriteOperation → table.newOverwrite() (SERIALIZABLE/SNAPSHOT isolation)
      +-- RewriteFiles → FileRewriteCoordinator 経由 (コンパクション)
```

行レベル操作 (UPDATE/DELETE/MERGE): `SparkPositionDeltaWrite` → `RowDelta` 操作

### SQL Extensions & Procedures

`IcebergSparkSessionExtensions` でカスタム SQL パーサー、ビュー解決ルール、オプティマイザルール注入。

主要 Procedure (CALL で実行可能):
- `RewriteDataFilesProcedure` (コンパクション)
- `ExpireSnapshotsProcedure` (スナップショット期限切れ)
- `RewriteManifestsProcedure` (マニフェスト最適化)
- `RemoveOrphanFilesProcedure` (孤立ファイル削除)
- `RollbackToSnapshotProcedure` / `RollbackToTimestampProcedure`
- `MigrateTableProcedure` / `SnapshotTableProcedure` (マイグレーション)
- `ComputeTableStatsProcedure` (統計計算)

---

## Flink 連携

### 対応バージョン
- Flink 1.20
- Flink 2.0
- Flink 2.1

### Catalog: `FlinkCatalog`

Flink の `AbstractCatalog` を継承し、Iceberg `Catalog` をラップ。`FlinkCatalogFactory` (SPI) で発見。

### Source (Read)

**新 API** (`IcebergSource`): FLIP-27 Source API。バウンデッド (バッチ) とアンバウンデッド (ストリーミング) 両対応。
- `StaticIcebergEnumerator` (バッチ) / `ContinuousIcebergEnumerator` (ストリーミング)
- Watermark サポート: `ColumnStatsWatermarkExtractor`

**旧 API**: `FlinkSource` / `FlinkInputFormat` (SourceFunction ベース)

### Sink (Write)

**新 API** (`IcebergSink`): Sink2 API。
- パイプライン: `IcebergSinkWriter` → `IcebergCommittable` → `IcebergCommitter`
- メンテナンスタスク統合 (スナップショット期限切れ、孤立ファイル削除、equality delete 変換)

**旧 API** (`FlinkSink`): DataStream API。
- `IcebergStreamWriter` → `IcebergFilesCommitter`

Dynamic Sink: `DynamicIcebergSink` — 単一パイプラインから複数 Iceberg テーブルへの書き込み

レンジベースシャッフル: `DataStatisticsOperator` / `RangePartitioner` でソートキーに基づくデータ分配

---

## Kafka Connect 連携

### アーキテクチャ: Coordinator-Worker パターン

Kafka コントロールトピック経由の**分散コミットプロトコル**:

```
Coordinator (リーダータスク)
  ├── StartCommit イベント送信
  ├── DataWritten / DataComplete レスポンス収集
  ├── AppendFiles / RowDelta で Iceberg コミット
  └── CommitComplete 送信

Worker (各タスク)
  ├── SinkWriter でバッファリング・書き込み
  └── StartCommit 受信 → フラッシュ → DataWritten 送信
```

### サブモジュール
- `kafka-connect/`: コアコネクタランタイム
- `kafka-connect-events/`: Coordinator-Worker 間イベントプロトコル
- `kafka-connect-transforms/`: SMT (Debezium, DMS, MongoDB 等)

---

## Delta Lake マイグレーション

`delta-lake/` モジュール: Delta Lake → Iceberg への**一方向マイグレーション**。

`BaseSnapshotDeltaLakeTableAction`:
1. Delta Lake トランザクションログ (`DeltaLog`) を読み取り
2. `AddFile`/`RemoveFile` アクションをリプレイ
3. 同スキーマの Iceberg テーブル作成 (`DeltaLakeTypeToType` で型変換)
4. `AppendFiles`/`DeleteFiles` でデータファイルをコミット
5. Delta のスナップショット履歴を対応する Iceberg スナップショットとして保存
