# 01 — C++ Fundamentals & Internals

## Why this matters for SIG

Susquehanna's Trading System Engineering org builds **low-latency C++ trading systems**. In an HFT shop, the difference between a profitable and an unprofitable strategy can be nanoseconds, so the interviewers are not testing whether you can write a `for` loop — they assume that. They are testing whether you understand **what the machine actually does** when your code runs:

- Where does this object live (stack, heap, static storage)? How many cache lines does it touch?
- Does this line copy, move, or elide? Does it allocate?
- Is this a virtual call (branch + indirect jump, defeats inlining) or a static call?
- Is this code Undefined Behavior, and could the optimizer therefore delete it?

SIG-style interviews reward **precise mechanical answers**. "A `shared_ptr` is reference counted" is a junior answer. "A `shared_ptr` holds two pointers — one to the object, one to a heap control block containing an atomic strong count and an atomic weak count; copying it does an atomic increment, which is a locked RMW that can cost ~15-40ns under contention, which is why we avoid passing them by value on the hot path" is the answer that gets you to the next round. This guide aims for that level throughout.

A note on your background: you do low-latency C/C++ networking at AWS, so you already have the *systems* intuition (kernel, sockets, cache, syscalls). The gap is **C++ language semantics** — move semantics, RAII, templates, the object model. This file closes that gap.

---

## 1. The process memory layout — where do variables live?

When the OS loads your executable, the virtual address space is carved into segments. From low to high addresses (typical Linux/x86-64):

```
high addr  ┌─────────────────────┐
           │   command-line args │
           │   & environment     │
           ├─────────────────────┤
           │       STACK         │  grows DOWN ↓ (toward lower addresses)
           │         │           │
           │         ▼           │
           │                     │
           │   (unmapped gap)    │
           │                     │
           │         ▲           │
           │         │           │
           │        HEAP         │  grows UP ↑ (malloc/new)
           ├─────────────────────┤
           │   BSS  (.bss)       │  zero-initialized / uninit globals & statics
           ├─────────────────────┤
           │   DATA (.data)      │  initialized globals & statics
           ├─────────────────────┤
           │   TEXT (.text)      │  machine code (read-only, executable)
low addr   └─────────────────────┘
```

| Segment | Holds | Lifetime | Notes |
|---|---|---|---|
| **.text** | compiled machine code, string literals (often in `.rodata`) | program lifetime | read-only, shared between processes |
| **.data** | globals/statics with a non-zero initializer | program lifetime | initialized from the binary on disk |
| **.bss** | globals/statics that are zero or uninitialized | program lifetime | takes **no space on disk**, zeroed by the loader |
| **heap** | `new`/`malloc` allocations | until you `delete`/`free` | dynamic size, managed by the allocator |
| **stack** | locals, function args, return addresses, saved registers | until the function returns | per-thread, fast (just move the stack pointer) |

**Where things live:**

```cpp
int g_init = 42;          // .data  (global, non-zero init)
int g_zero;               // .bss   (global, zero-init)
static int s = 7;         // .data  (file/function-static)
const char* lit = "hi";   // pointer in .data/.bss/stack; "hi" itself in .rodata

void f() {
    int x = 1;            // stack
    int* p = new int(5);  // p is on the stack; *p (the int 5) is on the heap
    static int counter;   // .bss — lives for whole program, but scoped to f
}                         // x and p destroyed here; the heap int 5 LEAKS unless delete'd
```

**Stack vs heap — the practical contrast:**

- **Stack**: allocation is *free* (just decrement the stack pointer by the frame size at function entry). Deallocation is automatic on scope exit. Excellent cache locality. Limited size (typically 1–8 MB; blowing it = stack overflow). Size must be known at compile time (no runtime-sized arrays, modulo VLAs which are non-standard).
- **Heap**: allocation goes through an allocator (`malloc`/`operator new`) — can take hundreds of ns, may lock, may touch a free list, can fragment, can be a syscall (`mmap`/`sbrk`) on growth. You must free it manually (or via RAII). Required for objects that outlive their creating scope or whose size/lifetime is dynamic.

> 🎤 **If they ask: "Where do local variables, globals, and `new`'d objects live?"**
> "Locals and function arguments go on the stack — allocation is just moving the stack pointer, freed automatically when the function returns. Globals and statics live in the data segment: `.data` if they have a non-zero initializer, `.bss` if they're zero-initialized — `.bss` takes no space in the binary. `new`/`malloc` returns heap memory that lives until I free it; the *pointer* might be on the stack but the *pointee* is on the heap. Code and string literals are in the read-only text/rodata segment. On the hot path I keep things on the stack because it's cache-friendly and allocation-free."

---

## 2. Pointers vs references

A **pointer** is a variable that stores an address. It can be null, can be reassigned to point elsewhere, has its own storage, and supports arithmetic. A **reference** is an *alias* for an existing object — another name for the same storage. It must be bound at initialization, can never be rebound, and (in the abstract machine) cannot be null.

```cpp
int a = 1, b = 2;

int* p = &a;   // p points to a
p = &b;        // OK: rebind p to b
p = nullptr;   // OK: pointers can be null
*p = 5;        // would be UB here (null deref); otherwise writes through p

int& r = a;    // r is an alias for a; MUST initialize
r = b;         // NOT rebinding — this assigns b's VALUE into a (a becomes 2)
int& r2;       // ERROR: references must be initialized
```

Key differences:

| | Pointer | Reference |
|---|---|---|
| Can be null | Yes | No (in well-defined code) |
| Rebindable | Yes | No — bound once |
| Must be initialized | No | Yes |
| Own storage / `sizeof` | Yes (8 bytes) | Conceptually none; usually compiled to a pointer under the hood |
| Arithmetic | Yes | No |
| Syntax to access | `*p`, `p->` | use it like the object directly |

**When to use which:**
- **Reference**: function parameters you want to pass without copying and that are always valid (`const T&` for read-only large objects, `T&` for output/in-out params). Return type when returning an existing object (e.g. `operator[]`). Default choice when "null" is not a meaningful state.
- **Pointer**: when null is a legitimate value ("not found", "optional"), when you need to rebind/iterate (`p++`), when interfacing with C APIs, when you need to express ownership transfer (prefer smart pointers). For "optional non-owning reference," a raw pointer or `std::optional<std::reference_wrapper<T>>` works.

**Dangling pointer/reference** — points to memory whose lifetime has ended. Dereferencing it is **Undefined Behavior**:

```cpp
int* dangling() {
    int x = 42;
    return &x;          // x dies when the function returns — returned pointer dangles
}
const int& bad() {
    return 5;           // binds to a temporary that dies at the end of the full expression
}
std::string_view sv = std::string("temp");  // sv dangles: the string is destroyed immediately
```

> 🎤 **If they ask: "What's the difference between a pointer and a reference?"**
> "A pointer is a variable holding an address — it can be null, reassigned, and you can do arithmetic on it. A reference is an alias for an existing object: it must be initialized when declared, can never be rebound, and isn't null in well-defined code. Under the hood the compiler usually implements a reference as a pointer, but the language semantics differ. I reach for references when the thing is always valid and I just want to avoid a copy, and pointers (ideally smart pointers) when null is meaningful or I'm transferring ownership."

---

## 3. const correctness

`const` is a contract: "this will not be modified through this name." It enables the compiler to catch bugs and gives the optimizer guarantees.

**Pointer flavors — read right-to-left:**

```cpp
int x = 1, y = 2;

const int*       p1 = &x;   // pointer to const int: can't modify *p1, CAN rebind p1
int const*       p2 = &x;   // identical to p1 (const binds to int)
int* const       p3 = &x;   // const pointer to int: CAN modify *p3, can't rebind p3
const int* const p4 = &x;   // const pointer to const int: neither

// p1 = &y;  OK        *p1 = 5;  ERROR
// p3 = &y;  ERROR     *p3 = 5;  OK
```

Mnemonic: read the declaration right to left. `const int* const p` → "p is a const pointer to a const int."

**const member functions** promise not to modify the object's observable state. Inside one, `this` is `const T*`, so you can't write to members (unless they're `mutable`):

```cpp
class Account {
    double balance_;
    mutable int read_count_ = 0;   // mutable: changeable even in const methods
public:
    double balance() const {       // const method: callable on const objects
        ++read_count_;             // OK because read_count_ is mutable
        return balance_;
    }
    void deposit(double d) { balance_ += d; }  // non-const: not callable on const objects
};

const Account a;
a.balance();   // OK
// a.deposit(10);  ERROR: can't call non-const method on const object
```

`const` is also part of the overload set — you can have `T& at(size_t)` and `const T& at(size_t) const`; the compiler picks based on whether the object is const.

**`constexpr` vs `const`:**
- `const` = "read-only" / "I won't modify this through this name." The value may still be computed at runtime.
- `constexpr` = "this can be evaluated at **compile time**." A `constexpr` variable is a true compile-time constant; a `constexpr` function *can* run at compile time (if given constant args) but may also run at runtime.

```cpp
const int    n = compute_at_runtime();   // const but runtime value
constexpr int m = 5 * 4;                  // computed at compile time; usable as array size, template arg

constexpr int square(int v) { return v * v; }
int arr[square(8)];        // OK: 64 known at compile time
int z = square(runtime_x); // also fine — just runs at runtime here
```

Use `constexpr` to push work to compile time (zero runtime cost) — very relevant for HFT where you precompute lookup tables, masks, etc. `consteval` (C++20) forces compile-time evaluation.

> 🎤 **If they ask: "Difference between `const int*` and `int* const`? And `const` vs `constexpr`?"**
> "`const int*` is a pointer to a const int — I can repoint it but can't write through it. `int* const` is a const pointer to a mutable int — I can write through it but can't repoint it. Read declarations right-to-left. `const` means read-only and can hold a value computed at runtime; `constexpr` means it can be evaluated at compile time — a `constexpr` variable is a genuine compile-time constant usable for array sizes and template arguments, and a `constexpr` function runs at compile time when given constant inputs. I use `constexpr` to move work off the hot path entirely."

---

## 4. RAII — the single most important C++ idiom

**RAII = Resource Acquisition Is Initialization.** Tie the lifetime of a resource (heap memory, file handle, socket, mutex lock, DB connection) to the lifetime of a stack object. You **acquire in the constructor** and **release in the destructor**. Because C++ guarantees destructors run deterministically when an object leaves scope — *including when an exception unwinds the stack* — the resource is always released exactly once, on every exit path, with no manual cleanup.

```cpp
// Without RAII — error-prone:
void bad() {
    int* p = new int[100];
    if (something()) return;   // LEAK — forgot delete[]
    might_throw();             // LEAK — exception skips delete[]
    delete[] p;
}

// With RAII:
void good() {
    std::vector<int> v(100);   // memory acquired in ctor
    if (something()) return;   // freed automatically
    might_throw();             // freed automatically during stack unwinding
}                              // freed here on normal exit
```

A minimal hand-rolled RAII wrapper for a C handle:

```cpp
class FileHandle {
    FILE* f_;
public:
    explicit FileHandle(const char* path) : f_(std::fopen(path, "r")) {
        if (!f_) throw std::runtime_error("open failed");
    }
    ~FileHandle() { if (f_) std::fclose(f_); }   // released no matter what

    // make it non-copyable (a handle shouldn't be double-closed)
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;

    FILE* get() const { return f_; }
};
```

The standard library is RAII everywhere: `std::vector`/`std::string` (memory), `std::unique_ptr`/`shared_ptr` (memory), `std::lock_guard`/`std::unique_lock`/`scoped_lock` (mutexes), `std::fstream` (files), `std::jthread` (threads). The famous lock example:

```cpp
std::mutex m;
void f() {
    std::lock_guard<std::mutex> lk(m);   // locks here
    do_work();                            // even if this throws...
}                                         // ...unlock() runs in lk's destructor
```

Why it's *the* idiom: it makes resource management **exception-safe and leak-free by construction**, and it composes — a class containing RAII members is itself automatically correct (the compiler-generated destructor destroys members in reverse order). This is the foundation of the Rule of 0 (§5).

> 🎤 **If they ask: "What is RAII and why does it matter?"**
> "Resource Acquisition Is Initialization — I bind a resource's lifetime to a stack object: acquire in the constructor, release in the destructor. Because C++ runs destructors deterministically on every scope exit, including exception unwinding, the resource is always cleaned up exactly once with no manual `delete` or `unlock`. It's the most important C++ idiom because it makes code leak-free and exception-safe by construction. `unique_ptr`, `lock_guard`, and `vector` are all RAII. It's the reason idiomatic modern C++ rarely needs an explicit `delete`."

---

## 5. The Rule of 0 / 3 / 5 — special member functions

A class can have five **special member functions** that manage copying, moving, and destruction:

```cpp
class T {
    T();                              // (default constructor — separate, not part of the rule)
    ~T();                             // destructor
    T(const T&);                      // copy constructor
    T& operator=(const T&);           // copy assignment
    T(T&&) noexcept;                  // move constructor
    T& operator=(T&&) noexcept;       // move assignment
};
```

**Rule of 3** (C++98): if you need to write *any one* of {destructor, copy constructor, copy assignment}, you almost certainly need all three — because that need implies the class manages a resource (e.g. owns a raw pointer), and the compiler-generated defaults would do a shallow copy → double-free / leak.

**Rule of 5** (C++11): once you add move semantics, the set becomes five. If you declare any of the five, the compiler may suppress generation of the others (e.g. declaring a destructor suppresses the move operations, silently falling back to copies).

**Rule of 0** (the one you want to follow): **write none of them.** Compose your class out of RAII members (`vector`, `string`, `unique_ptr`, …) and let the compiler generate all five correctly by recursing into the members. This is the modern best practice — fewer bugs, less code.

```cpp
// Rule of 0 — no special members needed; all five are correct automatically:
class Order {
    std::string symbol_;
    std::vector<int> fills_;
    std::unique_ptr<Strategy> strat_;
};   // copy/move/destroy all do the right thing member-wise

// Rule of 5 — needed only when managing a raw resource yourself:
class Buffer {
    char* data_;
    size_t size_;
public:
    explicit Buffer(size_t n) : data_(new char[n]), size_(n) {}
    ~Buffer() { delete[] data_; }

    Buffer(const Buffer& o) : data_(new char[o.size_]), size_(o.size_) {
        std::copy(o.data_, o.data_ + size_, data_);          // deep copy
    }
    Buffer& operator=(const Buffer& o) {
        if (this != &o) {                                    // self-assignment check
            char* tmp = new char[o.size_];                   // allocate first (strong guarantee)
            std::copy(o.data_, o.data_ + o.size_, tmp);
            delete[] data_;
            data_ = tmp; size_ = o.size_;
        }
        return *this;
    }
    Buffer(Buffer&& o) noexcept : data_(o.data_), size_(o.size_) {
        o.data_ = nullptr; o.size_ = 0;                      // steal + null out source
    }
    Buffer& operator=(Buffer&& o) noexcept {
        if (this != &o) {
            delete[] data_;
            data_ = o.data_; size_ = o.size_;
            o.data_ = nullptr; o.size_ = 0;
        }
        return *this;
    }
};
```

> 🎤 **If they ask: "What's the Rule of 5? When do you write these functions?"**
> "The five special members are destructor, copy constructor, copy assignment, move constructor, move assignment. The Rule of 5 says if you define one, you probably need all five, because defining one usually means you own a raw resource and the shallow-copy defaults would double-free. But the rule I actually follow is the Rule of 0: I build classes out of RAII members like `vector` and `unique_ptr` and write none of the five — the compiler generates all of them correctly by handling each member. I only hand-write the five when wrapping a raw pointer or C handle, and then I mark the moves `noexcept`."

---

## 6. Move semantics — lvalues, rvalues, `std::move`, perfect forwarding, RVO

This is the topic that most distinguishes a strong C++ candidate. Take your time on it.

### 6.1 Value categories: lvalues vs rvalues

- **lvalue** ("locator value"): has a name / identity, refers to a persistent object you can take the address of. `x`, `arr[i]`, `*p`, a function returning `T&`.
- **rvalue**: a temporary / about-to-expire value with no persistent identity. Literals (`42`), arithmetic results (`a + b`), a function returning `T` by value, anything you `std::move`.

The point: if a value is an rvalue, *nobody else will observe it again*, so we can **steal its guts** (pointers, buffers) instead of copying them.

### 6.2 rvalue references `&&`

```cpp
void f(const std::string& s);  // binds to lvalues AND rvalues (copies if it needs to keep it)
void f(std::string&& s);       // binds ONLY to rvalues — overload chosen for temporaries

std::string a = "hello";
f(a);             // calls f(const string&) — a is an lvalue
f("world");       // calls f(string&&)      — temporary is an rvalue
f(std::move(a));  // calls f(string&&)      — we cast a to an rvalue
```

A **move constructor** takes `T&&` and transfers ownership of the source's resources, leaving the source in a valid-but-unspecified state (so its destructor still runs safely). Moving a `vector` of a million elements is a few pointer assignments; copying it is a million-element allocation + copy. That's the whole win.

### 6.3 `std::move` — what it actually does

`std::move(x)` does **not move anything**. It is a `static_cast<T&&>(x)` — an unconditional cast to an rvalue reference. It just *changes the type/category* so that overload resolution picks the move constructor/assignment. The actual stealing happens inside that move constructor.

```cpp
// Roughly how it's defined:
template <class T>
constexpr std::remove_reference_t<T>&& move(T&& t) noexcept {
    return static_cast<std::remove_reference_t<T>&&>(t);
}
```

So `std::move` = "I'm done with this; whoever takes it may pillage it." If the target type has no move constructor, you silently get a copy.

### 6.4 `std::forward` and perfect forwarding

In a template, `T&&` is a **forwarding (universal) reference**, not an rvalue reference — it binds to *both* lvalues and rvalues via **reference collapsing** (`& && → &`, `&& && → &&`). Inside such a template, an argument named `arg` is itself an lvalue (it has a name!), so naively passing it on would always copy. `std::forward<T>(arg)` casts it back to its *original* value category — preserving lvalue-ness or rvalue-ness — so it's moved if it came in as an rvalue and copied if it came in as an lvalue.

```cpp
template <class T>
void wrapper(T&& arg) {                 // forwarding reference
    callee(std::forward<T>(arg));       // forwards lvalue as lvalue, rvalue as rvalue
}

// emplace_back uses this to construct in place from forwarded args:
template <class... Args>
void emplace_back(Args&&... args) {
    new (slot) T(std::forward<Args>(args)...);   // perfect forwarding into the ctor
}
```

**Rule of thumb:** `std::move` for rvalue references (you know it's an rvalue), `std::forward` for forwarding references `T&&` in templates (you want to preserve whatever came in).

### 6.5 When do moves happen?

- Constructing/assigning from a temporary (`std::string s = foo();`).
- Passing/returning by value with an rvalue argument.
- Explicit `std::move`.
- `push_back`/`insert` of an rvalue, `emplace_*`.
- Returning a *local variable* by value (it's treated as an rvalue for overload resolution — see NRVO below).

A subtlety: **don't `std::move` a local you're returning** — `return obj;` already enables RVO/move; `return std::move(obj);` *disables* RVO and forces a move (pessimization).

### 6.6 RVO / NRVO / copy elision

**Copy elision**: the compiler constructs the returned object *directly* in the caller's storage, skipping the copy/move entirely — zero cost.

- **RVO (Return Value Optimization)**: returning an unnamed temporary. **Mandatory** since C++17:
  ```cpp
  std::string make() { return std::string("hi"); }  // guaranteed: no copy, no move
  ```
- **NRVO (Named RVO)**: returning a named local. **Allowed but not mandatory** (compilers do it in practice):
  ```cpp
  std::string make() { std::string s = "hi"; return s; }  // usually elided
  ```

Because of guaranteed elision, **returning by value is cheap and idiomatic** — you do *not* need to return by pointer/reference to avoid a copy.

> 🎤 **If they ask: "What does `std::move` actually do?"**
> "Nothing at runtime by itself — it's a `static_cast` to an rvalue reference. It just changes the value category of the expression so overload resolution picks the move constructor or move assignment instead of the copy version. The actual resource stealing happens inside that move constructor, which transfers the pointers and leaves the source in a valid-but-empty state. So `std::move` is really 'permission to pillage this object.'"

> 🎤 **If they ask: "lvalue vs rvalue, and why do we care?"**
> "An lvalue has identity — a name and an address — and will be used again. An rvalue is a temporary that's about to expire, so no one will observe it afterward. That's why we care: for an rvalue we can steal its internal buffer instead of deep-copying it. rvalue references `&&` let us write overloads that bind only to rvalues — the move constructor — turning an O(n) copy into an O(1) pointer swap."

> 🎤 **If they ask: "What's RVO / copy elision?"**
> "The compiler builds the return value directly in the caller's storage instead of constructing a local and copying it out. Returning an unnamed temporary is *guaranteed* elision since C++17 — no copy and no move at all — and named-return-value optimization handles named locals in practice. The upshot is that returning big objects by value is cheap and idiomatic, and I should *not* write `return std::move(local)` because that actually disables NRVO."

---

## 7. Smart pointers

Smart pointers apply RAII to heap memory: they own a pointer and `delete` it in their destructor.

### 7.1 `std::unique_ptr<T>` — exclusive ownership, zero overhead

Owns the object alone; non-copyable, only movable (moving transfers ownership). With the default deleter it's the **same size as a raw pointer** and has **no runtime overhead** — the gold standard for owning heap memory.

```cpp
auto p = std::make_unique<Order>(args);   // preferred: single allocation, exception-safe
std::unique_ptr<Order> q = std::move(p);  // ownership moves to q; p is now null
// p.get() -> raw ptr; p.release() -> give up ownership; p.reset() -> delete now
```

Use it for: any single-owner heap object, pImpl, factory return types, storing polymorphic objects in containers (`vector<unique_ptr<Base>>`).

### 7.2 `std::shared_ptr<T>` — shared ownership, reference counted

Multiple owners; the object is destroyed when the **last** `shared_ptr` is gone. It holds **two pointers**: one to the managed object, one to a heap **control block** containing:
- a **strong count** (number of `shared_ptr`s) — when it hits 0, the object is destroyed;
- a **weak count** (number of `weak_ptr`s) — when *both* hit 0, the control block itself is freed;
- the deleter and allocator.

```cpp
auto sp = std::make_shared<Order>(args);  // ONE allocation for object + control block
auto sp2 = sp;                            // atomic ++strong_count (now 2)
// when sp and sp2 both die -> count 0 -> Order destroyed
```

**Costs (interview gold):**
- `sizeof(shared_ptr)` is **2 pointers** (16 bytes), vs 8 for `unique_ptr`/raw.
- Copying does an **atomic increment** on the strong count; destruction does an **atomic decrement**. Atomics are locked read-modify-write operations — cheap uncontended but a serialization point under contention (cache-line bouncing across cores), often tens of ns. This is why HFT code **avoids passing `shared_ptr` by value on the hot path** and prefers `unique_ptr` or raw/reference for non-owning access.
- `make_shared` does **one** allocation (object + control block together → better locality); `shared_ptr<T>(new T)` does **two**. (Caveat: with `make_shared`, the object's memory isn't freed until the last `weak_ptr` dies, since they share one block.)

### 7.3 `std::weak_ptr<T>` — non-owning observer; breaks cycles

A `weak_ptr` references an object managed by `shared_ptr`s **without** affecting the strong count. To use it you `lock()` it, which atomically returns a `shared_ptr` if the object is still alive, or null if it's gone.

```cpp
std::weak_ptr<Order> w = sp;
if (auto locked = w.lock()) {   // try to promote to shared_ptr
    locked->process();          // safe: object guaranteed alive in this scope
} else {
    // object already destroyed
}
```

### 7.4 Reference cycles

Two objects holding `shared_ptr`s to each other will **never** reach strong count 0 → memory leak. Break the cycle by making one direction a `weak_ptr`:

```cpp
struct Node {
    std::shared_ptr<Node> next;   // owns forward
    std::weak_ptr<Node>   prev;   // observes backward — no cycle, no leak
};
```

**Choosing:** default to `unique_ptr`. Use `shared_ptr` only when ownership is genuinely shared and lifetime is hard to reason about statically. Use `weak_ptr` for caches/observers/back-pointers. Use a raw pointer or reference for **non-owning** access when you know the owner outlives the use.

> 🎤 **If they ask: "unique_ptr vs shared_ptr — internals and cost?"**
> "`unique_ptr` is exclusive ownership: same size as a raw pointer, no overhead, move-only. `shared_ptr` is shared ownership and is two pointers — one to the object, one to a heap control block with an atomic strong count and atomic weak count. Copying it is an atomic increment, which is a locked RMW that bounces cache lines under contention, so it's not free. I default to `unique_ptr` and only use `shared_ptr` when ownership is truly shared, and I avoid passing it by value on the hot path. `weak_ptr` is a non-owning observer you `lock()` to use, and it's how you break reference cycles."

---

## 8. Virtual functions, vtables, and dynamic dispatch

### 8.1 Mechanics

When a class has a `virtual` function, the call target is resolved at **runtime** based on the object's dynamic type (runtime polymorphism). The standard implementation:

- Each polymorphic **class** has one **vtable** (virtual table) — a static array of function pointers, one per virtual function, pointing to that class's overrides.
- Each polymorphic **object** carries a hidden **vptr** (vtable pointer) — typically the first 8 bytes of the object — set by the constructor to point at its class's vtable.

A virtual call `base->foo()` compiles to: load the vptr from the object, index into the vtable to get the function pointer, call through it. Roughly:

```
mov  rax, [obj]        ; load vptr (first word of object)
call [rax + 8*index]   ; indirect call through vtable slot
```

```cpp
struct Base { virtual void f(); virtual ~Base(); };
struct Derived : Base { void f() override; };

Base* p = new Derived;
p->f();   // dynamic dispatch -> Derived::f, via Derived's vtable
```

### 8.2 Virtual destructors — why you need them

If you `delete` a derived object through a `Base*` and the destructor is **not** virtual, only `~Base` runs — `~Derived` is skipped → the derived part's resources leak, and it's **Undefined Behavior**.

```cpp
struct Base { /* NO virtual ~Base */ };
struct Derived : Base { std::vector<int> big; };
Base* p = new Derived;
delete p;   // UB: ~Derived never runs, `big` leaks
```

**Rule:** any class meant to be used as a polymorphic base (deleted through a base pointer) must have a **virtual destructor**. Making the destructor virtual makes the destructor call itself go through the vtable, so the most-derived destructor runs and chains up.

### 8.3 Pure virtual & abstract classes

```cpp
struct Strategy {
    virtual void on_tick(const Tick&) = 0;   // pure virtual: no body required here
    virtual ~Strategy() = default;
};
```

A class with at least one pure virtual function is **abstract** — it cannot be instantiated; it defines an interface. Derived classes must override all pure virtuals to become concrete.

### 8.4 Cost — and why HFT often avoids virtuals

- An **indirect call** through a function pointer. The CPU's branch/target predictor can hide much of the latency when the target is stable, but a misprediction costs ~10–20 cycles.
- It **defeats inlining**: the compiler usually can't see through a virtual call, so it can't inline the callee or optimize across the call boundary. For tiny hot functions, the lost inlining (and the optimizations it unlocks) often costs more than the indirect jump itself.
- The vptr adds 8 bytes to every object and breaks `memcpy`-style triviality.

For nanosecond-sensitive paths, HFT engineers replace runtime polymorphism with **compile-time polymorphism**: templates and CRTP (§10.5), `std::variant` + `std::visit`, or plain `if`/tag dispatch — all of which let the compiler resolve and inline the call.

> 🎤 **If they ask: "How do virtual functions work, and why might you avoid them in HFT?"**
> "Each polymorphic class has a vtable — an array of function pointers to its overrides — and each object stores a hidden vptr to its class's vtable, usually as the first 8 bytes. A virtual call loads the vptr, indexes the vtable, and does an indirect call, so the target is resolved at runtime by dynamic type. The catch for low latency is that it's an indirect call the compiler can't inline through, so for tiny hot functions you lose inlining and cross-call optimization, plus a possible branch misprediction. So on the hot path we prefer compile-time polymorphism — templates, CRTP, or `std::variant` — which the compiler can resolve and inline."

> 🎤 **If they ask: "Why do you need a virtual destructor?"**
> "If I delete a derived object through a base pointer and the base destructor isn't virtual, only the base destructor runs — the derived destructor is skipped, which leaks the derived part's resources and is technically Undefined Behavior. Making the destructor virtual routes the destructor call through the vtable so the most-derived destructor runs first and chains up. Rule: any class you delete polymorphically needs a virtual destructor."

---

## 9. Object memory layout: alignment, padding, cache

Every type has a **size** (`sizeof`) and an **alignment** (`alignof`) — the address must be a multiple of the alignment (e.g. an `int` is 4-byte aligned, a `double` 8-byte aligned). The compiler inserts **padding** between members and at the end of a struct so every member is properly aligned *and* arrays of the struct stay aligned. **Field ordering changes `sizeof`:**

```cpp
struct Bad {        // sizeof == 24
    char  a;        // offset 0
    // 7 bytes padding
    double b;       // offset 8 (needs 8-align)
    char  c;        // offset 16
    // 7 bytes tail padding (so array elements keep b 8-aligned)
};

struct Good {       // sizeof == 16  — same fields, reordered largest-first
    double b;       // offset 0
    char  a;        // offset 8
    char  c;        // offset 9
    // 6 bytes tail padding
};
```

**Rule of thumb:** order members **largest-alignment first** to minimize padding.

**Why this matters for performance (cache):** a cache line is 64 bytes. Smaller, well-packed structs fit more elements per line → fewer cache misses when iterating. Conversely, you sometimes want the *opposite* — to avoid **false sharing**, two atomics written by different threads should be on different cache lines, so you pad/align them apart with `alignas(64)`:

```cpp
struct alignas(64) PerCoreCounter { std::atomic<uint64_t> count; };  // own cache line
```

`#pragma pack(1)` (or `__attribute__((packed))`) removes padding — useful for wire/market-data formats that must match an exact byte layout — but unaligned access can be slower or, on some ISAs, faulting. `[[no_unique_address]]` (C++20) lets empty members take zero space.

> 🎤 **If they ask: "Why does field ordering change `sizeof`, and why care?"**
> "Each type has an alignment requirement, and the compiler inserts padding so every member lands on a valid boundary and arrays stay aligned. If I put a `char` before a `double`, I waste 7 bytes of padding; ordering members largest-first packs them tightly. It matters because a cache line is 64 bytes — a smaller struct means more elements per line and fewer cache misses on the hot path. The flip side is false sharing: if two threads write two atomics in the same line, I deliberately `alignas(64)` them onto separate lines."

---

## 10. Templates & zero-cost abstraction

### 10.1 Function and class templates

A template is a **compile-time code generator**. The compiler **instantiates** a concrete version for each set of template arguments actually used; nothing is generated for a template you never instantiate.

```cpp
template <class T>
T max_(T a, T b) { return a > b ? a : b; }   // function template
max_(3, 4);        // instantiates max_<int>
max_(1.5, 2.5);    // instantiates max_<double>

template <class T, std::size_t N>
class FixedArray { T data_[N]; /* ... */ };  // class template; N is a value parameter
```

### 10.2 Why templates = zero-cost abstraction

Because the type is known at compile time, the generated code is **fully specialized and inlinable** — no indirection, no runtime type checks. `std::sort` with a lambda comparator inlines the comparison and is typically *faster* than C's `qsort`, which calls a comparator through a function pointer it can't inline. You get generic, reusable code with the performance of hand-written type-specific code — the abstraction costs nothing at runtime. The tradeoffs are: longer compile times, **code bloat** (one copy per instantiation), and historically cryptic error messages (improved by C++20 **concepts**).

### 10.3 Template specialization

Provide a custom implementation for specific arguments:

```cpp
template <class T> struct Serializer        { /* generic */ };
template <>        struct Serializer<bool>  { /* full specialization for bool */ };
template <class T> struct Serializer<T*>    { /* partial specialization for pointers */ };
```

(`std::vector<bool>` is the famous — and famously awkward — full specialization that packs bits.)

### 10.4 SFINAE / concepts (brief)

"Substitution Failure Is Not An Error": if substituting template args into a signature fails, that overload is silently dropped rather than being a hard error — the basis of older trait-based dispatch (`std::enable_if`). C++20 **concepts** replace most of this with readable constraints:

```cpp
template <std::integral T>           // concept constraint
T twice(T x) { return x + x; }
```

### 10.5 CRTP — static polymorphism instead of virtuals

**Curiously Recurring Template Pattern:** a base class is templated on its *derived* class, so the base can call into the derived type **without a virtual call** — dispatch is resolved at compile time and **inlines**.

```cpp
template <class Derived>
struct StrategyBase {
    void run() {                                   // common interface
        static_cast<Derived*>(this)->on_tick();    // static dispatch — no vtable
    }
};

struct MyStrategy : StrategyBase<MyStrategy> {
    void on_tick() { /* ... */ }                   // "override" resolved at compile time
};

MyStrategy s;
s.run();   // on_tick() inlined directly — zero dispatch overhead
```

This gives you the *interface* benefit of polymorphism with **none** of the runtime cost — exactly why HFT codebases lean on CRTP and templates over virtual functions. The cost: it's compile-time only (you can't store mixed `StrategyBase*` in one container the way you can with a virtual base), and more template machinery.

> 🎤 **If they ask: "Why are templates 'zero-cost', and what's CRTP?"**
> "Templates are instantiated per type at compile time, so the compiler generates fully specialized code with no runtime indirection — generic code that's as fast as hand-written type-specific code, which is why `std::sort` beats `qsort`. The cost is compile time and code bloat, not runtime. CRTP is the static-polymorphism trick: the base class is templated on the derived class and `static_cast`s `this` to call the derived method, so dispatch happens at compile time and inlines — you get a polymorphic interface with zero vtable overhead, which is why HFT prefers it over virtual functions on the hot path."

---

## 11. Compilation model, ODR, linking, inlining

### 11.1 The four stages

```
source.cpp ──preprocess──► translation unit ──compile──► assembly ──assemble──► object file (.o) ──link──► executable
```

1. **Preprocess**: handle `#include` (textually paste headers), `#define` macros, `#ifdef`. Output is one big **translation unit (TU)**.
2. **Compile**: TU → assembly. Each TU is compiled **independently** — this is why the compiler needs declarations (from headers) for things defined in other TUs.
3. **Assemble**: assembly → machine-code **object file**, with a symbol table and unresolved external references.
4. **Link**: combine all `.o` files + libraries, **resolve symbols** (match each "I need `foo`" to its definition), produce the final binary.

### 11.2 Header vs source, and why headers exist

A header carries **declarations** so other TUs know an entity's type/signature; the definition lives in one `.cpp` (compiled once) or, for templates and `inline` functions, in the header. Use **include guards** (`#pragma once` or `#ifndef` guards) to avoid double-inclusion within a TU.

### 11.3 ODR — the One Definition Rule

- Every entity may be **declared** many times but **defined exactly once** across the whole program. Violating this for functions/variables = ODR violation (often a linker "multiple definition" error, sometimes silent UB).
- Exception: **`inline` functions, templates, and class definitions** may be defined in multiple TUs *as long as every definition is token-for-token identical* — that's precisely why you can (and must) put them in headers. The linker then merges the duplicate definitions into one.

This is the real meaning of `inline`: not "please inline this call" (that's a hint the optimizer ignores or applies on its own), but "**this may be defined in multiple TUs**" — it relaxes the ODR.

### 11.4 Inlining (the optimization)

Replacing a call with the callee's body, eliminating call overhead and — more importantly — exposing the callee to the caller's optimizer (constant propagation, dead-code elimination across the boundary). The compiler decides based on heuristics; `inline` is only a weak hint. This is why small functions and templates live in headers (the compiler must *see* the body to inline it) and why **virtual calls hurt** (the body isn't statically known). LTO (Link-Time Optimization) extends inlining across TUs.

### 11.5 Static vs dynamic linking

- **Static** (`.a` / `.lib`): library code is copied **into** the executable at link time. Bigger binary, no runtime dependency, can be faster (no PLT indirection, more inlining/LTO opportunity, better locality). Common in HFT for deterministic, dependency-free deploys.
- **Dynamic** (`.so` / `.dll`): library is loaded at **run time** and shared across processes. Smaller binaries, updatable without relinking, but calls go through an indirection table (PLT/GOT) and you depend on the right library being present.

> 🎤 **If they ask: "Walk me through compilation, and what's the ODR?"**
> "Four stages: preprocess expands includes and macros into a translation unit; compile turns each TU independently into assembly; assemble produces an object file with a symbol table; link resolves symbols across object files and libraries into the executable. Because TUs compile independently, headers exist to share declarations. The One Definition Rule says every function or variable is defined exactly once program-wide — except inline functions, templates, and classes, which may be defined identically in many TUs, which is exactly why those go in headers. That's what `inline` really means: it relaxes the ODR, not 'force inline.'"

---

## 12. Undefined Behavior

**UB** = the standard imposes *no requirements* on what happens. The compiler is allowed to assume UB **never occurs** and optimizes accordingly — so UB can do anything: crash, corrupt data, work today and break after a recompile, or cause the optimizer to **delete code** (e.g. a null check it "proved" redundant because you already dereferenced the pointer). UB is not just "might crash"; it poisons reasoning about the whole program. This is why it matters intensely in trading systems: a latent UB can pass tests and detonate in production.

Common UB to recognize:
- **Out-of-bounds** array/pointer access (`v[v.size()]`, reading past a buffer).
- **Signed integer overflow** (`INT_MAX + 1`). Unsigned overflow is *defined* (wraps); signed is UB — the compiler may assume `x + 1 > x` always.
- **Use-after-free / dangling pointer or reference** dereference (§2).
- **Null pointer dereference**; misaligned access.
- **Data race**: two threads access the same memory, at least one writes, with no synchronization — UB. (Atomics or mutexes make it well-defined.)
- **Uninitialized reads**, reading an inactive union member, **invalid downcast** (`static_cast` to the wrong type), violating strict aliasing (reinterpreting unrelated types), calling a pure virtual during construction, integer divide by zero, shifting by ≥ width.

Tools: compile with `-Wall -Wextra`, run with **sanitizers** (`-fsanitize=address,undefined,thread`), use Valgrind.

> 🎤 **If they ask: "What is Undefined Behavior and why does it matter?"**
> "UB is any operation the standard places no constraints on — out-of-bounds access, signed overflow, use-after-free, data races, null deref. The key insight is that the compiler is *allowed to assume UB never happens*, so it optimizes on that assumption — it can delete a null check it thinks is redundant, or reorder code — meaning a single UB can silently corrupt unrelated logic and may work until you change compilers or flags. In a trading system that's catastrophic, so I catch it with `-Wall -Wextra`, AddressSanitizer, UBSan, and ThreadSanitizer."

---

## 13. Exceptions, `noexcept`, and exception-safety guarantees

When code `throw`s, the runtime **unwinds the stack**, running destructors for every fully-constructed local along the way (this is what makes RAII work), until a matching `catch` is found; if none, `std::terminate`.

**Cost model:** the common "zero-cost exception" implementation adds *no* runtime cost on the **non-throwing** path (no checks in the happy case) — but the **throw** itself is expensive (table lookup, unwinding). So exceptions are fine for truly exceptional, rare conditions, and HFT often disables/avoids them on the hot path (using error codes, `std::expected`, or status returns) where even rare throws and the resulting non-determinism are unacceptable.

**`noexcept`** declares a function won't throw. If it does anyway, `std::terminate` is called immediately. Beyond documentation, it's **load-bearing for performance**: `std::vector` will only **move** its elements on reallocation if their move constructor is `noexcept` — otherwise it **copies** them (to preserve the strong guarantee). So marking move constructors `noexcept` is essential for fast containers.

```cpp
void f() noexcept;                 // promises not to throw
Buffer(Buffer&&) noexcept;         // lets vector move instead of copy on growth
```

**Exception-safety guarantees** (what state you're left in if an operation throws):
- **Nothrow / no-fail**: never throws (e.g. destructors, swaps, `noexcept` ops). Strongest.
- **Strong**: commit-or-rollback — if it throws, state is **unchanged** as if the call never happened (e.g. `vector::push_back` with reallocation: it copies/moves into new storage *before* freeing old). Implemented via the "copy-and-swap" idiom.
- **Basic**: if it throws, no leaks and all invariants hold, but the object may be in a **valid-but-modified** state.
- **None**: may leak / corrupt. Avoid.

> 🎤 **If they ask: "What's `noexcept` for, and the exception-safety guarantees?"**
> "`noexcept` declares a function won't throw; if it does, the program terminates. It's not just documentation — `std::vector` only moves its elements on reallocation if their move constructor is `noexcept`, otherwise it copies for safety, so I always mark moves `noexcept`. The safety guarantees describe the state after a throw: nothrow never throws; strong is commit-or-rollback so state is unchanged on failure, usually via copy-and-swap; basic guarantees no leaks and valid invariants but possibly modified state; and 'no guarantee' which I avoid. RAII is what makes all of these achievable, since unwinding runs destructors automatically."

---

## 14. Lambdas & closures

A lambda is syntactic sugar for an **anonymous function object** (a compiler-generated struct with an `operator()`). The **capture list** becomes the struct's data members.

```cpp
int threshold = 100;
auto pred = [threshold](int x) { return x > threshold; };  // captures a COPY of threshold
// roughly equivalent to:
// struct __lambda { int threshold; bool operator()(int x) const { return x > threshold; } };
```

**Captures:**
- `[x]` — **by value** (a copy stored in the closure; safe to outlive `x`).
- `[&x]` — **by reference** (stores a reference; **dangles** if the lambda outlives `x` — a classic bug when a `[&]` lambda is stored or run async).
- `[=]` / `[&]` — capture everything used, by value / by reference (use sparingly).
- `[this]` — captures the `this` pointer (members accessed by reference — beware lifetime); `[*this]` (C++17) copies the whole object.
- `[x = std::move(y)]` — **init capture**, e.g. to move a `unique_ptr` into the closure.

```cpp
auto p = std::make_unique<int>(5);
auto f = [p = std::move(p)]() { return *p; };   // move-capture; lambda now owns it
```

By default `operator()` is `const` (can't modify captured-by-value members); **`mutable`** removes that so the lambda can modify its own copies:

```cpp
auto counter = [n = 0]() mutable { return n++; };   // each call mutates its own copy of n
```

**Cost / `std::function`:** a lambda with no captures converts to a plain function pointer and is zero-overhead; a captureless or stateful lambda passed as a **template** parameter inlines fully (zero cost — this is how STL algorithms stay fast). But storing it in a **`std::function`** is *not* free: `std::function` is a type-erased wrapper that may **heap-allocate** the lambda (if it exceeds the small-buffer size) and calls it through an **indirect call** the compiler can't inline — similar cost to a virtual call. So on the hot path, prefer passing lambdas as template parameters (`template <class F>`) over `std::function`.

> 🎤 **If they ask: "What's a lambda capturing by value vs reference, and is `std::function` free?"**
> "A lambda is an anonymous function object; the capture list becomes its data members. By value stores a copy that's safe to outlive the original; by reference stores a reference that dangles if the lambda outlives the captured variable — a common async bug. `mutable` lets it modify its by-value copies. As for cost: passed as a template parameter, a lambda inlines fully and is zero-overhead, which is why STL algorithms are fast. But `std::function` type-erases it — it may heap-allocate and calls through an indirect jump the compiler can't inline, roughly like a virtual call — so I avoid `std::function` on the hot path."

---

## 15. `new`/`delete` vs `malloc`/`free`, placement new, what ctors/dtors do

**`malloc`/`free`** are C library functions that allocate/free **raw bytes** — they know nothing about types. They do **not** run constructors or destructors.

**`new`/`delete`** are C++ operators that do **two things**:
1. `new T(args)`: (a) call `operator new` to obtain raw memory, then (b) **run T's constructor** in that memory. Returns a typed `T*`.
2. `delete p`: (a) **run T's destructor**, then (b) call `operator delete` to free the memory.

| | `new`/`delete` | `malloc`/`free` |
|---|---|---|
| Runs ctor/dtor | **Yes** | No |
| Returns | typed `T*` | `void*` (cast needed) |
| Size | computed from type | you pass bytes |
| On failure | throws `std::bad_alloc` | returns `NULL` |
| Arrays | `new[]` / `delete[]` | one call |
| Overridable | `operator new`/`delete` | no |

**Never mix them** (`free` on a `new`'d pointer, or `delete` on `malloc`'d memory) — UB. And `new`/`delete[]` must match `new[]`/`delete[]`.

**What a constructor actually does:** allocate is *already done*; the constructor (1) constructs base classes, then (2) constructs members in **declaration order** (via the member initializer list — or default-inits otherwise), then (3) runs the body. The destructor runs the body, then destroys members in **reverse** declaration order, then destroys bases — establishing the lifetime an object has.

**Placement new** — construct an object *in memory you already own*, separating allocation from construction. The backbone of `std::vector` (it allocates a buffer once, then placement-constructs elements as you push), memory pools, and arenas — crucial in HFT to **avoid per-object heap allocation on the hot path**.

```cpp
alignas(Order) char buf[sizeof(Order)];        // raw, properly aligned storage (e.g. preallocated)
Order* o = new (buf) Order(args);              // placement new: construct IN buf, no allocation
// ... use o ...
o->~Order();                                   // MUST destroy manually — placement new has no matching delete
// (do NOT call delete o; buf is not heap memory)
```

> 🎤 **If they ask: "`new` vs `malloc`? What's placement new?"**
> "`malloc` just hands back raw bytes and runs no constructor; `new` does two steps — it calls `operator new` for memory *and* runs the constructor, returns a typed pointer, and throws `bad_alloc` on failure. `delete` runs the destructor then frees. You must never mix them. Placement new separates the two: it constructs an object in memory you already own — no allocation — which is how `vector` builds elements in its buffer and how HFT pools preallocate storage to avoid heap allocation on the hot path. With placement new you call the destructor explicitly since there's no matching `delete`."

---

## Rapid-fire Q&A

**1. Difference between a reference and a pointer?**
A pointer is a variable holding an address — nullable, reassignable, supports arithmetic. A reference is an alias bound once at initialization, never null in well-defined code, can't be rebound. Compiler usually implements references as pointers.

**2. Stack vs heap?**
Stack: automatic, LIFO, allocation is just moving the stack pointer, freed on scope exit, cache-friendly, limited size. Heap: dynamic, allocated via `new`/`malloc`, you must free it, slower, can fragment, needed for objects that outlive their scope.

**3. What does `std::move` do?**
Nothing at runtime — it's a `static_cast` to an rvalue reference that lets overload resolution pick the move constructor. The actual stealing happens in that move constructor.

**4. lvalue vs rvalue?**
lvalue has identity (a name/address) and persists; rvalue is a temporary about to expire. We can steal an rvalue's resources instead of copying.

**5. What's RAII?**
Tie a resource's lifetime to a stack object — acquire in the constructor, release in the destructor. Destructors run on every scope exit including exception unwinding, so cleanup is guaranteed and leak-free.

**6. What's the Rule of 5 / Rule of 0?**
Rule of 5: if you define one of {destructor, copy ctor, copy assign, move ctor, move assign}, you probably need all five. Rule of 0: prefer to define none — compose from RAII members and let the compiler generate them.

**7. Why do you need a virtual destructor?**
Deleting a derived object through a base pointer with a non-virtual destructor runs only `~Base`, skipping `~Derived` — leak and UB. Virtual routes it through the vtable so the full chain runs.

**8. How does a virtual call work?**
Object stores a vptr to its class's vtable (array of function pointers); the call loads the vptr, indexes the slot, and does an indirect call resolved by dynamic type.

**9. Why might HFT avoid virtual functions?**
Indirect call the compiler can't inline through → lost inlining and cross-call optimization, plus possible branch misprediction. Use CRTP/templates/`std::variant` for compile-time dispatch instead.

**10. unique_ptr vs shared_ptr?**
`unique_ptr`: exclusive ownership, raw-pointer size, zero overhead, move-only. `shared_ptr`: shared ownership, two pointers, heap control block with atomic ref counts — copying is an atomic increment, not free.

**11. What's in a shared_ptr control block?**
Strong count, weak count, the deleter, and allocator. Strong hits 0 → object destroyed; strong and weak both 0 → control block freed.

**12. How do you break a shared_ptr cycle?**
Make one direction a `weak_ptr` — it doesn't contribute to the strong count, so the count can reach 0.

**13. `make_shared` vs `shared_ptr(new T)`?**
`make_shared` does one allocation (object + control block, better locality) and is exception-safe; the raw form does two. Caveat: `make_shared` keeps the object's memory until the last weak_ptr dies.

**14. What is `constexpr` vs `const`?**
`const` = read-only, value may be runtime-computed. `constexpr` = evaluable at compile time; a `constexpr` variable is a true compile-time constant usable for array sizes and template args.

**15. What does `noexcept` buy you?**
Declares no-throw (terminate if violated) and is load-bearing: `vector` only moves elements on reallocation if the move ctor is `noexcept`, else it copies. Always mark moves `noexcept`.

**16. What's the One Definition Rule?**
Each entity is defined exactly once program-wide — except inline functions, templates, and classes, which may be defined identically across TUs. That's why those go in headers, and what `inline` actually means (relaxes ODR, not "force inline").

**17. What is Undefined Behavior — give examples?**
Operations the standard places no constraints on; the compiler assumes they never happen and optimizes accordingly. Examples: out-of-bounds, signed overflow, use-after-free, null deref, data races, uninitialized reads.

**18. What's std::forward / perfect forwarding?**
In a template, `T&&` is a forwarding reference binding to lvalues and rvalues via reference collapsing. `std::forward<T>(arg)` re-casts the named argument back to its original value category, so it's moved if passed an rvalue and copied if passed an lvalue.

**19. What is copy elision / RVO?**
Compiler constructs the return value directly in the caller's storage, skipping copy/move. Mandatory since C++17 for unnamed temporaries; NRVO for named locals is common. So return by value is cheap — and don't write `return std::move(local)`.

**20. `new` vs `malloc`?**
`new` calls `operator new` *and* runs the constructor, returns a typed pointer, throws `bad_alloc`; `delete` runs the destructor then frees. `malloc` just returns raw bytes, no ctor/dtor, returns NULL on failure. Never mix them.

**21. What's placement new?**
Constructs an object in memory you already own — no allocation. Backbone of `vector` and memory pools; you must call the destructor explicitly.

**22. Capture by value vs reference in a lambda?**
By value stores a copy (safe to outlive the original); by reference stores a reference (dangles if the lambda outlives it). `mutable` lets the lambda modify its by-value copies.

**23. Is `std::function` free?**
No — it type-erases the callable, may heap-allocate, and calls through an indirect jump the compiler can't inline (≈ virtual-call cost). Prefer template parameters on the hot path.

**24. Why does struct field ordering matter?**
Alignment-driven padding: ordering members largest-first minimizes wasted bytes, shrinking `sizeof` so more elements fit per 64-byte cache line → fewer misses.

---

## Self-test checklist

Can you, from memory, explain each of these in ~30 seconds with correct mechanics?

- [ ] Draw the process memory layout (text/data/bss/heap/stack) and place a given variable in the right segment.
- [ ] State three concrete differences between a pointer and a reference.
- [ ] Decode `const int*` vs `int* const` vs `const int* const`, and explain `const` member functions and `mutable`.
- [ ] Distinguish `const` from `constexpr` and give a use for each.
- [ ] Define RAII and explain why it makes code exception-safe; name four RAII types in the STL.
- [ ] List the five special member functions and state the Rule of 5 and Rule of 0.
- [ ] Write a correct move constructor and move assignment for a class owning a raw pointer.
- [ ] Explain what `std::move` actually compiles to and why moving a vector is O(1).
- [ ] Explain lvalue vs rvalue and what an rvalue reference `&&` binds to.
- [ ] Explain forwarding references, reference collapsing, and `std::forward`.
- [ ] Explain RVO/NRVO/guaranteed copy elision and why `return std::move(local)` is a pessimization.
- [ ] Describe `unique_ptr` vs `shared_ptr` internals, the control block, and the atomic refcount cost.
- [ ] Explain `weak_ptr`, `lock()`, and how to break a reference cycle.
- [ ] Explain vtable/vptr mechanics and the exact steps of a virtual call.
- [ ] Explain why a polymorphic base needs a virtual destructor (and what goes wrong without one).
- [ ] Explain why HFT prefers CRTP/templates over virtual functions, and write a CRTP example.
- [ ] Explain alignment, padding, and how field ordering changes `sizeof`; mention false sharing and `alignas`.
- [ ] Explain template instantiation and why templates are zero-cost; contrast `std::sort` with `qsort`.
- [ ] Explain the four compilation stages and the One Definition Rule (and what `inline` really means).
- [ ] Define Undefined Behavior, list five examples, and explain why the optimizer makes it dangerous.
- [ ] State the three exception-safety guarantees and why `noexcept` move ctors matter for `vector`.
- [ ] Explain lambda capture modes, `mutable`, init-capture, and the cost of `std::function`.
- [ ] Explain `new`/`delete` vs `malloc`/`free`, what a constructor/destructor actually does, and placement new.
