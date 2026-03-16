

# 📌 System Design Interview — Chapters 11–14

**Four system design problems: News Feed, Chat, Search Autocomplete, YouTube** | Type: System Design | Depth: Intermediate–Advanced

---

# 📑 Table of Contents

1. [Chapter 11: Design a News Feed System](#ch11)
2. [Chapter 12: Design a Chat System](#ch12)
3. [Chapter 13: Design a Search Autocomplete System](#ch13)
4. [Chapter 14: Design YouTube](#ch14)

---

# 🔹 Chapter 11: Design a News Feed System {#ch11}

## 🔹 1.1 Problem Definition & Scope

- **News feed** — constantly updating list of stories (status updates, photos, videos, links, app activity, likes) from people/pages/groups you follow
- Similar interview questions: Facebook feed, Instagram feed, Twitter timeline

### Requirements gathered (candidate ↔ interviewer)

| Aspect | Decision |
|--------|----------|
| Platform | Both mobile app and web |
| Key features | Publish a post; see friends' posts on feed |
| Feed ordering | Reverse chronological order |
| Max friends per user | 5,000 |
| Traffic volume | **10 million DAU** |
| Content types | Text, images, videos |

---

## 🔹 1.2 High-Level Design — Two Core Flows

> The entire design is divided into **feed publishing** and **news feed building**.

### Newsfeed APIs

- HTTP-based APIs for client ↔ server communication

**Feed publishing API:**

```
POST /v1/me/feed
Params: content (text), auth_token
```

**Newsfeed retrieval API:**

```
GET /v1/me/feed
Params: auth_token
```

### Feed Publishing Flow (High-Level)

- **User** → makes post via API (e.g., `/v1/me/feed?content=Hello&auth_token={token}`)
- **Load balancer** → distributes traffic to web servers
- **Web servers** → redirect to internal services
- **Post service** → persist post in DB and cache
- **Fanout service** → push content to friends' news feeds; stores in news feed cache
- **Notification service** → send push notifications to friends

### Newsfeed Building Flow (High-Level)

- **User** → sends `GET /v1/me/feed`
- **Load balancer** → routes to web servers
- **Web servers** → route to newsfeed service
- **Newsfeed service** → fetches from news feed cache
- **Newsfeed cache** → stores news feed IDs needed to render

---

## 🔹 1.3 Deep Dive — Feed Publishing

### Web Servers

- Enforce **authentication** (valid `auth_token` required) and **rate-limiting** (prevent spam/abuse)

### Fanout Service — Core Design Decision

**Fanout** — the process of delivering a post to all friends

🆚 **Fanout on Write vs Fanout on Read**

| Aspect | Fanout on Write (Push) | Fanout on Read (Pull) |
|--------|----------------------|---------------------|
| When computed | At write time (pre-computed) | At read time (on-demand) |
| Read speed | **Fast** — feed already built | **Slow** — computed on each request |
| Write cost | High for users with many friends (**hotkey problem**) | Low — no push needed |
| Inactive users | Wastes resources pre-computing | Efficient — no wasted work |
| Real-time | Yes — pushed immediately | No — delay on load |

### Adopted Solution: Hybrid Approach

- **Push model** for majority of users → fast feed retrieval
- **Pull model** for celebrities / users with many followers → avoids system overload
- **Consistent hashing** helps mitigate hotkey problem by distributing requests evenly

### Fanout Service — Detailed Steps

1. **Fetch friend IDs** from **graph database** (suited for friend relationships)
2. **Get friends info** from **user cache** → filter based on user settings (muted users, selective sharing, etc.)
3. **Send** friends list + new post ID to **message queue**
4. **Fanout workers** consume from message queue → store data in **news feed cache**
   - Cache structure: `<post_id, user_id>` mapping table
   - Only IDs stored (not full objects) → keeps memory small
   - **Configurable limit** on cached items — users rarely scroll thousands of posts → low cache miss rate
5. Store `<post_id, user_id>` in news feed cache

---

## 🔹 1.4 Deep Dive — Newsfeed Retrieval

### Retrieval Steps

1. User sends `GET /v1/me/feed`
2. Load balancer → web servers
3. Web servers → news feed service
4. News feed service gets **post IDs** from news feed cache
5. Fetches **complete user and post objects** from user cache + post cache → constructs **fully hydrated** news feed (username, profile picture, post content, images, etc.)
6. Returns fully hydrated feed as **JSON** to client
- **Media content** (images, videos) stored in **CDN** for fast retrieval

### Cache Architecture — 5 Layers

| Cache Layer | What It Stores | Sub-categories |
|-------------|---------------|----------------|
| **News Feed** | IDs of news feeds | — |
| **Content** | Post data | hot cache, normal |
| **Social Graph** | User relationships | follower, following |
| **Action** | User interactions | liked, replied, others |
| **Counters** | Aggregated counts | like counter, reply counter, other counters |

---

## 🔹 1.5 Wrap-Up & Scalability Talking Points

**Scaling the database:**
- Vertical vs Horizontal scaling
- SQL vs NoSQL
- Master-slave replication
- Read replicas
- Consistency models
- Database sharding

**Other talking points:**
- Keep web tier **stateless**
- **Cache data** as much as possible
- Support **multiple data centers**
- **Loosely couple** components with message queues
- **Monitor key metrics** — QPS during peak hours, latency on feed refresh

---

## 🌳 Concept Tree — News Feed System

```
News Feed System
├── Feed Publishing
│   ├── Web Servers (auth + rate limiting)
│   ├── Post Service (DB + cache)
│   ├── Fanout Service
│   │   ├── Fanout on Write (push)
│   │   ├── Fanout on Read (pull)
│   │   └── Hybrid Approach (push + pull for celebrities)
│   ├── Graph DB (friend IDs)
│   ├── Message Queue → Fanout Workers
│   └── Notification Service
├── Newsfeed Building / Retrieval
│   ├── News Feed Cache (post_id, user_id)
│   ├── User Cache + Post Cache (hydration)
│   └── CDN (media files)
└── Cache Architecture (5 layers)
    ├── News Feed
    ├── Content (hot + normal)
    ├── Social Graph
    ├── Action
    └── Counters
```

---

# 🔹 Chapter 12: Design a Chat System {#ch12}

## 🔹 2.1 Problem Definition & Scope

- Examples: WhatsApp, Facebook Messenger, WeChat, Slack, Discord

### Requirements gathered

| Aspect | Decision |
|--------|----------|
| Chat type | Both 1-on-1 and group chat |
| Platform | Both mobile and web |
| Scale | **50 million DAU** |
| Group member limit | Max **100 people** |
| Features | 1-on-1 chat, group chat, online indicator, text only |
| Message size limit | < 100,000 characters |
| End-to-end encryption | Not required initially |
| Chat history retention | **Forever** |
| Multiple device support | Yes |
| Push notifications | Yes |

---

## 🔹 2.2 High-Level Design

### Chat Service Core Functions

- Receive messages from clients
- Find right recipients and relay messages
- Hold messages for offline recipients until they come online

### Network Protocol Selection

**Sender side:** HTTP works fine (client-initiated)
- `keep-alive` header → persistent connection, fewer TCP handshakes
- Facebook initially used HTTP for sending

**Receiver side:** HTTP is client-initiated → need server-push techniques

🆚 **Polling vs Long Polling vs WebSocket**

| Technique | How It Works | Drawbacks |
|-----------|-------------|-----------|
| **Polling** | Client periodically asks server for new messages | Wastes resources; mostly empty responses |
| **Long Polling** | Client holds connection open until new messages arrive or timeout | Sender/receiver may hit different servers; no good disconnect detection; inefficient for inactive users |
| **WebSocket** | Bi-directional persistent connection; starts as HTTP then upgrades | Requires efficient connection management |

### WebSocket — Chosen Protocol

- **Bi-directional** and **persistent**
- Starts as HTTP → upgraded via handshake
- Uses port **80 or 443** → works through firewalls
- Used for **both sending and receiving** → simplifies design
- Server-side: efficient **connection management** is critical

### System Architecture — Three Categories

**1. Stateless Services** (over HTTP)
- Login, signup, user profile, group management
- Behind a **load balancer**
- Can be monolithic or microservices
- **Service discovery** — provides client with DNS hostnames of available chat servers

**2. Stateful Service** (over WebSocket)
- **Chat service** — each client maintains persistent connection
- Client doesn't switch servers unless server is unavailable
- Service discovery coordinates with chat service to avoid overloading

**3. Third-Party Integration**
- **Push notifications** — crucial for offline message alerts

### Adjusted High-Level Components

- **Chat servers** → message sending/receiving
- **Presence servers** → online/offline status
- **API servers** → login, signup, profile changes
- **Notification servers** → push notifications
- **Key-value store** → chat history persistence

### Scalability Note

- 1M concurrent users × 10K memory per connection = ~10GB → fits on one box in theory
- But **single server = single point of failure** → always design for distributed
- Fine to start with single server as a starting point, then scale

---

## 🔹 2.3 Storage Design

### Two Types of Data

| Data Type | Storage | Why |
|-----------|---------|-----|
| Generic (user profile, settings, friends list) | **Relational DB** with replication & sharding | Robust, reliable |
| Chat history | **Key-value store** | Horizontal scaling, low latency, handles long tail |

### Chat History Data Characteristics

- **Enormous volume** — Facebook Messenger + WhatsApp process **60 billion messages/day**
- **Recent chats** accessed frequently; old chats rarely
- Random access needed (search, mentions, jump to message)
- Read:write ratio ≈ **1:1** for 1-on-1 chat

### Why Key-Value Stores

- Easy horizontal scaling
- Very low latency
- RDBMs handle long-tail data poorly (expensive random access with large indexes)
- Proven: Facebook Messenger uses **HBase**, Discord uses **Cassandra**

### Data Models

**1-on-1 message table:**

| Column | Type |
|--------|------|
| **message_id** (PK) | bigint |
| message_from | bigint |
| message_to | bigint |
| content | text |
| created_at | timestamp |

- Can't rely on `created_at` for ordering — two messages can have same timestamp

**Group message table:**

| Column | Type |
|--------|------|
| **channel_id** (partition key) | bigint |
| **message_id** (sort key) | bigint |
| user_id | bigint |
| content | text |
| created_at | timestamp |

- `channel_id` is partition key because all group queries operate within a channel

### Message ID Generation

Requirements: **unique** + **sortable by time** (newer = higher ID)

| Approach | Notes |
|----------|-------|
| `auto_increment` (MySQL) | Not available in most NoSQL |
| Global 64-bit sequence generator (Snowflake) | Works but complex |
| **Local sequence number generator** | IDs unique only within a channel; sufficient for ordering within a conversation; easier to implement |

---

## 🔹 2.4 Deep Dive

### Service Discovery (Apache Zookeeper)

- Recommends **best chat server** based on geography, server capacity, etc.
- Registers all available chat servers
- Flow:
  1. User A logs in
  2. Load balancer → API servers (authentication)
  3. Service discovery picks best chat server (e.g., server 2)
  4. User A connects to chat server 2 via WebSocket

### 1-on-1 Chat Flow

1. User A → sends message to **Chat server 1**
2. Chat server 1 → gets **message ID** from ID generator
3. Chat server 1 → sends message to **message sync queue**
4. Message stored in **KV store**
5. a. User B online → forwarded to **Chat server 2** (where B is connected)
5. b. User B offline → **push notification** sent
6. Chat server 2 → forwards to User B via WebSocket

### Message Synchronization Across Multiple Devices

- Each device maintains `cur_max_message_id` (latest message ID on that device)
- New messages = where:
  - Recipient ID = currently logged-in user ID
  - Message ID in KV store > `cur_max_message_id`
- Each device independently fetches new messages from KV store

### Small Group Chat Flow

- Message from User A **copied to each member's message sync queue** (inbox)
- Each client only checks its own inbox → simplifies sync
- For small groups, storing a copy per recipient is acceptable
- **WeChat** uses similar approach; limits groups to **500 members**
- ⚠️ For very large groups, storing per-member copies is **not** acceptable

### Online Presence

**Presence servers** manage online status via WebSocket

**User Login:**
- WebSocket connection built → status set to `online` + `last_active_at` timestamp saved in KV store

**User Logout:**
- Status changed to `offline` in KV store

**User Disconnection — Heartbeat Mechanism:**
- Naive approach (mark offline on disconnect) causes too-frequent status flipping (e.g., going through tunnel)
- **Solution:** Client sends **heartbeat** every ~5 seconds
- If no heartbeat received within **x seconds** (e.g., 30s) → mark as offline

**Online Status Fanout:**
- **Publish-subscribe model** — each friend pair has a channel (e.g., A-B, A-C, A-D)
- Status change → published to relevant channels → friends subscribed receive update
- Communication via real-time WebSocket
- ⚠️ Works for **small groups** (WeChat caps at 500); for large groups (100K members) → each status change = 100K events → **fetch on-demand** instead (when entering group or refreshing friend list)

---

## 🔹 2.5 Wrap-Up & Additional Topics

- **Media file support** — compression, cloud storage, thumbnails
- **End-to-end encryption** — WhatsApp model; only sender/recipient can read
- **Client-side caching** — reduces data transfer
- **Improve load time** — Slack uses geo-distributed edge cache (Flannel)
- **Error handling:**
  - Chat server goes offline → Zookeeper provides new server for reconnection
  - Message resend → retry + queueing

---

## 🌳 Concept Tree — Chat System

```
Chat System
├── Communication Protocols
│   ├── HTTP (sender side, stateless services)
│   ├── Polling / Long Polling (rejected)
│   └── WebSocket (chosen: bidirectional, persistent)
├── Stateless Services (HTTP)
│   ├── Auth, Signup, Profile, Group Mgmt
│   └── Service Discovery (Zookeeper)
├── Stateful Services (WebSocket)
│   ├── Chat Servers
│   └── Presence Servers
├── Storage
│   ├── Relational DB (generic data)
│   └── Key-Value Store (chat history: HBase / Cassandra)
├── Message Flows
│   ├── 1-on-1: sender → chat server → msg sync queue → KV store → receiver's chat server
│   ├── Multi-device sync (cur_max_message_id)
│   └── Group chat: message copied to each member's inbox queue
├── Online Presence
│   ├── Login/Logout → KV store update
│   ├── Heartbeat mechanism (disconnect handling)
│   └── Pub-sub fanout (per friend-pair channel)
└── Third-Party: Push Notification Servers
```

---

# 🔹 Chapter 13: Design a Search Autocomplete System {#ch13}

## 🔹 3.1 Problem Definition & Scope

- Also called: **typeahead**, **search-as-you-type**, **incremental search**, "design top k"
- Example: Google search suggestions as you type

### Requirements gathered

| Aspect | Decision |
|--------|----------|
| Matching | Only at the **beginning** of a query (prefix matching) |
| Number of suggestions | **5** |
| Ranking | By **popularity** (historical query frequency) |
| Spell check | Not supported |
| Language | English only (lowercase alphabetic) |
| Users | **10 million DAU** |

### Derived Requirements

- **Fast response time** — must return within **100ms** (Facebook's standard); otherwise causes stuttering
- **Relevant** suggestions
- **Sorted** by popularity
- **Scalable** for high traffic
- **Highly available**

### Back-of-Envelope Estimation

- 10M DAU × 10 searches/day × 20 characters/query = **~24,000 QPS**
- **Peak QPS** ≈ 48,000
- Query size: ~20 bytes (4 words × 5 chars)
- Each keystroke triggers a request → 20 requests per query
- 20% of daily queries are new → **0.4 GB** new data/day

---

## 🔹 3.2 High-Level Design

### Two Sub-systems

| Component | Purpose |
|-----------|---------|
| **Data gathering service** | Collects and aggregates query data |
| **Query service** | Returns top 5 suggestions for a prefix |

### Data Gathering Service (simplified)

- Frequency table: `(query, frequency)`
- Each new query → increment frequency counter

### Query Service (simplified)

- SQL approach (small data sets):

```sql
SELECT * FROM frequency_table
WHERE query LIKE 'prefix%'
ORDER BY frequency DESC
LIMIT 5
```

- ⚠️ Becomes bottleneck at scale → need optimization

---

## 🔹 3.3 Deep Dive

### Trie Data Structure

- **Trie** (prefix tree) — tree-like structure that compactly stores strings
- Name from "re**trie**val"
- Properties:
  - Root = empty string
  - Each node stores a character; has up to **26 children**
  - Each node represents a word or prefix
  - Frequency info added to leaf/word nodes

### Basic Trie Algorithm for Top-K

Variables: `p` = prefix length, `n` = total nodes, `c` = children of prefix node

1. **Find prefix node** — O(p)
2. **Traverse subtree** to get all valid children — O(c)
3. **Sort children**, return top k — O(c log c)

**Total: O(p) + O(c) + O(c log c)** → too slow for large tries

### Two Key Optimizations

**Optimization 1: Limit max prefix length**
- Users rarely type very long queries → cap at ~50 chars
- O(p) → **O(1)**

**Optimization 2: Cache top-k queries at each node**
- Store top 5 queries directly on each trie node
- Trades space for time
- Example: node "be" stores `[best: 35, bet: 29, bee: 20, be: 15, beer: 10]`
- Lookup becomes **O(1)** → find prefix node O(1) + return cached top-k O(1)

### Data Gathering Service — Redesigned

**Pipeline:** Analytics Logs → Aggregators → Aggregated Data → Workers → Trie DB ← Trie Cache

| Component | Role |
|-----------|------|
| **Analytics Logs** | Raw query data; append-only, not indexed |
| **Aggregators** | Aggregate data (e.g., weekly for non-real-time; shorter intervals for Twitter-like apps) |
| **Aggregated Data** | Table: `(query, time, frequency)` — weekly sums |
| **Workers** | Build trie data structure asynchronously; store in Trie DB |
| **Trie Cache** | Distributed cache; weekly snapshot of DB; keeps trie in memory |
| **Trie DB** | Persistent storage |

**Trie DB storage options:**
1. **Document store** (e.g., MongoDB) — serialize trie, store periodically
2. **Key-value store** — each prefix → key; node data → value

### Query Service — Optimized

1. Search query → **load balancer**
2. Load balancer → **API servers**
3. API servers → **Trie Cache** → construct suggestions
4. Cache miss → replenish from **Trie DB**

**Additional optimizations:**
- **AJAX requests** — no full page refresh
- **Browser caching** — Google caches results for 1 hour (`cache-control: private, max-age=3600`)
- **Data sampling** — log only 1 out of every N requests (reduces processing/storage)

### Trie Operations

**Create:** Workers build from aggregated data

**Update — two options:**
- Option 1: **Replace entire trie weekly** (preferred)
- Option 2: **Update individual nodes** directly — slow because ancestors up to root must also be updated (they cache top-k of children)

**Delete:**
- **Filter layer** between API servers and Trie Cache
- Removes hateful, violent, explicit, or dangerous suggestions
- Physical removal from DB done asynchronously → correct data for next trie build

### Scale the Storage

- Naive: shard by first character (a–m on server 1, n–z on server 2)
- Can go to 26 servers, or further shard on 2nd/3rd character
- ⚠️ **Uneven distribution** — far more words start with 'c' than 'x'
- **Solution:** **Shard map manager** — analyzes historical data distribution, maintains lookup DB
  - Example: if 's' has as many queries as 'u'+'v'+'w'+'x'+'y'+'z' combined → give 's' its own shard

---

## 🔹 3.4 Wrap-Up & Follow-Up Questions

**Multi-language support:**
- Store **Unicode** characters in trie nodes

**Country-specific queries:**
- Build **different tries per country**
- Store in **CDN** for better response time

**Trending (real-time) search queries:**
- Weekly offline rebuild won't catch breaking news
- Ideas:
  - Reduce working data set by **sharding**
  - Adjust ranking model → **weight recent queries** more heavily
  - Use **stream processing** systems: Apache Hadoop MapReduce, Spark Streaming, Storm, Kafka

---

## 🌳 Concept Tree — Search Autocomplete

```
Search Autocomplete
├── Data Gathering Service
│   ├── Analytics Logs (raw, append-only)
│   ├── Aggregators (weekly or shorter)
│   ├── Workers (build trie async)
│   ├── Trie DB (document store or KV store)
│   └── Trie Cache (distributed, weekly snapshot)
├── Query Service
│   ├── Load Balancer → API Servers → Trie Cache
│   ├── AJAX requests
│   ├── Browser caching (1hr)
│   └── Data sampling (1/N)
├── Trie Data Structure
│   ├── Prefix tree with frequency info
│   ├── Optimization 1: limit prefix length → O(1)
│   └── Optimization 2: cache top-k at each node → O(1)
├── Trie Operations
│   ├── Create (workers from aggregated data)
│   ├── Update (weekly replace or node-level)
│   └── Delete (filter layer + async DB removal)
└── Storage Scaling
    ├── Shard by character (naive)
    └── Shard map manager (smart, based on data distribution)
```

---

# 🔹 Chapter 14: Design YouTube {#ch14}

## 🔹 4.1 Problem Definition & Scope

- Applies to: Netflix, Hulu, or any video sharing platform

### YouTube Stats (2020)

- **2 billion** monthly active users
- **5 billion** videos watched/day
- 73% of US adults use YouTube
- 50 million creators
- $15.1 billion ad revenue (2019, +36% from 2018)
- 37% of all mobile internet traffic
- Available in 80 languages

### Requirements gathered

| Aspect | Decision |
|--------|----------|
| Key features | Upload video + watch video |
| Clients | Mobile apps, web browsers, smart TV |
| DAU | **5 million** |
| Avg daily time | 30 minutes |
| International users | Yes, large percentage |
| Video resolutions | Most formats accepted |
| Encryption | Yes |
| Max video size | **1 GB** |
| Cloud infra | Leverage existing (AWS, GCP, Azure) |

### Design Focus

- Fast video upload
- Smooth video streaming
- Ability to change video quality
- Low infrastructure cost
- High availability, scalability, reliability

### Back-of-Envelope Estimation

- 5M DAU × 10% upload × 300 MB avg = **150 TB/day** storage
- CDN cost (CloudFront, US): ~$0.02/GB
  - 5M × 5 videos × 0.3 GB × $0.02 = **$150,000/day** for streaming alone

---

## 🔹 4.2 High-Level Design

> Use existing cloud services (CDN, blob storage) rather than building from scratch. Even Netflix uses AWS; Facebook uses Akamai CDN.

### Three Top-Level Components

| Component | Role |
|-----------|------|
| **Client** | Computer, mobile, smart TV |
| **CDN** | Stores and streams videos |
| **API servers** | Everything except video streaming (feed recommendation, upload URL generation, metadata, signup) |

### Video Uploading Flow — Components

| Component | Purpose |
|-----------|---------|
| **User** | Uploads from device |
| **Load balancer** | Distributes requests to API servers |
| **API servers** | Handle non-streaming requests |
| **Metadata DB** | Video metadata; sharded + replicated |
| **Metadata cache** | Cached video metadata + user objects |
| **Original storage** | Blob storage for original uploaded videos |
| **Transcoding servers** | Convert video formats (MPEG, HLS, etc.) |
| **Transcoded storage** | Blob storage for transcoded files |
| **CDN** | Caches transcoded videos for streaming |
| **Completion queue** | Message queue for transcoding completion events |
| **Completion handler** | Workers that update metadata DB/cache on completion |

### Video Upload — Two Parallel Processes

**Flow A: Upload the actual video**
1. Video uploaded to **original storage**
2. **Transcoding servers** fetch and transcode
3. In parallel:
   - 3a. Transcoded video → **transcoded storage** → distributed to **CDN**
   - 3b. Completion event → **completion queue** → **completion handler** → updates **metadata DB + cache**
4. API servers inform client: upload successful, ready for streaming

**Flow B: Update metadata**
- Client sends metadata (file name, size, format) to API servers in parallel with upload
- API servers update metadata cache + DB

### Video Streaming Flow

- **Streaming ≠ downloading** — client loads data incrementally
- Videos streamed from **CDN** (nearest edge server → low latency)

**Streaming Protocols:**
- **MPEG-DASH** (Dynamic Adaptive Streaming over HTTP)
- **Apple HLS** (HTTP Live Streaming)
- **Microsoft Smooth Streaming**
- **Adobe HDS** (HTTP Dynamic Streaming)
- Different protocols support different encodings and players → choose based on use case

---

## 🔹 4.3 Deep Dive

### Video Transcoding — Why Needed

- Raw video is huge (~hundreds of GB for 1hr HD at 60fps)
- Different devices/browsers support different formats
- Adaptive quality: higher resolution for high bandwidth, lower for low bandwidth
- Network conditions change (especially mobile) → auto-switch quality

**Encoding format parts:**
- **Container** — basket for video, audio, metadata (e.g., `.avi`, `.mov`, `.mp4`)
- **Codecs** — compression/decompression algorithms (H.264, VP9, HEVC)

### DAG (Directed Acyclic Graph) Model

- Different creators have different processing needs (watermarks, thumbnails, HD vs not)
- Facebook's streaming engine uses **DAG programming model** → tasks in stages, sequential or parallel

**DAG splits original video into:**
- **Video** → inspection, transcoding, thumbnail, watermark
- **Audio** → audio encoding
- **Metadata** → extracted directly
- Final step: **Assemble**

### Video Transcoding Architecture — 6 Components

```
Preprocessor → DAG Scheduler → Resource Manager → Task Workers → Encoded Video
                                       ↕
                               Temporary Storage
```

**1. Preprocessor:**
- **Video splitting** by GOP (Group of Pictures) — each GOP is independently playable, ~few seconds
- For old clients that can't split: server-side splitting by GOP
- **DAG generation** from client configuration files
- **Cache** segmented videos + metadata in temporary storage (for retry on failure)

**2. DAG Scheduler:**
- Splits DAG into **stages of tasks** → puts in resource manager's task queue
- Stage 1: split into video, audio, metadata
- Stage 2: video encoding + thumbnail; audio encoding

**3. Resource Manager:**
- **Task queue** — priority queue of tasks
- **Worker queue** — priority queue of worker utilization
- **Running queue** — currently executing task/worker pairs
- **Task scheduler** — picks highest priority task + optimal worker → assigns, tracks, removes on completion

**4. Task Workers:**
- Execute DAG-defined tasks (watermark, encoding, thumbnail, merger, etc.)

**5. Temporary Storage:**
- Metadata → cached in **memory**
- Video/audio data → **blob storage**
- Freed after processing complete

**6. Encoded Video:**
- Final output (e.g., `funny_720p.mp4`)

### System Optimizations

**Speed: Parallelize video uploading**
- Split video into **GOP chunks** on client side → upload in parallel
- Enables **fast resumable uploads** on failure

**Speed: Upload centers close to users**
- Multiple upload centers globally (use CDN as upload centers)
- US users → North America center; China users → Asian center

**Speed: Parallelism everywhere**
- Without message queues: each module waits for previous output (tight coupling)
- With **message queues** between each stage (download → encode → upload): modules work independently in parallel

**Safety: Pre-signed upload URL**
1. Client requests pre-signed URL from API servers
2. API servers return pre-signed URL
3. Client uploads using pre-signed URL
- Ensures only **authorized users** upload to correct locations
- Term used by AWS S3; Azure calls it "Shared Access Signature"

**Safety: Protect videos**
- **DRM** — Apple FairPlay, Google Widevine, Microsoft PlayReady
- **AES encryption** with authorization policy → decrypted on playback
- **Visual watermarking** — company logo/name overlay

**Cost-saving: CDN optimization**
- YouTube follows **long-tail distribution** — few popular videos, many with few/no viewers
- Optimizations:
  1. Popular videos → CDN; others → high-capacity **video servers**
  2. Less popular content → fewer encoded versions; short videos encoded **on-demand**
  3. Region-specific popular videos → don't distribute to other regions
  4. **Build your own CDN** + partner with ISPs (Netflix approach) → improves experience, reduces bandwidth costs

### Error Handling

| Error Type | Description | Strategy |
|------------|-------------|----------|
| **Recoverable** | e.g., transcode failure | Retry; if persistent, return error code |
| **Non-recoverable** | e.g., malformed video | Stop tasks, return error code |

**Component-specific error playbook:**

| Component | Action |
|-----------|--------|
| Upload error | Retry |
| Split video error | Server-side splitting for old clients |
| Transcoding error | Retry |
| Preprocessor error | Regenerate DAG |
| DAG scheduler error | Reschedule task |
| Resource manager queue down | Use replica |
| Task worker down | Retry on new worker |
| API server down | Stateless → redirect to another |
| Metadata cache down | Use replica node; spin up replacement |
| Metadata DB master down | Promote slave to master |
| Metadata DB slave down | Use another slave; spin up replacement |

---

## 🔹 4.4 Wrap-Up & Additional Topics

- **Scale API tier** — stateless, easy horizontal scaling
- **Scale database** — replication + sharding
- **Live streaming** — similar (upload, encode, stream) but:
  - Higher latency requirements → different protocol
  - Lower parallelism need (real-time small chunks)
  - Different error handling (no slow retries)
- **Video takedowns** — copyright, illegal content; detected during upload or via user flagging

---

## 🌳 Concept Tree — YouTube

```
YouTube / Video Streaming
├── Video Uploading
│   ├── Original Storage (blob)
│   ├── Transcoding Pipeline
│   │   ├── Preprocessor (split GOPs, generate DAG, cache)
│   │   ├── DAG Scheduler (stages of tasks)
│   │   ├── Resource Manager (task/worker/running queues)
│   │   ├── Task Workers (watermark, encode, thumbnail, merge)
│   │   └── Temporary Storage (memory + blob)
│   ├── Transcoded Storage → CDN distribution
│   ├── Completion Queue → Handler → Metadata DB/Cache update
│   └── Metadata update (parallel with upload)
├── Video Streaming
│   ├── CDN (nearest edge server)
│   └── Streaming Protocols (MPEG-DASH, HLS, Smooth, HDS)
├── Optimizations
│   ├── Speed: GOP splitting, upload centers, message queues
│   ├── Safety: pre-signed URLs, DRM, AES, watermarks
│   └── Cost: long-tail CDN, on-demand encoding, ISP partnerships
└── Error Handling
    ├── Recoverable (retry)
    └── Non-recoverable (stop + error code)
```

---

# ⚡ Quick-Recall Points

- News Feed: **Hybrid fanout** — push for normal users, pull for celebrities
- News Feed cache stores only **IDs** (not full objects) to save memory
- Chat: **WebSocket** chosen for bi-directional real-time; starts as HTTP upgrade
- Chat: **Key-value stores** for chat history (HBase at Facebook, Cassandra at Discord)
- Chat: **Heartbeat mechanism** for presence (not naive disconnect/reconnect)
- Chat: **`cur_max_message_id`** per device for multi-device sync
- Autocomplete: **Trie with cached top-k at each node** → O(1) lookup
- Autocomplete: Trie rebuilt **weekly** (not real-time) for most use cases
- Autocomplete: **Shard map manager** for uneven data distribution
- YouTube: **DAG model** for flexible video processing pipelines
- YouTube: **Pre-signed URLs** for secure uploads
- YouTube: Videos follow **long-tail distribution** → CDN only for popular content
- YouTube: CDN cost estimated at **$150K/day** for 5M DAU

---

# 🔗 Relationships & Dependencies

- **Fanout service** (News Feed) and **message sync queue** (Chat) both solve the "deliver to N recipients" problem — fan-out pattern
- **Service discovery** (Zookeeper) in Chat ↔ **Load balancer** in News Feed — both route requests but chat needs **stateful** routing
- **Key-value stores** appear in both Chat (message storage) and Autocomplete (trie storage) — chosen for horizontal scaling + low latency
- **CDN** is central to both News Feed (media) and YouTube (video streaming) — cost is a major concern at scale
- **Message queues** appear in all four systems for decoupling and parallelism
- **Consistent hashing** mitigates hotkey problems in both News Feed fanout and Autocomplete sharding

---

# 📝 Summary

- **News Feed**: Two flows (publish + retrieve). Hybrid push/pull fanout. 5-layer cache. Graph DB for friends. Message queue → fanout workers → news feed cache. Media via CDN. Key trade-off: write amplification (push) vs read latency (pull).
- **Chat System**: WebSocket for real-time bidirectional communication. Stateless services (HTTP) + stateful chat service (WS). KV store for chat history (60B msgs/day at FB scale). Zookeeper for service discovery. Heartbeat for presence. Per-recipient inbox queues for group chat. `cur_max_message_id` for multi-device sync.
- **Search Autocomplete**: Trie with top-k cached at each node gives O(1) lookup. Data gathered via analytics logs → aggregated weekly → workers rebuild trie. Trie Cache + Trie DB. Filter layer removes unwanted suggestions. Shard map manager handles uneven data distribution. Browser caching + AJAX + data sampling reduce load.
- **YouTube**: CDN + blob storage (leverage cloud). Two parallel flows for upload (video + metadata). DAG-based transcoding pipeline (preprocessor → scheduler → resource manager → task workers). GOP splitting for parallel upload/encoding. Pre-signed URLs for security. DRM/AES/watermarking for protection. Long-tail optimization: popular → CDN, others → video servers. Comprehensive error handling playbook per component.
