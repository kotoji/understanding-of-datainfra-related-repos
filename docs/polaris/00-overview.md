# Apache Polaris 1.7.0-SNAPSHOT コードリーディング

> 調査日: 2026-08-12
> 対象コミット: `a05566e8e` (main ブランチ, 2026-08-10)
> Java 21 (必須) / Gradle 9.7.0 / Quarkus 3.38.1 / Iceberg 1.11.0
>
> 本ドキュメント中の集計値 (モジュール数、ファイル数、コミット数等) はすべて上記コミット時点のもの。
> コミット数・コントリビュータ数は `HEAD` の履歴のみを対象とする (`git rev-list --count HEAD` / `git shortlog -sn HEAD`)。
> リリースタグ (`apache-polaris-1.7.0` 等) は `release/1.x` ブランチ上にあり main の祖先ではないため、`git describe` は解決できない。
>
> ソースの位置は Gradle モジュールのディレクトリからの相対パスで示す。`...` は `src/main/java/org/apache/polaris` の省略
> (例: `polaris-core/.../core/entity/PolarisEntityType.java` = `polaris-core/src/main/java/org/apache/polaris/core/entity/PolarisEntityType.java`)。

## Polaris とは

Apache Iceberg の **REST Catalog サーバー実装**。Iceberg 側 ([iceberg/06-catalog-and-rest.md](../iceberg/06-catalog-and-rest.md)) が定義した
REST Catalog API の「サーバー側」を担う。Iceberg リポジトリが提供するのはクライアント (`RESTCatalog`) と OpenAPI 仕様までで、
その裏側にある「誰が何をできるか」「メタデータをどこに永続化するか」「ストレージ認証情報をどう払い出すか」は仕様外だった。
Polaris はそこを埋める。

したがって Polaris の関心事は Iceberg とほとんど重ならない:

| | Iceberg | Polaris |
|---|---------|---------|
| 主な関心 | テーブルフォーマット、スキャンプランニング、書き込みパス | 認証・認可、マルチテナンシー、メタデータ永続化、認証情報払い出し |
| データファイル | 読み書きする | **一切触らない** (パスの検証と非同期削除のみ) |
| 主要な抽象 | `TableMetadata`, `Snapshot`, `Manifest` | `PolarisEntity`, `PolarisGrantRecord`, `RealmContext` |
| 状態の置き場所 | オブジェクトストレージ上の JSON/Avro | RDB / NoSQL (`entities` テーブル等) |

Polaris が管理するのは「テーブル名 → 現在の `metadata.json` の場所」という **ポインタ 1 個** と、その周辺の権限情報である。
テーブルの中身 (スナップショット、マニフェスト) は Iceberg のフォーマット仕様どおりストレージ上にあり、Polaris はそれを読まない。

## ドキュメント一覧

| ファイル | 内容 |
|---------|------|
| [00-overview.md](00-overview.md) | 本ファイル。概要とインデックス |
| [01-build-and-modules.md](01-build-and-modules.md) | ビルドシステム、モジュール構成、OpenAPI コード生成 |
| [02-entity-model.md](02-entity-model.md) | エンティティモデル、階層、Realm によるマルチテナンシー |
| [03-persistence.md](03-persistence.md) | 永続化層 (`BasePersistence` / `PolarisMetaStoreManager`)、JDBC スキーマ、NoSQL 実装 |
| [04-rbac-and-authorization.md](04-rbac-and-authorization.md) | 権限モデル、Grant、Resolver、認可判定アルゴリズム、OPA/Ranger |
| [05-authentication.md](05-authentication.md) | 内部認証 (OAuth2/JWT)、外部認証 (OIDC)、Realm 解決、Principal マッピング |
| [06-iceberg-rest-service.md](06-iceberg-rest-service.md) | REST リクエストの処理経路、テーブルコミットプロトコル、ロケーション検証 |
| [07-storage-and-credential-vending.md](07-storage-and-credential-vending.md) | ストレージ設定、認証情報払い出し (credential vending)、FileIO |
| [08-runtime-quarkus.md](08-runtime-quarkus.md) | Quarkus ランタイム、設定システム、非同期タスク、イベント、admin ツール |
| [09-federation-and-extensions.md](09-federation-and-extensions.md) | カタログフェデレーション、Generic Table、Policy、Spark プラグイン |
| [10-architecture-summary.md](10-architecture-summary.md) | 設計原則の総括、各要素の噛み合い方 |

## 規模感

- Gradle サブプロジェクト数: **79** (`gradle/projects.main.properties`) + Spark プラグインのバージョン別プロジェクト 3 個 + included build 2 個
  - うち **39 個が NoSQL 永続化フレームワーク** (`persistence/nosql/` 配下) — リポジトリの半分近くが「まだ既定ではない永続化実装」に費やされている
- 主要言語: Java 21。ビルドスクリプトは Kotlin DSL (`.kts` 98 ファイル)、Spark プラグインの一部に Scala
- Java ファイル数: 合計 1,514 (main 985 / test 381 / その他 intTest 等)
  - `polaris-core` 247 (main)、`runtime/service` 208、`persistence/nosql` 323、`tools` 59
- 総コミット数: 3,681 (2024-05-29 〜 2026-08-10)
- 主要コントリビュータ: Robert Stupp (523), Alexandre Dutra (287), Yong Zheng (190), JB Onofré (187), Dmitri Bourlatchkov (166), Yufei Gu (124)。
  なお最多は Mend Renovate (1,143) で、依存バージョン更新の自動コミット

Iceberg (9,034 コミット / 2017 年〜) と比べるとまだ若く、コード量も 1 桁小さい。

## アーキテクチャ概観

縦の `▼` は層をまたぐ関係を表すが、層ごとに意味が異なるため矢印の横に明記した。
横の `→` は同じ層の中での「インターフェース → 具象実装」または「包含関係」を表す。

```
Client:      Spark / Flink / Trino / StarRocks (Iceberg REST Catalog クライアント)
               │
               ▼ HTTP: /v1/{prefix}/namespaces/... + Authorization ヘッダ
Filter:      RequestId → RealmContext → MDC → Tracing → Idempotency         [08]
               │        (Realm = テナント識別。以降すべての層が realm スコープ)
               ▼ SecurityIdentity を Authenticator が PolarisPrincipal に変換
AuthN:       Authenticator → Internal (JWT) / External (OIDC)               [05]
               │
               ▼ REST リソース実装 (OpenAPI 生成インターフェースの実装)
Adapter:     IcebergCatalogAdapter / GenericTableCatalogAdapter /           [06]
               │  PolarisServiceImpl (Management API)
               ▼ 認可を先に済ませてからカタログ操作を呼ぶ
Handler:     CatalogHandler → IcebergCatalogHandler                         [06]
               │  ├─ Resolver: 名前パスを実体化 (+ grant records を同梱)    [04]
               │  └─ PolarisAuthorizer: 推移的権限チェック                   [04]
               ▼ Iceberg の Catalog インターフェース実装として振る舞う
Catalog:     LocalIcebergCatalog (implements Iceberg Catalog)               [06]
               │  → TableOperations: doRefresh / doCommit
               │  ├─ ロケーション検証 (重なり・逸脱の禁止)                    [06]
               │  └─ StorageAccessConfigProvider: 認証情報払い出し           [07]
               ▼ エンティティ CRUD と grant を一括で扱う API
MetaStore:   PolarisMetaStoreManager → AtomicOperationMetaStoreManager      [03]
               │                     → TransactionalMetaStoreManagerImpl
               ▼ 単一行の atomic な read/write に落とす SPI
Persistence: BasePersistence → JdbcBasePersistenceImpl / NoSQL / TreeMap    [03]
               │
               ▼
Storage:     PostgreSQL / H2 / CockroachDB / MongoDB / in-memory
```

`[NN]` は詳細を扱うドキュメント番号。

**別系統**として、データファイル用のストレージ (S3/GCS/Azure) に対する認証情報払い出し ([07](07-storage-and-credential-vending.md)) と、
外部カタログへのフェデレーション ([09](09-federation-and-extensions.md)) がある。前者は「クライアントがデータを直接読み書きするための鍵」を
Polaris が STS 等で subscope して渡す仕組みで、Polaris 自身はデータを経由しない。

## 設計目標

コードから読み取れる方針:

- **Iceberg REST 仕様への準拠** — サーバー側インターフェースは Iceberg の OpenAPI 仕様から生成する ([01](01-build-and-modules.md))
- **マルチテナンシー (Realm)** — 全永続化行が `realm_id` を持ち、1 デプロイで複数テナントを収容できる ([02](02-entity-model.md))
- **細粒度 RBAC** — 102 種類の privilege を階層エンティティに付与し、親から子へ推移的に効かせる ([04](04-rbac-and-authorization.md))
- **credential vending** — クライアントにストレージの長期鍵を渡さず、テーブルのパスに絞った短期認証情報を払い出す ([07](07-storage-and-credential-vending.md))
- **オープンな多段トランザクションを要求しない** — 永続化バックエンドに求めるのは単一行の compare-and-swap のみ ([03](03-persistence.md))
- **差し替え可能性** — 認可器 (OPA/Ranger)、認証、永続化、FileIO、イベントリスナがすべて CDI の `@Identifier` で差し替え可能 ([08](08-runtime-quarkus.md))
