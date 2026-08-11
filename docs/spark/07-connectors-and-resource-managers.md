# Connectors & Resource Managers

## Connectors 一覧

| Connector | Artifact | API |
|-----------|----------|-----|
| **Kafka** (Structured Streaming) | `spark-sql-kafka-0-10` | DataSource V2 (MicroBatch/Continuous/Batch) |
| **Kafka** (DStream, deprecated) | `spark-streaming-kafka-0-10` | DStream API |
| **Avro** | `spark-avro` | DataSource V2 |
| **Protobuf** | `spark-protobuf` | DataSource V2 |
| **Kinesis** (opt-in, ASL) | `spark-streaming-kinesis-asl` | DStream API |
| **Ganglia** (opt-in, LGPL) | `spark-ganglia-lgpl` | Metrics Sink |
| **Profiler** (opt-in) | `spark-profiler` | async profiler |

## Kafka Connector 詳細

`KafkaSourceProvider` が `"kafka"` として登録:
- `KafkaMicroBatchStream` — micro-batch ストリーミング読み取り。rate limit (`maxOffsetsPerTrigger`, `minOffsetPerTrigger`, `maxTriggerDelayMs`)
- `KafkaContinuousStream` — continuous processing
- `KafkaBatch` — バッチ読み取り
- Consumer/Producer pool (`InternalKafkaConsumerPool`, `CachedKafkaProducer`) で効率的な再利用
- Offset reader は Admin API / Consumer API の2実装

## Kubernetes

`KubernetesClusterManager` → `KubernetesClusterSchedulerBackend` (extends `CoarseGrainedSchedulerBackend`)

**Feature Step パターン:** Pod spec を構成する composable なステップ:
- `BasicDriverFeatureStep`, `BasicExecutorFeatureStep`
- `MountVolumesFeatureStep`, `MountSecretsFeatureStep`
- `HadoopConfDriverFeatureStep`, `KerberosConfDriverFeatureStep`
- `NetworkPolicyFeatureStep` 等

Executor 管理: `ExecutorPodsAllocator` → `ExecutorPodsLifecycleManager` → watch/polling snapshot

Volcano scheduler 統合あり。Fabric8 Kubernetes client 使用。

## YARN

`YarnClusterManager` → `YarnSchedulerBackend` (Client/Cluster モード)

- `ApplicationMaster` — executor allocation, credential delegation
- `Client` — YARN submit, classpath/resource localization
- `YarnAllocator` — container 要求・管理
- `ExecutorRunnable` — container 内で executor プロセス起動

(Mesos は削除済み)
