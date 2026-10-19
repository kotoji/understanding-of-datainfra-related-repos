# 永続化層

Polaris の永続化は 2 段のインターフェースに分かれている。

```
PolarisMetaStoreManager  (業務ロジック層: エンティティ CRUD + grant + policy + secrets)
        │  実装: AtomicOperationMetaStoreManager / TransactionalMetaStoreManagerImpl
        ▼
BasePersistence          (ストレージ SPI: 単一エンティティ/grant の読み書き)
        │  実装: JdbcBasePersistenceImpl / NoSQL / TreeMapTransactionalPersistenceImpl
        ▼
PostgreSQL / H2 / CockroachDB / MongoDB / in-memory
```

この分割の狙いは、**バックエンドに要求する保証を「単一行の atomic な read/write」だけに絞る**こと。
複数ステートメントにまたがるオープンなトランザクションを前提にしないため、トランザクションを持たない
NoSQL バックエンドも同じ業務ロジックで動かせる。

## `BasePersistence` (`polaris-core/.../core/persistence/BasePersistence.java`)

クラスコメントに明記されているとおり「**このインターフェースの各メソッドは atomic でなければならない**」。

| メソッド | 説明 |
|---------|------|
| `generateNewId(ctx)` | 一意な entity id を採番 |
| `writeEntity(ctx, entity, nameOrParentChanged, originalEntity)` | エンティティを 1 件書く。`originalEntity` が null なら INSERT、非 null なら **CAS UPDATE** |
| `writeEntities(...)` | 複数エンティティを atomic に書く |
| `writeToGrantRecords(ctx, grantRec)` | grant record を 1 件書く |
| `writeEvents(events)` | イベントを書く |
| `deleteEntity` / `deleteFromGrantRecords` / `deleteAllEntityGrantRecords` / `deleteAll` | 削除系 |
| `lookupEntity` / `lookupEntityByName` / `lookupEntities` | 参照系 |
| `lookupEntityVersion` / `lookupEntityGrantRecordsVersion` | 版番号のみの軽量参照 |
| `listEntities(...)` | 子エンティティ列挙 (ページネーション対応) |
| `hasChildren(...)` | 子の存在確認 (namespace drop の前提チェック) |
| `hasOverlappingSiblings(...)` | **ロケーション重なりの最適化チェック** (default 実装は `Optional.empty()` = 未対応) |
| `loadPrincipalSecrets` / `generateNewPrincipalSecrets` / `rotatePrincipalSecrets` (`IntegrationPersistence`) | プリンシパル資格情報 |

`hasOverlappingSiblings` が `Optional<Optional<String>>` という二重 Optional を返すのが独特。
外側が「この実装はこのチェックをサポートするか」、内側が「重なる兄弟が見つかったか (見つかればそのロケーション)」を表す。
JDBC 実装は `location_without_scheme` 列と部分インデックスでこれを O(log n) で答えられるが、
他の実装は答えられないので「サポートしない」を返し、呼び出し側は全件走査にフォールバックする ([06](06-iceberg-rest-service.md))。

`PolicyMappingPersistence` を継承しており、ポリシーのアタッチ情報も同じ SPI に含まれる。

## `PolarisMetaStoreManager` (`polaris-core/.../core/persistence/PolarisMetaStoreManager.java`)

より業務寄りの API。`PolarisSecretsManager`, `PolarisGrantManager`, `PolarisPolicyMappingManager`, `PolarisEventManager` を継承する。

| メソッド群 | 例 |
|-----------|---|
| ブートストラップ | `bootstrapPolarisService`, `purge` |
| 生成 | `createCatalog`, `createPrincipal`, `createEntityIfNotExists`, `createEntitiesIfNotExist` |
| 更新 | `updateEntityPropertiesIfNotChanged`, `updateEntitiesPropertiesIfNotChanged`, `renameEntity` |
| 削除 | `dropEntityIfExists` |
| 参照 | `loadEntity`, `readEntityByName`, `listEntities`, `listFullEntities`, `loadTasks` |
| **解決** | `loadResolvedEntityById`, `loadResolvedEntityByName`, `loadResolvedEntities`, `refreshResolvedEntity` |
| 版追跡 | `loadEntitiesChangeTracking` |

戻り値はすべて `*Result` 型 (`EntityResult`, `EntitiesResult`, `DropEntityResult`, ...) で、
`BaseResult.ReturnStatus` の enum を持つ。例外でなく戻り値でエラーを表現する設計。

「解決 (resolved)」系メソッドは **エンティティ本体と、そのエンティティを securable とする grant record 群をまとめて返す**。
これが認可判定の 1 リクエストあたりのラウンドトリップ数を抑える鍵になっている ([04](04-rbac-and-authorization.md))。

### 2 つの実装

| 実装 | 説明 |
|------|------|
| `AtomicOperationMetaStoreManager` | 「オープンな多段トランザクションを一切使わず、`BasePersistence` への one-shot atomic 操作のみに依存する」実装。1,925 行 |
| `TransactionalMetaStoreManagerImpl` (`transactional/`) | `TransactionalPersistence` (多段トランザクションを持つバックエンド) 向けの実装 |

`TreeMapTransactionalPersistenceImpl` + `TreeMapMetaStore` はインメモリの `TreeMap` ベース実装で、テストと
`polaris.persistence.type=in-memory` 用。

## JDBC 実装 (`persistence/relational-jdbc`)

**既定の永続化実装**。対応 DB は `DatabaseType` enum のとおり:

| DatabaseType | 最新スキーマ版 | 備考 |
|--------------|--------------|------|
| `POSTGRES` | 5 | 本番想定 |
| `COCKROACHDB` | 5 | PostgreSQL とスキーマを同期 |
| `H2` | 5 | PostgreSQL と同一スキーマを使用。テスト・軽量用途 |

DDL は `src/main/resources/{postgres,h2,cockroachdb}/schema-v{1..5}.sql` にベタ書きされている
(Flyway/Liquibase のようなマイグレーションツールは使わない)。`version` テーブルに現在のスキーマ版が入る。

### 主要テーブル (postgres/schema-v5.sql)

**`entities`** — すべてのエンティティ

```sql
PRIMARY KEY (realm_id, id)
UNIQUE (realm_id, catalog_id, parent_id, type_code, name)
```

列は [02](02-entity-model.md) のフィールド一覧に対応。`properties` / `internal_properties` は `JSONB`。
特徴的なのは 2 つ:

- **`location_without_scheme`** — テーブル/namespace のロケーションからスキーム (`s3://` 等) を除いた文字列。
  部分インデックス `idx_locations ON (realm_id, parent_id, location_without_scheme) WHERE location_without_scheme IS NOT NULL`
  が張られており、「兄弟エンティティとロケーションが重なっていないか」を prefix 検索で判定できる (`hasOverlappingSiblings`)
- **`grant_records_version`** — エンティティ本体とは独立した grant の版番号

**`grant_records`** — 権限

```sql
PRIMARY KEY (realm_id, securable_catalog_id, securable_id,
             grantee_catalog_id, grantee_id, privilege_code)
```

grantee 側 (`idx_grants_realm_grantee`) と securable 側 (`idx_grants_realm_securable`) の両方にインデックス。
「このロールは何ができるか」と「このテーブルには誰がアクセスできるか」の双方向の問い合わせを支える。

**`principal_authentication_data`** — プリンシパルの資格情報。
`main_secret_hash` / `secondary_secret_hash` / `secret_salt` を持ち、平文の secret は保存されない。
secondary があるのは、無停止でのローテーションのため (新旧両方を一時的に有効にする)。

**`policy_mapping_record`** — ポリシーのアタッチ関係 ([09](09-federation-and-extensions.md))

**`events`** — 永続化されたイベント ([08](08-runtime-quarkus.md))

**`scan_metrics_report` / `commit_metrics_report`** — Iceberg クライアントが `/metrics` エンドポイントに
報告するスキャン・コミットメトリクスを列に展開して保存する (`extensions/metrics-reports`)。
`otel_trace_id` / `otel_span_id` 列を持ち、OpenTelemetry のトレースと突き合わせられる。

### 楽観的並行制御の実装

`JdbcBasePersistenceImpl.writeEntity()` の UPDATE パスが Polaris の OCC の中核:

```
UPDATE entities SET ... WHERE id = ? AND catalog_id = ? AND realm_id = ?
                         AND entity_version = ? AND grant_records_version = ?
```

更新行数が 0 なら `RetryOnConcurrencyException` を投げる。
**CAS 条件に `entity_version` と `grant_records_version` の両方を含める**のがポイントで、
grant 操作は `grant_records_version` だけを上げて `entity_version` に触らないため、片方だけでは競合を取りこぼす。

INSERT パスでは一意制約違反を捕まえて名前で再検索し、
- 既存エンティティが見つかれば `EntityAlreadyExistsException`
- 見つからなければ (SERIALIZABLE 分離レベルで競合相手が自分のスナップショットから見えない場合) `RetryOnConcurrencyException`

と区別している。後者を `AlreadyExists` に丸めてしまうと、実際は再試行で成功するケースをクライアントエラーとして返してしまう。

## NoSQL 実装 (`persistence/nosql/**`, 39 モジュール)

既定ではないが、リポジトリのコード量の約 1/3 を占める別系統の永続化フレームワーク。
`persistence/nosql/persistence/README.md` が設計を詳述している。

### 前提

バックエンドに要求するのは **単一行の compare-and-swap のみ**。
加えて、行サイズのハードリミットを考慮して **共通の 350 kB 上限**を設けている
(DynamoDB の 400 kB 制限、MariaDB/MySQL の既定 512 kB パケット上限などに合わせた値)。

### オブジェクトモデル

- 永続化される値は `Obj` インターフェースを実装した **immutable な Jackson シリアライズ可能型**
- 型は `ObjType` (一意な名前 + 対象 Java 型) で記述し、Java の ServiceLoader で登録する
- 論理キーは `ObjId` = `ObjType.id()` + `long` ID
- DB 上の主キーは常に **realmId + object-ID**

### Named pointer と commit

カタログ全体の状態を「HEAD」(Git の HEAD のような named pointer) が指す。
HEAD が指す `Obj` はカタログエンティティのインデックスを保持し、複数エンティティの変更を **1 つの named pointer に対する
atomic な更新**として反映する。これによりマルチテーブルの atomic なコミットが成立する。

commit 機構はコミットログを作るので、`git log` のように変更履歴を順序どおり辿れる。
「あるコミット ID 以降の変更をすべて取得する」ポーリング API が自然に得られ、`Obj` が immutable なため
その問い合わせはキャッシュにヒットしやすい。

### Snowflake ID

`long` ID は Snowflake 形式:

| ビット | 内容 |
|-------|------|
| 41 | タイムスタンプ (ミリ秒精度、エポックは 2025-03-01 00:00:00 GMT、約 69 年分) |
| 10 | ノード ID (同時稼働 1,024 JVM まで) |
| 12 | シーケンス番号 (ノードあたり 4,096 ID/ms) |
| 1 | 予約 |

ノード ID は各 JVM が**リース**する。リース管理も DB の CAS で行い、
「リース期間を超えるタイムスタンプの ID は生成しない」「過去のタイムスタンプを再利用しない」を
monotonic clock で保証する。これが `polaris-nodes-*` と `polaris-idgen-*` モジュールの中身。

### 強整合 vs 結果整合

- **強整合書き込み**: シリアライズ後のサイズがハードリミット内に収まる必要がある
- **結果整合書き込み**: リミットを超えるオブジェクトを複数行に分割して書ける

インデックスも同様に、埋め込みインデックスがサイズ上限に近づくと別行へ spill out し、必要に応じて分割・結合する。
数千〜十数万テーブル規模のカタログ状態を 1 行に収められないための仕組み。

### DB 実装

現状 `polaris-persistence-nosql-inmemory` と `polaris-persistence-nosql-mongodb` の 2 つ。
新しい DB を追加する際は、Quarkus 拡張として `polaris-persistence-nosql-cdi-quarkus` に配線し、
`polaris-persistence-nosql-correctness` の正当性テストと `polaris-persistence-nosql-benchmark` に組み込む、と README が定めている。

## エンティティキャッシュ (`polaris-core/.../core/persistence/cache/`)

`EntityCache` インターフェースと `InMemoryEntityCache` 実装 (Caffeine ベース)。realm ごとに 1 個。

| 設定 | 値 |
|------|---|
| 退去方式 | `maximumWeight` + `EntityWeigher` (エントリのバイトサイズ推定で重み付け) |
| 上限 | `ENTITY_CACHE_WEIGHER_TARGET` (feature config) |
| 有効期限 | `expireAfterAccess(1, TimeUnit.HOURS)` — 最終アクセスからの経過 |
| ソフト参照 | 条件付きで `softValues()` — ヒープ逼迫時に GC が回収しうる |

キャッシュのキーは id と名前の 2 系統 (`EntityCacheByNameKey`)。

重要なメソッドは `getAndRefreshIfNeeded(ctx, entityToValidate, entityMinVersion, entityGrantRecordsMinVersion)`。
「キャッシュにあるが、指定した最小版より古ければリロードする」という条件付き取得で、
`entityVersion` と `grantRecordsVersion` を別々に指定できる。
権限だけが変わったときにエンティティ本体を読み直さずに済む ([02](02-entity-model.md) の版番号分離の目的)。

キャッシュを使うと「キャッシュから読んだ値が既に古い」可能性が常にあるため、Resolver は
**多段パス (multi-pass) での解決**を行う ([04](04-rbac-and-authorization.md))。

## `MetaStoreManagerFactory`

realm から永続化コンポーネントを取り出すファクトリ。実装は 3 つ:

| 実装 | モジュール |
|------|-----------|
| `LocalPolarisMetaStoreManagerFactory` (abstract) | `polaris-core` |
| `JdbcMetaStoreManagerFactory` | `persistence/relational-jdbc` |
| `NoSqlMetaStoreManagerFactory` | `persistence/nosql/persistence/metastore` |

```java
PolarisMetaStoreManager getOrCreateMetaStoreManager(RealmContext);
BasePersistence          getOrCreateSession(RealmContext);
EntityCache              getOrCreateEntityCache(RealmContext, RealmConfig);
Map<String, PrincipalSecretsResult> bootstrapRealms(BootstrapOptions);
Map<String, BaseResult>             purgeRealms(Iterable<String>);
```

`bootstrapRealms` / `purgeRealms` がここにあるのは、admin CLI がサーバーを起動せずに
このファクトリだけを立ち上げて realm を初期化・破棄するため ([08](08-runtime-quarkus.md))。

選択は Quarkus 設定 `polaris.persistence.type` (`in-memory` / `relational-jdbc` / NoSQL 系) で行う。

## ページネーション (`persistence/pagination/`)

`Page<T>` + `PageToken` + `Token` の組。`EntityIdToken` は「最後に返したエンティティ id」を持つ
カーソル方式のトークンで、offset 方式ではない。REST の list 系エンドポイントの `pageToken` パラメータに対応する。
