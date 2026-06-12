# 04 — DSA & Live Coding (C++ / HFT flavor)

> Study guide for the **Susquehanna (SIG) Trading System Engineering Internship** — low-latency C++.
> Pipeline: Recruiter brief → **Technical Live Coding** → Final Design & Team Fit.
> You're ~CF 1100 — easy/medium is comfortable. The win condition here is **fluency + clean code + clear narration**, not exotic algorithms.

---

## 1. Why this matters for SIG + the live-coding performance script

### Why this matters
SIG builds **low-latency C++ trading systems**. The live-coding round is *practical medium-level C++*. They are not testing whether you can solve a 2400-rated CF problem. They are testing:

1. **Can you write correct, clean, idiomatic C++ under mild pressure?**
2. **Do you communicate?** Trading desks live or die on clear communication. They want to *hear you think*.
3. **Do you understand cost?** Big-O, cache, allocations — "what's the complexity" and "what allocates" come up constantly.
4. **Do you handle edge cases?** In trading, an unhandled edge case is real money lost.

A 1100-rated competitive programmer who codes cleanly, narrates clearly, tests their own code, and reasons about complexity out loud will **out-perform** a 1900-rated coder who goes silent and writes spaghetti. Optimize for *that*.

### The performance script (memorize this loop)

```
CLARIFY → APPROACH → COMPLEXITY → CODE (narrating) → TEST → EDGE CASES → OPTIMIZE
```

**1. CLARIFY (30–60s).** Never start coding immediately. Ask:
- "What are the input ranges / sizes?" (drives whether O(n²) is fine)
- "Can the input be empty / null / have duplicates / be negative?"
- "Are inputs sorted? Any guarantees?"
- "What should I return on invalid input?"
- Restate the problem in your own words: *"So I need to … is that right?"*

**2. APPROACH (state it before coding).** "My first thought is brute force, which is O(n²). I think I can do better with a hash map for O(n). Let me go with the hash map approach — does that sound good?" → **Wait for a nod.** This invites the interviewer to steer you, and steering you = giving you hints.

**3. COMPLEXITY (say it out loud).** "Time O(n), space O(n)." State it *before* and *after* coding.

**4. CODE while NARRATING.** Talk as you type: *"I'll use an unordered_map from value to index… iterating once… for each element I check if the complement exists…"* Write top-down: signature first, then fill in. Use real names (`prices`, `bestBid`), not `a`, `b`, `c`.

**5. TEST with a concrete example.** Pick a small input and **trace your code by hand**, out loud, line by line. This is where you catch your own bugs — do it *before* the interviewer points at one.

**6. EDGE CASES.** Explicitly list: empty input, single element, all-same, overflow, negative, max size. Say how your code handles each (or add a guard).

**7. OPTIMIZE.** "It works. Can I do better? The hash map is O(n) space — if the array were sorted I could use two pointers for O(1) space." Even if they don't ask, *mention* the tradeoff.

### Behaviors that score points
- **Think aloud.** Silence is the #1 way to lose this round.
- **Compile in your head.** Don't write code you couldn't dry-run.
- **Use `const`, references, `auto`** — signals C++ maturity.
- **Mention allocations / cache** when relevant ("`reserve` here to avoid reallocs").
- **Recover gracefully** from a bug: "Let me trace this… ah, off-by-one, the loop should be `< n`."

### Things that lose points
- Going silent for 90 seconds.
- Coding before clarifying.
- Not testing, then being surprised it's wrong.
- `using namespace std;` is fine in interviews, but know it's bad in headers.
- Ignoring overflow (`int` sums of large arrays).

---

## 2. C++ STL fluency cheat sheet

### Containers — quick reference

| Container | Header | Lookup | Insert | Ordered? | Notes |
|---|---|---|---|---|---|
| `vector<T>` | `<vector>` | O(1) index | O(1) amortized push_back | by insertion | contiguous, cache-friendly, default choice |
| `array<T,N>` | `<array>` | O(1) | — fixed size | by index | stack-allocated, size known at compile time |
| `string` | `<string>` | O(1) index | O(1) amortized | sequence | like `vector<char>` + text ops |
| `deque<T>` | `<deque>` | O(1) index | O(1) front & back | by insertion | push_front/back O(1); not contiguous |
| `list<T>` | `<list>` | O(n) | O(1) at known iter | by insertion | doubly linked; rarely needed |
| `map<K,V>` | `<map>` | O(log n) | O(log n) | **sorted by key** | red-black tree; ordered iteration |
| `unordered_map<K,V>` | `<unordered_map>` | O(1) avg | O(1) avg | no | hash table; default for "dictionary" |
| `set<T>` | `<set>` | O(log n) | O(log n) | **sorted** | unique keys, ordered |
| `unordered_set<T>` | `<unordered_set>` | O(1) avg | O(1) avg | no | unique keys, hashed |
| `stack<T>` | `<stack>` | top O(1) | push O(1) | LIFO | adapter over deque |
| `queue<T>` | `<queue>` | front O(1) | push O(1) | FIFO | adapter over deque |
| `priority_queue<T>` | `<queue>` | top O(1) | push O(log n) | **max-heap default** | binary heap over vector |

### Construction & access idioms

```cpp
vector<int> v;                  v.reserve(1000);      // avoid reallocations
vector<int> v2(n, 0);           // n zeros
vector<vector<int>> grid(r, vector<int>(c, 0)); // 2D grid r x c
array<int, 4> a = {1,2,3,4};

unordered_map<string,int> cnt;  cnt.reserve(1000);    // also worth reserving
map<int,int> ordered;           // iterate in sorted key order

pair<int,int> p = {3, 4};       // p.first, p.second
auto [x, y] = p;                // structured bindings (C++17)
tuple<int,string,double> t{1,"a",2.0};  auto& [i,s,d] = t;
```

### Algorithms (`#include <algorithm>`, `<numeric>`)

```cpp
sort(v.begin(), v.end());                       // ascending
sort(v.begin(), v.end(), greater<int>());       // descending
sort(v.begin(), v.end(), [](auto&a, auto&b){ return a.second < b.second; }); // custom

// Binary search — REQUIRES sorted range
bool found = binary_search(v.begin(), v.end(), x);
auto lo = lower_bound(v.begin(), v.end(), x);   // first elem >= x
auto hi = upper_bound(v.begin(), v.end(), x);   // first elem >  x
int idx = lo - v.begin();                        // convert iter -> index
int countX = upper_bound(...) - lower_bound(...);// # of occurrences of x

int s   = accumulate(v.begin(), v.end(), 0);     // sum (note: 0 is int; use 0LL for long long)
long long s2 = accumulate(v.begin(), v.end(), 0LL);
auto mx = *max_element(v.begin(), v.end());
auto mn = *min_element(v.begin(), v.end());
int cnt2 = count(v.begin(), v.end(), x);
reverse(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());    // dedup (sort first!)
fill(v.begin(), v.end(), 0);
iota(v.begin(), v.end(), 0);                     // 0,1,2,3,...
auto it = find(v.begin(), v.end(), x);           // O(n) linear
nth_element(v.begin(), v.begin()+k, v.end());    // O(n) kth element / partial
partial_sort(v.begin(), v.begin()+k, v.end());   // top-k sorted
next_permutation(v.begin(), v.end());            // permutations
min(a,b)  max(a,b)  min({a,b,c})  swap(a,b)  abs(x)  __builtin_popcount(x)
```

### Iterators

```cpp
for (auto it = m.begin(); it != m.end(); ++it) { it->first; it->second; }
for (const auto& [k, v] : m) { ... }             // range-for, C++17 (preferred)
for (auto& x : vec) x *= 2;                       // by reference to mutate
for (const auto& x : vec) sum += x;               // const ref: no copy, read-only
```

### `emplace_back` vs `push_back`, `reserve`

```cpp
vector<pair<int,int>> v;
v.push_back({1, 2});        // constructs a pair, then moves/copies it in
v.emplace_back(1, 2);       // constructs the pair IN PLACE from args — avoids a temporary

v.reserve(N);               // pre-allocate capacity for N -> no reallocations during pushes
// Rule of thumb: if you know the size, reserve. Mention this at SIG — they care about allocs.
```

### Gotchas (interviewers probe these)

**1. `operator[]` on a map DEFAULT-CONSTRUCTS missing keys.**
```cpp
unordered_map<int,int> m;
if (m[5] == 0) { ... }   // BUG-ish: this INSERTS key 5 with value 0!
// To check existence without inserting:
if (m.count(5)) { ... }              // or m.find(5) != m.end()
if (m.contains(5)) { ... }           // C++20
// operator[] is great for counting though: ++cnt[x]; works because missing -> 0.
```

**2. Iterator invalidation.**
```cpp
// vector: push_back/insert/erase can REALLOCATE and invalidate ALL iterators/pointers.
for (auto it = v.begin(); it != v.end(); ) {
    if (bad(*it)) it = v.erase(it);   // erase returns next valid iterator — use it!
    else ++it;
}
// unordered_map/set: insert may rehash -> invalidates iterators (but not references to elements
//   in node-based maps... still, don't iterate while inserting).
// map/set/list: erase invalidates only the erased iterator.
```

**3. Integer overflow.** `int` is ~2.1e9. Sum of many large ints overflows — use `long long` / `0LL`. `mid = (lo+hi)/2` can overflow; use `lo + (hi-lo)/2`.

**4. Signed/unsigned.** `v.size()` is `size_t` (unsigned). `v.size() - 1` when empty = huge number. Guard with `if (!v.empty())` or cast `(int)v.size()`.

**5. Dangling references.** Don't return a reference to a local; don't hold a reference into a vector across a `push_back`.

**6. `priority_queue` is a MAX-heap by default.** For a min-heap:
```cpp
priority_queue<int, vector<int>, greater<int>> minHeap;
```

---

## 3. Core patterns with minimal C++ templates

### Two pointers
```cpp
// e.g., is there a pair summing to target in a SORTED array
bool twoSum(const vector<int>& a, int target) {
    int i = 0, j = (int)a.size() - 1;
    while (i < j) {
        int s = a[i] + a[j];
        if (s == target) return true;
        else if (s < target) ++i;
        else --j;
    }
    return false;
}
```

### Sliding window (variable size)
```cpp
// longest subarray with sum <= K (non-negative elems)
int longestWindow(const vector<int>& a, int K) {
    int left = 0, sum = 0, best = 0;
    for (int right = 0; right < (int)a.size(); ++right) {
        sum += a[right];
        while (sum > K) sum -= a[left++];   // shrink from left
        best = max(best, right - left + 1);
    }
    return best;
}
```

### Hashing / frequency map
```cpp
unordered_map<int,int> freq;
for (int x : a) ++freq[x];          // count occurrences (operator[] -> 0 for new keys)
// anagram check: build freq from s, decrement with t, all must hit zero.
```

### Prefix sums
```cpp
vector<long long> pre(n + 1, 0);
for (int i = 0; i < n; ++i) pre[i+1] = pre[i] + a[i];
// sum of a[l..r] inclusive = pre[r+1] - pre[l]
```

### Binary search on the answer
```cpp
// smallest feasible value in [lo, hi] where feasible() is monotonic (false...false,true...true)
int binarySearchAnswer(int lo, int hi, auto feasible) {
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;       // overflow-safe
        if (feasible(mid)) hi = mid;        // mid works, try smaller
        else lo = mid + 1;                  // mid too small
    }
    return lo;
}
```

### Sorting + greedy
```cpp
// e.g., max non-overlapping intervals: sort by end time, greedily take earliest end.
sort(iv.begin(), iv.end(), [](auto& a, auto& b){ return a.second < b.second; });
int end = INT_MIN, taken = 0;
for (auto& [s, e] : iv) if (s >= end) { ++taken; end = e; }
```

### Monotonic stack
```cpp
// next greater element to the right for each index
vector<int> nextGreater(const vector<int>& a) {
    int n = a.size();
    vector<int> res(n, -1);
    stack<int> st;                          // holds indices, values decreasing
    for (int i = 0; i < n; ++i) {
        while (!st.empty() && a[st.top()] < a[i]) {
            res[st.top()] = a[i];           // a[i] is the next greater for st.top()
            st.pop();
        }
        st.push(i);
    }
    return res;
}
```

### BFS (graph / grid)
```cpp
int bfs(int start, const vector<vector<int>>& adj) {
    int n = adj.size();
    vector<int> dist(n, -1);
    queue<int> q;
    dist[start] = 0; q.push(start);
    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (int v : adj[u]) if (dist[v] == -1) {
            dist[v] = dist[u] + 1;
            q.push(v);
        }
    }
    return 0; // (dist now holds shortest unweighted distances)
}
```

### DFS (recursive) + tree traversal
```cpp
void dfs(int u, const vector<vector<int>>& adj, vector<bool>& vis) {
    vis[u] = true;
    for (int v : adj[u]) if (!vis[v]) dfs(v, adj, vis);
}

struct TreeNode { int val; TreeNode* left; TreeNode* right; };
void inorder(TreeNode* node, vector<int>& out) {
    if (!node) return;
    inorder(node->left, out);
    out.push_back(node->val);
    inorder(node->right, out);
}
```

### Recursion / backtracking
```cpp
// all subsets
void subsets(int i, vector<int>& a, vector<int>& cur, vector<vector<int>>& res) {
    if (i == (int)a.size()) { res.push_back(cur); return; }
    subsets(i+1, a, cur, res);              // exclude a[i]
    cur.push_back(a[i]);
    subsets(i+1, a, cur, res);              // include a[i]
    cur.pop_back();                         // undo (backtrack)
}
```

### DP — 1D
```cpp
// climbing stairs / fib: ways[i] = ways[i-1] + ways[i-2]
int ways(int n) {
    if (n <= 2) return n;
    int a = 1, b = 2;
    for (int i = 3; i <= n; ++i) { int c = a + b; a = b; b = c; }
    return b;
}
```

### DP — 0/1 knapsack
```cpp
int knapsack(const vector<int>& w, const vector<int>& val, int W) {
    vector<int> dp(W + 1, 0);               // dp[c] = best value with capacity c
    for (int i = 0; i < (int)w.size(); ++i)
        for (int c = W; c >= w[i]; --c)     // iterate capacity DOWNWARD for 0/1
            dp[c] = max(dp[c], dp[c - w[i]] + val[i]);
    return dp[W];
}
```

### Heap / priority_queue
```cpp
priority_queue<int> maxHeap;                                  // top = largest
priority_queue<int, vector<int>, greater<int>> minHeap;       // top = smallest
// kth largest: push all, keep minHeap of size k, top() is the answer.
```

---

## 4. The HFT-flavored problems (fully worked)

> These are the problems that actually show up at trading firms. Know them cold.

### (a) LRU Cache — *extremely common*

**Problem.** Design a cache with capacity `C` supporting `get(key)` and `put(key, value)`, both in **O(1)**. When full, evict the least-recently-used entry. A `get` or `put` counts as a use.

**Approach.** Hash map for O(1) lookup + **doubly linked list** for O(1) reordering. Map key → list iterator/node. Most-recently-used at the front, LRU at the back. On access, splice the node to the front. On overflow, drop the back.

**Complexity.** `get`/`put` O(1) time, O(C) space.

```cpp
#include <list>
#include <unordered_map>
using namespace std;

class LRUCache {
    int cap;
    list<pair<int,int>> items;                          // front = MRU, back = LRU; {key,value}
    unordered_map<int, list<pair<int,int>>::iterator> pos;  // key -> node in list
public:
    explicit LRUCache(int capacity) : cap(capacity) {
        pos.reserve(capacity * 2);
    }

    int get(int key) {
        auto it = pos.find(key);
        if (it == pos.end()) return -1;                 // miss
        // move accessed node to front (most-recently-used)
        items.splice(items.begin(), items, it->second); // O(1) splice, no realloc
        return it->second->second;                      // iterator -> pair -> value
    }

    void put(int key, int value) {
        auto it = pos.find(key);
        if (it != pos.end()) {                          // existing key: update + promote
            it->second->second = value;
            items.splice(items.begin(), items, it->second);
            return;
        }
        if ((int)items.size() == cap) {                 // evict LRU from back
            int lruKey = items.back().first;
            pos.erase(lruKey);
            items.pop_back();
        }
        items.emplace_front(key, value);                // insert as MRU
        pos[key] = items.begin();
    }
};
```
*Talking points:* `std::list::splice` is O(1) and doesn't invalidate iterators — that's why we store iterators in the map. In production you'd hand-roll the linked list (nodes in an arena/pool) to avoid `std::list`'s per-node allocation and improve cache locality.

---

### (b) Limit Order Book — *signature trading question*

**Problem.** Process a stream of limit orders (BUY/SELL at a price, with quantity). Maintain the book; report the **best bid** (highest buy price) and **best ask** (lowest sell price). Incoming orders match against the opposite side when prices cross.

**Approach.** Two price-ordered maps:
- **Bids:** `map<price, qty, greater<>>` → highest price first (best bid = `begin()`).
- **Asks:** `map<price, qty>` → lowest price first (best ask = `begin()`).

A new BUY matches against asks while `buyPrice >= bestAsk`. Remaining quantity rests in the book. (Aggregated-by-price model; real books also track time priority within a level via a queue.)

**Complexity.** Each price-level op is O(log L) where L = number of distinct price levels.

```cpp
#include <map>
#include <iostream>
using namespace std;

class OrderBook {
    // price -> total resting quantity at that price
    map<int, int, greater<int>> bids;   // descending: best (highest) bid at begin()
    map<int, int>              asks;     // ascending:  best (lowest)  ask at begin()
public:
    // returns filled quantity; remainder rests in the book
    int addBuy(int price, int qty) {
        int filled = 0;
        // match while there is an ask we can afford
        while (qty > 0 && !asks.empty() && asks.begin()->first <= price) {
            auto best = asks.begin();
            int take = min(qty, best->second);
            qty       -= take;
            filled    += take;
            best->second -= take;
            if (best->second == 0) asks.erase(best);    // price level exhausted
        }
        if (qty > 0) bids[price] += qty;                // rest remaining on bid side
        return filled;
    }

    int addSell(int price, int qty) {
        int filled = 0;
        while (qty > 0 && !bids.empty() && bids.begin()->first >= price) {
            auto best = bids.begin();
            int take = min(qty, best->second);
            qty       -= take;
            filled    += take;
            best->second -= take;
            if (best->second == 0) bids.erase(best);
        }
        if (qty > 0) asks[price] += qty;
        return filled;
    }

    // -1 sentinel when a side is empty
    int bestBid() const { return bids.empty() ? -1 : bids.begin()->first; }
    int bestAsk() const { return asks.empty() ? -1 : asks.begin()->first; }
    int spread()  const {
        if (bids.empty() || asks.empty()) return -1;
        return bestAsk() - bestBid();
    }
};
```
*Talking points:* Best bid/ask in O(1) via `begin()`. For full per-order priority you'd map `price -> queue<Order>` (FIFO time priority) plus a `map<orderId, location>` for O(1) cancels. At HFT scale people replace the tree with **arrays indexed by price ticks** (prices are discrete) for O(1) level access and cache locality.

---

### (c) Rate Limiter / sliding-window counter

**Problem.** Allow at most `N` events per rolling time window `W`. `allow(timestamp)` returns whether the event is permitted.

**Approach.** Keep a `deque` of timestamps. On each call, pop timestamps older than `now - W` off the front, then check size.

**Complexity.** Amortized O(1) per call (each timestamp pushed/popped once); O(N) space.

```cpp
#include <deque>
using namespace std;

class RateLimiter {
    int maxEvents;
    long long windowSize;          // e.g. milliseconds
    deque<long long> events;       // timestamps, oldest at front
public:
    RateLimiter(int n, long long w) : maxEvents(n), windowSize(w) {}

    bool allow(long long now) {
        // evict timestamps that fell out of the window
        while (!events.empty() && events.front() <= now - windowSize)
            events.pop_front();
        if ((int)events.size() < maxEvents) {
            events.push_back(now);
            return true;
        }
        return false;              // limit hit
    }
};
```
*Talking points:* Mention the simpler **fixed-window counter** (just a count + window-start, O(1) memory, but allows bursts at window edges) and the **token bucket** (refill rate r, capacity c) as alternatives — bucket is the standard production choice.

---

### (d) Min Stack (O(1) min)

**Problem.** Stack with `push`, `pop`, `top`, and `getMin` all in O(1).

**Approach.** Auxiliary stack tracking the minimum at each level. (Same idea gives a max-stack.)

**Complexity.** All ops O(1). O(n) space.

```cpp
#include <stack>
#include <algorithm>
using namespace std;

class MinStack {
    stack<int> data;
    stack<int> mins;               // mins.top() == current minimum of data
public:
    void push(int x) {
        data.push(x);
        mins.push(mins.empty() ? x : min(x, mins.top()));
    }
    void pop()      { data.pop(); mins.pop(); }
    int  top()      { return data.top(); }
    int  getMin()   { return mins.top(); }
    bool empty()    { return data.empty(); }
};
```
*Talking points:* Space-optimization: only push to `mins` when `x <= mins.top()`, and pop from `mins` only when `data.top() == mins.top()`. Or store `(value, runningMin)` pairs. For a max-stack swap `min` → `max`.

---

### (e) SPSC ring buffer / circular buffer (array-based)

**Problem.** Fixed-capacity FIFO over a contiguous array. (HFT uses these as lock-free queues between a producer thread and a consumer thread — Single Producer, Single Consumer.)

**Approach.** Array + `head`/`tail` indices that wrap modulo capacity. Use a power-of-two capacity so wrap is a bitmask (`& (cap-1)`). For the lock-free SPSC version, head is written only by the consumer and tail only by the producer, with `atomic` + acquire/release ordering.

**Complexity.** push/pop O(1), no allocation after construction.

```cpp
#include <vector>
#include <atomic>
#include <cstddef>
using namespace std;

// Single-threaded version first (clean to write live):
template <typename T>
class RingBuffer {
    vector<T> buf;
    size_t head = 0;    // read index
    size_t tail = 0;    // write index
    size_t count = 0;
public:
    explicit RingBuffer(size_t capacity) : buf(capacity) {}
    bool full()  const { return count == buf.size(); }
    bool empty() const { return count == 0; }

    bool push(const T& x) {
        if (full()) return false;
        buf[tail] = x;
        tail = (tail + 1) % buf.size();   // wrap
        ++count;
        return true;
    }
    bool pop(T& out) {
        if (empty()) return false;
        out = buf[head];
        head = (head + 1) % buf.size();
        --count;
        return true;
    }
};

// Lock-free SPSC variant (mention if they push on concurrency):
template <typename T>
class SPSCQueue {
    vector<T> buf;
    const size_t mask;                                // cap is power of two
    alignas(64) atomic<size_t> head{0};               // consumer writes; cache-line padded
    alignas(64) atomic<size_t> tail{0};               // producer writes
public:
    explicit SPSCQueue(size_t cap) : buf(cap), mask(cap - 1) { /* cap must be pow2 */ }

    bool push(const T& x) {                           // producer only
        size_t t = tail.load(memory_order_relaxed);
        size_t next = (t + 1) & mask;
        if (next == head.load(memory_order_acquire)) return false; // full
        buf[t] = x;
        tail.store(next, memory_order_release);       // publish
        return true;
    }
    bool pop(T& out) {                                // consumer only
        size_t h = head.load(memory_order_relaxed);
        if (h == tail.load(memory_order_acquire)) return false;    // empty
        out = buf[h];
        head.store((h + 1) & mask, memory_order_release);
        return true;
    }
};
```
*Talking points:* Power-of-two capacity → `& mask` instead of `% cap`. `alignas(64)` separates head/tail onto different cache lines to avoid **false sharing**. `memory_order_acquire/release` gives the right visibility without a full barrier. This is *exactly* the kind of thing SIG loves — bring it up.

---

### (f) Parse / tokenize a market-data stream

**Problem.** Parse messages like `"BUY,AAPL,100,150.25"` (side, symbol, qty, price). Tokenize and convert to a struct. Generalize to splitting on a delimiter.

**Approach.** Split on the delimiter, then convert fields. Use `std::string_view` + manual scanning for zero-copy/low-latency parsing; show the clean version first.

**Complexity.** O(length of message).

```cpp
#include <string>
#include <string_view>
#include <vector>
#include <charconv>     // from_chars: fast, no-allocation numeric parse
using namespace std;

struct Order {
    string_view side;
    string_view symbol;
    int    qty   = 0;
    double price = 0.0;
};

// Generic split (returns views into the original buffer — zero copy)
vector<string_view> split(string_view s, char delim) {
    vector<string_view> out;
    size_t start = 0;
    while (start <= s.size()) {
        size_t pos = s.find(delim, start);
        if (pos == string_view::npos) { out.push_back(s.substr(start)); break; }
        out.push_back(s.substr(start, pos - start));
        start = pos + 1;
    }
    return out;
}

// Parse one CSV-style market-data line
bool parseOrder(string_view line, Order& o) {
    auto f = split(line, ',');
    if (f.size() != 4) return false;                  // malformed -> reject
    o.side   = f[0];
    o.symbol = f[1];
    // from_chars: fast integer parse, returns error code, no allocation/locale
    auto r1 = from_chars(f[2].data(), f[2].data() + f[2].size(), o.qty);
    if (r1.ec != errc{}) return false;
    // NOTE: floating-point from_chars exists in C++17 but is NOT yet implemented in
    // libc++ (Clang/macOS) as of clang 17. It works on GCC/libstdc++. For portability,
    // either keep prices as scaled integers (ticks!) or fall back to strtod:
    auto r2 = from_chars(f[3].data(), f[3].data() + f[3].size(), o.price);
    if (r2.ec != errc{}) return false;
    return true;
}

// Portable price parse if from_chars<double> is unavailable (needs a NUL-terminated buffer,
// hence the temporary string; on the real hot path you'd use integer "ticks" instead):
bool parsePrice(string_view sv, double& out) {
    string tmp(sv);                 // strtod needs a C-string
    char* end = nullptr;
    out = strtod(tmp.c_str(), &end);
    return end == tmp.c_str() + tmp.size();   // whole field consumed
}
```
*Talking points:* `std::from_chars` is the low-latency way to parse numbers — no allocations, no locale, faster than `stoi`/`stod`/`sstream`. Mention the **portability gotcha**: integer `from_chars` is everywhere, but the floating-point overload is missing in libc++ (Clang) for years — a good thing to flag, and a reason HFT systems store prices as **scaled integers (ticks)** rather than `double` anyway (exact, fast, no FP parse). `string_view` avoids copying substrings. For *real* HFT you parse a **binary** protocol (fixed-offset fields), not CSV, but the principle — no allocations on the hot path — is the same.

---

### (g) Merge k sorted lists / streams (heap)

**Problem.** Merge `k` sorted sequences into one sorted sequence (think: merging quote streams from k venues).

**Approach.** Min-heap of `(value, whichList, indexInList)`. Pop the smallest, output it, push the next element from that same list.

**Complexity.** O(N log k), N = total elements, k = number of lists. O(k) heap space.

```cpp
#include <vector>
#include <queue>
using namespace std;

vector<int> mergeK(const vector<vector<int>>& lists) {
    // node = {value, listIndex, elemIndex}; min-heap by value
    using Node = tuple<int,int,int>;
    priority_queue<Node, vector<Node>, greater<Node>> pq;

    for (int i = 0; i < (int)lists.size(); ++i)
        if (!lists[i].empty())
            pq.emplace(lists[i][0], i, 0);            // seed with each list's first elem

    vector<int> out;
    while (!pq.empty()) {
        auto [val, li, ei] = pq.top(); pq.pop();
        out.push_back(val);
        if (ei + 1 < (int)lists[li].size())           // push next from same list
            pq.emplace(lists[li][ei + 1], li, ei + 1);
    }
    return out;
}
```
*Talking points:* Heap stays size ≤ k, so each of the N pops/pushes is O(log k). Linked-list variant uses the same heap of node pointers.

---

### (h) Running median (two heaps)

**Problem.** Stream of numbers; after each insert, report the median in O(log n) insert / O(1) query.

**Approach.** A **max-heap** for the lower half and a **min-heap** for the upper half. Keep sizes balanced (differ by ≤ 1). Median is the top of the larger heap, or the average of both tops when equal.

**Complexity.** `add` O(log n), `median` O(1).

```cpp
#include <queue>
using namespace std;

class MedianFinder {
    priority_queue<int> lo;                                   // max-heap: smaller half
    priority_queue<int, vector<int>, greater<int>> hi;        // min-heap: larger half
public:
    void add(int x) {
        // route into the correct half
        if (lo.empty() || x <= lo.top()) lo.push(x);
        else                              hi.push(x);
        // rebalance so |lo| - |hi| is in {0, 1}
        if (lo.size() > hi.size() + 1) { hi.push(lo.top()); lo.pop(); }
        else if (hi.size() > lo.size()) { lo.push(hi.top()); hi.pop(); }
    }
    double median() const {
        if (lo.size() > hi.size()) return lo.top();           // odd count
        return (lo.top() + hi.top()) / 2.0;                   // even count
    }
};
```
*Talking points:* Invariant `lo.size() == hi.size()` or `lo.size() == hi.size() + 1`. For a *sliding-window* median you'd need lazy deletion or a balanced BST / `multiset` with a mid-iterator.

---

### (i) Top-K frequent elements

**Problem.** Given an array, return the `k` most frequent elements.

**Approach.** Count frequencies in a hash map, then keep a **min-heap of size k** keyed by frequency (evict the smallest when it overflows). The k survivors are the answer.

**Complexity.** O(n log k) time, O(n) space. (A bucket-sort approach gives O(n).)

```cpp
#include <vector>
#include <unordered_map>
#include <queue>
using namespace std;

vector<int> topKFrequent(const vector<int>& nums, int k) {
    unordered_map<int,int> freq;
    freq.reserve(nums.size());
    for (int x : nums) ++freq[x];                      // O(n) counting

    // min-heap of {frequency, value}; keep only the k largest frequencies
    using P = pair<int,int>;
    priority_queue<P, vector<P>, greater<P>> heap;
    for (auto& [val, f] : freq) {
        heap.emplace(f, val);
        if ((int)heap.size() > k) heap.pop();          // drop smallest freq
    }

    vector<int> out;
    while (!heap.empty()) { out.push_back(heap.top().second); heap.pop(); }
    return out;                                         // (ascending freq; reverse if needed)
}
```
*Talking points:* If interviewer wants O(n): **bucket sort** — index buckets by frequency `0..n`, scatter values into `bucket[freq]`, then sweep buckets from high to low collecting k. Mention `nth_element` / quickselect as another O(n)-average route.

---

## 5. Complexity analysis

### Common complexities table

| Big-O | Name | Example | n=10⁶ feel |
|---|---|---|---|
| O(1) | constant | hash lookup, array index | instant |
| O(log n) | logarithmic | binary search, heap op, map op | instant |
| O(n) | linear | single pass, prefix sum | fast |
| O(n log n) | linearithmic | sort, merge-k | fine (~2×10⁷ ops) |
| O(n²) | quadratic | nested loops, naive pair-sum | ~10¹² — too slow at 10⁶ |
| O(2ⁿ) | exponential | subsets, naive recursion | only tiny n (≤~25) |
| O(n!) | factorial | permutations | only n ≤ ~11 |

### Rule of thumb for input size → target complexity
- n ≤ 10–12 → O(n!) / O(2ⁿ) backtracking fine
- n ≤ a few thousand → O(n²) fine
- n ≤ 10⁶ → need O(n) or O(n log n)
- n ≤ 10⁸–10⁹ → need O(log n) or O(1) per query

### How to reason out loud
- Count the **dominant** nested loops: two nested over n → O(n²).
- A `sort` is O(n log n); a binary search inside a loop of n is O(n log n).
- **Drop constants and lower-order terms:** O(2n + 5) → O(n).
- State **time and space separately**: "O(n) time, O(n) extra space for the map."
- **Amortized vs worst-case:** `vector::push_back` is amortized O(1) (occasional O(n) realloc); `unordered_map` is O(1) *average* but O(n) worst case under adversarial hashing.
- Mention **constant factors / cache** when it matters at SIG: "Both are O(n), but the `vector` scan is cache-friendly and the `list` chases pointers, so the vector wins in practice."

### Container op complexities (memorize)
- `vector`: index O(1), push_back amortized O(1), insert/erase middle O(n)
- `unordered_map/set`: insert/find/erase O(1) average, O(n) worst
- `map/set`: insert/find/erase O(log n), in-order iteration free
- `priority_queue`: push/pop O(log n), top O(1)
- `deque`: push/pop both ends O(1), index O(1)

---

## 6. Drilling plan (solve each in C++, narrating out loud)

> Do them **in C++**, **talking through CLARIFY→APPROACH→COMPLEXITY→CODE→TEST→EDGE** every time. Speed *and* narration. Easy → medium within each group.

**Group 1 — Arrays / Hashing (warm-up, do first)**
1. Two Sum *(hash map)*
2. Contains Duplicate *(set/hash)*
3. Valid Anagram *(freq map)*
4. Group Anagrams *(hash of sorted key)*
5. Product of Array Except Self *(prefix/suffix)*

**Group 2 — Two Pointers / Sliding Window**
6. Valid Palindrome *(two pointers)*
7. 3Sum *(sort + two pointers)*
8. Best Time to Buy and Sell Stock *(running min)* ← trading-flavored
9. Longest Substring Without Repeating Characters *(sliding window)*
10. Minimum Size Subarray Sum *(sliding window)*

**Group 3 — Stack / Monotonic Stack**
11. Valid Parentheses *(stack)*
12. Min Stack *(aux stack)* ← from §4
13. Daily Temperatures *(monotonic stack)*
14. Next Greater Element I/II *(monotonic stack)*

**Group 4 — Binary Search**
15. Binary Search *(template)*
16. Search in Rotated Sorted Array
17. Find First and Last Position *(lower/upper_bound)*
18. Koko Eating Bananas *(binary search on answer)*

**Group 5 — Heaps / Priority Queue**
19. Kth Largest Element in an Array *(min-heap / quickselect)*
20. Top K Frequent Elements ← from §4
21. Merge k Sorted Lists ← from §4
22. Find Median from Data Stream *(two heaps)* ← from §4

**Group 6 — Linked List**
23. Reverse Linked List *(iterative pointers)*
24. Linked List Cycle *(Floyd's two pointers)*
25. LRU Cache *(hashmap + DLL)* ← from §4, **highest priority**

**Group 7 — Trees / Graphs (BFS/DFS)**
26. Invert Binary Tree / Max Depth *(DFS)*
27. Binary Tree Level Order Traversal *(BFS)*
28. Number of Islands *(grid BFS/DFS)*
29. Course Schedule *(topological sort / cycle detection)*

**Group 8 — DP (light — SIG rarely goes hard here)**
30. Climbing Stairs *(1D DP)*
31. House Robber *(1D DP)*
32. Coin Change *(unbounded knapsack)*

**Group 9 — Trading-specific design (do these last, they're the differentiators)**
33. Design a Limit Order Book *(maps)* ← from §4
34. Design a Rate Limiter *(sliding window / token bucket)* ← from §4
35. Implement a Ring Buffer / circular queue ← from §4 (and the SPSC variant)
36. Parse market-data messages *(string_view + from_chars)* ← from §4

**Priority order if time-crunched:** #25 LRU, #33 Order Book, #35 Ring Buffer, #22 Running Median, #12 Min Stack, then Groups 1–4. These are the ones SIG-style interviews lean on.

---

## 7. Self-test checklist

**Meta / communication**
- [ ] I clarify input ranges, nulls, duplicates, and return-on-invalid **before** coding.
- [ ] I state approach + Big-O **before** writing code, and wait for buy-in.
- [ ] I narrate continuously — no >15s silences.
- [ ] I trace my own code on a small example **before** saying "done".
- [ ] I enumerate edge cases: empty, size 1, all-equal, negative, overflow, max size.
- [ ] I propose an optimization or tradeoff even when not asked.

**C++ / STL fluency**
- [ ] I know which container to reach for and its op complexities.
- [ ] I use `reserve` when size is known and can explain why (alloc cost).
- [ ] I avoid the `operator[]` default-construction trap; use `count`/`find`/`contains`.
- [ ] I know `lower_bound`/`upper_bound` semantics and that they need a sorted range.
- [ ] I default `priority_queue` to max-heap and can make a min-heap (`greater<>`).
- [ ] I avoid iterator invalidation (use the return value of `erase`).
- [ ] I use `long long` / `0LL` to avoid overflow in sums and midpoints.
- [ ] I use `const auto&` in range-for to avoid copies.

**Patterns — can I write from scratch in <5 min each?**
- [ ] Two pointers, sliding window, prefix sums.
- [ ] Binary search (plain + on the answer), overflow-safe mid.
- [ ] Monotonic stack (next greater).
- [ ] BFS and DFS on a grid and a graph; tree traversals.
- [ ] Backtracking (subsets/permutations) with proper undo.
- [ ] 1D DP and 0/1 knapsack (downward capacity loop).
- [ ] Min-heap / max-heap usage.

**HFT problems — can I solve cold, correct, commented?**
- [ ] LRU cache (O(1) get/put, list::splice).
- [ ] Limit order book (best bid/ask in O(1), matching loop).
- [ ] Rate limiter (sliding window) + token-bucket alternative.
- [ ] Min/max stack (O(1) min).
- [ ] Ring buffer (and SPSC lock-free variant, false sharing, memory_order).
- [ ] Parse market data (`string_view`, `from_chars`, no hot-path allocs).
- [ ] Merge k sorted lists (heap, O(N log k)).
- [ ] Running median (two heaps, balance invariant).
- [ ] Top-k frequent (heap O(n log k) + bucket-sort O(n)).

**Complexity**
- [ ] I can map input size → required complexity instantly.
- [ ] I state time and space separately, and distinguish amortized/average/worst.
- [ ] I can argue cache-friendliness (vector vs list) when both are same Big-O.
