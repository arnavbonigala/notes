<!-- Wed, Sep 30, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 15: Asymptotics II

This lecture completes the asymptotic notation toolkit (adding Big Omega to the Big Theta and Big O from Lecture 14) and then spends most of its time doing what the title of the textbook chapter promises: "There's no magic shortcut." After a review of the cost-model recipe for runtime analysis and a quick look at why the disjoint sets performance table uses O for some entries and Θ for others, the lecture works through three nested-loop examples of increasing difficulty: a quadratic duplicate counter, a deceptive double loop whose inner loop actually runs a constant number of times, and a doubling loop whose print count jumps at powers of two but is nonetheless Θ(N). The big takeaways are that nested loops do not automatically mean N², that you should translate code into a sum and then close that sum (you only need two sums for this course: 1 + 2 + ... + Q = Q(Q+1)/2 ∈ Θ(Q²) and 1 + 2 + 4 + ... + Q = 2Q - 1 ∈ Θ(Q) for Q a power of 2), and that drawing grids, tabulating small cases, and computing areas are legitimate and often the fastest proofs at the 61B level.

---

## Key Concepts

### The runtime-analysis recipe (review)

Given a code snippet, its runtime is a function R(N) where N is some property of the input (often its size). We almost never want R(N) exactly: Josh's framing in lecture was that finding an exact runtime as a function of the input is essentially impossible, since it depends on your laptop, the JVM, how long `println` takes, and so on. Instead we want the **order of growth**.

The approach from Lecture 14, restated here (and explicitly labeled "one approach, not universal"):

1. Choose a **representative operation** (a cost model), and let C(N) be the count of how many times that operation happens as a function of N.
2. Find a simple f(N) with C(N) ∈ Θ(f(N)). Often, but not always, we use the worst-case count.
3. If that operation takes constant time, then R(N) ∈ Θ(f(N)).

Applied to the warm-up:

```java
public static void printParty(int N) {
    for (int i = 10; i < N; i += 2) {
        System.out.println("hello");
    }
}
```

Choosing `println` as the cost model, the number of calls is (N - 10) / 2, so R(N) ∈ Θ(N).

Why drop the `-10`? Because it becomes insignificant as N grows. Why drop the `/ 2`? Because we are describing *how the runtime grows with N*, and we have already thrown away far bigger constant factors: the number of machine operations per loop iteration (`i < N`, `i += 2`, the call itself), how fast `println` is, how fast the machine is. Keeping a factor of 2 while ignoring all of those would be pointless precision.

The choice of cost model matters. In the lecture, `i = 10` was rejected as a candidate operation precisely because it happens exactly once no matter how large N is, so counting it tells you nothing. A good cost model is an operation whose count actually tracks the total work.

### Big Theta, Big O, Big Omega as families of functions

The single most useful mental model offered in lecture: each notation names a **family** (equivalently, a **set**) of functions, and membership says how your function's growth compares to the function in parentheses.

| Notation | Informal meaning | Family | Example members |
|---|---|---|---|
| Θ(f(N)) | Order of growth **is** f(N) | Θ(N²) | N²/2, 2N², N² + 38N + N |
| O(f(N)) | Order of growth is **≤** f(N) | O(N²) | N²/2, 2N², lg(N) |
| Ω(f(N)) | Order of growth is **≥** f(N) | Ω(N²) | N²/2, 2N², e^N |

Order-of-growth practice from the review slide:

| R(N) | Order of growth |
|---|---|
| N³ + 3N⁴ | Θ(N⁴) |
| 1/N + N³ | Θ(N³) |
| 1/N + 5 | Θ(1) |
| Ne^N + N | Θ(Ne^N) |
| 40 sin(N) + 4N² | Θ(N²) |

**Big O is "less than or equal to."** All of these are true simultaneously:

- N³ + 3N⁴ ∈ Θ(N⁴)
- N³ + 3N⁴ ∈ O(N⁴)
- N³ + 3N⁴ ∈ O(N⁶)
- N³ + 3N⁴ ∈ O(N!)
- N³ + 3N⁴ ∈ O(N^N!)

**Big Omega is "greater than or equal to."** All of these are true simultaneously:

- N³ + 3N⁴ ∈ Θ(N⁴)
- N³ + 3N⁴ ∈ Ω(N⁴)
- N³ + 3N⁴ ∈ Ω(N³)
- N³ + 3N⁴ ∈ Ω(log N)
- N³ + 3N⁴ ∈ Ω(1)

The formal definitions differ only in which bounds you require. Θ needs two constants k₁ and k₂ sandwiching R(N) between k₁·f(N) and k₂·f(N) for all N greater than some N₀. O needs only the upper constant; Ω needs only the lower constant. The "for large N" clause (N > N₀) is what licenses ignoring small-N weirdness.

The Ω visualization example from the slides: 40 sin(N) + 4N² ∈ Ω(N), with R(N) = 40 sin(N) + 4N², f(N) = N, and k₁ = 20. That is, 20N ≤ 40 sin(N) + 4N² once N is large enough.

A student asked why you would ever use O or Ω instead of Θ. The answer given: for the same reason you sometimes write ≤ instead of =. If you know the exact order of growth, say Θ. If the runtime is sometimes N and sometimes N³ depending on the input, you can honestly say it is O(N³) without claiming it is always N³.

Also noted explicitly: **best case and worst case are separate ideas from Θ/O/Ω.** The definitions of the notations say nothing about cases; you pick a case first (worst, best, whatever), get a function, and then describe *that function's* order of growth. Conflating "Big O" with "worst case" is a very common error (returned to on Friday's lecture).

### Why Big Omega exists: floors on problem difficulty

Big Omega is often used to set **theoretical floors on problem difficulty**, i.e. statements about *every possible algorithm*, not about one piece of code.

Example from lecture: finding the median of a list of N numbers takes Ω(N). Among the space of ALL possible algorithms for finding the median, the best you could ever do is linear time. The intuition offered by a student and endorsed: you have to actually look at every number. If you skip even one, that unexamined number could have been the median, so you might be wrong. Therefore no algorithm can do better than linear. This style of argument gets used in a much later lecture.

### Why the disjoint sets table mixes Θ and O

| Implementation | constructor | connect | isConnected |
|---|---|---|---|
| `ListOfSetsDS` | Θ(N) (make N sets) | O(N) (find each element, combine 2 sets) | O(N) (check up to N sets) |
| `QuickFindDS` (`id` array holds set ID for each item) | Θ(N) (initialize size-N array) | Θ(N) (look through entire array, update IDs) | Θ(1) (two array accesses) |
| `QuickUnionDS` (`parent` array holds each item's parent) | Θ(N) | O(N) (climb tree to find root) | O(N) (climb 2 trees to compare roots) |
| `WeightedQuickUnionDS` (track tree size, connect smaller tree under bigger) | Θ(N) | O(log N) | O(log N) |

The reason some entries are O rather than Θ: **those operations might be much faster than the bound, e.g. constant time.** For `QuickUnionDS.connect`, if the tree happens to be a long chain and you start at the bottom, climbing costs Θ(N); but if the two items you are connecting are already roots, you do no climbing at all and it is constant time. Since the runtime is not always proportional to a single f(N), only an upper bound is honest, so we write O.

By contrast, `QuickFindDS.connect` genuinely always sweeps the entire `id` array, so its count is proportional to N on every input: Θ(N) is correct and more informative.

Josh noted in passing that the runtime of weighted quick union **with path compression** is so deep that it could consume a whole lecture or two, that multiple distinct analyses of it were published historically, and that simply determining the runtime was considered a breakthrough. That is why no runtime for it appeared in the table.

### Nested loops are not automatically N²

This is the conceptual heart of the lecture. Students want a pattern-matching shortcut: "two for loops, therefore N²." Code is too expressive for that. In Example 2 below, the inner loop is written as a loop but runs exactly 3 times regardless of N, so it is really just "print three times" in disguise. In Example 3, the outer loop runs only about log N times while the inner loop's length changes every iteration. There is no multiplication rule you can apply blindly.

### The two sums you must know

> There is no magic shortcut for asymptotic analysis problems (well... usually).

Runtime analysis often requires careful thought. CS70 and especially CS170 cover this in much more detail. 61B is not a math class, but you are expected to know exactly two sums:

- **Sum of first natural numbers:** 1 + 2 + 3 + ... + Q = Q(Q+1)/2 ∈ Θ(Q²)
- **Sum of first powers of 2:** 1 + 2 + 4 + 8 + ... + Q = 2Q - 1 ∈ Θ(Q), where Q is a power of 2

In lecture Josh stressed that the whole game is: translate code into a sum, then close the sum, and the sum will essentially always be one of these two. The wrinkle is that Q itself may be a function of N. He gave the example: if the sum is 1 + 2 + 4 + 8 + ... + log N, just substitute Q = log N to get Θ(log N).

**Strategies:** find the exact sum; write out examples (tabulate small N by hand); draw pictures. On drawing pictures, he showed a QR-decomposition runtime figure from Trefethen's *Numerical Linear Algebra* to make the point that picture-based runtime justifications are standard practice in real research papers and textbooks, not a 61B crutch: it is visually clear from the diagram that the runtime there is proportional to N²·m.

### Proof that 1 + 2 + 4 + ... + N = 2N - 1

**Claim:** If N is a power of 2, then 1 + 2 + 4 + ... + N = 2N - 1.

Proof by induction, as presented:

- **Base case**, N = 1: the sum is just 1, and 2·1 - 1 = 1. ✓
- **Inductive hypothesis:** 1 + 2 + 4 + ... + N = 2N - 1.
- After N, the next power of two is 2N, so we want to show 1 + 2 + 4 + ... + N + 2N = 4N - 1.
- Start from the hypothesis and add 2N to both sides: 1 + 2 + 4 + ... + N + 2N = (2N - 1) + 2N = 4N - 1. ✓

A deliberate pedagogical note from the slides ("Note to Mathy Students"): the more natural mathematical phrasing would be "suppose N = 2^k for some k, then Σ..." but experience has shown that 61B students get k and N confused, so the course uses the N-and-2N framing instead. In lecture Josh also described a lighter-weight approach he called **"artisanal induction"**: plug in some values of Q and confirm the pattern holds. CS70 gives you the tools for a rigorous proof.

---

## Definitions

- **R(N):** the runtime of a piece of code expressed as a function of N, where N is some property of the input (often input size).
- **C(N):** the count of how many times a chosen representative operation executes, as a function of N.
- **Cost model:** a chosen representative operation whose count stands in for total work. A *good* cost model is one whose count grows proportionally to the real runtime; a bad one (e.g. counting `int s = 0;`, which happens once) tells you nothing about growth.
- **Order of growth:** the shape of a function's growth with low-order terms and multiplicative constants discarded, e.g. the order of growth of (N - 10)/2 is N.
- **Big Theta, R(N) ∈ Θ(f(N)):** there exist positive constants k₁ and k₂ such that k₁·f(N) ≤ R(N) ≤ k₂·f(N) for all N greater than some N₀ (i.e. for very large N). Informally: order of growth **is** f(N).
- **Big O, R(N) ∈ O(f(N)):** there exists a positive constant k₂ such that R(N) ≤ k₂·f(N) for all N greater than some N₀. An **upper bound only**. Informally: order of growth is **less than or equal to** f(N).
- **Big Omega, R(N) ∈ Ω(f(N)):** there exists a positive constant k₁ such that k₁·f(N) ≤ R(N) for all N greater than some N₀. A **lower bound only**. Informally: order of growth is **greater than or equal to** f(N).
- **Family (or set) of functions:** the collection of all functions satisfying a given Θ/O/Ω membership, e.g. Θ(N²) contains N²/2, 2N², and N² + 38N + N.
- **Theoretical floor on problem difficulty:** a Ω bound that applies to every possible algorithm for a problem, e.g. median-finding is Ω(N).
- **Sum of first natural numbers:** 1 + 2 + 3 + ... + Q = Q(Q+1)/2 ∈ Θ(Q²).
- **Sum of first powers of 2:** 1 + 2 + 4 + 8 + ... + Q = 2Q - 1 ∈ Θ(Q), for Q a power of 2.

---

## Worked Examples

### Example 1: Counting duplicates (Θ(N²))

```java
public static void countDuplicates(int[] a) {
    int duplicates = 0;
    for (int i = 0; i < a.length; i++) {
        for (int j = i + 1; j < a.length; j++) {
            if (a[i] == a[j]) {
                duplicates += 1;
            }
        }
    }
    IO.println("Duplicates: " + duplicates);
}
```

**What it does.** This is a variant of the duplicate-finder from Lecture 14, except instead of returning `true`/`false` as soon as a duplicate is found, it counts *all* duplicate pairs. The outer loop fixes an index `i`; the inner loop walks `j` from `i + 1` to the end, so each unordered pair of distinct positions is compared exactly once. Note there is no early return, so the work does not depend on the array contents at all: every input of length N costs the same.

**Cost model.** `==` (the comparison `a[i] == a[j]`). The count of `==` operations equals the number of pairs checked.

**Grid reasoning (the slide's diagram, in words).** Draw an N × N grid with `i` down the rows and `j` across the columns; mark a cell when the comparison `a[i] == a[j]` actually executes. For N = 6:

```
          j = 0   1    2    3    4    5
  i = 0          ==   ==   ==   ==   ==
  i = 1               ==   ==   ==   ==
  i = 2                    ==   ==   ==
  i = 3                         ==   ==
  i = 4                              ==
  i = 5
```

Row `i = 0` has no mark at `j = 0` because `j` starts at `i + 1 = 1`; it has marks at `j = 1..5`, so 5 marks. Row 1 has 4, row 2 has 3, row 3 has 2, row 4 has 1, row 5 has 0. The marked cells form a **right triangle** occupying (just under) half the grid.

**Two ways to finish.**

*Picture / area argument (the one 61B wants):* the marked region is a right triangle with legs of length about N, so its area is about N²/2. There are off-by-one errors but the order of growth is exactly N². The lecture was explicit that this is a perfectly acceptable proof for 61B purposes.

*Exact sum:* the count is

```
1 + 2 + 3 + ... + (N - 3) + (N - 2) + (N - 1)
```

Pair up terms from the outside in: 1 + (N - 1) = N, 2 + (N - 2) = N, 3 + (N - 3) = N, and so on. There are (N - 1)/2 such pairs, each summing to N, giving N(N - 1)/2. Order of growth: **N²**.

So C(N) ∈ Θ(N²), and since `==` on `int`s is constant time, R(N) ∈ **Θ(N²)**.

### Example 2: A loop that looks nested but isn't (Θ(N))

```java
public static void scan(int N) {
    int s = 0;
    for (int i = 2; i <= N; i += 1) {
        for (int j = i-1; j <= i+1; j += 1) {
            s += i*j;
        }
    }
}
```

The in-lecture poll on this one split 56% N, 26% N², 8% N log N, 5% log N, rest other. The majority was right, and the N² answers came from pattern-matching "two for loops."

**What it does.** The outer loop runs `i` from 2 up through N. For each `i`, the inner loop runs `j` from `i - 1` to `i + 1` inclusive. That is a window of width 3 that *slides along with* `i` rather than growing: `j` takes the three values `i-1`, `i`, `i+1`, always three values, no matter how large N is. `j` is, as the lecture put it, a diversionary maneuver. The body `s += i*j` accumulates into a local `s` that is never used, which is fine for runtime purposes.

**Cost model.** `s += i*j`.

**Tabulating small cases (the method Josh used live).**

| N | `s += i*j` calls | reasoning |
|---|---|---|
| 0 | 0 | `i = 2`, condition `2 <= 0` fails immediately |
| 1 | 0 | `2 <= 1` fails immediately |
| 2 | 3 | only `i = 2`, inner loop runs for `j = 1, 2, 3` |
| 3 | 3 + 3 = 6 | `i = 2` and `i = 3`, three each |
| 4 | 3 + 3 + 3 = 9 | `i = 2, 3, 4` |
| 5 | 12 | `i = 2, 3, 4, 5` |

**Grid reasoning.** On the (j, i) grid, the executed cells form a **diagonal band of width 3** running down the grid, one short diagonal segment per value of `i`, rather than a filled triangle or square. The band has width 3 and length N - 1 (the number of `i` values from 2 to N), so the area is 3(N - 1).

**Conclusion.** C(N) = 3(N - 1) ∈ Θ(N), so R(N) ∈ **Θ(N)**. As N grows by 1, the count grows by exactly 3: a straight line.

**Two warnings attached to this example in lecture.**

1. R(N) ∈ Θ(N) is only valid *because `s += i*j` was a good cost model.* Counting `i = 2` (the outer loop initialization) would have been a bad cost model: it executes once, so it would have suggested Θ(1).
2. Nested `for` loops do not imply any multiplication of bounds. This code is equivalent to writing `s += ...` three times inside a single N-length loop.

### Example 3: The doubling loop (Θ(N)), worked in three attempts

```java
public static void printParty(int N) {
    for (int i = 1; i <= N; i = i * 2) {
        for (int j = 0; j < i; j += 1) {
            System.out.println("hello");
            int ZUG = 1 + 1;
        }
    }
}
```

**What it does.** The outer variable `i` starts at 1 and **doubles** each iteration: 1, 2, 4, 8, 16, ..., stopping once it exceeds N. So `i` only ever takes power-of-2 values; the lecture emphasized that `i` can never be 3, 5, 6, or 7, because the only way `i` changes is by doubling. The inner loop prints `i` times. `int ZUG = 1 + 1;` is a constant-time distractor declared and discarded each iteration; it does not change the order of growth. Cost model: `println`.

**Attempt #1** (poll before any analysis) was asked cold and most students could not get it; Josh explicitly said that was fine and then modeled the process.

**Attempt #2: tabulate C(N) for small N, carefully.**

| N | `println` calls | reasoning |
|---|---|---|
| 0 | 0 | `i = 1`, but `1 <= 0` fails immediately |
| 1 | 1 | `i = 1`: inner loop runs for `j = 0` only |
| 2 | 1 + 2 = 3 | `i = 1` (1 print), `i = 2` (2 prints); next `i` would be 4 > 2 |
| 3 | 3 | `i` takes 1, 2; `i = 4` exceeds 3, so identical to N = 2 |
| 4 | 1 + 2 + 4 = 7 | `i` takes 1, 2, 4 |
| 5 | 7 | same `i` values as N = 4 |
| 6 | 7 | same |
| 7 | 7 | same |
| 8 | 1 + 2 + 4 + 8 = 15 | `i` takes 1, 2, 4, 8 |
| 9 | 15 | same as N = 8 |

Two things to notice. First, C(N) is a **staircase**: it is flat for stretches and jumps only when N crosses a power of 2. It is non-decreasing but, as the lecture said, "increasing, but not always." Second, on the grid, the executed cells occupy only the power-of-2 rows (`i = 1, 2, 4, 8, ...`), with row `i` filled from `j = 0` to `j = i - 1`; the rows for non-powers of 2 are entirely blank. Rows double in length each time they appear, so the filled region is dominated by its single longest row.

**Attempt #3: close the sum.** If N is a power of 2, then

```
C(N) = 1 + 2 + 4 + ... + N = 2N - 1
```

Check: N = 8 gives 1 + 2 + 4 + 8 = 15 = 2(8) - 1. ✓ N = 16 gives 31. ✓ N = 32 gives 63. ✓ The line 2N - 1 passes exactly through the **peaks** of the staircase.

**Why Θ(N) despite the staircase.** C(N) ∈ Θ(N) means we can find k₁ and k₂ with k₁·N ≤ C(N) ≤ k₂·N for large N. Taking k₁ = 0.5 and k₂ = 4 works. (So would k₁ = 0.001 and k₂ = 10³; the constants are completely arbitrary, and all you must do is exhibit *some* pair that works.)

The verification table from the slides:

| N | C(N) | 0.5N (lower) | 4N (upper) |
|---|---|---|---|
| 1 | 1 | 0.5 | 8 |
| 4 | 1 + 2 + 4 = 7 | 2 | 16 |
| 7 | 1 + 2 + 4 = 7 | 3.5 | 28 |
| 8 | 1 + 2 + 4 + 8 = 15 | 4 | 32 |
| 27 | 1 + 2 + 4 + 8 + 16 = 31 | 13.5 | 108 |
| 185 | ... + 64 + 128 = 255 | 92.5 | 740 |
| 715 | ... + 256 + 512 = 1023 | 357.5 | 2,860 |

Every C(N) lands between 0.5N and 4N. The flat stretches are what keep C(N) from exceeding 4N just after a jump, and the fact that a jump always doubles is what keeps C(N) from falling below 0.5N just before the next jump. So R(N) ∈ **Θ(N)**: the function makes larger and larger jumps, exponentially less often, and the net effect is linear.

**Why this example was chosen: it is the resizing argument.** The pattern "every time N is a power of 2 we do something expensive, and the total is Θ(N)" is exactly **array resizing**. The slides label it "ArrayList resizing"; in lecture Josh framed it in terms of `ArrayDeque`: every time you run out of space you resize, and if you *double* the capacity, the total cost of N additions is linear, i.e. **constant average (amortized) cost per operation**. One thousand adds to an `ArrayDeque` take time proportional to one thousand. Josh flagged one difference: in the real resizing case the count also grows linearly *between* the powers of 2, because every individual add does a constant amount of work, whereas `printParty`'s count is perfectly flat between jumps. The mathematics is otherwise very, very close.

*(extra context: "amortized" is the standard name for this constant-average-cost-per-operation claim; the lecture said "constant average transaction time" rather than naming the term.)*

### Example 4: Modifying `printParty` to change its order of growth

This was assigned at the end of lecture as a think-about-it problem to be discussed on Friday, but the slides include the answers. Starting from:

```java
public static void printParty(int N) {
    for (int i = 1; i <= N; i = i * 2) {
        for (int j = 0; j < i; j += 1) {
            System.out.println("hello");
            int ZUG = 1 + 1; }}}
```

| New order of growth | Modification |
|---|---|
| 1 | Replace inner loop with `return;` |
| log N | Replace inner loop with a constant-time operation (e.g. just `System.out.println("hello");`) |
| N | (none, this is the original) |
| N log N | Inner loop always runs N times (change `j < i` to `j < N`) |
| N² | `i` increments (change `i = i * 2` to `i = i + 1`) |

**Why each works.**

- **Θ(1):** `return;` in place of the inner loop exits the method during the first outer iteration, so a constant amount of work happens regardless of N.
- **Θ(log N):** the outer loop body now costs a constant, and the outer loop body executes once per power of 2 up to N, which is about log₂ N times. Constant × log N iterations = Θ(log N).
- **Θ(N):** the original. About log N outer iterations with inner lengths 1, 2, 4, ..., N summing to 2N - 1.
- **Θ(N log N):** with `j < N`, every one of the ~log N outer iterations costs exactly N, giving N·log N.
- **Θ(N²):** with `i = i + 1`, `i` runs 1, 2, 3, ..., N and the inner loop runs `i` times, so the count is 1 + 2 + 3 + ... + N = N(N+1)/2 ∈ Θ(N²). This is the sum-of-first-natural-numbers formula and the filled-triangle grid picture from Example 1.

Notice that the five answers are obtained by changing *only* the loop increment or the loop bound, which is the cleanest demonstration that you cannot read off a runtime from the nesting depth.

### A note on logarithm bases

A student asked what base the logarithm is in "N log N." Answer: it does not matter, because changing base only multiplies by a constant factor (log_a N = log_b N / log_b a), and Θ discards constant factors. All logarithms grow at the same rate in Big Theta notation.

---

## Common Pitfalls

- **Assuming two nested loops means N².** Example 2 (`scan`) is two nested loops and is Θ(N). Example 3 is two nested loops and is also Θ(N). The inner loop's *trip count* is what matters, and it may be constant, may depend on `i`, or may be independent of `i`.
- **Picking a bad cost model.** Counting an operation that executes a fixed number of times (`int s = 0;`, `i = 2`, the method call itself) gives you Θ(1) no matter what the code does. Pick an operation inside the innermost loop whose count tracks the real work.
- **Forgetting to check whether the cost model's operation is constant time.** Step 3 of the recipe ("if the operation takes constant time, then R(N) ∈ Θ(f(N))") has a hypothesis. If your representative operation is itself expensive and input-dependent, counting it is not enough.
- **Thinking a staircase or jumpy function cannot be Θ of anything.** `printParty`'s count is flat for long stretches and then jumps, yet it is squarely Θ(N) because it stays between 0.5N and 4N. Θ permits wobble by constant factors; it is not a claim of smoothness.
- **Confusing Big O with "worst case" and Big Omega with "best case."** These are orthogonal. The notations describe growth of a function; which case you analyzed determines *which* function. You can perfectly well say "the best-case runtime is Θ(1)" or "the worst-case runtime is O(N²)."
- **Thinking Big O statements are false if loose.** N³ + 3N⁴ ∈ O(N^N!) is a true statement, just uselessly weak. If you can say Θ, say Θ, because it conveys more.
- **Misremembering the powers-of-2 sum.** It is 1 + 2 + 4 + ... + Q = 2Q - 1 where Q is the *last term* (and a power of 2), not 2^Q or anything involving the exponent. Checking against N = 8 → 15 takes five seconds and catches the error.
- **Off-by-one anxiety.** The triangle in Example 1 has area N²/2 "with small errors per unit." Those errors are exactly what Θ throws away. Do not spend exam minutes deciding between N(N-1)/2 and N(N+1)/2 if you only need the order of growth.
- **Forgetting that `i = i * 2` skips values.** When tabulating, N = 3, 5, 6, 7 give the same counts as the nearest smaller power of 2. Several students in lecture tripped on this.
- **Hunting for a universal shortcut.** Explicitly warned against: "there is no magic shortcut for asymptotic analysis problems (well... usually)." Course staff have, in Josh's words, endless creativity for strange problems.

---

## Likely Exam Points

**1. Order of growth of a function given in closed form.**

*Q:* Give simple f(N) with 40 sin(N) + 4N² ∈ Θ(f(N)), and state whether 40 sin(N) + 4N² ∈ Ω(N) is true.

*A:* f(N) = N². The Ω(N) claim is **true**: Ω is a lower bound and N² grows at least as fast as N, so a valid witness is k₁ = 20 (20N ≤ 40 sin(N) + 4N² for large N), exactly the slide's example. The `40 sin(N)` term is bounded between -40 and 40 and so is a low-order term swamped by 4N².

**2. True/false on Θ vs O vs Ω membership.**

*Q:* Which of these are true for R(N) = N³ + 3N⁴? (a) O(N⁶) (b) Ω(N⁵) (c) Θ(N⁴) (d) Ω(1) (e) O(N³)

*A:* (a) true, N⁶ is an upper bound. (b) **false**, N⁵ grows faster than N⁴, so R(N) is not bounded below by k₁N⁵. (c) true. (d) true, everything that grows is Ω(1). (e) **false**, R(N) grows faster than N³.

**3. Nested loop where the inner loop runs a constant number of times.**

*Q:* Give simple f(N) with R(N) ∈ Θ(f(N)):

```java
for (int i = 0; i < N; i += 1) {
    for (int j = i; j < i + 5; j += 1) {
        System.out.println("hi");
    }
}
```

*A:* **Θ(N)**. The inner loop runs exactly 5 times for every `i` because its bounds both slide with `i`, so the count of prints is 5N. This is the `scan` pattern: a width-5 diagonal band of length N on the grid, area 5N, order of growth N.

**4. The triangular nested loop.**

*Q:* Find the exact count of `==` operations in `countDuplicates` on an array of length N, and give the order of growth.

*A:* 1 + 2 + ... + (N - 1) = N(N - 1)/2 comparisons, which is **Θ(N²)**. Equivalently: the executed (i, j) cells form a right triangle with legs about N, area about N²/2.

**5. The doubling loop.**

*Q:* For

```java
for (int i = 1; i <= N; i = i * 2) {
    for (int j = 0; j < i; j += 1) {
        System.out.println("hello");
    }
}
```

how many prints occur when N = 16, and what is the order of growth?

*A:* `i` takes 1, 2, 4, 8, 16, so the count is 1 + 2 + 4 + 8 + 16 = 31 = 2(16) - 1. Order of growth is **Θ(N)** (the peaks lie exactly on 2N - 1, and C(N) is sandwiched between 0.5N and 4N for large N).

**6. Modify code to hit a target order of growth.**

*Q:* Change one thing in the Example 3 `printParty` so its runtime becomes Θ(N log N), and one thing so it becomes Θ(N²).

*A:* For Θ(N log N), change the inner loop condition to `j < N` so each of the ~log N outer iterations costs N. For Θ(N²), change the outer update to `i = i + 1` so the counts become 1 + 2 + ... + N = N(N+1)/2.

**7. Why an entry in a performance table is O rather than Θ.**

*Q:* `QuickUnionDS.connect` is listed as O(N) rather than Θ(N). Why?

*A:* Because the actual cost depends on the input: climbing a long chain from the bottom costs Θ(N), but if both items are already roots, `connect` is constant time. Since no single f(N) is proportional to the cost on every input, only an upper bound is honest. By contrast `QuickFindDS.connect` always scans the whole `id` array, so Θ(N) is correct there.

**8. Big Omega as a floor on problem difficulty.**

*Q:* Explain why finding the median of N numbers is Ω(N).

*A:* Any correct algorithm must examine every element. If it skipped an element, that element could have been (or could have changed) the median, so the algorithm could be wrong. Hence every possible algorithm does at least N units of work: Ω(N). Note this is a statement about *all* algorithms, not about one implementation.

**9. The two required sums.**

*Q:* State both sums the course expects you to know, with their orders of growth.

*A:* 1 + 2 + 3 + ... + Q = Q(Q+1)/2 ∈ Θ(Q²), and 1 + 2 + 4 + 8 + ... + Q = 2Q - 1 ∈ Θ(Q) for Q a power of 2.

**10. A sum where Q is itself a function of N.**

*Q:* Suppose the number of operations in some code is 1 + 2 + 4 + ... + log N (the last term being log N, a power of 2). What is the order of growth?

*A:* Substitute Q = log N into 1 + 2 + 4 + ... + Q = 2Q - 1 to get 2 log N - 1 ∈ **Θ(log N)**. This substitution trick was called out explicitly in lecture.

---

## Summary

- **The recipe:** pick a representative operation (cost model), count it as C(N), find simple f(N) with C(N) ∈ Θ(f(N)), and if the operation is constant time then R(N) ∈ Θ(f(N)). Usually, but not always, analyze the worst case.
- **Simplifications:** drop low-order terms (they become insignificant as N grows) and drop multiplicative constants (we only want the shape, and far bigger constants like machine speed are already gone).
- **Θ ≈ "equals," O ≈ "≤," Ω ≈ "≥."** Each names a family of functions. Θ needs constants k₁ and k₂ sandwiching R(N); O needs only the upper constant; Ω needs only the lower; all "for N greater than some N₀."
- **Prefer Θ when you can.** Use O (or Ω) when the runtime is not proportional to a single f(N) on all inputs, which is why the disjoint sets table lists O(N) for `QuickUnionDS.connect` (it might be constant time) but Θ(N) for `QuickFindDS.connect` (it always scans the whole array).
- **Big Omega's main use is theoretical floors:** median-finding is Ω(N) because every algorithm must look at every element.
- **Θ/O/Ω are independent of best/worst case.** You choose a case, get a function, then describe that function's growth.
- **`countDuplicates` is Θ(N²):** the executed (i, j) pairs form a right triangle of area ~N²/2; exactly, 1 + 2 + ... + (N-1) = N(N-1)/2.
- **`scan` is Θ(N):** the inner loop `for (j = i-1; j <= i+1; ...)` always runs exactly 3 times, a width-3 band of length N - 1, total 3(N - 1). Nested loops do **not** imply multiplication.
- **The doubling `printParty` is Θ(N):** `i` takes only power-of-2 values, giving C(N) = 1 + 2 + 4 + ... + N = 2N - 1 at the peaks, a staircase that nonetheless stays between 0.5N and 4N (constants are arbitrary; any valid pair suffices).
- **That staircase is the array-resizing argument:** doubling capacity means the expensive resize happens only at powers of 2, and N operations cost Θ(N) total, i.e. constant average cost per operation.
- **Know two sums:** 1 + 2 + 3 + ... + Q = Q(Q+1)/2 ∈ Θ(Q²); 1 + 2 + 4 + ... + Q = 2Q - 1 ∈ Θ(Q) for Q a power of 2. Q may itself be a function of N (e.g. Q = log N).
- **Log base is irrelevant** under Θ, since changing base is a constant factor.
- **Three strategies:** find the exact sum, write out small examples in a table, draw pictures (grids and areas). Picture-based arguments are accepted in 61B and used in real research literature.
- **There is no magic shortcut.** Runtime analysis requires careful thought; CS70 and CS170 go much deeper (the runtime of weighted quick union with path compression was a research-level result).
