<!-- Mon, Nov 09, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 32: Basic Sorts

This lecture opens Phase 3 of CS 61B (Algorithms and Software Engineering) and begins a five-lecture arc on sorting. It starts by asking what sorting even means: you cannot sort a collection without a way to compare its elements, and not every comparison works. Knuth's two laws (trichotomy and transitivity) define exactly what a valid ordering relation needs, and divisibility serves as the counterexample that fails. The lecture then reframes sorting as inversion removal, and walks through three algorithms: selection sort (simple, in-place, quadratic), heapsort (both a naive version that builds a separate heap and output array, and an in-place version using bottom-up heapification on a max heap), and mergesort (recursive, quasi-linear, but requiring an auxiliary array). The recurring theme is a tradeoff between time complexity and space complexity, and the recurring trick is to notice that the heap and the sorted output never need the same array slots at the same time.

## Key Concepts

### Why we care about sorting

Sorting is a useful end in itself, but it is more often a preprocessing step. Once a sequence is sorted:

- Equivalent items are adjacent, so duplicates can be found rapidly.
- Items are in increasing order, so binary search becomes available.
- The data can be converted into various balanced data structures (for example, BSTs).

The lecture also framed sorting as a set of case studies in how to approach basic computational problems, and several of the solutions reuse the data structures built earlier in the semester (heaps in particular).

### Sorting requires an ordering relation, and not every relation works

The central idea from the warm-up: you cannot put elements in order until you decide how to compare them, and the comparison must be well behaved. Knuth (in *The Art of Computer Programming*) defines a **total order** on a relation (written `<`) by two laws:

- **Law of Trichotomy**: exactly one of `a < b`, `a = b`, `b < a` is true.
- **Law of Transitivity**: if `a < b` and `b < c`, then `a < c`.

Intuitively, these two properties are precisely what you need to line elements up.

**The counterexample from lecture: divisibility.** Let the relation be "divides". Take `a = 2`, `b = 3`. Then 2 does not divide 3, 2 is not equal to 3, and 3 does not divide 2. All three options in trichotomy are false, so divisibility fails trichotomy. The same holds for 2 and 9. Divisibility is transitive (2 divides 4, 4 divides 8, so 2 divides 8), but losing trichotomy is fatal: there is no way to decide whether 2 or 9 goes first. The lecture mentioned that the structure you get instead of a line is a web called a **lattice** (extra context: a lattice is a partial order where every pair of elements has a least upper bound and a greatest lower bound; the course does not pursue this).

The warm-up applied this to three collections:

- **Numbers** (4, 2, 6, 0, 9, 1): sortable, using `<` or `>`.
- **Words** ("hello", "bye", "hi", "zebra", "allow"): sortable alphabetically / lexicographically (compare first letters, break ties on second letters, and so on), or by length.
- **Sets** ({1,2}, {1,2,3}, {a,b}, {3,4,5}): sortable by **size** (the two-element sets come first, in either order, then the three-element sets). Sortable by number of subsets (size of the power set), which is really the same thing as sorting by size. **Not** sortable by **subset**: {1,2} is not a subset of {a,b}, {a,b} is not a subset of {1,2}, and they are not equal, so trichotomy fails exactly as it did for divisibility.

### Equality under a relation is not Java's `.equals`

Take the ordering relation "length of string". Trichotomy holds: exactly one of `len(a) < len(b)`, `len(a) = len(b)`, `len(b) < len(a)` is true. Transitivity holds too. So `["cows", "get", "going", "the"]` has **two** valid sorts:

- `["the", "get", "cows", "going"]`
- `["get", "the", "cows", "going"]`

Under this relation "the" **equals** "get", because `len("the") == len("get")`. That `=` is the relation's notion of equality, not `String.equals`. `"the".equals("get")` is `false`, but as far as the length-based sort is concerned they are interchangeable. This is why the same input can have multiple correct sorted outputs.

### A sort is a permutation

Formally: a sort is a permutation (re-arrangement) of a sequence of elements that puts the keys into non-decreasing order relative to a given ordering relation, so that `x1 ≤ x2 ≤ x3 ≤ ... ≤ xN`. The `≤` here is a placeholder for whatever relation you supplied. It could be greater-than-or-equal (which looks visually wrong but is perfectly valid), it could be lexicographic, it could be by word length. The only requirement is that the relation satisfies the two laws and that consecutive elements are related in one consistent direction down the whole line.

### Sorting as inversion removal

An **inversion** is a pair of elements that are out of order with respect to `<`. Crucially, the pair does not need to be adjacent.

The lecture's sequence: `0 1 1 2 3 4 8 6 9 5 7`. It has **6 inversions**: 8-6, 8-5, 8-7, 6-5, 9-5, 9-7. The transcript noted the common mistake of only spotting the adjacent-looking ones (8-6 and 9-5) and answering 2. Non-adjacent pairs count: 8 is also inverted with 5 and with 7.

The **maximum** number of inversions for a sequence of this length is 55, achieved by reverse-sorted order. The counting argument: with 11 distinct elements, the largest element is inverted with all 10 elements after it, the next is inverted with 9 more (its pair with the largest was already counted), then 8, then 7, and so on, giving `10 + 9 + 8 + ... + 2 + 1 = 55`. (The transcript noted that the actual lecture sequence has duplicate 1s, so it cannot quite reach 55; changing one 1 to a distinct value fixes that.)

So: a sequence with 6 inversions out of a possible 55 is already "mostly sorted". This gives an alternate statement of the sorting goal:

> Given a sequence of elements with Z inversions, perform a sequence of operations that reduces inversions to 0.

### Time complexity and space complexity

Characterizations of runtime efficiency are called the **time complexity** of an algorithm (example: Dijkstra's has time complexity `O(E log V)`). Each primitive operation (addition, array access) counts as 1 unit of time, provided its cost is independent of input size. This is about counting steps, not wall-clock time: a faster chip does not change an algorithm's time complexity.

Characterizations of **extra** memory usage are called the **space complexity**. Dijkstra's has space complexity `Θ(V)`, for the priority queue, `distTo`, and `edgeTo`. The graph itself takes `Θ(V + E)` space, but that is not counted, because the graph already exists and is an **input** to Dijkstra's. Each primitive object (one variable, one list element) counts as 1 unit of space if its size is independent of input size. The lecture noted that people define this differently, but counting only extra memory beyond the input is the common convention.

### Selection sort

The algorithm:

1. Find the smallest item in the unsorted portion of the array.
2. Move it to the end of the sorted portion of the array.
3. Selection sort the remaining unsorted items.

On the lecture's input `[32, 15, 2, 17, 19, 26, 41, 17, 17]`: the smallest item is 2, so **swap** it with 32 to get `[2, 15, 32, 17, 19, 26, 41, 17, 17]`, and the boundary between sorted and unsorted moves right by one. Then 15 is already the smallest remaining, so it stays. Then 17 (any of the three 17s is fine) swaps into position, and so on until the whole array is `[2, 15, 17, 17, 17, 19, 26, 32, 41]`.

The swap is what makes it in-place. Rather than setting 32 aside and figuring out where it goes, you just trade it with the item you want at the front, and grow the sorted prefix one slot at a time.

**Time complexity: Θ(N²).** You look through N items for the smallest, then N-1, then N-2, down to 1. `N + (N-1) + (N-2) + ... + 1 = Θ(N²)`. Geometrically this is a triangle, which is half a square, and big-O / Θ ignore the constant 1/2.

**Space complexity: Θ(1)**, because we swap in place. Beyond a constant number of variables, no extra memory is used.

**Best case is also Θ(N²).** The lecture was explicit about this: selection sort is naive, so even on an already-sorted array it still scans front-to-back for the minimum every iteration. No cleverness is built in to detect sortedness.

The problem with selection sort is obvious once you see it: we rescan the entire remaining array every single time just to find the minimum, discarding all the work from the previous scan. The guiding question for the rest of the lecture:

> How could we make it easier to find the next smallest item?

### Naive heapsort

The answer: store items in a heap, so getting the extreme item is cheap. A priority queue always keeps the extreme element at the root, so you pull from the root and the next one bubbles up for free.

The lecture uses a **max-oriented heap**. A min heap would work equally well for naive heapsort, but it would not permit the in-place trick coming later.

Naive heapsort on N items:

1. Insert all items into a max heap, discard the input array. Create an output array.
2. Repeat N times:
   - Delete the largest item from the max heap.
   - Put the deleted item at the end of the unused part of the output array.

**Phase 1 (heapification).** Insert `[32, 15, 2, 17, 19, 26, 41, 17, 17]` one at a time into a max heap. Position 0 is left unused in this heap implementation, so the backing array becomes `[_, 41, 19, 32, 17, 17, 2, 26, 15, 17]` with size 9. As a tree: 41 is the root, its children are 19 and 32; 19's children are 17 and 17; 32's children are 2 and 26; and 17 (the first child of 19) has children 15 and 17.

Runtime to heapify this way: **O(N log N)**. You insert N items, and each insertion is O(log N). The lecture flagged why it is `O` rather than `Θ`: sometimes an inserted item does not need to bubble at all, so O(log N) is an upper bound per insertion, not an exact cost.

**Phase 2 (deletion).** Delete the max (41) and write it into the last unused output slot. The heap then repairs itself: the last item (17) moves to the root and sinks, swapping with 32 and then with 26, while 32 and 26 percolate up. The heap becomes `[_, 32, 19, 26, 17, 17, 2, 17, 15, 0]` with size 8, and output is `[0, 0, 0, 0, 0, 0, 0, 0, 41]`. Repeat: 32 goes to the second-to-last output slot, then 26, then 19, then 17, 17, 17, 15, 2, until the heap has size 0 and the output is `[2, 15, 17, 17, 17, 19, 26, 32, 41]`.

**Time complexity: O(N log N).**
- Getting items into the heap: O(N log N).
- Selecting the largest item: Θ(1) (it is the root).
- Removing the largest item: O(log N) per removal, done N times, so O(N log N).
- Total: `O(N log N) + Θ(N) + O(N log N) = O(N log N)`.

This is far better than selection sort. The lecture stressed how big that jump is: `N²` is `N × N`, and replacing one factor of `N` with `log N` shrinks it exponentially, since log is the inverse of exponentiation. `N log N` is sometimes called **quasi-linear**, because the log factor grows so slowly that the result is nearly linear.

**Space complexity: Θ(N)**, to build the additional copy of all the data. That is strictly worse than selection sort's Θ(1). The lecture flagged it as "probably no big deal (??)" and then immediately set about fixing it.

### In-place heapsort: the key observation

Look at any mid-algorithm snapshot of naive heapsort. The heap array has a run of filler 0s at the right end, and the output array has a run of filler 0s at the left end. Those 0s are placeholders with no meaning. And the pattern is exact: at each step the heap shrinks by one item and the output grows by one item. The two never need more slots than one array provides.

So: **store the heap and the output in the same array.**

1. Convert the input into a heap, ideally in place so no extra memory is used.
2. Repeat N times: delete the largest item from the max heap and move it into the vacated array slot, that is, swap the root with the last item in the heap and shrink the heap size by one.

A mid-algorithm array looks like `[17, 15, 17, 2, | 17, 19, 26, 32, 41]` with heap size 4: the left part is an unsorted max heap, the right part is the sorted suffix, and the dividing line slides left as the sort proceeds.

**This is why max heap, not min heap.** With a max heap, the deleted item is the largest remaining, which belongs at the **right** end of the array, while the heap lives on the left. The two regions grow toward each other and never contend for a slot. With a min heap, the smallest items belong on the left, which is exactly where the heap lives, so the heap and the sorted output would compete for the same side of the array. The max heap keeps the dividing line clean.

Also note: for in-place heapsort we **do** use position 0, because leaving spot 0 unused would require allocating a new, larger array, defeating the purpose.

### Bottom-up heapification

Converting the input array into a heap in place needs a different approach than inserting one item at a time. Two parameters are in play:

- **Min heap vs max heap**: max heap, for the reason above (we always want to move the largest remaining item to the vacated spot).
- **Top-down heapification vs bottom-up heapification**: bottom up, which is asymptotically faster.

**Top-down** means building from the root down (what inserting one at a time effectively does). **Bottom-up** means building from the leaves up:

> Sink nodes in reverse level order: `sink(k)`. After sinking, the tree rooted at position k is guaranteed to be a heap.

Walking the lecture's example on `[32, 15, 2, 17, 19, 26, 41, 17, 17]` (as a tree: 32 at the root, children 15 and 2; 15's children are 17 and 19; 2's children are 26 and 41; 17's children are 17 and 17). Note at the start this is **not** a heap: 15 is not larger than its children 17 and 19.

Process nodes right to left:

- The last 17 (a leaf): a single node with nothing below it **is** a valid max heap. Sinking it has no effect. Locked in.
- The previous 17 (a leaf): same. No effect.
- 41 (a leaf): same. No effect.
- 26 (a leaf): same. No effect.
- 19 (a leaf): same. No effect.
- 17 (the one with children 17 and 17): 17 is at least as big as both children, so the heap property holds. No effect. All three 17s are now part of the same heap.
- **2** (children 26 and 41): **this is the first sink that does something.** 2 is smaller than its children, so swap it with its larger child, 41. Array becomes `[32, 15, 41, 17, 19, 26, 2, 17, 17]`.
- **15** (children 17 and 19): also violates the property. Swap with the larger child, 19. Array becomes `[32, 19, 41, 17, 15, 26, 2, 17, 17]`.
- **32** (the root, children 19 and 41): violates the property. Swap with 41. Array becomes `[41, 19, 32, 17, 15, 26, 2, 17, 17]`.

Punchline: since the tree rooted at position 0 is the root of a heap, the entire array is a heap. All of this happened in place, using Θ(1) extra space.

Note the subtlety when 32 sinks: after swapping 32 with 41, 32 lands where 41 was and its new children are 26 and 2, both smaller, so it stops there. In the general case a sinking node may travel several levels.

### In-place heapsort: the deletion phase

Starting from the heapified array `[41, 19, 32, 17, 15, 26, 2, 17, 17]` with heap size 9:

- Delete 41: swap the root with the last item in the heap, then sink. Array becomes `[32, 19, 26, 17, 15, 17, 2, 17, | 41]`, heap size 8, with 41 now in the sorted suffix.
- Delete 32: `[26, 19, 17, 17, 15, 17, 2, | 32, 41]`, heap size 7.
- Delete 26: the root 26 is removed, 2 takes its place and sinks (2 swaps with 19, then with 17), giving `[19, 17, 17, 2, 15, 17, | 26, 32, 41]`, heap size 6.
- Continue identically until heap size 0 and the array reads `[2, 15, 17, 17, 17, 19, 26, 32, 41]`, fully sorted.

### In-place heapsort complexity

**Time: O(N log N).** Bottom-up heapification is N sink operations, each taking no more than O(log N), so heapification is O(N log N). Selecting the largest is Θ(1). Removing the largest is O(log N) per removal, N times. Total O(N log N).

The slides flag two "extra for experts" results: heapsort is Θ(N log N) in the **worst** case, and bottom-up heapification is actually **Θ(N)** time (tighter than the O(N log N) bound just derived).

**Space: Θ(1).** The only extra memory is a constant number of instance variables, for example `size`. One unimportant caveat from the slides: if heap operations are implemented recursively, space complexity becomes Θ(log N) due to the call stack. The difference between Θ(log N) and Θ(1) space is effectively nothing.

**Best case: Θ(N).** The slides note that an array of all duplicates yields linear runtime for heapsort, because no element ever needs to sink.

The slides also list a practical drawback: heapsort has **bad cache performance** (a CS 61C topic), because the heap's parent/child jumps scatter accesses across the array.

### Mergesort

Top-down mergesort on N items:

1. Split items into 2 roughly even pieces.
2. Mergesort each half (recursive, so the steps are not shown).
3. Merge the two sorted halves to form the final result.

On `[32, 15, 2, 17, 19, 26, 41, 17, 17]`, the split gives a left half `[32, 15, 2, 17]` and a right half `[19, 26, 41, 17, 17]`. After recursively sorting each, the input array holds `[2, 15, 17, 32, | 17, 17, 19, 26, 41]`, with each side sorted.

The **merge** keeps three indices: `i` into the left half, `j` into the right half, and `p` into an auxiliary array. Repeatedly:

- Compare `input[i] < input[j]` (if necessary).
- Copy the smaller item into `aux[p]` and increment `p` and whichever of `i` or `j` was used.

Tracing it: `i` starts at 2 and `j` starts at 17. 2 < 17, so copy 2, advance `i`. Now 15 vs 17: copy 15. Now 17 vs 17: under `<`, the left 17 is not strictly less, so the comparison resolves in favor of one side; the lecture's trace copies the left 17 first, then the right 17, then the other right 17. Then 32 vs 19: copy 19. Then 32 vs 26: copy 26. Then 32 vs 41: copy 32. At this point the **left side has run out of items**, so no comparison is made; 41 is copied directly. The aux array holds `[2, 15, 17, 17, 17, 19, 26, 32, 41]`.

The "if necessary" qualifier on the comparison matters: once one side is exhausted, the remaining items on the other side are copied without any comparison at all.

**Time complexity: Θ(N log N)**, from the asymptotics lecture. **Space complexity: Θ(N)** for the auxiliary array.

The slides note it is possible to do an in-place mergesort, but the algorithm is very complicated and runtime performance suffers by a significant constant factor. The slides also note mergesort is **faster than heapsort** in practice, which connects to the cache remark: mergesort's sequential scans are cache-friendly while heapsort's are not.

### The scoreboard

| Sort | Best Case Runtime | Worst Case Runtime | Space | Notes |
| --- | --- | --- | --- | --- |
| Selection Sort | Θ(N²) | Θ(N²) | Θ(1) | Find smallest item in remaining items, N times |
| Heapsort (in place) | Θ(N)\* | Θ(N log N) | Θ(1)\*\* | Bad cache (61C) performance |
| Mergesort | Θ(N log N) | Θ(N log N) | Θ(N) | Faster than heapsort |

\* An array of all duplicates yields linear runtime for heapsort.
\*\* Assumes heap operations implemented iteratively, not recursively.

## Definitions

**Total order**: a relation (written `<`) satisfying both the Law of Trichotomy and the Law of Transitivity. Exactly the structure needed to put elements in a line.

**Law of Trichotomy**: for any `a` and `b`, exactly one of `a < b`, `a = b`, `b < a` is true.

**Law of Transitivity**: if `a < b` and `b < c`, then `a < c`.

**Ordering relation**: the comparison rule used for sorting. In Java it is typically supplied as a `compareTo` or `compare` method.

**Sort**: a permutation (re-arrangement) of a sequence of elements that puts the keys into non-decreasing order relative to a given ordering relation, so `x1 ≤ x2 ≤ ... ≤ xN`.

**Inversion**: a pair of elements that are out of order with respect to `<`. The pair need not be adjacent. A sorted sequence has 0 inversions; a reverse-sorted sequence of N distinct elements has the maximum, `(N-1) + (N-2) + ... + 1`.

**Time complexity**: a characterization of an algorithm's runtime efficiency, counting each primitive operation (addition, array access) as 1 unit of time when its cost is independent of input size.

**Space complexity**: a characterization of an algorithm's **extra** memory usage, beyond the input. Each primitive object (one variable, one list element) counts as 1 unit of space when its size is independent of input size.

**In-place**: an algorithm that mutates the input array itself rather than allocating a new array or data structure, using only Θ(1) extra memory.

**Selection sort**: repeatedly find the smallest item in the unsorted portion, swap it to the front of the unsorted portion, and fix it in place.

**Max-oriented heap (max heap)**: a heap where every node is at least as large as its children, so the maximum element sits at the root.

**Naive heapsort**: insert all items into a max heap, then repeatedly delete the max and write it into the end of the unused part of a separate output array.

**In-place heapsort**: bottom-up heapify the input array, then repeatedly swap the root with the last item in the heap and shrink the heap by one, leaving the deleted maxima as a sorted suffix.

**Bottom-up heapification**: converting an array into a heap by calling `sink(k)` on nodes in reverse level order (from the last node backward to position 0). After sinking node k, the tree rooted at k is guaranteed to be a heap.

**Top-down heapification**: building a heap from the root down, effectively what N successive inserts do. Asymptotically slower than bottom-up.

**Sink (also "sink down")**: move a node downward in a heap by repeatedly swapping it with its larger child (in a max heap) until the heap property holds.

**Mergesort (top-down)**: split the items into two roughly even halves, mergesort each half recursively, then merge the two sorted halves.

**Merge**: combine two sorted sequences into one sorted sequence by repeatedly copying the smaller of the two current front elements, advancing past exhausted sides without comparison.

**Quasi-linear**: the informal name for Θ(N log N) growth, so called because the log factor grows slowly enough that the result is nearly linear.

**Lattice**: the web-like structure you get from a relation that is transitive but fails trichotomy, for example divisibility. Elements like 2 and 9 are simply incomparable. (The lecture mentioned this name but did not develop it.)

## Worked Examples

### 1. An ordering relation as a Java `Comparator`

This is the lecture's own example. Ordering relations are typically given in the form of `compareTo` or `compare` methods.

```java
import java.util.Comparator;

public class LengthComparator implements Comparator<String> {
    public int compare(String x, String b) {
        return x.length() - b.length();
    }
}
```

What it does: `compare` returns a negative number when `x` is shorter than `b`, zero when they are the same length, and a positive number when `x` is longer. That sign convention is the whole interface: callers never look at the magnitude, only whether it is negative, zero, or positive.

Why it is a valid ordering relation: trichotomy holds because for any two strings exactly one of `len(a) < len(b)`, `len(a) = len(b)`, `len(b) < len(a)` is true. Transitivity holds because integer `<` is transitive.

The subtle part: with respect to this order, `"the"` **equals** `"get"`. Both have length 3, so `compare` returns 0. This usage of `=` is not the same as the equality given by the `String` method: `"the".equals("get")` is `false`. A sorting algorithm driven by this comparator is free to put either one first, which is why `["cows", "get", "going", "the"]` has two valid sorts under this relation.

Environment reasoning: a `LengthComparator` object has no instance variables, so it is a tiny box on the heap with only its class pointer. When a sort method takes a `Comparator<String>` parameter, the parameter in the sort method's frame holds a reference pointing at that same single object; no copy is made. Every comparison the sort performs dispatches dynamically to `LengthComparator.compare`, and the two `String` arguments are themselves references passed by value, so `compare` reads the strings but cannot modify what the caller's variables point to.

### 2. Selection sort

```java
public class SelectionSort {
    public static void sort(int[] a) {
        for (int i = 0; i < a.length; i += 1) {
            int minIndex = i;
            for (int j = i + 1; j < a.length; j += 1) {
                if (a[j] < a[minIndex]) {
                    minIndex = j;
                }
            }
            swap(a, i, minIndex);
        }
    }

    private static void swap(int[] a, int p, int q) {
        int temp = a[p];
        a[p] = a[q];
        a[q] = temp;
    }
}
```

(Extra context: the lecture presented selection sort diagrammatically rather than as code. This is the standard rendering of the three bullet points it gave.)

Step by step on the lecture's input `[32, 15, 2, 17, 19, 26, 41, 17, 17]`:

- `i = 0`. The inner loop scans indices 1 through 8 looking for anything smaller than `a[0] = 32`. It finds `a[2] = 2`, and no smaller value after. `minIndex` ends at 2. Swap positions 0 and 2: `[2, 15, 32, 17, 19, 26, 41, 17, 17]`. The sorted prefix is now `[2]`.
- `i = 1`. Scan indices 2 through 8 against `a[1] = 15`. Nothing is smaller. `minIndex` stays 1, so `swap(a, 1, 1)` is a no-op: `[2, 15, 32, 17, 19, 26, 41, 17, 17]`. Sorted prefix `[2, 15]`.
- `i = 2`. Scan against `a[2] = 32`. The first smaller value found is `a[3] = 17`. Then `a[4] = 19` is not smaller than 17, `a[7] = 17` is not strictly less than 17, so `minIndex` stays 3. Swap positions 2 and 3: `[2, 15, 17, 32, 19, 26, 41, 17, 17]`.
- `i = 3`. Scan against `a[3] = 32`. `a[4] = 19` is smaller, then `a[7] = 17` is smaller still, and `a[8] = 17` is not strictly less. `minIndex = 7`. Swap positions 3 and 7: `[2, 15, 17, 17, 19, 26, 41, 32, 17]`.
- Continuing this way produces `[2, 15, 17, 17, 17, 19, 26, 32, 41]`.

Why the inner loop starts at `i + 1`: everything at indices `< i` is already fixed and known to be in final position, so rescanning it would waste work and could incorrectly select an already-placed element.

Why `swap` works: `temp` holds a copy of `a[p]` before `a[p]` is overwritten. Without it, the assignment `a[p] = a[q]` would destroy the only copy of the old `a[p]`. In a box-and-pointer picture, `a` in the `sort` frame and `a` in the `swap` frame are two separate boxes holding the **same** arrow to the same array object on the heap, which is exactly why mutations inside `swap` are visible to `sort` after it returns. `int temp` is a primitive, so it is a plain value box in `swap`'s frame and disappears when `swap` returns.

Why the time complexity is Θ(N²): the inner loop runs `N-1` times, then `N-2`, down to 0. Summing gives `(N-1) + (N-2) + ... + 1 = N(N-1)/2 = Θ(N²)`. The `if` inside is Θ(1), so the comparison count drives the whole runtime. Note the loop structure makes no attempt to detect sortedness, which is precisely why the best case is also Θ(N²).

Why the space complexity is Θ(1): the only allocations are `i`, `j`, `minIndex`, and `temp`, four variables regardless of N. No second array is ever created.

### 3. Naive heapsort

```java
import java.util.PriorityQueue;
import java.util.Collections;

public class NaiveHeapsort {
    public static int[] sort(int[] input) {
        PriorityQueue<Integer> maxHeap =
                new PriorityQueue<>(Collections.reverseOrder());
        for (int x : input) {
            maxHeap.add(x);
        }

        int[] output = new int[input.length];
        for (int p = input.length - 1; p >= 0; p -= 1) {
            output[p] = maxHeap.poll();
        }
        return output;
    }
}
```

(Extra context: the lecture described naive heapsort with diagrams and relied on the course's own heap implementation. `PriorityQueue` with `Collections.reverseOrder()` is the stdlib way to get the max-oriented behavior the lecture used. The `Collections.reverseOrder()` comparator is itself an instance of exactly the kind of ordering relation discussed above.)

Step by step on `[32, 15, 2, 17, 19, 26, 41, 17, 17]`:

Phase 1, heap construction. Nine `add` calls, each O(log N). The resulting heap, in the lecture's array representation with position 0 unused, is `[_, 41, 19, 32, 17, 17, 2, 26, 15, 17]` with size 9. As a tree, 41 is the root with children 19 and 32; 19 has children 17 and 17; 32 has children 2 and 26; and the first 17 has children 15 and 17. Note that the heap array is **not** sorted; it only guarantees that each parent is at least as large as its children.

Phase 2, repeated deletion, filling `output` from the back:

- `p = 8`. `poll()` returns 41. The heap removes the root, promotes the last item (17) to the root, and sinks it: 17 swaps down past 32, then past 26. Heap becomes `[_, 32, 19, 26, 17, 17, 2, 17, 15]` with size 8. `output = [0,0,0,0,0,0,0,0,41]`.
- `p = 7`. `poll()` returns 32, with another sink. Heap size 7, `output = [0,0,0,0,0,0,0,32,41]`.
- `p = 6`. `poll()` returns 26. Heap size 6, `output = [0,0,0,0,0,0,26,32,41]`.
- `p = 5`. 19. Heap size 5.
- `p = 4`. 17. Heap size 4.
- `p = 3`. 17. Heap size 3.
- `p = 2`. 17. Heap size 2.
- `p = 1`. 15. Heap size 1.
- `p = 0`. 2. Heap size 0. `output = [2,15,17,17,17,19,26,32,41]`.

Why fill `output` from the back: the heap is max-oriented, so the first item removed is the largest, which belongs at the **last** index. Decrementing `p` places each successive (smaller) maximum one slot to the left.

Why the space complexity is Θ(N): `new int[input.length]` allocates a second array of N slots, and the `PriorityQueue` holds N boxed `Integer`s. That is the cost the in-place version eliminates.

Environment reasoning: `input` in `sort`'s frame is a reference to the caller's array, which is never mutated here. `maxHeap` is a reference to a `PriorityQueue` object whose own internal array lives separately on the heap, so during phase 1 there are two full copies of the data in memory at once. `output` is a third reference to a freshly allocated array. Returning `output` hands back that arrow; the array object outlives the frame because the heap, not the stack, owns it.

### 4. Bottom-up heapification, in place

```java
public class Heapify {
    public static void heapify(int[] a, int n) {
        for (int k = n - 1; k >= 0; k -= 1) {
            sink(a, k, n);
        }
    }

    private static void sink(int[] a, int k, int n) {
        while (2 * k + 1 < n) {
            int child = 2 * k + 1;
            if (child + 1 < n && a[child + 1] > a[child]) {
                child = child + 1;
            }
            if (a[k] >= a[child]) {
                break;
            }
            swap(a, k, child);
            k = child;
        }
    }

    private static void swap(int[] a, int p, int q) {
        int temp = a[p];
        a[p] = a[q];
        a[q] = temp;
    }
}
```

(Extra context: the lecture showed this phase as a slide-by-slide tree walk. The code is the standard 0-indexed rendering of "sink nodes in reverse level order".)

The indexing: because in-place heapsort uses position 0, node `k` has children at `2k + 1` and `2k + 2`, and the parent of node `k` is at `(k - 1) / 2`. This differs from the lecture's earlier heap implementation, which left position 0 blank so that node `k` had children at `2k` and `2k + 1`. The lecture called out the change explicitly: there is no room to leave an unused spot without allocating a new, larger array.

Step by step on `[32, 15, 2, 17, 19, 26, 41, 17, 17]` (n = 9):

- `k = 8`. `2*8 + 1 = 17`, not `< 9`, so the `while` condition fails immediately. A node with no children is already a valid heap. No effect.
- `k = 7`. Same. No effect.
- `k = 6` (value 41). `2*6 + 1 = 13`, not `< 9`. Leaf. No effect.
- `k = 5` (value 26). Leaf. No effect.
- `k = 4` (value 19). Leaf. No effect.
- `k = 3` (value 17). Children at 7 and 8, both 17. The larger child is 17, and `a[3] = 17 >= 17`, so `break`. No effect. The lecture noted all three 17s are now part of the same heap.
- `k = 2` (value 2). Children at 5 (26) and 6 (41). The larger child is 41 at index 6. `a[2] = 2 >= 41` is false, so swap indices 2 and 6: `[32, 15, 41, 17, 19, 26, 2, 17, 17]`. Then `k = 6`, whose children would be at 13 and 14, out of range, so the loop ends. **First sink that does something.**
- `k = 1` (value 15). Children at 3 (17) and 4 (19). Larger child is 19 at index 4. `15 >= 19` is false, so swap: `[32, 19, 41, 17, 15, 26, 2, 17, 17]`. Then `k = 4`, a leaf, loop ends.
- `k = 0` (value 32). Children at 1 (19) and 2 (41). Larger child is 41 at index 2. `32 >= 41` is false, so swap: `[41, 19, 32, 17, 15, 26, 2, 17, 17]`. Then `k = 2`, whose children are at 5 (26) and 6 (2). Larger child is 26. `a[2] = 32 >= 26` is true, so `break`.

Result: `[41, 19, 32, 17, 15, 26, 2, 17, 17]`, a valid max heap, produced entirely in place.

Why reverse level order is the right order: `sink(a, k, n)` assumes both subtrees of `k` are already valid heaps; it only fixes a violation at `k` itself. Processing from the last index backward guarantees that when `sink` reaches `k`, every node with a larger index (which includes both of `k`'s children and all their descendants) has already been sunk. That is the invariant the lecture stated as "after sinking, guaranteed that tree rooted at position k is a heap".

Why we pick the **larger** child: if we swapped with the smaller child, the promoted node might still be smaller than its new sibling, immediately violating the heap property at the node we just fixed.

Why the `n` parameter matters: it defines where the heap ends. During the sort phase, `n` shrinks, and the elements beyond it are the sorted suffix. Passing `n` is what keeps `sink` from reaching into the already-sorted region.

### 5. In-place heapsort

```java
public class InPlaceHeapsort {
    public static void sort(int[] a) {
        int n = a.length;
        Heapify.heapify(a, n);
        while (n > 1) {
            swap(a, 0, n - 1);
            n -= 1;
            sink(a, 0, n);
        }
    }
}
```

(Extra context: assumes the `sink` and `swap` from the previous example. The lecture presented this phase as a sequence of array snapshots.)

Step by step, continuing from the heapified `[41, 19, 32, 17, 15, 26, 2, 17, 17]` with `n = 9`:

- `swap(a, 0, 8)` puts 41 at index 8 and the last 17 at the root: `[17, 19, 32, 17, 15, 26, 2, 17, | 41]`. `n` becomes 8, so index 8 is now outside the heap and permanently in place. `sink(a, 0, 8)`: 17's children are 19 and 32; the larger is 32 at index 2, so swap. Now at index 2, children are 26 (index 5) and 2 (index 6); the larger is 26, and `17 >= 26` is false, so swap again. Heap is `[32, 19, 26, 17, 15, 17, 2, 17]`. Full array: `[32, 19, 26, 17, 15, 17, 2, 17, | 41]`.
- `swap(a, 0, 7)` puts 32 at index 7: `[17, 19, 26, 17, 15, 17, 2, | 32, 41]`. `n` becomes 7, sink the root. Result: `[26, 19, 17, 17, 15, 17, 2, | 32, 41]`.
- `swap(a, 0, 6)` puts 26 at index 6 and 2 at the root: `[2, 19, 17, 17, 15, 17, | 26, 32, 41]`. `n` becomes 6, sink: 2 swaps with 19 (the larger of 19 and 17), then at index 1 its children are 17 and 15, so 2 swaps with 17. Result: `[19, 17, 17, 2, 15, 17, | 26, 32, 41]`.
- The process repeats identically. The final array is `[2, 15, 17, 17, 17, 19, 26, 32, 41]`.

Why `while (n > 1)` and not `n > 0`: when one element remains in the heap, it is already in its correct position (everything larger is to its right), so no further work is needed. Looping down to `n = 0` would just perform a pointless `swap(a, 0, 0)`.

Why `n -= 1` comes **before** `sink`: the swap put the deleted maximum at index `n - 1`, which is now final. Shrinking `n` first excludes that slot from the heap, so `sink` will not move the sorted element back into the heap region. Reversing these two lines would corrupt the sorted suffix.

Why the space complexity is Θ(1): `n`, `k`, `child`, and `temp` are the only allocations, independent of input size. There is no second array. The loops are iterative, so there is no recursion stack either, which is the condition under which the Θ(1) figure in the summary table holds.

Environment reasoning: `a` in `sort`'s frame, `a` in `heapify`'s frame, `a` in `sink`'s frame, and `a` in `swap`'s frame are four separate boxes all holding arrows to the **same** array object on the heap. There is exactly one array in memory for the entire sort. `n` is a primitive copied into each frame, which is why `sink` cannot change `sort`'s `n`; `sort` manages the shrinking itself with `n -= 1`.

### 6. Merge

```java
public class Merge {
    public static void merge(int[] input, int lo, int mid, int hi, int[] aux) {
        int i = lo;
        int j = mid;
        for (int p = lo; p < hi; p += 1) {
            if (i >= mid) {
                aux[p] = input[j];
                j += 1;
            } else if (j >= hi) {
                aux[p] = input[i];
                i += 1;
            } else if (input[i] <= input[j]) {
                aux[p] = input[i];
                i += 1;
            } else {
                aux[p] = input[j];
                j += 1;
            }
        }
        for (int p = lo; p < hi; p += 1) {
            input[p] = aux[p];
        }
    }
}
```

(Extra context: the lecture traced the merge step with `i`, `j`, `p` pointers on slides rather than giving code. This is the standard rendering of "compare `input[i] < input[j]` (if necessary), copy the smaller item and increment `p` and `i` or `j`".)

Step by step on the lecture's post-recursion array `[2, 15, 17, 32, 17, 17, 19, 26, 41]` with `lo = 0`, `mid = 4`, `hi = 9`. The left half is indices 0 through 3 (`[2, 15, 17, 32]`) and the right half is indices 4 through 8 (`[17, 17, 19, 26, 41]`). `i = 0`, `j = 4`.

- `p = 0`. Both sides have items. `input[0] = 2 <= input[4] = 17`, so copy 2. `aux = [2, ...]`, `i = 1`, matching the lecture's trace.
- `p = 1`. `15 <= 17`, copy 15. `aux = [2, 15, ...]`, `i = 2`.
- `p = 2`. `input[2] = 17 <= input[4] = 17`. True, so copy the **left** 17. `aux = [2, 15, 17, ...]`, `i = 3`. (The `<=` here is what makes the merge stable: when two items compare equal, the one from the left half goes first, preserving their original relative order. Extra context: the lecture did not discuss stability by name in this lecture, but the `<=` versus `<` choice is exactly what controls it.)
- `p = 3`. `input[3] = 32 <= input[4] = 17` is false, so copy the right 17. `aux = [2, 15, 17, 17, ...]`, `j = 5`.
- `p = 4`. `32 <= input[5] = 17` is false, copy 17. `aux = [2, 15, 17, 17, 17, ...]`, `j = 6`.
- `p = 5`. `32 <= input[6] = 19` is false, copy 19. `j = 7`.
- `p = 6`. `32 <= input[7] = 26` is false, copy 26. `j = 8`.
- `p = 7`. `32 <= input[8] = 41` is true, copy 32. `i = 4`.
- `p = 8`. Now `i >= mid` (4 >= 4), so the **left side has run out of items**. The first branch fires and 41 is copied with **no comparison made at all**. This is exactly the "if necessary" point the lecture highlighted. `j = 9`.

`aux = [2, 15, 17, 17, 17, 19, 26, 32, 41]`, then the final loop copies it back into `input`.

Why the exhaustion checks come first: if the code tried `input[i] <= input[j]` when `i` has run past `mid`, it would read an element belonging to the right half (or past the array end), silently producing wrong output or throwing. Testing `i >= mid` and `j >= hi` before any comparison is what makes the merge correct at the boundaries.

Why `p` always advances exactly once per iteration: every iteration copies exactly one element and advances exactly one of `i` or `j`. Since the two halves together hold `hi - lo` elements and the loop runs `hi - lo` times, every element is copied exactly once. That one-pass structure is why a single merge is Θ(hi - lo), and why the whole mergesort is Θ(N log N): Θ(N) work per level, with log N levels of recursion.

Why the space complexity is Θ(N): `aux` must hold up to N elements. The lecture noted this is mergesort's one real weakness against heapsort, and that in-place mergesort exists but is very complicated with a significantly worse constant factor.

Environment reasoning: `input` and `aux` in `merge`'s frame are two arrows to two **different** array objects. The copy-back loop at the end is what makes the merged result visible through the `input` arrow, so the caller (and the recursive parent call) sees sorted data in the original array. Without that loop the sorted result would be stranded in `aux`. In a typical implementation `aux` is allocated once at the top level and the same array is reused by every recursive call, which is why the total space is Θ(N) and not Θ(N log N).

## Common Pitfalls

**Counting only adjacent inversions.** This was the single most common error in lecture: in `0 1 1 2 3 4 8 6 9 5 7`, students who answered "2" spotted 8-6 and 9-5 and stopped. An inversion is **any** out-of-order pair, adjacent or not, so 8-5, 8-7, and 6-5 also count, giving 6.

**Off-by-one in the maximum inversion count.** For N distinct elements the maximum is `(N-1) + (N-2) + ... + 1`, not `N + (N-1) + ...`. The largest element is inverted with the N-1 elements after it; the second largest adds N-2 **new** inversions (its pair with the largest was already counted); and so on. For the lecture's 11-element sequence that is `10 + 9 + ... + 1 = 55`, not 54 and not 66. Also note that duplicates reduce the achievable maximum, since equal elements are never an inversion.

**Thinking selection sort has a good best case.** It does not. Θ(N²) best **and** worst, because the algorithm always scans the full remaining array for the minimum with no sortedness check.

**Confusing the relation's `=` with `.equals`.** Under a length comparator `"the" = "get"`, but `"the".equals("get")` is `false`. Forgetting this leads to the wrong conclusion that a sorted output is unique when in fact several orderings are valid.

**Assuming any comparison rule gives a sort.** Divisibility and subset are both transitive but both fail trichotomy, so neither produces a total order. If you cannot decide between two elements, you cannot line them up.

**Counting the input in space complexity.** Dijkstra's space complexity is Θ(V), not Θ(V + E), because the graph already exists as an input. Likewise selection sort and in-place heapsort are Θ(1) even though they handle N elements: the input array is not extra memory.

**Using a min heap for in-place heapsort.** The whole trick depends on the heap and the sorted output occupying opposite ends of the array. A min heap deletes the smallest item, which belongs on the left, exactly where the heap lives, so the two regions contend for the same slots. Max heap keeps the dividing line clean.

**Mixing up the two heap array index conventions.** The lecture's earlier heap left position 0 unused, so node `k` had children at `2k` and `2k + 1`. In-place heapsort **uses** position 0 (leaving it blank would require allocating a bigger array), so node `k` has children at `2k + 1` and `2k + 2`. Applying the wrong formula silently corrupts the heap.

**Shrinking the heap after sinking instead of before.** In the deletion loop, `n -= 1` must come between the swap and the sink. If you sink with the old `n`, the just-finalized maximum is still inside the heap and will be pulled back in, destroying the sorted suffix.

**Sinking with the smaller child.** In a max heap you must swap with the **larger** child. Swapping with the smaller one can leave the promoted node below its new sibling, re-violating the property at the node you were trying to fix.

**Claiming bottom-up heapification is Θ(N log N).** O(N log N) is a correct upper bound (N sinks, each at most O(log N)), and it is what the lecture derived. But the tight bound is **Θ(N)**, flagged as "extra for experts". Most nodes are near the leaves and sink hardly at all.

**Comparing before checking for exhaustion in merge.** Once one half runs out, the remaining items must be copied with **no comparison**. Testing `input[i] <= input[j]` after `i` has passed `mid` reads data from the wrong half or past the end.

**Forgetting the copy-back in merge.** If the merged result stays in `aux` and is never written back into `input`, the recursive parent call sees unsorted data and the whole sort fails.

**Reporting mergesort as Θ(1) space.** The auxiliary array makes it Θ(N). In-place mergesort exists but, per the lecture, the algorithm is very complicated and the runtime suffers by a significant constant factor.

## Likely Exam Points

### 1. Counting inversions

**Q.** How many inversions are in `[3, 1, 4, 1, 5]`? What is the maximum possible number of inversions in a 5-element sequence of distinct values, and what arrangement achieves it?

**A.** 3 inversions: 3-1 (index 0 with index 1), 3-1 (index 0 with index 3), and 4-1 (index 2 with index 3). Note 1 and 1 are equal, so they are not an inversion with each other, and 5 is larger than everything before it so it contributes nothing. The maximum for 5 distinct elements is `4 + 3 + 2 + 1 = 10`, achieved by reverse-sorted order.

### 2. Validity of an ordering relation

**Q.** Is "is a substring of" a valid ordering relation on strings for the purpose of sorting? Justify using Knuth's laws.

**A.** No. It is transitive (if `a` is a substring of `b` and `b` of `c`, then `a` is a substring of `c`), but it fails trichotomy: for `a = "cat"` and `b = "dog"`, `a` is not a substring of `b`, `a` does not equal `b`, and `b` is not a substring of `a`. All three trichotomy options are false, exactly like divisibility with 2 and 3, so these elements are incomparable and cannot be placed in a line.

### 3. Selection sort best case

**Q.** What is the best-case runtime of selection sort, and what input achieves it?

**A.** Θ(N²), and **no** input does better. The algorithm unconditionally scans the full unsorted remainder for a minimum on every iteration, performing `(N-1) + (N-2) + ... + 1` comparisons regardless of input order. Even an already-sorted array costs Θ(N²). Contrast this with heapsort, whose best case **is** better (Θ(N), for an array of all duplicates).

### 4. Heapsort time and space

**Q.** State the worst-case runtime and space complexity of in-place heapsort, and justify each in one or two sentences.

**A.** Worst-case runtime Θ(N log N): bottom-up heapification is N sink operations each costing at most O(log N) (tight bound Θ(N), as extra-for-experts), then N deletions, each a Θ(1) root access plus an O(log N) sink, giving Θ(N log N) overall. Space Θ(1): the only extra memory is a constant number of variables such as `size`, `k`, `child`, and `temp`, because both the heap and the sorted output share the input array. Caveat: Θ(log N) if heap operations are implemented recursively, due to the call stack, which is effectively the same as Θ(1).

### 5. Why a max heap for in-place heapsort

**Q.** In-place heapsort uses a max-oriented heap. Why not a min heap?

**A.** The sorted region and the heap must occupy opposite ends of the single shared array so they never need the same slot. With a max heap, each deletion yields the largest remaining item, which belongs at the **right** end, while the heap occupies the left; the dividing line slides left and the two regions meet exactly once. With a min heap, the deleted item is the smallest and belongs at the left end, which is where the heap already lives, so the heap and the output would contend for the same side.

### 6. Bottom-up heapification by hand

**Q.** Bottom-up heapify `[4, 9, 2, 8, 1]` into a max heap (0-indexed, children of `k` at `2k+1` and `2k+2`). Show each sink that changes the array.

**A.** Nodes 4, 3, and 2 are leaves (`2*2+1 = 5`, not `< 5`), so sinking them has no effect. `k = 1` (value 9): children at 3 (8) and 4 (1); the larger is 8, and `9 >= 8`, so no change. `k = 0` (value 4): children at 1 (9) and 2 (2); the larger is 9 at index 1, and `4 >= 9` is false, so swap, giving `[9, 4, 2, 8, 1]`. Now `k = 1` with children 8 (index 3) and 1 (index 4); the larger is 8, and `4 >= 8` is false, so swap, giving `[9, 8, 2, 4, 1]`. Now `k = 3`, whose children would be at 7 and 8, out of range, so the loop ends. Result: `[9, 8, 2, 4, 1]`.

### 7. One step of in-place heapsort

**Q.** Given the max heap `[9, 8, 2, 4, 1]` with heap size 5, show the array and heap size after one deletion.

**A.** Swap the root with the last heap element: `[1, 8, 2, 4, | 9]`. Decrement the heap size to 4, fixing index 4. Sink the root 1 within the first 4 slots: children of 0 are 8 (index 1) and 2 (index 2); the larger is 8, and `1 >= 8` is false, so swap, giving `[8, 1, 2, 4, | 9]`. Now `k = 1`, child at index 3 (value 4) and index 4 is outside the heap; `1 >= 4` is false, so swap, giving `[8, 4, 2, 1, | 9]`. Final: array `[8, 4, 2, 1, 9]`, heap size 4, sorted suffix `[9]`.

### 8. Merge tracing with duplicates

**Q.** Merge the sorted halves `[2, 5, 5]` and `[1, 5, 9]`. List the order in which items are copied, and say how many comparisons were made.

**A.** With `<=` favoring the left half: compare 2 vs 1, copy **1**. Compare 2 vs 5, copy **2**. Compare 5 vs 5, `5 <= 5` is true, copy the **left 5**. Compare 5 vs 5 again, copy the **second left 5**. The left half is now exhausted, so copy the right **5** and then **9** with no comparison. Output `[1, 2, 5, 5, 5, 9]`, with 4 comparisons and 2 uncompared copies. Using `<` instead of `<=` would copy the right 5 before the left ones, producing the same multiset but swapping the relative order of equal elements.

### 9. Filling in the summary table

**Q.** Fill in best case, worst case, and space for selection sort, in-place heapsort, and mergesort, and name one practical drawback of heapsort relative to mergesort.

**A.** Selection sort: Θ(N²) / Θ(N²) / Θ(1). In-place heapsort: Θ(N) best (an array of all duplicates) / Θ(N log N) worst / Θ(1) assuming iterative heap operations. Mergesort: Θ(N log N) / Θ(N log N) / Θ(N). Heapsort's drawback: bad cache performance (a CS 61C topic), because parent-to-child index jumps scatter memory accesses, whereas mergesort's sequential scans are cache-friendly. In practice mergesort is faster than heapsort despite the same asymptotic runtime.

### 10. Space complexity bookkeeping

**Q.** A student claims naive heapsort has Θ(1) space complexity because heaps are just arrays. What is wrong, and what is the correct answer?

**A.** Naive heapsort allocates a **separate** heap holding all N items and a **separate** output array of N slots, both beyond the input, so its space complexity is Θ(N). The Θ(1) figure belongs to the **in-place** version, which stores the shrinking heap and the growing sorted suffix in the single input array, exploiting the fact that the heap loses exactly one slot each time the output gains one. Only memory beyond the input counts, and here there demonstrably is some.

## Summary

- Sorting requires an ordering relation satisfying Knuth's two laws: **trichotomy** (exactly one of `a < b`, `a = b`, `b < a`) and **transitivity** (`a < b` and `b < c` implies `a < c`). Together they are exactly what is needed to line elements up.
- **Divisibility and subset fail trichotomy** (2 and 3 do not divide each other; {1,2} and {a,b} are not subsets of each other), so neither can sort. What you get instead of a line is a lattice.
- A relation's `=` is **not** Java's `.equals`. Under a length comparator `"the" = "get"`, so `["cows","get","going","the"]` has two valid sorts.
- A **sort** is a permutation putting keys into non-decreasing order relative to the given relation: `x1 ≤ x2 ≤ ... ≤ xN`.
- An **inversion** is any out-of-order pair, adjacent or not. `0 1 1 2 3 4 8 6 9 5 7` has 6 inversions (8-6, 8-5, 8-7, 6-5, 9-5, 9-7) out of a maximum 55, achieved by reverse-sorted order. Sorting is reducing inversions to 0.
- **Time complexity** counts primitive operations, not wall-clock time. **Space complexity** counts only **extra** memory beyond the input.
- **Selection sort**: find the smallest remaining item, swap it to the front, repeat. Θ(N²) best **and** worst (no sortedness detection), Θ(1) space because it swaps in place. Wasteful because it rescans the whole remainder every iteration.
- **Naive heapsort**: insert all N items into a max heap (O(N log N)), then repeatedly delete the max (Θ(1) to select, O(log N) to remove) into the back of a separate output array. O(N log N) time but Θ(N) space.
- **N² to N log N is an exponential improvement** in one factor, since log is the inverse of exponentiation. Θ(N log N) is called **quasi-linear**.
- **In-place heapsort** exploits the fact that the heap shrinks by one slot exactly as the output grows by one: store both in the input array. **Max heap** is required so the deleted maxima accumulate at the right end while the heap stays on the left and the two regions never contend.
- **Bottom-up heapification** sinks nodes in reverse level order, so each sink can assume its subtrees are already heaps. Leaves are trivially heaps; on `[32,15,2,17,19,26,41,17,17]` only sinking 2, then 15, then 32 changes anything, yielding `[41,19,32,17,15,26,2,17,17]`. In-place heapsort uses position 0, so children of `k` are at `2k+1` and `2k+2`.
- **In-place heapsort**: O(N log N) time, Θ(1) space (Θ(log N) if recursive, effectively the same), Θ(N) best case on all-duplicate input. Bottom-up heapification is actually Θ(N) (extra for experts). Drawback: bad cache performance.
- **Mergesort**: split into two roughly even halves, mergesort each recursively, merge with `i`, `j`, `p` pointers, copying the smaller front element and skipping the comparison entirely once a side is exhausted. Θ(N log N) best and worst, Θ(N) space for the auxiliary array. In-place mergesort exists but is very complicated with a significantly worse constant factor. Faster than heapsort in practice.
