# エンティティモデルと Realm

Polaris のデータモデルは、Iceberg のような専用構造体の集まりではなく、**単一のジェネリックなエンティティテーブル**に集約されている。
カタログもテーブルもプリンシパルもロールもポリシーも、すべて同じ `PolarisBaseEntity` として保存され、`type_code` / `sub_type_code` で区別される。

## エンティティ型 (`polaris-core/.../core/entity/PolarisEntityType.java`)

| 型 | code | 親型 | grantee か | 自己参照 |
|----|------|------|-----------|---------|
| `NULL_TYPE` | 0 | - | - | - |
| `ROOT` | 1 | - | no | no |
| `PRINCIPAL` | 2 | `ROOT` | **yes** | no |
| `PRINCIPAL_ROLE` | 3 | `ROOT` | **yes** | no |
| `CATALOG` | 4 | `ROOT` | no | no |
| `CATALOG_ROLE` | 5 | `CATALOG` | **yes** | no |
| `NAMESPACE` | 6 | `CATALOG` | no | **yes** |
| `TABLE_LIKE` | 7 | `NAMESPACE` | no | no |
| `TASK` | 8 | `ROOT` | no | no |
| `FILE` | 9 | `TABLE_LIKE` | no | no |
| `POLICY` | 10 | `NAMESPACE` | no | no |

型の 2 つのフラグが重要:

- **`isGrantee`** — grant の受け手になれるか。`PRINCIPAL`, `PRINCIPAL_ROLE`, `CATALOG_ROLE` のみ true。
  つまり「テーブルに直接権限を付ける」ことはできず、必ずロール経由になる ([04](04-rbac-and-authorization.md))
- **`isParentSelfReference`** — 同じ型を親に持てるか。`NAMESPACE` のみ true。これがネストした namespace (`a.b.c`) を表現する

`getParentType()` が親型を返すので、階層の妥当性チェックがコードでなく型定義から導ける。

### サブタイプ (`PolarisEntitySubType.java`)

`TABLE_LIKE` だけがサブタイプで分岐する:

| サブタイプ | code | 説明 |
|-----------|------|------|
| `ANY_SUBTYPE` | -1 | 保存されない。「サブタイプを問わず」を意味する検索用の擬似値 |
| `NULL_SUBTYPE` | 0 | サブタイプなし (該当なし) |
| `ICEBERG_TABLE` | 2 | Iceberg テーブル |
| `ICEBERG_VIEW` | 3 | Iceberg ビュー |
| `GENERIC_TABLE` | 4 | Delta / Hudi / Paimon 等の非 Iceberg テーブル ([09](09-federation-and-extensions.md)) |

`ANY_SUBTYPE` が「テーブルもビューも同じ名前空間を共有する」ことを実現している。
`ns.foo` という名前のテーブルとビューは共存できない — 存在チェックは `ANY_SUBTYPE` で行われ、
別サブタイプが見つかれば `AlreadyExistsException` になる。

## エンティティの共通フィールド

`PolarisEntityCore` (識別と版) → `PolarisBaseEntity` (状態) → `PolarisEntity` (Map 化されたプロパティ付き) の 3 段継承。

### `PolarisEntityCore` — 同一性

| フィールド | 説明 |
|-----------|------|
| `catalogId` | 所属カタログの id (トップレベルエンティティは `NULL_ID`=0) |
| `id` | エンティティ id (realm 内で一意) |
| `parentId` | 親エンティティ id |
| `typeCode` | `PolarisEntityType` の code |
| `name` | 名前 |
| `entityVersion` | **楽観的並行制御用の版番号**。更新のたびにインクリメントされる |

一意制約は `(realm_id, catalog_id, parent_id, type_code, name)`。
`catalogId` を `parentId` とは別に持つのは、テーブルから「どのカタログに属するか」を親を辿らず 1 ホップで引くため。

### `PolarisBaseEntity` — 状態

| フィールド | 説明 |
|-----------|------|
| `subTypeCode` | `PolarisEntitySubType` |
| `createTimestamp` / `lastUpdateTimestamp` | 作成 / 最終更新 |
| `dropTimestamp` | ドロップ時刻 (0 なら生存)。**論理削除**である |
| `purgeTimestamp` / `toPurgeTimestamp` | 実削除時刻 / 実削除予定時刻 (既定の保持期間は 24 時間: `PolarisEntityConstants.RETENTION_TIME_IN_MS`) |
| `properties` | ユーザー可視プロパティ (JSON 文字列) |
| `internalProperties` | **Polaris 内部プロパティ** (JSON 文字列)。ストレージ設定、接続設定、`metadata.json` の場所などが入る |
| `grantRecordsVersion` | このエンティティに紐づく grant record 群の版番号 |

`properties` と `internalProperties` の分離が重要。クライアントが REST API 経由で設定できるのは前者だけで、
`metadataLocation` (Iceberg の `metadata.json` へのポインタ)、`storageConfigInfo`、`connectionConfigInfo` などは後者に入る。

`grantRecordsVersion` がエンティティ本体の `entityVersion` と別に存在するのは、**権限変更でエンティティ本体を書き換えたくない**ため。
権限だけ変わったときにキャッシュの本体部分を無効化せずに済む ([04](04-rbac-and-authorization.md) の Resolver)。

## 型付きエンティティ (ラッパー)

`PolarisBaseEntity` の生の `properties`/`internalProperties` に型付きアクセサを被せたクラス群:

| クラス | 主な内部プロパティ |
|-------|------------------|
| `CatalogEntity` | `catalogType` (`INTERNAL` / `EXTERNAL`)、`storageConfigInfo`、`connectionConfigInfo`、`default-base-location` |
| `NamespaceEntity` | `location` |
| `IcebergTableLikeEntity` (`entity/table/`) | `metadata-location`、`base-location`、ユーザー指定 write location |
| `GenericTableEntity` (`entity/table/`) | フォーマット名、doc、プロパティ |
| `PolicyEntity` (`policy/`) | ポリシー種別、内容 (JSON)、バージョン |
| `PrincipalEntity` / `PrincipalRoleEntity` / `CatalogRoleEntity` | `client_id`、`principal_type_name`、資格情報ローテーション要求フラグ |
| `TaskEntity` | 非同期タスクのデータ ([08](08-runtime-quarkus.md)) |
| `EventEntity` | 永続化されたイベント ([08](08-runtime-quarkus.md)) |

`IcebergTableLikeEntity.getMetadataLocation()` が返す値こそが、Polaris がカタログとして保持している本質的な情報である。
Iceberg テーブルの実体 (スキーマ、スナップショット、マニフェスト) はこのパスの先にあり、Polaris は中身を保存しない。

## Grant Record (`PolarisGrantRecord.java`)

権限はエンティティとは別テーブルに、5 つ組で保存される:

```
(securableCatalogId, securableId, granteeCatalogId, granteeId, privilegeCode)
```

- **securable** = 権限を付ける対象 (カタログ、namespace、テーブル、ロールそのもの)
- **grantee** = 権限を受け取る側 (`isGrantee` が true の型のみ)
- **privilege** = `PolarisPrivilege` の code

これがそのまま JDBC の `grant_records` テーブルの複合主キーになる ([03](03-persistence.md))。

## エンティティ階層の具体例

```
ROOT (id=0, name="root_container")            ← すべてのトップレベル grant の securable
 ├── PRINCIPAL "root"
 ├── PRINCIPAL_ROLE "service_admin"
 ├── PRINCIPAL_ROLE "data_engineer"
 └── CATALOG "prod_catalog"
      ├── CATALOG_ROLE "catalog_admin"
      ├── CATALOG_ROLE "reader"
      └── NAMESPACE "sales"
           ├── NAMESPACE "raw"                 ← NAMESPACE は自己参照可
           │    └── TABLE_LIKE "events" (ICEBERG_TABLE)
           ├── TABLE_LIKE "orders" (ICEBERG_TABLE)
           ├── TABLE_LIKE "orders_v" (ICEBERG_VIEW)
           └── POLICY "retention_policy"
```

`ROOT` エンティティ (`root_container`) は実体のない「入れ物」で、`CATALOG_CREATE` や `PRINCIPAL_LIST` のような
特定のカタログに属さない権限の securable として使われる。

## ブートストラップ (`polaris-core/.../core/auth/AuthBootstrapUtil.java`)

新しい realm の初期化時に作られるものは以下だけ:

1. `ROOT` エンティティ `root_container` (id=0)
2. `PRINCIPAL` `root` — 資格情報 (client id / secret) が生成され、返される
3. `PRINCIPAL_ROLE` `service_admin`
4. grant: `root` プリンシパルに `service_admin` ロールの `PRINCIPAL_ROLE_USAGE`
5. grant: `service_admin` ロールに `root_container` 上の `SERVICE_MANAGE_ACCESS`

`SERVICE_MANAGE_ACCESS` が全権限を包含する super privilege なので、これだけで `root` は何でもできる状態になる。
以降のカタログ・ユーザ作成は Management API 経由で行う。冪等で、既に root プリンシパルが存在すれば `ENTITY_ALREADY_EXISTS` を返して何もしない。

実行は admin CLI (`polaris-admin bootstrap`) から ([08](08-runtime-quarkus.md))。

## Realm — マルチテナンシー

Polaris のマルチテナンシーは「Realm」という単位で表現される。

- **すべての永続化行が `realm_id` (文字列) を持つ**。JDBC では複合主キーの先頭、NoSQL では `realmId + objectId` が主キー
- Realm はリクエストごとに HTTP ヘッダ (既定 `Polaris-Realm`) から解決される ([05](05-authentication.md))
- `MetaStoreManagerFactory.getOrCreateMetaStoreManager(RealmContext)` が realm ごとに `PolarisMetaStoreManager` を返す。
  エンティティキャッシュも realm ごとに分かれる
- 設定も realm ごとに上書き可能 (`RealmConfig`) ([08](08-runtime-quarkus.md))
- ブートストラップと purge は realm 単位 (`bootstrapRealms(Iterable<String>)` / `purgeRealms(...)`)

つまり Realm は「同一プロセス・同一 DB を共有しつつ、エンティティ空間と設定を完全に分離する」テナント境界である。
DB を分けるかどうかは運用の選択であって、コードは 1 DB に複数 realm が同居することを前提に書かれている。

## `CallContext` — リクエストスコープの束

`polaris-core/.../core/context/CallContext.java` が 1 リクエスト分のコンテキストを保持する:

- `RealmContext` — realm ID
- `RealmConfig` — この realm に解決済みの設定
- `PolarisCallContext` — 永続化層に渡す下位コンテキスト (`BasePersistence`, 診断, 時計)

`polaris-core` 側の API はほぼすべて第 1 引数に `PolarisCallContext` を取る。これにより `polaris-core` は
スレッドローカルや CDI のリクエストスコープに依存せず、純粋なライブラリとして使える。
リクエストスコープの管理は `runtime/service` 側の CDI が担当する。
