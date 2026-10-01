<!-- Fri, Sep 25, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 13: Asymptotics I

## Overview

This lecture opens the second phase of CS 61B: instead of asking "how expensive is this code to *write* and maintain?" we start asking "how expensive is this code to *run*?" The motivating problem is deciding, rigorously, why `dup2` (scan a sorted array and compare each element to its neighbor) is better than `dup1` (compare every possible pair). A raw answer like "it took 0.3 seconds" is useless because it depends on the machine and on the particular input, so we build a characterization that is machine-independent, input-robust, mathematical, and short. The tool is the **order of growth** of a function: take the function, throw away low-order terms, throw away multiplicative constants, and keep the shape. Formally this is written with **Big Theta**, `R(N) ∈ Θ(f(N))`, which is just set membership: `R` belongs to the family of functions that grow like `f`. The lecture builds this bottom-up (intuition from graphs, then the order-of-growth rule, then the formal two-sided-bound definition with its geometric reading), then comes back to code: count operations in a table, restrict to the **worst case**, argue that whatever the per-operation times are the largest term dominates, and conclude `dup1` has worst-case runtime in `Θ(N²)` while `dup2` is in `Θ(N)`. Because the tedious counting table is overkill, the lecture introduces the **cost model**: pick one representative operation deep inside the loops and count only that. Finally, the table of runtimes across orders of growth shows why this matters at all: for `N = 100,000`, linear is about a second, `N²` is hours, `N³` is decades, and exponential is effectively never.

A note on sources: the posted slide deck is the Spring 2026 deck (footers say "Lecture 12," and it is flagged as not yet updated), and the instructor rebuilt the live lecture that morning around `dup1`/`dup2` instead of the deck's `countEvens`. Both sets of examples are covered below. **Big O was on the slides but was not reached in lecture** and was explicitly deferred to the next lecture; it is included here, marked, because the slides are fair game.

---

## Key Concepts

### 1. Two flavors of efficiency

"An engineer will do for a dime what any fool will do for a dollar."

- **Programming cost**: how long it takes you to develop the program, and how easy it is to read, modify, and maintain. The majority of software cost is in *maintenance*, not initial development. This was the focus of the course so far, and the lecture notes it will be revisited later (including a guest lecture on agent-assisted programming).
- **Execution cost**: how much *time* your program takes to run and how much *memory* it needs. This is the subject from here to the end of the course.

### 2. The motivating question: `dup1` vs `dup2`

Given a **sorted** `int[]`, does it contain duplicates?

- `dup1`: check every possible pair `(i, j)` with `j > i`.
- `dup2`: check only adjacent pairs `(i, i+1)`. This is correct *because the array is sorted*: equal values must be adjacent.

Intuitively `dup2` is better because each element only consults one neighbor instead of all later elements. But the lecture pushed harder with two student-answered questions:

- Is `dup2` *always* much better? No. On a tiny array they are comparable, and if the duplicate pair is at the very front (e.g. `[-3, -3, ...]`) both return almost immediately.
- So the characterization we want must deliberately ignore the lucky, cheap cases and focus on the cases where the algorithms actually have to do work, which is exactly where they differ. That motivates the **worst case** choice later.

### 3. Runtime depends on the machine and on the input

If someone asks how long `countEvens` takes, the honest answer has three parts:

1. It depends how fast the computer is.
2. It depends on the size of the array.
3. It does some work per item, so the runtime "grows like `N`."

Points 1 and 2 are precisely the things we want to abstract away (point 1 entirely, point 2 by making `N` the variable). What survives is the *shape* in point 3.

### 4. Order of growth: the two simplifications

For a one-variable function `Q(N)`, the order of growth is what you get after:

- **Ignoring low-order terms.** As `N` gets large, smaller terms become negligible. `3N³ + N² → 3N³`.
- **Ignoring multiplicative constants.** `3N³ → N³`. Also `8N` grows the same way `N` does.

Two warnings from lecture:

- You drop *multiplicative constants only*. In `Ne^N + N` you may not drop the leading `N`, because `N` is not a constant. The answer is `Ne^N`, not `e^N`. (If you dropped non-constant factors you could "simplify" `N⁴` to `N³`, which is obviously wrong.)
- Keeping the constant, as in `3N³`, is a legitimate but different convention (tilde notation). It is not what this course uses.

### 5. Growth-rate intuition, built from graphs

The lecture demonstrated this live by plotting `100,000N`, `N²`, `N³`, and `3N³ + N²` and then zooming out.

- At `N = 50`, `100,000N` is already about `5 × 10⁶` and dwarfs `N²`. Small-`N` behavior is misleading.
- Zoom out and the parabola crosses the line and never looks back. Zoom out more and the cubics cross the parabola.
- Zoom out far enough and `100,000N` and `N²` are visually pinned to the x-axis: they have become irrelevant.
- `N³` and `3N³ + N²` keep a fixed ratio forever. One is always bigger, but it never pulls away. **That is what "grows at the same rate" means.** (A calculus-flavored way to see it: look at the limit of the ratio of the two functions as `N → ∞`. The instructor noted the limit-based definition is a valid alternative to the one taught here.)

Student poll result on "which grows fastest": roughly 3% / 6% / 39% / 90% across the four functions, with the plurality view picking only `3N³ + N²`, and a minority correctly picking both cubics as tied.

### 6. Why asymptotics, not stopwatches

We care about very large `N` because the real workloads are large: billions of interacting particles in a materials simulation, billions of social network users, billions of logged transactions, billions of bytes of video, millions of tokens through an attention mechanism. Algorithms that scale well (look like lines) beat algorithms that scale poorly (look like parabolas).

The `glorp` example from the slides: `glorp1` takes `2N²` operations, `glorp2` takes `500N`. For small `N`, `glorp1` is faster, but as the dataset grows the parabola falls farther and farther behind.

(The instructor's aside on attention using "millions" of tokens while other examples use "billions" is exactly this effect: a quadratic mechanism cannot be pushed to billions.)

The runtime table (from Kleinberg & Tardos, *Algorithm Design*), for `N = 100,000` under the slide's assumptions:

| order of growth | time to solve `N = 100,000` |
| --- | --- |
| `N` | about a second |
| `N log N` | still very feasible |
| `N²` | about 3 hours |
| `N³` | about 32 years |
| `2^N`, `N!` | "very long time," defined as more than `10²⁵` years |

Two consequences drawn in lecture: improving `N³` to `N²` on such a problem turns 32 years into 3 hours, and the hopelessness of exponential work is *why cryptography works at all* (a `2^N` algorithm on `N = 100` on a slow machine needs on the order of `10¹⁷` years). Much of theoretical CS is about moving problems leftward in that table, and quantum computing is largely about moving specific problems leftward.

### 7. Big Theta is just "order of growth," formalized

If `R(N)` has order of growth `f(N)`, we write `R(N) ∈ Θ(f(N))`.

The `∈` is literal set membership. `Θ(N⁴)` is the *set of all one-variable functions whose order of growth is `N⁴`*, an infinite family; `N³ + 3N⁴` is one member of it. Examples from the slides:

| function `R(N)` | order of growth | Big Theta |
| --- | --- | --- |
| `N³ + 3N⁴` | `N⁴` | `Θ(N⁴)` |
| `1/N + N³` | `N³` | `Θ(N³)` |
| `1/N + 5` | `1` | `Θ(1)` |
| `Ne^N + N` | `Ne^N` | `Θ(Ne^N)` |
| `40 sin(N) + 4N²` | `N²` | `Θ(N²)` |

Notes from lecture on this table: for `1/N + 5` the dominant piece is the constant `5`, and after dropping the multiplicative constant the order of growth is `1`. `1/N` by itself is in `Θ(1/N)`. And `40 sin(N)` is pure bounded noise (range `[-1, 1]` scaled by 40), so it cannot affect the order of growth of anything polynomial.

Crucially: **switching from the words "order of growth" to the symbol `Θ` does not change how you analyze code at all.** You never need to produce `k1` or `k2` when analyzing a program.

A poll asked which of several functions were in `Θ(x²)`; three of the listed functions were, and the class got about 97% right. The takeaway the instructor stressed is that this is a question you must not miss.

### 8. The formal definition and its geometric meaning

`R(N) ∈ Θ(f(N))` means there exist positive constants `k1` and `k2` such that

```
k1 · f(N)  ≤  R(N)  ≤  k2 · f(N)
```

for all `N` greater than some `N0` (that is, for all sufficiently large `N`).

Geometrically: draw a "small" copy of `f` and a "big" copy of `f`. `R` is eventually sandwiched between them forever. In the lecture's picture for `R(N) = 40 sin(N) + 4N²` with `f(N) = N²`, `k1 = 3` and `k2 = 5`: near the origin `R` may wiggle outside the band, but past some critical `N0` (the demo showed about 5.48) it stays inside no matter how far you zoom out. The specific value of `N0` carries no meaning; all that matters is that one exists.

"What does it mean to grow like `N²`? It means you have a small `N²` below you and a big `N²` above you, and you are always in between."

A student asked whether the upper curve is "worst case" and the lower is "best case." The answer given: not really. They are upper and lower *bounds on a single function*; best case and worst case are a separate idea (which input you feed the algorithm). Conflating the two is the root of most misuse of this notation, and the instructor flagged that "most people use Big O incorrectly," promising to return to it.

Historical note from lecture: the notation convention comes from Donald Knuth's 1970s paper ("after discussing this problem with people for several years, I have concluded that the following definitions will be useful..."), and **Big Theta was suggested to Knuth by Bob Tarjan**.

### 9. From a table of operation counts to a single runtime function

This is the step the instructor called "extremely important, make sure you understand it."

An operation-count table gives, for each `N`, a *tuple* of numbers, not a runtime. To convert it to a runtime function you assign a per-operation time: suppose `i = 0` costs `α` nanoseconds, `j = i + 1` costs `β`, each `<` costs `γ`, each increment `δ`, each `==` costs `ε`, each array access `ζ`. Then the total time is a sum like

```
α · 1 + β · (count of j = i+1) + γ · (count of <) + δ · (...) + ...
```

which for `dup1` is a polynomial whose biggest term is a constant times `N²`. **For very large `N` that term dominates regardless of the values of `α, β, γ, δ, ε, ζ`.** So the Θ class is determined by the counting table alone, and the unknown hardware constants never matter.

The slides give the same argument abstractly:

| operation | count |
| --- | --- |
| `<` | `100N² + 3N` |
| `>` | `2N³ + 1` |
| `&&` | `5,000` |

Total time `= α(100N² + 3N) + β(2N³ + 1) + 5000γ` nanoseconds, so the runtime is in `Θ(N³)` because the `2βN³` term eventually swamps everything else.

Honest caveat stated in lecture: real machines have branch prediction, caching, and so on, so "each operation costs a fixed number of nanoseconds" is a simplified mental model. It is close enough, and it works out in the end.

### 10. Worst case

An algorithm's runtime is not a function of `N` alone; it also depends on *which* input of size `N` you get. `dup1` on `[-3, -3, ...]` returns immediately; `dup1` on an array with no duplicates does all the work. That is why the count table entries are ranges (`0 to (N² + N)/2`, etc.).

The resolution: **throw away the lucky cases and define `R(N)` to be the worst-case runtime over inputs of size `N`.** Justification: when comparing algorithms we usually care about the worst case, and the worst case is exactly where the two algorithms differ. The slides add the caveat "often (but not always) we consider the worst case count," so this is a convention, not a law.

### 11. Cost models

Building the whole table is "extremely exhausting" and unnecessary. Instead:

> Choose one representative operation as your **cost model**, count only that operation, and take its order of growth as the order of growth of the runtime.

The implicit assumption is that the runtime is proportional to the count of the chosen operation.

For `dup1`, which operations are good cost models?

- `i = 0`: bad, it happens once.
- `return true`: bad, happens at most once.
- `j = i + 1`: bad, it is `Θ(N)`, not `Θ(N²)`, so it does not track the runtime.
- `<`: good.
- `==`: good.
- array accesses: good.

The pattern: **the good cost models are exactly the ones whose counts came out `Θ(N²)`, that is, the operations buried deepest in the loops.** How do you find them? Intuition, by asking what is executed over and over in the innermost loop. The warning: with sufficiently clever code the operation you *think* is representative may not be, so this takes care. (Term borrowed from Sedgewick and Wayne, *Algorithms*, 4th edition.)

### 12. The triangle trick (preview of the next lecture)

Rather than summing a series, draw the grid of all `(i, j)` pairs and mark the ones the nested loop actually visits. For `dup1`/`countDuplicates` the marked cells form a right triangle with legs of about `N`, so the count is about `N² / 2`, which is in `Θ(N²)`. The instructor explicitly said he would restart the next lecture from this picture.

### 13. Big O (on the slides, deferred in lecture)

Informally, `Θ` behaves like "equals" and `O` behaves like "less than or equal."

`R(N) ∈ O(f(N))` means there exists a positive constant `k2` such that `R(N) ≤ k2 · f(N)` for all `N` greater than some `N0`. Only the upper bound is required.

Consequence: all of the following are true simultaneously.

```
N³ + 3N⁴ ∈ Θ(N⁴)
N³ + 3N⁴ ∈ O(N⁴)
N³ + 3N⁴ ∈ O(N⁶)
N³ + 3N⁴ ∈ O(N!)
N³ + 3N⁴ ∈ O(N^(N!))
```

| | informal meaning | family | some members |
| --- | --- | --- | --- |
| `Θ(f(N))` | order of growth *is* `f(N)` | `Θ(N²)` | `N²/2`, `2N²`, `N² + 38N + N` |
| `O(f(N))` | order of growth is *at most* `f(N)` | `O(N²)` | `N²/2`, `2N²`, `lg(N)` |

Visualization example from the slides: `40 sin(N) + 4N² ∈ O(N⁴)` with `f(N) = N⁴` and `k2 = 1` (only a ceiling is needed, and a very loose ceiling is still valid). The slides promise that Big O becomes practically useful in the Disjoint Sets lecture.

---

## Definitions

- **Programming cost**: the human cost of software, that is, time to develop plus difficulty of reading, modifying, and maintaining code. Most of it is maintenance.
- **Execution cost**: the machine cost of software, that is, running time and memory usage.
- **`N`**: a chosen property of the input, usually its size (in these examples, the length of the array).
- **`R(N)`**: the runtime of a code snippet expressed as a function of `N`. Usually taken to be the *worst-case* runtime over inputs of size `N`.
- **Asymptotic behavior**: the behavior of a function for very large `N`.
- **Order of growth of `Q(N)`**: the function obtained from `Q(N)` by discarding low-order terms and discarding multiplicative constants.
- **Low-order term**: a term whose contribution becomes negligible relative to another term as `N → ∞` (for example `N²` inside `3N³ + N²`).
- **Multiplicative constant**: a constant factor multiplying a term (the `3` in `3N³`). Non-constant factors such as the `N` in `Ne^N` may *not* be dropped.
- **Big Theta**: `R(N) ∈ Θ(f(N))` iff there exist positive constants `k1`, `k2` and a threshold `N0` such that `k1·f(N) ≤ R(N) ≤ k2·f(N)` for all `N > N0`. Equivalently, `R` has order of growth `f`. `Θ(f(N))` is a *set* of functions.
- **Big O**: `R(N) ∈ O(f(N))` iff there exists a positive constant `k2` and a threshold `N0` such that `R(N) ≤ k2·f(N)` for all `N > N0`. An upper bound only. (Slides; not reached in lecture.)
- **`N0`**: the threshold past which the bounds must hold. Its particular value is meaningless; only its existence matters.
- **Cost model**: a single representative operation chosen to stand in for the whole runtime; you count only that operation, assuming runtime is proportional to its count.
- **Worst case**: the most expensive input of a given size `N`; used to pin `R(N)` down to a single well-defined function.

---

## Worked Examples

### Example 1: `countEvens` (the slide deck's running example)

```java
public static void countEvens(int[] numbers) {
    int evens = 0;
    for (int i = 0; i < numbers.length; i++) {
        if (numbers[i] % 2 == 0) {
            evens += 1;
        }
    }
    IO.println("Number of evens: " + evens);
}
```

What it does: walk the array once, bump a counter for each even element, print the total. Let `N = numbers.length`.

Full operation count table:

| operation | count |
| --- | --- |
| `evens = 0` | 1 |
| `i = 0` | 1 |
| `i < numbers.length` | `N + 1` |
| `i++` | `N` |
| `% 2` | `N` |
| `== 0` | `N` |
| `evens += 1` | `0 to N` |
| `IO.println` | 1 |

Step by step:

1. The two initializations and the final print happen once each, so they contribute constants. As `N` grows, the time to initialize `evens` and to print becomes negligible: these are low-order terms.
2. The loop guard runs `N + 1` times (one extra evaluation to discover it is false and exit). The increment, the `% 2`, and the `== 0` each run `N` times.
3. `evens += 1` is input-dependent: `0` times if no element is even, `N` times if all are. Worst case `N`.
4. Every nonconstant entry is `Θ(N)`, so under any reasonable cost model the answer is the same. Pick `==` as the cost model: count is `N`, so the **order of growth is `N`** and the runtime is in `Θ(N)`.
5. Sanity check in plain language: if `N` doubles, the runtime doubles.

The slides make the point that `evens = 0`, `i = 0`, `i < numbers.length`, `i++`, `% 2`, `==`, `evens += 1`, and `IO.println` are "all reasonable cost models" in the sense that the counting table is dominated by linear entries; the shortcut is to pick one and count it.

(Extra context on box-and-pointer reasoning: `numbers` is a local variable holding a *reference* to an array object on the heap, so each `numbers[i]` is "follow the reference, then index." This is why "array access" appears as its own row in the `dup1` table below. For asymptotics it does not change anything, since following a reference is constant time.)

### Example 2: `count1` vs `count2`, one pass or two

```java
public static void count1(int[] numbers) {
    int evens = 0;
    int odds = 0;
    for (int i = 0; i < numbers.length; i++) {
        if (numbers[i] % 2 == 0) {
            evens += 1;
        } else {
            odds += 1;
        }
    }
    IO.println("Number of evens: " + evens);
    IO.println("Number of odds: " + odds);
}
```

```java
public static void count2(int[] numbers) {
    int evens = 0;
    for (int i = 0; i < numbers.length; i++) {
        if (numbers[i] % 2 == 0) {
            evens += 1;
        }
    }
    int odds = 0;
    for (int i = 0; i < numbers.length; i++) {
        if (numbers[i] % 2 != 0) {
            odds += 1;
        }
    }
    IO.println("Number of evens: " + evens);
    IO.println("Number of odds: " + odds);
}
```

`count1` makes one pass and branches; `count2` makes two separate passes. Picking `%` as the cost model: `count1` does about `N` of them, `count2` does about `2N`. Since we ignore multiplicative constants, `2N` and `N` have the same order of growth.

**Both are `Θ(N)`. If `N` doubles, runtime doubles for both.**

This does not mean they are equally fast in wall-clock terms; `count2` plausibly does roughly twice the work. It means they *scale* identically, and scaling is what order of growth measures.

### Example 3: `countDuplicates` and the triangle

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

What it does: for each index `i`, compare `a[i]` against every *later* element `a[j]`, counting matches. It never returns early, so the count is the same for every input of size `N`.

Cost model: `==`. The intuitive question is "how many pairs do we check?"

For `N = 6`, the visited `(i, j)` cells are (the lecture's grid, reconstructed):

| `i` \ `j` | 0 | 1 | 2 | 3 | 4 | 5 |
| --- | --- | --- | --- | --- | --- | --- |
| **0** | | == | == | == | == | == |
| **1** | | | == | == | == | == |
| **2** | | | | == | == | == |
| **3** | | | | | == | == |
| **4** | | | | | | == |
| **5** | | | | | | |

Two ways to count the marks:

1. **Exact.** Row `i = 0` has `N - 1` marks, row `1` has `N - 2`, and so on down to `1` and then `0`. So the total is `1 + 2 + ... + (N - 1) = N(N - 1)/2`. (The pairing argument: there are `N` values arranged so that you get `(N - 1)/2` groups of `N`, giving `N(N-1)/2`.)
2. **The triangle shortcut.** The marks form a right triangle of width about `N` and height about `N`, so the area is about `N²/2`.

Either way, `N(N-1)/2 = (N² - N)/2`, drop the low-order `-N/2` and the constant `1/2`, and the **order of growth is `N²`**, that is, runtime `Θ(N²)`.

### Example 4: `countZerps`, when a helper is involved

```java
public static void countZerps(int[] a) {
    int zerps = 0;
    for (int i = 0; i < a.length; i++) {
        for (int j = i + 1; j < a.length; j++) {
            if (isZerp(a[i], a[j])) {
                zerps += 1;
            }
        }
    }
    IO.println("Zerps: " + zerps);
}
```

Structurally identical to `countDuplicates`, with `==` replaced by an opaque call `isZerp`. The loops visit about `N²/2` pairs, so the order of growth is `N²`, **but only under a stated assumption: `isZerp`'s runtime depends only on its two parameters and not on `N`.** If `isZerp` were itself, say, linear in `N`, the whole thing would be `Θ(N³)`.

The lesson: a method call is not automatically constant time. You must know or assume something about the callee.

### Example 5: `count`, dropping a low-order term across methods

```java
public static void count(int[] numbers) {
    countEvens(numbers);       // Theta(N)
    countDuplicates(numbers);  // Theta(N^2)
}
```

The two calls run in sequence, so the runtime is the *sum*: something like `c1·N + c2·N²`. The `c1·N` term is a low-order term, so drop it. **`count` is `Θ(N²)`.**

This is also the slides' worked instance of the formal Θ definition. With `R(N) = c1·N + c2·N²` and `f(N) = N²`:

```
c2 · N²   ≤   c1·N + c2·N²   ≤   (c1 + c2) · N²     for all N ≥ 1
```

so `k1 = c2` and `k2 = c1 + c2` witness `R(N) ∈ Θ(N²)`. The right inequality holds because `c1·N ≤ c1·N²` once `N ≥ 1`.

### Example 6: `dup1` vs `dup2`, the lecture's live example

```java
public static boolean dup1(int[] a) {
    for (int i = 0; i < a.length; i += 1) {
        for (int j = i + 1; j < a.length; j += 1) {
            if (a[i] == a[j]) {
                return true;
            }
        }
    }
    return false;
}
```

```java
public static boolean dup2(int[] a) {
    for (int i = 0; i < a.length - 1; i += 1) {
        if (a[i] == a[i + 1]) {
            return true;
        }
    }
    return false;
}
```

Both answer "does this **sorted** array contain a duplicate?" `dup1` checks all pairs; `dup2` relies on sortedness, so duplicates must be adjacent, and checks only neighbors. Both return early on the first match found.

**Step 1: count operations.** The lecture did this concretely for `N = 10,000`, stressing that you should recognize these magnitudes as plausible without needing to rederive them, and that off-by-one errors are fine.

`dup1`:

| operation | count for `N = 10,000` | symbolic |
| --- | --- | --- |
| `i = 0` | 1 | 1 |
| `j = i + 1` | 1 to 10,000 | 1 to `N` |
| `<` | 2 to 50,015,001 | 2 to `(N² + 3N + 2)/2` |
| increment (`i += 1`, `j += 1`) | 0 to 50,005,000 | 0 to `(N² + N)/2` |
| `==` | 1 to 49,995,000 | 1 to `(N² - N)/2` |
| array accesses | 2 to 99,990,000 | 2 to `N² - N` |

Reading the table: the first row is once, `j = i + 1` runs once per outer iteration so it is only linear, and everything that lives inside the inner loop is quadratic. Array accesses are twice the `==` count because `a[i] == a[j]` performs two accesses. The low ends of each range come from returning `true` immediately.

`dup2`:

| operation | count | symbolic |
| --- | --- | --- |
| `i = 0` | 1 | 1 |
| `<` | 1 to 10,000 | 1 to `N` |
| increment | 0 to 9,999 | 0 to `N - 1` |
| `==` | 1 to 9,999 | 1 to `N - 1` |
| array accesses | 2 to 19,998 | 2 to `2(N - 1)` |

Everything here grows like `N`. Again the array-access row is about `2N` because `a[i] == a[i + 1]` does two accesses.

**Step 2: collapse the ranges with the worst case.** Define `R1(N)` as the worst-case runtime of `dup1` and `R2(N)` as the worst-case runtime of `dup2`, so we take the right-hand end of each range (the no-duplicates input).

**Step 3: turn the table into a runtime.** Assign per-operation times `α, β, γ, δ, ε, ζ`. `R1(N)` becomes a polynomial such as

```
R1(N) = α + β·N + γ·(N² + 3N + 2)/2 + δ·(N² + N)/2 + ε·(N² - N)/2 + ζ·(N² - N)
```

whose largest term is a positive constant times `N²`, whatever the per-operation constants are.

**Step 4: read off the Θ class.**

```
R1(N) ∈ Θ(N²)
R2(N) ∈ Θ(N)
```

Class poll on `R1`: about 75% answered `Θ(N²)`.

**Step 5: compare.** `Θ(N)` is the better class, because less work means a faster program: better to wait about 100 units of time than about 100² units. This is the formal, machine-independent, input-robust statement that `dup2` beats `dup1`, which is exactly what we set out to produce.

**Step 6: the fast route.** You would never actually build those tables. Pick `==` as the cost model for `dup1`, note the inner loop visits a triangle of `(i, j)` pairs with legs about `N`, conclude about `N²/2` comparisons, so `Θ(N²)`. For `dup2`, `==` happens at most `N - 1` times, so `Θ(N)`. Two lines instead of two tables.

The instructor's analogy for why the tables were worth doing once: exactly modeling your commute by measuring wind, surface friction, and skateboard wheels is absurd overkill, but going through it once tells you which simplifications you are allowed to make.

### Example 7: order of growth drills (slides)

| function | order of growth | reasoning |
| --- | --- | --- |
| `N³ + 3N⁴` | `N⁴` | `N⁴` is the bigger term; drop the `3` |
| `1/N + N³` | `N³` | `1/N` shrinks toward 0 |
| `1/N + 5` | `1` | the constant `5` dominates; drop the multiplicative constant |
| `Ne^N + N` | `Ne^N` | `N` is a low-order term; you may *not* drop the leading `N` since it is not a constant |
| `40 sin(N) + 4N²` | `N²` | `sin(N)` is confined to `[-1, 1]`, so `40 sin(N)` is bounded noise |

Side note raised in lecture: these are just mathematical functions, not runtimes, which is why one of them can take negative values. Runtimes cannot.

### Example 8: the Big Theta challenge (the one time you produce `k1` and `k2`)

Let `R(N) = (4N² + 3N·ln(N)) / 2`. Find a simple `f(N)` with constants `k1`, `k2`.

Step by step:

1. Simplify: `R(N) = 2N² + 1.5·N·ln(N)`.
2. Which term dominates, `N²` or `N·ln(N)`? Cancel a factor of `N` from each and compare `N` against `ln(N)`. The natural log grows very slowly, `N` grows linearly, so `N²` wins and `N·ln(N)` is the low-order term.
3. So `f(N) = N²`.
4. The coefficient on `N²` after simplifying is `2` (the instructor flagged the `4` as a trap, since the whole expression is divided by 2). Therefore you need `k1 < 2` and `k2 > 2`.
5. Answer: `f(N) = N²`, `k1 = 1`, `k2 = 3`. So `R(N) ∈ Θ(N²)`.

Check the upper bound informally: `2N² + 1.5N·ln(N) ≤ 3N²` requires `1.5N·ln(N) ≤ N²`, which holds once `ln(N) ≤ N/1.5`, true for all large `N`. The lower bound `1·N² ≤ 2N² + 1.5N·ln(N)` holds for all `N ≥ 1`. Other valid answers exist: `k2 = 1,000,000` also works.

The instructor's framing: "this is like eating vegetables," the only time in the course you will produce explicit constants, done so that later runtime analysis rests on something rigorous.

### Example 9: from a count table straight to Θ (slides)

| operation | count |
| --- | --- |
| `<` | `100N² + 3N` |
| `>` | `2N³ + 1` |
| `&&` | `5,000` |

Even though the `<` count has a huge constant (100) and the `&&` count is 5,000, the runtime is `α(100N² + 3N) + β(2N³ + 1) + 5000γ`, and the `2βN³` term eventually dominates for any positive `α, β, γ`. So the runtime is in **`Θ(N³)`**.

---

## Common Pitfalls

- **Dropping non-constant factors.** `Ne^N` is not `e^N`. Only multiplicative *constants* go away. The reductio from lecture: if you were allowed to drop non-constant factors, `N⁴` would "simplify" to `N³`.
- **Judging by small `N`.** `100,000N` is far larger than `N²` at `N = 50`. Order of growth is a statement about large `N` only. (Real caveat from lecture: small-`N` behavior genuinely matters in practice sometimes. Naive `N³` matrix multiplication beats Strassen's asymptotically-faster algorithm on small matrices, which is why real libraries switch over at a threshold.)
- **Thinking `Θ(N)` and `Θ(2N)` are different.** They are the same set. Write `Θ(N)`. Likewise, do not write `Θ(3N³ + N²)` when `Θ(N³)` is meant.
- **Writing `=` instead of `∈`.** `Θ(f(N))` is a set of functions, so membership is the right relation.
- **Forgetting that the runtime depends on the input, not just on `N`.** Without fixing a convention (normally worst case) there is no single function `R(N)` to talk about. Say which case you mean.
- **Confusing the two-sided Θ bound with best case / worst case.** `k1·f(N)` and `k2·f(N)` are bounds on one function; best and worst case are about different *inputs*. The lecture called out this conflation as the reason "most people use Big O incorrectly."
- **Picking a cost model that is not representative.** For `dup1`, `j = i + 1` happens only `Θ(N)` times even though the runtime is `Θ(N²)`. Pick an operation from the innermost loop, and in tricky code double-check that it really tracks the total work.
- **Assuming helper calls are constant time.** `countZerps` is `Θ(N²)` only because we assume `isZerp` does not depend on `N`.
- **Counting the loop guard as `N` instead of `N + 1`.** True, but irrelevant to the order of growth. Do not lose points by being sloppy if an exam asks for an exact count, and do not waste time on it if it asks for Θ.
- **Keeping the low-order term when combining pieces.** Sequential `Θ(N)` then `Θ(N²)` code is `Θ(N²)`, not "`Θ(N + N²)`."
- **Trying to compute `k1`, `k2`, `N0` during ordinary code analysis.** Not needed. `Θ` is purely a notational replacement for the phrase "order of growth."
- **Assuming all functions have an order of growth.** The instructor noted the notion is not perfectly well defined in general ("yet people use it all the time in computer science"), and that functions of two variables (such as `N³ + K³`) open a separate can of worms deferred to later.

---

## Likely Exam Points

### 1. Give the order of growth of a mathematical function

**Q:** Give the simplest `f(N)` such that `R(N) = 7N² + 100N·lg(N) + 2^10 + 1/N ∈ Θ(f(N))`.

**A:** `f(N) = N²`. The `N·lg(N)` term is lower order than `N²` (compare `lg(N)` against `N`), `2^10` is just the constant 1024, and `1/N` tends to 0. Drop the leading `7` as a multiplicative constant.

### 2. Analyze a single loop

**Q:** Using `==` as the cost model, give the worst-case runtime of `countEvens` in Θ notation, and say what happens to the runtime when `N` doubles.

**A:** `Θ(N)`. The `==` executes `N` times; the initialization and the print are low-order constants. If `N` doubles, the runtime doubles.

### 3. Analyze nested loops where the inner bound depends on the outer index

**Q:** How many times does `a[i] == a[j]` execute in `countDuplicates` on an array of length `N`, exactly, and what is the Θ class?

**A:** Exactly `1 + 2 + ... + (N - 1) = N(N - 1)/2 = (N² - N)/2` times. Dropping the low-order `-N/2` and the constant `1/2` gives `Θ(N²)`. The quick route is the triangle of visited `(i, j)` cells: legs about `N`, so area about `N²/2`.

### 4. Sequential code: which term survives

**Q:** Method `m` calls a `Θ(N)` helper, then a `Θ(N²)` helper, then a `Θ(N)` helper, in sequence. What is `m`'s runtime?

**A:** `Θ(N²)`. Sequential costs add, and `N + N² + N` has order of growth `N²`. Both linear terms are low-order terms and get dropped.

### 5. The "hardware constants do not matter" argument

**Q:** An algorithm's operation counts are: `<` occurs `500N²` times, `>` occurs `N³/1000` times, `&&` occurs `10⁹` times. Someone claims the runtime is `Θ(N²)` because 500 is much bigger than 1/1000. Are they right?

**A:** No. The runtime is `α·500N² + β·N³/1000 + γ·10⁹` for some positive per-operation times. For large enough `N`, the cubic term dominates regardless of how small `β/1000` is and how large `α·500` is. The runtime is `Θ(N³)`. Constants never beat a higher order of growth asymptotically.

### 6. Produce `k1`, `k2`, `f(N)` from the formal definition

**Q:** State the formal definition of `R(N) ∈ Θ(f(N))`, then give `f`, `k1`, `k2` for `R(N) = 40 sin(N) + 4N²`.

**A:** `R(N) ∈ Θ(f(N))` means there exist positive constants `k1`, `k2` such that `k1·f(N) ≤ R(N) ≤ k2·f(N)` for all `N` greater than some `N0`. For this `R`: `f(N) = N²`, `k1 = 3`, `k2 = 5`, since `3N² ≤ 40 sin(N) + 4N² ≤ 5N²` once `N` is large enough (`40 sin(N)` is confined to `[-40, 40]`, so it is swamped by `N²`). Other constants work too, for example `k1 = 2`, `k2 = 1,000,000`.

### 7. Choosing a cost model

**Q:** For `dup1`, which of `i = 0`, `j = i + 1`, `<`, `==`, `return true` are valid cost models, and why?

**A:** `<` and `==` (and array accesses). They occur `Θ(N²)` times, matching the runtime's order of growth. `i = 0` and `return true` happen at most once, and `j = i + 1` happens only `Θ(N)` times, so none of those three tracks the total work.

### 8. Big O versus Big Theta (slides; deferred in lecture, but examinable)

**Q:** Which of these are true for `R(N) = N³ + 3N⁴`? (a) `R ∈ Θ(N⁴)` (b) `R ∈ O(N⁴)` (c) `R ∈ Θ(N⁶)` (d) `R ∈ O(N⁶)` (e) `R ∈ O(N!)`

**A:** (a), (b), (d), (e) are true. (c) is false: `Θ` requires a matching lower bound as well, and `N³ + 3N⁴` is not bounded below by any `k1·N⁶` for large `N`. Big O is only an upper bound, so arbitrarily loose O statements remain true.

### 9. Scaling reasoning

**Q:** Algorithm A takes `2N²` operations, B takes `500N`. Which is faster, and for which `N`?

**A:** They cross at `2N² = 500N`, that is, `N = 250`. A is faster below 250 and B is faster above. Asymptotically B is better, `Θ(N)` versus `Θ(N²)`, and the gap keeps widening, so B is the one you want for large data.

### 10. Why worst case

**Q:** On an input like `[-3, -3, 0, 7]`, `dup1` returns after one comparison. Does that mean `dup1 ∈ Θ(1)`?

**A:** No. Runtime depends on the input as well as on `N`, so `dup1`'s cost for a given `N` is a *range*. We pin it down by taking the worst case over inputs of size `N` (an array with no duplicates), giving `Θ(N²)`. Best-case behavior is `Θ(1)`, and saying so is fine as long as you label the case you mean.

---

## Summary

- Efficiency has two flavors: **programming cost** (development plus maintenance, dominated by maintenance) and **execution cost** (time and memory). From here on, execution cost.
- Raw runtimes depend on the machine and on the particular input, so we characterize code by a function `R(N)` and then by the **order of growth** of `R(N)`.
- Order of growth = **drop low-order terms** + **drop multiplicative constants**. Only *constant* factors may be dropped: `Ne^N + N → Ne^N`, not `e^N`.
- `Θ` is just notation for order of growth: `R(N) ∈ Θ(f(N))` iff there exist positive `k1`, `k2` with `k1·f(N) ≤ R(N) ≤ k2·f(N)` for all `N > N0`. Geometrically, `R` is eventually sandwiched between a small copy and a big copy of `f` forever. `Θ(f(N))` is a *set* of functions, hence `∈`.
- Using `Θ` instead of the words "order of growth" changes nothing about how you analyze code. You never produce `k1` or `k2` during code analysis.
- Pipeline for analyzing code: count operations in terms of `N`, restrict to the **worst case** to get a single function, note that for any positive per-operation constants the largest term dominates, and read off the `Θ` class. The last step is the point the instructor called extremely important.
- Counting every operation is tedious and unnecessary. Choose a **cost model**: one representative operation, typically from the innermost loop, and count only it. For `dup1`, `==` works; `j = i + 1` does not.
- The **triangle trick**: nested loops with `j` starting at `i + 1` visit about `N²/2` pairs, so `Θ(N²)`. More of this next lecture.
- Lecture results: `countEvens`, `count1`, `count2`, `dup2` are `Θ(N)`; `countDuplicates`, `countZerps` (assuming `isZerp` is independent of `N`), `count`, `dup1` are `Θ(N²)`.
- Scaling dominates everything at large `N`. At `N = 100,000`: linear is about a second, `N²` about 3 hours, `N³` about 32 years, exponential effectively never (and that is why cryptography works). Much of theoretical CS, including quantum computing, is about moving problems into a better column.
- **Big O** (on the slides, deferred in lecture) is "less than or equal to": only the upper bound `R(N) ≤ k2·f(N)` is required, so `N³ + 3N⁴` is in `O(N⁴)`, `O(N⁶)`, and `O(N!)` all at once. `Θ` behaves like "equals." Big O becomes genuinely useful at Disjoint Sets.
- Historical footnote: this notation convention for CS comes from Knuth's 1970s paper, and Big Theta was Bob Tarjan's suggestion.
