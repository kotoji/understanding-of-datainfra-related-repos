# アーキテクチャ上の設計パターン

## Tree Transformation with Rules

Catalyst の全パイプライン (解析、最適化、物理計画) が同一の `Rule[TreeType]` / `RuleExecutor` パターンを使用。モジュラーで合成可能な変換を実現。

## Lazy Evaluation Chain

`QueryExecution` が `LazyTry` で各パイプラインステージを保持。各ステージは最大1回計算され、エラーは綺麗に伝搬。

## Extensibility Hooks

`SparkSessionExtensions` は 17 の `inject*` メソッドを持つ。主な分類:

- **Parser** — `injectParser`
- **Analyzer** — `injectResolutionRule`, `injectHintResolutionRule`, `injectPostHocResolutionRule`, `injectCheckRule`
- **Optimizer** — `injectOptimizerRule`, `injectPreOperatorOptimizationRule`, `injectPreCBORule`, `injectPlanNormalizationRule`
- **Planner** — `injectPlannerStrategy`, `injectQueryPostPlannerStrategyRule`
- **AQE** — `injectQueryStagePrepRule`, `injectRuntimeOptimizerRule`, `injectQueryStageOptimizerRule`
- **Columnar** — `injectColumnar`
- **関数登録** — `injectFunction`, `injectTableFunction`

## Separation of Concerns

Catalyst (純粋論理、SparkContext 非依存) と SQL Core (物理実行、SparkContext 依存) を分離。Optimizer のテスタビリティと再利用性を確保。

## Dual-Mode Dispatch (PySpark)

公開 API → Classic 実装 (Py4J) or Connect 実装 (gRPC) を `is_remote()` で透過的に切り替え。

## Composable Feature Steps (K8s)

Pod spec 構築を小さな Feature Step に分解し、Profile に応じて合成する Builder パターン。個々の Feature Step クラスの一覧は [07-connectors-and-resource-managers.md](07-connectors-and-resource-managers.md#kubernetes) 参照。

## Two-Tier Memory Management

Execution / Storage メモリを soft boundary で管理。相互借用可能だが、Execution は Storage から回収されないという非対称性で、実行中のタスクの安定性を優先。具体的なメモリ計算式は [02-core.md](02-core.md#storage--キャッシュ) 参照。
