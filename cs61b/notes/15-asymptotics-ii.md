<!-- Wed, Sep 30, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 15: Asymptotics II

This lecture finishes the asymptotic notation toolkit and then spends most of its time doing actual analysis. First we review the workflow from Asymptotics I (pick a representative operation, count it as a function of `N`, find the order of growth of that count), restate the formal definition of Big Theta, and add the third piece of notation: Big Omega, a lower bound, which pairs with Big O (upper bound) and Big Theta ("order of growth is exactly"). We use that new vocabulary to explain something that was left dangling in the disjoint sets lectures: why the performance table lists `O(N)` and `O(log N)` in some cells instead of `Θ(...)`. Then come the analysis examples: a classic nested loop over all pairs (`countDuplicates`, which is `Θ(N²)` via the triangle sum `1 + 2 + ... + (N-1)`), a nested loop whose inner loop only runs 3 times regardless of `N` (`scan`, which is only `Θ(N)`), and the trickiest one, a doubling outer loop with an inner loop that runs `i` times (`printParty`, which is `Θ(N)`, not `Θ(N log N)`, because `1 + 2 + 4 + ... + N = 2N - 1`). The moral of the lecture, repeated verbatim on the slides, is that **there is no magic shortcut for asymptotic analysis**: you need careful thought, two memorized sums, and the habits of finding exact sums, writing out small examples, and drawing pictures.

---

## Key Concepts

### 1. The analysis workflow (review from Asymptotics I)

Given a code snippet, we express its runtime as a function `R(N)`, where `N` is some property of the input, often its size. We almost never want `R(N)` exactly; we want its **order of growth**.

The (non-universal) recipe:

1. Choose a **representative operation**, also called a **cost model**, and let `C(N)` be the number of times that operation executes as a function of `N`.
2. Determine the order of growth `f(N)` of `C(N)`, i.e. find `f` with `C(N) ∈ Θ(f(N))`. Often, but not always, we count the worst case.
3. If that operation takes constant time, then `R(N) ∈ Θ(f(N))`.

The lecture's opening example:

```java
public static void printParty(int N) {
    for (int i = 10; i < N; i += 2) {
        System.out.println("hello");
    }
}
```

Using `println` as the cost model, the count is `(N - 10) / 2`, so the runtime is `Θ(N)`.

Why drop the `-10`? Because it becomes insignificant as `N` gets big. Why drop the `/2`? Because we are describing how the runtime *grows* with `N`, and we have already thrown away a pile of other constant factors anyway (how long does `println` actually take? how fast is my laptop?). Once you have accepted those unknown constants, carrying a `1/2` around is pointless.

The two simplifications, stated on the slide:

- Ignore lower order terms.
- Ignore multiplicative constants.

### 2. Big Theta as a mathematical object, not a code property

Big Theta describes the order of growth of a *function*. Code enters the picture only because we assume the runtime is a function of the input.

| function `R(N)` | order of growth |
| --- | --- |
| `N³ + 3N⁴` | `Θ(N⁴)` |
| `1/N + N³` | `Θ(N³)` |
| `1/N + 5` | `Θ(1)` |
| `N·eᴺ + N` | `Θ(N·eᴺ)` |
| `40 sin(N) + 4N²` | `Θ(N²)` |

Note the last one: the `sin(N)` term wiggles forever and never dies out, but it is bounded, so it cannot affect the order of growth. And `1/N + 5` is `Θ(1)`: as `N` grows, `1/N` vanishes, leaving a constant.

### 3. Big O: "less than or equal"

Informally, if Big Theta is like `=` for orders of growth, Big O is like `≤`. It is an **upper bound only**, so it is deliberately loose. All of the following are true simultaneously:

- `N³ + 3N⁴ ∈ Θ(N⁴)`
- `N³ + 3N⁴ ∈ O(N⁴)`
- `N³ + 3N⁴ ∈ O(N⁶)`
- `N³ + 3N⁴ ∈ O(N!)`
- `N³ + 3N⁴ ∈ O(Nᴺ!)`

Saying `N³ + 3N⁴ ∈ O(N!)` is a true statement and a useless one. True but loose is the normal state of a Big O claim.

### 4. Big Omega: "greater than or equal"

Big Omega is the mirror image: a **lower bound**. All of the following are true:

- `N³ + 3N⁴ ∈ Θ(N⁴)`
- `N³ + 3N⁴ ∈ Ω(N⁴)`
- `N³ + 3N⁴ ∈ Ω(N³)`
- `N³ + 3N⁴ ∈ Ω(log N)`
- `N³ + 3N⁴ ∈ Ω(1)`

The lecture's worked instance of the definition: `40 sin(N) + 4N² ∈ Ω(N)`, with `R(N) = 40 sin(N) + 4N²`, `f(N) = N`, and `k₁ = 20`. That is, `20N ≤ 40 sin(N) + 4N²` for all sufficiently large `N`. (Check it at `N = 10`: `200 ≤ 400 + 40 sin(10)`, and the gap only widens.)

**What Big Omega is actually used for:** setting theoretical floors on *problem* difficulty, not on a particular implementation. The slide's example: finding the median of a list of numbers takes `Ω(N)`. That is a claim about *every possible algorithm* for the problem: the best anyone could ever do is linear time. The lecture asks "Why?" and leaves it open, promising to use `Ω` again in a much later lecture. *(extra context: the intuition is that if an algorithm never examines some element, an adversary can change that element to move the true median, so the algorithm must be wrong on one of the two inputs. Hence every correct algorithm must read all `N` elements.)*

### 5. The three notations side by side

| | Informal meaning | Family | Some family members |
| --- | --- | --- | --- |
| Big Theta `Θ(f(N))` | Order of growth **is** `f(N)` | `Θ(N²)` | `N²/2`, `2N²`, `N² + 38N + N` |
| Big O `O(f(N))` | Order of growth is **less than or equal** to `f(N)` | `O(N²)` | `N²/2`, `2N²`, `lg(N)` |
| Big Omega `Ω(f(N))` | Order of growth is **greater than or equal** to `f(N)` | `Ω(N²)` | `N²/2`, `2N²`, `eᴺ` |

Two things to notice. `N²/2` and `2N²` appear in all three families, because a function that *is* `Θ(N²)` is automatically both `O(N²)` and `Ω(N²)`. And the distinguishing members are the loose ones: `lg(N)` is in `O(N²)` but not `Θ(N²)`, and `eᴺ` is in `Ω(N²)` but not `Θ(N²)`.

*(The Θ(N²) row is transcribed from the slide as `N² + 38N + N`; the final term was likely a low-order term such as `√N` that the PDF extraction lost. Either way the function is `Θ(N²)`.)*

### 6. Why the disjoint sets table uses Big O

Here is the performance summary the lecture revisits:

| Implementation | constructor | connect | isConnected |
| --- | --- | --- | --- |
| `ListOfSetsDS` | `Θ(N)` (make N sets) | `O(N)` (find each element, combine 2 sets) | `O(N)` (check up to N sets) |
| `QuickFindDS` (`id` array holds set ID for each item) | `Θ(N)` (initialize size-N array) | `Θ(N)` (look through entire array, update IDs) | `Θ(1)` (two array accesses) |
| `QuickUnionDS` (`parent` array holds each item's parent in tree structure) | `Θ(N)` (initialize size-N array) | `O(N)` (climb tree to find root) | `O(N)` (climb 2 trees to compare roots) |
| `WeightedQuickUnionDS` (like `QuickUnionDS`, but track tree size; connect smaller tree under bigger tree) | `Θ(N)` (initialize size-N array) | `O(log N)` (climb tree to find root) | `O(log N)` (climb 2 trees to compare roots) |

The question on the slide: why are some entries Big O instead of Big Theta? The answer given: because those operations **might be much faster, e.g., constant time**.

Think about `QuickUnionDS.isConnected(3, 5)`. If both 3 and 5 happen to be roots, you do two array reads and stop: constant time. If they sit at the bottom of a long spindly tree, you climb `Θ(N)` levels. The runtime is not a function of `N` alone; it also depends on which items you ask about and what the tree looks like. So there is no single `f(N)` with runtime `∈ Θ(f(N))` covering all cases, and the honest statement is the upper bound: it is never worse than `O(N)`.

Contrast `QuickFindDS.connect`, which is `Θ(N)`: it always scans the entire `id` array, no matter what. Same work every time, so Big Theta applies.

### 7. The central lesson: no magic shortcut

The slide says it twice and asks you to repeat it: **there is no magic shortcut for asymptotic analysis problems** (well... usually). Runtime analysis often requires careful thought. CS70 and especially CS170 cover this in much more detail. CS 61B is not a math class, but you are expected to know exactly two sums:

- **Sum of the first natural numbers:** `1 + 2 + 3 + ... + Q = Q(Q+1)/2 = Θ(Q²)`
- **Sum of the first powers of 2:** `1 + 2 + 4 + 8 + ... + Q = 2Q - 1 = Θ(Q)` (where `Q` is a power of 2)

And three strategies:

1. Find the exact sum.
2. Write out examples (build a table of `N` versus count for small `N`).
3. Draw pictures.

The slide underscores the "no shortcut" point with an intimidating runtime expression for QR decomposition taken from Trefethen's *Numerical Linear Algebra*: real analysis in the wild does not reduce to pattern matching on loop nesting depth.

### 8. Geometric sums and the ArrayList connection

The doubling example (`printParty` in Example 3 below) has this shape: every time `N` is a power of 2, we do something expensive, and the total runtime is `Θ(N)`. The lecture points out we have seen this pattern before: **ArrayList resizing**. Each time the array fills up, you allocate a new array of double the size and copy everything over, which costs `Θ(current size)`. Summing those copy costs over all insertions up to `N` gives `1 + 2 + 4 + ... + N = 2N - 1 ∈ Θ(N)` total work, which is why resizing by doubling is cheap on average even though individual resizes are expensive.

---

## Definitions

**`R(N)`:** the runtime of a code snippet expressed as a function of `N`, where `N` is some property of the input (often its size).

**Order of growth:** the dominant behavior of a function as its input grows large, with lower order terms and multiplicative constants discarded. `Θ`, `O`, and `Ω` are the notations for talking about it.

**Cost model (representative operation):** a single operation chosen to stand in for the whole snippet's work. `C(N)` denotes how many times it executes as a function of `N`. A cost model is *good* when the total runtime is proportional to `C(N)`; a conclusion drawn from a bad cost model is simply wrong.

**Big Theta, `R(N) ∈ Θ(f(N))`:** there exist positive constants `k₁` and `k₂` such that
`k₁ · f(N) ≤ R(N) ≤ k₂ · f(N)`
for all values of `N` greater than some `N₀` (i.e. for very large `N`). Informally: the order of growth *is* `f(N)`.

**Big O, `R(N) ∈ O(f(N))`:** there exists a positive constant `k₂` such that
`R(N) ≤ k₂ · f(N)`
for all values of `N` greater than some `N₀`. Big O is just an upper bound. Informally: the order of growth is less than or equal to `f(N)`. *(The slide's wording mentions `k₁` and `k₂`, carried over from the Big Theta slide, but the inequality shown for Big O is the upper bound only.)*

**Big Omega, `R(N) ∈ Ω(f(N))`:** there exists a positive constant `k₁` such that
`k₁ · f(N) ≤ R(N)`
for all values of `N` greater than some `N₀`. Informally: the order of growth is greater than or equal to `f(N)`.

**"Simple `f(N)`":** as used in the lecture's poll questions, an answer with no unnecessary multiplicative constants and no additive terms. `Θ(N)`, not `Θ(3N - 3)`.

**Theoretical floor on a problem:** a `Ω(...)` bound that holds for every possible algorithm solving the problem, not just for one implementation. Example: median finding is `Ω(N)`.

**Sum of the first natural numbers:** `1 + 2 + 3 + ... + Q = Q(Q+1)/2 ∈ Θ(Q²)`.

**Sum of the first powers of 2:** for `Q` a power of 2, `1 + 2 + 4 + 8 + ... + Q = 2Q - 1 ∈ Θ(Q)`.

---

## Worked Examples

A note on diagrams before we start. These examples are all `static` methods over `int`s and `int[]`s, so there are no reference types to draw box-and-pointer diagrams for. The pictures the lecture draws instead are **iteration grids**: a 2D plot with `i` on one axis and `j` on the other, where each marked cell is one execution of the inner loop body. The order of growth is the *area* of the marked region. I reconstruct those grids in words and ASCII below, since the slide images did not survive PDF extraction.

Mentally, the environment for these methods is tiny: a frame holding the parameter `N` (or the reference `a`), plus the loop variables `i` and `j`, which live only for the duration of their loops. Counting executions means tracing how those `int` values change.

---

### Example 1: `countDuplicates`, the all-pairs triangle

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

**What it does:** counts how many pairs of positions in the array hold equal values. The inner loop starts at `j = i + 1` rather than `j = 0`, so each unordered pair `{i, j}` is examined exactly once and no element is compared to itself.

**Cost model:** the `==` comparison. The intuitive question is "how many pairs do we check?"

**Trace the grid.** Let `N = a.length = 6`. For each value of `i`, the inner loop runs for `j = i+1 .. N-1`:

```
        j: 0   1   2   3   4   5
   i=0:      ==  ==  ==  ==  ==      (5 comparisons)
   i=1:          ==  ==  ==  ==      (4)
   i=2:              ==  ==  ==      (3)
   i=3:                  ==  ==      (2)
   i=4:                      ==      (1)
   i=5:                              (0)
```

The marked cells form a right triangle whose legs have length about `N`.

**Two ways to finish.**

*Geometric:* the area of a right triangle with legs `N` is about `N²/2`, so the order of growth is `N²`.

*Exact arithmetic:* the counts per row, read from the bottom up, are `1 + 2 + 3 + ... + (N-3) + (N-2) + (N-1)`. The slide's picture-proof pairs up terms: arrange the sum in an `N` by `N` square and observe it is `(N-1)/2` pairs each summing to `N`, giving

```
C(N) = N(N - 1)/2
```

For `N = 6` that is `6 · 5 / 2 = 15`, which matches the 15 cells above. Dropping the multiplicative constant and the lower order term:

```
R(N) ∈ Θ(N²)
```

**Why this is the canonical example:** whenever an inner loop's trip count grows linearly with the outer index, you get a triangle, and triangles are `Θ(N²)`.

---

### Example 2: `scan`, the loop nest that is *not* quadratic

```java
public static void scan(int N) {
    int s = 0;
    for (int i = 2; i <= N; i += 1) {
        for (int j = i - 1; j <= i + 1; j += 1) {
            s += i * j;
        }
    }
}
```

This was a poll question: find a simple `f(N)` with `R(N) ∈ Θ(f(N))`, choosing among `1`, `log N`, `N`, `N log N`, `N²`, other.

**What it does:** for each `i` from 2 to `N`, it adds `i·j` into the accumulator `s` for the three values `j = i-1`, `j = i`, `j = i+1`. The function returns nothing and `s` is never read, so the code is pure busywork, which is fine; we only care about its cost.

**Cost model:** the `s += i*j` statement.

**Trace the count.** The key observation is that the inner loop's bounds both move with `i`, so its *trip count is constant*: `(i+1) - (i-1) + 1 = 3` iterations, always, regardless of `N`.

| `N` | `s += i*j` calls |
| --- | --- |
| 0 | 0 |
| 1 | 0 |
| 2 | 3 |
| 3 | 3 + 3 = 6 |
| 4 | 3 + 3 + 3 = 9 |
| 5 | 12 |

(For `N = 0` and `N = 1` the outer loop never runs, since `i` starts at 2.)

**The picture.** On an `i` versus `j` grid, the marked cells are not a triangle but a **diagonal band of width 3**, running from `i = 2` up to `i = N`:

```
        j: 0   1   2   3   4   5
   i=2:  *   *   *
   i=3:      *   *   *
   i=4:          *   *   *
   i=5:              *   *   *
```

The band has width 3 and length `N - 1`, so its area is `3(N - 1)`.

**Answer:** `C(N) = 3(N - 1) ∈ Θ(N)`, so `R(N) ∈ Θ(N)`. The poll answer is `N`.

**Two warnings the slide attaches to this example.**

1. `R(N) ∈ Θ(N)` is only true *because `s += i*j` was a good cost model*. Everything else in the loop body (the multiply, the add, the comparison, the increment) happens a constant number of times per `s += i*j`, so the total runtime really is proportional to that count.

2. **"Counting `i = 2` would have been a bad cost model."** The initialization `int i = 2` executes exactly once, no matter how big `N` is. If you had chosen it as your representative operation you would have computed `C(N) = 1 ∈ Θ(1)` and concluded the method runs in constant time, which is false. A cost model has to be an operation that the work actually scales with.

---

### Example 3: `printParty`, the doubling loop (the tricky one)

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

Find a simple `f(N)` such that `R(N) ∈ Θ(f(N))`. Same choices: `1`, `log N`, `N`, `N log N`, `N²`, other.

**What it does:** the outer loop variable `i` takes only the values `1, 2, 4, 8, 16, ...` up to `N`, because it is *multiplied* by 2 each time rather than incremented. For each such `i`, the inner loop prints `hello` exactly `i` times. (`int ZUG = 1 + 1;` is a constant-time distractor; it declares a local, computes 2, and is never used.)

The lecture presents this as three successive attempts, because the tempting answer is wrong.

#### Attempt 1: the trap

The instinct is: outer loop runs `log N` times, inner loop runs "up to `N`" times, so `Θ(N log N)`. That reasoning is wrong because the inner loop does **not** run `N` times on every pass. It runs `i` times, and `i` is small for most passes. Only the very last pass has `i` near `N`.

#### Attempt 2: write out examples

Build the table of `N` versus number of `println` calls:

| `N` | `println` calls |
| --- | --- |
| 0 | 0 |
| 1 | 1 |
| 2 | 1 + 2 = 3 |
| 3 | 3 |
| 4 | 1 + 2 + 4 = 7 |
| 5 | 7 |
| 6 | 7 |
| 7 | 7 |
| 8 | 1 + 2 + 4 + 8 = 15 |
| 9 | 15 |

Notice the staircase: `N = 2` and `N = 3` both print 3 times, because after `i = 2` the next value `i = 4` already exceeds `N = 3`. The count only jumps when `N` crosses a power of 2.

**The picture.** On the `i` versus `j` grid, only the rows `i = 1, 2, 4, 8, ...` are marked at all, and row `i` is filled from `j = 0` to `j = i - 1`:

```
        j: 0 1 2 3 4 5 6 7
   i=1:  *
   i=2:  * *
   i=3:  (row never visited)
   i=4:  * * * *
   i=5..7: (never visited)
   i=8:  * * * * * * * *
```

So the shape is a small number of rows whose lengths double, and the last row alone accounts for half the total area. That is the structural reason the answer is linear and not `N log N`.

**The exact count.** If `N` is a power of 2:

```
C(N) = 1 + 2 + 4 + ... + N
```

Example, `N = 8`: `C(N) = 1 + 2 + 4 + 8 = 15`.

#### Attempt 3: find the exact sum

The pattern in the table (`1, 3, 7, 15, 31, ...`) is one less than twice `N`:

```
C(N) = 1 + 2 + 4 + ... + N = 2N - 1      (N a power of 2)
```

**Proof by induction** (given on the slide):

- *Base case,* `N = 1`: the sum is just `1`, and `2·1 - 1 = 1`. ✓
- *Inductive hypothesis:* the sum of powers of 2 up to `N` is `1 + 2 + 4 + ... + N = 2N - 1`.
- After `N`, the next power of 2 is `2N`, so we want to show `1 + 2 + 4 + ... + N + 2N = 4N - 1` (since `2·(2N) - 1 = 4N - 1`).
- Start from the hypothesis and add `2N` to both sides:
  `1 + 2 + 4 + ... + N + 2N = (2N - 1) + 2N = 4N - 1`. ✓

*Note to mathy students (from the slide):* you might prefer to write "suppose `N = 2ᵏ` for some `k`, then `Σᵏᵢ₌₀ 2ⁱ = 2ᵏ⁺¹ - 1`." That is the same statement. Josh's experience is that 61B students get `k` and `N` confused, so the more formal framing ends up being a distraction.

#### The answer, and handling the `N`s that are not powers of 2

`C(N) = 2N - 1 ∈ Θ(N)`, so `R(N) ∈ Θ(N)`. The poll answer is `N`.

But `2N - 1` was derived only for powers of 2. For other `N` the function is a staircase, and we appeal directly to the definition. `C(N) ∈ Θ(N)` means we can find `k₁` and `k₂` such that for large `N`:

```
k₁ · N ≤ C(N) ≤ k₂ · N
```

The slide offers `k₁ = 0.5, k₂ = 4`, and notes that `k₁ = 0.001, k₂ = 10³` would work too: the constants just have to exist, they do not have to be tight. Checking against the table:

| `N` | `C(N)` | `0.5N` (below) | `4N` (above) |
| --- | --- | --- | --- |
| 1 | 1 | 0.5 | 8 |
| 4 | 1 + 2 + 4 = 7 | 2 | 16 |
| 7 | 1 + 2 + 4 = 7 | 3.5 | 28 |
| 8 | 1 + 2 + 4 + 8 = 15 | 4 | 32 |
| 27 | 1 + 2 + 4 + 8 + 16 = 31 | 13.5 | 108 |
| 185 | ... + 64 + 128 = 255 | 92.5 | 740 |
| 715 | ... + 256 + 512 = 1023 | 357.5 | 2860 |

For large `N`, `C(N)` is always squeezed between `0.5N` and `4N`, so it is `Θ(N)` even though it is a staircase rather than a straight line. *(extra context: the general formula is `C(N) = 2·2^⌊lg N⌋ - 1`, and since `N/2 < 2^⌊lg N⌋ ≤ N` you get `N - 1 < C(N) ≤ 2N - 1`, which is exactly why constants like `0.5` and `4` comfortably work.)*

**Connection back to course material:** this is the ArrayList resizing pattern. Every time `N` is a power of 2 we do something expensive, and the total is `Θ(N)`.

---

### Example 4: modifying `printParty` to hit other orders of growth

The lecture closes by taking the same `Θ(N)` code and asking, for each target order of growth, what single change produces it.

```java
public static void printParty(int N) {
    for (int i = 1; i <= N; i = i * 2) {
        for (int j = 0; j < i; j += 1) {
            System.out.println("hello");
            int ZUG = 1 + 1; }}}
```

| New order of growth | Modification |
| --- | --- |
| `1` | Replace inner loop with `return;` |
| `log N` | Replace inner loop with a constant-time operation |
| `N` | (none, this is the original) |
| `N log N` | Inner loop always runs `N` times (`j < N` instead of `j < i`) |
| `N²` | `i` increments (`i = i + 1` instead of `i = i * 2`) |

Reasoning through each:

**`Θ(1)`:** with `return;` as the entire body of the outer loop, the method exits during the first iteration. Constant work.

```java
for (int i = 1; i <= N; i = i * 2) {
    return;
}
```

**`Θ(log N)`:** the outer loop alone executes about `log₂ N` times (`i` goes `1, 2, 4, ...` until it passes `N`). Replace the inner loop with one constant-time statement and the total is `Θ(log N)`.

```java
for (int i = 1; i <= N; i = i * 2) {
    System.out.println("hello");
}
```

**`Θ(N)`:** the original. `log N` outer passes, inner trip counts `1, 2, 4, ..., N` summing to `2N - 1`.

**`Θ(N log N)`:** make the inner loop's trip count independent of `i` and equal to `N`. Now the grid is a genuine rectangle: `log N` rows, each `N` cells wide.

```java
for (int i = 1; i <= N; i = i * 2) {
    for (int j = 0; j < N; j += 1) {   // N, not i
        System.out.println("hello");
    }
}
```

Note that this is the answer people *wrongly guess* for the original code. Seeing the version that really is `N log N` right next to the version that is only `N` is the point of the exercise.

**`Θ(N²)`:** keep `j < i` but let `i` increment instead of double. Now every row `i = 1, 2, 3, ..., N` is visited, and row `i` has `i` cells, giving the triangle `1 + 2 + 3 + ... + N = N(N+1)/2 ∈ Θ(N²)`.

```java
for (int i = 1; i <= N; i = i + 1) {
    for (int j = 0; j < i; j += 1) {
        System.out.println("hello");
    }
}
```

Both memorized sums appear here: the doubling version uses the powers-of-2 sum (`Θ(N)`), the incrementing version uses the natural-numbers sum (`Θ(N²)`).

---

## Common Pitfalls

**Assuming two nested loops means `Θ(N²)`.** `scan` has two nested loops and is `Θ(N)`. `printParty` has two nested loops and is `Θ(N)`. Loop nesting depth tells you nothing by itself; you have to count the inner loop's trip count as a function of the outer variable.

**Multiplying "outer loop count" by "inner loop maximum."** This is the specific error behind the wrong `N log N` answer for `printParty`. The inner loop runs `i` times, and `i` equals `N` only on the final pass. You must sum the trip counts, not multiply by the largest one.

**Confusing the two required sums.** `1 + 2 + 3 + ... + Q` is `Θ(Q²)`; `1 + 2 + 4 + ... + Q` is `Θ(Q)`. They look similar written out and differ by an entire order of growth. The doubling sum is dominated by its last term; the incrementing sum is not.

**Thinking Big O means "worst case" and Big Omega means "best case."** They are bounds on *functions*, not statements about which input you chose. You can give a Big O bound on the best case or a Big Omega bound on the worst case. The reason the disjoint sets table uses `O` is that the runtime is not a function of `N` alone, so no single `Θ` applies; it is not because `O` is secretly a synonym for "worst case."

**Writing `Θ` when the operation's cost depends on more than `N`.** `QuickUnionDS.isConnected` can be constant time or `Θ(N)` depending on the arguments and the tree shape, so `Θ(f(N))` is not available and `O(N)` is the right claim.

**Treating a true-but-loose Big O as an error.** `N³ + 3N⁴ ∈ O(N!)` is true. If a question asks for a *simple* `f(N)` with no unnecessary constants or additive terms, though, give the tight answer.

**Leaving constants and lower order terms in the final answer.** `Θ(3N - 3)` and `Θ(N(N-1)/2)` are not "simple" answers. Write `Θ(N)` and `Θ(N²)`.

**Picking a cost model that does not scale with the work.** Counting `i = 2` in `scan` gives `Θ(1)`, which is flatly wrong. Counting something that happens once per inner-loop iteration is safe; counting an initialization is not.

**Forgetting that `i = i * 2` gives `log N` iterations, not `N`.** And, symmetrically, forgetting that `i = i + 1` gives `N` iterations. The update expression, not the loop bound, determines the outer trip count.

**Assuming a staircase function cannot be `Θ` of a smooth function.** `C(N)` for `printParty` is flat between powers of 2 and then jumps. It is still `Θ(N)`, because the definition only demands constants `k₁` and `k₂` sandwiching it for large `N`.

**Proving something only for powers of 2 and stopping.** `C(N) = 2N - 1` holds only when `N` is a power of 2. To claim `Θ(N)` for all `N` you need the sandwich argument or the table of values.

---

## Likely Exam Points

### 1. Simplifying a function to its order of growth

Given a closed-form function, state `Θ`.

**Practice:** Give simple `f` such that each is `Θ(f(N))`: (a) `1/N + 100`, (b) `40 sin(N) + 4N²`, (c) `N·eᴺ + N`, (d) `N³ + 3N⁴`.

**Answer:** (a) `Θ(1)`, since `1/N → 0` and a constant remains. (b) `Θ(N²)`, since `sin(N)` is bounded and cannot influence growth. (c) `Θ(N·eᴺ)`, since `N·eᴺ` dwarfs `N`. (d) `Θ(N⁴)`, since `3N⁴` dominates and the constant 3 is dropped.

### 2. True/false on `Θ`, `O`, `Ω`

**Practice:** Let `R(N) = N³ + 3N⁴`. Mark each true or false: (a) `R ∈ Θ(N⁴)`, (b) `R ∈ O(N⁶)`, (c) `R ∈ Ω(N⁶)`, (d) `R ∈ Ω(log N)`, (e) `R ∈ Θ(N³)`, (f) `R ∈ O(N!)`.

**Answer:** (a) T. (b) T, `O` is an upper bound and loose bounds are still true. (c) F, `N⁶` grows faster than `R`, so it cannot be a lower bound. (d) T, any slower-growing function is a valid lower bound. (e) F, `Θ` must be exact and `R` grows faster than `N³`. (f) T, true and useless.

### 3. Stating a formal definition and exhibiting constants

**Practice:** Show `40 sin(N) + 4N² ∈ Ω(N)` by giving `k₁` and explaining.

**Answer:** `Ω` requires a positive `k₁` with `k₁ · N ≤ 40 sin(N) + 4N²` for all `N` beyond some `N₀`. Take `k₁ = 20` (the lecture's choice). Since `40 sin(N) ≥ -40` always, the right side is at least `4N² - 40`, and `4N² - 40 ≥ 20N` for all `N ≥ 6`. So the definition is satisfied with `k₁ = 20`, `N₀ = 6`.

### 4. All-pairs nested loop

**Practice:** Give the order of growth of the worst case runtime of `countDuplicates` on an array of length `N`, and justify with an exact count.

**Answer:** `Θ(N²)`. Using `==` as the cost model, for each `i` the inner loop runs from `j = i+1` to `N-1`, giving `N-1-i` comparisons. Summing over `i` gives `(N-1) + (N-2) + ... + 2 + 1 = N(N-1)/2`, the area of a right triangle with legs `N`. Dropping the constant `1/2` and the lower order term gives `Θ(N²)`. Note the count does not depend on the array contents at all, only the `duplicates += 1` count does, so best and worst case are both `Θ(N²)`.

### 5. Loop nest whose inner trip count is constant

**Practice:**
```java
for (int i = 2; i <= N; i += 1) {
    for (int j = i - 1; j <= i + 1; j += 1) {
        s += i * j;
    }
}
```
Give simple `f(N)` with `R(N) ∈ Θ(f(N))`.

**Answer:** `Θ(N)`. The inner loop always runs exactly `(i+1) - (i-1) + 1 = 3` times, independent of `N`, so the total number of `s += i*j` executions is `3(N-1)`. On an `i`-`j` grid the marked cells form a band of width 3 and length `N-1`, whose area is linear. Beware of choosing a bad cost model: counting the one-time `int i = 2` initialization would falsely suggest `Θ(1)`.

### 6. The doubling loop (the flagship question)

**Practice:**
```java
for (int i = 1; i <= N; i = i * 2) {
    for (int j = 0; j < i; j += 1) {
        System.out.println("hello");
    }
}
```
Give simple `f(N)` with `R(N) ∈ Θ(f(N))`, and say why the answer is not `N log N`.

**Answer:** `Θ(N)`. The values of `i` are `1, 2, 4, ..., ` up to `N`, and for each the inner loop prints `i` times, so `C(N) = 1 + 2 + 4 + ... + N = 2N - 1` when `N` is a power of 2. The tempting `N log N` answer comes from multiplying the `log N` outer passes by the *maximum* inner trip count `N`, but the inner loop only runs `N` times on the final pass; earlier passes are geometrically smaller, and the whole sum is dominated by the last term. For `N` not a power of 2, `C(N)` is a staircase, but `0.5N ≤ C(N) ≤ 4N` for large `N`, which satisfies the `Θ` definition with `k₁ = 0.5`, `k₂ = 4`.

### 7. Prove a sum by induction

**Practice:** Prove that if `N` is a power of 2 then `1 + 2 + 4 + ... + N = 2N - 1`.

**Answer:** Base case `N = 1`: the sum is `1` and `2(1) - 1 = 1`. Inductive step: assume `1 + 2 + ... + N = 2N - 1`. The next power of 2 is `2N`; adding `2N` to both sides gives `1 + 2 + ... + N + 2N = 2N - 1 + 2N = 4N - 1 = 2(2N) - 1`, which is the claim for `2N`. Done.

### 8. Modify code to achieve a target order of growth

**Practice:** Starting from the `Θ(N)` doubling `printParty`, make a single change so the runtime becomes (a) `Θ(N log N)`, (b) `Θ(N²)`, (c) `Θ(log N)`, (d) `Θ(1)`.

**Answer:** (a) change the inner condition to `j < N` so the inner loop always runs `N` times, giving a `log N` by `N` rectangle. (b) change `i = i * 2` to `i = i + 1`, so every row `1..N` is visited with `i` cells each, giving `1 + 2 + ... + N = Θ(N²)`. (c) replace the entire inner loop with a single constant-time statement, leaving only the `log N` outer passes. (d) replace the inner loop with `return;`, so the method exits on the first outer iteration.

### 9. Why Big O rather than Big Theta in a performance table

**Practice:** The disjoint sets summary lists `QuickFindDS.connect` as `Θ(N)` but `QuickUnionDS.connect` as `O(N)`. Explain the difference.

**Answer:** `QuickFindDS.connect` always scans and potentially updates the whole `id` array, so its runtime is the same order for every input of size `N`, and `Θ(N)` is accurate. `QuickUnionDS.connect` climbs from the given items to their roots, and that climb might be a single step (if the items are already roots, so constant time) or as long as `N` in a degenerate spindly tree. Because the cost depends on which items are passed in and on the current tree shape, not on `N` alone, no single `Θ(f(N))` describes it; we state the upper bound `O(N)` instead. The same reasoning explains `WeightedQuickUnionDS` at `O(log N)`: the weighting caps tree height at `Θ(log N)`, but individual operations may be much faster.

### 10. Knowing what `Ω` is for

**Practice:** What does it mean to say "finding the median of a list of `N` numbers is `Ω(N)`"? Is this a statement about a particular algorithm?

**Answer:** No. It is a theoretical floor on the *problem*: among the space of all possible algorithms for finding the median, none can do better than linear time, so `Ω(N)` bounds every correct algorithm from below. This is different from using `Ω` to bound one implementation's runtime function. *(extra context: the lecture leaves the "why" as an open question and defers `Ω`'s serious use to a later lecture; the standard argument is that an algorithm ignoring any element can be fooled by changing that element.)*

---

## Summary

- `R(N)` is a snippet's runtime as a function of the input; we describe only its order of growth, dropping lower order terms and multiplicative constants.
- Workflow: pick a representative operation (cost model), count it as `C(N)`, find `f` with `C(N) ∈ Θ(f(N))`, and if the operation is constant time then `R(N) ∈ Θ(f(N))`.
- `Θ(f(N))`: order of growth *is* `f(N)`. Formally, positive `k₁, k₂` exist with `k₁·f(N) ≤ R(N) ≤ k₂·f(N)` for all large `N`.
- `O(f(N))`: order of growth is `≤ f(N)`. Upper bound only, so it can be arbitrarily loose and still true.
- `Ω(f(N))`: order of growth is `≥ f(N)`. Formally, a positive `k₁` exists with `k₁·f(N) ≤ R(N)` for all large `N`. Chiefly used for theoretical floors on problem difficulty, e.g. median finding is `Ω(N)`.
- A function that is `Θ(g)` is automatically both `O(g)` and `Ω(g)`; the interesting members of `O` and `Ω` families are the loose ones (`lg N ∈ O(N²)`, `eᴺ ∈ Ω(N²)`).
- Disjoint sets entries use `O` rather than `Θ` when the cost depends on more than `N` (arguments, tree shape) and so might be much faster, e.g. constant time. `QuickFindDS.connect` is `Θ(N)` because it always does the same amount of work.
- `countDuplicates` (inner loop from `i+1` to `N`): triangle of area `N(N-1)/2`, so `Θ(N²)`.
- `scan` (inner loop from `i-1` to `i+1`): band of width 3 and length `N-1`, so `Θ(N)`. Nested loops do not imply quadratic.
- `printParty` with `i = i * 2` and inner loop to `i`: `C(N) = 1 + 2 + 4 + ... + N = 2N - 1`, so `Θ(N)`, not `Θ(N log N)`. The last pass dominates.
- For `N` not a power of 2, `C(N)` is a staircase but still sits between `0.5N` and `4N` for large `N`, which is all `Θ` requires.
- That doubling pattern (expensive work each time `N` hits a power of 2, `Θ(N)` total) is exactly ArrayList resizing.
- Small mutations change everything: `j < N` gives `Θ(N log N)`, `i = i + 1` gives `Θ(N²)`, dropping the inner loop gives `Θ(log N)`, `return;` gives `Θ(1)`.
- Memorize two sums: `1 + 2 + 3 + ... + Q = Q(Q+1)/2 = Θ(Q²)` and `1 + 2 + 4 + ... + Q = 2Q - 1 = Θ(Q)` for `Q` a power of 2.
- Strategies when stuck: find the exact sum, write out examples in a table, draw pictures. There is no magic shortcut for asymptotic analysis.
