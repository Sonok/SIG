# 05 — Operating Systems & Networking for Low-Latency Trading

> **Why this matters for SIG.** SIG's Trading System Engineering role is about moving information from the exchange wire into a decision and back out as an order *as fast as physically possible*. Every microsecond you "save" comes from understanding where time is spent: the kernel, context switches, page faults, the NIC, the network stack, TCP vs UDP, multicast feeds. You already work on this exact problem at AWS — kernel and userspace data paths *are* the data plane of a trading system. Your job in the interview is to map your AWS vocabulary (RSS, zero-copy, busy-poll, kernel bypass, interrupt coalescing) onto exchange connectivity. Interviewers love this topic because it separates people who "know C++" from people who understand *the machine the C++ runs on*. Aim to sound like someone who has profiled a data path, not someone who memorized definitions.

A recurring theme: **latency is about the tail, not the average.** A trading system that is fast 99% of the time but stalls for 2ms on a page fault or a context switch loses the trade. So most of OS knowledge here is framed as "how do I avoid the rare expensive event."

---

## OPERATING SYSTEMS

### 1. Processes vs Threads (recap — what's shared)

A **process** is an instance of a running program with its own virtual address space, file descriptor table, and OS resources. A **thread** is a unit of execution *within* a process.

- **Shared across threads in a process:** the virtual address space (heap, global/static data, code/text segment), open file descriptors, signal handlers, current working directory.
- **Private per thread:** the stack, the register set (including the program counter), thread-local storage (`thread_local`), and the kernel-scheduled thread state.

**Why it matters for latency:** threads share memory, so communication between them is just a memory write + synchronization — far cheaper than inter-process communication. But sharing means you pay for cache-coherence traffic and synchronization (locks, atomics). In HFT you often go *single-threaded on the hot path* precisely to avoid contention and cross-core cache bouncing, and use other cores only for non-critical work (logging, telemetry).

**🎤 If they ask "difference between a process and a thread?":**
"A process has its own address space and resources; threads live inside a process and share its heap, globals, and file descriptors, but each has its own stack and registers. Threads are cheaper to create and communicate, but sharing memory means you pay for synchronization and cache coherency. On a trading hot path I'd often keep it single-threaded to avoid lock contention and cross-core cache traffic, and offload logging or stats to other threads."

---

### 2. Context Switches and Why They Hurt Latency

A **context switch** is when the OS saves the state of one execution context (registers, program counter, stack pointer) and restores another. It happens on:
- **Preemption** (the scheduler's timer interrupt fires, time slice expired),
- **Blocking** (a thread calls a blocking syscall — `read`, `recv`, `futex` wait, `sleep`),
- **A higher-priority task becoming runnable**,
- An **interrupt** that wakes a different thread.

**Direct cost:** saving/restoring registers and switching the kernel's bookkeeping — on the order of **1–5 microseconds**.

**The real cost is indirect — cache and TLB pollution.** When you switch to another thread/process:
- Its working set isn't in L1/L2/L3, so you take **cold-cache misses** when you resume the hot thread.
- If you switch to a different *process*, the **TLB may be flushed** (or partially, with PCID/ASIDs), so you re-walk page tables.
- Branch predictor / prefetcher state is polluted.

So the *measured* latency hit of an unlucky context switch can be **tens of microseconds** once you count refilling caches — which in HFT terms is catastrophic.

**How HFT avoids context switches:**
- **Pin (affinitize) the hot thread to an isolated core** so nothing else runs there.
- **Busy-poll / spin** instead of blocking, so you never voluntarily yield and never call a blocking syscall.
- **Isolate the core from the kernel scheduler and IRQs** (`isolcpus`, `nohz_full`, IRQ affinity away from the core).
- Avoid `sleep`, condition variables, and mutexes on the hot path.

**🎤 If they ask "why are context switches bad for latency?":**
"The direct cost — saving and restoring registers — is only a microsecond or two. The expensive part is indirect: the cache and TLB are now cold for your hot thread, so when it resumes it eats a wave of cache and TLB misses, and the branch predictor is polluted. That can turn into tens of microseconds. So on the hot path we avoid switches entirely: pin the thread to an isolated core, busy-poll instead of blocking, and keep the kernel and other threads off that core."

---

### 3. Virtual Memory: Addresses, Page Tables, MMU, TLB, Page Faults

**Virtual vs physical addresses.** Every process sees a private, contiguous **virtual address space**. The CPU's **MMU (Memory Management Unit)** translates virtual addresses to **physical addresses** using **page tables** maintained by the OS. Memory is managed in **pages** (typically 4 KB).

**Page tables** are a multi-level tree (x86-64 uses 4 or 5 levels). A translation that isn't cached requires a **page-table walk** — several dependent memory accesses, each potentially a cache miss. That's slow, so the CPU caches recent translations in the…

**TLB (Translation Lookaside Buffer)** — a small, fast cache of virtual→physical translations inside the MMU.
- **TLB hit:** translation is essentially free.
- **TLB miss:** the hardware walks the page table (a few extra memory accesses), then fills the TLB. A miss costs maybe tens of nanoseconds, but in a tight loop over a large working set, misses add up and are a real, measurable latency source.

**Page faults** — raised when a virtual address has no valid physical mapping at access time:
- **Minor (soft) fault:** the page is in physical memory but not mapped into this process's page table yet (e.g., first touch of freshly `malloc`'d/`mmap`'d memory, or copy-on-write). The kernel just updates the mapping — microseconds.
- **Major (hard) fault:** the page must be fetched from disk/swap or a file. This is **milliseconds** — catastrophic on a hot path.

**Demand paging:** the OS lazily allocates physical pages only when first touched. Great for general computing, terrible for predictable latency, because the *first* touch of your buffers faults exactly when you're trying to be fast.

**Why page faults cause latency spikes:** a single major fault is ~1000x your normal trade latency budget. Even minor faults add jitter. The fault is handled by the kernel → mode switch + page-table manipulation + possibly I/O.

**How HFT avoids them:**
- **`mlock` / `mlockall(MCL_CURRENT | MCL_FUTURE)`** — pin pages in RAM so they're never swapped out (no major faults).
- **Pre-fault / pre-touch** all memory at startup (write a byte to every page) so minor faults happen during warm-up, not in the trading loop.
- **Huge pages (2 MB / 1 GB)** — one TLB entry covers 2 MB instead of 4 KB, drastically reducing TLB misses and page-table walk depth for large working sets. Also fewer page-table levels touched.
- **Disable swap** on trading hosts.
- **Pre-allocate and reuse** buffers/object pools instead of allocating on the hot path.

**🎤 If they ask "what is a TLB and why does a TLB miss hurt?":**
"The TLB caches virtual-to-physical address translations. On a hit, translation is free. On a miss, the hardware has to walk the multi-level page table — several dependent memory accesses that can themselves miss in cache — before it can even fetch your data. In a hot loop over a big working set, those misses pile up. We mitigate with huge pages, which let one TLB entry cover 2 MB instead of 4 KB, so far fewer entries and far fewer misses."

**🎤 If they ask "why do page faults matter in a trading system?":**
"A major page fault means going to disk or swap — milliseconds, which is a thousand times our latency budget. Even minor faults add jitter. So we eliminate them: `mlockall` to pin memory so nothing swaps, pre-touch every page at startup so first-touch faults happen during warm-up, and pre-allocate buffers so we never fault in the trading loop. Huge pages help too, both for TLB pressure and fewer faults."

---

### 4. CPU Scheduling & Getting Low-Latency Scheduling

The OS scheduler decides which runnable thread runs on which core. Linux's default (CFS — Completely Fair Scheduler) optimizes for fairness and throughput, **not** tail latency. Key concepts:

- **Preemption:** the scheduler can interrupt a running thread (timer interrupt every tick, or when a higher-priority thread becomes runnable) and switch it out. Good for fairness, bad for a thread that wants to run uninterrupted.
- **Time slice / quantum:** how long a thread runs before the scheduler reconsiders.
- **Priorities & scheduling classes:** `SCHED_OTHER` (normal), `SCHED_FIFO` / `SCHED_RR` (real-time, run until they yield/block or a higher-priority RT task preempts).

**Techniques for low-latency scheduling (this connects directly to your perf file):**
- **CPU pinning / affinity** (`pthread_setaffinity_np`, `taskset`): bind the hot thread to a specific core so it doesn't migrate (migration = cold cache on the new core).
- **Core isolation** (`isolcpus=`, `cpuset`, `nohz_full=`, `rcu_nocbs=`): remove cores from the general scheduler so *only* your pinned thread runs there. `nohz_full` stops the periodic timer tick on that core (no per-tick interrupt jitter).
- **IRQ affinity:** steer device interrupts away from the isolated core so the NIC IRQ doesn't preempt your strategy.
- **Real-time priority** (`SCHED_FIFO`): ensures your thread isn't preempted by normal tasks — but combined with busy-polling you must be careful not to starve everything else (use isolated cores).
- **Disable frequency scaling / C-states / turbo variance:** set the CPU governor to `performance`, disable deep C-states so the core doesn't go to sleep and pay a wake-up penalty when the next packet arrives.

The end state: **one core, one thread, spinning, never preempted, never interrupted, caches warm.**

**🎤 If they ask "how do you get deterministic low-latency scheduling on Linux?":**
"You take the hot thread off the general scheduler. Pin it to an isolated core with `isolcpus`/`cpuset`, use `nohz_full` to stop the timer tick on that core, steer NIC and other IRQs to different cores, and run it `SCHED_FIFO` so nothing preempts it. Combine that with busy-polling so it never blocks, pin the memory with `mlockall`, and set the governor to performance with deep C-states disabled. The goal is one thread spinning on a dedicated core with warm caches and zero interference."

---

### 5. System Calls: User Space vs Kernel Space, and Their Cost

Code runs in two privilege levels: **user space** (your application, restricted) and **kernel space** (the OS, full hardware access). A **system call** is the controlled gateway: when your app needs the kernel to do something privileged — read a socket, write a file, allocate memory, send a packet — it traps into the kernel.

**The cost of a syscall:**
- A **mode switch** (user→kernel→user) via the `syscall` instruction. This is *not* a full context switch (no thread swap), but it still costs on the order of **~100 ns to a few hundred ns** — and more if it touches caches/TLB or triggers scheduling.
- Post-Spectre/Meltdown mitigations (KPTI — page-table isolation) made syscalls more expensive because they may flush/switch page tables.
- A blocking syscall (`recv` on an empty socket) can cause a *real context switch* if it sleeps — much worse.

**Why minimizing syscalls matters:** on a hot path, every `recv`/`send` is a trip into the kernel. At millions of packets/sec, that overhead dominates. This is the entire motivation for **kernel bypass** (Section 11): get the data straight from the NIC into user space, no syscall per packet.

Tactics to reduce syscalls: batch I/O, use `recvmmsg`/`sendmmsg` (multiple messages per call), use `epoll` to avoid scanning many fds, memory-map files, and ultimately bypass the kernel for the NIC entirely.

**🎤 If they ask "what's the cost of a system call?":**
"It's a mode switch from user to kernel and back — not a full thread context switch, but still on the order of a hundred nanoseconds to a few hundred, more with Meltdown/Spectre page-table-isolation mitigations, and far more if it blocks and triggers an actual context switch. At packet-processing rates that overhead dominates, which is exactly why low-latency systems batch syscalls or bypass the kernel entirely so there's no per-packet syscall."

---

### 6. Memory: Stack vs Heap (OS View), mmap, brk/sbrk

- **Stack:** per-thread, grows downward, managed automatically (push/pop on function entry/exit). Allocation is a register adjustment — essentially free. The OS grows it via page faults as needed (the stack guard page). Limited size. Great for hot-path locals.
- **Heap:** process-wide, dynamically allocated (`malloc`/`new`). The allocator gets memory from the OS via `brk`/`sbrk` (moving the "program break" — the top of the data segment) for small requests, and via **`mmap`** for large requests. `malloc` then sub-allocates within those regions.
- **`mmap`:** maps pages into the address space — used for large allocations, memory-mapped files, and shared memory between processes. Anonymous `mmap` gives zeroed pages on demand (lazy — first touch faults).
- **`brk`/`sbrk`:** the classic way to grow the heap by extending the data segment. Largely an allocator implementation detail today; `mmap` is used for big chunks.

**Latency angle:** `malloc`/`new` on the hot path is dangerous — it can call into the kernel (`mmap`/`brk`), take page faults on first touch, take locks (in multithreaded allocators), and is unpredictable. HFT pre-allocates **object pools / arenas / ring buffers** at startup and reuses them, so the hot path does zero dynamic allocation. Custom allocators and `placement new` into pre-faulted memory are common.

**🎤 If they ask "stack vs heap, and why care on the hot path?":**
"Stack is per-thread, automatically managed, allocation is just a register move — fast and predictable. Heap is dynamic via malloc, which under the hood calls `brk` or `mmap`, can fault on first touch, and may take locks. On the hot path that unpredictability is unacceptable, so we pre-allocate pools and arenas at startup and reuse memory — ideally zero allocation in the trading loop."

---

### 7. Interrupts vs Polling

Two ways to learn that a device (e.g., a NIC) has data:
- **Interrupts:** the device raises an interrupt; the CPU stops what it's doing, runs an interrupt handler. Efficient when events are *rare* (CPU can do other work / sleep), but each interrupt costs a context switch into the handler and adds **jitter** — it preempts your hot thread.
- **Polling (busy-poll / spin):** the CPU repeatedly checks the device/ring for new data in a tight loop. Burns 100% of a core and 100% of power, but gives the **lowest and most deterministic latency** — no interrupt overhead, no wake-up delay, no scheduler involvement.

In HFT, the hot path **busy-polls** the NIC's receive ring (often via kernel-bypass like Solarflare onload or DPDK). You're trading a whole CPU core and electricity for nanoseconds and determinism — a trade worth making when a tick is worth money. Interrupt-driven I/O is fine for the cold path (config, logging).

There's also **interrupt coalescing** (the NIC batches several packets per interrupt to raise throughput) — good for throughput, *bad* for latency, so latency-sensitive setups reduce or disable coalescing (or just bypass the kernel and poll).

**🎤 If they ask "interrupts or polling for low latency?":**
"Polling. Interrupts are efficient when events are rare because the core can do other work, but each one is a context switch into a handler and it adds jitter by preempting your hot thread, plus there's wake-up latency. Busy-polling burns a whole core but gives the lowest, most deterministic latency — no interrupt, no scheduler, no wake-up. In HFT we spin on the NIC's RX ring, usually with kernel bypass. Interrupts are fine for the cold path."

---

## NETWORKING

### 8. The OSI / TCP-IP Stack (practical view)

You don't need to recite all seven OSI layers in order, but you should reason about them:

| Practical layer | What it does | Trading relevance |
|---|---|---|
| **Physical / Link (L1/L2)** | Bits on the wire, Ethernet frames, MAC addresses, switches | Cut-through switches, microwave/fiber links, NIC timestamps |
| **Network (L3)** | IP addressing, routing | IP multicast for market data |
| **Transport (L4)** | TCP / UDP — ports, reliability, ordering | The big TCP-vs-UDP decision |
| **Application (L7)** | Exchange protocols (FIX, ITCH, OUCH, proprietary binary) | Parsing market data, order entry |

The key practical model: **app → transport (TCP/UDP) → IP → Ethernet → NIC → wire**, and back up on receive. Every layer adds headers (encapsulation) and processing. In the kernel, each layer is code that touches your packet — and **kernel bypass collapses these layers into a thin user-space path.**

**🎤 If they ask "walk me through the network stack":**
"From the app down: the application protocol (say a binary market-data or FIX message), then transport — TCP or UDP — which adds ports and, for TCP, sequencing and reliability, then IP for addressing and routing, then Ethernet framing, then the NIC puts it on the wire. Each layer adds a header and costs processing time in the kernel. Latency-sensitive systems bypass most of the kernel stack and do a thin user-space version of this."

---

### 9. TCP vs UDP (deep)

| | **TCP** | **UDP** |
|---|---|---|
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Reliability | Guaranteed delivery (acks, retransmits) | Best-effort, no retransmit |
| Ordering | In-order delivery | No ordering guarantee |
| Flow/Congestion control | Yes (windowing, slow start, etc.) | No |
| Header overhead | 20+ bytes, stateful | 8 bytes, stateless |
| One-to-many | No (point-to-point) | Yes (supports multicast/broadcast) |
| Latency profile | Lower average reliability cost, but **jitter from retransmits & head-of-line blocking** | Lowest, most predictable latency; you handle loss yourself |

**Head-of-line (HOL) blocking — the killer for market data over TCP:** TCP delivers bytes strictly in order. If packet #5 is lost, packets #6, #7, #8 may have already arrived but TCP **will not hand them to your app** until #5 is retransmitted and arrives. So one lost packet stalls *everything behind it* — a latency spike of at least one round-trip, right when you can least afford it. For market data, where a stale quote is worthless, you'd rather get #6, #7, #8 *now* and deal with the gap at #5 yourself. UDP gives you exactly that freedom.

**Congestion control:** TCP backs off (shrinks its window) when it senses loss. Good for the internet, bad for a deterministic low-latency feed — you don't want your transport unilaterally slowing down. UDP doesn't do this.

**When each is used:**
- **UDP (usually multicast): market data feeds.** One-to-many, lowest latency, no retransmit jitter, no HOL blocking, no congestion backoff. The application handles gaps via sequence numbers.
- **TCP: order entry / order management, drop copies, anything that must not be lost or duplicated.** You're sending an order — you absolutely need to know it arrived exactly once and in order.

**🎤 If they ask "TCP vs UDP — when do you use each in trading?":**
"Market data uses UDP, almost always multicast: it's one-to-many so the exchange sends each update once to thousands of subscribers, it has the lowest and most predictable latency, and there's no retransmit jitter or head-of-line blocking — one lost packet doesn't stall everything behind it. The trade-off is you handle gaps yourself with sequence numbers. Order entry uses TCP because an order must arrive exactly once, in order, reliably — you can't drop or duplicate an order, and the per-message latency cost of reliability is acceptable there because volume is far lower than market data."

---

### 10. Multicast & the Gap-Recovery Problem

**Multicast** is one-to-many IP delivery: a sender transmits to a multicast group address, and the network (switches/routers via IGMP) replicates the packet to all subscribers. One send → many receivers, with the replication done in the network fabric rather than by the sender.

**Why exchanges use multicast for market data:**
- **Scalability:** thousands of participants get the same feed without the exchange sending thousands of copies — one packet, network replicates it.
- **Fairness:** every subscriber receives the update at (close to) the same time — no per-recipient unicast ordering advantage.
- **Lowest latency:** UDP multicast, no per-connection state, no retransmits.

**The gap-recovery problem.** UDP is best-effort — packets can be lost or reordered. Market data carries **monotonic sequence numbers** so each receiver can detect a gap (got 100, 101, 103 → 102 is missing). Recovery mechanisms:

1. **A/B feeds (line arbitration):** the exchange publishes the *same* feed twice over two independent network paths (Feed A and Feed B). The receiver listens to both and arbitrates: take whichever packet of a given sequence number arrives first, and use the other feed to fill gaps. A loss on one path is usually covered by the other. This is the primary, lowest-latency recovery — no request needed.
2. **Retransmit / recovery server:** if a sequence number is missing from *both* A and B, the receiver requests a retransmission from a TCP-based recovery service, or fetches from a periodic snapshot/refresh feed.
3. **Snapshot + incremental model (e.g., ITCH-style):** a periodic full snapshot lets a late-joiner or a hopelessly-behind receiver rebuild book state, then resume applying incremental updates by sequence number.

**🎤 If they ask "how does a feed handler recover from a dropped multicast packet?":**
"Market data carries sequence numbers, so the handler detects a gap immediately. First line of defense is A/B arbitration — the exchange sends the same feed over two independent paths, and we take whichever copy arrives first and use the other to fill any gap, so a single-path loss is invisible. If a sequence number is missing on both, we fall back to a retransmit request over a TCP recovery service or rebuild from a periodic snapshot feed. The design philosophy is: optimize the common case with redundant multicast and only pay the slow retransmit cost on a true double-loss."

---

### 11. Sockets API, Blocking vs Non-blocking, select/poll/epoll

**Core BSD sockets calls:**
- `socket()` — create an endpoint (specify family, type `SOCK_STREAM`=TCP / `SOCK_DGRAM`=UDP).
- `bind()` — assign a local address/port.
- `listen()` / `accept()` — server side TCP: mark passive, accept incoming connections.
- `connect()` — client side: establish a connection (TCP handshake; for UDP just sets the default peer).
- `send()`/`recv()` (or `write`/`read`, `sendto`/`recvfrom` for UDP) — data transfer.
- For multicast: `setsockopt(IP_ADD_MEMBERSHIP)` to join a group.

**Blocking vs non-blocking:**
- **Blocking:** `recv` on an empty socket *sleeps the thread* (context switch, scheduler) until data arrives. Simple, but introduces wake-up latency and a context switch — bad for the hot path.
- **Non-blocking** (`O_NONBLOCK`): `recv` returns immediately with `EAGAIN`/`EWOULDBLOCK` if there's nothing. Lets you **busy-poll** in a tight loop — lowest latency, burns CPU.

**Multiplexing many sockets — select / poll / epoll:**
- **`select`:** wait on a set of fds; O(n) scan every call, fixed fd limit (FD_SETSIZE), copies the whole set in/out each call. Doesn't scale.
- **`poll`:** like select without the fd-count limit, but still O(n) per call.
- **`epoll` (Linux):** you register fds *once* (`epoll_ctl`), and `epoll_wait` returns only the **ready** fds. Cost scales with the number of *active* fds, not the total — **O(active)**, not O(total). This is why epoll scales to tens of thousands of connections (the classic C10K solution). Supports edge-triggered mode for fewer wakeups.

**Why epoll scales:** select/poll re-scan and re-copy the entire fd set on every call, so cost grows with total connections even when only a few are active. epoll keeps the interest set in the kernel and only reports ready fds, so cost is proportional to activity, not connection count.

**Latency note:** epoll is the scalable *event-driven* model, but the absolute-lowest-latency path is still **busy-polling a non-blocking socket (or kernel-bypass ring)** — even epoll involves a syscall and potentially a context switch. Use epoll when you have many connections and can tolerate that; busy-poll the single hot feed when every nanosecond counts.

**🎤 If they ask "select vs poll vs epoll — why epoll?":**
"select and poll re-scan and re-copy the whole set of file descriptors on every call, so they're O(n) in total connections even if only one is active — they don't scale. epoll registers the fds once in the kernel and only returns the ones that are ready, so it's O(active), which is how you handle tens of thousands of connections. For the single hot market-data socket, though, I'd skip event notification entirely and busy-poll a non-blocking socket — or bypass the kernel — because even epoll costs a syscall."

---

### 12. Kernel Networking Stack Latency & Kernel Bypass

**Where the kernel stack spends time on receive:**
1. NIC raises an **interrupt** (or coalesces several) → context into the IRQ handler / softirq (NAPI).
2. Packet copied / DMA'd into a kernel **sk_buff**, walked up through IP and TCP/UDP layers (checksums, routing, demux to a socket).
3. Data queued in the **socket receive buffer**.
4. Your app's `recv` **syscall** copies the data from kernel buffer to user buffer (a memory copy + mode switch), possibly after a context switch to wake the thread.

That's interrupts, multiple buffer copies, layer processing, syscalls, and scheduler involvement — each adding latency and, worse, **jitter**. Total: several microseconds and unpredictable.

**Kernel bypass** removes the kernel from the data path: the NIC DMAs packets straight into **user-space memory**, and the application polls the receive ring directly — no per-packet interrupt, no kernel protocol processing, no copy into the kernel, no syscall.

- **DPDK (Data Plane Development Kit):** poll-mode drivers, hugepages, the app owns the NIC. You implement (or link) your own UDP/TCP. Common in market-data feed handlers and packet processing — and this is *exactly the world of an AWS data-plane.*
- **Solarflare / Onload (now AMD/Xilinx):** `onload` is a transparent kernel-bypass TCP/UDP stack — your normal sockets code runs, but I/O is serviced in user space via the Solarflare NIC. Hugely popular in HFT because you keep the sockets API but get bypass latency.
- **User-space TCP stacks:** (e.g., onload's stack, mTCP, F-Stack) implement TCP in user space so you avoid the kernel while keeping reliability for order entry.

**🎤 If they ask "what is kernel bypass and why?":**
"The normal path takes an interrupt, runs the packet up through the kernel's IP and TCP/UDP layers with checksums and buffer copies, queues it in a socket buffer, and then your `recv` syscall copies it to user space — several microseconds with a lot of jitter. Kernel bypass has the NIC DMA the packet straight into user-space memory and the app polls the ring directly: no interrupt, no kernel protocol processing, no extra copy, no syscall. DPDK gives you raw poll-mode access and you bring your own protocol; Solarflare onload gives you the same sockets API but services I/O in user space. You trade a CPU core spinning for roughly an order of magnitude lower, more deterministic latency."

---

### 13. NIC Concepts: Busy-poll, RSS, Zero-copy

- **Interrupts vs busy-poll (at the NIC):** covered above — latency setups poll the RX ring rather than take interrupts, and disable/limit interrupt coalescing.
- **RSS (Receive Side Scaling):** the NIC hashes each packet's flow (5-tuple) and steers it to one of several RX queues, each bound to a different core. This spreads receive load across cores and lets you **pin a given flow to a known core** — so a specific feed always lands on the core running its handler (cache locality, no cross-core hops). Related: flow steering / Flow Director for explicit queue placement.
- **Zero-copy:** avoid copying packet data between buffers. Normally data is copied NIC→kernel→user; zero-copy (`sendfile`, `MSG_ZEROCOPY`, or kernel-bypass DMA-into-user-memory) lets the app read/write the DMA buffer directly. Fewer copies = less CPU, lower latency, less cache pollution.
- **Hardware timestamping:** NICs (esp. Solarflare/Exablaze) timestamp packet arrival in hardware (PTP), so you measure wire-to-app latency precisely without software clock jitter.
- **TSO/GRO/LRO (offloads):** segmentation/aggregation offloads boost *throughput* but **add latency** (they batch). Latency-sensitive configs often disable GRO/LRO and rely on small, immediate packets.

**🎤 If they ask "what is RSS?":**
"Receive Side Scaling. The NIC hashes each packet's flow tuple and distributes packets across multiple receive queues, each tied to a core, so receive processing scales across CPUs instead of bottlenecking one core. In trading you also use it deliberately for affinity — steer a specific feed's flow to the exact core running its handler, so the data lands warm in that core's cache with no cross-core hop."

---

### 14. Nagle's Algorithm, TCP_NODELAY, Socket Buffer Tuning

**Nagle's algorithm** batches small TCP writes: it withholds a small segment until either a prior segment is ACKed or enough data accumulates to fill a full segment (MSS). It was designed to avoid flooding the network with tiny "tinygram" packets (think telnet keystrokes). The cost: it can **delay your small message by up to a round-trip time** waiting for an ACK.

**Why HFT disables Nagle (`TCP_NODELAY`):** an order message is small and must go out **immediately**. Waiting even a fraction of an RTT to "batch" it is unacceptable — you'd be deliberately delaying your own order. So order-entry sockets set `TCP_NODELAY` to send each write right away. (Beware the even worse interaction of Nagle + delayed-ACK, which can stall ~40 ms — a notorious latency bug.)

**Related socket tuning:**
- **`TCP_QUICKACK`** — disable delayed ACKs where appropriate.
- **Socket buffer sizes (`SO_RCVBUF`/`SO_SNDBUF`):** size receive buffers large enough that a momentary scheduling delay or burst doesn't drop UDP market-data packets, but not so large you add bufferbloat latency. For bursty multicast feeds, a generously sized RX buffer prevents drops during micro-stalls.
- **`SO_BUSY_POLL` / `SO_REUSEPORT`**, ring sizing, and disabling offloads (GRO/LRO) as above.

**🎤 If they ask "why disable Nagle's algorithm?":**
"Nagle batches small TCP sends — it holds a small segment back until a previous one is ACKed or it can fill a full segment, to avoid flooding the network with tiny packets. But an order is a small message that has to go out immediately; with Nagle you'd delay your own order by up to a round-trip waiting to batch it. So we set `TCP_NODELAY` on order-entry sockets to send each message instantly. There's also a nasty Nagle-plus-delayed-ACK interaction that can stall around 40 milliseconds, which is another reason to turn it off."

---

### 15. End-to-End Latency Path of a Trade

When the market moves, here's where the microseconds (and nanoseconds) go — *know this path cold*:

1. **Event happens at the exchange** → exchange's matching engine publishes a market-data update.
2. **Wire:** propagation over fiber/microwave to your colocated server. Speed of light / cable length dominates here — hence **colocation** (your box in the exchange's data center, equal cable lengths for fairness).
3. **NIC RX:** packet arrives at the NIC. With kernel bypass + busy-poll, it's DMA'd into user memory and you see it almost immediately; otherwise interrupt + softirq.
4. **Kernel (or bypass) stack:** parse Ethernet/IP/UDP. Bypass skips most of this.
5. **Feed handler / app:** parse the binary market-data message, check sequence number (A/B arbitration / gap detection), update the in-memory **order book**.
6. **Strategy:** the trading logic evaluates the new book state and decides to act (this is the "alpha" compute — kept tiny and branch-predictable on the hot path).
7. **Order construction:** build the outbound order message (often pre-built templates / pre-serialized to save time).
8. **Order out:** `send` over the **TCP order-entry** session (or via bypass/user-space TCP), through the NIC, back over the wire to the exchange's order gateway.
9. **Exchange ack / fill.**

The whole loop ("tick-to-trade") in a top-tier system is **single-digit microseconds or less**, with the most aggressive paths pushing logic into **FPGAs/smartNICs** so the order leaves the NIC without the CPU ever touching it.

**Every section of this guide maps to one hop:** context switches/page faults/scheduling = keep steps 5–7 jitter-free; kernel bypass/NIC/zero-copy = steps 3–4 and 8; UDP multicast vs TCP = steps 2/3 vs 8; syscalls = avoid them in 3–8.

**🎤 If they ask "walk me through the latency of a trade end to end":**
"It starts with a market-data update leaving the exchange's matching engine. It propagates over the wire to our colocated box — that's why we colocate and equalize cable lengths. The NIC receives it; with kernel bypass and busy-polling we pull it straight from a user-space ring with no interrupt or syscall. The feed handler parses the binary message, checks the sequence number with A/B arbitration, and updates the in-memory order book. The strategy evaluates the new book and decides to act, we build the order — often from a pre-serialized template — and send it out over the TCP order-entry session through the NIC back to the exchange. Tick-to-trade in a strong system is single-digit microseconds, and the most aggressive setups put the whole path in an FPGA so the CPU never touches it."

---

## Why Market Data = UDP Multicast and Order Entry = TCP (airtight)

This is a SIG favorite. Here's the tight version, reasoned from first principles.

**Market data → UDP multicast. Three independent reasons:**

1. **One-to-many (multicast):** an exchange feeds thousands of participants the *same* stream. TCP is point-to-point — the exchange would need a separate connection and a separate copy per subscriber, which doesn't scale and creates per-recipient timing differences (unfair). UDP multicast sends one packet that the network fabric replicates to everyone, scalably and (near-)simultaneously — **fair and scalable.**

2. **Latency & jitter (no retransmit, no HOL blocking):** TCP guarantees in-order delivery, so a single lost packet causes **head-of-line blocking** — everything after it waits for the retransmit, a multi-millisecond stall exactly when the market is moving. TCP also does congestion control, unilaterally slowing your feed on perceived loss. UDP has none of this: packets arrive as fast as the wire allows, and a loss doesn't stall what's behind it.

3. **Stale data is worthless anyway:** in market data, a quote that's a few milliseconds old is useless. There's no point in TCP reliably re-delivering an old packet that you'd ignore. You'd rather have the *newest* data now and handle gaps yourself — which is exactly UDP's model. Gaps are recovered out-of-band via **A/B feed arbitration**, sequence numbers, and snapshot/retransmit channels — paying the recovery cost only on actual loss.

**Order entry → TCP. Reasoned:**

1. **Correctness is non-negotiable:** an order must arrive **exactly once, reliably, in order.** A dropped order means a missed trade or — worse — an order you *think* you sent but didn't. A duplicated order means an unintended double position. TCP's acks, retransmits, sequencing, and dedup give you this for free.

2. **It's point-to-point and low-volume:** order entry is *your* session to *your* gateway — one-to-one, so there's no multicast benefit. And order message rates are vastly lower than market-data rates, so the per-message reliability overhead is affordable.

3. **You tune TCP for latency:** since you do use TCP, you remove its latency warts — `TCP_NODELAY` to disable Nagle so each order goes out immediately, often over a kernel-bypass/user-space TCP stack. You keep the reliability, drop the batching delay.

**One-liner to memorize:** *"Market data is a one-to-many firehose where speed beats reliability and stale data is useless — UDP multicast, recover gaps yourself with A/B feeds and sequence numbers. Order entry is a one-to-one channel where every message must land exactly once — TCP, with Nagle disabled."*

---

## How to Talk About Your AWS Data-Plane Work

Your AWS networking-infrastructure / data-plane experience is **directly the same problem space** as exchange connectivity — make the interviewer see that. Mapping:

- **"I work on kernel and userspace data paths"** → "That's exactly the kernel-bypass story in trading. At AWS I've worked on moving packets through fast data paths and minimizing the kernel's involvement — the same motivation as DPDK/onload feed handlers: get the packet to the application without per-packet interrupts, copies, and syscalls."
- **Concepts you can claim fluency in (use the ones that are true for you):**
  - **Zero-copy / DMA into user space** — "I understand why copies and the per-packet syscall dominate at high packet rates, and how to eliminate them."
  - **Busy-poll vs interrupt-driven I/O, interrupt coalescing** — "I've reasoned about the throughput-vs-latency trade-off of coalescing and polling."
  - **RSS / multi-queue NICs / flow steering** — "I know how flows are hashed across RX queues and pinned to cores for cache locality."
  - **CPU pinning, NUMA awareness, core isolation** — "I've dealt with placing data-path threads on the right cores/NUMA node."
  - **Packet processing pipelines / parsing at line rate** — directly analogous to a feed handler parsing ITCH/binary market data.
- **The honest bridge:** "The trading-specific layer I'd be learning is the financial protocols — FIX, ITCH/OUCH, the A/B feed recovery model, the order book — but the *systems* layer underneath, the part that determines latency, is exactly what I do now: kernel bypass, NIC behavior, polling, cache and TLB effects, avoiding context switches and page faults."

**Sample 30-second pitch:**
"At AWS I work on networking data-plane infrastructure — the kernel and userspace paths that packets take. The core challenge is the same one trading systems face: the kernel's normal network stack, with its interrupts, buffer copies, and syscalls, is too slow and too jittery for the latency you need, so you move to polling and kernel bypass, you care intensely about cache locality, NIC queue steering, and keeping the hot path off the scheduler. So when I look at a market-data feed handler or an order-entry path, the systems problems — busy-polling a NIC ring, avoiding page faults with mlocked memory, pinning to isolated cores — are problems I already work on daily. What I'd ramp up on is the domain: the exchange protocols and the trading semantics on top."

---

## Rapid-fire Q&A

**Q1. TCP vs UDP — which for market data and why?**
UDP, almost always multicast. One-to-many so the exchange sends each update once and the network replicates it; lowest, most predictable latency; no retransmit jitter and no head-of-line blocking, where one lost packet would stall everything behind it. We handle gaps ourselves with sequence numbers and A/B feed arbitration.

**Q2. Why is order entry TCP and not UDP?**
An order must arrive exactly once, in order, reliably — you can't drop or duplicate an order. It's point-to-point and low-volume, so TCP's reliability overhead is affordable, and we disable Nagle with TCP_NODELAY so each order still goes out immediately.

**Q3. What is head-of-line blocking?**
TCP delivers bytes strictly in order, so if one packet is lost, later packets that already arrived are held back until the lost one is retransmitted — one loss stalls everything behind it. That's why latency-sensitive market data uses UDP instead.

**Q4. What is a TLB and why does a TLB miss hurt?**
The TLB caches virtual-to-physical address translations. A miss forces a multi-level page-table walk — several dependent memory accesses that can themselves miss in cache — before your data can even be fetched. Huge pages reduce misses by making one entry cover 2 MB instead of 4 KB.

**Q5. Minor vs major page fault?**
Minor: the page is in RAM but not yet mapped into the process (e.g., first touch) — the kernel just fixes the mapping, microseconds. Major: the page must be fetched from disk/swap — milliseconds, catastrophic on a hot path. We use mlockall and pre-touch pages to eliminate both from the trading loop.

**Q6. What's the cost of a system call?**
A user-to-kernel mode switch and back — roughly a hundred to a few hundred nanoseconds, more with Spectre/Meltdown page-table-isolation mitigations, and far more if it blocks and causes a real context switch. At packet rates that overhead dominates, which motivates batching syscalls or kernel bypass.

**Q7. Why are context switches bad for latency?**
The direct register save/restore is only a microsecond or two; the real cost is cold caches and a flushed/polluted TLB on resume, plus branch-predictor pollution — tens of microseconds of effective hit. We avoid them by pinning to isolated cores and busy-polling instead of blocking.

**Q8. Why disable Nagle's algorithm?**
Nagle batches small TCP sends, holding a segment until a prior one is ACKed, which can delay a small order message by up to a round-trip. We set TCP_NODELAY so each order goes out immediately. The Nagle-plus-delayed-ACK interaction can also cause ~40 ms stalls.

**Q9. Interrupts or polling for the lowest latency?**
Polling. Interrupts add jitter by preempting the hot thread and incur wake-up latency; busy-polling burns a core but gives the lowest, most deterministic latency with no scheduler involvement. We spin on the NIC RX ring, usually with kernel bypass.

**Q10. What is kernel bypass and why use it?**
The NIC DMAs packets straight into user-space memory and the app polls the ring — no interrupt, no kernel IP/TCP processing, no buffer copy, no syscall — cutting several microseconds and most of the jitter. DPDK gives raw poll-mode access; Solarflare onload keeps the sockets API but services I/O in user space.

**Q11. Why does epoll scale better than select/poll?**
select and poll re-scan and re-copy the whole fd set every call, so they're O(total connections). epoll registers fds once in the kernel and returns only ready ones, so it's O(active connections) — that's how you handle tens of thousands of sockets.

**Q12. What is RSS?**
Receive Side Scaling: the NIC hashes each packet's flow tuple to spread packets across multiple RX queues bound to different cores, scaling receive processing. In trading we also use it for affinity — steering a feed to the exact core running its handler for cache locality.

**Q13. How does a feed handler recover a dropped multicast packet?**
Sequence numbers detect the gap instantly. Primary recovery is A/B arbitration — the same feed over two independent paths; take whichever copy arrives first and fill gaps from the other. If both miss it, request a retransmit over a TCP recovery service or rebuild from a periodic snapshot feed.

**Q14. How do you get deterministic low-latency scheduling on Linux?**
Pin the hot thread to an isolated core (isolcpus/cpuset), use nohz_full to kill the timer tick, steer IRQs away, run SCHED_FIFO so nothing preempts it, busy-poll so it never blocks, mlock the memory, and set the governor to performance with deep C-states disabled. One thread spinning on a dedicated core, warm caches, no interference.

**Q15. What's shared between threads and what's private?**
Shared: the address space — heap, globals, code — plus file descriptors and signal handlers. Private: each thread's stack, registers/program counter, and thread-local storage. Sharing makes communication cheap but costs synchronization and cache coherency, which is why hot paths often stay single-threaded.

**Q16 (bonus). Walk me through tick-to-trade latency.**
Update leaves the exchange's matching engine → propagates over the wire to our colocated box → NIC receives it, pulled from a user-space ring via busy-poll/bypass → feed handler parses the message and checks the sequence number → updates the in-memory order book → strategy decides → we build the order from a pre-serialized template → send over the TCP order-entry session back to the exchange. Single-digit microseconds in a strong system; the most aggressive paths run it in an FPGA.

---

## Self-Test Checklist

Tick each only when you can explain it out loud, unprompted, in under 30 seconds:

- [ ] What's shared vs private between threads, and why hot paths go single-threaded.
- [ ] The direct *and* indirect cost of a context switch (cache/TLB pollution).
- [ ] Virtual→physical translation: page tables, MMU, TLB, page-table walk.
- [ ] Minor vs major page fault and the latency difference (µs vs ms).
- [ ] How `mlockall`, pre-touching, and huge pages eliminate fault/TLB jitter.
- [ ] Core isolation toolbox: pinning, isolcpus, nohz_full, IRQ affinity, SCHED_FIFO, C-states/governor.
- [ ] What a syscall costs and why minimizing/bypassing them matters.
- [ ] Stack vs heap (OS view), brk/sbrk vs mmap, and why no malloc on the hot path.
- [ ] Interrupts vs polling, interrupt coalescing, and the latency trade-off.
- [ ] The practical network stack (app→TCP/UDP→IP→Ethernet→NIC→wire).
- [ ] TCP vs UDP across every axis (reliability, ordering, HOL blocking, congestion control, one-to-many).
- [ ] **Airtight: why market data = UDP multicast and order entry = TCP.**
- [ ] Multicast mechanics and the gap-recovery model (sequence numbers, A/B feeds, retransmit/snapshot).
- [ ] Sockets API call sequence; blocking vs non-blocking; select vs poll vs epoll and why epoll scales.
- [ ] Kernel-stack latency sources and what kernel bypass (DPDK, onload, user-space TCP) removes.
- [ ] NIC concepts: busy-poll, RSS/flow steering, zero-copy, hardware timestamping, offloads.
- [ ] Nagle's algorithm, TCP_NODELAY, and socket buffer tuning.
- [ ] The full end-to-end tick-to-trade path and which hop each technique optimizes.
- [ ] Your AWS data-plane 30-second pitch and the honest "domain I'd learn" bridge.
