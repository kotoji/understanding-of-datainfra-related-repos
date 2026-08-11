# コミットプロトコルと Write パス

## 楽観的並行制御 (OCC)

`SnapshotProducer.commit()` が実装する中核プロトコル:

```
1. Tasks.foreach(ops) でリトライループ開始
   - デフォルト: 4 リトライ、指数バックオフ (x2.0)
   - CommitFailedException のみリトライ

2. 各試行で apply() を呼ぶ:
   a. 現在のメタデータを再読込
   b. バリデーション実行
   c. サブクラスの apply(metadata, snapshot) で新マニフェスト生成
   d. マニフェストリストファイル書き込み

3. 新 BaseSnapshot 構築 (sequence number, parent ID, summary, manifest list)

4. TableMetadata.buildFrom(base) で新メタデータ作成

5. ops.commit(base, updated) — アトミック CAS

6. 成功時: 未コミットマニフェスト・リストをクリーンアップ
   不明時 (CommitStateUnknownException): クリーンアップせず再スロー

7. CreateSnapshotEvent を発火
```

## Transaction (`api/.../Transaction.java`)

複数テーブル操作を単一アトミックコミットにバッチ化:
- `table()`: トランザクショナルビュー
- 全 mutation メソッド利用可能
- `commitTransaction()`: 全保留変更を一括適用

`BaseTransaction` は 4 モード: `CREATE_TABLE`, `REPLACE_TABLE`, `CREATE_OR_REPLACE_TABLE`, `SIMPLE`。`TransactionTable` プロキシと `TransactionTableOperations` で変更をローカルに蓄積し、`commitTransaction()` で実際の `TableOperations` に一括 `commit()`。

## 競合検出

`MergingSnapshotProducer` とサブクラスが提供:
- `validateFromSnapshot(snapshotId)`: 競合検出の開始スナップショットを設定
- `validateNoConflictingData()`: 競合パーティションに新データファイルがないか検証
- `validateNoConflictingDeletes()`: 新削除ファイルがないか検証
- `conflictDetectionFilter(Expression)`: 競合検出スコープの絞り込み

## Write パス: クラス階層

```
SnapshotProducer<ThisT> (abstract)
  |
  +-- FastAppend (AppendFiles)
  |     マニフェストマージなし。新マニフェストを直接追加。コミット高速だが読取が遅くなりうる
  |
  +-- BaseRewriteManifests (RewriteManifests)
  |     マニフェスト構造の最適化
  |
  +-- MergingSnapshotProducer<ThisT> (abstract)
        既存マニフェストと新データ/削除ファイルのマージを処理
        |
        +-- MergeAppend (AppendFiles)
        |     新ファイルを既存マニフェストにマージ。マニフェスト数を最適化
        |
        +-- BaseOverwriteFiles (OverwriteFiles)
        |     式にマッチするファイルを削除 + 新ファイル追加
        |
        +-- BaseReplacePartitions (ReplacePartitions)
        |     パーティション全体をアトミックに置換 (dynamic overwrite)
        |
        +-- BaseRowDelta (RowDelta)
        |     行レベル操作: データファイル + 削除ファイルの追加・削除
        |
        +-- BaseRewriteFiles (RewriteFiles)
        |     コンパクション: 古いファイル削除 + 新ファイル追加
        |
        +-- StreamingDelete (DeleteFiles)
              パスまたは式によるデータファイル削除
```

## 各操作の API

| API | メソッド | 操作種別 |
|-----|---------|---------|
| `AppendFiles` | `appendFile(DataFile)`, `appendManifest(ManifestFile)` | APPEND |
| `OverwriteFiles` | `overwriteByRowFilter(Expression)`, `addFile()`, `deleteFile()` | OVERWRITE/DELETE/APPEND |
| `ReplacePartitions` | `addFile(DataFile)` | OVERWRITE |
| `RowDelta` | `addRows(DataFile)`, `addDeletes(DeleteFile)`, `removeRows()`, `removeDeletes()` | OVERWRITE/DELETE/APPEND |
| `RewriteFiles` | `deleteFile()`, `addFile()` | REPLACE |
| `DeleteFiles` | `deleteFile(path)`, `deleteFromRowFilter(Expression)` | DELETE |
| `RewriteManifests` | `clusterBy(Function)`, `rewriteIf(Predicate)` | REPLACE |

## 削除メカニズム

### 3種類の削除

| 種類 | 導入 | 形式 | 仕組み |
|------|------|------|--------|
| **Position Delete** | v2 | Avro/Parquet: `(file_path, pos)` ペア | ファイル内の特定行位置を指定して削除 |
| **Equality Delete** | v2 | Avro/Parquet: equality カラム値 | 指定カラム値と一致する全行を削除 |
| **Deletion Vector (DV)** | v3 | Puffin: Roaring bitmap | ファイル単位の bitmap で削除行位置を記録 |

### 削除の適用ルール

- **シーケンス番号ベース**: 削除ファイルは、自身より**厳密に低い** data sequence number を持つデータファイルにのみ適用
- **Position delete / DV**: `referenced_data_file` で対象データファイルを特定
- **Equality delete**: パーティションマッチング + equality ID による行レベルマッチング

### Deletion Vector の詳細 (v3)

- Roaring bitmap で削除行位置をエンコード
- Puffin ファイル内に `deletion-vector-v1` blob として格納
- スナップショットごとに各データファイルに対し最大1つの DV
- Writer は既存 DV と新しい position delete をマージ必須
- v3 では position delete は deprecated

## コンパクション (Actions)

`api/.../actions/` に定義:
- `RewriteDataFiles`: メインコンパクション。サイズベースの戦略、部分進行、ファイルサイズ設定可能
- `RewritePositionDeleteFiles`: position delete ファイルのコンパクション
- `RewriteManifests`: マニフェストレイアウト最適化
- `ConvertEqualityDeleteFiles`: equality delete → position delete 変換
- `RemoveDanglingDeleteFiles`: 参照先のないデータファイルに紐づく削除ファイルの除去
- `DeleteOrphanFiles`: 孤立ファイルのクリーンアップ
- `ExpireSnapshots`: 古いスナップショットの削除
