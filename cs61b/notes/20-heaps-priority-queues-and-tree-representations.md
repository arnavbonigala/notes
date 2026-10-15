<!-- Mon, Oct 12, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 20: Heaps, Priority Queues, and Tree Representations

## Overview

This lecture closes out the search tree arc (BSTs, 2-3/B-trees, LLRBs) and pivots to a brand new abstract data type: the **priority queue** (PQ), which tracks and removes the "smallest" (or "best") item seen so far. After motivating PQs with a particle detector example that must find the top M of potentially billions of events using only Θ(M) memory, the lecture evaluates several candidate implementations (ordered array, bushy BST, hash table) and finds each lacking. The winner is the **binary min-heap**: a complete binary tree obeying the min-heap property, supporting `getSmallest` in Θ(1) and `add`/`removeSmallest` in Θ(log N) via the **swim** and **sink** operations. The final third of the lecture steps back and asks a more general question: how do we represent a tree in Java at all? Six approaches are surveyed (1a fixed-width nodes, 1b array of child links, 1c sibling tree, 2 keys array plus parents array, 3 keys array only, 3b keys array offset by one), and the punchline is that because heaps are *complete*, the parents array is always the same and therefore completely unnecessary: a heap can be stored as a bare array with `parent(k) = (k - 1) / 2`.

---

## Key Concepts

### 1. The end of the search tree journey

A quick recap of the last three lectures before moving on:

- **Binary search trees** are simple but subject to imbalance.
- **2-3 trees (B-trees)** are balanced but painful to implement and relatively slow. A 2-3 tree is the special case of a B-tree where the node capacity limit is 2.
- **LLRBs** have simple insertion (delete is hard, not covered) and work by maintaining a *bijection* (one-to-one correspondence) with a 2-3 tree.
- **Java's `TreeMap`** is a red-black tree, but **not** left-leaning. It corresponds to a 2-3-**4** tree, and that correspondence is **not** one-to-one: multiple red-black trees can represent the same 2-3-4 tree, because glue links are allowed on either side. More complex to implement, but significantly faster.

Beyond these there are hundreds of self-balancing trees (AVL, WAVL (the fastest of the ones mentioned), splay trees, treaps). And there are entirely different ways to implement sets and maps: **skip lists** (linked lists with express lanes) and **hashing**, which is the most common alternative and is coming in a later lecture.

### 2. The priority queue ADT

A priority queue is a "magic bag." You throw things in; the only thing you can pull out is the smallest. The lecture's live demo: put three objects into a bag, then `removeSmallest()` three times and get them out in increasing order of size.

```java
public interface MinPQ<Item> {
    public void add(Item x);          // add the item
    public Item getSmallest();        // return (don't remove) the smallest
    public Item removeSmallest();     // remove and return the smallest
    public int size();
}
```

Critically, a PQ is **not** a set or a map. It does not retrieve by key identity or by index. It retrieves **by key order**. You cannot ask a PQ "do you contain X?" efficiently, and you cannot iterate it in sorted order without destroying it. It is useful precisely when all you care about is the extreme item.

Note that MinPQ and MaxPQ are not usually considered different data structures, since you can just supply the opposite comparator.

### 3. Why PQs matter: the Θ(M) memory trick

The motivating problem: a particle detector streams in particles all day; you want the **M highest energy** particles. The naive solution stores every particle, sorts, and takes the top M: Θ(N) memory where N could be 100 quadrillion.

The PQ solution is slightly counterintuitive: **to track the M largest items, use a MinPQ of size M.** Add every incoming particle; whenever the size exceeds M, `removeSmallest()`. The smallest item in the bag is exactly the one you least care about, so throwing it away is free. Memory stays Θ(M).

This pattern (min-heap of size M to track the top M) is worth memorizing; it generalizes to "top K" problems everywhere.

### 4. Bad implementations of a MinPQ

| | Ordered Array | Bushy BST | Hash Table | Heap |
|---|---|---|---|---|
| `add` | Θ(N) | Θ(log N) | Θ(1) | Θ(log N) |
| `getSmallest` | Θ(1) | Θ(log N) | Θ(N) | Θ(1) |
| `removeSmallest` | Θ(N) | Θ(log N) | Θ(N) | Θ(log N) |
| Caveats | | Dups tough | | Dups fine |

(Worst-case Θ runtimes.)

- **Ordered array**: `getSmallest` is trivially index 0, but `add` has to find the insertion point and scoot everything over: Θ(N). Worst case is inserting a new smallest.
- **Bushy BST**: find the smallest by walking left until there is no left child; that is where you stop, and nothing can be smaller. All three operations are Θ(log N) in a bushy tree. Reasonable! But **duplicate priorities are awkward** in a BST, and keeping it bushy is annoying. This matters in practice: Dijkstra's algorithm (coming later in the course for driving directions) definitely produces ties.
- **Hash table**: Θ(1) add, but items go into *random* places, so finding the smallest requires scanning everything: Θ(N). No good.

### 5. The binary min-heap

**Binary min-heap**: a rooted binary tree that is (a) **complete** and (b) obeys the **min-heap property**.

- **Min-heap property**: every node is **less than or equal to** both of its children.
- **Complete**: every level is full (level i has 2^i nodes) except possibly the bottom level, and on the bottom level all nodes are as far **left** as possible with no gaps.

Contrast with a BST: a BST says "everything to my left is smaller, everything to my right is larger," which is a strong and fiddly property. A heap says only "everything below me is greater than or equal to me." This is a **weaker, less picky** constraint, which is why it naturally tolerates equal items, and why there is no left/right ordering guarantee between siblings whatsoever.

Checking examples from the lecture, with root at top:

```
   8          5          4          0
  / \        / \        / \        / \
 8   8      6   2      7   1      7   1
/|\        /|  / \    /|\        /|\
8 8 8     8  0 3  5  9...
```
- A tree whose bottom level has gaps that are not left-packed is **incomplete**, even if the min-heap property holds.
- A tree can be perfectly **complete** and still fail because some node is greater than a child (e.g. a `4` sitting above a `1`).
- A tree whose bottom level is partly empty but left-packed **is** fine.

A max-heap is the exact mirror image: every node is greater than or equal to both of its children.

### 6. Heap operations

**`getSmallest()`**: return the root. Θ(1). The min-heap property guarantees the root is ≤ everything below it, which is everything.

**`add(x)`: place at the end, then swim.**
1. Put `x` in the **next open position** at the bottom level (leftmost gap). This preserves completeness by construction, so now you only have to fix the min-heap property.
2. **Swim**: while `x` is less than its parent, swap with the parent. Stop as soon as you reach a parent you should not swap with, i.e. a parent that is ≤ you.

Worked trace from lecture, inserting 3 into:
```
        1
      /   \
     5     1
    / \   / \
   6   5 6   3
  / \  |
 7  7  8
```
3 is placed as the next child under the `6` subtree, swims past 5, swims past 5 again (per the slide sequence 3 rises through the levels), and finally lands where its parent is ≤ 3: "this is my true place." Then inserting 5 into the resulting heap: it is placed at the next open slot under the right subtree, swims up one level past a 6, and stops because the parent above is ≤ 5.

**`removeSmallest()`: swap last into root, then sink.**
1. The answer is the root; take it.
2. Move the **last item in the heap** (the rightmost node on the bottom level) into the root. This is the only choice that preserves completeness: removing the last slot leaves the tree complete.
3. **Sink**: repeatedly compare the node with its children. If a child is smaller, swap with the **smaller of the two children**. Stop when both children are ≥ you (or you have no children).

Why the **smaller** child specifically? If you swapped with the larger child, the item you promoted would then be larger than its sibling, creating a new violation at the root you just "fixed." Taking the smaller child guarantees the promoted item is ≤ both of its new children.

Lecture trace: deleting min from
```
        1
      /   \
     3     1
    / \   / \
   6   5 5   3
  / \  |  \
 7  7  8  5  6
```
The root `1` is removed, the last item `6` is moved to the root ("what's happening to me??"), and 6 sinks: it swaps with the smaller child `1`, then continues down, swapping with `3`, until all of its children are ≥ it.

Deleting again, the last item (a `5`) becomes the temporary root and sinks. When there is a **tie** between the two children, **break it arbitrarily**: both resulting heaps are valid, just as many different BSTs can hold the same data. (A student suggested breaking ties toward the right might save a tiny bit of sinking; Josh agreed it is at best a small optimization, roughly one level in the worst case.)

**Why sinking never overshoots**: you only ever swap when you have a child strictly smaller than you, which is exactly the situation that is a violation. So you stop precisely when there is nothing left to fix.

**The company analogy** used throughout:
- `getSmallest()`: look at the president.
- `add(x)`: hire the new employee into the last position, then promote them as high as they deserve.
- `removeSmallest()`: assassinate the president, promote the rightmost person in the company to president, then demote them repeatedly, always swapping with the "better" (smaller) successor.

### 7. How do you represent a tree in Java at all?

This is the generalization step. Six approaches, of which the first three use explicit 64-bit reference links.

**Approach 1a: Fixed-width nodes.** One hard-coded reference field per child. This is what `BSTMap` used (and similar to a linked list).
```java
public class Tree1A<Key> {
    Key k;
    Tree1A left;
    Tree1A middle;
    Tree1A right;
}
```
Fastest pointer traversal: W straight to X, one hop. But the branching factor is baked in.

**Approach 1b: Variable-width nodes (array of child links).** Supports any number of children per node.
```java
public class Tree1B<Key> {
    Key k;
    Tree1B[] children;
}
```
Box-and-pointer reasoning: `W`'s `children` field holds a reference to a *separate array object*, and each slot of that array holds a reference to a child node. So getting from W to X is **two hops** (W → array → X) instead of one. More flexible, slightly less performant due to that extra level of indirection.

**Approach 1c: Sibling tree (firstborn/sibling links).** Attributed to Paul Hilfinger.
```java
public class Tree1C<Key> {
    Key k;
    Tree1C favoredChild;
    Tree1C sibling;
}
```
Each node points to one "favored" child, and children point to their next sibling. So W's three children X, Y, Z are stored as W → X, X → Y (sibling), Y → Z (sibling). Weird, but exam problems about it exist. (Josh: "I don't know of any reason you would do this, but I have seen an exam problem about this before.")

**Approach 2: Array of keys + array of structure.** Similar to disjoint sets.
```java
public class Tree2<Key> {
    Key[] keys;
    int[] parents;
}
```
Two parallel arrays. `keys[i]` is the key at node i; `parents[i]` is the index of node i's parent. The root is its own parent. For the lecture's example tree with keys `k e v b g p y a d f j m r x` at indices 0-13, `parents` is `0 0 0 1 1 2 2 3 3 4 4 5 5 6`.

**Why two arrays when disjoint sets used only one?** In disjoint sets the nodes had no identity: they were just the integers 0 through N-1, so the single parents array sufficed. Here nodes have *identities* (keys), so you need a second array to say which key lives at which node.

This representation captures **any** tree. The lecture demonstrated editing it live: change `parents[13]` from `6` to `0`, and node 13 is now a direct child of the root. Every tree you can draw has a corresponding parents array, and vice versa.

**Approach 3: Array of keys only. Don't store structure anywhere.**
```java
public class Tree3<Key> {
    Key[] keys;
}
```
The key insight of the lecture: look at that parents array again. `0 0 0 1 1 2 2 3 3 4 4 5 5 6`. It is *extremely* regular. Why? Because **the tree is complete**. Every complete tree of the same size has *exactly* that parents array. So storing it is pointless: it carries zero information once you know the tree is complete.

And heaps are always complete. So approach 3 stores just the keys and **assumes** the tree is complete. "So where is the tree? Well, it doesn't exist. We don't have to store the tree structure at all. That's why completeness is one of the properties of a heap." The structure is *computed*, not stored:

```java
public int parent(int k) {
    return (k - 1) / 2;
}
```

Check it: node 11's parent is (11-1)/2 = 5. Node 13's parent is (13-1)/2 = 6. Node 12's parent is (12-1)/2 = 11/2 = 5 (Java integer division truncates toward zero). Node 6's parent is (6-1)/2 = 5/2 = 2. Correct in each case.

A nice side effect: the earlier student question of "how do you find where the next item spawns?" dissolves. The next open position is just the next unoccupied array index. You find it instantly rather than walking the tree.

(extra context) The matching children formulas for approach 3 are `leftChild(k) = 2*k + 1` and `rightChild(k) = 2*k + 2`; the lecture gave only the 3b versions explicitly.

**Approach 3b: Offset by one, leave index 0 empty.** This is what the textbook (and the Princeton algorithms book) does.
- `leftChild(k) = k * 2`
- `rightChild(k) = k * 2 + 1`
- `parent(k) = k / 2`

Same idea, nicer arithmetic, at the cost of one wasted array slot. Some older 61B material and many references use this; either is fine, just be consistent.

Josh's historical note: interpreting a linear array as a complete tree is a very old-school trick from an era when memory and time were extremely tight, and it remains useful even today.

### 8. Final tradeoffs

Notes from the summary slide:
- **Why "priority queue"?** Think of a node's **position in the tree** as its priority: highest priority at the top.
- Heap operations are Θ(log N) **amortized**, because the backing array occasionally needs resizing (when it gets too full on `add`, or you might downsize when too empty on `remove`). No big deal.
- A BST **can** achieve constant `getSmallest` if you maintain a pointer to the smallest element.
- Heaps handle **duplicate priorities** much more naturally than BSTs. This is the main advantage.
- **Array-based heaps use far less memory**: very roughly about 1/3 the memory of representing the same tree with approach 1a (no per-node object headers, no left/right reference fields).

### 9. The big data structures picture

| Name | Storage Operation(s) | Primary Retrieval Operation | Retrieve By |
|---|---|---|---|
| List | `add(key)`, `insert(key, index)` | `get(index)` | index |
| Map | `put(key, value)` | `get(key)` | key identity |
| Set | `add(key)` | `containsKey(key)` | key identity |
| PQ | `add(key)` | `getSmallest()` | key order (key "size") |
| Disjoint Sets | `connect(int1, int2)` | `isConnected(int1, int2)` | two int values |

**Abstraction happens in layers.** A PQ is implemented by a heap-ordered tree, which is in turn implemented by one of approaches 1a/1b/1c/2/3/3b. Similarly, a separate-chaining hash table is implemented by an array of buckets, each bucket implemented by an ArrayList / resizing array / linked list / BST.

Remaining data structures for the course: **tries** and **graphs**.

---

## Definitions

- **Priority Queue (PQ)**: An abstract data type supporting `add`, `getSmallest`, `removeSmallest`, and `size`; retrieval is by **key order**, not by key identity or index.
- **MinPQ / MaxPQ**: Min-oriented and max-oriented priority queues. Not considered distinct data structures, since one can be built from the other by supplying the opposite comparator.
- **Binary min-heap**: A rooted binary tree that is **complete** and obeys the **min-heap property**.
- **Min-heap property**: Every node is less than or equal to **both** of its children.
- **Max-heap property**: Every node is greater than or equal to both of its children.
- **Complete (tree)**: Every level is entirely full (level i contains 2^i nodes) except possibly the bottom level; on the bottom level all nodes are as far left as possible, with no gaps.
- **Swim**: The upward-fixing operation used by `add`. Repeatedly swap a node with its parent while the node is smaller than its parent.
- **Sink**: The downward-fixing operation used by `removeSmallest`. Repeatedly swap a node with the **smaller** of its two children while a child is smaller than it.
- **Approach 1a (fixed-width nodes)**: Each node holds a fixed number of hard-coded child reference fields.
- **Approach 1b (variable-width nodes)**: Each node holds a reference to an array of child references.
- **Approach 1c (sibling tree)**: Each node holds a reference to one favored child and a reference to its next sibling.
- **Approach 2**: Two parallel arrays, `Key[] keys` and `int[] parents`; works for any tree.
- **Approach 3**: A single `Key[] keys` array; tree structure is *assumed* to be complete and computed arithmetically. `parent(k) = (k - 1) / 2`.
- **Approach 3b**: Approach 3 with index 0 left empty. `parent(k) = k / 2`, `leftChild(k) = 2k`, `rightChild(k) = 2k + 1`.
- **Bijection (in the LLRB recap)**: A one-to-one correspondence; every LLRB maps to exactly one 2-3 tree and vice versa. Red-black trees and 2-3-4 trees do **not** have this property.
- **Skip list**: An ordered linked list with "express lanes," an alternative way to implement a set or map.

---

## Worked Examples

### Example 1: The naive Θ(N) approach (what not to do)

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

**What it does**: collects every particle detected over 24 hours into an ArrayList, sorts the whole list in descending energy order, and returns the first M.

**Why it is bad**: the ArrayList grows to hold all N particles. If the detector produces 100 quadrillion events and you only want the top 50, you have paid Θ(N) memory for a Θ(M) answer. (The slides' type signatures here are slightly inconsistent, `Comparator<String>` for a list of `Particle`; that is a slide typo, not a conceptual point.)

### Example 2: The Θ(M) MinPQ solution

```java
public List<Particle> highestEnergyParticles(Detector det, int M) {
    Comparator<Particle> cmptr = new EnergyComparator();
    MinPQ<Particle> highEnergyParticles = new HeapMinPQ<>(cmptr);

    for (Timer timer = new Timer(); timer.hours() < 24; ) {
        highEnergyParticles.add(det.getNextParticle());
        if (highEnergyParticles.size() > M) {
            highEnergyParticles.removeSmallest();
        }
    }

    ArrayList<String> returnList = new ArrayList<String>();
    while (highEnergyParticles.size() > 0) {
        returnList.add(highEnergyParticles.removeSmallest());
    }
    return returnList;
}
```

**Step by step:**

1. **The comparator** is passed into the `HeapMinPQ` constructor. The Particle class may have no natural order, so the comparator is what tells the heap what "smallest" means: lowest energy. This is why the implementing class takes a comparator at construction.
2. **The loop invariant**: after each iteration, `highEnergyParticles` holds the M highest-energy particles seen so far (or all of them, if fewer than M have arrived). Particles 1 through M just accumulate. Starting with particle M+1, every `add` pushes size to M+1 and the immediately following `removeSmallest` evicts the weakest of those M+1.
3. **Why discarding is safe**: the evicted particle is, by definition, the lowest-energy one currently retained. Since it was already beaten by M others, it can never be in the final top M. We do not store it anywhere; we genuinely do not care.
4. **Memory**: the heap's backing array never exceeds roughly M+1 entries, so Θ(M). Note the returned list is also Θ(M).
5. **Draining**: the final `while` loop empties the heap into a list. Because `removeSmallest` is called repeatedly, the results come out in **ascending** energy order, i.e. the Mth best first and the single best last. (extra context) If you need descending order, reverse the list afterward or push onto a stack.
6. **Readability**: notice there is not a single explicit `if (a.energy > b.energy)` comparison in the body. The PQ API absorbs all of that.

### Example 3: `swim` in approach-3 array representation

```java
public void swim(int k) {
    if (keys[parent(k)] > keys[k]) {   // '>' stands in for the comparator
        swap(k, parent(k));
        swim(parent(k));
    }
}

public int parent(int k) {
    return (k - 1) / 2;
}
```

**Tracing `parent`:** suppose the heap array is laid out as in the lecture's example tree with 14 nodes at indices 0-13.
- `parent(13) = (13 - 1) / 2 = 12 / 2 = 6`. Correct: node 13 is a child of node 6.
- `parent(11) = (11 - 1) / 2 = 10 / 2 = 5`. Correct.
- `parent(12) = (12 - 1) / 2 = 11 / 2`. Java integer division **truncates**, giving 5. Correct: node 12 is the left child of node 5, node 13 the right child of node 6. This truncation is load-bearing; in a language with true division this formula breaks.

**Why there is no explicit base case:** the recursion stops in two ways. Normally, it stops as soon as `keys[parent(k)]` is not greater than `keys[k]`, meaning the node has reached its rightful place. At the root, `parent(0) = (0 - 1) / 2 = -1 / 2 = 0` in Java (truncation toward zero), so index 0 is its own parent, the comparison `keys[0] > keys[0]` is false, and the recursion terminates. The lecture examined whether an explicit `if (k == 0) return;` guard is needed and concluded it works without one (Josh: "I think it works. Like 80% sure"), while noting that adding the guard is the safest and clearest option. **For an exam, writing the explicit `k == 0` base case is the defensible choice.**

**Box-and-pointer / memory reasoning**: there are no boxes and no pointers here. That is the whole point. In approach 1a, a 14-node tree is 14 separate heap-allocated objects, each with an object header plus a key reference plus two 64-bit child references, and traversing from a node to its parent is **impossible** without a parent pointer. In approach 3, there is one array object holding 14 key references, and "traversal" is pure arithmetic on an `int`. No dereferencing, no cache misses chasing pointers, roughly a third of the memory, and the parent is reachable in one subtraction and one shift.

### Example 4: Reading a heap out of an array

Given `keys = [1, 3, 1, 6, 5, 5, 3, 7, 7, 8, 5, 6]` under approach 3, reconstruct the tree:

- Index 0 (`1`) is the root.
- Indices 1-2 (`3`, `1`) are level 1.
- Indices 3-6 (`6`, `5`, `5`, `3`) are level 2.
- Indices 7-11 (`7`, `7`, `8`, `5`, `6`) are the partial bottom level, left-packed.

```
            1(0)
         /        \
      3(1)         1(2)
     /    \       /    \
   6(3)   5(4)  5(5)   3(6)
   / \    /  \
 7(7)7(8)8(9)5(10)  ... 6(11) under index 5
```
Verify the min-heap property by checking each index k against `2k+1` and `2k+2`. Verify completeness by confirming the occupied indices are exactly 0 through size-1 with no holes, which is automatic in this representation. **That is the elegance: completeness cannot be violated, because there is nowhere to put a gap.**

### Example 5: Hand-tracing `removeSmallest` on an array

Start with `[1, 3, 1, 6, 5, 5, 3, 7, 7, 8, 5, 6]` (size 12).

1. Save `keys[0] = 1` as the return value.
2. Move the last element, `keys[11] = 6`, into index 0. Size becomes 11. Array: `[6, 3, 1, 6, 5, 5, 3, 7, 7, 8, 5]`.
3. Sink from index 0. Children are indices 1 (`3`) and 2 (`1`). The smaller is `1` at index 2, and `1 < 6`, so swap: `[1, 3, 6, 6, 5, 5, 3, 7, 7, 8, 5]`. Now at index 2.
4. Children of index 2 are indices 5 (`5`) and 6 (`3`). Smaller is `3` at index 6, and `3 < 6`, so swap: `[1, 3, 3, 6, 5, 5, 6, 7, 7, 8, 5]`. Now at index 6.
5. Children of index 6 would be indices 13 and 14, both out of range (size 11). Stop.

Return `1`. The result is a valid heap, and note how the two swaps correspond exactly to the slide sequence where `6` traded places first with `1` and then with `3`.

---

## Common Pitfalls

1. **Thinking a heap is sorted.** The array is *not* in sorted order and there is no ordering relationship between siblings or between cousins. Only the parent-child relationship is constrained. A heap of size N tells you the minimum instantly and tells you essentially nothing else for free.

2. **Confusing "complete" with "full" or "perfect."** Complete allows a partially-filled bottom level, as long as it is left-packed. A tree with a missing node in the middle of the bottom level is incomplete even if the min-heap property holds perfectly.

3. **Sinking into the larger child.** If you swap a sinking node with the larger of its two children, the node you promoted is now larger than its sibling and you have created a new violation at the very position you just vacated. **Always take the smaller child.**

4. **Swimming when you should sink, or vice versa.** `add` swims (new item starts at the bottom). `removeSmallest` sinks (displaced item starts at the top). Mixing them up is a classic exam error.

5. **Removing the root and "pulling up" a child.** A tempting but wrong `removeSmallest`: delete the root and promote the smaller child, recursively. This produces a correct min-heap-property tree but **destroys completeness**, because the hole propagates down an arbitrary path rather than to the last position. The last-element-to-root-then-sink approach is what preserves completeness.

6. **Off-by-one between approach 3 and 3b.** `parent(k) = (k-1)/2` in approach 3; `parent(k) = k/2` in approach 3b. Using the 3b children formula (`2k`, `2k+1`) with a 0-indexed heap gives garbage. Pick one convention and state it.

7. **Forgetting Java's integer truncation is required.** `(k-1)/2` only works because Java truncates. Writing it in a language with float division, or accidentally using a floating-point type, breaks the formula.

8. **Assuming a BST can't do PQ work.** It can, at Θ(log N) for all three operations, and with a smallest-pointer it even achieves Θ(1) `getSmallest`. The real reasons to prefer a heap are duplicate-priority handling, simpler implementation, and roughly 1/3 the memory.

9. **Expecting `getSmallest` to remove.** `getSmallest` returns a reference; it leaves the item in the queue. Only `removeSmallest` removes.

10. **Forgetting the heap bound is amortized.** The array resizes occasionally, so Θ(log N) for `add` is amortized, not strictly worst-case per operation.

11. **Assuming each tree has exactly one valid heap arrangement.** When sinking hits a tie between children, either choice yields a valid heap. Different insertion orders of the same multiset give different (equally correct) heaps.

---

## Likely Exam Points

### 1. Identify valid binary min-heaps

**Q:** Which of the following are valid binary min-heaps? (a) root 5, children 6 and 2, with 6's children 8 and 0, and 2's children 3 and 5. (b) root 0, children 7 and 1, with 7's children 9 and nothing, and 1 having no children. (c) root 8, children 8 and 8, with the left 8 having children 8 and 8 and the right 8 having one child 7.

**A:** (a) is **complete** but **violates the min-heap property**: 6 > 0 (and 2 > 0 as well, following the subtree). Invalid. (b) is complete (bottom level left-packed with 9 as the leftmost slot) and obeys min-heap: 0 ≤ 7, 0 ≤ 1, 7 ≤ 9. **Valid.** (c) violates min-heap: a node with value 8 has child 7. Invalid. Remember equal values are fine (8 ≤ 8), so all-8s trees pass the heap property as long as they are complete.

### 2. Trace an insertion

**Q:** Insert 2 into the min-heap stored as `[1, 3, 4, 7, 5, 9, 6]`. Show the resulting array.

**A:** Place 2 at index 7: `[1, 3, 4, 7, 5, 9, 6, 2]`. Swim from 7: `parent(7) = (7-1)/2 = 3`, `keys[3] = 7 > 2`, swap → `[1, 3, 4, 2, 5, 9, 6, 7]`, now at index 3. `parent(3) = 1`, `keys[1] = 3 > 2`, swap → `[1, 2, 4, 3, 5, 9, 6, 7]`, now at index 1. `parent(1) = 0`, `keys[0] = 1`, not greater than 2, stop. Final: **`[1, 2, 4, 3, 5, 9, 6, 7]`**.

### 3. Trace a removal

**Q:** Call `removeSmallest()` on `[1, 2, 4, 3, 5, 9, 6, 7]`. What is returned and what is the resulting array?

**A:** Returns **1**. Move last element `7` to the root, size 7: `[7, 2, 4, 3, 5, 9, 6]`. Sink from 0: children at 1 (`2`) and 2 (`4`); smaller is 2; `2 < 7`, swap → `[2, 7, 4, 3, 5, 9, 6]`, at index 1. Children at 3 (`3`) and 4 (`5`); smaller is 3; `3 < 7`, swap → `[2, 3, 4, 7, 5, 9, 6]`, at index 3. Children would be at 7 and 8, out of range. Stop. Final: **`[2, 3, 4, 7, 5, 9, 6]`**.

### 4. Index arithmetic

**Q:** In a 0-indexed array heap (approach 3), what are the parent, left child, and right child of index 10? In a 1-indexed heap (approach 3b), what is the parent of index 10?

**A:** Approach 3: `parent(10) = (10-1)/2 = 9/2 = 4`; `leftChild(10) = 2(10)+1 = 21`; `rightChild(10) = 2(10)+2 = 22`. Approach 3b: `parent(10) = 10/2 = 5`. Always state which convention you are using.

### 5. Runtime table

**Q:** Fill in the worst-case runtimes for `add`, `getSmallest`, and `removeSmallest` for an ordered array, a bushy BST, a hash table, and a heap. Note any caveats.

**A:** Ordered array: Θ(N), Θ(1), Θ(N). Bushy BST: Θ(log N), Θ(log N), Θ(log N), but duplicate priorities are awkward and bushiness must be maintained. Hash table: Θ(1), Θ(N), Θ(N), useless for PQ because items land in random buckets. Heap: Θ(log N), Θ(1), Θ(log N), amortized because of array resizing, and handles duplicates naturally.

### 6. The top-M pattern

**Q:** You are streaming N numbers and want the M largest. Should you use a MinPQ or a MaxPQ, and why? What is the memory usage?

**A:** A **MinPQ** of capacity M. Add each item; if size exceeds M, `removeSmallest()`. The smallest of your current best-M is exactly the one that falls out of contention first, so evicting it is free. Memory is Θ(M), not Θ(N). (A MaxPQ would let you pop the largest, which is the opposite of what you want to discard.)

### 7. Why completeness matters

**Q:** Why does the binary heap require completeness? Give two distinct reasons.

**A:** (1) Completeness bounds the height at Θ(log N), guaranteeing logarithmic swim and sink. (2) Completeness means the parents array of approach 2 is always identical for a given size and therefore carries no information, so it can be dropped entirely: the structure is computable from indices (`parent(k) = (k-1)/2`). This makes the array representation possible, which saves roughly 2/3 of the memory versus approach 1a and makes "find the last position" an O(1) array lookup rather than a tree walk.

### 8. Tree representations

**Q:** A node in a sibling tree (approach 1c) representation has fields `favoredChild` and `sibling`. Node W has children X, Y, Z in that order. Describe the references.

**A:** `W.favoredChild = X`, `X.sibling = Y`, `Y.sibling = Z`, `Z.sibling = null`. W does not directly reference Y or Z; to reach Z you go W → X → Y → Z. This representation handles arbitrary branching factors with exactly two reference fields per node, at the cost of O(number of children) access to the i-th child.

### 9. Approach 2 vs disjoint sets

**Q:** Disjoint sets used a single `int[] parent` array. Approach 2 uses `Key[] keys` **and** `int[] parents`. Why the extra array?

**A:** In disjoint sets the elements *were* the integers 0 through N-1, so the index already identified the element and no separate key storage was needed. In approach 2 the nodes carry arbitrary keys (identities) distinct from their positions, so `keys[i]` is required to say what lives at node i while `parents[i]` says where node i sits in the structure.

### 10. Why the smaller child when sinking

**Q:** When sinking, why must you swap with the smaller child rather than either child?

**A:** Swapping with the larger child promotes it above its sibling, which is smaller, immediately violating the min-heap property at that position. Swapping with the smaller child promotes a value that is ≤ both the sinking node and its sibling, so the position you vacate is left valid.

### 11. Conceptual: LLRB vs red-black recap

**Q:** Java's `TreeMap` maintains a correspondence with which tree, and is the correspondence a bijection?

**A:** A 2-3-**4** tree, and it is **not** one-to-one: glue (red) links may lean either direction, so multiple red-black trees can encode the same 2-3-4 tree. LLRBs, by contrast, are in one-to-one correspondence with 2-3 trees because red links are forced to lean left.

---

## Summary

- **Search tree wrap-up**: BSTs are simple but imbalance-prone; 2-3/B-trees are balanced but painful and slow; LLRBs have easy insert and a bijection with 2-3 trees; Java's `TreeMap` is a non-left-leaning red-black tree corresponding (non-bijectively) to a 2-3-4 tree. Other options exist: AVL, WAVL (fastest), splay, treaps, skip lists, and hashing (coming later).
- **Priority queue** is an ADT with `add`, `getSmallest`, `removeSmallest`, `size`, retrieving by **key order**.
- **Killer application**: track the top M of a stream of N items using a MinPQ of size M, giving Θ(M) memory instead of Θ(N).
- **Bad implementations**: ordered array (Θ(N) add), hash table (Θ(N) getSmallest, random placement), bushy BST (all Θ(log N) but duplicates are awkward and bushiness is annoying).
- **Binary min-heap** = complete binary tree + min-heap property (every node ≤ both children). Weaker and friendlier than the BST invariant; duplicates are natural.
- **`getSmallest`** = return the root, Θ(1).
- **`add`** = insert at the next open bottom-left slot (preserving completeness by construction), then **swim** up while smaller than the parent. Θ(log N).
- **`removeSmallest`** = take the root, move the **last** element into the root (preserving completeness), then **sink** down, always swapping with the **smaller** child. Ties broken arbitrarily. Θ(log N).
- **Six tree representations**: 1a fixed-width child links (fastest, fixed branching), 1b array of child links (flexible, extra indirection), 1c firstborn/sibling links (odd but exam-worthy), 2 keys array + parents array (works for any tree), 3 keys array only (requires completeness), 3b same with index 0 empty.
- **The punchline**: for a complete tree, the parents array is always the same, so it carries no information and can be discarded. Structure becomes arithmetic: `parent(k) = (k-1)/2` (or `k/2` in 3b, where `leftChild(k) = 2k`, `rightChild(k) = 2k+1`).
- **Heap tradeoffs**: Θ(1) getSmallest, Θ(log N) amortized add/removeSmallest (resizing), natural duplicate handling, roughly 1/3 the memory of approach 1a. Comparison order is supplied via a `Comparator` passed to the constructor (or via a `Comparable` type bound for a default ordering).
- **Remaining structures in the course**: tries and graphs.
