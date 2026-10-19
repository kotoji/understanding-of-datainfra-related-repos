# Quarkus ランタイム

`polaris-core` が Quarkus を知らない純粋ライブラリなのに対し、`runtime/service` はほぼすべてが CDI 配線である。
差し替え可能性のほとんどはこの層で実現されている。

## 差し替えパターン: `@Identifier`

Polaris のコンポーネント差し替えはほぼ一律で以下の形をとる:

```java
@ApplicationScoped
@Identifier("default")                      // ← 設定値と対応する識別子
public class DefaultCatalogPrefixParser implements CatalogPrefixParser { ... }
```

設定 `polaris.<component>.type = default` で選択される。主要な差し替え点:

| 設定キー | 既定 | 差し替え対象 |
|---------|------|------------|
| `polaris.realm-context.type` | `default` | `RealmContextResolver` ([05](05-authentication.md)) |
| `polaris.authentication.type` | `internal` | 認証モード (`internal`/`external`/`mixed`) |
| `polaris.authentication.authenticator.type` | `default` | `Authenticator` |
| `polaris.authentication.token-broker.type` | `rsa-key-pair` | `TokenBrokerFactory` |
| `polaris.authentication.token-service.type` | `default` | OAuth2 サービス (`disabled` もある) |
| `polaris.oidc.principal-mapper.type` | `default` | `PrincipalMapper` |
| `polaris.oidc.principal-roles-mapper.type` | `default` | `PrincipalRolesMapper` |
| `polaris.authorization.type` | (RBAC) | `PolarisAuthorizerFactory` — OPA / Ranger ([04](04-rbac-and-authorization.md)) |
| `polaris.persistence.type` | `in-memory` | `MetaStoreManagerFactory` ([03](03-persistence.md)) |
| `polaris.secrets-manager.type` | `in-memory` | シークレット管理 |
| `polaris.file-io.type` | `default` | `FileIOFactory` ([07](07-storage-and-credential-vending.md)) |
| `polaris.credential-manager.type` | `default` | 外部接続用の認証情報管理 ([09](09-federation-and-extensions.md)) |
| `polaris.rate-limiter.filter.type` | `no-op` | レート制限 |
| `polaris.iceberg-metrics.reporting.type` | `default` | メトリクスレポートの扱い |
| `polaris.event-listener.types` | - | イベントリスナ (複数指定可。単数形 `.type` は 1.5.0 で非推奨) |

**既定値が本番向きでないもの**が混ざっている点に注意 (`persistence.type=in-memory`,
`secrets-manager.type=in-memory`, `rate-limiter=no-op`)。後述の production readiness check がこれを検出する。

CDI の配線は `runtime/service/.../service/config/ServiceProducers.java` に集約されている。

## 設定システム

Polaris の設定は 2 段構えになっている。

### ① Quarkus / SmallRye Config (デプロイ設定)

`application.properties` + 環境変数。`@ConfigMapping` インターフェースで型付けする。

```java
@ConfigMapping(prefix = "polaris.authentication")
public interface AuthenticationConfiguration {
  String DEFAULT_REALM_KEY = "<default>";

  @WithParentName
  @WithUnnamedKey(DEFAULT_REALM_KEY)
  @WithDefaults
  Map<String, AuthenticationRealmConfiguration> realms();

  default AuthenticationRealmConfiguration forRealm(String realmIdentifier) {
    return realms().containsKey(realmIdentifier)
        ? realms().get(realmIdentifier) : realms().get(DEFAULT_REALM_KEY);
  }
}
```

`@WithUnnamedKey("<default>")` により `polaris.authentication.type=internal` は既定 realm の設定に、
`polaris.authentication.prod.type=external` は realm `prod` の設定になる。
**この realm 別上書きパターンが設定全体で繰り返し使われている**。

### ② `PolarisConfiguration` / `RealmConfig` (機能フラグ)

`polaris-core` 側の機能フラグ機構。`FeatureConfiguration` に 59 個の定数が宣言されている。

```java
public interface RealmConfig {
  <T> T getConfig(PolarisConfiguration<T> config);
  // カタログのプロパティで上書きできるバリアントもある
}
```

`RealmConfigImpl` が `PolarisConfigurationStore` を通して値を解決する。優先順位は
**カタログのプロパティ > realm 別設定 > 全体設定 > コード上の既定値**。

`BehaviorChangeConfiguration` は「振る舞いを変える (機能追加ではない) フラグ」を分けたもので、
バージョンアップ時の互換性維持に使う。設定側も `FeaturesConfiguration` /
`BehaviorChangesConfiguration` (と `Raw*` / `Resolved*` の対) で分かれている。

`tools/config-docs/` が `@ConfigDocs` アノテーションから設定リファレンスドキュメントを生成する。

### 主要な feature config (抜粋)

分類は [06](06-iceberg-rest-service.md) / [07](07-storage-and-credential-vending.md) にも分散しているが、
機能の有効化系をここにまとめる:

| 設定 | 内容 |
|-----|------|
| `ENABLE_GENERIC_TABLES` | Generic Table API ([09](09-federation-and-extensions.md)) |
| `ENABLE_POLICY_STORE` | Policy API |
| `ENABLE_SEMANTIC_MODELS` | セマンティックモデル API |
| `ENABLE_CATALOG_FEDERATION` | カタログフェデレーション |
| `ENABLE_SUB_CATALOG_RBAC_FOR_FEDERATED_CATALOGS` | フェデレーション先に対する RBAC の適用 |
| `ENABLE_FINE_GRAINED_UPDATE_TABLE_PRIVILEGES` | 細粒度テーブル更新権限 ([04](04-rbac-and-authorization.md)) |
| `ENABLE_CREDENTIAL_RESET` | 管理者による資格情報リセット |
| `SUPPORTED_CATALOG_STORAGE_TYPES` | 許可するストレージ種別 (既定 `["S3","GCS","AZURE"]`) |
| `SUPPORTED_CATALOG_CONNECTION_TYPES` | 許可する外部接続種別 (既定 `["ICEBERG_REST"]`) |
| `ICEBERG_ROLLBACK_COMPACTION_ON_CONFLICTS` | 競合時にコンパクションをロールバック |
| `CLEANUP_ON_NAMESPACE_DROP` / `CLEANUP_ON_CATALOG_DROP` / `DROP_WITH_PURGE_ENABLED` | 削除時のクリーンアップ |
| `POLARIS_TASK_TIMEOUT_MILLIS` | 非同期タスクのタイムアウト |
| `ENTITY_CACHE_WEIGHER_TARGET` | エンティティキャッシュ上限 |

## Production readiness check

Polaris 独特の仕組み。起動時に「この設定は本番で使ってよいか」を検査する。

```java
@PolarisImmutable
public interface ProductionReadinessCheck {
  ProductionReadinessCheck OK = ...;
  interface Error {
    static Error of(String message, String offendingProperty);        // 警告
    static Error ofSevere(String message, String offendingProperty);  // 重大
    String message(); String offendingProperty(); boolean severe();
  }
}
```

各モジュールが `@Produces ProductionReadinessCheck` の CDI プロデューサを提供し、
`ProductionReadinessChecks` が `Instance<ProductionReadinessCheck>` で全部集めて評価する。

- 通常の `Error` → WARN ログ
- `severe` な `Error` → ERROR ログ + **起動失敗**

ただし `polaris.readiness.ignore-severe-issues=true` を明示すれば起動を許す。
そのときのログには「ユーザーがリスク (denial-of-service 等) を明示的に受け入れた」と記録される。

実装例: `RelationalJdbcProductionReadinessChecks` (JDBC 設定)、`OpaProductionReadinessChecks` (OPA 設定)、
`ProductionReadinessChecks.checkUserPrincipalMetricTag` (メトリクスタグに principal 名を入れるとカーディナリティが爆発する、など)。

## admin ツール (`runtime/admin`)

Picocli ベースの CLI。サーバーを起動せずに永続化層だけを立ち上げる。

```
polaris-admin-tool.jar
  ├── bootstrap   realm を初期化 (root プリンシパルと資格情報を作る)
  ├── purge       realm のデータを全削除
  └── nosql       NoSQL 永続化向けサブコマンド群
```

`bootstrap` の root 資格情報は「標準入力から与える」「ファイルから読む」の 2 択で、
Picocli の `@ArgGroup(multiplicity = "1")` により**どちらか必須**として強制される。
資格情報をコマンドライン引数で渡す経路を作らないための設計。

実処理は `MetaStoreManagerFactory.bootstrapRealms(BootstrapOptions)` / `purgeRealms(...)` ([03](03-persistence.md))。

## 非同期タスク (`service/task/`)

テーブルやビューを削除しても、データファイルの削除は同期的には行わない。
`TASK` エンティティ ([02](02-entity-model.md)) をキューとして使い、バックグラウンドで処理する。

```java
public enum AsyncTaskType {
  ENTITY_CLEANUP_SCHEDULER(1),   // エンティティ削除に伴うクリーンアップの起点
  MANIFEST_FILE_CLEANUP(2),      // マニフェストが参照するデータファイルの削除
  BATCH_FILE_CLEANUP(3);         // ファイル群の一括削除
}
```

`TaskExecutorImpl` が `@Startup` でハンドラを登録する:

| ハンドラ | 役割 |
|---------|------|
| `TableCleanupTaskHandler` | 削除されたテーブル/ビューの `metadata.json` を読み、マニフェスト単位の後続タスクを積む |
| `ManifestFileCleanupTaskHandler` | マニフェストを読んでデータファイルを削除。**仮想スレッド** (`newVirtualThreadPerTaskExecutor`) で並列実行 |
| `BatchFileCleanupTaskHandler` | ファイルのバッチ削除。同じく仮想スレッド |

タスクは「テーブル 1 個 → マニフェスト N 個 → ファイル M 個」と段階的に展開される。
巨大テーブルの削除で 1 タスクが何時間も走らないようにするための分割。

`TaskFileIOSupplier` が各タスク用の FileIO を供給する。
タスクは HTTP リクエストの外で走るので、`RealmContextHolder` / `PolarisPrincipalHolder` に
コンテキストを注入して実行する。

エラー時は `TASK_RETRY_DELAY` (1 秒) 後にリトライ。
`@Identifier("task-error-handler")` の `TriConsumer` を CDI で提供すればエラー処理を差し込める。

## イベント (`service/events/`)

`PolarisEventType` は `BEFORE_*` / `AFTER_*` の対で定義され、カテゴリ (CATALOG, CATALOG_ROLE, PRINCIPAL, ...) ごとに
100 番刻みのコードを持つ。

```java
BEFORE_CREATE_CATALOG(100, Category.CATALOG),
AFTER_CREATE_CATALOG(101, Category.CATALOG),
...
```

発火は `PolarisEventDispatcher.dispatch()`。呼び出し側は必ず `hasListeners(type)` でガードしてから
イベントオブジェクトを組み立てる ([06](06-iceberg-rest-service.md))。

REST 層では、Adapter 本体にイベント発火を混ぜず**デコレータ**で包む方式をとる:

- `IcebergRestCatalogEventServiceDelegator` (1,140 行)
- `CatalogGenericTableEventServiceDelegator`
- `PolarisCatalogsEventServiceDelegator` / `PolarisPrincipalsEventServiceDelegator` / `PolarisPrincipalRolesEventServiceDelegator`

### リスナ実装

| リスナ | 出力先 |
|-------|-------|
| `PolarisPersistenceEventListener` | `events` テーブルへ永続化 ([03](03-persistence.md)) |
| `InMemoryBufferEventListener` | インメモリバッファ (テスト・デバッグ) |
| `OpenTelemetryEventListener` | OpenTelemetry |
| `AwsCloudWatchEventListener` | AWS CloudWatch Logs (JSON) |
| Kafka (`extensions/events/kafka`) | Kafka トピック |

`EventSanitizer` (`DefaultEventSanitizer`) がイベントから機微情報を落とす。
`RawEventAccess` は生イベントへのアクセスを分離したインターフェースで、
サニタイズ前のデータを見るリスナを限定できる。

`FallbackEventListenerTypeConfigInterceptor` は設定インターセプタで、
未設定時のリスナ種別を補う。

## 観測性

| 機能 | 実装 |
|------|------|
| メトリクス | Micrometer + Prometheus (`quarkus.micrometer.export.prometheus.enabled=true`)。管理ポート 8182 |
| トレース | OpenTelemetry (`service/tracing/`)。既定は無効 (`quarkus.otel.sdk.disabled=true`) |
| ログ | `service/logging/`。**MDC に `requestId` / `realmId` / `traceId` / `spanId` を注入** |
| 構造化ログ | `StructuredLogKeys` (polaris-core) が SLF4J の `addKeyValue` 用のキーを定数化 |

ログフォーマット:

```
%d{...} %-5p [%c{3.}] [%X{requestId},%X{realmId}] [%X{traceId},%X{parentId},%X{spanId},%X{sampled}] (%t) %s%e%n
```

realm ID が全ログ行に入るので、マルチテナント環境でテナント単位のログ抽出ができる。

`polaris.metrics.tags.*` でメトリクスのタグを追加できるが、
principal 名や realm ID をタグにするとカーディナリティが爆発するため
production readiness check が警告する。

## レート制限 (`service/ratelimiter/`)

トークンバケット方式。既定は `no-op` (無効)。

| クラス | 役割 |
|-------|------|
| `RateLimiterFilter` | JAX-RS フィルタ (priority `Priorities.USER` = 認証の後) |
| `RealmTokenBucketRateLimiter` | **realm ごとに**バケットを持つ実装 |
| `TokenBucket` / `DefaultTokenBucketFactory` | バケット本体 |

realm 単位なので、あるテナントの過剰なリクエストが他テナントに波及しない。
設定は `polaris.rate-limiter.token-bucket.requests-per-second` (既定 9999)。

## その他のランタイム設定 (`runtime/defaults/application.properties`)

| 設定 | 値 | 意味 |
|-----|---|------|
| `quarkus.http.port` | 8181 | サービスポート |
| `quarkus.management.port` | 8182 | 管理ポート (メトリクス、ヘルスチェック) |
| `quarkus.http.auth.proactive` | `false` | **プロアクティブ認証を無効化** — 認証をエンドポイント到達時まで遅延させる |
| `quarkus.http.auth.basic` | `false` | Basic 認証は使わない |
| `quarkus.http.limits.max-body-size` | 10240K | リクエストボディ上限 |
| `quarkus.http.body.handle-file-uploads` | `false` | ファイルアップロードを扱わない |
| `quarkus.datasource.db-kind` | `postgresql` | JDBC 既定 |
| `quarkus.config.mapping.validate-unknown` | `true` | **未知の設定キーをエラーにする** (タイポ検出) |
| `quarkus.fault-tolerance.global.timeout.enabled` | `false` | グローバルタイムアウト無効 |
| `quarkus.arc.ignored-split-packages` | (多数) | 複数モジュールにまたがるパッケージを CDI が許容するための宣言 |

`ignored-split-packages` のリストが長いのは、`org.apache.polaris.service.*` の各サブパッケージが
`runtime/service` と生成コードモジュール (`api/*`) の両方に現れるため。

## デプロイ

| 成果物 | 場所 |
|-------|------|
| Dockerfile | `runtime/server/src/main/docker/Dockerfile.jvm` |
| Helm チャート | `helm/polaris/` |
| 配布物パッケージング | `runtime/distribution/` |
| getting-started | `getting-started/{quickstart,jdbc,minio,keycloak,spark,flink,trino,telemetry,ceph,ozone,rustfs,cockroachdb}/` |
| 回帰テスト | `regtests/` (Docker Compose + pyspark / spark-sql / CLI / OAuth / フェデレーション) |

`getting-started/` にストレージ (MinIO, Ceph, Ozone, RustFS)、IdP (Keycloak)、
クエリエンジン (Spark, Flink, Trino) ごとの Docker Compose 一式が揃っている。
「エンジン非依存」を掲げるプロジェクトが実際にそれを検証している形。
