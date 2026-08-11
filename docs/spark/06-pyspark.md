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
| Pandas UDF (Scalar) | `SCALAR` / `SCALAR_ITER` | pandas Series | ベクトル化処理 |
| Pandas UDF (Grouped) | `GROUPED_MAP` / `GROUPED_AGG` | pandas DataFrame | グループ処理 |
| Arrow UDF (4.1.0~) | `SCALAR` / `GROUPED_AGG` 等 | pyarrow Array | 最新の高速処理 |
| UDTF | (各種) | pickle/Arrow | テーブル関数 |

型ヒントから自動的に eval type を推論する機能あり。

## Pandas API on Spark

`pyspark.pandas` パッケージ (旧 Koalas):
- `InternalFrame` が中核。Spark DataFrame をラップし index メタデータを保持
- `frame.py` (14,525行) が pandas DataFrame API をミラー
- `data_type_ops/` で型ごとの演算子ディスパッチ

## Arrow 統合

PyArrow >= 18.0.0 が必須。全 UDF タイプ、Connect データ転送、DataFrame-to-pandas 変換で Arrow を使用する方向に統一が進んでいる。
