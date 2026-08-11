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

`Projections` クラスが**行レベルの述語**を**パーティションレベルの述語**に変換する。これが hidden partitioning ([04-partitioning-and-sorting.md](04-partitioning-and-sorting.md#hive-パーティショニングとの決定的な違い)) の実装本体で、ユーザーが `ts` について書いた条件を、エンジンが `day(ts)` の条件に翻訳する部分にあたる。

射影には 2 種類あり、**用途が正反対**:

- **Inclusive projection** (`Projections.inclusive(spec)`): 「行が元の述語にマッチするなら、射影述語もそのパーティションにマッチする」。対偶を取ると「射影述語にマッチしないパーティションには、マッチする行が絶対にない」→ **安全に読み飛ばせるファイルを判定する**のに使う
- **Strict projection** (`Projections.strict(spec)`): 「射影述語がパーティションにマッチするなら、そのパーティションの全行が元の述語にマッチする」→ **行レベルのフィルタを省略できるファイルを判定する**のに使う

### 具体例

パーティション仕様が `day(ts)`、クエリが以下だとする:

```sql
WHERE ts > timestamp '2023-01-15 10:00:00'
```

`day` transform で 2023-01-15 は `19372` (1970-01-01 からの日数)。2 つの射影はこうなる:

| | 射影結果 | 意味 |
|---|---------|------|
| Inclusive | `day(ts) >= 19372` | 19372 の日にも 10:00 以降の行がありうる → **この日のファイルも読む必要がある** |
| Strict | `day(ts) > 19372` | 19373 以降の日なら全行が 10:00 より後 → **行フィルタ不要** |

境界の日 (19372) の扱いが両者で分かれるのがポイント。結果として、各ファイルの扱いは:

| パーティション | 読むか (inclusive) | 残余述語 (strict の結果) |
|--------------|-------------------|------------------------|
| `day = 19371` | 読まない | - |
| `day = 19372` | 読む | `ts > '2023-01-15 10:00:00'` が残る |
| `day = 19373` | 読む | `true` (全行マッチ確定、行フィルタをスキップ) |

つまり inclusive が「**どのファイルを開くか**」、strict が「**開いたファイルの中で行フィルタが要るか**」を決める。後者が `ResidualEvaluator` による残余計算 (下記ステップ 6) の中身。

> 未知の Transform に対しては inclusive projection は常に `true` を返す (= 読み飛ばさない)。安全側に倒す設計。

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
