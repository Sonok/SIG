# Project Deep-Dive Script (Technical round + Final round)

> Verified from candidate reports: SIG's 90-min technical round AND the final round both ask you to **"walk through a technical project in depth"** — why you built it, the problem it solves, libraries/algorithms/data structures, complexity, and a demo/whiteboard of the architecture. This is **graded technical content, not chit-chat.** And it plays straight to your strengths.

---

## How they probe (be ready for ALL of these on whichever project you pick)

1. *What problem does it solve and why did it matter?*
2. *What was YOUR individual contribution?* (avoid "we" — SIG says this explicitly)
3. *Walk me through the architecture.* (be ready to whiteboard a box diagram)
4. *Why did you choose X over Y?* (language, data structure, library, design)
5. *What was the hardest bug / bottleneck, and how did you find and fix it?*
6. *What's the time/space complexity of the core piece?*
7. *How did you test / validate it?*
8. *What would you do differently or how would it scale?*

**Pick ONE project to go deepest on. Recommended: AWS** (most role-relevant) — with **SIF** as your backup if they want a markets-flavored one.

---

## Deep-dive framework (use for any project)

**Problem → Your role → Architecture → Key decisions (why) → Hardest challenge → Results → What's next.** ~2 min unprompted, then let them dig.

---

## 🅰 Project 1 — AWS low-latency networking (your strongest, most role-relevant)

**One-liner:** *"I build performance-critical C/C++ components in AWS's network data plane, optimizing kernel and userspace data paths for sub-microsecond latency at line rate."*

**Walk-through (the beats):**
- **Problem:** the data plane has to move packets at line rate with sub-microsecond latency; any inefficiency on the hot path shows up as tail latency across the network fabric.
- **My role:** I own [specific component] — profiling it and optimizing the hot path.
- **Architecture:** packet ingress → [kernel/userspace path] → processing → egress. *(Draw the box diagram; mark the hot path.)*
- **Key decisions / techniques:** cache-locality-aware data layout, lock-free concurrency to remove contention, minimizing allocations and copies on the hot path, tail-latency tuning.
- **Hardest challenge:** *(have a concrete one — a latency spike you root-caused via profiling/percentiles, a false-sharing or contention issue, etc.)*
- **Results:** [your metric — latency reduction / throughput].

**Anticipated probes & how to answer (ties to your prep files):**
- *"Why does cache locality matter here?"* → latency table; L1 ~1ns vs DRAM ~100ns; cache-line/false-sharing (see `02`).
- *"What does 'lock-free' buy you?"* → no blocking, bounded latency, no priority inversion; mention SPSC ring buffer (see `03`).
- *"Why is allocating on the hot path bad?"* → malloc is slow & non-deterministic; use pools/preallocation (see `02`).
- *"How did you measure latency?"* → p99/p99.9, not averages; warmup, steady-state, rdtsc (see `02`).
- **The kill shot:** explicitly connect it — *"this is the exact toolkit a trading system uses; exchange connectivity and market-data handling are the same performance problem, which is why this role excites me."*

⚠️ **Only claim what you actually did.** Have the real specifics (component, metric, the one hard bug) locked. If you're fuzzy on a number, say "on the order of…" — never invent precise figures.

---

## 🅱 Project 2 — Live Systematic Equity Trading System (SIF) — your markets-flavored project

**One-liner:** *"I built a live systematic equity strategy end-to-end: a composite microstructure signal, a backtest framework, and live deployment via Alpaca."*

**Walk-through:**
- **Problem:** find a tradeable edge from microstructure and deploy it live with proper risk controls.
- **My contribution:** designed the composite signal (Kyle λ, Amihud illiquidity, bid-ask spread, inverse turnover), built a Python alpha-class framework to backtest and deploy, added automated position sizing and stop-losses.
- **Architecture:** data ingestion → signal computation → backtest engine → live execution via Alpaca → risk/sizing layer. *(Whiteboard it.)*
- **Key decisions:** why those four signal components; how you combined them; how you avoided look-ahead/overfitting.
- **Results:** **1.8 Sharpe over 18 months, 12% max drawdown**, robust to FF5 + momentum controls.
- **Rigor signal:** "results held after controlling for known factors" — shows you know the difference between real alpha and data-mined noise.

**Anticipated probes:**
- *"How did you avoid overfitting / look-ahead bias?"* → out-of-sample, factor controls, point-in-time data.
- *"What's the complexity / how does it scale to more symbols?"* → vectorized signal computation, where the bottleneck is.
- *"Why microstructure signals?"* → tie to your understanding of why latency/microstructure matters — bridges to TSE.

---

## 🅲 Project 3 — Probanker (optimization story, good as a "performance win" example)

**One-liner:** *"I cut an interbank-simulation pipeline's runtime by 40% with vectorization, C/C++ extensions, and OpenMP parallelization with cache-aware layouts."*

Use this if they want a **"tell me about a time you optimized something"** story. Beats: slow pipeline → profiled to find hot loops → moved heavy paths to C/C++ + vectorized → parallelized with OpenMP + cache-aware memory → **40% faster, scaled to 1K+ concurrent users.** Same instincts as the AWS work, different domain.

---

## Prep checklist for the deep-dive

- [ ] For **AWS**: lock down your specific component, one concrete hard problem you solved, and one real metric.
- [ ] For **SIF**: be able to whiteboard the pipeline and defend the overfitting question.
- [ ] Have **Probanker** ready as the "optimization" story.
- [ ] Practice the 2-minute unprompted walk-through of each, **out loud**, in "I" form.
- [ ] Be ready to **draw the architecture** of any of them on a whiteboard/shared screen.
- [ ] Know the **complexity** of the core algorithm in each.
- [ ] For every project, prepare the "**what would you do differently / how would it scale**" answer.
