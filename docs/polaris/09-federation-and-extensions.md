# フェデレーションと拡張機能

「Iceberg REST Catalog」以外の機能群。いずれも feature config で無効化でき、既定では控えめに設定されている。

## カタログフェデレーション

Polaris のカタログには 2 種類ある (`CatalogEntity` の `catalogType` 内部プロパティ):

| 種別 | 説明 |
|-----|------|
| `INTERNAL` | Polaris 自身がメタデータを保持する通常のカタログ |
| `EXTERNAL` | **外部のカタログへのプロキシ**。メタデータの正本は外部にある |

`EXTERNAL` カタログには `connectionConfigInfo` (接続設定) が付き、
Polaris はリクエストを外部カタログへ委譲しつつ、自身の RBAC を手前で適用する。

feature config `ENABLE_CATALOG_FEDERATION` で有効化。
`ENABLE_SUB_CATALOG_RBAC_FOR_FEDERATED_CATALOGS` で、フェデレーション先のオブジェクトに対して
Polaris の RBAC を効かせるかを制御する。

### 接続種別 (`ConnectionType`)

| 種別 | code | ファクトリ識別子 | モジュール |
|-----|------|----------------|-----------|
| `ICEBERG_REST` | 1 | `iceberg_rest` | `runtime/service` (`IcebergRESTFederatedCatalogFactory`) |
| `HADOOP` | 2 | `hadoop` | `extensions/federation/hadoop` |
| `HIVE` | 3 | `hive` | `extensions/federation/hive` |
| `BIGQUERY` | 4 | `bigquery` | `extensions/federation/bigquery` |

許可される種別は feature config `SUPPORTED_CATALOG_CONNECTION_TYPES` (既定 `["ICEBERG_REST"]`) で制限される。
Hive や BigQuery を使うには明示的に有効化が必要。

### `FederatedCatalogFactory`

```java
Catalog             createCatalog(ConnectionConfigInfoDpo, PolarisCredentialManager, Map<String,String>);
GenericTableCatalog createGenericCatalog(ConnectionConfigInfoDpo, PolarisCredentialManager, Map<String,String>);
```

`catalogProperties` は `ExternalCatalog` エンティティ側の追加プロパティ (プロキシ設定、タイムアウト等) で、
接続設定より**低い優先度**でマージされる。

### 接続設定 (`ConnectionConfigInfoDpo`)

Jackson の `@JsonSubTypes` で接続種別ごとにサブクラスへ分岐する DPO (data persistence object)。
`internalProperties` に JSON で保存される ([02](02-entity-model.md))。

保持するのは URI、認証パラメータ、**サービス識別子**、追加プロパティ。

### 外部への認証 (`connection/AuthenticationType`)

| 種別 | DPO |
|-----|-----|
| `OAUTH` | `OAuthClientCredentialsParametersDpo` |
| `BEARER` | `BearerAuthenticationParametersDpo` |
| `IMPLICIT` | `ImplicitAuthenticationParametersDpo` |
| `SIGV4` | `SigV4AuthenticationParametersDpo` (AWS SigV4 署名) |
| `GCP` | `GcpAuthenticationParametersDpo` |

### サービス識別 (`core/identity/`)

Polaris が**外部システムに対して自分を名乗る**ときの識別子。ユーザーの認証情報 ([05](05-authentication.md)) とは別物。

```java
public enum ServiceIdentityType { NULL_TYPE(0), AWS_IAM(1) }
```

`ConnectionCredentialVendor` が「Polaris 管理のサービス識別子」と「ユーザーが設定した認証パラメータ」を
組み合わせて外部接続用の認証情報を作る:

> The connection configuration contains both the Polaris-managed service identity (e.g., an IAM user)
> and user-configured authentication settings (e.g., which role to assume, signing region).

つまり「Polaris の IAM ユーザーが、ユーザーの指定したロールを AssumeRole する」という形で
外部への接続認証が組み立てられる。テーブルデータ向けの credential vending ([07](07-storage-and-credential-vending.md)) と
似た構造だが、方向が逆 (Polaris → 外部) である点が違う。

管理は `PolarisCredentialManager` (`polaris.credential-manager.type`)。

### passthrough facade

`Resolver.getIsPassthroughFacade()` ([04](04-rbac-and-authorization.md)) が true になるカタログでは、
カタログ内のパス解決の厳密さが緩められる。Polaris 側に namespace/table のエンティティが存在しなくても、
外部カタログには存在しうるため。

## Generic Table

Iceberg でないテーブル (Delta Lake, Hudi, Paimon 等) も同じカタログ・同じ RBAC で扱うための仕組み。
feature config `ENABLE_GENERIC_TABLES`。

- エンティティは `TABLE_LIKE` 型の `GENERIC_TABLE` サブタイプ ([02](02-entity-model.md))
- `ICEBERG_TABLE` / `ICEBERG_VIEW` と**同じ名前空間を共有する** (`ANY_SUBTYPE` での存在チェック)
- privilege は `TABLE_*` 系を流用 — `PolarisPrivilege` の securableSubTypes に
  `List.of(ICEBERG_TABLE, GENERIC_TABLE)` と併記されている

API は Polaris 独自 (`spec/polaris-catalog-apis/generic-tables-api.yaml`):

```
/polaris/v1/{prefix}/namespaces/{namespace}/generic-tables
/polaris/v1/{prefix}/namespaces/{namespace}/generic-tables/{generic-table}
```

実装は `service/catalog/generic/`:

| クラス | 役割 |
|-------|------|
| `GenericTableCatalogAdapter` | REST 実装 |
| `GenericTableCatalogHandler` | 認可 |
| `PolarisGenericTableCatalog` | `GenericTableCatalog` (polaris-core) の実装 |

`GenericTableEntity` が保持するのはフォーマット名 (`delta` / `hudi` / `paimon` など)、
base location、doc、プロパティ。**Iceberg のような metadata.json ポインタは持たない** —
テーブルの実体の解釈は各エンジンに任せる。

## Policy

テーブル・namespace・カタログにポリシーをアタッチする仕組み。feature config `ENABLE_POLICY_STORE`。

### 事前定義ポリシー種別 (`PredefinedPolicyTypes`)

| 種別 | 名前 | 継承 |
|-----|------|------|
| `DATA_COMPACTION` | `system.data-compaction` | yes |
| `METADATA_COMPACTION` | `system.metadata-compaction` | yes |
| `ORPHAN_FILE_REMOVAL` | `system.orphan-file-removal` | yes |
| `SNAPSHOT_EXPIRY` | `system.snapshot-expiry` | yes |

いずれも **inheritable** で、カタログや namespace に付けたポリシーが配下のテーブルに継承される。
内容は Polaris が実行するわけではなく、外部のメンテナンスジョブが `applicable-policies` API で
「このテーブルにはどのポリシーが適用されるか」を問い合わせて使う、という前提の設計。

### 実装

| 要素 | 場所 |
|-----|------|
| エンティティ | `POLICY` 型 (`NAMESPACE` 配下) + `PolicyEntity` |
| アタッチ関係 | `policy_mapping_record` テーブル ([03](03-persistence.md)) |
| SPI | `PolicyMappingPersistence` (`BasePersistence` が継承) |
| マネージャ | `PolarisPolicyMappingManager` (`PolarisMetaStoreManager` が継承) |
| 検証 | `policy/validator/` — `PolicyValidator` / `PolicyValidators`、`maintenance/` に種別別の検証 |
| 内容 | `policy/content/` — JSON スキーマに沿った内容の型 |
| 権限 | `POLICY_{CREATE,READ,WRITE,DROP,LIST,ATTACH,DETACH}` と `{CATALOG,NAMESPACE,TABLE}_{ATTACH,DETACH}_POLICY` ([04](04-rbac-and-authorization.md)) |

アタッチ側とアタッチされる側の両方に権限が要る点が `PolicyAttachmentAuthorizationIntent` に表れている
(target = policy, secondary = attachedTo)。

## セマンティックモデル

`extensions/semantic-models`。feature config `ENABLE_SEMANTIC_MODELS`。
API は `spec/polaris-catalog-apis/semantic-models-api.yaml`:

```
/polaris/v1/{prefix}/namespaces/{namespace}/semantic-models
/polaris/v1/{prefix}/namespaces/{namespace}/semantic-models/{semantic-model-name}
```

BI ツール等が使うメトリクス・ディメンジョン定義をカタログに載せる機能。

## 通知 (`SupportsNotifications`)

```
/v1/{prefix}/namespaces/{namespace}/tables/{table}/notifications
```

外部システム (例: オブジェクトストレージのイベント) から
「このテーブルの metadata.json が更新された」と Polaris に伝えるためのエンドポイント。
`EXTERNAL` カタログで、Polaris を経由しない書き込みが行われる構成を想定している。
認可操作は `SEND_NOTIFICATIONS`。

## メトリクスレポート (`extensions/metrics-reports`)

Iceberg クライアントは `/v1/{prefix}/namespaces/{ns}/tables/{t}/metrics` に
スキャン・コミットのメトリクスを報告する。Polaris はこれを永続化できる。

| モジュール | 内容 |
|-----------|------|
| `polaris-extensions-metrics-reports` (`base`) | `NoOpMetricsReporter` (既定) |
| `polaris-extensions-metrics-reports-jdbc` | `PersistingMetricsReporter` — `scan_metrics_report` / `commit_metrics_report` テーブルへ ([03](03-persistence.md)) |

設定は `polaris.iceberg-metrics.reporting.type`。
テーブルには `otel_trace_id` / `otel_span_id` 列があり、credential vending のセッションタグ
([07](07-storage-and-credential-vending.md)) と同じトレース ID で突き合わせられる。

## Spark プラグイン (`plugins/spark/`)

Polaris が提供する Spark 側のカタログ実装。**`polaris-core` に依存せず、Iceberg + Spark にのみ依存する**
独立したクライアントライブラリ。

対象バージョン (`spark-scala.properties`):

| Spark | Scala |
|-------|-------|
| 3.5 | 2.12, 2.13 |
| 4.0 | 2.13 |

### 何のためにあるか

Iceberg の `SparkCatalog` をそのまま使えば Iceberg テーブルは扱えるが、**Generic Table は扱えない**。
Polaris の `SparkCatalog` は 1 つのカタログ名で Iceberg テーブルと Delta/Hudi/Paimon テーブルを
同時に扱えるようにする。

```java
public class SparkCatalog extends BasePolarisSparkCatalog
    implements ViewCatalog, SupportsReplaceView

public abstract class BasePolarisSparkCatalog
    implements StagingTableCatalog, TableCatalog, SupportsNamespaces
```

`createTable` / `loadTable` の中で `TABLE_PROVIDER_KEY` プロパティを見て振り分ける:

```java
String provider = properties.get(PolarisCatalogUtils.TABLE_PROVIDER_KEY);
if (PolarisCatalogUtils.useIceberg(provider))      → Iceberg の SparkCatalog に委譲
else if (PolarisCatalogUtils.useDelta(provider))   → DeltaHelper 経由で Delta のカタログへ
else if (PolarisCatalogUtils.useHudi(provider))    → HudiHelper
else if (PolarisCatalogUtils.usePaimon(provider))  → PaimonHelper
```

### `*Helper` のリフレクション

`DeltaHelper` / `HudiHelper` / `PaimonHelper` は、対象フォーマットのカタログ実装を
**実行時にリフレクションで読み込んで設定する**。Polaris のビルドがそれらに依存しないため。

`DeltaHelper` のコメントによれば、Delta Catalog を Unity Catalog と同じ挙動になるよう構成する。
Delta のバージョンによってフィールドがどのクラスに宣言されるかが違う (4.1 未満は具象 `DeltaCatalog`、
以降は変更) ため、**クラス階層を辿ってフィールド名のサフィックスで照合する**という防御的な実装になっている。
Scala が生成するバッキングフィールド名がバージョンで揺れるための対処。

差し替えは `delta-catalog-impl` オプションで可能。

### REST クライアント部分

`PolarisRESTCatalog` と `spark/rest/` の DTO 群 (`CreateGenericTableRequest`,
`LoadGenericTableResponse`, `ListGenericTablesResponse` 等) が、
Polaris 独自の Generic Table API を叩くクライアント実装。
認証セッションは Iceberg の `SparkCatalog` から取り出して共有する
(`PolarisCatalogUtils.getAuthSession`) ので、トークンの二重管理にならない。

統合テストは `runtime/spark-tests` (サーバー側から見た E2E) と
`plugins/spark/v3.5/regtests/` (クライアント側の回帰テスト)。

## OPA / Ranger 認可拡張

[04](04-rbac-and-authorization.md) を参照。`extensions/auth/{opa,ranger}`。

## Kafka イベント拡張

`extensions/events/kafka`。`PolarisEventListener` の実装としてイベントを Kafka トピックへ送出する
([08](08-runtime-quarkus.md))。
