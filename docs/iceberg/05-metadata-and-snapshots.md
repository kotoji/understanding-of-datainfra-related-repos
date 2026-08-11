# メタデータ構造とスナップショット

## メタデータ階層

```
Table Metadata (JSON ファイル)
  └── Snapshot (manifest-list パスを持つ)
        └── Manifest List (Avro ファイル: ManifestFile エントリのリスト)
              └── Manifest (Avro ファイル: DataFile/DeleteFile エントリのリスト)
                    └── DataFile / DeleteFile (実データファイルへのポインタ + メトリクス)
```

## TableMetadata (`core/.../TableMetadata.java`)

テーブルの全状態を表す**不変**オブジェクト:

| フィールド | 説明 |
|-----------|------|
| `formatVersion` | 1-4 (デフォルト: 2、サポート最大: 4) |
| `uuid` | テーブル UUID |
| `location` | ベースロケーション |
| `lastSequenceNumber` | 単調増加シーケンス番号 |
| `schemas` / `currentSchemaId` | スキーマ進化履歴 |
| `specs` / `defaultSpecId` | パーティション spec 進化履歴 |
| `sortOrders` / `defaultSortOrderId` | ソートオーダー |
| `properties` | テーブルプロパティ (Map) |
| `currentSnapshotId` | 現在のスナップショット |
| `snapshots` | 全保持スナップショット (遅延ロード) |
| `refs` | ブランチ・タグの Map |
| `snapshotLog` | スナップショット変更履歴 |
| `previousFiles` | メタデータログ (過去のメタデータファイル位置) |
| `nextRowId` | Row Lineage 用 (v3+) |

メタデータファイルは JSON でシリアライズ (`TableMetadataParser`)。ファイル名パターン: `v{N}.metadata.json` (UUID 付きの場合もあり)。GZIP 圧縮対応。

## Snapshot (`api/.../Snapshot.java`)

テーブルデータのポイントインタイムビュー:

| フィールド | 説明 |
|-----------|------|
| `snapshotId` | 一意識別子 |
| `parentId` | 親スナップショット (チェーン形成) |
| `sequenceNumber` | 単調増加、コミット時付与 |
| `timestampMillis` | コミットタイムスタンプ |
| `operation` | APPEND / REPLACE / OVERWRITE / DELETE |
| `summary` | key-value メトリクス (追加/削除レコード数等) |
| `manifestListLocation` | マニフェストリストファイルへのパス |
| `firstRowId` / `addedRows` | Row Lineage (v3+) |

## ManifestFile (`api/.../ManifestFile.java`)

マニフェストリスト内のエントリ。マニフェストファイルのメタデータ:

| フィールド | 説明 |
|-----------|------|
| `path` / `length` | マニフェストファイルの位置とサイズ |
| `content` | DATA (0) or DELETES (1) |
| `partitionSpecId` | 使用パーティション spec |
| `sequenceNumber` / `minSequenceNumber` | シーケンス番号追跡 |
| `addedFilesCount` / `existingFilesCount` / `deletedFilesCount` | ファイル数 |
| `partitions` | `PartitionFieldSummary` のリスト (containsNull, lowerBound, upperBound) |
| `firstRowId` | Row Lineage (v3+) |

マニフェストはスナップショット間で**再利用**される。変更のないマニフェストは書き直さない。

## DataFile / DeleteFile (`api/.../ContentFile.java`)

マニフェスト内のエントリ。データファイル or 削除ファイルへのポインタ + 統計情報:

| フィールド | 説明 |
|-----------|------|
| `location` / `format` | ファイルパスとフォーマット |
| `partition` | パーティション値 |
| `recordCount` / `fileSizeInBytes` | 基本統計 |
| `columnSizes` | カラムごとのディスクサイズ (Map<colId, Long>) |
| `valueCounts` / `nullValueCounts` / `nanValueCounts` | 値カウント |
| `lowerBounds` / `upperBounds` | カラムごとの上下限 (バイナリシリアライズ) |
| `splitOffsets` | 推奨分割ポイント |
| `dataSequenceNumber` / `fileSequenceNumber` | シーケンス番号 |
| `firstRowId` | Row Lineage (v3+) |

## ブランチ・タグ (Snapshot References)

`api/.../SnapshotRef.java`, `api/.../ManageSnapshots.java`

テーブルメタデータの `refs` マップに格納:

### タグ (Tag)
- スナップショットへの**不変**ラベル
- `max-ref-age-ms`: 自動有効期限

### ブランチ (Branch)
- 新スナップショットのコミットで更新される**可変**参照
- `min-snapshots-to-keep`: 有効期限処理時の最小保持スナップショット数
- `max-snapshot-age-ms`: スナップショットの最大年齢
- `main` ブランチは常に存在し、`current-snapshot-id` を指す

### ManageSnapshots API

| メソッド | 説明 |
|---------|------|
| `createBranch(name)` / `createTag(name, snapshotId)` | ブランチ/タグ作成 |
| `removeBranch(name)` / `removeTag(name)` | 削除 |
| `renameBranch(name, newName)` | リネーム |
| `replaceBranch(name, snapshotId)` | 付け替え |
| `cherrypick(snapshotId)` | チェリーピック |
| `rollbackTo(snapshotId)` | ロールバック |

### スナップショット保持ポリシー

1. 保持セットを空で開始
2. `max-ref-age-ms` を超えた ref (main 除く) を削除
3. 各ブランチ/タグの参照スナップショットを保持セットに追加
4. 各ブランチについて、`max-snapshot-age-ms` AND `min-snapshots-to-keep` 両方を満たすまで祖先を追加
5. 保持セットにないスナップショットを期限切れ
