

# 📌 System Design: Google Maps & Distributed Message Queue

One-line summary: Two system design deep-dives covering Google Maps (location updates, navigation, map rendering) and Distributed Message Queue (Kafka-like system with topics, partitions, replication, and delivery semantics) | Type: System Design | Depth: Advanced

---

# 📑 Table of Contents

1. [Chapter 3: Google Maps](#chapter-3-google-maps)
   - [Requirements & Scope](#requirements--scope)
   - [Map 101 — Foundational Concepts](#map-101--foundational-concepts)
   - [Back-of-the-Envelope Estimation](#back-of-the-envelope-estimation)
   - [High-Level Design](#high-level-design-google-maps)
   - [Design Deep Dive — Data Model](#design-deep-dive--data-model)
   - [Design Deep Dive — Services](#design-deep-dive--services)
   - [Adaptive ETA & Rerouting](#adaptive-eta--rerouting)
   - [Delivery Protocols](#delivery-protocols)
2. [Chapter 4: Distributed Message Queue](#chapter-4-distributed-message-queue)
   - [Requirements & Scope](#requirements--scope-1)
   - [Messaging Models](#messaging-models)
   - [Topics, Partitions, and Brokers](#topics-partitions-and-brokers)
   - [Consumer Groups](#consumer-groups)
   - [High-Level Architecture](#high-level-architecture)
   - [Data Storage — WAL](#data-storage--wal)
   - [Message Data Structure](#message-data-structure)
   - [Producer Flow](#producer-flow)
   - [Consumer Flow](#consumer-flow)
   - [Consumer Rebalancing](#consumer-rebalancing)
   - [State & Metadata Storage](#state--metadata-storage)
   - [Replication & In-Sync Replicas](#replication--in-sync-replicas)
   - [Scalability](#scalability)
   - [Data Delivery Semantics](#data-delivery-semantics)
   - [Advanced Features](#advanced-features)

---

# Chapter 3: Google Maps

## Requirements & Scope

### Functional Requirements
- **User location update** — periodically record user's position
- **Navigation service** — including ETA (Estimated Time of Arrival)
- **Map rendering** — display map tiles on mobile devices
- Support **different travel modes** (driving, walking, bus, etc.)
- **Traffic conditions** considered for accurate time estimation
- Multi-stop directions acknowledged but **not in scope**
- Business places and photos **not in scope**
- Primary device: **mobile phones**

### Non-Functional Requirements
- **Accuracy** — users must not receive wrong directions
- **Smooth navigation** — client-side map rendering must be smooth
- **Data and battery usage** — minimize for mobile devices (critical constraint)
- General **availability and scalability**

### Key Numbers
- **1 billion DAU** (as of March 2021)
- **99% world coverage**
- **25 million daily updates** of location information
- Road data: **terabytes (TBs)** of raw data from external sources

---

## Map 101 — Foundational Concepts

### 📖 Key Definitions

| Term | Meaning | Example/Context |
|------|---------|-----------------|
| **Latitude (Lat)** | How far north or south a point is | 37.423021 |
| **Longitude (Long)** | How far east or west a point is | -122.083739 |
| **Map Projection** | Translating 3D globe points to a 2D plane | Almost all distort actual geometry |
| **Web Mercator** | Modified Mercator projection used by Google Maps | Google's chosen projection |
| **Geocoding** | Converting addresses → lat/lng coordinates | "1600 Amphitheatre Parkway" → (37.423021, -122.083739) |
| **Reverse Geocoding** | Converting lat/lng → human-readable address | Opposite of geocoding |
| **Geohashing** | Encoding geographic area into short string of letters/digits | Recursively divides earth into sub-grids |
| **Map Tiling** | Breaking world into smaller tiles instead of one large image | Client downloads only relevant tiles |
| **Routing Tiles** | Binary files of road data for graph-based pathfinding | Different from map tiles (PNG images) |

### Geohashing Mechanics
- Depicts earth as flattened surface (initial: 20,000km × 10,000km)
- First division → 4 grids of 10,000km × 5,000km (labeled 00, 01, 10, 11)
- Recursively divides each grid into 4 sub-grids
- Each sub-grid represented by string of numbers (0–3)
- Continues until grid reaches size threshold
- **Used for map tiling** in this design

### Map Rendering Basics
- World broken into **smaller tiles** (not one large image)
- Client downloads **only relevant tiles** for current area and stitches them like a mosaic
- **Different tile sets at different zoom levels** → client chooses appropriate set
- Each tile: **256 × 256 pixels**
- Example: zoomed all the way out → 1 tile for entire world at lowest zoom level

### Road Data Processing for Navigation

```
Graph Structure:
- Nodes = Intersections
- Edges = Roads
```

- Routing algorithms: variations of **Dijkstra's** or **A* pathfinding**
- Performance extremely sensitive to **graph size**
- Entire world as single graph → too much memory, too slow
- **Solution: Routing tiles** — divide world into small grids using geohashing
  - Each grid → small graph (nodes + edges within that area)
  - Each tile holds **references to connected tiles**
  - Algorithms stitch together bigger road graph by traversing interconnected tiles
  - **Loaded on demand** → reduces memory consumption

> **Key distinction:** Map tiles are PNG images; Routing tiles are binary files of road data. Both cover geographical grid areas.

### Hierarchical Routing Tiles
- **Three levels of detail** for efficient cross-distance routing:

| Level | Tile Size | Contains | Use Case |
|-------|-----------|----------|----------|
| Most detailed | Small | Local roads only | Short-distance navigation |
| Medium | Bigger | Arterial roads connecting districts | Medium-distance |
| Least detailed | Large areas | Major highways connecting cities/states | Cross-country routing |

- Edges can connect tiles **at different zoom levels**
  - Example: freeway entrance from local street A → freeway F creates reference from small tile node to big tile node
- **Why hierarchical?** Running routing against highly detailed street-level tiles for cross-country would produce too large a graph

---

## Back-of-the-Envelope Estimation

### Storage Usage

**Three types of data to store:**
1. Map of the world (tiles)
2. Metadata (negligible per tile)
3. Road info (TBs → routing tiles also TBs)

**Map tile storage calculation:**

| Zoom Level | Number of Tiles |
|------------|----------------|
| 0 | 1 |
| 1 | 4 |
| 10 | ~1 million |
| 15 | ~1 billion |
| 21 (highest) | ~4.4 trillion |

- At zoom level 21: ~4.4 trillion tiles × 100KB each = **440PB**
- ~90% of world surface is uninhabited (oceans, deserts, mountains) → highly compressible
- Reduce by 80–90% → **44 to 88PB** → round to **~50PB** for highest zoom
- Each lower zoom level: tiles reduce by **4x** (half in each direction)
- Total across all zoom levels: `50 + 50/4 + 50/16 + 50/64 + ... ≈ 67PB`
- **~100PB total** for all map tiles at all zoom levels

### Server Throughput

**Two request types:**
1. **Navigation requests** — initiate navigation session
2. **Location update requests** — sent during navigation

**Location update QPS calculation:**
- 1 billion DAU × 35 min/week navigation = **5 billion minutes/day**
- Sending GPS every second → 5B × 60 = 300 billion requests/day = **3 million QPS**
- **Batching every 15 seconds** → 3M / 15 = **200,000 QPS**
- **Peak QPS** (5× average) = **1 million QPS**

---

## High-Level Design (Google Maps)

### 🌳 Component Tree

```
High-Level Design
├── Mobile User
│   ├── → CDN (map tiles)
│   └── → Load Balancer
│       ├── Navigation Service
│       │   ├── Geocoding DB
│       │   └── Routing Tiles (Object Storage)
│       └── Location Service
│           └── User Location DB
└── CDN ← Precomputed Map Images (Origin)
```

### Three Core Services

#### 1. Location Service
- Records user location updates
- Client sends updates every `t` seconds (configurable)
- **Benefits of periodic updates:**
  - Monitor live traffic
  - Detect new/closed roads
  - Analyze user behavior for personalization
  - Near real-time ETA updates and rerouting
- **Batching optimization:** GPS recorded every second, sent in batch every **15 seconds**
  - Significantly reduces total update traffic
- **Database choice:** Write-heavy → needs high-write-volume, horizontally scalable DB → **Cassandra**
- Also logs to **Kafka** for downstream stream processing
- **Protocol:** HTTP with **keep-alive** option (efficient)

```
POST /v1/locations
Parameters:
  locs: JSON encoded array of (latitude, longitude, timestamp) tuples
```

#### 2. Navigation Service
- Finds reasonably fast route from A to B
- Tolerates some latency; **accuracy is critical**
- Route doesn't have to be fastest, but must not be wrong

```
GET /v1/nav?origin=1355+market+street,SF&destination=Disneyland
```

- Response includes: distance, duration, start/end locations, polyline, geocoded waypoints, travel mode
- Reroute and traffic changes handled by **Adaptive ETA service** (deep dive)

#### 3. Map Rendering

**Two options for serving tiles:**

| Aspect | Option 1: Dynamic | Option 2: Pre-generated (Chosen) |
|--------|-------------------|----------------------------------|
| How | Server builds tiles on-the-fly | Static pre-generated tiles at each zoom |
| Server load | Huge load | Minimal (served by CDN) |
| Caching | Hard to cache (infinite combos) | Highly cacheable (static files) |
| Scalability | Poor | Excellent |

**Chosen approach (Option 2):**
- Pre-generated tiles using geohashing for grid subdivision
- Each tile has unique geohash → tile URL constructed from geohash
- Example URL: `https://cdn.map-provider.com/tiles/9q9hvu.png`
- Served via **CDN** from nearest POP (Point of Presence)
- CDN flow: client → CDN (cache miss → origin server → cache → return) → subsequent requests served from cache

**Client fetches tiles when:**
- User zooms/pans map viewport
- User moves out of current tile during navigation

**Data usage calculation:**
- At 30km/h, 200m × 200m blocks, 100KB per image
- 1km × 1km = 25 images = 2.5MB
- 30km/h → **75MB/hour** or **1.25MB/minute**

**CDN throughput:**
- 5 billion nav minutes/day × 1.25MB = 6.25 billion MB/day
- = 62,500 MB/second total
- With ~200 POPs → ~312 MB/sec per POP (manageable)

**Tradeoff: Client-side vs Server-side tile URL construction:**

| Approach | Pros | Cons |
|----------|------|------|
| Client-side geohash | Simple, no server call | Hardcoded algorithm, risky to change |
| Map Tile Service (intermediary) | Operational flexibility, easy to update | Extra service hop |

**Alternative flow with Map Tile Service:**
1. Client calls map tile service for tile URLs
2. Load balancer → map tile service
3. Service returns **9 URLs** (current tile + 8 surrounding)
4. Client downloads tiles from CDN

---

## Design Deep Dive — Data Model

### Four Data Types

#### 1. Routing Tiles
- Initial road data from external sources (TBs)
- **Routing tile processing service** — periodic offline pipeline transforms raw road data into routing tiles
- Three sets at different resolutions (local roads, arterial roads, highways)
- Each tile: list of graph nodes/edges + references to other connected tiles
- **Storage:** Object storage (S3) — not database
  - Too many tiles for in-memory adjacency lists
  - Database features unnecessary; expensive for simple storage
  - Serialize adjacency lists to binary files
  - Organize by geohash for fast lat/lng lookup
  - Cache aggressively on routing service

#### 2. User Location Data
- Write-heavy → **Cassandra** (NoSQL, column-oriented)
- Key: `(user_id, timestamp)` → value: lat/lng pair
- `user_id` = partition key, `timestamp` = clustering key
- Efficient retrieval of location data for specific user within time range

| user_id | timestamp | user_mode | driving_mode | location |
|---------|-----------|-----------|--------------|----------|
| 101 | 1635740977 | active | driving | (20.0, 30.5) |

#### 3. Geocoding Database
- Stores places → lat/lng pairs
- **Redis** (key-value) — frequent reads, infrequent writes
- Used to convert origin/destination to lat/lng before passing to route planner

#### 4. Precomputed Map Images
- Heavy computation to render map area with roads → precompute and cache
- Stored on **CDN backed by cloud storage (S3)**
- Images at different zoom levels

---

## Design Deep Dive — Services

### Location Service (Deep Dive)

**Database design:**
- 1 million location updates/second → NoSQL key-value or column-oriented DB
- Location constantly changing → **prioritize availability over consistency** (CAP theorem)
- Choose **Availability + Partition tolerance** → **Cassandra**

| key (user_id) | timestamp | lat | long | user_mode | navigation_mode |
|---------------|-----------|-----|------|-----------|-----------------|
| 51 | 132053000 | 21.9 | 89.8 | active | driving |

**How location data is used (via Kafka streaming):**

```
Location Service → Kafka → Multiple Consumers:
├── Traffic Update Service → Traffic DB
├── ML Service for Personalization → Personalization DB
├── Routing Tile Processing Service → Routing Tiles (Object Storage)
└── Analytics → Analytics DB
```

- **Traffic service:** digests stream, updates live traffic database
- **Routing tile processing:** detects new/closed roads, updates affected routing tiles
- **Other services** tap into stream for various purposes

### Map Rendering (Deep Dive)

**Precomputed tiles:**
- Google Maps uses **21 zoom levels**
- Level 0: entire map = single 256×256 pixel tile
- Each zoom increment: tiles **double** in N-S and E-W directions (4× total)
  - Level 1: 2×2 tiles (512×512 combined)
  - Level 2: 4×4 tiles (1024×1024 combined)
- Higher zoom = more detail, higher pixel count

**Optimization: Vector tiles (WebGL)**
- Instead of sending PNG images → send **vector information** (paths, polygons)
- Client renders from vector data
- **Advantages:**
  - Vector data compresses **much better** than images → substantial bandwidth savings
  - **Better zooming experience** — rasterized images get pixelated between zoom levels; vectors can interpolate smoothly

### Navigation Service (Deep Dive)

**Sub-services:**

```
Navigation Service
├── Geocoding Service → Geocoding DB
├── Route Planner Service
│   ├── Shortest Path Service → Routing Tiles (Object Storage)
│   ├── ETA Service → Traffic DB
│   └── Ranker / Filter Service
└── Adaptive ETA & Rerouting → Active Users DB
```

#### Shortest-Path Service
- Receives origin/destination as lat/lng pairs
- Returns **top-k shortest paths** WITHOUT considering traffic
- Depends only on road structure → **caching beneficial** (graph rarely changes)
- Runs **A* pathfinding variant** against routing tiles

**Algorithm overview:**
1. Convert lat/lng pairs → geohashes → load start/end routing tiles
2. Start from origin tile, traverse graph
3. **Hydrate additional neighboring tiles** from object storage (or local cache) as search expands
4. Can "enter" bigger tiles (highways) via cross-level connections
5. Continue expanding until best routes found

#### ETA Service
- Route planner sends possible shortest paths → ETA service returns time estimates
- Uses **machine learning** to predict ETAs based on current traffic + historical data
- Challenge: predict traffic 10–20 minutes ahead (algorithmic, not covered)

#### Ranker Service
- Receives ETA predictions from route planner
- Applies **user-defined filters** (avoid tolls, avoid freeways, etc.)
- Ranks routes fastest → slowest
- Returns **top-k results** to navigation service

#### Updater Services
- Tap into Kafka location stream, **asynchronously update databases**
- **Routing tile processing service:** transforms road dataset with new roads/closures → updated routing tiles → helps shortest path be more accurate
- **Traffic update service:** extracts traffic conditions from location streams → feeds live traffic DB → enables better ETA

---

## Adaptive ETA & Rerouting

**Problem:** Current design doesn't support real-time ETA updates or rerouting when traffic changes.

**Key questions:**
- How to track actively navigating users?
- How to efficiently find users affected by traffic changes among millions of routes?

### Naive Approach
- Store each user's route as list of routing tiles: `user_1: r_1, r_2, r_3, ..., r_k`
- Traffic incident in tile `r_2` → scan all rows, check if `r_2` in route
- Time complexity: **O(n × m)** where n = users, m = average route length
- Too slow at scale

### Optimized Approach (Hierarchical Routing Tiles)
- For each navigating user, store: current routing tile → next resolution level tile containing it → recursively until tile contains both origin AND destination

```
user_1, r_1, super(r_1), super(super(r_1)), ...
```

- To check if user affected by traffic change: only check if changed tile is **inside the last (largest) routing tile** in the row
  - If not → user not impacted (quick filter)
  - If yes → user potentially affected
- **Quickly filters out many users** without scanning entire route

**When traffic clears:**
- Keep track of all possible routes for navigating user
- Recalculate ETAs regularly
- Notify user if new route with shorter ETA found

---

## Delivery Protocols

**Server → Client push options:**

| Protocol | Verdict | Reason |
|----------|---------|--------|
| Mobile push notification | ❌ Rejected | Payload limit (4,096 bytes iOS), no web support |
| Long polling | ❌ Rejected | Higher server overhead |
| SSE (Server-Sent Events) | Possible | One-directional |
| **WebSocket** | ✅ **Chosen** | Bi-directional, light footprint, supports last-mile delivery features |

---

## 📝 Summary — Google Maps

- **Three core features:** location update, navigation (with ETA), map rendering
- **Map tiles** are pre-generated PNG images at 21 zoom levels, served via CDN using geohash-based URLs
- **Routing tiles** are binary graph data structures (nodes = intersections, edges = roads) stored in object storage with 3 hierarchical levels
- **Location data** batched every 15 seconds → Cassandra for storage, Kafka for streaming to downstream services (traffic, routing, analytics)
- **Navigation** uses shortest-path (A*) on routing tiles → ETA via ML → ranking/filtering → top-k routes returned
- **Adaptive ETA/rerouting** tracks active users with hierarchical tile references for efficient affected-user lookup
- **Storage:** ~100PB for map tiles, TBs for routing tiles
- **Scale:** 1B DAU, 200K avg QPS for location updates (1M peak)
- **WebSocket** chosen for server-to-client delivery during navigation

---
---

# Chapter 4: Distributed Message Queue

## Requirements & Scope

### Context: Message Queues vs Event Streaming

| Aspect | Traditional Message Queue | Event Streaming Platform |
|--------|--------------------------|------------------------|
| Examples | RabbitMQ, ActiveMQ, ZeroMQ | Apache Kafka, Pulsar |
| Retention | Short (in memory until consumed) | Long (days/weeks on disk) |
| Re-consumption | No (message removed after delivery) | Yes (multiple consumers) |
| Message ordering | Not typically guaranteed | Guaranteed within partition |
| Convergence | RabbitMQ added optional streams | Pulsar can work as traditional queue |

> This design covers a distributed message queue **with additional event streaming features** (long retention, repeated consumption, ordering).

### Functional Requirements
- Producers send messages to a queue
- Consumers consume messages from a queue
- Messages can be consumed **repeatedly or only once**
- Historical data can be **truncated**
- Message size: **kilobyte range**
- Deliver messages **in order** they were added
- **Configurable delivery semantics:** at-least once, at-most once, exactly once

### Non-Functional Requirements
- **High throughput OR low latency** — configurable per use case
- **Scalable** — distributed, handles sudden surges
- **Persistent and durable** — data on disk, replicated across multiple nodes

### Traditional Queue Simplifications
- Weaker retention (in-memory, on-disk overflow much smaller)
- No ordering guarantee
- Messages deleted after consumption

---

## Messaging Models

### Point-to-Point
- Message sent to queue → consumed by **one and only one consumer**
- Multiple consumers may wait, but each message goes to single consumer
- Traditional model: **no data retention** after acknowledgment
- This design **simulates** point-to-point via consumer groups

### Publish-Subscribe
- Message sent to a **topic** → received by **all subscribing consumers**
- Topics = categories with unique names across entire service
- Naturally maps to this design's capabilities

---

## Topics, Partitions, and Brokers

### Partitions (Sharding)
- **Problem:** Single topic data volume too large for one server
- **Solution:** Divide topic into **partitions** — small subsets of messages
- Partitions distributed across servers in the cluster

```
Topic-A → Partition-1 (on Broker-1)
        → Partition-2 (on Broker-2)
```

- Each partition operates as **FIFO queue**
- Message position in partition = **offset**
- **Ordering guaranteed within a partition** (not across partitions)

### Message Key Routing
- Each message has optional **message key** (e.g., user ID)
- Same key → same partition (via `hash(key) % numPartitions`)
- No key → random partition assignment

### Brokers
- Servers that hold partitions
- Distribution of partitions among brokers → key to **high scalability**
- Scale topic capacity by expanding number of partitions

---

## Consumer Groups

- **Consumer group** = set of consumers working together to consume from topics
- Each group can subscribe to **multiple topics**
- Each group maintains its **own consuming offsets**
- Groups **isolated** from each other

**Critical constraint:** A single partition can only be consumed by **one consumer in the same group**
- Fixes ordering problem (parallel consumption would break order within partition)
- If consumers > partitions in a group → some consumers idle
- Equivalent to **point-to-point model** when all consumers in same group

**Pub-sub behavior:**
- Topic subscribed by **multiple consumer groups** → same message consumed by multiple groups

---

## High-Level Architecture

### 🌳 Architecture Tree

```
Distributed Message Queue
├── Clients
│   ├── Producers → push messages to topics
│   └── Consumer Groups → subscribe & consume
├── Core Service
│   └── Brokers (hold partitions)
│       ├── Data Storage (messages in partitions)
│       ├── State Storage (consumer states)
│       └── Metadata Storage (topic configs)
└── Coordination Service (ZooKeeper/etcd)
    ├── Service discovery (which brokers alive)
    └── Leader election (active controller for partition assignment)
```

---

## Data Storage — WAL

### Traffic Pattern Analysis
- **Write-heavy, read-heavy**
- **No update or delete** (append-only)
- **Predominantly sequential** read/write access

### 🆚 Storage Options

| Option | Pros | Cons |
|--------|------|------|
| **Database (Relational/NoSQL)** | Handles storage | Hard to optimize for both write-heavy AND read-heavy at scale; not ideal for sequential access patterns |
| **Write-Ahead Log (WAL)** ✅ | Pure sequential access; great disk performance; large capacity; affordable | — |

### WAL Design
- Plain file, **append-only** log
- New message appended to tail with **monotonically increasing offset**
- Easiest offset = line number of log file

**Segmentation:**
- File cannot grow infinitely → divide into **segments**
- New messages → **active segment only**
- When active segment reaches size limit → new active segment created, old becomes inactive
- Inactive segments: **read-only** (serve read requests)
- Old segments **truncated** when exceeding retention/capacity limit

```
Topic-A Partition-1:
[0|1|2|3|4|5|6|7|8|9] [10|11|12|13|14|15]
  ← segment-1 →          ← segment-2 →
```

**File organization:**
```
/Topic-A/
  /Partition-1/
    segment-1
    segment-2
    segment-3
  /Partition-2/
    segment-1
    segment-2
```

### Disk Performance Note
- Common misconception: rotational disks are slow → **only for random access**
- Sequential access on RAID configuration → **several hundred MB/sec** read/write
- Modern OS caches disk data in main memory **aggressively** (uses all available free memory)
- WAL benefits from OS disk caching

---

## Message Data Structure

> Critical for high throughput. Defines contract between producers, queue, and consumers. No-copy design: message passes through system without modifications.

| Field | Data Type | Purpose |
|-------|-----------|---------|
| `key` | byte[] | Determines partition (hash-based routing) |
| `value` | byte[] | Message payload (plain text or compressed binary) |
| `topic` | string | Topic the message belongs to |
| `partition` | integer | Partition ID |
| `offset` | long | Position in partition (unique with topic + partition) |
| `timestamp` | long | When message was stored |
| `size` | integer | Message size |
| `crc` | integer | Cyclic redundancy check for data integrity |

> **Key vs KV store key:** In message queue, keys don't need to be unique and aren't used for lookup. In KV store, keys are unique identifiers.

**Optional fields:** Tags (for message filtering), etc.

### Batching

> Pervasive throughout the design — producer, queue, and consumer all batch.

**Why batching is critical:**
1. Allows OS to **group messages** in larger chunks → amortizes **network round trips**
2. Broker writes messages to append logs in **larger chunks** → sequential writes → larger contiguous disk cache blocks → **much greater sequential throughput**

**Tradeoff: Batch size vs Latency**
- Large batch → **higher throughput, higher latency** (longer wait to accumulate)
- Small batch → **lower latency, lower throughput**
- Configurable per use case
- For traditional MQ (latency-focused): smaller batches, more partitions per topic to compensate

---

## Producer Flow

### Initial Design: Routing Layer
1. Producer → routing layer → "correct" broker (leader replica)
2. Routing layer reads **replica distribution plan** from metadata storage, caches locally

**Drawbacks:**
- Additional network latency (extra hop)
- No batching consideration

### Improved Design: Embedded Routing + Buffer

```
Producer
├── Buffer (batches messages in memory)
└── Routing (determines target partition/broker)
    → Broker (leader replica)
```

**Benefits:**
- Fewer network hops → **lower latency**
- Producers define **own partition logic**
- Buffer batches messages → sends larger batches in single request → **higher throughput**

### Producer-to-Broker Flow
1. Producer sends batch to leader replica of target partition
2. Leader receives message
3. Follower replicas **pull** data from leader
4. When "enough" replicas synchronized → leader **commits** (persists to disk)
5. Leader responds to producer

---

## Consumer Flow

- Consumer specifies **offset in partition** → receives chunk of events from that position
- Each consumer group tracks its own **last consumed offset** per partition

### 🆚 Push vs Pull Model

| Aspect | Push | Pull (Chosen ✅) |
|--------|------|------|
| Latency | Low (immediate push) | Slightly higher |
| Rate control | Broker controls → can overwhelm consumers | Consumer controls → handles diverse processing power |
| Diverse consumers | Difficult (broker doesn't know capacity) | Easy (each consumes at own rate) |
| Batch processing | Poor (broker sends one-at-a-time) | Excellent (pull all available up to max) |
| Empty queue | N/A | Wastes resources → solved by **long polling** |
| Scaling | Hard if consumption < production | Scale out consumers or catch up later |

**Most message queues choose pull model.**

### Consumer Pull Workflow
1. Consumer connects to leader replica of assigned partition
2. Sends pull request with current offset
3. Receives batch of messages from that offset
4. Processes messages
5. Commits new offset to state storage

**Why read from leader only (not ISRs)?**
- Design and operational simplicity
- One partition → one consumer per group → limited connections to leader
- Connections not large as long as topic not super hot
- Hot topic → scale by expanding partitions and consumers
- Exception: consumer in different data center → read from closest ISR may be worthwhile

---

## Consumer Rebalancing

**Triggered when:** consumer joins, leaves, or crashes in a group.

**Key role: Coordinator** (one of the brokers designated for a consumer group)

### New Consumer Joins
1. Initial consumer A consumes all partitions, keeps heartbeat with coordinator
2. Consumer B sends JoinGroup request
3. Coordinator tells A (via heartbeat response) to rejoin → triggers rebalance
4. All consumers rejoin → coordinator elects one as **leader**
5. Leader generates **partition dispatch plan** → sends to coordinator
6. Follower consumers get plan from coordinator
7. All consumers start consuming from newly assigned partitions

### Consumer Leaves (Graceful)
1. Consumer A sends LeaveGroup request
2. Coordinator acknowledges
3. Remaining Consumer B notified via heartbeat to rejoin
4. Rebalance proceeds as above

### Consumer Crashes
1. Consumer A stops sending heartbeats
2. Coordinator detects no heartbeat within **specified timeout** → marks consumer dead
3. Triggers rebalance for remaining consumers

---

## State & Metadata Storage

### State Storage
Stores:
- Mapping between partitions and consumers
- **Last consumed offsets** per consumer group per partition

**Access patterns:**
- Frequent read/write, low volume
- Updated frequently, rarely deleted
- Random read/write
- **Data consistency important**

**Choice:** KV store like **ZooKeeper** (Kafka later moved offsets to Kafka brokers)

### Metadata Storage
Stores:
- Topic configuration and properties
- Number of partitions, retention period
- Distribution of replicas

**Characteristics:** Infrequent changes, small volume, **high consistency requirement**

**Choice:** **ZooKeeper**

### ZooKeeper Role in Final Design
- Stores metadata and state
- Provides **service discovery** (which brokers alive)
- Handles **leader election** (active controller in broker cluster)
- Broker only maintains **data storage** for messages

---

## Replication & In-Sync Replicas

### Replication Basics
- Each partition has **multiple replicas** across different broker nodes
- One replica = **leader** (receives writes), others = **followers** (pull from leader)
- Producers write **only to leader**
- Followers continuously pull new messages from leader
- **Replica distribution plan** = which broker holds which replica for each partition
  - Generated by elected broker controller
  - Persisted in metadata storage

### In-Sync Replicas (ISR)

**Definition:** Replicas that are "in-sync" with the leader.

**"In-sync" criteria:** configurable — e.g., `replica.lag.max.messages = 4` means follower within 3 messages of leader stays in ISR.

**How ISR works:**
- Leader has committed offset (e.g., 13) — all messages at/before this offset synchronized to ALL ISR replicas
- New messages written to leader but **not committed** until ISR catches up
- Followers that fall behind configured lag → **removed from ISR** (can rejoin when caught up)

**Why ISR?** Tradeoff between **performance and durability**:
- Waiting for ALL replicas = safest but slow replica blocks entire partition
- ISR allows configurable guarantee level

### ACK Settings (Configurable)

| ACK Setting | Behavior | Durability | Latency | Use Case |
|-------------|----------|------------|---------|----------|
| **ACK=all** | Wait for ALL ISRs to receive | Strongest | Highest | Data cannot be lost |
| **ACK=1** | Wait for leader to persist | Medium (lost if leader fails before replication) | Medium | Low latency, occasional loss OK |
| **ACK=0** | No waiting, no retry | Lowest (potential message loss) | Lowest | Metrics/logging, high volume |

---

## Scalability

### Producer Scalability
- Simpler than consumer (no group coordination)
- Add/remove producer instances freely

### Consumer Scalability
- Consumer groups isolated → easy to add/remove groups
- Within group: **rebalancing mechanism** handles add/remove/crash
- Consumer groups + rebalancing → scalability + fault tolerance

### Broker Failure Recovery

**Example scenario:**
1. 4 brokers, partition distribution plan defined
2. Broker-3 crashes → all its partitions lost
3. Broker controller detects crash → generates **new distribution plan** for remaining brokers
4. New replicas work as followers, **catch up with leader**

**Fault tolerance considerations:**
- **Minimum ISR count** — higher = safer but more latency
- Replicas must be on **different nodes** (same node = can't tolerate that node's failure)
- All replicas crash → **data lost forever**
- Cross-data-center replication: safer but higher latency/cost → **data mirroring** as workaround

### Broker Scaling (Adding Nodes)
- **Naive:** Redistribute replicas → potential data loss
- **Better:** Temporarily allow MORE replicas than configured
  1. New broker added
  2. New replica starts copying from leader
  3. After catch up → remove redundant old replica
  4. **No data loss** during process
- Same process reversed for removing brokers

### Partition Scaling

**Increasing partitions:**
- Persisted messages stay in old partitions (no migration)
- New messages distributed across all partitions (including new)
- Straightforward

**Decreasing partitions:**
- Decommissioned partition stops receiving new messages
- **Cannot remove immediately** — consumers may still need data
- Must wait for **retention period to expire** before truncating
- During transition: producers → remaining partitions only; consumers → all partitions (including decommissioned)
- After retention expires → consumer groups rebalance

---

## Data Delivery Semantics

### At-Most Once
- Message delivered **≤ 1 time** — may be lost, never redelivered
- Producer: ACK=0, no retry
- Consumer: commits offset **before** processing → crash after commit = message lost
- **Use case:** Monitoring metrics (small data loss acceptable)

### At-Least Once
- Message delivered **≥ 1 time** — no loss, possible duplicates
- Producer: ACK=1 or ACK=all, retries on failure/timeout
- Consumer: commits offset **after** successful processing → crash before commit = re-consume (duplicate)
- **Use case:** Most applications where deduplication possible (e.g., unique key in DB rejects duplicates)

### Exactly Once
- Message delivered **exactly 1 time** — no loss, no duplicates
- Most difficult to implement, highest cost to performance/complexity
- **Use case:** Financial systems (payment, trading, accounting) where duplication unacceptable and downstream doesn't support idempotency

---

## Advanced Features

### Message Filtering
- **Problem:** Consumer groups may only want certain subtypes from a topic
- **Naive:** Consumer fetches all, filters locally → wastes bandwidth
- **Better:** Filter on **broker side** using message **tags** (metadata, not payload)
  - Broker doesn't decrypt/deserialize payload (performance preserved, security maintained)
  - Tags support multi-dimensional filtering
  - Consumer subscribes with specified tags
- Complex logic (math formulae) → grammar parser/script executor → too heavyweight for MQ

### Delayed Messages & Scheduled Messages
- **Delayed:** Deliver to consumer after specified time period
  - Example: order cancellation check 30 minutes after creation
- **Implementation:** Send to **temporary storage** on broker → deliver to topic when time expires
- **Timing solutions:**
  - Dedicated delay queues with predefined levels (RocketMQ: 1s, 5s, 10s, ... 1h, 2h)
  - Hierarchical time wheel
- **Scheduled messages:** Similar design, deliver at specific scheduled time

---

## 📝 Summary — Distributed Message Queue

- **Core model:** Topics → Partitions → Brokers; messages append to partitions with monotonically increasing offsets
- **Storage:** Write-Ahead Log (WAL) on disk — append-only, sequential access, segmented files; leverages OS disk caching
- **Message structure** designed for zero-copy transit (producer → queue → consumer without modification)
- **Batching** is pervasive (producer, broker, consumer) — critical for high throughput; tradeoff with latency
- **Consumer groups** enable both pub-sub (multiple groups) and point-to-point (single group) models
- **Pull model** chosen over push for rate control, diverse consumer support, and batch processing
- **Replication** via leader-follower with ISR (In-Sync Replicas) — configurable ACK levels trade durability for performance
- **ZooKeeper** handles metadata, state, service discovery, and leader election
- **Consumer rebalancing** via coordinator — handles join, leave, crash scenarios through heartbeat mechanism
- **Three delivery semantics:** at-most-once (ACK=0), at-least-once (ACK=1/all + retry), exactly-once (most complex)
- **Scalability:** producers (simple add/remove), consumers (rebalancing), brokers (temporary extra replicas), partitions (add freely, decommission carefully)
- **Advanced features:** tag-based filtering on broker side, delayed/scheduled messages via temporary storage + timing function

---

## ⚡ Quick-Recall Points

- Google Maps map tiles: ~100PB total across all zoom levels; each tile 256×256 pixels, ~100KB
- Location update QPS: 200K average, 1M peak (batched every 15 seconds)
- Routing tiles ≠ map tiles: routing = binary graph data, map = PNG images
- Hierarchical routing: 3 levels (local → arterial → highway)
- WAL sequential disk writes can achieve **several hundred MB/sec** on RAID
- ISR = In-Sync Replica; determined by `replica.lag.max.messages` config
- ACK=all → strongest durability; ACK=0 → lowest latency
- Single partition → single consumer per group (ordering guarantee)
- Consumer offset committed AFTER processing → at-least-once; BEFORE → at-most-once
- Decreasing partitions requires waiting for retention period to expire before cleanup
- WebSocket chosen for Google Maps server→client push (bi-directional, light footprint)

---

## 🆚 Key Comparisons

### Map Tiles vs Routing Tiles

| Aspect | Map Tiles | Routing Tiles |
|--------|-----------|---------------|
| Format | PNG images | Binary graph data (adjacency lists) |
| Purpose | Visual map rendering | Pathfinding algorithms |
| Served via | CDN | Object storage (S3) with caching |
| Zoom levels | 21 levels of detail | 3 hierarchical levels |
| Size | ~100PB total | TBs |

### Push vs Pull (Message Queue)

| Aspect | Push | Pull |
|--------|------|------|
| Latency | Lower | Slightly higher |
| Consumer overwhelm | Possible | Consumer controls rate |
| Batch processing | Poor | Excellent |
| Empty queue handling | N/A | Long polling needed |
| Chosen | No | Yes ✅ |

### ACK Settings Comparison

| ACK | Wait for | Durability | Latency | Risk |
|-----|----------|------------|---------|------|
| all | All ISRs | Highest | Highest | Slowest ISR blocks |
| 1 | Leader only | Medium | Medium | Leader crash = loss |
| 0 | Nothing | Lowest | Lowest | Any failure = loss |
