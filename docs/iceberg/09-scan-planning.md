# 式・フィルタシステムとスキャンプランニング

## Expression 階層

```
Expression (interface)
  +-- True, False (定数)
  +-- And, Or, Not (論理結合)
  +-- Predicate (Term + Operation)
       +-- UnboundPredicate (名前参照、スキーマ未解決)
       +-- BoundPredicate (ID 参照、スキーマ解決済)
            +-- BoundLiteralPredicate (単一リテラル比較)
            +-- BoundSetPredicate (IN / NOT_IN)
            +-- BoundUnaryPredicate (IS_NULL, NOT_NULL 等)
```

Operations: `LT`, `LT_EQ`, `GT`, `GT_EQ`, `EQ`, `NOT_EQ`, `IS_NULL`, `NOT_NULL`, `IS_NAN`, `NOT_NAN`, `IN`, `NOT_IN`, `STARTS_WITH`, `NOT_STARTS_WITH`, `AND`, `OR`, `NOT`, `TRUE`, `FALSE`, `COUNT`, `COUNT_NULL`, `COUNT_STAR`, `MAX`, `MIN`

## Unbound → Bound 解決

`Binder.bind(struct, expr)` が `UnboundPredicate` (カラム名参照) を `BoundPredicate` (フィールド ID + 型検証済) に変換。required フィールドへの `isNull` は `alwaysFalse` に簡約化など。

## 述語射影 (Predicate Projection)

`Projections` クラスが行レベル式をパーティションレベル式に変換:

- **Inclusive projection** (`Projections.inclusive(spec)`): 行が元の述語にマッチするなら、射影述語がそのパーティションにマッチ。ファイル選択 (scan planning) に使用
- **Strict projection** (`Projections.strict(spec)`): 射影述語がパーティションにマッチするなら、そのパーティションの全行が元の述語にマッチ。残余計算に使用

## エンドツーエンド スキャンプランニング

```
1. スナップショットロード
   テーブルメタデータ読込 → 現在のスナップショット (またはブランチ/タグ/タイムトラベル)

2. マニフェストリスト読込
   スナップショットの manifest-list から全マニフェスト取得

3. マニフェストフィルタリング (ManifestEvaluator)
   パーティションサマリー統計でマニフェスト単位スキップ

4. データファイルフィルタリング (InclusiveMetricsEvaluator)
   ファイルごとのカラムメトリクス (min/max/null/NaN) でスキップ

5. 削除ファイルマッチング
   シーケンス番号、パーティションマッチング、参照データファイルで適用対象決定

6. 残余計算 (ResidualEvaluator)
   パーティション値で部分評価後の残余行レベル述語を計算

7. 行読込・フィルタリング (Evaluator)
   残余述語を個々の行に適用
```

## 各レベルの評価器

| 評価器 | レベル | 入力 | 判定 |
|--------|--------|------|------|
| `ManifestEvaluator` | マニフェスト | パーティションサマリー | マッチする行を含むかもしれない → true |
| `InclusiveMetricsEvaluator` | ファイル | カラム統計 | マッチする行を含むかもしれない → true |
| `StrictMetricsEvaluator` | ファイル | カラム統計 | 全行がマッチする → true |
| `ResidualEvaluator` | パーティション | パーティション値 | 残余述語を返す |
| `Evaluator` | 行 | `StructLike` | 行がマッチ → true |

## Scan インターフェース

`Scan<ThisT, T, G>` (不変、スレッドセーフ、refinement メソッドは新インスタンスを返す):
- `project(schema)` / `select(columns)`: カラム射影
- `filter(expression)`: 行フィルタリング
- `planFiles()`: `CloseableIterable<T>` (ファイル単位タスク)
- `planTasks()`: `CloseableIterable<G>` (バランスされたタスクグループ)
- `targetSplitSize()` / `splitLookback()` / `splitOpenFileCost()`: チューニング

`TableScan` は `useSnapshot(id)` / `useRef(name)` / `asOfTime(ts)` でタイムトラベル対応。
