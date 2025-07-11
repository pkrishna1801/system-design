# Real-Time AWS Data Pipeline for Clickstream Processing

This pipeline ingests clickstream data from Elasticsearch, processes it in real time, and sends model-ready features to an ML model hosted behind a REST API. It’s optimized for sub-100ms latency for an assumed traffic of 50k events per second.

---

## Pipeline Overview

Elasticsearch → Data Ingestion → Kinesis Data Streams → Stream Processing → Enrichment/Cache → Queue → ML API → Storage


![Pipeline](image%20(4).png)


CloudWatch is used throughout the pipeline for monitoring and alerting on all critical components to ensure system reliability and performance.


---

##  Component Breakdown

###  Elasticsearch → Custom Producer
 A custom producer is used to extract data from Elasticsearch in real time. Unlike Logstash, which is more generic, this custom producer offers greater flexibility for high-throughput scenarios. It Supports JSON normalization, timestamp adjustments, and schema validation before sending to Kinesis

###  Custom Producer → Kinesis Data Streams
a fully managed and horizontally scalable stream ingestion service. Kinesis is responsible for ingesting and distributing the data in real time. It supports enhanced fan-out (EFO), which allows consumers like Flink to receive data with low latency and high throughput. This step serves as the primary ingestion buffer, providing ordering guarantees, durability, and built-in integration with downstream AWS analytics services.

### Kinesis Data Analytics (Flink)

ingested data is then consumed by Kinesis Data Analytics, where Apache Flink jobs perform real-time processing. Flink is used here to extract and compute features such as hour of the day, day of the week, and session duration. It also handles event enrichment using metadata such as zip-code-based location or behavioral tags. The advantage of using Flink over Lambda or Kafka Streams lies in its powerful support for both stateless and stateful stream transformations, windowing, and dynamic joins. This allows the pipeline to remain responsive and accurate even under evolving data conditions.

### Redis (Lookup)

During stream processing in Flink, auxiliary data such as zip-code-to-region mappings or customer segments is fetched from Redis. Redis is chosen for its sub-millisecond response time, allowing Flink to perform lookups without compromising throughput or latency. Rather than persisting feature data in Redis, the system treats it purely as a fast, in-memory lookup table. This approach helps minimize state management within Flink

### Kinesis (Enriched) → Lambda
he processed events are written to a second Kinesis stream and consumed by an AWS Lambda function. Lambda is configured with batch processing enabled, which allows it to process groups of records efficiently. Within Lambda, additional tasks such as schema validation, field filtering, and transformation into the ML API’s input format are performed. It also handles retries, error logging, and asynchronous communication with the ML inference endpoint.

### ML REST API

###  Predictions → DynamoDB
Finally, the output from the ML API, including predictions and associated metadata, is written to DynamoDB. DynamoDB is selected for its ability to handle high write volumes with single-digit millisecond latency. It supports partitioning and indexing for efficient querying

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

  - Kinesis iterator age, shard health
  - Lambda duration, error rate, and throttles
  - Redis connection failures or latency spikes
  - API error rates and latencies
  - DynamoDB throttles or capacity issues

- **DLQ (Dead Letter Queue)** can be implemented at Flink, Lambda, and DynamoDB stages using Kinesis error streams, SQS queues, or S3 for failed record capture and replay.

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

This real-time, AWS-native architecture processes high-throughput clickstream data from Elasticsearch, enriches it with auxiliary data, performs fast inference, and logs results—all within ~90ms. It combines Kinesis, Flink, Redis, and Lambda for streaming, and uses DynamoDB for fast persistence. CloudWatch ensures visibility, while Redis lookups and batch Lambda processing help stay well within latency budgets.

The system is scalable, low-latency, and fault-tolerant, ready for production workloads up to 50,000 events/sec and beyond
