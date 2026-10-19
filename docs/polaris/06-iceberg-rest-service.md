# Iceberg REST サービス

Polaris の「Iceberg REST Catalog サーバー」としての本体。
リクエストの処理は **Adapter → Handler → Catalog** の 3 層を通る。

## 3 層の役割分担

| 層 | クラス | 責務 |
|---|-------|------|
| **Adapter** | `IcebergCatalogAdapter` (775 行) | OpenAPI 生成インターフェースの実装。HTTP パラメータのパース、prefix → カタログ名の解決、Handler の生成 |
| **Handler** | `IcebergCatalogHandler` (1,809 行) / 基底 `CatalogHandler` (560 行) | **認可**。解決マニフェストを組み立て、`PolarisAuthorizer` を呼び、認可が通ってからカタログを初期化する |
| **Catalog** | `LocalIcebergCatalog` (3,186 行) | Iceberg の `Catalog` / `TableOperations` を実装。ロケーション検証、FileIO 取得、メタデータ書き込み、永続化 |

```java
public class IcebergCatalogAdapter
    implements IcebergRestCatalogApiService, IcebergRestConfigurationApiService, CatalogAdapter
```

`IcebergRestCatalogApiService` は `api/iceberg-service` が OpenAPI から生成したインターフェース ([01](01-build-and-modules.md))。
仕様との齟齬はコンパイルエラーになる。

### prefix の扱い

Iceberg REST 仕様の `/v1/{prefix}/...` の `{prefix}` は、Polaris ではカタログ名そのもの。
`CatalogPrefixParser` インターフェースで抽象化されており、既定実装 `DefaultCatalogPrefixParser` は恒等写像:

```java
public String prefixToCatalogName(String prefix) { return prefix; }
public String catalogNameToPrefix(String catalogName) { return catalogName; }
```

差し替え可能にしてあるのは、prefix にテナント情報などを埋め込むデプロイを想定してのこと。

## 認可を先に済ませる構造 (`CatalogHandler`)

`CatalogHandler` は abstract クラスで、Immutables で生成される。重要なのは処理順序の強制:

```java
protected PolarisResolutionManifest resolutionManifest = null;

/** Initialize the catalog once authorized. Called after all `authorize...` methods. */
protected abstract void initializeCatalog();
```

つまり **`authorize...` 系メソッドが成功するまでカタログは初期化されない**。
認可より先にカタログを触ることが構造的にできないようにしてある。

`authorizeBasicNamespaceOperationOrThrow(op, namespace, extraPassthroughNamespaces, extraPassthroughTableLikes, extraPassThroughPolicies)`
のようなメソッドが、

1. `newResolutionManifest()` で解決マニフェストを作る
2. 認可対象のパスを `addPath()`、参考情報を `addPassthroughPath()` で登録
3. `resolveAll()` → `PolarisAuthorizer.authorizeOrThrow()`

を一括で行う。「必須解決」と「passthrough (失敗しても続行)」の区別は [04](04-rbac-and-authorization.md) を参照。

## テーブルロードとコミットのプロトコル

`LocalIcebergCatalog` は Iceberg の `BaseMetastoreCatalog` 系の枠組みに乗り、
`BasePolarisTableOperations` (内部クラス) が `doRefresh` / `doCommit` を実装する。

### `doRefresh`

```
passthrough で解決したテーブルエンティティを取得
  ↓
entity.getMetadataLocation() を読む
  ↓ null なら refresh を無効化 (テーブルは存在しない)
FileIO を取得 (READ, LIST のみのスコープ)
  ↓
TableMetadataParser.read(fileIO, metadataLocation)
```

**認可用に静的解決したエンティティではなく、passthrough で改めて解決したものを使う**点が明示的にコメントされている。
認可時点のスナップショットは古い可能性があるため。

`refreshFromMetadataLocation` には `SHOULD_RETRY_REFRESH_PREDICATE` と
`MAX_METADATA_REFRESH_RETRIES` (feature config) が渡され、
オブジェクトストレージの一時的な read-after-write 不整合をリトライで吸収する。

### `doCommit` — 二層 CAS

Polaris のコミットは Iceberg のカタログ実装と同じく OCC だが、**2 段階の CAS**になっている
(コード内の TODO コメントが「two-layer CAS」と呼んでいる)。

```
① 前提チェック
   - base == null かつ namespace が存在しない → NoSuchNamespaceException
   - 既存エンティティのサブタイプが ICEBERG_TABLE でない → AlreadyExistsException

② ロケーション検証
   - locationsToValidate を組む
       ロケーションが変わった場合: テーブルが使う全ロケーション + 直接参照するメタデータファイル
       変わっていない場合      : 直接参照するメタデータファイルのみ
     + 次に書く metadata.json のパス
   - validateLocationsForTableLike(): 許可ロケーション配下か
   - (ロケーション変更時のみ) validateNoLocationsOverlap(): 他テーブルと重ならないか
   - (同上) validateMetadataFileInTableDir(): metadata.json がテーブルディレクトリ配下か

③ FileIO を取得 (READ, WRITE, LIST スコープ)

④ 新しい metadata.json をストレージに書く (writeNewMetadataIfRequired)

⑤ 第 1 層 CAS: 現在のエンティティの metadataLocation が
   base のもの (oldLocation) と一致するか
     不一致 & base == null    → AlreadyExistsException
     不一致 & existing == null → NotFound
     不一致 (その他)          → CommitFailedException

⑥ 第 2 層 CAS: 永続化層の updateEntityPropertiesIfNotChanged
   (entity_version / grant_records_version の CAS)  ([03])

⑦ 成功後にのみインメモリ状態を更新
   失敗時は finally で ④ で書いたファイルを削除
```

**第 1 層が「`doRefresh` と `doCommit` の間の並行変更」を、第 2 層が「第 1 層のチェックと実際の永続化の間の並行変更」を検出する**。
コード内 TODO は「`doRefresh` のエンティティを直接使えば一層にできるのでは」と示唆している。

⑦ の順序についても明示的なコメントがある — Iceberg の `BaseMetastoreTableOperations` は
永続化前にインメモリ状態を更新するが、Polaris はあえて後にしている。
先に更新して永続化が失敗すると、finally の後片付けが `newLocation` を削除する一方で
この ops インスタンスはそこを指したままになり、呼び出し元が refresh するまで壊れた参照が残るため。

リトライは Iceberg の `Tasks.foreach(ops).retry(maxCommitRetries)` (`CatalogHandlerUtils`) で行う。
回数は feature config `ICEBERG_COMMIT_MAX_RETRIES`。
**テーブル作成はリトライしない** — 「テーブルが既に存在するなら再試行は解決策ではない」というコメントどおり。

ビュー側も `BasePolarisViewOperations` が同じ構造を持つ。

## ロケーション検証

Polaris がデータファイルを触らない代わりに厳密に行っているのがロケーションの検証。3 種類ある。

### ① 許可ロケーション配下か (`LocationRestrictions.validate`)

`PolarisStorageConfigurationInfo.forEntityPath()` が、エンティティの親階層 (テーブル → namespace → カタログ) を遡って
最初に見つかったストレージ設定を返す。そこから `LocationRestrictions` を作る:

| フィールド | 意味 |
|-----------|------|
| `allowedLocations` | 許可された URI 群 (カタログ設定 + 条件付きでユーザー指定分) |
| `parentLocation` | 非 null なら「新規テーブルはこの配下でなければならない」(structured table enforcement) |

ストレージ設定が見つからない場合でも、`SUPPORTED_CATALOG_STORAGE_TYPES` に `FILE` が含まれないなら
`file:` / `http` で始まるロケーションを拒否する。

関連する feature config:

| 設定 | 効果 |
|-----|------|
| `ALLOW_UNSTRUCTURED_TABLE_LOCATION` | `parentLocation` による構造強制を外す |
| `ALLOW_EXTERNAL_TABLE_LOCATION` | 許可ロケーション外を許す |
| `ALLOW_CLIENT_SPECIFIED_TABLE_LOCATION` | クライアントによる location 指定を許す |
| `ALLOW_EXTERNAL_METADATA_FILE_LOCATION` | metadata.json がテーブルディレクトリ外にあることを許す |
| `ALLOW_WILDCARD_LOCATION` | ワイルドカードを含む許可ロケーションを許す |
| `ALLOW_INSECURE_STORAGE_TYPES` | `FILE` などの安全でないストレージ種別を許す |

既定はいずれも安全側 (禁止)。これらを緩めることは、他テーブルのデータを読めてしまう経路を開くことを意味する。

### ② 兄弟との重なりがないか (`validateNoLocationOverlap`)

2 つのテーブルが同じディレクトリを指すと、片方の purge が他方のデータを消しうる。それを禁止する。

```java
if (OPTIMIZED_SIBLING_CHECK) {
  Optional<Optional<String>> r = metaStoreManager.hasOverlappingSiblings(ctx, entity);
  if (r.isPresent()) {          // 永続化層がこのチェックをサポートしている
    if (r.get().isPresent()) throw ForbiddenException(...);
    return;
  }
}
// フォールバック: 兄弟 namespace と兄弟テーブルを全件列挙して比較
```

JDBC 実装は `location_without_scheme` 列の部分インデックスで prefix 検索できるので O(log n)、
それ以外は全件走査 ([03](03-persistence.md))。二重 `Optional` の意味がここに現れる。

feature config `ALLOW_TABLE_LOCATION_OVERLAP` / `ALLOW_NAMESPACE_LOCATION_OVERLAP` で無効化できる。

### ③ metadata.json がテーブルディレクトリ配下か (`validateMetadataFileInTableDir`)

`ALLOW_EXTERNAL_METADATA_FILE_LOCATION` が無効なら、`metadata.json` はテーブルの base location 配下でなければならない。
`registerTable` で任意のパスの `metadata.json` を登録されると、
許可ロケーション外のデータを Polaris 管理下に持ち込めてしまうため。

## ETag による条件付きロード

`loadTable` は `If-None-Match` ヘッダを解釈する。
ETag は `IcebergHttpUtil.generateETagForMetadataFileLocation(metadataLocation)` で生成される
— つまり **metadata.json のパスがそのままバージョン識別子**になる。

クライアントの ETag が一致すれば 304 を返し、`TableMetadata` の読み込み (= オブジェクトストレージへのアクセス) を丸ごと省略できる。
Iceberg のテーブルは「metadata.json のパスが変わる = 内容が変わる」なので、この対応付けは正確である。

## 冪等性 (`service/idempotency/`)

`IdempotencyKeyFilter` がリクエストの冪等キーを拾い、`IdempotencyRequestContext` に載せる。
キーは `EntityIdempotency.recordKey()` によってエンティティの `internalProperties` に
**有効期限付きのウィンドウとして記録される**。

```java
private Map<String, String> idempotencyInternalProperties(
    Map<String, String> internalProperties, @Nullable IcebergTableLikeEntity existing) {
  // internalProperties は書き込み時に丸ごと置き換わるので、既存のウィンドウを引き継ぐ
  if (existing != null) { ...prior window を put... }
  if (idempotencyRequestContext.isActive()) {
    internalProperties = EntityIdempotency.recordKey(
        internalProperties, pendingKey, pendingExpiry, Instant.now());   // 期限切れキーは purge
  }
  return internalProperties;
}
```

キーの記録がメタデータ変更と**同じ書き込みに乗る**ので、「コミットは成功したがキーの記録に失敗した」状態が生じない。

## アクセス委譲モード (`AccessDelegationMode`)

Iceberg REST 仕様の `X-Iceberg-Access-Delegation` ヘッダに対応:

| 値 | プロトコル値 |
|---|------------|
| `VENDED_CREDENTIALS` | `vended-credentials` |
| `REMOTE_SIGNING` | `remote-signing` |

`fromProtocolValuesList()` にはレガシー互換がある。仕様化前の古いクライアントは `true` という
非公式な値を送っていたので、それを `VENDED_CREDENTIALS` として受ける。
ただし `true` が他の値と併記されている場合は「仕様を知っているクライアントのはず」として `UNKNOWN` 扱いにする。

要求されたモードは `AccessDelegationModeResolver` で解決され、
`buildLoadTableResponseWithDelegationCredentials(...)` が認証情報付きのレスポンスを組み立てる ([07](07-storage-and-credential-vending.md))。

対応する認可操作も分かれている ([04](04-rbac-and-authorization.md)):

| 操作 | 必要 privilege |
|-----|---------------|
| `LOAD_TABLE` | `TABLE_READ_PROPERTIES` |
| `LOAD_TABLE_WITH_READ_DELEGATION` | `TABLE_READ_DATA` |
| `LOAD_TABLE_WITH_WRITE_DELEGATION` | `TABLE_WRITE_DATA` |

## エンドポイント一覧

### Iceberg REST (仕様準拠)

`spec/polaris-catalog-service.yaml` より:

| パス | 備考 |
|-----|------|
| `/v1/config` | カタログ設定。クライアントが最初に呼ぶ |
| `/v1/oauth/tokens` | OAuth2 トークン ([05](05-authentication.md)) |
| `/v1/{prefix}/namespaces` (+ `/{namespace}`, `/properties`) | namespace CRUD |
| `/v1/{prefix}/namespaces/{namespace}/tables` (+ `/{table}`) | テーブル CRUD |
| `/v1/{prefix}/namespaces/{namespace}/tables/{table}/credentials` | 認証情報のみの取得 |
| `/v1/{prefix}/namespaces/{namespace}/tables/{table}/metrics` | スキャン/コミットメトリクス報告 |
| `/v1/{prefix}/namespaces/{namespace}/register` | 既存 metadata.json の登録 |
| `/v1/{prefix}/tables/rename`, `/views/rename` | リネーム |
| `/v1/{prefix}/transactions/commit` | マルチテーブルコミット |
| `/v1/{prefix}/namespaces/{namespace}/views` (+ `/{view}`, `/register-view`) | ビュー |

Iceberg 側の仕様にある `/plan` (サーバーサイドスキャンプランニング) は、
`polaris-catalog-service.yaml` 内で **コメントアウトされた状態**で置かれている
(`# /v1/{prefix}/namespaces/{namespace}/tables/{table}/plan:` と `.../plan/{plan-id}:`)。
将来対応の予定地として枠だけ確保してある形で、現時点では未実装。

### Polaris 拡張

| パス | 内容 |
|-----|------|
| `/v1/{prefix}/namespaces/{namespace}/tables/{table}/notifications` | 外部テーブル更新通知 (`SupportsNotifications`) |
| `/polaris/v1/{prefix}/namespaces/{namespace}/generic-tables` (+ `/{generic-table}`) | Generic Table ([09](09-federation-and-extensions.md)) |
| `/polaris/v1/{prefix}/namespaces/{namespace}/policies` (+ `/{policy-name}`, `/mappings`) | Policy |
| `/polaris/v1/{prefix}/applicable-policies` | 適用可能ポリシーの照会 |
| `/polaris/v1/{prefix}/namespaces/{namespace}/semantic-models` (+ `/{name}`) | セマンティックモデル |

### Management API (`spec/polaris-management-service.yml`)

Iceberg 仕様外の管理面。`PolarisServiceImpl` / `PolarisAdminService` が実装:

| パス | 内容 |
|-----|------|
| `/catalogs` (+ `/{catalogName}`) | カタログ CRUD |
| `/principals` (+ `/{name}`, `/rotate`, `/reset`, `/principal-roles`) | プリンシパル CRUD、資格情報ローテーション |
| `/principal-roles` (+ `/{name}`, `/principals`, `/catalog-roles/{catalogName}`) | プリンシパルロール |
| `/catalogs/{catalogName}/catalog-roles` (+ `/{name}`, `/principal-roles`, `/grants`) | カタログロールと grant |

`/grants` が RBAC の grant record ([04](04-rbac-and-authorization.md)) を操作するエンドポイント。

## イベントディスパッチ

各操作の前後で `PolarisEventDispatcher.dispatch()` が呼ばれる。
`hasListeners(eventType)` で先にガードしてからイベントオブジェクトを構築しているのは、
リスナが無いときにアロケーションを避けるため。詳細は [08](08-runtime-quarkus.md)。

`IcebergRestCatalogEventServiceDelegator` (1,140 行) は `IcebergRestCatalogApiService` の
デコレータ実装で、全メソッドを before/after イベント発火で包む。
Adapter 本体にイベント発火コードを混ぜない構造になっている。
