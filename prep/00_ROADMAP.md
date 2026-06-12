# SIG Trading System Engineering — Interview Prep Roadmap

> Candidate: Sonok Mahapatra · Role: Trading System Engineering Internship, Summer 2027
> Phone screen: **Tue, June 16**. This kit is built for the technical rounds that follow.

---

## Where you are in the pipeline

```
[R1] Recruiter Interview      ← YOU ARE HERE (Tue June 16)
      experience · interest · technical acumen · BRIEF technical exercise
         │
[R2] Technical Interview — LIVE CODING
      practical C++, data structures, narrate-while-you-code
         │
[R3] Final Round — DESIGN & TEAM FIT
      multiple interviewers · problem-solving scenarios · system design · fit
         │
[Offer]
```

**Important (from SIG's own guide in this repo):** even R1 the recruiter says they "delve into your technical skills by completing a brief exercise." So keep something technical warm even for Tuesday — but it's light. The heavy technical bar is R2 and R3.

---

## Your honest scouting report

**Strengths (lean into these — they're elite):**
- Competitive math / EV / estimation: Putnam Top 700, 5× AIME, Jane Street Estimation Top 3, Bridgewater Prediction Market Top 35 world, Zetamac 60+. → SIG's probability/market-making rounds are your home turf.
- **Current AWS low-latency C/C++ networking internship.** This is *the* perfect résumé line for this role — sub-microsecond latency, cache locality, lock-free concurrency, kernel/userspace data paths. You can speak about HFT-relevant performance engineering from real experience.
- Real markets knowledge: live systematic trading system, microstructure signals. Most candidates can't talk markets — you can.
- Behavioral: you've prepped these and present well.

**The gap to close (where this kit points):**
- **C++ language depth / "how it works under the hood"** — the predictable internals questions. Very learnable.
- **Coding cleanly while narrating out loud** — not algorithm difficulty (CF 1100 is fine for SIG), but calm, clear, tested code under observation.

---

## The kit (read in this order)

| # | File | Round it serves | Priority |
|---|------|-----------------|----------|
| 01 | `01_cpp_fundamentals.md` | R2 + R3 | 🔴 highest — your gap |
| 02 | `02_low_latency_performance.md` | R3 + talking about AWS | 🔴 high — your edge if mastered |
| 03 | `03_concurrency.md` | R2 + R3 | 🟠 high |
| 04 | `04_dsa_live_coding.md` | R2 | 🔴 highest — the live coding round |
| 05 | `05_systems_os_networking.md` | R3 | 🟠 medium-high |
| 06 | `06_probability_ev_mental_math.md` | R1/R3 sprinkles | 🟢 maintain (already strong) |
| 07 | `07_system_design_trading.md` | R3 | 🟠 high |
| 08 | `08_behavioral_and_stories.md` | R1 + every round | 🟢 polish |
| 09 | `09_code_optimization_oop_design.md` | **R2 core** + R3 | 🔴 highest — SIG's signature round |
| 10 | `10_project_deep_dive.md` | **R2 + R3** | 🔴 high — they WILL ask |

---

## Suggested study schedule

Assumes ~1 hr/day, R2 likely ~1–2 weeks after you pass Tuesday (SIG says feedback can take up to 2 weeks, so you have runway).

### Before Tuesday June 16 (light — you're mostly ready)
- **Mon/Tue:** Read `08_behavioral_and_stories.md`. Say your AWS + SIF stories out loud in "I" form. Skim the `01` rapid-fire Q&A so a surprise "what's a reference vs a pointer?" can't rattle you. Have your questions-for-them ready.

### Week 1 after (assuming you advance) — build the C++ core
- Day 1–2: `01_cpp_fundamentals.md` — sections 1–8 (memory, pointers/refs, const, RAII, rule of 5, move, smart ptrs, virtual). Re-explain each out loud.
- Day 3: `01` sections 9–15 + the rapid-fire Q&A.
- Day 4–5: `04_dsa_live_coding.md` — STL cheat sheet + patterns. Start the drilling plan (solve in C++, narrate aloud).
- Weekend: keep solving from the `04` drilling list; warm up Zetamac 10 min/day.

### Week 2 — performance, systems, design
- Day 1–2: `02_low_latency_performance.md` (this is where you fuse AWS experience + HFT).
- Day 3: `03_concurrency.md` — focus on the SPSC ring buffer; be able to write/critique it.
- Day 4: `05_systems_os_networking.md` — nail "UDP multicast vs TCP for market data."
- Day 5: `07_system_design_trading.md` — walk through the order-book design out loud.
- Throughout: a few `06` probability problems to stay sharp.

### Always-on
- Zetamac / mental math: 10 min, a few times a week (you're at 60+, just stay warm).
- Re-explain concepts out loud, as if teaching — that's the test of real understanding.
- Every C++ problem: solve it *while talking*, then test it. The narration is half the grade.

---

## The 10 things that, if mastered, clear this interview

1. Stack vs heap, pointers vs references, const correctness — instant, fluent answers.
2. RAII + rule of 0/3/5 + move semantics — explain *why*, not just *what*.
3. Smart pointers — unique vs shared (+ the atomic cost of shared_ptr).
4. Virtual functions / vtables — and *why HFT avoids them on the hot path* (→ CRTP).
5. Cache lines, locality, false sharing — the latency table in your head.
6. Why avoid allocation on the hot path; pools/preallocation.
7. The SPSC lock-free ring buffer — write it, explain the memory ordering.
8. UDP multicast for market data vs TCP for order entry — and why.
9. Live-code a medium problem in clean C++ while narrating + testing.
10. Design a basic order book and reason about its data-structure trade-offs.

Master these ten and you are a strong candidate. You already own #5–#6 ideas from AWS and the math behind everything. Go get it.
