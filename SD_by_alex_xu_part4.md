

# 📌 Design Google Drive

A cloud file storage and synchronization service supporting upload, download, sync, versioning, sharing, and notifications across web and mobile clients. | Type: **System Design** | Depth: **Advanced**

---

## 📑 Table of Contents

1. [Requirements & Scope](#-step-1--requirements--scope)
2. [Back-of-the-Envelope Estimation](#-back-of-the-envelope-estimation)
3. [High-Level Design — Single Server to Scale](#-step-2--high-level-design)
4. [APIs](#-apis)
5. [Scaling Beyond a Single Server](#-move-away-from-single-server)
6. [Sync Conflicts](#-sync-conflicts)
7. [High-Level Architecture Components](#-high-level-architecture-components)
8. [Deep Dive — Block Servers](#-block-servers-deep-dive)
9. [Deep Dive — Strong Consistency](#-high-consistency-requirement)
10. [Deep Dive — Metadata Database Schema](#-metadata-database-schema)
11. [Deep Dive — Upload Flow](#-upload-flow)
12. [Deep Dive — Download Flow](#-download-flow)
13. [Deep Dive — Notification Service](#-notification-service)
14. [Deep Dive — Save Storage Space](#-save-storage-space)
15. [Deep Dive — Failure Handling](#-failure-handling)
16. [Wrap-Up & Alternative Designs](#-wrap-up--alternative-designs)
17. [Concept Tree](#-concept-tree)
18. [Key Definitions](#-key-definitions)
19. [Formulas & Estimations](#-formulas--estimations)
20. [Comparisons](#-comparisons)
21. [Quick-Recall Points](#-quick-recall-points)
22. [Summary](#-summary)

---

## 🔹 Step 1 — Requirements & Scope

### Functional Requirements (In Scope)

- **Add files** — drag-and-drop upload to Google Drive
- **Download files**
- **Sync files across multiple devices** — file added on one device auto-syncs to others
- **See file revisions** — version history
- **Share files** with friends, family, coworkers
- **Notifications** — sent when a file is edited, deleted, or shared with the user

### Out of Scope

- **Google Docs real-time collaborative editing** — multiple people editing the same document simultaneously is explicitly excluded

### Clarified Constraints

| Question | Answer |
|----------|--------|
| Platforms | Both mobile app and web app |
| File formats | Any file type |
| Encryption | Yes, files must be encrypted at rest |
| File size limit | ≤ 10 GB per file |
| Users | 10 million DAU |

### Non-Functional Requirements

- **Reliability** — data loss is unacceptable; this is the most critical requirement for a storage system
- **Fast sync speed** — slow sync → user abandonment
- **Bandwidth efficiency** — minimize unnecessary network traffic, especially important for mobile data users
- **Scalability** — handle high traffic volumes
- **High availability** — system usable even when some servers are offline, slow, or experiencing network errors

---

## 📐 Back-of-the-Envelope Estimation

| Metric | Calculation | Result |
|--------|-------------|--------|
| Signed-up users | Given | 50 million |
| DAU | Given | 10 million |
| Free space per user | Given | 10 GB |
| **Total storage allocated** | 50M × 10 GB | **500 Petabytes** |
| Uploads per user per day | Given | 2 files |
| Average file size | Given | 500 KB |
| Read:Write ratio | Given | 1:1 |
| **Upload QPS** | 10M × 2 / 86,400 | **~240 QPS** |
| **Peak upload QPS** | 240 × 2 | **~480 QPS** |

---

## 🔹 Step 2 — High-Level Design

### Starting Simple: Single-Server Setup

- **Web server** — Apache; handles upload and download
- **Database** — MySQL; stores metadata (user data, login info, file info)
- **File storage** — local directory `drive/` with 1 TB allocated
  - Under `drive/`, each user has a **namespace** (directory)
  - Each namespace holds all uploaded files for that user
  - Filename on server = original filename
  - Unique file identification = **namespace + relative path**

```
drive/
├── user1/
│   └── recipes/
│       └── chicken_soup.txt
├── user2/
│   ├── football.mov
│   └── sports.txt
└── user3/
    └── best_pic_ever.png
```

---

## 🔹 APIs

Three primary APIs needed, all requiring **user authentication** via **HTTPS** (SSL protects data in transit).

### 1. Upload a File

Two upload types supported:

| Type | When to Use |
|------|-------------|
| **Simple upload** | File size is small |
| **Resumable upload** | File size is large OR high chance of network interruption |

**Resumable upload API:**

```
POST https://api.example.com/files/upload?uploadType=resumable
```

**Params:**
- `uploadType=resumable`
- `data`: local file to be uploaded

**Resumable upload steps:**
1. Send initial request → retrieve the **resumable URL**
2. Upload data and **monitor upload state**
3. If upload is disturbed → **resume** the upload from where it stopped

### 2. Download a File

```
GET https://api.example.com/files/download
```

**Params:**
- `path`: download file path

```json
{
  "path": "/recipes/soup/best_soup.txt"
}
```

### 3. Get File Revisions

```
GET https://api.example.com/files/list_revisions
```

**Params:**
- `path`: path to the file
- `limit`: maximum number of revisions to return

```json
{
  "path": "/recipes/soup/best_soup.txt",
  "limit": 20
}
```

---

## 🔹 Move Away from Single Server

### Problem
- Single server storage fills up (e.g., 10 MB free of 1 TB) → users can't upload

### Evolution Steps

1. **Shard data** across multiple storage servers (e.g., by `user_id % 4` across Server1–Server4)
   - Solves space issue but risk of data loss on server outage remains

2. **Adopt Amazon S3** for file storage
   - Used by Netflix, Airbnb, and other leading companies
   - S3 = object storage offering industry-leading scalability, availability, security, performance
   - Supports **same-region replication** and **cross-region replication**
     - **Region** = geographic area where AWS has data centers
     - **Bucket** = like a folder in file systems
   - Redundant files stored in multiple regions → guards against data loss + ensures availability

3. **Add a Load Balancer**
   - Distributes traffic evenly across web servers
   - If a web server goes down → redistributes traffic to healthy servers

4. **Scale Web Servers**
   - With load balancer in place, web servers can be added/removed elastically based on traffic

5. **Separate Metadata Database**
   - Move DB off the single server → avoid single point of failure
   - Set up **data replication** and **sharding** for availability and scalability

6. **Use Amazon S3 for File Storage**
   - Files replicated in **two separate geographical regions** for availability and durability

> **Key Principle:** Decouple web servers, metadata database, and file storage into independent components.

### Resulting Architecture (Figure 15-7)

```
User (Browser / Mobile)
        ↓
   Load Balancer
        ↓
   API Servers (scalable pool)
      ↙        ↘
Metadata DB    File Storage (S3)
```

---

## 🔹 Sync Conflicts

- **When:** Two users modify the same file/folder at the same time
- **Strategy:** First version processed wins; the later version receives a **conflict**
- **Conflict resolution for the losing user:**
  - System presents **both copies**: user's local copy AND the latest server version
  - User can choose to **merge both files** or **override one version** with the other
  - Example: conflicted copy is named like `SystemDesignInterview_user 2_conflicted_copy_2019-05-01`

> Real-time collaborative editing (like Google Docs) requires differential synchronization — out of scope for this design.

---

## 🔹 High-Level Architecture Components

| Component | Role |
|-----------|------|
| **User** | Accesses via browser or mobile app |
| **Block servers** | Split files into blocks, compress, encrypt, upload to cloud storage; handle delta sync |
| **Cloud storage** | Stores file blocks (S3) |
| **Cold storage** | Stores inactive/rarely accessed data (cheaper) |
| **Load balancer** | Distributes requests evenly among API servers |
| **API servers** | Handle everything except uploading flow: auth, user profiles, file metadata updates |
| **Metadata database** | Stores metadata for users, files, blocks, versions (NOT the files themselves) |
| **Metadata cache** | Caches frequently accessed metadata for fast retrieval |
| **Notification service** | Pub/sub system; notifies clients when files are added/edited/removed so they can pull latest changes |
| **Offline backup queue** | Stores pending change info for offline clients; synced when client comes back online |

### Data Flow Overview

- **Upload path:** Client → Block servers → Cloud storage; Client → API servers → Metadata DB
- **Notification path:** API servers → Notification service → Clients (via **long polling**)
- **Offline path:** Changes queued in **offline backup queue** → synced when client reconnects

---

## 🔹 Block Servers (Deep Dive)

### Why Block Servers?

- Sending the **whole file** on every update wastes bandwidth for large, frequently updated files
- Block servers handle the heavy lifting of file upload processing

### Two Key Optimizations

1. **Delta sync** — only **modified blocks** are synced, not the entire file
   - Uses sync algorithms (e.g., rsync)
   - Example: file has blocks 1–10; only blocks 2 and 5 changed → only blocks 2 and 5 uploaded
2. **Compression** — reduces data size per block
   - **Text files:** gzip, bzip2
   - **Images and videos:** different specialized compression algorithms

### Block Server Processing Pipeline

```
File → Split into blocks → Compress each block → Encrypt each block → Upload to Cloud Storage
```

- Each block has a **unique hash value** stored in the metadata database
- Each block is an **independent object** in S3
- To reconstruct a file → join blocks in correct order
- **Max block size: 4 MB** (Dropbox reference)

---

## 🔹 High Consistency Requirement

- **Strong consistency is mandatory** — a file must NOT be shown differently by different clients simultaneously
- Applies to both **metadata cache** and **database layers**

### Challenges with Caches

- Memory caches default to **eventual consistency** → different replicas may have different data
- To achieve strong consistency in cache:
  - Ensure data in **cache replicas and master** is consistent
  - **Invalidate caches on database write** → cache and DB always hold same value

### Why Relational Database?

- RDBMS natively supports **ACID** (Atomicity, Consistency, Isolation, Durability)
- NoSQL databases do **not** support ACID by default → would require programmatic ACID in sync logic
- **Design choice: relational database** for metadata because ACID is natively supported

---

## 🔹 Metadata Database Schema

Six tables in the simplified schema:

### User Table
| Column | Type |
|--------|------|
| `user_id` (PK) | bigint |
| `user_name` | varchar |
| `created_at` | timestamp |

### Device Table
| Column | Type |
|--------|------|
| `device_id` (PK) | uuid |
| `user_id` (FK) | bigint |
| `last_logged_in_at` | timestamp |

- A user can have **multiple devices**
- `push_id` used for sending/receiving mobile push notifications

### Workspace (Namespace) Table
| Column | Type |
|--------|------|
| `id` (PK) | bigint |
| `owner_id` (FK) | bigint |
| `is_shared` | boolean |
| `created_at` | timestamp |

- Root directory of a user

### File Table
| Column | Type |
|--------|------|
| `id` (PK) | bigint |
| `file_name` | varchar |
| `relative_path` | varchar |
| `is_directory` | boolean |
| `latest_version` | bigint |
| `checksum` | bigint |
| `workspace_id` (FK) | bigint |
| `created_at` | timestamp |
| `last_modified` | timestamp |

- Stores everything related to the **latest file**

### File_Version Table
| Column | Type |
|--------|------|
| `id` (PK) | bigint |
| `file_id` (FK) | bigint |
| `device_id` (FK) | uuid |
| `version_number` | bigint |
| `last_modified` | timestamp |

- Stores **version history**
- Existing rows are **read-only** to preserve revision history integrity

### Block Table
| Column | Type |
|--------|------|
| `block_id` (PK) | bigint |
| `file_version_id` (FK) | bigint |
| `block_order` | int |

- A file of any version can be reconstructed by **joining all blocks in correct order**

---

## 🔹 Upload Flow

Two parallel requests originate from the uploading client:

### Request A: Add File Metadata

1. Client 1 sends request to add new file metadata
2. Metadata DB stores new file metadata; file upload status set to **"pending"**
3. Notification service is notified that a new file is being added
4. Notification service informs relevant clients (e.g., Client 2) that a file is being uploaded

### Request B: Upload File to Cloud Storage

1. (2.1) Client 1 uploads file content to **block servers**
2. (2.2) Block servers **chunk → compress → encrypt** blocks, upload to cloud storage
3. (2.3) Cloud storage triggers **upload completion callback** → sent to API servers
4. (2.4) File status changed to **"uploaded"** in Metadata DB
5. (2.5) Notification service notified that file status is now "uploaded"
6. (2.6) Notification service informs relevant clients (Client 2) that file is fully uploaded

> File edit flow follows the same pattern.

---

## 🔹 Download Flow

**Triggered when:** a file is added or edited elsewhere

### How Does a Client Know About Changes?

| Scenario | Mechanism |
|----------|-----------|
| Client is **online** | Notification service informs client immediately to pull latest data |
| Client is **offline** | Changes saved to offline backup queue; client pulls changes when back online |

### Download Sequence (9 steps)

1. Notification service informs Client 2 that a file changed
2. Client 2 sends request to **fetch metadata** from API servers
3. API servers query **Metadata DB** for change metadata
4. Metadata returned to API servers
5. Client 2 receives the metadata
6. Client 2 sends request to **block servers** to download blocks
7. Block servers download blocks from **cloud storage**
8. Cloud storage returns blocks to block servers
9. Client 2 **downloads all new blocks** and reconstructs the file

---

## 🔹 Notification Service

### Purpose
- Inform other clients about local file mutations to reduce sync conflicts
- Pub/sub model: data transferred to clients as events happen

### Options Considered

| Option | Description |
|--------|-------------|
| **Long polling** | Client opens connection; server holds it until data is available or timeout. Dropbox uses this. |
| **WebSocket** | Persistent bidirectional connection between client and server |

### Decision: Long Polling

**Reasons:**
- Notification communication is **not bidirectional** — server sends file change info to client, not vice versa
- WebSocket is suited for **real-time bi-directional** communication (e.g., chat apps)
- Google Drive notifications are sent **infrequently** with **no burst of data**

### How Long Polling Works Here

1. Each client establishes a **long poll connection** to notification service
2. When changes to a file are detected → client **closes** the long poll connection
3. Closing connection = signal for client to connect to metadata server and **download latest changes**
4. After receiving response (or connection timeout) → client **immediately opens a new** long poll connection

---

## 🔹 Save Storage Space

### Problem
- Multiple versions across multiple data centers → storage fills up quickly

### Three Techniques to Reduce Storage Costs

1. **De-duplicate data blocks**
   - Eliminate redundant blocks at the **account level**
   - Two blocks are identical if they have the **same hash value**

2. **Intelligent data backup strategy**
   - **Set a version limit:** when limit is reached, oldest version replaced by newest
   - **Keep valuable versions only:** heavily edited files could produce 1000+ versions in a short period
     - Limit number of saved versions
     - **Weight recent versions more heavily** than older ones
     - Experimentation needed to find optimal version count

3. **Move infrequently used data to cold storage**
   - **Cold data** = data not accessed for months or years
   - **Amazon S3 Glacier** is much cheaper than standard S3

---

## 🔹 Failure Handling

| Component | Failure Strategy |
|-----------|-----------------|
| **Load balancer** | Secondary LB becomes active; LBs monitor each other via **heartbeat** (periodic signal); missed heartbeat = failed LB |
| **Block server** | Other servers pick up unfinished/pending jobs |
| **Cloud storage** | S3 buckets replicated across **multiple regions**; fetch from alternate region if unavailable |
| **API server** | **Stateless service**; LB redirects traffic to other API servers |
| **Metadata cache** | Cache servers replicated; if one node fails, access others; spin up replacement server |
| **Metadata DB — Master** | Promote a **slave to master**; bring up a new slave node |
| **Metadata DB — Slave** | Use another slave for reads; bring up replacement DB server |
| **Notification service** | Each server holds ~**1 million connections** (Dropbox 2012 data); if server goes down, all long poll connections lost; clients must reconnect to different server; **reconnection is slow** — cannot reconnect all at once |
| **Offline backup queue** | Queues replicated; if one fails, consumers re-subscribe to backup queue |

---

## 🔹 Wrap-Up & Alternative Designs

### Design Summary
- Two core flows: **manage file metadata** + **file sync**
- Key qualities: **strong consistency**, **low network bandwidth**, **fast sync**
- Notification service via **long polling** keeps clients updated

### Alternative: Direct Client-to-Cloud Upload (Skip Block Servers)

| Aspect | Advantage | Disadvantage |
|--------|-----------|--------------|
| Speed | Faster — file transferred only once to cloud | — |
| Engineering effort | — | Chunking, compression, encryption must be implemented on **every platform** (iOS, Android, Web) — error-prone |
| Security | — | Client-side encryption is not ideal — clients can be **hacked or manipulated** |

> Current design centralizes chunking/compression/encryption logic in **block servers**, which is safer and more maintainable.

### Future Evolution
- **Presence service** — extract online/offline logic from notification servers into a separate service
  - Enables easy integration of presence functionality by other services

---

## 🌳 Concept Tree

```
Design Google Drive
├── Requirements
│   ├── Functional: upload, download, sync, revisions, sharing, notifications
│   ├── Non-functional: reliability, fast sync, bandwidth efficiency, scalability, HA
│   └── Constraints: 10M DAU, 10GB/file limit, encrypted at rest, any file type
├── Estimation
│   ├── 500 PB total storage
│   ├── ~240 QPS upload (480 peak)
│   └── 1:1 read/write ratio
├── High-Level Design
│   ├── Single server → sharding → S3 → decoupled architecture
│   ├── APIs: upload (simple/resumable), download, list_revisions
│   ├── Sync conflict resolution: first-write-wins + present both copies
│   └── Components: block servers, cloud storage, cold storage, LB, API servers,
│       metadata DB/cache, notification service, offline backup queue
├── Deep Dive
│   ├── Block Servers
│   │   ├── Delta sync (only modified blocks)
│   │   ├── Compression (gzip/bzip2 for text)
│   │   ├── Encryption before storage
│   │   └── Max block size: 4 MB
│   ├── Strong Consistency
│   │   ├── RDBMS chosen (native ACID)
│   │   └── Cache invalidation on DB write
│   ├── Metadata DB Schema (6 tables)
│   │   ├── user, device, workspace, file, file_version, block
│   │   └── file_version rows are read-only
│   ├── Upload Flow (parallel metadata + block upload)
│   ├── Download Flow (9-step notification → metadata → blocks)
│   ├── Notification Service
│   │   ├── Long polling chosen over WebSocket
│   │   └── Reason: unidirectional, infrequent notifications
│   ├── Storage Optimization
│   │   ├── Block deduplication (hash comparison)
│   │   ├── Version limit + weight recent versions
│   │   └── Cold storage (S3 Glacier)
│   └── Failure Handling (7 component types covered)
└── Wrap-Up
    ├── Alternative: direct client-to-cloud upload (faster but less secure)
    └── Future: extract presence service from notification servers
```

---

## 📖 Key Definitions

| Term | Meaning | Context |
|------|---------|---------|
| **Block server** | Server that splits files into blocks, compresses, encrypts, and uploads them to cloud storage | Core upload pipeline component |
| **Delta sync** | Only modified blocks are transferred instead of the whole file | Reduces bandwidth; key optimization |
| **Namespace** | Root directory for a user's files on the server | Maps to `workspace` table in DB |
| **Cold storage** | Storage system for inactive data not accessed for months/years | Amazon S3 Glacier; much cheaper than S3 |
| **Resumable upload** | Upload type that can resume from interruption point for large files | 3-step process: get URL → upload → resume if needed |
| **Long polling** | Client opens connection; server holds it until data available or timeout | Used by Dropbox; chosen for notification service |
| **Offline backup queue** | Queue storing pending changes for offline clients | Synced when client reconnects |
| **ACID** | Atomicity, Consistency, Isolation, Durability — transaction properties | Reason for choosing RDBMS over NoSQL |
| **Heartbeat** | Periodic signal between load balancers to monitor health | Missed heartbeat = failed LB |

---

## 📐 Formulas & Estimations

```
Total storage = num_users × space_per_user
             = 50M × 10 GB = 500 PB

Upload QPS   = DAU × uploads_per_user / seconds_per_day
             = 10M × 2 / 86400 ≈ 240

Peak QPS     = QPS × 2 = 480
```

---

## 🆚 Comparisons

### Long Polling vs. WebSocket for Notifications

| Aspect | Long Polling | WebSocket |
|--------|-------------|-----------|
| Direction | Server → Client (unidirectional sufficient) | Bidirectional |
| Best for | Infrequent, non-bursty updates | Real-time chat, gaming |
| Chosen? | **Yes** | No |
| Reason | Notifications are infrequent, unidirectional | Overkill for this use case |

### Simple Upload vs. Resumable Upload

| Aspect | Simple | Resumable |
|--------|--------|-----------|
| File size | Small files | Large files |
| Network interruption handling | None (restart) | Resume from interruption point |
| Complexity | Low | Higher (3-step process) |

### RDBMS vs. NoSQL for Metadata

| Aspect | RDBMS | NoSQL |
|--------|-------|-------|
| ACID support | **Native** | Must be programmatically added |
| Strong consistency | Easy to achieve | Harder to guarantee |
| Chosen? | **Yes** | No |

### Block Server Upload vs. Direct Client-to-Cloud Upload

| Aspect | Via Block Servers (chosen) | Direct to Cloud |
|--------|--------------------------|-----------------|
| Upload speed | Slower (two hops) | Faster (one hop) |
| Code duplication | Centralized logic | Must implement on iOS, Android, Web |
| Security | Encryption on server side (safer) | Client-side encryption (hackable) |
| Maintainability | Higher | Lower |

---

## ⚡ Quick-Recall Points

- **Max block size: 4 MB** (Dropbox reference)
- **500 PB** total storage for 50M users × 10 GB each
- **~240 QPS** upload; **~480 peak**
- **RDBMS** chosen for metadata → native ACID → strong consistency
- **Long polling** for notifications (not WebSocket) — because unidirectional + infrequent
- **Delta sync** = only modified blocks uploaded, not the whole file
- **File_version rows are read-only** to preserve revision history integrity
- **Cache invalidation on DB write** to maintain strong consistency
- S3 Glacier for **cold storage** is much cheaper than standard S3
- Notification server can hold **~1M connections per machine** (Dropbox 2012)
- Upload flow sends **two parallel requests**: metadata update + block upload
- Sync conflict resolution: **first-write-wins**; loser gets both copies to merge/override
- Files stored as blocks in S3; metadata DB stores **only metadata, not file content**
- Each block has a **unique hash value** in metadata DB
- A user can have **multiple devices** (device table)

---

## ⚠️ Mistakes & Misconceptions

- ❌ "Store files and metadata on the same server" → ✅ Decouple them; files in S3, metadata in RDBMS → 💬 Single server = single point of failure + space limits
- ❌ "Upload the entire file on every edit" → ✅ Use delta sync (only modified blocks) → 💬 Uploading full file wastes bandwidth, especially for large frequently-edited files
- ❌ "Use NoSQL for metadata because it scales better" → ✅ Use RDBMS for native ACID/strong consistency → 💬 File consistency is critical; NoSQL requires extra programmatic effort
- ❌ "Use WebSocket for notifications" → ✅ Long polling suffices → 💬 Notifications are server→client only and infrequent; WebSocket is overkill
- ❌ "Implement encryption on the client side" → ✅ Encrypt in block servers → 💬 Clients can be hacked; centralized encryption is safer

---

## 🔗 Relationships & Dependencies

- **Block servers** depend on **cloud storage (S3)** for persisting encrypted blocks
- **Notification service** depends on **metadata DB** events (file status changes) to trigger notifications
- **Offline backup queue** acts as a buffer between **notification service** and **offline clients**
- **Metadata cache** must be **invalidated on every DB write** → strong consistency between cache and DB
- **File reconstruction** depends on **block table** having correct `block_order` for each `file_version_id`
- **Delta sync** depends on block-level hashing → compare hashes to identify changed blocks
- **Cold storage** is downstream of **cloud storage** — infrequently accessed data is migrated there to save cost

---

## 🎯 Actionable Takeaways

- **Start simple, scale incrementally** — single server → sharding → cloud storage → decoupled architecture (demonstrates design evolution in interviews)
- **Always separate storage from metadata** — different scaling characteristics, different failure modes
- **Use resumable uploads** for any file that could be large — essential for user experience on unreliable networks
- **Delta sync + compression + encryption** is the standard pipeline for block-level file sync systems
- **Choose database type based on consistency needs** — RDBMS when strong consistency is non-negotiable
- **Invalidate cache on write** when strong consistency required — don't rely on eventual consistency of cache replicas
- **Design for every failure mode** — have a strategy for each component (LB, DB master/slave, cache, notification server, queue)
- **Consider cold storage** for cost optimization — not all data is accessed equally
- **Long polling** is the pragmatic choice when notifications are unidirectional and infrequent

---

## 📝 Summary

- **Google Drive** is a file storage & sync service; design focuses on upload, download, multi-device sync, versioning, sharing, and notifications
- **Scale:** 50M users, 10M DAU, 500 PB storage, ~240 QPS uploads (480 peak)
- **Architecture evolves** from single server → sharded storage → Amazon S3 → fully decoupled (LB + API servers + Metadata DB + Cloud Storage)
- **Block servers** are central: they split files into 4 MB blocks, compress, encrypt, and upload; enable **delta sync** (only changed blocks transferred)
- **Strong consistency** is mandatory → RDBMS with ACID chosen for metadata; cache invalidated on every DB write
- **Notification service** uses **long polling** (not WebSocket) because communication is server-to-client only and infrequent
- **Sync conflicts** resolved by first-write-wins; losing user gets both copies to merge or override
- **Storage optimization** via block deduplication (hash comparison), version limits (weight recent versions), and cold storage (S3 Glacier)
- **Failure handling** covers all 7+ component types with specific strategies (heartbeat for LB, slave promotion for DB master failure, multi-region replication for cloud storage, etc.)
- **Alternative design:** direct client-to-cloud upload is faster but requires duplicating logic across platforms and is less secure
