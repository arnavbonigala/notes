<!-- Fri, Oct 02, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 16: Asymptotics III

## Overview

This lecture closes out the asymptotics unit by applying the counting techniques from Lectures 14 and 15 to **recursive** code, and then clarifying the relationship between Big O and Big Theta. We analyze three recursive programs of increasing importance: a toy exponential recursion `f3` (runtime Θ(2^N)), binary search (runtime Θ(log N) in the worst case), and mergesort (runtime Θ(N log N)). Along the way we see three distinct analysis strategies (intuition about how work changes as N grows, exact counting of operations with a cost model, and recurrence relations, the last of which is explicitly out of scope for 61B), plus three handy simplification properties: floors, ceilings, and logarithm bases never change an order of growth. The lecture ends with a careful treatment of a subtle point: Big O is *not* a synonym for "worst case." Big O is an upper bound (like ≤) and Big Theta is an exact order of growth (like =), so "Θ(N) in the worst case" is a strictly more informative statement than "O(N)." The overarching message, repeated from last time, is that there are no magic shortcuts: runtime analysis requires careful thought, and in this course exact counting or intuitive analysis is enough (no formal proofs required).

---

## Key Concepts

### 1. The two sums you must know (carried over from Lecture 15)

Essentially every counting problem in this unit collapses into one of these:

| Sum | Closed form | Order of growth |
| --- | --- | --- |
| 1 + 2 + 3 + ... + Q | Q(Q+1)/2 | Θ(Q²) |
| 1 + 2 + 4 + 8 + ... + Q | 2Q - 1 | Θ(Q) |

The second one is the one people get wrong. Read it literally: **Q is the last term of the sum, not the number of terms.** The sum of a doubling sequence is dominated by its final term, so it is only about twice as big as that final term. This is why a recursion that doubles its work at each level is dominated by its bottom level, while a recursion that does equal work at each level (mergesort) is not.

### 2. Three strategies for analyzing recursion

The lecture presents the same problem (`f3`) three different ways, which is the whole point: pick whichever is easiest for the problem in front of you.

**Approach 1, intuitive.** Ask: "what happens to the total work when I bump N up by 1 (or double N)?" For `f3`, increasing N by 1 doubles the work, which is exactly the signature of Θ(2^N). This approach is fast, and it is legitimate in 61B.

**Approach 2, exact counting.** Pick a **cost model** (some operation whose count is proportional to the runtime), draw the call tree, count how many of those operations occur at each level, sum the levels with one of the two known sums, and then convert to Big Theta. This is the workhorse method.

**Approach 3, recurrence relations.** Write C(1) = 1, C(N) = 2C(N-1) + 1 and solve the recurrence. The lecture explicitly marks this as **out of scope for CS 61B** and shows it only so you recognize what it is. You will never be required to solve a recurrence on a 61B exam.

### 3. Call trees vs. the call stack

For `f3(4)`, the call tree is a complete binary tree: one call at the top, two below it, four below those, eight leaves. Reading the tree *horizontally* (level by level) gives you the sum 1 + 2 + 4 + 8. Reading it *vertically* (root to leaf) gives you the recursion depth, N.

(extra context) In environment/box-and-pointer terms: each call to `f3` pushes a new stack frame holding its own `n` parameter, and that frame pops as soon as the call returns. So although 2^N - 1 frames are *created* over the life of the computation, at most N of them exist *simultaneously*, because the left `f3(n-1)` fully finishes and pops before the right `f3(n-1)` is even pushed. Exponential *time*, linear *space*. The lecture measures time, not space, but confusing the two is a classic error.

### 4. Binary search: halving the problem

Binary search looks at the middle element and throws away half of the remaining range. The sequence of problem sizes is N, ≈N/2, ≈N/4, ≈N/8, ... and the recursion stops when the size reaches 1. So if C is the number of calls, we solve 1 = N/2^C, giving C = log₂(N). That is the intuitive argument, and it is sufficient.

The lecture emphasizes that binary search is deceptively hard to get right in practice: the idea was published in 1946, the first correct implementation appeared in 1962, and a bug in **Java's own** binary search was discovered in 2006. (extra context) That 2006 bug was integer overflow in `(lo + hi) / 2` when `lo + hi` exceeds `Integer.MAX_VALUE`; the fix is `lo + (hi - lo) / 2`. The lecture points to Jon Bentley's *Programming Pearls* for the story.

### 5. Logarithmic time is practically constant

| N | log₂ N | Typical runtime |
| --- | --- | --- |
| 100 | 6.6 | 1 ns |
| 100,000 | 16.6 | 2.5 ns |
| 100,000,000 | 26.5 | 4 ns |
| 100,000,000,000 | 36.5 | 5.5 ns |
| 100,000,000,000,000 | 46.5 | 7 ns |

A trillion-fold increase in N costs you a factor of 7 in time. For all practical purposes, a Θ(log N) algorithm is a constant-time algorithm.

### 6. Three Big Theta simplification properties

Once you have an exact count like ⌊log₂(N)⌋ + 1, you simplify using:

- ⌊f(N)⌋ = Θ(f(N))  (floors don't matter)
- ⌈f(N)⌉ = Θ(f(N))  (ceilings don't matter)
- log_P(N) = Θ(log_Q(N))  (**the base of a logarithm does not matter**)

The base is irrelevant because changing base multiplies by a constant: log_P(N) = log_Q(N) / log_Q(P). That is why we write just **Θ(log N)**, with no base and (by convention in this course) no parentheses around N.

### 7. Building up to mergesort

The mergesort story is told as a sequence of improvements, which is a much better way to remember *why* N log N beats N²:

- **Selection sort** is Θ(N²). Scan all N unfixed items to find the smallest, fix it, then scan the remaining N-1, and so on: N + (N-1) + ... + 2, which is the first known sum, Θ(N²).
- **Merge** of two sorted arrays into one is Θ(N) where N is the total number of items. Cost model: array writes. Merging does exactly N writes, one per item.
- **One layer of merging:** selection sort each half, then merge. Still Θ(N²), because N + 2(N/2)² is still quadratic, but with a much smaller constant: N + 2(N/2)² < N².
- **Mergesort:** stop using selection sort entirely and merge all the way down.

The concrete arithmetic the lecture uses, in "arbitrary units" (AU), with N = 64 and selection sort costing N² AU:

| Strategy | Cost at N = 64 |
| --- | --- |
| Selection sort only | 64² = ~4096 AU |
| One layer of merges | 64 + 2·32² = 64 + 2048 = ~2112 AU |
| Two layers of merges | (64 + 2·32) + 4·16² = 128 + 1024 = ~1152 AU |
| Merge all the way down (mergesort) | 64 · 6 = ~384 AU |

### 8. Why mergesort is Θ(N log N)

Two independent observations multiply together:

- **Each level does Θ(N) work.** The top level merges N items. The next level does two merges of N/2 items each, totaling N. The next does four merges of N/4 items, totaling N. Every level touches every item exactly once.
- **There are Θ(log N) levels.** Levels go N, N/2, N/4, ... until size 1, so there are log₂ N of them.

Total: ~N per level × log₂ N levels = Θ(N log N). The slides note that the exact count is tedious and omit it, deferring to the textbook. The recurrence-relation version (counting calls plus array writes) is marked **Extra**, and the slides observe that the clean recurrence only works when N = 2^k, and generalizing requires separately finding Big O and Big Omega bounds.

### 9. Linear vs. linearithmic vs. quadratic

N log N is "basically as good as N" and "vastly better than N²." For N = 1,000,000, log₂ N is only about 20, so N log N is only 20× worse than N, while N² is 1,000,000× worse. The practical lesson stated in the summary: **going from N² to N log N is an enormous win; going from N log N to N is nice but not a radical change.**

### 10. Big O vs. Big Theta

This is the conceptual punchline of the lecture. The analogy given: *if we have `=`, why do we also have `≤`?*

- **Θ is like `=`** (exact order of growth, both an upper and a lower bound).
- **O is like `≤`** (upper bound only).

The running example is quick union tree height. A quick union tree with N connected items can have height anywhere from 1 (a flat tree, where all items point directly at one root) to N - 1 (a spindly tree, a single chain). So:

- Height is **Θ(1) in the best case** (flat).
- Height is **Θ(N) in the worst case** (spindly). This is why `isConnected(0, N-1)` can take linear time.
- Height is **O(N)**: every tree's height is linear or better. True.
- Height is **O(N²)**: every tree's height is quadratic or better. Also true, just weak.

**All of these statements are true simultaneously.** The two middle ones are the informative ones.

The hotel analogy makes the asymmetry vivid: "the most expensive room in the hotel is $639 per night" (a Θ-style statement about the worst case) tells you more than "every room is at most $639 per night" (an O-style statement), because the latter is also true if every room costs $1.

**Big O is NOT mathematically the same thing as "worst case."** Worst case is a statement about *which input* you consider; Big O is a statement about *which kind of bound* you are giving. You can describe the best case with Big O, and you can describe the worst case with Big Theta. It is more precise to say "Θ(N) in the worst case" than "O(N)." That said, the lecture acknowledges Big O is very often used as shorthand for worst case in practice, including by textbooks and interviewers.

### 11. Why Big O is still useful

1. **Blanket statements.** "Binary search is O(log N)" is shorter than "binary search is Θ(log N) in the worst case" and covers every case at once.
2. **When you genuinely don't know the exact runtime.** The best known algorithm for finding the shortest route visiting all world cities (the traveling salesman problem) is O(2^N) under certain assumptions. Nobody knows whether something faster exists, so an upper bound is all we can honestly claim.
3. **Proofs are easier.** For mergesort, a Big O proof lets you round N up to the next power of 2; a tight Big Theta proof is fussier. (Beyond the scope of 61B.)
4. **O plus Ω gives you Θ.** If f is O(N²) and also Ω(N²), then f is Θ(N²), exactly as a ≤ b and b ≤ a give a = b.

---

## Definitions

- **Cost model:** A chosen operation (array writes, recursive calls, comparisons) whose count is used as a proxy for runtime. Valid when the total runtime is proportional to that count, which requires each occurrence to take constant time.
- **R(N):** The runtime of a piece of code as a function of input size N.
- **C(N):** The count of cost-model operations for input size N. If each costs constant time, R(N) = Θ(C(N)).
- **Call tree:** The tree of recursive invocations, where each node's children are the calls it makes directly. Levels correspond to recursion depth; the total node count is the total number of calls.
- **Recurrence relation:** A definition of C(N) in terms of C of smaller inputs, plus base cases. Example: C(1) = 1, C(N) = 2C(N-1) + 1. **Out of scope for CS 61B.**
- **Binary search:** An algorithm for locating a key in a *sorted* array by repeatedly comparing against the middle element and recursing into the half that could contain the key. Worst case Θ(log N).
- **Selection sort:** Repeatedly find the smallest unfixed item, move it to the front, fix it in place, and sort the remainder the same way. Θ(N²).
- **Merge:** Given two sorted arrays, produce a single sorted array by repeatedly copying the smaller of the two current front items into a target array. Θ(N) in the total number of items, using array writes as the cost model (exactly N writes).
- **Mergesort:** If the array has size 1, return. Otherwise mergesort the left half, mergesort the right half, and merge the two sorted results. Θ(N log N) in the worst case.
- **Linearithmic:** The order of growth N log N.
- **Big O, O(f(N)):** An upper bound on order of growth. "R(N) is O(f(N))" means R grows no faster than f, up to constants. Behaves like ≤.
- **Big Omega, Ω(f(N)):** A lower bound on order of growth. Behaves like ≥.
- **Big Theta, Θ(f(N)):** An exact order of growth: both O(f(N)) and Ω(f(N)). Behaves like =.
- **Flat tree:** A quick union tree of height 1 where all non-root items point directly at the root. Height Θ(1).
- **Spindly tree:** A quick union tree that is a single chain, of height N - 1. Height Θ(N).
- **Floor/ceiling/log-base properties:** ⌊f(N)⌋ = Θ(f(N)), ⌈f(N)⌉ = Θ(f(N)), and log_P(N) = Θ(log_Q(N)).

---

## Worked Examples

### Example 1: The toy recursion `f3`

```java
public static int f3(int n) {
    if (n <= 1)
        return 1;
    return f3(n - 1) + f3(n - 1);
}
```

**What the code does.** For n ≤ 1 it returns 1 immediately. Otherwise it computes `f3(n-1)` twice and adds the results. Note that it recomputes the *identical* subproblem twice rather than storing it, which is precisely why it is slow. (The returned *value* is 2^(n-1), but we care about the runtime, not the value.)

**Approach 1: intuition.** Compare the work for n and n+1. The call `f3(n+1)` does a constant amount of its own work and then performs the entire computation of `f3(n)` twice. So every time N increases by 1, the work doubles. A quantity that doubles with each unit increase in N is exponential: **R(N) ∈ Θ(2^N)**.

**Approach 2: exact counting.** Cost model: number of calls to `f3`, call it C(N). (The lecture notes that if counting calls feels odd, you can instead count executions of the `n <= 1` test, which yields the same C(N), since every call performs exactly one such test.)

Draw the call tree and count by level:

- `f3(1)`: 1 call total.
- `f3(2)`: 1 + 2 = 3 calls (itself plus two `f3(1)`s).
- `f3(3)`: 1 + 2 + 4 = 7 calls.
- `f3(4)`: 1 + 2 + 4 + 8 = 15 calls.

In general the tree is a complete binary tree of depth N, so

C(N) = 1 + 2 + 4 + ... + 2^(N-1).

**The key step is identifying Q.** In the formula 1 + 2 + 4 + ... + Q = 2Q - 1, the symbol Q is the *final term*, which here is 2^(N-1), **not** N. Substituting:

C(N) = 2 · 2^(N-1) - 1 = 2^N - 1.

Each call does a constant amount of work outside of its recursive calls (one comparison, one addition, one return), so

**R(N) = Θ(2^N).**

Sanity check against the table: N = 4 gives 2⁴ - 1 = 15 calls, matching the hand count.

**Approach 3: recurrence relation (out of scope, shown for recognition only).**

```
C(1) = 1
C(N) = 2C(N-1) + 1
```

Read this as: the cost of `f3(n)` is the cost of two copies of `f3(n-1)` plus 1 for the call itself. Solving it yields C(N) = 2^N - 1, the same answer. **You will not be asked to solve recurrences in 61B.**

**Environment / stack reasoning (extra context).** When `f3(4)` runs, the JVM pushes a frame for `f3(4)`, which pushes a frame for the first `f3(3)`, which pushes `f3(2)`, which pushes `f3(1)`. At that instant the stack holds four frames, each with its own `n` box. `f3(1)` returns 1 and its frame pops; then the second `f3(1)` is pushed into the space just vacated. Maximum simultaneous depth is N = 4, even though 15 frames are created in total over the whole execution.

### Example 2: Binary search

```java
static int binarySearch(String[] sorted, String x, int lo, int hi) {
    if (lo > hi) return -1;
    int m = (lo + hi) / 2;
    int cmp = x.compareTo(sorted[m]);
    if (cmp < 0) return binarySearch(sorted, x, lo, m - 1);
    else if (cmp > 0) return binarySearch(sorted, x, m + 1, hi);
    else return m;
}
```

**What the code does.** `lo` and `hi` are inclusive indices bounding the region still under consideration, so the problem size is **N = hi - lo + 1**. If the region is empty (`lo > hi`), the key is absent and we return -1. Otherwise we compute the midpoint `m` and compare `x` against `sorted[m]` using `compareTo` (negative if `x` comes first, positive if `x` comes later, zero if equal). If `x` is smaller we recurse on `[lo, m-1]`; if larger, on `[m+1, hi]`; if equal, we return the index `m`.

(extra context) Note that this does not copy the array: all recursive calls share a single reference to the same underlying `String[]` object, and only the two `int` bounds change from frame to frame. That is why no extra array work shows up in the analysis.

**Intuitive analysis.** Each call eliminates the midpoint and one of the two halves, so the sizes go N, ≈N/2, ≈N/4, ≈N/8, ..., down to 1. If C is the number of calls, we need 1 = N/2^C, so C = log₂(N). Each call does constant work (one comparison, a few arithmetic ops), so the worst case runtime is **Θ(log N)**.

**Exact count (the bonus/pre-recorded section, in scope).** Cost model: number of `binarySearch` calls, worst case over all keys `x`.

Trace N = 6 with indices `lo = 0, hi = 5`:
1. Call with N = 6. `m = (0+5)/2 = 2`. Worst case we recurse right: `lo = 3, hi = 5`.
2. Call with N = 3. `m = (3+5)/2 = 4`. Worst case we recurse into a side of size 1.
3. Call with N = 1.

So **C(6) = 3**, with problem sizes 6, 3, 1. Filling in the whole table:

| N | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| C(N) | 1 | 2 | 2 | 3 | 3 | 3 | 3 | 4 | 4 | 4 | 4 | 4 | 4 |

The pattern is unmistakable: C jumps by 1 exactly at the powers of 2 (N = 1, 2, 4, 8, ...). That is the signature of a floor-of-log, and the closed form is

**C(N) = ⌊log₂(N)⌋ + 1.**

Since each call takes constant time, R(N) = Θ(⌊log₂(N)⌋). This f(N) is far too complicated to be a useful answer, so we simplify with the three properties: drop the `+1` (lower-order term), drop the floor (⌊f⌋ = Θ(f)), and drop the base (log₂ = Θ(log)):

**R(N) = Θ(log N).**

**Recurrence version (out of scope).** Measuring string comparisons: C(0) = 0, C(1) = 1, C(N) = 1 + C((N-1)/2). This can be shown to be Θ(log N), but we do not solve it in this course.

### Example 3: Selection sort

```java
// Sketch of the selection sort discussed earlier in the course.
public static void selectionSort(int[] a) {
    for (int i = 0; i < a.length; i += 1) {
        int smallestIndex = i;
        for (int j = i + 1; j < a.length; j += 1) {   // scan the unfixed region
            if (a[j] < a[smallestIndex]) {
                smallestIndex = j;
            }
        }
        swap(a, i, smallestIndex);                     // fix a[i] in place
    }
}
```

**What the code does.** The array is conceptually split into a fixed prefix `a[0..i-1]` (already in final sorted position) and an unfixed suffix `a[i..]`. The inner loop scans the unfixed suffix to find the smallest element, which is then swapped to the front of the suffix and declared fixed. Running on `6 3 7 2 8 1`, the lecture's trace is:

```
6 3 7 2 8 1   (start, N = 6)
1 3 7 2 8 6   (1 fixed)
1 2 7 3 8 6   (2 fixed)
1 2 3 7 8 6   (3 fixed)
1 2 3 6 8 7   (6 fixed)
...
```

**Analysis.** Cost model: comparisons / items examined.
- First pass examines all N unfixed items.
- Second pass examines the remaining N - 1.
- ... and so on, down to the last pass examining 2 items (once a single item remains, there is nothing left to do).

Total = 2 + 3 + 4 + ... + N. This is the first known sum (missing only the `1` term, which changes nothing), so the total is Θ(N²):

**R(N) = Θ(N²).**

For the AU accounting that follows, read this as "N = 64 costs about 64² = 4096 arbitrary units."

### Example 4: Merge

```java
// Merge two sorted halves of `a` into `target`. (Code written out here;
// the lecture presented merge via a demo rather than as source.) (extra context)
private static void merge(int[] left, int[] right, int[] target) {
    int i = 0, j = 0, k = 0;
    while (i < left.length && j < right.length) {
        if (left[i] <= right[j]) {
            target[k] = left[i];
            i += 1;
        } else {
            target[k] = right[j];
            j += 1;
        }
        k += 1;
    }
    while (i < left.length)  { target[k] = left[i];  i += 1; k += 1; }
    while (j < right.length) { target[k] = right[j]; j += 1; k += 1; }
}
```

**What the code does.** Three pointers walk forward: `i` into the left sorted array, `j` into the right, `k` into the output. At each step we copy whichever of `left[i]` and `right[j]` is smaller and advance that pointer. When one input is exhausted, the leftovers from the other are copied straight across. The lecture's example:

```
left:  2  3  6  10  11
right: 4  5  7  8
--------------------------------
out:   2  3  4  5  6  7  8  10  11
```

Step by step: compare 2 vs 4, write 2. Compare 3 vs 4, write 3. Compare 6 vs 4, write 4. Compare 6 vs 5, write 5. Compare 6 vs 7, write 6. Compare 10 vs 7, write 7. Compare 10 vs 8, write 8. Right side exhausted, copy 10 and 11.

**Analysis.** Cost model: array writes into `target`. Every one of the N total items is written exactly once, and nothing is ever written twice, so C(N) = N exactly, regardless of the data. **R(N) = Θ(N).**

This is the crucial fact: merging is *linear*, not quadratic, which is what makes divide-and-conquer pay off.

### Example 5: Mergesort, and why merging beats sorting

Take N = 64 and assume selection sort costs N² AU and merge costs N AU.

**Step 0, selection sort alone.** 64² = **~4096 AU**.

**Step 1, one layer of merges.** Selection sort the left 32, selection sort the right 32, merge the results.
- Two selection sorts: 2 × 32² = 2 × 1024 = 2048 AU.
- One merge of 64 items: 64 AU.
- Total: **~2112 AU**, roughly half the cost.

Note this is *still* Θ(N²): N + 2(N/2)² = N + N²/2 is quadratic. We only improved the constant. But the improvement is real, and it suggests doing it again.

**Step 2, two layers of merges.** Selection sort four blocks of 16, merge into two blocks of 32, merge into one block of 64.
- Four selection sorts: 4 × 16² = 4 × 256 = 1024 AU.
- Two merges of 32: 2 × 32 = 64 AU.
- One merge of 64: 64 AU.
- Total: **~1152 AU**.

**Step 3, mergesort (merge all the way down).**

```java
public static void mergeSort(int[] a) {
    if (a.length <= 1) return;          // base case: size 1 is already sorted
    int[] left  = Arrays.copyOfRange(a, 0, a.length / 2);
    int[] right = Arrays.copyOfRange(a, a.length / 2, a.length);
    mergeSort(left);                    // sort the left half
    mergeSort(right);                   // sort the right half
    merge(left, right, a);              // Theta(N) combine
}
```
*(Java code written out here; the lecture describes mergesort in prose and pictures rather than source. (extra context))*

Now there is no selection sort at all. Every level of the recursion tree costs about 64 AU:
- Top layer: one merge of 64 = 64 AU.
- Second layer: two merges of 32 = 64 AU.
- Third layer: four merges of 16 = 64 AU.
- ... and so on.

Total = 64 × k, where k is the number of layers. Since k = log₂(64) = 6, the total is **~384 AU**, more than ten times faster than plain selection sort.

**Generalizing to arbitrary N.** The recursion tree has sizes N at the top, then N/2 and N/2, then four of N/4, then eight of N/8, and so on.
- **Work per level is Θ(N):** at level d there are 2^d merges of N/2^d items each, and 2^d × (N/2^d) = N. Every item is touched exactly once per level.
- **Number of levels is log₂ N:** we halve until we reach size 1, which takes log₂ N halvings.

Total runtime = (work per level) × (number of levels) = **Θ(N log N)**.

The lecture notes the exact count is tedious and omits it (see the textbook, section 14.3). The recurrence-relation version counts calls plus array writes and strictly speaking only works cleanly when N = 2^k; the general case requires bounding Big O and Big Omega separately.

### Example 6: Quick union tree height, Big O vs. Big Theta

Two extreme quick union trees with N = 5:

```
Flat (H = 1):            Spindly (H = 4):
       0                    0
    / / \ \                  \
   1 2   3 4                  1
                               \
                                2
                                 \
                                  3
                                   \
                                    4
```

- The flat tree's height does not grow with N at all: **H = Θ(1)**.
- The spindly tree's height is N - 1: **H = Θ(N)**.

Now evaluate the three statements from the lecture:

| Statement | True? | Why |
| --- | --- | --- |
| Worst case quick union tree height is Θ(N) | ✓ | A spindly tree's height grows *exactly* linearly ("is equal to linear") |
| Quick union tree height is O(N) | ✓ | Every tree's height is linear or better ("is less than or equal to linear") |
| Quick union tree height is O(N²) | ✓ | Every tree's height is quadratic or better. True but weak |

All three are true. Statements 1 and 2 are the informative ones; statement 3 wastes most of what you could have said. And we can add a fourth true statement: height is **Θ(1) in the best case**.

The practical consequence: `isConnected(0, N-1)` on a spindly quick union tree takes linear time, which is exactly the motivation for weighting and path compression.

---

## Common Pitfalls

1. **Misreading Q in the powers-of-2 sum.** 1 + 2 + 4 + ... + Q = 2Q - 1 where **Q is the final term**. For `f3`, the final term is 2^(N-1), not N. Plugging in N gives the wrong answer 2N - 1 instead of 2^N - 1. Always write down what Q is before substituting.

2. **Thinking "Big O" means "worst case."** These are orthogonal. Big O is a *type of bound* (upper); worst case is a *choice of input*. You can say "Θ(1) in the best case" and you can say "O(N²)" about an algorithm whose worst case is Θ(N). Prefer "Θ(f(N)) in the worst case" when you know it.

3. **Thinking an O statement is "wrong" because it isn't tight.** "Quick union tree height is O(N²)" is perfectly true. It is just uninformative. On exams, a question asking "which of these are true?" is different from "which is most informative?"

4. **Keeping the log base, the floor, or the +1 in a Theta answer.** Θ(⌊log₂(N)⌋ + 1) should be written Θ(log N). Conversely, do not *start* by discarding these: get the exact count first, then simplify.

5. **Confusing recursion depth with total call count.** `f3` has depth N but 2^N - 1 total calls. Mergesort has depth log N but does Θ(N log N) total work. The number of levels alone never gives you the runtime; you need work-per-level too.

6. **Thinking "divide into halves" automatically means log N.** Binary search is log N because it *discards* one half. Mergesort is N log N because it *processes both* halves and then does Θ(N) work to combine them. The number of subproblems you recurse into matters enormously.

7. **Assuming one layer of merging changes the order of growth.** Selection sort both halves plus a merge is still Θ(N²). It is roughly 2× faster at N = 64, but the asymptotic class is unchanged. Only merging all the way down gets you to Θ(N log N).

8. **Forgetting that binary search requires a sorted array.** The Θ(log N) bound is meaningless otherwise.

9. **Trying to use recurrence relations on an exam.** They are explicitly out of scope. If you reach for one and get stuck, you have burned time on a technique you were never required to know. Use intuition or exact counting.

10. **Assuming the cost model is free to choose.** It must be proportional to actual runtime. Counting `binarySearch` calls works because each call does constant work. Counting calls to a method that itself loops over the array would not.

11. **Writing R(N) = O(...) when the question asks for a simple f(N) with R(N) ∈ Θ(f(N)).** Read what is asked for.

---

## Likely Exam Points

### 1. Order of growth of an exponential recursion

**Q.** Give a simple f(N) such that the runtime of the following is Θ(f(N)).
```java
public static int g(int n) {
    if (n <= 1) return 1;
    return g(n - 1) + g(n - 1) + g(n - 1);
}
```
**A.** Each increase of N by 1 triples the work, so the call tree is a complete *ternary* tree of depth N. Counting calls: C(N) = 1 + 3 + 9 + ... + 3^(N-1) = (3^N - 1)/2. The runtime is **Θ(3^N)**. The structural lesson: the branching factor is the base of the exponent. (Compare `f3`, where branching factor 2 gives Θ(2^N).)

### 2. Exact count for binary search

**Q.** Using "number of calls to `binarySearch`" as the cost model, what is C(N) in the worst case, and what is the order of growth? What is C(12)?

**A.** C(N) = ⌊log₂(N)⌋ + 1, which is Θ(log N). For N = 12: ⌊log₂(12)⌋ = ⌊3.58⌋ = 3, so C(12) = 4, matching the table. Each call is constant time, so R(N) = **Θ(log N)**.

### 3. Simplification properties

**Q.** Simplify each: (a) Θ(⌈N/3⌉), (b) Θ(log₁₀ N), (c) Θ(3⌊log₂ N⌋ + 7).

**A.** (a) Θ(N), since ceilings and constant factors do not matter. (b) Θ(log N), since the log base is irrelevant. (c) Θ(log N): drop the floor, the constant multiplier 3, and the lower-order additive 7.

### 4. Which statements are true vs. which is most informative

**Q.** A data structure's operation takes 1 step in the best case and grows linearly in the worst case. Which of the following are true: (i) Θ(N) in the worst case, (ii) O(N), (iii) O(N³), (iv) Θ(N), (v) Ω(1)? Which is most informative?

**A.** True: (i), (ii), (iii), (v). **(iv) is false**: Θ(N) with no qualifier claims *every* case grows linearly, but the best case is constant, so no single Θ describes the whole runtime. (i) is the most informative, because it pins down an exact order of growth for a specified case. Note (iii) and (v) are true but nearly vacuous.

### 5. Mergesort runtime derivation

**Q.** Explain in two sentences why mergesort is Θ(N log N), and state the merge cost model.

**A.** Each level of the recursion does Θ(N) total work, because the 2^d merges at level d each handle N/2^d items and 2^d · (N/2^d) = N, so every item is touched exactly once per level. There are log₂ N levels, since halving N repeatedly reaches size 1 after log₂ N steps, giving N · log N total. The merge cost model is **array writes**: merging two sorted sequences totaling N items performs exactly N writes, hence Θ(N).

### 6. The AU arithmetic (hybrid selection sort / merge)

**Q.** With selection sort costing N² AU and merge costing N AU, compute the cost of sorting N = 64 using exactly one layer of merging, and state its order of growth.

**A.** Two selection sorts of size 32 cost 2 × 32² = 2048 AU, plus one merge of 64 items at 64 AU, for **~2112 AU** (versus 4096 for plain selection sort). The order of growth is still **Θ(N²)**, since N + 2(N/2)² = N + N²/2; only the constant improved.

### 7. Identifying the right sum

**Q.** Which of the two known sums applies, and what is the order of growth?
(a) Total work is 1 + 2 + 3 + ... + N.  (b) Total work is 1 + 2 + 4 + ... + N.

**A.** (a) Sum of first natural numbers: N(N+1)/2 = **Θ(N²)**. (b) Sum of first powers of 2 with Q = N: 2N - 1 = **Θ(N)**. Same-looking notation, wildly different answers. In (b) the last term dominates everything before it.

### 8. Recursion depth vs. total work

**Q.** True or false: because `f3(n)` recurses only to depth N, its runtime is Θ(N).

**A.** **False.** Depth is N, but the call tree branches twice at every node, so the total number of calls is 2^N - 1 and the runtime is Θ(2^N). Depth bounds the *stack space*, not the time. (The Θ(N) space claim would be correct. (extra context))

### 9. Comparing growth rates in practice

**Q.** For N = 1,000,000, roughly how much worse is N log N than N? How much worse is N² than N?

**A.** log₂(1,000,000) ≈ 20, so N log N is about **20× worse** than N, while N² is about **1,000,000× worse**. This is the lecture's closing point: the N² to N log N jump is transformative; the N log N to N jump is a modest tune-up.

---

## Summary

- **No shortcuts.** Asymptotic analysis requires careful thought. In CS 61B, intuitive reasoning or exact counting is acceptable; you will not write mathematical proofs.
- **Know two sums:** 1 + 2 + ... + Q = Q(Q+1)/2 = Θ(Q²), and 1 + 2 + 4 + ... + Q = 2Q - 1 = Θ(Q). Q is the *last term*.
- **Three approaches to recursion:** intuition (how does work change as N grows by 1 or doubles?), exact counting with a cost model and a call tree, and recurrence relations (**out of scope**).
- **`f3`** doubles work with each increment of N; C(N) = 1 + 2 + ... + 2^(N-1) = 2^N - 1, so **R(N) = Θ(2^N)**. Depth is only N, so space is Θ(N). (extra context, for the space claim)
- **Binary search** halves the problem each call; solving 1 = N/2^C gives C = log₂ N. Exactly, C(N) = ⌊log₂(N)⌋ + 1, so **R(N) = Θ(log N)** in the worst case. Requires a sorted array, and is notoriously easy to implement incorrectly.
- **Simplify with three properties:** ⌊f⌋ = Θ(f), ⌈f⌉ = Θ(f), and log_P N = Θ(log_Q N). Write Θ(log N), no base.
- **Logarithmic time is effectively constant** in practice: 7 ns at N = 10¹⁴.
- **Selection sort is Θ(N²)** (sum 2 + 3 + ... + N). **Merge is Θ(N)** (exactly N array writes).
- **Mergesort is Θ(N log N):** Θ(N) work per level × log₂ N levels. At N = 64: ~384 AU vs. ~4096 AU for selection sort.
- Hybrid approaches (selection sort the halves, then merge) improve the *constant* but remain Θ(N²). Only merging all the way down changes the order of growth.
- **Θ is like `=`; O is like `≤`.** A quick union tree's height is Θ(1) best case, Θ(N) worst case, O(N), and O(N²): all true at once, with the first two most informative.
- **Big O is not a synonym for worst case**, though it is widely used that way. "Θ(N) in the worst case" is strictly more precise than "O(N)."
- **Big O is still useful:** blanket statements, honest upper bounds when the true runtime is unknown (for example, O(2^N) for the traveling salesman problem), easier proofs, and O + Ω = Θ.
- **Practical scale:** N² to N log N is an enormous improvement; N log N to N is pleasant but minor.
