# Catalog 抽象と REST Catalog

## Catalog インターフェース (`api/.../catalog/Catalog.java`)

| メソッド | 説明 |
|---------|------|
| `createTable(id, schema, spec, location, props)` | テーブル作成 |
| `loadTable(id)` | テーブルロード |
| `dropTable(id, purge)` | テーブル削除 |
| `renameTable(from, to)` | リネーム |
| `registerTable(id, metadataLocation)` | 既存メタデータの登録 |
| `buildTable(id, schema)` | Fluent ビルダー |
| `initialize(name, properties)` | 初期化 (エンジンが呼ぶ) |

関連インターフェース:
- `SupportsNamespaces`: namespace CRUD
- `SessionCatalog`: セッション情報付き (REST Catalog 用)
- `ViewCatalog`: View ライフサイクル

## CatalogUtil によるカタログ解決

`CatalogUtil.buildIcebergCatalog(name, options, conf)` で `type` プロパティ → クラス解決:

| type | クラス |
|------|--------|
| `hadoop` | `HadoopCatalog` |
| `hive` | `HiveCatalog` |
| `rest` | `RESTCatalog` |
| `glue` | `GlueCatalog` |
| `nessie` | `NessieCatalog` |
| `jdbc` | `JdbcCatalog` |
| `bigquery` | `BigQueryMetastoreCatalog` |

ロードパターン: no-arg コンストラクタ → Hadoop Configuration 注入 (if `Configurable`) → `catalog.initialize()`

## TableOperations: SPI 層

Catalog と Table の間の SPI (Service Provider Interface):

| メソッド | 説明 |
|---------|------|
| `current()` | 現在の `TableMetadata` を返す |
| `refresh()` | ストレージからメタデータを再読込 |
| `commit(base, metadata)` | **アトミック CAS** (base が現在のものか検証、失敗時 `CommitFailedException`) |
| `io()` | FileIO |
| `metadataFileLocation(fileName)` | メタデータファイルパス解決 |
| `locationProvider()` | 新データファイルの配置先提供 |
| `newSnapshotId()` | 一意スナップショット ID 生成 |

## 各 Catalog 実装のコミット方式

| Catalog | コミット方式 | 特徴 |
|---------|------------|------|
| **HadoopCatalog** | ファイルシステムの atomic rename | `v{N+1}.metadata.json` へのリネームがコミット |
| **HiveCatalog** | HMS ロック + `alter_table` | `metadata_location` プロパティを更新 |
| **GlueCatalog** | Glue `UpdateTable` + versionId | 楽観ロック |
| **DynamoDbCatalog** | DynamoDB conditional write | `ConditionalCheckFailedException` で衝突検知 |
| **NessieCatalog** | Nessie API (Git-like) | ブランチ・タグによるバージョニング |
| **JdbcCatalog** | SQL トランザクション | 任意の JDBC DB |
| **RESTCatalog** | HTTP POST + `UpdateTableRequest` | サーバー側でメタデータ管理 |

## CachingCatalog

任意の `Catalog` をラップする Caffeine ベースのキャッシュデコレータ。TTL 設定可能、ケースインセンシティブキー対応。ベーステーブルの有効期限切れ時にメタデータテーブルエントリも自動無効化。

## REST Catalog

### OpenAPI 仕様 (`open-api/rest-catalog-open-api.yaml`)

主要エンドポイント:

| パス | 操作 |
|------|------|
| `/v1/config` | カタログ設定取得 |
| `/v1/oauth/tokens` | OAuth2 トークン交換 |
| `namespaces` | リスト/作成 |
| `namespaces/{ns}/tables` | リスト/作成 |
| `namespaces/{ns}/tables/{table}` | ロード/コミット/削除 |
| `namespaces/{ns}/tables/{table}/credentials` | 認証情報取得 |
| `namespaces/{ns}/tables/{table}/plan` | スキャンプラン送信/取得 |
| `transactions/commit` | マルチテーブルコミット |
| `namespaces/{ns}/views` | ビュー操作 |

### クライアント実装

`RESTCatalog` → `RESTSessionCatalog` → `HTTPClient` (Apache HttpClient 5)

コミット方式: `UpdateTableRequest` = `UpdateRequirement` (前提条件) + `MetadataUpdate` (変更) のリスト。サーバー側で原子的に適用。クライアントはメタデータファイルを直接書かない。

認証: `OAuth2Manager`, `BasicAuthManager`, `NoopAuthManager` のプラグイン方式。
FileIO: デフォルト `ResolvingFileIO`。サーバーからの払い出しストレージ認証情報 (vended credentials) 対応。
キャッシュ: `RESTTableCache` による ETag ベースキャッシュ。
