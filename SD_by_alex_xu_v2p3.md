

# 📌 System Design: Metrics Monitoring, Ad Click Aggregation & Hotel Reservation

One-line summary: Three system design chapters covering metrics monitoring/alerting, ad click event aggregation at scale, and hotel reservation systems with concurrency handling. | Type: System Design | Depth: Advanced

---

# 📑 Table of Contents

1. [Chapter 5: Metrics Monitoring and Alerting System](#ch5)
2. [Chapter 6: Ad Click Event Aggregation](#ch6)
3. [Chapter 7: Hotel Reservation System](#ch7)

---

# 🌳 Master Concept Tree

```
System Design Chapters
├── Ch5: Metrics Monitoring & Alerting
│   ├── Data Model (time-series)
│   ├── Data Collection (pull vs push)
│   ├── Storage (time-series DBs)
│   ├── Alerting Pipeline
│   └── Visualization
├── Ch6: Ad Click Event Aggregation
│   ├── Data Model & APIs
│   ├── MapReduce Aggregation (DAG)
│   ├── Streaming vs Batching (Lambda/Kappa)
│   ├── Delivery Guarantees (exactly-once)
│   └── Scaling & Fault Tolerance
└── Ch7: Hotel Reservation System
    ├── Data Model (room_type_inventory)
    ├── Concurrency Control (3 locking options)
    ├── Scalability (sharding + caching)
    └── Microservice Data Consistency
```

---

<a id="ch5"></a>
# 🔹 Chapter 5: Metrics Monitoring and Alerting System

## 🔹 Step 1 — Requirements & Scope

- **Purpose** — internal system for a large company (like Facebook/Google) to provide visibility into infrastructure health for high availability and reliability
- **Popular tools** — Datadog, InfluxDB, Prometheus, Grafana, Nagios, Graphite, Splunk, Munin

### Functional Requirements

- Collect **operational system metrics** (not business metrics, not logs, not distributed tracing)
  - Low-level: CPU load, memory usage, disk space
  - High-level: requests per second, running server count of a web pool
- **Alert** on anomalies via email, phone, PagerDuty, webhooks (HTTP endpoints)
- **Visualization** of metrics data

### Non-Functional Requirements

- **Scalability** — accommodate growing metrics and alert volume
- **Low latency** — for dashboard queries and alerts
- **Reliability** — must not miss critical alerts
- **Flexibility** — pipeline should easily integrate new technologies

### Scale & Numbers

| Parameter | Value |
|-----------|-------|
| DAU | 100 million |
| Server pools | 1,000 |
| Machines per pool | 100 |
| Metrics per machine | 100 |
| Total metrics | ~10 million |
| Data retention | 1 year |

### Data Retention Policy (Downsampling)

| Age | Resolution |
|-----|-----------|
| 0–7 days | Raw form |
| 7–30 days | 1 minute resolution |
| 30 days–1 year | 1 hour resolution |

### Out of Scope

- **Log monitoring** — use ELK stack (Elasticsearch, Logstash, Kibana)
- **Distributed system tracing** — e.g., Dapper, Zipkin

---

## 🔹 Step 2 — High-Level Design

### Five Core Components

1. **Data collection** — collect metrics from different sources
2. **Data transmission** — transfer data from sources to the monitoring system
3. **Data storage** — organize and store incoming data
4. **Alerting** — analyze data, detect anomalies, generate alerts to various channels
5. **Visualization** — present data in graphs/charts for pattern identification

### Data Model — Time Series

- Metrics data = **time series** = set of values with associated timestamps
- Each series uniquely identified by its **name** + optional **labels** (key:value pairs)

**Example data point:**

| Field | Value |
|-------|-------|
| metric_name | `cpu.load` |
| labels | `host:i631,env:prod` |
| timestamp | `1613707265` |
| value | `0.29` |

**Line protocol format** (used by Prometheus, OpenTSDB):
```
CPU.load host=webserver01,region=us-west 1613707265 50
```

**Every time series consists of:**

| Component | Type |
|-----------|------|
| Metric name | String |
| Tags/labels | List of key:value pairs |
| Values + timestamps | Array of (value, timestamp) pairs |

### Data Access Pattern

- **Write load is heavy** — ~10 million metrics written per day, collected at high frequency → **write-heavy**
- **Read load is spiky** — visualization and alerting services send bursty queries
- System is under **constant heavy write load** with **spiky read load**

### Data Storage System — Why Time-Series DB

- ❌ **Don't build your own** storage or use general-purpose DB (e.g., MySQL)
- **Why not relational DB:**
  - Not optimized for time-series operations (e.g., moving average requires complicated SQL)
  - Need index for each tag → overhead
  - Doesn't perform well under constant heavy write load
- **Why not NoSQL (Cassandra, Bigtable):**
  - Could work but requires deep internal knowledge to devise scalable schema
  - Not appealing when industrial-scale time-series DBs are readily available

**Time-Series Database Examples:**

| DB | Notes |
|----|-------|
| OpenTSDB | Distributed, based on Hadoop/HBase (adds complexity) |
| MetricsDB | Used by Twitter |
| Amazon Timestream | AWS managed service |
| **InfluxDB** | Most popular, in-memory cache + on-disk storage |
| **Prometheus** | Most popular alongside InfluxDB |

**InfluxDB Benchmarks:**

| vCPU | RAM | IOPS | Writes/sec | Queries/sec | Unique series |
|------|-----|------|-----------|------------|---------------|
| 2-4 cores | 2-4 GB | 500 | < 5,000 | < 5 | < 100,000 |
| 4-6 cores | 8-32 GB | 500-1000 | < 250,000 | < 25 | < 1,000,000 |
| 8+ cores | 32+ GB | 1000+ | > 250,000 | > 25 | > 1,000,000 |

> **Key insight:** Labels should be of **low cardinality** (small set of possible values) — critical for visualization and fast lookup via label indexes.

### High-Level Design Components

```
Metrics Source → Metrics Collector → Time Series DB → Query Service → Visualization System
                                                    → Alerting System → Email/SMS/PagerDuty/Webhooks
```

- **Metrics source** — app servers, SQL databases, message queues, etc.
- **Metrics collector** — gathers and writes data to time-series DB
- **Time-series database** — stores data, provides custom query interface, indexes on labels
- **Query service** — thin wrapper over time-series DB; could be replaced by DB's own query interface
- **Alerting system** — sends notifications to various destinations
- **Visualization system** — graphs/charts (e.g., Grafana)

---

## 🔹 Step 3 — Design Deep Dive

### Metrics Collection

- For counters/CPU usage, **occasional data loss is acceptable** — fire and forget

### Pull vs Push Models

#### Pull Model

- Dedicated **metrics collector** periodically pulls metric values from running applications via HTTP
- Applications expose a `/metrics` endpoint (requires client library)
- **Service Discovery** (etcd, ZooKeeper) provides the list of service endpoints
  - Contains config rules: pulling interval, IP addresses, timeout/retry params
  - Metrics collector registers for change event notifications OR polls periodically

**Scaling pull with multiple collectors:**
- Problem: multiple collectors might pull from the same source → duplicate data
- Solution: **Consistent hash ring** — each collector assigned a range, each server mapped by unique name → one server handled by one collector only

#### Push Model

- **Collection agent** installed on every monitored server
  - Long-running software collecting metrics from services
  - Pushes metrics periodically to the metrics collector
  - Can **aggregate locally** (e.g., counters) before sending → reduces volume
- If push traffic is high and collector rejects → agent keeps small **local buffer** (disk) and resends later
  - Risk: in auto-scaling groups, servers rotated out → data loss if collector falls behind
- Solution: metrics collector in **auto-scaling cluster with load balancer** in front, scales based on CPU load

### 🆚 Pull vs Push Comparison

| Aspect | Pull | Push |
|--------|------|------|
| **Easy debugging** | `/metrics` endpoint can be used to view metrics anytime, even from laptop. **Pull wins** | — |
| **Health check** | If server doesn't respond to pull → know it's down. **Pull wins** | If collector doesn't receive metrics, could be network issues |
| **Short-lived jobs** | — | Jobs may not last long enough to be pulled. **Push wins** (or use push gateways for pull) |
| **Firewall/complex network** | Requires all metric endpoints reachable; problematic in multi-DC | With load balancer + auto-scaling, can receive from anywhere. **Push wins** |
| **Performance** | Typically uses TCP | Typically uses UDP → lower-latency transport. Counterargument: TCP connection overhead is small vs payload |
| **Data authenticity** | Servers defined in config files in advance → guaranteed authentic | Any client can push; fix with whitelisting or authentication |

> **Conclusion:** No clear winner. Large organizations likely need **both**, especially with serverless where you can't install an agent.

---

### Scale the Metrics Transmission Pipeline

- Metrics collector cluster set up for **auto-scaling** (push or pull)
- **Risk:** data loss if time-series DB is unavailable
- **Solution:** Introduce **Kafka** as queuing component between collector and DB

```
Metrics Source → Metrics Collector → Kafka → Consumers (Storm/Flink/Spark) → Time Series DB
```

**Advantages of Kafka:**
- Highly reliable and scalable distributed messaging platform
- **Decouples** data collection from data processing
- Prevents data loss when DB is unavailable by **retaining data in Kafka**

**Scaling through Kafka:**
- Configure number of partitions based on throughput requirements
- Partition by **metric name** → consumers aggregate by metric names
- Further partition with **tags/labels**
- **Categorize and prioritize** metrics → important ones processed first

**Alternative to Kafka:**
- Facebook's **Gorilla** — in-memory time-series DB designed to remain highly available for writes even during partial network failure
- Could be argued as reliable as having intermediate queue

### Where Aggregations Can Happen

| Location | Description | Trade-offs |
|----------|-------------|------------|
| **Collection agent** (client-side) | Simple logic only (e.g., aggregate counter every minute) | Limited capability |
| **Ingestion pipeline** (before storage) | Stream processing (Flink); significantly reduced write volume | Late-arriving events challenging; lose data precision and flexibility (no raw data) |
| **Query side** (after storage) | Aggregate raw data at query time | No data loss; slower queries (computed against whole dataset) |

---

### Query Service

- Cluster of query servers accessing time-series DB
- Handles requests from visualization and alerting systems
- Dedicated servers **decouple** time-series DB from clients → flexibility to change either side
- **Cache layer** added to store query results → reduce DB load

**Case against query service:** Most industrial-scale visual/alerting systems have powerful plugins for well-known time-series DBs. With a good time-series DB, may not need custom caching either.

### Time-Series Database Query Language

- Prometheus and InfluxDB **don't use SQL** — have their own query languages
- Reason: SQL for time-series is **complicated and hard to read**

**Example — Exponential moving average:**
- SQL: ~20 lines of complex nested queries
- Flux (InfluxDB): 4 lines, much easier:

```
from(db:"telegraf")
  |> range(start:-1h)
  |> filter(fn: (r) => r._measurement == "foo")
  |> exponentialMovingAverage(size:-10s)
```

---

### Storage Layer

- Per Facebook research: **85% of queries** are for data collected in the **past 26 hours**
  - A time-series DB that harnesses this property → significant performance impact

### Space Optimization

#### 1. Data Encoding and Compression (Double-Delta Encoding)

- Timestamps like `1610087371` and `1610087381` differ by only 10 seconds
- Delta = 10, which takes only **4 bits** instead of full 32-bit timestamp
- Store: base value + deltas → `1610087371, 10, 10, 9, 11`

#### 2. Downsampling

- Convert high-resolution data to low-resolution to reduce disk usage
- Rules per the retention policy:
  - 7 days: no sampling
  - 30 days: downsample to 1 minute
  - 1 year: downsample to 1 hour

**Example — 10s to 30s rollup:**

| metric | timestamp | hostname | value |
|--------|-----------|----------|-------|
| cpu | 2021-10-24T19:00:00Z | host-a | 10 |
| cpu | 2021-10-24T19:00:10Z | host-a | 16 |
| cpu | 2021-10-24T19:00:20Z | host-a | 20 |

→ Rolled up to:

| metric | timestamp | hostname | avg |
|--------|-----------|----------|-----|
| cpu | 2021-10-24T19:00:00Z | host-a | **15.33** (shown as 19 in source) |

#### 3. Cold Storage

- Storage of **inactive, rarely used data** at much lower financial cost

---

### Alerting System

**Alert flow (7 steps):**

1. **Load config files** to cache servers — rules defined as YAML on disk
   ```yaml
   - name: instance_down
     rules:
     - alert: instance_down
       expr: up == 0
       for: 5m
       labels:
         severity: page
   ```
2. **Alert manager** fetches alert configs from cache
3. Alert manager **calls query service** at predefined intervals. If value violates threshold → alert event created. Alert manager responsibilities:
   - **Filter, merge, and dedupe** alerts (e.g., merge multiple alerts from same instance within short time)
   - **Access control** — restrict alert management to authorized individuals
   - **Retry** — check alert states, ensure notification sent at least once
4. **Alert store** (key-value DB like Cassandra) — keeps state of all alerts: `inactive`, `pending`, `firing`, `resolved`
5. Eligible alerts **inserted into Kafka**
6. **Alert consumers** pull events from Kafka
7. Alert consumers **send notifications** via email, text, PagerDuty, HTTP endpoints

> **Build vs Buy:** Many industrial-scale alerting systems available off-the-shelf with tight integration to popular time-series DBs. Hard to justify building your own.

### Visualization System

- Built on top of the data layer
- Metrics dashboard + alerts dashboard
- **Strong argument for off-the-shelf** (e.g., Grafana) — integrates well with popular time-series DBs

---

### Final Design

```
Metrics Source → Metrics Collector → Kafka → Consumers → Time Series DB
                                                              ↓
                                              Query Service ← Cache
                                              ↓              ↓
                                         Alerting System   Visualization System
                                         ↓
                                    Email/SMS/PagerDuty/Webhooks
```

---

## 📝 Chapter 5 Summary

- **Pull vs push** — no clear winner; large orgs need both
- **Kafka** scales the transmission pipeline, decouples collection from processing
- **Time-series DB** (InfluxDB/Prometheus) — purpose-built for the workload
- **Downsampling** reduces data size over time
- **Build vs buy** — strongly consider off-the-shelf for alerting and visualization
- **Data encoding** (double-delta) significantly compresses time-series data

---

<a id="ch6"></a>
# 🔹 Chapter 6: Ad Click Event Aggregation

## 🔹 Step 1 — Requirements & Scope

### Context: Online Advertising

- Digital advertising uses **Real-Time Bidding (RTB)**: Advertiser → DSP → Ad Exchange → SSP → Publisher
- RTB occurs in **< 1 second**
- **Ad click event aggregation** measures advertising effectiveness → impacts billing
- Key metrics: **CTR** (click-through rate), **CVR** (conversion rate)

### Input Data

- Log files on different servers; events appended to end
- Event attributes: `ad_id`, `click_timestamp`, `user_id`, `ip`, `country`

### Functional Requirements

- Aggregate click count of `ad_id` in the last *M* minutes
- Return **top 100 most clicked** `ad_id` in the last *M* minutes (configurable, aggregated every minute)
- Support **filtering** by `ip`, `user_id`, or `country`

### Non-Functional Requirements

- **Correctness** — data used for RTB and billing
- Handle **delayed or duplicate** events properly
- **Robustness** — resilient to partial failures
- **Latency** — end-to-end a few minutes at most

### Back-of-the-Envelope Estimation

| Parameter | Value |
|-----------|-------|
| Ad clicks/day | 1 billion |
| Total ads | 2 million |
| Growth | 30% year-over-year |
| Avg QPS | 10,000 |
| Peak QPS (5x) | 50,000 |
| Per-event storage | 0.1 KB |
| Daily storage | 100 GB |
| Monthly storage | ~3 TB |

---

## 🔹 Step 2 — High-Level Design

### Query API Design

**API 1:** `GET /v1/ads/{:ad_id}/aggregated_count`

| Param | Description | Type |
|-------|-------------|------|
| `from` | Start minute (default: now - 1 min) | long |
| `to` | End minute (default: now) | long |
| `filter` | Identifier for filtering strategy | long |

Response: `{ ad_id: string, count: long }`

**API 2:** `GET /v1/ads/popular_ads`

| Param | Description | Type |
|-------|-------------|------|
| `count` | Top N most clicked ads | integer |
| `window` | Aggregation window size (M) in minutes | integer |
| `filter` | Filtering strategy identifier | long |

Response: `{ ad_ids: array }`

### Data Model

#### Raw Data

```
[AdClickEvent] ad001, 2021-01-01 00:00:01, user1, 207.148.22.22, USA
```

Fields: `ad_id`, `click_timestamp`, `user_id`, `ip`, `country`

#### Aggregated Data

| ad_id | click_minute | count |
|-------|-------------|-------|
| ad001 | 202101010000 | 5 |
| ad001 | 202101010001 | 7 |

**With filters (star schema):**

| ad_id | click_minute | filter_id | count |
|-------|-------------|-----------|-------|
| ad001 | 202101010000 | 0012 | 2 |
| ad001 | 202101010000 | 0023 | 3 |

**Top N most clicked:**

| Field | Type | Description |
|-------|------|-------------|
| `window_size` | integer | Aggregation window (M minutes) |
| `update_time_minute` | timestamp | Last updated minute granularity |
| `most_clicked_ads` | array | List of ad IDs in JSON format |

### 🆚 Raw Data vs Aggregated Data

| | Raw data only | Aggregated data only |
|---|---|---|
| **Pros** | Full data set; supports filter and recalculation | Smaller data set; fast query |
| **Cons** | Huge storage; slow query | Data loss (derived); e.g., 10 entries aggregated to 1 |

**Recommendation: Store both**
- Raw data for debugging, recalculation, backup → move to cold storage over time
- Aggregated data as active data → tuned for query performance

### Database Choice

- System is **write-heavy** (avg 10K QPS, peak 50K)
- Raw data: read volume low (backup/recalculation) → **Cassandra or InfluxDB** (optimized for writes and time-range queries)
  - Alternative: Amazon S3 with columnar formats (ORC, Parquet, AVRO)
- Aggregated data: **both read and write heavy** (query every minute per ad for 2M ads + written every minute)

### High-Level Design with Async Processing

- Synchronous design is bad — producer/consumer capacity mismatch → OOM errors, unexpected shutdowns
- Solution: **Kafka** (message queue) decouples producers and consumers

```
Log Watcher → Message Queue → Data Aggregation Service → Message Queue → Database Writer → Aggregation DB
                     ↓                                                                              ↓
              Database Writer → Raw Data DB                                              Query Service (Dashboard)
```

**First message queue** stores: `ad_id`, `click_timestamp`, `user_id`, `ip`, `country`

**Second message queue** stores:
1. Ad click counts per minute: `ad_id`, `click_minute`, `count`
2. Top N most clicked per minute: `update_time_minute`, `most_clicked_ads`

> **Why second message queue?** Needed to achieve **end-to-end exactly-once semantics** (atomic commit)

### Aggregation Service — MapReduce / DAG Model

- Break system into small computing units: **Map → Aggregate → Reduce**
- Each node is responsible for one task, sends results downstream

**Map node:** Reads data, filters and transforms. E.g., routes ads with `ad_id % 2 = 0` to Aggregate Node 1, rest to Node 2.
- Why Map node? Input may need cleaning/normalization; same `ad_id` might land in different Kafka partitions

**Aggregate node:** Counts ad click events by `ad_id` in memory every minute (part of Reduce in MapReduce paradigm)

**Reduce node:** Reduces aggregated results from all Aggregate nodes to final result (e.g., merges top-3 from each node → final top-3)

### Three Use Cases

**Use case 1: Aggregate click count**
- Input events partitioned by `ad_id % N` in Map nodes → aggregated by Aggregate nodes → per-ad click count per minute

**Use case 2: Top N most clicked ads**
- Each Aggregate node maintains a **heap data structure** for top N within its partition
- Reduce node merges all partial top-N → final top N

**Use case 3: Data filtering (star schema)**
- Pre-define filtering criteria → aggregate based on them
- Filtering fields = **dimensions**
- Pros: simple, reuses existing aggregation, fast pre-calculated access
- Cons: creates many more buckets/records with many filtering criteria

---

## 🔹 Step 3 — Design Deep Dive

### Streaming vs Batching

| | Services (Online) | Batch (Offline) | Streaming (Near real-time) |
|---|---|---|---|
| **Responsiveness** | Respond quickly | No response needed | No response needed |
| **Input** | User requests | Bounded, finite, large | Unbounded (infinite streams) |
| **Output** | Responses to clients | Materialized views, metrics | Materialized views, metrics |
| **Performance** | Availability, latency | Throughput | Throughput, latency |
| **Example** | Online shopping | MapReduce | Flink |

**Our design uses both:**
- **Stream processing** for real-time aggregation
- **Batch processing** for historical data backup

### Lambda vs Kappa Architecture

- **Lambda** — two processing paths (batch + streaming) → two codebases to maintain
- **Kappa** — single stream processing engine handles both real-time and reprocessing → simpler

> Our design uses **Kappa architecture** — historical data replay also goes through real-time aggregation service

### Data Recalculation

- When a bug is discovered, need to **replay historical data** from raw storage
- Steps:
  1. Recalculation service retrieves from raw data storage (batch job)
  2. Sends to **dedicated** aggregation service (doesn't impact real-time)
  3. Results go to second message queue → update aggregation DB

### Time: Event Time vs Processing Time

| | Pros | Cons |
|---|---|---|
| **Event time** | More accurate (client knows when ad clicked) | Depends on client-side timestamp; could be wrong or malicious |
| **Processing time** | Server timestamp more reliable | Inaccurate if event arrives much later |

**Recommendation:** Use **event time** (data accuracy is critical for billing)

**Handling delayed events — Watermark technique:**
- Extend the aggregation window by a configurable amount (e.g., 15 seconds)
- Catches slightly delayed events
- **Trade-off:** Long watermark = more accurate but higher latency; short watermark = less accurate but lower latency
- Very late events (e.g., 5 hours) handled by **end-of-day reconciliation**

### Aggregation Window Types

| Window | Description | Use Case |
|--------|-------------|----------|
| **Tumbling (fixed)** | Same-length, non-overlapping chunks | Aggregate click count every minute |
| **Sliding** | Window slides across data stream; can overlap | Top N most clicked in last M minutes |

### Delivery Guarantees

- Kafka provides: at-most once, at-least once, exactly-once
- For billing: **exactly-once** is required (few percent difference = millions of dollars)

### Data Deduplication

**Two common sources of duplicates:**

1. **Client-side** — client resends same event (malicious intent handled by ad fraud/risk control)
2. **Server outage** — aggregation node goes down mid-aggregation, upstream hasn't received ACK → same events re-sent

**Deduplication via offset management:**
- Aggregator stores upstream Kafka **offset** in external storage (HDFS/S3)
- Problem: if offset saved before sending downstream, and step 4 fails → data loss (events 100-110 never processed because offset already at 110)
- Fix: Save offset **after** receiving downstream ACK
- But: if Aggregator dies before saving offset after ACK → events sent downstream again
- **Solution:** Steps 4-6 must be in a **distributed transaction** — if any fails, whole transaction rolls back

### Scale the System

Three independent components scaled independently: message queue, aggregation service, database

**Scale the Message Queue (Kafka):**
- **Producers** — no limit on instances
- **Consumers** — rebalancing mechanism adds/removes nodes
  - With hundreds of consumers, rebalance can take **minutes** → do during off-peak
- **Brokers:**
  - Use `ad_id` as **hashing key** for Kafka partitions
  - **Pre-allocate enough partitions** (changing partition count remaps events)
  - **Topic physical sharding** by geography or business type
    - Pros: increased throughput, faster consumer rebalance
    - Cons: extra complexity, maintenance costs

**Scale the Aggregation Service:**
- Option 1: Allocate events with different `ad_id`s to **different threads** (multi-threading)
- Option 2: Deploy on resource providers like **Apache Hadoop YARN** (multi-processing) — more common in practice

**Scale the Database:**
- Cassandra supports **horizontal scaling** via consistent hashing with virtual nodes
- Data evenly distributed with proper replication factor
- Adding nodes → automatic virtual node rebalancing

### Hotspot Issue

- Popular ads clicked much more → some aggregation nodes overloaded
- **Solution:** Allocate more aggregation nodes for popular ads via **resource manager**
  1. Node detects overload (e.g., 300 events but capacity 100)
  2. Applies for extra resources through resource manager
  3. Splits events into groups, each handled by a new node
  4. Results written back to original node
- Advanced: **Global-Local Aggregation** or **Split Distinct Aggregation**

### Fault Tolerance

- Aggregation in memory → lost when node goes down
- **Rebuild** by replaying events from upstream Kafka
- **Faster recovery:** Save **snapshots** of system status (offset + aggregation state like top-N data)
- On failure: bring up new node → recover from latest snapshot → replay events after snapshot from Kafka

### Data Monitoring and Correctness

**Continuous monitoring:**
- **Latency** — track timestamps at each stage, expose as latency metrics
- **Message queue size** — sudden increase → add aggregation nodes (for Kafka: monitor `records-lag`)
- **System resources** — CPU, disk, JVM on aggregation nodes

**Reconciliation:**
- Sort ad click events by event time per partition at **end of day** (batch job)
- Compare with real-time aggregation result
- For higher accuracy: smaller window (e.g., 1 hour)
- Batch result may not match exactly due to late events

### Alternative Design

- Store ad click data in **Hive** with **ElasticSearch** layer for faster queries
- Aggregation in OLAP databases: **ClickHouse** or **Druid**

---

## 📝 Chapter 6 Summary

- **MapReduce/DAG paradigm** for aggregating click events (Map → Aggregate → Reduce)
- **Kafka** decouples producers/consumers, enables exactly-once with atomic commit
- **Kappa architecture** — single processing path for real-time and historical replay
- **Event time** with **watermarks** for handling delayed events
- **Exactly-once delivery** critical for billing accuracy
- **Distributed transactions** for deduplication
- **Hotspot mitigation** via resource manager allocating extra aggregation nodes
- **Snapshot-based fault tolerance** for fast aggregation node recovery
- **End-of-day reconciliation** for data correctness verification

---

<a id="ch7"></a>
# 🔹 Chapter 7: Hotel Reservation System

> Also applicable to: Design Airbnb, flight reservation, movie ticket booking

## 🔹 Step 1 — Requirements & Scope

### Functional Requirements

- Show hotel-related page
- Show hotel room-related detail page
- **Reserve a room**
- **Admin panel** — add/remove/update hotel or room info
- Support **10% overbooking** (sell more rooms than available, anticipating cancellations)
- Hotel prices are **dynamic** (depend on expected occupancy per day)
- Customers **pay in full** at reservation time
- Customers can **cancel** reservations
- Booking via hotel website or app only

### Non-Functional Requirements

- **High concurrency** — peak season/big events, many users booking same room
- **Moderate latency** — a few seconds acceptable for reservation processing

### Back-of-the-Envelope Estimation

| Parameter | Value |
|-----------|-------|
| Hotels | 5,000 |
| Total rooms | 1 million |
| Occupancy rate | 70% |
| Average stay | 3 days |
| Daily reservations | ~240,000 |
| Reservation TPS | ~3 |

**QPS Funnel (10% conversion at each step):**

| Step | QPS |
|------|-----|
| View hotel/room detail | 300 |
| Order booking page | 30 |
| Reserve rooms | 3 |

---

## 🔹 Step 2 — High-Level Design

### API Design (RESTful)

**Hotel APIs:**

| API | Detail |
|-----|--------|
| `GET /v1/hotels/ID` | Get hotel info |
| `POST /v1/hotels` | Add hotel (staff only) |
| `PUT /v1/hotels/ID` | Update hotel (staff only) |
| `DELETE /v1/hotels/ID` | Delete hotel (staff only) |

**Room APIs:** Similar CRUD pattern with `/v1/hotels/ID/rooms/ID`

**Reservation APIs:**

| API | Detail |
|-----|--------|
| `GET /v1/reservations` | Reservation history of logged-in user |
| `GET /v1/reservations/ID` | Get reservation detail |
| `POST /v1/reservations` | Make new reservation |
| `DELETE /v1/reservations/ID` | Cancel reservation |

**POST /v1/reservations request body:**
```json
{
  "startDate": "2021-04-28",
  "endDate": "2021-04-30",
  "hotelID": "245",
  "roomTypeID": "U12354673389",
  "reservationID": "13422445"
}
```

> `reservationID` serves as **idempotency key** to prevent double booking

### Data Model — Why Relational Database

| Reason | Explanation |
|--------|-------------|
| Read-heavy workflow | Visitors >> reservation makers; RDBMS works well for reads |
| ACID guarantees | Prevents negative balance, double charge, double reservations |
| Clear data model | Stable relationships between entities (hotel, room, room_type) |

### Schema Design (Initial → Improved)

**Initial issue:** Schema had `room_id` in reservation → but hotels reserve by **room type**, not specific room (room numbers assigned at check-in)

**Key tables after improvement:**

- `hotel` — hotel_id (PK), name, address, location
- `room` — room_id (PK), room_type_id, floor, number, hotel_id, name, is_available
- `room_type_rate` — hotel_id (PK), date (PK), rate
- `reservation` — reservation_id (PK), hotel_id, room_type_id, start_date, end_date, status, guest_id
- `room_type_inventory` — hotel_id, room_type_id, date, total_inventory, total_reserved
- `guest` — guest_id (PK), first_name, last_name, email

**Reservation status state machine:**
```
Pending → Paid → Refunded
Pending → Canceled
Pending → Rejected
```

### room_type_inventory Table (Critical)

| Column | Description |
|--------|-------------|
| `hotel_id` | Hotel identifier |
| `room_type_id` | Room type identifier |
| `date` | Single date |
| `total_inventory` | Total rooms minus those taken off for maintenance |
| `total_reserved` | Total rooms booked for this hotel/type/date |

- **Composite PK:** (hotel_id, room_type_id, date)
- Pre-populated for all future dates within 2 years (daily scheduled job)
- Storage estimate: 5,000 hotels × 20 types × 2 years × 365 days = **73 million rows** → single DB sufficient

### Reservation Check Logic

```sql
SELECT date, total_inventory, total_reserved
FROM room_type_inventory
WHERE room_type_id = ${roomTypeId} AND hotel_id = ${hotelId}
AND date BETWEEN ${startDate} AND ${endDate}
```

For each entry:
```
if (total_reserved + numberOfRoomsToReserve <= total_inventory) → available
```

**With 10% overbooking:**
```
if (total_reserved + numberOfRoomsToReserve <= 110% * total_inventory)
```

### High-Level Design — Microservice Architecture

```
User → CDN → Public API Gateway → Hotel Service (+ Cache + Hotel DB)
                                 → Rate Service (+ Rate DB)
                                 → Reservation Service (+ Reservation DB)
                                 → Payment Service (+ Payment DB)
Admin → Internal API → Hotel Management Service
```

Key services:
- **Hotel Service** — hotel/room info (static, easily cached)
- **Rate Service** — room rates for future dates (dynamic pricing)
- **Reservation Service** — handles reservations, tracks room inventory
- **Payment Service** — executes payment, updates status to `paid` or `rejected`
- **Hotel Management Service** — admin-only features (view, reserve, cancel)

Inter-service communication: **gRPC** (modern high-performance RPC framework)

---

## 🔹 Step 3 — Design Deep Dive

### Concurrency Issues

**Problem 1: Same user clicks "book" multiple times**

Solutions:
- **Client-side:** Gray out/disable submit button → not reliable (JS can be disabled)
- **Idempotent API:** Use `reservation_id` as idempotency key (primary key of reservation table)
  1. Generate reservation order → unique `reservation_id` from global ID generator
  2. Show confirmation page with `reservation_id`
  3. On submit, `reservation_id` included in request
  4. Second submit → **unique constraint violation** on primary key → rejected

**Problem 2: Multiple users book same room type simultaneously (only 1 left)**

Race condition example:
- Both users read `total_reserved = 99`, `total_inventory = 100`
- Both see 1 room left, both try to reserve
- Both succeed → overbooking beyond acceptable limit

### 🆚 Three Locking Solutions

#### Option 1: Pessimistic Locking

- `SELECT ... FOR UPDATE` locks rows returned by query
- Transaction 2 **waits** for Transaction 1 to finish

**Pros:**
- Prevents updating stale data
- Easy to implement, serializes updates
- Useful when data contention is **heavy**

**Cons:**
- **Deadlocks** may occur with multiple locked resources
- **Not scalable** — long locks block other transactions
- Significant impact on DB performance

> **Not recommended** for hotel reservation due to scalability limitations

#### Option 2: Optimistic Locking

- Add `version` column to table
- Read version → modify → write with `version + 1`
- DB validates: new version must exceed current by 1
- If validation fails → abort and retry

**Pros:**
- Prevents editing stale data
- No database locking — logic handled by application
- Good when data contention is **low**

**Cons:**
- Performance drops dramatically when concurrency is **high**
- Many clients read same count → only one succeeds → rest retry → frustrating UX

> **Good option** for hotel reservation (QPS for reservations is usually low)

#### Option 3: Database Constraints

- Add constraint: `CHECK (total_inventory - total_reserved >= 0)`
- If `total_reserved` would exceed `total_inventory`, constraint violated → transaction rolled back

**Pros:**
- Easy to implement
- Works well when data contention is minimal

**Cons:**
- High contention → high failure volume, frustrating UX
- Constraints not easily version-controlled
- Not all databases support constraints (portability issue)

> **Another good option** for hotel reservation (low contention)

### Scalability

**When system is used at booking.com/expedia scale (1000x higher QPS):**
- All services are **stateless** → easily expanded by adding servers
- **Database is the bottleneck**

#### Database Sharding

- Shard by `hotel_id` (most queries filter by hotel)
- `hash(hotel_id) % number_of_servers`
- Example: QPS 30,000 across 16 shards → 1,875 QPS per shard (within MySQL capacity)

#### Caching (Redis)

- Only current and future inventory data matters → Redis with **TTL** and **LRU eviction**
- Move check inventory + reserve room logic to cache layer
- Most ineligible requests blocked by inventory cache → only small percentage hits DB

**Cache key structure:**
```
key: hotelID_roomTypeID_{date}
value: number of available rooms
```

**Cache update flow:**
1. Query inventory → reads from **inventory cache**
2. Update inventory → writes to **inventory DB first**, then propagated to cache **asynchronously**
   - Via application code or **CDC (Change Data Capture)** using Debezium

> **Key insight:** Cache-DB inconsistency **doesn't matter** as long as database does **final inventory validation**. User might see room available in cache, but DB rejects → error message → user refreshes → sees updated data

**Pros:** Reduced DB load, high read performance
**Cons:** Hard to maintain cache-DB consistency; must consider UX impact

### Data Consistency Among Microservices

**Monolithic approach:** Single shared DB → wrap operations in single transaction → ACID guaranteed

**Pure microservice approach:** Each service has its own DB → single logical operation spans multiple services → **cannot use single transaction**

**Solutions:**

| Approach | Description |
|----------|-------------|
| **Two-phase commit (2PC)** | Database protocol for atomic commit across multiple nodes. All succeed or all fail. **Blocking** — single node failure blocks progress. Not performant. |
| **Saga** | Sequence of local transactions. Each publishes message triggering next step. On failure → **compensating transactions** undo preceding changes. Relies on **eventual consistency**. |

> **Pragmatic decision:** Store reservation and inventory data in **same relational database** under Reservation Service → leverage ACID without added complexity of distributed transactions

---

## 📝 Chapter 7 Summary

- Users reserve **room types**, not specific rooms (room numbers assigned at check-in)
- `room_type_inventory` table with composite PK (hotel_id, room_type_id, date) is central to the design
- **10% overbooking** implemented via simple check: `total_reserved + N <= 110% * total_inventory`
- **Idempotency key** (`reservation_id`) prevents double booking from same user
- For concurrent users: **optimistic locking** or **database constraints** preferred over pessimistic locking
- **Scalability:** database sharding by `hotel_id`, Redis cache for inventory with async DB updates
- **Microservice data consistency:** 2PC or Saga patterns exist, but pragmatic approach stores related tables in same DB

---

# 📖 Key Definitions

| Term | Meaning | Context |
|------|---------|---------|
| **Time-series data** | Set of values with associated timestamps, identified by name + labels | Metrics monitoring |
| **Line protocol** | Common input format: `metric_name labels timestamp value` | Prometheus, OpenTSDB |
| **Downsampling** | Converting high-resolution data to low-resolution | Storage optimization |
| **Double-delta encoding** | Store deltas between consecutive timestamps instead of absolute values | Data compression |
| **Watermark** | Extension of aggregation window to capture slightly delayed events | Stream processing |
| **Tumbling window** | Fixed-size, non-overlapping time window | Aggregate per minute |
| **Sliding window** | Window that slides across data stream, can overlap | Top N in last M minutes |
| **Lambda architecture** | Dual processing paths (batch + streaming) | Two codebases |
| **Kappa architecture** | Single stream processing engine for both real-time and reprocessing | Simpler design |
| **Star schema** | Filtering fields (dimensions) pre-computed in aggregation | Data warehousing |
| **Reconciliation** | Comparing different data sets to ensure integrity | End-of-day batch vs real-time |
| **Idempotency key** | Unique key ensuring API call produces same result regardless of repetition | Prevent double booking |
| **Pessimistic locking** | Lock record on first access; others wait | `SELECT ... FOR UPDATE` |
| **Optimistic locking** | Allow concurrent reads; validate version on write | Version number column |
| **CDC (Change Data Capture)** | Reads data changes from database and applies to another system | Debezium → Redis |
| **2PC (Two-phase commit)** | Atomic commit across multiple nodes; blocking protocol | Distributed transactions |
| **Saga** | Sequence of local transactions with compensating rollbacks | Eventual consistency |

---

# ⚡ Quick-Recall Points

- InfluxDB with 8 cores/32GB RAM handles **> 250,000 writes/sec**
- Facebook research: **85% of queries** are for data from the **past 26 hours**
- Ad click aggregation needs **exactly-once delivery** (few % difference = millions of dollars)
- Kafka consumer rebalance with hundreds of consumers can take **several minutes**
- Hotel inventory: 5,000 hotels × 20 types × 730 days = **73 million rows** (single DB sufficient)
- Hotel reservation average TPS is only **~3** (very low, but must handle peak season surges)
- For metrics: write load is **constant and heavy**, read load is **spiky**
- Labels in time-series DBs should have **low cardinality** for efficient indexing
- In hotel systems, users reserve **room types**, not specific rooms
- Optimistic locking performance drops dramatically when **concurrency is high**

---

# 🔗 Relationships & Dependencies

- **Kafka** appears in all three systems as the central decoupling/buffering mechanism
- **Time-series DBs** (Ch5) vs **Cassandra/RDBMS** (Ch6/7) — data nature drives DB choice
- **Pull vs Push** debate (Ch5) has no winner → mirrors the **event time vs processing time** tradeoff (Ch6)
- **Consistent hashing** used for both metrics collector assignment (Ch5) and Cassandra scaling (Ch6/Ch7)
- **Exactly-once semantics** (Ch6) requires distributed transactions, similar to **2PC/Saga** patterns (Ch7)
- **Overbooking** (Ch7) is analogous to **watermark tolerance** (Ch6) — accepting imprecision within bounds
- All three systems use **caching** strategically: query results (Ch5), inventory (Ch7), but NOT for raw aggregation (Ch6)

---

# ⚠️ Mistakes & Misconceptions

| ❌ Wrong | ✅ Right | 💬 Why confused |
|---------|---------|----------------|
| Use MySQL for metrics storage | Use purpose-built time-series DB | SQL works in theory but requires expert tuning and can't handle constant write load at scale |
| Hotel reservation reserves specific rooms | Reserves room **types**; room number assigned at check-in | Airbnb model (specific listing) is different from hotel model |
| Pessimistic locking is always safest | It causes deadlocks and doesn't scale | "Safe" ≠ "best"; optimistic locking is preferred when contention is low |
| Cache-DB inconsistency for inventory is critical | It's acceptable as long as DB does final validation | Cache is "best effort" filter; DB is source of truth |
| Lambda architecture is always better | Kappa is simpler (single processing path) | Lambda has two codebases; Kappa reuses same streaming engine for replay |
| Save Kafka offset before sending downstream | Save offset **after** receiving downstream ACK | Otherwise, events lost if aggregation fails after offset saved |

---

# 🎯 Actionable Takeaways

1. **Always clarify scope** — distinguish between metrics monitoring vs log monitoring vs distributed tracing
2. **Choose purpose-built storage** — don't force general-purpose DBs for specialized workloads
3. **Decouple with message queues** — Kafka between every major component for resilience and independent scaling
4. **Use idempotency keys** for any financial/critical operations to prevent duplicate processing
5. **Prefer optimistic locking or DB constraints** when data contention is low (typical for reservations)
6. **Store both raw and aggregated data** — raw for debugging/recalculation, aggregated for queries
7. **Pre-allocate Kafka partitions** — changing partition count disrupts key-based routing
8. **Use event time with watermarks** for stream processing when accuracy matters
9. **End-of-day reconciliation** catches what real-time processing misses
10. **Build vs Buy** — strongly consider off-the-shelf for alerting, visualization, and OLAP aggregation
