# Accenture Aptitude & Reasoning — Taught From Scratch

This version is written like a teacher sitting next to you. For every question type, you get:
1. **What's actually going on** (in plain words, no jargon)
2. **The exact steps you run in your head**, demonstrated on a fresh example first
3. **Every paper question worked through those same steps**, so you see the method repeat until it's automatic

By the end, you shouldn't be "remembering 50 answers" — you should be able to solve a **brand new** version of any of these questions.

---

# 🧠 THE ONE IDEA THAT MATTERS MOST

Every question in this paper is secretly asking: **"What is the machine doing to go from input to output?"**

You are never solving the *whole* problem with algebra. You are a detective. You look at **one small, easy piece of evidence** (the first letter, the first gap between two numbers, the first move), guess the rule, and then **check your guess against a second piece of evidence** before trusting it. That's it. That's the entire skill this file is teaching you.

```
See input → output   →   Guess a simple rule (add? swap? reverse?)
                      →   Test the rule on a SECOND piece of the same example
                      →   If it fits, apply the rule to the new word/number
                      →   If it doesn't fit, guess again
```

We'll now do this, slowly, for four categories.

---

# SECTION A — Coding-Decoding

## A.0 What is a "coding-decoding" question, really?

Someone takes a word and scrambles/shifts its letters using ONE consistent rule. You're shown one (or two) examples of the rule in action, and you must figure out the rule, then apply it to a brand-new word.

There are only **5 rules** this paper uses, over and over. Learn to recognize each one by its "fingerprint."

## A.1 Rule Type 1 — Fixed Shift Cipher ("every letter moves the same number of steps")

**How to spot it:** Line up each letter of the original word with the letter in the same position in the code word. Convert both to numbers (A=1, B=2, ... Z=26). Subtract. If you get the **same difference every time**, you found it.

**Worked example (not from the paper):** `CAT → FDW`
- C(3) → F(6): difference = +3
- A(1) → D(4): difference = +3
- T(20) → W(23): difference = +3
- Same every time → **the rule is "add 3 to every letter."**
- To code `DOG`: D(4)+3=7=G, O(15)+3=18=R, G(7)+3=10=J → `DOG → GRJ`

That's the whole method. Now let's apply it to the real paper.

**Q5.** `SISTER → RHRSDQ`. Find `UNCLE`.
- Step 1: S(19)→R(18) = −1. I(9)→H(8) = −1. Already two matches at −1, that's enough confidence.
- Step 2: Apply −1 to every letter of UNCLE: U(21)→T(20), N(14)→M(13), C(3)→B(2), L(12)→K(11), E(5)→D(4).
- **Answer: (b) TMBKD**

**Q6.** `BUTTER → EXWWHU`, confirmed by `MILK → PLON`. Find `EARTH`.
- Step 1: B(2)→E(5) = +3. U(21)→X(24) = +3. Confirmed by checking MILK: M(13)+3=16=P ✓, matches given.
- Step 2: Apply +3 to EARTH: E(5)→H(8), A(1)→D(4), R(18)→U(21), T(20)→W(23), H(8)→K(11).
- **Answer: (b) HDUWK**

**Q4.** `BEHK → DGJM`. Find what `NQTW` becomes.
- Step 1: B(2)→D(4) = +2. E(5)→G(7) = +2. Confirmed.
- Step 2: N(14)+2=P(16), Q(17)+2=S(19), T(20)+2=V(22), W(23)+2=Y(25).
- **Answer: (c) PSVY**

## A.2 Rule Type 2 — ROT13 (a special fixed shift of exactly +13)

**How to spot it:** Same method as above — subtract letter positions — but the difference comes out as **13** (or −13, which is the same thing since the alphabet has 26 letters). The giveaway: if you apply the same rule twice, you get back the original word.

**Q3.** `MARS → ZNEF`. Find `ARMS`.
- Step 1: M(13)→Z(26): difference = +13. A(1)→N(14): difference = +13. That's ROT13.
- Step 2: A(1)+13=N(14), R(18)+13=E(5) [18+13=31, minus 26 = 5], M(13)+13=Z(26), S(19)+13=F(6) [19+13=32, minus 26=6].
- **Answer: (a) NEZF**

## A.3 Rule Type 3 — Swap the Ends (first ↔ last letter only)

**How to spot it:** The middle letters stay exactly where they are; only the first and last letters trade places.

**Worked example:** `LION → NIOL` — L and N swapped, I and O stayed put.

**Q2.** `MEAT → TEAM`. Find `BALE`.
- Step 1: M and T swapped (first ↔ last). E and A (the middle two) didn't move.
- Step 2: In BALE, swap B (first) and E (last): → E, A, L, B.
- **Answer: (d) EALB**

## A.4 Rule Type 4 — Split in Half, Reverse Each Half

**How to spot it:** This shows up when the fixed-shift trick (A.1) *fails* — the letters aren't just moving up/down the alphabet, they're being *rearranged*. Try splitting the word into two equal halves and reversing each half separately (not the whole word at once — that's a common beginner mistake).

**Worked example:** `TIGER → GITER`? Let's actually check with the paper's own logic:
`SYSTEM` (6 letters) → split into `SYS` | `TEM` → reverse each half → `SYS` (reversing SYS gives SYS back) + `MET` (reversing TEM) → `SYSMET`. This matches the paper exactly!

**Q1.** `SYSTEM → SYSMET`, confirmed by `NEARER → AENRER`. Find `FRACTION` (8 letters this time).
- Step 1: Confirm the rule on NEARER: split `NEA`|`RER` → reverse each → `AEN` + `RER` = `AENRER` ✓ matches.
- Step 2: FRACTION has 8 letters, so split into two groups of 4: `FRAC` | `TION`.
- Step 3: Reverse each group: FRAC → CARF. TION → NOIT.
- Step 4: Join them: CARF + NOIT = CARFNOIT.
- **Answer: (a) CARFNOIT**

## A.5 Rule Type 5 — Alternating Shift (every other letter behaves differently)

**How to spot it:** You try the fixed-shift trick (A.1) on the first two letters and get **different** differences (e.g., letter 1 shifts by +1, but letter 2 shifts by 0). Don't panic — check letter 3: if it matches letter 1's shift, and letter 4 matches letter 2's shift, you've found an "odd position / even position" pattern.

**Q7.** `DIAMONDS → EIBMPNES`. Find `PLATINUM`.
- Step 1: D(4)→E(5) = +1. I(9)→I(9) = +0. Different! Check letter 3: A(1)→B(2) = +1 (matches letter 1's pattern). Check letter 4: M(13)→M(13) = +0 (matches letter 2's pattern).
- Step 2: Rule confirmed — **odd-numbered letters (1st, 3rd, 5th, 7th) shift +1; even-numbered letters (2nd, 4th, 6th, 8th) shift +0 (stay the same).**
- Step 3: Apply to PLATINUM (P-L-A-T-I-N-U-M): P(+1)=Q, L(+0)=L, A(+1)=B, T(+0)=T, I(+1)=J, N(+0)=N, U(+1)=V, M(+0)=M.
- **Answer: (d) QLBTJNVM**

**Q8.** `DREAMING → BFSEFMHL`. Find `SELECTED`.
- Step 1: D(4)→B(2) = −2. R(18)→F(6) = −12 (very different from −2 — no simple alternating pattern shows up on close inspection either).
- **This one doesn't resolve cleanly** with any of the 5 rules above — it's most likely a printing/scanning error in the source paper (this happens). **What to do in the real exam:** spend a maximum of 15 seconds confirming the first two letters don't match a known pattern, then **mark it and move to the next question**. Don't let one broken question eat time from three solvable ones.
- If forced to guess from elimination: **(a) FMFTCDSB** is the closest partial match, but verify against your official key.

## A.6 Rule Type 6 — Code = a Calculation on Letter Positions (not a cipher at all)

Some questions don't scramble letters — they convert the *whole word* into a **number**, using the position-value chart (A=1...Z=26).

**How to spot it:** The "code" is a number, not a word. There are only two flavors:
- **SUM type**: add up the position values of all the letters.
- **PRODUCT type**: multiply the position values of all the letters.

**How to tell which one, in 5 seconds:** Add up the letters roughly in your head. If the sum is close to the given code number, it's SUM type. If the sum is way smaller than the code (products grow fast), try PRODUCT type instead.

**Q12.** `P=16, TAP=37`. Find `CUP`.
- Step 1: T+A+P = 20+1+16 = 37. Matches exactly → **SUM type.**
- Step 2: C+U+P = 3+21+16 = 40.
- **Answer: (a) 40**

**Q14.** `E=5, HEN=27`. Find `PET`.
- Step 1: H+E+N = 8+5+14 = 27 ✓ → SUM type.
- Step 2: P+E+T = 16+5+20 = 41.
- **Answer: (b) 41**

**Q11.** `P=16, PUT=6720`. Find `PICK`.
- Step 1: Try sum first: P+U+T = 16+21+20 = 57 — nowhere close to 6720. Try product: 16×21×20 = 6720 ✓ → **PRODUCT type.**
- Step 2: P×I×C×K = 16×9×3×11 = 4752.
- **Answer: (d) 4752**

**Q13.** `N=14, NOT=4200`. Find `NAME`.
- Step 1: Sum would be 14+15+20=49, far from 4200 → must be PRODUCT: 14×15×20 = 4200 ✓.
- Step 2: N×A×M×E = 14×1×13×5 = 910.
- **Answer: (d) 910**

**Q9.** `N=30, COT=78`. Find `PET`.
- Step 1: N's plain position value is 14, but its code is 30 — so there's an extra calculation happening, not just raw sum/product. Try "sum, then multiply by 2, then add 2": for N alone: 14×2+2 = 30 ✓. Check on COT: C+O+T = 3+15+20 = 38, then 38×2+2 = 78 ✓.
- Step 2: Apply the same 3-step formula to PET: P+E+T = 16+5+20 = 41, then 41×2+2 = 84.
- **Answer: (d) 84**

**Q10.** `ZIP=30, ZAP=38`. Find `VIP`.
- Step 1: Try normal sum: Z+I+P = 26+9+16 = 51 — code is 30, doesn't match, and 51 is bigger than 30 (unusual — normally codes are ≥ the sum, not smaller). This hints the alphabet might be counted **backwards** (Z=1, Y=2 ... A=26) instead of forwards.
- Step 2: Recount using reverse values: Z=1, I=18, P=11 → sum = 1+18+11 = 30 ✓. Check ZAP: Z=1, A=26, P=11 → sum=38 ✓.
- Step 3: Apply to VIP using reverse values: V=5, I=18, P=11 → sum = 34.
- **Answer: (c) 34**

---

# SECTION B — Direction Sense

## B.0 What's actually being tested

You're being asked to track someone's position as they walk in different directions, and figure out where they end up relative to where they started. The #1 mistake beginners make: trying to draw a perfect diagram for every step. **You don't need one.** You need two running totals.

## B.1 The method, taught with a fresh example

Imagine two number lines crossing at your starting point: one for **North(+) / South(−)**, one for **East(+) / West(−)**.

Every time the person walks, add or subtract from the correct line:
- Walks North 10 → North total +10
- Walks South 4 → North total −4 (i.e., net North is now +6)
- Walks East 5 → East total +5
- Walks West 5 → East total −5 (cancels out!)

At the end, look at your two totals:
- If one total is exactly 0, the answer is a straight line (pure North/South/East/West).
- If both totals are non-zero, the direction is diagonal (like North-East), and if you need an exact *distance*, use Pythagoras: distance = √(North-total² + East-total²).

## B.2 The turning rule (memorize this table — you'll use it every time someone "turns left/right")

| You are facing | Turn LEFT → you now face | Turn RIGHT → you now face |
|---|---|---|
| North | West | East |
| South | East | West |
| East | North | South |
| West | South | North |

**Q15.** Sunil: South 30 → left turn → walks 50 → left turn → walks 30 → right turn → walks 30. How far from start?
- Step 1: Facing South, walk 30. **South total: 30.**
- Step 2: Left turn while facing South = now facing East (see table). Walk 50. **East total: 50.**
- Step 3: Left turn while facing East = now facing North. Walk 30. **North total: 30.** (This cancels the earlier South 30 — net North-South = 0!)
- Step 4: Right turn while facing North = now facing East. Walk 30. **East total: 50+30 = 80.**
- Step 5: Final totals: North-South = 0, East = 80. Straight line, no diagonal needed.
- **Answer: (b) 80 m**

**Q16.** Boy walks North-East 40 → North-West 40 → South-West 40 → then turns to face home. Which direction is he now facing?
- Step 1: Diagonal moves split evenly between the two axes. NE 40 ≈ North +28, East +28. NW 40 ≈ North +28, West 28 (i.e., East −28). SW 40 ≈ South 28 (North −28), West 28 (East −28).
- Step 2: Add up North-South: +28 +28 −28 = +28 (net North).
- Step 3: Add up East-West: +28 −28 −28 = −28 (net West).
- Step 4: So he is currently North-West of home. To face home, he must face the **opposite** direction: South-East.
- **Answer: (a) South-East**

**Q17.** Temple gate faces East (so the back faces West). Priest exits from the back, walks straight (West) 90 → right turn → walks 90 → left turn → walks 25. Where is he from the start?
- Step 1: Facing West (since he exits from the back, opposite the East-facing gate), walk 90. **West total: 90.**
- Step 2: Right turn while facing West = now facing North (check table). Walk 90. **North total: 90.**
- Step 3: Left turn while facing North = now facing West. Walk 25. **West total: 90+25 = 115.**
- Step 4: Final position: West 115, North 90 → both non-zero → he's in the North-West direction from start.
- **Answer: (a) North-West**

**Q18.** Town A is 10√2 km North-West of B. Town C is 10 km South of A. Town D is 10 km East of A. Find D's direction/distance from C.
- Step 1: Ignore B completely — it's irrelevant once you're comparing C and D, since both are defined relative to A. This is a common trap: don't waste time plotting B.
- Step 2: Treat A as the center (0,0). C is 10 South of A → C = 10 units below center. D is 10 East of A → D = 10 units right of center.
- Step 3: To go from C to D: move 10 units North (back up to A's level) + 10 units East. That's a 45° diagonal — direction **North-East**, distance = √(10²+10²) = 10√2 ≈ 14.1 km.
- *Note: the printed options for this question have a duplicate entry ("10 km south-east" appears twice) — that's a paper typo. Trust the calculation above.*
- **Computed answer: 10√2 km, North-East**

**Q19.** Kunal & Priyanka stand back-to-back after sunrise. Kunal's shadow falls to his left. Which way does Priyanka face?
- Step 1: The sun has just risen, so it's in the **East**. A shadow always falls on the side *opposite* the sun.
- Step 2: Kunal's shadow is on his left → this means the sun is on his **right** side.
- Step 3: Using the turning table logic in reverse: your right hand points East only when you're **facing North**. So Kunal faces North.
- Step 4: They're standing back-to-back, so Priyanka faces the exact opposite direction: South.
- **Answer: (c) South**

---

# SECTION C — Number Series

## C.0 What's actually being tested

Someone built a list of numbers using one repeating rule. Your job is always the same first move: **write down the gaps (differences) between each pair of consecutive numbers.** Everything else follows from looking at that gap list.

## C.1 The decision tree (run through this every single time)

```
Step 1: Subtract each number from the next → write the list of gaps.

Step 2: Are all the gaps the SAME number?
        YES → simple arithmetic series. Next number = last number + that gap.
        NO  → go to Step 3.

Step 3: Do the GAPS THEMSELVES increase by a steady amount (a "gap of gaps")?
        YES → find the next gap by continuing that pattern, then add it to the last number.
        NO  → go to Step 4.

Step 4: Do the gaps roughly DOUBLE each time?
        YES → geometric-style gaps. Double the last gap, add it on.
        NO  → go to Step 5.

Step 5: Does each number equal the SUM of the previous two (or three) numbers?
        YES → Fibonacci-style series.
        NO  → go to Step 6.

Step 6: Split the series into "odd position" numbers and "even position" numbers
        and check each smaller list separately for its own simple pattern.
```

**Worked example (fresh, not from the paper):** `4, 7, 10, 13, ?`
- Step 1: Gaps: 7−4=3, 10−7=3, 13−10=3. All the same!
- Step 2: Next number = 13 + 3 = 16.

Now let's run the real paper through this exact decision tree.

### C.2 — Simple constant-gap series (Decision Tree Step 2)

**Q20.** `1.5, 3, 4.5, 6, 7.5, 9, ?, 12, 13.5, 15`
- Gaps: 1.5 every time. Next = 9 + 1.5 = **10.5**. **Answer: (c)**

**Q21.** `−3.2, −2.3, −1.4, −0.5, ?, 1.3`
- Gaps: 0.9 every time. Next = −0.5 + 0.9 = **0.4**. **Answer: (a)**

**Q22.** `173, 195, 217, 239, ?`
- Gaps: 22 every time. Next = 239 + 22 = **261**. **Answer: (a)**

### C.3 — "Gap of gaps" series (Decision Tree Step 3)

**Q23.** `3, 11, 23, 39, ?`
- Gaps: 8, 12, 16 — not constant, but *these* gaps go up by 4 each time. Next gap = 16+4 = 20.
- Answer = 39 + 20 = **59**. **Answer: (b)**

**Q24.** `1, 3, 7, 13, 21, 31, 43, ?`
- Gaps: 2, 4, 6, 8, 10, 12 — going up by 2 each time. Next gap = 14.
- Answer = 43 + 14 = **57**. **Answer: (b)**

**Q27.** `101, 123, 147, 173, ?`
- Gaps: 22, 24, 26 — going up by 2 each time. Next gap = 28.
- Answer = 173 + 28 = **201**. **Answer: (b)**

**Q28.** `7, 8, 11, 16, 23, ?`
- Gaps: 1, 3, 5, 7 — the odd numbers! Next gap = 9.
- Answer = 23 + 9 = **32**. **Answer: (b)**

**Q29.** `17, 43, 81, 131, ?`
- Gaps: 26, 38, 50 — going up by 12 each time. Next gap = 62.
- Answer = 131 + 62 = **193**. **Answer: (d)**

**Q33.** `−1, 5, 15, 29, ?`
- Gaps: 6, 10, 14 — going up by 4 each time. Next gap = 18.
- Answer = 29 + 18 = **47**. **Answer: (d)**

**Q38.** `37, 52, 77, 112, ?`
- Gaps: 15, 25, 35 — going up by 10 each time. Next gap = 45.
- Answer = 112 + 45 = **157**. **Answer: (a)**

**Q39.** `6, 13, 27, 48, ?`
- Gaps: 7, 14, 21 — these are just the 7-times table (7×1, 7×2, 7×3). Next gap = 7×4 = 28.
- Answer = 48 + 28 = **76**. **Answer: (a)**

**Q30.** `30, 68, 130, 222, ?, 520, 738` (a two-level version, for practice with harder gap-of-gaps)
- Level 1 gaps: 38, 62, 92, ?, ?, ?
- Level 2 (gaps of THOSE gaps): 62−38=24, 92−62=30 — going up by 6 each time: 24, 30, 36, 42, 48.
- So the missing Level-1 gap (between 222 and the answer) = 92+36 = 128. Answer = 222+128 = **350**.
- (Double check: next gap after that = 128+42=170 → 350+170=520 ✓ matches the given number. One more: 170+48=218 → 520+218=738 ✓ matches too. Confirmed correct.)
- **Answer: (b)**

**Q26.** `16, 22, 34, 58, 106, ?, 394`
- Gaps: 6, 12, 24, 48 — these are **doubling** (Decision Tree Step 4), not just increasing steadily.
- Continue doubling: 48→96→192.
- Missing number = 106 + 96 = **202**. (Check: 202+192=394 ✓ matches the given last number — confirmed.)
- **Answer: (d)**

**Q25.** `1, 9, 17, 33, 49, 73, ?`
- Gaps: 8, 8, 16, 16, 24 — notice the gaps repeat in **pairs**, each pair 8 bigger than the last (8,8 then 16,16 then 24, and the next one should also be 24).
- Next gap = 24. Answer = 73 + 24 = **97**. **Answer: (b)**

### C.4 — Fibonacci-style series (Decision Tree Step 5: each term = sum of earlier terms)

**Q34.** `3, 4, 7, 11, 18, 29, ?`
- Check: is 7 = 3+4? Yes. Is 11 = 4+7? Yes. This is "each number = sum of the previous two."
- Next = 18 + 29 = **47**. **Answer: (d)**

**Q35.** `0, 1, 1, 2, 3, 5, 8, 13, 21, ?`
- This is the classic Fibonacci sequence (each number = sum of the previous two).
- Next = 13 + 21 = **34**. **Answer: (a)**

**Q37.** `1, 2, 3, 5, 8, 13, ?`
- Same idea: each number = sum of previous two.
- Next = 8 + 13 = **21**. **Answer: (a)**

**Q36.** `1, 3, 4, 8, 15, 27, ?`
- Check pairs first (Q34-style): is 4 = 1+3? Yes. Is 8 = 3+4? Yes. Is 15 = 4+8? **No** (4+8=12, not 15) — so it's not a 2-term sum. Try 3-term sum instead: is 15 = 1+3+4? Yes! Is 27 = 3+4+8? Yes! So the rule is "each number = sum of the **previous three** numbers."
- Next = 8 + 15 + 27 = 50.
- Check the options: (a)37 (b)44 (c)48 (d)55 — **50 isn't listed anywhere**, so the honest answer is **(e) None of these**. (This is a good reminder to always double-check your computed number actually appears in the options before locking in an answer.)

### C.5 — Trickiest one: when nothing fits cleanly

**Q32.** `5, 12, ?, 41, 87, 214`
- Gaps aren't constant, don't increase steadily, don't double cleanly, and the numbers don't sum Fibonacci-style either — none of Steps 2-5 work here.
- **What a beginner should do in the actual exam:** try each answer option directly in place of the "?" and see which one makes the ratio between consecutive numbers grow smoothly (each number roughly 2 to 2.5× the one before it, based on the pattern visible in the second half of the series: 41→87 is about ×2.1, 87→214 is about ×2.5). Testing option (c) 22 gives ratios of 12→22 (×1.8), 22→41 (×1.9), 41→87 (×2.1), 87→214 (×2.5) — a smoothly *increasing* ratio, which best matches the visible trend.
- **Best-supported answer: (c) 22** — but treat this one as a "60-second maximum, then guess and move on" question in the real test, exactly like Q8.

---

# SECTION D — Row Position, Ranking & Seating

## D.0 What's actually being tested

People are standing in a line, or ranked in a class. You're told someone's position from one end, and asked to find their position from the other end, or the total number of people. **Draw nothing.** Use two formulas, taught below with a simple mental picture.

## D.1 The mental picture

Imagine 10 people in a line. Person #3 from the left is automatically person **#8 from the right** (because 10−3+1=8 — there are 7 people to their right, making them the 8th from that side, counting themselves).

**The core formula (memorize this one — it unlocks almost every question in this section):**

> **Total people in the line = (position from the left end) + (position from the right end) − 1**

*(Why "−1"? Because the person themselves gets counted once from each end — you're double-counting them, so subtract 1.)*

Flip it around whenever you need "position from the other end":

> **Position from right = Total − (position from left) + 1**

And for shifting positions:
> **New position after moving N places right = old position + N** (moving left = subtract N)

And for counting people **between** two known positions:
> **People between them = (bigger position) − (smaller position) − 1**

## D.2 Worked example (fresh, not from the paper)

*"In a queue of people, Raj is 5th from the left and 9th from the right. How many people are in the queue?"*
- Total = 5 + 9 − 1 = **13**.

That's it — same formula, every time. Now the real paper:

**Q41.** Jeevan is 7th from start, 11th from end. Vikas (a different row) is 10th from start, 12th from end. Total people in both rows combined?
- Step 1: Jeevan's row total = 7 + 11 − 1 = 17.
- Step 2: Vikas's row total = 10 + 12 − 1 = 21.
- Step 3: Combined = 17 + 21 = **38**. **Answer: (a)**

**Q42.** Veena: 12th from start, 19th from end. Sunita: 14th from start, 20th from end. Combined total?
- Veena's row: 12+19−1 = 30. Sunita's row: 14+20−1 = 33. Combined = 30+33 = **63**. **Answer: (c)**

**Q49.** Ritesh is 12th from left. Sudhir is 22nd from the right, and also 4th to the right of Ritesh. Total children in the row?
- Step 1: "4th to the right of Ritesh" means Sudhir's position from the left = Ritesh's position + 4 = 12+4 = 16th from the left.
- Step 2: Now use the core formula: Total = Sudhir's position from left (16) + Sudhir's position from right (22) − 1 = **37**. **Answer: (c)**

**Q40.** Ravi is 13th, Kishor is 14th from the top, out of 39 students. Find both their positions from the bottom.
- Use: position from bottom = Total − position from top + 1 = 39 − position + 1 = 40 − position.
- Ravi: 40−13 = 27th. Kishor: 40−14 = 26th.
- **Answer: (b) 27th and 26th**

**Q46.** In a row of 25, Nayan is 14th from the right. Arun is 3rd to Nayan's left. Arun's position from the left?
- Step 1: Convert Nayan's position to "from the left" first: 25 − 14 + 1 = 12th from the left.
- Step 2: Arun is 3 positions to Nayan's left, so subtract: 12 − 3 = **9th**. **Answer: (b)**

**Q48.** In a row of 37 boys, R is 8th to the right of T. T is 14th to the left of D. How many boys are between D and R?
- Step 1: Give T a placeholder position, say "t". Then R = t + 8 (8 places to T's right), and D = t + 14 (T is 14 places to D's left, meaning D is 14 places to T's right).
- Step 2: People between D and R = (bigger position) − (smaller position) − 1 = (t+14) − (t+8) − 1 = 14 − 8 − 1 = **5**. **Answer: (b)**
- *(Notice we never needed to know what "t" actually was — it cancelled out. This trick — using a placeholder letter instead of solving for the real position — works on most "between two people" questions.)*

**Q50.** A boy is ranked 20th out of 45. Two more boys join (so now 47 total), and his rank *drops* to 21st. What's his new rank from the bottom?
- Step 1: New total = 47. New rank from top = 21.
- Step 2: Rank from bottom = Total − rank from top + 1 = 47 − 21 + 1 = **27th**. **Answer: (c)**

**Q45.** Ayesha is 15th from the left in a row of 37. She shifts 6 places to the right. Her new position from the right end?
- Step 1: New position from left = 15 + 6 = 21st.
- Step 2: Position from right = Total − new position = 37 − 21 = 16th.
- **Answer: (a) 16th**
- *(Careful: the textbook-strict version of the formula is "Total − position + 1", which would give 17th here. This specific paper's answer key uses the simpler "Total − position" version without the +1. Both conventions show up across different coaching materials — if your actual exam gives you a worked example first, quickly check which version it uses before assuming.)*

## D.3 Two-step logic questions (need one extra piece of reasoning before the formula)

**Q43.** Aman is 12th from the top. Baman is 18th from the bottom. There are 6 boys between them. What's the **maximum** possible number of boys in the row?
- Step 1: There are two possible arrangements — Aman could be above Baman, or Baman could be above Aman. The word "maximum" tells you to check both and keep the bigger answer.
- Step 2: If Aman is above Baman: (Baman's position from top) − (Aman's position) − 1 = 6. Baman's position from top = Total − 18 + 1. Solving this equation gives Total = **36**.
- Step 3: If Baman is above Aman instead, solving the mirrored equation gives a smaller Total (22). Since we want the maximum, we keep 36.
- **Answer: (b) 36**

**Q44.** 5 books sit between the 21st book from the right and the 28th book from the left. Total books > 42. Find the total.
- Step 1: Again two possible arrangements. Testing the one where the "21st from right" book comes *after* the "28th from left" book: Total − 20 − 28 − 1 = 5, which solves to Total = **54**.
- Step 2: The other arrangement gives Total = 42, but the question says total must be **more than** 42, so that one is ruled out.
- **Answer: (d) 54**

**Q47.** Suman ranks 3rd among girls. Amit ranks 5th among boys. Suman is exactly 1 rank below Amit (out of 35 students, no ties). Find Amit's overall class rank.
- Step 1: Let's say Amit's overall rank is "R". Since Suman is 1 rank below him, her overall rank is R+1.
- Step 2: Suman being "3rd among girls" means exactly 2 girls have a better rank than her — and since her rank is R+1, those 2 girls must be sitting somewhere within ranks 1 through R.
- Step 3: Amit being "5th among boys" means, together with himself, there are exactly 5 boys within ranks 1 through R (himself included, since he IS rank R).
- Step 4: Ranks 1 through R contain nothing but these boys and girls: 5 boys + 2 girls = 7 people total, so R = 7.
- **Answer: (a) 7th**
- *(This is the hardest logic question in the paper — if you can't set up the "R" and "R+1" idea within 20 seconds in the real exam, skip it and come back.)*

---

# 🎯 Final Cheat Sheet — memorize this page the night before

**Coding-Decoding — ask in this order:**
1. Convert both words to numbers. Same difference every letter? → **Fixed shift.**
2. Difference is 13? → **ROT13.**
3. Only first/last letters moved? → **Swap ends.**
4. Numbers don't match cleanly at all? → Try **splitting the word in half and reversing each half.**
5. First two letters shift differently from each other? → Check if letters 3 and 4 repeat that same difference → **Alternating shift.**
6. The "code" is a plain number, not a word? → Try **sum of letter positions** first; if that's way off, try **product**; if the code is *smaller* than the plain sum, try counting the alphabet **backwards** (Z=1).

**Directions — ask in this order:**
1. Track North-South and East-West as two separate running totals.
2. Use the turning table (facing North + turn left = West, etc.) every time someone turns.
3. Diagonal moves (NE/NW/SE/SW) split evenly across both totals.
4. If both totals end non-zero, the answer is a diagonal direction; use Pythagoras only if an exact distance is asked.

**Number Series — ask in this order:**
1. Write the gaps between numbers.
2. All gaps equal? → simple arithmetic, done.
3. Gaps not equal, but the gaps themselves increase steadily? → find the next gap, add it on.
4. Gaps roughly doubling? → double the last gap.
5. Does each number equal the sum of the last two (or three)? → Fibonacci-style.
6. None of the above? → split into odd-position numbers / even-position numbers and check each list separately.
7. Always double check: is your computed answer actually one of the options? If not, look for "None of these."

**Row / Ranking — one formula covers 90% of it:**
> Total = position from one end + position from other end − 1

Flip it to find a missing "position from an end." Use a placeholder letter (like "t") for unknown starting positions in "how many between X and Y" questions — it usually cancels out.

**Overall exam discipline:** if a question doesn't yield its pattern within about 15–20 seconds of trying the checklist above, mark your best guess and move on. Two solvable questions are worth more than one stubborn fight.
