<!-- Mon, Oct 12, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 20: Heaps, Priority Queues, and Tree Representations

## Overview

This lecture closes out the search-tree arc (BSTs, 2-3/B-trees, LLRBs) and pivots to a new abstract data type: the **priority queue (PQ)**, which tracks and removes the "smallest" (or "best") item seen so far rather than supporting lookup by key identity. After motivating the ADT with a concrete problem (keeping the M highest-energy particles from a detector stream using only Θ(M) memory instead of Θ(N)), the lecture surveys bad implementations (ordered array, bushy BST, hash table) and then introduces the **binary min-heap**: a binary tree that is both *complete* and obeys the *min-heap property*. Heap `add` works by appending at the end of the heap and **swimming** up; `removeSmallest` works by swapping the last item into the root and **sinking** down, always toward the smaller child. Both are Θ(log N), and `getSmallest` is Θ(1) because the minimum is always at the root. The second half of the lecture asks a more general question: how do you represent a tree in Java at all? It catalogs approaches 1a (fixed-width child links), 1b (array of child links), 1c (favored-child/sibling links), 2 (array of keys plus array of parent indices, like disjoint sets), 3 (array of keys only, assuming completeness), and 3b (same as 3 but with index 0 left empty so that `leftChild(k) = 2k`, `rightChild(k) = 2k+1`, `parent(k) = k/2`). Because heaps are complete by definition, approach 3/3b is the natural representation, which is why array-backed heaps are both fast and memory-efficient (roughly 1/3 the memory of approach 1a). The lecture ends with a summary table of all the search data structures and ADTs covered in the course so far.

---

## Key Concepts

### 1. Wrapping up search trees

Before starting the new material, the lecture recaps the previous three lectures:

- **Binary search trees** are simple but subject to imbalance (worst case Θ(N) spindly).
- **2-3 Trees (B-trees)** are balanced, but painful to implement and relatively slow.
- **LLRB insertion** is simple to implement (delete is hard). It works by maintaining a mathematical bijection with a 2-3 tree.
- **Java's `TreeMap`** is a red-black tree, but *not* left-leaning. It maintains a correspondence with a **2-3-4 tree** (which is *not* a 1-1 correspondence), allows glue links on either side, and is more complex but significantly (?) faster. The question mark is the lecture's own hedging.

And "beyond": there are hundreds of other self-balancing trees (AVL trees, WAVL trees, which the slides call the fastest, splay trees, treaps), other linked structures (**skip lists**, described as "linked lists with express lanes"), and entirely different ideas, the most common being **hashing**, deferred to a later lecture.

### 2. The priority queue ADT

A priority queue is not a search tree and not a general map. It supports a deliberately narrow interface:

```java
/** (Min) Priority Queue: Allowing tracking and removal of the
  * smallest item in a priority queue. */
public interface MinPQ<Item> {
    /** Adds the item to the priority queue. */
    public void add(Item x);
    /** Returns the smallest item in the priority queue. */
    public Item getSmallest();
    /** Removes the smallest item from the priority queue. */
    public Item removeSmallest();
    /** Returns the size of the priority queue. */
    public int size();
}
```

The crucial point is what is *absent*: there is no `get(key)`, no `contains(key)`, no iteration in sorted order, no "give me the median." You can only see and remove one distinguished element. That restriction is exactly what buys you efficiency. In the course's taxonomy table, the PQ is the data structure that retrieves **by key order (a.k.a. key size)**, as opposed to a List (retrieve by index), a Map/Set (retrieve by key identity), or Disjoint Sets (retrieve by two int values).

Note also the lecture's remark that we don't usually consider `MinPQ` and `MaxPQ` to be different data structures, since you can just supply the opposite comparator.

### 3. Why a PQ saves memory: the particle detector example

The motivating problem: a particle detector streams in the energies of incoming particles for 24 hours, and you want the **M highest-energy particles** of the day.

The naive approach stores every particle in an `ArrayList`, sorts with an energy comparator, and takes the first M. Correct, but it uses Θ(N) memory where N is the (potentially enormous) number of particles detected.

The insight is that you never need to remember more than M particles at once. If you are holding the best M you have seen so far and a new particle arrives, then either it belongs in the top M (in which case the weakest of your current M is now out) or it does not (in which case it is itself the weakest and gets dropped). Either way, you add then remove the smallest, and the "smallest of the current best M" is exactly what a **MinPQ** gives you in Θ(1), with removal in Θ(log M). That is why a *min*-PQ is the right tool for finding *maxima*: the min-PQ's root is the item on the chopping block.

### 4. Heap definitions, and why both halves matter

**Binary min-heap**: a binary tree that is **complete** and obeys the **min-heap property**.

- **Min-heap property**: every node is less than or equal to **both** of its children.
- **Complete**: missing items only at the bottom level (if any), and all nodes are as far left as possible.

The min-heap property is much weaker than the BST property. It says nothing about left-versus-right order, only about parent-versus-child. So the root is the global minimum, but the second-smallest could be either child, and the ordering within a level is arbitrary. This weakness is a feature: it makes the invariant easy to restore after a single structural change, with only one path's worth of work.

Note the `≤`: **equal keys are allowed** between parent and child. This is precisely why heaps "handle duplicate priorities much more naturally than BSTs," where duplicate keys are awkward.

Completeness matters for two separate reasons:

1. It forces the height to be Θ(log N), guaranteeing the log-time bounds without any rotations or rebalancing machinery. Compare to the LLRB gymnastics of the previous lecture: a heap gets balance *for free* from the shape invariant.
2. It makes the array representation (approach 3) possible, since a complete tree's shape is fully determined by its size, so no links need to be stored at all.

The lecture's own examples of invalid heaps include a tree that is **incomplete** (a gap in the bottom level or nodes not pushed left) and a tree that **lacks the min-heap property**. The comprehension question presented four trees and asked how many were binary min-heaps; the answer was **2** (one of the four was incomplete, one lacked the min-heap property). Worth absorbing: the valid examples included trees like `8 / (8, 8) / (8, 8, 8, 7)`... no, careful: in the slides' set, a tree with root 8 and a descendant 7 beneath an 8 *violates* the min-heap property. The takeaway to internalize is just that you must check both conditions independently on every node.

A heap full of all-equal values (e.g. all 8s) *is* a valid min-heap, since `≤` holds everywhere.

### 5. `getSmallest`: the easy one

The lecture calls this the "hopefully easy question." The min-heap property applied transitively down any path means the root is `≤` everything beneath it, so the root is the global minimum. `getSmallest()` just returns the root's item: **Θ(1)**, no traversal.

### 6. `add(x)`: append, then swim

The algorithm, from the add demos:

1. **Add to the end of the heap temporarily**, meaning the next open position in the bottom level (leftmost available), which is the only position that preserves completeness.
2. **Swim up the hierarchy to your rightful place**: while the new item is smaller than its parent, swap with the parent and repeat.

Swimming stops as soon as the parent is `≤` the item ("This is my true place"). Each swap moves the item up one level, so at most height = Θ(log N) swaps.

Why does this restore the invariant? Appending can only break the single parent-child relationship between the new node and its parent. Each swap pushes the possible violation one level up while fixing everything below: the parent that moved down is `≤` its other child already (it was before), and `≤` the subtree beneath where the new item came from.

The demos use two inserts into the heap `1 / (5, 1) / (6, 5, 6, 3) / (7, 7, 8)`:
- **Insert 3**: placed at the end (under the 6 that sits leftmost-available), then swims: 3 is less than its parent 5, so they swap; then 3 is less than its new parent 5... the demo shows 3 ending up at the position previously held by the 5 in the second level, with 5 pushed down. Final state: `1 / (3, 1) / (6, 5, 6, 3) / (7, 7, 8, 5)`.
- **Insert 5**: placed at the end, then swims past a 6 (5 < 6, so swap), then stops under its parent ("I am home"). Final state: `1 / (3, 1) / (6, 5, 5, 3) / (7, 7, 8, 5, 6)`.

Notice in the second case the 5 swam exactly one level and halted, illustrating that swim cost is "however far up you need to go," not always to the root.

### 7. `removeSmallest()`: swap last into root, then sink

The algorithm, from the delete demos:

1. **Swap the last item in the heap into the root** (and remove the old root, which is the answer you return). Taking the *last* item is what preserves completeness: removing the last position never leaves a hole in the middle.
2. **Sink down the hierarchy, "yielding to most qualified folks"**: repeatedly compare the item to its children; if a child is smaller, swap with the **smaller of the two children** and continue. Stop when both children are `≥` the item ("I won't descend further") or when you reach a leaf.

Starting from `1 / (3, 1) / (6, 5, 5, 3) / (7, 7, 8, 5, 6)`, the demo deletes the min: the last item 6 goes to the root, then sinks. 6 is bigger than its children, so it swaps with the smaller child 1, then continues: it swaps with 3, landing in place. Result: `1 / (3, 3) / (6, 5, 5, 6) / (7, 7, 8, 5)`.

The next delete demo takes the last item 5 into the root and sinks it; the slides note **"Break tie arbitrarily"** when the two candidate children tie, and the sink stops early with "I won't descend further."

Two details people get wrong:

- You must swap with the **smaller** child, not just any child that is smaller than you. If you swap with the larger one, that larger value becomes a parent of the smaller one and the min-heap property is violated.
- **Ties are broken arbitrarily**, which is sound: if a child equals the parent, `≤` still holds, so you may stop or swap freely.

### 8. The employee/company metaphor

The lecture summarizes the operations with a corporate analogy that is genuinely useful for remembering the shape of the algorithms:

- `getSmallest()`: return the item in the root node (the president).
- `add(x)`: place the new employee in the last position, and promote as high as possible.
- `removeSmallest()`: "assassinate the president," promote the rightmost person in the company to president, then demote repeatedly, always taking the 'better' successor.

The "better successor" clause is the smaller-child rule. The small-value-equals-high-rank orientation is what makes this a *min*-heap.

### 9. Why the other implementations are worse

| | Ordered Array | Bushy BST | Hash Table | Heap |
|---|---|---|---|---|
| `add` | Θ(N) | Θ(log N) | Θ(1) | Θ(log N) |
| `getSmallest` | Θ(1) | Θ(log N) | Θ(N) | Θ(1) |
| `removeSmallest` | Θ(N) | Θ(log N) | Θ(N) | Θ(log N) |
| Caveats | | Dups tough | Items go into random places | Items with same priority hard to handle |

(These are worst-case Θ runtimes.)

- **Ordered array**: `getSmallest` is trivial, but inserting in order requires shifting, and removing the front requires shifting too, so Θ(N).
- **Bushy BST**: all three are Θ(log N), which sounds competitive, but "maintaining bushiness is annoying" (you need the whole LLRB/B-tree apparatus) and "handling duplicate priorities is awkward."
- **Hash table**: "No good! Items go into random places." A hash table destroys order information entirely, so finding the smallest requires scanning everything: Θ(N).

The lecture's closing notes on the heap row:

- "Why 'priority queue'? Can think of position in tree as its 'priority.'"
- "Heap is log N time **AMORTIZED** (some resizes, but no big deal)." The array backing the heap occasionally needs to grow, and resizing is Θ(N), but amortized over the inserts this is fine (same argument as `ArrayList`).
- "BST can have constant `getSmallest` if you keep a pointer to smallest." So the heap's Θ(1) `getSmallest` is not by itself decisive.
- "Heaps handle duplicate priorities much more naturally than BSTs."
- "Array based heaps take less memory (very roughly about 1/3rd the memory of representing a tree with approach 1a)." Intuition: approach 1a stores a key plus two or three reference fields per node; approach 3 stores only the key.

Note the honest caveat in the table under Heap: **"Items with same priority hard to handle."** This refers to a different issue than duplicate *keys*: if you want to change or track the priority of a particular item already in the PQ, heaps alone do not give you a handle on where that item lives. (The lecture lists this as a caveat without elaborating.)

### 10. Representing trees in Java: the six approaches

This is a general-purpose topic, not heap-specific. The question is: given a tree, how do you store the parent-child structure?

**Approach 1: mapping from node to children (recursive, link-based).**

- **1a: Fixed-width nodes.** Each node holds a key and a fixed number of child links, one per possible child. This is what `BSTMap` used.
  ```java
  public class Tree1A<Key> {
      Key k;            // e.g. 0
      Tree1A left;
      Tree1A middle;
      Tree1A right;
      ...
  ```
  Simple, but wastes space if nodes often have fewer children than the maximum, and the maximum must be known in advance.

- **1b: Variable-width nodes (array of child links).** Each node holds a key and an array of children.
  ```java
  public class Tree1B<Key> {
      Key k;            // e.g. 0
      Tree1B[] children;
      ...
  ```
  Handles arbitrary branching factors; costs an extra array object per node.

- **1c: Sibling tree (firstborn/sibling links).** Each node holds a key, a link to its *favored* (first) child, and a link to its next sibling.
  ```java
  public class Tree1C<Key> {
      Key k;            // e.g. 0
      Tree1C favoredChild;
      Tree1C sibling;
      ...
  ```
  Every node has exactly two reference fields regardless of branching factor; you reach a node's children by going to `favoredChild` and then walking the `sibling` chain. (extra context: this is the classic "left-child right-sibling" representation, which turns an arbitrary tree into a binary structure.)

**Approach 2: array of keys plus array of structure.** Store keys in a `Key[]` and parent indices in an `int[]`, indexed in parallel. "Similar to what we did with disjointSets."
```java
public class Tree2<Key> {
    Key[] keys;
    int[] parents;
    ...
```
In the slide's example, the tree with root `k` (index 0), children `e` (1) and `v` (2), then `b, g, p, y` (3-6), etc., gives
```
Key[] keys    = { k, e, v, b, g, p, y, a, d, f, j, m, r, x }
int[] parents = { 0, 0, 0, 1, 1, 2, 2, 3, 3, 4, 4, 5, 5, 6 }
```
So `parents[7] == 3` says that `a` (index 7) hangs off `b` (index 3). The root's parent entry points at itself (index 0 has parent 0), exactly as in disjoint sets. This works for *any* tree shape, since structure is recorded explicitly, and it avoids one object per node.

**Approach 3: array of keys only, structure stored nowhere.**
```java
public class Tree3<Key> {
    Key[] keys;
    ...
```
"To interpret array: Simply assume tree is complete." "Obviously only works for 'complete' trees." The `parents` array from approach 2 was redundant for a complete tree, because the parent index is a pure arithmetic function of the child index. The same key array as above,
```
Key[] keys = { k, e, v, b, g, p, y, a, d, f, j, m, r, x }
```
represents the same tree with no structure array at all. This is the representation of choice for heaps, precisely because heaps are complete by definition.

**Index arithmetic for approach 3 (0-indexed).** The lecture poses this as a challenge: write `parent(k)`.
```java
public int parent(int k) {
    return (k - 1) / 2;
}
```
Check it against the example: `parent(1) = 0`, `parent(2) = 0` (both children of the root map to 0, since integer division truncates); `parent(7) = 3`, `parent(8) = 3`. Correspondingly (extra context, stated implicitly by the slide's indices): `leftChild(k) = 2k + 1` and `rightChild(k) = 2k + 2`.

**Approach 3b (the book implementation): leave one empty spot.** Same as 3, but offset everything by one spot, leaving index 0 empty. This "makes computation of children/parents 'nicer'":
- `leftChild(k) = k * 2`
- `rightChild(k) = k * 2 + 1`
- `parent(k) = k / 2`

The same tree becomes
```
Key[] keys = { -, k, e, v, b, g, p, y, a, d, f, j, m, r, x }
               0  1  2  3  4  5  6  7  8  9 10 11 12 13 14
```
The arithmetic is cleaner because the "times two" and "divide by two" relationship is exact (in binary, a node's index is its parent's index with one bit appended). The cost is one wasted array slot.

### 11. Abstraction happens in layers

The lecture's closing diagram makes a point worth absorbing conceptually: the PQ ADT is implemented by a heap-ordered tree, and that tree is in turn implemented by one of approaches 1a/1b/1c/2/3/3b. Likewise a separate-chaining hash table is implemented as an array of buckets, and each bucket can be an `ArrayList`, resizing array, linked list, BST (requires comparable items), or even a `HashSet` (called a "weird choice"). So "which data structure" is often a question at several levels at once.

### 12. The search data structures summary

| Name | Storage Operation(s) | Primary Retrieval Operation | Retrieve By |
|---|---|---|---|
| List | `add(key)`, `insert(key, index)` | `get(index)` | index |
| Map | `put(key, value)` | `get(key)` | key identity |
| Set | `add(key)` | `containsKey(key)` | key identity |
| PQ | `add(key)` | `getSmallest()` | key order (a.k.a. key size) |
| Disjoint Sets | `connect(int1, int2)` | `isConnected(int1, int2)` | two int values |

Implementations mentioned in the ADT-to-implementation diagram: Set/Map by chaining HT, linear probing HT, linked list, resizing array, vanilla BST, red-black, B-trees (2-3/2-3-4), and heap; PQ by heap, balanced tree, ordered linked list, or chaining HT ("lacks order, very odd"); List by linked list or resizing array; Disjoint Sets by quick find, quick union, weighted QU, WQUPC. Specialized List-flavored ADTs: Stack, Queue, Deque. Sorted Set / Sorted Map correspond to `java.util.SortedSet` and `java.util.SortedMap`, backed by BST, 2-3 tree, or red-black.

"Data Structure: A particular way of organizing data." Two more are still to come in the course: **tries and graphs**.

### 13. Open implementation questions the lecture leaves you with

1. How does a PQ know how to determine which item in a PQ is larger? What could we change so that there is a default comparison?
2. What constructors are needed to allow for different orderings?

The slides pose these without answering them. (extra context: the standard answers are to bound the type parameter as `<Item extends Comparable<Item>>` for a natural default ordering, and to provide both a no-arg constructor that uses `compareTo` and a constructor taking a `Comparator<Item>`, which is what `new HeapMinPQ<>(cmptr)` in the usage example implies. Java's own `PriorityQueue` does exactly this.)

---

## Definitions

- **Priority Queue (PQ)**: An abstract data type supporting `add(Item x)`, `getSmallest()`, `removeSmallest()`, and `size()`. Useful if you want to keep track of the "smallest", "largest", "best", etc. seen so far. Retrieval is by key order (key size), not key identity or index.
- **MinPQ**: A priority queue whose distinguished element is the smallest. A MaxPQ is not considered a separate data structure, since you can supply the opposite comparator.
- **Binary min-heap**: A binary tree that is complete and obeys the min-heap property.
- **Min-heap property**: Every node is less than or equal to both of its children.
- **Complete (tree)**: Missing items occur only at the bottom level (if any), and all nodes are as far left as possible.
- **Swim (up)**: The operation used by `add`: starting from a node, repeatedly swap with the parent while the node is smaller than its parent, "promoting as high as possible."
- **Sink (down)**: The operation used by `removeSmallest`: starting from the root, repeatedly swap with the smaller child while a child is smaller than the node ("yielding to the most qualified folks"), breaking ties arbitrarily.
- **Approach 1a (fixed-width nodes)**: Tree representation where each node object stores a key and a fixed number of child references, one per possible child. Used by `BSTMap`.
- **Approach 1b (variable-width nodes)**: Each node stores a key and an array of child references.
- **Approach 1c (sibling tree / firstborn-sibling links)**: Each node stores a key, a reference to its favored (first) child, and a reference to its next sibling.
- **Approach 2 (array of keys, array of structure)**: Keys in a `Key[]`, parent indices in a parallel `int[]`. Similar to the disjoint sets representation. Works for any tree shape.
- **Approach 3 (array of keys)**: Keys in a `Key[]` with no structure stored; the tree is assumed complete, so parent/child indices are computed arithmetically. Zero-indexed: `parent(k) = (k - 1) / 2`.
- **Approach 3b**: Approach 3 with index 0 left empty, giving `leftChild(k) = k*2`, `rightChild(k) = k*2 + 1`, `parent(k) = k/2`. This is the textbook's implementation.
- **Skip list**: A linked structure described in the lecture as "linked lists with express lanes," an alternative to search trees for implementing sets/maps.
- **Data Structure**: A particular way of organizing data.

---

## Worked Examples

### Example 1: The naive, memory-hungry approach to the top-M problem

```java
public List<Particle> highestEnergyParticles(Detector det, int M) {
    ArrayList<Particle> allParticles = new ArrayList<>();

    for (Timer timer = new Timer(); timer.hours() < 24; ) {
        allParticles.add(det.getNextParticle());
    }

    Comparator<String> cmptr = new EnergyComparator();
    Collections.sort(allParticles, cmptr, Collections.reverseOrder());

    return allParticles.sublist(0, M);
}
```

**What it does, step by step.** An empty `ArrayList` is created. The loop runs for 24 hours of wall-clock time (the `Timer` is created in the init slot and checked in the condition; there is no update expression, since `timer.hours()` advances on its own). Every particle the detector produces is appended. After the day ends, the whole list is sorted by energy in reverse (descending) order, and the first M elements are returned.

**Why it is unsatisfying.** It "potentially uses a huge amount of memory Θ(N), where N is number of particles." Box-and-pointer reasoning in words: `allParticles` is a single reference variable pointing at one `ArrayList` object, but that object's backing array holds N references, each pointing to a distinct `Particle` object on the heap. All N particles stay reachable for the whole day, so none can be garbage collected, even though N - M of them are ultimately irrelevant. The sort also costs time proportional to N log N.

(Note: the slide code has small inconsistencies, `Comparator<String>` where `Comparator<Particle>` is meant, `sublist` rather than `subList`, and a three-argument `Collections.sort` that does not exist. These are slide typos, not the point of the example.)

**The stated goal**: do this in Θ(M) memory using a MinPQ, constructed as

```java
MinPQ<Particle> highEnergyParticles = new HeapMinPQ<>(cmptr);
```

### Example 2: The MinPQ solution, "track the M best"

```java
public List<Particle> highestEnergyParticles(Detector det, int M) {
    Comparator<Particle> cmptr = new EnergyComparator();
    MinPQ<Particle> highEnergyParticles = new HeapMinPQ<>(cmptr);
    for (Timer timer = new Timer(); timer.hours() < 24; ) {
        highEnergyParticles.add(det.getNextParticle());
        if (highEnergyParticles.size() > M)
           { highEnergyParticles.removeSmallest(); }
    }
    ArrayList<String> returnList = new ArrayList<String>();
    while (highEnergyParticles.size() > 0) {
            returnList.add(highEnergyParticles.removeSmallest());
    }
    return returnList;
}
```

**Step by step.**

1. A `Comparator<Particle>` that orders by energy is created, and a heap-backed MinPQ is constructed with it. The PQ will therefore treat "smallest" as "lowest energy."
2. For each incoming particle: `add` it, then if the PQ now holds M+1 items, `removeSmallest()` discards the lowest-energy one.
3. **The invariant**: at the top of every iteration, the PQ contains exactly the M highest-energy particles seen so far (or all of them, if fewer than M have arrived). Adding one item makes it M+1 items, which are the M+1 best seen so far; discarding the minimum of those M+1 leaves the M best. The invariant is restored.
4. After the loop, the PQ holds the day's top M. Draining it with repeated `removeSmallest()` appends them to `returnList` in **ascending** energy order (lowest first), which is worth noticing: the returned list is sorted opposite to the naive version's descending order.

**Why this is Θ(M) memory.** The PQ's backing array never exceeds about M+1 slots, and only M+1 `Particle` objects are reachable at any moment. Every discarded particle becomes unreachable immediately after `removeSmallest()` returns and its reference is dropped, so it is eligible for garbage collection. The lecture's summary: "Can track top M transactions using only M memory. API for MinPQ also makes code very simple (don't need to make explicit comparisons)." That last clause is the API-design lesson: the comparator is handed to the PQ once at construction, and the loop body contains no `if (energy > ...)` logic at all.

**Runtime** (extra context, since the slides emphasize memory): each of the N particles costs one `add` plus possibly one `removeSmallest`, both Θ(log M), so total Θ(N log M) time, versus Θ(N log N) for sort-everything. Better in both time and space.

(The slide code declares `ArrayList<String> returnList` while the method returns `List<Particle>`; another slide typo.)

### Example 3: `swim` on an array-backed heap

This is the lecture's actual code for approach 3:

```java
public void swim(int k) {
    if (keys[parent(k)] ≻ keys[k]) {
        swap(k, parent(k));
        swim(parent(k));
    }
}

public int parent(int k) {
    return (k - 1) / 2;
}
```

The `≻` is the slide's shorthand for "comes after in priority order," i.e. the parent is larger than the child, which is the violation condition for a min-heap.

**Step by step.** `swim(k)` asks: is my parent bigger than me? If so, the min-heap property is violated at that edge, so swap the two array entries and recursively swim from the parent's index, which is now where the item lives. If not, the item is in its rightful place and the recursion stops.

**Why this terminates and is correct.** Each recursive call strictly decreases the index, since `(k-1)/2 < k` for `k >= 1`. The recursion bottoms out either when the parent is `≤` the item, or at the root. At the root, `parent(0) = (0-1)/2 = 0` in Java (because integer division of -1 by 2 truncates toward zero, giving 0), so `keys[0] ≻ keys[0]` is false and the recursion stops. The index arithmetic happens to make the root a safe base case without an explicit guard, though writing `if (k == 0) return;` explicitly is clearer and does not rely on that coincidence.

**Box-and-pointer reasoning in words.** There are no node objects here at all. `keys` is one reference variable pointing at a single array object; the array's slots hold references to the `Key` objects. A "swap" rewires two slots of that one array; nothing about tree structure is stored or changed, because the structure is implicit in the indices. This is exactly why approach 3 is memory-frugal: there is no per-node object header, and no `left`/`right`/`middle` fields.

**Trace.** Suppose the heap is, in approach-3 array form, `{1, 3, 1, 6, 5, 6, 3, 7, 7, 8}` (the lecture's pre-insert-5 heap, read level by level). We append 5 at index 10 and call `swim(10)`:

- `parent(10) = (10-1)/2 = 4`. `keys[4] = 5`, `keys[10] = 5`. Is 5 ≻ 5? No. Stop.

So that particular 5 would not move at all. Compare to inserting 3 into `{1, 5, 1, 6, 5, 6, 3, 7, 7, 8}` at index 9, i.e. `swim(9)`:

- `parent(9) = 4`. `keys[4] = 5 ≻ 3`? Yes. Swap indices 9 and 4: `{1, 5, 1, 6, 3, 6, 3, 7, 7, 5}`. Recurse on `swim(4)`.
- `parent(4) = (4-1)/2 = 1`. `keys[1] = 5 ≻ 3`? Yes. Swap indices 4 and 1: `{1, 3, 1, 6, 5, 6, 3, 7, 7, 5}`. Recurse on `swim(1)`.
- `parent(1) = 0`. `keys[0] = 1 ≻ 3`? No. Stop.

Final array `{1, 3, 1, 6, 5, 6, 3, 7, 7, 5}`, which matches the lecture's drawn result: root 1, second level 3 and 1, third level 6, 5, 6, 3, fourth level 7, 7, 8... with the 8 having been displaced; the key point is the 3 ended up two levels up and the 5s shifted down along that one path. Two swaps, Θ(log N).

### Example 4: `sink`, reconstructed from the delete demos

The lecture draws `sink` but does not show its code. Reconstructing it for approach 3 (extra context: the code below is written to match the lecture's described algorithm; the slides give only `swim`):

```java
public void sink(int k) {
    int left = 2 * k + 1;
    int right = 2 * k + 2;
    if (left >= size) { return; }            // no children: we are a leaf
    int smallerChild = left;
    if (right < size && keys[right] ≺ keys[left]) {
        smallerChild = right;                // ties: keep left, "break tie arbitrarily"
    }
    if (keys[k] ≻ keys[smallerChild]) {
        swap(k, smallerChild);
        sink(smallerChild);
    }
}
```

**Step by step.** Compute both child indices. If the left child does not exist, neither does the right (completeness guarantees children fill left to right), so we are a leaf and stop. Otherwise pick the smaller child; if the right child does not exist, the left is the only candidate. If the node is bigger than that smaller child, swap and continue sinking from the child's position; otherwise stop ("I won't descend further").

**Why the smaller child.** If you swapped with the larger child instead, you would place the larger value above the smaller one, creating a fresh violation at the node you just vacated. Choosing the smaller child guarantees the promoted value is `≤` both the sinking item and its sibling, so the edge you just created is valid.

**Trace of the first delete demo.** Heap array: `{1, 3, 1, 6, 5, 5, 3, 7, 7, 8, 5, 6}` (root 1; level 2: 3, 1; level 3: 6, 5, 5, 3; level 4: 7, 7, 8, 5, 6). `removeSmallest()`:

1. Save `keys[0] = 1` as the return value.
2. Move the last item (`6` at index 11) into index 0, shrink size to 11: `{6, 3, 1, 6, 5, 5, 3, 7, 7, 8, 5}`. "Wuh oh..." as the slides put it: the root is now wildly out of place.
3. `sink(0)`: children are indices 1 (`3`) and 2 (`1`). Smaller is `1` at index 2. Is `6 ≻ 1`? Yes. Swap: `{1, 3, 6, 6, 5, 5, 3, 7, 7, 8, 5}`. Recurse on `sink(2)`.
4. `sink(2)`: children are indices 5 (`5`) and 6 (`3`). Smaller is `3` at index 6. Is `6 ≻ 3`? Yes. Swap: `{1, 3, 3, 6, 5, 5, 6, 7, 7, 8, 5}`. Recurse on `sink(6)`.
5. `sink(6)`: left child would be index 13, which is `>= size`, so index 6 is a leaf. Stop.

Final array `{1, 3, 3, 6, 5, 5, 6, 7, 7, 8, 5}`, which is the lecture's drawn result (root 1; level 2: 3, 3; level 3: 6, 5, 5, 6; level 4: 7, 7, 8, 5). Return value: `1`. Two swaps, Θ(log N).

### Example 5: Reading approach 2 versus approach 3

Given the lecture's tree (root `k`; children `e`, `v`; then `b, g` under `e` and `p, y` under `v`; then `a, d` under `b`, `f, j` under `g`, `m, r` under `p`, `x` under `y`):

**Approach 2** stores both arrays:
```
Key[] keys    = { k, e, v, b, g, p, y, a, d, f, j, m, r, x }
int[] parents = { 0, 0, 0, 1, 1, 2, 2, 3, 3, 4, 4, 5, 5, 6 }
```
To find `x`'s ancestors: `x` is at index 13, `parents[13] = 6` which is `y`, `parents[6] = 2` which is `v`, `parents[2] = 0` which is `k`. Path: `x, y, v, k`.

**Approach 3** stores only
```
Key[] keys = { k, e, v, b, g, p, y, a, d, f, j, m, r, x }
```
and recovers the same answer arithmetically: `parent(13) = (13-1)/2 = 6`, `parent(6) = (6-1)/2 = 2`, `parent(2) = (2-1)/2 = 0`. Identical path. The `parents` array was pure redundancy, because the tree is complete.

**Approach 3b** stores
```
Key[] keys = { -, k, e, v, b, g, p, y, a, d, f, j, m, r, x }
```
and `x` sits at index 14: `parent(14) = 7` (`y`), `parent(7) = 3` (`v`), `parent(3) = 1` (`k`). Notice how clean this is in binary: 14 = `1110`, 7 = `111`, 3 = `11`, 1 = `1`. Each step just drops the last bit. That is the entire reason 3b exists.

---

## Common Pitfalls

1. **Confusing the heap property with the BST property.** A heap says nothing about left versus right. Do not assume the left child is smaller than the right, do not assume an in-order traversal is sorted, and do not try to "search" a heap for an arbitrary key in log time. The only thing you know for free is the minimum.

2. **Forgetting that completeness is part of the definition.** A tree can satisfy "every node ≤ its children" and still not be a heap if it has a gap in the bottom level or nodes not pushed as far left as possible. Both of the lecture's invalid examples fail for exactly one reason each: one is incomplete, the other lacks the min-heap property.

3. **Misreading "complete."** Complete does not mean perfect/full. The bottom level may be partially filled, as long as it is filled left to right with no gaps.

4. **Sinking into the wrong child.** Always swap with the **smaller** of the two children. Swapping with a child that merely happens to be smaller than the parent (but is the larger child) breaks the invariant.

5. **Removing the wrong node to fill the root.** In `removeSmallest`, you move the **last item in the heap** (bottom level, rightmost occupied) into the root. Moving any other node, or promoting a child directly, destroys completeness.

6. **Adding anywhere other than the end.** `add` must place the new item at the next open bottom-level position; inserting "wherever it fits" breaks completeness and destroys the Θ(log N) height guarantee.

7. **Mixing up 0-indexed and 1-indexed heap arithmetic.** Zero-indexed (approach 3): `parent(k) = (k-1)/2`, `leftChild(k) = 2k+1`. One-indexed with slot 0 empty (approach 3b): `parent(k) = k/2`, `leftChild(k) = 2k`. Using `k/2` on a 0-indexed array is wrong (e.g. `parent(3)` would give 1 instead of the correct 1... check more carefully: `(3-1)/2 = 1` and `3/2 = 1` agree here, but `parent(4)`: correct is `(4-1)/2 = 1`, while `4/2 = 2` is wrong). The formulas disagree on exactly the cases that matter, so pick one convention and state it.

8. **Assuming heaps are sorted, or that a heap gives you sorted iteration cheaply.** The array `{1, 3, 1, 6, 5, 5, 3, ...}` is a perfectly valid heap and is not sorted. You can get sorted order only by repeatedly removing the smallest, at Θ(N log N) total.

9. **Thinking a min-PQ is the wrong tool for a max problem.** The particle example uses a **MinPQ** to find the **highest**-energy particles. Reason it through each time: the thing you repeatedly need to identify is the current worst-of-the-best, which is a minimum.

10. **Forgetting the discard step in the top-M idiom.** `add` without the `if (size() > M) removeSmallest()` guard silently reverts you to Θ(N) memory.

11. **Overstating the heap's advantage over a BST.** The lecture explicitly notes that a BST can also have Θ(1) `getSmallest` if you keep a pointer to the smallest node. The heap's real advantages are simplicity of balance, natural handling of duplicate priorities, and roughly 1/3 the memory.

12. **Forgetting the "amortized" qualifier.** Array-backed heap `add` is Θ(log N) amortized, since the backing array occasionally resizes in Θ(N).

13. **Treating the tree-representation approaches as heap-only.** Approaches 1a through 2 work for arbitrary trees; only 3 and 3b require completeness. Conversely, 3/3b is not the only way to store a heap, just the natural one.

14. **Assuming approach 2's parent array marks the root with -1.** In the lecture's example the root's parent entry is its own index (0), matching the disjoint-sets convention.

---

## Likely Exam Points

### 1. Validating a heap

**Q.** Is the following a valid binary min-heap? Root 4; children 7 and 1; then under 7: 5 and 9; under 1: nothing.

**A.** No, for two independent reasons. (i) The min-heap property fails at the root: 4 > 1, so the root is not ≤ both children. (ii) It is incomplete: the bottom level has nodes under the 7 but the position under the 1 is... actually in level-order left-to-right the filled bottom positions are the two under 7, and there is no gap to their left, so completeness holds here. The decisive failure is the min-heap property at the root (4 > 1). Always check both conditions separately and name which one fails.

### 2. Hand-simulating `add`

**Q.** Starting from the heap (level order) `{2, 4, 3, 8, 5, 9, 7}`, perform `add(1)`. Give the resulting array.

**A.** Append 1 at index 7: `{2, 4, 3, 8, 5, 9, 7, 1}`. Swim from 7: `parent(7) = 3`, `keys[3] = 8 > 1`, swap → `{2, 4, 3, 1, 5, 9, 7, 8}`. Swim from 3: `parent(3) = 1`, `keys[1] = 4 > 1`, swap → `{2, 1, 3, 4, 5, 9, 7, 8}`. Swim from 1: `parent(1) = 0`, `keys[0] = 2 > 1`, swap → `{1, 2, 3, 4, 5, 9, 7, 8}`. Swim from 0: at the root, stop. Result: `{1, 2, 3, 4, 5, 9, 7, 8}`, with three swaps.

### 3. Hand-simulating `removeSmallest`

**Q.** Starting from `{1, 2, 3, 4, 5, 9, 7, 8}`, perform `removeSmallest()`. What is returned, and what is the resulting array?

**A.** Returns 1. Move the last item (8, index 7) to the root and shrink to size 7: `{8, 2, 3, 4, 5, 9, 7}`. Sink from 0: children are 2 (index 1) and 3 (index 2); smaller is 2; 8 > 2, swap → `{2, 8, 3, 4, 5, 9, 7}`; recurse from index 1. Children of 1 are indices 3 (`4`) and 4 (`5`); smaller is 4; 8 > 4, swap → `{2, 4, 3, 8, 5, 9, 7}`; recurse from index 3. Index 3's left child is 7, which is ≥ size 7, so it is a leaf. Stop. Result: `{2, 4, 3, 8, 5, 9, 7}`.

### 4. Index arithmetic

**Q.** In a 0-indexed approach-3 heap array, what are the indices of the parent, left child, and right child of index 6? What about in approach 3b (index 0 empty)?

**A.** Approach 3: `parent(6) = (6-1)/2 = 2`; `leftChild(6) = 13`; `rightChild(6) = 14`. Approach 3b: `parent(6) = 6/2 = 3`; `leftChild(6) = 12`; `rightChild(6) = 13`. The two conventions describe different trees for the same array, so always say which you are using.

### 5. Runtime table

**Q.** Fill in worst-case Θ for `add`, `getSmallest`, `removeSmallest` for ordered array, bushy BST, hash table, and heap. What caveat does the lecture attach to the BST and the heap?

**A.** Ordered array: Θ(N), Θ(1), Θ(N). Bushy BST: Θ(log N), Θ(log N), Θ(log N). Hash table: Θ(1), Θ(N), Θ(N). Heap: Θ(log N), Θ(1), Θ(log N) (log N amortized, due to array resizes). BST caveat: duplicates are tough, and maintaining bushiness is annoying (though a pointer to the smallest node gives Θ(1) `getSmallest`). Heap caveat: items with the same priority are hard to handle (in the sense of tracking/updating a specific item's priority), though duplicate priorities themselves are handled more naturally than in a BST.

### 6. The top-M idiom

**Q.** You are given a stream of N numbers and must output the M largest using Θ(M) memory. Which PQ flavor, and what is the loop body?

**A.** A **MinPQ** of capacity M+1. Loop body: `pq.add(x); if (pq.size() > M) pq.removeSmallest();`. The MinPQ's root is the weakest of your current best M, which is exactly the candidate for eviction. Memory is Θ(M); time is Θ(N log M).

### 7. Which tree representation, and why

**Q.** Why is approach 3 suitable for a heap but not for a general BST? And what does approach 3b buy over approach 3?

**A.** Approach 3 stores no structure at all and instead *assumes the tree is complete*, deriving parent/child relationships from index arithmetic. A heap is complete by definition, so this is lossless. A general BST can be arbitrarily spindly, so index arithmetic would require leaving huge numbers of array slots empty (exponential in the height, in the worst case), making it hopeless. Approach 3b buys nicer arithmetic (`2k`, `2k+1`, `k/2` instead of `2k+1`, `2k+2`, `(k-1)/2`) at the cost of one wasted slot; it is the textbook's implementation.

### 8. Memory comparison

**Q.** Roughly how much memory does an array-based heap use compared to representing the same tree with approach 1a?

**A.** Very roughly about 1/3. Approach 1a allocates a node object per item with a key field plus child reference fields (plus object overhead); approach 3 stores only the keys in one array, with the structure implicit.

### 9. Why not a hash table

**Q.** A hash table gives Θ(1) `add`. Why is it a bad PQ?

**A.** "Items go into random places." Hashing deliberately scatters keys to spread load, which destroys all order information, so `getSmallest` requires scanning every bucket: Θ(N). A PQ retrieves by key *order*, which is fundamentally the one thing a hash table does not preserve. (The summary diagram lists chaining HT as a possible PQ implementation but labels it "lacks order, very odd.")

### 10. Comparator plumbing

**Q.** The usage example writes `new HeapMinPQ<>(cmptr)`. What two implementation questions does the lecture raise about this, and what are the standard answers?

**A.** (1) How does the PQ know which item is larger, and what could we change so there is a default comparison? (2) What constructors are needed to allow different orderings? The slides leave these open. (extra context: bound the type parameter with `<Item extends Comparable<Item>>` so `compareTo` serves as the default, and supply two constructors, a no-arg one using natural ordering and one taking a `Comparator<Item>` stored in a field and consulted by `swim`/`sink`. This is also how a MaxPQ is obtained: pass the reversed comparator rather than writing a new class.)

### 11. Search-tree recap (carryover from Lectures 17-19)

**Q.** What kind of tree is Java's `TreeMap`, and what does it correspond to?

**A.** A red-black tree, not left-leaning. It corresponds to a **2-3-4 tree**, and the correspondence is *not* 1-1 (glue links are allowed on either side). It is more complex to implement than an LLRB but significantly faster.

### 12. ADT classification

**Q.** For each of List, Map, Set, PQ, Disjoint Sets, name the primary retrieval operation and what you retrieve by.

**A.** List: `get(index)`, by index. Map: `get(key)`, by key identity. Set: `containsKey(key)`, by key identity. PQ: `getSmallest()`, by key order (key size). Disjoint Sets: `isConnected(int1, int2)`, by two int values.

---

## Summary

- **Search tree wrap-up**: BSTs are simple but imbalance-prone; 2-3/B-trees are balanced but painful; LLRB insert is easy (delete is hard) via a bijection with 2-3 trees; Java's `TreeMap` is a non-left-leaning red-black tree corresponding (not 1-1) to a 2-3-4 tree. Many other balanced trees exist (AVL, WAVL, splay, treaps), plus skip lists and hashing.
- **Priority queue ADT**: `add`, `getSmallest`, `removeSmallest`, `size`. Retrieves by **key order**, not key identity. MaxPQ is just MinPQ with the opposite comparator.
- **Killer application**: tracking the top M of a stream of N items in Θ(M) memory (and Θ(N log M) time) instead of Θ(N) memory, by `add`-then-`removeSmallest` whenever the size exceeds M.
- **Binary min-heap** = **complete** binary tree + **min-heap property** (every node ≤ both children). Both conditions are required; check them independently.
- The `≤` allows equal keys, so heaps handle **duplicate priorities** far more naturally than BSTs.
- **`getSmallest`**: return the root. Θ(1).
- **`add(x)`**: append at the next bottom-level position (preserving completeness), then **swim up** while smaller than the parent. Θ(log N).
- **`removeSmallest()`**: return the root, move the **last** item into the root, then **sink down**, always swapping with the **smaller child**, breaking ties arbitrarily. Θ(log N).
- Completeness gives Θ(log N) height for free, with no rotations or rebalancing.
- **Runtimes**: heap is Θ(log N) / Θ(1) / Θ(log N) (log N amortized because of array resizing), beating ordered array (Θ(N) add and remove) and hash table (Θ(N) for both order-dependent ops), and tying bushy BST on asymptotics while being simpler and ~1/3 the memory of a linked tree.
- **Tree representations**: 1a fixed-width child links (used by `BSTMap`), 1b array of child links, 1c favored-child/sibling links, 2 key array + parent-index array (disjoint-sets style, any shape), 3 key array only (assumes complete; `parent(k) = (k-1)/2`), 3b key array with slot 0 empty (`leftChild = 2k`, `rightChild = 2k+1`, `parent = k/2`; the textbook's choice).
- Approach 3/3b works for heaps precisely because heaps are complete, so the structure array of approach 2 is redundant.
- **Abstraction is layered**: PQ → heap-ordered tree → one of approaches 1a/1b/1c/2/3/3b.
- Course ADT table: List (by index), Map/Set (by key identity), PQ (by key order), Disjoint Sets (by two ints). Still to come: **tries and graphs**.
- Open questions the lecture leaves hanging: how the PQ decides which item is larger (default comparison via `Comparable`) and what constructors support alternative orderings (a `Comparator`-taking constructor).
