# 06 — Probability, EV & Mental Math (SIG Trading Systems Engineering)

## Why this matters & how to use this file

SIG is built on poker and expected value. Even for a **Trading Systems Engineering** role, they screen heavily for the EV/probability instinct — it's the firm's culture and they want engineers who think like traders. The good news: **you're already strong here** (Putnam Top 700, 5x AIME, Jane Street Estimation Top 3, Bridgewater Top 35, Zetamac 60+). So this is not a tutorial. It's:

1. A **refresher of the exact question types** SIG asks.
2. The **trading framing** — how each puzzle maps to market making, fair value, and EV-of-a-bet. This is what separates a mathlete from someone SIG wants to hire. They don't just want the number; they want you to *price* it.
3. **Lots of representative problems with full solutions** so you stay sharp.

Read the framing section first, then drill the categories, then do the self-test cold.

---

## 1. The market-making / EV mindset (read this first)

Everything SIG asks is secretly the same question: **"What is the fair value, and would you bet on it?"**

### Fair value, bid/ask, edge
- **Fair value (FV):** your best point estimate of a random quantity — its expected value, the price at which you're indifferent to buying or selling.
- **Bid / Ask:** you quote a **two-sided market**: a price you'll *buy* at (bid) and a higher price you'll *sell* at (ask). The midpoint ≈ your FV. The **width** (ask − bid) reflects your uncertainty.
- **Edge:** the difference between price and true value. You only want to transact when price gives you positive expected edge.

> Trader's rule: **buy below fair, sell above fair.** If someone offers to sell you a die roll's payoff for \$3 and the EV is \$3.50, you buy. If they'll pay you \$4 for it, you sell.

### "Make me a market on X"
You'll be asked: *"Make me a market on the number of windows in this building"* / *"...on the sum of two dice"* / *"...on how many people are flying right now."*

Procedure:
1. Compute or estimate the **expected value** → that's your midpoint.
2. Set a **width** around it based on confidence. Confident → tight (e.g. ±0.5). Unsure → wide.
3. Quote: *"7 at 8"* means **bid 7, ask 8** (I buy at 7, sell at 8).

Then the interviewer **trades against you** and may reveal info. Key skills:
- **Adverse selection / information asymmetry:** if they eagerly hit your bid (sell to you), assume the true value is *below* your bid — they know something. A good market maker **widens** when facing informed flow and **moves the quote** in the direction of the trade. After being lifted (they bought your ask), raise your market.
- **Tighten** your market when you have hard knowledge (e.g. a market on a sum of dice — you can compute it exactly, so quote tight). **Widen** for genuine unknowns (population estimates, things you can be badly wrong about).
- Never quote a market so wide it's useless, nor so tight you get picked off. The interviewer is testing whether you understand that **a quote is a commitment to trade at those prices.**

### "Would you take this bet?"
Reduce to EV (and sometimes risk/variance):
- Compute EV of the bet. Positive EV → take it (if sizing is reasonable). Negative → decline or take the other side.
- State the EV, the breakeven, and your edge. *"EV is +\$0.50 per dollar risked, I take it."*
- Mention **risk** when relevant: a +EV bet you can't survive the variance of (bet-the-farm, ruin risk) may be declined. SIG loves when you raise the Kelly/ruin consideration unprompted.

### The master template for every problem
1. **State assumptions** (fair die? independent? with replacement?).
2. **Identify the random variable** and what's being priced.
3. **Compute EV / probability** (lean on linearity of expectation — see below).
4. **Sanity check** (bounds, symmetry, extreme cases).
5. **Give the trading answer**: fair value, would-I-take-the-bet, where I'd quote.

---

## 2. Expected value & the linearity superpower

**Definition:** EV[X] = Σ xᵢ·P(xᵢ) (discrete) or ∫ x·f(x) dx (continuous).

**Linearity of expectation:** E[X + Y] = E[X] + E[Y], **always — even when X and Y are dependent.** This is the single most powerful trick for these puzzles. Whenever you're asked "expected number of ___," try to write the count as a **sum of indicator variables** and add up their probabilities. No independence required.

- **Indicator trick:** Let Iₖ = 1 if event k happens. Then E[count] = Σ P(event k).
- **Expected value of a geometric process:** if each trial succeeds with prob p, expected trials until first success = 1/p.
- **First-step / recursion:** for "expected time until X," condition on the first step and set up E = 1 + Σ P(next state)·E[from next state].

---

## 3. Problem categories with worked solutions

### (a) Dice problems

**P1. EV of a single fair die.** (3.5.) Sum 1..6 = 21, /6 = **3.5**. *Trading framing: fair value of one roll is 3.5; market "3 at 4" is reasonable, tight because you know it exactly.*

**P2. Roll a die; you may re-roll once and must take the second roll. What's the EV? Where's the optimal stopping threshold?**
You re-roll only when the first roll is below the EV of a fresh roll (3.5), i.e. re-roll on 1,2,3; keep 4,5,6.
- Keep 4,5,6: each prob 1/6, contributes (4+5+6)/6 = 15/6.
- Re-roll on 1,2,3 (prob 3/6): get a fresh roll worth 3.5 → contributes (3/6)·3.5 = 10.5/6.
- EV = (15 + 10.5)/6 = 25.5/6 = **4.25.**
*Optional stopping logic: keep any roll ≥ the value of continuing.*

**P3. Two re-rolls allowed (three rolls max), take the last if you reach it. EV?**
Work backwards. With one roll left, value = 3.5. With two rolls left: keep if ≥ 3.5 → that's P2's answer, **4.25.** With three rolls left: keep first roll if ≥ 4.25 → keep 5,6 (each 1/6), else continue with value 4.25.
- Keep 5,6: (5+6)/6 = 11/6.
- Continue (rolls 1–4, prob 4/6): worth 4.25 → (4/6)·4.25 = 17/6.
- EV = (11 + 17)/6 = 28/6 ≈ **4.667.**

**P4. Expected number of rolls to see all six faces (coupon collector).**
E = 6·(1/6 + 1/5 + 1/4 + 1/3 + 1/2 + 1/1) = 6·H₆ = 6·2.45 = **14.7.**

**P5. Expected number of rolls until you get a 6.** Geometric, p = 1/6 → **6.**

**P6. Roll until the running sum is ≥ a target — or "keep rolling, bust if sum exceeds 12"-style.** Solve by first-step recursion on the sum state. (See Markov section P-Mk for the technique; e.g. expected rolls to reach sum ≥ N grows ~ 2N/7 since average step is 3.5.)

---

### (b) Coin problems

**P7. Expected flips to get the first head.** p = 1/2 → **2.**

**P8. Expected flips until HH vs until HT.** Classic, and the answer is counterintuitive — *great* trading question because people misprice it.
- **HT:** wait for an H (2 flips expected), then you need a T; once you have an H, a non-T just keeps you "primed." E[HT] = **4.**
- **HH:** E[HH] = **6.** Why more? After H then T you fall *all the way back to start*; HT never loses progress that way.
- Recursion for HH: let a = E[flips | empty], b = E[flips | one H].
 a = 1 + ½a + ½b; b = 1 + ½·0 + ½a. Solve: b = 1 + ½a, a = 1 + ½a + ½(1+½a) ⇒ a = 1.5 + ¾a ⇒ ¼a = 1.5 ⇒ a = **6.**

**P9. Expected flips for pattern HTH vs HHH.** Use Conway's leading-number / correlation method, or recursion.
- **HHH:** E = 2³⁺¹ − 2 ... use the formula E[pattern] = Σ over self-overlaps 2^(overlap length). HHH overlaps itself at lengths 3,2,1 → 2³+2²+2¹ = **14.**
- **HTH:** overlaps at length 3 and length 1 (HTH's prefix "H" = suffix "H") → 2³ + 2¹ = **10.**
*Framing: HHH is "slower" — if asked to bet which pattern appears first in a race, HTH is favored. You'd quote the race accordingly.*

**P10. Gambler's ruin.** Start with \$k, bet \$1 per fair coin flip, stop at \$0 or \$N.
- P(reach N before 0) = **k/N** (fair game).
- Expected number of flips until absorption = **k·(N − k).**
- Biased version (win prob p ≠ ½, q = 1−p, r = q/p): P(reach N) = (1 − rᵏ)/(1 − rᴺ).
*Framing: this is risk-of-ruin and bankroll math — directly relevant to position sizing.*

---

### (c) Card problems

**P11. P(both cards red) when drawing 2 from a 52-card deck (no replacement).** (26/52)(25/51) = 25/102 ≈ **0.245.**

**P12. Expected number of aces in a 5-card hand.** Linearity: each card is an ace with prob 4/52; E = 5·(4/52) = 20/52 = **5/13 ≈ 0.385.** (Indicator trick beats hypergeometric algebra.)

**P13. Expected number of "rising sequences"/adjacent same-suit, or: shuffle a deck, expected number of cards in their original position (fixed points).** Each of 52 positions is a fixed point with prob 1/52; E = 52·(1/52) = **1.** (Same as the hat-check / derangement expectation — always 1 regardless of n.)

**P14. Draw cards one at a time without replacement until the first ace. Expected position of the first ace.** The 4 aces split the 48 non-aces into 5 gaps of equal expected size 48/5 = 9.6; first ace expected at position 9.6 + 1 = **10.6.** (Equivalently (52+1)/(4+1) = 53/5 = 10.6.)

---

### (d) Combinatorics & counting

**P15. How many ways to arrange the letters of "BANANA"?** 6!/(3!·2!·1!) = 720/12 = **60.**

**P16. Number of paths from (0,0) to (m,n) moving only right/up.** C(m+n, m). E.g. (4,4): C(8,4) = **70.**

**P17. How many ways to choose a committee of 3 from 10?** C(10,3) = **120.** Probability a specific person is on it = C(9,2)/C(10,3) = 36/120 = **0.3** (or just 3/10 by symmetry).

**P18. Stars and bars: nonnegative integer solutions to x₁+x₂+x₃+x₄ = 10.** C(10+3, 3) = C(13,3) = **286.**

**P19. Birthday problem (estimate).** P(no shared birthday among n) = Π(1 − k/365). Crossover to >50% at **n = 23.** Mental approximation: P(collision) ≈ 1 − e^(−n²/730).

---

### (e) Conditional probability & Bayes

**P20. Two children, at least one is a boy — P(both boys)?** Sample space {BB, BG, GB} (GG excluded) → **1/3.** Contrast: "the *older* is a boy" → **1/2.** (The information you condition on matters; this is an information/adverse-selection lesson.)

**P21. Disease testing / base rates.** Disease prevalence 1%, test 99% sensitive & 99% specific. P(disease | positive)?
- Bayes: (.99·.01) / (.99·.01 + .01·.99) = .0099 / .0198 = **0.5.**
- Lesson: with rare events, false positives dominate. SIG cares because base rates = priors = how you read order flow.

**P22. Monty Hall.** Switch → win prob **2/3**; stay → **1/3.** Host's action reveals information (he never opens the car). *Framing: same structure as Monty being an "informed counterparty."*

**P23. Bayesian coin.** Box has a fair coin and a two-headed coin; pick one at random, flip 3 heads. P(two-headed)?
- P(2H | HHH) = (½·1) / (½·1 + ½·(1/8)) = (1/2)/(9/16) = **8/9.**

---

### (f) Geometric / uniform / continuous probability

**P24. Two people each arrive uniformly at random in [0,60] min, each waits 10 min. P(they meet)?**
Meet iff |X−Y| ≤ 10. Area outside = 2·(½·50²) = 2500; total 3600; P(meet) = 1 − 2500/3600 = 1100/3600 = **11/36 ≈ 0.306.**

**P25. Break a stick of length 1 at a uniform random point; expected length of the longer piece.** Longer piece is max(U, 1−U); E = ∫₀¹ max(u,1−u)du = **3/4.** Shorter piece expectation = **1/4.**

**P26. Break a stick at TWO uniform random points; P(three pieces form a triangle)?** **1/4.** (Each piece must be < 1/2.)

**P27. Expected distance between two uniform points on [0,1].** E|X−Y| = **1/3.** (In a unit square, expected distance between two random points ≈ 0.5214 — worth knowing as an estimation anchor.)

**P28. Uniform point on [0,1]; EV is 1/2; but what's E[max of n uniforms]?** **n/(n+1).** E[min] = 1/(n+1). (Order-statistic facts SIG likes.)

---

### (g) Optional stopping / "should I re-roll / take it"

**General principle:** **Continue iff the EV of continuing exceeds the value in hand.** Solve backward from the terminal step. (Dice P2–P3 are the canonical instances.)

**P29. "I'll roll a die. You can stop anytime and take \$(current roll), or roll again. Infinite rolls allowed. Optimal strategy & EV?"**
Let V = EV of the game under optimal play. Stop iff current roll ≥ V. Rolls ≥ ⌈V⌉ are kept.
- Guess threshold = keep 5,6 → V = (5+6)/2-weighted... solve: V = (1/6)(5+6) + (4/6)V ⇒ V = 11/6 + (2/3)V ⇒ (1/3)V = 11/6 ⇒ V = **5.5.** Check: 5 ≥ 5.5? No. So we should only keep 6.
- Re-solve keeping only 6: V = (1/6)(6) + (5/6)V ⇒ (1/6)V = 1 ⇒ V = **6.** But 6 ≥ 6 keep, 5 < 6 continue. Consistent. EV = **6.** Wait—check keeping 5,6 again with V between 4 and 5: if V≤5 we keep 5. Threshold self-consistent solution: keeping 5&6 gives V=5.5 (>5, contradiction → don't keep 5). Keeping only 6 gives V=6 (>5 consistent). **Optimal: keep only a 6, EV = 6**... 

  *(Subtlety: with unlimited free re-rolls and no cost, waiting for a 6 is optimal and EV = 6. If each roll costs \$c, the threshold drops — set up V = max(stop, −c + E[next]) and solve. Always state the cost assumption out loud.)*

**P30. Coin-flipping payout: flip until first tail, payout = number of heads. Stop option each flip to bank current heads or risk one more flip (lose all on a tail).** Continue iff expected gain > expected loss: from h heads, flipping again gives ½(h+1) + ½(0) = (h+1)/2 vs keeping h. Continue iff (h+1)/2 > h ⇒ h < 1 ⇒ only continue when h = 0. So stop after the first head. EV = **0.5** heads banked... but at h=0 keeping gives 0 so you must flip: EV = ½·(stop at 1) + ½·0 = **0.5.** *(Tweak the payout and the threshold shifts — show you can re-derive it.)*

---

### (h) Markov chain / random walk (expected time to absorption)

**Technique:** assign Eₛ = expected steps to absorption from state s; write Eₛ = 1 + Σ P(s→s')·Eₛ'; solve the linear system. Absorbing states have E = 0.

**P-Mk1. Frog on lily pads 0..N, hops +1 or −1 with prob ½ (reflecting/absorbing at ends).** This is gambler's ruin; expected time from k to hit {0,N} = **k(N−k).**

**P31. Ant on a triangle (vertices A,B,C); each step moves to a random adjacent vertex. Expected steps to return to start? Expected steps from A to reach C?**
- From A, to reach C: by symmetry E_A = E_B = E (B and A are symmetric w.r.t. C). E = 1 + ½(0) + ½·E_B, and E_B = 1 + ½(0) + ½E_A ⇒ E = 1 + ½E ⇒ E = **2.** Both A and B reach C in expected 2 steps.

**P32. Random walk on a line, start 0, ±1 each with ½. Expected number of steps to first reach +1?** **Infinite** (recurrent but null — expected hitting time diverges) for the symmetric walk on all of ℤ. *Good trap question; the answer is ∞, not a finite number.* (If there's a reflecting barrier or bias, it becomes finite.)

**P33. Snakes-and-ladders-style / die-driven board:** set up Eₛ = 1 + (1/6)ΣEₛ₊ᵢ and solve from the end backward. Mention you'd just code it for large boards — relevant for the *engineering* role (you can offer the closed-form intuition AND "I'd solve the linear system / simulate").

---

### (i) Estimation / Fermi (you're elite here — keep the structure crisp)

**Method:** decompose into factors you can each estimate to within a small multiple; multiply; sanity-check against an independent anchor; **state your assumptions and give a range, then a point estimate.** Round to convenient numbers and track orders of magnitude.

**P34. How many golf balls fit in a school bus?**
Bus ≈ 2.5m × 2.5m × 12m ≈ 75 m³ ≈ 75×10⁶ cm³. Golf ball ≈ 4 cm diameter → ~34 cm³, but with packing inefficiency (~65% sphere packing) use ~50 cm³ effective. 75×10⁶ / 50 ≈ **~1.5 million.** State the packing-factor assumption.

**P35. How many piano tuners in Chicago?** (The original Fermi.) Pop ~3M → ~1M households → ~1 in 20 has a piano → 50k pianos + institutions ~100k pianos. Tuned ~once/yr. One tuner does ~4/day × ~250 days = 1000/yr. → 100k/1000 = **~100 tuners.** (Actual: ~50–100.)

**P36. How many people are airborne over the US right now?** ~ (flights in air ~5,000) × (avg ~100 passengers) ≈ **~500,000.** Anchor: ~45,000 US flights/day, avg flight ~2h, so ~45000·(2/24) ≈ 3,750 in air × ~120 pax ≈ 450k. Consistent.

**Verbalizing an estimate:** "I'll break this into [factors]. I estimate each as [value] because [reason]. Multiplying gives X. Sanity check against [independent path] gives Y — same order of magnitude, so I'll quote **X**, market **[low] at [high]**." Always close with a market.

---

### (j) Game theory / fair value of a game

**P37. St. Petersburg.** Pay \$2ᵏ where k = flips to first tail. EV = Σ (½)ᵏ·2ᵏ = Σ 1 = **∞**, yet no one pays much. Lesson: **EV isn't everything — utility/risk/bankroll matter.** Great answer to "would you pay \$100 to play?": *"EV is infinite but variance is infinite too; with finite bankroll and risk-aversion (log utility/Kelly), the value is small — I'd pay maybe \$10–20."*

**P38. You and I each roll a die; higher roll wins. Fair price to enter (winner takes \$2 stake, \$1 each)?** Symmetric → fair, EV-neutral. P(tie) = 6/36; P(win) = 15/36. *Pricing twist: if ties go to the house, you'd need a discount to play.*

**P39. Bid for an envelope of unknown value — "Winner's curse" / acquire-a-company.** A company is worth U(0,100) to the seller, worth 1.5× that to you, but you must bid blind and only learn value if you win. Naively bid your expected 1.5·50 = 75; but **conditional on winning** (your bid ≥ value) the expected value is lower. Optimal bid = **\$0** (any positive bid loses in expectation: E[1.5V | V ≤ b] = 1.5·(b/2) = 0.75b < b). **The winner's curse / adverse selection in pure form** — exactly the market-making lesson: condition on the fact that your counterparty agreed to trade.

**P40. Two-envelope, Nash-bargaining, and "guess 2/3 of the average" (Keynesian beauty contest).** For 2/3-of-average: iterated dominance → equilibrium is **0** (or 1 if integers ≥1). Mention that against real people you'd guess ~20–30 (people don't iterate infinitely) — *modeling your counterparty's sophistication is the whole game.*

---

## 4. "Make a market" practice

Format: compute FV → set width by confidence → quote "bid at ask."

| Prompt | Fair value (EV) | Suggested quote | Why this width |
|---|---|---|---|
| Sum of two dice | 7 | **6.5 @ 7.5** | Known exactly; tight |
| Number of heads in 10 fair flips | 5 | **4.5 @ 5.5** | Known dist; tight |
| Number of countries in Africa | ~54 | **48 @ 58** | You roughly know; medium |
| Number of windows in this office building | guess via floors×windows/floor | **wide, e.g. 150 @ 250** | Genuine unknown; wide |
| Your roll on one die (you'll get paid that) | 3.5 | **3 @ 4** | Exact |
| Number of pages in War and Peace | ~1200 | **1000 @ 1400** | Fuzzy; medium-wide |
| Distance NYC→LA in miles | ~2450 | **2300 @ 2600** | You half-know |

**Trading dynamics to demonstrate:**
- When they **lift your ask** (buy from you), **raise both quotes** — they may know it's high, and you're now short.
- When they **hit your bid** (sell to you), **lower both quotes.**
- If they trade aggressively/repeatedly in one direction, **widen** — you're facing informed flow (adverse selection).
- Volunteer: "I'd quote tighter if I could compute it exactly; here it's an estimate so I'll keep width." Show you know *why* the width is what it is.

---

## 5. How to verbalize (interviewers grade the process)

They are listening for a trader's reasoning loop, not just the final number. Out loud, do this:

1. **Restate & state assumptions.** "Assuming a fair die, independent rolls, with replacement..."
2. **Name the tool.** "This is a linearity-of-expectation problem — I'll write it as a sum of indicators." Naming the technique signals fluency.
3. **Compute transparently.** Talk through the arithmetic; don't go silent. If you blank, narrate the structure while you recover.
4. **Sanity-check aloud.** "Answer should be between 0 and 1 — check. Should exceed 3.5 because we re-roll low values — 4.25 does — check. Symmetry says X = Y — check."
5. **Give the trading answer.** Don't stop at the number: "So fair value is 4.25; I'd make a market 4 at 4.5; I'd take that bet at any price under 4.25."
6. **Handle being wrong gracefully.** If they push back, treat it as new information (like order flow): update, don't defend. "If you're hitting my bid that fast, I'm probably too high — let me re-check."
7. **Flag risk when relevant.** Mention variance/ruin/Kelly on bet-sizing questions even unprompted — it's the SIG mindset.

Mantra: **assumptions → EV → sanity check → make the market.**

---

## 6. Mental math (you have Zetamac 60+, so just stay warm)

You're already fast. Maintenance, not learning:
- **Keep doing Zetamac** (arithmetic.zetamac.com) a few minutes daily right up to interviews; aim to hold 60+. Trading firms (SIG included) administer a **timed arithmetic test** — typically ~80 seconds to 2 minutes of rapid +−×÷ on integers (and sometimes decimals), scored on number correct. Some rounds are oral.
- **Speed tricks worth having reflexive:**
 - ×11: ab → a(a+b)b (e.g. 11×52 = 572). 
 - Squares near a base: 48² = (48)(48): (50−2)² = 2500 − 200 + 4 = 2304. (a±b)² expansion.
 - ×5 = ×10/2; ×25 = ×100/4; ÷5 = ×2/10.
 - Percentages: x% of y = y% of x (e.g. 16% of 25 = 25% of 16 = 4).
 - Fraction→decimal anchors: 1/7≈.1428, 1/3=.333, 1/6=.1667, 1/8=.125, 1/9=.111, 1/11=.0909.
 - Difference of squares for products: 47×53 = 50²−3² = 2491.
 - Estimating divisions: convert to multiplication and bracket (is the answer between these two round numbers?).
- For the **EV puzzles**, the bottleneck is rarely arithmetic — it's setup. Spend warmup time on *recognizing the category fast*, not on raw computation.

---

## 7. Self-test — 15 mixed problems (drill cold, then check the key)

1. Roll two dice. What's the probability the sum is 7? What's the EV of the sum?
2. Expected number of flips of a fair coin to first see the pattern **TT**.
3. You draw cards until the first king. Expected position of that first king?
4. A test for a disease is 95% sensitive and 90% specific; prevalence is 2%. P(disease | positive)?
5. Three points are chosen uniformly on a circle. P(all three lie in some common semicircle)?
6. Roll a fair die repeatedly; you may stop and bank the latest roll. Each *extra* roll after the first costs \$1. What's your optimal threshold and EV? (State assumptions.)
7. Two friends agree to meet between 1:00 and 2:00; each arrives uniformly and waits 15 minutes. P(they meet)?
8. Expected number of fixed points when you randomly shuffle 7 distinct cards.
9. In 5 fair coin flips, P(exactly 3 heads)? Expected number of heads?
10. Make a market on the number of times the digit "7" appears in the integers 1 to 100.
11. A fair coin: expected flips until you've seen **both** a head and a tail.
12. Gambler starts with \$3, bets \$1 per fair flip, quits at \$0 or \$7. P(reaching \$7)? Expected number of flips?
13. A stick of length 1 is broken at two uniform random points. Expected length of the **middle** piece.
14. You roll a die; if you don't like it you may re-roll exactly once (and must keep the second). EV?
15. Estimate: how many basketballs would fit inside a typical classroom? (Show your structure and end with a market.)

---

<details>
<summary><b>Answer key (click / scroll to reveal)</b></summary>

1. P(sum 7) = 6/36 = **1/6.** EV of sum = 3.5 + 3.5 = **7** (linearity).
2. **E[TT] = 6** (same recursion as HH; after a failure you reset fully).
3. 4 kings split 48 others into 5 gaps; expected position = (52+1)/(4+1) = **53/5 = 10.6.**
4. Bayes: (.95·.02) / (.95·.02 + .10·.98) = .019 / (.019+.098) = .019/.117 ≈ **0.162** (16.2%). Base rate dominates.
5. P = 3/2^(3−1) = 3/4 = **0.75.** (General n points: n/2^(n−1).)
6. Continue iff E[continue] > current roll. With cost \$1/extra roll, let V = EV. V = max over keeping; threshold: keep r if r ≥ E[next roll value − cost]. Solve: you re-roll when current < V, paying 1. V = (sum of kept faces)/6 + (P(reroll))·(V − 1). Try keep 4,5,6: V = 15/6 + (3/6)(V−1) ⇒ V = 2.5 + 0.5V − 0.5 ⇒ 0.5V = 2 ⇒ V = **4.** Check threshold: keep r ≥ 4 → keep 4,5,6 ✓ (since continuing is worth 4, minus the \$1 cost = 3, and 4 > 3... refine: keep r if r ≥ V−1 = 3, so keep 4,5,6, possibly 3). Re-test keep 3,4,5,6: V = 18/6 + (2/6)(V−1) ⇒ V = 3 + (V−1)/3 ⇒ 3V = 9 + V −1 ⇒ 2V = 8 ⇒ V = 4, keep r ≥ 3 ✓ consistent. **EV = \$4, keep 3,4,5,6** (boundary at 3). *(Accept either clean derivation; key point is you set up the cost-adjusted threshold and got ~\$4.)*
7. Wait 15 of 60 min: P(meet) = 1 − (45/60)² = 1 − (3/4)² = 1 − 9/16 = **7/16 ≈ 0.4375.**
8. **1** (E[fixed points] = n·(1/n) = 1 for any n).
9. P(exactly 3) = C(5,3)/2⁵ = 10/32 = **5/16.** Expected heads = **2.5.**
10. Count: 7 appears in units place 10 times (7,17,...,97) + tens place 10 times (70–79) = **20.** (No double-count except 77 contributes one to each = it's counted twice correctly, total 20 digit-occurrences.) FV = 20; quote **18 @ 22** (you can compute it, so tight-ish).
11. First flip is "free"; then wait for the *other* face, geometric with p=½ → 2 more. E = 1 + 2 = **3.**
12. P(reach 7) = k/N = 3/7 ≈ **0.4286.** Expected flips = k(N−k) = 3·4 = **12.**
13. Each of the 3 pieces has expected length **1/3** by symmetry (middle piece included). So **1/3.**
14. Re-roll on 1,2,3; keep 4,5,6 → EV = (4+5+6)/6 + (3/6)·3.5 = 2.5 + 1.75 = **4.25.**
15. Classroom ≈ 8m×8m×3m ≈ 192 m³. Basketball diameter ≈ 0.24m → volume ≈ 0.0072 m³; with ~65% packing, effective ~0.011 m³. 192/0.011 ≈ **~17,000.** State packing assumption; market **~12k @ 22k** (genuine unknown, wide).

</details>
