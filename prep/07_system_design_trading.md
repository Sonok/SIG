# 07 — System & Component Design for Low-Latency Trading (SIG Final Round)

> Final round: **"Design and Team Fit."** This is *not* a LeetCode round and *not* a staff-engineer
> distributed-systems round. It's a conversation where they watch **how you think** about a trading
> system: do you ask the right questions, do you reason about latency, do you know where the
> bottleneck is, and can you defend a trade-off without hand-waving.

---

## 1. Why this matters for SIG + the design framework

### What they're actually grading (for an intern)

SIG runs low-latency C++ trading systems. In the design conversation they want to see:

- **Structured thinking** — you decompose the problem instead of blurting a memorized architecture.
- **Clarifying questions** — you nail down requirements *before* designing. This is the single
  biggest signal. A candidate who asks "what's the latency budget? what throughput? one symbol or
  many? do we need to recover from gaps?" already looks senior.
- **Latency/throughput awareness** — you can say *where* the nanoseconds and microseconds go.
- **Knowing the bottleneck** — you can point at the hot path and say "this is what matters, the rest
  can be slow."
- **Trade-offs, stated explicitly** — every choice costs something. Naming the cost is the whole game.

What they are **NOT** grading: whether you've memorized LMAX Disruptor, whether you can name 12
exchange protocols, or whether you produce a "perfect" answer. **Reasoning out loud > correctness.**
A wrong answer you reason through beats a right answer you assert.

> 🎤 **If they ask "have you designed trading systems before?"** Be honest: "I built a live
> systematic equity trading system end-to-end — market data ingestion, microstructure signals, order
> generation — so I've felt where latency and data quality bite. I haven't built an exchange-grade
> matching engine, but I understand the components and the trade-offs." Honesty + demonstrated
> instinct beats fake expertise.

### The framework — use this on EVERY design prompt

```
1. CLARIFY requirements   → What exactly are we building? Scope it. Ask questions.
2. CONSTRAINTS            → Latency budget? Throughput (msgs/sec, orders/sec)? Reliability needs?
                            Number of symbols? Single-threaded or multi? Recovery semantics?
3. COMPONENTS             → Break it into stages/modules. Draw boxes.
4. DATA FLOW              → How does a message move through? What's on the hot path vs off it?
5. BOTTLENECKS            → Where's the slowest step? Where does allocation/locking/syscalls hide?
6. TRADE-OFFS             → For each design choice, what did I pay? Memory vs speed, simplicity vs
                            latency, generality vs the hot path.
```

Say this framework out loud at the start: *"Let me first make sure I understand the requirements,
then talk constraints, sketch components, walk a message through, and call out bottlenecks and
trade-offs."* That sentence alone signals structured thinking.

**Latency vocabulary** (so you size budgets correctly):

| Scale | Order of magnitude | Example |
|-------|-------------------|---------|
| ns | 1–100 ns | L1/L2/L3 cache hit, a few hundred instructions, lock-free queue push |
| µs | 1–10 µs | RAM access chain, parse a market-data packet, full tick-to-trade in a good system |
| ~10 µs | | crossing the kernel network stack (why we use kernel bypass) |
| 100 µs–ms | | syscalls under contention, disk, logging done wrong, GC in a managed language |

A tuned tick-to-trade (market data in → order out) lands in the **single-digit microseconds**, with
the truly competitive shops in **hundreds of nanoseconds** using FPGAs. As an intern, knowing the
*scale* matters more than the exact number.

---

## 2. Anatomy of an electronic trading system (end-to-end)

```
                         ┌─────────────────────────────────────────────┐
                         │                 EXCHANGE                     │
                         │  (matching engine, publishes market data)    │
                         └───────────────┬──────────────▲──────────────┘
                                         │              │
                       market data (UDP  │              │ orders (TCP/UDP,
                       multicast, A/B)    │              │ binary protocol)
                                         ▼              │
   ┌───────────────────────────────────────────────────┴───────────────────────┐
   │                          YOUR TRADING SYSTEM                                │
   │                                                                            │
   │  ┌──────────────┐   ┌────────────┐   ┌──────────────┐                      │
   │  │ FEED HANDLER │──▶│ BOOK       │──▶│ STRATEGY /   │                      │
   │  │ parse, seq   │   │ BUILDER    │   │ SIGNAL       │                      │
   │  │ gap-detect,  │   │ (limit     │   │ (alpha,      │                      │
   │  │ A/B arb,     │   │  order     │   │  decisions)  │                      │
   │  │ normalize    │   │  book)     │   │              │                      │
   │  └──────────────┘   └────────────┘   └──────┬───────┘                      │
   │        ▲                                    │ "send order X"               │
   │        │                                    ▼                              │
   │   raw UDP                            ┌──────────────┐    ┌──────────────┐  │
   │   multicast                          │ ORDER MGMT   │──▶│ EXCHANGE     │──┼─▶ to exchange
   │                                      │ (OMS): risk  │   │ GATEWAY      │  │
   │                                      │ checks, order│   │ encode wire  │  │
   │                                      │ state machine│◀──│ proto, ACKs  │◀─┼── from exchange
   │                                      └──────────────┘   └──────────────┘  │
   └────────────────────────────────────────────────────────────────────────────┘
         ── HOT PATH: feed handler → book → strategy → OMS → gateway → wire ──
```

Component by component, with **latency sensitivity**:

| Component | Job | Latency sensitivity |
|-----------|-----|---------------------|
| **Feed handler** | Receive UDP multicast packets, validate sequence numbers, detect gaps, arbitrate A/B feeds, parse the exchange's binary format, normalize into internal events | **Extremely hot.** First on the path. Every market data update flows through it. Parsing must be allocation-free and branch-light. |
| **Book builder** | Apply each normalized update to the **limit order book**; maintain best bid/ask, depth, per-level queues | **Extremely hot.** Touched on every update. The order book data structure choice dominates here. |
| **Strategy / signal** | Read the book + other state, compute signals/alpha, decide whether/what to trade | **Hot**, but this is *your* logic. Keep the decision path tight; heavy research-y computation belongs off-path or precomputed. |
| **Order management (OMS)** | **Pre-trade risk checks**, maintain order state machine (new → pending → acked → filled/canceled), throttles, position/exposure tracking | **Hot** — sits between decision and wire. Risk checks must be O(1)-ish; they're on the critical path to sending. |
| **Exchange gateway** | Encode the order into the exchange's wire protocol, manage the session/sequence, handle ACKs/fills | **Hot** — last step before the wire. Encoding must be cheap; often kernel-bypass sockets here too. |
| **Logging / telemetry** | Record what happened for debugging, compliance, PnL | **Off hot path** (must be) — async, never block the trader. |

Key mental model: **the hot path is a pipeline.** Market data comes in one side, an order goes out
the other. Everything that isn't strictly required to make that decision (logging, analytics,
persistence, monitoring) gets pushed *off* the path.

> 🎤 **If they ask "what's the critical path / tick-to-trade?"**
> "From a market-data packet hitting the NIC, through feed handler → book update → strategy
> decision → risk check → gateway encode → packet back on the wire. That round of work is what we
> minimize. Anything not needed to make that decision goes off-path."

---

## 3. THE LIMIT ORDER BOOK — the single most important design topic

If you nail one thing, nail this. Expect "design a fast order book" as a prompt.

### 3.1 What it is

The limit order book (LOB) holds all resting limit orders for a symbol, organized by price:

- **Bids** (buy orders) on one side, **asks/offers** (sell orders) on the other.
- Each side is a set of **price levels**. At each price level there's a **FIFO queue** of orders
  (price-time priority: same price → first in, first matched).
- **Best bid** = highest buy price; **best ask** = lowest sell price. The gap between them is the
  **spread**. Best bid/ask (the "top of book" / BBO) is read constantly, so getting it must be O(1).

```
        ASKS (sell)                         BIDS (buy)
   price   qty   orders (FIFO)         price   qty   orders (FIFO)
   ─────────────────────────          ─────────────────────────
   101.03   500  [o7, o8]             100.99   300  [o3]
   101.02   200  [o5, o6]    ◀ best   100.98   900  [o1, o2, o4]  ◀ best bid
   101.01   100  [o4b]          ask   100.97   150  [o9]
   ...                                100.96   400  [o10, o11]
                                       ...
              spread = 101.01 − 100.98 = 0.03
```

### 3.2 Operations the book must support (fast)

| Op | What it does | Frequency |
|----|--------------|-----------|
| **Add** | New limit order rests at a price level (append to that level's FIFO tail) | very high |
| **Cancel** | Remove a specific order (need O(1) lookup by order id, then unlink) | very high |
| **Modify** | Change qty/price. Qty-down can keep queue position; price change or qty-up usually = cancel + re-add (loses time priority) | high |
| **Match / execute** | An aggressive order crosses the spread and consumes resting orders at the best level FIFO-first | high |
| **Best bid/ask** | Read top of book | constant |

### 3.3 How to design a FAST order book

This is the meat. Walk through it as a series of design decisions.

**Naïve design (and why it's too slow):**

```cpp
// "Textbook" version — DO NOT propose this as the fast design
std::map<Price, std::list<Order>> bids; // sorted tree
std::map<Price, std::list<Order>> asks;
```

Why `std::map` is often too slow on the hot path:
- It's a **red-black tree**: node-per-price, scattered across the heap → **cache misses** everywhere.
- Every insert/erase may **allocate/free** a node → allocation on the hot path (a cardinal sin).
- O(log n) per op with bad constants, and pointer-chasing kills you more than the log n.
- `std::list<Order>` also allocates per node and is cache-hostile.

**The classic fast design:** price levels in a flat **array indexed by price**, plus **intrusive
linked lists** of orders, plus a **hashmap from order id → order** for O(1) cancel.

Core idea: prices are discrete (a tick grid). Convert price to an integer index:
`idx = (price − min_price) / tick_size`. Then a level is just `levels[idx]` — a direct array access,
**O(1)**, cache-friendly, no tree, no allocation.

```cpp
struct Order {
    OrderId  id;
    uint32_t qty;
    PriceIdx price_idx;
    Order*   prev;   // intrusive list pointers — Order IS the node
    Order*   next;   // no separate list-node allocation
};

struct PriceLevel {
    uint64_t total_qty = 0;   // aggregate qty at this level (fast depth reads)
    uint32_t order_count = 0;
    Order*   head = nullptr;  // FIFO: match from head
    Order*   tail = nullptr;  // FIFO: append new orders at tail
};

struct Book {
    // Flat arrays indexed by price tick. One per side.
    std::vector<PriceLevel> bid_levels;   // size = number of ticks in range
    std::vector<PriceLevel> ask_levels;

    PriceIdx best_bid_idx;   // track top of book directly → O(1) BBO
    PriceIdx best_ask_idx;

    // O(1) cancel/modify: id -> the Order object (often from a preallocated pool)
    std::unordered_map<OrderId, Order*> by_id;

    // Preallocated pool of Order objects — NO new/delete on the hot path
    ObjectPool<Order> order_pool;
};
```

**Add** (O(1)):
```cpp
void add(OrderId id, Side s, PriceIdx p, uint32_t qty) {
    Order* o = order_pool.alloc();           // pop from free list, no malloc
    *o = {id, qty, p, nullptr, nullptr};
    PriceLevel& lvl = level(s, p);
    push_back(lvl, o);                        // intrusive append at tail
    lvl.total_qty += qty;  lvl.order_count++;
    by_id[id] = o;
    update_best_on_add(s, p);                 // maybe move best_bid/ask_idx
}
```

**Cancel** (O(1)):
```cpp
void cancel(OrderId id) {
    Order* o = by_id[id];                     // O(1) lookup
    PriceLevel& lvl = level_of(o);
    unlink(lvl, o);                           // splice out via prev/next, O(1)
    lvl.total_qty -= o->qty;  lvl.order_count--;
    by_id.erase(id);
    order_pool.free(o);                       // return to pool, no free()
    if (lvl.order_count == 0) maybe_advance_best(); // only then scan for new best
}
```

**Best bid/ask** is just `bid_levels[best_bid_idx]` — **O(1)**, no traversal.

**Trade-offs to articulate** (this is where you earn points):

| Choice | Buys you | Costs you |
|--------|----------|-----------|
| Array-of-price-levels (vs `std::map`) | O(1) access, cache locality, no per-level alloc | **Memory** — you allocate slots for the whole price range even if sparse. Fine for liquid symbols on a tight tick grid; wasteful if the price range is huge. |
| Intrusive linked list (vs `std::list`) | No per-order node allocation, FIFO O(1) add/remove, cache-friendlier with pooling | More manual pointer code; easy to introduce bugs |
| Object pool / preallocation | **Zero allocation on hot path** (the big win) | Must size the pool up front; complexity |
| Hashmap id→order for cancel | O(1) cancel/modify | Hash cost + memory; some shops use a flat array indexed by a dense order id instead |
| Tracking best_bid/ask_idx | O(1) BBO reads | Must maintain it on add/cancel; need to rescan when the best level empties (bounded scan, amortized cheap) |

**Refinements worth mentioning if pressed:**
- If the price range is large/sparse, a pure flat array wastes memory. Hybrids: a flat array for a
  **window around the inside** (where all the action is) + a fallback structure for far-away levels.
- For "I only ever read top-of-book N levels," some systems keep a small fixed-size BBO/depth cache.
- L3 (full order-by-order) vs L2 (aggregated per-level) vs L1 (just BBO): the more detail you keep,
  the more work per update. Design to the data the *strategy* actually needs.

> 🎤 **If they ask "why not just use `std::map`?"**
> "It's a balanced tree — log n with poor constants, pointer-chasing across the heap so lots of
> cache misses, and it allocates a node per insert which I never want on the hot path. Prices live
> on a discrete tick grid, so I'd index a flat array by price tick for O(1), cache-friendly access,
> and use intrusive lists plus an object pool so there's zero allocation per order."

> 🎤 **If they ask "how do you cancel fast?"**
> "Keep a map (or flat array) from order id to the order object, so cancel is an O(1) lookup, then
> an O(1) unlink from the intrusive FIFO list at that level. The order object goes back to a pool,
> not freed."

---

## 4. Matching engine basics

You're more likely to *consume* a matching engine than build one, but know the model.

- **Price-time priority** (the standard for most equity/futures venues): match best price first;
  within a price level, **oldest order first** (FIFO / time priority). This is exactly why the
  per-level queue is FIFO. (Some venues use pro-rata or size-priority — mention you know variants
  exist.)
- **A match**: an aggressive incoming order crosses the spread. The engine walks the opposite side
  from the best price, consuming resting orders FIFO until the incoming order is filled or no more
  price levels cross. Each fill generates an execution report to both sides.

**Order types** (know what each means for matching):

| Type | Behavior |
|------|----------|
| **Limit** | Rest at a price; only fill at that price or better. The default resting order. |
| **Market** | Execute immediately at best available price(s), walking levels until filled. No price protection. |
| **IOC** (Immediate-Or-Cancel) | Fill whatever you can right now; cancel the remainder. Never rests. |
| **FOK** (Fill-Or-Kill) | Fill the **entire** quantity immediately or cancel the whole thing. All-or-nothing, no resting. |

> 🎤 **If they ask "difference between IOC and FOK?"**
> "IOC takes partial fills and cancels the rest. FOK is all-or-nothing — either the full size fills
> immediately or nothing does."

---

## 5. Market data feed handler design

Prompt: "Design a market data feed handler." Walk the path of a packet.

**Transport:** exchanges publish market data over **UDP multicast** — UDP because it's low-latency
and fan-out friendly (one publisher, many subscribers, no per-subscriber TCP state). The cost of UDP
is **no delivery guarantee**: packets can drop or arrive out of order.

**So the feed handler must:**

1. **Receive** packets (often via kernel bypass — see §6).
2. **Sequence numbers + gap detection.** Each message carries a monotonically increasing seq number.
   The handler tracks the expected next seq. If it jumps (e.g. expected 1005, got 1008), there's a
   **gap** — you missed 1006–1007. Your book is now potentially wrong.
3. **A/B feed arbitration.** Exchanges publish the *same* data on **two independent lines (A and B)**.
   The handler listens to both and takes whichever message arrives first for each seq number, dropping
   the duplicate. If A drops a packet, B usually has it → you survive single-line loss without a slow
   recovery. This is the cheap, fast redundancy.
4. **Gap recovery** (when both A and B miss it): request a retransmit / snapshot from a recovery
   service, or mark the book **stale** and refuse to trade it until resynced. Recovery is **off the
   hot path** — slow, but correctness matters more than speed when you're already broken.
5. **Normalize.** Each exchange has its own binary wire format. Parse it (allocation-free, bounded
   work) into an **internal normalized event** (add/cancel/modify/trade with symbol, side, price,
   qty). Downstream code is then exchange-agnostic.
6. **Fan-out.** Publish normalized events to the book builder and any strategies that care — typically
   over **lock-free SPSC queues**, one per consumer, so the parse thread never blocks.

```
 A line ─┐
         ├─▶ [arbitrate by seq] ─▶ [gap detect] ─▶ [parse/normalize] ─▶ [fan-out SPSC] ─▶ book/strategies
 B line ─┘                              │
                                        └─▶ gap? ─▶ [recovery: snapshot/retransmit]  (OFF hot path)
```

**Trade-offs / things to call out:**
- **Throughput**: feeds burst hard (open/close, news). Size queues for the burst, not the average,
  or you drop/backpressure.
- **Correctness vs latency**: gap detection + A/B arbitration cost a little work per packet, but a
  wrong book is worse than a slightly slower one. Recovery is allowed to be slow.
- **Parsing**: no allocation, minimal branching, parse straight into stack/preallocated structs.
- **Multicast** is what makes one-to-many fan-out cheap; you join groups rather than open N connections.

> 🎤 **If they ask "how do you handle dropped packets?"**
> "Sequence numbers detect the gap. First line of defense is A/B arbitration — same data on two
> lines, take whichever arrives, so a single-line drop is invisible. If both miss it, request a
> retransmit/snapshot from the recovery service and mark the book stale until resynced — and that
> recovery path is off the hot path."

---

## 6. Order management / gateway + pre-trade risk

**OMS responsibilities:**

- **Pre-trade risk checks** — the gate every order passes before hitting the wire:
  - Max order size, max position / exposure limits
  - Price sanity ("fat finger" — is this price wildly off the market?)
  - Order rate / message throttles (exchanges fine you for excessive messaging)
  - Self-trade prevention, instrument tradeable/enabled, capital/buying-power
- **Order state machine**: each order moves through states and the OMS must track it accurately:

```
   NEW ──send──▶ PENDING_NEW ──ack──▶ LIVE ──(partial fill)──▶ LIVE ──(fill)──▶ FILLED
                     │                  │
                     │                  ├──cancel req──▶ PENDING_CANCEL ──ack──▶ CANCELED
                     └──reject──▶ REJECTED
```

You need this because the exchange is asynchronous: you send, then ACKs / fills / rejects come back
later. The OMS reconciles what you *think* is live with what the exchange says.

**Why pre-trade risk must be FAST:** it sits **on the critical path to sending an order.** Every
microsecond in the risk check is a microsecond of edge lost — by the time you send, the price you saw
may be gone. So risk checks are designed as **O(1) precomputed lookups**: limits cached in memory,
running position maintained incrementally (not recomputed), comparisons that are a handful of
instructions. You do *not* do a database query or recompute exposure from scratch on the hot path.

> 🎤 **If they ask "why does pre-trade risk have to be fast — isn't safety worth latency?"**
> "It's a gate every order crosses, so its latency is pure tax on getting to market. The trick is
> it doesn't need heavy computation — it's bounded O(1) checks against limits kept in memory and a
> position you maintain incrementally. So you get safety *and* speed; you just can't implement it as
> a slow lookup or recompute."

> 🎤 **If they ask "where do you keep risk limits — DB?"**
> "Loaded into memory at startup (and on update), never read from a DB on the hot path. The hot path
> only does in-memory comparisons."

---

## 7. Low-latency design patterns (systems context recap)

These are the levers. Sprinkle them into any design answer with the *reason* attached.

| Pattern | What | Why |
|---------|------|-----|
| **Lock-free SPSC queues** between stages | Single-producer/single-consumer ring buffer, no mutex | Pass messages stage-to-stage with no locking, no kernel involvement, predictable latency. One thread owns each stage. |
| **Busy-polling / spinning** | Thread spins reading the NIC/queue instead of blocking | Avoids the ~µs+ cost and jitter of waking a sleeping thread (context switch). Trades a burned CPU core for latency. |
| **No allocation on the hot path** | Preallocate everything; object pools; ring buffers | `malloc`/`new` can take µs and is non-deterministic (can hit the kernel). Allocation = jitter. |
| **Cache-friendly layouts** | Flat arrays, struct-of-arrays, hot fields together, avoid pointer chasing | A cache miss to RAM is ~100ns; you want L1/L2 hits. This is why the order book uses arrays not trees. |
| **Kernel bypass** (e.g. DPDK / Solarflare-style user-space networking) | NIC writes packets straight into user memory, skipping the kernel TCP/IP stack | The kernel network stack adds several µs and jitter per packet. Bypass is one of the biggest single latency wins. |
| **Thread pinning per pipeline stage** | Pin each stage's thread to a dedicated core (CPU affinity), isolate cores from the scheduler | No migration, warm caches, no being preempted by the OS. One core per hot stage. |
| **Branch-light, predictable code** | Minimize branches/mispredicts on the hot path; avoid virtual calls in inner loops | Mispredicts and indirect calls cost cycles and defeat the pipeline. |

Mental rule: **the hot path should make no syscalls, no allocations, and no locks** — just arithmetic,
memory it already owns, and lock-free handoffs between pinned threads.

> 🎤 **If they ask "why busy-poll instead of blocking?"**
> "Blocking puts the thread to sleep; waking it on a packet costs a context switch — order of a
> microsecond plus jitter. Spinning keeps the core hot and reacts immediately. You pay a CPU core for
> deterministic low latency, which is the right trade in a trading hot path."

> 🎤 **If they ask "what's the most impactful single optimization?"**
> "Usually kernel bypass for the network path and eliminating allocation on the hot path — both remove
> microseconds *and* jitter. After that, cache-friendly data layout in the book."

---

## 8. Logging / telemetry without hurting the hot path

You must log (debugging, compliance, PnL) but logging the naïve way (format a string, take a mutex,
write to disk/stdout) is a **disaster on the hot path** — it allocates, locks, and may do a syscall
that blocks for milliseconds.

**The pattern: async logging.**

1. On the hot path, the thread does the **minimum**: write a small **binary record** of the raw data
   (timestamp + an event id + raw fields/args) into a **per-thread lock-free ring buffer**. No string
   formatting, no lock, no I/O. This is a few nanoseconds.
2. A **separate background thread** drains the ring buffer, does the **deferred formatting** (turn the
   binary record into human-readable text), and writes to disk. All the expensive work happens here,
   off the hot path.

```
 hot thread: [event] ─▶ push binary record ─▶ [ lock-free ring buffer ] 
                                                        │
 background thread: ───────────────────────────────────┘─▶ format string ─▶ write to disk
```

**Key ideas to name:**
- **Defer formatting** — formatting is expensive; do it on the slow thread, not the trader.
- **Ring buffer** — fixed-size, preallocated, no allocation; if it overflows under extreme load you
  drop or overwrite oldest (a design choice: never block the hot path).
- **Binary on the hot path** — store raw args, not formatted text.
- This is essentially what real low-latency loggers (e.g. NanoLog / Quill-style) do.

> 🎤 **If they ask "how do you log without slowing the strategy?"**
> "The hot thread only writes a small binary record — timestamp and raw args — into a preallocated
> lock-free ring buffer, no formatting, no lock, no I/O. A background thread drains the buffer,
> formats the text, and writes to disk. The expensive part is fully off the hot path."

---

## 9. Reliability vs latency: what lives off the hot path

You can't make everything fast *and* safe *and* durable simultaneously — you choose per component.

**On the hot path (must be fast, kept minimal):** feed parse, book update, signal decision,
pre-trade risk (as O(1) checks), order encode.

**Off the hot path (allowed to be slow / async):**
- **Logging & telemetry** (async, §8)
- **Gap recovery / snapshots** (correctness over speed — slow is fine when already degraded)
- **Persistence / journaling / DB writes** (don't block the trader to write to disk)
- **Monitoring, metrics aggregation, dashboards**
- **Heavy/research-y signal computation** — precompute or do on a slower thread; the hot decision
  reads a cached result.
- **Config / risk-limit loading** — at startup or on update, not per-order.

**The trade-off statements they want to hear:**
- "A **stale or wrong book** is worse than a **slightly slower** one, so I'll spend a little latency
  on gap detection and A/B arbitration — but push *recovery* off the hot path."
- "Durability (writing every order to disk before sending) would add a syscall to the critical path,
  so I journal **asynchronously**; if I need stronger guarantees I accept the latency cost explicitly,
  and that's a business decision, not a free lunch."
- "Reliability often means **redundancy** (A/B feeds, failover gateways), which I can get *without*
  hot-path latency cost — that's the cheap kind of reliability and I take it."

> 🎤 **If they ask "latency vs reliability — how do you choose?"**
> "Per component. Anything required to make the trading decision stays on the hot path and minimal.
> Everything else — logging, persistence, recovery, monitoring — goes async/off-path. And I grab the
> reliability that's free, like redundant feeds, while pushing the expensive durability work off the
> critical path."

---

## 10. Worked example design prompts (reason out loud like this)

### (a) "Design a low-latency order book"

**Clarify first:** "One symbol or many? Do we need full order-by-order detail (L3) or just aggregated
per-level (L2)? What's the price range / tick size? Read-heavy (lots of BBO reads) or balanced with
adds/cancels? Single-threaded per symbol?" — *Assume: one liquid symbol, L3, tight tick grid,
single-threaded, very high add/cancel rate, constant BBO reads.*

**Constraints:** sub-microsecond per operation; zero allocation on the hot path; O(1) BBO; O(1)
cancel.

**Components / structure:** flat **array of price levels indexed by price tick** per side; each level
is a **FIFO intrusive linked list** of orders; a **hashmap (or flat array) from order id → order** for
O(1) cancel; an **object pool** of preallocated Order structs; **tracked best_bid/ask index** for O(1)
top-of-book. (Reference §3.3.)

**Data flow:** normalized event arrives → if add, pool-alloc an order, append at the level's tail,
update aggregate qty and maybe the best index → if cancel, id-lookup, unlink, return to pool, advance
best only if the level emptied → BBO is a direct array read.

**Bottlenecks:** cache misses (solved by flat arrays + pooling), allocation (solved by the pool),
finding the new best when the inside level empties (bounded scan, amortized cheap).

**Trade-offs:** array costs memory across the price range (fine on a tight grid near liquid prices,
wasteful if sparse → hybrid window-around-inside if needed); intrusive lists are fast but more manual
pointer code; `std::map` rejected for cache misses + per-node allocation + log-n constants.

### (b) "Design a market data feed handler"

**Clarify:** "Which transport — UDP multicast, I assume? Is there an A/B feed? Is there a recovery /
snapshot service? Do I need to rebuild the book or just forward normalized events? How many downstream
consumers?" — *Assume: UDP multicast, A/B lines, a retransmit service, fan-out to a book builder and
several strategies.*

**Constraints:** parse and forward in low single-digit µs; handle bursty throughput; correct under
packet loss/reordering.

**Components:** A/B **arbitration by sequence number** → **gap detection** → **allocation-free parser**
→ **normalizer** → **lock-free SPSC fan-out** to consumers → an off-path **recovery** path. (Reference
§5.) Likely **kernel bypass** on receive and a **pinned, busy-polling** receive thread.

**Data flow:** packet arrives on A or B → dedupe/order by seq, take first arrival → check seq
continuity → on gap, trigger recovery and/or mark stale (off path) → parse binary → emit normalized
event → push to each consumer's SPSC queue.

**Bottlenecks:** kernel stack (→ bypass), allocation in parsing (→ none), queue overflow on bursts
(→ size for the burst), the recovery path (→ keep it off the hot path).

**Trade-offs:** UDP gives low latency + cheap multicast fan-out at the cost of reliability, which you
buy back with A/B + recovery; gap checking costs a little per-packet work but a wrong book is far
worse.

### (c) "Build a system to detect when your strategy is losing money / a kill switch"

**Clarify:** "What triggers the kill — realized PnL drawdown, position limit, error rate, fill rate
anomaly, fat-finger? How fast must it react? Manual override too? Per-strategy or firm-wide?" —
*Assume: auto kill on PnL drawdown / position breach / abnormal behavior, fast, with a manual button,
per-strategy and a firm-wide master.*

**Design:**
- **Real-time risk/PnL monitor** maintaining **incremental** position and mark-to-market PnL as fills
  arrive (don't recompute from scratch). Cheap to update per fill.
- **Thresholds**: max drawdown, max position, max order rate, max error/reject rate, sanity on
  realized vs expected fills.
- **Trip mechanism**: when a threshold breaches → **immediately stop sending new orders** and
  (depending on policy) cancel resting orders. The cleanest enforcement point is the **OMS / gateway**
  — a single gate that, once tripped, rejects all outbound orders. That guarantees nothing escapes,
  regardless of strategy bugs.
- **Layers of defense**: pre-trade limits (per order, §6) catch the obvious; the kill switch is the
  *aggregate / behavioral* safety net on top. A **manual kill button** for humans, and a **firm-wide
  master** above per-strategy switches.
- **Latency**: the *enforcement* (stop sending) must be on the order path and instant; the
  *detection* computation is cheap because PnL/position are maintained incrementally. Heavy analytics
  can be off-path, but the trip itself is a fast in-memory flag check at the gateway.
- **Fail-safe bias**: if monitoring is uncertain or data is stale, err toward *not trading*. Tripping
  is cheap; a runaway strategy is catastrophic.

**Trade-offs:** tight thresholds = safer but more false trips (lost opportunity / babysitting);
enforcing at the gateway is robust (catches all bugs) but adds a flag check to the path (trivially
cheap). Cancel-on-trip protects you but spams the exchange — usually worth it.

> 🎤 **If they ask "where do you enforce the kill switch?"**
> "At the OMS/gateway — the single choke point every order passes. A tripped flag there rejects all
> outbound orders no matter what the strategy does, so even a buggy strategy can't get an order out.
> Detection is cheap because I keep PnL and position incrementally; enforcement is just a flag check."

### (d) "Design an async logger that doesn't slow the hot path"

**Clarify:** "What's the volume? Do we need every message durably, or is dropping under extreme load
OK? Human-readable or is binary fine for later decode? One thread or many producing?" — *Assume: very
high volume, dropping-under-overload is acceptable (never block the trader), multiple producer
threads.*

**Design (reference §8):** per-producer-thread **lock-free ring buffer**; on the hot path write only a
**binary record** (timestamp + event id + raw args), no formatting, no lock, no I/O; a **background
consumer thread** drains all ring buffers, does **deferred formatting**, batches **writes to disk**.

**Data flow:** hot thread → `push(binary record)` (nanoseconds) → ring buffer → background thread
pops, formats, batch-writes.

**Bottlenecks avoided:** string formatting (deferred), locks (per-thread SPSC ring), syscalls/disk
(only the background thread, batched), allocation (preallocated ring).

**Trade-offs:** on buffer overflow, **drop or overwrite** rather than block — you prioritize the
trader over completeness (state this as a deliberate choice; if full durability is required, you pay
latency or need bigger buffers / backpressure). Binary-on-hot-path means logs need a decoder, which is
a small operational cost for a big latency win.

---

## 11. Self-test checklist (be able to walk through each, out loud, using the §1 framework)

- [ ] State the 6-step design framework from memory and apply it to a fresh prompt.
- [ ] Draw the end-to-end trading system and name each component's job + latency sensitivity.
- [ ] Define a limit order book: bids/asks, price levels, FIFO queues, BBO, spread.
- [ ] Design a fast order book: array-of-levels + intrusive lists + id→order map + object pool;
      give O(1) add/cancel/BBO; explain why `std::map` is too slow.
- [ ] List the order-book trade-offs (memory vs speed, sparse price range, hybrid window).
- [ ] Explain price-time priority and how a match consumes resting orders FIFO.
- [ ] Distinguish limit / market / IOC / FOK.
- [ ] Design a feed handler: UDP multicast, sequence numbers, gap detection, A/B arbitration,
      normalization, lock-free fan-out, off-path recovery.
- [ ] Explain how dropped packets are handled (A/B first, then recovery, mark stale).
- [ ] Describe pre-trade risk checks and *why they must be O(1) and fast*.
- [ ] Sketch an order state machine and why it's needed (async exchange).
- [ ] Recap the low-latency levers: SPSC queues, busy-poll, no-alloc, cache layout, kernel bypass,
      thread pinning — each with its *reason*.
- [ ] Design an async logger (binary on hot path, ring buffer, deferred formatting, drop-on-overflow).
- [ ] Decide what lives off the hot path and articulate latency-vs-reliability trade-offs.
- [ ] Design a kill switch: incremental PnL/position monitor, thresholds, enforce at the gateway,
      layers of defense, fail-safe bias.
- [ ] For ANY answer: open with clarifying questions, name the bottleneck, and state every trade-off.

> **Final reminder:** they're hiring an intern, not a staff engineer. Calm, structured reasoning —
> "let me clarify the requirements, here are the constraints, here's the hot path, here's the
> bottleneck, here's what I'd trade off" — beats a memorized architecture every time. Think out loud.
