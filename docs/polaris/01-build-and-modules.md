# ビルドシステムとモジュール構成

## ビルドシステム

- **ビルドツール**: Gradle 9.7.0 (wrapper)、Kotlin DSL
- **Java 要件**: **Java 21 必須** (`settings.gradle.kts` で `JavaVersion.VERSION_21` 未満なら `GradleException`)
- **バージョンカタログ**: `gradle/libs.versions.toml`
- **モジュール定義**: `include(...)` を直接書かず、`gradle/projects.main.properties` (`プロジェクト名=ディレクトリ` の properties ファイル) を読んで動的に `include` する
- **included build**: `build-logic` (共有ビルドロジック) と `gradle/server-test-runner` の 2 つ

`settings.gradle.kts` には Polaris らしいガードが 2 つある:

- **CI では Gradle configuration cache を禁止** — configuration cache がビルド設定の状態をディスクに永続化し、そこからシークレットが漏れうるため、`CI` 環境変数がある状態で configuration cache が要求されたらビルドを失敗させる
- **IntelliJ 使用時は Spark プラグインの重複プロジェクトをスキップ** — 同じ `projectDir` を指す複数プロジェクト (Scala バージョン違い) が IDE のプロジェクト同期を壊すため、`idea.active` システムプロパティが立っていたら最初の 1 個だけ include する

## build-logic

`build-logic/src/main/kotlin/` に convention plugin 群を置き、各モジュールはそれを `plugins { id("polaris-...") }` で適用する。

| Plugin | 役割 |
|--------|------|
| `polaris-root` | ルートプロジェクト設定 |
| `polaris-base` / `polaris-java` | 共通設定 / Java コンパイル、Checkstyle、Error Prone |
| `polaris-client` | クライアントライブラリ (公開される成果物) 向け設定 |
| `polaris-runtime` / `polaris-server` | Quarkus ランタイム / サーバー向け設定 |
| `polaris-spotless` | コードフォーマット (google-java-format) |
| `polaris-shadow-jar` | shade JAR 生成 |
| `polaris-bom` / `polaris-license-report` | BOM 生成 / ライセンスレポート集約 |
| `polaris-reproducible` | 再現可能ビルド (タイムスタンプ固定等) |

`build-logic/src/main/kotlin/asf/`, `licenses/`, `copiedcode/` には ASF リリース要件 (LICENSE/NOTICE の整合、他プロジェクトからコピーしたコードの由来管理) 向けのタスクがある。

## OpenAPI からのコード生成

Polaris の REST インターフェースは **手書きしない**。`spec/` の OpenAPI 定義から `jaxrs-resteasy` ジェネレータで JAX-RS インターフェースと DTO を生成し、
`runtime/service` 側がそれを `implements` する。

### 仕様ファイル (`spec/`)

| ファイル | 内容 |
|---------|------|
| `iceberg-rest-catalog-open-api.yaml` | Iceberg 本家の REST Catalog 仕様 (取り込み) |
| `polaris-catalog-service.yaml` | 上記を土台に Polaris 拡張を足したカタログ API |
| `polaris-management-service.yml` | Polaris 独自の管理 API (カタログ/プリンシパル/ロール/グラント CRUD) |
| `polaris-catalog-apis/*.yaml` | 拡張 API 断片: `generic-tables-api`, `policy-apis`, `semantic-models-api`, `notifications-api`, `oauth-tokens-api` |
| `generated/bundled-polaris-catalog-service.yaml` | `$ref` を解決して 1 ファイルに束ねた成果物 (リポジトリにコミットされている) |

### 生成の分割 (`api/` 配下 4 モジュール)

同じ仕様ファイルから、`globalProperties` の `apis` / `models` フィルタで生成対象を切り分けている。
これにより「モデルは 1 箇所だけで生成され、API インターフェースは用途別モジュールに分かれる」という構成になる。

| モジュール | inputSpec | 生成する API | 生成するモデル |
|-----------|-----------|-------------|---------------|
| `polaris-api-management-model` | management | (なし) | `org.apache.polaris.core.admin.model` |
| `polaris-api-management-service` | management | `org.apache.polaris.service.admin.api` | (なし) |
| `polaris-api-iceberg-service` | catalog | `CatalogApi`, `ConfigurationApi`, `OAuth2Api` | (なし: Iceberg 本家のクラスを使う) |
| `polaris-api-catalog-service` | catalog | `GenericTableApi`, `PolicyApi` | `org.apache.polaris.service.types` |

`api/iceberg-service` がモデルを生成しないのは、リクエスト/レスポンス型に Iceberg 本家の `org.apache.iceberg.rest.requests.*` /
`responses.*` をそのまま使うため。仕様の齟齬がコンパイルエラーとして出る。

生成テンプレートは各モジュールの `src/main/resources/templates/` (`templateDir`) で上書きしている。

## モジュール一覧

以下のカテゴリ分けは説明のためのもので、Gradle 上の階層ではない。

### コア (1)

| モジュール | ディレクトリ | Java (main) | 役割 |
|-----------|-------------|------------|------|
| `polaris-core` | `polaris-core` | 247 | エンティティ定義、権限モデル、永続化 API、Resolver、ストレージ抽象 |

`polaris-core` は **Quarkus に依存しない** 純粋な Java ライブラリ。CDI アノテーションもここにはほぼ無い。

### API (4)

前節の表のとおり。`polaris-api-*`。

### ランタイム (7)

| モジュール | ディレクトリ | Java (main) | 役割 |
|-----------|-------------|------------|------|
| `polaris-runtime-service` | `runtime/service` | 208 | サービス本体。REST 実装、認証認可の配線、タスク、イベント |
| `polaris-server` | `runtime/server` | 1 | Quarkus アプリケーションのエントリポイント (実質は依存の束ね役 + Dockerfile) |
| `polaris-admin` | `runtime/admin` | 13 | Picocli ベースの admin CLI (bootstrap / purge) |
| `polaris-runtime-defaults` | `runtime/defaults` | 1 | `application.properties` の既定値のみを持つモジュール |
| `polaris-runtime-common` | `runtime/common` | - | ランタイム共通ユーティリティ |
| `polaris-runtime-test-common` | `runtime/test-common` | - | テスト共通 |
| `polaris-distribution` | `runtime/distribution` | - | 配布物パッケージング |

`polaris-server` と `polaris-runtime-defaults` に Java ファイルが 1 個ずつしかないのは、中身が `Readme.java`
(モジュールの目的を書いた javadoc 専用クラス) だけだから。実体は `build.gradle.kts` の依存宣言と `application.properties`。

### 永続化 (1 + 39)

| モジュール | ディレクトリ | Java (main) | 役割 |
|-----------|-------------|------------|------|
| `polaris-relational-jdbc` | `persistence/relational-jdbc` | 19 | **既定の永続化実装**。PostgreSQL / H2 / CockroachDB |
| `polaris-persistence-nosql-*` ほか 39 | `persistence/nosql/**` | 323 | NoSQL 永続化フレームワーク (後述) |

NoSQL 側はさらに機能軸で細かく分かれている。API / SPI / impl / store-nosql という 4 分割が繰り返し現れるのが特徴:

| グループ | モジュール例 | 役割 |
|---------|------------|------|
| async | `polaris-async-{api,java,vertx}` | Executor 抽象 |
| idgen | `polaris-idgen-{api,spi,impl,mocks}` | Snowflake ID 生成 |
| nodes | `polaris-nodes-{api,spi,impl,store-nosql}` | ノード (JVM) 管理 — Snowflake ID のノード ID 割当 |
| authz | `polaris-persistence-nosql-authz-{api,spi,impl,store-nosql}` | 認可情報の永続化 |
| realms | `polaris-persistence-nosql-realms-{api,spi,impl,store-nosql}` | Realm 管理 |
| persistence | `polaris-persistence-nosql-{api,impl,metastore,varint,...}` | DB 非依存の永続化コア |
| maintenance | `polaris-persistence-nosql-maintenance-{api,spi,impl,cel}` | GC / 保持ポリシー (CEL 式で記述) |
| CDI 配線 | `polaris-persistence-nosql-cdi-{common,quarkus,quarkus-distcache,weld}` | DI コンテナごとの配線 |
| DB 実装 | `polaris-persistence-nosql-{inmemory,mongodb}` | 実際のバックエンド (現状 2 つ) |

### 拡張 (10)

| モジュール | ディレクトリ | 役割 |
|-----------|-------------|------|
| `polaris-extensions-federation-{hadoop,hive,bigquery}` | `extensions/federation/*` | 外部カタログへのフェデレーション |
| `polaris-extensions-auth-{opa,ranger}` | `extensions/auth/*` | 外部認可エンジン連携 |
| `polaris-extensions-events-kafka` | `extensions/events/kafka` | イベントを Kafka へ送出 |
| `polaris-extensions-semantic-models` | `extensions/semantic-models` | セマンティックモデル API |
| `polaris-extensions-metrics-reports{,-jdbc}` | `extensions/metrics-reports/*` | Iceberg のスキャン/コミットメトリクスレポートの永続化 |

### ツール (12)

`tools/immutables` (Immutables スタイル定義)、`tools/misc-types`、`tools/version`、`tools/config-docs/{annotations,generator,site}` (設定リファレンス自動生成)、
`tools/container-spec-helper`、`tools/testcontainers/{floci-aws,floci-az,floci-gcp,keycloak,minio,rustfs,hms}`。

Testcontainer モジュールが 7 つあるのは、クラウドストレージ (AWS/Azure/GCP のエミュレータ)、OIDC プロバイダ (Keycloak)、
S3 互換 (MinIO / RustFS)、Hive Metastore をすべてコンテナで立てて統合テストするため。

### テスト・その他 (4)

| モジュール | 役割 |
|-----------|------|
| `polaris-tests` (`integration-tests`) | 正規統合テスト。**下流の Polaris ディストリビューションから再利用できる**よう独立モジュールになっている |
| `polaris-runtime-spark-tests` | Spark プラグインの統合テスト |
| `polaris-bom` | Maven BOM |
| `aggregated-license-report` | ライセンスレポート集約 |

### Spark プラグイン (動的 include)

`plugins/spark/spark-scala.properties` から生成される:

```
sparkVersions=3.5,4.0
scalaVersions.3.5=2.12,2.13
scalaVersions.4.0=2.13
```

→ `polaris-spark-3.5_2.12`, `polaris-spark-3.5_2.13`, `polaris-spark-4.0_2.13` の 3 プロジェクト。
実装は `plugins/spark/common/` と `plugins/spark/v{3.5,4.0}/spark/` にある ([09](09-federation-and-extensions.md))。

## モジュール依存関係グラフ

```
polaris-core  (エンティティ、権限、永続化 API、Resolver、ストレージ抽象; Quarkus 非依存)
  │
  ├── polaris-api-management-model ─┐
  ├── polaris-api-management-service ┤
  ├── polaris-api-iceberg-service    ├── polaris-runtime-service
  ├── polaris-api-catalog-service   ─┘      │  (REST 実装、認証認可配線、タスク、イベント)
  │                                         │
  ├── polaris-relational-jdbc ──────────────┤
  ├── polaris-persistence-nosql-* ──────────┤
  ├── polaris-extensions-* ─────────────────┤
  │                                         │
  │                                    ┌────┴────┐
  │                                    │         │
  │                          polaris-server  polaris-admin
  │                                    │         │
  │                              polaris-distribution
  │
  └── plugins/spark (polaris-spark-*)  ← polaris-core に依存せず Iceberg + Spark のみに依存
```

設計原則: `polaris-core` が Quarkus/CDI を知らないことで、永続化実装や認可器をライブラリとして単体テストできる。
CDI の配線はすべて `runtime/service` 側 (`ServiceProducers` ほか) に集約されている。

## 主要ライブラリバージョン

| 依存 | バージョン | 用途 |
|------|-----------|------|
| Quarkus | 3.38.1 | ランタイム (CDI, REST, 設定, OIDC, Micrometer, OpenTelemetry) |
| Iceberg | 1.11.0 | REST リクエスト/レスポンス型、`Catalog` インターフェース、`TableMetadata` |
| Jackson | 2.22.1 (BOM) | JSON |
| Guava | 33.6.0-jre | 内部ユーティリティ |
| Caffeine | 3.2.4 | エンティティキャッシュ、認証情報キャッシュ |
| AWS SDK | 2.44.7 (BOM) | STS (credential vending)、S3 |
| Azure SDK | 1.3.8 (BOM) | ADLS SAS トークン |
| Google Cloud Storage | 2.70.0 (BOM) | GCS |
| Auth0 java-jwt | 4.6.0 | 内部 JWT の発行・検証 |
| Immutables | - | 値オブジェクト生成 (`@PolarisImmutable`) |
| CEL | 0.8.0 (BOM) | NoSQL 保持ポリシーの式評価 |
| Hive Metastore | 3.1.3 | Hive フェデレーション |
| Hadoop | - | Hadoop フェデレーション、HadoopFileIO |
| H2 | 2.4.240 | テスト・軽量デプロイ用 JDBC バックエンド |
