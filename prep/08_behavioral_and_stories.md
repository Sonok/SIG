# Behavioral & Your Stories (R1 + every round)

> You said you've prepped behaviorals and present well — so this is **polish, not teaching.** The goal: convert your résumé into crisp, spoken, **"I"-statement** stories (SIG's guide explicitly says *avoid "we"; tell us YOUR individual contribution*).

---

## SIG's own tips (from the guide in this repo)

- Practice talking through your résumé out loud to a peer.
- For any project/task: **approach → results → significance** (specific numbers).
- **Avoid "we."** Say what *I* designed, *I* built, *I* measured.
- They want passion for tech, learning, and problem-solving to show through.

---

## "Tell me about yourself" (60–90 sec) — beats to hit

1. **Who:** "I'm a CS + Applied Math double major at Maryland, graduating December 2027, with a computational-finance focus."
2. **What drives you:** "I love problem-solving under pressure — that's what pulled me into competitive math and competitive programming, and it's why I'm drawn to trading systems."
3. **Most relevant now:** "Right now I'm interning at AWS building low-latency C/C++ networking — sub-microsecond data paths, cache-aware code, lock-free concurrency. That performance-engineering work is basically the toolkit of trading systems, which is exactly why this role excites me."
4. **The trading thread:** "On the side I built a live systematic equity trading system, so I'm comfortable with markets and microstructure too."
5. **Close:** "So I'm looking for a place where deep technical performance work meets markets — and SIG is the clearest fit I've found."

> Memorize the **beats**, not the words. Say it 5× out loud until it flows.

---

## Why SIG / Why trading systems engineering (your strongest pitch)

You have an unusually honest, strong answer — use it:
- **Performance engineering is what I already love and do** (AWS low-latency C/C++).
- **Markets reward exactly the thing I'm good at** — EV, fast reasoning under pressure, competition (Jane Street estimation, Bridgewater prediction markets, competitive math).
- **It's the intersection** of low-latency systems + markets + math, and few places live at that intersection like SIG.
- A line that lands: *"Trading systems are competitive programming with real stakes — correctness, speed, and expected value under time pressure. That's the exact game I've been training for."*

---

## Your STAR stories (converted to "I")

Use **Situation → Task → Action → Result → Significance**. ~60 sec each. Numbers included.

### Story 1 — Technical depth / performance (AWS) ⭐ your strongest
- **S:** On the AWS Networking Infrastructure team, the data-plane code paths needed to hit sub-microsecond latency at line rate.
- **T:** I was responsible for performance-critical C/C++ components in the kernel/userspace data path.
- **A:** I profiled hot paths, optimized for cache locality, applied lock-free concurrency to remove contention, and tuned for tail-latency.
- **R:** Reduced tail latency / improved throughput on production network infrastructure. *(Insert the specific metric you're comfortable sharing.)*
- **Significance:** "This is the same toolkit trading systems use — which is why I'm confident I'd ramp fast on SIG's low-latency C++."

### Story 2 — Measurable impact / optimization (Probanker)
- **S:** Interbank market-simulation pipelines were too slow for large concurrent scenario runs.
- **T:** I owned cutting the end-to-end runtime.
- **A:** I vectorized hot loops, wrote C/C++ extensions for the heavy paths, and parallelized the simulation engine with OpenMP and cache-aware memory layouts.
- **R:** **Cut end-to-end runtime by 40%** and scaled to 1K+ concurrent users.
- **Significance:** Shows I find and kill bottlenecks — the core instinct of a systems engineer.

### Story 3 — Markets + end-to-end ownership (SIF trading system)
- **S:** I wanted to build a systematic equity strategy that actually traded live.
- **T:** Design the signal, the backtest framework, and the live deployment.
- **A:** I built a composite microstructure signal (Kyle λ, Amihud illiquidity, spreads, inverse turnover), wrote a Python alpha framework to backtest and deploy via Alpaca with automated sizing and stop-losses.
- **R:** Best strategy hit **1.8 Sharpe over 18 months, 12% max drawdown**, robust to FF5 + momentum controls.
- **Significance:** I understand markets and microstructure — not just systems — so I'd "get" why the latency matters.

### Story 4 — Struggle / failure / learning (have one ready)
- Pick a real one (a bug that took days, a research dead-end at WCSU, a strategy that overfit). Structure: what went wrong → what *I* did → what *I* learned. Interviewers love genuine reflection. **Prepare this — it's the most commonly missed story.**

---

## Questions to ask THEM (have 4–5; asking good ones is a real signal)

- "What does the C++ low-latency work actually look like day-to-day for an intern on a strategy team?"
- "How do interns get ramped on markets knowledge during the summer?"
- "What separates an intern who gets a return offer from one who doesn't?"
- "How much of the work is greenfield vs. optimizing existing systems?"
- "What's the most interesting performance problem the team has hit recently?"

---

## Logistics checklist for Tuesday

- [ ] Confirm Calendly time + timezone (it's a phone call).
- [ ] Quiet room, good signal, water nearby.
- [ ] Résumé printed/open in front of you.
- [ ] This file + the `01` rapid-fire Q&A skimmed (in case of the "brief exercise").
- [ ] Smile when you talk — it carries through the phone.
- [ ] One practice run of "tell me about yourself" + "why SIG" right before.
