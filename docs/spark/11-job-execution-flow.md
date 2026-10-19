# ジョブ実行フロー: SparkContext 初期化から結果返却まで

> 調査日: 2026-10-19
> 対象: master ブランチ (commit `2572d806923`)
>
> [02-core.md](02-core.md) のスケジューラ節の深掘り。1 回の action が driver / executor をどう往復するかをメソッド単位で追う。
> ソースの位置は `core/.../X.scala` 形式で示す。`...` は `src/main/scala/org/apache/spark` の省略。行番号は上記コミット時点のもの。

題材は以下の `collect()` 1 回分。shuffle が 1 回入るので Stage は 2 つになる。

```scala
sc.textFile(path).flatMap(_.split(" ")).map((_, 1)).reduceByKey(_ + _).collect()
//  ├──────────── Stage 0 (ShuffleMapStage) ─────────┤├─ Stage 1 (ResultStage) ─┤
```

## 全体フロー

```
══════════════════ driver JVM ══════════════════
[0] SparkContext 初期化
     SparkEnv (RpcEnv, BlockManager, MapOutputTracker, ShuffleManager …)
     createTaskScheduler → (SchedulerBackend, TaskSchedulerImpl)
     new DAGScheduler → taskScheduler.start() → backend.start() (executor 確保開始)
                                            │
[1] action: rdd.collect() → sc.runJob → dagScheduler.runJob → submitJob
     └ JobSubmitted を eventProcessLoop に投入 (単一スレッドのイベントループ)
[2] DAGScheduler (Stage 単位)
     handleJobSubmitted → createResultStage (ShuffleDependency を遡り親 ShuffleMapStage も生成)
     submitStage(final) → 未計算の親から再帰的に submit
     submitMissingTasks → (rdd, dep / func) を serialize して broadcast → ShuffleMapTask / ResultTask
     taskScheduler.submitTasks(TaskSet)
[3] TaskSchedulerImpl (Task 単位)
     TaskSetManager を Pool (FIFO/FAIR) に登録 → backend.reviveOffers()
[4] CoarseGrainedSchedulerBackend (driver 側 RPC endpoint)
     makeOffers: 空き executor の WorkerOffer を作る
     scheduler.resourceOffers: locality を考慮して task を割り当て
     launchTasks → LaunchTask(serializedTask) ────────────┐
══════════════════ executor JVM ══════════════════         ▼
[5] CoarseGrainedExecutorBackend.receive(LaunchTask) → Executor.launchTask
     TaskRunner (スレッドプール) → Task.run → runTask
       ShuffleMapTask: rdd.iterator → ShuffleWriter → MapStatus
       ResultTask    : func(context, rdd.iterator)  → 結果値
     結果を serialize (大きければ BlockManager に置いて IndirectTaskResult)
     statusUpdate(FINISHED) ─────────────────────────────┐
══════════════════ driver JVM ══════════════════         ▼
[6] backend の StatusUpdate → TaskSchedulerImpl.statusUpdate → TaskResultGetter (別スレッドプール)
     → TaskSetManager.handleSuccessfulTask → DAGScheduler.taskEnded (CompletionEvent)
[7] DAGScheduler.handleTaskCompletion
     ShuffleMapTask: MapOutputTracker.registerMapOutput → Stage 完了なら子 Stage を submit (→[2])
     ResultTask    : JobWaiter.taskSucceeded → resultHandler → 全部揃えば runJob が return
```

## [0] SparkContext 初期化

`core/.../SparkContext.scala` のコンストラクタ本体 (おおよそ 400〜700 行目) を上から読むと、以下の順で生成される。

| 順  | 生成物                                                                                                       | 役割                                                                                                                                                                                                                                                         |
| --- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | `_conf`, `LiveListenerBus`, `AppStatusStore`                                                                 | 設定。イベントバス (UI / event log はこれを購読)                                                                                                                                                                                                             |
| 2   | **`SparkEnv`** (`createSparkEnv`, 497 行目)                                                                  | driver 側実行環境一式: `RpcEnv`, `SerializerManager`, `BlockManager`, `MapOutputTrackerMaster`, `ShuffleManager`, `MemoryManager`, `BroadcastManager`, `OutputCommitCoordinator`。executor 側にも同型の `SparkEnv` が作られる (`SparkEnv.createExecutorEnv`) |
| 3   | `HeartbeatReceiver` (589 行目)                                                                               | executor からの heartbeat を受ける endpoint (メトリクス + 生存確認)                                                                                                                                                                                          |
| 4   | `PluginContainer`, `ResourceProfileManager`                                                                  | プラグイン、リソースプロファイル                                                                                                                                                                                                                             |
| 5   | **`createTaskScheduler(this, master)`** (603 行目)                                                           | master URL ごとに `(SchedulerBackend, TaskSchedulerImpl)` を生成。`local*` → `LocalSchedulerBackend`、`spark://` → `StandaloneSchedulerBackend`、それ以外 → `ExternalClusterManager` (YARN/K8s を ServiceLoader で発見)                                      |
| 6   | **`new DAGScheduler(this)`** (606 行目)                                                                      | Stage 計画役。内部で `DAGSchedulerEventProcessLoop` を起動                                                                                                                                                                                                   |
| 7   | `_taskScheduler.start()` (632 行目)                                                                          | `backend.start()` を呼ぶ。Standalone なら `StandaloneAppClient` が Master に `RegisterApplication` を送る。speculation チェックもここで開始                                                                                                                  |
| 8   | `blockManager.initialize(appId)` (653 行目), ExecutorAllocationManager, ContextCleaner, EventLoggingListener | ストレージ初期化、動的アロケーション、GC 連動の後始末、event log                                                                                                                                                                                             |
| 9   | `postApplicationStart()` (700 行目)                                                                          | `SparkListenerApplicationStart` を発行                                                                                                                                                                                                                       |

- 7 の時点で executor はまだいない。executor JVM は起動後 `CoarseGrainedExecutorBackend.onStart` で driver に `RegisterExecutor` を送り、`CoarseGrainedSchedulerBackend.executorDataMap` に載った時点で初めて task を割り当てられる。
- `TaskScheduler` (どの task をどこで) と `SchedulerBackend` (実際に executor とやり取り) が分離されていることが、cluster manager を差し替え可能にしている。

## [1] action → runJob

| 場所                                              | 処理                                                                                                                     |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `RDD.collect`                                     | `sc.runJob(this, (iter: Iterator[T]) => iter.toArray)`                                                                   |
| `SparkContext.runJob` (2484 行目〜)               | クロージャを clean して `dagScheduler.runJob` へ。終了後 `rdd.doCheckpoint()`                                            |
| `DAGScheduler.runJob` (1485) → `submitJob` (1426) | jobId 採番、`JobWaiter` 生成、`JobSubmitted` を event loop へ。呼び出しスレッドは `JobWaiter` の Future を待ってブロック |

`DAGScheduler` の状態変更はすべて**単一スレッドの event loop** (`DAGSchedulerEventProcessLoop.doOnReceive`, 4657 行目) で行われる: `JobSubmitted`, `CompletionEvent`, `ExecutorLost`, `ResubmitFailedStages` 等。DAGScheduler の内部状態にほとんどロックが無いのはこのため。

## [2] DAGScheduler: lineage から Stage を作って投入

| メソッド (`core/.../scheduler/DAGScheduler.scala`)                                                               | 処理                                                                                                                                                                                                                                                                                                                                                 |
| ---------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `handleJobSubmitted` (2019)                                                                                      | `createResultStage` で最終 Stage を作り、`ActiveJob` を登録して `submitStage(finalStage)`                                                                                                                                                                                                                                                            |
| `createResultStage` (912) / `getOrCreateParentStages` (936) / `getShuffleDependenciesAndResourceProfiles` (1009) | RDD 依存を遡る。`NarrowDependency` は同じ Stage にまとめ、**`ShuffleDependency` を境界に親 `ShuffleMapStage` を作る**。ShuffleMapStage は shuffleId ごとに再利用 (`getOrCreateShuffleMapStage`, 625)                                                                                                                                                 |
| `submitStage` (2232)                                                                                             | `getMissingParentStages` (1045) で出力未生成の親を探す。無ければ `submitMissingTasks`、あれば親を再帰 submit して自分は `waitingStages` へ                                                                                                                                                                                                           |
| `submitMissingTasks` (2560)                                                                                      | `stage.findMissingPartitions()` の各パーティションについて `getPreferredLocs` (キャッシュ位置 / HDFS ブロック位置 / shuffle 出力位置) を求める。`(rdd, shuffleDep)` または `(rdd, func)` を **serialize して broadcast** (`taskBinary`)、パーティションごとに `ShuffleMapTask` / `ResultTask` を作り `TaskSet` として `taskScheduler.submitTasks` へ |

- Stage 境界 = shuffle 境界。narrow な変換は 1 task 内の iterator 連鎖としてパイプライン実行される。
- task 本体 (RDD lineage + クロージャ) は broadcast され、各 task が運ぶのはパーティション番号等の軽い情報だけ。
- master には `isPipelinedProducer` / `submitWaitingPipelinedChildStages` による親子 Stage の同時スケジュールも入っている ([02-core.md](02-core.md#shuffle-システム) の `PipelinedShuffleDependency` 関連)。基本フローを追う段階では読み飛ばしてよい。

## [3] TaskSchedulerImpl / TaskSetManager: task をどこに置くか

| クラス・メソッド                                                                             | 処理                                                                                                                                                                                                                                          |
| -------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TaskSchedulerImpl.submitTasks` (276)                                                        | `TaskSetManager` を作り `schedulableBuilder` (FIFO / FAIR の Pool) に登録 → `backend.reviveOffers()`                                                                                                                                          |
| `TaskSchedulerImpl.resourceOffers` (550)                                                     | backend から渡される `WorkerOffer` (executor ごとの空き CPU / リソース) に対し、Pool 優先順に TaskSetManager を並べ、locality レベル (PROCESS_LOCAL → NODE_LOCAL → RACK_LOCAL → ANY) を下げながら `resourceOfferSingleTaskSet` (435) で埋める |
| `TaskSetManager.resourceOffer` (471) / `dequeueTask` (383) / `getAllowedLocalityLevel` (632) | 1 TaskSet 内でこの executor にどの task を出すか決定。**delay scheduling** (`spark.locality.wait`) で良い locality を少し待つ                                                                                                                 |
| `TaskSetManager.handleSuccessfulTask` (823) / `handleFailedTask` (992)                       | 成否の記録。失敗 task の再試行 (`spark.task.maxFailures`)、excludeOnFailure、投機実行 (`checkSpeculatableTasks`)                                                                                                                              |

DAGScheduler は「Stage とどのパーティションが必要か」、TaskScheduler は「task をどこで実行し、失敗したら再試行するか」を担う。task 単位の失敗はここで吸収され、shuffle 出力の消失 (`FetchFailed`) のときだけ DAGScheduler に戻って Stage ごと再実行される。

## [4] CoarseGrainedSchedulerBackend: driver 側の送り出し口

`core/.../scheduler/cluster/CoarseGrainedSchedulerBackend.scala` の `DriverEndpoint`:

| メッセージ / メソッド                    | 処理                                                                                                                   |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `RegisterExecutor` (256)                 | executor 登録。`executorDataMap` に追加し `makeOffers`                                                                 |
| `ReviveOffers` (195。定期的にも送られる) | `makeOffers()`                                                                                                         |
| `makeOffers` (380)                       | 生きている executor の空きを `WorkerOffer` にし `scheduler.resourceOffers` → `launchTasks`                             |
| `launchTasks` (434)                      | `TaskDescription` を serialize して `executorEndpoint.send(LaunchTask(...))`。`spark.rpc.message.maxSize` 超過時は中止 |
| `StatusUpdate` (172)                     | executor からの状態更新を `scheduler.statusUpdate` へ転送。終わった分の CPU を空きに戻して再度 `makeOffers`            |

"Coarse-grained" = executor を長寿命にして多数の task を流し込む方式 (task ごとにプロセスを起動しない)。

## [5] executor 側: task の実行

| クラス・メソッド                                                              | 処理                                                                                                                                                                                                                                                                             |
| ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CoarseGrainedExecutorBackend.main` / `run` (`core/.../executor/`)            | executor JVM の入口。driver から設定を取得、`SparkEnv.createExecutorEnv`、endpoint 起動                                                                                                                                                                                          |
| `onStart` (74) → `RegisteredExecutor` (168)                                   | driver に登録、成功したら `new Executor(...)`                                                                                                                                                                                                                                    |
| `receive(LaunchTask)` (181)                                                   | `TaskDescription` を deserialize して `executor.launchTask`                                                                                                                                                                                                                      |
| `Executor.launchTask` (551)                                                   | `TaskRunner` を作り `threadPool.execute`。1 executor 内で CPU コア数分の task が並列に動く                                                                                                                                                                                       |
| `TaskRunner.run` (815)                                                        | 依存 jar 更新 → task deserialize → `statusUpdate(RUNNING)` → **`task.run(...)`** → 結果 serialize。`maxDirectResultSize` 超なら BlockManager に置いて `IndirectTaskResult`、`spark.driver.maxResultSize` 超なら破棄 → `statusUpdate(FINISHED)`。例外種別に応じて FAILED / KILLED |
| `Task.run` (`scheduler/Task.scala`, 87)                                       | `TaskContext` 生成 (スレッドローカルに設定)、メトリクス・`TaskMemoryManager` 準備 → `runTask`                                                                                                                                                                                    |
| `ShuffleMapTask.runTask` (82)                                                 | broadcast された `(rdd, dep)` を deserialize → `ShuffleWriteProcessor.write` → `shuffleManager.getWriter(...)` に `rdd.iterator(partition)` を書き込み → **`MapStatus`** (どのブロックに何バイト書いたか) を返す                                                                 |
| `ResultTask.runTask` (78)                                                     | `(rdd, func)` を deserialize → `func(context, rdd.iterator(partition))`。collect なら配列を返す                                                                                                                                                                                  |
| `RDD.iterator` (334) → `getOrCompute` / `computeOrReadCheckpoint` → `compute` | キャッシュ・チェックポイントがあれば使い、無ければ `compute`。narrow 依存は親の `iterator` を呼ぶ連鎖になる。shuffle 読み側 (`ShuffledRDD.compute`) は `shuffleManager.getReader` で各 map 出力を取得                                                                            |
| `reportHeartBeat` (1544)                                                      | 実行中 task のメトリクス・accumulator 途中経過を driver に送る                                                                                                                                                                                                                   |

Stage 1 の `ShuffledRDD.compute` は `MapOutputTracker` に各 map 出力の所在を問い合わせ、`ShuffleBlockFetcherIterator` が他 executor (または External Shuffle Service) から取得する。shuffle の深掘りはここが入口。

## [6][7] 結果の回収と次の Stage

| クラス・メソッド                           | 処理                                                                                                                                                                                                                                                                                                                                                                                      |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `TaskSchedulerImpl.statusUpdate` (842)     | FINISHED → `TaskResultGetter.enqueueSuccessfulTask`、FAILED → `enqueueFailedTask`                                                                                                                                                                                                                                                                                                         |
| `TaskResultGetter`                         | 別スレッドプールで結果を deserialize。`IndirectTaskResult` なら `blockManager.getRemoteBytes` で取得してから `scheduler.handleSuccessfulTask`                                                                                                                                                                                                                                             |
| `TaskSetManager.handleSuccessfulTask`      | 成功記録 → `dagScheduler.taskEnded` → `CompletionEvent` を event loop へ                                                                                                                                                                                                                                                                                                                  |
| `DAGScheduler.handleTaskCompletion` (3151) | **ShuffleMapTask**: `mapOutputTracker.registerMapOutput` で MapStatus を登録。Stage の全パーティションが揃えば `markStageAsFinished` → **`submitWaitingChildStages`** (待っていた Stage 1 を submit → [2] へ戻る)。**ResultTask**: `job.listener.taskSucceeded` (= `JobWaiter`) → `resultHandler` で結果配列の該当位置に格納。全部揃えば job 完了、[1] でブロックしていた `runJob` が戻る |
| `FetchFailed` 処理 (3391〜)                | shuffle 出力の取得失敗 → 該当 map 出力を unregister し、親 ShuffleMapStage と現 Stage を再実行対象に (`ResubmitFailedStages`)                                                                                                                                                                                                                                                             |

## 責務の分担

| 層                                     | 単位        | 担当する                                                              | 担当しない               |
| -------------------------------------- | ----------- | --------------------------------------------------------------------- | ------------------------ |
| `DAGScheduler`                         | job / stage | Stage 分割、親子順序、shuffle 出力管理、FetchFailed 時の Stage 再実行 | どの executor で動かすか |
| `TaskSchedulerImpl` + `TaskSetManager` | task        | 配置 (locality, delay scheduling)、task 再試行、投機実行、Pool 優先度 | RDD lineage              |
| `SchedulerBackend`                     | executor    | executor 登録・管理、RPC 送受信、cluster manager との交渉             | どの task を出すか       |
| `Executor`                             | thread      | task 実行、メモリ、結果返送                                           | スケジューリング全般     |

## SQL との接続

SQL の場合、`QueryExecution.toRdd` が作った RDD に対して `SparkPlan.executeCollect` 等から `sc.runJob` が呼ばれ、[1] 以降は全く同じ経路を通る ([03-sql-catalyst.md](03-sql-catalyst.md))。AQE は ShuffleMapStage を 1 つずつ実行し (`submitMapStage` → `handleMapStageSubmitted`)、その統計を見て残りのプランを書き換える。

## プロセス配置との関係: Master / Worker / Driver / Executor

- **Master / Worker** = **クラスタ層**。リソース管理とプロセス起動を担う。Standalone モード固有の常駐デーモン (`core/.../deploy/`)。
- **Driver / Executor** = **アプリケーション層**。計算の指揮と実行を担う。どの cluster manager でも共通。

```
                 ┌──────────── クラスタ層 (Standalone の常駐デーモン) ────────────┐
                 │  Master (1 台が ALIVE。HA なら standby あり)                      │
                 │    ▲ RegisterWorker / Heartbeat      │ LaunchExecutor / LaunchDriver │
                 │    │                                 ▼                              │
                 │  Worker (各ノードに 1 つ) ── 子プロセスとして JVM を起動・監視       │
                 └────────────────────────────────┬─────────────────────────────┘
       RegisterApplication / RequestExecutors     │ fork
                 ▲                                ▼
 ┌──────── アプリケーション層 (アプリごとに起動・終了) ────────┐
 │  Driver (1 アプリに 1 つ)                                      │
 │    SparkContext / DAGScheduler / TaskScheduler / SchedulerBackend│
 │          │ LaunchTask        ▲ StatusUpdate / Heartbeat        │
 │          ▼                   │                                  │
 │  Executor (1 アプリに N 個。各ノードの別 JVM)                   │
 │    スレッドプールで task 実行 / BlockManager / shuffle 読み書き   │
 │          ◀──── shuffle / broadcast / キャッシュのブロックを相互取得 ────▶ │
 └──────────────────────────────────────────────────────┘
```

### 4 者の比較

|                        | Master                                                                                        | Worker                                                                         | Driver                                                                               | Executor                                                      |
| ---------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------ | ------------------------------------------------------------- |
| 層                     | クラスタ                                                                                      | クラスタ                                                                       | アプリ                                                                               | アプリ                                                        |
| 数                     | クラスタに 1 台 (+ standby)                                                                   | ノードごとに 1 つ                                                              | アプリごとに 1 つ                                                                    | アプリごとに N 個                                             |
| 寿命                   | 常駐                                                                                          | 常駐                                                                           | アプリ実行中のみ                                                                     | アプリ実行中のみ (動的アロケーションで増減)                   |
| 主な責務               | Worker 資源 (コア / メモリ) の台帳、アプリへの executor 割り当て、cluster mode の driver 配置 | 自ノードで executor / driver の JVM を起動・監視・停止し、状態を Master に報告 | ユーザーの `main` を実行。ジョブを stage / task に分割し、配置・再試行・結果回収     | task を実行。shuffle 出力の書き込み、キャッシュ保持、結果返送 |
| ユーザーコードを知るか | 知らない                                                                                      | 知らない                                                                       | **知る** (main そのもの)                                                             | **知る** (serialize されたクロージャと RDD lineage)           |
| ジョブ / task を知るか | 知らない                                                                                      | 知らない                                                                       | **知る**                                                                             | 割り当てられた task のみ                                      |
| 主なクラス             | `deploy.master.Master`                                                                        | `deploy.worker.Worker`, `ExecutorRunner`, `DriverRunner`                       | `SparkContext`, `DAGScheduler`, `TaskSchedulerImpl`, `CoarseGrainedSchedulerBackend` | `CoarseGrainedExecutorBackend`, `Executor`, `TaskRunner`      |
| RPC endpoint の種別    | `ThreadSafeRpcEndpoint` (共有ループ)                                                          | `ThreadSafeRpcEndpoint` (共有ループ)                                           | `DriverEndpoint`: `IsolatedThreadSafeRpcEndpoint` (専用ループ)                       | `IsolatedThreadSafeRpcEndpoint` (専用ループ)                  |

### ライフサイクル (Standalone / client mode)

1. **クラスタ起動**: `start-master.sh` → Master。`start-worker.sh spark://m:7077` → Worker が `RegisterWorker`、以後 Heartbeat
2. **アプリ投入**: `spark-submit` の JVM がユーザーの `main` を実行し、その JVM が **Driver** になる (`SparkSubmit.runMain` → `JavaMainApplication.start` → リフレクションで `main` を invoke)
3. **executor 確保**: Driver の `SparkContext` → `StandaloneAppClient` が Master に `RegisterApplication` → Master の `schedule()` → 各 Worker に `LaunchExecutor` ([0] の 7)
4. **executor 起動**: Worker の `ExecutorRunner` が `CoarseGrainedExecutorBackend` の JVM を fork → Worker が Master に `ExecutorStateChanged` → Master が Driver に `ExecutorAdded`
5. **Driver ↔ Executor 直結**: Executor が **Driver に直接** `RegisterExecutor`。**以降 Master / Worker は経路に入らない**
6. **ジョブ実行**: action ごとに [1]〜[7] の往復 (`LaunchTask` / `StatusUpdate`)。shuffle データは Executor 同士 (または External Shuffle Service) でやり取り
7. **終了**: `sc.stop()` → Driver が Master に `UnregisterApplication` → Master が Worker に executor 停止を指示 → Worker が JVM を終了

**cluster mode** では 2 が「`spark-submit` は `ClientApp` / `RestSubmissionClientApp` を実行 → Master に `RequestSubmitDriver` → Worker の `DriverRunner` が `DriverWrapper` 経由で Driver JVM を起動」に変わる。3 以降は同じ。Master が担うのは driver の JVM をどこで起動するかの決定までで、ジョブの指揮は起動された Driver が行う。

### よくある誤解

| 誤解                                    | 実際                                                                                              |
| --------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Master がジョブを実行・指揮する         | 指揮するのは **Driver**。Master はリソース割り当てまで                                            |
| Worker が task を実行する               | task を実行するのは **Executor** (Worker が起動した別 JVM)。Worker はプロセス管理役               |
| Master が落ちると実行中のジョブも止まる | Driver ↔ Executor は直結なので**止まらない**。新規アプリ登録と executor の増減ができなくなるだけ |
| Driver が落ちても Executor は動き続ける | **アプリ全体が終わる**。Executor は Driver との接続が切れると自分も終了する                       |
| executor は 1 ノードに 1 つ             | 1 Worker が複数アプリ・複数 executor を同時に抱えられる (コア / メモリの空きの範囲で)             |

### cluster manager ごとの対応

| 役割                         | Standalone                                                              | YARN                                                                                  | Kubernetes                                 | `local`                                | `local-cluster`                             |
| ---------------------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------ | -------------------------------------- | ------------------------------------------- |
| クラスタ資源の管理           | Master                                                                  | ResourceManager                                                                       | API Server + scheduler                     | なし                                   | driver JVM 内の Master                      |
| ノード上のプロセス起動       | Worker                                                                  | NodeManager                                                                           | kubelet                                    | なし                                   | driver JVM 内の Worker                      |
| Driver の居場所              | client: 投入元の JVM / cluster: Worker の `DriverRunner` が起動した JVM | client: 投入元の JVM / cluster: ApplicationMaster の中                                | client: 投入元の JVM / cluster: driver Pod | 投入元の JVM                           | 投入元の JVM                                |
| Executor の実体              | Worker の `ExecutorRunner` が起動した JVM                               | Container の JVM                                                                      | executor Pod                               | driver JVM 内 (スレッドで task を実行) | Worker の `ExecutorRunner` が起動した別 JVM |
| driver 側の SchedulerBackend | `StandaloneSchedulerBackend`                                            | `YarnSchedulerBackend` (`YarnClientSchedulerBackend` / `YarnClusterSchedulerBackend`) | `KubernetesClusterSchedulerBackend`        | `LocalSchedulerBackend`                | `StandaloneSchedulerBackend`                |

どのモードでも **Driver ↔ Executor の関係 (`CoarseGrainedSchedulerBackend` ↔ `CoarseGrainedExecutorBackend`) は共通**で (`local` のみ `LocalSchedulerBackend` が driver 内で直接実行)、cluster manager ごとに変わるのは「誰が executor の JVM / Pod / Container を確保・起動するか」の部分だけ。

## 読む順番

1. `SparkContext` コンストラクタ本体を上から読み、[0] の表と照合
2. `RDD.collect` → `SparkContext.runJob` → `DAGScheduler.runJob` / `submitJob` → `doOnReceive`
3. `handleJobSubmitted` → `createResultStage` → `submitStage` → `submitMissingTasks` (Stage 分割が DAGScheduler の核心)
4. `TaskSchedulerImpl.submitTasks` → `resourceOffers` (locality の細部は後回しでよい)
5. `CoarseGrainedSchedulerBackend` の `makeOffers` / `launchTasks`
6. `CoarseGrainedExecutorBackend.receive` → `Executor.TaskRunner.run` → `Task.run` → `ShuffleMapTask` / `ResultTask`
7. `TaskResultGetter` → `DAGScheduler.handleTaskCompletion` (往路と復路がここで繋がる)
8. 次の深掘り候補: shuffle (`SortShuffleManager`, `ShuffleBlockFetcherIterator`, `MapOutputTracker`) または障害処理 (`FetchFailed`, `ExecutorLost`)

## 動かしながら読む

`spark-shell` で `sc.setLogLevel("INFO")` にして題材のコードを実行すると、`Submitting ShuffleMapStage 0`、`Missing parents found for ResultStage 1`、`Starting task 0.0 in stage 0.0 (... PROCESS_LOCAL ...)` といったログが上記の順に流れる。

プロセス配置も確認したい場合は `sbin/start-master.sh` + `sbin/start-worker.sh spark://localhost:7077` で Standalone を立て、`spark-shell --master spark://localhost:7077` で接続する。`jps` で `Master` / `Worker` / `CoarseGrainedExecutorBackend` / `SparkSubmit` が別 JVM として見え、UI は Master 8080 / Worker 8081 / アプリ 4040。

デーモンを別途立てずに試すなら `--master local-cluster[2,1,1024]`。`SparkContext.createTaskScheduler` の `LOCAL_CLUSTER_REGEX` 分岐で `LocalSparkCluster.start()` が driver JVM 内に Master / Worker の RpcEnv を立てる (`Master.main` は経由せず `Master.startRpcEnvAndEndpoint` をポート 0 で直接呼ぶ。実ポートは `BoundPortsRequest` で取得)。ただし executor は Worker の `ExecutorRunner` が別 JVM として起動するので、1 JVM にはならない (ビルド済み `SPARK_HOME` が必要)。完全に 1 JVM で動かすのは `local[N]` (`LocalSchedulerBackend` が driver 内に executor を直接作る)。
