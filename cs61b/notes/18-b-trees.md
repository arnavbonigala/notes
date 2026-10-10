<!-- Wed, Oct 07, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 18: B-Trees

## Overview

This lecture closes out the binary search tree story and opens the balanced-tree story. The first half formalizes **depth**, **height**, and **average depth**, uses them to pin down BST runtimes (height + 1 comparisons in the worst case, average depth + 1 on average), and then asks what real BSTs look like: worst case height is Θ(N), best case is Θ(log N), and randomly built trees have Θ(log N) height and average depth (expected average depth ~2 ln N, expected height ~4.311 ln N per Reed, 2003). Along the way Josh hammers on the distinction between **Big O** (an upper bound on *all* cases) and **worst case Big Theta** (an exact description of *one* case), since both "BST height is O(N)" and "BST height is O(N²)" are true statements while "best case BST height is Θ(N)" is false. The catch is that we cannot rely on random insertion order, because real data arrives **over time** (log timestamps, usernames signing up one by one, or an adversary adding Z1, Z2, Z3, ...), which produces spindly Θ(N) trees. The second half invents the fix from scratch: freeze the tree so no new leaves are ever created, "overstuff" leaf nodes instead, cap each node at **L** items, and when a node exceeds L, **split** it by pushing its left-middle item up into the parent, turning the parent into a node with one more item and one more child. Splits can chain-react upward, and splitting the **root** is the only event that increases height, and it increases it for every leaf at once. The resulting structure keeps two invariants (all leaves equidistant from the root; a non-leaf node with k items has exactly k+1 children), is therefore perfectly balanced, and gives O(L log N) = O(log N) `contains` and `add` for **any** insertion order. These are really called **B-trees**: L = 2 is a 2-3 tree, L = 3 is a 2-3-4 tree (also called a 2-4 tree).

The textbook section for this lecture (Chapter 17) only frames the topic: "we build off our knowledge of binary search trees to understand a new self-balancing search tree structure: B-Trees." All substantive content below comes from the slides and the lecture.

---

## Key Concepts

### 1. Depth, height, average depth (and why we care)

- **Depth of a node**: how far it is from the root, counted in links. The root is at depth 0.
- **Height of a tree**: the maximum depth over all nodes, that is, the depth of the deepest leaf.
- **Average depth of a tree**: the average of all node depths.

These are not trivia, they are *runtime proxies*:

- **Height determines the worst case** cost of `contains`: you walk at most height + 1 nodes, so you do at most height + 1 comparisons.
- **Average depth determines the average case** cost of `contains`: average depth + 1 comparisons for a random key that is in the tree.

The slide's example tree:

```
depth 0                       k
depth 1              e                 v
depth 2         b         g        p       y
depth 3       a   d     f   j        r       z
depth 4                               s
```

Here `depth(g) = 2` and `height(T) = 4`. The worst case lookup is `contains(s)`: compare against k, v, p, r, s, which is 5 = height + 1 comparisons. The slide computes average depth as

```
(0x1 + 1x2 + 2x4 + 3x6 + 4x1) / (1 + 2 + 4 + 6 + 1) = 2.35
```

and therefore an average case of 3.35 comparisons. *(extra context: the fraction as literally written on the slide evaluates to 32/14 ≈ 2.29, so do not memorize the decimal; memorize the method of weighting each depth by how many nodes sit at it.)*

### 2. Big O is not "worst case"

This is the lecture's most-repeated conceptual point, delivered via an in-class poll. Reconstructing the four statements from the transcript (the poll slide text was not captured):

| Statement | Class answer | Truth |
|---|---|---|
| Worst case BST height is Θ(N) | ~80% true | **True** (insert in sorted or reverse-sorted order) |
| Best case BST height is Θ(N) | 16% true | **False**, best case is Θ(log N) |
| BST height is O(N) | 64% true | **True** |
| BST height is O(N²) | 50% true | **True** |

The mental model Josh gives: Θ is "grows exactly like", O is "grows at most like". A *case* (best, worst, average) picks out a family of inputs; Big O with no case attached is a statement about *all* BSTs. "BST height is O(N²)" is true but useless, like saying "I am less than 38 feet tall": factually correct, informationally empty. He explicitly promises this style of question appears again on homework and on Midterm 2, and frames the discomfort of getting it wrong as the point of the exercise.

### 3. Random BSTs are bushy, but we cannot count on randomness

Simulation (`https://joshh.ug/61b/bst.html`) with random inserts:

- 50 random items: average depth about 7, theoretical best about 3.86, worst possible height 49. Not great, not terrible.
- About 550 random items: average depth 10.49, theoretical best 7.16. Far closer to the ideal than to 549.

The provable facts quoted (proofs out of scope):

- **Average depth** of a BST built from N random distinct inserts is ~2 ln N = Θ(log N), so average case `contains` is Θ(log N).
- **Height** is ~4.311 ln N (Reed, 2003, a 27-page proof), so even *worst case* `contains` on a randomly built tree is Θ(log N).
- With deletion included, random insert/delete keeps Θ(log N) height **if** you randomly choose between predecessor and successor in Hibbard deletion. Aside from lecture: always choosing the successor degrades height to about √N, for reasons Josh says he does not know.

Here `~` means "Big Theta but keep the multiplicative constant and drop lower-order terms" (so 3N + 5N² is ~5N²).

**Why we still have a problem.** The class guessed several wrong answers (ordered data, cost of shuffling, adversarial luck) before the real one: **you do not have the data up front**. Data arrives temporally. Examples from the slides and lecture:

```
add("01-Jan-2019, 10:31:00")
add("01-Jan-2019, 18:51:00")
add("02-Jan-2019, 00:05:00")
add("02-Jan-2019, 23:10:00")
```

or users signing up on your site in sequence, or a jerk registering Z1, Z2, Z3, Z4, ... and flattening your `TreeMap` into a linked list. If you had everything at once you could Knuth-shuffle it in linear time and be fine. You do not. So we need a structure robust to *any* arrival order.

### 4. Step 1 of the invention: freeze the tree, overstuff the leaves

The root cause of imbalance is **adding new leaves at the bottom**. So: *never add new leaves*. Build whatever tree you have, then "freeze" it: no new nodes allowed, only new items, which get packed into the existing leaf they would have landed under.

Starting from the (lucky, balanced) tree built by inserting 4, 2, 6, 1, 5, 3, 7:

```
        4
    2       6
  1   3   5   7
```

Freeze, then add 8 (goes right of 4, right of 6, joins leaf 7), then 0 (joins leaf 1), then 9:

```
        4
    2          6
  0 1  3   5   7 8 9
```

This is logically consistent: `contains(18)` on the slides' version walks 18 > 13 (right), 18 > 15 (right), then linearly scans the leaf [16 17 18 19]. Height never changes because leaf depths never change. **But** the leaves grow without bound, so lookups into them cost Θ(N) in the worst case. You have a tree whose first few keys are fast and whose everything-else is a giant array scan.

### 5. Step 2: cap node size at L, and push an item up

Set a limit **L** = max items per node. If a node ever exceeds L, hand one of its items to its parent.

First attempt, naive "move up": from

```
        13
    5           15
  2   7     14    16 17 18 19       (L = 3, this leaf has 4 items)
```

move 17 up into the parent to get `15 17` with children `14` and `16 18 19`. **This is broken**: 16 now sits in a subtree to the *right* of 17. Search order is violated.

The fix, which is the whole idea of the lecture: **pulling an item out of a full node splits that node into a left half and a right half.** Josh's physical analogy: when you eat too much you do not explode into pieces, you rupture into two halves at the point where the item leaves. The parent gains one item *and* one child:

```
        13
    5          15 17
  2   7     14    16    18 19
```

Now `contains(18)`: 18 > 13 go right; 18 > 15 so compare with 17; 18 > 17 go right; find 18. Examining one node costs O(L) comparisons, which is fine because L is a constant we chose.

**Which item goes up?** Arbitrary but fixed convention. The lecture uses the **left-middle** item. With L = 3 an overfull node has 4 items (left, left-middle, right-middle, right) and the left-middle goes up; right-middle would be equally valid, the outermost two would not make sense. With L = 2 an overfull node has 3 items and the single middle item goes up.

### 6. Chain reaction splitting, and the only way height grows

When an item is pushed into a parent, the parent may itself exceed L and split, pushing into *its* parent, and so on up to the root. Slide trace (L = 3), starting from

```
            13 17
     5        15        19 21
```

Add 22, 23 so the right leaf becomes `19 21 22 23` (4 items). Split out left-middle 21 into the root:

```
            13 17 21
     5        15      19     22 23
```

Add 24, 25 so `22 23 24 25` overflows. Split out 23:

```
            13 17 21 23
     5        15     19     22     24 25
```

Now the **root** has 4 items. Split out its left-middle, 17:

```
                  17
         13               21 23
      5     15        19    22    24 25
```

**Key observation (perfect balance).** Splitting a leaf or an internal node does not change the height at all. Splitting the **root** pushes *every* node down by exactly one level simultaneously. So no leaf can ever get deeper than another, and the tree is perfectly balanced at all times. Every operation is therefore guaranteed O(log N), with no "hope the data is random" caveat.

### 7. The invariants

Two properties fall out of the construction:

1. **All leaves are the same distance from the root.**
2. **A non-leaf node with k items has exactly k+1 children.**

The slides' impossible tree:

```
        4
    2 3      5 6 7
  1
```

This violates both: leaves `[1]` and `[5 6 7]` are at different depths, and the non-leaf `[2 3]` has 2 items but only 1 child when it must have 3. The lecture is explicit that these invariants are **not proven** in class (induction would be the tool), only argued to be intuitively plausible: the only way depth is ever added is to add it to everybody.

### 8. Terminology hill

- The real name for "splitting trees" is **B-trees**. Josh thinks "splitting tree" is the better name but he did not invent them.
- **L = 2** is a **2-3 tree**: max 2 items per node, max 3 non-null children.
- **L = 3** is a **2-3-4 tree**, also written **2-4 tree**: max 3 items per node, max 4 non-null children.
- The 2-3-4 / 2-3 names count **children**, not items. (Some tools, including the `tinyurl.com/balanceYD` demo, use "max-degree" to mean max number of children, so max-degree 3 is a 2-3 tree.)
- The "B" has never been explained by the authors. Comer's *The Ubiquitous B-Tree*: "balanced", "broad", or "bushy" might apply; some suggest Boeing; given his contributions it may as well be "Bayer"-trees.
- Two practical regimes: **small L** (L = 2 or 3) as a conceptually simple balanced search tree, which is today's use, and **very large L** (thousands) for databases and filesystems with very large records.

### 9. Height and runtime

With L the max items per node and N total items:

- Tallest possible B-tree: every non-leaf node holds only 1 item, so it degenerates to a binary shape, height ~log₂(N).
- Shortest possible: every node holds L items and has L+1 children, height ~log_{L+1}(N).
- Either way, **height is Θ(log N)**.

Slide examples: N = 26, L = 2, H = 2 is the best case (height grows like log₃ N). N = 8, L = 2, H = 2 is near worst case (height grows like log₂ N).

`contains`: at most H + 1 nodes inspected, at most L items inspected per node, so **O(HL) = O(L log N) = O(log N)** since L is a constant.

`add`: same traversal, plus at most H + 1 split operations, so again **O(HL) = O(log N)**.

### 10. Not covered

Deletion (extra slides only, explicitly out of scope) and how splitting works for L > 3 ("see some other class"). The lecture ends by teasing the next one: B-trees are *ugly to implement*, so we will see what we do instead (*extra context: red-black trees / left-leaning red-black trees, which are the standard CS 61B follow-up*).

---

## Definitions

- **Depth (of a node)**: the number of links from the root to that node. The root has depth 0.
- **Height (of a tree)**: the maximum depth of any node, equivalently the depth of the deepest leaf. A single-node tree has height 0.
- **Average depth (of a tree)**: the sum of all node depths divided by the number of nodes.
- **Spindly tree**: a tree with height Θ(N), the maximum possible shape, produced for a BST by inserting in sorted or reverse-sorted order.
- **Bushy tree**: a tree with height Θ(log N), the minimum possible shape, produced for a BST by inserting medians first (for example 4, 2, 6, ... for {1..7}).
- **Big O**: an upper bound that applies to all cases, not a synonym for worst case. "Height is O(N²)" is a true but weak claim about every BST.
- **Worst case Big Theta**: an exact asymptotic description of the worst-case input family. For BST height this is Θ(N).
- **`~` (tilde) notation**: like Big Theta, except you keep the leading multiplicative constant and only drop lower-order terms. 3N + 5N² is ~5N².
- **Overstuffed tree**: the intermediate invention where no new nodes may be created and extra items are packed into existing leaves. Balanced in height but with unbounded node size, hence Θ(N) worst case lookup.
- **L**: the maximum number of items permitted in a single node of a B-tree.
- **Split (split up)**: when a node exceeds L items, the chosen middle item is moved up into the parent and the node ruptures into a left node and a right node, which become two adjacent children of the parent.
- **Left-middle item**: in an overfull node of L+1 items, the item just left of center (index 1 when L = 3 or L = 2); the lecture's arbitrary but fixed choice for what gets pushed up.
- **Chain reaction splitting**: a split whose promoted item overfills the parent, causing the parent to split, and so on up toward the root.
- **B-tree of order L**: the splitting tree described above, with between 1 and L items per node (the root may transiently hold L+1 during an `add` before splitting).
- **2-3 tree**: a B-tree with L = 2; nodes have 2 or 3 children.
- **2-3-4 tree (2-4 tree)**: a B-tree with L = 3; nodes have 2, 3, or 4 children.
- **B-tree invariant 1**: all leaves are the same distance from the root.
- **B-tree invariant 2**: a non-leaf node with k items has exactly k+1 children.
- **Perfect balance**: the property, enjoyed by B-trees, that all leaves are at identical depth, so height is Θ(log N) for every insertion order.

---

## Worked Examples

### Example 1: Warmup, insert 4, 3, 2, 1, 6, 5, 7 into a BST

Reasoning the way the lecture did it, link by link:

- 4 becomes the root.
- 3 < 4, so 3 becomes the root's left child.
- 2 < 4, then 2 < 3, so 2 becomes 3's left child.
- 1 < 4, < 3, < 2, so 1 becomes 2's left child.
- 6 > 4, so 6 becomes the root's right child.
- 5 > 4, then 5 < 6, so 5 becomes 6's left child.
- 7 > 4, > 6, so 7 becomes 6's right child.

```
        4
    3       6
  2       5   7
1
```

Box-and-pointer view in words: there is one `Node` box for each key, each with a `left` and `right` reference field. The left spine 4 → 3 → 2 → 1 is a chain of boxes each pointing left to the next, with the other reference `null`. Nothing about the right subtree compensates for that chain, which is exactly why height is driven by the longest such chain.

### Example 2: Warmup 2, extremes for {1, 2, 3, 4, 5, 6, 7}

- **Maximum height ("spindly")**: insert 1, 2, 3, 4, 5, 6, 7 or 7, 6, 5, 4, 3, 2, 1. Every new key is larger (or smaller) than everything present, so it hangs off the bottom of a single chain. Height for N items is N - 1, that is **Θ(N)**.
- **Minimum height ("bushy")**: insert the median first, then the medians of each half: 4, then 2, then 6, then the rest in any order (4, 6, 2, ... works equally well). Height for N items is Θ(log N).

```
        4
    2       6
  1   3   5   7
```

### Example 3: searching an overstuffed tree, and why it is bad

```
        13
    5           15
  2   7     14    16 17 18 19
```

`contains(18)`:

1. Is 18 > 13? Yes, go right.
2. Is 18 > 15? Yes, go right.
3. Now scan the leaf array: 16 = 18? No. 17 = 18? No. 18 = 18? Yes, found.

Correct, but if the leaf is `[16 17 18 19 20 21 22 23 24]` the scan is linear in the number of items. The three top keys are fast and everything else is an array scan: **Θ(N) worst case**.

### Example 4: the broken fix versus the real fix

Overfull leaf (L = 3): `16 17 18 19` under parent `15`.

**Broken (move the item up without splitting):**

```
        13
    5           15 17
  2   7     14    16 18 19        <-- 16 is to the RIGHT of 17. Search order violated.
```

**Correct (split):** pull 17 up, rupture the leaf into `16` and `18 19`, and attach both as children of the now-3-child parent.

```
        13
    5           15 17
  2   7     14   16   18 19
```

Note the parent went from 1 item / 2 children to 2 items / 3 children, preserving invariant 2.

### Example 5: `add` understanding check, insert 20 then 21 (L = 3)

Start:

```
        13
    5           15 17
  2   7     14   16   18 19
```

Add 20: 20 > 13 right, 20 > 17 right, leaf becomes `18 19 20`. Legal (3 items ≤ L).
Add 21: same path, leaf becomes `18 19 20 21`, which is 4 > L. Split out the left-middle, 19:

```
              13
    5                  15 17 19
  2   7        14    16    18    20 21
```

The parent now has 3 items and 4 children. Still legal, no chain reaction needed.

### Example 6: chain reaction, insert 25 then 26 (L = 3)

Continuing from Example 5, add 25 and 26 so the rightmost leaf is `20 21 25 26`. Split out 21:

```
              13
    5                  15 17 19 21
  2   7        14    16    18    20    25 26
```

That node now has 4 items, which is over the cap. Split out its left-middle, 17, pushing it into the root:

```
                     13 17
       5                15                19 21
   2      7          14    16         18     20     25 26
```

Height did not change, because the split happened below the root. The 4 items that used to be in one node are now spread over a parent item plus two nodes, and the 5 children redistributed 2-to-the-left / 3-to-the-right, exactly matching the k items / k+1 children rule on both halves.

### Example 7: the root splits, height grows for everyone (L = 3)

```
            13 17 21 23
     5        15     19     22     24 25
```

Split out the root's left-middle, 17. The left half keeps `13` with children `5` and `15`; the right half keeps `21 23` with children `19`, `22`, and `24 25`; 17 becomes a brand new root:

```
                  17
         13               21 23
      5     15        19    22    24 25
```

Every leaf moved from depth 1 to depth 2 **simultaneously**. This is the only operation in the entire data structure that changes height, and it changes it uniformly. That single fact is the proof sketch for perfect balance.

A student asked during this step why the grandchildren "happened to land in the right place." The answer given: they were already attached in the correct order before the split, and the rupture only cuts the parent, so the child pointers to the left of the promoted item stay with the left half and those to the right stay with the right half. For instance 15 must hold keys greater than 13 and less than 17, which is precisely where it ends up.

### Example 8: insert 1, 2, 3, 4, 5, 6, 7 into a 2-3 tree (L = 2), full trace

For L = 2, an overfull node has 3 items and the **middle** one goes up.

```
add 1:   [1]
add 2:   [1 2]
add 3:   [1 2 3]  -> overfull, split up 2

              2
           1     3

add 4:        2
           1     3 4

add 5:        2
           1    3 4 5    -> overfull, split up 4

             2 4
          1   3   5

add 6:       2 4
          1   3   5 6

add 7:       2 4
          1   3   5 6 7  -> overfull, split up 6

            2 4 6
         1   3   5   7   -> root overfull, split up 4

               4
           2       6
         1   3   5   7
```

This is the punchline of the lecture: the *same* sorted insertion order that produced a 7-deep spindly BST in Example 1's cousin produces a **perfectly balanced** tree here, with all leaves at depth 2.

### Example 9: forcing height 1 (recorded-viewers exercise)

Find an order for 1..7 whose resulting 2-3 tree has height 1. One answer from the slides: **2, 3, 4, 5, 6, 1, 7**, giving

```
        3 5
    1 2   4   6 7
```

with all leaves at depth 1. The takeaway stated on the slide: no matter the insertion order, the resulting B-tree is always bushy. Height can vary a little, bushiness cannot.

### Example 10: Java, measuring height and average depth *(extra context: the lecture wrote no code; this makes the Section 1 definitions concrete)*

```java
public class TreeStats {
    private static class Node {
        int key;
        Node left, right;
    }

    /** Height with the lecture's convention: a single node has height 0. */
    public static int height(Node x) {
        if (x == null) {
            return -1;                                   // empty tree, so a leaf gets 0
        }
        return 1 + Math.max(height(x.left), height(x.right));
    }

    private static int size(Node x) {
        return (x == null) ? 0 : 1 + size(x.left) + size(x.right);
    }

    /** Sum of the depths of every node in the subtree rooted at x, where x sits at depth d. */
    private static int depthSum(Node x, int d) {
        if (x == null) {
            return 0;
        }
        return d + depthSum(x.left, d + 1) + depthSum(x.right, d + 1);
    }

    public static double averageDepth(Node root) {
        return (double) depthSum(root, 0) / size(root);
    }
}
```

What this does and why: `height` recurses to the deepest `null` and unwinds adding 1 per link, which is literally "max depth over all nodes." `depthSum` threads the current depth *downward* as a parameter (each recursive call passes `d + 1`, so each stack frame's `d` is that node's depth) and sums upward, which is exactly the numerator `0x1 + 1x2 + 2x4 + ...` on the slide: instead of grouping nodes by depth and multiplying, it visits each node once and adds its own depth. `height(root) + 1` is the worst case comparison count for `contains`; `averageDepth(root) + 1` is the average case count.

### Example 11: Java, a B-tree node, `contains`, and splitting *(extra context: reconstructed implementation; CS 61B does not ask you to implement B-trees, and the next lecture explains why they are painful. Study it to make the invariants concrete, not to memorize.)*

```java
public class BTree {
    private final int L;          // max items allowed to live in a node
    private Node root;

    private class Node {
        int[] items = new int[L + 1];    // one spare slot: we overstuff, then split
        Node[] kids = new Node[L + 2];   // kids[i] holds keys between items[i-1] and items[i]
        int n;                           // how many items are actually present

        boolean isLeaf() {
            return kids[0] == null;      // invariant 2: a non-leaf with n items has n+1 kids
        }
    }

    public BTree(int maxItemsPerNode) {
        L = maxItemsPerNode;
        root = new Node();
    }
```

**`contains`, which is "almost exactly like a normal BST":**

```java
    public boolean contains(int key) {
        Node p = root;
        while (p != null) {
            int i = 0;
            while (i < p.n && key > p.items[i]) {
                i += 1;                      // scan this node's items: at most L compares
            }
            if (i < p.n && key == p.items[i]) {
                return true;
            }
            p = p.kids[i];                   // null when p is a leaf, so the key is absent
        }
        return false;
    }
```

Step by step on the Example 4 tree with `contains(18)`: `p` points at the root box `[13]`. The inner scan stops at `i = 1` because 18 > 13, so `p = kids[1]`, the box `[15 17]`. There the scan passes 15 and 17, stopping at `i = 2`, so `p = kids[2]`, the box `[18 19]`. Now the scan stops immediately at `i = 0` because 18 is not greater than 18, and `items[0] == 18`, so we return `true`. Three node visits, each costing at most L comparisons: that is the O(HL) bound from the slides, in code.

**`add`, with bottom-up splitting:**

```java
    public void add(int key) {
        insert(root, key);
        if (root.n > L) {                    // the root overflowed
            Node newRoot = new Node();
            newRoot.kids[0] = root;
            root = newRoot;
            splitChild(newRoot, 0);          // old root ruptures; tree gets exactly one taller
        }
    }

    /** Inserts key below p. May leave p holding L + 1 items, one too many. */
    private void insert(Node p, int key) {
        int i = 0;
        while (i < p.n && key > p.items[i]) {
            i += 1;
        }
        if (i < p.n && key == p.items[i]) {
            return;                          // no duplicates: our trees use strict < only
        }
        if (p.isLeaf()) {
            for (int j = p.n; j > i; j -= 1) {
                p.items[j] = p.items[j - 1]; // make room, keeping the node sorted
            }
            p.items[i] = key;
            p.n += 1;
            return;                          // adds ALWAYS happen in an existing leaf
        }
        insert(p.kids[i], key);
        if (p.kids[i].n > L) {
            splitChild(p, i);                // p gains one item and one child
        }
    }

    /** Splits the overstuffed child p.kids[i], pushing its left-middle item up into p. */
    private void splitChild(Node p, int i) {
        Node c = p.kids[i];                  // c.n == L + 1
        int mid = L / 2;                     // "left-middle": index 1 for both L = 2 and L = 3
        Node right = new Node();

        for (int j = mid + 1; j < c.n; j += 1) {
            right.items[j - mid - 1] = c.items[j];      // items right of the rupture
        }
        for (int j = mid + 1; j <= c.n; j += 1) {
            right.kids[j - mid - 1] = c.kids[j];        // and their children, one more than items
            c.kids[j] = null;
        }
        right.n = c.n - mid - 1;

        int promoted = c.items[mid];
        c.n = mid;                           // c keeps items[0..mid-1] and kids[0..mid]

        for (int j = p.n; j > i; j -= 1) {
            p.items[j] = p.items[j - 1];     // open a slot in the parent
        }
        for (int j = p.n + 1; j > i + 1; j -= 1) {
            p.kids[j] = p.kids[j - 1];
        }
        p.items[i] = promoted;
        p.kids[i + 1] = right;
        p.n += 1;                            // p may now hold L + 1 items; the caller splits it
    }
}
```

Why this mirrors the lecture exactly:

- `insert` only ever creates items inside an **existing leaf**. No new leaf is ever appended at the bottom, which was the whole point of "freezing" the tree.
- Each node's array has one spare slot, which is the "overstuffing" step. The overstuffing is transient: the caller immediately checks `kids[i].n > L` and splits.
- `splitChild` moves `promoted` up and partitions both the items and the children at the rupture point. The `right.kids` loop runs one step further than the `right.items` loop, which is invariant 2 (k items need k+1 children) expressed as array arithmetic. For a leaf, every copied child is `null`, so `right.isLeaf()` stays true and the structure of leaves is preserved.
- Because `insert` recurses first and splits afterward, overflow propagates **upward**, which is the chain reaction.
- Height changes in exactly one place: the `if (root.n > L)` block in `add`. A fresh root is allocated above the old one, so every existing node's depth increases by one at the same instant. That is perfect balance, enforced structurally rather than by any rebalancing logic.

Environment/box-and-pointer reasoning for a root split: before the split there is one root box with L+1 items and L+2 child pointers. After it, there are three boxes in play: the new root (1 item, 2 pointers), the old box trimmed to `mid` items (so it only looks at `kids[0..mid]`), and a fresh `right` box holding the suffix of items and the suffix of child pointers. No child box is ever copied or re-created; only the pointers to them are redistributed between two parents.

---

## Common Pitfalls

1. **Saying "Big O" when you mean "worst case."** They are different axes. A case restricts the inputs; O/Θ describe growth. "Best case height is O(N)" is true (and weak); "best case height is Θ(N)" is false.
2. **Thinking a loose Big O bound is wrong.** `O(N²)` for BST height is a correct statement. On an exam, mark it true even though it is uninformative.
3. **Assuming random insert order saves you.** It does, mathematically, but you rarely get to choose the order because data arrives over time. Being able to *state why* (temporal arrival, adversarial usernames, logged timestamps) is part of the lecture.
4. **Moving an item up without splitting the node.** This is the single most common B-tree error. If you promote the left-middle item and leave the rest of the node intact, keys end up on the wrong side of their separator (16 to the right of 17).
5. **Forgetting to re-check the parent after a promotion.** Splits chain. In Example 6 the first split fixed the leaf but broke the parent; the answer is not done until nothing exceeds L.
6. **Creating a new leaf.** B-tree `add` never grows the tree downward. If your drawing has a new node dangling below an old leaf, you have drawn a BST, not a B-tree.
7. **Violating the k items / k+1 children rule.** After drawing a split, count: a node with 2 items must show 3 children, a node with 3 items must show 4.
8. **Drawing leaves at different depths.** Impossible in a B-tree. If your picture has one, you made an arithmetic slip earlier in the trace.
9. **Confusing the "2-3-4" naming with item counts.** 2-3-4 means 2, 3, or 4 **children**, so L = 3 **items**. A 2-3 tree is L = 2. Similarly, demos that say "max-degree 3" mean a 2-3 tree.
10. **Dropping L from the runtime analysis mid-proof.** The honest chain is O(HL) → O(L log N) → O(log N) *because L is a constant we chose*. If L were allowed to grow with N (as in database B-trees with L in the thousands) you could not drop it so casually.
11. **Picking the wrong item to promote for L = 2.** With 3 items there is a single true middle; "left-middle" collapses to it. Do not promote an outer item.
12. **Height off-by-one.** A single node has height 0, and `contains` costs height + 1 comparisons in the worst case. Do not report the comparison count as the height.
13. **Expecting deletion.** Not covered, explicitly out of scope, extra slides only.

---

## Likely Exam Points

**1. Big O versus worst case Big Theta.**
*Q:* True or false, with justification: (a) the worst case height of a BST on N nodes is Θ(N); (b) the best case height is Θ(N); (c) BST height is O(N); (d) BST height is O(N log N).
*A:* (a) True, insert in sorted order. (b) False, the best case is Θ(log N). (c) True, no BST on N nodes is taller than N - 1. (d) True, N log N is a valid (loose) upper bound on something that is at most N. The trap is (b): a *case* pins the input, so Θ must then be exact.

**2. Draw the B-tree after a given insertion sequence.**
*Q:* Insert 1, 2, 3, 4, 5 into a 2-3 tree (L = 2), promoting the middle item. Draw the final tree and give its height.
*A:* `[1]` → `[1 2]` → `[1 2 3]` splits to root `[2]` with children `[1]`, `[3]` → `[3 4]` → `[3 4 5]` splits to root `[2 4]` with children `[1]`, `[3]`, `[5]`. Height 1.

**3. Chain reaction up through the root.**
*Q:* In a 2-3-4 tree (L = 3), the root is `[13 17 21 23]` with children `[5] [15] [19] [22] [24 25]`. Show the result of splitting the root using the left-middle convention, and state what happens to the height.
*A:* Root becomes `[17]`; left child `[13]` with children `[5]`, `[15]`; right child `[21 23]` with children `[19]`, `[22]`, `[24 25]`. Height increases by exactly 1, and it increases for every leaf at once, which is why balance is preserved.

**4. Spot the invariant violation.**
*Q:* Why is the following not a valid B-tree? Root `[4]`, children `[2 3]` and `[5 6 7]`, where `[2 3]` has a single child `[1]`.
*A:* Two violations. The leaves `[1]` and `[5 6 7]` are at different distances from the root, breaking invariant 1. The non-leaf `[2 3]` has k = 2 items but only 1 child, when it must have exactly k+1 = 3, breaking invariant 2.

**5. Terminology translation.**
*Q:* A 2-3-4 tree is a B-tree with what value of L? How many items and how many non-null children can one of its nodes have? What about a 2-3 tree?
*A:* 2-3-4 (also called 2-4) is L = 3: at most 3 items and at most 4 non-null children per node. 2-3 is L = 2: at most 2 items and at most 3 non-null children. The names count children, not items.

**6. Height bounds as a function of L and N.**
*Q:* Give the asymptotic range of a B-tree's height with limit L, and say which extreme corresponds to which node occupancy.
*A:* Between ~log_{L+1}(N) (every node holds L items and L+1 children, maximally bushy) and ~log₂(N) (every non-leaf holds just 1 item, so the tree is effectively binary). Both are Θ(log N), so B-tree operations are Θ(log N) height regardless.

**7. Runtime derivation, not just the answer.**
*Q:* Derive the worst case runtime of `contains` and of `add` on a B-tree with limit L.
*A:* `contains` inspects at most H + 1 nodes and at most L items per node, giving O(HL). `add` does the same traversal plus at most H + 1 splits, also O(HL). Since H = Θ(log N), both are O(L log N), and since L is a constant, both are O(log N).

**8. Counting items for a given height (extension of the slide's N = 26 and N = 8 examples).**
*Q:* What are the minimum and maximum number of items in a 2-3 tree (L = 2) of height 2?
*A:* Minimum: every node holds 1 item with 2 children, a perfect binary tree of 3 levels, so 2³ - 1 = 7 items (the slide's "near worst case" example has 8 items at H = 2). Maximum: every node holds 2 items with 3 children, so 1 + 3 + 9 = 13 nodes times 2 items = 26 items, matching the slide's best case. *(extra context: in general, min = 2^(h+1) - 1 and max = (L+1)^(h+1) - 1 items.)*

**9. Why BSTs are not good enough.**
*Q:* Random inserts give Θ(log N) BST height. Give a concrete reason we still need balanced trees.
*A:* We usually cannot control insertion order because data arrives over time: event timestamps inserted as they occur, or users registering sequentially. A monotone arrival order (or an adversary adding Z1, Z2, Z3, ...) builds a spindly Θ(N) tree. B-trees handle *any* order in O(log N).

**10. Height, depth, average depth, comparison counts.**
*Q:* For the tree with root k, children e and v, with e's children b and g, v's children p (left empty, right child r) and y (right child z), r's right child s, b's children a and d, g's children f and j: what is `height(T)`, `depth(g)`, and the worst case number of comparisons for `contains`?
*A:* `height(T) = 4` (the path k, v, p, r, s), `depth(g) = 2`, and the worst case comparison count is height + 1 = 5, realized by `contains(s)`.

**11. Insertion order to hit a target shape.**
*Q:* Give an order for inserting 1 through 7 so the resulting 2-3 tree has height 1.
*A:* 2, 3, 4, 5, 6, 1, 7 yields root `[3 5]` with children `[1 2]`, `[4]`, `[6 7]`, all leaves at depth 1.

**12. What makes B-trees balanced.**
*Q:* In one sentence, why is a B-tree always perfectly balanced?
*A:* Splitting a leaf or an internal node does not change any leaf's depth, and splitting the root increases every leaf's depth by exactly one, so all leaves remain at identical depth forever.

---

## Summary

- **Depth** of a node is its distance from the root (root = 0); **height** is the max depth; **average depth** is the mean over nodes. Worst case `contains` costs height + 1 comparisons, average case costs average depth + 1.
- **BSTs** have best case height Θ(log N) (bushy) and worst case Θ(N) (spindly, from sorted or reverse-sorted insertion).
- **Big O is not worst case.** "Height is O(N)" and "height is O(N²)" are both true; "best case height is Θ(N)" is false. Expect this on Midterm 2.
- **Random** inserts give Θ(log N) height: expected average depth ~2 ln N, expected height ~4.311 ln N (Reed 2003). With deletion, randomly alternating predecessor/successor keeps Θ(log N); always using the successor degrades toward ~√N.
- We still cannot rely on this, because **data arrives over time** (timestamps, sign-ups, adversarial Z1/Z2/Z3), and we do not get to shuffle it.
- **The invention, in three steps:** (1) never add new leaves, overstuff existing ones instead, which freezes height but makes lookups Θ(N); (2) cap nodes at **L** items; (3) when a node exceeds L, **split** it, promoting its left-middle item into the parent and rupturing the node into two children.
- Simply moving an item up **without** splitting breaks search order. Splitting is the fix, and it makes the parent a node with one more item and one more child.
- Splits **chain-react** upward. Splitting a leaf or internal node leaves height unchanged; **splitting the root is the only way height grows**, and it grows for every leaf simultaneously.
- **Invariants** (asserted, not proven in lecture): all leaves are the same distance from the root, and a non-leaf node with k items has exactly k+1 children. Together they force perfect balance and bushiness.
- **Names:** these are **B-trees**. L = 2 is a **2-3 tree**; L = 3 is a **2-3-4** or **2-4 tree**; the digits count children. Small L is a teaching-friendly balanced tree; L in the thousands is used in real databases and filesystems. The "B" is unexplained (balanced, broad, bushy, Boeing, or Bayer).
- **Height** lies between ~log_{L+1}(N) and ~log₂(N), so Θ(log N). `contains` and `add` are both O(HL) = O(L log N) = **O(log N)** for constant L.
- **Not covered:** deletion (extra slides, out of scope) and splitting for L > 3. **Next up:** B-trees are ugly to implement, so we will see the alternative *(extra context: red-black trees)*.
