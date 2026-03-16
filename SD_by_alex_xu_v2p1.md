

# 📌 System Design Interview Vol. 2 — Ch. 1: Proximity Service & Ch. 2: Nearby Friends

One-line summary: Two location-based system designs — one for finding nearby businesses (static data) and one for tracking nearby friends (dynamic, real-time data) — covering geospatial indexing, caching, pub/sub messaging, and scaling strategies.

**Type:** System Design | **Depth:** Intermediate–Advanced

---

## 📑 Table of Contents

1. [Chapter 1 — Proximity Service](#ch1)
   - Requirements & Estimation
   - API Design
   - Data Model
   - High-Level Design
   - Geospatial Indexing Algorithms
   - Geohash vs Quadtree vs Google S2
   - Deep Dive: Scaling, Caching, Regions
   - Final Architecture & Flow
2. [Chapter 2 — Nearby Friends](#ch2)
   - Requirements & Estimation
   - High-Level Design & WebSocket Architecture
   - Redis Pub/Sub Routing Layer
   - API & Data Model
   - Deep Dive: Scaling Each Component
   - Distributed Redis Pub/Sub Cluster
   - Operational Considerations
   - Alternative: Erlang
3. [Comparisons](#comparisons)
4. [Key Definitions](#definitions)
5. [Summary](#summary)

---

## 🔹 Chapter 1 — Proximity Service {#ch1}

### Requirements & Scope

**Functional Requirements:**
- Return all businesses based on user's location (lat/long) and radius
- Business owners can add/delete/update a business — changes **not** reflected in real-time (next-day effective)
- Customers can view detailed business information

**Non-Functional Requirements:**
- **Low latency** — users see nearby businesses quickly
- **Data privacy** — comply with GDPR, CCPA; location is sensitive
- **High availability & scalability** — handle peak-hour spikes in dense areas

**Key Assumptions:**
- 100 million DAU, 200 million businesses
- User makes ~5 searches/day
- Max radius: 20km; radius options: 0.5km, 1km, 2km, 5km, 20km
- Slow user movement → no need for constant refresh
- Newly added/updated businesses effective next day

**Back-of-Envelope:**
- Seconds/day ≈ 10⁵ (used throughout the book)
- Search QPS = (100M × 5) / 10⁵ = **5,000 QPS**

---

### API Design

**Search API (core):**

```
GET /v1/search/nearby
```

| Field | Description | Type |
|-------|-------------|------|
| `latitude` | Latitude of user location | decimal |
| `longitude` | Longitude of user location | decimal |
| `radius` | Optional, default 5000m | int |

Response: `{ "total": 10, "businesses": [{business object}] }`

**Business CRUD APIs:**

| API | Detail |
|-----|--------|
| `GET /v1/businesses/:id` | Get business details |
| `POST /v1/businesses` | Add a business |
| `PUT /v1/businesses/:id` | Update a business |
| `DELETE /v1/businesses/:id` | Delete a business |

- Real-world references: Google Places API, Yelp business endpoints
- Pagination is relevant but not the chapter's focus

---

### Data Model

**Read/Write Ratio:**
- **Read-heavy** — nearby search + viewing business detail are very common
- **Write volume low** — adding/editing businesses is infrequent
- → Relational database (e.g., MySQL) is a good fit

**Business Table:**
- PK: `business_id`
- Fields: address, city, state, country, latitude, longitude

**Geo Index Table:**
- Used for efficient spatial queries
- Discussed in detail in the scaling section

---

### High-Level Design

**Two main services:**

1. **Location-Based Service (LBS)**
   - Core service finding nearby businesses for given radius + location
   - Read-heavy, no write requests
   - High QPS especially during peak hours in dense areas
   - **Stateless** → easy horizontal scaling

2. **Business Service**
   - Handles business CRUD (write operations, low QPS)
   - Also handles customer viewing business details (high QPS at peak)

**Database Cluster:**
- **Primary-secondary setup** — primary handles writes, multiple replicas handle reads
- Replication delay → some inconsistency between LBS reads and primary writes
- Acceptable because business info doesn't need real-time updates

**Scalability:**
- Both services are stateless → auto-scale for peak traffic (e.g., mealtime)
- Cloud deployment across regions and availability zones for improved availability

---

### Geospatial Indexing Algorithms

> **Core insight:** The key challenge is mapping 2D spatial data to enable efficient search. DB indexes improve speed in only one dimension. We need to map 2D data → 1D for efficient indexing.

**Two broad categories:**
- **Hash-based:** Even grid, Geohash, Cartesian tiers
- **Tree-based:** Quadtree, Google S2, R-tree

Most widely used in practice: **Geohash, Quadtree, Google S2**

---

#### Option 1: Two-Dimensional Search (Naive)

- Draw circle with radius, find all businesses inside
- SQL: `WHERE latitude BETWEEN ... AND longitude BETWEEN ...`
- **Problem:** Even with indexes on lat/long columns, each dimension returns huge datasets; intersection is inefficient
- Not practical at scale

#### Option 2: Evenly Divided Grid

- Divide world into equal-sized grids
- **Problem:** Business distribution is uneven — dense in NYC, empty in deserts/oceans
- Produces very uneven data distribution
- Finding neighboring grids is also challenging

#### Option 3: Geohash ⭐

**How it works:**
- Reduces 2D (lat/long) → 1D string of letters and digits
- Recursively divides world into smaller grids with each additional bit
- Uses base32 representation

**Process:**
1. Divide planet into 4 quadrants along prime meridian and equator
   - Lat [-90, 0] → 0; Lat [0, 90] → 1
   - Long [-180, 0] → 0; Long [0, 180] → 1
2. Recursively subdivide each grid into 4 smaller grids
3. Each grid represented by alternating longitude bit and latitude bit
4. Repeat until desired precision

**Geohash precision table (key lengths 4–6 most useful):**

| Length | Grid Size |
|--------|-----------|
| 4 | 39.1km × 19.5km |
| 5 | 4.9km × 4.9km |
| 6 | 1.2km × 609.4m |

**Radius → Geohash length mapping:**

| Radius | Geohash Length |
|--------|---------------|
| 0.5km | 6 |
| 1km | 5 |
| 2km | 5 |
| 5km | 4 |
| 20km | 4 |

**Boundary Issues:**

⚠️ **Issue 1:** Two locations can be very close but have **no shared prefix** (e.g., on opposite sides of equator/prime meridian). Example: La Roche-Chalais (`u000`) and Pomerol (`ezzz`) — 30km apart, zero shared prefix.
- A simple `WHERE geohash LIKE '9q8zn%'` query fails to fetch all nearby businesses

⚠️ **Issue 2:** Two positions can have a long shared prefix but belong to different geohash grids (on a grid boundary)

**Solution:** Fetch businesses from current grid **AND all 8 neighboring grids**. Neighbor geohashes can be calculated in constant time.

**Expanding search if not enough results:**
- Remove last digit of geohash → larger grid
- Keep expanding until enough results found

---

#### Option 4: Quadtree ⭐

**What it is:** In-memory tree data structure that recursively subdivides 2D space into 4 quadrants until each grid meets a criterion (e.g., ≤100 businesses per grid)

> **Key:** Quadtree is an **in-memory** data structure, **not** a database solution. Built on each LBS server at startup time.

**Building process:**
- Root = entire world map
- Recursively break into 4 quadrants (NW, NE, SW, SE)
- Stop subdividing when grid has ≤100 businesses

```
public void buildQuadtree(TreeNode node) {
  if (countNumberOfBusinessesInCurrentGrid(node) > 100) {
    node.subdivide();
    for (TreeNode child : node.getChildren()) {
      buildQuadtree(child);
    }
  }
}
```

**Memory estimation (200M businesses):**

| Node Type | Data | Size |
|-----------|------|------|
| Leaf node | Grid coordinates (4×8 bytes) + business IDs (100×8 bytes) | 832 bytes |
| Internal node | Grid coordinates (4×8 bytes) + 4 child pointers (4×8 bytes) | 64 bytes |

- Leaf nodes: ~200M / 100 = ~2 million
- Internal nodes: 2M × 1/3 ≈ 0.67 million
- **Total memory: ~1.71 GB** — easily fits on one server
- But may need multiple servers for CPU/network bandwidth

**Build time:** O((n/100) × log(n/100)) — a few minutes for 200M businesses

**Search process:**
1. Build quadtree in memory
2. Traverse from root to leaf containing search origin
3. If leaf has 100 businesses, return; otherwise add from neighbors

**Operational considerations:**
- Server can't serve traffic during build → **incremental rollout** (small subset at a time)
- Blue/green deployment possible but risky (all new servers fetching 200M records simultaneously)
- **Updating the quadtree:**
  - Easiest: incrementally rebuild (nightly job) — some stale data temporarily
  - Risk: tons of cache keys invalidated at once
  - Alternative: update on-the-fly — requires locking, complicates implementation

---

#### Option 5: Google S2

- In-memory solution mapping sphere → 1D index using **Hilbert curve** (space-filling curve)
- Key property: points close on Hilbert curve are close in 1D space → efficient 1D search
- Used by Google Maps, Tinder

**Advantages:**
- Great for **geofencing** — can cover arbitrary areas with varying levels
- **Region Cover algorithm** — flexible cell sizes (min/max level, max cells) → more granular than fixed-precision geohash

---

### 🆚 Geohash vs Quadtree

| Aspect | Geohash | Quadtree |
|--------|---------|----------|
| Implementation | Easy, no tree needed | Harder, must build tree |
| Radius search | Supports specified radius | Better for k-nearest queries |
| Grid size | Fixed per precision level | Dynamic based on density |
| Index updates | Easy — delete row by (geohash, business_id) | Complex — traverse root→leaf, O(log n), needs locking for multi-thread |
| Rebalancing | N/A | Complex when leaf is full |
| Storage | Database (persistent) | In-memory (rebuilt at startup) |

**Who uses what:**

| Geo Index | Companies |
|-----------|-----------|
| Geohash | Bing Maps, Redis, MongoDB, Lyft |
| Quadtree | Yext |
| Both | Elasticsearch |
| S2 | Google Maps, Tinder |

> **Interview tip:** Choose geohash or quadtree — S2 is too complex to explain clearly.

---

### Deep Dive

#### Scaling the Database

**Business table:** Shard by `business_id` — even load distribution, easy maintenance

**Geospatial index table (Geohash):**

Two schema options:

| Option | Structure | Recommendation |
|--------|-----------|----------------|
| Option 1 | One row per geohash with JSON array of business_ids | ❌ — requires scanning array, locking for concurrent updates |
| Option 2 | One row per (geohash, business_id) compound key | ✅ — simple add/remove, no locking needed |

**Scaling the geo index:**
- Full dataset is small (~1.71GB) — fits in working set of modern DB server
- ⚠️ **Common mistake:** jumping to sharding when not needed
- **Better approach:** Use **read replicas** instead of sharding — simpler to develop and maintain
- Sharding adds complexity (application-layer logic) and isn't justified here

---

#### Caching

> **Important caveat:** Caching is NOT an obvious win here. The dataset is small, fits in DB working set, queries are not I/O bound. Adding read replicas may suffice. Discuss with interviewer carefully — requires benchmarking and cost analysis.

**Cache key considerations:**
- ❌ Raw lat/long coordinates — inaccurate from phones, change slightly with movement
- ✅ **Geohash** — small location changes map to same cache key

**Two types of cached data:**

| Key | Value | Redis Cache Name |
|-----|-------|-----------------|
| geohash | List of business IDs in grid | "Geohash" |
| business_id | Business object (name, address, images, etc.) | "Business info" |

**Caching business IDs per geohash:**
1. Query: `SELECT business_id FROM geohash_index WHERE geohash LIKE '{:geohash}%'`
2. On cache miss → run query → store in Redis with 1-day TTL
3. On business add/edit/delete → update DB + invalidate cache

**Memory estimation for geohash cache:**
- 8 bytes × 200M businesses × 3 precisions (geohash_4, geohash_5, geohash_6) ≈ **~5GB**
- Fits in single modern Redis server, but deploy globally for availability + latency

**Cache update strategy:** Nightly job (matches next-day business agreement)

---

#### Region and Availability Zones

**Benefits of multi-region deployment:**
- Users physically closer to system → lower latency
- Spread traffic across population density (e.g., Japan/Korea in separate regions)
- **Privacy law compliance** — some countries require local data storage; use DNS routing to restrict requests to local region

---

#### Filtering Results

- When geohash/quadtree divides into small grids, result set is small
- Strategy: Return business IDs first → hydrate business objects → **filter by opening time or business type** in application layer
- Assumes opening time and business type are stored in business table

---

#### Final Architecture Flow — "Get Nearby Businesses"

1. Client sends location (lat=37.776720, long=-122.416730) + radius (500m) to load balancer
2. Load balancer → LBS
3. LBS determines geohash length (500m → length 6)
4. LBS calculates neighboring geohashes → `[my_geohash, neighbor1..neighbor8]`
5. For each geohash, LBS calls "Geohash" Redis to fetch business IDs (**in parallel**)
6. LBS fetches hydrated business objects from "Business info" Redis → calculates distances → ranks → returns to client

**Business CRUD flow:**
- Business service checks "Business info" Redis cache first
- Cache hit → return; cache miss → fetch from DB → store in Redis
- Updates via nightly job (next-day agreement)

---

## 🔹 Chapter 2 — Nearby Friends {#ch2}

### Key Difference from Proximity Service

> **Proximity Service:** Business locations are **static**
> **Nearby Friends:** User locations are **dynamic** — change frequently, requiring real-time updates

---

### Requirements & Scope

**Functional Requirements:**
- View nearby friends on mobile app (each entry shows distance + last-updated timestamp)
- Nearby friend list updates every few seconds

**Non-Functional Requirements:**
- **Low latency** — receive friend location updates without much delay
- **Reliability** — overall reliable, occasional data point loss acceptable
- **Eventual consistency** — few seconds delay across replicas is fine

**Key Assumptions:**
- "Nearby" = within 5-mile radius (configurable)
- 1 billion total users, 10% use nearby friends feature → **100M DAU**
- Concurrent users = 10% of DAU → **10 million**
- Average 400 friends per user
- Location refresh interval: **30 seconds** (human walking speed ~3-4 mph → 30s distance change is insignificant)
- Inactive >10 minutes → disappear from list
- 20 nearby friends per page

**Back-of-Envelope:**

- Location update QPS = 10M / 30 = **~334,000 QPS**
- Location updates to forward per second: 334K × 400 friends × 10% online&nearby = **~14 million/sec**

---

### High-Level Design

**Why not peer-to-peer?**
- Conceptually each user could maintain persistent connections to every active nearby friend
- Not practical for mobile: flaky connections, tight power budget

**Shared backend responsibilities:**
- Receive location updates from all active users
- For each update, find all active friends who should receive it → forward to their devices
- If distance exceeds threshold → don't forward

**Core Components:**

| Component | Role |
|-----------|------|
| **Load Balancer** | Distributes traffic across WebSocket + API servers |
| **RESTful API Servers** | Stateless HTTP servers for auxiliary tasks (add/remove friends, profile updates, auth) |
| **WebSocket Servers** | Stateful servers for real-time bi-directional location updates; each client maintains one persistent connection |
| **Redis Location Cache** | Stores most recent location per active user with TTL; expired = inactive |
| **User Database** | User profiles + friendship data (relational or NoSQL) |
| **Location History DB** | Historical location data (for ML, etc.); not directly for nearby friends feature |
| **Redis Pub/Sub** | Lightweight message bus routing location updates from one user to all online friends |

---

### Redis Pub/Sub Design

**How it works:**
- Each user has their own **channel** in Redis Pub/Sub
- When user updates location → published to their channel
- Each friend's WebSocket connection handler **subscribes** to the user's channel
- On receiving update → handler recomputes distance → if within radius, forwards to friend's client

**Why Redis Pub/Sub?**
- Channels are very cheap to create (hash table + linked list per channel)
- Modern Redis with GBs of memory can hold millions of channels
- Messages published to channel with no subscribers are simply dropped (minimal load)
- No CPU cycles used for idle channels

---

### Periodic Location Update Flow (detailed)

1. Mobile client sends location update via WebSocket to load balancer
2. Load balancer → persistent WebSocket connection on the server
3. WebSocket server saves to **location history database**
4. WebSocket server updates **location cache** (refreshes TTL) + saves in connection handler variable
5. WebSocket server publishes to user's channel in **Redis Pub/Sub**
   - Steps 3–5 can execute **in parallel**
6. Redis Pub/Sub broadcasts to all subscribers (friends' WebSocket handlers)
7. Each subscriber's WebSocket server computes distance (sender location from message vs. subscriber location from handler variable)
8. If distance ≤ search radius → send location + timestamp to subscriber's client; otherwise drop

**Scale numbers per update:** ~400 friends average, ~10% online+nearby → **~40 location forwards per user update**

---

### API Design

**WebSocket APIs:**

| API | Request | Response |
|-----|---------|----------|
| Periodic location update | lat, long, timestamp | Nothing |
| Receive location updates | — | Friend location + timestamp |
| WebSocket initialization | lat, long, timestamp | All nearby friends' locations |
| Subscribe to new friend | friend_id | Friend's latest location + timestamp |
| Unsubscribe a friend | friend_id | Nothing |

**HTTP APIs:** Standard CRUD for friends, profiles (handled by RESTful API servers)

---

### Data Model

**Location Cache (Redis):**

| Key | Value |
|-----|-------|
| user_id | {latitude, longitude, timestamp} |

- Only stores **current** location — one entry per user
- TTL-based auto-purge for inactive users
- If Redis goes down → replace with empty instance, let cache warm from new updates (acceptable: miss 1-2 update cycles)

**Location History Database:**

| user_id | latitude | longitude | timestamp |
|---------|----------|-----------|-----------|

- Heavy-write workload → **Cassandra** is a good candidate
- Or relational DB sharded by user_id

---

### Deep Dive: Scaling Each Component

#### API Servers
- Stateless → standard auto-scaling by CPU/load/IO

#### WebSocket Servers
- **Stateful** — care needed when removing nodes
- Mark as "draining" at load balancer → no new connections routed
- Wait for existing connections to close (or reasonable timeout) → then remove
- Same care needed for software releases
- Cloud load balancers handle this well

#### Client Initialization Flow
1. Update user's location in cache
2. Save location in connection handler variable
3. Load all friends from user database
4. **Batched request** to location cache for all friends' locations
   - Inactive friends won't be in cache (TTL expired)
5. For each returned location, compute distance → if within radius, send to client
6. Subscribe to **every friend's** channel in Redis Pub/Sub (active AND inactive)
   - Simplifies design: no need to subscribe/unsubscribe when friends go online/offline
   - Inactive friends use small memory but no CPU/IO
7. Publish user's location to user's own channel

#### User Database
- Shard by user_id — standard relational DB sharding
- At this scale, likely managed by dedicated team via internal API

#### Location Cache
- 10M active users × ≤100 bytes/location → fits in single Redis server memory-wise
- **BUT** 334K updates/sec is likely too high for one server
- **Solution:** Shard by user_id across multiple Redis servers (location data is independent per user)
- Replicate each shard to standby node for availability

#### Redis Pub/Sub Server Scaling

**Memory usage:**
- 100M channels (1B × 10%)
- ~100 active friends per user × 20 bytes/subscriber pointer = 200GB total
- Needs ~2 Redis servers (at 100GB each)

**CPU usage (the bottleneck):**
- ~14M subscriber pushes/second
- Conservative estimate: 100K pushes/sec per server
- Needs ~**140 Redis servers** (conservative, likely fewer needed)

> **Key insight:** CPU is the bottleneck, not memory. Need a **distributed Redis Pub/Sub cluster**.

---

### Distributed Redis Pub/Sub Cluster

**Sharding approach:** Channels are independent → shard by publisher's user_id using **consistent hashing**

**Service Discovery (etcd / ZooKeeper):**
- Stores hash ring config: `Key: /config/pub_sub_ring, Value: ["p_1", "p_2", "p_3", "p_4"]`
- WebSocket servers subscribe to hash ring updates
- Each WebSocket server caches hash ring locally for efficiency

**How publishing works:**
1. WebSocket server consults hash ring → determines correct Redis Pub/Sub server
2. Publishes location update to user's channel on that server

**Scaling considerations — treat as stateful cluster:**
- Messages on channels are NOT persisted (stateless in that sense)
- BUT subscriber lists per channel ARE state → if channel moves, all subscribers must re-subscribe
- **Over-provision** cluster to handle daily peaks with headroom
- Avoid unnecessary resizing

**When resizing is needed:**
- Mass resubscription events → some location updates may be missed
- Minimize by resizing during **lowest usage period**
- Steps: determine new ring size → provision servers → update hash ring keys → monitor dashboard

**When a Pub/Sub server dies:**
- Only channels on that server are affected (much lower risk than full resize)
- Operator updates hash ring to replace dead node with standby
- WebSocket servers notified → check each subscribed channel against new hash ring → re-subscribe affected channels

---

### Adding/Removing Friends

- Client registers callback for friend add/remove events
- On add: sends message to WebSocket server → subscribe to new friend's channel
- On remove: sends message → unsubscribe from friend's channel
- Same mechanism for friend opt-in/opt-out of location sharing

---

### Users with Many Friends

- Hard cap on friends (e.g., Facebook: 5,000) — bidirectional friendships, not follower model
- Subscribers scattered across many WebSocket servers → update load is spread
- "Whale" users spread across 100+ Pub/Sub servers → incremental load manageable

---

### Nearby Random Person (Extra Credit)

**Design extension:** Show random people (not just friends) who opted-in to location sharing

**Approach:** Pool of Pub/Sub channels by **geohash**
- Each geohash grid has its own channel
- Users within a grid subscribe to same channel
- When user updates location → WebSocket handler computes geohash → publishes to that geohash's channel
- All subscribers in the grid (except sender) receive the update

**Border handling:** Each client subscribes to **9 geohash grids** (the user's grid + 8 surrounding grids)

---

### Alternative to Redis Pub/Sub: Erlang

**Why Erlang is a strong alternative:**
- Built for highly distributed, concurrent applications
- **Lightweight processes:** ~300 bytes each, millions per server, no CPU when idle
- Model each of 10M active users as individual Erlang process
- Subscription is native in Erlang/OTP
- Easy to distribute across servers; strong deployment + debugging tools

**How it would work:**
- WebSocket service + routing layer both implemented in Erlang
- Each user = Erlang process
- User process receives location updates from WebSocket server
- Subscribes to friend Erlang processes → forms mesh of connections for efficient routing

**Tradeoff:** Erlang is niche → hard to hire; but if team has expertise, it's arguably better than Redis Pub/Sub

---

## 🆚 Comparisons {#comparisons}

### Proximity Service vs Nearby Friends

| Aspect | Proximity Service | Nearby Friends |
|--------|-------------------|----------------|
| Data nature | Static (business addresses) | Dynamic (user locations change constantly) |
| Communication | HTTP request/response | WebSocket (persistent, bi-directional) |
| Update frequency | Next-day updates | Every 30 seconds |
| Indexing | Geohash/Quadtree/S2 | Redis Pub/Sub channels |
| Scale challenge | Search QPS (5K) | Location update forwarding (14M/sec) |
| Consistency | Eventual (fine) | Eventual (fine) |
| State | Stateless LBS | Stateful WebSocket servers |

---

## 📖 Key Definitions {#definitions}

| Term | Meaning | Context |
|------|---------|---------|
| **Geohash** | Algorithm encoding lat/long into 1D base32 string by recursively halving coordinate ranges | Used for spatial indexing; length determines grid precision |
| **Quadtree** | In-memory tree structure recursively dividing 2D space into 4 quadrants | Built at server startup; adapts grid size to data density |
| **Google S2** | Geometry library mapping sphere to 1D via Hilbert curve | Used by Google Maps, Tinder; supports geofencing |
| **Hilbert Curve** | Space-filling curve where nearby points in 2D stay close in 1D | Basis for S2's efficient spatial search |
| **Geofence** | Virtual perimeter for a real-world geographic area | Can be dynamic (radius) or predefined (boundaries) |
| **LBS** | Location-Based Service — finds nearby entities given location + radius | Core service in proximity system |
| **Redis Pub/Sub** | Lightweight message bus with channels; publishers send, subscribers receive | Used for routing location updates to friends |
| **TTL** | Time To Live — auto-expiry mechanism on cache entries | Used to purge inactive users from location cache |
| **WebSocket** | Protocol for persistent, bi-directional client-server communication | Used instead of HTTP for real-time location updates |
| **Service Discovery** | Component (etcd/ZooKeeper) storing configuration; clients subscribe to updates | Manages hash ring for distributed Pub/Sub cluster |

---

## 📐 Formulas & Key Calculations

```
Search QPS = (DAU × queries_per_day) / seconds_per_day
           = (100M × 5) / 10^5 = 5,000

Location Update QPS = concurrent_users / refresh_interval
                    = 10M / 30 = ~334,000

Location Forwards/sec = update_QPS × avg_friends × pct_online_nearby
                      = 334K × 400 × 10% = ~14 million

Quadtree Memory = (leaf_nodes × 832B) + (internal_nodes × 64B)
                = (2M × 832) + (0.67M × 64) ≈ 1.71 GB

Redis Pub/Sub Memory = channels × subscribers × pointer_size
                     = 100M × 100 × 20B = 200 GB

Redis Pub/Sub Servers (CPU) = forwards_per_sec / pushes_per_server
                            = 14M / 100K ≈ 140 servers
```

---

## ⚡ Quick-Recall Points

- **10⁵ seconds per day** — used throughout the book for estimation
- Geohash lengths **4–6** are the useful range for proximity search
- Two close locations can have **zero shared geohash prefix** (boundary issue)
- Quadtree memory for 200M businesses ≈ **1.71 GB** — fits on one server
- Quadtree is **in-memory, not a database** — built at server startup
- Geohash index scaling: prefer **read replicas over sharding** (data is small)
- Location coordinates are **bad cache keys** (inaccurate, change slightly)
- Redis Pub/Sub bottleneck is **CPU, not memory**
- Treat Redis Pub/Sub cluster as **stateful** — careful scaling needed
- Nearby friends uses **WebSocket** (not HTTP) for bi-directional real-time updates
- Each user gets their own Pub/Sub **channel**; friends subscribe to it
- Subscribe to ALL friends (active + inactive) to simplify design — inactive ones use tiny memory, no CPU
- **Erlang** process = ~300 bytes — can model each of 10M users as individual process

---

## ⚠️ Mistakes & Misconceptions

- ❌ Sharding the geospatial index table immediately → ✅ Check if data fits in working set first; use read replicas instead
  - 💬 Engineers default to sharding in interviews, but it adds unnecessary complexity when data is small

- ❌ Using raw lat/long as cache keys → ✅ Use geohash as cache key
  - 💬 Phone GPS is imprecise; slight movement changes coordinates but shouldn't change results

- ❌ Treating Redis Pub/Sub cluster as stateless → ✅ It's stateful (subscriber lists are state)
  - 💬 Messages are stateless, but channel-subscriber mappings require coordination when servers change

- ❌ Scaling Pub/Sub cluster up/down daily like stateless servers → ✅ Over-provision and resize carefully during low-traffic periods
  - 💬 Resizing triggers mass resubscription → potential missed updates

---

## 🔗 Relationships & Dependencies

- **Geohash length** determines **grid size** which determines **search coverage** for a given radius
- **Quadtree subdivision depth** adapts to **business density** → smaller grids in dense areas, larger in sparse
- **Redis Pub/Sub channel count** scales with **total users using feature** (not just concurrent)
- **WebSocket server count** scales with **concurrent users** (each user = 1 persistent connection)
- **Location cache sharding** is independent per user → easy to distribute by user_id
- **Service discovery** (etcd/ZooKeeper) is the **source of truth** for the Pub/Sub hash ring → WebSocket servers cache it locally
- **Nearby friends** builds on concepts from **proximity service** (geohash) but adds real-time messaging layer

---

## 🌳 Concept Trees

### Proximity Service

```
Proximity Service
├── Requirements (Functional + Non-Functional)
├── API Design (Search + Business CRUD)
├── Data Model (Business table + Geo index table)
├── High-Level Design
│   ├── LBS (stateless, read-heavy)
│   ├── Business Service (CRUD)
│   └── DB Cluster (primary-secondary)
├── Geospatial Indexing
│   ├── 2D Search (naive, inefficient)
│   ├── Even Grid (uneven distribution)
│   ├── Geohash ⭐
│   │   ├── Recursive binary subdivision
│   │   ├── Base32 encoding
│   │   ├── Precision levels 4-6
│   │   └── Boundary issues (2 types)
│   ├── Quadtree ⭐
│   │   ├── In-memory tree structure
│   │   ├── Density-adaptive grids
│   │   ├── ~1.71 GB for 200M businesses
│   │   └── Operational: incremental rebuild
│   └── Google S2
│       ├── Hilbert curve mapping
│       ├── Geofencing support
│       └── Region Cover algorithm
└── Deep Dive
    ├── Scale DB (shard business table, replicate geo index)
    ├── Caching (geohash + business info in Redis)
    ├── Multi-region deployment
    └── Filtering (by time/type in application layer)
```

### Nearby Friends

```
Nearby Friends
├── Requirements (real-time, 5-mile radius, 30s refresh)
├── High-Level Design
│   ├── WebSocket Servers (stateful, bi-directional)
│   ├── RESTful API Servers (stateless, auxiliary)
│   ├── Redis Location Cache (TTL-based)
│   ├── Redis Pub/Sub (per-user channels)
│   ├── Location History DB (Cassandra)
│   └── User DB (profiles + friendships)
├── Periodic Location Update Flow (8 steps)
├── API Design (5 WebSocket APIs)
├── Data Model (cache + history)
└── Deep Dive: Scaling
    ├── WebSocket Servers (drain before remove)
    ├── Client Initialization (7-step process)
    ├── Location Cache (shard by user_id)
    ├── Redis Pub/Sub Cluster
    │   ├── CPU is bottleneck (~140 servers)
    │   ├── Consistent hashing + service discovery
    │   ├── Stateful: careful scaling
    │   └── Server replacement procedure
    ├── Adding/removing friends (callbacks)
    ├── Users with many friends (distributed load)
    ├── Nearby random person (geohash channels)
    └── Alternative: Erlang (lightweight processes)
```

---

## 🎯 Actionable Takeaways

1. **Always clarify scope first** — narrow down features, ask about radius, real-time requirements, data freshness
2. **Choose geohash or quadtree** for interview — S2 is too complex to explain well
3. **Check data size before sharding** — if data fits in working set, read replicas are simpler and sufficient
4. **Geohash/quadtree as cache key** > raw coordinates — handles GPS imprecision and minor movement
5. **For real-time features, use WebSocket** (not HTTP polling) for bi-directional communication
6. **Redis Pub/Sub is excellent for lightweight fan-out** — cheap channels, but treat cluster as stateful
7. **Over-provision stateful clusters** — resizing has operational overhead and risks missed updates
8. **Subscribe to all friends' channels** (active + inactive) — simplifies design; idle channels use minimal resources
9. **Use service discovery** (etcd/ZooKeeper) for managing distributed Pub/Sub hash ring
10. **Deploy across regions** for latency, availability, and privacy law compliance

---

## 📝 Summary {#summary}

- **Proximity Service** designs a Yelp-like nearby business search using **geospatial indexing** (geohash preferred for simplicity) with a read-heavy, stateless LBS backed by MySQL with replicas and Redis cache (~5GB for 200M businesses across 3 precisions)
- Five indexing approaches discussed: naive 2D search, even grid, **geohash** (hash-based, 1D string), **quadtree** (in-memory density-adaptive tree, ~1.71GB), and **Google S2** (Hilbert curve, geofencing)
- Geohash boundary issues require fetching from current grid + 8 neighbors; expanding search removes trailing geohash digits
- **Nearby Friends** adds real-time complexity — **WebSocket** for persistent connections, **Redis Pub/Sub** as routing layer with per-user channels, and **Redis location cache** with TTL for active user tracking
- At 10M concurrent users: ~334K location updates/sec generating ~14M forwards/sec → requires distributed Redis Pub/Sub cluster (~140 servers) managed via **consistent hashing** and **service discovery**
- Key architectural distinction: Proximity Service handles **static** location data with read-heavy patterns; Nearby Friends handles **dynamic** locations with real-time write-heavy pub/sub patterns
- Both designs emphasize **multi-region deployment**, **horizontal scaling** of stateless components, and careful treatment of **stateful components** (quadtree rebuild, Pub/Sub cluster resizing)
- **Erlang** is presented as a superior alternative to Redis Pub/Sub for teams with the expertise, thanks to lightweight processes (~300 bytes each) and native distributed messaging
