

# 📌 System Design Interview — Chapters 6–10

**Five system design problems: Key-Value Store, Unique ID Generator, URL Shortener, Web Crawler, Notification System**
| Type: System Design | Depth: Intermediate–Advanced |

---

# 📑 Table of Contents

1. [Chapter 6: Design a Key-Value Store](#ch6)
2. [Chapter 7: Design a Unique ID Generator](#ch7)
3. [Chapter 8: Design a URL Shortener](#ch8)
4. [Chapter 9: Design a Web Crawler](#ch9)
5. [Chapter 10: Design a Notification System](#ch10)

---

<a id="ch6"></a>
# 🔹 CHAPTER 6: Design a Key-Value Store

## 🔹 What Is a Key-Value Store

- **Key-value store** — a non-relational database storing unique key → associated value pairs
- **Key** must be unique; value accessed via the key
  - Keys can be plain text (`"last_logged_in_at"`) or hashed (`253DDEC4`)
  - Short keys preferred for performance
- **Value** can be strings, lists, objects — treated as an **opaque object**
- Real-world examples: Amazon DynamoDB, Memcached, Redis
- Two core operations:
  - `put(key, value)` — insert/update
  - `get(key)` — retrieve

---

## 🔹 Design Scope & Requirements

- No perfect design — every design balances **read, write, memory usage** trade-offs plus **consistency vs. availability**
- Target characteristics:
  - Key-value pair size: **< 10 KB**
  - Ability to store **big data**
  - **High availability** (responds quickly even during failures)
  - **High scalability** (supports large data sets)
  - **Automatic scaling** (servers added/removed based on traffic)
  - **Tunable consistency**
  - **Low latency**

---

## 🔹 Single Server Key-Value Store

- Simplest approach: **in-memory hash table**
- Memory is fast but limited → two optimizations:
  1. **Data compression**
  2. **Store only frequently used data in memory**, rest on disk
- Even with optimizations, single server hits capacity quickly → need **distributed key-value store**

---

## 🔹 Distributed Key-Value Store (Distributed Hash Table)

### 📖 CAP Theorem

| Term | Definition |
|------|-----------|
| **Consistency** | All clients see the same data at the same time regardless of which node they connect to |
| **Availability** | Any client requesting data gets a response even if some nodes are down |
| **Partition Tolerance** | System continues to operate despite network partitions (communication breaks between nodes) |

- **CAP theorem**: impossible for a distributed system to simultaneously guarantee all three — must sacrifice one
- Classification of key-value stores:

| Type | Supports | Sacrifices |
|------|----------|-----------|
| **CP** | Consistency + Partition Tolerance | Availability |
| **AP** | Availability + Partition Tolerance | Consistency |
| **CA** | Consistency + Availability | Partition Tolerance |

- ⚠️ **CA cannot exist in real-world** distributed systems because network failures are unavoidable → must tolerate partitions

### Concrete Example (3 replica nodes: n1, n2, n3)

- **Ideal**: no partition → data written to n1 auto-replicates to n2, n3 → both C and A achieved
- **Partition occurs** (n3 goes down):
  - **CP choice**: block all writes to n1/n2 to prevent inconsistency → system becomes unavailable
    - Example: bank systems need strong consistency; return error until inconsistency resolved
  - **AP choice**: keep accepting reads/writes on n1/n2; sync to n3 when partition resolves → may serve stale data

---

## 🔹 System Components (Based on Dynamo, Cassandra, BigTable)

### 🌳 Concept Tree
```
Distributed Key-Value Store
├── Data Partition (Consistent Hashing)
├── Data Replication
├── Consistency (Quorum Consensus)
│   └── Consistency Models
├── Inconsistency Resolution (Vector Clocks)
├── Handling Failures
│   ├── Failure Detection (Gossip Protocol)
│   ├── Temporary Failures (Sloppy Quorum + Hinted Handoff)
│   ├── Permanent Failures (Anti-Entropy + Merkle Trees)
│   └── Data Center Outage (Cross-DC Replication)
├── System Architecture
├── Write Path (Commit Log → Memory Cache → SSTable)
└── Read Path (Memory Cache → Bloom Filter → SSTables)
```

---

### 🔹 Data Partition

- Challenge: distribute data evenly + minimize data movement on node add/remove
- Solution: **Consistent Hashing**
  - Servers placed on a hash ring (e.g., s0–s7)
  - Key hashed onto ring → stored on **first server encountered clockwise**
  - Example: key0 hashed between s0 and s1 → stored on s1
- Advantages:
  - **Automatic scaling**: servers added/removed with minimal redistribution
  - **Heterogeneity**: number of virtual nodes proportional to server capacity (higher-capacity servers get more virtual nodes)

---

### 🔹 Data Replication

- Data replicated asynchronously over **N servers** (N = configurable parameter)
- Replication logic: from key's position on hash ring, walk clockwise → choose **first N unique servers**
  - Example: N=3, key0 replicated at s1, s2, s3
- With virtual nodes: first N nodes on ring may map to fewer than N physical servers → **only choose unique physical servers**
- For reliability: place replicas in **distinct data centers** connected via high-speed networks
  - Protects against power outages, network issues, natural disasters

---

### 🔹 Consistency — Quorum Consensus

📖 Key definitions:
| Symbol | Meaning |
|--------|---------|
| **N** | Number of replicas |
| **W** | Write quorum — write must be ACKed by W replicas to be successful |
| **R** | Read quorum — read must wait for responses from at least R replicas |

- **Coordinator** acts as proxy between client and nodes
- W=1 does NOT mean data written on only one server — data still goes to all N replicas, but coordinator only waits for 1 ACK

📐 Key formulas / configurations:

| Configuration | Effect |
|--------------|--------|
| `R=1, W=N` | Optimized for **fast reads** |
| `W=1, R=N` | Optimized for **fast writes** |
| `W+R > N` | **Strong consistency** guaranteed (overlapping node has latest data). Typical: N=3, W=R=2 |
| `W+R ≤ N` | Strong consistency **NOT** guaranteed |

- Trade-off: lower W or R → faster but less consistent; higher W or R → slower but more consistent (wait for slowest replica)

---

### 🔹 Consistency Models

| Model | Description |
|-------|-------------|
| **Strong consistency** | Read always returns most recent write; client never sees stale data |
| **Weak consistency** | Subsequent reads may not see latest write |
| **Eventual consistency** | Specific form of weak consistency; given enough time, all replicas converge |

- Strong consistency achieved by blocking new reads/writes until all replicas agree — **not ideal for high-availability systems**
- **Dynamo and Cassandra use eventual consistency** (recommended for this design)
- Eventual consistency allows inconsistent values from concurrent writes → **client-side reconciliation** needed

---

### 🔹 Inconsistency Resolution: Versioning & Vector Clocks

**Problem**: concurrent writes to different replicas create conflicting versions
- Example: n1 has name="johnSanFrancisco" (v1), n2 has name="johnNewYork" (v2) — no clear resolution

**Vector Clock** — a `[server, version]` pair associated with each data item
- Notation: `D([S1, v1], [S2, v2], …, [Sn, vn])`
- Rules when data item D is written to server Si:
  - If `[Si, vi]` exists → increment vi
  - Otherwise → create new entry `[Si, 1]`

**Worked example** (5 steps):
1. Client writes D1, handled by Sx → `D1([Sx, 1])`
2. Client reads D1, updates to D2, writes back to Sx → `D2([Sx, 2])`
3. Client reads D2, updates to D3, writes to Sy → `D3([Sx, 2], [Sy, 1])`
4. Another client reads D2, updates to D4, writes to Sz → `D4([Sx, 2], [Sz, 1])`
5. Client reads D3 and D4 → **detects conflict** (D2 modified by both Sy and Sz) → reconciles → `D5([Sx, 3], [Sy, 1], [Sz, 1])`

**Conflict detection rules**:
- **Ancestor (no conflict)**: version X is ancestor of Y if every counter in X ≤ corresponding counter in Y
  - `D([s0,1],[s1,1])` is ancestor of `D([s0,1],[s1,2])` ✅
- **Sibling (conflict)**: any counter in Y is less than corresponding counter in X
  - `D([s0,1],[s1,2])` vs `D([s0,2],[s1,1])` → conflict ⚠️

**Downsides of vector clocks**:
1. Adds complexity to client (must implement conflict resolution logic)
2. `[server:version]` pairs can grow rapidly → fix by setting length threshold, removing oldest pairs
   - Can cause reconciliation inefficiency, but Amazon hasn't encountered this in production

---

### 🔹 Handling Failures

#### Failure Detection — Gossip Protocol

- Naive approach: all-to-all multicasting → inefficient with many servers
- Better: **Gossip protocol** (decentralized failure detection)
  - Each node maintains a **membership list** (member IDs + heartbeat counters)
  - Each node periodically **increments** its heartbeat counter
  - Each node periodically **sends heartbeats to random nodes** → propagated further
  - On receiving heartbeats: membership list updated
  - If heartbeat counter hasn't increased for predefined period → member marked **offline**
- Requires at least **2 independent sources** to confirm a server is down

#### Handling Temporary Failures — Sloppy Quorum + Hinted Handoff

- Strict quorum could block reads/writes during failures
- **Sloppy quorum**: instead of enforcing strict quorum, choose first W healthy servers for writes, first R healthy for reads on hash ring; **ignore offline servers**
- **Hinted handoff**: if server unavailable (e.g., s2 down), another server (s3) temporarily handles its requests → when s2 recovers, s3 pushes data back to s2

#### Handling Permanent Failures — Anti-Entropy + Merkle Trees

- **Anti-entropy protocol**: compare data on replicas, update each to newest version
- **Merkle tree** (hash tree): every non-leaf node labeled with hash of its children → efficient inconsistency detection
- Building a Merkle tree (key space 1-12, 4 buckets):
  1. Divide key space into buckets
  2. Hash each key in a bucket using uniform hashing
  3. Create a single hash node per bucket
  4. Build tree upward by calculating hashes of children
- **Comparison**: compare root hashes → if match, data identical; if differ, traverse down to find differing buckets → sync only those buckets
- Data synced is proportional to **differences**, not total data
- Real-world: ~1 million buckets per 1 billion keys → each bucket ≈ 1000 keys

#### Handling Data Center Outage

- Replicate data across **multiple data centers**
- If entire DC goes offline, users access data through other DCs

---

### 🔹 System Architecture

Key features:
- Client APIs: `get(key)` and `put(key, value)`
- **Coordinator**: node acting as proxy between client and key-value store
- Nodes distributed on ring via **consistent hashing**
- **Completely decentralized** → automatic node add/remove
- Data replicated at multiple nodes
- **No single point of failure** — every node has same responsibilities

Each node performs: Client API, Failure detection, Conflict resolution, Failure repair mechanism, Replication, Storage engine

---

### 🔹 Write Path (based on Cassandra architecture)

1. Write request persisted to **commit log** file (on disk)
2. Data saved in **memory cache**
3. When memory cache full or reaches threshold → flushed to **SSTable** on disk
   - **SSTable** (Sorted-String Table): sorted list of `<key, value>` pairs

### 🔹 Read Path

**If data in memory cache**: return directly to client

**If data NOT in memory cache**:
1. Check memory cache → miss
2. Check **Bloom filter** (probabilistic data structure to determine which SSTables might contain the key)
3. Bloom filter identifies candidate SSTables
4. SSTables return data
5. Result returned to client

---

### 📝 Chapter 6 Summary Table

| Goal/Problem | Technique |
|-------------|-----------|
| Store big data | Consistent hashing to spread load |
| High availability reads | Data replication + multi-data center |
| Highly available writes | Versioning + vector clocks |
| Dataset partition | Consistent hashing |
| Incremental scalability | Consistent hashing |
| Heterogeneity | Consistent hashing (virtual nodes) |
| Tunable consistency | Quorum consensus |
| Temporary failures | Sloppy quorum + hinted handoff |
| Permanent failures | Merkle tree |
| Data center outage | Cross-data center replication |

---

<a id="ch7"></a>
# 🔹 CHAPTER 7: Design a Unique ID Generator in Distributed Systems

## 🔹 Problem Statement

- `auto_increment` doesn't work in distributed environments — single DB not large enough; generating unique IDs across multiple DBs with minimal delay is challenging
- Requirements:
  - IDs must be **unique**
  - **Numerical values** only
  - Fit into **64-bit**
  - **Ordered by date** (IDs created later are larger)
  - Generate **over 10,000 unique IDs per second**

---

## 🔹 High-Level Design: Four Approaches

### 🆚 Comparison of Approaches

| Approach | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Multi-master replication** | DB `auto_increment` by k (k = number of servers) | Scales with DB servers | Hard to scale across DCs; IDs not time-ordered across servers; doesn't scale well on server add/remove |
| **UUID** | 128-bit number generated independently per server | Simple; no coordination; easy to scale | 128 bits (not 64); not time-ordered; could be non-numeric |
| **Ticket Server** | Centralized `auto_increment` in single DB (Flickr's approach) | Numeric IDs; easy to implement for small/medium scale | Single point of failure; multiple ticket servers → data sync challenges |
| **Twitter Snowflake** | 64-bit ID divided into sections | Meets all requirements; scalable | Depends on clock synchronization |

---

### 🔹 Twitter Snowflake — Chosen Approach

**64-bit ID layout:**

```
| 1 bit | 41 bits    | 5 bits       | 5 bits     | 12 bits         |
| sign  | timestamp  | datacenter ID| machine ID | sequence number |
```

| Section | Bits | Details |
|---------|------|---------|
| **Sign bit** | 1 | Always 0; reserved for future (signed/unsigned distinction) |
| **Timestamp** | 41 | Milliseconds since custom epoch (Twitter default: 1288834974657 = Nov 04, 2010, 01:42:54 UTC) |
| **Datacenter ID** | 5 | 2^5 = **32 datacenters** |
| **Machine ID** | 5 | 2^5 = **32 machines** per datacenter |
| **Sequence number** | 12 | Incremented per ID on same machine/ms; reset to 0 every ms |

---

## 🔹 Deep Dive

### Timestamp

- 41 bits → max value = 2^41 - 1 = 2,199,023,255,551 ms ≈ **~69 years**
- Using custom epoch close to today delays overflow
- After 69 years: need new epoch or migration technique
- IDs are **sortable by time** because timestamp is the most significant portion

### Sequence Number

- 12 bits → 2^12 = **4,096 combinations**
- Field is 0 unless multiple IDs generated in same millisecond on same server
- Max throughput: **4,096 IDs per millisecond per machine**

### Operational Considerations

- **Datacenter IDs and machine IDs**: fixed at startup; changes require careful review (accidental change → ID conflicts)
- **Timestamp and sequence numbers**: generated at runtime

---

## 🔹 Additional Talking Points

- **Clock synchronization**: multi-core/multi-machine servers may have clock drift → **Network Time Protocol (NTP)** is standard solution
- **Section length tuning**: fewer sequence bits + more timestamp bits → good for low concurrency + long-term use
- **High availability**: ID generator is mission-critical → must be highly available

---

<a id="ch8"></a>
# 🔹 CHAPTER 8: Design a URL Shortener

## 🔹 Requirements & Estimations

**Functional requirements:**
1. URL shortening: long URL → short URL
2. URL redirecting: short URL → redirect to original URL
3. High availability, scalability, fault tolerance
- Shortened URLs: combination of [0-9, a-z, A-Z]
- Cannot be deleted or updated (simplified)

**Back-of-the-envelope estimation:**
- Write: **100 million URLs/day** → **~1,160 writes/sec**
- Read (10:1 ratio): **~11,600 reads/sec**
- 10-year storage: 100M × 365 × 10 = **365 billion records**
- Avg URL length: 100 bytes → **365 TB** storage over 10 years

---

## 🔹 API Design (REST)

```
POST api/v1/data/shorten
  Request: {longUrl: longURLString}
  Response: shortURL

GET api/v1/shortUrl
  Response: longURL (HTTP redirect)
```

---

## 🔹 URL Redirecting: 301 vs 302

| Aspect | 301 Redirect | 302 Redirect |
|--------|-------------|-------------|
| Meaning | **Permanently** moved | **Temporarily** moved |
| Browser behavior | Caches response; subsequent requests go directly to long URL server | Sends every request to shortener service first |
| Best for | **Reducing server load** | **Analytics** (track click rate, source) |

- Implementation: hash table stores `<shortURL, longURL>` → lookup → redirect

---

## 🔹 Hash Function Design

### Hash Value Length

- Characters: [0-9, a-z, A-Z] = **62 possible characters**
- Need smallest n where 62^n ≥ 365 billion
- **n = 7** → 62^7 ≈ 3.5 trillion (sufficient)

### Two Approaches

#### 1. Hash + Collision Resolution

- Use hash functions (CRC32, MD5, SHA-1) → take first 7 characters
- Problem: collisions → resolve by recursively appending predefined string and re-hashing until no collision
- DB lookup to check collision is expensive → use **Bloom filter** to improve performance
- Flow: longURL → hash → shortURL → check DB → collision? → append string → re-hash → repeat

#### 2. Base 62 Conversion

- Convert a unique numeric ID to base-62 representation
- Mapping: 0-0, ..., 9-9, 10-a, ..., 35-z, 36-A, ..., 61-Z
- Example: 11157₁₀ = 2×62² + 55×62¹ + 59×62⁰ = `[2, T, X]` → short URL: `2TX`

### 🆚 Hash + Collision Resolution vs Base 62 Conversion

| Aspect | Hash + Collision Resolution | Base 62 Conversion |
|--------|---------------------------|-------------------|
| URL length | Fixed | Variable (grows with ID) |
| ID generator needed? | No | Yes |
| Collision | Possible, must be resolved | Impossible (ID unique) |
| Predictability | Can't figure out next short URL | Easy to predict next URL → **security concern** |

---

## 🔹 URL Shortening Flow (Base 62 chosen)

1. Input: longURL
2. Check if longURL already in DB → if yes, return existing shortURL
3. If new: generate unique ID (primary key) via distributed ID generator
4. Convert ID to shortURL using base 62
5. Save (ID, shortURL, longURL) to DB

**Example**: longURL → ID generator returns `2009215674938` → base62 → `"zn9edcu"`

---

## 🔹 URL Redirecting Flow (Detailed)

1. User clicks short URL
2. **Load balancer** → web servers
3. Check **cache** for shortURL → if found, return longURL
4. If not in cache → query **database**
5. Return longURL to user

- Cache used because reads >> writes

---

## 🔹 Additional Talking Points

- **Rate limiter**: protect against malicious mass URL shortening requests
- **Web server scaling**: stateless → easy horizontal scaling
- **Database scaling**: replication and sharding
- **Analytics**: track click count, click times, sources

---

<a id="ch9"></a>
# 🔹 CHAPTER 9: Design a Web Crawler

## 🔹 What Is a Web Crawler

- Also called **robot** or **spider**
- Discovers new/updated web content (pages, images, videos, PDFs)
- Starts with seed URLs → follows links to collect new content

**Use cases:**
| Use Case | Description |
|----------|-------------|
| **Search engine indexing** | Most common; e.g., Googlebot |
| **Web archiving** | Preserve data; e.g., US Library of Congress, EU web archive |
| **Web mining** | Extract knowledge; e.g., financial firms crawl annual reports |
| **Web monitoring** | Monitor copyright/trademark infringement; e.g., Digimarc |

---

## 🔹 Requirements & Estimations

**Requirements:**
- Purpose: search engine indexing
- **1 billion pages/month** (HTML only)
- Consider newly added/edited pages
- Store crawled pages for **5 years**
- Ignore duplicate content

**Characteristics of a good crawler:**
- **Scalability**: parallelization for billions of pages
- **Robustness**: handle bad HTML, unresponsive servers, crashes, malicious links
- **Politeness**: don't overwhelm any single website
- **Extensibility**: minimal changes needed for new content types

**Estimations:**
- QPS: 1B / 30 / 24 / 3600 ≈ **~400 pages/sec** (peak: 800)
- Avg page size: **500 KB**
- Storage/month: 1B × 500KB = **500 TB**
- 5-year storage: 500 TB × 12 × 5 = **30 PB**

---

## 🔹 High-Level Design

### 🌳 Component Tree
```
Web Crawler
├── Seed URLs
├── URL Frontier (queue of URLs to download)
├── HTML Downloader
│   └── DNS Resolver
├── Content Parser
├── Content Seen? (dedup via hash comparison)
├── Content Storage (disk + memory)
├── Link Extractor (parse links from HTML)
├── URL Filter (exclude blacklisted/unwanted URLs)
├── URL Seen? (Bloom filter / hash table)
└── URL Storage (already visited URLs)
```

**Component details:**

- **Seed URLs**: starting point; strategy: divide by locality (countries) or topics (shopping, sports, etc.)
- **URL Frontier**: stores URLs to be downloaded; FIFO queue; handles prioritization + politeness
- **HTML Downloader**: fetches web pages; gets IPs from DNS Resolver
- **DNS Resolver**: URL → IP address translation
- **Content Parser**: validates downloaded pages (malformed pages waste storage); separate component to avoid slowing crawl
- **Content Seen?**: detects duplicate content via **hash comparison** of pages (not character-by-character); ~29% of web pages are duplicates
- **Content Storage**: disk for most content; memory for popular/frequently accessed content
- **URL Extractor**: parses HTML, extracts links, converts relative paths to absolute URLs
- **URL Filter**: excludes certain content types, file extensions, error links, blacklisted sites
- **URL Seen?**: tracks visited/queued URLs; prevents adding same URL multiple times (avoids infinite loops); implemented via Bloom filter or hash table
- **URL Storage**: stores already-visited URLs

---

### 🔹 Crawler Workflow (11 Steps)

1. Add seed URLs to URL Frontier
2. HTML Downloader fetches URLs from URL Frontier
3. HTML Downloader gets IPs from DNS Resolver → downloads pages
4. Content Parser validates HTML pages
5. Parsed content passed to "Content Seen?"
6. Content Seen checks if HTML already stored → if duplicate, discard; if new, pass to Link Extractor
7. Link Extractor extracts links from HTML
8. Extracted links passed to URL Filter
9. Filtered links passed to "URL Seen?"
10. URL Seen checks if URL already stored → if yes, skip
11. If URL is new → add to URL Frontier

---

## 🔹 Deep Dive

### DFS vs BFS

- **DFS**: not good — depth can be very deep
- **BFS**: commonly used via FIFO queue, but has problems:
  1. Most links from same page link back to same host → floods that server ("impolite")
  2. Doesn't consider **URL priority**

### URL Frontier — Two-Module Design

**Front queues → manage prioritization:**
- **Prioritizer**: computes URL priority (PageRank, traffic, update frequency)
- **Queues f1–fn**: each has assigned priority level
- **Queue selector**: randomly picks queue with bias toward higher priority

**Back queues → manage politeness:**
- **Queue router**: ensures each queue contains URLs from same host only
- **Mapping table**: host → queue mapping
- **FIFO queues b1–bn**: each contains URLs from one host
- **Queue selector**: maps each worker thread to a queue
- **Worker threads**: download one page at a time from assigned host; delay between downloads

### Freshness

- Pages constantly added/deleted/edited → must periodically recrawl
- Strategies:
  - Recrawl based on **update history**
  - Prioritize and recrawl **important pages first** and more frequently

### Storage for URL Frontier

- Hundreds of millions of URLs → too many for memory alone; disk too slow
- **Hybrid approach**: majority on disk; maintain **in-memory buffers** for enqueue/dequeue; periodically flush to disk

### HTML Downloader Optimizations

**Robots.txt (Robots Exclusion Protocol):**
- Standard for websites to communicate what crawlers may download
- Must check robots.txt before crawling a site
- Cache robots.txt results; re-download periodically

**Performance optimizations:**
1. **Distributed crawl**: partition URL space across multiple servers, each running multiple threads
2. **Cache DNS Resolver**: DNS is bottleneck (10–200ms response, synchronous); maintain DNS cache updated via cron jobs
3. **Locality**: distribute crawl servers geographically close to target websites
4. **Short timeout**: set max wait time; skip unresponsive hosts

### Robustness

- **Consistent hashing**: distribute load among downloaders; easy server add/remove
- **Save crawl states and data**: persist to storage for easy restart after disruptions
- **Exception handling**: handle errors gracefully without crashing
- **Data validation**: prevent system errors

### Extensibility

- System designed as pluggable modules
- Examples: add PNG Downloader module, Web Monitor module

### Detect and Avoid Problematic Content

1. **Redundant content**: ~30% of web is duplicate → use hashes/checksums
2. **Spider traps**: infinite loops (e.g., infinitely deep directory) → set **max URL length**; manual identification and blacklisting
3. **Data noise**: ads, spam, code snippets → exclude when possible

---

## 🔹 Additional Talking Points

- **Server-side rendering**: handle JavaScript/AJAX-generated links via dynamic rendering before parsing
- **Anti-spam filtering**: filter low-quality/spam pages
- **DB replication and sharding** for data layer
- **Horizontal scaling**: stateless servers; hundreds/thousands of download servers
- **Analytics**: collect data for fine-tuning

---

<a id="ch10"></a>
# 🔹 CHAPTER 10: Design a Notification System

## 🔹 Requirements

- Three notification types: **push notification** (iOS/Android), **SMS**, **Email**
- **Soft real-time** (deliver ASAP; slight delay acceptable under high load)
- Devices: iOS, Android, laptop/desktop
- Triggered by client apps or scheduled server-side
- Users can **opt-out**
- Volume: **10M mobile push**, **1M SMS**, **5M emails** per day

---

## 🔹 Notification Types & Third-Party Services

| Type | Service Flow | Key Components |
|------|-------------|----------------|
| **iOS Push** | Provider → **APNs** (Apple Push Notification Service) → iOS device | Device token + JSON payload |
| **Android Push** | Provider → **FCM** (Firebase Cloud Messaging) → Android device | Similar to iOS flow |
| **SMS** | Provider → **SMS Service** (Twilio, Nexmo) → Phone | Third-party commercial services |
| **Email** | Provider → **Email Service** (Sendgrid, Mailchimp) → Email | Commercial services for better delivery rate + analytics |

**iOS push notification payload example:**
```json
{
  "aps": {
    "alert": {
      "title": "Game Request",
      "body": "Bob wants to play chess",
      "action-loc-key": "PLAY"
    },
    "badge": 5
  }
}
```

---

## 🔹 Contact Info Gathering

- On app install or sign up → API servers collect contact info → store in DB
- **User table**: email, phone_number, country_code
- **Device table**: device_token, user_id, last_logged_in_at
- One user can have **multiple devices** → push notification sent to all

---

## 🔹 High-Level Design

### Initial Design Problems

| Problem | Explanation |
|---------|-------------|
| **SPOF** | Single notification server |
| **Hard to scale** | Everything in one server; can't scale components independently |
| **Performance bottleneck** | HTML rendering + third-party waits → overload at peak |

### Improved Design

Improvements:
- Move **DB and cache** out of notification server
- Add **multiple notification servers** with auto horizontal scaling
- Introduce **message queues** to decouple components

**Architecture flow:**
1. Services (1 to N) call notification server APIs
2. Notification servers fetch metadata (user info, device token, notification settings) from cache/DB
3. Notification events sent to **per-type message queues** (iOS PN queue, Android PN queue, SMS queue, Email queue)
4. **Workers** pull events from message queues
5. Workers send to **third-party services**
6. Third-party services deliver to user devices

**Notification server responsibilities:**
- Provide APIs (internally accessible or verified clients only to prevent spam)
- Basic validation (verify emails, phone numbers)
- Query DB/cache for rendering data
- Put notification data to message queues

**Key design decisions:**
- Each notification type gets its own **distinct message queue** → outage in one third-party service doesn't affect others
- Workers retry on error

---

## 🔹 Deep Dive

### Reliability

**Preventing data loss:**
- Persist notification data in **notification log database**
- Implement **retry mechanism**
- Notifications can be delayed or re-ordered but **never lost**

**Exactly-once delivery:**
- Short answer: **No** — distributed nature can cause duplicates
- Mitigation: **dedupe mechanism** — check event ID on arrival; discard if seen before; otherwise send

### Additional Components

| Component | Purpose |
|-----------|---------|
| **Notification templates** | Preformatted notifications with customizable parameters; maintains consistent format, reduces errors, saves time |
| **Notification settings** | Per-user opt-in/opt-out by channel (push/email/SMS); check before sending |
| **Rate limiting** | Cap notifications per user to prevent overwhelming; users may turn off all notifications if too frequent |
| **Retry mechanism** | Failed notification → back to message queue; if persistent failure → alert developers |
| **Security** | appKey/appSecret for push notification APIs; only authenticated clients can send |
| **Monitor queued notifications** | Track queue depth; if too large → add more workers |
| **Events tracking** | Track: pending → sent → delivered → click/unsubscribe → error; analytics service integration for open rate, click rate, engagement |

### Updated Design Additions

- Notification servers: + **authentication** + **rate-limiting**
- **Retry on error** back to message queue
- **Notification templates** for efficient creation
- **Analytics service** for tracking
- **Notification log** for persistence and monitoring

---

## 📝 Chapter 10 Summary

- Supports push notification, SMS, email
- **Message queues** decouple system components (per-channel queues)
- **Reliability**: notification log DB + retry mechanism
- **Security**: appKey/appSecret pairs
- **Tracking/monitoring**: at every stage of notification flow
- **Respect user settings**: check opt-in before sending
- **Rate limiting**: frequency capping per user

---

# ⚡ Quick-Recall Points (All Chapters)

- **CAP theorem**: can only have 2 of 3 (C, A, P); real-world distributed systems must tolerate P → choose CP or AP
- **CA systems cannot exist** in real-world distributed systems
- **W + R > N** → strong consistency guaranteed
- **Vector clock**: `[server, version]` pairs for conflict detection; ancestor if all counters ≤, sibling if any counter <
- **Gossip protocol**: decentralized failure detection via heartbeat counters
- **Merkle tree**: sync data proportional to differences, not total data
- **Snowflake ID**: 1 sign + 41 timestamp + 5 datacenter + 5 machine + 12 sequence = 64 bits; ~69 years; 4096 IDs/ms/machine
- **URL shortener hash length**: n=7 with base-62 → 3.5 trillion possible URLs
- **301 vs 302**: 301 = permanent (browser caches), 302 = temporary (better for analytics)
- **Web crawler BFS > DFS**; URL frontier handles both **politeness** (per-host queues) and **priority** (priority queues)
- **Bloom filter** used in: URL shortener (collision detection), web crawler (URL Seen?, Content Seen?), key-value store (read path SSTable lookup)
- **Notification system**: separate message queues per channel type prevent cascading failures

---

# 🎯 Actionable Takeaways

- Always clarify **requirements and trade-offs** before designing (consistency vs availability, read vs write optimization)
- Use **consistent hashing** for any system needing data distribution with minimal redistribution
- Use **quorum consensus** (tune N, W, R) for configurable consistency levels
- **Message queues** are essential for decoupling components in any large-scale system
- For **failure handling**, layer defenses: gossip → sloppy quorum → hinted handoff → Merkle trees → cross-DC replication
- **Bloom filters** are a versatile tool for probabilistic membership testing across many system designs
- When designing ID generators, **Snowflake-style** approaches satisfy most requirements (unique, sortable, 64-bit, distributed)
- For URL shorteners, **base-62 conversion** is cleaner than hash+collision resolution when a unique ID generator is available
- Web crawlers need **politeness** (per-host rate limiting) and **priority** (PageRank-based queue selection) built into the URL frontier
- Notification systems must handle **reliability** (no data loss), **deduplication**, **rate limiting**, and **user preferences**
