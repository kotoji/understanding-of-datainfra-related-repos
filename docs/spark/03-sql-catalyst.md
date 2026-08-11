# Spark SQL / Catalyst

## 全体パイプライン

```
SQL Text / DataFrame API
    │
    ▼
[Parser] ─── AstBuilder, CatalystSqlParser
    │         未解決 LogicalPlan を生成
    ▼
[Analyzer] ─── 219KB, 60+ ルール (Resolution batch)
    │           参照解決, 関数解決, 型強制, サブクエリ解決
    ▼
[Optimizer] ─── 136KB, 47 最適化ルールファイル
    │            Push-down, Folding, Join Reorder, CBO
    ▼
[SparkPlanner] ─── 16 Strategy (JoinSelection, Aggregation 等)
    │               LogicalPlan → SparkPlan 変換
    ▼
[Preparations] ─── 16 ルール
    │               AQE挿入, Shuffle/Sort挿入, CodeGen挿入
    ▼
[Execution] ─── SparkPlan.execute() → RDD[InternalRow]
```

## Catalyst ルールフレームワーク

```scala
abstract class Rule[TreeType <: TreeNode[_]] {
  def apply(plan: TreeType): TreeType
}
```

`RuleExecutor` がルールを `Batch` にグループ化。`Strategy` は `Once` か `FixedPoint(maxIterations)`:
- `Once` — 1回適用
- `FixedPoint` — 計画が変化しなくなるまで繰り返し

## Analyzer の主要 Batch

1. **Substitution** — CTE 置換、Union 除去
2. **Hints** — Join Strategy / Coalesce ヒント解決
3. **Resolution** (FixedPoint, 60+ ルール) — カタログ解決、参照解決、関数解決、型強制、サブクエリ解決
4. **Post-Hoc Resolution** — IF EXISTS コマンド
5. **UDF Handling** — NULL 入力処理
6. **Cleanup** — エイリアス整理

## Optimizer の主要最適化カテゴリ

- **Predicate Push-Down:** `PushDownPredicates`, `PushExtraPredicateThroughJoin`
- **Projection Push-Down:** `ColumnPruning`, `PushProjectionThroughUnion`
- **Constant Folding:** `ConstantFolding`, `ConstantPropagation`, `NullPropagation`, `BooleanSimplification`
- **Join 最適化:** `ReorderJoin`, `CostBasedJoinReorder`, `EliminateOuterJoin`, `StarSchemaDetection`
- **Aggregate 最適化:** `RewriteDistinctAggregates`, `RemoveRedundantAggregates`
- **Subquery 最適化:** `RewriteCorrelatedScalarSubquery`, `DecorrelateInnerQuery`
- **Filter 推論:** `InferFiltersFromConstraints`
- **Runtime Filter:** `InjectRuntimeFilter`

**SparkOptimizer** は基本 Optimizer に加えて:
- V2 Scan Push-Down (`V2ScanRelationPushDown`)
- Partition Pruning (`PartitionPruning`)
- Python UDF 抽出 (`ExtractPythonUDFs`)

## Physical Plan Operators

| カテゴリ | 主要オペレータ |
|---------|--------------|
| Scan | `FileSourceScanExec`, `BatchScanExec` (V2) |
| Project/Filter | `ProjectExec`, `FilterExec` |
| Join | `BroadcastHashJoinExec`, `SortMergeJoinExec`, `ShuffledHashJoinExec`, `BroadcastNestedLoopJoinExec` |
| Aggregate | `HashAggregateExec`, `SortAggregateExec`, `ObjectHashAggregateExec` |
| Sort | `SortExec` (external sort with spill) |
| Exchange | `ShuffleExchangeExec`, `BroadcastExchangeExec` |
| Window | Window function execution |
| Limit | `LocalLimitExec`, `GlobalLimitExec`, `TakeOrderedAndProjectExec` |

## Whole-Stage Code Generation (WSCG)

複数の physical operator を単一の Java 関数に融合し、仮想関数呼び出しと中間データのマテリアライズを排除する。

- `CodegenSupport` trait: `produce` (top-down) + `doConsume` (bottom-up)
- `WholeStageCodegenExec`: `BufferedRowIterator` を拡張する Java クラスを生成
- Janino によるインメモリ Java コンパイル
- コンパイル失敗時はインタプリタモードにフォールバック

**パイプラインモデル:** scan → blocking operator (aggregate, sort, join build side) が1つの codegen ステージ。`InputAdapter` で codegen 外のオペレータとブリッジ。

## Adaptive Query Execution (AQE)

`AdaptiveSparkPlanExec` がプラン全体をラップし、shuffle 境界でステージ分割:

1. **Bottom-up ステージ作成** — exchange ノードで分割
2. **ステージ実行** — 子ステージを全てマテリアライズ
3. **再最適化** — 実行時統計 (partition サイズ) を用いてプラン再構築

**主要ルール:**
- `CoalesceShufflePartitions` — 小さい partition を統合
- `OptimizeSkewedJoin` — skew partition を複数タスクに分割
- `ConvertSortMergeJoinToShuffledHashJoin` — 実際のサイズに基づく join 戦略変更
- `DemoteBroadcastHashJoin` — broadcast データが大きすぎる場合 SMJ に戻す

## Data Source API

**V1 API (Scala trait ベース):**
- `RelationProvider` → `BaseRelation` → `PrunedFilteredScan`
- 組込み: parquet, orc, json, csv, text, jdbc, xml

**V2 API (Java interface ベース, 推奨):**
- `TableCatalog` / `Table` / `ScanBuilder` → `Scan` → `Batch` / `MicroBatchStream` / `ContinuousStream`
- Push-down mixin: Filters, RequiredColumns, Aggregates, Limit, TopN, Join, VariantExtractions
- Reporting mixin: Statistics, Partitioning, Ordering
- `V2ScanRelationPushDown` optimizer ルールで push-down を制御
