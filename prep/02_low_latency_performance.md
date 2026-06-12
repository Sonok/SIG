# Low-Latency & Performance Engineering — SIG Trading System Engineering

## Why this matters for SIG

Susquehanna is a market maker. The core loop is: receive a market data packet, decide whether to quote/trade, and emit an order — faster and smarter than the other firm doing the exact same thing. When two firms react to the same signal, **the faster one gets the fill and the slower one gets adversely selected** (you only get filled when you're wrong). At that point you are no longer competing on cleverness; you are competing on nanoseconds. The entire trading-system-engineering discipline at SIG is about removing every avoidable nanosecond from the tick-to-trade path while keeping the system correct and observable.

This is the single most important file in your prep. Trading System Engineering at SIG ≈ low-latency C++ systems engineering. Almost everything you'll be asked in the Live Coding and Final Design rounds connects back to the ideas below. And critically: **you already do this at AWS** (cache locality, lock-free concurrency, tail-latency reduction in a data plane). Your job in the interview is to map that vocabulary onto HFT and speak about it with authority. There's a dedicated section below on exactly how to do that.

The mental model to carry into the room: *latency is a budget*. Tick-to-trade might be a few hundred nanoseconds to a couple microseconds end-to-end. Every cache miss (~100ns), every branch mispredict (~5-20ns), every page fault (microseconds), every malloc (hundreds of ns to microseconds), every context switch (microseconds) is a line item you can't afford. Performance engineering is the practice of accounting for that budget.

---

## The latency mindset

### Why microseconds and nanoseconds matter
In HFT the signal everyone reacts to (a price move, a new top-of-book quote) arrives at every fast participant at almost the same instant. The economic value of reacting decays *extremely* fast — often the edge is gone within microseconds. So the relevant question isn't "is my code fast?" it's "am I faster than the next firm reacting to this same event?" It's a race with a winner-take-most payoff.

### Latency vs throughput
- **Latency** = time for *one* operation to complete (tick-in to order-out). This is what wins the race.
- **Throughput** = operations per second the system can sustain.

They are different and often in tension. Batching improves throughput but adds latency (you wait to fill the batch). A queue with deep buffering has great throughput and terrible tail latency. In HFT you optimize the **latency of the hot path** first; throughput matters only insofar as you must not fall behind the market data feed. A useful framing: "I care about how fast a *single* message gets through the system under load, not how many messages per second I can average."

### The cost of being slow
- **Adverse selection:** Your resting quote is stale. The market moved, a faster trader picks you off at the old (now bad) price before you can cancel. You systematically get filled only when the trade is bad for you. This is *the* cost of latency for a market maker.
- **Missing fills / queue position:** On price-time-priority venues, being slower means you're behind in the queue and don't get filled on the trades you *did* want.
- **Wider quotes:** If you can't react fast, you defend yourself by quoting wider (less competitive), which means less volume and less edge. Speed lets you quote tighter safely.

> 🎤 **If they ask: "Why does latency matter so much in trading?"**
> "Market makers race to react to the same public signal. If I'm slower, I get adversely selected — my resting quotes get picked off at stale prices by faster traders before I can cancel, and I lose queue position on the fills I actually want. So I'm not really competing on average speed, I'm competing on whether I beat the next firm to the same event by nanoseconds. That's why we obsess over the *tail* of the latency distribution, not the average — the slow tick is the one that costs money."

---

## CPU cache hierarchy & the latency numbers

Modern CPUs are fast; memory is slow. The whole game of cache-friendly code is keeping the data the CPU needs close to the core. Each level up the hierarchy is bigger but slower.

### Latency Numbers Every Programmer Should Know
(Canonical order-of-magnitude figures — exact numbers vary by CPU, but the *ratios* are what matter and what you should quote.)

| Operation | Approx latency | In cycles (~3 GHz) | Intuition |
|---|---|---|---|
| 1 CPU cycle | ~0.3 ns | 1 | the clock tick |
| Register access | ~0.3 ns | ~1 | basically free |
| L1 cache hit | ~1 ns | ~3-4 | the data you just touched |
| L2 cache hit | ~4 ns | ~12-14 | recently touched |
| L3 cache hit (shared) | ~10-20 ns | ~40 | shared across cores |
| Main memory (DRAM) access | ~100 ns | ~300 | a *cache miss* costs you this |
| Branch mispredict | ~5-20 ns | ~15-20 | pipeline flush |
| Mutex lock/unlock (uncontended) | ~15-25 ns | — | |
| Same-core context switch | ~1-5 µs | — | 1000s of ns |
| Main memory vs L1 | **~100x** | — | the headline ratio |
| SSD random read | ~16 µs | — | |
| Network round trip same datacenter | ~0.5 ms = 500 µs | — | |
| Network round trip cross-country | ~40-70 ms | — | physics (speed of light) |

The two numbers to have on your tongue: **L1 ≈ 1 ns, main memory ≈ 100 ns**. A cache miss is ~100x slower than a hit. That single fact justifies almost every data-layout optimization in this guide.

- **Registers:** Inside the core, ~1 cycle. The compiler's job (register allocation) is to keep hot values here.
- **L1** (~32-64 KB, per-core, split I/D): ~1 ns. Your hot loop's working set should fit here.
- **L2** (~256 KB-1 MB, per-core): ~4 ns.
- **L3 / LLC** (several-tens of MB, shared across cores): ~10-40 ns. Shared means cross-core traffic and coherence live here.
- **DRAM:** ~100 ns, and that's *latency* — bandwidth is plentiful, latency is the killer for pointer-chasing code.

> 🎤 **If they ask: "Walk me through the memory hierarchy and rough latencies."**
> "Registers and L1 are about a nanosecond, L2 a few nanoseconds, L3 ten to forty, and a trip to main memory is around a hundred nanoseconds — roughly 100x an L1 hit. So a cache miss on the hot path can cost more than the rest of my logic combined. That ratio is why I lay data out so the bytes I need next are already in cache: I'd rather do more arithmetic on data that's hot than do less work but chase a pointer into DRAM."

---

## Cache lines, locality, and data layout

### Cache lines (64 bytes)
Memory moves between DRAM and cache in fixed-size blocks called **cache lines — 64 bytes** on essentially all modern x86/ARM. You never load one byte; you load the whole 64-byte line containing it. Consequences:
- Touching one field pulls in its 63 neighbors for free → put fields used together *next to each other*.
- Two unrelated variables that happen to share a line get tangled together in the coherence protocol (see false sharing).

### Spatial vs temporal locality
- **Spatial locality:** if you access address X, you'll likely access X+1 soon. Sequential array traversal is the ideal — the hardware prefetcher sees the pattern and pulls the next lines in before you ask.
- **Temporal locality:** if you access X now, you'll likely access X again soon. Keep frequently reused data resident; don't blow the cache with a giant scan in between.

### Cache-friendly data layout
- Prefer **contiguous arrays** (`std::vector`, `std::array`) over node-based structures (`std::list`, `std::map`, pointer trees) on the hot path. A linked list is a *cache-miss generator*: every `->next` is a likely miss.
- A linear scan of a `vector` can crush a "theoretically better" O(log n) tree lookup at realistic sizes purely because it's prefetcher-friendly and cache-resident.
- Pack hot fields together; push cold fields (rarely-touched metadata, debug strings) out of the hot struct or to the end so they don't waste cache line space.

> 🎤 **If they ask: "Why might a vector beat a linked list even when Big-O says otherwise?"**
> "Big-O counts operations but ignores the constant factor of where the data lives. A vector is contiguous, so traversal is sequential and the hardware prefetcher streams the next cache lines in ahead of time — most accesses hit L1/L2. A linked list scatters nodes across memory, so each `next` pointer is likely a cache miss at ~100ns. For realistic sizes the vector wins by a wide margin even with 'worse' asymptotics, because we're memory-latency-bound, not operation-bound."

---

## Array of Structs (AoS) vs Struct of Arrays (SoA) — data-oriented design

```cpp
// AoS — Array of Structs
struct Order { double price; int qty; int id; char side; /* +cold fields */ };
std::vector<Order> orders;

// SoA — Struct of Arrays
struct Orders {
    std::vector<double> price;
    std::vector<int>    qty;
    std::vector<int>    id;
    std::vector<char>   side;
};
```

If a hot loop only needs `price` for every order, **AoS** wastes bandwidth: each 64-byte line you load is mostly `qty`/`id`/`side`/cold fields you don't use. **SoA** stores all prices contiguously, so every byte loaded is a price you actually want — denser cache usage, better prefetching, and it vectorizes cleanly (SIMD can chew through a flat `price[]`).

**Data-oriented design (DOD)** is the broader philosophy: *design around the data and how it's accessed in bulk, not around objects.* Ask "what transformations run over what data, in what order?" and lay memory out to make those transformations sequential cache hits. The opposite is naive OOP where each object scatters its own little cluster of fields and the program pointer-chases between them.

- **Use AoS** when you touch most fields of each element together (good locality per element).
- **Use SoA** when hot loops sweep a single field across many elements (analytics, risk sweeps, SIMD).

> 🎤 **If they ask: "AoS vs SoA — when and why?"**
> "It's about which bytes a hot loop actually consumes. AoS keeps an object's fields together, which is great when I process one element at a time using most of its fields. SoA stores each field in its own contiguous array, which wins when a hot loop sweeps one field across many elements — every cache line is 100% useful data and it vectorizes well. So if I'm summing or filtering on just `price` over a million orders, I'd go SoA. It's the core idea of data-oriented design: lay out memory for the access pattern, not for the object model."

---

## False sharing

**False sharing** is when two cores write to two *different* variables that happen to live on the *same 64-byte cache line*. Even though the variables are logically independent, the cache coherence protocol (MESI) treats the line as one unit. Every write by core A invalidates the line in core B's cache and vice versa, so the line "ping-pongs" between cores over the interconnect. Each access that should have been a ~1ns L1 hit becomes a coherence miss of tens to ~100+ ns. It silently destroys the scaling of otherwise lock-free, per-thread code.

Classic example: an array of per-thread counters, `int counters[N]`, where each thread bumps its own slot. The slots share lines → false sharing → your "parallel" code runs slower than single-threaded.

**Fix: pad/align so each independently-written variable owns its own cache line.**

```cpp
#include <new>      // std::hardware_destructive_interference_size (often 64)
constexpr size_t CL = 64;

struct alignas(CL) PaddedCounter {
    std::atomic<long> value{0};
    char pad[CL - sizeof(std::atomic<long>)];   // fill out the line
};
PaddedCounter counters[N];   // each counter now on its own line
```

`alignas(64)` ensures each element starts on a fresh cache line; the padding ensures it *occupies* a whole line so the next element can't share it. C++17 gives you `std::hardware_destructive_interference_size` for the magic number.

> 🎤 **If they ask: "What is false sharing and how do you fix it?"**
> "False sharing is when two cores write to two different variables that sit on the same 64-byte cache line. The variables are independent, but cache coherence works at line granularity, so each write invalidates the other core's copy and the line ping-pongs between cores over the interconnect — turning ~1ns L1 hits into ~100ns coherence misses. It silently kills the scaling of per-thread/lock-free code. The fix is to pad and align each hot, independently-written variable to its own cache line with `alignas(64)` plus padding, so no two cores ever contend on the same line."

---

## Branch prediction & branchless programming

The CPU pipeline is deep (15-20+ stages) and runs many instructions in flight. At a conditional branch it can't wait to know the outcome, so it **predicts** and speculatively executes down the predicted path. Predict right → effectively free. Predict wrong → **pipeline flush**: discard the speculative work and restart, costing ~5-20ns (~15-20 cycles).

- **Predictable branches are cheap.** A condition that's almost always true (or follows a regular pattern) the predictor nails ~99%+ of the time.
- **Unpredictable / data-dependent branches are expensive.** A branch that's true/false ~50/50 based on random data mispredicts constantly. The famous "sorting an array makes the loop faster" result is exactly this: sorted data makes the branch predictable.

### Branchless programming
Replace data-dependent branches with arithmetic / conditional moves so there's no branch to mispredict:
```cpp
// branchy
int m = (a > b) ? a : b;
// branchless-friendly: compiler may emit cmov; or explicit bit tricks
int m = b ^ ((a ^ b) & -(a > b));   // max without a branch
```
Often you don't hand-write this; you structure code so the compiler emits a `cmov`. Branchless isn't always faster (it does both sides' work) — it wins specifically when the branch is *unpredictable*.

### Hinting the predictor
```cpp
if (__builtin_expect(error, 0)) { /* cold error path */ }   // GCC/Clang
if (likely(x)) { ... }                                      // common macro wrapper
// C++20:
if (cond) [[likely]]   { hot_path(); }
else      [[unlikely]] { cold_path(); }
```
These tell the compiler which path is hot so it lays out code to keep the hot path straight-line (fewer taken branches, better I-cache locality) and pushes cold code out of line.

> 🎤 **If they ask: "What's the cost of a branch mispredict and how do you avoid it?"**
> "A mispredict flushes the pipeline — roughly 15-20 cycles, call it 5-20ns of wasted work. The fix is to make branches predictable or remove them. On the hot path I mark the error/rare paths `[[unlikely]]` (or `__builtin_expect`) so the compiler keeps the common path straight-line, and for genuinely 50/50 data-dependent decisions I go branchless — let the compiler emit a `cmov` or use arithmetic — so there's no branch to mispredict. Branchless does both sides' work though, so it only pays off when the branch is actually unpredictable."

---

## Hot path vs cold path

The **hot path** is the code that runs on every message in the latency-critical loop (parse market data → update book → risk check → emit order). The **cold path** is everything else: error handling, logging, reconnection, startup, config reload, end-of-day.

Rules:
- Keep the hot path **tight and straight-line**: no allocations, no locks, no virtual dispatch, no syscalls, minimal branching, minimal data touched.
- Push everything possible to the cold path. Errors and logging go through `[[unlikely]]` branches and out-of-line functions so they don't bloat the hot path's instruction footprint (I-cache matters too).
- Do expensive work *off* the hot path: preallocate, precompute, warm caches at startup, not while a tick is in flight.

> 🎤 **If they ask: "What do you mean by the hot path?"**
> "It's the code that executes on every message in the latency-critical loop — for a trading system, tick-to-trade. I keep it as tight as possible: no heap allocation, no locking, no virtual calls, no syscalls, predictable branches, and a small working set that stays in cache. Everything non-essential — logging, error recovery, reconnection — gets pushed to the cold path behind unlikely branches and out-of-line functions, so it never bloats the instructions or data the hot path touches."

---

## Memory allocation on the hot path

`malloc`/`new` are **forbidden on the hot path**. Why:
- Non-deterministic cost: a fast path of a few hundred ns, but it may take a lock, walk free lists, hit the allocator's metadata (cache misses), or fall into the kernel (`mmap`/`brk`) for microseconds.
- Causes **jitter** — the occasional slow allocation is exactly the p99.9 tail spike you're trying to kill.
- Can fragment, can page-fault on first touch.
- `free`/`delete` and destructors are also non-trivial.

**Techniques to avoid it:**
- **Preallocation:** allocate all buffers/objects up front at startup; reuse them forever.
- **Object pools / free lists:** a fixed set of preallocated objects; "allocate" = pop from a pool, "free" = push back. O(1), no syscall, cache-resident.
- **Arena / bump allocator:** one big preallocated block; allocation is just `ptr += size` (a pointer bump). Free everything at once by resetting the pointer (great for per-message scratch that's discarded each tick).
- **Stack allocation / small-buffer optimization:** keep transient data on the stack; use `std::array` or fixed buffers instead of `std::vector` where size is bounded.
- **`reserve()`** containers once so they never reallocate mid-loop.

> 🎤 **If they ask: "Why is allocating on the hot path bad?"**
> "`new`/`malloc` is non-deterministic — usually a few hundred nanoseconds, but it can take a lock, miss cache walking allocator metadata, or drop into the kernel for microseconds. That variance is exactly what blows up my tail latency. So on the hot path I never allocate: I preallocate everything at startup and reuse it via object pools or a bump/arena allocator, and I `reserve()` containers so they don't reallocate. Allocation, if it has to happen, happens on the cold path."

---

## Inlining & function-call overhead

A function call isn't free: set up arguments, push a stack frame, jump (possible I-cache miss + branch), execute, return, tear down. For a tiny function called in a tight loop that overhead can dwarf the actual work, and the call boundary also **blocks optimization** — the compiler can't see across an un-inlined call to keep values in registers, fold constants, or vectorize.

**Inlining** pastes the callee's body into the caller, removing call overhead *and* opening up cross-boundary optimization (constant propagation, dead-code elimination, vectorization). This is why small hot helpers should be inlinable.

- `inline` is a *hint/linkage* keyword; the compiler decides. `[[gnu::always_inline]]` / `__attribute__((always_inline))` forces it.
- Header-defined / template code is visible to the compiler and inlines naturally.
- **Virtual functions normally can't be inlined** (the target isn't known until runtime) — a key reason to avoid `virtual` on the hot path (see CRTP).
- Inlining isn't free: too much bloats the I-cache and can hurt. It's a balance.

> 🎤 **If they ask: "Why does inlining matter for latency?"**
> "Two reasons. First, it removes call overhead — frame setup, the jump, the return — which matters for tiny functions in tight loops. Second, and bigger, it lets the compiler optimize across the call boundary: keep values in registers, propagate constants, vectorize. An un-inlined call is an optimization barrier. That's also why virtual calls hurt on the hot path: the target's unknown at compile time so they can't inline, and you eat an indirect call plus a possible mispredict every time."

---

## Mechanical sympathy: pipelining, OoO & speculation

"**Mechanical sympathy**" (Martin Thompson's term, borrowed from racing) = writing software that works *with* the hardware's grain instead of against it. You don't need to fight the CPU; you need to feed it the way it wants to be fed.

- **Pipelining:** the CPU overlaps stages (fetch/decode/execute/...) of many instructions so it retires ~several per cycle. Branches and data dependencies stall the pipeline.
- **Out-of-order (OoO) execution:** the core reorders independent instructions to hide latency — while one instruction waits on a memory load, it executes others that don't depend on it. Code with lots of *independent* work gives the OoO engine room to hide stalls; long dependency chains serialize it.
- **Speculative execution:** the core runs ahead past unresolved branches on the predicted path, committing results only if the prediction holds.
- **Superscalar + SIMD:** multiple execution ports run several instructions per cycle; SIMD does the same op on a vector of data.

Practical takeaways: keep working sets in cache (so OoO has data to work with), avoid long serial dependency chains, make branches predictable, prefer data layouts the prefetcher loves. That *is* mechanical sympathy.

> 🎤 **If they ask: "What's mechanical sympathy?"**
> "It's writing code that cooperates with how the CPU actually works rather than fighting it. The core pipelines and reorders instructions and speculates past branches to stay busy — but cache misses, mispredicts, and long dependency chains stall it. So I lay data out contiguously so the prefetcher feeds it, keep the working set in cache, make branches predictable, and expose independent work so out-of-order execution can hide latency. Same idea as my AWS work: I'm shaping the program so the hardware can do what it's already good at."

---

## TLB, page faults, huge pages, memory warming

Virtual addresses are translated to physical via page tables; the **TLB** (Translation Lookaside Buffer) caches recent translations. A TLB miss costs a page-table walk (extra memory accesses); a **page fault** (page not yet mapped to physical RAM) traps into the kernel and costs *microseconds* — catastrophic on the hot path.

- **First-touch faulting:** freshly allocated memory isn't backed by physical pages until you write it. So a buffer "allocated" at startup still faults on first use mid-trade unless you **pre-touch / warm** it.
- **Memory warming:** at startup, write to every page of your preallocated buffers (and run the hot path with dummy data) so pages are resident, TLB is warm, and caches/branch predictors are primed *before* real traffic. You pay the faults during warmup, not during trading.
- **Huge pages** (2 MB / 1 GB instead of 4 KB): one TLB entry covers far more memory → fewer TLB misses and fewer page-table walks for large working sets. Common for big preallocated arenas/order books.
- **`mlock`/`mlockall`:** pin pages in RAM so they're never swapped out (swap on the hot path = disaster).

> 🎤 **If they ask: "What's a page fault and why do you warm memory?"**
> "A page fault is when you touch virtual memory that isn't backed by a physical page yet — it traps into the kernel and costs microseconds. The problem is freshly allocated memory faults on *first touch*, so a buffer I 'allocated' at startup can still fault during a live trade. So I warm it: at startup I write to every page and run the hot path with dummy data, which makes the pages resident, warms the TLB and caches, and primes the branch predictor. I also use huge pages to cut TLB misses and `mlock` to keep pages from being swapped. I pay all that cost before the market opens, not during a tick."

---

## NUMA basics & pinning

On multi-socket servers, each CPU socket has its own attached memory. Accessing your *local* NUMA node is fast; reaching across the interconnect to *remote* memory on another socket is meaningfully slower (and lower bandwidth). That's **NUMA — Non-Uniform Memory Access**.

For low latency:
- **Pin the thread and its memory to the same NUMA node.** Allocate the buffers a thread uses on the node where that thread runs (Linux first-touch policy: a page is placed on the node of the thread that first writes it — so warm memory on the right thread).
- Keep the hot path (thread + data + NIC) on one node; many setups put the NIC's PCIe lanes on a specific socket and pin the networking thread there to avoid cross-socket hops.
- `numactl`, `libnuma`, and CPU/memory affinity APIs control placement.

> 🎤 **If they ask: "What is NUMA and why care?"**
> "On multi-socket boxes each socket has local memory; reaching memory attached to another socket goes over the interconnect and is slower with less bandwidth. For low latency I pin a thread to a core and make sure the memory it touches lives on that same NUMA node — Linux places a page on the node that first writes it, so I warm memory from the right thread. I also keep the NIC, the networking thread, and its buffers on the same socket so a packet never has to hop across the interconnect on the hot path."

---

## CPU affinity, core isolation, spinning vs blocking

The OS scheduler will happily migrate your hot thread between cores, preempt it for other work, and let interrupts land on it — every one of those is a latency spike. For determinism you take the core *away* from the OS.

- **CPU affinity / pinning** (`pthread_setaffinity_np`, `taskset`): bind the hot thread to one specific core so it never migrates (migration = cold caches/TLB on the new core).
- **Core isolation** (`isolcpus`, `nohz_full`, `irqaffinity`): tell the kernel to keep its scheduler, timer ticks, and IRQs *off* those cores. The hot thread effectively owns the core — no other process, no scheduler tick, no interrupt jitter.
- **Avoid context switches:** they cost ~1-5µs *and* trash your caches/TLB (cold restart). On an isolated, pinned, busy-polling core there's nothing to switch to.

### Busy-polling (spinning) vs blocking
- **Blocking** (sleep on a condition variable, `epoll`, interrupt-driven wakeup): CPU-efficient, but waking a sleeping thread costs microseconds (scheduler + context switch + cold caches). Great for throughput / cool paths, bad for latency.
- **Busy-poll / spin:** the thread loops reading the queue/NIC in a tight loop, never sleeping, so it reacts in *nanoseconds* the instant data arrives. It burns a whole core at 100% and wastes power — but on the hot path that's exactly the trade you want. HFT systems spin.

```cpp
// busy-poll a lock-free SPSC queue, no syscall, no sleep
while (running.load(std::memory_order_relaxed)) {
    if (Msg* m = queue.try_pop())   // hot: usually true under load
        handle(*m);
    // optional: _mm_pause();  // hint on hyperthreaded cores
}
```

> 🎤 **If they ask: "Spinning vs blocking — when do you spin?"**
> "Blocking parks the thread and is CPU-efficient, but the wakeup costs microseconds — scheduler, context switch, and cold caches. Spinning keeps the thread hot in a tight poll loop so it reacts in nanoseconds the moment data lands, at the cost of burning a full core. On the latency-critical hot path I spin, and I make it deterministic: pin the thread to an isolated core with `isolcpus`/`nohz_full` so the kernel keeps its scheduler tick and interrupts off that core — nothing preempts it, nothing migrates it, caches stay warm. Off the hot path I'd block to save the core."

---

## Kernel bypass & user-space networking

A normal packet path is expensive: NIC interrupt → kernel network stack (driver, IP/TCP processing) → copy from kernel buffer to user buffer → syscall (`recv`) with its user/kernel transition → schedule your thread. That's microseconds and a pile of jitter, all in the kernel, none of it under your control.

**Kernel bypass** maps the NIC directly into user space so packets go straight to your application, skipping the kernel stack, the copies, the syscalls, and the interrupts:
- **DPDK:** poll-mode driver, the app owns the NIC and busy-polls it from user space — no interrupts, no kernel, zero-copy.
- **Solarflare / Onload (now AMD/Xilinx):** a user-space TCP/IP stack; `onload` transparently accelerates sockets. Very common in HFT.
- The pattern: **zero-copy + poll-mode + user-space stack** → tick-to-trade in the low microseconds / sub-microsecond range.

This is exactly the territory of **data-plane** networking — and where your AWS experience plugs in directly (see next section).

> 🎤 **If they ask: "What's kernel bypass and why does it help?"**
> "The normal path — NIC interrupt, kernel TCP/IP stack, a copy into user space, and a `recv` syscall — adds microseconds and a lot of jitter, and none of it is under my control. Kernel bypass maps the NIC into user space so packets go straight to the application: no kernel stack, no copies, no syscalls, and instead of interrupts the app poll-mode-drives the NIC. DPDK does this with a poll-mode driver; Solarflare Onload gives you a user-space TCP stack under the normal socket API. It's the standard way HFT gets tick-to-trade into the low microseconds. It's the same zero-copy, poll-mode, user-space data-plane mindset I work in at AWS."

---

## How to talk about your AWS work in this interview

You do low-latency C/C++ networking at AWS — cache locality, lock-free concurrency, tail-latency reduction in a data plane. That is *enormous* signal for SIG; most candidates have only academic exposure. Your job is to translate, not to overclaim. A few moves:

**1. Lead with the shared vocabulary.** Say the words: "data plane," "hot path," "tail latency," "p99.9," "lock-free," "cache locality," "zero-copy," "busy-poll." These map 1:1 onto HFT and signal you've actually shipped this.

**2. Map the analogy explicitly.**
- *Data plane ↔ tick-to-trade hot path.* "At AWS the data plane is the per-packet hot path — same shape as tick-to-trade: a tight loop on every message where allocations, locks, and cache misses are the enemy."
- *Tail-latency reduction ↔ adverse selection.* "We obsessed over p99.9, not averages, because the slow tail is what hurts. In trading the slow tail is the tick where you get picked off — same principle, higher stakes."
- *Lock-free concurrency ↔ hot-path lock-free queues.* Describe a real structure (SPSC/MPSC ring buffer, atomics, memory ordering) and the false-sharing padding you did.
- *Kernel/networking ↔ kernel bypass.* "We worked close to the data plane to cut stack overhead and copies — conceptually the same motivation as DPDK/Onload in HFT."

**3. Tell one concrete story (STAR-ish).** Pick a real win: a tail-latency reduction or a lock-free/cache-locality change. Situation (where the latency spike came from), what you measured (percentiles, a profiler/perf, rdtsc), the change (e.g. removed an allocation, padded for false sharing, restructured for cache locality, replaced a lock with an atomic), and the *measured* result (p99/p99.9 dropped by X). Quantify. Interviewers love a number tied to a mechanism.

**4. Bridge to HFT honestly.** "I haven't worked on a matching engine, but the engineering is the same discipline: keep the hot path allocation-free and lock-free, lay data out for cache, pin and isolate cores, busy-poll, and measure the tail. I'd be coming in already fluent in those tools."

**5. Don't oversell specifics you didn't do.** If you didn't personally write the DPDK driver, say "conceptually similar to" — credibility is the whole game and these interviewers will probe.

> 🎤 **One-liner to open with:** "At AWS I work on a low-latency C++ data plane — the per-packet hot path — where I do cache-locality and lock-free work and drive down tail latency at p99.9. That's the same engineering as a trading hot path; I'm excited to apply it where the latency budget is even tighter."

---

## Measuring latency properly

You cannot optimize what you measure wrong, and **most people measure latency wrong** by reporting averages.

### Percentiles & why averages lie
- Latency distributions are **right-skewed with fat tails** — a few very slow events, lots of fast ones. The **mean** is dragged around by outliers and hides them; it tells you nothing about the worst case.
- Report **percentiles**: **p50** (median, typical), **p99** (1 in 100), **p99.9** (1 in 1000), **p99.99** (1 in 10,000). In trading the tail *is* the product — the slow tick is the one that loses money — so p99/p99.9/p99.99 are what matter.
- "Average latency is fine" + "p99.9 is 50x the median" = a system that periodically gets picked off. The average literally cannot show you that.
- Beware **coordinated omission**: if your load generator waits for a response before sending the next request, it under-samples exactly the slow periods and *understates* the tail. (Tools like HdrHistogram/wrk2 correct for this.)

### Jitter
**Jitter** = variability/spread of latency (the *consistency*, not the speed). Low and consistent often beats low-on-average-but-spiky, because spikes are the adverse-selection events. Sources: GC/allocation, page faults, context switches, interrupts, NUMA hops, thermal/frequency scaling, false sharing.

### Warmup & steady-state
- The **first** iterations are unrepresentative: cold caches, cold TLB, cold branch predictor, JIT-equivalents, lazy faulting, CPU still ramping frequency.
- Always **warm up** (run the path many times), discard warmup samples, and measure **steady-state**. This mirrors production where the system has been running.

### High-res timing
- **`rdtsc` / `rdtscp`** (read time-stamp counter): cycle-granular, very low overhead — the standard for nanosecond timing of short code regions. Use `rdtscp` (or a fence) so the read isn't reordered. Convert cycles→ns via the invariant TSC frequency.
- `std::chrono::steady_clock` / `clock_gettime(CLOCK_MONOTONIC)` for portability; monotonic so it won't jump.
- Watch out: `rdtsc` can be reordered (serialize it), and TSC must be invariant across frequency scaling on modern CPUs (it is on recent chips).

### Microbenchmark pitfalls
- The compiler may **optimize away** the work you're timing (dead-code elimination) → use `benchmark::DoNotOptimize` / a `volatile` sink so results are "used."
- Measuring in a tight loop gives unrealistically warm caches/predictors vs production.
- Timer overhead can dominate tiny measurements — measure a batch and amortize.
- Frequency scaling / turbo / thermal throttling skews runs — pin frequency.
- Use a real harness (**Google Benchmark**) and look at the *distribution*, not one number.

> 🎤 **If they ask: "Why measure p99 instead of the average?"**
> "Because latency distributions are fat-tailed and the mean hides the tail — a handful of slow events get averaged away. In trading the tail *is* what matters: the slow tick is the one where I get adversely selected, so I track p99, p99.9, even p99.99. A system with a great average and a 50x p99.9 spike is a system that periodically gets picked off, and the average can't show me that. I also watch for coordinated omission so my load test doesn't quietly under-sample the slow periods."

> 🎤 **If they ask: "How do you time nanosecond-scale code?"**
> "I use `rdtscp` for cycle-granular, low-overhead timestamps around the region, with a serializing read so it isn't reordered, and convert cycles to nanoseconds with the invariant TSC frequency. I warm up first and measure steady-state, discard the cold runs, and report the distribution — p50/p99/p99.9 — not a single average. And I guard against the compiler optimizing the work away with a `DoNotOptimize`-style sink, and amortize timer overhead by measuring batches. For longer regions `clock_gettime(CLOCK_MONOTONIC)` is fine."

---

## Compiler optimizations

The compiler is doing enormous work for you; understand it.
- **`-O2`** is the standard production level: inlining, constant folding/propagation, dead-code elimination, loop optimizations, register allocation, some vectorization. Safe and aggressive.
- **`-O3`** adds more aggressive vectorization/loop unrolling/inlining. Sometimes faster, sometimes *slower* (I-cache bloat) — **measure**, don't assume bigger number = faster.
- **`-march=native`** lets it use your CPU's full instruction set (AVX2/AVX-512 etc.) — big for SIMD, but binaries aren't portable across CPU generations.
- **LTO (link-time optimization)** lets inlining/optimization cross translation-unit boundaries.
- **PGO (profile-guided optimization)** feeds real execution profiles back so the compiler optimizes the actually-hot paths and predicts branches correctly. Used in serious HFT builds.
- **`-ffast-math`** relaxes IEEE float rules for speed — be careful, it changes results.

**Reading what the compiler did:** disassemble (`objdump -d`, or **Compiler Explorer / godbolt.org**) to see if your hot function inlined, vectorized, emitted `cmov` instead of a branch, etc. Knowing godbolt is a strong signal in interview.

**When to trust/distrust it:** trust it for the bulk; verify the *hot path*. The compiler can't fix a bad algorithm or a cache-hostile data layout, and it can be blocked by aliasing, un-inlined calls (incl. virtual), or volatile. If a hot loop didn't vectorize, find out why (data dependency? aliasing? `__restrict__`?).

> 🎤 **If they ask: "-O2 vs -O3?"**
> "-O2 is the safe, aggressive default — inlining, constant folding, dead-code elimination, register allocation, loop opts. -O3 piles on more vectorization and unrolling, which sometimes helps and sometimes hurts by bloating the I-cache, so I always *measure* rather than assume. For latency builds I'd also consider `-march=native` for the full SIMD instruction set, LTO for cross-TU inlining, and PGO so the compiler optimizes and predicts the genuinely hot paths. And I check the actual assembly on Compiler Explorer to confirm the hot function inlined and vectorized."

---

## SIMD / vectorization

**SIMD** = Single Instruction, Multiple Data: one instruction operates on a vector of values at once (e.g. AVX2 adds 8 floats, AVX-512 adds 16, in one op). For data-parallel work — summing/scaling an array of prices, computing across many instruments — this is a multi-x speedup.

- The compiler **auto-vectorizes** simple, contiguous, dependency-free loops (another reason SoA + contiguous data matters — vectorization loves flat arrays).
- For full control: **intrinsics** (`_mm256_add_ps`, etc.) or libraries (Highway, xsimd, std::simd).
- Requirements/enablers: contiguous data, no loop-carried dependencies, aligned data helps, `-march`/`-O3` to enable the ISA, `__restrict__` to defeat aliasing assumptions.
- Reality check: SIMD helps the *compute* part; if you're memory-latency-bound (pointer chasing) it won't save you — fix the data layout first.

> 🎤 **If they ask: "What is SIMD?"**
> "Single Instruction Multiple Data — one instruction processes a whole vector at once, like adding 8 or 16 floats in a single op with AVX. It's a big win for data-parallel work like sweeping a calc across an array of prices. The compiler auto-vectorizes simple contiguous loops, which is another reason I favor SoA and flat arrays; for hand control I'd use intrinsics. But it only helps the compute side — if I'm memory-latency-bound I fix the data layout first, then vectorize."

---

## Real techniques grab-bag (hot-path C++)

- **Minimize copies / move semantics:** pass by `const&`, return by value relying on RVO, use `std::move` to transfer ownership instead of deep-copying. Every avoided copy is avoided memory traffic.
- **`reserve()`** vectors/strings up front so they never reallocate-and-copy mid-loop.
- **Avoid `virtual` on the hot path:** an indirect vtable call can't inline and may mispredict. Use **CRTP** (static polymorphism) to get compile-time dispatch with no runtime cost. (See OOP/templates file for CRTP detail.)
- **Lock-free on the hot path:** SPSC/MPSC ring buffers with `std::atomic` and explicit memory ordering instead of mutexes; a contended lock means a syscall and a context switch — death for latency. Pad shared atomics against false sharing.
- **Cache warming / prefetch:** warm caches at startup; `__builtin_prefetch` to hint the next line when you know your access pattern ahead of time.
- **`alignas` / data packing:** align hot structures to cache lines; order struct members to minimize padding and group hot fields.
- **`noexcept`** where true — lets the compiler skip unwinding machinery and optimize harder.
- **`const` / `constexpr`** to push work to compile time.
- **`__restrict__`** to promise no aliasing so the compiler can vectorize/reorder.
- **Batch syscalls / avoid them** on the hot path entirely (no logging, no `printf`, no `malloc`).

> 🎤 **If they ask: "Give me concrete things you'd do to keep a hot path fast."**
> "No allocation — preallocate and use object pools or an arena. No locks — lock-free SPSC ring buffers with atomics and the right memory ordering, padded against false sharing. No virtual — CRTP for static dispatch so it inlines. No syscalls or logging on the path. Lay data out contiguously and SoA so it's cache- and prefetcher-friendly, `reserve()` containers, move instead of copy, mark things `noexcept`/`const`. Then pin the thread to an isolated core, busy-poll, warm the memory and caches at startup, and measure the tail with p99.9 to prove it."

---

## Rapid-fire Q&A

**1. Why does latency matter in HFT?**
Firms race to react to the same public signal; the slower one gets adversely selected — stale quotes picked off before they can cancel — and loses queue position. It's a winner-take-most race measured in nanoseconds, so we optimize the tail, not the average.

**2. Latency vs throughput?**
Latency is how long one operation takes (tick-to-trade); throughput is operations per second. They trade off — batching raises throughput but adds latency. HFT optimizes single-message latency, ensuring only that throughput keeps up with the feed.

**3. Roughly, the memory hierarchy latencies?**
Register/L1 ~1ns, L2 ~4ns, L3 ~10-40ns, main memory ~100ns. A cache miss is ~100x an L1 hit — that ratio drives all data-layout work.

**4. What's a cache line and why does it matter?**
The 64-byte unit memory moves in. You always load the whole line, so fields used together should sit together (good locality), and unrelated fields sharing a line cause false sharing.

**5. What is false sharing and how do you fix it?**
Two cores writing different variables on the same cache line; coherence ping-pongs the line between cores, turning ~1ns hits into ~100ns misses. Fix by padding/aligning each independently-written variable to its own line with `alignas(64)`.

**6. AoS vs SoA?**
Array-of-structs keeps each object's fields together (good when you use most fields per element); struct-of-arrays stores each field contiguously (good when a hot loop sweeps one field across many elements — denser cache use, vectorizable). Choose by access pattern.

**7. Why avoid allocating on the hot path?**
`new`/`malloc` is non-deterministic — can lock, miss cache, or hit the kernel for microseconds — causing tail-latency spikes. Preallocate and reuse via object pools or arena/bump allocators; `reserve()` containers.

**8. Cost of a branch mispredict and how to reduce it?**
~15-20 cycles (~5-20ns) pipeline flush. Make branches predictable, mark rare paths `[[unlikely]]`/`__builtin_expect`, and go branchless (cmov/arithmetic) when a branch is genuinely unpredictable.

**9. What's the hot path?**
The code on every message in the latency-critical loop. Keep it tight: no allocation, no locks, no virtual, no syscalls, predictable branches, small cache-resident working set; everything else goes to the cold path.

**10. Spinning vs blocking — when do you spin?**
Blocking is CPU-efficient but wakeups cost microseconds; spinning burns a core but reacts in nanoseconds. Spin on the latency-critical hot path, on a pinned, isolated core; block off the hot path.

**11. Why pin and isolate cores?**
To stop thread migration (cold caches), scheduler preemption, timer ticks, and interrupt jitter. `taskset`/affinity pins the thread; `isolcpus`/`nohz_full`/IRQ affinity give the thread sole ownership of the core for deterministic latency.

**12. What's a page fault and why warm memory?**
Touching virtual memory not yet backed by a physical page traps to the kernel for microseconds; fresh memory faults on first touch. Warm it at startup by writing every page (and running the path with dummy data) so pages, TLB, caches, and the predictor are primed before trading. Huge pages and `mlock` help.

**13. What is kernel bypass?**
Mapping the NIC into user space so packets skip the kernel stack, copies, syscalls, and interrupts (DPDK poll-mode driver, Solarflare Onload user-space stack). Cuts tick-to-trade to low microseconds — same data-plane mindset as my AWS work.

**14. Why measure p99 instead of average?**
Latency is fat-tailed; the mean hides the slow events that actually cost money (adverse selection). p99/p99.9/p99.99 expose the tail. Also guard against coordinated omission so the load test doesn't under-sample slow periods.

**15. Why avoid virtual functions on the hot path?**
The target is unknown at compile time, so the indirect vtable call can't be inlined and may mispredict, plus it blocks cross-call optimization. Use CRTP for compile-time (static) dispatch with zero runtime overhead.

**16. What's mechanical sympathy?**
Writing code that works with the CPU — contiguous cache/prefetcher-friendly data, predictable branches, independent work for out-of-order execution, warm caches — so pipelining, OoO, and speculation can do their job instead of stalling.

**17. How do you time nanosecond code correctly?**
`rdtscp` (serialized) for cycle-granular timing, warm up and measure steady-state, report the distribution (p50/p99/p99.9) not the average, prevent the compiler from optimizing the work away with a sink, and amortize timer overhead over batches.

---

## Self-test checklist

Can you, out loud and unprompted:

- [ ] Quote the latency numbers: L1 ~1ns, L2 ~4ns, L3 ~10-40ns, DRAM ~100ns, mispredict ~5-20ns, context switch ~µs — and state the ~100x L1-vs-DRAM ratio?
- [ ] Explain adverse selection and tie latency directly to losing money?
- [ ] Distinguish latency vs throughput and why HFT prioritizes the former?
- [ ] Define a cache line (64 B) and explain spatial vs temporal locality?
- [ ] Explain AoS vs SoA and pick the right one for a given access pattern?
- [ ] Define false sharing and write the `alignas(64)` + padding fix from memory?
- [ ] State the mispredict cost and give both a predictability fix and a branchless fix (`[[likely]]`, cmov)?
- [ ] Define hot path vs cold path and list what's banned on the hot path?
- [ ] Explain why `malloc` on the hot path is bad and name object pool / arena / bump allocator / `reserve()`?
- [ ] Explain why inlining matters (overhead + cross-boundary optimization) and why virtual blocks it?
- [ ] Explain mechanical sympathy: pipelining, OoO, speculation in plain terms?
- [ ] Explain page faults, first-touch, memory warming, huge pages, `mlock`?
- [ ] Explain NUMA and local-vs-remote memory pinning?
- [ ] Explain CPU affinity, core isolation (`isolcpus`/`nohz_full`), and spinning vs blocking?
- [ ] Explain kernel bypass (DPDK/Onload) and connect it to your AWS data-plane work?
- [ ] Explain why averages lie, what jitter is, and what p99/p99.9/p99.99 and coordinated omission mean?
- [ ] Describe correct measurement: `rdtscp`, warmup, steady-state, distributions, microbenchmark pitfalls?
- [ ] Explain -O2 vs -O3, LTO, PGO, `-march=native`, and that you read assembly on godbolt?
- [ ] Explain SIMD/vectorization at a high level and why SoA + contiguous data enable it?
- [ ] Deliver your AWS-to-HFT bridge with one concrete, quantified tail-latency story?
- [ ] Rattle off the hot-path techniques grab-bag (no alloc, lock-free + padding, CRTP, move, reserve, noexcept, prefetch, pin+isolate+spin, warm, measure)?
