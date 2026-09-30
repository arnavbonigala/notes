<!-- Mon, Sep 28, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 14: Disjoint Sets

## Overview

This lecture kicks off the "data structures" portion of the course by deriving one specific data structure from scratch: **Disjoint Sets** (also called **Union-Find**), which solves the **Dynamic Connectivity** problem. The API is deliberately tiny: `connect(p, q)` joins two items, and `isConnected(p, q)` asks whether two items are joined, possibly through a transitive chain of connections. The real content of the lecture is not the API but the *design process*: we start with a naive "record every line" idea, reject it, discover the key insight that we only need to track **connected components** (not individual connections), and then iterate through five implementations (ListOfSetsDS, QuickFindDS, QuickUnionDS, WeightedQuickUnionDS, WeightedQuickUnionDS with path compression), each of which improves on the previous one. Along the way the lecture reviews Big O versus Big Theta (Big O is useful here precisely because runtimes depend on the shape of the input), and shows how the choice of underlying representation (the instance variables) determines both the asymptotic runtime and how ugly the code is. The end result, weighted quick union with path compression, is the standard real-world implementation, running in nearly constant time per operation.

---

## Key Concepts

### 1. Big Theta versus Big O (review)

Both are statements about *families* of functions, not about particular algorithms.

- `R(N) ∈ Θ(f(N))` means there exist positive constants `k1` and `k2` with `k1·f(N) ≤ R(N) ≤ k2·f(N)` for all `N` greater than some `N0` (i.e. for very large `N`). Informally: the **order of growth is** `f(N)`. It is a two-sided sandwich.
- `R(N) ∈ O(f(N))` means there exists a positive constant `k2` with `R(N) ≤ k2·f(N)` for all `N > N0`. Informally: the **order of growth is less than or equal to** `f(N)`. Big O is **just an upper bound**.

The slide's family table:

| | Informal meaning | Family | Members |
|---|---|---|---|
| Big Theta `Θ(f(N))` | order of growth **is** `f(N)` | `Θ(N²)` | `N²/2`, `2N²`, `N² + 38N + N` |
| Big O `O(f(N))` | order of growth is **≤** `f(N)` | `O(N²)` | `N²/2`, `2N²`, `lg(N)` |

Note that `lg(N)` is in `O(N²)` but not in `Θ(N²)`.

**Why this matters today:** several of the disjoint sets operations have a runtime that depends on the *input*, not just on `N`. For example, `ListOfSetsDS.isConnected` might find both items in the very first set it checks (fast) or might have to scan all `N` sets (slow). The worst case is `Θ(N)`, but since other cases can be much better, the lecture records it as `O(N)`: an honest upper bound that covers every case. Whenever you see `O` in today's runtime table, it means "the worst case is this, but the runtime genuinely varies."

### 2. Sets and Disjoint Sets

A **Set** is a collection of elements with no duplicates. Adding an element that is already present does nothing:

```
add(Russia); add(Mongolia); add(China); add(Mongolia)  // no effect, already present
add(Canada); contains(Mongolia)  // true
```

The lecture asserts (and will prove later in the course, when we cover hashing) that set operations are `Θ(1)`.

A **Disjoint Sets** data structure exposes only two operations:

- `connect(x, y)`: connects `x` and `y`.
- `isConnected(x, y)`: returns `true` if `x` and `y` are connected, where connections are **transitive** (they need not be direct).

The lecture's motivating example (a map of countries):

```
connect(Russia, China)
connect(Russia, Mongolia)
isConnected(China, Mongolia)?   // true, via Russia (transitive)
connect(USA, Canada)
isConnected(USA, Mongolia)?     // false
connect(Russia, USA)            // the "Bering strait crossing"
isConnected(USA, Mongolia)?     // now true
```

Note that the answer to a query can *change over time* as connections are added. That is what "dynamic" in dynamic connectivity means.

**Why anyone cares:**
- **Percolation theory**, including a computational chemistry application (the lecture mentions this shows up on a homework/project in the class).
- As a building block inside other algorithms, notably **Kruskal's algorithm** for minimum spanning trees (covered later in the course).

### 3. Simplifying assumptions

To keep the derivation clean, the lecture restricts the problem without loss of generality:

- All items are **integers** `0` through `N-1` (so `8` instead of `USA`).
- The number of items `N` is **declared in advance**, and everything starts fully disconnected.

The performance constraints we must respect:

- `N` (number of elements) can be huge: millions or hundreds of millions.
- `M` (number of method calls) can be huge.
- Calls are **interspersed**: you may not assume all `connect` calls come before all `isConnected` calls. (If you could assume that, the problem would be easier.)

The Java interface:

```java
public interface DisjointSets {
    /** Connects two items p and q. */
    void connect(int p, int q);

    /** Checks to see if two items are connected. */
    boolean isConnected(int p, int q);
}
```

There is deliberately **no** `disconnect` operation, and no way to enumerate a component.

### 4. The naive approach and why it fails

Naive idea: on `connect(p, q)`, record the individual connecting line, e.g. in a `List<Integer[]>` of pairs. After the lecture's example sequence you would hold:

```
[0, 1], [1, 2], [0, 4], [2, 4], [3, 5]
```

`connect` is easy (append a pair). But `isConnected(3, 0)` is **hard**: you have to do some computation over all of the pairs, chasing neighbors of neighbors recursively, to see whether one item is reachable from another. (This is really a graph reachability search, which we have not built yet.) The lecture stops here and pivots.

### 5. The key insight: connected components

We do not need to know *how* things are connected, only *that* they are connected. So instead of storing lines, store, for each item, the **set of all items connected to it**: its **connected component**. The connected components always form a partition of the `N` items (this is why the sets are "disjoint": every item is in exactly one component).

The lecture's trace, showing the state as a collection of sets:

```
                          {0}, {1}, {2}, {3}, {4}, {5}, {6}
connect(0, 1)             {0, 1}, {2}, {3}, {4}, {5}, {6}
connect(1, 2)             {0, 1, 2}, {3}, {4}, {5}, {6}
connect(0, 4)             {0, 1, 2, 4}, {3}, {5}, {6}
connect(3, 5)             {0, 1, 2, 4}, {3, 5}, {6}
isConnected(2, 4)         true
isConnected(3, 0)         false
connect(4, 2)             {0, 1, 2, 4}, {3, 5}, {6}     <-- no change, already together
connect(4, 6)             {0, 1, 2, 4, 6}, {3, 5}
connect(3, 6)             {0, 1, 2, 3, 4, 5, 6}
isConnected(3, 0)         true
```

Two things to notice, both of which are exam fodder:

1. `connect(4, 2)` changes **nothing**, because 4 and 2 are already in the same component. Redundant connects are no-ops at the level of connected components, even though the naive line-based approach would have stored a new line.
2. `isConnected` reduces to "are these two items in the same set?", which is far simpler than a reachability search.

Under this model, `connect(p, q)` = "merge the set containing `p` with the set containing `q`", and `isConnected(p, q)` = "are `p` and `q` in the same set".

### 6. Implementation #1: ListOfSetsDS

The most popular student answer: store a list of sets of integers, in Java `List<Set<Integer>>`, e.g. `[{0, 1, 2, 4}, {3, 5}, {6}]`.

This is intuitive, and it is also **terrible**. To do anything, you must first *find* which set an item lives in, which requires iterating over the list of sets. Worst case, nothing is connected, so there are `N` singleton sets and you scan through up to `N` sets to locate one item (and then again for the second item).

| Implementation | constructor | connect | isConnected |
|---|---|---|---|
| ListOfSetsDS | `Θ(N)` (make N sets) | `O(N)` (find each element, combine 2 sets) | `O(N)` (check up to N sets) |

The code is also genuinely complicated to write (you must find two sets, merge them, and remove one from the list, all while iterating).

**The most important lesson of this section:** the data structure choice, "list of sets," doomed us immediately to code that is both complicated *and* slow. When you implement a high-level data structure out of lower-level building blocks, your choice of instance variables deeply affects both code complexity and performance.

### 7. Implementation #2: QuickFindDS (array of set ids)

Replace the list of sets with a single `int[] id`, where `id[i]` is the **set number** (the "id") of item `i`. Items are in the same component exactly when their ids match.

```
{0, 1, 2, 4}, {3, 5}, {6}       int[] id:  4 4 4 5 4 5 6
                                 index:    0 1 2 3 4 5 6
```

The particular id value is arbitrary: `{0,1,2,4}` could just as well have id `9`. The slides pick a member of the set (here 4) because it reads nicely.

- `isConnected(p, q)`: `id[p] == id[q]`. Two array accesses, `Θ(1)`. Very fast.
- `connect(p, q)`: change **every** entry equal to `id[p]` so that it equals `id[q]`. This requires a full pass over the array: `N+2` to `2N+2` array accesses, so `Θ(N)` always (the loop never exits early).

Example, `connect(2, 3)` on the array above: every `4` becomes `5`, giving `5 5 5 5 5 5 6`.

| Implementation | constructor | connect | isConnected |
|---|---|---|---|
| ListOfSetsDS | `Θ(N)` | `O(N)` | `O(N)` |
| QuickFindDS | `Θ(N)` (initialize size-N array) | `Θ(N)` (look through entire array, update ids) | `Θ(1)` (two array accesses) |

Note `connect` is `Θ(N)` not `O(N)`: the `for` loop always visits all `N` entries, so there is no "lucky" fast case. This is a genuine improvement over ListOfSetsDS (constant-time queries, much simpler code), but `Θ(N)` per connect is still too slow for practical use. Time to do something more radical.

### 8. Implementation #3: QuickUnionDS (array of parent ids)

The design question the lecture poses: **how can we change our set representation so that merging two sets requires changing only one value?**

Answer: keep the same `int[]` instance variable, but change what the numbers *mean*. Instead of storing a set id, store each item's **parent**. Roots (items with no parent) store `-1`. This gives the array a **tree-like shape**.

```
parent:  -1  0  1  -1  0  3  -1
index:    0  1  2   3  4  5   6

Tree 1 (root 0):  0 -> children 1, 4 ; 1 -> child 2      {0, 1, 2, 4}
Tree 2 (root 3):  3 -> child 5                           {3, 5}
Tree 3 (root 6):  6 alone                                {6}
```

The lecture calls this "innocuous sounding, seemingly arbitrary," but notes it unlocks a whole universe of theory.

Now, membership is determined by **which root you reach when you climb**. So:

- `isConnected(p, q)`: `root(p) == root(q)`.
- `connect(p, q)`: find `root(p)`, find `root(q)`, then `parent[root(p)] = root(q)`. Exactly **one** value changes.

**Why you cannot just set `parent[5] = 2`** (the lecture's hint question for `connect(5, 2)`): if 5's parent becomes 2 directly, then 3, which was 5's root, is orphaned: "we've lost 3!" 3 would become a singleton component and would no longer be connected to 5. You must relink at the **root** level, not the item level. So `connect(5, 2)` does `parent[root(5)] = root(2)`, i.e. `parent[3] = 0`, which drags 3 and its whole subtree (including 5) under 0.

**The defect:** trees can get tall. If we always hang the first item's tree below the second item's tree, then

```
connect(4, 3); connect(3, 2); connect(2, 1); connect(1, 0)
```

produces a single path (a "spindly" tree) of height `M` after `M` operations. Climbing to the root is then `Θ(N)` in the worst case, so both `connect` and `isConnected` are `Θ(N)` in the worst case. In the worst case, half the items are in one long path and half are in another, and connecting them requires finding both roots.

| Implementation | constructor | connect | isConnected |
|---|---|---|---|
| ListOfSetsDS | `Θ(N)` | `O(N)` | `O(N)` |
| QuickFindDS | `Θ(N)` | `Θ(N)` | `Θ(1)` |
| QuickUnionDS | `Θ(N)` | `O(N)` (climb tree to find root) | `O(N)` (climb 2 trees to compare roots) |

Here `O` rather than `Θ` because the runtime can be anywhere between constant (already at a root) and linear (a spindly tree). Note that QuickUnion is arguably *worse* than QuickFind overall: `isConnected` degraded from `Θ(1)` to `O(N)`. But it is the foundation of everything good that follows.

**Observation that drives the next step:** everything would be fine if we just kept our trees balanced.

### 9. Implementation #4: WeightedQuickUnionDS

When `connect(2, 5)` gives you a choice of which root becomes the child, the choice matters:

```
connect(2, 5), where 0 is root of {0,1,2,4} (height 2 tree) and 3 is root of {3,5}

A. Make 5's root (3) a child of 2's root (0):  resulting height 2   <-- better
B. Make 2's root (0) a child of 5's root (3):  resulting height 3
```

Since root-finding costs time proportional to tree height, we want to **minimize height**. One approach is to track each tree's height and always link the shorter tree below the taller one, breaking ties arbitrarily. But tracking height becomes difficult once we add path compression. Fortunately, tracking the tree's **size** (also called **weight**, meaning the total number of items in the tree) works just as well asymptotically.

**Weighted Quick Union rule: always link the root of the smaller tree to the root of the larger tree.**

The lecture's clicker example:

```
parent: -1  0  0  0  0  0  -1  6  6  8
index:   0  1  2  3  4  5   6  7  8  9

Tree rooted at 0: children 1,2,3,4,5              -> size 6
Tree rooted at 6: children 7,8 ; 8 -> child 9     -> size 4

connect(3, 8): which entry of parent[] changes?
```

Answer: `parent[6]`. We never change `parent[3]` or `parent[8]`, because that would rip an individual item out of its tree and break the other members' connections. We change one of the two *roots*, and since 6's tree (size 4) is smaller than 0's tree (size 6), 6 becomes a child of 0, giving `parent[6] = 0`.

**Implementing it** requires minimal changes: same `parent[]` array, `isConnected` unchanged, and `connect` must track sizes. Two common approaches:

1. Store `-weight` in the root's slot instead of `-1`. So `parent = [-2, -1, -1, -1, -1, 0, -4, 6, 6, 8]` means the tree rooted at 0 has 2 items and the tree rooted at 6 has 4 items. A negative entry still identifies a root.
2. Keep a separate parallel `size[]` array: `size = [2, 1, 1, 1, 1, 1, 4, 1, 2, 1]`.

**Worst-case height analysis (proof sketch from the lecture).** Build the shortest possible tree of each height using the weighted rule:

| N (items) | worst-case height |
|---|---|
| 1 | 0 |
| 2 | 1 |
| 4 | 2 |
| 8 | 3 |
| 16 | 4 |

To grow the height by one, you must link two trees of equal size and equal (current maximum) height, because linking a *smaller* tree under a larger one never increases the larger tree's height. So each increment of height requires **doubling** the number of items. You cannot build a height-2 tree with 3 items, and you cannot build a height-3 tree with 5, 6, or 7 items. Therefore the worst-case height is `Θ(log N)`.

| Implementation | constructor | connect | isConnected |
|---|---|---|---|
| ListOfSetsDS | `Θ(N)` | `O(N)` | `O(N)` |
| QuickFindDS | `Θ(N)` | `Θ(N)` | `Θ(1)` |
| QuickUnionDS | `Θ(N)` | `O(N)` | `O(N)` |
| WeightedQuickUnionDS | `Θ(N)` | `O(log N)` | `O(log N)` |

**Why weights instead of heights?** A "HeightedQuickUnionDS" has asymptotically the *same* worst-case performance, `Θ(log N)`. You occasionally get a slightly shorter tree, but there is no asymptotic gain, and the resulting code is more complicated. More complexity with no asymptotic benefit is a bad trade.

**What we have achieved.** Performing `M` operations on a disjoint sets object with `N` elements:

- Naive (ListOfSetsDS): `O(N + MN)`, i.e. `O(MN)`.
- WeightedQuickUnionDS: `O(N + M log N)`.
- For `N = 10⁹` and `M = 10⁹`, that is roughly the difference between **30 years and 6 seconds**.

Key point: a good data structure unlocks solutions to problems that could otherwise not be solved at all. WQU is good enough for all practical uses. But could we theoretically do better?

### 10. Implementation #5: WQU with Path Compression (a CS 170 spoiler)

**Clever idea:** whenever you climb the tree to find a root (which you must do anyway, in both `connect` and `isConnected`), **tie every node you passed directly to the root**.

Starting from a worst-case WQU tree, `isConnected(15, 10)` climbs `15 -> 11 -> 5 -> 1 -> 0` and `10 -> 3 -> 0`. All of `15, 11, 5, 1, 3, 10` get `parent` set to `0`.

Why this is legitimate: the exact sequence of connections carries no information we care about. Two trees with the same set of nodes and the same root represent the **same connected component**, so they are semantically equivalent. We are free to rearrange the internal shape.

Why this is cheap: we already paid for the climb. Tying the nodes to the root is the same order of growth as the climb itself, so the additional cost is insignificant. You pay a little now, and every future query along that path is faster.

**Intuition:** as the number of nodes `N` grows, trees tend to get taller; but as the number of operations `M` grows, the trees tend to get **shorter**, because every operation flattens the paths it touches. For enough operations, tree height shrinks toward 1. The data structure gets more efficient the more you use it. (The lecture notes this has a vaguely similar flavor to memoization from CS 61A / CS 88.)

A striking consequence: the worst-case WQU tree we started the exercise with is **impossible to generate** if you are using path compression. Path compression rules out certain structures entirely, and that is part of why the runtime is so good.

**The runtime.** In CS 170 you will show that with path compression, each `connect` or `isConnected` operation takes on average `lg* N` time, where `lg* N` is how many times you must press the `log₂` button on a calculator before reaching a number that is 1 or less:

| N | lg* N |
|---|---|
| 1 | 0 |
| 2 | 1 |
| 4 | 2 |
| 16 | 3 |
| 65536 | 4 |
| 2^65536 | 5 |

So `M` operations on `N` nodes take `O(M lg* N)`, and `lg* N ≤ 5` for any realistic input. An even tighter bound shows each operation takes on average `α(N)` time, where `α` is the **inverse Ackermann function**, which grows even more slowly (to reach 5 you would need `N` around a tower of 2's of height 65536). See Bob Tarjan's "Efficiency of a Good But Not Linear Set Union Algorithm," written while he was at UC Berkeley in 1975. The title says it all: this is nearly linear, but not quite linear.

### 11. The iterative design process, summarized

The end result is the standard way disjoint sets are implemented today: **quick union plus path compression**. The ideas that got us there:

- Represent sets as **connected components** (don't track individual connections).
  - **ListOfSetsDS**: store connected components as a `List<Set<Integer>>` (slow, complicated).
  - **QuickFindDS**: store connected components as **set ids**.
  - **QuickUnionDS**: store connected components as **parent ids**.
    - **WeightedQuickUnionDS**: also track the **size** of each set, and use size to decide the new root.
      - **WQU with Path Compression**: on calls to `connect` and `isConnected`, set the parent id to the root for all items seen.

Total runtime for `M` operations on `N` elements (an operation being a call to `connect` or `isConnected`):

| Implementation | Runtime for M operations |
|---|---|
| ListOfSetsDS | `O(NM)` |
| QuickFindDS | `Θ(NM)` |
| QuickUnionDS | `O(NM)` |
| WeightedQuickUnionDS | `O(M log N)` |
| WQU with Path Compression | `O(M α(N))` |

All of these runtimes come down to the time it takes to climb the tree, which is exactly what each successive optimization improved.

---

## Definitions

- **Set**: a collection of elements with no duplicates; adding an element already present has no effect.
- **Disjoint Sets (Union-Find)**: a data structure over a fixed collection of `N` items supporting `connect(p, q)` and `isConnected(p, q)`, where connectedness is transitive. The name reflects that the items are partitioned into non-overlapping (disjoint) sets.
- **Dynamic Connectivity problem**: the problem Disjoint Sets solves: answer connectivity queries about a set of items while connections are being added over time, with queries and connections interspersed.
- **Connected component (of an item)**: the set of all items that are connected to that item (including itself). Components partition the items.
- **`connect(p, q)`**: connects `p` and `q`; equivalently, merges the connected component of `p` with the connected component of `q`. A no-op if they are already in the same component.
- **`isConnected(p, q)`**: returns `true` if `p` and `q` are in the same connected component, whether by a direct connection or a transitive chain.
- **Big Theta, `Θ(f(N))`**: the family of functions `R(N)` for which there exist positive constants `k1, k2` with `k1·f(N) ≤ R(N) ≤ k2·f(N)` for all `N > N0`. Informally, "order of growth is `f(N)`."
- **Big O, `O(f(N))`**: the family of functions `R(N)` for which there exists a positive constant `k2` with `R(N) ≤ k2·f(N)` for all `N > N0`. Informally, "order of growth is less than or equal to `f(N)`." An upper bound only.
- **id (in QuickFind)**: an arbitrary integer label shared by exactly the members of one connected component; `id[i]` is the label of item `i`'s component.
- **parent (in QuickUnion)**: `parent[i]` is the index of item `i`'s parent in its tree, or `-1` if `i` is a root. (The optional textbook instead stores an item's own index as its parent when it is a root.)
- **root (of an item)**: the item reached by repeatedly following `parent` links until reaching an item with no parent. Two items are connected exactly when they have the same root. Called `find` in the textbook and in the lecture's final code.
- **Size / weight (of a tree)**: the total number of items in that tree. The two words mean the same thing here.
- **Height (of a tree)**: the number of links on the longest root-to-leaf path. A single node has height 0.
- **Weighted Quick Union**: quick union with the rule that the root of the smaller (fewer items) tree is always linked under the root of the larger tree, ties broken arbitrarily.
- **Path compression**: during any root-finding climb, resetting the `parent` of every node visited to point directly at the root.
- **`lg* N` (log star)**: the number of times you must apply `log₂` to `N` before obtaining a value `≤ 1`. At most 5 for any realistic input.
- **`α(N)` (inverse Ackermann function)**: an even more slowly growing function than `lg*`, giving the tightest known amortized bound on WQU with path compression.
- **N and M (as used in the runtime tables)**: `N` is the number of elements in the disjoint sets object; `M` is the number of operations performed, where an operation is a call to `connect` or `isConnected`.

---

## Worked Examples

### Example 1: Tracing the lecture's canonical sequence

```java
DisjointSets ds = new DisjointSets(7);   // items 0..6, all disconnected
ds.connect(0, 1);
ds.connect(1, 2);
ds.connect(0, 4);
ds.connect(3, 5);
ds.isConnected(2, 4);   // true
ds.isConnected(3, 0);   // false
ds.connect(4, 2);
ds.connect(4, 6);
ds.connect(3, 6);
ds.isConnected(3, 0);   // true
```

Step by step, in terms of connected components:

1. Start: `{0} {1} {2} {3} {4} {5} {6}`. Seven singleton components.
2. `connect(0, 1)`: merge `{0}` and `{1}` giving `{0,1} {2} {3} {4} {5} {6}`.
3. `connect(1, 2)`: 1's component is `{0,1}`, so we get `{0,1,2} {3} {4} {5} {6}`. Notice 0 and 2 are now connected even though we never called `connect(0, 2)`: that is transitivity.
4. `connect(0, 4)`: `{0,1,2,4} {3} {5} {6}`.
5. `connect(3, 5)`: `{0,1,2,4} {3,5} {6}`.
6. `isConnected(2, 4)`: both are in `{0,1,2,4}`, so `true`.
7. `isConnected(3, 0)`: 3 is in `{3,5}`, 0 is in `{0,1,2,4}`, different components, so `false`.
8. `connect(4, 2)`: both are already in `{0,1,2,4}`. **Nothing changes.** State stays `{0,1,2,4} {3,5} {6}`. This is the case people forget: a connect between two already-connected items is a no-op.
9. `connect(4, 6)`: `{0,1,2,4,6} {3,5}`.
10. `connect(3, 6)`: merges the last two, giving `{0,1,2,3,4,5,6}`.
11. `isConnected(3, 0)`: now `true`. The same query returned `false` in step 7. Answers change over time, which is the "dynamic" in dynamic connectivity.

### Example 2: QuickFindDS

```java
public class QuickFindDS implements DisjointSets {
    private int[] id;

    public QuickFindDS(int N) {
        id = new int[N];
        for (int i = 0; i < N; i++) {
            id[i] = i;     // each item starts in its own set
        }
    }

    public boolean isConnected(int p, int q) {
        return id[p] == id[q];          // two array accesses: Theta(1)
    }

    public void connect(int p, int q) {
        int pid = id[p];                // cache BEFORE the loop
        int qid = id[q];
        for (int i = 0; i < id.length; i++) {
            if (id[i] == pid) {
                id[i] = qid;
            }
        }
    }                                   // N+2 to 2N+2 array accesses: Theta(N)
}
```

(Extra context: the slide's constructor writes `id[i] = -1`, which would make `isConnected` return `true` for every pair at startup. The version above, `id[i] = i`, is what actually makes the invariant "same id means same component" hold from the start. Treat the slide line as a typo.)

**Box-and-pointer / environment reasoning in words.** `id` is one reference variable in the `QuickFindDS` object, pointing at a single array object on the heap. That array object is a row of `N` boxes, each holding a primitive `int`, not a reference. There are no `Node` objects and no pointers between items: the "structure" is entirely encoded in the *values* in the boxes. This is why the whole data structure is so memory-efficient, and also why `connect` is forced to do a linear scan: nothing in the array tells you *where* the members of a given set are, so you must look at every box.

**Trace of `connect(2, 3)`** starting from `id = [4, 4, 4, 5, 4, 5, 6]`:

- `pid = id[2] = 4`, `qid = id[3] = 5`. Both are read *before* the loop.
- `i = 0`: `id[0]` is 4, equals `pid`, so set `id[0] = 5`.
- `i = 1`: 4 matches, set to 5.
- `i = 2`: 4 matches, set to 5.
- `i = 3`: 5 does not match 4, skip.
- `i = 4`: 4 matches, set to 5.
- `i = 5`: 5 does not match, skip.
- `i = 6`: 6 does not match, skip.
- Final: `id = [5, 5, 5, 5, 5, 5, 6]`, representing `{0,1,2,3,4,5} {6}`.

Then `isConnected(3, 6)` evaluates `id[3] == id[6]`, that is `5 == 6`, which is `false`. Two array reads, done.

### Example 3: QuickUnionDS, and why `connect` must relink roots

```java
public class QuickUnionDS implements DisjointSets {
    private int[] parent;

    public QuickUnionDS(int N) {
        parent = new int[N];
        for (int i = 0; i < N; i++) {
            parent[i] = -1;             // -1 marks a root
        }
    }

    /** Climbs to the root of p's tree. */
    private int find(int p) {
        int r = p;
        while (parent[r] >= 0) {
            r = parent[r];
        }
        return r;
    }

    public boolean isConnected(int p, int q) {
        return find(p) == find(q);
    }

    public void connect(int p, int q) {
        int i = find(p);
        int j = find(q);
        parent[i] = j;                  // exactly ONE value changes
    }
}
```

(The lecture's earlier slides call this helper `root`; the final slides and the optional textbook call it `find`. They are the same thing: `find` finds the root.)

**Trace of `connect(5, 2)`** starting from `parent = [-1, 0, 1, -1, 0, 3, -1]`, i.e. trees `{0 -> 1, 4; 1 -> 2}`, `{3 -> 5}`, `{6}`:

- `find(5)`: `r = 5`. `parent[5] = 3 >= 0`, so `r = 3`. `parent[3] = -1 < 0`, stop. Returns `3`.
- `find(2)`: `r = 2`. `parent[2] = 1`, so `r = 1`. `parent[1] = 0`, so `r = 0`. `parent[0] = -1`, stop. Returns `0`. Note this climb took 2 hops, which is the cost we will attack with weighting and compression.
- `parent[3] = 0`. New array: `[-1, 0, 1, 0, 0, 3, -1]`.

**Why not `parent[5] = 2`?** If we set 5's parent directly, the array becomes `[-1, 0, 1, -1, 0, 2, -1]`. Now 5 sits under 2 (fine), but 3 is still a root with no children: **we've lost 3**. 3 was connected to 5 before this call, and now it is not. The invariant "an item's component is the set of all items sharing its root" is broken. Relinking at the root level drags the entire subtree along, preserving every existing connection.

**Worst case.** With the rule "always hang the first item's tree below the second item's," the sequence

```java
ds.connect(4, 3);
ds.connect(3, 2);
ds.connect(2, 1);
ds.connect(1, 0);
```

produces a single chain `0 -> 1 -> 2 -> 3 -> 4`, height 4 after 4 operations. In general, `M` such operations give height `M`. For `N` items, `find` is `Θ(N)` in the worst case, so both `connect` and `isConnected` are `Θ(N)` in the worst case. To ask whether 4 and 3 are connected you climb `N` links for one and `N-1` for the other.

### Example 4: Weighted Quick Union, `connect(3, 8)`

Given:

```
parent: -1  0  0  0  0  0  -1  6  6  8
index:   0  1  2  3  4  5   6  7  8  9
```

Tree A: root 0, children 1, 2, 3, 4, 5. Size 6, height 1.
Tree B: root 6, children 7 and 8; 8's child is 9. Size 4, height 2.

`connect(3, 8)`:

1. `find(3)`: `parent[3] = 0`, then `parent[0] = -1`, so root is `0`.
2. `find(8)`: `parent[8] = 6`, then `parent[6] = -1`, so root is `6`.
3. Compare sizes: tree at 0 has 6 items, tree at 6 has 4 items. The smaller tree's root is 6.
4. `parent[6] = 0`.

Result: `parent = [-1, 0, 0, 0, 0, 0, 0, 6, 6, 8]`. The merged tree has height 3 (`0 -> 6 -> 8 -> 9`), whereas the other choice (`parent[0] = 6`) would have given height 3 as well here, but in general the weighted rule is what bounds height by `log N`.

The four candidate answers and why three are wrong: `parent[3]` and `parent[8]` are wrong because changing a non-root's parent rips that single item out of its tree and disconnects it from its former siblings. `parent[0]` is wrong because that would put the *larger* tree under the smaller one, violating the weighting rule.

**A weighted version of `connect`** (extra context: the lecture deliberately leaves the implementation to lab, but here is the shape using the "store `-weight` at the root" trick):

```java
public void connect(int p, int q) {
    int i = find(p);
    int j = find(q);
    if (i == j) { return; }            // already connected, do nothing
    int sizeI = -parent[i];            // roots store -weight
    int sizeJ = -parent[j];
    if (sizeI < sizeJ) {
        parent[i] = j;                 // smaller (i) goes under larger (j)
        parent[j] = -(sizeI + sizeJ);
    } else {
        parent[j] = i;
        parent[i] = -(sizeI + sizeJ);
    }
}
```

Note the `if (i == j) return;` guard. Without it, you would write `parent[i] = i`, creating a self-loop that makes `find` spin forever (since `parent[i] >= 0` would always hold). This is a real bug people hit on the lab.

### Example 5: Path compression, `isConnected(14, 13)`

The lecture's tree after a first compressing query (`isConnected(15, 10)`) is:

```
                     0
   /    /    /   /       \    \    \    \
  15   11    5   1        2    3   10    4
             |   | \      | \
            12   6  7     8   9
                 |        |
                13       14
```

Now call `isConnected(14, 13)`:

1. `find(14)` climbs `14 -> 8 -> 2 -> 0`. Visited on the way: 14, 8, 2. Root is 0.
2. `find(13)` climbs `13 -> 6 -> 1 -> 0`. Visited: 13, 6, 1 (1 is already a child of 0). Root is 0.
3. Roots match, so return `true`.
4. **Side effect:** every node seen on both paths is tied directly to the root. `parent[14] = 0`, `parent[8] = 0`, `parent[2] = 0`, `parent[13] = 0`, `parent[6] = 0`, `parent[1] = 0`.

Resulting tree:

```
                              0
 /    /   /   /   /   /   /   /   \   \    \    \
15   11   5   1  13   6  14   8    2   3   10    4
          |                        |
         12                        7*   9
```

Matching the slide: 0's children are now `15, 11, 5, 1, 13, 6, 14, 8, 2, 3, 10, 4`, with 5 keeping child 12, 1 keeping child 7, and 2 keeping child 9. (Nodes *not* on the two climb paths keep their old parents; only visited nodes get relinked.)

The tree went from height 3 to height 2 as a side effect of answering a query. The answer `true` is unchanged, and the *meaning* of the structure is unchanged: 0's tree still contains exactly the same items. A few more strategic queries and the tree would be completely flat.

**What path compression looks like in code** (extra context, since the lecture describes it rather than showing it):

```java
private int find(int p) {
    int r = p;
    while (parent[r] >= 0) {        // first pass: locate the root
        r = parent[r];
    }
    int curr = p;
    while (parent[curr] >= 0) {     // second pass: relink everything seen to r
        int next = parent[curr];
        parent[curr] = r;
        curr = next;
    }
    return r;
}
```

The second loop costs the same order of growth as the first, which is exactly why the lecture says "additional cost is insignificant (same order of growth)." Note the `int next = parent[curr]` before overwriting: if you overwrite first you lose your place in the chain.

---

## Common Pitfalls

1. **Confusing `O` and `Θ` in the runtime table.** `ListOfSetsDS.isConnected` is `O(N)` because a lucky input finishes in constant time, but `QuickFindDS.connect` is `Θ(N)` because its `for` loop always scans the entire array with no early exit. Writing `O` where `Θ` is correct is not *wrong* (an upper bound is still true), but writing `Θ` where only `O` holds *is* wrong.
2. **Setting `parent[p]` instead of `parent[root(p)]` in `connect`.** This orphans `p`'s former root and its subtree, silently breaking existing connections. The lecture's "we've lost 3!" slide exists to burn this in.
3. **Forgetting that `connect` on already-connected items is a no-op.** In the trace, `connect(4, 2)` changes nothing. In code, forgetting the `if (find(p) == find(q)) return;` guard in a weighted implementation can produce a root whose parent is itself, causing `find` to loop forever or corrupt the stored weight.
4. **Mixing up "weight" and "height."** Weight (= size) is the number of items in the tree. Height is the longest root-to-leaf path length. WQU links by *weight* but the thing we care about is *height*. Linking by weight happens to bound height at `log N` anyway.
5. **Assuming "link by height" is better than "link by weight."** They are asymptotically identical, both `Θ(log N)`, and the height version is more complex to code, plus height is difficult to maintain once path compression is added. This is a classic trap question.
6. **Thinking the tree in `parent[]` is the "shape of the connections."** It is not. The tree shape is an implementation artifact. Two very different tree shapes over the same item set represent the identical disjoint sets state. This is precisely the freedom that makes path compression legal.
7. **Thinking path compression changes answers.** It never does. It only changes representation, and therefore future runtime.
8. **Believing path compression makes every operation `Θ(1)`.** The bound is *amortized/average* over `M` operations (`O(M lg* N)` or `O(M α(N))` total); an individual early operation can still climb a `log N`-tall tree.
9. **Confusing the root sentinel conventions.** The lecture uses `-1` (or `-weight`) at roots; the optional textbook stores the root's own index as its parent. Both are fine, but the loop condition differs: `while (parent[r] >= 0)` versus `while (parent[r] != r)`. Pick one and be consistent, and read exam problems carefully to see which they use.
10. **Writing the `find` helper's condition as `while (parent[r] != -1)` when you have switched to storing `-weight`.** A root with weight 4 stores `-4`, not `-1`, so that check fails. Use `>= 0` to test "is not a root."
11. **In `QuickFindDS.connect`, reading `id[p]` inside the loop instead of caching it first.** Once the loop overwrites `id[p]`, the comparison target changes mid-loop and you merge the wrong items.
12. **Assuming connects and queries are separated.** The problem statement explicitly forbids this. Any solution that preprocesses all connections first is invalid for dynamic connectivity.

---

## Likely Exam Points

### 1. Fill in the performance table

**Q.** Give the worst-case runtime of `connect` and `isConnected` for QuickFindDS and QuickUnionDS on `N` items, using `Θ` where justified and `O` otherwise. Explain one place where `Θ` is *not* appropriate.

**A.** QuickFindDS: `connect` is `Θ(N)`, `isConnected` is `Θ(1)`. QuickUnionDS: `connect` is `O(N)`, `isConnected` is `O(N)`. `Θ` is not appropriate for QuickUnion because the runtime depends on tree shape: if `p` and `q` are already roots, `find` is constant time, so the runtime ranges from constant to linear and only an upper bound holds for all inputs. QuickFind's `connect`, by contrast, always executes a full `N`-iteration loop, so `Θ(N)` is exact.

### 2. Trace a sequence on Weighted Quick Union and give the parent array

**Q.** Starting from `WeightedQuickUnionDS ds = new WeightedQuickUnionDS(6)` (using `-1` for roots), give `parent[]` after `connect(0, 1); connect(2, 3); connect(0, 2); connect(4, 5); connect(0, 4);`. Break ties by making the *second* argument's root the new root.

**A.**
- Start: `[-1, -1, -1, -1, -1, -1]`.
- `connect(0, 1)`: roots 0 and 1, sizes 1 and 1, tie, so 1 becomes root: `parent[0] = 1` -> `[1, -1, -1, -1, -1, -1]`. Sets: `{0,1} {2} {3} {4} {5}`.
- `connect(2, 3)`: tie, 3 becomes root: `parent[2] = 3` -> `[1, -1, 3, -1, -1, -1]`.
- `connect(0, 2)`: `find(0) = 1` (size 2), `find(2) = 3` (size 2), tie, so 3 becomes root: `parent[1] = 3` -> `[1, 3, 3, -1, -1, -1]`. Tree at 3 has size 4, height 2 (`3 -> 1 -> 0`).
- `connect(4, 5)`: tie, `parent[4] = 5` -> `[1, 3, 3, -1, 5, -1]`.
- `connect(0, 4)`: `find(0) = 3` (size 4), `find(4) = 5` (size 2). The **smaller** tree's root (5) goes under the larger (3), regardless of argument order: `parent[5] = 3` -> `[1, 3, 3, -1, 5, 3]`.

Final: `parent = [1, 3, 3, -1, 5, 3]`, one component containing all 6 items, root 3.

### 3. "Which entry of `parent[]` changes?"

**Q.** With `parent = [-1, 0, 0, 0, 0, 0, -1, 6, 6, 8]` and weighted quick union, which entry (or entries) change on `connect(3, 8)`?

**A.** Only `parent[6]`, which becomes `0`. We relink roots, not the arguments themselves, and since 6's tree has 4 items while 0's has 6, the smaller root (6) is linked under the larger root (0). Changing `parent[3]` or `parent[8]` would disconnect an individual item from its component; changing `parent[0]` would violate the weighting rule.

### 4. Minimum items for a given height under WQU

**Q.** Using weighted quick union, what is the minimum number of items needed to build a tree of height 4? Can you build a height-3 tree with 7 items?

**A.** 16 items. Each time the height increases by one, you must link two trees of equal size and equal maximum height, since a strictly smaller tree linked under a larger one never increases the larger tree's height. So the minimum size doubles per unit of height: `N = 1, 2, 4, 8, 16` for heights `0, 1, 2, 3, 4`. You cannot build a height-3 tree with 7 items; you need at least 8. This is why the worst-case height is `Θ(log N)`.

### 5. Draw the tree after path compression

**Q.** Given the tree with root 0, children `{1, 2}`; 1's children `{3}`; 3's children `{5}`; 2's children `{4}`, draw the tree after `isConnected(5, 4)` using WQU with path compression.

**A.** `find(5)` climbs `5 -> 3 -> 1 -> 0`, so 5, 3, and 1 all get `parent = 0`. `find(4)` climbs `4 -> 2 -> 0`, so 4 and 2 get `parent = 0`. The result is a flat tree: root 0 with children `1, 2, 3, 4, 5`. The return value is `true`, and no connectivity information changed; only the representation did.

### 6. Why weight rather than height?

**Q.** Your friend proposes "HeightedQuickUnionDS," which tracks tree height and links the shorter tree under the taller one. Is this asymptotically better than WeightedQuickUnionDS? Should you use it?

**A.** It is not asymptotically better: both have worst-case tree height `Θ(log N)`, so both give `O(log N)` per operation. You should not use it: the code is more complicated with no asymptotic performance gain, and (per the lecture) tracking height becomes difficult once you add path compression, since compression changes heights in ways that are awkward to maintain.

### 7. Total runtime for M operations

**Q.** You perform `M` operations on a disjoint sets object with `N` items. Give the total runtime for ListOfSetsDS, WeightedQuickUnionDS, and WQU with path compression. For `N = M = 10⁹`, roughly how do the first two compare?

**A.** ListOfSetsDS: `O(N + MN) = O(MN)`. WeightedQuickUnionDS: `O(N + M log N)`. WQU with path compression: `O(M α(N))` (equivalently `O(M lg* N)` for the looser bound). For `N = M = 10⁹`, the difference between the naive and the weighted version is roughly 30 years versus 6 seconds: a good data structure makes otherwise-impossible problems solvable.

### 8. `lg*` values

**Q.** What is `lg*(65536)`? What is the largest `N` for which `lg* N ≤ 4`?

**A.** `lg*(65536) = 4`: `log₂ 65536 = 16`, `log₂ 16 = 4`, `log₂ 4 = 2`, `log₂ 2 = 1`, which is `≤ 1` after 4 presses. `lg* N ≤ 4` for all `N` up to `2^65536 - 1`; at `N = 2^65536` it becomes 5. Consequently `lg* N ≤ 5` for any realistic input.

### 9. Which states are reachable?

**Q.** True or false: the worst-case (tall) weighted-quick-union tree shown in lecture can also arise if you use WQU **with** path compression.

**A.** False. The lecture explicitly notes that the tall structure used in the exercise is impossible to generate under path compression. Every operation that builds depth also flattens the path it climbed, so certain deep configurations can never be produced. This structural restriction is part of why the amortized bound is so strong.

### 10. Conceptual: what abstraction did we choose, and why?

**Q.** Explain why the lecture rejects storing the list of individual connections, and what it stores instead.

**A.** Storing individual connections (a `List<Integer[]>` of pairs) makes `connect` trivial but makes `isConnected` hard: determining whether one item is reachable from another requires a search over all recorded pairs. Instead we store **connected components**: for each item, which set of mutually-connected items it belongs to. *How* items are connected is information we do not need; only *whether* they are. With this abstraction, `isConnected` becomes "same set?" and `connect` becomes "merge two sets."

---

## Summary

- **Disjoint Sets / Union-Find** solves **dynamic connectivity**: `connect(p, q)` and `isConnected(p, q)`, with transitive connections, `N` fixed in advance, items as integers `0..N-1`, and calls interspersed.
- **Big O is an upper bound; Big Theta is an order of growth.** Big O is the right tool today because several operations' runtimes depend on the shape of the data, not just on `N`.
- **Key insight:** do not record individual connections. Record **connected components**. A `connect` between already-connected items is a no-op.
- **ListOfSetsDS** (`List<Set<Integer>>`): constructor `Θ(N)`, `connect` `O(N)`, `isConnected` `O(N)`. Intuitive but complicated and slow. Lesson: a bad choice of underlying abstraction dooms you immediately.
- **QuickFindDS** (`int[] id`, value = set id): `isConnected` `Θ(1)` (two array accesses), `connect` `Θ(N)` (rewrite every matching id). Simple, fast queries, unacceptably slow connects.
- **QuickUnionDS** (`int[] parent`, `-1` at roots, tree-shaped): `connect` changes exactly **one** value, `parent[find(p)] = find(q)`. Both operations `O(N)` because trees can become spindly (height `M` after `M` operations). Never relink a non-root: you would orphan its former root's subtree.
- **WeightedQuickUnionDS**: track each tree's **size/weight** and always link the smaller tree's root under the larger tree's root. Worst-case height is `Θ(log N)` because growing the height by one requires doubling the item count. Both operations `O(log N)`. Store weight either as `-weight` at the root or in a parallel `size[]` array.
- **Weight, not height**, because heighted quick union is asymptotically identical (`Θ(log N)`) but more complicated, and height is hard to maintain under path compression.
- **Path compression**: during every root-finding climb, point every node visited directly at the root. Legal because tree shape carries no semantic information; cheap because it is the same order of growth as the climb you already performed. The structure gets faster the more you use it.
- **Final performance for `M` operations on `N` items:** ListOfSetsDS `O(NM)`, QuickFindDS `Θ(NM)`, QuickUnionDS `O(NM)`, WeightedQuickUnionDS `O(M log N)`, WQU with path compression `O(M lg* N)` and more tightly `O(M α(N))`. `lg* N ≤ 5` for any realistic input.
- For `N = M = 10⁹`, naive versus weighted is roughly **30 years versus 6 seconds**: good data structures unlock problems that would otherwise be unsolvable.
- The end product of this iterative design process, **quick union plus path compression**, is the standard real-world implementation. It is nearly linear but not quite (Tarjan, "Efficiency of a Good But Not Linear Set Union Algorithm," UC Berkeley, 1975).
