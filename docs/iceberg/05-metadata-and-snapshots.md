# メタデータ構造とスナップショット

## メタデータ階層

```
[第1層] Table Metadata (JSON ファイル)
          └── [第2層] Snapshot (manifest-list パスを持つ)
                └── [第3層] Manifest List (Avro: ManifestFile エントリのリスト)
                      └── [第4層] Manifest (Avro: DataFile/DeleteFile エントリのリスト)
                            └── [第5層] DataFile / DeleteFile
                                        (実データファイルへのポインタ + メトリクス)
```

以下、各層を上から順に見ていく。

## TableMetadata (第1層)

`core/.../TableMetadata.java`

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

## Snapshot (第2層)

`api/.../Snapshot.java`

テーブルデータのポイントインタイムビュー。第3層の Manifest List への参照 (`manifestListLocation`) を持つ:

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

## ManifestFile (第3層)

`api/.../ManifestFile.java`

Manifest List (第3層) の中の 1 エントリで、第4層のマニフェストファイルを指すメタデータ:

| フィールド | 説明 |
|-----------|------|
| `path` / `length` | マニフェストファイルの位置とサイズ |
| `content` | DATA (0) or DELETES (1) |
| `partitionSpecId` | 使用パーティション spec |
| `sequenceNumber` / `minSequenceNumber` | シーケンス番号追跡 |
| `addedFilesCount` / `existingFilesCount` / `deletedFilesCount` | ファイル数 |
| `partitions` | `PartitionFieldSummary` のリスト (containsNull, lowerBound, upperBound) |
| `firstRowId` | Row Lineage (v3+) |

マニフェストはスナップショット間で**再利用**される。変更のないマニフェストは書き直さない。1 回のコミットで新規に書くのは Manifest List (必ず全体を書き直す) と、変更分のマニフェストだけ。ここが O(1) スキャンプランニングとコミットコスト一定性の基盤になっている。

`partitions` の `PartitionFieldSummary` は、スキャンプランニング時にマニフェスト単位で枝刈りするために使われる ([09-scan-planning.md](09-scan-planning.md#エンドツーエンド-スキャンプランニング) のステップ 3)。

## DataFile / DeleteFile (第5層)

`api/.../ContentFile.java`

マニフェスト (第4層) 内の 1 エントリ。データファイル or 削除ファイルへのポインタ + 統計情報:

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

## Sequence Number の役割

`sequenceNumber` は上記のほぼ全ての層に登場するが、単なる連番ではなく **row-level delete の適用範囲を決める論理時刻**として機能する。v2 で導入された ([02-format-spec.md](02-format-spec.md#v2-row-level-deletes))。

### なぜ必要か

削除ファイルは「どのデータファイルに効くか」を自分では完全に特定できない。equality delete は「`id = 5` の行を消す」としか言っておらず、対象データファイルを列挙しない。そこで「**削除ファイルより古いデータファイルにのみ適用する**」という時間順序のルールが必要になる。この「古い/新しい」を定義するのが sequence number。

これがないと、削除をコミットした後に追加された `id = 5` の行まで消えてしまう。

### 2 種類の sequence number

| 種類 | 意味 | 用途 |
|------|------|------|
| **data sequence number** | ファイルの**中身**の論理的な古さ。一度割り当てたら永久に変わらない | 削除ファイルの適用判定 |
| **file sequence number** | そのファイルを**追加した**スナップショットの sequence number | 追加時期の追跡のみ |

両者が食い違うのは、コンパクションのように「古いデータを新しいファイルに書き直す」場合。このとき file sequence number は新しくなるが、data sequence number は元の値を明示的に引き継ぐ。**削除の判定に file sequence number を使ってはいけない**のはこのため — 使うと、コンパクション後に既存の削除ファイルが効かなくなる。

### 継承の仕組み

```
TableMetadata.last-sequence-number      ← コミット成功時に確定
      │
      ▼ 次の値を楽観的に割り当て
Snapshot.sequence-number
      │
      ▼ Manifest List にマニフェストごとの値として書き込む
ManifestFile.sequence_number
      │
      ▼ マニフェスト内エントリが null なら、読み取り時にこの値で埋める
ManifestEntry.sequence_number / file_sequence_number
```

新規追加されるファイルのエントリは、sequence number を **`null` で書く**。読み取り時に、そのエントリが属するマニフェストの値で置き換えられる (status が `added` の場合のみ)。既存ファイルとして書き直す場合や削除済みマークの場合は、継承済みの値を明示的に書く。

### 継承がもたらすもの

コミットは OCC なのでリトライされうるが、リトライのたびに sequence number は付け直される ([08-commit-and-write-path.md](08-commit-and-write-path.md#楽観的並行制御-occ))。それでもこの継承設計のおかげで、**書き直しが必要なのは Manifest List だけ**で済む。

- データファイル: 書き直し不要 (中身に sequence number が入っていない)
- マニフェストファイル: 書き直し不要 (エントリは `null` のまま)
- Manifest List: 書き直す — が、これはどのみち毎回書き直すもの

つまり「リトライのコストを Manifest List 1 個の書き直しに抑える」ための設計。同じ発想が Row Lineage の `first_row_id` 継承にも使われている ([11-advanced-features.md](11-advanced-features.md#割り当てメカニズム-多段継承))。

> v1 のマニフェストには sequence number カラムがないため、読み取り時はすべて `0` として扱う。

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
- `main` ブランチは `current-snapshot-id` を指す。スナップショットが 1 つも無いテーブルでは `refs` は空で、`main` のエントリも存在しない。一方、一度作られた `main` は `max-ref-age-ms` の対象外で期限切れにならない (`format/spec.md` の Snapshot References)

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
