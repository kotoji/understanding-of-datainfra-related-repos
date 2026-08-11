# アーキテクチャ上の設計パターン

## Tree Transformation with Rules

Catalyst の全パイプライン (解析、最適化、物理計画) が同一の `Rule[TreeType]` / `RuleExecutor` パターンを使用。モジュラーで合成可能な変換を実現。

## Lazy Evaluation Chain

`QueryExecution` が `LazyTry` で各パイプラインステージを保持。各ステージは最大1回計算され、エラーは綺麗に伝搬。

## Extensibility Hooks

`SparkSessionExtensions` で以下を注入可能:
- Analyzer ルール
- Optimizer ルール
- Planner Strategy
- Columnar ルール
- AQE ルール

## Separation of Concerns

Catalyst (純粋論理、SparkContext 非依存) と SQL Core (物理実行、SparkContext 依存) を分離。Optimizer のテスタビリティと再利用性を確保。

## Dual-Mode Dispatch (PySpark)

公開 API → Classic 実装 (Py4J) or Connect 実装 (gRPC) を `is_remote()` で透過的に切り替え。

## Composable Feature Steps (K8s)

Pod spec 構築を小さな Feature Step に分解し、Profile に応じて合成する Builder パターン。

## Two-Tier Memory Management

Execution / Storage メモリを soft boundary で管理。相互借用可能だが、Execution は Storage から回収されないという非対称性で、実行中のタスクの安定性を優先。

## 付録: ファイルサイズ上位 (規模感の参考)

| ファイル | サイズ | 内容 |
|---------|--------|------|
| `catalyst/analysis/Analyzer.scala` | 219KB | 解析器 (60+ ルール) |
| `catalyst/optimizer/Optimizer.scala` | 136KB | 最適化器 (47 ルールファイル) |
| `project/SparkBuild.scala` | 96KB | SBT ビルド定義 |
| `sql/core/.../adaptive/AdaptiveSparkPlanExec.scala` | 44KB | AQE 実行エンジン |
| `core/SparkContext.scala` | ~100KB | SparkContext (3,617行) |
| `core/scheduler/DAGScheduler.scala` | ~130KB | DAG スケジューラ (4,464行) |
| `pom.xml` (root) | 138KB | Maven 設定 (3,755行) |
| `python/pyspark/pandas/frame.py` | ~400KB | Pandas API DataFrame (14,525行) |
