# 高度な機能

v3 以降で追加された機能を扱う。全体のアーキテクチャ総括は [12-architecture-summary.md](12-architecture-summary.md) 参照。

## Row Lineage (v3)

2つのメタデータカラム:
- `_row_id` (field id 2147483540): 行ごとの一意 long 識別子
- `_last_updated_sequence_number` (field id 2147483539): 最後に行を更新したコミットの sequence number

### 割り当てメカニズム (多段継承)

```
Table metadata → next-row-id
  └── Snapshot → first-row-id = table's next-row-id
        └── Manifest List → first_row_id (累積 added_rows_count + existing_rows_count)
              └── Manifest → DataFile → first_row_id (累積 record_count)
                    └── Row → _row_id = data_file.first_row_id + row_pos
```

### なぜ継承にするのか

`_row_id` は「テーブル全体で一意な通し番号」なので、素直に実装するなら書き込み時に確定値を各行に埋め込むことになる。しかしそれは OCC と両立しない。

`next-row-id` はテーブルメタデータ上の値なので、コミットが成功するまで**自分が何番から始められるかが分からない**。競合コミットが先に入れば開始位置はずれる。確定値を埋めていたら、リトライのたびにデータファイルを全部書き直すことになってしまう。

そこで実ファイルには何も書かず、`first_row_id` をメタデータ階層に沿って伝播させ、読み取り時に

```
_row_id = data_file.first_row_id + 行のファイル内位置 (row_pos)
```

として算出する。こうすると:

| 成果物 | リトライ時 |
|--------|-----------|
| データファイル | 書き直し不要 (`_row_id` を含まない) |
| マニフェストファイル | 書き直し不要 (`first_row_id` は継承で埋まる) |
| Manifest List | 書き直す (どのみち毎回書き直す) |

これは sequence number の継承 ([05-metadata-and-snapshots.md](05-metadata-and-snapshots.md#継承の仕組み)) とまったく同じ発想で、「**コミットのリトライコストを Manifest List 1 個に抑える**」という Iceberg 共通の設計制約から導かれている。

`_last_updated_sequence_number` も同様に、行を変更したときは `null` を書いておき、読み取り時にマニフェストエントリの sequence number で埋める。

### 制約

Equality delete で更新された行は追跡されない (完全な削除 + 新行挿入として扱う)。理由は spec が明示している通り、**equality delete を使うエンジンは書き込み前に既存データを読まない**ことを前提にしているため — 元の行を読んでいない以上、新しい行に引き継ぐべき `_row_id` を知りようがない。equality delete のそもそもの利点 (書き込み前の read を回避できる) と row lineage が原理的に両立しない、ということ。

---

## Puffin ファイル (統計)

仕様: `format/puffin-spec.md`

統計 blob を格納するバイナリフォーマット:
```
Magic (0x50 0x46 0x41 0x31) | Blob1 | ... | BlobN | Footer
```

サポート blob 種別:
- `apache-datasketches-theta-v1`: NDV (Number of Distinct Values) 推定用 Theta スケッチ
- `deletion-vector-v1`: 削除行位置の Roaring bitmap

圧縮: LZ4, Zstd, 非圧縮

テーブルメタデータの `statistics` フィールドで Puffin ファイルを参照。`partition-statistics` でパーティションごとの集計 (レコード数、ファイル数、削除数) を追跡。

---

## View サポート

仕様: `format/view-spec.md`

### テーブルとの違い

| 観点 | Table | View |
|------|-------|------|
| データファイル | あり | なし (SQL 定義のみ) |
| バージョニング | Snapshot | ViewVersion (SQL 定義の版) |
| マニフェスト | あり | なし |
| 操作 | データ操作多数 | `replaceVersion()`, `updateProperties()` のみ |
| スキーマ | 入力スキーマ | 出力スキーマ (クエリ結果の型) |
| フォーマットバージョン | v1-v4 | v1 |

### 主要クラス

- `View` (interface): `currentVersion()`, `versions()`, `history()`, `schema()`
- `ViewVersion`: SQL 表現を持つバージョン (複数 dialect 対応可能)
- `ViewCatalog`: `listViews()`, `loadView()`, `buildView()`, `dropView()`
- `ViewMetadata`: UUID, schemas, versions, history, properties (Immutables ベース)

Spark 連携: `SparkView` でラップ、`SparkCatalog` が Spark の `ViewCatalog` を実装
