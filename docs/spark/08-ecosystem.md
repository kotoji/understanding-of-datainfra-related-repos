# Ecosystem: MLlib, GraphX, Common

## MLlib

**2つの並行 API:**

| API | パッケージ | ベース | 状態 |
|-----|-----------|--------|------|
| 新 API | `org.apache.spark.ml` | DataFrame | 推奨 |
| 旧 API | `org.apache.spark.mllib` | RDD | メンテナンスのみ |

**新 API の主要カテゴリ:**
- Classification: LogisticRegression, RandomForest, GBT, MLP, NaiveBayes, LinearSVC
- Regression: LinearRegression, RandomForest, GBT, GLM
- Clustering: KMeans, GaussianMixture, LDA, BisectingKMeans
- Recommendation: ALS
- Feature: Tokenizer, TF-IDF, Word2Vec, VectorAssembler, Scaler, PCA, Bucketizer
- Evaluation: Binary/Multiclass/Regression/Clustering/Ranking Evaluator
- Tuning: CrossValidator, TrainValidationSplit

`mllib-local` — Spark 非依存のローカル線形代数 (BLAS, Vector, Matrix)

## GraphX

`Graph[VD, ED]` — 頂点 (VD) とエッジ (ED) に型パラメータを持つ不変グラフ。

- `VertexRDD[VD]` / `EdgeRDD[ED]` / `EdgeTriplet[VD, ED]`
- **Pregel API** — Bulk-synchronous メッセージパッシング
- 組込みアルゴリズム: PageRank, ConnectedComponents, TriangleCount, ShortestPaths, LabelPropagation, SVD++
- Partition Strategy: RandomVertexCut, EdgePartition1D/2D

## Common モジュール

| モジュール | 用途 |
|-----------|------|
| `variant` | VARIANT 型 (semi-structured data) |
| `unsafe` | `UnsafeRow`, off-heap メモリ管理 |
| `sketch` | Count-Min Sketch 等の確率的データ構造 |
| `network-common` | Netty ベースの RPC/transport |
| `kvstore` | LevelDB ベースの KV ストア (Spark UI 用) |
