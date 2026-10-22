# Industrial IoT Streaming & Near-Real-Time ML Inference Platform

**System design interview reference** · Platform Data Engineer · AWS, Apache Flink on EKS, Kinesis/MSK, Databricks Delta Lake, MLflow

## 1. Problem and goals

Design a large-scale event-driven platform that ingests industrial sensor and operational telemetry from drones, robots, and equipment, produces failure-risk predictions in near real time, persists trustworthy historical data, and distributes results reliably to independently operated downstream integrations.

**Illustrative assumptions (confirm with interviewer):**

| Dimension | Design target |
|---|---|
| Steady incoming rate | 100,000 sensor events/second |
| Burst load to design/test | 300,000 events/second |
| Average payload | 1 KB/event |
| Prediction latency | p99 under 2 seconds from cloud-stream ingestion to durable prediction publication, for eligible events under the agreed load envelope |
| Ordering | Per-device where required, not global ordering |
| Processing guarantees | At-least-once transport with deduplication and idempotent business effects |
| Persistence | Raw/audit data and validated/curated Delta tables in S3/Databricks |
| Availability | Multi-AZ high availability; regional DR targets defined separately |

**Capacity check:** At 100K × 1 KB/s, the nominal ingestion payload rate is ~100 MB/s, ~8.64 TB/day; at 300K/s it is ~300 MB/s, ~25.92 TB/day if sustained. Excludes protocol overhead, replication, compression, and metadata. Not all telemetry should trigger an ML invocation: use changes/windows/policies to control inference request rate.

## 2. Architecture: complete Mermaid diagram

```mermaid
flowchart TB
    subgraph EDGE[Industrial edge and ingestion]
        DEV[IoT sensors / drones / robots]
        GW[Edge gateway / AWS IoT Core<br/>Device auth, buffering, protocol translation]
        DEV -->|MQTT / HTTPS / gRPC; Protobuf / Avro| GW
    end

    GW -->|Validated transport; device_id key| STREAM[(Kinesis Data Streams<br/>or Amazon MSK / Kafka)]
    SCHEMA[Schema registry / data contracts<br/>versioned Protobuf or Avro]
    SCHEMA -.-> GW
    SCHEMA -.-> STREAM

    subgraph HOT[HOT PATH — low-latency prediction]
        FLINK[Apache Flink on EKS<br/>Flink Kubernetes Operator<br/>source → parse → validate → dedup]
        STATE[(Flink keyed state<br/>keyBy(device_id); rolling features)]
        META[(DynamoDB / Redis OPTIONAL<br/>device metadata, shared online features)]
        SERVE[Real-time model endpoint<br/>Databricks Model Serving / SageMaker / EKS]
        RESULT[(Durable prediction results stream<br/>Kinesis or Kafka topic)]
        FLINK --> STATE
        STATE -->|Async inference, bounded concurrency| SERVE
        META -.->|Optional asynchronous lookup| STATE
        SERVE -->|Prediction + model version + event_id| RESULT
    end

    STREAM -->|Independent source consumer| FLINK

    subgraph LAKE[LAKE PATH — independent analytical consumer]
        DBX[Databricks Structured Streaming<br/>schema validation + data quality]
        BRONZE[(S3 + Delta Bronze<br/>raw payload / ingest metadata)]
        SILVER[(S3 + Delta Silver<br/>typed, deduplicated, validated)]
        GOLD[(S3 + Delta Gold<br/>curated features / training datasets)]
        DBX --> BRONZE --> SILVER --> GOLD
    end
    STREAM -->|Separate consumer / consumer group| DBX

    subgraph MLOPS[ML development / model lifecycle]
        TRAIN[Databricks training / evaluation<br/>Argo Workflows or Databricks Jobs]
        MLFLOW[MLflow tracking + Model Registry<br/>approved model version]
        TRAIN --> MLFLOW
    end
    GOLD --> TRAIN
    MLFLOW -.->|Versioned deployment / canary / rollback| SERVE

    subgraph DELIVERY[Downstream integration — independent reliability domain]
        ROUTE[Alert processor + idempotency<br/>incident state / dedup]
        BUS[EventBridge rules / SQS fan-out]
        MAINT[Maintenance ticket API]
        DASH[Dashboard / device status API]
        PARTNER[External integration / webhook]
        DLQ[(DLQ + replay / reconciliation)]
        ROUTE --> BUS
        BUS -->|Queue + worker; retry| MAINT
        BUS -->|Queue + worker| DASH
        BUS -->|Queue + worker; backoff| PARTNER
        BUS -->|Poison/exhausted deliveries| DLQ
    end
    RESULT --> ROUTE
    ROUTE -->|Latest prediction/status| META

    S3CP[(S3 Flink checkpoints / savepoints)]
    FLINK -.->|State recovery| S3CP
    OPS[Cross-cutting: Terraform, IAM/Pod Identity, TLS/KMS,<br/>CloudWatch/Datadog, traces, alerting, SLO dashboards]
    OPS -.-> FLINK
    OPS -.-> DBX
    OPS -.-> ROUTE

    classDef stream fill:#d5eee4,stroke:#33836c,color:#174b3c;
    classDef compute fill:#e4e4fb,stroke:#7665b5,color:#302764;
    classDef store fill:#dfeeff,stroke:#5487be,color:#1f4573;
    classDef ml fill:#f9e8d4,stroke:#bf8744,color:#72440f;
    class STREAM,RESULT stream;
    class FLINK,DBX,GW,ROUTE,BUS compute;
    class BRONZE,SILVER,GOLD,S3CP,META,DLQ store;
    class SERVE,TRAIN,MLFLOW ml;
```

**How to read this:** The two solid branches out of the ingestion stream are *independent consumers*. No Databricks job is synchronously required for a prediction. The dashed arrows are control-plane, optional lookup, deployment, or recovery relationships; they are not necessarily per-event calls. The diagram is conceptual: choose *one* primary streaming backbone first (Kinesis or MSK), not both without a clear reason.

## 3. End-to-end workflow

1. **Sensor produces a telemetry event.** Example: `event_id`, `device_id`, `event_time`, `sequence_number`, temperature, vibration, pressure, `schema_version`. Device gateways authenticate/authorize devices, buffer during outages, and transmit Protobuf or Avro payloads using an agreed schema contract.
2. **Gateway publishes to Kinesis/MSK.** Partition by `device_id` when per-device input ordering is useful; watch for hot devices and key skew. A stream is a durable finite-retention event log, not the historical lake.
3. **Flink consumes from a source connector on EKS.** Source readers are subtasks on TaskManager pods; the Flink Kubernetes Operator manages JobManager, TaskManagers, upgrades, and recovery. Configure Pod Identity/IRSA, private network access, and compatible connector JARs.
4. **Flink validates and performs event-time processing.** Parse schema, quarantine malformed events, deduplicate on stable event ID, apply watermarks for late records, then `keyBy(device_id)` to maintain per-device rolling aggregates/state. `keyBy()` partitions *Flink computation*, separate from Kafka partitions or Kinesis shards. Flink keyed state does not require DynamoDB.
5. **Produce online features.** Maintain recent average temperature, max vibration, pressure trend, anomaly counts. Make predictions on meaningful changes, sliding evaluations, or agreed schedules. Use optional DynamoDB/Redis only for shared device metadata or externally accessible online features—not for every per-event state update.
6. **Call an approved model endpoint.** Use bounded asynchronous I/O, timeout, capacity control, and circuit breakers to Databricks Model Serving (JD alignment), SageMaker, or EKS hosting. Track `event_id`, feature timestamp, `model_version`, and `prediction_id`. If strict latency requires it, benchmark a local EKS model server.
7. **Publish results durably.** Write prediction outcomes to a separate Kafka topic/Kinesis stream with a stable key and an idempotent downstream design. Model endpoint calls and partner APIs are not automatically exactly-once merely because Flink checkpoints.
8. **Distribute alerts independently.** Alert processor applies risk policy, alert hysteresis/cooldown and incident-level deduplication, then sends through EventBridge to SQS-backed workers. Workers perform bounded retries, DLQ processing and reconciliation. An idempotency key should represent the *incident/action*, not only the unique sensor event.
9. **Land historical data separately.** Databricks Structured Streaming independently reads from Kinesis/MSK. Bronze preserves raw payloads and ingest metadata; Silver validates/types/deduplicates; Gold provides feature/training aggregates. Use Delta transaction logs, checkpointed streaming writes, compaction/optimization, retention policy and appropriate data layout rather than partitions for every device ID.
10. **Train and deploy models asynchronously.** Gold dataset snapshot/version → Databricks training/experiments → MLflow tracking/registry → approval → canary/shadow serving rollout → monitor latency, quality and drift → rollback when needed. No training job is in the per-event hot path.

## 4. Example record and prediction

Example telemetry (illustrative):

```json
{
  "event_id": "evt-10001",
  "schema_version": 2,
  "device_id": "motor-123",
  "sequence_number": 10482,
  "event_time": "2026-10-22T12:00:00Z",
  "temperature_c": 95.0,
  "vibration": 0.92,
  "pressure_kpa": 240.0
}
```

Flink computes a feature vector:

```json
{
  "device_id": "motor-123",
  "feature_time": "2026-10-22T12:00:01Z",
  "avg_temperature_5m": 93.8,
  "max_vibration_1m": 0.92,
  "pressure_trend_5m": 12.5
}
```

Hypothetical inference result (not a prediction from an actual trained model):

```json
{
  "prediction_id": "motor-123:incident-456:failure-model-v3",
  "device_id": "motor-123",
  "model_version": "failure-model-v3",
  "failure_probability": 0.94,
  "risk_level": "CRITICAL",
  "prediction_time": "2026-10-22T12:00:01Z"
}
```

## 5. Data semantics, state, and storage

| Layer | Role | Why not another layer? |
|---|---|---|
| Kinesis/MSK | Durable streaming transport, consumer replay | Not a permanent historical analytical store |
| Flink keyed state / RocksDB | Fast incremental rolling aggregates per device | No cross-service query API; recover via checkpoints |
| DynamoDB (optional) | Latest device status, alert incident state, shared metadata/features | Avoid one network write per raw sensor event |
| S3 + Delta Bronze | Immutable-ish raw capture + ingest metadata | Retains original evidence for backfill |
| S3 + Delta Silver | Validated, deduplicated, typed event records | Trusted query and downstream datasets |
| S3 + Delta Gold | ML-ready feature tables, aggregate analytics | Offline training / analysis, not synchronous inference |
| MLflow | Experiments, model versions and deployment lineage | Does not substitute for a low-latency serving endpoint |

**Serialization vs file storage:** Use versioned Protobuf/Avro per sensor event; Parquet is a columnar on-disk storage format commonly used by Delta tables. Schema compatibility and table-format evolution are related but distinct responsibilities.

## 6. Scaling to 100K–300K events/sec

**Kinesis provisioned mode:** At ~1 KB/event, plan approximately 100 shards minimum for 100K/sec and 300 shards minimum for 300K/sec, subject to both bytes/sec and 1,000 records/sec per-shard write limits. With ~30% headroom, use ~130 and ~390 as *starting capacity estimates*. Hot shards and actual payload distribution change the result. Plan/pre-warm for predictable spikes; do not count on reactive shard resizing to preserve a strict two-second latency SLO during an instantaneous 3× burst.

**MSK/Kafka:** Broker CPU/network/EBS throughput, producer batching/compression, replication, partition count, skew, and consumer lag drive effective capacity. Benchmark per-partition throughput; do **not** blindly equate one Kafka partition to one Kinesis shard. Use replication across AZs, `acks=all`, idempotent producer and suitable minimum in-sync replicas.

**Flink/EKS:** Scale source parallelism in relation to shard/partition count; scale downstream keyed operators separately. Distribute slots among TaskManager pods and provide EKS node capacity. Watch checkpoints, state size, rebalance costs and backpressure. Hot keys do not improve merely because parallelism increased.

**Inference capacity:** Let eligible prediction rate = event rate × fraction requiring inference. For example, a 300K/s spike with 10% invoking the model creates 30K predictions/sec. At mean service latency 100 ms, Little's Law implies ~3,000 mean in-flight requests before headroom/queueing, not 3,000 pods. Reduce unnecessary inference, batch when the latency budget allows and separately scale model serving.

**Producer & consumer tuning:**

- **Kinesis:** `PutRecords` batching, retry only failed records, jittered backoff, balanced partition keys, per-shard metrics, enhanced fan-out for multiple latency-sensitive consumers where supported, monitor iterator age and read/write throttling.
- **MSK:** tune `linger.ms`, `batch.size`, compression (`lz4`/`zstd`), `acks`, idempotence, broker/partition sizing, fetch settings, replication/ISR, client network and `BytesIn/Out`, under-replicated partitions, CPU and lag. Benchmark p99, not throughput alone.
- **Flink:** checkpoint interval and duration, source reader throughput, RocksDB state and compaction, key skew, event-time watermarks, async inference concurrency and sink backpressure.

## 7. Latency objective

Illustrative **budget allocations** (not measured guarantees):

| Stage | Budget |
|---|---:|
| Ingest/gateway to durable stream | 200 ms |
| Stream pickup/validation | 250 ms |
| Feature updates/lookups | 300 ms |
| Model inference | 500 ms |
| Durable result publication | 150 ms |
| Contingency | 600 ms |
| **Total planned budget** | **2,000 ms** |

Measure **end-to-end p99 directly** using event/correlation IDs, traces and synchronized timestamps. A sum of per-stage p99 values is not itself proof of end-to-end p99. Clarify whether offline-device upload delays and downstream delivery are included in the SLO; preferably define separate SLOs for event publication, lake freshness and alert delivery.

## 8. Failure, replay and delivery guarantees

| Failure | Response |
|---|---|
| Edge network outage | Buffer locally with bounded queue, retry with stable IDs, preserve event time |
| Kinesis write throttling / hot shard | Metered producer retries, scale/warm shards, adjust key distribution if ordering allows |
| Kafka broker outage | Multi-AZ replicas, leader failover, monitor ISR/under-replicated partitions |
| Flink TaskManager/node crash | Restore Flink state + source position from a completed checkpoint in S3; verify finite stream retention |
| Flink backpressure | Trace the slowest downstream operator; bound model calls; tune/scale the true bottleneck |
| Model endpoint unavailable | Circuit breaker, bounded retries, retained input backlog, age-based policy for stale predictions |
| Databricks unavailable | Continue hot path; lake consumer catches up from retained stream or durable archive |
| External partner API unavailable | Dedicated SQS worker, retry/backoff, DLQ, reconciliation; avoid blocking inference |
| Duplicate event / repeated alert | Event-level dedup in processing; incident-level idempotency at business side-effect boundary |
| Regional failure | Cross-region copy/DR if required, explicit RPO/RTO and tested failover; replication is typically asynchronous |

**Exactly-once nuance:** Checkpointing and transactional sinks can ensure consistent Flink-managed state and supported sink writes, but do not make arbitrary HTTP model invocations or external ticket creations exactly-once. Prefer at-least-once delivery with deterministic IDs, conditional writes, transactional outbox and reconciliation.

## 9. Operations, security and deployment

- **Infrastructure:** Terraform for VPC, EKS, IAM, Kinesis/MSK, S3, KMS, monitoring, and where supported Databricks resources. Helm plus the Flink Kubernetes Operator for Flink job deployments. Argo Workflows for ML/batch pipeline DAGs; Argo CD, if used, for GitOps deployment.
- **Identity/network:** Device authentication at the edge; private networking/VPC endpoints; EKS Pod Identity or IRSA for Flink AWS access; broker authentication/TLS where appropriate; least privilege and secrets management.
- **Data governance:** Protobuf/Avro schema versions and compatibility checks, data contracts with Edge/AI teams, data classifications, PII policies, retention, Unity Catalog and audit events.
- **Observability:** Kinesis write errors/shard saturation/iterator age, Kafka consumer lag/broker metrics, Flink backpressure/checkpoint health, model endpoint p95/p99/throughput, Delta batch lag/quality, SQS queue age/DLQ, end-to-end prediction latency and alert success.
- **Release strategy:** Contract tests, load and failure tests, versioned model registry, canary/shadow model rollouts, savepoint-aware Flink upgrades, rollback plans and DR rehearsals.

## 10. Key design tradeoffs to defend aloud

1. **Kinesis vs MSK:** Kinesis is simpler AWS-managed streaming and explicit in the JD; MSK gives Kafka ecosystem/clients and deeper protocol tuning. Choose based on existing platform, consumer tooling, throughput and cost.
2. **Flink on EKS vs managed Flink:** EKS fits Kubernetes/Helm/operator ownership and custom connectors; managed Flink can reduce operational load. Compare required runtime/control and team capability.
3. **Databricks Model Serving vs EKS/SageMaker:** Databricks aligns with MLflow and the JD; EKS may lower round-trip latency or support special model runtimes; benchmark actual endpoint p99 and cost.
4. **Event-by-event vs selective inference:** Infer for every record only if justified by business rules and model capacity; incremental windows/change triggers are often more efficient.
5. **Latency vs throughput:** Larger batches improve throughput and compression but add waiting; tune end-to-end p99, not just maximum records/sec.
6. **Ordering vs distribution:** `device_id` helps order within partition/shard and group Flink state, but creates potential hot keys. Flink `keyBy()` is a separate computation repartition and does not fix out-of-order source events by itself.
7. **Availability vs exactly-once claims:** Multi-AZ, checkpointing and replay are essential, but external side effects need explicit idempotency and business-level consistency.

## 11. Interview walk-through (60 minutes)

| Minutes | Focus |
|---|---|
| 0–5 | Clarify requirements, prediction cadence and latency clock |
| 5–10 | Capacity math: 100K/300K, data/day, shard/broker bounds |
| 10–20 | Draw Mermaid architecture; trace one device event |
| 20–35 | Flink keyed state, watermarks, model serving, Databricks Bronze/Silver/Gold |
| 35–45 | Scaling, backpressure, hot keys, checkpoints, replay |
| 45–55 | Exactly-once boundaries, downstream alerts, HA, security, observability, costs |
| 55–60 | Explain tradeoffs and how to evolve after load tests |

### Two-minute verbal summary

> I would ingest authenticated industrial telemetry through an edge gateway into Kinesis or MSK, partitioning by device ID when per-device ordering matters. The same retained event stream feeds two independent consumers. On the hot path, Flink on EKS validates and deduplicates events, maintains per-device keyed feature state, invokes an approved model endpoint asynchronously, and publishes predictions to a durable results stream. An independent alert integration tier applies incident-level idempotency and delivers through EventBridge and SQS-backed workers with retries and DLQs. On the lake path, Databricks Structured Streaming writes Bronze, Silver, and Gold Delta tables on S3 for analytics and historical ML training, managed with MLflow. I would test 100K/sec sustained and 300K/sec bursts, size stream partitions/shards and Flink/model-serving capacity independently, checkpoint state to S3, and measure end-to-end p99 latency. The core tradeoff is preserving a two-second inference objective without coupling it to historical processing or unreliable downstream APIs.

## 12. Official documentation starting points

- [Apache Flink Kubernetes Operator](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-stable/)
- [Apache Flink DataStream concepts](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/overview/)
- [Amazon Kinesis Data Streams](https://docs.aws.amazon.com/streams/latest/dev/introduction.html)
- [Amazon MSK](https://docs.aws.amazon.com/msk/latest/developerguide/what-is-msk.html)
- [Databricks Structured Streaming](https://docs.databricks.com/aws/en/structured-streaming/)
- [Delta Lake documentation](https://docs.delta.io/)
- [MLflow documentation](https://mlflow.org/docs/latest/index.html)

---

*Interview design, not an assertion of the employer's actual implemented architecture. Capacity, latencies and throughput must be verified through representative production load testing and the chosen service/version limits.*
