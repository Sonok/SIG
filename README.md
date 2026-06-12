# 🌊 SIG Trading System Engineering — Interview Prep

**Candidate:** Sonok Mahapatra · **Role:** Trading System Engineering Internship, Summer 2027
**📞 Phone screen:** Tuesday, **June 16** · **Today:** June 11

> This is your daily checklist. Open it, find today's date, do the ~1 hour, check the boxes.
> Full study material lives in [`prep/`](prep/). Start with [`prep/00_ROADMAP.md`](prep/00_ROADMAP.md).

---

## 🧭 The big picture (verified from real candidate reports)

```
[✅ CodeSignal OA]  →  [R1] Recruiter  →  [R2] 90-min Technical  →  [R3] Final Day  →  Offer
   DONE                 Tue Jun 16          ~1–2 wks later           after R2
                        (mostly behavioral) (the real test)          (design + fit)
```

**R2 — the 90-min technical (SIG's signature format):**
1. **Code optimization / refactoring** — they hand you bad code; improve complexity + apply SOLID → `prep/09`
2. **DSA problem-solving** with follow-ups (binary search / BFS style) → `prep/04`
3. **OOP class design + project deep-dive** → `prep/09` + `prep/10`

**R3 — final day:** OOP system design (e.g. grocery-store checkout) `prep/09` · programming before a senior eng `prep/04` · project deep-dive `prep/10` · 2× manager behavioral chats `prep/08`.

**Strategy:** Tuesday is mostly behavioral (you've got this). Don't wait for results to start the technical ramp — begin it the day after the screen so you're sharp whenever R2 lands.

---

## 📅 PHASE 0 — Phone Screen Week (light, you're 90% ready)

### ☐ Thu Jun 11 (today) — Story foundation `~1h`
- [ ] Read [`prep/00_ROADMAP.md`](prep/00_ROADMAP.md) (10 min)
- [ ] Read [`prep/08_behavioral_and_stories.md`](prep/08_behavioral_and_stories.md)
- [ ] Draft + say out loud **"Tell me about yourself"** (5×) and **"Why SIG"**

### ☐ Fri Jun 12 — STAR stories `~1h`
- [ ] Say your **AWS**, **Probanker**, **SIF** stories out loud in "I"-form
- [ ] Prepare **Story 4** (a struggle/failure — the one people forget)
- [ ] Skim the `01` Rapid-fire Q&A so a surprise C++ question can't rattle you

### ☐ Sat Jun 13 — Light `~30m`
- [ ] Finalize your **4–5 questions to ask them**
- [ ] One clean run of "Tell me about yourself" + "Why SIG"

### ☐ Sun Jun 14 — Light `~30m`
- [ ] Re-say all 4 STAR stories once
- [ ] Logistics check: confirm Calendly **time + timezone**

### ☐ Mon Jun 15 — Dress rehearsal `~45m`
- [ ] Full out-loud run: intro → why SIG → 4 stories → your questions
- [ ] Quiet room, signal, water, résumé printed. Then **relax** — no cramming.

### ⭐ Tue Jun 16 — PHONE SCREEN
- [ ] 30 min before: re-read `08` + skim `01` rapid-fire
- [ ] Smile (it carries through the phone). Speak in "I." You've got this.

---

## 📅 PHASE 1 — C++ Core & Live Coding (Week of Jun 17)

> Start regardless of screen results. This is the highest-leverage week.

### ☐ Wed Jun 17 — C++ part 1 `~1h`
- [ ] [`01_cpp_fundamentals`](prep/01_cpp_fundamentals.md) §1–4: memory layout · pointers vs refs · const · RAII
- [ ] Re-explain each **out loud** as if teaching

### ☐ Thu Jun 18 — C++ part 2 `~1h`
- [ ] `01` §5–8: rule of 0/3/5 · move semantics · smart pointers · virtual/vtables

### ☐ Fri Jun 19 — C++ part 3 `~1h`
- [ ] `01` §9–15: layout/padding · templates/CRTP · UB · exceptions · lambdas
- [ ] Do all 24 **Rapid-fire Q&A** aloud

### ☐ Sat Jun 20 — Live coding setup `~1h`
- [ ] [`04_dsa_live_coding`](prep/04_dsa_live_coding.md): performance script + STL cheat sheet
- [ ] Solve **5** from the drilling plan **in C++, narrating out loud**

### ☐ Sun Jun 21 — Patterns `~1h`
- [ ] `04` core patterns (two pointers, sliding window, hashing, BFS/DFS)
- [ ] Solve **5 more** problems aloud

### ☐ Mon Jun 22 — HFT problems `~1h`
- [ ] `04`: implement **LRU cache** + **order book** from scratch, narrating
- [ ] 10 min Zetamac warm-up

### ☐ Tue Jun 23 — ⭐ Code optimization & SOLID (R2 CORE) `~1h`
- [ ] [`09_code_optimization_oop_design`](prep/09_code_optimization_oop_design.md) Part A + B: refactor bad code → better complexity + SOLID
- [ ] Practice the "correctness → complexity → clarity → design" script aloud

---

## 📅 PHASE 2 — R2 finishers, Performance & Design (Week of Jun 24)

> ⭐ = directly tested in the 90-min technical round (highest yield). The rest is depth + passion signal for the final round and shows you "get" the actual job.

### ☐ Wed Jun 24 — ⭐ OOP design (R2 CORE) `~1h`
- [ ] `09` Part C: OOD framework + **grocery-store checkout** in depth + parking lot
- [ ] Whiteboard the checkout classes out loud (extend it with a new discount type)

### ☐ Thu Jun 25 — ⭐ Project deep-dive (R2 CORE) `~1h`
- [ ] [`10_project_deep_dive`](prep/10_project_deep_dive.md): lock your **AWS** + **SIF** walk-throughs
- [ ] Practice the 2-min unprompted version of each, in "I" form, ready to whiteboard

### ☐ Fri Jun 26 — Concurrency `~1h`
- [ ] [`03_concurrency`](prep/03_concurrency.md) §1–6 + write the **SPSC ring buffer** by hand

### ☐ Sat Jun 27 — Low latency `~1h`
- [ ] [`02_low_latency_performance`](prep/02_low_latency_performance.md): latency table · cache · false sharing
- [ ] Read **"How to talk about your AWS work"** — rehearse it aloud

### ☐ Sun Jun 28 — Systems + trading design `~1h`
- [ ] [`05_systems_os_networking`](prep/05_systems_os_networking.md): nail **UDP multicast vs TCP**
- [ ] [`07_system_design_trading`](prep/07_system_design_trading.md): **order book** walkthrough aloud

### ☐ Mon Jun 29 — Mock day `~1h`
- [ ] One mock **code-optimization** problem + one **DSA** problem, narrated and timed
- [ ] One mock **OOD** prompt aloud · note weak spots

### ☐ Tue Jun 30 — Final review `~1h`
- [ ] Review the **"10 things"** + the **R2 must-knows** below
- [ ] Self-test checklists at the bottom of each file — anything you can't explain → revisit

---

## 🔁 Always-on (throughout)

- [ ] **Zetamac / mental math** — 10 min, few times a week (stay at 60+)
- [ ] **Narrate everything** — never solve a problem silently; the talking is half the grade
- [ ] **Teach-back test** — if you can't explain it out loud simply, you don't own it yet

---

## 🎯 R2 must-knows (the 90-min technical — your next real hurdle)

1. Take ugly code → state its Big-O → improve it → then clean it with **SOLID** (`09`)
2. Name & apply the **5 SOLID principles** with a quick C++ example each (`09`)
3. Design the **grocery-store checkout** (or parking lot) OOP system, and extend it without editing old code (Open/Closed) (`09`)
4. Live-code a **DSA medium** (binary-search/BFS style) in clean C++ while narrating + testing (`04`)
5. 2-minute **project deep-dive** on AWS + SIF — architecture, key decisions, complexity, hardest bug, in "I" form (`10`)

## ✅ The 10 must-knows (from `00_ROADMAP.md`)

1. Stack vs heap · pointers vs references · const correctness
2. RAII + rule of 0/3/5 + move semantics — explain the *why*
3. Smart pointers (unique vs shared + shared_ptr's atomic cost)
4. Virtual/vtables — and why HFT avoids them on the hot path (→ CRTP)
5. Cache lines · locality · false sharing (+ the latency table)
6. Why no allocation on the hot path (pools / preallocation)
7. SPSC lock-free ring buffer — write it + explain memory ordering
8. UDP multicast (market data) vs TCP (order entry) — and why
9. Live-code a medium problem in clean C++ while narrating + testing
10. Design a basic order book + reason about data-structure trade-offs

> Master these ten and you're a strong candidate. Go get it. 🌊
