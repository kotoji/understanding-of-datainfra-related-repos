# Structured Streaming

## 実行エンジン

- **`MicroBatchExecution`** — デフォルト。offset tracking + commit log + checkpoint による exactly-once
- **`ContinuousExecution`** — 低遅延連続処理。epoch ベースのチェックポイント

## Trigger

| Trigger | 説明 |
|---------|------|
| `ProcessingTime(interval)` | 固定間隔の micro-batch |
| `AvailableNow()` | 利用可能データを全処理して終了 |
| `Continuous(interval)` | 連続処理、非同期チェックポイント |
| `RealTime(batchDuration)` | リアルタイムモード (新規) |
| `Once()` | 非推奨 (3.4.0~) |

## State Management

| Provider | 説明 |
|----------|------|
| `HDFSBackedStateStoreProvider` | delta ファイル + snapshot。HDFS/S3 対応 |
| `RocksDBStateStoreProvider` | RocksDB ベース。大規模 state に適する |

**Stateful Operator:**
- `StateStoreRestoreExec` / `StateStoreSaveExec` — streaming aggregation
- `StreamingSymmetricHashJoinExec` — stream-stream join
- `FlatMapGroupsWithStateExec` — mapGroupsWithState / flatMapGroupsWithState
- `TransformWithStateExec` — **新 API**: `ValueState`, `ListState`, `MapState` + TTL サポート

## Built-in Sources / Sinks

**Sources:** Kafka (primary), FileStream, Rate, TextSocket, Memory (test用)

**Sinks:** FileStream, Console, ForeachBatch, ForeachWriter, Kafka
