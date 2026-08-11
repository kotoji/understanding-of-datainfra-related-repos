# PySpark

## デュアルモードアーキテクチャ

PySpark は **dispatch パターン** で Classic (Py4J) と Connect (gRPC) を透過的に切り替える:

```
公開 API (DataFrame, Column 等)
  │  dispatch_df_method / dispatch_col_method
  ├── Classic: pyspark.sql.classic.*  (Py4J → JVM)
  └── Connect: pyspark.sql.connect.* (gRPC → protobuf)
```

- Facade (`sql/dataframe.py`, 7,263行) — docstring + method stub
- Classic 実装 (`sql/classic/dataframe.py`, 2,164行)
- Connect 実装 (`sql/connect/dataframe.py`, 2,562行)

## UDF システム

| UDF 種類 | eval type | データ転送 | 用途 |
|----------|-----------|-----------|------|
| Regular Python UDF | `SQL_BATCHED_UDF` | pickle | 行単位処理 |
| Arrow Python UDF | `SQL_ARROW_BATCHED_UDF` | Arrow | 高速行単位処理 |
| Pandas UDF (Scalar) | `SQL_SCALAR_PANDAS_UDF` / `SQL_SCALAR_PANDAS_ITER_UDF` | pandas Series | ベクトル化処理 |
| Pandas UDF (Grouped) | `SQL_GROUPED_MAP_PANDAS_UDF` / `SQL_GROUPED_AGG_PANDAS_UDF` | pandas DataFrame | グループ処理 |
| Arrow UDF (4.1.0~) | `SQL_SCALAR_ARROW_UDF` / `SQL_SCALAR_ARROW_ITER_UDF` / `SQL_GROUPED_AGG_ARROW_UDF` / `SQL_GROUPED_AGG_ARROW_ITER_UDF` / `SQL_WINDOW_AGG_ARROW_UDF` | pyarrow Array | 最新の高速処理 |
| UDTF | `SQL_TABLE_UDF` / `SQL_ARROW_TABLE_UDF` / `SQL_ARROW_UDTF` | pickle/Arrow | テーブル関数 |

eval type の定数は `pyspark/util.py` の `PythonEvalType` に定義されている (上表以外にも cogrouped map、`transformWithState` 系など多数)。ユーザ向けデコレータの `PandasUDFType.SCALAR` 等はこれとは別の enum で、内部で `PythonEvalType` にマップされる。

型ヒントから自動的に eval type を推論する機能あり。

## Pandas API on Spark

`pyspark.pandas` パッケージ (旧 Koalas):
- `InternalFrame` が中核。Spark DataFrame をラップし index メタデータを保持
- `frame.py` (14,525行) が pandas DataFrame API をミラー
- `data_type_ops/` で型ごとの演算子ディスパッチ

## Arrow 統合

PyArrow >= 18.0.0 が必須。全 UDF タイプ、Connect データ転送、DataFrame-to-pandas 変換で Arrow を使用する方向に統一が進んでいる。
