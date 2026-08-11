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

### なぜ毎回 `apply()` からやり直すのか

`apply()` は先頭で `refresh()` を呼び、最新のメタデータを読み直してから検証とマニフェスト生成をやり直す (`SnapshotProducer.apply()`)。「CAS だけリトライする」のでは不十分で、これは serializable isolation の要請から来ている。

CAS が失敗したということは、自分が読んだ後に他のコミットが入ったということ。そのコミットが自分の前提を壊している可能性がある — たとえば `OverwriteFiles` で「`WHERE ts < X` にマッチするファイルを消す」としたとき、競合コミットが同じ範囲に新しいファイルを追加していれば、そのファイルも消さなければ結果が変わる。**新しい状態の上で検証をやり直して初めて、直列化可能な結果になる**。

一方で、リトライしても以下は変わらないため再生成されない:

- **snapshot ID**: 初回に決めた値をキャッシュして使い回す
- **データファイル**: そもそも `SnapshotProducer` の外で書き終わっている
- **マニフェストファイル**: sequence number を `null` で書いてあるので内容が変わらない ([05](05-metadata-and-snapshots.md#継承の仕組み))

実際に書き直されるのは Manifest List だけ。

### `CommitStateUnknownException` を握りつぶさない理由

コミット後の後始末では、失敗時に書きかけのマニフェスト・マニフェストリストを削除する (`cleanAll()`)。ただし `CommitStateUnknownException` — 「コミットが成功したか失敗したか判断できない」状態 (HTTP タイムアウト等) — のときだけは、**クリーンアップせずにそのまま再スローする**:

```java
} catch (CommitStateUnknownException commitStateUnknownException) {
  throw commitStateUnknownException;          // クリーンアップしない
} catch (RuntimeException e) {
  if (!strictCleanup || e instanceof CleanableFailure) {
    Exceptions.suppressAndThrow(e, this::cleanAll);
  }
  throw e;
}
```

実際にはコミットが成功していた場合、新しいスナップショットがそれらのファイルを参照している。ここで消すと**参照先が存在しないテーブルが出来上がる = データ損失**になる。孤立ファイルが残るリスクを取ってでも消さない、という判断。残ったゴミは後から `DeleteOrphanFiles` で回収する。

なお、リトライ対象は `CommitFailedException` のみ (`onlyRetryOn`)。デフォルト 4 回、指数バックオフ (x2.0)。

## Transaction (`api/.../Transaction.java`)

**単一テーブル**に対する複数の操作を、1 回のアトミックコミットにまとめる仕組み (Javadoc: "A transaction for performing multiple updates to a table")。たとえば「スキーマ変更 + データ追加」を 1 スナップショットとして反映したいときに使う:

- `table()`: トランザクショナルビュー (未コミットの変更が見える `Table`)
- `updateSchema()` / `updateSpec()` / `updateProperties()` / `newAppend()` など、テーブルの mutation メソッドを一通り持つ
- `commitTransaction()`: 全保留変更を一括適用

`BaseTransaction` は 4 モード: `CREATE_TABLE`, `REPLACE_TABLE`, `CREATE_OR_REPLACE_TABLE`, `SIMPLE`。`TransactionTable` プロキシと `TransactionTableOperations` で変更をローカルに蓄積し、`commitTransaction()` で実際の `TableOperations` に一括 `commit()`。

> **複数テーブルにまたがるアトミックコミットではない**点に注意。`TableOperations.commit()` が 1 テーブルの CAS である以上、この API の原子性は 1 テーブルに閉じる。マルチテーブルコミットは REST Catalog の `/v1/{prefix}/transactions/commit` エンドポイントでのみ提供され、原子性の保証はサーバー実装側の責務になる ([06-catalog-and-rest.md](06-catalog-and-rest.md#openapi-仕様-open-apirest-catalog-open-apiyaml))。

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

判定の軸は data sequence number ([05-metadata-and-snapshots.md](05-metadata-and-snapshots.md#sequence-number-の役割)) とパーティション。**比較演算子が削除の種類によって違う**点に注意 (`format/spec.md` の "scope rules")。

| 削除の種類 | sequence number 条件 | その他の条件 |
|-----------|---------------------|-------------|
| **Deletion Vector** | データファイルの data seq **≦** DV の data seq | `referenced_data_file` がパス一致 + パーティション一致 |
| **Position delete** | データファイルの data seq **≦** 削除ファイルの data seq | `referenced_data_file` が非 null ならパス一致 + パーティション一致。かつ、そのデータファイルに適用される DV が存在しないこと |
| **Equality delete** | データファイルの data seq **<** 削除ファイルの data seq (厳密に小さい) | パーティション一致、**または**削除ファイルの spec が unpartitioned |

演算子が違う理由:

- **position delete / DV が `≦`** — 同一コミット内で追加した行を、同じコミットの中で削除できるようにするため。position delete は削除対象を「ファイルパス + 行位置」で名指ししているので、同時刻のファイルに適用しても曖昧さがない
- **equality delete が `<`** — こちらは「`id = 5` の行」のように値で指定するため、同一コミットで追加された行にまで適用すると「同時に書いた行が消える」ことになる。厳密に古いファイルに限定する必要がある

例外: **unpartitioned な spec で書かれた equality delete は global delete として扱われ**、全パーティションに適用される。それ以外の削除ファイルは他パーティションのデータには一切効かない。

### Deletion Vector の詳細 (v3)

- Roaring bitmap で削除行位置をエンコード
- Puffin ファイル内に `deletion-vector-v1` blob として格納 (Puffin フォーマット自体は [11-advanced-features.md](11-advanced-features.md#puffin-ファイル-統計) 参照)
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
