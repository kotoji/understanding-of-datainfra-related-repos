# テーブルフォーマット仕様 (v1-v4)

仕様ファイル: `format/spec.md` (2,142 行)

## v1: Analytic Data Tables

- 不変ファイルフォーマット (Parquet, Avro, ORC) を用いた大規模分析テーブル管理
- スナップショット、マニフェスト、パーティション仕様の基本構造を定義
- テーブル状態はメタデータファイルで管理、アトミックにスワップ (楽観的並行制御)

## v2: Row-level Deletes

- **削除ファイル** (position deletes, equality deletes) による行レベルの更新/削除 → [08-commit-and-write-path.md](08-commit-and-write-path.md#削除メカニズム)
- `sequence-number` の導入 (操作の論理的順序付け) → [05-metadata-and-snapshots.md](05-metadata-and-snapshots.md#sequence-number-の役割)
- マニフェストリストの `content` フィールド (0=data, 1=deletes)
- `data_file` に `content` フィールド追加
- `manifest-list` が snapshot で必須に (インライン `manifests` を置換)
- マニフェストエントリの `snapshot_id` がオプショナルに (継承サポート)

## v3: Extended Types and Capabilities

- **新データ型**: `timestamp_ns`, `timestamptz_ns`, `unknown`, `variant`, `geometry(C)`, `geography(C, A)` → [03-type-system-and-schema.md](03-type-system-and-schema.md#型階層)
- **カラムデフォルト値**: `initial-default` (追加時設定、不変), `write-default` (追加時設定、変更可能)
- **Row Lineage**: `_row_id` (行の一意識別子)、`_last_updated_sequence_number` → [11-advanced-features.md](11-advanced-features.md#row-lineage-v3)
- **Deletion Vectors (DV)**: Puffin ファイル内の Roaring bitmap。Position delete ファイルを置き換え → [08](08-commit-and-write-path.md#deletion-vector-の詳細-v3), [11](11-advanced-features.md#puffin-ファイル-統計)
- **テーブル暗号化キー** のメタデータ追跡
- **マルチ引数 Transform**: `source-ids` フィールド (パーティション/ソート用)
- Position delete ファイルは deprecated

## v4: Metadata Structure (開発中)

- メタデータ内の **相対パス** (テーブル再配置が容易に)
- `location` がオプショナルに (Catalog が提供)
- **Content Stats**: メトリクスマップを型付き・ID ベースのフィールドレベル統計 (`content_stats` struct) に置換
- パス解決ルール: 相対パスはテーブルロケーションと `/` で結合

## 関連仕様ファイル

| ファイル | 内容 |
|---------|------|
| `format/spec.md` | メインテーブル仕様 (2,142 行) |
| `format/view-spec.md` | View メタデータ仕様 |
| `format/puffin-spec.md` | Puffin 統計ファイルフォーマット |
| `format/expressions-spec.md` | 式の形式仕様 (CHECK 制約、サーバー側スキャン) |
| `format/udf-spec.md` | UDF/UDTF メタデータ仕様 |
| `format/gcm-stream-spec.md` | AES-GCM ストリーム暗号化仕様 |
| `format/mumbling-spec.md` | Mumbling bitmap (DV 用圧縮ビットマップ) |
