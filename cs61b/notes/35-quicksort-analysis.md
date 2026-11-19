<!-- Wed, Nov 18, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 35: Quicksort Analysis

Quicksort's runtime depends entirely on where the pivot lands after partitioning. If the pivot always lands at an edge (as happens when you pick the leftmost item on a sorted array), you get N levels of recursion with Θ(N) work each, giving Θ(N²). If it lands near the middle, you get Θ(log N) levels with Θ(N) work each, giving Θ(N log N). The interesting result of this lecture is that "near the middle" is a very forgiving requirement: even a pivot that always lands at least 10% from either edge still yields O(N log N), and a randomly ordered array gives Θ(N log N) expected time with extremely high probability. This lecture develops the two intuitive arguments for average-case Θ(N log N) (the 10% argument and the "Quicksort is BST sort" equivalence), surveys four philosophies for avoiding the worst case (randomness, smarter pivot selection, introspection, preprocessing), shows empirically that Quicksort beats Mergesort only with a good partitioning scheme (Tony Hoare's two-pointer scheme: 0.9s vs Mergesort's 1.3s, while a naive 3-scan partition loses at 3.2s), and closes with Quickselect, a Θ(N) expected-time median-finding algorithm built from partitioning, which turns out to be a bad way to pick Quicksort pivots in practice.

## Key Concepts

### The warm-up: best and worst pivots

Given the input array `32 15 2 17 19 26 41 17 17`, the lecture asks which element would be the worst pivot and which the best.

- **Worst pivot**: an extreme value. Picking `2` (the minimum) means partitioning puts it at index 0 and leaves all 8 remaining items on one side. You did Θ(N) work and reduced the problem size by exactly 1. Picking `41` (the maximum) is equally bad in the other direction.
- **Best pivot**: the median. Sorting the array mentally gives `2 15 17 17 17 19 26 32 41`, so the median is `17`. Picking a `17` splits the remaining items into two roughly equal halves (4 and 4), which is the optimal split.

The entire lecture is an elaboration of this one observation.

### Worst case: pivot always lands at the beginning

Suppose the pivot is always the smallest remaining item. The lecture traces this on the warm-up array:

```
N     : 32  15   2  17  19  26  41  17  17      (pivot 32? no, see below)
N-1   :  2  32  15  17  19  26  41  17  17      (2 in its place)
N-2   :  2  15  32  17  19  26  41  17  17      (15 in its place)
N-3   :  2  15  17  32  19  26  41  17  17      (17 in its place)
...
1     :  2  15  17  17  17  19  26  32  41
```

Each level places exactly one item in its final position and recurses on everything else. Work per level: N, then N-1, then N-2, down to 1.

Total: N + (N-1) + (N-2) + ... + 2 + 1 = N(N+1)/2 = **Θ(N²)**.

Note the space cost too: the recursion depth is N in this case, not log N, so the call stack is Θ(N) in the worst case even though the summary table lists Θ(log N) as the typical/expected space.

**Which inputs cause this?** With the lecture's pivot scheme (always use the leftmost item), the answer is an already-sorted array. The slide gives `123456` as the example. On `[1 2 3 4 5 6]`, the leftmost item `1` is also the minimum, so it lands at index 0, and the right subproblem `[2 3 4 5 6]` has the same structure. Almost-sorted arrays are nearly as bad. Reverse-sorted arrays are also bad for the same reason (the leftmost item is the maximum, so it lands at the far right).

### Best case: pivot is always the median

Suppose we always pick the median. On the warm-up array:

```
Level 0:  32  15   2  17  19  26  41  17  17        work: N
Level 1:  17  15   2  17 [17] 26  41  32  19        work: ~N/2 + ~N/2 = ~N
Level 2:   2  15  17  17 [17] 26  19  32  41        work: 4 x ~N/4 = ~N
...
```

The key structural fact: **work at each level is Θ(N)**, because partitioning all the subproblems at a given level touches each element at most once. And the **number of levels is Θ(log N)**, because each level halves the problem size. So:

Total = (work per level) × (number of levels) = Θ(N) × Θ(log N) = **Θ(N log N)**.

Written more generally, the runtime is Θ(N·H) where H is the height of the recursion tree. Everything in the analysis reduces to the question: how tall is H?

### Quicksort vs. Mergesort, theoretically

| | Best case | Worst case |
|---|---|---|
| Quicksort | Θ(N log N) | Θ(N²) |
| Mergesort | Θ(N log N) | Θ(N log N) |

On paper Mergesort looks strictly better, and Θ(N log N) vs. Θ(N²) is a huge difference, not a technicality. So how is Quicksort empirically the fastest sort? Because **on average it is Θ(N log N)**. The rigorous proof needs probability theory and calculus, which is out of scope for 61B, but the lecture gives two intuitive arguments and empirical evidence.

### Argument #1: The 10% argument

Suppose the pivot is never anywhere near the middle, but is always at least 10% from either edge. This is a deliberately pessimistic assumption: we are granting the adversary almost everything.

```
Level 0:                      N
Level 1:         N/10               9N/10
Level 2:   N/100   9N/100     9N/100   81N/100
```

Two observations:

1. **Work at each level is still O(N)**: the subproblem sizes at each level sum to at most N (N/10 + 9N/10 = N, and N/100 + 9N/100 + 9N/100 + 81N/100 = N).
2. **The height is still O(log N)**: the *largest* subproblem shrinks by a factor of 9/10 each level. After H levels the biggest piece has size N(9/10)^H, so H is approximately log_(10/9) N. Since changing the base of a logarithm only multiplies by a constant, log_(10/9) N = Θ(log N).

Runtime is O(N·H) = **O(N log N)**.

The punchline from the slide: even if you are unlucky enough to have a pivot that never lands anywhere near the middle, but at least always lands 10% from the edge, the runtime is still O(N log N). Lopsided splits are fine. Only *pathologically* lopsided splits (a constant-size chunk peeled off each time, so the problem shrinks by subtraction rather than by a constant factor) produce Θ(N²).

### Argument #2: Quicksort is BST sort

This is a lovely structural equivalence. Run Quicksort on `5 3 2 1 7 8 4 6` with leftmost-pivot selection and watch the recursion tree:

```
Quicksort recursion:                      Corresponding BST
5  3  2  1  7  8  4  6                            5
   3  2  1  4  [5]  7  8  6                   3       7
2  1  [3]  4  [5]  6  [7]  8                2    4   6    8
   1                                       1
```

**Key idea**: the `compareTo` calls are the same for BST insertion and for Quicksort.

- In Quicksort, every element gets compared to `5` during the first partition. In BST insertion (inserting 5, 3, 2, 1, 7, 8, 4, 6 in that order), every subsequent element gets compared to `5` at the root.
- In Quicksort, `3` gets compared only to `1`, `2`, `4`, and `5`. In the BST, the elements that pass through node `3` are exactly `1`, `2`, and `4` (plus the comparison with `5` on the way down).

Since the comparison counts are identical, the runtimes are asymptotically identical. And we already know from the BST material that **random insertion into a BST takes O(N log N) time** (expected height of a randomly built BST is Θ(log N)). Therefore Quicksort on a randomly ordered array takes O(N log N) expected time. We get the average-case result for free from a theorem we already proved.

This also explains the worst case in BST language: inserting a sorted sequence into a BST produces a degenerate spindly tree of height N, which is exactly the N-level Quicksort recursion.

### Empirical evidence: how concentrated is the average?

For N items:

- Mean number of compares to complete Quicksort: **~2N ln N**
- Standard deviation: (given on the slide as a formula; the diagram is missing, but the shape of the result is what matters)

The slide shows an empirical histogram of compare counts over 10,000 trials with N = 1000:

- Lots of arrays take around 12,000 compares.
- A very small number take around 15,000 compares.
- The chance of taking around 1,000,000 compares (which is roughly N², the worst case) is **effectively zero**.

So the Θ(N²) worst case is not merely rare on average, it is so rare that the distribution of compare counts is tightly clustered around 2N ln N. This is why the summary table can honestly say Quicksort is the fastest sort in practice despite its quadratic worst case.

Sanity check on the mean: 2 × 1000 × ln(1000) ≈ 2 × 1000 × 6.9 ≈ 13,800, which lands right in the range the histogram describes.

### Updated performance picture

| | Case | Runtime |
|---|---|---|
| Quicksort | Best | Θ(N log N) |
| Quicksort | Worst | Θ(N²) (for our pivot/partition strategies: sorted or close to sorted input) |
| Quicksort | Randomly chosen array | Θ(N log N) expected, with extremely high probability |
| Mergesort | Best | Θ(N log N) |
| Mergesort | Worst | Θ(N log N) |

The lecture is explicit that *why* Quicksort is faster than Mergesort (not just asymptotically equal) requires empirical analysis, and there is no obvious theoretical reason. (extra context: the usual explanations are better cache locality from in-place partitioning and a smaller constant factor per comparison, but the lecture does not claim this, and the slide on Heapsort does note that cache behavior is a CS 61C topic.)

### Quicksort flavors: the three design decisions

Quicksort is not one algorithm, it is a family. Saying "Quicksort is the fastest" is only true if you make the right decisions about:

1. **Pivot selection** (leftmost? random? median? median approximation?)
2. **Partition algorithm** (3-scan with an array copy? Tony Hoare's two-pointer scheme? two-pivot?)
3. **How to avoid the worst case** (which can be folded into the first two choices)

The lecture names specific flavors and benchmarks them (1000 arrays of 10,000 ints, unoptimized versions with no switch to insertion sort for small arrays):

| Flavor | Pivot selection | Partition | Worst-case avoidance | Time | Worst case |
|---|---|---|---|---|---|
| Mergesort | N/A | N/A | N/A | 1.3 s | Θ(N log N) |
| Quicksort L3S | Leftmost | 3-scan | Shuffle | 3.2 s | Θ(N²) |
| Quicksort LTHS | Leftmost | Tony Hoare | Shuffle | 0.9 s | Θ(N²) |
| Quicksort QSTH | Quickselect | Tony Hoare | Exact median | 4.9 s | Θ(N²) |
| Quicksort PickTH | Exact median (PICK/BFPRT) | Tony Hoare | Exact median | ~15 s | Θ(N log N) |

The headline results:

- **Quicksort L3S loses to Mergesort** (3.2 s vs 1.3 s). The 3-scan partition from the previous lecture makes an array copy and does three scans (for red, white, and blue items). That extra allocation and extra passes cost more than Quicksort's asymptotic advantage is worth, and in fact they share the same asymptotics, so the constants decide.
- **Quicksort LTHS beats Mergesort** (0.9 s vs 1.3 s). Swapping in Tony Hoare's in-place two-pointer partition is the single change that flips the result. The partitioning scheme is not an implementation detail, it is the difference between winning and losing.
- Even better schemes exist. The best known Quicksort uses a **two-pivot scheme**. Amusing historical note from the slide: this version was introduced to the world by a previously unknown person posting in a Java developers forum.

### Quicksort's origins

Quicksort was originally developed by Tony Hoare for a machine translation project, and the lecture notes it was the wrong tool for that job for two reasons:

1. The language Tony was using did not support recursion, so he could not easily implement Quicksort.
2. Sentences are usually shorter than 15 words, and for such small inputs **insertion sort is faster**.

The second point is a real lesson, not just trivia: asymptotic superiority does not matter at small N, which is why real library sorts switch to insertion sort below some threshold.

### Four philosophies for avoiding the worst case

The worst case does happen in practice, because sorted and almost-sorted arrays are extremely common in real data. The lecture's crowd-sourced answers included shuffling, finding the median, checking if almost sorted and switching to insertion sort, choosing a random pivot, human-in-the-loop, and choosing the middle element. Josh's organized answer is four philosophies:

**1. Randomness** (the lecture's preferred approach)

Two concrete strategies:
- **Strategy #1**: pick pivots randomly.
- **Strategy #2**: shuffle the array before you sort.

Either one destroys the correlation between the input's order and the pivot choice, so an adversary cannot construct a bad input. Important caveat from the slide: the shuffling strategy **requires care in the partitioning code to avoid Θ(N²) behavior on arrays of duplicates**. This is a common bug, present even in a well-known 2010s textbook. Shuffling makes the *order* random, but it cannot help if your partition code mishandles equal keys: an array of all-identical elements has no "good" ordering to shuffle into.

**2. Smarter pivot selection**

Split into two sub-cases:

- **2a, constant-time pivot picking**: Randomness is *necessary* for best Quicksort performance. For any pivot selection procedure that is both **deterministic** and **constant time**, the resulting Quicksort has a family of dangerous inputs that an adversary could easily generate. The reference is McIlroy's "A Killer Adversary for Quicksort." This kills "just pick the middle element" as a real defense: it fixes the sorted-array case and nothing more, because an adversary can construct the input that defeats whatever fixed rule you chose.
- **2b, linear-time pivot picking**: You *can* compute the actual median in linear time. "Exact median Quicksort" is genuinely safe, with worst case Θ(N log N), but it is **slower than Mergesort** (~15 s vs 1.3 s in the benchmark). The theoretical guarantee is real and the practical cost is prohibitive.

**3. Introspection**

Watch your own recursion depth. If it exceeds some critical value (say 10 ln N), switch to Mergesort. This is the pragmatic hybrid: you get Quicksort's speed on the overwhelming majority of inputs and a Θ(N log N) fallback on the rare bad ones. Java does this, in addition to approximating the median.

**4. Preprocess the array**

Analyze the array up front to see if Quicksort will be slow. There is **no obvious way to do this**. You cannot just check whether the array is sorted, because almost-sorted arrays are also slow, and "almost sorted" is not a cheap property to test. The slide also phrases this philosophy as "try to cheat: if the array is already sorted, don't sort (this doesn't work)."

### Quickselect: the selection problem (extra material)

Computing the exact median would give us a "safe Quicksort." But how do you find a median?

Obvious approach: sort the array and return the middle item. That takes Θ(N log N), which is already as expensive as the sort we were trying to speed up. We want better.

The insight: **partitioning can be used to find the exact median**, and the resulting algorithm is the best known median identification algorithm in practice.

**How Quickselect works.** To find the item at sorted index k:

1. Partition around some pivot.
2. Look at where the pivot landed.
   - If it landed exactly at index k, you are done. That item is the answer.
   - If it landed to the *left* of k, the answer must be in the right subarray. Recurse right only.
   - If it landed to the *right* of k, recurse left only.

The lecture traces finding the median of `9 550 14 6 10 5 330 817 913` (N = 9, so the median is at index 4):

```
Start:                   9  550  14   6  10   5  330 817 913
Partition, pivot (9) lands at index 2:
                         6    5   9  550  14  10  330 817 913
  Not the median (we want index 4). The median cannot be to the left,
  so partition the right subproblem only.

Partition right subproblem, pivot lands at index 6:
                         ...      550  14  10  330 550 817 913
  Not the median. Recurse again.

Partition, pivot lands at index 4:
                         4    5       10  14  330  ...
  Done. (The slide notes 9/2 = 4, confirming index 4 is the target.)
```

Note that unlike Quicksort, Quickselect **discards one side entirely** at each step rather than recursing on both. That is the whole source of its speed advantage.

**Worst case for Quickselect**: Θ(N²), and the input that causes it (with leftmost-pivot selection) is a **sorted array**:

```
[1 2 3 4 5 6 7 8 9 10 ... N]
[1 2 3 4 5 6 7 8 9 10 ... N]
[1 2 3 4 5 6 7 8 9 10 ... N]
...
[1 2 3 4 5 ... N/2 ... N]
```

Each partition peels off exactly one element from the front, so you do N + (N-1) + (N-2) + ... work before reaching the middle: Θ(N²), the same pathology as Quicksort.

**Expected performance**: Θ(N). Intuitive picture (the lecture is explicit that this is *not a proof*): on average the pivot ends up about halfway, so you do ~N compares, then ~N/2, then ~N/4, and so on:

N + N/2 + N/4 + ... + 1 = 2N - 1 = **Θ(N)**

This is a geometric series that sums to a constant multiple of N, which is why Quickselect is linear while Quicksort (which must recurse on *both* halves) is N log N. The difference between the two algorithms is one recursive call.

**Does Quickselect fix Quicksort?** No. Quicksort QSTH (use Quickselect to find the exact median, then Tony Hoare partition) runs in 4.9 s, far worse than Mergesort's 1.3 s, and it **does not even fix the worst-case runtime** (still Θ(N²), because Quickselect itself is Θ(N²) in the worst case). The slide's wry comment: it is a little strange to do a bunch of partitions to identify the optimal item to partition around.

**Can you find the median in guaranteed Θ(N) time?** Yes. Use **BFPRT**, called **PICK** in the original paper, developed in 1972. In practice it is rarely used. Quicksort PickTH (exact median via PICK, plus Tony Hoare partition) achieves worst case Θ(N log N) but runs in ~15 seconds, which is terrible. The cost to compute medians is too high.

**The practical conclusion**: you have to live with worst case Θ(N²) if you want good practical performance. (Or use introspection, which gets you most of both.)

### Sorting summary table

| | Best Case Runtime | Worst Case Runtime | Space | Notes |
|---|---|---|---|---|
| Selection Sort | Θ(N²) | Θ(N²) | Θ(1) | Find smallest item in remaining items, N times |
| Heapsort (in place) | Θ(N) (all duplicates) | Θ(N log N) | Θ(1)** | Bad cache (61C) performance |
| Mergesort | Θ(N log N) | Θ(N log N) | Θ(N) | Faster than Heapsort |
| Insertion Sort (in place) | Θ(N) (cN inversions) | Θ(N²) | Θ(1) | Best for small N or almost sorted |
| Quicksort (in place) | Θ(N log N) (also the average runtime) | Θ(N²) | Θ(log N) call stack | Fastest sort |

Listed by mechanism:

- **Selection sort**: find the smallest item and put it at the front.
- **Insertion sort**: figure out where to insert the current item.
- **Mergesort**: merge two sorted halves into one sorted whole.
- **Partition (quick) sort**: partition items around a pivot.

Final version of the memory/time table, with Quicksort relabeled honestly as "Random Quicksort":

| | Memory | Time | Notes |
|---|---|---|---|
| Heapsort | Θ(1) | Θ(N log N) | Bad caching (61C) |
| Insertion | Θ(1) | Θ(N²) | Θ(N) if almost sorted |
| Mergesort | Θ(N) | Θ(N log N) | |
| Random Quicksort | Θ(log N) expected | Θ(N log N) expected | Fastest sort |

Note the shift in the last row between the two tables: the final table says Θ(log N) **expected** memory and Θ(N log N) **expected** time. Both are expectations, because the recursion depth is itself random.

## Definitions

**Pivot**: the element chosen during partitioning that ends up in its final sorted position, with all smaller elements to its left and all larger elements to its right.

**Partition**: rearranging a subarray around a pivot so that the pivot is in its correct final sorted position, everything less than it is to the left (in any order), and everything greater is to the right (in any order).

**Best pivot**: the median, because it splits the problem into two subproblems of size N/2, minimizing the recursion tree height.

**Worst pivot**: the minimum or maximum element, because it reduces the problem size by exactly 1, maximizing the recursion tree height.

**Recursion tree height (H)**: the number of levels in the recursive decomposition. Quicksort's runtime is Θ(N·H), since each level does Θ(N) partitioning work.

**10% argument**: the observation that if every pivot lands at least 10% away from either edge of its subarray, the recursion height is log_(10/9) N = Θ(log N), so the runtime is O(N log N). Generalizes: any pivot guaranteed to land a constant *fraction* from the edge gives O(N log N).

**BST sort equivalence**: the fact that the `compareTo` calls performed by Quicksort (with leftmost pivot) on an array are identical to those performed by inserting that array's elements into a BST in order. Since random BST insertion is O(N log N), Quicksort on a random array is O(N log N) expected.

**Expected runtime**: the average runtime over the randomness in the algorithm (random pivot or shuffle) or over random input orderings. Quicksort's expected runtime is Θ(N log N), while its worst case remains Θ(N²).

**Quicksort L3S**: the lecture's label for the flavor with leftmost pivot selection, 3-scan partitioning (array copy plus three scans for red/white/blue items), and shuffling for worst-case avoidance. 3.2 s on the benchmark.

**Quicksort LTHS**: leftmost pivot selection, Tony Hoare's two-pointer partition, shuffling. 0.9 s, faster than Mergesort.

**Quicksort QSTH**: Quickselect pivot selection (exact median), Tony Hoare partition. 4.9 s, and still Θ(N²) worst case.

**Quicksort PickTH**: exact-median pivot selection via PICK/BFPRT, Tony Hoare partition. Worst case Θ(N log N) but ~15 s.

**Two-pivot scheme**: the best known Quicksort partitioning approach, which partitions around two pivots into three regions instead of one pivot into two. Introduced in a Java developers forum.

**Introspection**: a worst-case-avoidance philosophy where the algorithm monitors its own recursion depth and switches to a safer sort (e.g. Mergesort) if the depth exceeds a threshold such as 10 ln N. Used by Java.

**Deterministic constant-time pivot selection**: any pivot rule that makes its choice without randomness and in O(1) time (leftmost, middle, median-of-three). Always has a family of adversarial inputs that force Θ(N²) (McIlroy's killer adversary).

**Selection problem**: given an array and an index k, find the element that would be at index k if the array were sorted. The median is the special case k = N/2.

**Quickselect**: an algorithm for the selection problem that partitions repeatedly but recurses into only the one side that can contain index k. Θ(N) expected, Θ(N²) worst case.

**BFPRT (PICK)**: a 1972 algorithm that finds the exact median in guaranteed Θ(N) worst-case time. Rarely used in practice because its constant factor is too large.

## Worked Examples

### Example 1: Tracing the worst case by hand

The lecture's worst-case claim is that leftmost-pivot Quicksort on a sorted array is Θ(N²). Here is instrumented code to confirm it.

```java
public class QuicksortTrace {
    private static int partitionCalls = 0;
    private static int comparisons = 0;

    /** Partitions a[lo..hi] around pivot a[lo]. Returns the pivot's final index. */
    private static int partition(int[] a, int lo, int hi) {
        partitionCalls += 1;
        int pivot = a[lo];
        int smaller = lo;
        for (int i = lo + 1; i <= hi; i += 1) {
            comparisons += 1;
            if (a[i] < pivot) {
                smaller += 1;
                swap(a, smaller, i);
            }
        }
        swap(a, lo, smaller);
        return smaller;
    }

    private static void swap(int[] a, int i, int j) {
        int tmp = a[i];
        a[i] = a[j];
        a[j] = tmp;
    }

    private static void sort(int[] a, int lo, int hi) {
        if (lo >= hi) {
            return;
        }
        int p = partition(a, lo, hi);
        System.out.println("pivot " + a[p] + " landed at index " + p
                           + " for range [" + lo + ", " + hi + "]");
        sort(a, lo, p - 1);
        sort(a, p + 1, hi);
    }

    public static void sort(int[] a) {
        partitionCalls = 0;
        comparisons = 0;
        sort(a, 0, a.length - 1);
        System.out.println("partition calls: " + partitionCalls
                           + ", comparisons: " + comparisons);
    }
}
```

**What this does, step by step.** `partition` uses a single-pass scheme: it treats `a[lo]` as the pivot and maintains an index `smaller` marking the end of the "known smaller than pivot" region. It walks `i` from `lo + 1` to `hi`. Every time it finds an element smaller than the pivot, it grows the smaller-region by one and swaps the found element into that newly opened slot. At the end, it swaps the pivot from `lo` into position `smaller`, which is exactly the boundary between the smaller and larger regions. The pivot is now in its final sorted position, so it never moves again.

**Box-and-pointer reasoning in words.** The array `a` is a single object on the heap; every recursive `sort` call receives a *copy of the reference*, not a copy of the array. So all the recursive calls mutate the same underlying boxes of memory. What differs between calls is only the `lo`/`hi` integers, which are passed by value and live in each frame's own slots. This is why Quicksort is in-place and why its space cost is just the call stack: no new array object is ever allocated. (Contrast Mergesort, which allocates auxiliary arrays and therefore costs Θ(N) space.)

**Run on a sorted array of size 6 (`[1, 2, 3, 4, 5, 6]`):**

| Call range | Pivot | Scan comparisons | Pivot lands at | Left subproblem | Right subproblem |
|---|---|---|---|---|---|
| [0, 5] | 1 | 5 | 0 | empty | [1, 5] |
| [1, 5] | 2 | 4 | 1 | empty | [2, 5] |
| [2, 5] | 3 | 3 | 2 | empty | [3, 5] |
| [3, 5] | 4 | 2 | 3 | empty | [4, 5] |
| [4, 5] | 5 | 1 | 4 | empty | empty |

Total comparisons: 5 + 4 + 3 + 2 + 1 = 15 = 6·5/2. For general N this is N(N-1)/2 = Θ(N²), and the recursion depth is N-1, matching the lecture's worst-case diagram exactly.

**Run on `[32, 15, 2, 17, 19, 26, 41, 17, 17]` (the warm-up array):**

First partition picks pivot `32`. Scanning the other 8 elements: `15 < 32` (swap into slot 1, no-op), `2 < 32` (swap into slot 2, no-op), `17 < 32`, `19 < 32`, `26 < 32`, `41 > 32` (skip), `17 < 32`, `17 < 32`. Seven elements are smaller, so `smaller` ends at index 7, and the final swap puts `32` at index 7 with `41` moving to index 2's old position. The result is a 7-element left subproblem and a 1-element right subproblem: lopsided, but not maximally so, because `32` is not the maximum.

### Example 2: The two randomness strategies

```java
import java.util.Random;

public class RandomizedQuicksort {
    private static final Random RAND = new Random();

    /** Strategy #1: pick a random pivot. */
    private static int partitionRandomPivot(int[] a, int lo, int hi) {
        int pivotIndex = lo + RAND.nextInt(hi - lo + 1);
        swap(a, lo, pivotIndex);
        return partition(a, lo, hi);
    }

    /** Strategy #2: shuffle once up front, then always use leftmost. */
    public static void sortWithShuffle(int[] a) {
        shuffle(a);
        sortLeftmost(a, 0, a.length - 1);
    }

    private static void shuffle(int[] a) {
        for (int i = a.length - 1; i > 0; i -= 1) {
            int j = RAND.nextInt(i + 1);
            swap(a, i, j);
        }
    }
}
```

**Why both work.** The Θ(N²) case requires a *correlation* between the input's structure and the pivot rule. Strategy #1 breaks the correlation by randomizing the rule; Strategy #2 breaks it by randomizing the input. In both cases, an adversary who knows your code but not your random bits cannot construct a reliably bad input. This is precisely the point of Philosophy 2a: randomness is what you are buying, and a deterministic constant-time rule cannot buy it.

Note the shuffle uses Fisher-Yates (walk from the back, swap with a uniformly chosen index at or below the current position). Getting this loop subtly wrong (for example, choosing `j` from the whole array each time) produces a non-uniform shuffle, which weakens the guarantee.

**The duplicates caveat.** Strategy #2 "requires care in partitioning code to avoid Θ(N²) behavior on arrays of duplicates." Consider `[7, 7, 7, 7, 7, 7, 7, 7]`. Shuffling does nothing: every permutation is identical. Now the behavior depends entirely on how your partition handles elements *equal* to the pivot:

- If equal elements all go to one side, the pivot lands at an edge every single time, giving Θ(N²). The shuffle provided zero protection.
- If equal elements are split between the two sides (as Tony Hoare's two-pointer scheme does, because both pointers stop on equality and swap), the pivot lands near the middle and you get Θ(N log N).

The lecture notes this is a common bug, present even in a well-known 2010s textbook. The lesson: a worst-case-avoidance strategy is only as good as the partition it is paired with.

### Example 3: Quickselect

```java
public class QuickSelect {
    /** Returns the element that would be at index k if a were sorted. */
    public static int select(int[] a, int k) {
        return select(a, 0, a.length - 1, k);
    }

    private static int select(int[] a, int lo, int hi, int k) {
        if (lo == hi) {
            return a[lo];
        }
        int p = partition(a, lo, hi);
        if (p == k) {
            return a[p];
        } else if (p < k) {
            return select(a, p + 1, hi, k);
        } else {
            return select(a, lo, p - 1, k);
        }
    }

    public static int median(int[] a) {
        return select(a, a.length / 2);
    }
}
```

**Step by step on the lecture's example.** Input `[9, 550, 14, 6, 10, 5, 330, 817, 913]`, N = 9, so `median` calls `select(a, 4)`.

**Call 1: `select(a, 0, 8, 4)`.** Partition around pivot `9`. Elements smaller than 9: `6` and `5`. So `9` lands at index 2:

```
6  5  [9]  550  14  10  330  817  913
```

Is 2 == 4? No. Is 2 < 4? Yes, so the answer is somewhere to the *right* of index 2. We can discard indices 0 through 2 entirely, including the pivot. The median cannot be to the left, because only 2 elements are smaller than `9` and we need the 5th smallest.

**Call 2: `select(a, 3, 8, 4)`.** Partition `[550, 14, 10, 330, 817, 913]` around pivot `550`. Elements smaller than 550 in that range: `14`, `10`, `330`. Three of them, so `550` lands at index 3 + 3 = 6:

```
6  5  [9]  14  10  330  [550]  817  913
```

Is 6 == 4? No. Is 6 < 4? No, so recurse *left*.

**Call 3: `select(a, 3, 5, 4)`.** Partition `[14, 10, 330]` around pivot `14`. Only `10` is smaller, so `14` lands at index 3 + 1 = 4:

```
6  5  [9]  10  [14]  330  [550]  817  913
```

Is 4 == 4? Yes. Return `14`.

**Verify**: sorted, the array is `5 6 9 10 14 330 550 817 913`. Index 4 is `14`. Correct.

**The crucial difference from Quicksort.** Look at the recursive calls. Quicksort does:

```java
sort(a, lo, p - 1);
sort(a, p + 1, hi);   // both sides
```

Quickselect does:

```java
if (p < k) { return select(a, p + 1, hi, k); }
else       { return select(a, lo, p - 1, k); }   // exactly one side
```

One recursive call instead of two. That single change turns N + N/2 + N/4 + ... (a geometric series summing to 2N, so Θ(N)) into N per level × log N levels (Θ(N log N)). The lecture's intuitive picture is exactly this: ~N compares, then ~N/2, then ~N/4, under the assumption that on average the pivot ends up about halfway.

**Worst case.** On `[1, 2, 3, ..., N]` seeking the median, the leftmost pivot is always the minimum of its range, so it lands at `lo` every time and we only ever shave off one element. Work: N + (N-1) + (N-2) + ... + N/2, which is Θ(N²). The lecture's trace shows exactly this, the array never changing while the range creeps rightward one index at a time.

### Example 4: Why introspection works

```java
public class IntrospectiveQuicksort {
    public static void sort(int[] a) {
        int depthLimit = (int) (10 * Math.log(a.length));
        sort(a, 0, a.length - 1, depthLimit);
    }

    private static void sort(int[] a, int lo, int hi, int depthRemaining) {
        if (lo >= hi) {
            return;
        }
        if (depthRemaining <= 0) {
            mergesort(a, lo, hi);
            return;
        }
        int p = partition(a, lo, hi);
        sort(a, lo, p - 1, depthRemaining - 1);
        sort(a, p + 1, hi, depthRemaining - 1);
    }
}
```

**Why this gives a worst-case guarantee.** The depth counter is the direct measurement of H, and we know the runtime is Θ(N·H). Before the limit trips, we have done at most 10 ln N levels of Θ(N) partitioning work, which is Θ(N log N). After the limit trips, Mergesort finishes in Θ(N log N). Sum: Θ(N log N) worst case.

**Why it is nearly free in the common case.** The expected recursion depth on random input is about 2 lg N ≈ 1.39 ln N, comfortably under the 10 ln N threshold. So on the overwhelming majority of inputs the limit never trips and you pay only for one integer decrement per call. You get Quicksort's practical speed with Mergesort's guarantee, which is why Java does this (alongside approximating the median).

**Contrast with the alternatives.** Philosophy 2b (exact median) also gives a Θ(N log N) guarantee, but it pays the median cost on *every* call, which is why Quicksort PickTH takes ~15 seconds. Introspection pays only when things actually go wrong. This is the difference between insuring against a rare event and preventing it at great expense.

## Common Pitfalls

**Thinking a lopsided split means Θ(N²).** This is the single most common misunderstanding, and the 10% argument exists to correct it. A 10/90 split is fine. A 1/99 split is fine. Even a 1-in-a-million/rest split is O(N log N), just with a large constant, because the problem still shrinks by a constant *factor*. Θ(N²) requires the problem to shrink by a constant *amount* (like 1 element) each level. Fraction good, fixed amount bad.

**Saying "Quicksort is Θ(N log N)" without qualification.** Quicksort's worst case is genuinely Θ(N²). The correct statements are: best case Θ(N log N), worst case Θ(N²), expected (on random input or with random pivots) Θ(N log N). Writing Θ(N log N) as the unqualified runtime is wrong on an exam.

**Thinking shuffling eliminates the Θ(N²) possibility.** It makes it astronomically unlikely, not impossible. The histogram's tail is thin, not absent. The lecture's own wording is "the very rare Θ(N2) cases do happen in practice."

**Forgetting that duplicates defeat shuffling.** Shuffling randomizes order, and an array of identical elements has no meaningful order to randomize. Protection against duplicate-heavy input comes from the *partition* handling equal keys well, not from the shuffle. This was a bug in a published textbook.

**Thinking "pick the middle element" is a real fix.** It fixes the one input you were thinking about (a sorted array) and nothing else. Philosophy 2a is categorical: *any* deterministic constant-time pivot rule has an adversarial input family (McIlroy). Picking the middle just changes which arrays are catastrophic, it does not reduce their number.

**Thinking exact-median Quicksort is the best Quicksort.** It has the best worst-case guarantee (Θ(N log N)) and nearly the worst practical performance (~15 s, vs 1.3 s for Mergesort and 0.9 s for Quicksort LTHS). Theoretical safety and practical speed are different objectives here, and the lecture is blunt: you have to live with worst case Θ(N²) if you want good practical performance.

**Thinking Quickselect fixes Quicksort's worst case.** It does not. Quicksort QSTH is still Θ(N²) worst case, because Quickselect is itself Θ(N²) in the worst case, so using it to pick pivots inherits the problem while adding overhead. Only guaranteed-linear median finding (PICK/BFPRT) gives the Θ(N log N) guarantee, and that one is the ~15 s version.

**Confusing Quickselect's Θ(N) with Quicksort's Θ(N log N) and forgetting why.** The entire difference is one recursive call. If you recurse on both sides you get N log N; if you recurse on one side you get a geometric series summing to Θ(N). If you can state this, you understand both algorithms.

**Mixing up Quicksort's space bound.** The standard answer is Θ(log N) for the call stack (expected), because that is the expected recursion depth. But in the worst case the recursion depth is N, so the worst-case space is Θ(N). The lecture's final table is careful to write "Θ(log N) expected."

**Thinking the partition algorithm is an implementation detail.** Quicksort L3S (3.2 s) loses to Mergesort; Quicksort LTHS (0.9 s) beats it. Same asymptotics, same pivot rule, same worst-case avoidance. The only difference is the partition, and it decides the outcome. "Quicksort is the fastest sort" is a claim about a specific well-engineered flavor, not about the idea of partitioning.

**Assuming asymptotically-better always means faster.** The Quicksort origin story is the counterexample: Tony Hoare's sentences were under 15 words, so insertion sort was faster. Real libraries switch to insertion sort below a threshold for this reason, and the lecture explicitly notes its benchmarks use *unoptimized* versions that skip this switch.

**Claiming you can preprocess to detect bad input.** Philosophy 4 says there is no obvious way. Checking whether the array is sorted is not enough: almost-sorted arrays are also slow, and there is no cheap test for "almost sorted" in the relevant sense.

## Likely Exam Points

### 1. Give an input that forces Quicksort's worst case, for a stated pivot rule

**Question.** Quicksort always uses the leftmost element as the pivot. Give an array of 6 distinct integers that causes Θ(N²) runtime, and state the number of comparisons performed.

**Answer.** `[1, 2, 3, 4, 5, 6]` (any sorted array). Each partition picks the minimum of its range, so the pivot lands at the far left and the right subproblem shrinks by exactly 1. Comparisons: 5 + 4 + 3 + 2 + 1 = 15 = N(N-1)/2 = Θ(N²). `[6, 5, 4, 3, 2, 1]` also works: the leftmost item is the maximum each time, so the pivot lands at the far right and the left subproblem shrinks by 1.

### 2. Reason about a guaranteed-fraction pivot

**Question.** Suppose a magic pivot-selection procedure guarantees the pivot always lands at least 1% from either edge of its subarray. What is the runtime of Quicksort? What if instead the pivot always lands exactly 3 positions from the left edge?

**Answer.** With the 1% guarantee the runtime is O(N log N). The largest subproblem has size at most 0.99N, so after H levels it has size N(0.99)^H, giving H ≈ log_(100/99) N = Θ(log N), and work per level is O(N). The base of the logarithm only changes the constant factor, so the asymptotic result matches the 10% argument. With the "exactly 3 positions from the left" rule, the runtime is Θ(N²): the problem shrinks by a fixed *amount* (4 elements per level, the 3 on the left plus the pivot) rather than a constant *factor*, so there are Θ(N) levels of Θ(N) work. This distinction (constant fraction vs constant amount) is the thing being tested.

### 3. State and use the BST sort equivalence

**Question.** Explain why Quicksort on a randomly ordered array takes Θ(N log N) expected time, by relating it to a data structure from earlier in the course.

**Answer.** Quicksort with leftmost pivot performs exactly the same `compareTo` calls as inserting the array's elements into a BST in array order. In both, every element is compared against the first element (the root / the first pivot), and then the comparison structure recurses identically on the smaller-than and greater-than groups. For instance, on `5 3 2 1 7 8 4 6`, every element is compared to `5` in both, and `3` is compared only to `1, 2, 4, 5` in both. Since the comparison counts are identical, the runtimes are asymptotically identical. Random insertion into a BST produces an expected height of Θ(log N) and takes O(N log N) total time, so Quicksort on a random array is O(N log N) expected. The worst case corresponds to the degenerate spindly BST you get from inserting sorted data.

### 4. Classify worst-case-avoidance strategies

**Question.** For each strategy, say whether it gives Quicksort a Θ(N log N) *worst-case* guarantee, and whether it is practical: (a) shuffle before sorting, (b) always pick the middle element as pivot, (c) compute the exact median with BFPRT, (d) switch to Mergesort if recursion depth exceeds 10 ln N.

**Answer.**
(a) No guarantee. Worst case stays Θ(N²), it just becomes astronomically unlikely. Practical and the lecture's preferred approach (with care around duplicates).
(b) No guarantee. It is deterministic and constant time, so by McIlroy's killer-adversary result there is a family of inputs forcing Θ(N²). Practical but offers no real protection.
(c) Yes, Θ(N log N) worst case, since BFPRT is guaranteed linear. Not practical: Quicksort PickTH runs ~15 s vs Mergesort's 1.3 s, because the median cost is paid at every level.
(d) Yes, Θ(N log N) worst case. At most 10 ln N levels of Θ(N) Quicksort work (Θ(N log N)) plus a Mergesort finish (Θ(N log N)). Practical, which is why Java uses it. This is the only option that is both.

### 5. Quickselect runtime, both cases, with the reason

**Question.** State Quickselect's expected and worst-case runtimes. Explain why its expected runtime is asymptotically better than Quicksort's.

**Answer.** Expected Θ(N), worst case Θ(N²) (triggered by a sorted array under leftmost-pivot selection, since the pivot lands at `lo` every time and only one element is eliminated per level). The expected runtime beats Quicksort's because Quickselect recurses into only *one* side of the partition: the side that cannot contain index k is discarded whole. Assuming the pivot lands about halfway, the work is N + N/2 + N/4 + ... + 1 = 2N - 1 = Θ(N), a geometric series. Quicksort must recurse on both sides, so every level does Θ(N) total work across Θ(log N) levels, giving Θ(N log N). The difference is literally one recursive call.

### 6. Fill in the sorting summary table

**Question.** Give the best case, worst case, and space for Insertion sort, Heapsort, Mergesort, and Quicksort, and name the input that triggers each best case where one applies.

**Answer.**

| | Best | Worst | Space | Best-case trigger |
|---|---|---|---|---|
| Insertion sort | Θ(N) | Θ(N²) | Θ(1) | cN inversions (almost sorted) |
| Heapsort | Θ(N) | Θ(N log N) | Θ(1) | all duplicates |
| Mergesort | Θ(N log N) | Θ(N log N) | Θ(N) | n/a, always Θ(N log N) |
| Quicksort | Θ(N log N) | Θ(N²) | Θ(log N) call stack | also the average case |

Notes worth including: Heapsort has bad cache performance (a 61C topic), Mergesort is faster than Heapsort in practice despite matching asymptotics, Insertion sort is best for small N or almost-sorted input, and Quicksort's Θ(log N) space is the recursion stack and is an *expected* bound (worst case Θ(N)).

### 7. Interpret the empirical histogram

**Question.** For N = 1000, the mean number of compares for Quicksort is ~2N ln N. Approximately what is that? The histogram over 10,000 trials shows most arrays near that mean, a few near 15,000, and effectively none near 1,000,000. Explain what the 1,000,000 figure corresponds to and why it never appears.

**Answer.** 2 × 1000 × ln(1000) ≈ 2 × 1000 × 6.91 ≈ 13,800 compares. The 1,000,000 figure is roughly N² = 1,000,000, the worst-case compare count. It effectively never appears because the Θ(N²) behavior requires the pivot to land at or near an edge at essentially every level, which for a randomly ordered array has probability so small it does not occur in 10,000 trials. The distribution is tightly concentrated around the mean, with a thin tail reaching only ~15,000 (a small constant factor above the mean, not a factor of 70). This concentration, not the worst case, is what justifies calling Quicksort the fastest sort in practice.

### 8. Explain the benchmark result

**Question.** Quicksort L3S (leftmost pivot, 3-scan partition, shuffle) takes 3.2 s on the benchmark while Mergesort takes 1.3 s, but Quicksort LTHS (leftmost pivot, Tony Hoare partition, shuffle) takes 0.9 s. Both Quicksort versions have the same asymptotic runtime. Explain the gap.

**Answer.** All three are Θ(N log N) on this input (random arrays), so asymptotics cannot distinguish them, and the constant factors decide. The 3-scan partition allocates an array copy and makes three passes over the data (for red, white, and blue items), which costs extra memory traffic and extra allocation at every level of recursion. Tony Hoare's two-pointer scheme partitions in place with a single pass, eliminating both. That change alone moves Quicksort from 2.5x slower than Mergesort to 1.4x faster. The lesson: "Quicksort is the fastest sort" is a claim about a well-engineered flavor, and the partitioning scheme is the deciding engineering choice, not an implementation detail.

### 9. The duplicates trap

**Question.** You shuffle before sorting to avoid the worst case. A colleague reports that your Quicksort takes Θ(N²) on an array of 100,000 identical integers. Is this possible? Explain.

**Answer.** Yes. Shuffling randomizes the *order* of elements, but every permutation of an all-identical array is the same array, so the shuffle accomplishes nothing. The behavior then depends entirely on how the partition code handles elements equal to the pivot. If all equal elements are sent to one side, the pivot lands at an edge at every level, producing N levels of Θ(N) work: Θ(N²). The fix is in the partition, not the shuffle: a scheme that splits equal elements across both sides (as Tony Hoare's two-pointer scheme does, since both pointers stop on equality and swap) keeps the pivot near the middle and restores Θ(N log N). The lecture notes this is a common bug, present even in a well-known 2010s textbook.

## Summary

- Quicksort's runtime is Θ(N·H), where H is the recursion tree height. The entire analysis reduces to bounding H.
- Worst case Θ(N²): pivot always lands at an edge, giving N levels. With leftmost-pivot selection this happens on sorted and almost-sorted (and reverse-sorted) arrays. Work is N + (N-1) + ... + 1.
- Best case Θ(N log N): pivot is the median, giving log N levels of Θ(N) work each.
- **10% argument**: a pivot that always lands at least 10% from either edge still gives O(N log N), since H ≈ log_(10/9) N. A constant *fraction* from the edge is fine; a constant *amount* is what produces Θ(N²).
- **BST sort equivalence**: Quicksort's `compareTo` calls are identical to BST insertion's. Since random BST insertion is O(N log N), Quicksort on a random array is Θ(N log N) expected.
- Empirically, mean compares ≈ 2N ln N, tightly concentrated. For N = 1000 that is ~13,800, with a thin tail to ~15,000 and effectively zero chance of the ~1,000,000 worst case.
- Quicksort: best Θ(N log N), worst Θ(N²), random-array Θ(N log N) expected. Mergesort: Θ(N log N) both cases. Quicksort still wins in practice, and the lecture is honest that *why* requires empirical analysis with no obvious theoretical reason.
- Four philosophies for avoiding the worst case: **randomness** (random pivot or shuffle, the preferred approach), **smarter pivot selection** (approximate or compute the median), **introspection** (switch to Mergesort past a depth limit, what Java does), **preprocessing** (no obvious way to do it, checking for sortedness is insufficient).
- Shuffling does not protect against duplicate-heavy arrays. Equal-key handling in the partition does. This was a published-textbook bug.
- Any **deterministic, constant-time** pivot rule has an adversarial input family forcing Θ(N²) (McIlroy's killer adversary). Randomness is necessary, not optional.
- Quicksort is a family, not an algorithm. Three choices define a flavor: pivot selection, partition algorithm, worst-case avoidance.
- Benchmarks (1000 arrays of 10,000 ints, unoptimized): Mergesort 1.3 s, Quicksort L3S 3.2 s, Quicksort LTHS 0.9 s, Quicksort QSTH 4.9 s, Quicksort PickTH ~15 s. The partition scheme alone flips Quicksort from losing to winning.
- **Quickselect** (extra material): partition repeatedly, recursing into only the side that can contain index k. Θ(N) expected (N + N/2 + N/4 + ... = Θ(N)), Θ(N²) worst case on sorted input. The single-recursive-call structure is the whole reason it is linear.
- Using Quickselect for pivots (QSTH) is slow and still Θ(N²) worst case. **BFPRT/PICK** (1972) finds the exact median in guaranteed Θ(N) time and gives Quicksort a Θ(N log N) worst case, but at ~15 s it is unusable. You have to accept Θ(N²) worst case for good practical performance, or use introspection to get most of both.
- Quicksort's origin story: the wrong tool for Tony Hoare's own job, since his language lacked recursion and his inputs were under 15 words (insertion sort wins at small N).
