# Structured Streaming

## 実行エンジン

- **`MicroBatchExecution`** — デフォルト。offset tracking + commit log + checkpoint による exactly-once
- **`ContinuousExecution`** — 低遅延連続処理。epoch ベースのチェックポイント

## Trigger

| Trigger | 説明 |
|---------|------|
| `ProcessingTime(interval)` | 固定間隔の micro-batch |
| `AvailableNow()` | 利用可能データを全処理して終了 |
| `Continuous(interval)` | 連続処理、epoch 単位で非同期チェックポイント |
| `RealTime(batchDuration)` | 低遅延な read 用トリガー (新規、デフォルト 5分)。下記参照 |
| `Once()` | 非推奨 (3.4.0~) |

**`RealTime` と `Continuous` の違い:** `Continuous` は epoch ベースでコーディネートされた連続実行モデル。`RealTime` はそれとは別の仕組みで、`SupportsRealTimeMode` / `SupportsRealTimeRead` を実装したソースに対し `nextWithTimeout` によるタイムアウト付きポーリング読み取りを行うことで低遅延化する。利用できる Source/Sink/Operator は `RealTimeModeAllowlist` で明示的に許可されたものに限定される (例: Kafka source, Console/Memory/Foreach sink)。任意の複雑なクエリを低遅延化するものではなく、対応済みの限定的な構成でのみ有効。

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

## チェックポイントディレクトリ構造

`resolvedCheckpointRoot` 配下に以下のログを保持する (`StreamingQueryCheckpointMetadata`):

| ディレクトリ | 管理クラス | 内容 |
|-------------|-----------|------|
| `offsets/` | `OffsetSeqLog` | バッチ毎の入力オフセット。処理**前**に書き込むことで「同じバッチは常に同じデータを指す」ことを保証する |
| `commits/` | `CommitLog` | 完了したバッチ ID。再起動時にどのバッチから再開すべきかの判定に使う |
| (metadata) | `StreamMetadata` | クエリ ID など、クエリ全体に紐づくメタデータ |
| `state/` | `HDFSBackedStateStoreProvider` / `RocksDBStateStoreProvider` | stateful operator の状態データ |

`offsets/` の N 番目のレコードが「現在処理中のバッチ」、N-1 番目が「sink にコミット済みのバッチ」を意味する。この2ログの整合性が exactly-once 保証の基盤になっている。
