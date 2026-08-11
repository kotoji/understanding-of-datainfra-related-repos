# Spark Connect

## アーキテクチャ

gRPC ベースのクライアント・サーバ分離アーキテクチャ。クライアントは SparkSession API を提供するが、実行は全てサーバ側で行う。

```
Client (JVM/Python/JDBC)
    │  gRPC (protobuf Plan)
    ▼
SparkConnectService (gRPC server)
    │  SparkConnectPlanner
    ▼
Spark SQL Engine (Catalyst → Execution)
    │  Arrow (data transfer)
    ▼
Client (結果受信)
```

## Protocol (11 proto ファイル)

- `base.proto` — `Plan` (Relation | Command), `ExecutePlanRequest/Response`
- `relations.proto` — 40+ relation type (Read, Project, Filter, Join, Aggregate, SQL, ...)
- `expressions.proto` — Expression tree
- `commands.proto` — Write 操作、session コマンド
- `catalog.proto` — カタログ操作
- `ml.proto` / `ml_common.proto` — ML 操作

## Server Side

`sql/connect/server` の主要パッケージ:

| パッケージ | 役割 |
|-----------|------|
| `service` | `SparkConnectService` (gRPC AsyncService)、session/artifact ハンドラ |
| `planner` | `SparkConnectPlanner` — protobuf Plan → Spark SQL LogicalPlan 変換 |
| `execution` | `ExecuteGrpcResponseSender` 等、結果ストリーミング |
| `plugin` | `SparkConnectPlugin` による拡張ポイント |
| `ml` / `pipelines` | ML 操作、Declarative Pipelines の Connect 対応 |
| `config` / `ui` / `utils` | 設定、Spark UI 統合、共通処理 |

- `SparkConnectSessionManager` — user/session 毎の SparkSession 管理
- **Artifact 管理:** クライアントが送った jar/file/pyfile を `SparkConnectAddArtifactsHandler` が受信し、`SparkConnectArtifactStatusesHandler` で登録状況を確認できる。セッション毎に独立したディレクトリに配置され、UDF 実行時の classpath/PYTHONPATH に追加される

## Client Side

- `SparkConnectClient` — gRPC ManagedChannel で通信
- Arrow ベースのデータシリアライズ
- `ExecutePlanResponseReattachableIterator` — 耐障害性のある結果ストリーミング
- Retry / reattach サポート
