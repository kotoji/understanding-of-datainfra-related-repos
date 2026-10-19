# アーキテクチャの総括

## 一言でいうと

**Polaris は「Iceberg テーブルの現在位置を指すポインタ」と「そのポインタに誰が触れてよいかの権限」を、
マルチテナントに、かつストレージ認証情報の払い出しまで含めて管理するサービス**である。

テーブルの中身 (スキーマ、スナップショット、マニフェスト、データファイル) には一切関与しない。
関与するのは以下 3 点のみ:

1. `metadata.json` のパスを保持し、atomic に差し替える ([06](06-iceberg-rest-service.md))
2. そのパスが許可された範囲に収まっているかを検証する ([06](06-iceberg-rest-service.md), [07](07-storage-and-credential-vending.md))
3. そのパスの配下に絞った短期認証情報を発行する ([07](07-storage-and-credential-vending.md))

## 設計原則と、それがコードのどこに現れるか

### 1. 永続化バックエンドへの要求を最小化する

**要求するのは単一行の compare-and-swap のみ。** オープンな多段トランザクションを前提にしない。

| 現れ方 | 場所 |
|-------|------|
| `BasePersistence` の全メソッドが atomic であることを契約とする | [03](03-persistence.md) |
| `AtomicOperationMetaStoreManager` — 「one-shot atomic 操作のみに依存する」実装 | [03](03-persistence.md) |
| NoSQL フレームワークが「CAS on a single row がすべての要件」と宣言 | [03](03-persistence.md) |
| JDBC の `writeEntity` が `entity_version` + `grant_records_version` の CAS UPDATE | [03](03-persistence.md) |

代償として、複数エンティティにまたがる整合性は上位で組み立てる必要がある。
NoSQL 側は named pointer (Git の HEAD に相当) を導入してこれを解決している。

### 2. すべてを 1 つのエンティティテーブルに畳む

カタログもテーブルもプリンシパルもポリシーも `PolarisBaseEntity`。型は `type_code` で判別する。

| 得るもの | 失うもの |
|---------|---------|
| 権限チェックが「解決済みパスを走査して grant record を見る」という単一のアルゴリズムで済む | 型ごとのスキーマ最適化ができない (プロパティは JSON 文字列) |
| 新しいエンティティ種別 (POLICY など) の追加が enum への 1 行追加で済む | クエリが `type_code` フィルタだらけになる |
| 階層の妥当性が `PolarisEntityType` の `parentType` から導出できる | 型安全性はラッパークラス (`CatalogEntity` 等) で後付けする |

### 3. 認可を構造で強制する

「認可を書き忘れる」ことをレビューでなく型と制御フローで防ぐ。

```java
// CatalogHandler
protected PolarisResolutionManifest resolutionManifest = null;

/** Initialize the catalog once authorized. Called after all `authorize...` methods. */
protected abstract void initializeCatalog();
```

`authorize...` を通らないとカタログが初期化されないので、認可なしで操作が実行される経路が構造的に存在しない
([06](06-iceberg-rest-service.md))。

### 4. 権限は grant のみ、deny はない

`hasTransitivePrivilege` は「解決済みパス上のどこか 1 箇所に該当 grant があれば許可」。
明示的な拒否は表現できない ([04](04-rbac-and-authorization.md))。

これは Unix のパーミッションよりも Snowflake / BigQuery の RBAC に近いモデルで、
「例外的に 1 テーブルだけ禁止する」ができない代わりに、権限の評価が単調で予測しやすい。

包含関係 (`SUPER_PRIVILEGES`) は階層から自動導出せず、静的テーブルに手で書き下している。
`TABLE_READ_DATA ⊒ TABLE_READ_PROPERTIES` のような**意味的な**関係を機械的に導けないため。

### 5. 認可の結果を IAM ポリシーとして具体化する

RBAC の判定結果が、そのまま STS の session policy になる ([07](07-storage-and-credential-vending.md))。

```
TABLE_READ_DATA を持つ  →  LocationGrant(s3://bucket/db/tbl, {READ, LIST})
                        →  IamPolicy: Allow s3:GetObject on arn:aws:s3:::bucket/db/tbl/*
                        →  AssumeRole(policy=...)  →  短期認証情報
```

「Polaris のカタログ権限」と「ストレージの IAM 権限」を二重管理せずに済む。
これが Polaris を単なるメタデータサーバーと分ける最大の特徴。

空ポリシーになる場合に明示的な `Deny *` を出すガードや、
IAM グロブ特殊文字のエスケープは、この変換が失敗すると即座に権限昇格になることの裏返し。

### 6. Realm を全層に貫通させる

realm は「設定して終わり」ではなく、あらゆる層のキーになっている。

| 層 | realm の効き方 |
|---|--------------|
| フィルタ | 認証より**前**に解決される ([05](05-authentication.md)) |
| 設定 | `Map<String, XxxConfiguration> realms()` パターンで全設定が realm 別上書き可能 ([08](08-runtime-quarkus.md)) |
| 認証 | realm ごとに `INTERNAL` / `EXTERNAL` / `MIXED` を選べる ([05](05-authentication.md)) |
| 永続化 | 全テーブルの主キー先頭が `realm_id` ([03](03-persistence.md)) |
| キャッシュ | エンティティキャッシュは realm ごと ([03](03-persistence.md)) |
| レート制限 | `RealmTokenBucketRateLimiter` が realm ごとにバケット ([08](08-runtime-quarkus.md)) |
| ログ | MDC に `realmId` を注入し全ログ行に出力 ([08](08-runtime-quarkus.md)) |
| 監査 | STS セッションタグ `polaris:realm` ([07](07-storage-and-credential-vending.md)) |
| 運用 | bootstrap / purge が realm 単位 ([08](08-runtime-quarkus.md)) |

realm は列挙された許可リストで管理され、任意の文字列で勝手に生えない。

### 7. 差し替え可能性は CDI の `@Identifier` に統一する

認可器、認証、永続化、FileIO、イベントリスナ、レート制限、prefix パーサ、
principal マッパー — すべて同じパターン。設定 1 行で実装が切り替わる ([08](08-runtime-quarkus.md))。

これを支えているのが `polaris-core` の Quarkus 非依存性。
`polaris-core` は CDI もスレッドローカルも使わず、`PolarisCallContext` を明示的に引数で回す。
おかげで認可ロジックや永続化実装をライブラリとして単体テストできる。

### 8. REST インターフェースは仕様から生成する

`spec/*.yaml` → `jaxrs-resteasy` → JAX-RS インターフェース → `runtime/service` が `implements`。
Iceberg 本家のリクエスト/レスポンス型をそのまま使うので、仕様の齟齬がコンパイルエラーになる ([01](01-build-and-modules.md))。

同じ仕様から `apis` / `models` フィルタで生成対象を切り分けて 4 モジュールに分ける構成が、
「モデルの生成箇所は 1 つ、API は用途別」を実現している。

### 9. 設定の危険性を起動時に検査する

`ProductionReadinessCheck` は Polaris 独特の仕組み ([08](08-runtime-quarkus.md))。
各モジュールが `@Produces` でチェックを提供し、severe なものがあれば**起動を止める**。
明示的にリスクを受け入れる設定 (`polaris.readiness.ignore-severe-issues=true`) を書かないと起動しない。

既定値が本番向きでない箇所 (`persistence.type=in-memory` など) を意図的に残しつつ、
それを本番で使わせないための安全装置。

## 全体の噛み合い方

1 回の `loadTable` (アクセス委譲つき) を通して見ると:

```
GET /v1/prod_catalog/namespaces/sales.raw/tables/events
Polaris-Realm: prod
Authorization: Bearer <token>
X-Iceberg-Access-Delegation: vended-credentials
  │
  ▼ [05] RealmContextFilter (認証より前)
RealmContext("prod") → この realm の認証設定・永続化・キャッシュが決まる
  │
  ▼ [05] InternalAuthenticationMechanism (priority +100) → JWTBroker.verify()
     または Quarkus OIDC → PrincipalMapper / PrincipalRolesMapper
  ▼ [05] AuthenticatingAugmentor → Authenticator
PolarisPrincipal("alice", roles=[data_engineer])
  │
  ▼ [06] IcebergCatalogAdapter: prefix "prod_catalog" → カタログ名
  ▼ [06] IcebergCatalogHandler.authorize...()
  │
  ├─ [04] Resolver: "prod_catalog"/"sales"/"raw"/"events" を実体化
  │        multi-pass でキャッシュの陳腐化を吸収
  │        各エンティティの grantRecordsAsSecurable を同梱
  │        principal → principal_role → catalog_role を辿り activatedGranteeIds を作る
  │
  ├─ [04] RbacOperationSemantics: LOAD_TABLE_WITH_READ_DELEGATION → {TABLE_READ_DATA}
  │
  └─ [04] hasTransitivePrivilege:
           ROOT → CATALOG → NAMESPACE(sales) → NAMESPACE(raw) → TABLE(events)
           を親から走査。どこかで TABLE_READ_DATA (または SUPER) の grant が
           activatedGranteeIds のいずれかに向いていれば許可
  │
  ▼ [06] initializeCatalog() → LocalIcebergCatalog
  ▼ [06] doRefresh(): passthrough 解決 → entity.getMetadataLocation()
  │       ETag が If-None-Match と一致すれば 304 で終了 (ストレージアクセスなし)
  │
  ├─ [07] StorageAccessConfigProvider:
  │        エンティティ階層を遡って storageConfigInfo を発見
  │        LocationGrant [(s3://bucket/sales/raw/events, {READ, LIST})]
  │        StorageCredentialCache (残存期間の半分で失効)
  │        → AwsCredentialsStorageIntegration: IamPolicy 生成 → STS AssumeRole
  │           session tags: polaris:realm=prod, polaris:principal=alice, ...
  │
  ├─ [07] DefaultFileIOFactory → FileIO (READ, LIST スコープ)
  ▼ [06] TableMetadataParser.read(fileIO, metadataLocation)
  │
  ▼ [08] PolarisEventDispatcher: AFTER_LOAD_TABLE
  ▼
200 LoadTableResponse {
      metadata: {...},
      config: { "s3.access-key-id": ..., "s3.session-token": ...,
                "s3.session-token-expires-at-ms": ...,
                "s3.refresh-credentials-endpoint": ... }
    }
```

この後クライアントは、受け取った認証情報で S3 から直接データを読む。**Polaris はデータ経路に入らない。**

## Iceberg との対比

| 観点 | Iceberg | Polaris |
|-----|---------|---------|
| OCC の対象 | `TableMetadata` の差し替え | エンティティ行の `entity_version` |
| OCC の実装 | `TableOperations.commit(base, metadata)` を Catalog 実装が提供 | 二層 CAS (metadataLocation + entity_version) |
| リトライ | `Tasks.foreach(ops).retry(...)` | 同じ (Iceberg の `Tasks` を利用) |
| 分離レベルの目標 | serializable isolation | 同左 (永続化層の CAS で担保) |
| プランニング | クライアント側 (O(1) スキャンプランニング) | 関与しない |
| ストレージ抽象 | `FileIO` (S3FileIO 等) | `FileIO` を利用しつつ、認証情報を注入する側 |
| マルチテナンシー | REST の `{prefix}` に委ねる | Realm として全層に貫通 |
| 権限 | 仕様外 | 102 privilege の RBAC |

Iceberg 側の `TableOperations.commit` が「atomic な CAS」を SPI として要求していた
([iceberg/06-catalog-and-rest.md](../iceberg/06-catalog-and-rest.md)) 部分に、
Polaris が RDB の条件付き UPDATE という具体的な回答を与えている、という関係になる。

## 気になった点

- **`LocalIcebergCatalog` が 3,186 行**、`IcebergCatalogHandler` が 1,809 行と、中心クラスが大きい。
  ロケーション検証・FileIO 取得・コミット・イベント発火が 1 クラスに同居している
- **二層 CAS** はコード内の TODO 自身が「`doRefresh` のエンティティを直接使えば一層にできるのでは」と認めている
- **NoSQL 永続化が 39 モジュール・323 ファイル**あるが、DB 実装は inmemory と MongoDB の 2 つで、
  既定は JDBC。フレームワークとしての完成度に対して実装が追いついていない段階に見える
- `PolarisAuthorizer` に**旧 API (`authorizeOrThrow(principal, activatedEntities, op, target, secondary)`) と
  新 SPI (`resolveAuthorizationInputs` / `authorize`) が併存**しており、移行途中の状態
- 生成されるモデルと Iceberg 本家のモデルが混在するため、`org.apache.polaris.service.types` と
  `org.apache.iceberg.rest.*` のどちらを使うかがモジュールごとに違う
