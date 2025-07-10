
# Real-Time AWS Data Pipeline for Clickstream ML Inference

##  Problem Statement

Design a **real-time, low-latency data pipeline** that:

- Ingests **clickstream data from Elasticsearch**
- Processes and extracts **model-ready features** (e.g., `hour_of_day`, `zip → income`)
- Sends them to an **ML model hosted behind a REST API**
- Maintains **latency under 100ms**
- Supports **failure monitoring and recovery**
- Is built entirely using **AWS-native or AWS-compatible services**

---

## Architecture

```
[Elasticsearch]
      ↓
[Logstash]
      ↓
[Amazon MSK (Kafka)]
      ↓
[Kafka Streams (Feature Engineering)]
      ↓
[Redis (Aux Lookup / Feature Store)]
      ↓
[ML Model (REST API: SageMaker / ECS / Fargate)]

Optional:
      ↓
[DynamoDB + DAX or S3 + Athena for persistent storage]
```

---

## Component-by-Component Breakdown

### 1. **Ingestion: Elasticsearch → Logstash**

- **Chosen**: **Logstash**
- **Why**:
  - Native plugin support for **Elasticsearch input** and **Kafka output**
  - Low-setup, reliable batch-to-stream converter

- **Alternatives Considered**:
  | Contender | Why Not Chosen |
  |-----------|----------------|
  | Fluent Bit | Lighter, but less robust support for Kafka output and transformation logic |
  | Custom ETL Script | Increases engineering overhead; not reliable for large-scale ingestion |

---

### 2. **Buffering & Stream Transport: Amazon MSK (Kafka)**

- **Chosen**: **Amazon MSK (Managed Kafka)**
- **Why**:
  - Industry-standard for **high-throughput, durable stream transport**
  - Allows message replays, partitioning, strong order guarantees
  - Seamlessly integrates with **Kafka Streams** for downstream processing

- **Alternatives Considered**:
  | Contender | Why Not Chosen |
  |-----------|----------------|
  | **Kinesis** | Good for real-time ingestion, but limited processing capabilities and replay options; Kafka is better for stream + processing pipeline |
  | **SQS** | Meant for simple queuing, not high-throughput pipelines |
  | **EventBridge** | Not suitable for large data volume or processing |

---

### 3. **Feature Engineering: Kafka Streams**

- **Chosen**: **Kafka Streams**
- **Why**:
  - Native support for **event-time processing**, **joins**, and **windowing**
  - Handles **stateful joins** for auxiliary data (e.g., ZIP → income)
  - Scales horizontally, no cold-starts, and fault-tolerant
  - In-place integration with MSK, no extra setup

- **Alternatives Considered**:
  | Contender | Why Not Chosen |
  |-----------|----------------|
  | **AWS Lambda (Go)** | Fast for stateless transforms, but not built for **stateful joins** and high-throughput streaming (cold start, concurrency limits) |
  | **AWS Flink (KDA)** | Also strong, but adds operational complexity unless Flink team expertise exists |
  | **Step Functions** | More suitable for orchestrated pipelines, not real-time event processing |

---

### 4. **Feature Store / Auxiliary Lookup: Redis**

- **Chosen**: **Redis (AWS ElastiCache)**
- **Why**:
  - Ultra-low latency in-memory store (sub-millisecond)
  - Excellent for **ZIP code-based lookups**, region info, or session state
  - Can double as a **lightweight feature store** if needed

- **Alternatives Considered**:
  | Contender | Why Not Chosen |
  |-----------|----------------|
  | **DynamoDB** | Slower than Redis for real-time lookups (2–10ms typical), but better for persistence |
  | **Aurora/RDS** | Latency too high; not designed for sub-ms lookups or key-value patterns |

---

### 5. **Model Inference: ML REST API**

- **Chosen**: **REST API hosted on SageMaker Endpoint / ECS / Lambda**
- **Why**:
  - Supports any ML framework
  - REST interface allows easy versioning, load balancing, monitoring
  - Deployable on scalable infrastructure (Fargate, ECS, SageMaker)

- **Alternatives Considered**:
  | Contender | Why Not Chosen |
  |-----------|----------------|
  | **gRPC Endpoint** | Faster than REST but not as widely supported or easy to integrate across tools |
  | **Batch Inference** | Doesn’t meet real-time latency goal |
  | **Streaming via WebSocket** | Adds complexity without strong benefit here |

---

### 6. **Optional: Persistent Storage**

- **Chosen Options**:
  - **DynamoDB + DAX**: For real-time persistent key-value access
  - **S3 + Athena**: For batch analytics, dashboards, retraining datasets

- **Why**:
  - **DAX** accelerates DynamoDB to near-Redis speeds for auxiliary or fallback lookups
  - **S3 + Athena** supports SQL-like analytics for audit, BI, and model evaluation

- **Alternatives Considered**:
  | Contender | Why Not Chosen |
  |-----------|----------------|
  | **Redshift** | Good for analytics but slower, and overkill for real-time lookup |
  | **RDS** | Higher operational complexity and slower than S3/Athena for large-scale batch analysis |

---

### 7. **Monitoring and Failure Handling**

- **Chosen**:
  - **CloudWatch**: Logs, metrics, alarms
  - **DLQ (Kafka Dead Letter Topic or SQS)**: For failed event capture

- **Why**:
  - **CloudWatch** supports custom metrics from Kafka Streams, Lambda, Redis, etc.
  - **DLQs** prevent message loss and support retries/debugging

- **Fallback Strategy**:
  - Retry policy with backoff
  - Auto-scaling inference layer (ECS/Fargate/SageMaker)

---

## How This Architecture Ensures <100ms Latency

| Component            | Est. Latency | Optimization Notes |
|---------------------|--------------|--------------------|
| Logstash → Kafka    | 10–30 ms     | Small batch size and flush interval |
| Kafka Streams        | 5–20 ms      | Stateful map/filter/join in-memory |
| Redis Lookup         | <1 ms        | Auxiliary lookup sub-millisecond |
| ML REST Inference    | 30–50 ms     | Use compiled model (e.g., TorchScript) and load balancer |

> Total End-to-End: ~50–90ms under typical load

---

## Summary: Why This Setup

| Need | Solution | Justification |
|------|----------|---------------|
| High-throughput ingestion | Logstash + MSK | Elastic → Kafka is reliable, scalable |
| Real-time processing | Kafka Streams | Stateful + streaming + join support |
| Fast feature lookup | Redis | Sub-millisecond in-memory access |
| Inference | REST API | Framework-agnostic, scalable |
| Monitoring | CloudWatch + DLQ | End-to-end observability and recovery |
| Persistent storage | DynamoDB/DAX or S3 | Optional offline analytics and training |
