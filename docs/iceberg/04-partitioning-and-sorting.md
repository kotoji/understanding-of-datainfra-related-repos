# パーティショニングとソートオーダー

## PartitionSpec (`api/.../PartitionSpec.java`)

パーティション仕様 = `PartitionField` のリスト:
- `sourceId`: テーブルスキーマ内のカラム ID
- `fieldId`: パーティションフィールド ID (1000 から開始、一意)
- `name`: パーティションフィールド名
- `transform`: 変換関数

Builder: `PartitionSpec.builderFor(schema)` で fluent 構築。

## Transform 一覧

| Transform | 説明 | 出力例 |
|-----------|------|--------|
| `identity` | ソース値そのまま | `"2023-01-15"` |
| `bucket[N]` | 32-bit Murmur3 hash mod N | `7` |
| `truncate[W]` | 幅 W で切り捨て | `"abc"` → `"ab"` (W=2) |
| `year` | 年抽出 | `53` (2023) |
| `month` | 月抽出 | `636` (2023-01) |
| `day` | 日抽出 | `19372` (2023-01-15) |
| `hour` | 時間抽出 | `464928` |
| `void` | 常に null (フィールド無効化用) | `null` |

`Transform<S,T>` インターフェースの主要メソッド:
- `bind(Type)`: 型固有の `SerializableFunction` を返す
- `preservesOrder()`: 単調変換かどうか (述語射影に重要)
- `satisfiesOrderOf(other)`: ソート互換性 (例: day は month を満たす)
- `project(name, predicate)` / `projectStrict(name, predicate)`: 述語のパーティション空間への射影

## パーティション進化

- フィールドの追加・削除・リネーム → 新しい `spec-id` で新 spec 作成
- **古い spec と新しい spec は共存**: 各マニフェストが `partition-spec-id` を記録
- 古いファイルは古いマニフェスト (古い spec) に、新しいファイルは新しいマニフェスト (新しい spec) に
- スキャンプランニング時、各マニフェストの spec を使って独立に述語変換
- 未知の Transform に対する inclusive projection は常に `true` (フィールドを無視)

**Hive パーティショニングとの決定的な違い**: Iceberg ではパーティショニングはテーブル設定であり、ディレクトリ構造ではない。`WHERE ts > '2023-01-01'` と書けば、エンジンが自動的に `day(ts)` パーティションにフィルタを射影する。ユーザーが `WHERE day = '2023-01-01'` と書く必要はない (hidden partitioning)。

## ソートオーダー (`api/.../SortOrder.java`)

`orderId` + `SortField` のリスト。Order ID 0 は unsorted 予約。

各 `SortField`:
- `source-id`: カラム ID
- `transform`: パーティションと同じ変換関数
- `direction`: `asc` / `desc`
- `null-order`: `nulls-first` / `nulls-last`

浮動小数点ソート順: `-NaN < -Infinity < -value < -0 < 0 < value < Infinity < NaN`

各データ/削除ファイルがマニフェスト内で `sort_order_id` を参照可能。Writer はデフォルトソートオーダーを使うべきだが強制ではない。`SortOrder.satisfies(other)` で互換性チェック。
