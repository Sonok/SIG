# 09 — Code Optimization / Refactoring + OOP Design (SIG Signature Round)

## Why this matters for SIG

This file targets the **single most under-prepared part** of Susquehanna's (SIG) ~90-minute technical round for Trading System Engineering / Software Developer internships. Per verified candidate reports, that round has a **signature three-part shape** that is *not* a generic LeetCode grind:

1. **Code optimization / refactoring** — the interviewer hands you working-but-ugly code and says *"make it faster / make it cleaner / now make it better."* You iterate live, improving both **Big-O complexity** and **design quality (SOLID / readability / reuse)**. This is the part most candidates fumble because they only practiced solving from a blank page, not *improving existing code*.
2. **DSA with follow-ups** — a normal algorithm problem, then "what if the input is huge / streamed / sorted?" (Covered in other files — see `0X_dsa_followups.md`. Only referenced here.)
3. **Object-oriented design** — implement a class with specified methods from scratch, and in the final round a **full OOD system** (the famous reported one: **design a grocery-store checkout** with items, produce by weight, coupons, tax, receipts).

You (Sonok) are strong at algorithms (CF ~1100) but have **not** drilled refactoring-for-design or OOD. This guide makes you fluent in both. Primary language: **C++**, with Python notes where useful.

The meta-skill SIG is testing: *can you take real, messy production-style code and make it both faster and well-engineered, while narrating your reasoning like a teammate?* Talk out loud. They care as much about your **process and communication** as the final code.

---

# PART A — Code Optimization / Refactoring Round

## A.0 — The performance script (say this out loud, in this order)

When they hand you code, do **not** start typing. Run this script:

1. **Read & restate.** "Let me read this and tell you what I think it does... It takes X, loops over Y, returns Z." Confirm understanding *before* changing anything. This catches misreads and shows discipline.
2. **Correctness first.** "Before optimizing, is this even correct? Let me check edge cases: empty input, duplicates, overflow, off-by-one, null/empty." Fix bugs before speed — a fast wrong answer is worthless.
3. **Identify inefficiencies & state current Big-O.** "This nested loop is O(n²) time, O(1) extra space. The bottleneck is the repeated linear search inside the loop."
4. **Propose a better data structure / algorithm & state new Big-O.** "If I put the seen values in an `unordered_set`, lookup drops to O(1) amortized, so the whole thing is O(n) time, O(n) space. That's a classic time-space trade-off."
5. **Then improve readability / reuse / naming.** "Now that it's fast, let me clean it up: rename `x`/`tmp`, extract this block into a named helper, replace the magic number `0.07` with a `kTaxRate` constant."
6. **Discuss SOLID / extensibility.** "This function does three things — parsing, computing, formatting. I'd split responsibilities (SRP). And the discount logic is hard-coded; I'd make it an interface so new discounts plug in without editing this (OCP)."
7. **Ask for the next direction.** "Want me to push further — e.g., make it generic over the element type, or handle the streaming case?" Let them steer the iteration.

> **Golden rule:** correctness → complexity → clarity → design. Never reorder. And **always state the Big-O before and after** every change — that's the phrase they're listening for.

---

## A.1 — Catalog of common inefficiencies (BEFORE → AFTER)

### 1. O(n²) nested-loop lookup → hash map / set O(n)

The single most common fix. Any "for each element, search the rest" is a red flag.

```cpp
// BEFORE — O(n^2) time: linear search inside the loop
bool hasPair(const std::vector<int>& a, int target) {
    for (size_t i = 0; i < a.size(); ++i)
        for (size_t j = i + 1; j < a.size(); ++j)
            if (a[i] + a[j] == target) return true;
    return false;
}
```

```cpp
// AFTER — O(n) time, O(n) space: one pass, O(1) hash lookups
bool hasPair(const std::vector<int>& a, int target) {
    std::unordered_set<int> seen;
    for (int x : a) {
        if (seen.count(target - x)) return true;
        seen.insert(x);
    }
    return false;
}
```
**Win:** O(n²) → O(n). The mantra: *"I'm trading O(n) space to remove a factor of n from time."*

### 2. Repeated recomputation → precompute / memoize / cache

```cpp
// BEFORE — recomputes the running sum every query: O(n) per call
int rangeSum(const std::vector<int>& a, int l, int r) {  // called many times
    int s = 0;
    for (int i = l; i <= r; ++i) s += a[i];
    return s;
}
```

```cpp
// AFTER — precompute prefix sums once: O(n) build, O(1) per query
struct PrefixSum {
    std::vector<long long> pre;           // pre[i] = sum of a[0..i-1]
    explicit PrefixSum(const std::vector<int>& a) : pre(a.size() + 1, 0) {
        for (size_t i = 0; i < a.size(); ++i) pre[i + 1] = pre[i] + a[i];
    }
    long long query(int l, int r) const { return pre[r + 1] - pre[l]; }
};
```
**Win:** O(n) per query → O(1) per query after an O(n) precompute. For recursion with overlapping subproblems, the analog is **memoization** (cache results in a map/array) turning exponential into polynomial.

### 3. Wrong container

```cpp
// BEFORE — std::list for random access / push_back-heavy work, then linear find
std::list<int> data;                 // no random access, cache-unfriendly
bool contains = std::find(data.begin(), data.end(), key) != data.end(); // O(n)
```

```cpp
// AFTER — pick the container that matches the access pattern
std::vector<int> data;               // contiguous, cache-friendly, O(1) index
// For membership tests:
std::unordered_set<int> lookup;      // O(1) avg contains
bool contains = lookup.count(key) > 0;
// Need sorted order / range queries instead? std::set / std::map = O(log n).
```
**Container cheat-sheet (interview-ready):**
| Need | Use | Lookup |
|---|---|---|
| index access, iterate, append | `std::vector` | O(1) index |
| membership, no order | `std::unordered_set/map` | O(1) avg |
| membership + sorted / range | `std::set/map` | O(log n) |
| FIFO / LIFO | `std::queue` / `std::stack` | O(1) ends |
| frequent middle insert/erase by iterator | `std::list` | O(1) at iterator |

Say: *"`unordered_map` is O(1) average but O(n) worst-case on collisions and has no ordering; `map` is O(log n) guaranteed and keeps keys sorted. I default to `unordered_map` unless I need ordering or worst-case guarantees."*

### 4. Unnecessary copies → const ref, std::move, reserve()

```cpp
// BEFORE — copies the whole vector on every call; reallocates repeatedly
std::vector<int> process(std::vector<int> v) {   // by value = copy
    std::vector<int> out;
    for (int x : v) out.push_back(x * 2);         // many reallocations
    return out;
}
```

```cpp
// AFTER — pass by const ref, reserve once, move on return (RVO usually handles it)
std::vector<int> process(const std::vector<int>& v) {  // no copy in
    std::vector<int> out;
    out.reserve(v.size());                              // one allocation
    for (int x : v) out.push_back(x * 2);
    return out;                                         // NRVO / move, no copy out
}
```
**Win:** removes O(n) copies and amortizes away reallocation. Rules: **take read-only params by `const&`**, `reserve()` when you know the size, use `std::move` to transfer ownership instead of copying, and prefer `emplace_back` to construct in place.

### 5. Repeated string concatenation; recomputing size()

```cpp
// BEFORE — O(n^2) string building; size() recomputed every iteration
std::string join(const std::vector<std::string>& parts) {
    std::string s = "";
    for (int i = 0; i < (int)parts.size(); ++i)   // size() each loop check
        s = s + parts[i] + ",";                    // realloc + copy each time
    return s;
}
```

```cpp
// AFTER — append in place, cache the bound, reserve capacity
std::string join(const std::vector<std::string>& parts) {
    std::string s;
    size_t total = 0;
    for (const auto& p : parts) total += p.size() + 1;
    s.reserve(total);                              // one allocation
    for (const auto& p : parts) { s += p; s += ','; }  // += appends, no temp
    return s;
}
```
**Win:** O(n²) → O(n). Two ideas: `+=` appends without building a temporary (unlike `s = s + ...`), and hoist invariant work (like `size()` or any value that doesn't change) **out of the loop**.

### 6. Redundant passes → merge into one

```cpp
// BEFORE — three passes over the data
double mean = sum(v) / v.size();
double mx   = *std::max_element(v.begin(), v.end());
double mn   = *std::min_element(v.begin(), v.end());
```

```cpp
// AFTER — single pass computes all three
double s = 0, mx = v[0], mn = v[0];
for (double x : v) { s += x; mx = std::max(mx, x); mn = std::min(mn, x); }
double mean = s / v.size();
```
**Win:** 3n → n operations (same Big-O, but a real constant-factor and cache win — and a good thing to *name* out loud).

### 7. Poor early-exit / no short-circuiting

```cpp
// BEFORE — keeps scanning after the answer is known; expensive check runs first
bool allValid(const std::vector<Item>& v) {
    bool ok = true;
    for (auto& it : v)
        if (!expensiveCheck(it) || !it.cheapFlag) ok = false;  // no break; bad order
    return ok;
}
```

```cpp
// AFTER — return on first failure; test the cheap condition first (&& short-circuits)
bool allValid(const std::vector<Item>& v) {
    for (const auto& it : v)
        if (!it.cheapFlag || !expensiveCheck(it))  // cheap test first; skips expensive
            return false;                          // early exit
    return true;
}
```
**Win:** avoids wasted work; reorder `&&`/`||` so the cheap, likely-decisive test runs first.

---

## A.2 — Full worked "refactor it live" examples

### Worked example 1 — nested loops + bad naming

**Messy version they hand you:**
```cpp
// find how many students share a score with at least one other student
int f(std::vector<int> d) {
    int c = 0;
    for (int i = 0; i < d.size(); i++) {
        bool found = false;
        for (int j = 0; j < d.size(); j++) {
            if (i != j && d[i] == d[j]) { found = true; }
        }
        if (found == true) c = c + 1;
    }
    return c;
}
```

**Narration:** "It counts elements that appear more than once. Correctness looks OK. Problems: (1) O(n²) because of the inner scan; (2) it never breaks early; (3) names `f`, `d`, `c` are opaque; (4) `found == true` is redundant; (5) the vector is taken by value — an unnecessary copy. I'll fix complexity first with a frequency map, then clean up."

**Optimized + clean version:**
```cpp
// Count students whose score is shared by at least one other student. O(n) time, O(n) space.
int countSharedScores(const std::vector<int>& scores) {
    std::unordered_map<int, int> frequency;
    for (int s : scores) ++frequency[s];      // pass 1: tally

    int sharedCount = 0;
    for (int s : scores)                       // pass 2: count those with freq > 1
        if (frequency[s] > 1) ++sharedCount;
    return sharedCount;
}
```
"O(n²) → O(n). Names now self-document, param is `const&`, and the boolean noise is gone. If they want it generic, I'd template it on the element type. If memory is tight and the input is sorted, I could do it in O(1) extra space with a single linear scan comparing neighbors."

### Worked example 2 — magic-number discount calculator (bridges to OOD)

**Messy version:**
```cpp
double calc(double p, int type, int qty) {
    double t = p * qty;
    if (type == 1) t = t * 0.9;                 // 10% off
    else if (type == 2) t = t - 5;              // $5 off
    else if (type == 3) { if (qty >= 3) t = t * 0.8; }  // bulk 20% off if 3+
    t = t * 1.07;                                // tax
    return t;
}
```

**Narration:** "It computes a line total, applies one of three discount types via magic numbers, then adds 7% tax. Issues: magic numbers (`0.9`, `5`, `0.8`, `1.07`), an `int type` that's really an enum, and a giant `if/else` that I must *edit* every time a new discount appears — that violates the **Open/Closed Principle**. Step one: extract constants. Step two: replace the type-switch with polymorphic discount objects so new discounts plug in without touching this code."

**Step 1 — readability (constants + enum):**
```cpp
constexpr double kTaxRate = 0.07;
constexpr double kPercentOff = 0.10;
constexpr double kFlatOff = 5.0;
constexpr double kBulkOff = 0.20;
constexpr int    kBulkThreshold = 3;

enum class DiscountType { None, Percent, Flat, Bulk };

double lineTotal(double unitPrice, DiscountType type, int qty) {
    double subtotal = unitPrice * qty;
    switch (type) {
        case DiscountType::Percent: subtotal *= (1 - kPercentOff); break;
        case DiscountType::Flat:    subtotal -= kFlatOff;          break;
        case DiscountType::Bulk:
            if (qty >= kBulkThreshold) subtotal *= (1 - kBulkOff); break;
        case DiscountType::None: break;
    }
    return subtotal * (1 + kTaxRate);
}
```

**Step 2 — Open/Closed via a Discount interface (the design payoff):**
```cpp
struct LineItem { double unitPrice; int qty; };

// Strategy interface: each discount knows how to transform a subtotal.
class Discount {
public:
    virtual ~Discount() = default;
    virtual double apply(double subtotal, const LineItem& item) const = 0;
};

class PercentDiscount : public Discount {
    double rate_;
public:
    explicit PercentDiscount(double rate) : rate_(rate) {}
    double apply(double s, const LineItem&) const override { return s * (1 - rate_); }
};

class FlatDiscount : public Discount {
    double amount_;
public:
    explicit FlatDiscount(double amount) : amount_(amount) {}
    double apply(double s, const LineItem&) const override { return std::max(0.0, s - amount_); }
};

class BulkDiscount : public Discount {
    int threshold_; double rate_;
public:
    BulkDiscount(int t, double r) : threshold_(t), rate_(r) {}
    double apply(double s, const LineItem& it) const override {
        return it.qty >= threshold_ ? s * (1 - rate_) : s;
    }
};

// This function NEVER changes when a new discount type is added.
double lineTotal(const LineItem& item, const Discount* discount, double taxRate) {
    double subtotal = item.unitPrice * item.qty;
    if (discount) subtotal = discount->apply(subtotal, item);
    return subtotal * (1 + taxRate);
}
```
"Now adding, say, a `BuyOneGetOneDiscount` is a *new class*, not an edit to existing tested code. That's OCP, and it's exactly the muscle the checkout OOD problem rewards."

---

# PART B — SOLID & Clean Code

## B.1 — The five SOLID principles (violation → fix)

### S — Single Responsibility Principle
*A class should have one reason to change.*
```cpp
// VIOLATION: Report both computes AND formats AND saves — three reasons to change.
class Report {
public:
    double computeTotal();
    std::string toHtml();
    void saveToDisk(const std::string& path);
};
```
```cpp
// FIX: split responsibilities.
class Report           { public: double computeTotal() const; };
class ReportFormatter  { public: std::string toHtml(const Report&) const; };
class ReportStorage    { public: void save(const std::string& html, const std::string& path); };
```

### O — Open/Closed Principle
*Open for extension, closed for modification.*
```cpp
// VIOLATION: must edit area() for every new shape.
double area(const Shape& s) {
    if (s.type == CIRCLE) return 3.14159 * s.r * s.r;
    else if (s.type == SQUARE) return s.side * s.side;   // edit here every time
}
```
```cpp
// FIX: polymorphism — new shapes are new classes, no edits to existing code.
class Shape { public: virtual ~Shape() = default; virtual double area() const = 0; };
class Circle : public Shape { double r; public: double area() const override { return 3.14159 * r * r; } };
class Square : public Shape { double s; public: double area() const override { return s * s; } };
```

### L — Liskov Substitution Principle
*Subtypes must be usable anywhere their base type is, without surprises.*
```cpp
// VIOLATION: Square "is-a" Rectangle breaks expectations (setWidth changes height too).
class Rectangle { public: virtual void setWidth(int w); virtual void setHeight(int h); };
class Square : public Rectangle { /* setWidth also sets height — violates the contract */ };
```
```cpp
// FIX: don't force a false is-a. Model a common abstraction instead.
class Shape { public: virtual int area() const = 0; virtual ~Shape() = default; };
class Rectangle : public Shape { int w, h; public: int area() const override { return w * h; } };
class Square    : public Shape { int s;    public: int area() const override { return s * s; } };
```
Rule of thumb: a subclass must not strengthen preconditions or weaken postconditions, and must not throw new surprises.

### I — Interface Segregation Principle
*Don't force clients to depend on methods they don't use.*
```cpp
// VIOLATION: a fat interface; a SimplePrinter is forced to implement scan/fax.
class Machine { public: virtual void print()=0; virtual void scan()=0; virtual void fax()=0; };
```
```cpp
// FIX: small, focused interfaces; classes implement only what they need.
class Printer { public: virtual void print() = 0; virtual ~Printer() = default; };
class Scanner { public: virtual void scan()  = 0; virtual ~Scanner() = default; };
class AllInOne : public Printer, public Scanner { /* implements both */ };
class SimplePrinter : public Printer { void print() override {} };  // no dead methods
```

### D — Dependency Inversion Principle
*Depend on abstractions, not concretions. High-level code shouldn't depend on low-level details.*
```cpp
// VIOLATION: NotificationService hard-wired to a concrete EmailSender.
class EmailSender { public: void send(const std::string&); };
class NotificationService { EmailSender email; public: void notify(const std::string& m){ email.send(m); } };
```
```cpp
// FIX: depend on an interface; inject the concrete implementation.
class MessageSender { public: virtual void send(const std::string&) = 0; virtual ~MessageSender() = default; };
class EmailSender : public MessageSender { public: void send(const std::string&) override {} };
class SmsSender   : public MessageSender { public: void send(const std::string&) override {} };

class NotificationService {
    MessageSender& sender_;                                  // abstraction, injected
public:
    explicit NotificationService(MessageSender& s) : sender_(s) {}
    void notify(const std::string& m) { sender_.send(m); }   // works with any sender
};
```

## B.2 — Clean-code essentials

- **Naming:** intention-revealing. `unitPrice`, `taxRate`, `isExpired()` — not `p`, `t`, `flag`. A good name removes the need for a comment.
- **Small functions:** one job, ideally fits on a screen. If you need "and" to describe it, split it.
- **No magic numbers:** name them — `constexpr double kTaxRate = 0.07;`. The constant documents *why*.
- **DRY:** don't repeat yourself. Duplicated logic → extract a function. Three copies of a formula is a bug waiting to diverge.
- **Encapsulation:** keep data `private`, expose behavior through methods; maintain invariants inside the class (e.g., balance can't go negative).
- **Const-correctness:** mark read-only methods `const`, params `const&`. Communicates intent and lets the compiler help.
- **Composition over inheritance:** prefer "has-a" (a `Car` *has* an `Engine`) over deep "is-a" hierarchies — more flexible, less coupling, no fragile-base-class problems.
- **Program to interfaces:** depend on abstract types so implementations are swappable (this *is* DIP in practice).

## B.3 — 🎤 If they ask: crisp spoken SOLID definitions

- **S — Single Responsibility:** "A class should have one job, one reason to change."
- **O — Open/Closed:** "Open for extension, closed for modification — add behavior with new code, not by editing old code."
- **L — Liskov Substitution:** "You should be able to use any subclass wherever the base class is expected, without breaking behavior."
- **I — Interface Segregation:** "Many small, focused interfaces beat one fat one — clients shouldn't depend on methods they don't use."
- **D — Dependency Inversion:** "Depend on abstractions, not concrete classes — and inject the dependency from outside."

---

# PART C — Object-Oriented Design (OOD)

## C.1 — OOP fundamentals for design

- **Encapsulation:** bundle data + behavior; hide internals behind a public API; protect invariants. Data `private`, methods control access.
- **Abstraction:** expose *what* something does, hide *how*. Callers use `Discount::apply`, not its math.
- **Inheritance (is-a):** a `SavingsAccount` *is an* `Account`. Use for genuine subtype relationships + shared interface/polymorphism.
- **Composition (has-a):** a `Car` *has an* `Engine`; a `Receipt` *has* `LineItem`s. **Prefer composition** — it's looser coupling, easier to change, and avoids deep brittle hierarchies. Inherit for behavior you must substitute polymorphically; compose for everything else.
- **Polymorphism:** one interface, many implementations — call `discount->apply(...)` and the right override runs. The engine of Open/Closed.
- **Interface / abstract class:** a pure-virtual C++ class defines a contract (`= 0` methods). Program against it so implementations are swappable.

**Decision guide:** is-a + needs substitutability → inheritance. Otherwise → composition. When in doubt, compose.

## C.2 — A framework for any OOD problem

1. **Clarify requirements & scope.** Ask questions: "Are we handling payment? Multiple registers? Returns?" Pin down what's in/out. *Always do this first — it's graded.*
2. **Identify core entities (the nouns).** Item, Produce, Coupon, Receipt, Register... each becomes a class.
3. **Define responsibilities & relationships.** Who owns what? has-a vs is-a? Draw the diagram.
4. **Key methods / APIs.** What's the minimal public interface of each class?
5. **Handle the tricky rules.** Weight-priced produce, discount stacking, tax order, edge cases.
6. **Extensibility (Open/Closed).** "Where does a *new* requirement plug in?" — usually a new subclass of an interface. Call this out explicitly; it's what separates strong candidates.

---

## C.3 — Worked OOD #1: Grocery-store checkout (THE SIG problem — deepest treatment)

### Requirements to clarify aloud
"Items have a fixed unit price; **produce is priced by weight** (price per pound × weight). Customers add items to a cart, the register **applies discounts/coupons**, computes **tax**, and prints a **receipt** with line items, subtotal, discounts, tax, and total. I want new discount types to be addable without rewriting the register. I'll assume single currency, tax applies after discounts, and some items (e.g. groceries) may be tax-exempt."

### Text class diagram
```
Product (abstract)            <-- abstraction over anything sellable
  ├─ UnitProduct  (price per unit, isTaxable)
  └─ ProduceProduct (pricePerPound, isTaxable=false typically)

LineItem        has-a Product, holds quantity OR weight, computes lineSubtotal()
ShoppingCart    has-many LineItem
Discount (abstract interface)        <-- Strategy / OCP extension point
  ├─ PercentDiscount
  ├─ FlatCouponDiscount
  └─ BuyNGetMFreeDiscount   (added later WITHOUT touching Register)
Register        orchestrates: cart + discounts + taxRate -> Receipt
Receipt         value object: lines, subtotal, totalDiscount, tax, total
```

### C++ skeletons
```cpp
#include <memory>
#include <string>
#include <vector>
#include <algorithm>

// ---- Product hierarchy: is-a, because pricing is polymorphic ----
class Product {
protected:
    std::string name_;
    bool taxable_;
public:
    Product(std::string name, bool taxable) : name_(std::move(name)), taxable_(taxable) {}
    virtual ~Product() = default;
    const std::string& name() const { return name_; }
    bool taxable() const { return taxable_; }
    // price for a given amount: amount = quantity (unit) or weight in lbs (produce)
    virtual double priceFor(double amount) const = 0;
};

class UnitProduct : public Product {
    double unitPrice_;
public:
    UnitProduct(std::string n, double price, bool taxable = true)
        : Product(std::move(n), taxable), unitPrice_(price) {}
    double priceFor(double quantity) const override { return unitPrice_ * quantity; }
};

class ProduceProduct : public Product {   // priced by weight
    double pricePerPound_;
public:
    ProduceProduct(std::string n, double perPound, bool taxable = false)
        : Product(std::move(n), taxable), pricePerPound_(perPound) {}
    double priceFor(double weightLbs) const override { return pricePerPound_ * weightLbs; }
};

// ---- LineItem: has-a Product (composition) ----
struct LineItem {
    std::shared_ptr<Product> product;
    double amount;                       // quantity for unit items, weight for produce
    double subtotal() const { return product->priceFor(amount); }
    bool taxable() const { return product->taxable(); }
};

// ---- Discount: the Open/Closed extension point (Strategy pattern) ----
class Discount {
public:
    virtual ~Discount() = default;
    // returns the discount AMOUNT (>= 0) to subtract from the cart subtotal
    virtual double computeDiscount(const std::vector<LineItem>& items) const = 0;
    virtual std::string label() const = 0;   // for the receipt
};

class PercentDiscount : public Discount {
    double rate_; std::string label_;
public:
    PercentDiscount(double rate, std::string label) : rate_(rate), label_(std::move(label)) {}
    double computeDiscount(const std::vector<LineItem>& items) const override {
        double sub = 0; for (const auto& it : items) sub += it.subtotal();
        return sub * rate_;
    }
    std::string label() const override { return label_; }
};

class FlatCouponDiscount : public Discount {   // "$5 off, min spend $20"
    double amount_, minSpend_; std::string label_;
public:
    FlatCouponDiscount(double amt, double minSpend, std::string label)
        : amount_(amt), minSpend_(minSpend), label_(std::move(label)) {}
    double computeDiscount(const std::vector<LineItem>& items) const override {
        double sub = 0; for (const auto& it : items) sub += it.subtotal();
        return sub >= minSpend_ ? std::min(amount_, sub) : 0.0;
    }
    std::string label() const override { return label_; }
};

// ---- Receipt: value object ----
struct ReceiptLine { std::string name; double amount; double subtotal; };
struct Receipt {
    std::vector<ReceiptLine> lines;
    double subtotal = 0, totalDiscount = 0, tax = 0, total = 0;
};

// ---- Register: orchestrates everything; depends on the Discount abstraction (DIP) ----
class Register {
    double taxRate_;
public:
    explicit Register(double taxRate) : taxRate_(taxRate) {}

    Receipt checkout(const std::vector<LineItem>& items,
                     const std::vector<std::shared_ptr<Discount>>& discounts) const {
        Receipt r;
        double taxableBase = 0;
        for (const auto& it : items) {
            double s = it.subtotal();
            r.lines.push_back({it.product->name(), it.amount, s});
            r.subtotal += s;
            if (it.taxable()) taxableBase += s;
        }
        for (const auto& d : discounts)              // polymorphic dispatch — OCP
            r.totalDiscount += d->computeDiscount(items);

        // tax applies to the taxable base after discounts (proportional, simplified)
        double discountedTaxable = std::max(0.0, taxableBase - r.totalDiscount);
        r.tax   = discountedTaxable * taxRate_;
        r.total = std::max(0.0, r.subtotal - r.totalDiscount) + r.tax;
        return r;
    }
};
```

### Why this design (narrate this)
- **`Product` is an abstract base** so `UnitProduct` and `ProduceProduct` share one interface but price differently (**polymorphism**). The `Register` never asks "is this produce?" — it just calls `priceFor`. That's the **LSP/OCP** payoff.
- **`Discount` is a Strategy interface** — the *single most important* design choice here. Adding `BuyNGetMFreeDiscount` or a "loyalty member 5%" is a **new class**; `Register::checkout` is untouched. **This is Open/Closed, and it's the question they'll push on.**
- **`Register` depends on the `Discount` abstraction**, not concrete discounts (**DIP**), and discounts are *injected* into `checkout`.
- **`Receipt` is a plain value object** (SRP: it holds data, doesn't compute pricing).
- **Composition everywhere:** `LineItem` *has-a* `Product`; `Cart`/`Register` *have* `LineItem`s. Only the genuine subtype relationship (product pricing) uses inheritance.

### "Add a new discount type without changing existing code" — the answer
```cpp
// Brand-new file, zero edits to Register/Product/LineItem:
class BuyNGetMFreeDiscount : public Discount {
    std::string targetName_; int n_, m_;
public:
    BuyNGetMFreeDiscount(std::string name, int n, int m)
        : targetName_(std::move(name)), n_(n), m_(m) {}
    double computeDiscount(const std::vector<LineItem>& items) const override {
        for (const auto& it : items) {
            if (it.product->name() == targetName_) {
                int qty = static_cast<int>(it.amount);
                int groups = qty / (n_ + m_);
                double unit = it.subtotal() / qty;
                return groups * m_ * unit;        // m free per (n+m) bought
            }
        }
        return 0.0;
    }
    std::string label() const override { return "Buy " + std::to_string(n_) + " get " + std::to_string(m_) + " free"; }
};
```
"Just instantiate it and pass it into `checkout`. No existing class changed — that's exactly OCP." If asked about **discount stacking order**, note that since each discount returns an amount independently, ordering/exclusivity is a policy you can add via a `DiscountPolicy` object (another extension point) rather than `if`-soup.

---

## C.4 — Worked OOD #2: Parking Lot

### Clarify
"Multiple levels, each with spots of different sizes (motorcycle / compact / large). A vehicle takes the smallest spot that fits. Track availability, park/unpark, and (optionally) compute a fee by duration."

### Text diagram
```
Vehicle (abstract): size()  ──┐ is-a: Motorcycle, Car, Bus
ParkingSpot: SpotSize, occupied, canFit(Vehicle)
Level: vector<ParkingSpot>, findSpot(), park(), availableCount()
ParkingLot: vector<Level>, parkVehicle(), unpark(), ticketing
Ticket: vehicle, spot, entryTime  (fee = rate * duration)
```

### C++ skeleton
```cpp
enum class Size { Motorcycle, Compact, Large };

class Vehicle {
public:
    virtual ~Vehicle() = default;
    virtual Size size() const = 0;
};
class Motorcycle : public Vehicle { public: Size size() const override { return Size::Motorcycle; } };
class Car        : public Vehicle { public: Size size() const override { return Size::Compact; } };
class Bus        : public Vehicle { public: Size size() const override { return Size::Large; } };

class ParkingSpot {
    Size size_;
    Vehicle* current_ = nullptr;
public:
    explicit ParkingSpot(Size s) : size_(s) {}
    bool isFree() const { return current_ == nullptr; }
    bool canFit(const Vehicle& v) const { return isFree() && v.size() <= size_; }
    void park(Vehicle* v)  { current_ = v; }
    void leave()           { current_ = nullptr; }
};

class Level {
    std::vector<ParkingSpot> spots_;
public:
    ParkingSpot* findSpot(const Vehicle& v) {
        for (auto& s : spots_) if (s.canFit(v)) return &s;   // smallest-fit if spots are size-sorted
        return nullptr;
    }
    int availableCount() const {
        int c = 0; for (const auto& s : spots_) if (s.isFree()) ++c; return c;
    }
};

class ParkingLot {
    std::vector<Level> levels_;
public:
    ParkingSpot* park(Vehicle& v) {
        for (auto& lvl : levels_)
            if (auto* spot = lvl.findSpot(v)) { spot->park(&v); return spot; }
        return nullptr;   // lot full
    }
};
```
**SOLID notes:** `Vehicle` polymorphism + `size()` keeps `canFit` open to new vehicle types (OCP). Each class has one responsibility (SRP): `ParkingSpot` knows occupancy, `Level` knows allocation, `ParkingLot` orchestrates. Fee strategy could be its own injectable interface (DIP) — same pattern as discounts.

---

## C.5 — Worked OOD #3: Deck of Cards / Card Game

### Clarify
"Standard 52-card deck, 4 suits × 13 ranks. Need shuffle, deal, and reusability for *any* card game (Blackjack, Poker). Keep generic game rules out of the deck."

### Text diagram
```
enum Suit; enum Rank;
Card: suit, rank, value()
Deck: vector<Card>, shuffle(), deal(), size()      <-- reusable, game-agnostic
Hand: vector<Card>, add(), score()  (game-specific subclasses)
  └─ BlackjackHand : score() with ace logic
Game (abstract): play()             <-- specific games extend
```

### C++ skeleton
```cpp
#include <random>
#include <algorithm>

enum class Suit { Hearts, Diamonds, Clubs, Spades };
enum class Rank { Two=2, Three, Four, Five, Six, Seven, Eight, Nine, Ten, Jack, Queen, King, Ace };

struct Card {
    Suit suit; Rank rank;
    int faceValue() const { return std::min(static_cast<int>(rank), 10); } // J/Q/K = 10
};

class Deck {
    std::vector<Card> cards_;
public:
    Deck() {
        for (int s = 0; s < 4; ++s)
            for (int r = 2; r <= 14; ++r)
                cards_.push_back({static_cast<Suit>(s), static_cast<Rank>(r)});
    }
    void shuffle() {
        static std::mt19937 rng{std::random_device{}()};
        std::shuffle(cards_.begin(), cards_.end(), rng);
    }
    Card deal() { Card c = cards_.back(); cards_.pop_back(); return c; } // O(1)
    bool empty() const { return cards_.empty(); }
    size_t size() const { return cards_.size(); }
};

// Game-specific scoring lives in Hand subclasses, NOT in Deck (SRP + reuse).
class Hand {
protected:
    std::vector<Card> cards_;
public:
    virtual ~Hand() = default;
    void add(const Card& c) { cards_.push_back(c); }
    virtual int score() const = 0;
};

class BlackjackHand : public Hand {
public:
    int score() const override {
        int total = 0, aces = 0;
        for (const auto& c : cards_) {
            total += c.faceValue();
            if (c.rank == Rank::Ace) ++aces;
        }
        while (total <= 11 && aces > 0) { total += 10; --aces; }  // Ace as 11 when safe
        return total;
    }
};
```
**SOLID notes:** `Deck` is fully **reusable** and knows nothing about any game (SRP). Game rules live in `Hand` subclasses (OCP: a new game = a new `Hand`, no deck edits). `deal()` is O(1) by popping the back.

---

## C.6 — Worked OOD #4: Vending Machine (state-machine example)

### Clarify
"Holds products with prices and stock. Accepts coins, dispenses when enough money is inserted, returns change, refunds on cancel. Behavior depends on **state** (idle → collecting money → dispensing) — a natural **State pattern**."

### Text diagram
```
Product: name, price, stock
Inventory: map<slot, Product>
VendingMachine (context): currentState, balance, inventory
State (abstract): insertMoney(), selectProduct(), dispense(), cancel()
  ├─ IdleState
  ├─ HasMoneyState
  └─ DispensingState
```

### C++ skeleton
```cpp
#include <unordered_map>

class VendingMachine;   // forward decl (context)

class State {
public:
    virtual ~State() = default;
    virtual void insertMoney(VendingMachine&, int cents) = 0;
    virtual void selectProduct(VendingMachine&, const std::string& code) = 0;
    virtual void cancel(VendingMachine&) = 0;
};

struct Product { std::string name; int priceCents; int stock; };

class VendingMachine {
    std::unique_ptr<State> state_;
    int balance_ = 0;
    std::unordered_map<std::string, Product> inventory_;
public:
    explicit VendingMachine(std::unique_ptr<State> initial) : state_(std::move(initial)) {}
    void setState(std::unique_ptr<State> s) { state_ = std::move(s); }
    int  balance() const { return balance_; }
    void addBalance(int c) { balance_ += c; }
    void resetBalance()    { balance_ = 0; }
    Product* product(const std::string& code) {
        auto it = inventory_.find(code);
        return it == inventory_.end() ? nullptr : &it->second;
    }
    // delegate to current state — behavior changes with state, no big switch
    void insertMoney(int c)               { state_->insertMoney(*this, c); }
    void selectProduct(const std::string& code) { state_->selectProduct(*this, code); }
    void cancel()                         { state_->cancel(*this); }
};

class HasMoneyState; // fwd

class IdleState : public State {
public:
    void insertMoney(VendingMachine& m, int c) override;       // -> HasMoneyState
    void selectProduct(VendingMachine&, const std::string&) override {/* require money first */}
    void cancel(VendingMachine&) override {}
};
// HasMoneyState::selectProduct checks balance >= price && stock, dispenses,
// returns change, decrements stock, then transitions back to IdleState.
```
**SOLID notes:** each `State` encapsulates the behavior for one mode (SRP), transitions are explicit, and **no monolithic `if/switch` on a status enum** scattered across methods — adding a "maintenance" state is a new `State` class (OCP). *(Elevator is the alternative state-machine prompt — same pattern: states like Idle/MovingUp/MovingDown, a `Request` queue, and a scheduling strategy you can swap out via DIP.)*

---

# Rapid-fire Q&A

1. **What is the Single Responsibility Principle?** A class should have exactly one reason to change — one job. Split parsing, computing, and formatting into separate classes.
2. **What is the Open/Closed Principle, and how do you apply it?** Open for extension, closed for modification. Use polymorphism/strategy interfaces so new behavior is a new subclass, not an edit to tested code (e.g., the `Discount` interface in checkout).
3. **What is Liskov Substitution?** Any subclass must be drop-in usable for its base class without breaking behavior — no strengthened preconditions or surprising side effects (the Square/Rectangle trap).
4. **Interface Segregation in one line?** Prefer many small, focused interfaces over one fat interface so clients don't depend on methods they don't use.
5. **Dependency Inversion in one line?** Depend on abstractions, not concretions; inject the concrete implementation from outside (constructor injection).
6. **When do you prefer composition over inheritance?** Almost always — when the relationship is "has-a", when you want loose coupling and runtime flexibility, or to avoid deep/fragile hierarchies. Reserve inheritance for true "is-a" needing polymorphic substitution.
7. **`unordered_map` vs `map`?** `unordered_map` is O(1) average (O(n) worst-case), unordered; `map` is O(log n) guaranteed and keeps keys sorted. Default to `unordered_map` unless you need ordering or worst-case guarantees.
8. **How do you turn an O(n²) lookup into O(n)?** Replace the inner linear search with an `unordered_set`/`unordered_map` for O(1) membership — trade O(n) space for an n-factor of time.
9. **How would you make the checkout support a new discount type without changing existing code?** Implement a new class deriving from the `Discount` interface (override `computeDiscount`) and inject it into `Register::checkout`. No existing class is modified — that's OCP via the Strategy pattern.
10. **How do you handle produce priced by weight in the checkout?** A `ProduceProduct` overrides `priceFor(weight)` returning `pricePerPound * weight`, while `UnitProduct` returns `unitPrice * quantity`. The register calls `priceFor` polymorphically and never branches on type.
11. **What does it mean to "program to an interface"?** Depend on an abstract type (pure-virtual base) rather than a concrete class, so implementations are swappable — this is DIP in practice and what makes code testable/extensible.
12. **First thing you do when handed code to optimize?** Read it back and confirm what it does, then check correctness/edge cases — *before* touching performance. Then state the current Big-O, propose the better structure with its new Big-O, then clean up naming/SOLID.
13. **How do you avoid unnecessary copies in C++?** Pass read-only params by `const&`, `reserve()` when the size is known, `std::move` to transfer ownership, `emplace_back` to construct in place, and rely on RVO/NRVO for returns.

---

# Self-test checklist

Refactoring round:
- [ ] I read the code back and confirmed its purpose before editing.
- [ ] I checked correctness and edge cases (empty, duplicates, overflow, off-by-one) first.
- [ ] I stated the **current Big-O**, then the **new Big-O** after each change.
- [ ] I can instantly spot: O(n²) lookup → hash map; recompute → precompute/memoize; wrong container; copies → `const&`/`reserve`/`move`; string `+` in a loop; redundant passes; missing early-exit.
- [ ] After speeding it up, I improved naming, killed magic numbers, applied DRY, and split responsibilities.
- [ ] I named which SOLID principle each cleanup served and asked "want me to push further?"

SOLID & clean code:
- [ ] I can give the one-line spoken definition of all five SOLID letters.
- [ ] For each, I can show a violation and the fix in C++.
- [ ] I can argue composition-over-inheritance with a concrete example.

OOD:
- [ ] I run the framework: clarify scope → entities → responsibilities/relationships → APIs → tricky rules → extensibility.
- [ ] I can design the **checkout system** end-to-end and explain how a new discount plugs in via the `Discount` interface (OCP) and how produce-by-weight works via polymorphism.
- [ ] I can sketch parking lot, deck of cards, and a state-machine (vending/elevator) with class skeletons.
- [ ] For every design I can point to where SRP, OCP, and DIP show up — and where a *new requirement* would plug in without editing existing classes.
