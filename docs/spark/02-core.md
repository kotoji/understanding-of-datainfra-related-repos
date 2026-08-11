# Spark Core

## SparkContext (3,617行)

Spark クラスタへの接続を表す中央コーディネータ。JVM あたり1インスタンスのみ。

**初期化シーケンス:**
1. `SparkConf` のバリデーション
2. `LiveListenerBus` / `AppStatusStore` 作成
3. `SparkEnv.createDriverEnv` → RPC環境, Serializer, MapOutputTracker, BroadcastManager, BlockManager, MetricsSystem 生成
4. `HeartbeatReceiver` RPC endpoint 登録
5. `createTaskScheduler(master)` で master URL に基づくスケジューラ構築
6. `DAGScheduler` 作成
7. `TaskScheduler` 開始

**master URL ディスパッチ:**
- `local[N]` → `LocalSchedulerBackend`
- `spark://...` → `StandaloneSchedulerBackend`
- その他 → `ServiceLoader` で `ExternalClusterManager` (YARN, K8s)

## RDD (Resilient Distributed Dataset)

RDD は5つの性質で特徴付けられる:
1. **Partition のリスト**
2. 各 Partition の**計算関数** (`compute`)
3. 他 RDD への**依存関係** (`Dependency`)
4. (optional) `Partitioner` (key-value RDD)
5. (optional) **preferred locations** (データ局所性)

**依存関係の型:**
```
Dependency[T]
  ├─ NarrowDependency (パイプライン実行, shuffle 不要)
  │    ├─ OneToOneDependency (map)
  │    └─ RangeDependency (union)
  └─ ShuffleDependency (shuffle 境界)
       └─ PipelinedShuffleDependency (Spark 5.0 新規: インクリメンタル読み出し)
```

**主要 RDD サブクラス** (`core/.../rdd/` は 42 の `.scala` ファイル。うち `RDDCheckpointData` 等の非サブクラスも含む):
- `MapPartitionsRDD` — map/flatMap/filter の結果
- `ShuffledRDD` — shuffle 操作の結果
- `ParallelCollectionRDD` — `sc.parallelize()` で生成
- `HadoopRDD` / `NewHadoopRDD` — HDFS 読み込み
- `UnionRDD`, `CoalescedRDD`, `CoGroupedRDD`, `CartesianRDD` 等

## スケジューラアーキテクチャ

3層構造: **DAGScheduler → TaskScheduler → SchedulerBackend**

### DAGScheduler (4,464行)

RDD の lineage グラフから Stage の DAG を構築する。

**実行フロー:**
1. `submitJob` → `JobSubmitted` イベントをイベントループに投入
2. `handleJobSubmitted` → RDD 依存グラフを逆走、`ShuffleDependency` 境界で `ShuffleMapStage` を生成、最終 RDD は `ResultStage`
3. `submitStage` → 親 Stage を再帰的に先行 submit (トポロジカルソート)
4. `submitMissingTasks` → 未計算 Partition に対して `ShuffleMapTask` / `ResultTask` を生成、`TaskSet` として `TaskScheduler` に渡す
5. `handleTaskCompletion` → shuffle map output を `MapOutputTracker` に登録 / result を `JobWaiter` に配信

**Stage の種類:**
- `ShuffleMapStage` — shuffle 出力を生成する中間ステージ
- `ResultStage` — アクション結果を計算する最終ステージ

### TaskScheduler / SchedulerBackend

- `TaskSchedulerImpl` — `TaskSetManager` でタスクセットを管理、FIFO / Fair スケジューリング対応
- `CoarseGrainedSchedulerBackend` — RPC で executor と通信、`executorDataMap` でリソース追跡
- `LocalSchedulerBackend` — ドライバ JVM 内で executor を直接作成

## Shuffle システム

Spark 5.0 では2種類の ShuffleManager が並存する。`BlockingShuffleManager` / `PipelinedShuffleManager` は共に `ShuffleManager` trait を継承する trait で、各 Manager 実装はどちらか一方を extends する:

| Manager (実装クラス) | 設定キー | 継承元 trait | 説明 |
|---------|----------|------|------|
| `SortShuffleManager` | `spark.shuffle.manager` | `BlockingShuffleManager` | 従来のソートベース shuffle。デフォルト |
| `StreamingShuffleManager` | `spark.shuffle.manager.incremental` | `PipelinedShuffleManager` | **5.0 新規**: producer 実行中に consumer が読み出し開始可能 |

`SparkEnv.shuffleManagerFor(dep)` で `PipelinedShuffleDependency` なら pipelined manager、それ以外なら blocking manager にルーティング。

**SortShuffleManager の3つの書き込みパス:**
1. `BypassMergeSortShuffleHandle` — map-side combine なし、かつ partition 数 ≤ `spark.shuffle.sort.bypassMergeThreshold` (デフォルト 200)
2. `SerializedShuffleHandle` — map-side combine なし、serializer が relocation 対応、partition 数 ≤ 16M (`1 << 24`)
3. `BaseShuffleHandle` — デフォルト (deserialize → sort → spill)

## Storage / キャッシュ

**BlockManager (2,600行):** 全ノード (driver + executor) で動作。メモリ/ディスク/リモートの block ストレージを提供。

**BlockId の種類:** `RDDBlockId`, `ShuffleBlockId`, `BroadcastBlockId`, `TaskResultBlockId` 等

**UnifiedMemoryManager:**
- 管理メモリ = (heap - 300MB) x 0.6
- Storage / Execution で soft boundary、相互借用可能
- Storage が借りた Execution メモリは、キャッシュ eviction で回収可能
- Execution メモリは Storage から回収されない (この非対称設計の意図は [10-design-patterns.md](10-design-patterns.md#two-tier-memory-management) 参照)

## SparkEnv

ランタイムオブジェクトの入れ物。主要コンポーネント:
- `rpcEnv` — RPC 通信
- `serializer` / `closureSerializer` — シリアライザ
- `mapOutputTracker` — shuffle 出力位置追跡
- `broadcastManager` — broadcast 変数管理
- `blockManager` — block ストレージ
- `memoryManager` — メモリ管理
- `_blockingShuffleManager` / `_pipelinedShuffleManager` — 2つの shuffle manager

## Executor (1,659行)

- `launchTask` → `TaskRunner` をスレッドプールに submit
- `TaskRunner.run()` → task deserialize → `TaskContext` セットアップ → `task.run()` → 結果を `ExecutorBackend` 経由で report
- `IsolatedSessionState` — session 毎の classloader/files/jars を LRU キャッシュで管理
