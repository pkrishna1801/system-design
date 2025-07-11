# Real-Time AWS Data Pipeline for Clickstream Processing

This pipeline ingests clickstream data from Elasticsearch, processes it in real time, and sends model-ready features to an ML model hosted behind a REST API. It’s optimized for sub-100ms latency for an assumed traffic of 50k events per second.

---

## Pipeline Overview

Elasticsearch → Custom Producer → Kinesis Data Streams → Kinesis Data Analytics (Flink) → Redis (Lookup) → Kinesis Data Streams (Enriched) → Lambda → ML REST API → DynamoDB

CloudWatch monitors Kafka, Kafka Streams, Redis, API, and DynamoDB. A Dead Letter Queue is used to handle failed or malformed events.

---

##  Component Breakdown

###  Elasticsearch → Custom Producer
The custom producer extracts structured clickstream data from Elasticsearch and provides better control than Logstash for high-throughput pipelines. It Supports JSON normalization, timestamp adjustments, and schema validation before sending to Kinesis

###  Custom Producer → Kinesis Data Streams
* Kinesis handles real-time, durable, ordered ingestion at massive scale.
* Elastic scaling and enhanced fan-out (EFO) support ensures low-latency delivery to consumers.

Approx latency: 5–10ms.

### Kinesis Data Analytics (Flink)

* Apache Flink runs within Kinesis Analytics to compute features:

* Time-of-day, day-of-week

* Session-based aggregations

* Joins with static data (e.g., IP geolocation, zip code)

* Stateless and stateful operations supported.

* Output written to a second enriched Kinesis stream.

### Redis (Lookup)

Flink pulls reference/auxiliary data from Redis during stream processing (e.g., metadata per zip code).

High-speed in-memory lookups ensure <5ms latency.

Keeps Flink stateless and lean.

### Kinesis (Enriched) → Lambda
Lambda processes enriched records in batches
Performs:

Record validation

JSON transformation for the ML model

API request assembly

### ML REST API

###  Predictions → DynamoDB
Stores:

Prediction

Enriched features

Timestamps, metadata

Chosen for:

High write throughput

Single-digit ms reads

TTL support for data expiry

Predictions and metadata are stored in DynamoDB for durability and downstream usage. It scales well with write-heavy workloads and provides fast querying capabilities. S3 was considered but is better suited for batch storage or archival purposes.

---

## Latency Estimates

| Step                          | Approx Latency |
| ----------------------------- | -------------- |
| Custom Producer → Kinesis     | \~5–10ms       |
| Kinesis → Flink               | \~10–20ms      |
| Redis lookup within Flink     | \~5ms          |
| Kinesis (Enriched) → Lambda   | \~5ms          |
| Lambda → ML API (batch)       | \~50ms         |
| API response → DynamoDB write | \~10ms         |
| **Total**                     | **\~85–95ms**  |


Leaves enough headroom within the 100ms budget.

---

##  Monitoring and Failure Handling

- **CloudWatch** is configured to monitor:

  - Kafka consumer lag
  - Kafka Streams exception rates
  - Redis latency and availability
  - API error rates and latencies
  - DynamoDB throttles or capacity issues

- **DLQ (Dead Letter Queue)** handles malformed or failed records coming out of Kafka Streams for later inspection.

- If Redis is temporarily unavailable, Kafka Streams can either fall back to cached data or flag the record for retry or DLQ.

---

## Alternatives Considered

| Component    | Used                 | Alternatives          | Rationale                                                                                   |
| ------------ | -------------------- | --------------------- | ------------------------------------------------------------------------------------------- |
| Ingestion    | Kinesis Data Streams | MSK (Kafka), SQS      | Kinesis is AWS-native, serverless, and easily scales with EFO for low latency               |
| Processing   | Flink on KDA         | Lambda, Kafka Streams | Flink handles stateful streaming + joins better than Lambda; no need for self-managed Kafka |
| Lookup Store | Redis                | DynamoDB              | Redis provides in-memory lookup performance critical for sub-100ms latency                  |
| Inference    | REST API (FastAPI)   | SageMaker Batch       | REST gives low-latency response; SageMaker batch adds delay and is better for async jobs    |
| Storage      | DynamoDB             | S3, Aurora            | DynamoDB offers single-digit ms write latency and scales easily                             |

---

##  Summary

This setup uses Kafka as the streaming backbone, Kafka Streams for feature generation and enrichment, Redis for fast auxiliary data access, and DynamoDB for result storage. Keeping Redis out of the prediction path improves reliability and reduces latency, while CloudWatch and a DLQ ensure visibility into failures.

The system is horizontally scalable, low-latency, and production-ready under real-time constraints.
