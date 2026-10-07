<!-- Wed, Oct 07, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 18: B-Trees

This lecture closes out the "BSTs are only fast if they are bushy" story and gives the first real fix. We start by making the height/depth vocabulary precise, separate the idea of *Big O* (an upper bound on a function) from *worst case* (a choice of input), and observe empirically and mathematically that BSTs built from *random* inserts have Θ(log N) height (~2 ln N average depth, ~4.311 ln N height). But real programs cannot always insert in random order (timestamped event data arrives in sorted order), and sorted inserts give a spindly Θ(N) tree. The fix developed in lecture is deliberately strange at first: never add new leaves at the bottom, instead "overstuff" existing leaves. Overstuffing keeps height perfectly balanced but makes nodes arbitrarily juicy, so we cap each node at L items and, when a node overflows, push one item up into the parent while *splitting* the overflowing node into two children. Splits can chain-react up to the root, and splitting the root is the only event that changes the height, pushing every leaf down by exactly one level. The result is a perfectly balanced "splitting tree", whose real name is a **B-tree** (L=2 gives a 2-3 tree, L=3 gives a 2-3-4 tree), with two invariants (all leaves at the same depth; a non-leaf with k items has exactly k+1 children) that guarantee bushiness and therefore O(log N) `contains` and `add` for *any* insertion order.

---

## Key Concepts

### 1. Depth, height, average depth, and why we care

For the BST drawn on the slides (root `k`; depth 1: `e`, `v`; depth 2: `b`, `g`, `p`, `y`; depth 3: `a`, `d`, `f`, `j`, `r`, `z`; depth 4: `s`):

```
depth 0                 k
depth 1          e             v
depth 2       b     g      p      y
depth 3      a d   f j    r      z
depth 4                  s
```

- **depth(node)**: number of links from the root down to that node. `depth(g) = 2`.
- **height(tree)**: depth of the deepest leaf. `height(T) = 4`.
- **average depth**: average over all nodes of their depth. The slide computes it as a weighted average of (depth × number of nodes at that depth) divided by the total node count, reporting ≈ 2.35 for this tree.

The reason these two numbers matter is a direct mapping to runtime:

| property | what it determines | slide example |
|---|---|---|
| height | **worst case** number of comparisons to find a key: height + 1 | `contains(s)` takes 5 comparisons |
| average depth | **average case** number of comparisons: average depth + 1 | ≈ 3.35 comparisons |

The "+1" is because a search that walks from the root to a node at depth d touches d+1 nodes.

> Note: the exact per-level node counts come from a slide diagram, and the arithmetic as transcribed from the PDF does not quite land on 2.35, but the method (weighted average of depths) and the reported values 2.35 / 3.35 are what the lecture used. (extra context: the discrepancy is almost certainly an artifact of the PDF text extraction, not something you need to reconcile.)

### 2. Big O is not the same thing as "worst case"

This is one of the few genuinely conceptual points of the lecture, and it is tested.

- A BST on N keys has **best case height Θ(log N)** (perfectly bushy) and **worst case height Θ(N)** (a spindly chain).
- Because the height depends on the *shape*, there is no single Θ bound on "the height of a BST with N keys". There is an upper bound: the height is O(N) always.
- So: *Big O* is a statement about bounding a function from above. *Worst case* is a statement about which input you chose. You can combine them ("worst case height is Θ(N)"), but they are different axes. Saying "Big O means worst case" is wrong.

Warmup 1 (from slides): insert 4, 3, 2, 1, 6, 5, 7 into a BST.

```
        4
      /   \
     3     6
    /     / \
   2     5   7
  /
 1
```
Height 3.

Warmup 2: with the key set {1, ..., 7},
- **maximum height ("spindly")**: insert in sorted order 1, 2, 3, 4, 5, 6, 7, giving a right-leaning chain of height 6, i.e. Θ(N) height;
- **minimum height ("bushy")**: insert 4, 2, 6, 1, 3, 5, 7, giving a complete tree of height 2, i.e. Θ(log N) height.

### 3. Real-world BSTs: the randomized case is good news

The lecture uses a simulation (https://joshh.ug/61b/bst.html) plus known results:

- **Average depth** of a BST built by inserting N distinct keys in random order is ~2 ln N = Θ(log N), so average-case `contains` on such a tree is Θ(log N).
- **Height** of a BST built from N random inserts is ~4.311 ln N (Reed, 2003), so even the worst-case search in such a tree is Θ(log N). The proof is 27 pages and far out of scope.
- `~` here means "Big Theta, but keeping the multiplicative constant".
- Random trees including *deletions* are still Θ(log N) height, provided you randomly choose between predecessor and successor in Hibbard deletion. (Covered only in the extra slides.)

Bottom line: random inserts give Θ(log N) per operation.

### 4. The bad news that motivates B-trees

We usually cannot choose a random insertion order, because data arrives over time and is often already sorted. The lecture's example is storing event timestamps:

```java
add("01-Jan-2019, 10:31:00");
add("01-Jan-2019, 18:51:00");
add("02-Jan-2019, 00:05:00");
add("02-Jan-2019, 23:10:00");
```

Each new key is larger than all previous ones, so the tree degenerates into a right-leaning chain: Θ(N) height, Θ(N) `contains`. Something structurally different is needed.

### 5. Crazy idea: never add leaves at the bottom (overstuffing)

The *only* way a BST gets taller is by hanging a new leaf below an existing leaf. So: forbid that. When a key arrives, cram it into the existing leaf where it belongs.

```
            13                             13
      5          15        add 17,18   5        15
    2   7     14   16      ------->  2   7   14   16 17 18
```

Because no leaf ever moves to a new depth, and height is just max(depth), the height of an **overstuffed tree** literally cannot change. It is perfectly balanced by construction.

It is still a logically consistent search structure. `contains(18)` on the tree with leaf `16 17 18 19`:
- 18 > 13, go right
- 18 > 15, go right
- is 16 == 18? no
- is 17 == 18? no
- is 18 == 18? yes, found it

The problem: a leaf can hold unboundedly many items, e.g. `16 17 18 19 20 21 22 23 24`. Scanning that one node is Θ(N). We traded a tall tree for a fat node, which is no better.

### 6. First repair attempt (broken): push an item up without splitting

Fix: set a limit **L** on items per node (say L=3). If a node exceeds L items, hand one item (say, arbitrarily, the left-middle one) to the parent.

```
            13                                   13
      5           15                      5            15 17
   2     7     14   16 17 18 19   --->  2    7     14    16 18 19
```

This is **wrong**, and the lecture asks you to spot why: the parent `15 17` now has only three children, so the leaf `16 18 19` sits in the "greater than 17" slot, yet it contains 16. The key 16 is now to the *right* of 17, so `contains(16)` would never find it. Search-tree ordering is violated.

### 7. The actual repair: node splitting

Pulling an item out of a full node **splits** that node into a left half and a right half, so the parent gains both an item and a child:

```
            13                                   13
      5           15                      5            15 17
   2     7     14   16 17 18 19   --->  2    7     14   16   18 19
```

Now the parent has items `15 17` and three children `14`, `16`, `18 19`, which are exactly the "< 15", "between 15 and 17", "> 17" ranges. `contains(18)`: 18 > 13 go right; compare against 15 then 17; 18 > 17 so go to the rightmost child; scan `18 19` and find it.

Examining one node costs O(L) comparisons, but L is a constant chosen by us, so that is fine.

Promotion rule used in lecture:
- **L=2 (2-3 tree)**: a node with 3 items splits and passes the **middle** item up. `[1 2 3]` becomes `[1]` and `[3]` with `2` promoted.
- **L=3 (2-3-4 tree)**: a node with 4 items splits and passes the **left-middle** item up. `[16 17 18 19]` becomes `[16]` and `[18 19]` with `17` promoted.
- The lecture explicitly does **not** cover how splitting works for L > 3.

### 8. Chain reaction splitting

A promotion can overflow the parent, which then splits and promotes into *its* parent, and so on. The lecture's example, adding 25 then 26 to

```
                 13
       5                    15 17 19
    2     7         14    16    18    20 21
```
- 25 and 26 land in the leaf `20 21`, making it `20 21 25 26`, which has 4 > L=3 items.
- Split it, promoting the left-middle item 21:
```
                 13
       5                 15 17 19 21
    2     7        14   16   18   20   25 26
```
- Now the node `15 17 19 21` has 4 items. Split it, promoting left-middle item 17:
```
                      13 17
          5          15          19 21
       2    7     14    16    18   20   25 26
```

### 9. What happens when the root is too full

The root is split the same way, except the promoted item has nowhere to go, so it becomes a brand new root with two children. From the slides:

```
            13 17 21                                13 17 21 23
    5    15    19    22 23 24 25   -- split(23) -->  5  15  19  22  24 25
```
and then the root itself overflows:

```
                       17
            13                  21 23
        5       15         19     22    24 25
```

### 10. Perfect balance, and the invariants

**Observation (perfect balance).**
- Splitting a leaf or an internal node does not change the height at all; it only widens a level.
- Splitting the **root** pushes *every* node down by exactly one level, so all leaves stay at the same depth.

Therefore all leaves are always at the same depth, and all operations are O(log N).

**The two B-tree invariants.**
1. All leaves are the same distance from the root.
2. A non-leaf node with k items has exactly k+1 children.

The lecture's example of an *impossible* tree:

```
            4
      2 3       5 6 7
    1
```
It violates both: leaf `1` and leaf `5 6 7` are at different depths, and the non-leaf `2 3` has 2 items but only 1 child when it should have 3.

These invariants are stated as intuitively plausible consequences of how splitting works, not proved rigorously in lecture.

**Consequence:** no matter the insertion order, the resulting B-tree is bushy. Heights may differ slightly between orders, but bushiness is guaranteed. Insert 1, 2, 3, 4, 5, 6, 7 in order into a 2-3 tree and you get height 2 with all leaves at depth 2; insert 2, 3, 4, 5, 6, 1, 7 and you get height 1:

```
        3 5
    1 2  4  6 7        (all leaves at depth 1)
```

Interactive demo used in lecture: https://tinyurl.com/balanceYD (there, "max-degree" means maximum number of *children*, so max-degree 3 is a 2-3 tree).

### 11. Terminology

"Splitting tree" is the honest name, but the real name is **B-tree**.

- B-tree of order **L=2**: max 2 items, max 3 non-null children per node, also called a **2-3 tree**.
- B-tree of order **L=3**: max 3 items, max 4 non-null children per node, also called a **2-3-4 tree** or **2-4 tree**. The "2-3-4" names the legal numbers of children.
- Two popular regimes: small L (L=2 or L=3) as a conceptually simple balanced search tree, which is what this lecture does; and very large L (thousands) for databases and filesystems with very large records.
- Nobody knows what the B stands for. Quoting Comer (*The Ubiquitous B-Tree*): "balanced", "broad", or "bushy" might apply, some suggest Boeing, and it seems appropriate to think of them as "Bayer"-trees.

(extra context) Be aware that outside 61B, "order" of a B-tree usually means the maximum number of *children*, not items. This lecture's L is the max number of **items** per node. Use the course's convention on exams.

### 12. Height and runtime

Let L be the max items per node, N the number of items, H the height.

- **Tallest possible** B-tree: every non-leaf node holds just 1 item (so 2 children), giving H ≈ log₂(N).
- **Shortest possible**: every node holds L items (so L+1 children), giving H ≈ log_{L+1}(N).
- So H is between ~log_{L+1}(N) and ~log₂(N), and since both are Θ(log N), **H = Θ(log N)**.

Slide sanity checks: N = 26 items with L = 2 can reach H = 2 (best case, height grows like log₃ N); N = 8 items with L = 2 is near worst case and still has H = 2 (height grows like log₂ N).

**`contains`**
- nodes inspected: H + 1
- items inspected per node: up to L
- total: O(HL) = O(L log N) = **O(log N)** since L is a constant.

**`add`**
- nodes inspected: H + 1
- items per node: up to L
- split operations: up to H + 1 (full chain reaction all the way to the root)
- total: O(HL) = O(L log N) = **O(log N)**.

Not covered (and in the "extra slides" / out of scope): deletion, and splitting for L > 3.

---

## Definitions

- **depth (of a node)**: the number of edges on the path from the root to that node; the root has depth 0.
- **height (of a tree)**: the depth of its deepest leaf, i.e. max over all nodes of depth.
- **average depth (of a tree)**: the arithmetic mean of the depths of all nodes in the tree.
- **spindly tree**: a tree whose height is Θ(N), the maximum possible for N nodes.
- **bushy tree**: a tree whose height is Θ(log N), the minimum order of growth possible for N nodes.
- **Big O**: an upper bound on a function. It is *not* a synonym for "worst case"; worst case refers to a choice of input, Big O refers to bounding a function from above.
- **random BST**: a BST produced by inserting N distinct keys in a uniformly random order. Expected average depth ~2 ln N; expected height ~4.311 ln N.
- **overstuffed tree**: a search tree in which new keys are inserted into existing leaf nodes rather than hanging new leaves, so leaf depths (and thus the height) never change. Logically correct but allows unbounded node size.
- **L (order, in this course)**: the maximum number of items permitted in a single node of a B-tree.
- **juicy / too full node**: a node holding more than L items; it must be split.
- **split**: the operation that removes one item (the middle for L=2, the left-middle for L=3) from an over-full node, promotes it into the parent, and divides the remaining items into a left node and a right node, which become two adjacent children of the parent.
- **chain reaction splitting**: the cascade in which promoting an item overflows the parent, which splits and promotes into its parent, possibly repeating up to the root.
- **root split**: splitting the root; the promoted item becomes a new root with exactly two children, and the height of the tree increases by exactly 1 for every leaf simultaneously.
- **B-tree**: the real name for a splitting tree; a search tree whose nodes hold between 1 and L items, in which all leaves are equidistant from the root and every non-leaf node with k items has exactly k+1 children.
- **2-3 tree**: a B-tree with L = 2 (max 2 items, max 3 children per node).
- **2-3-4 tree (= 2-4 tree)**: a B-tree with L = 3 (max 3 items, max 4 children per node).
- **B-tree invariant 1**: all leaves are the same distance from the root.
- **B-tree invariant 2**: a non-leaf node with k items has exactly k+1 children.
- **perfect balance**: the property, implied by invariant 1, that all leaves share a single depth, hence height = Θ(log N) unconditionally.

---

## Worked Examples

This lecture was presented with handwritten diagrams and an interactive demo, so there is no lecture Java code for B-trees. The traces below follow the lecture's own examples exactly. The Java is provided as a faithful rendering of the algorithms described verbally and is marked where it goes beyond what lecture showed.

### Example 1: Why sorted inserts kill a BST (box-and-pointer reasoning)

The baseline BST search, so the "height + 1 comparisons" claim has something concrete attached to it:

```java
private static class BSTNode<K extends Comparable<K>> {
    K key;
    BSTNode<K> left, right;
    BSTNode(K key) { this.key = key; }
}

static <K extends Comparable<K>> boolean contains(BSTNode<K> n, K key) {
    if (n == null) {
        return false;                        // fell off the bottom
    }
    int cmp = key.compareTo(n.key);
    if (cmp == 0) {
        return true;
    } else if (cmp < 0) {
        return contains(n.left, key);        // one step deeper
    } else {
        return contains(n.right, key);
    }
}
```

Reasoning about `add("01-Jan-2019, 10:31:00")`, then `18:51:00`, then `02-Jan-2019, 00:05:00`, ...: the first call creates a node on the heap and the static field `root` points at it. The second key compares greater, so insertion walks to `root.right`, finds `null`, and allocates a node there. The third key is greater than both, so it walks `root -> root.right -> root.right.right`. Every box's `left` pointer stays `null` forever. After N timestamped inserts there is a single chain of N boxes, each pointing right to the next. The recursion in `contains` therefore makes up to N frames, i.e. Θ(N) comparisons. No matter how the comparison logic is written, the *shape* is the problem, which is why the lecture changes the shape rules instead.

### Example 2: Inserting 1, 2, 3, 4, 5, 6, 7 into a 2-3 tree (L = 2, promote the middle)

This is the lecture's exercise, step by step. Compare with the spindly BST from the same input.

```
add 1:      [1]

add 2:      [1 2]

add 3:      [1 2 3]  <- 3 items > L = 2, split, promote the middle item 2

                [2]
             [1]   [3]

add 4:          [2]
             [1]   [3 4]

add 5:          [2]
             [1]   [3 4 5]   <- split, promote middle item 4 into the parent

              [2 4]
           [1] [3] [5]

add 6:        [2 4]
           [1] [3] [5 6]

add 7:        [2 4]
           [1] [3] [5 6 7]   <- split leaf, promote 6

              [2 4 6]
           [1] [3] [5] [7]   <- root now has 3 items > L = 2, split root, promote 4

                   [4]
              [2]       [6]
           [1]  [3]   [5]  [7]
```

Two things to notice, both emphasized in lecture:
- Every intermediate tree satisfies both invariants. For instance `[2 4]` has 2 items and exactly 3 children.
- The only step that increased the height was the final root split, and it pushed *all* leaves from depth 1 to depth 2 at once. All leaves end at depth 2. Compare with the BST on the same input, which has height 6.

Lecture also asks for an order on {1, ..., 7} producing a 2-3 tree of height 1. One answer: **2, 3, 4, 5, 6, 1, 7**, yielding

```
        [3 5]
   [1 2] [4] [6 7]
```

### Example 3: Chain reaction in a 2-3-4 tree (L = 3, promote the left-middle)

Starting tree, then `add(25)` and `add(26)`:

```
                  [13]
       [5]                 [15 17 19]
   [2]   [7]      [14]  [16]   [18]   [20 21]

add 25, 26 into the rightmost leaf:
                  [13]
       [5]                 [15 17 19]
   [2]   [7]      [14]  [16]   [18]   [20 21 25 26]      <- 4 items > L = 3

split, promote left-middle item 21:
                  [13]
       [5]                [15 17 19 21]                  <- now 4 items, also too full
   [2]   [7]     [14]  [16]  [18]  [20]  [25 26]

split, promote left-middle item 17:
                       [13 17]
          [5]           [15]            [19 21]
      [2]   [7]      [14]   [16]    [18]  [20]  [25 26]
```

And the root-split version, straight from the slides:

```
            [13 17 21]
   [5]  [15]  [19]  [22 23 24 25]      <- leaf too full, promote 23

            [13 17 21 23]
   [5]  [15]  [19]  [22]  [24 25]      <- root too full, promote 17

                     [17]
         [13]                [21 23]
      [5]    [15]        [19]  [22]  [24 25]
```

Height went from 1 to 2 for every leaf at the same instant.

### Example 4: A B-tree node in Java, and `contains` (extra context: code is beyond the slides, the algorithm is not)

```java
public class BTree<K extends Comparable<K>> {

    /** Max items per node. L = 3 gives a 2-3-4 tree; L = 2 gives a 2-3 tree. */
    private static final int L = 3;

    private static class Node<K> {
        int numItems;          // how many slots of items[] are live
        K[] items;             // sorted: items[0] < items[1] < ... < items[numItems - 1]
        Node<K>[] children;    // children[i] holds keys between items[i-1] and items[i]

        @SuppressWarnings("unchecked")
        Node() {
            // L + 1 item slots so a node can be *temporarily* over-full before we split it.
            items = (K[]) new Comparable[L + 1];
            children = (Node<K>[]) new Node[L + 2];
        }

        boolean isLeaf() {
            return children[0] == null;
        }
    }

    private Node<K> root;

    public boolean contains(K key) {
        return contains(root, key);
    }

    private boolean contains(Node<K> node, K key) {
        if (node == null) {
            return false;
        }
        int i = 0;
        while (i < node.numItems) {              // the O(L) scan within one node
            int cmp = key.compareTo(node.items[i]);
            if (cmp == 0) {
                return true;
            }
            if (cmp < 0) {
                break;                           // key belongs in children[i]
            }
            i += 1;
        }
        // If node is a leaf, children[i] is null and we correctly report "not found".
        return contains(node.children[i], key);
    }
}
```

Step by step on `contains(18)` against the tree `[13] -> { [5], [15 17] }` with leaves `[2] [7] [14] [16] [18 19]`:

1. At the root, the `while` loop compares 18 to 13. Not equal, not less, so `i` becomes 1 and the loop ends (`numItems == 1`). We recurse into `children[1]`, the "> 13" subtree.
2. At `[15 17]`: compare 18 to 15 (greater, `i = 1`), compare 18 to 17 (greater, `i = 2`), loop ends. Recurse into `children[2]`, the "> 17" subtree.
3. At leaf `[18 19]`: compare 18 to 18, equal, return `true`.

This is exactly the trace the slides walk through by hand. Note the cost accounting the lecture makes: 3 nodes visited (H + 1 = 3), at most L = 3 comparisons each, so O(HL).

Why `i` doubles as both "which item matched" and "which child to descend into": for a node with k items there are k+1 gaps between/around the items, and the loop stops at the index of the gap the key falls into. That is precisely invariant 2 (k items, k+1 children) showing up in the code.

### Example 5: `add` with splitting, in Java (extra context: implementation detail not shown in lecture)

The recursion inserts into a leaf, then, on the way back up, each level handles a child that split by absorbing the promoted item plus a new right child, and splits itself if that pushed it past L.

```java
    /** Returned by a child that split: the item to promote and the new right node. */
    private static class Split<K> {
        K promoted;
        Node<K> right;
    }

    public void add(K key) {
        if (root == null) {
            root = new Node<>();
            root.items[0] = key;
            root.numItems = 1;
            return;
        }
        Split<K> s = add(root, key);
        if (s != null) {                      // the root itself split: tree gets taller by 1
            Node<K> newRoot = new Node<>();
            newRoot.items[0] = s.promoted;
            newRoot.numItems = 1;
            newRoot.children[0] = root;       // old root is the left half
            newRoot.children[1] = s.right;    // new node is the right half
            root = newRoot;
        }
    }

    private Split<K> add(Node<K> node, K key) {
        int i = 0;
        while (i < node.numItems && key.compareTo(node.items[i]) > 0) {
            i += 1;
        }
        if (i < node.numItems && key.compareTo(node.items[i]) == 0) {
            return null;                      // already present, nothing to do
        }
        if (node.isLeaf()) {
            insertItem(node, i, key, null);   // ALWAYS inserted into an existing leaf
        } else {
            Split<K> s = add(node.children[i], key);
            if (s == null) {
                return null;                  // child absorbed it, no chain reaction
            }
            insertItem(node, i, s.promoted, s.right);
        }
        if (node.numItems <= L) {
            return null;
        }
        return splitNode(node);               // over-full: split and promote upward
    }

    /** Insert key at item index i, with rightChild becoming children[i + 1]. */
    private void insertItem(Node<K> node, int i, K key, Node<K> rightChild) {
        for (int j = node.numItems; j > i; j -= 1) {
            node.items[j] = node.items[j - 1];
            node.children[j + 1] = node.children[j];
        }
        node.items[i] = key;
        node.children[i + 1] = rightChild;    // null when node is a leaf
        node.numItems += 1;
    }

    /** Node has L + 1 items. Promote the middle (L=2) / left-middle (L=3) item. */
    private Split<K> splitNode(Node<K> node) {
        int mid = (node.numItems - 1) / 2;    // 3 items -> 1 (middle); 4 items -> 1 (left-middle)
        int rightCount = node.numItems - mid - 1;

        Node<K> right = new Node<>();
        for (int j = 0; j < rightCount; j += 1) {
            right.items[j] = node.items[mid + 1 + j];
            right.children[j] = node.children[mid + 1 + j];
        }
        right.children[rightCount] = node.children[node.numItems];
        right.numItems = rightCount;

        node.numItems = mid;                  // left half keeps items 0..mid-1, children 0..mid

        Split<K> s = new Split<>();
        s.promoted = node.items[mid];
        s.right = right;
        return s;
    }
```

Walking the pointers for the lecture's chain reaction (`add(26)` into the 2-3-4 tree above):

1. The recursion descends root `[13]` -> `[15 17 19]` -> leaf `[20 21 25]`.
2. `insertItem` shifts nothing (26 is largest) and the leaf box becomes `[20 21 25 26]` with `numItems == 4 > L`.
3. `splitNode` computes `mid = (4 - 1) / 2 = 1`, allocates a fresh `right` box holding `[25 26]`, and sets the original box's `numItems` back to 1 so it reads `[20]`. It returns `promoted = 21`.
4. One frame up, `[15 17 19]` calls `insertItem(node, 3, 21, right)`, so it becomes `[15 17 19 21]` with 5 children `[14] [16] [18] [20] [25 26]`. That is 4 items > L, so it splits with `mid = 1`: left keeps `[15]` with children `[14] [16]`, the new right box is `[19 21]` with children `[18] [20] [25 26]`, and 17 is promoted.
5. Back at the root, `insertItem` makes the root `[13 17]` with children `[5] [15] [19 21]`. `numItems == 2 <= L`, so it returns `null` and `add` finishes without creating a new root. Height unchanged.

Notice that `mid = (numItems - 1) / 2` is exactly the lecture's rule: with 3 items it picks index 1 (the middle, the L=2 rule), with 4 items it picks index 1 (the left-middle, the L=3 rule).

### Example 6: Checking the invariants in code (extra context)

Useful because exams ask you to *recognize* invariant violations; this is the same check mechanized.

```java
    /** Returns the height if the subtree is a legal B-tree, or -1 if an invariant fails. */
    private int checkInvariants(Node<K> node) {
        if (node.numItems < 1 || node.numItems > L) {
            return -1;                        // nodes hold between 1 and L items
        }
        if (node.isLeaf()) {
            return 0;
        }
        int childHeight = -1;
        for (int i = 0; i <= node.numItems; i += 1) {
            if (node.children[i] == null) {
                return -1;                    // invariant 2: k items need exactly k + 1 children
            }
            int h = checkInvariants(node.children[i]);
            if (h == -1) {
                return -1;
            }
            if (childHeight == -1) {
                childHeight = h;
            } else if (childHeight != h) {
                return -1;                    // invariant 1: all leaves at the same depth
            }
        }
        if (node.children[node.numItems + 1] != null) {
            return -1;                        // too many children
        }
        return childHeight + 1;
    }
```

Run this on the lecture's impossible tree (root `[4]`, children `[2 3]` and `[5 6 7]`, with `[2 3]` having the single child `[1]`): the node `[2 3]` has `numItems == 2`, so the loop demands `children[0]`, `children[1]`, `children[2]` all be non-null, but only one child exists, so it returns -1. Even if that were patched, the subtree heights under the root would be 1 and 0, tripping the equal-depth check.

---

## Common Pitfalls

1. **Equating Big O with worst case.** They answer different questions. The worst-case BST height is Θ(N); the height is always O(N); the best case is Θ(log N). "Big O" alone says nothing about which input you are discussing.
2. **Saying "a BST has Θ(log N) height".** Not true in general. Only for bushy trees, including trees built by random insertion. Sorted insertion gives Θ(N).
3. **Confusing L with the number of children.** In this course L is the max number of **items**. For L=3 (a 2-3-4 tree) the max number of children is 4. Also remember external sources often define "order" as the child count instead.
4. **Thinking a node must be full.** A B-tree node holds between **1 and L** items. `[5]` is a perfectly legal node in a 2-3-4 tree.
5. **Pushing an item up without splitting.** This is the lecture's explicitly broken idea; it leaves keys on the wrong side of their separator (16 ending up right of 17). The promotion and the split are one atomic operation.
6. **Forgetting the invariant only constrains non-leaf nodes.** Leaves have 0 children regardless of how many items they hold. "k items means k+1 children" applies to internal nodes.
7. **Thinking any split increases the height.** Only a **root** split does, and it increases the depth of every leaf by exactly 1 at once. Leaf and internal splits widen a level and leave the height alone.
8. **Adding new keys as new leaves.** In a B-tree, insertion *always* places the key in an existing leaf. New nodes appear only as the product of a split.
9. **Using the wrong promoted item for the given L.** L=2: middle. L=3: left-middle (the lecture's arbitrary but fixed convention). Splitting for L > 3 was explicitly out of scope, so do not invent a rule for it on an exam without being told one.
10. **Stopping the cascade too early.** After a promotion, you must re-check the parent, and possibly its parent, all the way to the root. Up to H+1 splits can happen in a single `add`.
11. **Off-by-one on comparison counts.** A search touches height + 1 nodes, not height nodes. The slides stress `contains(s)` on a height-4 tree needing 5 comparisons.
12. **Dropping L from the runtime and then forgetting why that is legal.** `contains` is O(HL) = O(L log N); it collapses to O(log N) only because L is a constant fixed in advance, not a function of N.
13. **Thinking deletion was covered.** It was not. Deletion and L > 3 splitting are out of scope for this class.
14. **Claiming a particular B-tree shape for a given key set.** Different insertion orders can give different heights (height 1 vs height 2 for {1..7} in a 2-3 tree). What is guaranteed is bushiness, not a unique shape.

---

## Likely Exam Points

**1. Big O versus worst case (conceptual true/false).**

*Q:* True or false: "The height of a BST containing N nodes is Θ(log N)." And: "The height of a BST containing N nodes is O(N)."

*A:* The first is **false**: it depends on shape, with worst case Θ(N) (spindly) and best case Θ(log N) (bushy), so no single Θ bound applies. The second is **true**: O(N) is a valid upper bound on the height of any BST, and it is tight in the worst case.

**2. Insert a sequence into a 2-3 tree and draw the result.**

*Q:* Insert 1, 2, 3, 4, 5 in that order into a 2-3 tree (L=2, promote the middle). Draw the final tree and state the height.

*A:*
```
        [2 4]
     [1] [3] [5]
```
Height 1. Trace: `[1]`, `[1 2]`, `[1 2 3]` splits into root `[2]` with children `[1] [3]`; then `[3 4]`; then `[3 4 5]` splits, promoting 4 into the root to give `[2 4]` with children `[1] [3] [5]`.

**3. Chain reaction and root splits.**

*Q:* In the 2-3-4 tree below, perform `add(26)`. How many splits occur, and does the height change?
```
                  [13]
       [5]                 [15 17 19]
   [2]   [7]      [14]  [16]   [18]   [20 21 25]
```
*A:* Two splits. The leaf becomes `[20 21 25 26]` and splits (promoting 21), which makes `[15 17 19 21]` over-full, so it splits (promoting 17). The root absorbs 17 and becomes `[13 17]` with 3 children, which is legal, so no root split and the **height does not change**. Final tree:
```
                       [13 17]
          [5]           [15]            [19 21]
      [2]   [7]      [14]   [16]    [18]  [20]  [25 26]
```

**4. Spot the invariant violation.**

*Q:* Why is this not a valid B-tree?
```
            [4]
      [2 3]       [5 6 7]
    [1]
```
*A:* Both invariants fail. (i) Leaves `[1]` and `[5 6 7]` are at different distances from the root (depths 2 and 1), violating "all leaves the same distance from the root". (ii) The non-leaf `[2 3]` has k=2 items but only 1 child; it must have exactly k+1 = 3 children.

**5. Height bounds given N and L.**

*Q:* A B-tree with L=2 holds N=8 items. What are the smallest and largest possible heights? Which end of the slide's range does this illustrate?

*A:* The tallest case has every node holding 1 item and 2 children, giving height ≈ log₂(N) = 3 as an upper bound on the order of growth; concretely, with every node holding a single item a height of 2 already fits only 7 items, so 8 items forces height ≥ 2, and 8 items is near the **worst case** for L=2 (this is exactly the slide's "N: 8, L: 2, H: 2, height grows with log₂ N" example). The shortest case has every node holding L=2 items and 3 children, giving height ≈ log₃(N), which is why the slide's best-case example packs N=26 items into height 2. Overall, height is between ~log_{L+1}(N) and ~log₂(N), i.e. Θ(log N).

**6. Runtime derivation, with and without the constant.**

*Q:* State the worst-case runtime of `contains` and `add` on a B-tree with limit L, first in terms of both L and N, then with L treated as a constant. Justify each factor.

*A:* `contains`: H+1 nodes inspected, up to L items compared per node, so O(HL) = **O(L log N)**, which is **O(log N)** for constant L. `add`: the same O(HL) search cost, plus up to H+1 splits, each costing O(L) work, which is still O(HL) = **O(L log N)** = **O(log N)**. The key step in both is H = Θ(log N), which follows from perfect balance.

**7. Insertion order and bushiness.**

*Q:* Give an insertion order of {1, ..., 7} that produces a 2-3 tree of height 1, and explain why *no* insertion order can produce a spindly 2-3 tree.

*A:* One answer: 2, 3, 4, 5, 6, 1, 7, giving root `[3 5]` with leaves `[1 2] [4] [6 7]`, all at depth 1. No order can be spindly because B-tree insertion never adds a leaf below an existing leaf: height changes only via root splits, which push every leaf down together, so all leaves are always at equal depth and the tree is perfectly balanced. (Height can vary between orders, 1 vs 2 here, but it is always Θ(log N).)

**8. Maximum and minimum item counts (extra context: a standard follow-on exercise the slides' height bounds set up).**

*Q:* What is the maximum number of items a B-tree with limit L and height h can hold? The minimum?

*A:* Maximum when every node is full: L items at each of (L+1)^d nodes on level d, summed over d = 0..h, which telescopes to (L+1)^{h+1} - 1. For L=2, h=2 this gives 3³ - 1 = 26, matching the slide's best-case example. Minimum when every node holds 1 item (2 children): 2^{h+1} - 1. For L=2, h=2 that is 7, which is why 8 items can be squeezed into height 2 only in a near-worst-case shape.

**9. Terminology translation.**

*Q:* A "2-3-4 tree" corresponds to what value of L, and how many items and children can one node hold?

*A:* L = 3. A node holds 1 to 3 items and, if it is not a leaf, 2, 3, or 4 children (the name lists the legal child counts). It is also called a 2-4 tree. Similarly, a 2-3 tree is L = 2: 1 to 2 items, 2 or 3 children.

---

## Summary

- **depth** = distance from root; **height** = max depth; **average depth** = mean depth. Height + 1 gives worst-case search comparisons; average depth + 1 gives average-case.
- BSTs have **best case height Θ(log N)** and **worst case height Θ(N)**. **Big O is not the same thing as worst case.**
- Random inserts are kind to BSTs: expected average depth ~2 ln N, expected height ~4.311 ln N, both Θ(log N). Random trees are bushy, not spindly. Deletions mixed in keep Θ(log N) if you randomize predecessor/successor.
- But we cannot force random order: data arriving over time (timestamps) is sorted, which produces the Θ(N) spindly worst case.
- **Key idea:** never add leaves at the bottom. Insert into existing leaves ("overstuffing"), so leaf depths, and hence the height, can never drift.
- Pure overstuffing makes nodes unboundedly juicy, so cap nodes at **L items**. Over-full nodes **split**: one item is promoted to the parent and the node divides into a left and a right child. Promoting without splitting is wrong (it misorders keys).
- Promotion rule in lecture: **L=2 promote the middle**, **L=3 promote the left-middle**. L > 3 is out of scope.
- Splits can **chain-react** upward. A leaf or internal split does not change the height; a **root split** adds a new 2-child root and increases every leaf's depth by exactly 1.
- **Invariants:** (1) all leaves are the same distance from the root; (2) a non-leaf with k items has exactly k+1 children. Together these force **perfect balance** and bushiness, for any insertion order.
- Real name: **B-tree**. L=2 is a **2-3 tree**; L=3 is a **2-3-4 tree** (2-4 tree), where the name lists legal child counts. Small L is used as a simple balanced BST; very large L (thousands) is used by databases and filesystems.
- **Height** is between ~log_{L+1}(N) and ~log₂(N), so H = Θ(log N). **`contains`** and **`add`** are both O(HL) = O(L log N) = **O(log N)** for constant L; `add` performs at most H+1 splits.
- `contains` works almost exactly like a normal BST search, with an extra O(L) scan inside each node.
- **Not covered:** deletion, and splitting for L > 3. B-trees are more complex than BSTs but handle *any* insertion order efficiently.
- Demos used in lecture: random BST simulation https://joshh.ug/61b/bst.html and the B-tree visualizer https://tinyurl.com/balanceYD (where "max-degree" means max *children*).
