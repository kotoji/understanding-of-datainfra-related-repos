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

- `SparkConnectService` — gRPC AsyncService 実装
- `SparkConnectPlanner` — protobuf Plan → Spark SQL LogicalPlan 変換
- `SparkConnectSessionManager` — user/session 毎の SparkSession 管理
- `ExecuteGrpcResponseSender` — 結果ストリーミング
- Plugin system (`SparkConnectPlugin`) で拡張可能

## Client Side

- `SparkConnectClient` — gRPC ManagedChannel で通信
- Arrow ベースのデータシリアライズ
- `ExecutePlanResponseReattachableIterator` — 耐障害性のある結果ストリーミング
- Retry / reattach サポート
