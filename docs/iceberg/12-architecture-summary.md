# アーキテクチャ総括

全体のレイヤ構成図は [00-overview.md](00-overview.md#アーキテクチャ概観) 参照。ここでは、そのレイヤ構成を成立させている設計判断を整理する。

## コア設計原則

### 1. 関心の分離

Iceberg の各レイヤは、他のレイヤの実装を知らずに差し替えられる:

| 関心事 | 担当 | 詳細 |
|--------|------|------|
| メタデータをどこで追跡するか | `Catalog` | [06](06-catalog-and-rest.md) |
| メタデータをどうシリアライズするか | `TableMetadataParser` | [05](05-metadata-and-snapshots.md) |
| ファイルをどこに格納するか | `FileIO` | [07](07-fileio-and-formats.md) |
| ファイルをどう符号化するか | Parquet / ORC / Arrow | [07](07-fileio-and-formats.md) |

たとえば「Glue Catalog + S3FileIO + Parquet」と「REST Catalog + GCSFileIO + ORC」は、どちらも同じ `core` の実装をそのまま使う。

### 2. 不変性

`TableMetadata`, `Snapshot`, `Schema`, `PartitionSpec`, `SortOrder` はすべて不変オブジェクト。変更は必ず新インスタンスの生成として表現される (`TableMetadata.buildFrom(base)`)。

これが効いてくるのが後述の OCC で、「読んだ時点の状態」を値として保持したまま、それを前提条件としてコミットできる。

### 3. 楽観的並行制御 (OCC)

テーブルの更新は「メタデータポインタの compare-and-swap」1 点に集約される ([08](08-commit-and-write-path.md#楽観的並行制御-occ))。ロックを取らないため、書き込みの大部分 (データファイル・マニフェストの生成) は並行実行でき、競合が起きた場合だけリトライする。

### 4. フィールド ID ベースのスキーマ追跡

スキーマ進化をカラム名ではなく一意な ID で追跡する ([03](03-type-system-and-schema.md#カラム-id-の設計))。ID は再利用されないため、リネーム・並べ替え・削除後もデータファイルを正しく読める。

### 5. マニフェスト再利用

変更のないマニフェストはスナップショット間で共有され、書き直されない ([05](05-metadata-and-snapshots.md#manifestfile-第3層))。1 回のコミットで書くのはマニフェストリストと差分マニフェストだけなので、テーブル全体のサイズによらずコミットコストが一定に保たれる。

### 6. 隠れたパーティショニング (hidden partitioning)

ユーザーはパーティション式を意識せずに `WHERE` を書き、エンジンが述語をパーティション空間へ自動射影する ([04](04-partitioning-and-sorting.md#hive-パーティショニングとの決定的な違い), [09](09-scan-planning.md#述語射影-predicate-projection))。パーティション構成を後から変更しても、既存のクエリは書き換え不要。

### 7. プラグイン方式

Catalog, FileIO, ファイルフォーマット、認証 (`AuthManager`) がすべて差し替え可能。設定文字列からクラスを解決する共通パターン (`CatalogUtil.buildIcebergCatalog` など) を採る。

## これらがどう噛み合っているか

Iceberg の中心的な主張である「**巨大テーブルに対する O(1) スキャンプランニングと serializable isolation の両立**」は、上記が組み合わさって成立している:

```
不変性 (2)
  └→ 「読んだ状態」を前提条件として保持できる
        └→ OCC (3) が compare-and-swap 1 回で済む
              └→ ロック不要 = メタストアが中央ボトルネックにならない

マニフェスト再利用 (5)
  └→ コミットで書く量がテーブルサイズに依存しない
        └→ マニフェストの partition summary が安定して残る
              └→ スキャン時にマニフェスト単位で枝刈りできる (O(1) プランニング)

フィールド ID (4) + 隠れたパーティショニング (6)
  └→ スキーマ・パーティションを変えても過去データと過去クエリが生き続ける
```

逆に言うと、Row Lineage ([11](11-advanced-features.md#row-lineage-v3)) の継承設計のような後発機能も、「OCC のリトライでファイルを書き直さない」という制約を最優先に設計されている。この制約が Iceberg の設計判断を貫く軸になっている。
