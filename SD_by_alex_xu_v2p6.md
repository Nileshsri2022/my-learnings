

# 📌 Designing an Electronic Stock Exchange

One-line summary: System design of a low-latency, highly available electronic stock exchange covering matching engine, order book, event sourcing, and fault tolerance. | Type: System Design | Depth: Advanced

---

## 📑 Table of Contents

1. [Problem Scope & Requirements](#1-problem-scope--requirements)
2. [Business Knowledge 101](#2-business-knowledge-101)
3. [High-Level Design & Three Flows](#3-high-level-design--three-flows)
4. [Component Deep Dive (Trading Flow)](#4-component-deep-dive-trading-flow)
5. [API Design](#5-api-design)
6. [Data Models](#6-data-models)
7. [Performance & Low-Latency Design](#7-performance--low-latency-design)
8. [Event Sourcing](#8-event-sourcing)
9. [High Availability](#9-high-availability)
10. [Fault Tolerance](#10-fault-tolerance)
11. [Matching Algorithms](#11-matching-algorithms)
12. [Determinism](#12-determinism)
13. [Market Data Publisher Optimizations](#13-market-data-publisher-optimizations)
14. [Fairness, Multicast & Network Security](#14-fairness-multicast--network-security)

---

## 🌳 Concept Tree

```
Stock Exchange System
├── Step 1: Requirements
│   ├── Functional: place/cancel limit orders, real-time order book, risk checks, wallet
│   ├── Non-Functional: 99.99% availability, fault tolerance, ms-level latency, security
│   └── Estimation: 100 symbols, 1B orders/day, ~43K QPS, 215K peak QPS
├── Step 2: High-Level Design
│   ├── Three Flows
│   │   ├── Trading Flow (critical path) — gateway → order manager → sequencer → matching engine
│   │   ├── Market Data Flow — matching engine → MDP → data service → brokers
│   │   └── Reporting Flow — reporter → DB (settlement, compliance)
│   ├── Business Knowledge (broker, limit/market order, L1/L2/L3 data, candlestick, FIX)
│   ├── API Design (REST: order, execution, order book, candlestick)
│   └── Data Models (Product, Order, Execution, Order Book DS, Candlestick)
├── Step 3: Deep Dive
│   ├── Performance (single-server, mmap, application loops, CPU pinning)
│   ├── Event Sourcing (immutable event log, mmap event store, SBE encoding)
│   ├── High Availability (hot-warm matching engine, event store replication)
│   ├── Fault Tolerance (Raft consensus, RTO, RPO, chaos engineering)
│   ├── Matching Algorithms (FIFO, LMM, dark pool)
│   ├── Determinism (functional + latency; 99th percentile)
│   ├── MDP Optimizations (ring buffers, pre-allocated memory)
│   ├── Distribution Fairness (multicast via reliable UDP)
│   ├── Colocation (paid low-latency co-located servers)
│   └── Network Security (DDoS defenses, rate limiting, caching, URL hardening)
└── Step 4: Wrap Up
    └── Modern trend: single-server for big exchanges, cloud for crypto, AMM for DeFi
```

---

## 🔹 1. Problem Scope & Requirements

### Functional Requirements

- **Securities traded**: Stocks only (no options or futures)
- **Order operations**: Place new order, cancel order (no replace)
- **Order type**: **Limit order** only (no market or conditional orders)
- **Trading hours**: Normal hours only (no after-hours)
- **Core functions**:
  - Place new limit orders or cancel them
  - Receive matched trades in **real-time**
  - View real-time **order book** (list of buy/sell orders)
- **Scale**: Tens of thousands of concurrent users, ≥100 symbols, billions of orders/day
- **Risk checks**: Simple rules, e.g., max 1 million shares of Apple per user per day
- **Wallet management**: Sufficient funds required before order placement; funds **withheld** for pending orders to prevent overspending

### Non-Functional Requirements

| Requirement | Detail |
|---|---|
| **Availability** | ≥ 99.99% — even seconds of downtime harm reputation |
| **Fault tolerance** | Fast recovery mechanism to limit production incident impact |
| **Latency** | Millisecond-level round-trip; focus on **99th percentile** latency. Measured from market order entry to filled execution return |
| **Security** | Account management, **KYC** (Know Your Client) checks, DDoS prevention for public resources |

### Back-of-the-Envelope Estimation

- **100 symbols**
- **1 billion orders/day**
- NYSE hours: Mon–Fri, 9:30 AM – 4:00 PM ET = **6.5 hours**

```
QPS = 1,000,000,000 / (6.5 × 3,600) ≈ 43,000
Peak QPS = 5 × QPS ≈ 215,000
```

- Peak volume at market open (morning) and close (afternoon)

---

## 🔹 2. Business Knowledge 101

### Broker
- Intermediary between retail clients and exchange (e.g., Charles Schwab, Robinhood, E*Trade, Fidelity)
- Provides friendly UI for placing trades and viewing market data

### Institutional Client
- Trades in large volumes using **specialized trading software**
- **Pension funds**: Trade infrequently but in large volume; need **order splitting** to minimize market impact
- **Hedge funds** (market makers): Earn via commission rebates; require **very low latency** — cannot use web/mobile like retail clients

### Limit Order
- Buy or sell order with a **fixed price**
- May not find immediate match or may be **partially matched**

### Market Order
- No price specified — executed at **prevailing market price immediately**
- Sacrifices cost for guaranteed execution
- Useful in fast-moving market conditions

### Market Data Levels

| Level | Content |
|---|---|
| **L1** | Best bid price, best ask price, and quantities |
| **L2** | Multiple price levels with aggregated quantities per level |
| **L3** | All price levels + **queued individual order quantities** at each level |

- **Bid price** — highest price a buyer is willing to pay
- **Ask price** — lowest price a seller is willing to sell

### Candlestick Chart
- Represents stock price over a time interval
- Shows: **open, close, high, low** prices
- Common intervals: 1-min, 5-min, 1-hour, 1-day, 1-week, 1-month
- Anatomy: upper shadow, real body, lower shadow

### FIX Protocol
- **Financial Information eXchange** protocol, created in **1991**
- Vendor-neutral protocol for exchanging securities transaction information
- Example encoded message:

```
8=FIX.4.2 | 9=176 | 35=8 | 49=PHLX | 56=PERS |
52=20071123-05:30:00.000 | 11=ATOMNOCCC9990900 | 20=3 | 150=E | 39=E
| 55=MSFT | 167=CS | 54=1 | 38=15 | 40=2 | 44=15 | 58=PHLX EQUITY
TESTING | 59=0 | 47=C | 32=0 | 31=0 | 151=15 | 14=0 | 6=0 | 10=128 |
```

---

## 🔹 3. High-Level Design & Three Flows

The exchange has **three distinct flows** with different latency requirements:

### 3.1 Trading Flow (Critical Path — Strict Latency)

1. **Step 1**: Client places order via broker's web/mobile app
2. **Step 2**: Broker sends order to exchange
3. **Step 3**: Order enters via **Client Gateway** → input validation, rate limiting, authentication, normalization → forwards to Order Manager
4. **Steps 4–5**: **Order Manager** performs **risk checks** (rules from Risk Manager)
5. **Step 6**: Order Manager verifies **sufficient wallet funds**
6. **Steps 7–9**: Order sent to **Matching Engine**; on match → emits **two executions** (fills) — one for buy side, one for sell side. Both orders and executions are **sequenced** by the Sequencer
7. **Steps 10–14**: Executions returned to client via Client Gateway → Broker

### 3.2 Market Data Flow (Not on Critical Path)

1. **Step M1**: Matching Engine generates execution stream → sends to **Market Data Publisher** (MDP)
2. **Step M2**: MDP constructs **candlestick charts** and **order books** from execution/order stream → sends to Data Service
3. **Step M3**: Market data saved to specialized storage for real-time analytics. Brokers connect to Data Service → relay to clients

### 3.3 Reporting Flow (Not on Critical Path)

- **Steps R1–R2**: Reporter collects reporting fields (`client_id`, `price`, `quantity`, `order_type`, `filled_quantity`, `remaining_quantity`) from orders + executions → writes consolidated records to DB
- Provides: trading history, tax reporting, compliance reporting, settlements
- **Accuracy and compliance** are key factors (not latency)

> The trading flow (steps 1–14) is on the critical path; market data and reporting flows are not. Different latency requirements.

---

## 🔹 4. Component Deep Dive (Trading Flow)

### 4.1 Matching Engine (Cross Engine)

**Primary responsibilities:**
1. **Maintain order book** for each symbol (list of buy/sell orders)
2. **Match buy and sell orders** — produces two executions per match (buy side + sell side). Must be **fast and accurate**
3. **Distribute execution stream** as market data

> A highly available matching engine must produce matches in a **deterministic order**: given the same input sequence of orders, it must produce the same output sequence of executions on replay. This determinism is the foundation of high availability.

### 4.2 Sequencer

- **Key component** making the matching engine deterministic
- **Stamps every incoming order** with a sequence ID before matching engine processes it
- **Stamps every pair of outgoing executions** with sequence IDs
- Has **inbound** and **outbound** instances, each maintaining its own sequences
- Sequences must be **sequential numbers** → missing numbers easily detected

**Why sequence IDs?**
1. **Timeliness and fairness**
2. **Fast recovery / replay**
3. **Exactly-once guarantee**

**Sequencer as message queue:**
- Functions as both sequence generator AND message queue
- One queue for incoming orders → matching engine
- Another queue for executions → order manager
- Also serves as **event store** for orders and executions
- Similar to two Kafka streams, but Kafka's latency is too high and unpredictable for exchange use

### 4.3 Order Manager

**Inbound (from Client Gateway):**
- Sends order for **risk checks** (e.g., verify user's daily trade volume < $1M)
- Checks order against **user's wallet** for sufficient funds
- Sends order to **sequencer** → stamped with sequence ID → processed by matching engine
- Only sends **necessary attributes** to matching engine to minimize message size

**Outbound (from Matching Engine):**
- Receives executions via sequencer
- Returns executions for filled orders to brokers via client gateway

**Key characteristics:**
- Must be fast, efficient, and accurate
- Maintains **current states** for orders
- Managing state transitions is the **major source of complexity** — tens of thousands of cases in real systems
- **Event sourcing** is perfect for order manager design

### 4.4 Client Gateway

**Functions:**
- Authentication
- Input validation
- Rate limiting
- Normalization
- FIX/T protocol support

**Design principles:**
- On critical path → must be **lightweight**
- Pass orders to destinations **as quickly as possible**
- Trade-off: keep complex logic in matching engine and risk check, not gateway

**Gateway types by client:**

| Client Type | Gateway | Latency |
|---|---|---|
| Retail (website/app) | App/Web Gateway (HTTP) | Standard |
| Broker/Dealer | API Gateway (FIX/Non-FIX) | Low |
| HFT/Other API users | **Colocation (Colo) Engine** | Ultra-low (speed of light in cable) |

- **Colocation engine**: Trading engine software running on servers rented by the broker **in the exchange's data center**

### 4.5 Market Data Publisher (MDP)

- Receives executions from matching engine
- Builds **order books** and **candlestick charts** from execution stream
- Collectively called **market data**
- Sends to Data Service for subscriber access

### 4.6 Reporter

- Collects attributes from both incoming orders and outgoing executions
- Incoming new order → order details; outgoing execution → order ID, price, quantity, execution status
- **Merges** both sources for reports
- Outputs: Settlement & Clearing, Books & Records, Reporting

---

## 🔹 5. API Design

RESTful API between brokers and client gateway. Institutional clients may use different protocols but same basic functionality.

### POST `/v1/order` — Place Order

**Parameters:**

| Field | Type | Description |
|---|---|---|
| `symbol` | String | Stock symbol |
| `side` | String | buy or sell |
| `price` | Long | Limit order price |
| `orderType` | String | limit or market (only limit supported) |
| `quantity` | Long | Order quantity |

**Response Body:** `id`, `creationTime`, `filledQuantity`, `remainingQuantity`, `status` (new/cancelled/filled) + input params

**Status Codes:** 200, 40x, 500

### GET `/v1/execution` — Query Executions

**Parameters:** `symbol` (String), `orderId` (optional, String), `startTime` (Long, epoch), `endTime` (Long, epoch)

**Response:** Array of executions, each with: `id`, `orderId`, `symbol`, `side`, `price`, `orderType`, `quantity`

### GET `/v1/marketdata/orderBook/L2` — Order Book

**Parameters:** `symbol` (String), `depth` (Int), `startTime` (Long), `endTime` (Long)

**Response:** `bids` (array w/ price+size), `asks` (array w/ price+size)

### GET `/v1/marketdata/candles` — Historical Prices (Candlestick)

**Parameters:** `symbol` (String), `resolution` (Long, seconds), `startTime` (Long), `endTime` (Long)

**Response:** Array of candles, each with: `open`, `close`, `high`, `low` (all Double)

---

## 🔹 6. Data Models

### 6.1 Three Main Data Types

1. **Product, Order, Execution**
2. **Order Book**
3. **Candlestick Chart**

### 6.2 Product

- Attributes: product type, trading symbol, UI display symbol, settlement currency, lot size, tick size
- Doesn't change frequently → highly cacheable, any DB works

### 6.3 Order (Logical Model)

| Field | Type |
|---|---|
| `orderID` | UUID |
| `productID` | int |
| `price` | long |
| `quantity` | long |
| `side` | Side |
| `orderStatus` | OrderStatus |
| `orderType` | OrderType |
| `timeInForce` | TimeInForce |
| `symbol` | long |
| `userID` | long |
| `clientOrderID` | string |
| `broker` | string |
| `accountID` | long |
| `entryTime` | long |
| `transactionTime` | long |

### 6.4 Execution (Logical Model)

| Field | Type |
|---|---|
| `execID` | UUID |
| `orderID` | UUID |
| `price` | long |
| `quantity` | long |
| `side` | Side |
| `orderStatus` | OrderStatus |
| `orderType` | OrderType |
| `symbol` | long |
| `userID` | long |
| `feeCurrency` | Currency |
| `feeRate` | long |
| `feeAmount` | long |
| `accountID` | long |
| `execStatus` | ExecStatus |
| `transactionTime` | long |

**Relationship:** Order (1) → Execution (0..n) → Product (1)

**Storage in different flows:**
- **Critical trading path**: NOT stored in a database. Executes in memory, uses hard disk or shared memory. Stored in sequencer for fast recovery, archived after market close
- **Reporter**: Writes to DB for reconciliation and tax reporting
- **MDP**: Receives executions to reconstruct order book and candlestick data

### 6.5 Order Book Data Structure

An order book is a list of buy/sell orders organized by **price level**. Key requirements:

| Requirement | Complexity Target |
|---|---|
| Lookup volume at a price level | O(1) |
| Place new order | O(1) |
| Cancel an order | O(1) |
| Match an order | O(1) |
| Update (replace) an order | Fast |
| Query best bid/ask | Fast |
| Iterate through price levels | Fast |

**Implementation:**

```java
class PriceLevel {
    private Price limitPrice;
    private long totalVolume;
    private List<Order> orders;  // Use doubly-linked list for O(1)
}
class Book<Side> {
    private Side side;
    private Map<Price, PriceLevel> limitMap;
}
class OrderBook {
    private Book<Buy> buyBook;
    private Book<Sell> sellBook;
    private PriceLevel bestBid;
    private PriceLevel bestOffer;
    private Map<OrderID, Order> orderMap;  // Helper for O(1) cancel
}
```

**Why doubly-linked list for `orders`?**
1. **Place** new order → append to **tail** of PriceLevel → O(1)
2. **Match** an order → remove from **head** of PriceLevel → O(1)
3. **Cancel** an order:
   - Step 1: Find Order in `orderMap` via OrderID → O(1)
   - Step 2: Remove from doubly-linked list → O(1) (has pointer to previous)
   - With singly-linked list, cancellation would be O(n) due to traversal

**Order execution example:**
- Buy 2,700 shares of Apple (market order)
- Matches all sell orders in best ask queue + first order in next price level
- After: bid/ask spread widens, best ask price increases by one level

### 6.6 Candlestick Chart Data Structure

```java
class Candlestick {
    private long openPrice;
    private long closePrice;
    private long highPrice;
    private long lowPrice;
    private long volume;
    private long timestamp;
    private int interval;
}
class CandlestickChart {
    private LinkedList<Candlestick> sticks;
}
```

**Memory optimizations:**
1. **Pre-allocated ring buffers** to hold sticks → reduces object allocations
2. **Limit sticks in memory**, persist rest to disk

- Market data persisted in **in-memory columnar database** (e.g., KDB) for real-time analytics
- After market close → persisted in historical database

---

## 🔹 7. Performance & Low-Latency Design

> Some large exchanges run almost everything on a **single gigantic server**.

### Latency Formula

```
Latency = Σ executionTimeAlongCriticalPath
```

**Two ways to reduce latency:**
1. **Decrease number of tasks** on critical path
2. **Shorten time per task:**
   - a. Reduce/eliminate network and disk usage
   - b. Reduce execution time for each task

### Critical Path Components

```
gateway → order manager → sequencer → matching engine
```

- Only necessary components — even **logging is removed** from critical path

### Problem with Multi-Server Design

- Round-trip network latency ≈ **500 microseconds**
- Multiple network hops → total network latency = **single-digit milliseconds**
- Sequencer persists to disk → even sequential writes ≈ **tens of milliseconds**
- Total end-to-end: **tens of milliseconds** — no longer competitive

### Single-Server Low-Latency Design

- **Eliminate network hops** by putting everything on **one server**
- Components communicate via **mmap** (memory-mapped files) as event store
- Components on single server: Order Manager, Matching Engine, Market Data Publisher, Reporter, Logging, Aggregated Risk Check, Position Keeper
- All share a common `mmap` layer

### Application Loops

- Each component runs in a **while loop** (application loop) that continuously polls for tasks
- Only **most mission-critical tasks** processed in the loop
- **Single-threaded**, thread **pinned to a fixed CPU core**

**Benefits of CPU pinning:**
1. **No context switch** — CPU fully allocated to that component's loop
2. **No locks, no lock contention** — only one thread updates state
3. Both contribute to **low 99th percentile latency**

**Trade-off:** More complex coding — engineers must carefully analyze time per task to avoid blocking subsequent tasks

### mmap (Memory-Mapped Files)

- `mmap(2)` — POSIX-compliant UNIX system call that maps a file into process memory
- Provides **high-performance inter-process memory sharing**
- When backing file is in `/dev/shm` (memory-backed filesystem), access has **zero disk I/O**
- Used to implement **message bus** between critical-path components
- Sending a message on mmap bus takes **sub-microsecond**

---

## 🔹 8. Event Sourcing

### Concept

- Traditional: Store current state in DB → hard to trace issue origins
- **Event sourcing**: Keep an **immutable log of all state-changing events** → events are the golden source of truth
- States recoverable by **replaying events in sequence**

| Aspect | Non-Event Sourcing | Event Sourcing |
|---|---|---|
| Storage | Current state (Order: V2, Filled) | Events (Seq 100: NewOrderEvent, Seq 101: OrderFilledEvent) |
| Debugging | Hard to trace | Full audit trail |
| Recovery | Complex | Replay events |

### Event Sourcing in Exchange Design

**Architecture using mmap event store as message bus:**

1. **Gateway** transforms FIX → "FIX over Simple Binary Encoding" (SBE) for fast/compact encoding → sends `NewOrderEvent` via Event Store Client
2. **Order Manager** (embedded in matching engine) receives `NewOrderEvent`, validates, updates internal order states, sends to Matching Core
3. On match → `OrderFilledEvent` generated → sent to event store
4. Other components (MDP, Reporter) subscribe to event store and process events asynchronously

**Key design differences from high-level design:**

1. **Order Manager is embedded** in different components (not centralized)
   - Makes sense because each component maintains order states by itself
   - With event sourcing, states are guaranteed **identical and replayable**
2. **Sequencer is absorbed** into the event store design
   - Event store entry contains a `sequence` field injected by sequencer
   - **Only one sequencer** per event store — bad practice to have multiple (lock contention)
   - Sequencer is a **single writer** — sequences events before sending to event store
   - Much simpler than high-level design sequencer (does one thing, super fast)

**Sequencer in mmap environment:**
- Pulls events from **ring buffer** local to each component
- Stamps sequence ID on event → sends to event store
- Backup sequencers for high availability

---

## 🔹 9. High Availability

**Target: 4 nines (99.99%)** → only **8.64 seconds** of downtime per day → requires almost immediate recovery.

### Strategy

1. **Identify single-points-of-failure** (e.g., matching engine failure = disaster) → set up **redundant instances**
2. **Fast failure detection + fast failover** to backup

### Stateless vs Stateful

- **Stateless** (client gateway): Horizontally scale by adding servers
- **Stateful** (order manager, matching engine): Must **copy state data across replicas**

### Hot-Warm Architecture

- **Hot** matching engine = primary instance, processes and emits events
- **Warm** matching engine = receives and processes **same events** but does **NOT** emit events to event store
- On primary failure → warm **immediately takes over** as primary, starts emitting events
- On warm failure + restart → **recovers all states from event store** (event sourcing makes this easy)

**Failure detection:**
- Normal monitoring of hardware/processes
- **Heartbeats** from matching engine — missed heartbeat indicates potential problem

### Cross-Server Replication

- Hot-warm works within single server boundary
- For HA across machines/data centers: **entire server** is either hot or warm
- **Entire event store replicated** from hot to all warm replicas
- Use **reliable UDP** for efficient broadcast (e.g., Aeron design)

---

## 🔹 10. Fault Tolerance

### Key Questions

1. How and when to failover to backup?
2. How to choose leader among backups?
3. What is the **RTO** (Recovery Time Objective)?
4. What is the **RPO** (Recovery Point Objective)? Can system operate degraded?

### Challenges of "Down" Detection

1. **False alarms** → unnecessary failovers
2. **Code bugs** might bring down primary, then same bug brings down backup after failover → all instances knocked out

**Suggestions:**
- Initially perform failovers **manually** until enough operational experience gathered
- Automate only with confidence
- **Chaos engineering** surfaces edge cases faster

### Leader Election with Raft

- Raft cluster with **N servers**, each with own event store
- Leader sends data to all followers
- Minimum votes for operation: **n/2 + 1** (e.g., 5 servers → minimum 3)
- Followers receive events via **AppendEntries RPCs** → save to own mmap event store

**Leader election process:**
1. Leader sends heartbeat messages (empty AppendEntries) to followers
2. If follower doesn't receive heartbeat within timeout → triggers **election timeout** → initiates new election
3. First follower reaching timeout becomes **candidate** → sends `RequestVote` to others
4. If receives **majority votes** → becomes new leader
5. If candidate has **lower term value** than another node → cannot be leader
6. If multiple candidates simultaneously → **"split vote"** → election times out → new election

### RTO and RPO

| Metric | Definition | Exchange Target |
|---|---|---|
| **RTO** | Max tolerable downtime before significant damage | **Seconds** — requires automatic failover |
| **RPO** | Max tolerable data loss before significant harm | **Near zero** — data loss not acceptable |

- Categorize services by priority → define **degradation strategy** for minimum service level
- With Raft: many data copies → state consensus guaranteed → new leader functions immediately after crash

---

## 🔹 11. Matching Algorithms

### FIFO Matching (Pseudocode)

```java
Context handleOrder(OrderBook orderBook, OrderEvent orderEvent) {
    if (orderEvent.getSequenceId() != nextSequence) {
        return Error(OUT_OF_ORDER, nextSequence);
    }
    if (!validateOrder(symbol, price, quantity)) {
        return ERROR(INVALID_ORDER, orderEvent);
    }
    Order order = createOrderFromEvent(orderEvent);
    switch (msgType):
        case NEW:    return handleNew(orderBook, order);
        case CANCEL: return handleCancel(orderBook, order);
        default:     return ERROR(INVALID_MSG_TYPE, msgType);
}

Context handleNew(OrderBook orderBook, Order order) {
    if (BUY.equals(order.side)) {
        return match(orderBook.sellBook, order);
    } else {
        return match(orderBook.buyBook, order);
    }
}

Context handleCancel(OrderBook orderBook, Order order) {
    if (!orderBook.orderMap.contains(order.orderId)) {
        return ERROR(CANNOT_CANCEL_ALREADY_MATCHED, order);
    }
    removeOrder(order);
    setOrderStatus(order, CANCELED);
    return SUCCESS(CANCEL_SUCCESS, order);
}

Context match(OrderBook book, Order order) {
    Quantity leavesQuantity = order.quantity - order.matchedQuantity;
    Iterator<Order> limitIter = book.limitMap.get(order.price).orders;
    while (limitIter.hasNext() && leavesQuantity > 0) {
        Quantity matched = min(limitIter.next.quantity, order.quantity);
        order.matchedQuantity += matched;
        leavesQuantity = order.quantity - order.matchedQuantity;
        remove(limitIter.next);
        generateMatchedFill();
    }
    return SUCCESS(MATCH_SUCCESS, order);
}
```

### FIFO Matching

- **First In First Out** — order arriving first at a price level gets matched first

### Other Algorithms

- **FIFO with LMM (Lead Market Maker)**: Allocates certain quantity to LMM based on pre-negotiated ratio **ahead** of FIFO queue
- Commonly used in **futures trading**
- Also used in **dark pools**

---

## 🔹 12. Determinism

### Functional Determinism

- Guaranteed by **sequencer + event sourcing**
- If events replayed in same order → **same results**
- **Actual time** of event doesn't matter — only the **order** matters
- Event timestamps converted from discrete uneven dots → continuous dots → reduces replay/recovery time

### Latency Determinism

- Almost the **same latency for each trade** through the system
- Measured by **99th percentile latency** (or even 99.99th)
- Use **HdrHistogram** to calculate latency
- Low 99th percentile → stable performance across almost all trades

**Common latency fluctuation causes (Java):**
- Safe points
- HotSpot JVM **Stop-the-World garbage collection**

---

## 🔹 13. Market Data Publisher Optimizations

- L3 order book data is the most detailed market view
- Detailed L2/L3 data is **expensive** — many hedge funds record via exchange real-time API
- **MDP** receives matched results → rebuilds order book + candlestick charts → publishes to subscribers

**Tiered access:**
- Retail: default 5 levels of L2 data; pay extra for 10 levels
- MDP memory has **upper limit** on candlesticks

### Ring Buffers

- **Ring buffer** (circular buffer) — fixed-size queue, head connected to tail
- Space **pre-allocated** → no object creation/deallocation
- **Lock-free** data structure
- **Padding** ensures ring buffer's sequence number is never in a cache line with anything else

---

## 🔹 14. Fairness, Multicast & Network Security

### Distribution Fairness

- Lower latency = unfair advantage (like seeing the future)
- Regulated exchanges must guarantee **all receivers get data at the same time**
- Problem: If MDP subscriber list is ordered by connection time, first subscriber always gets data first → clients fight to be first

**Mitigations:**
- **Multicast via reliable UDP** — broadcast to many participants at once
- Assign **random order** on subscriber connection

### Multicast

| Protocol | Description |
|---|---|
| **Unicast** | One source → one destination |
| **Broadcast** | One source → entire subnetwork |
| **Multicast** | One source → set of hosts (can be different subnetworks) |

- Configure receivers in **same multicast group** → receive data simultaneously (in theory)
- UDP is unreliable → handle **retransmission** (e.g., NACK-Oriented Reliable Multicast)

### Colocation

- Exchanges offer colocation services — hedge fund/broker servers in **same data center** as exchange
- Latency proportional to **cable length**
- Not unfair — considered a **paid VIP service**

### Network Security (DDoS Defense)

1. **Isolate** public services/data from private services → DDoS doesn't impact key clients; use read-only copies
2. **Caching layer** for infrequently-updated data → most queries don't hit DB
3. **Harden URLs** — avoid query-string parameterized URLs (easy to generate unique requests); use cacheable URLs like `/data/recent`
4. **Safelist/blocklist** mechanisms via network gateway products
5. **Rate limiting**

---

## 📐 Formulas / Rules / Frameworks

```
QPS = total_orders_per_day / (trading_hours_in_seconds)
    = 1,000,000,000 / (6.5 × 3,600)
    ≈ 43,000

Peak QPS = 5 × QPS ≈ 215,000

Latency = Σ executionTimeAlongCriticalPath

Raft minimum votes = n/2 + 1
```

---

## 📖 Key Definitions

| Term | Meaning |
|---|---|
| **Order Book** | List of buy/sell orders for a symbol, organized by price level |
| **Execution (Fill)** | Outbound matched result from the matching engine |
| **Sequencer** | Component that stamps sequential IDs on orders and executions for determinism |
| **mmap** | POSIX system call mapping a file into process memory for inter-process communication |
| **Event Sourcing** | Pattern of storing immutable log of all state-changing events as source of truth |
| **FIX** | Financial Information eXchange protocol (1991), vendor-neutral for securities transactions |
| **SBE** | Simple Binary Encoding — fast, compact encoding for internal exchange communication |
| **RTO** | Recovery Time Objective — max tolerable downtime |
| **RPO** | Recovery Point Objective — max tolerable data loss |
| **Colocation** | Broker servers physically located in exchange's data center for minimal latency |
| **Application Loop** | Single-threaded while loop pinned to CPU core, processing tasks with minimal latency |
| **Ring Buffer** | Fixed-size circular queue with pre-allocated space, lock-free |
| **Dark Pool** | Private exchange where orders are hidden from public order book |

---

## 🆚 Comparisons

### Trading Flow vs Market Data Flow vs Reporting Flow

| Aspect | Trading Flow | Market Data Flow | Reporting Flow |
|---|---|---|---|
| **On critical path?** | ✅ Yes | ❌ No | ❌ No |
| **Latency requirement** | Microseconds–milliseconds | Low but not ultra | Relaxed |
| **Key concern** | Speed, accuracy | Timeliness, fairness | Accuracy, compliance |
| **Data direction** | Client → Exchange → Client | Exchange → Subscribers | Exchange → DB |

### Multi-Server vs Single-Server Design

| Aspect | Multi-Server | Single-Server |
|---|---|---|
| **Network latency** | ~500µs per hop, multiple hops | Eliminated |
| **Disk latency** | Tens of ms for event store | mmap on /dev/shm → zero disk |
| **Total latency** | Tens of milliseconds | Tens of **micro**seconds |
| **Complexity** | Distributed systems challenges | Complex single-process design |
| **Fault tolerance** | Easier horizontal scaling | Requires hot-warm replication |

### Event Sourcing vs Traditional State Storage

| Aspect | Traditional | Event Sourcing |
|---|---|---|
| **Stored** | Current state | All state-changing events |
| **Debugging** | Only see current state | Full event history |
| **Recovery** | Restore from backup | Replay events |
| **Determinism** | Not guaranteed | Guaranteed via sequence |
| **Immutability** | Mutable records | Immutable event log |

---

## ⚡ Quick-Recall Points

- NYSE trades **billions of matches per day**; HKEX about **200 billion shares per day**
- Exchange open: **6.5 hours** (9:30 AM – 4:00 PM ET), Mon–Fri
- Round-trip network latency ≈ **500 microseconds**
- mmap message bus latency: **sub-microsecond**
- Availability target: **99.99%** = 8.64 seconds max downtime/day
- Matching engine produces **two executions** per match (buy + sell side)
- Order book uses **doubly-linked list** for O(1) place/match/cancel
- `orderMap` (HashMap<OrderID, Order>) enables O(1) cancel lookup
- Sequencer must be **single writer** to avoid lock contention
- FIX protocol created in **1991**
- CPU pinning → no context switch, no locks → low p99 latency
- `/dev/shm` is a **memory-backed filesystem** → mmap over it = zero disk access
- RPO for exchange is **near zero** — data loss not acceptable

---

## ⚠️ Mistakes & Misconceptions

- ❌ Using a database on the critical trading path → ✅ Use in-memory processing with mmap event store → 💬 DB latency (even SSDs) is too slow for exchange-level requirements
- ❌ Multiple sequencers for one event store → ✅ Single sequencer (single writer) → 💬 Multiple writers create lock contention, wasting time in a busy system
- ❌ Singly-linked list for order book → ✅ Doubly-linked list → 💬 Cancel requires O(n) traversal with singly-linked list vs O(1) with doubly-linked
- ❌ Automating failover immediately on new system → ✅ Start with manual failover, automate after gaining operational confidence → 💬 False alarms and code bugs can cascade failures
- ❌ Kafka for exchange sequencer → ✅ mmap-based event store → 💬 Kafka's latency is too high and unpredictable for exchange use

---

## 🎯 Actionable Takeaways

- **Minimize critical path components** — only gateway → order manager → sequencer → matching engine
- **Eliminate network hops** by colocating all critical-path components on a single server
- **Use mmap over `/dev/shm`** for zero-disk inter-process communication
- **Pin application loop threads to CPU cores** for no context switches, no locks
- **Use event sourcing** for order state management — enables deterministic replay, fast recovery, and audit trails
- **Implement order book with doubly-linked list + HashMap** for O(1) on all core operations
- **Use Raft consensus** for fault-tolerant leader election across replicated event stores
- **Broadcast market data via multicast (reliable UDP)** for fair distribution
- **Use ring buffers** with pre-allocated memory in MDP to avoid GC pressure
- **Investigate JVM safe points and GC pauses** as sources of latency spikes
- **Apply chaos engineering** to surface edge cases and build operational confidence before automating failover

---

## 📝 Summary

- **Stock exchange** core function: efficiently match buyers and sellers; modern exchanges process billions of orders/day
- **Requirements**: 100 symbols, 1B orders/day (~43K QPS, 215K peak), 99.99% availability, ms-level latency, risk checks, wallet management
- **Three flows**: Trading (critical path), Market Data (not critical), Reporting (not critical) — each with different latency requirements
- **Matching Engine** maintains order book per symbol, matches orders, outputs execution stream; must be **deterministic** for HA
- **Sequencer** stamps sequence IDs for timeliness, fairness, replay, and exactly-once guarantee; acts as single-writer event store
- **Order Book** uses doubly-linked list + HashMap for O(1) place/match/cancel; organized by price level
- **Low-latency design**: All critical components on **single server**, communicate via **mmap** over `/dev/shm` (sub-microsecond), CPU-pinned single-threaded application loops
- **Event sourcing** stores immutable event log instead of current state; enables deterministic replay, fast recovery, audit
- **High availability** via hot-warm matching engine replication; event store replicated via reliable UDP across servers
- **Fault tolerance** via **Raft** consensus (leader election, state replication); RPO near zero, RTO at second level
- **Market data fairness** ensured via **multicast (reliable UDP)** to all subscribers simultaneously; colocation is a paid VIP service
