# 03 — Concurrency, Atomics & Lock-Free Programming in C++

## Why this matters for SIG

Susquehanna's Trading System Engineering is **low-latency C++**. The whole game is moving data between threads — market-data feed handlers, strategy threads, order gateways — with the smallest possible, most *predictable* latency. Locks introduce blocking, context switches, priority inversion, and unbounded tail latency. That is why HFT shops obsess over **lock-free** data structures, the **C++ memory model**, and **cache behavior**.

You put "lock-free concurrency" on your resume, so the bar is: you can implement and defend a **SPSC ring buffer** from memory, explain every `memory_order` you choose, define a data race precisely, and explain CAS and ABA without hand-waving. Interviewers will push on *why* each memory ordering is correct — vague answers ("it makes it thread-safe") fail. This guide gets you to crisp, correct answers.

A note on accuracy: concurrency is full of statements that are *almost* true. I've tried to keep every memory-ordering claim precise. When in doubt in the room, reason from **happens-before**, not from intuition.

---

## 1. Processes vs Threads

A **process** has its own virtual address space, file descriptors, and resources. A **thread** is a unit of execution *within* a process.

Threads of the same process **share**:
- The address space (heap, globals, static data, code)
- File descriptors, signal handlers

Each thread has its **own**:
- Stack (local variables, call frames)
- Registers (including the program counter and stack pointer)
- `thread_local` storage

Because threads share the heap and globals, **unsynchronized access to shared mutable state is the entire source of concurrency bugs**. Processes are isolated by the MMU, so they're safer but communication is more expensive (IPC, shared memory, pipes). Threads are cheap to communicate between (shared memory directly) but dangerous.

**Context switch cost**: switching between threads of the same process is cheaper than between processes (no page-table / TLB flush for the address space), but it's still ~1µs+ and pollutes caches — in HFT you often **pin** threads to cores and *avoid* switching entirely.

> 🎤 **If they ask "difference between a process and a thread?":**
> "A process owns an isolated virtual address space; threads live inside a process and share its address space — heap, globals, code, file descriptors. Each thread has its own stack and registers. That sharing is why threads communicate cheaply but also why we need synchronization: two threads touching the same memory without coordination is a data race. In low-latency work we often pin threads to cores to avoid the cache and TLB cost of context switches."

---

## 2. `std::thread` Basics — join / detach

```cpp
#include <thread>
#include <iostream>

void work(int id) { std::cout << "thread " << id << "\n"; }

int main() {
    std::thread t(work, 42);   // starts running immediately
    // ... do other work concurrently ...
    t.join();                  // block until t finishes
}
```

- **`join()`** — the calling thread blocks until the target finishes. Required before the `std::thread` object is destroyed.
- **`detach()`** — the thread runs independently; you lose the handle. Its resources are reclaimed on completion.
- **Critical rule:** a `std::thread` that is still *joinable* (neither joined nor detached) at destruction calls `std::terminate()`. Always join or detach.
- Prefer `std::jthread` (C++20): it **auto-joins** in its destructor (RAII) and supports cooperative cancellation via `std::stop_token`.

Arguments are **copied/moved by value** into the thread by default — use `std::ref()` to pass by reference (and then *you* must guarantee the referent outlives the thread).

> 🎤 **If they ask "join vs detach?":**
> "join blocks the caller until the thread completes and reclaims it; detach lets it run unsupervised and gives up the handle. If a joinable thread is destroyed without either, the program calls std::terminate. I default to jthread in C++20 because it joins in its destructor, which is RAII-correct."

---

## 3. Data Races vs Race Conditions, Critical Sections

These are **not** the same thing — a favorite interview distinction.

**Data race (precise definition):** Two threads access the *same* memory location, **at least one access is a write**, the accesses are **not ordered by a happens-before relationship**, and at least one is **non-atomic**. In C++ a data race is **undefined behavior** — full stop. The compiler is allowed to assume it never happens.

**Race condition:** A broader, logical bug where the program's correctness depends on the relative *timing/interleaving* of operations. You can have a race condition with **no** data race — e.g. a check-then-act on an atomic:

```cpp
if (atomic_balance.load() >= amount)   // check
    atomic_balance.fetch_sub(amount);  // act — another thread may have drained it in between
```

Each atomic op is race-free, but the *logic* is wrong: classic TOCTOU.

**Critical section:** a region of code that accesses shared state and must execute atomically with respect to other threads — protected by a mutex, atomic, or other mechanism.

> 🎤 **If they ask "data race vs race condition?":**
> "A data race is a specific, formal thing: two threads touch the same location, one writes, there's no happens-before between them, and at least one access is non-atomic — that's undefined behavior in C++. A race condition is a logic bug where correctness depends on timing. They overlap but neither implies the other: a check-then-act on atomics has a race condition but no data race, because every individual access is properly synchronized."

---

## 4. Mutexes, Lock Guards, Deadlock

`std::mutex` provides mutual exclusion. Always lock via RAII wrappers, never raw `lock()/unlock()` (exceptions would leak the lock):

- **`std::lock_guard<std::mutex>`** — simplest; locks on construction, unlocks on destruction. No flexibility.
- **`std::unique_lock<std::mutex>`** — movable, can `unlock()`/`relock()`, supports deferred locking and timed locks; required for `std::condition_variable`.
- **`std::scoped_lock`** (C++17) — locks **multiple** mutexes at once using a deadlock-avoidance algorithm. Prefer it whenever you need 2+ mutexes.

```cpp
std::mutex m;
void f() {
    std::lock_guard<std::mutex> lk(m);   // unlocked automatically at scope exit
    // critical section
}
```

### Deadlock — the 4 Coffman conditions
All four must hold simultaneously; break any one to prevent deadlock:
1. **Mutual exclusion** — resources held in non-shareable mode.
2. **Hold and wait** — a thread holds one resource while waiting for another.
3. **No preemption** — resources can't be forcibly taken.
4. **Circular wait** — a cycle of threads each waiting on the next.

**Lock ordering** is the standard fix: define a global total order over mutexes and always acquire them in that order — this breaks **circular wait**. `std::scoped_lock(m1, m2)` does this safely (it uses `std::lock`, an all-or-nothing algorithm), so the order you pass them doesn't matter.

**Livelock:** threads aren't blocked but keep responding to each other and make no progress (e.g. both back off and retry in lockstep forever). **Starvation:** a thread never gets the resource because others keep winning.

> 🎤 **If they ask "how do you avoid deadlock?":**
> "Deadlock needs all four Coffman conditions — mutual exclusion, hold-and-wait, no preemption, and circular wait. The practical fix is to break circular wait by enforcing a global lock ordering: always acquire mutexes in the same order. If I need several at once I use std::scoped_lock, which uses an all-or-nothing locking algorithm so it can't deadlock regardless of argument order."

---

## 5. Condition Variables, Spurious Wakeups, the Predicate Pattern

A condition variable lets a thread **sleep until** some condition becomes true, without busy-waiting. It must be paired with a mutex and a `std::unique_lock`.

```cpp
std::mutex m;
std::condition_variable cv;
std::queue<int> q;

void consumer() {
    std::unique_lock<std::mutex> lk(m);
    cv.wait(lk, []{ return !q.empty(); });  // predicate overload — handles spurious wakeups
    int x = q.front(); q.pop();
    lk.unlock();
    // use x
}

void producer(int x) {
    {
        std::lock_guard<std::mutex> lk(m);
        q.push(x);
    }
    cv.notify_one();   // wake one waiter
}
```

**Why the predicate (lambda) form is mandatory:**
- **Spurious wakeups:** `wait()` may return *without* any `notify`. The OS is allowed to do this. The predicate overload loops internally: `while(!pred()) wait(lk);` so a spurious wake just re-checks and re-sleeps.
- It also handles the **lost-wakeup**/ordering issue: if the condition is *already* true before you wait, the predicate is checked first and you don't block forever.

`wait()` atomically releases the mutex and sleeps; on wakeup it re-acquires the mutex before returning. Notify under or just after the lock — but you must modify the shared state *under* the lock or the wakeup can be lost.

`notify_one` vs `notify_all`: use `notify_all` when multiple waiters may now be able to proceed or wait on different predicates.

> 🎤 **If they ask "what's a spurious wakeup and how do you handle it?":**
> "A condition variable's wait can return without anyone calling notify — that's a spurious wakeup, allowed by the standard and the OS. The fix is to always wait with a predicate: cv.wait(lock, pred). Internally it loops, re-checking the predicate, so a spurious wake just goes back to sleep. The same loop also prevents lost wakeups when the condition was already true."

---

## 6. The C++ Memory Model

Single-threaded C++ behaves "as if" executed in program order. But across threads, both the **compiler** (instruction scheduling, register caching, dead-store elimination) and the **CPU** (out-of-order execution, store buffers, cache coherence delays) can **reorder** memory operations to go faster. Each is only required to preserve *single-threaded* observable behavior.

The model is built on **happens-before**: if operation A happens-before operation B, then A's effects are visible to B. Within one thread, program order gives this (sequenced-before). *Across* threads, you only get happens-before through **synchronization** — a release operation that is observed by an acquire operation (or mutex lock/unlock, thread start/join, etc.).

**Sequential consistency (SC):** the strongest model — all threads observe a single, total order of all SC operations, consistent with each thread's program order. It's the intuitive "interleaving" model and the **default** for `std::atomic` operations (`memory_order_seq_cst`). It's also the most expensive (often needs full memory fences / `mfence` / `LOCK`-prefixed ops or `DMB` on ARM).

**Key point:** *any* access to shared mutable data must be either protected by a lock or done through `std::atomic`. Otherwise it's a data race = UB, and "it worked on my machine" means nothing.

> 🎤 **If they ask "why do compilers and CPUs reorder memory operations?":**
> "For speed. The compiler reorders, caches values in registers, and eliminates redundant loads/stores; the CPU executes out of order, uses store buffers, and has per-core caches with coherence latency. Each only guarantees single-threaded as-if behavior. Across threads you get no ordering guarantees unless you establish happens-before — via a mutex or via release/acquire on atomics. The whole memory model is defined in terms of happens-before."

---

## 7. `std::atomic`, CAS, Atomicity vs Visibility

`std::atomic<T>` gives **race-free** access to a single location: loads, stores, and read-modify-write (RMW) ops are indivisible and (with appropriate ordering) establish happens-before.

Key operations:
- `load()`, `store()`
- `exchange()` — set and return old value
- `fetch_add`, `fetch_sub`, `fetch_or`, ... — atomic RMW, return the *previous* value
- `compare_exchange_strong/weak(expected, desired)` — **CAS**, the workhorse of lock-free code

**Atomicity ≠ visibility, two distinct guarantees:**
- **Atomicity:** the operation is indivisible — no torn read/write; observers see either the old or new value, never a half-written one.
- **Visibility / ordering:** *when* and in *what order* other threads see this and surrounding operations — controlled by the `memory_order` argument.

A `relaxed` atomic gives atomicity but **no** ordering guarantees relative to other variables. That's the distinction interviewers probe.

`std::atomic<T>::is_lock_free()` / `is_always_lock_free` tells you whether the type uses a hidden lock (large/odd-sized T may not be lock-free). On x86-64, atomics up to 8 bytes (and 16 with `cmpxchg16b`) are lock-free.

### Compare-And-Swap (CAS) explained

```cpp
bool compare_exchange_weak(T& expected, T desired, ...);
```
Atomically: *if* the atomic currently equals `expected`, set it to `desired` and return `true`; *otherwise* load the current value into `expected` and return `false`. This is the primitive you build lock-free algorithms on — "try to commit my update only if nobody changed it underneath me."

- **`_weak`** may fail *spuriously* (return false even when the value matched) on some architectures (LL/SC machines like ARM), but is cheaper — use it inside a retry loop.
- **`_strong`** never fails spuriously — use it when you're *not* already looping.

The canonical CAS loop:
```cpp
std::atomic<int> v{0};
int old = v.load(std::memory_order_relaxed);
int desired;
do {
    desired = compute(old);
} while (!v.compare_exchange_weak(old, desired,
                                  std::memory_order_release,   // on success
                                  std::memory_order_relaxed)); // on failure
// on failure, `old` is auto-updated to the current value; recompute and retry
```

> 🎤 **If they ask "what's the difference between atomicity and visibility?":**
> "Atomicity means the operation is indivisible — no torn reads or writes; you always see a whole old or whole new value. Visibility, or ordering, is about *when* other threads see that write and how it's ordered relative to other memory operations — that's what the memory_order argument controls. A relaxed atomic is fully atomic but gives no ordering guarantees against other variables."

---

## 8. Memory Ordering — what each guarantees

This is the part interviewers love. Be precise.

### `memory_order_relaxed`
Only guarantees **atomicity** and **modification-order consistency** for *that single variable* (all threads agree on the order of writes to one atomic). **No** ordering with respect to other memory operations, no happens-before across threads. Safe when you only need an atomic counter and don't use it to publish other data — e.g. an independent statistics counter incremented with `fetch_add(1, relaxed)`, or a reference count *increment* (the *decrement-to-zero* that frees needs acquire/release).

### `memory_order_release` (store / RMW)
Pairs with an acquire. A release store ensures all memory writes **sequenced-before** it in this thread become visible to any thread that performs an **acquire** load reading that value. Think "publish": "everything I wrote before this point is now safe to read." It is a one-way barrier — earlier ops can't move *past* it.

### `memory_order_acquire` (load / RMW)
Pairs with a release. An acquire load ensures that once you read a value written by a release store, all writes that happened-before that release are visible to you, and no subsequent operations move *before* it. Think "consume / subscribe."

**Release/acquire pairing** is the key mechanism for publishing data without a full SC fence:
```cpp
// Publisher
data = 42;                                 // (1) ordinary write
ready.store(true, std::memory_order_release); // (2) release — publishes (1)

// Consumer
if (ready.load(std::memory_order_acquire)) {  // (3) acquire — synchronizes with (2)
    assert(data == 42);                       // (4) GUARANTEED to see 42
}
```
Because (3) reads the value stored by (2), (2) **synchronizes-with** (3), so (1) happens-before (4). Without acquire/release, reading `data` here would be a data race.

### `memory_order_acq_rel`
For RMW operations (e.g. `fetch_add`, `compare_exchange`) that need to be **both** an acquire (on the load part) **and** a release (on the store part). Used when an op both consumes published state and publishes its own.

### `memory_order_seq_cst` (default)
Everything `acq_rel` gives, **plus** a single global total order over *all* seq_cst operations across all threads. This is what you need for correctness in patterns like Dekker's / IRIW where independent stores must be globally ordered. It's the safe default; the cost is stronger fences. Optimize down to acquire/release only when you've reasoned it through.

(There's also `memory_order_consume`, intended for data-dependency ordering, but it's effectively deprecated/unimplemented — compilers promote it to acquire. Mention you know it exists; don't use it.)

> 🎤 **If they ask "what does memory_order_acquire guarantee?":**
> "An acquire load, when it reads a value written by a release store on another thread, establishes synchronizes-with: every write that happened-before that release store becomes visible to me, and nothing after my acquire can be reordered before it. It's the 'subscribe' half of a publish/subscribe pair. On its own it does nothing — it only has teeth when paired with a release store whose value it actually reads."

> 🎤 **If they ask "when is relaxed safe?":**
> "When you only need atomicity of one variable and don't use that variable to publish or order access to other data. The classic case is an independent counter — a metrics counter or a reference-count increment. The moment the atomic gates visibility of other memory (a flag that publishes a buffer), you need release/acquire."

---

## 9. Lock-Free vs Wait-Free vs Blocking; ABA

Progress guarantees, from weakest to strongest:
- **Blocking:** a thread can be stalled indefinitely if another holding a lock is descheduled. (Mutexes.)
- **Lock-free:** *some* thread always makes progress in a bounded number of steps, system-wide. Individual threads can starve, but the system never stalls. Typically built on CAS retry loops.
- **Wait-free:** *every* thread makes progress in a **bounded** number of its own steps — no starvation, no unbounded retries. Strongest and hardest; rare in practice.

Lock-free ≠ "no locks used loosely" — it's a formal **progress** property. A spinlock uses no OS mutex but is **still blocking** (if the holder is preempted, everyone spins).

### The ABA problem
In a CAS loop you check "is the value still the `expected` one?" But a value can change **A → B → A**: thread T1 reads A, stalls; T2 changes it to B then back to A (often freeing and reallocating the *same* address for a node). T1's CAS sees A, succeeds — but the world changed underneath it; the A is not the *same* A. This silently corrupts lock-free stacks/queues that CAS on pointers.

**Fixes:**
- **Tagged pointers / version counters:** pack a monotonically increasing counter with the pointer and CAS the pair (`compare_exchange` on a 128-bit value via `cmpxchg16b`). The counter makes A→B→A detectable.
- **Hazard pointers / RCU / epoch-based reclamation:** prevent a node from being freed (and thus its address reused) while another thread might still CAS on it.
- Note: ABA is mainly a problem when memory is **reclaimed and reused**. A SPSC ring buffer with indices doesn't suffer ABA because indices are monotonically increasing values, not reused pointers.

> 🎤 **If they ask "what is the ABA problem?":**
> "In a CAS-based algorithm you assume that if the value is still the expected one, nothing changed. But another thread can change it from A to B and back to A — classically by freeing a node and the allocator handing back the same address. Your CAS sees A and succeeds, but the underlying state is different, corrupting the structure. You fix it with version-tagged pointers so A-then-A looks different, or with safe reclamation like hazard pointers or epochs so the address can't be reused while in flight."

---

## 10. ⭐ SPSC Lock-Free Ring Buffer — the centerpiece

**The classic HFT interview question.** Single producer thread, single consumer thread, fixed-capacity ring. No mutexes. Because there's exactly **one** producer and **one** consumer, we never need CAS — each index is written by only one thread. We only need correct **release/acquire** publishing so the consumer sees the data the producer wrote (and vice versa for the free slot).

Design choices:
- Capacity is a power of two so we can mask instead of modulo (cheaper). Equivalent could use modulo.
- We keep one slot empty to distinguish "full" from "empty" (head == tail means empty).
- `head` (write index) is written **only** by the producer; `tail` (read index) **only** by the consumer. Each loads the *other's* index with **acquire** and stores *its own* index with **release**.
- We pad `head` and `tail` to separate cache lines to avoid **false sharing** (Section 11).

```cpp
#include <atomic>
#include <cstddef>
#include <new>          // std::hardware_destructive_interference_size
#include <type_traits>
#include <utility>

template <typename T, std::size_t Capacity>
class SpscRingBuffer {
    static_assert((Capacity & (Capacity - 1)) == 0,
                  "Capacity must be a power of two");
    static constexpr std::size_t kMask = Capacity - 1;

    // Cache-line size; fall back to 64 if the constant isn't provided.
#ifdef __cpp_lib_hardware_interference_size
    static constexpr std::size_t kCacheLine =
        std::hardware_destructive_interference_size;
#else
    static constexpr std::size_t kCacheLine = 64;
#endif

    // Storage. T is constructed/destructed in-place on push/pop.
    // (Using a raw aligned buffer keeps it general; for trivially copyable T
    //  you could just hold T[] directly.)
    alignas(T) std::byte storage_[Capacity * sizeof(T)];

    T* slot(std::size_t i) noexcept {
        return reinterpret_cast<T*>(&storage_[(i & kMask) * sizeof(T)]);
    }

    // head_ = next write position, written only by the producer.
    // tail_ = next read  position, written only by the consumer.
    // Pad each onto its own cache line so the producer's writes to head_
    // don't invalidate the consumer's cache line holding tail_ (false sharing).
    alignas(kCacheLine) std::atomic<std::size_t> head_{0};
    alignas(kCacheLine) std::atomic<std::size_t> tail_{0};

public:
    SpscRingBuffer() = default;
    SpscRingBuffer(const SpscRingBuffer&) = delete;
    SpscRingBuffer& operator=(const SpscRingBuffer&) = delete;

    // Producer side. Returns false if the buffer is full.
    bool push(const T& item) {
        // We are the only writer of head_, so a relaxed load of our own
        // index is fine — no other thread modifies it.
        const std::size_t head = head_.load(std::memory_order_relaxed);
        const std::size_t next = head + 1;

        // Read the consumer's progress. Acquire so that if the consumer has
        // freed a slot (advanced tail_), we observe that freeing and can
        // safely overwrite the slot.
        if (next - tail_.load(std::memory_order_acquire) > Capacity) {
            return false;   // full (one slot kept empty via > Capacity check)
        }

        // Construct the element into the slot BEFORE publishing the new head.
        ::new (static_cast<void*>(slot(head))) T(item);

        // Release: publishes both the constructed element and the index bump.
        // A consumer that acquires this head_ value is guaranteed to see the
        // fully-constructed item. This is the producer->consumer handshake.
        head_.store(next, std::memory_order_release);
        return true;
    }

    // Consumer side. Returns false if the buffer is empty.
    bool pop(T& out) {
        // We are the only writer of tail_, so relaxed load of our own index.
        const std::size_t tail = tail_.load(std::memory_order_relaxed);

        // Acquire load of head_: synchronizes-with the producer's release
        // store in push(). Guarantees we see the item the producer wrote.
        if (tail == head_.load(std::memory_order_acquire)) {
            return false;   // empty
        }

        T* p = slot(tail);
        out = std::move(*p);   // hand the element to the caller
        p->~T();               // destroy in-place

        // Release: publishes that this slot is now free. The producer's
        // acquire load of tail_ will observe the freed slot before reusing it.
        tail_.store(tail + 1, std::memory_order_release);
        return true;
    }

    bool empty() const {
        return head_.load(std::memory_order_acquire) ==
               tail_.load(std::memory_order_acquire);
    }
};
```

### Why each memory order is correct (be ready to defend this)

| Access | Order | Why |
|---|---|---|
| Producer reads `head_` | relaxed | Only the producer writes `head_`; reading your own variable needs no synchronization. |
| Producer reads `tail_` | **acquire** | Synchronizes with the consumer's release store of `tail_`, so the producer sees freed slots before overwriting them. |
| Producer writes `head_` | **release** | Publishes the just-constructed element; pairs with the consumer's acquire load of `head_`. Guarantees the consumer sees a fully built item. |
| Consumer reads `tail_` | relaxed | Only the consumer writes `tail_`. |
| Consumer reads `head_` | **acquire** | Synchronizes with the producer's release store of `head_`; sees the published element. |
| Consumer writes `tail_` | **release** | Publishes that the slot is free; pairs with the producer's acquire load of `tail_`. |

**Common critiques an interviewer wants you to catch:**
- Using `relaxed` on the *cross-thread* index loads → the consumer could read a stale `head_` and read a half-constructed element. **Bug.**
- Bumping the index *before* constructing the element → consumer reads garbage. Construct first, then release-store the index. **Ordering of the data write vs the publish is the whole point.**
- No cache-line padding → `head_` and `tail_` share a line → false sharing → producer and consumer ping-pong the line, killing throughput.
- No power-of-two / capacity check → overflow or wraparound bugs. (The `next - tail > Capacity` form is wrap-safe with unsigned arithmetic.)
- Using `seq_cst` everywhere → correct but slower; acquire/release is sufficient and the right call for low latency.

> 🎤 **If they ask "implement / walk me through a SPSC queue":**
> "Two indices, head written only by the producer, tail only by the consumer, on a power-of-two ring. Each thread reads its own index relaxed and the other thread's index with acquire. The producer constructs the element, then does a release store to head — that publish guarantees the consumer, which acquire-loads head, sees a fully constructed element. The consumer reads the element then release-stores tail to free the slot, which the producer acquire-loads before reusing it. No CAS is needed because each index has exactly one writer. I pad head and tail onto separate cache lines to avoid false sharing."

---

## 11. False Sharing & Cache-Line Padding

Caches move data in **cache lines** (typically 64 bytes). If two variables touched by two different threads land on the *same* line, every write by one thread **invalidates** the line in the other core's cache — they "ping-pong" the line over the coherence protocol even though they're logically independent. This is **false sharing**, and it can cost 10–100× throughput.

Fix: put each hot, independently-written variable on its **own** cache line:
```cpp
struct alignas(64) Counter {        // align the whole struct to a line
    std::atomic<long> value{0};
    char pad[64 - sizeof(std::atomic<long>)];   // fill the rest of the line
};
```
or use `alignas(std::hardware_destructive_interference_size)` (C++17). In the ring buffer above, that's exactly why `head_` and `tail_` are `alignas(kCacheLine)`.

The flip side, `std::hardware_constructive_interference_size`, tells you the max size to *keep together* on one line when you *want* sharing.

> 🎤 **If they ask "what is false sharing?":**
> "Two independent variables used by two different threads happen to sit on the same 64-byte cache line. Even though the threads never touch the same variable, each write invalidates the other core's copy of the line, so the line ping-pongs across cores through the coherence protocol — huge, invisible slowdown. The fix is padding/aligning each hot variable to its own cache line, e.g. alignas(64) or hardware_destructive_interference_size."

---

## 12. Spinlocks vs Mutexes — when busy-waiting wins

A **mutex** that contends typically **sleeps** the thread (futex → kernel), causing a context switch (~1µs+) and cache pollution. A **spinlock** busy-waits in user space on an atomic flag — no syscall, no context switch.

```cpp
class SpinLock {
    std::atomic_flag flag_ = ATOMIC_FLAG_INIT;
public:
    void lock() {
        while (flag_.test_and_set(std::memory_order_acquire)) {
            // optional: __builtin_ia32_pause() / _mm_pause() to reduce
            // power and ease the contending core (PAUSE instruction)
        }
    }
    void unlock() { flag_.clear(std::memory_order_release); }
};
```

**Spinlocks win when:**
- The critical section is **very short** (a few instructions) so spinning costs less than a context switch.
- Threads are **pinned** to dedicated cores and won't be preempted while holding the lock (otherwise everyone spins waiting for a descheduled holder — wasted cycles, even priority inversion).
- You need **bounded, predictable** latency and can't tolerate the tail latency of a syscall — exactly the HFT case.

**Mutexes win when** the critical section may be long or the holder may block/sleep — spinning would burn a whole core for nothing.

Note a spinlock is **blocking** (not lock-free as a progress property): if the holder is descheduled, others spin uselessly. That's why pinning matters. Use the **PAUSE** instruction in the spin loop and consider exponential backoff to reduce coherence traffic.

> 🎤 **If they ask "spinlock or mutex for low latency?":**
> "Depends on the critical section and whether threads are pinned. A mutex sleeps the thread on contention — a syscall and a context switch, ~1µs plus cache pollution and unpredictable tail latency. A spinlock busy-waits in user space, which is far cheaper *if* the critical section is tiny and the holder won't be preempted, which is true when threads are pinned to cores. In HFT we usually want the spinlock for the bounded latency, with a PAUSE in the loop. If the section can be long or the holder can block, a mutex is better so we don't burn a core."

---

## 13. `thread_local`, `std::async` / futures / promises

**`thread_local`** — each thread gets its own instance of the variable; storage is per-thread. Great for per-thread buffers, RNG state, allocators — avoids sharing entirely (and thus avoids synchronization).

**Futures/promises** — a one-shot channel to move a result (or exception) from a producer to a consumer:
- `std::promise<T>` — the producer side; `set_value` / `set_exception`.
- `std::future<T>` — the consumer side; `get()` blocks until the value is ready (and rethrows a stored exception).
- `std::async(policy, fn, args...)` — runs `fn` (possibly on a new thread with `std::launch::async`, or lazily/deferred with `std::launch::deferred`) and returns a `future`. Caveat: the `future` returned by `std::async` **blocks in its destructor** until completion — a known gotcha.
- `std::packaged_task<Sig>` — wraps a callable so its result feeds a future; handy for thread pools.

These are **convenient but not low-latency** primitives (allocation, synchronization, blocking) — fine for setup/control flow, not the hot path.

> 🎤 **If they ask "what's thread_local for?":**
> "It gives each thread its own copy of a variable, so there's nothing shared and nothing to synchronize — ideal for per-thread scratch buffers, RNG state, or arena allocators. It sidesteps contention entirely."

---

## 14. Common Bugs

- **Torn reads/writes:** a non-atomic 64-bit (or larger) value read while being written → you observe half-old, half-new bits. Use `std::atomic` (or a lock) for any concurrently-accessed multi-byte value. (Even on x86, the compiler may split or the type may be misaligned/oversized.)
- **Missing memory barrier / wrong order:** writing data then a flag with `relaxed`, so the reader sees the flag set before the data write is visible → reads stale/garbage data. Fix: release on the publish, acquire on the read.
- **Publishing a pointer without release:** storing a freshly-`new`-ed object's pointer with a plain/relaxed store — the constructor's writes may not be visible to the reader. Needs a release store + acquire load.
- **Double-checked locking done wrong:**
  ```cpp
  // BROKEN: non-atomic flag/pointer → data race, may publish a
  // partially constructed object.
  if (!instance) {                 // unsynchronized read — UB
      std::lock_guard<std::mutex> lk(m);
      if (!instance) instance = new Singleton();  // write visible torn
  }
  ```
  **Right, with atomics:**
  ```cpp
  std::atomic<Singleton*> instance{nullptr};
  std::mutex m;
  Singleton* get() {
      Singleton* p = instance.load(std::memory_order_acquire);   // fast path
      if (!p) {
          std::lock_guard<std::mutex> lk(m);
          p = instance.load(std::memory_order_relaxed);
          if (!p) {
              p = new Singleton();
              instance.store(p, std::memory_order_release);       // publish
          }
      }
      return p;
  }
  ```
  **Best:** just use a function-local `static` — C++11 guarantees thread-safe, exactly-once initialization ("magic statics"):
  ```cpp
  Singleton& get() { static Singleton s; return s; }
  ```

> 🎤 **If they ask "what's wrong with classic double-checked locking?":**
> "The outer check reads the pointer without synchronization, and the assignment publishes it without a release barrier. So one thread can see a non-null pointer to an object whose constructor's writes aren't visible yet — a torn/partially-constructed read, which is undefined behavior. The fix is to make the pointer a std::atomic and use an acquire load on the fast path with a release store when publishing. Better still, use a function-local static — C++11 guarantees thread-safe one-time init."

---

## 15. Rapid-fire Q&A

**Q1. Difference between a data race and a race condition?**
A data race: two threads access the same location, ≥1 writes, no happens-before between them, ≥1 non-atomic — undefined behavior. A race condition: a logic bug depending on timing/interleaving. Neither implies the other; check-then-act on atomics is a race condition with no data race.

**Q2. What does a thread share with its siblings vs keep private?**
Shares the address space — heap, globals, code, file descriptors. Private: stack, registers/PC, thread_local storage.

**Q3. What does `memory_order_acquire` guarantee?**
When an acquire load reads a value written by a release store, it synchronizes-with that store: all writes sequenced-before the release become visible, and nothing after the acquire reorders before it. It does nothing on its own — only paired with a release whose value it reads.

**Q4. What does `memory_order_release` guarantee?**
A release store makes all writes sequenced-before it visible to any thread that acquire-loads the stored value. It's the "publish" barrier; earlier writes can't move past it.

**Q5. When is `memory_order_relaxed` safe?**
When you need only atomicity of a single variable, not ordering of other memory — e.g. an independent stats counter or a refcount increment. Not safe when the atomic gates visibility of other data.

**Q6. What is `seq_cst` and why is it the default?**
Sequential consistency: a single global total order of all seq_cst ops consistent with program order. It's the intuitive, hardest-to-misuse model, so it's the default. It's also the most expensive (stronger fences); drop to acquire/release once you've reasoned it through.

**Q7. What is CAS and what's `compare_exchange_weak` vs `strong`?**
Compare-and-swap: atomically set the value to `desired` only if it currently equals `expected`, else load the current value into `expected` and fail. `weak` may fail spuriously but is cheaper — use in retry loops; `strong` never fails spuriously — use when not looping.

**Q8. What is the ABA problem and how do you fix it?**
A value goes A→B→A; a stalled thread's CAS sees A and succeeds though the state changed (often via freeing/reusing the same address). Fix with version-tagged pointers (CAS the pair) or safe reclamation (hazard pointers, epochs, RCU).

**Q9. Lock-free vs wait-free vs blocking?**
Blocking: a thread can stall others indefinitely (mutexes). Lock-free: some thread always makes progress system-wide (no global stall, but individual starvation possible). Wait-free: every thread finishes in a bounded number of its own steps.

**Q10. Is a spinlock lock-free?**
No. A spinlock uses no OS mutex but is still blocking as a progress property: if the holder is preempted, everyone else spins making no progress.

**Q11. What are the four Coffman conditions and how do you break deadlock?**
Mutual exclusion, hold-and-wait, no preemption, circular wait — all four needed. Break circular wait with a global lock ordering, or use std::scoped_lock for multiple mutexes (all-or-nothing locking).

**Q12. Why must you wait on a condition variable with a predicate?**
To handle spurious wakeups (wait can return with no notify) and lost wakeups (condition already true). The predicate overload loops, re-checking the condition each wake.

**Q13. What is false sharing and how do you fix it?**
Two independent variables on the same cache line, written by two threads, ping-pong the line through cache coherence. Fix: pad/align each to its own line — alignas(64) or hardware_destructive_interference_size.

**Q14. Why are lock-free structures attractive for low latency?**
No blocking or context switches, no priority inversion, no convoying behind a sleeping lock-holder, and bounded, predictable tail latency — which is what HFT cares about more than average latency.

**Q15. In a SPSC ring buffer, why don't you need CAS?**
Each index has exactly one writer (producer writes head, consumer writes tail). With a single writer there's no contention on a given index, so a plain release store + acquire load on the other thread's index suffices — no read-modify-write/CAS needed.

**Q16. What's wrong with naive double-checked locking, and the right way?**
The outer check is an unsynchronized read and the publish lacks a release barrier, so you can see a non-null pointer to a not-yet-constructed object — UB. Fix: atomic pointer, acquire load on the fast path, release store to publish. Or just use a function-local static (C++11 magic statics).

**Q17. Why do CPUs and compilers reorder memory ops, and what restores order?**
Speed — out-of-order execution, store buffers, register caching, instruction scheduling; each preserves only single-threaded as-if behavior. Cross-thread order is restored by establishing happens-before: mutexes or release/acquire (or seq_cst) atomics.

---

## 16. Self-test checklist

- [ ] I can give the **precise** definition of a data race and explain why it's UB.
- [ ] I can distinguish a data race from a race condition with an example of each in isolation.
- [ ] I can state what a thread shares vs keeps private.
- [ ] I can list the four Coffman conditions and explain lock ordering / scoped_lock.
- [ ] I can explain spurious wakeups and write the predicate wait pattern.
- [ ] I can explain *why* compilers/CPUs reorder, and what happens-before means.
- [ ] I can explain acquire, release, acq_rel, relaxed, seq_cst — and pair them correctly.
- [ ] I can say exactly when `relaxed` is safe (and when it bites).
- [ ] I can write a CAS retry loop and explain weak vs strong.
- [ ] I can explain ABA and at least two fixes.
- [ ] I can distinguish blocking / lock-free / wait-free as progress guarantees.
- [ ] **I can implement a SPSC ring buffer from scratch and defend every memory_order.**
- [ ] I can explain false sharing and fix it with cache-line padding.
- [ ] I can argue spinlock vs mutex for a given low-latency scenario.
- [ ] I can spot and fix broken double-checked locking.
- [ ] I can explain why lock-free matters for HFT tail latency specifically.

---

*Drill the SPSC buffer until you can write it cold and explain each `memory_order` line-by-line. That single question, answered crisply, signals you can actually back up "lock-free concurrency" on your resume.*
