<!-- Fri, Oct 09, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 19: Red-Black Trees

## Overview

This lecture closes a three-lecture arc on search trees. Last lecture gave us B-trees (and in particular 2-3 trees, the B-tree with `L = 2`), which are perfectly balanced but miserable to implement: multiple node types, interconversion between 2-nodes and 3-nodes, and splitting while walking back up the tree. So we put B-trees on a shelf and go back to an ordinary binary search tree, armed with one new operation: **tree rotation**, which rearranges a BST without changing which keys it contains or the search tree property. Rotation alone lets you reshape any BST into any other BST on the same keys, but it does not tell you *which* rotations to do. The beautiful idea of the lecture is to let 2-3 trees be our guide: we color some BST links "red" so that a red link glues two BST nodes together into one conceptual 3-node, giving an exact one-to-one correspondence (an **isometry**) between a BST and a 2-3 tree. A BST that maintains this correspondence, with the extra convention that red glue links always lean left, is a **Left-Leaning Red-Black tree (LLRB)**. Because 2-3 trees have logarithmic height and an LLRB is never more than about twice as tall as its corresponding 2-3 tree, LLRBs are balanced, with `O(log N)` `contains` and `insert`. Insertion is implemented as "insert into a BST as usual with a red link, then apply zero or more rotations and color flips to restore the correspondence," and the lecture derives those three fixup rules by repeatedly asking "What Would 2-3 Tree Do?"

---

## Key Concepts

### 1. Why not just implement 2-3 trees?

2-3 trees and 2-3-4 trees are conceptually gorgeous and perfectly balanced, but for small `L` they are a real pain in code and suffer performance problems. The lecture lists three specific sources of pain:

- **Maintaining different node types** (a 2-node and a 3-node hold different numbers of items and children).
- **Interconversion of nodes** between 2-nodes and 3-nodes (you need code that says "you are a 3-node now").
- **Walking up the tree to split nodes** after an overfull node appears.

The slides show "fantasy 2-3 code" (via Kevin Wayne) to make the point about how awkward this gets:

```java
public void put(Key key, Value val) {
   Node x = root;
   while (x.getTheCorrectChildKey(key) != null) {
      x = x.getTheCorrectChildKey();
      if (x.is4Node()) { x.split(); }
   }
   if (x.is2Node()) { x.make3Node(key, val); }
   if (x.is3Node()) { x.make4Node(key, val); }
}
```

Quoting Knuth on the slide: "Beautiful algorithms are, unfortunately, not always the most useful." The lecture also notes the alternative we are about to build is faster in practice, not merely easier.

### 2. How many BSTs are there, and can we move between them?

For the keys 1, 2, 3 there are exactly **five** valid BSTs, and which one you get depends only on insertion order:

```
1        1          2        3          3
 \        \        / \      /          /
  2        3      1   3    1          2
   \      /                 \        /
    3    2                   2      1
```

More generally, `N` keys admit `Catalan(N)` different BSTs, a number that grows very quickly. The key structural fact: **given any BST, you can reach any other configuration on the same keys using rotations**. This is unlike a Rubik's cube or a chess position, where some states are unreachable. The slides cite "Rotation Distance, Triangulations, and Hyperbolic Geometry" for the bound of `2n - 6` rotations to move between any two configurations (the lecture attributes the paper to Tarjan and co-authors).

### 3. Tree rotation: definition and intuition

> `rotateLeft(G)`: Let `x` be the right child of `G`. Make `G` the new left child of `x`.
>
> `rotateRight(P)`: Let `x` be the left child of `P`. Make `P` the new right child of `x`.

Both preserve the search tree property; there is **no change to the semantics of the tree** (same key set, same ordering invariant).

Three complementary ways the lecture builds intuition:

- **"Who is the new boss?"** If you `rotateLeft(x)`, then `x` goes down and to the left, and `x`'s **right child** becomes its new parent. Symmetrically, `rotateRight(x)` sends `x` down and to the right, and `x`'s **left child** becomes its new parent. Josh's alternate name for `rotateLeft(G)` is "promote the right child of `G`."
- **The orphaned subtree.** Walking through `rotateLeft(G)` where `G` has left child `C` and right child `P`, and `P` has children `k` and `r`: after you hoist `P` above `G`, node `P` momentarily has *two* left children (`G` and `k`), which "doesn't make sense." The fix is forced: `k` becomes `G`'s new right child. This is legal precisely because `k`'s subtree contains exactly the keys greater than `G` and less than `P` (you reached it by going right from `G`, then left from `P`), so it belongs to the right of `G`.
- **Merge then split.** Think of `rotateLeft(G)` as temporarily merging `G` and `P` into one node `GP`, then sending `G` down and to the left. `rotateRight(P)` merges `GP` and sends `P` down and to the right.

Before and after `rotateLeft(G)`:

```
        G                              P
      /   \                          /   \
     C     P        ===>            G     r
    / \   / \                      / \
   A   k j   l                    C   k
                                      / \
                                     j   l
```

In this particular example `rotateLeft(G)` **increased** the height of the tree, and the mirror-image `rotateRight(P)` **decreased** it. Rotation can shorten or lengthen a tree; it is a reshaping tool, not automatically a balancing tool.

**Some rotations are undefined.** If there is no appropriate child to take your place, the operation is invalid. Example from the slides: with `1` as the root whose only child is a right child `4`, `rotateRight(1)` is an **INVALID OPERATION** because there is no left child to take over.

### 4. Rotation as a (clumsy) balancing tool

The lecture's balancing demo on seven keys:

```
Start:                After rotateLeft(9):      After rotateLeft(6):
      6                      6                         8
    /   \                  /   \                      / \
   1     8                1     8                    6   13
    \     \                \     \                  /   /  \
     4     9                4     13                1   9   17
            \                    /  \                \
             13                 9    17               4
               \
                17

After rotateLeft(1):      After rotateRight(6):
      8                        8
    /   \                    /   \
   6     13                 4     13
  /     /  \               / \   /  \
 4     9   17             1   6 9   17
/
1
```

So `rotateLeft(9)`, `rotateLeft(6)`, `rotateLeft(1)`, `rotateRight(6)` takes a spindly 7-node BST to a perfectly balanced one. Consistent with the `2n - 6` bound, four rotations sufficed here.

The takeaway from this section: **rotation allows balancing of a BST in `O(N)` moves, but paying `O(N)` to occasionally rebalance is not ideal.** We want balance maintained incrementally, as part of insertion. For that we need a rule that tells us which rotations to perform, and that rule comes from 2-3 trees.

### 5. The big idea: maintain a one-to-one correspondence with a 2-3 tree

Wild idea: perform rotations such that our BST always maintains a **1-to-1 correspondence** with some 2-3 tree. Since 2-3 trees are always balanced, our BST will also be balanced. The lecture calls this correspondence an **isometry** (the relevant "distance" being root-to-leaf path lengths).

The obstacle: 2-3 trees are not binary. A 3-node has two items and three children. How can a binary node represent that?

**Possibility 1: dummy "glue" nodes.** Represent the 3-node `df` as an empty glue node whose left child is `d` and right child is `f`, forming a "supernode." The three children of the 3-node hang off `d.left`, `f.left`, and `f.right`. The result is **inelegant**: there is a wasted link (one of `d`'s links is never used), nodes that contain no key, and ugly code.

**Possibility 2: glue *links*.** Drop the glue node. Represent a 3-node `df` as two ordinary BST nodes with `f` on top and `d` as `f`'s **left** child, connected by a link we agree to call **red**:

```
2-3 node:   df        becomes:      f
                                   /
                                  d       (link f -> d is red)
```

The three children of the 3-node then attach as `d.left`, `d.right`, and `f.right`. This is the idea used in practice, for example in `java.util.TreeSet`. Marking glue links red is purely "for convenience."

A full example from the slides:

```
2-3 tree:               m                  LLRB:            m
                      /   \                                /   \
                    df     o                              f     o
                   / | \  / \                            / \   / \
                  b  e  g n   p                         d   g n   p
                                                       /
                                                      b   (and b's parent link... see below)
```
Reconstructing the slide's LLRB exactly: root `m`; `m.left = f` (black) with `f.left = d` (**red**), `d.left = b`, `d.right = e`, `f.right = g`; `m.right = o` with children `n` and `p`.

### 6. Left-Leaning Red-Black BSTs (LLRBs)

A BST with left glue links that represents a 2-3 tree is called a **Left-Leaning Red Black Binary Search Tree (LLRB)**. Three points the lecture hammers on:

- **LLRBs are normal BSTs.** Nothing about the data structure changes.
- There is a **1-1 correspondence** between an LLRB and an equivalent 2-3 tree.
- **The red is a convenient fiction.** Red links don't "do" anything special; searching an LLRB is *exactly* BST search, ignoring colors. Colors exist only as bookkeeping that tells us which rotations to perform when inserting.

The "left-leaning" restriction (red links only ever go left) comes from Sedgewick and is stricter than what Java does. Josh's stated reason: it makes the lecture shorter and you get most of the conceptual benefit without considering so many cases.

**Worked conversion from the lecture.** Convert this 2-3 tree to its LLRB:

```
2-3 tree:        uw
               /  |  \
            as    v    xy
```

The 3-node `uw` becomes `w` with red left child `u`. Node `uw`'s three children attach as `u.left = ` (the node for `as`), `u.right = v`, `w.right = ` (the node for `xy`). Node `as` becomes `s` with red left child `a`; node `xy` becomes `y` with red left child `x`:

```
LLRB:            w
               /   \
              u     y
             / \   /
            s   v x
           /
          a

red links: w->u, s->a, y->x        black links: u->s, u->v, w->y
```

### 7. LLRB properties (the invariants)

Two handy properties, each a direct translation of a 2-3 tree rule:

1. **No node has two red links** touching it, because that would be analogous to a 4-node, which is disallowed in a 2-3 tree (`L = 2`). (During insertion we allow this *temporarily*; see rules below.)
2. **Every path from the root to a null link has the same number of black links**, because every leaf of a 2-3 tree is at the same depth. LLRBs are therefore balanced. This is sometimes called **perfect black balance**.

**Subtle but important (from the slides' "Extra Slide: Root-to-Leaf vs. Root-to-Null").** An older version of this lecture incorrectly stated the invariant as "the number of black links to any **leaf** must be the same." The correct invariant counts black links to any **null link**. Counterexample given: root `G` with a red left child `B`, `B.left = A` (black), `G.right = X` (black), and `B.right = null`.

```
      G
     / \
  (red) \
    B    X
   /
  A        <-- Invalid LLRB, but passes the erroneous "root-to-leaf" test
```
Root-to-leaf black counts: `G->B` (red, 0) then `B->A` (1) gives 1; `G->X` gives 1. Equal, so the bad invariant is satisfied. But root-to-**null**: the path through `B.right` has only 1 black link, while the path through `A` has 2 and the path through `X` has 2. The corresponding "2-3 tree" would be a 3-node `BG` with a **missing child**, which is not a legal 2-3 tree.

### 8. LLRB height, and why it is logarithmic

Each 3-node of the 2-3 tree becomes two nodes (one extra red link) in the LLRB. In the lecture's height-3 example tree, the longest LLRB path runs `L -> P -> U -> S -> R -> Q`: three black links plus two red links, so **height 5**.

```
2-3 tree of height 3:
                       L
            /                    \
          DE                      P
       /   |   \               /     \
      B    G    J             N       SU
     /\   /\   /\            / \     / | \
    A  C F  H I  K          M   O  QR  T  VW

Longest LLRB path (3 black + 2 red = 5):  L -> P -> U -(red)- S -> R -(red)- Q
```

In general, if the 2-3 tree has height `H`, the corresponding LLRB has height at most `H` (black) `+ H + 1` (red) `= 2H + 1`. The `H + 1` comes from a root-to-leaf path touching `H + 1` nodes, each of which might be a 3-node contributing one red link.

**Conclusion:** 2-3 trees have `Θ(log N)` height, and `2 log N + 1 = Θ(log N)`, so **LLRBs have logarithmic height too**. A factor of two off perfection is "pretty good." This answers the natural student question raised in lecture ("why do we care about the 2-3 tree height if the LLRB is taller?"): the order of growth is the same.

### 9. Where do LLRBs come from? Maintaining the isometry with rotations

It would make no sense to build a 2-3 tree and then convert, since 2-3 trees are the thing we were trying to avoid building. Instead:

> **LLRB insert = insert as usual into a BST, then use zero or more rotations (and color flips) to maintain the 1-1 mapping.**

"Zero or more" because sometimes nothing is broken. The design method is: pretend you are a 2-3 tree, ask what the 2-3 tree would do (WW23TD), then find the LLRB operation that produces the LLRB representation of that 2-3 tree.

**Design Task #1: insertion color.** Red or black for a new link? **Red.** In a 2-3 tree, new values are *always* added into an existing leaf node first; you never create a new node from thin air. Adding a red link is the LLRB equivalent of "stuffing" another item into an existing node.

```
2-3:   S     --add(E)-->   ES
LLRB:  S     --add(E)-->   S
                          /
                         E      (red)
```

**Design Task #2: insertion on the right (right-leaning violation).** Start with `B` as root, `A` left, `E` right (no red links, so LLRB = 2-3 tree). Insert `S`. BST insertion puts `S` as `E`'s right child with a red link, but **right-leaning red links aren't allowed**. In 2-3 world we created node `ES`, whose LLRB representation must be `S` on top with red left child `E`. The rotation that gets there is **`rotateLeft(E)`**:

```
     B               B                      B
    / \             / \                    / \
   A   E   add(S)  A   E     rotateLeft(E) A   S
                        \                     /
                         S (red)             E (red)

2-3 world:   B(A, E)  --add(S)-->  B(A, ES)
```

**New rule: representing temporary 4-nodes.** 2-3 trees do not allow 4-nodes, but during insertion we transiently have them before splitting. In LLRB world we represent a temporary 4-node as a BST node with **two red links, one left and one right**:

```
2-3:  B(A, ES)  --add(Z)-->  B(A, ESZ)       (temporarily violates "no 4 nodes")

LLRB:      B                       B
          / \                     / \
         A   S         add(Z)    A   S
            /                       / \
           E (red)                 E   Z      (both red; temporarily violates "no red right links")
```

**Design Task #3: double insertion on the left (incorrect 4-node shape).** Start with `B` root, `A` left, `Z` right, `Z.left = S` (red), i.e. 2-3 tree `B(A, SZ)`. Insert `E`: BST insertion makes `E` a red left child of `S`, giving **two consecutive left red links**. The 2-3 tree is now the temporary 4-node `ESZ`, whose required LLRB shape is `S` with red children `E` and `Z`. The fix is **`rotateRight(Z)`**:

```
     B                 B                           B
    / \               / \                         / \
   A   Z   add(E)    A   Z      rotateRight(Z)    A   S
      /                 /                            / \
     S (red)           S (red)                      E   Z   (both red)
                      /
                     E (red)
```

**Design Task #4: splitting temporary 4-nodes (color flip).** Now we have `G` root, `B` as `G`'s left child with **two red children** `A` and `C`, and `G.right = X`. In 2-3 world the root node is the 4-node `ABC` with sibling `X`, and 2-3 trees respond by **splitting**: `B` moves up into the parent, leaving `A` and `C` as separate children, giving root `BG` with children `A`, `C`, `X`. What LLRB operation produces that? **Not a rotation.** The LLRB shape is already correct: all we need is for `B` to be glued to `G` (red) and for `A` and `C` to be real children (black). So we **flip the colors of all three edges touching `B`**:

```
        G                         G
       / \        flip(B)        / \
      B   X       ======>       B   X        B->G link now RED,
     / \                       / \           B->A and B->C now BLACK
    A   C   (both red)        A   C

2-3 world:   G(ABC, X)  --split(A/B/C)-->  BG(A, C, X)
```

"BST, the magic was inside you all along": a color flip does **not** change the BST structure or shape at all.

### 10. The three rules, and cascading

> Congratulations, you just invented the red-black BST.
>
> - When inserting: **use a red link**.
> - If there is a **right leaning "3-node"** (red right link), you have a **Left Leaning Violation**: `rotateLeft` the appropriate node.
> - If there are **two consecutive left red links**, you have an **Incorrect 4 Node Violation**: `rotateRight` the appropriate node.
> - If a node has **two red children**, you have a **Temporary 4 Node**: **color flip** that node to emulate the split operation.

**One last detail: cascading operations.** A rotation or flip can create a *new* violation higher up that must also be fixed. The lecture's example: start from `B` root, `A` left, `S` right, `S.left = E` (red), i.e. 2-3 tree `B(A, ES)`. Insert `Z`:

```
     B                    B                          B                          S
    / \      add(Z)      / \        flip(S)         / \      rotateLeft(B)     / \
   A   S     =====>     A   S       =====>         A   S      =====>          B   Z
      /                   / \                        / \                     / \
     E (red)             E   Z (both red)           E   Z (now black)       A   E
                                                   B->S link now RED              (B->S red, so S is
                                                   (right leaning!)                 the top of 3-node BS)

2-3 world:  B(A, ES) --add(Z)--> B(A, ESZ) --split(E/S/Z)--> BS(A, E, Z)
```

The flip fixed the 4-node but created a **right-leaning red link** `B -> S`, so we cascade into `rotateLeft(B)`. Note that in the final tree the red link is `S -> B`, matching the 3-node `BS`.

### 11. Runtime (marked Extra in the slides)

The analysis is simple if you trust the 2-3 tree result:

- LLRB height is `O(log N)`.
- `contains` is trivially `O(log N)` (plain BST search).
- `insert` is `O(log N)`: `O(log N)` to walk down and add the node, plus `O(log N)` rotation and color flip operations on the way back up.

**LLRB delete is not covered** in CS 61B. "Not too terrible really, but it's just not interesting enough to cover."

### 12. Search tree summary

| Structure | Balanced? | Implementation |
|---|---|---|
| Binary search tree | No, subject to imbalance (`Θ(N)` worst case) | Simple |
| 2-3 / B-trees | Yes, perfectly | Painful and relatively slow |
| LLRB | Yes (within ~2x of a 2-3 tree) | Insert is simple, delete is hard |

- LLRBs work by maintaining a mathematical bijection with a 2-3 tree.
- **Java's `TreeMap` is a red-black tree, but not left leaning.** It maintains a correspondence with a **2-3-4** tree, which is *not* a 1-1 correspondence, and it allows glue links on either side. More complex to implement, but significantly faster.
- Beyond this course: AVL trees, WAVL trees (described on the slide as fastest), splay trees, treaps, and hundreds of others; skip lists ("linked lists with express lanes"); and hashing, the most common alternative entirely, coming in a later lecture.

---

## Definitions

- **B-tree:** a search tree with a hard limit `L` on the number of items per node; perfectly balanced by construction.
- **2-3 tree:** a B-tree with `L = 2`. Every internal node is a 2-node or a 3-node; all leaves are at the same depth.
- **2-node:** a node with 1 item and 2 children. **3-node:** a node with 2 items and 3 children. **4-node:** a node with 3 items and 4 children (illegal in a 2-3 tree except transiently during insertion).
- **Catalan(N):** the number of distinct binary search trees on `N` keys. For `N = 3` it is 5.
- **Rotation distance fact:** any BST configuration can be transformed into any other in at most `2n - 6` rotations (cited: "Rotation Distance, Triangulations, and Hyperbolic Geometry").
- **`rotateLeft(G)`:** let `x` be the right child of `G`; make `G` the new left child of `x`. `x`'s old left subtree becomes `G`'s new right subtree. Preserves the search tree property. Invalid if `G` has no right child.
- **`rotateRight(P)`:** let `x` be the left child of `P`; make `P` the new right child of `x`. `x`'s old right subtree becomes `P`'s new left subtree. Preserves the search tree property. Invalid if `P` has no left child.
- **Glue node:** a dummy, item-free BST node used to represent a 3-node (Possibility 1, rejected as inelegant because it wastes a link).
- **Glue link / red link:** a BST link designating that its two endpoint nodes are really one 3-node of the corresponding 2-3 tree (Possibility 2, adopted).
- **Red-Black BST:** a BST whose links are colored, with red links serving as glue links representing 3-nodes.
- **Left-Leaning Red Black BST (LLRB):** a red-black BST in which glue (red) links may only point left, in 1-1 correspondence with a 2-3 tree.
- **Isometry / 1-1 correspondence:** the exact structural bijection between an LLRB and a 2-3 tree; every LLRB maps to exactly one 2-3 tree and vice versa.
- **Height / "tallness":** the number of **links** on the longest path from root to leaf.
- **Perfect black balance:** every path from the root to a **null** link contains the same number of black links.
- **Left Leaning Violation:** a red link pointing right. Fix with `rotateLeft`.
- **Incorrect 4 Node Violation:** two consecutive left red links. Fix with `rotateRight`.
- **Temporary 4 Node:** a node with two red children (one left, one right). This is the sanctioned transient representation of a 4-node; fix with a **color flip**, which emulates a 2-3 tree `split`.
- **Color flip:** flip the colors of all three links touching a node (the link into it and its two child links). Changes no pointers and no shape.
- **Cascading:** a rotation or color flip can introduce a new violation at an ancestor, which must then be fixed in turn.

---

## Worked Examples

### Example A: `rotateLeft` and `rotateRight` in Java, with box-and-pointer reasoning

*(extra context: the lecture presents rotation with diagrams only, and marks implementation as "Extra." This is the standard Sedgewick-style implementation consistent with the lecture's definitions. The convention that a node stores the color of the link pointing **into** it is the standard one and is implicit in the slides' pictures.)*

```java
public class LLRB<K extends Comparable<K>, V> {
    private static final boolean RED   = true;
    private static final boolean BLACK = false;

    private class Node {
        K key;
        V val;
        Node left, right;
        boolean color;   // color of the link from this node's PARENT into this node

        Node(K key, V val, boolean color) {
            this.key = key;
            this.val = val;
            this.color = color;
        }
    }

    private Node root;

    /** A null link is black. This matters for the "root to null" invariant. */
    private boolean isRed(Node x) {
        if (x == null) { return false; }
        return x.color == RED;
    }

    /** rotateLeft(h): h's right child becomes h's new parent; h goes down-left. */
    private Node rotateLeft(Node h) {
        Node x = h.right;        // x is the "new boss"
        h.right = x.left;        // x's left subtree holds keys between h and x: belongs right of h
        x.left = h;              // h becomes x's left child
        x.color = h.color;       // x inherits the link color that pointed into h
        h.color = RED;           // the link x -> h becomes the glue link
        return x;                // caller must re-attach: the subtree root changed
    }

    /** rotateRight(h): h's left child becomes h's new parent; h goes down-right. */
    private Node rotateRight(Node h) {
        Node x = h.left;
        h.left = x.right;        // keys between x and h: belongs left of h
        x.right = h;
        x.color = h.color;
        h.color = RED;
        return x;
    }

    /** Emulates a 2-3 tree split: no pointers change, only colors. */
    private void flipColors(Node h) {
        h.color       = RED;
        h.left.color  = BLACK;
        h.right.color = BLACK;
    }
}
```

**Box-and-pointer reasoning for `rotateLeft(h)`.** Think of three boxes `h`, `x = h.right`, and the subtree `k = x.left`, each just an address. No `new` objects are created, and no keys move between boxes; only three arrows are rewritten:

1. The arrow `h.right` is redirected from `x` to `k`. `h` now has `k` hanging on its right.
2. The arrow `x.left` is redirected from `k` to `h`. `h` is now below `x`.
3. The caller reattaches whatever pointed at `h` (a parent's `left`/`right` field, or `root`) to point at `x` instead. This is why both methods **return the new subtree root** and why the recursive `put` below writes `h.left = put(h.left, ...)`: the subtree root may change under you.

Correctness of step 1 is the ordering argument from lecture: every key in `k` was reached by going *right* from `h` and then *left* from `x`, so every key in `k` satisfies `h.key < key < x.key`. Hanging `k` on `h.right` keeps that true.

Note that `rotateLeft(h)` dereferences `h.right` and `rotateRight(h)` dereferences `h.left`. If that child is `null`, the rotation is the **invalid operation** from the slides (`rotateRight(1)` when `1` has no left child), and this code would throw a `NullPointerException`.

### Example B: Balancing by hand, the lecture's 3-node example

Give a sequence of rotations that balances the tree on the left:

```
   1                                      2
    \                      goal:         / \
     3                                  1   3
    /
   2
```

Step 1, `rotateRight(3)`: `3`'s left child is `2`, so `2` is promoted and `3` becomes `2`'s right child. `2`'s old right child is `null`, which becomes `3`'s left child.

```
   1
    \
     2
      \
       3
```

Step 2, `rotateLeft(1)`: `1`'s right child is `2`, so `2` is promoted and `1` becomes `2`'s left child. `2`'s old left child (`null`) becomes `1`'s right child.

```
     2
    / \
   1   3
```

Balanced. The slides note there are other correct answers as well.

### Example C: Which of these four are valid LLRBs? (the lecture's attendance question)

Dashed/red links are marked explicitly. All four have `G` and `X` with the same silhouette.

```
(1)  G          (2)  G          (3)  G          (4)  G
    / \             / \             / \             / \
   C   X           B   X           B   X        (red) \
  /                / \             / \             B   X
 B           (red)A   C           A   C           / \
 |                                                A   C
 (reds: G-C? no: G-C black, C-B red, B-A red)   (A, C black)
/
A
```

Translating each to its corresponding 2-3 tree:

| Tree | Corresponding 2-3 tree | Verdict |
|---|---|---|
| (1) `C`, `B`, `A` chained left with `C-B` and `B-A` red | root `G` with children `ABC` and `X` | **Invalid: has a 4-node.** `B` has two red links touching it. |
| (2) `B` black child of `G`, `A` red child of `B`, `C` black child of `B` | root `G` with children (3-node `AB` having only one child, `C`) and `X` | **Invalid: not balanced / not black balanced.** (Count black links to nulls: via `B.right`-ish path you get a different count than via `X`.) |
| (3) No red links at all | identical in shape to the BST: `G` with children `B` and `X`; `B` with children `A`, `C` | **Invalid: not balanced.** `X` is one hop from the root, `A` and `C` are two. |
| (4) `G-B` red, `A` and `C` black children of `B`, `X` black child of `G` | root is 3-node `BG` with children `A`, `C`, `X` | **Valid.** |

**Answer: exactly 1 of the 4.** Note the trap flagged in lecture: tree (2) was the most popular wrong answer. You *can* draw a 2-3-ish picture for it, but you could never generate that tree by 2-3 insertion, because it is not balanced.

Black-link check for (4): `G -> B` (red, 0) `-> A` (1) `-> null` (1) = 2; `G -> B` (0) `-> C` (1) `-> null` (1) = 2; `G -> X` (1) `-> null` (1) = 2. All equal.

### Example D: LLRB insertion in Java, with the three fixup rules

*(extra context: real Java code for the fixups; the lecture derived the rules but did not write this method.)*

```java
public void put(K key, V val) {
    root = put(root, key, val);
    root.color = BLACK;          // the root is never a glue link
}

private Node put(Node h, K key, V val) {
    if (h == null) {
        return new Node(key, val, RED);          // Design Task #1: new links are RED
    }

    int cmp = key.compareTo(h.key);
    if      (cmp < 0) { h.left  = put(h.left,  key, val); }
    else if (cmp > 0) { h.right = put(h.right, key, val); }
    else              { h.val = val; }

    // ----- restore the 1-1 correspondence on the way back up -----

    // Rule 1: Left Leaning Violation (red link pointing right)
    if (isRed(h.right) && !isRed(h.left)) { h = rotateLeft(h); }

    // Rule 2: Incorrect 4 Node Violation (two consecutive left reds)
    if (isRed(h.left) && isRed(h.left.left)) { h = rotateRight(h); }

    // Rule 3: Temporary 4 Node (two red children) -> emulate split
    if (isRed(h.left) && isRed(h.right)) { flipColors(h); }

    return h;
}
```

Why this matches the lecture exactly:

- The `put` recursion is plain BST insertion. The new node is created with a **red** link (Design Task #1).
- The three `if`s run **after** the recursive call, so they execute from the insertion point back up to the root. That is precisely the **cascading** behavior: a `flipColors` at a deep node can leave a red right link at its parent, and the parent's own `if` block will `rotateLeft` it, exactly as in the `B/A/S/E + add(Z)` example.
- The guard `!isRed(h.left)` in Rule 1 is what lets a *legitimate* temporary 4-node (red left **and** red right) fall through to Rule 3 instead of being rotated.
- Rule 2's second condition is `h.left.left`, matching "two consecutive left links."

### Example E: Inserting 7, 6, 5, 4, 3, 2, 1 into an empty LLRB (the optional exercise)

The slides promise you end up with a perfectly balanced BST. Tracing it with the three rules:

| Insert | Immediately after BST insert | Violation | Fix | Resulting LLRB (red links marked) |
|---|---|---|---|---|
| 7 | `7` | none (root recolored black) | | `7` |
| 6 | `6` red left of `7` | none | | `7`, red left `6` |
| 5 | `5` red left of `6` | two consecutive left reds (`7-6`, `6-5`) | `rotateRight(7)` then `flipColors(6)` | `6` black root, black children `5`, `7` |
| 4 | `4` red left of `5` | none | | `6`; `5` with red left `4`; `7` |
| 3 | `3` red left of `4` | two consecutive left reds (`5-4`, `4-3`) | `rotateRight(5)` then `flipColors(4)` | `6` root; red left `4` with black children `3`, `5`; `7` |
| 2 | `2` red left of `3` | none (`4-3` is black, so no two reds in a row) | | `6`; red `4`(children `3`(red left `2`), `5`); `7` |
| 1 | `1` red left of `2` | two left reds (`3-2`, `2-1`) | `rotateRight(3)`, `flipColors(2)`; this makes `4-2` red while `6-4` is already red, so two left reds again: `rotateRight(6)`, `flipColors(4)` | **`4` root; `2`(children `1`,`3`); `6`(children `5`,`7`)** |

Final tree, exactly as the slide shows, with no red links at all (so the LLRB equals its own 2-3 tree, all 2-nodes):

```
        4
      /   \
     2     6
    / \   / \
   1   3 5   7
```

### Example F: The "Rule derivation" worksheet at the end of the slides

**Rule 1 derivation.** In 2-3 world, starting from root `F` with children `B` and `EZ`, adding `C` gives root `F` with children `BC` and `EZ`. In LLRB world, starting from `F` with left child `B` and right child `Z` (with red left child `E`), inserting `C` with a red link makes `C` the **right** child of `B`. That is a right-leaning red link, so perform **`rotateLeft(B)`**:

```
before fix:            after rotateLeft(B):
      F                      F
     / \                    / \
    B   Z                  C   Z
     \   \                /     \
      C   E              B       E     (B-C red, E-Z red; F-C and F-Z black)
   (C red, E red)
```

**Rule 2 derivation.** Now insert `A` into that tree. `A` goes as a red left child of `B`, and `C-B` is already red, so we have two consecutive left reds under `F`. **`rotateRight(C)`** gives:

```
      F                          F
     / \                        / \
    C   Z      rotateRight(C)  B   Z
   /     \       ======>       / \   \
  B       E                   A   C   E     (B-A and B-C both red: correct temp 4-node shape)
 /
A
```

**Rule 3 derivation.** In 2-3 world this is the 4-node `ABC`, and we would call `split(B)`. In LLRB world we call **`colorFlip(B)`**: `B-A` and `B-C` become black, and `F-B` becomes red. Shape unchanged:

```
      F                                      Equivalent 2-3 tree:
     / \
 (red) \                                             BF
    B   Z                                          / | \
   / \   \                                        A  C  EZ
  A   C   E   (E-Z still red)
  (A, C now black)
```
All leaves of that 2-3 tree (`A`, `C`, `EZ`) sit one link below the root `BF`: balanced, with no node holding more than 2 items. The isometry is restored.

---

## Common Pitfalls

1. **Getting the direction of rotation backwards.** `rotateLeft(x)` does **not** promote `x`; it demotes `x` down-and-left and promotes `x`'s **right** child. Always ask "who is the new boss?" The answer is the child on the *opposite* side from the direction named.
2. **Forgetting that rotation can increase height.** The lecture's own first example (`rotateLeft(G)`) made the tree taller. Rotation is shape-neutral with respect to correctness, not with respect to balance.
3. **Attempting an invalid rotation.** `rotateRight(x)` requires `x.left != null`; `rotateLeft(x)` requires `x.right != null`.
4. **Forgetting to re-attach the rotated subtree.** The rotated subtree has a new root, so a parent pointer (or `root`) must be updated. In code, this is why `rotateLeft`/`rotateRight` return a `Node` and why callers write `h.left = put(h.left, ...)`.
5. **Thinking the colors affect search.** They do not. `get`/`contains` on an LLRB is identical to BST search. "The red is just a convenient fiction."
6. **Using root-to-leaf instead of root-to-null for black balance.** The slides explicitly flag this as a past error in the lecture itself. Count black links to **null** links, which is what catches a 3-node with a missing middle child.
7. **Checking only one invariant.** A tree can have no 4-nodes and still be invalid because it is not black balanced (trees (2) and (3) in Example C), or be perfectly balanced in silhouette while harboring a 4-node (tree (1)). Check both.
8. **Allowing a right-leaning red link to persist.** In *this* course's LLRBs, right-leaning red links are legal only as half of a temporary 4-node, and must be cleaned up before the insertion finishes. (Java's `TreeMap` does allow red links on either side, but that is a different structure.)
9. **Thinking a color flip rearranges pointers.** It does not; the shape is untouched. Conversely, thinking a rotation changes which 2-3 tree you correspond to: a rotation during insertion is precisely what *restores* the correspondence.
10. **Forgetting cascading.** One fix can cause another violation at the parent. Fixups must be applied all the way up to the root, and the root must be recolored black at the end.
11. **Building a 2-3 tree and converting it.** Never the plan; the whole point is that 2-3 trees are hard to build. We insert into a BST and rotate.
12. **Assuming LLRB height is exactly `2H`.** The bound is `2H + 1` in the worst case, and the LLRB may be exactly `H` tall if there are no 3-nodes.
13. **Confusing "number of BSTs" with `2^N` or `N!`.** It is `Catalan(N)`; for `N = 3` it is 5, not 6 or 8.
14. **Expecting delete.** LLRB deletion is not covered in this course.

---

## Likely Exam Points

**1. Perform a specified rotation and draw the result.**

*Q:* In the tree below, what is the result of `rotateLeft(9)`? Who becomes `9`'s new parent?

```
      6
    /   \
   1     8
    \     \
     4     9
            \
             13
               \
                17
```

*A:* `13`. `rotateLeft(9)` promotes `9`'s right child, and `13`'s old left child (`null`) becomes `9`'s new right child:

```
      6
    /   \
   1     8
    \     \
     4     13
          /  \
         9    17
```

**2. Give a rotation sequence that balances a given BST.**

*Q:* Balance the 7-node tree `6 (1 (-,4), 8 (-, 9 (-, 13 (-, 17))))` using rotations.

*A:* `rotateLeft(9)`, `rotateLeft(6)`, `rotateLeft(1)`, `rotateRight(6)`, yielding root `8` with children `4` and `13`, and leaves `1`, `6`, `9`, `17`. Other sequences also work.

**3. Convert a 2-3 tree to its LLRB (or back).**

*Q:* Draw the LLRB corresponding to the 2-3 tree with root `uw` and children `as`, `v`, `xy`.

*A:* Root `w`; `w.left = u` (**red**); `w.right = y` with `y.left = x` (**red**); `u.left = s` with `s.left = a` (**red**); `u.right = v`. Three red links total (`w-u`, `y-x`, `s-a`), because there are three 3-nodes.

**4. Decide whether a given red-black tree is a valid LLRB.**

*Q:* Is this a valid LLRB? Root `G`; `G.left = B` via a **red** link; `B.left = A` (black); `B.right = null`; `G.right = X` (black).

*A:* No. Although every root-to-**leaf** path has the same number of black links, the root-to-**null** path through `B.right` has only 1 black link while the others have 2. Equivalently, the corresponding 2-3 tree would be the 3-node `BG` with children `A`, (missing), `X`, and a 3-node must have three children.

**5. Height relationships between a 2-3 tree and its LLRB.**

*Q:* A 2-3 tree has height `H`. What is the maximum possible height of the corresponding LLRB, and why?

*A:* `2H + 1`. A longest root-to-leaf path crosses `H` black links and touches `H + 1` nodes; in the worst case every one of those is a 3-node, contributing one extra red link each, for `H + 1` red links. Hence `H + (H + 1) = 2H + 1`, which is still `Θ(log N)`.

*Variant Q:* For the lecture's height-3 example 2-3 tree with 3-nodes `DE`, `SU`, `QR`, `VW`, `MO` (and others), how tall is the LLRB?

*A:* 5. The longest path is `L -> P -> U -(red)- S -> R -(red)- Q`: 3 black links plus 2 red links.

**6. Name the violation and the fix.**

*Q:* For each situation, state the violation and the operation that repairs it: (a) a node `h` whose right child is red and left child is black; (b) a node `h` with `h.left` red and `h.left.left` red; (c) a node `h` with both children red.

*A:* (a) Left Leaning Violation, fix with `rotateLeft(h)`. (b) Incorrect 4 Node Violation, fix with `rotateRight(h)`. (c) Temporary 4 Node, fix with a **color flip** of `h` (emulating 2-3 `split`); no rotation needed.

**7. Trace an insertion, including cascading.**

*Q:* Starting from LLRB `B` (black root) with `B.left = A` and `B.right = S`, where `S.left = E` is red, insert `Z`. Show every step.

*A:* `Z` enters as a red right child of `S`, creating the temporary 4-node `ESZ` (correct shape: `S` with red `E` and red `Z`). `flipColors(S)` makes `S-E` and `S-Z` black and `B-S` **red**, which is a right-leaning red link. Cascade with `rotateLeft(B)`: new root `S`, with red left child `B` (children `A` and `E`) and black right child `Z`. The corresponding 2-3 tree is root `BS` with children `A`, `E`, `Z`.

**8. Runtimes.**

*Q:* Give the worst-case running time of `contains` and `insert` on an LLRB with `N` keys, and justify.

*A:* Both `Θ(log N)`. The LLRB is in 1-1 correspondence with a 2-3 tree of height `Θ(log N)`, and the LLRB is at most about twice as tall, so height is `Θ(log N)`. `contains` is plain BST search down one path. `insert` is a search plus `O(log N)` rotations and color flips performed on the way back up, each `Θ(1)`.

**9. Why not implement 2-3 trees directly?**

*Q:* Name three implementation difficulties of 2-3 trees that LLRBs avoid.

*A:* Maintaining different node types; interconverting 2-nodes and 3-nodes; walking up the tree to split nodes. An LLRB is just a BST with one extra boolean per node.

**10. Conceptual trap about colors.**

*Q:* True or false: searching an LLRB requires following red links differently from black links.

*A:* False. Red is bookkeeping only. Search is identical to ordinary BST search; colors matter only to insertion's fixup rules.

**11. Counting BSTs / reachability.**

*Q:* How many distinct BSTs contain exactly the keys 1, 2, 3? Can you always get from one to another with rotations?

*A:* Five, and yes: any BST configuration is reachable from any other, in at most `2n - 6` rotations.

**12. Java library knowledge.**

*Q:* Is `java.util.TreeMap` an LLRB?

*A:* No. It is a red-black tree that is **not** left leaning: it allows glue links on either side and corresponds to a 2-3-**4** tree, which is not a 1-1 correspondence. It is more complex but significantly faster.

---

## Summary

- **B-trees are balanced but painful:** multiple node types, 2-node/3-node interconversion, and splitting while walking up the tree. Hence the search for something implementable.
- **Rotation** reshapes a BST without changing its key set or search tree property. `rotateLeft(x)`: `x`'s right child becomes `x`'s new parent, and that child's old left subtree becomes `x`'s new right subtree. `rotateRight(x)` is the mirror image. A rotation is invalid if the needed child is missing.
- Rotation can **shorten or lengthen** a tree; `Catalan(N)` BSTs exist on `N` keys and all are mutually reachable (at most `2n - 6` rotations). Rebalancing a BST from scratch costs `O(N)` moves, which is why we want balance maintained incrementally instead.
- **The big idea:** color some links **red** so each red link glues two BST nodes into one 3-node. This gives an exact **1-1 correspondence (isometry)** between a BST and a 2-3 tree. Red links point only left, giving an **LLRB**.
- **LLRBs are ordinary BSTs.** Search ignores color. Red is "a convenient fiction" used only to guide insertion.
- **Invariants:** no node has two red links (that would be a 4-node), and every root-to-**null** path has the same number of black links (perfect black balance). Root-to-*leaf* is the wrong formulation.
- **Height:** a 2-3 tree of height `H` maps to an LLRB of height at most `H + (H + 1) = 2H + 1`, so LLRBs have `Θ(log N)` height.
- **Insertion:** insert into the BST as usual with a **red** link, then on the way back up apply, as needed: `rotateLeft` for a right-leaning red link, `rotateRight` for two consecutive left red links, and a **color flip** for a node with two red children (which emulates 2-3 `split` and changes no pointers). Fixes may **cascade** upward; finish by recoloring the root black.
- **Runtime:** `contains` and `insert` are `O(log N)`, with `O(log N)` rotations and flips per insert. **Delete is not covered.**
- **Landscape:** plain BSTs are simple but can degenerate; 2-3/B-trees are balanced but clumsy; LLRBs give easy balanced insertion via the 2-3 bijection; `java.util.TreeMap` is a non-left-leaning red-black tree tracking a 2-3-4 tree. Other options exist: AVL, WAVL, splay trees, treaps, skip lists, and hashing (coming later).
