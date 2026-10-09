<!-- Fri, Oct 09, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 19: Red-Black Trees

## Overview

This lecture closes the loop on the balanced-search-tree story. 2-3 trees (B-trees with small `L`) give us guaranteed logarithmic height, but they are genuinely painful to implement: multiple node types, conversions between 2-nodes and 3-nodes, and splits that propagate back up the tree. So we go back to the plain binary search tree and ask: can we keep a *BST* balanced? The key new tool is **tree rotation**, an O(1) pointer surgery that changes a BST's shape while preserving its ordering (and therefore its semantics). Rotations alone can rebalance any BST, but figuring out *which* rotations to perform is the hard part. The wild idea of the lecture is to maintain a **one-to-one correspondence (isometry) between a BST and a 2-3 tree**: we represent each 3-node as two BST nodes joined by a "glue" link that we *pretend* is colored red and that we always make lean left. The result is a **Left-Leaning Red-Black BST (LLRB)**, which is just a normal BST, but because it is in bijection with a 2-3 tree it inherits logarithmic height (at most about 2x the 2-3 tree's height). Insertion is then "insert as a normal BST with a red link, then apply zero or more rotations and color flips on the way back up" to restore the correspondence, giving O(log N) `contains` and O(log N) `insert` with code that fits in a handful of lines.

---

## Key Concepts

### 1. Why we abandon direct 2-3 / 2-3-4 tree implementations

2-3 trees and 2-3-4 trees are beautiful on the whiteboard and awful in code. The lecture lists the specific pain points:

- **Maintaining different node types** (a 2-node holds 1 item and 2 children; a 3-node holds 2 items and 3 children).
- **Interconversion of nodes** between 2-nodes and 3-nodes (and temporary 4-nodes).
- **Walking back up the tree to split nodes** after an overflow.

The slides show "fantasy 2-3 code" (credited to Kevin Wayne) to illustrate how much machinery you would need, and quote Knuth: *"Beautiful algorithms are, unfortunately, not always the most useful."*

### 2. The BST shape depends on insertion order, and rotations let you change it

For keys 1, 2, 3 there are five distinct BSTs, and which one you get depends on insertion order. In general, for N items there are **Catalan(N)** different BST shapes. Crucially, **any BST shape can be converted into any other BST shape over the same keys using rotations**, and the lecture cites a bound of **2n - 6 rotations** in general (reference: *Rotation Distance, Triangulations, and Hyperbolic Geometry*, or Amy Liu). So "rotation" is expressive enough to be the only structural tool we need.

### 3. Tree rotation: definition and intuition

> `rotateLeft(G)`: Let `x` be the right child of `G`. Make `G` the new left child of `x`.
>
> `rotateRight(P)`: Let `x` be the left child of `P`. Make `P` the new right child of `x`.

Three intuitions the lecture gives for the same operation:

**(a) "I'm going left."** `rotateLeft(G)` sends `G` down and to the left; `G`'s new parent is `G`'s old **right** child. That child becomes the new root of the subtree ("I'll be G's new boss"). Mirror image for `rotateRight`.

**(b) The orphan rule.** When `G` moves down-left under `x`, `x`'s old left subtree `k` "doesn't make sense" where it is: it holds keys greater than `G` but less than `x`. The only legal home for it is **the right child of `G`**. The slides literally narrate this: `k` says "I got transferred from P to G."

**(c) Merge-then-demote.** Think of `rotateLeft(G)` as temporarily merging `G` and its right child `P` into one node `GP` (which then has 4 children), then sending `G` down and left out of that merged node. This is the interpretation that will matter later, because *merging a parent with a child and re-splitting it differently* is exactly what a 3-node in an LLRB is.

**Rotation preserves the search tree property.** There is "no change to the semantics of the tree": the set of keys is the same and the in-order traversal is the same. Only the shape changes. (extra context: that in-order traversal is unchanged is the cleanest one-line proof that rotation preserves BST-ness.)

**Rotations can increase or decrease height.** In the lecture's example, `rotateLeft(G)` *increased* the height, while the inverse `rotateRight(P)` *decreased* it. Rotation is not inherently a balancing operation; it is a shape-changing operation that you aim at balance.

**Some rotations are undefined.** If the node has no appropriate child to take its place, the operation is invalid. Example from the slides: `rotateRight(1)` where node `1` has only a right child `4` is an **INVALID OPERATION**, because there is no left child to become the new subtree root.

### 4. Rotation for balance, and why "rebalance occasionally" is not good enough

Since rotation can shorten a tree and preserves ordering, you can use rotations to balance any BST, and this takes **O(N) moves**. But paying O(N) occasionally to rebalance is not ideal. We want balance maintained *incrementally*, with O(log N) work per insert. The rest of the lecture builds that.

### 5. The big idea: maintain a 1-1 correspondence with a 2-3 tree

If a BST is imbalanced, rotations can always fix it; the tricky part is knowing *which* rotations. The lecture's answer:

> **Wild idea:** perform rotations such that we maintain a 1-to-1 correspondence with a 2-3 tree. Since 2-3 trees are always balanced, our BST will also be balanced.

So the algorithm never has to "think about balance" at all. It only has to think: *is my BST still a faithful picture of some legal 2-3 tree?* If yes, balance is free.

### 6. How do you draw a 2-3 tree as a BST?

- A 2-3 tree containing **only 2-nodes** is trivial to represent: the BST is literally identical.
- **3-nodes are the problem.** A 3-node holds two items and three children; a BST node holds one item and two children.

Two possibilities were considered:

**Possibility 1: dummy "glue" nodes.** Represent 3-node `d f` as a glue node labeled `df` with `d` as left child and `f` as right child. The lecture rejects this: "Result is inelegant. Wasted link. Code will be ugly."

**Possibility 2: glue *links*, smaller item off to the left.** Represent 3-node `d f` as BST node `f` with `d` as its **left child**, and mark the link `f -> d` as **red**. No extra nodes, no wasted links. This is what is "commonly used in practice (e.g. `java.util.TreeSet`)."

The red color is **a convenient fiction**. Red links do not "do" anything special at runtime; they are bookkeeping that tells us which BST nodes are secretly part of the same 2-3 node.

### 7. LLRBs are just BSTs

> A BST with left glue links that represents a 2-3 tree is called a **Left Leaning Red Black Binary Search Tree (LLRB)**.
>
> - LLRBs are normal BSTs.
> - There is a 1-1 correspondence between an LLRB and an equivalent 2-3 tree.
> - The red is just a convenient fiction.

Consequence: **searching an LLRB is exactly BST search.** No color logic at all in `contains`. Colors are consulted only during insertion, to restore the correspondence.

### 8. LLRB invariants

The lecture's "handy LLRB properties":

1. **No node has two red links.** (That would be analogous to a 4-node, which is disallowed in 2-3 trees.) In the left-leaning discipline, this also means no two consecutive left red links and no simultaneous red-left plus red-right, except *temporarily* during insertion.
2. **Every path from the root to a null has the same number of black links** (because every leaf in a 2-3 tree is at the same depth). This is **perfect black balance**, and it is why LLRBs are balanced.
3. Red links lean **left** (in a settled tree).

**Subtle point the lecture explicitly calls out:** the invariant is about root-to-**null** paths, not root-to-**leaf** paths. An older version of the lecture said "the number of black links to any leaf must be the same," which is wrong. Counterexample from the slides: root `G` with children `B` and `X`, where `B` has a single child `A`. This can satisfy the leaf-based version while having no corresponding 2-3 tree, because the 2-3 node `B G` would be missing a child. Counting to nulls catches it.

### 9. LLRB height, hence balance

Each 3-node of the 2-3 tree becomes **two** nodes in the LLRB, joined by a red link. So a root-to-leaf path in the 2-3 tree with `H` links turns into a path with `H` black links plus up to one extra red link per node on the path.

- Lecture's concrete case: a 2-3 tree with "tallness" (number of links on the longest root-to-leaf path) of **3** produced an LLRB of height **3 black + 2 red = 5**.
- General bound from the slides: for a 2-3 tree of height `H`, the corresponding LLRB has height at most `H` (black) `+ H + 1` (red) `= 2H + 1`, i.e. **never more than about 2x the 2-3 tree's height**.
- Since 2-3 trees have logarithmic height, **LLRBs have logarithmic height**. That is the entire balance argument.

(extra context: the `H + 1` red count comes from there being `H + 1` nodes on a root-to-leaf path, each of which could be a 3-node contributing one red link.)

### 10. Where do LLRBs come from? Insertion by local repair

We do **not** build a 2-3 tree and convert it; 2-3 trees are hard to build, which was the whole problem. Instead:

> - Insert as usual into a BST.
> - Use zero or more rotations (and color flips) to maintain the 1-1 mapping.

The guiding slogan when deciding what to do: **"What Would 2-3 Tree Do?" (WW23TD?)** Simulate the 2-3 tree operation in your head, then make the LLRB look like the correct picture of the resulting 2-3 tree.

The four design tasks of the lecture derive the whole algorithm:

**Design Task #1: what color is a new link?** **Red.** In a 2-3 tree, a new value is always added into an existing leaf node first (making a 2-node into a 3-node, or a 3-node into a temporary 4-node). A new red link is exactly "this item joined an existing node."

**Design Task #2: insertion on the right.** Insert `S` as the red right child of leaf `E`. The 2-3 world says `E` and `S` now form one 3-node `ES`, which must be drawn as `S` on top with red-left child `E`. The LLRB currently has a red *right* link, which is illegal. Fix: **`rotateLeft(E)`**. This is the **Left Leaning Violation** rule.

**Design Task #3: double insertion on the left.** Starting from the 3-node `S Z` (drawn as `Z` with red-left `S`) and inserting `E`, you get `Z -> S -> E`, two consecutive left red links. In the 2-3 world this is the temporary 4-node `E S Z`, which should be drawn as the middle item `S` on top with **two** red children, `E` on the left and `Z` on the right. Fix: **`rotateRight(Z)`**. This is the **Incorrect 4 Node Violation** rule. New representation rule introduced here: **temporary 4-nodes are BST nodes with two red links**, and that temporary violation of "left leaning" is OK because it is about to be resolved.

**Design Task #4: splitting a temporary 4-node.** Given `G` with left child `B`, where `B` has two red children `A` and `C`, the 2-3 world calls `split(A/B/C)`: `B` moves up to join `G`, forming the 3-node `B G` with children `A`, `C`, `X`. The correct LLRB picture of that is: `G` with a **red** left link to `B`, and `B` with **black** links to `A` and `C`. Compare to what we have: `B` black from `G`, red to `A` and `C`. The transformation is **flip the colors of all three edges touching `B`** (`colorFlip(B)` / `flipColors`). No rotation needed, and **the BST structure/shape does not change at all**. The slide's punchline: "BST, the magic was inside of you all along."

### 11. The complete LLRB insertion rule set

> Congratulations, you just invented the red-black BST.
> - When inserting: use a **red** link.
> - If there is a right-leaning "3-node" (**Left Leaning Violation**): **rotate left** the appropriate node.
> - If there are two consecutive left red links (**Incorrect 4 Node Violation**): **rotate right** the appropriate node.
> - If a node has two red children (**Temporary 4 Node**): **color flip** that node to emulate `split`.

**Cascading operations.** A rotation or flip can create a *new* violation one level up, which then needs fixing, exactly as a split in a 2-3 tree can cascade upward. The lecture's example: from `B` with children `A` and the 3-node `ES` (drawn `S` with red-left `E`), `add(Z)` makes `S` have two red children, so `flip(S)`. Now the link `B -> S` is red and right-leaning: a Left Leaning Violation one level up. Fix with **`rotateLeft(B)`**, giving root `S` with red-left `B` (children `A`, `E`) and right child `Z`, which corresponds to the 2-3 tree `B S` over `A`, `E`, `Z`.

### 12. Runtime

- LLRB height is O(log N).
- `contains` is trivially **O(log N)** (plain BST search).
- `insert` is **O(log N)**: O(log N) to walk down and add the node, plus O(log N) rotations and color flips on the way back up (constant work per level).
- **Delete is not covered** in CS 61B. ("Not too terrible really, but it's just not interesting enough to cover.")

### 13. Search tree summary (the last three lectures in one place)

- **BSTs**: simple, but subject to imbalance (worst case Θ(N) height).
- **2-3 / B-trees**: balanced, but painful to implement and relatively slow.
- **LLRBs**: insertion is simple to implement (delete is hard), and it works by maintaining a mathematical bijection with a 2-3 tree.
- **Java's `TreeMap`** is a red-black tree but **not left-leaning**: it maintains a correspondence with a **2-3-4** tree, which is **not** a 1-1 correspondence (glue links are allowed on either side). More complex implementation, but significantly faster.
- Beyond: AVL trees, WAVL trees (described in the slides as fastest), splay trees, treaps, and hundreds of others. Also non-tree approaches: **skip lists** ("linked lists with express lanes") and **hashing**, the most common alternative, coming in a later lecture.

---

## Definitions

- **Tree rotation**: a local, O(1) restructuring of a BST that changes the shape while preserving the search tree property (and the in-order key order).
- **`rotateLeft(G)`**: let `x` be the right child of `G`; make `G` the new left child of `x`. `x` takes `G`'s former position; `x`'s old left subtree becomes `G`'s new right subtree. Undefined if `G` has no right child.
- **`rotateRight(P)`**: let `x` be the left child of `P`; make `P` the new right child of `x`. `x` takes `P`'s former position; `x`'s old right subtree becomes `P`'s new left subtree. Undefined if `P` has no left child.
- **Rotation distance**: the minimum number of rotations to turn one BST shape into another over the same keys; the lecture cites a general bound of `2n - 6`.
- **Catalan(N)**: the number of distinct BST shapes on N items.
- **Glue node (Possibility 1, rejected)**: a dummy BST node used to hold together the two items of a 3-node, with the smaller item as its left child and the larger as its right child.
- **Glue link / red link (Possibility 2, adopted)**: a BST link, drawn red, connecting the two items of a single 2-3 tree 3-node. The smaller item hangs off to the **left**. Red links do not change BST behavior; they are a bookkeeping fiction.
- **Left Leaning Red Black BST (LLRB)**: a BST with left-leaning red glue links that is in 1-1 correspondence with a 2-3 tree.
- **Isometry / 1-1 correspondence / bijection**: the exact structural mapping between an LLRB and a 2-3 tree, under which every legal LLRB corresponds to exactly one legal 2-3 tree and vice versa. Maintaining this correspondence is the entire implementation strategy.
- **Perfect black balance**: every path from the root to a **null** link passes through the same number of black links. (Root-to-null, not root-to-leaf.)
- **Height / "tallness"**: the number of **links** on the longest path from root to leaf.
- **Temporary 4-node**: a BST node with **two** red child links, representing a 2-3 tree node that momentarily holds three items; must be resolved by a color flip.
- **Left Leaning Violation**: a red link that leans right. Fixed by rotating that node left.
- **Incorrect 4 Node Violation**: two consecutive left-leaning red links (a red node whose left child is also red). Fixed by rotating the top node right.
- **Color flip (`colorFlip` / `flipColors`)**: recoloring all three links touching a node (the link to its parent becomes red; the links to its two children become black). Emulates a 2-3 tree `split`. Does **not** change tree shape.
- **Cascading**: a rotation or color flip creating a new violation at an ancestor, which must then be repaired, analogous to a split propagating up a 2-3 tree.

---

## Worked Examples

### Example 0: the "fantasy" 2-3 code (why we are doing any of this)

This is the code from the slides, shown precisely because it is *not* code you want to write:

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

Every single method here (`is2Node`, `is3Node`, `is4Node`, `split`, `make3Node`, `make4Node`) hides a pile of case analysis over variable-arity nodes, plus the parent pointers or recursion needed to walk back up and split. Compare with the ~15 lines of LLRB insert at the end of this section. That contrast is the argument of the lecture.

### Example 1: `rotateLeft` in code, with box-and-pointer reasoning

```java
private Node rotateLeft(Node h) {
    Node x = h.right;     // x is the node that will take h's place
    h.right = x.left;     // x's old left subtree is re-parented as h's right subtree
    x.left = h;           // h becomes x's left child ("h goes left")
    return x;             // caller must re-link: the subtree root changed!
}
```

Step by step, using the lecture's tree. Before `rotateLeft(G)`:

```
        G
      /   \
     C     P
    /     / \
   A     k   r
    \   / \
     B j   l
```

1. `Node x = h.right;` creates a new local reference `x` pointing at the same box as `G.right`, i.e. node `P`. No boxes change yet, only a new arrow from the stack frame.
2. `h.right = x.left;` overwrites `G`'s right pointer so it now points at `k`'s box. `G` has temporarily "lost" `P`, but we saved it in `x`. This is legal by the BST property: everything in `k` is `> G` and `< P`.
3. `x.left = h;` overwrites `P`'s left pointer to point at `G`'s box. Now `P` is above `G`, and `G`'s subtree is `C` on the left and `k` on the right.
4. `return x;` is essential. The box holding `G`'s *parent* still has a pointer to `G`. The caller is responsible for reassigning that pointer to `P`. This is the standard CS 61B recursive-BST idiom: a helper returns the (possibly new) root of the subtree it was given, and the caller writes `h.left = rotateLeft(h.left)` or `root = rotateLeft(root)`.

After:

```
        P
      /   \
     G     r
    / \
   C   k
  /   / \
 A   j   l
  \
   B
```

Note that in this example the rotation **increased** the tree's height, which the slides highlight. In-order traversal before and after is identical: `A B C G j k l P r`.

### Example 2: `rotateRight` is the inverse

```java
private Node rotateRight(Node h) {
    Node x = h.left;
    h.left = x.right;     // x's old right subtree becomes h's left subtree
    x.right = h;          // h becomes x's right child ("h goes right")
    return x;
}
```

Applying `rotateRight(P)` to the tree we just produced takes us straight back to the original, and the slides point out the key re-parenting fact: **"`k` was `G`'s right child. Now it is `P`'s left child."** For this example, `rotateRight(P)` **decreased** the height of the tree. `rotateLeft` and `rotateRight` are exact inverses of each other at a given edge.

### Example 3: the lecture's full balancing demo (4 rotations)

Start with this badly right-leaning BST (height 4):

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

**Step 1: `rotateLeft(9)`.** `9` goes left; `9`'s new parent is its right child, `13`. `13`'s old left child is null, which becomes `9`'s right child.

```
      6
    /   \
   1     8
    \     \
     4     13
          /  \
         9    17
```

*(This was the clicker question "If `rotateLeft(9)`, who will be 9's new parent?" Answer: **C, 13**. Takeaway stated on the slide: "If you `rotateLeft(x)`, `x` goes left, and `x`'s new parent is its right child.")*

**Step 2: `rotateLeft(6)`.** `6` goes **left**, and its new parent is its right child **8**. `8` becomes the new root.

```
        8
      /   \
     6     13
    /     /  \
   1     9    17
    \
     4
```

*(Clicker question "If `rotateLeft(6)`, then 6 goes ____, and who will be 6's new parent?" Answer: **C, Left, 8**.)*

**Step 3: `rotateLeft(1)`.** `1` goes left under its right child `4`.

```
        8
      /   \
     6     13
    /     /  \
   4     9    17
  /
 1
```

**Step 4: `rotateRight(6)`.** `6` goes right under its left child `4`; `4`'s old right child (null) becomes `6`'s left child.

```
        8
      /   \
     4     13
    / \   /  \
   1   6 9    17
```

Perfectly balanced, height 2, same keys, same in-order order `1 4 6 8 9 13 17`. Full recipe from the summary slide: `rotateLeft(9)`, `rotateLeft(6)`, `rotateLeft(1)`, `rotateRight(6)`.

### Example 4: the small in-class exercise

Balance this tree:

```
   1
    \
     3
    /
   2
```

Answer given in the slides (other correct answers exist):

1. **`rotateRight(3)`**: `3`'s left child is `2`, so `2` takes `3`'s place as `1`'s right child, and `3` becomes `2`'s right child.

```
   1
    \
     2
      \
       3
```

2. **`rotateLeft(1)`**: `1`'s right child is `2`, so `2` becomes the root and `1` becomes its left child.

```
     2
    / \
   1   3
```

The slides also point to an interactive game at `https://joshh.ug/61b/bst_rotation/` to build this intuition, and recommend playing it after class.

### Example 5: drawing the LLRB for a given 2-3 tree

2-3 tree (root is the 3-node `u w`):

```
          [u w]
         /  |  \
     [a s] [v] [x y]
```

Procedure: every 2-node stays as-is; every 3-node `p q` becomes `q` with a **red left** child `p`. Children attach in sorted order.

Resulting LLRB (red links marked `*`):

```
         w
       */  \
      u     y
     / \  */
    s   v x
  */
  a
```

Reading it out: root `w`; `w`'s red-left child `u` (together they are the 3-node `u w`); `u`'s left child `s` with red-left child `a` (the 3-node `a s`); `u`'s right child `v` (a 2-node, so just itself); `w`'s right child `y` with red-left child `x` (the 3-node `x y`).

Check the invariants: no node has two red links; and every root-to-null path has the same number of black links (for example `w -> u` is red, `u -> s` black, `s -> a` red, `a -> null` black: 2 black; and `w -> y` black, `y -> null` black: 2 black).

**Searching this tree is plain BST search.** To find `v`: `v > w`? No, go left to `u`. `v > u`? Yes, go right. Found. Colors never consulted.

### Example 6: LLRB validity (LLRB Problem #1)

The lecture showed four candidate trees over the keys `A`, `B`, `C`, `G`, `X` that differ only in link colors and shape, and asked how many have a 1-1 correspondence with a *valid* 2-3 tree. **Exactly one is valid.** The three failure modes identified were:

1. **Has a 4-node.** A node reached by a chain `G -> C -> B -> A` where `C -> B` and `B -> A` are both red glues three items into one node, giving the 2-3 node `A B C` under root `G`. 2-3 trees do not allow three items in a node, so this is invalid.
2. **Not balanced (variant 1).** If `G -> B` is black while `B -> A` is red, the 2-3 node is `A B` with child `C` on the right but nothing on the other two child slots. A 3-node with one child is not a legal 2-3 tree, and the corresponding LLRB fails perfect black balance.
3. **Not balanced (variant 2).** If all of `G -> B`, `B -> A`, `B -> C` are black, the 2-3 tree is `G` with left child `B` (holding `A`, `C`) and right child `X`. Leaf `X` is at depth 1 while `A` and `C` are at depth 2, so the 2-3 tree is not balanced. Equivalently, root-to-null black counts differ.
4. **Valid.** `G` with a **red** left link to `B`, where `B` has black children `A` and `C`, and `G` has black right child `X`. The 2-3 tree is the 3-node `B G` with children `A`, `C`, `X`: all leaves at depth 1. Black counts from root to every null are equal.

The same four shapes reappear on the "LLRB Properties" slide labeled with the two invariants being violated: "Invalid, `B` has two red links", "Invalid, not black balanced", "Invalid, not black balanced", "Valid". (The exact coloring of each diagram is not recoverable from the extracted slide text; what matters for the exam is the *checklist*, below.)

**Validity checklist for an LLRB:**
- No node has two red links (no red-left + red-right, no red-left whose child link is also red).
- Red links lean left.
- Every root-to-**null** path has the same number of black links.
- Equivalently and most reliably: **convert it to a 2-3 tree and check that the 2-3 tree is legal** (every node has 1 or 2 items, every node is either a leaf or has exactly #items + 1 children, and all leaves are at the same depth).

### Example 7: LLRB height (LLRB Problem #2)

2-3 tree (3-nodes shown in brackets with two items):

```
                       [L]
              /                  \
           [D E]                  [P]
         /   |   \              /     \
      [B]   [G]  [J]         [N]      [S U]
      / \   / \  / \         / \      /  |  \
     A  C  F  H I  K        M   O  [Q R]  T  [V W]
```

The 2-3 tree has height ("tallness") **3**. The LLRB's height is **not** 3.

Reasoning: each 3-node becomes two LLRB nodes joined by a red link, so the longest LLRB path picks up one extra link per 3-node along the way. Take the path `L -> P -> [S U] -> [Q R]`:

- `L -> P`: black
- `P -> U`: black (`U` is the top of the 3-node `S U`)
- `U -> S`: **red** (the glue link of `S U`)
- `S -> R`: black (`R` is the top of the 3-node `Q R`)
- `R -> Q`: **red** (the glue link of `Q R`)

Total: **3 black + 2 red = height 5**. The general statement from the slides: a 2-3 tree of height `H` yields an LLRB of height at most `H + (H + 1) = 2H + 1`, i.e. an LLRB is **never more than about 2x the height of its 2-3 tree**. Since 2-3 trees have logarithmic height, LLRBs do too, and `contains` is O(log N).

### Example 8: the four design tasks, as pictures and rules

Notation: `*` marks a red link to the node below it.

**Task 1 (new links are red).** `add(E)` to a tree whose relevant leaf is `S`. In 2-3 world, `S` becomes the 3-node `E S`. If we inserted `E` with a *black* link we would be claiming the 2-3 tree grew a new level, which it did not. So:

```
LLRB:   S      add(E)        S
                          */
                          E

2-3:   [S]     add(E)      [E S]
```

**Task 2 (Left Leaning Violation -> rotate left).**

```
Before:      B                 add(S)          B                 rotateLeft(E)        B
           /   \                             /   \                                  /   \
          A     E                           A     E                                A     S
                                                   \*  (illegal red right)             */
                                                    S                                 E

2-3:     [B] over A, [E]       add(S)      [B] over A, [E S]
```

The 2-3 tree says `E` and `S` are one 3-node; the only legal LLRB drawing of `E S` is `S` on top with red-left `E`; `rotateLeft(E)` produces exactly that.

**New representation rule (temporary 4-nodes).** From `B` with children `A` and the 3-node `E S` (drawn `S` with red-left `E`), `add(Z)` gives `S` a red right child too:

```
      B
    /   \
   A     S
       */ \*
       E   Z
```

This is the LLRB picture of the temporary 4-node `E S Z`. It temporarily violates "no red right links" and the 2-3 rule "no 4-nodes", and that is fine because it is about to be fixed.

**Task 3 (Incorrect 4 Node Violation -> rotate right).**

```
Before:      B            add(E)        B                rotateRight(Z)       B
           /   \                      /   \                                 /   \
          A     Z                    A     Z                               A     S
                */                       */                                    */ \*
                S                        S                                     E   Z
                                       */
                                       E

2-3:    [B] over A, [S Z]  add(E)   [B] over A, [E S Z]
```

Two consecutive left red links (`Z -> S -> E`) are the wrong drawing of the temporary 4-node; the right drawing puts the **middle** item `S` on top with red children on both sides. `rotateRight(Z)` does it.

**Task 4 (Temporary 4 Node -> color flip).**

```
Before:         G             flip(B)          G
              /   \                          */ \
             B     X                        B     X
           */ \*                           /  \
           A    C                          A    C

2-3:  [A B C] under G, sibling X    split(A/B/C)    [B G] over A, C, X
```

`split` in 2-3 world promotes the middle item `B` into its parent. In LLRB world, "promoting `B` into `G`'s node" means "the link `G -> B` becomes a glue (red) link," and `A` and `C` stop being glued to `B`, so their links become black. That is exactly `flip(B)`: recolor all three edges touching `B`. **The BST shape does not change.**

### Example 9: cascading repair (the lecture's example)

```
Start:        B                      2-3:  [B] over A and [E S]
            /   \
           A     S
               */
               E
```

`add(Z)`: `Z > S`, so it becomes a red right child of `S`.

```
        B                            2-3:  [B] over A and [E S Z]   (temporary 4-node)
      /   \
     A     S
         */ \*
         E   Z
```

`S` has two red children, so **`flip(S)`**: the link `B -> S` becomes red, and `S -> E`, `S -> Z` become black.

```
        B                            2-3:  [B S] over A, E, Z       (split pushed S up)
      /   \*
     A     S
          / \
         E   Z
```

The 2-3 tree is now legal, but the LLRB drawing is wrong: the red link `B -> S` leans **right**. That is a Left Leaning Violation at `B`, one level **above** where we were working. This is the cascade. Fix with **`rotateLeft(B)`**:

```
          S
       */   \
      B      Z
     / \
    A   E
```

Now `S` is the root with red-left `B`, matching the 3-node `B S` with children `A`, `E`, `Z`. Done.

### Example 10: the LLRB insertion code (extra context: the slides' "Runtime and Implementation (Extra)" section stated the runtime but the extracted text does not include the code; this is the standard CS 61B / Sedgewick implementation that the four design tasks derive)

```java
public class LLRB<Key extends Comparable<Key>, Value> {
    private static final boolean RED = true;
    private static final boolean BLACK = false;

    private Node root;

    private class Node {
        Key key;
        Value val;
        Node left, right;
        boolean color;   // color of the link from this node's PARENT to this node

        Node(Key key, Value val, boolean color) {
            this.key = key;
            this.val = val;
            this.color = color;
        }
    }

    /** A null link is black, which is what makes perfect black balance
     *  a statement about root-to-null paths. */
    private boolean isRed(Node x) {
        if (x == null) { return false; }
        return x.color == RED;
    }

    private Node rotateLeft(Node h) {
        Node x = h.right;
        h.right = x.left;
        x.left = h;
        x.color = h.color;   // x inherits the color of the link into this subtree
        h.color = RED;       // h is now glued to x, so the h link is red
        return x;
    }

    private Node rotateRight(Node h) {
        Node x = h.left;
        h.left = x.right;
        x.right = h;
        x.color = h.color;
        h.color = RED;
        return x;
    }

    /** Emulates 2-3 tree split: promote h into its parent's node. */
    private void flipColors(Node h) {
        h.color = RED;
        h.left.color = BLACK;
        h.right.color = BLACK;
    }

    public void put(Key key, Value val) {
        root = insert(root, key, val);
        root.color = BLACK;   // the root is never a glue node
    }

    private Node insert(Node h, Key key, Value val) {
        if (h == null) { return new Node(key, val, RED); }   // Task 1: insert red

        int cmp = key.compareTo(h.key);
        if (cmp < 0)      { h.left  = insert(h.left,  key, val); }
        else if (cmp > 0) { h.right = insert(h.right, key, val); }
        else              { h.val = val; }

        // Repair on the way back up. Order matters.
        if (isRed(h.right) && !isRed(h.left))    { h = rotateLeft(h);  } // Task 2
        if (isRed(h.left) && isRed(h.left.left)) { h = rotateRight(h); } // Task 3
        if (isRed(h.left) && isRed(h.right))     { flipColors(h);      } // Task 4

        return h;
    }
}
```

Things worth noticing about this code:

- **Color lives on the node but *means* the link from its parent.** `h.color == RED` means "the link into `h` is a glue link," i.e. `h` is the smaller (or, momentarily, the larger) item of a multi-item 2-3 node.
- **`isRed(null) == false`** is what makes "root to null black counts are equal" expressible: null links are black.
- **The repairs happen after recursion returns**, which is how the cascade of Example 9 is handled with no extra machinery: each level fixes its own local violation, possibly handing a new one to its parent. At most O(log N) levels, O(1) work each.
- **The three `if`s are in the derived order** (left-lean, then double-left, then flip) and a single insert can trigger more than one of them at the same node, since each can create the precondition of the next.
- **Returning the new subtree root and reassigning `h.left` / `h.right` / `root`** is the only reason we never need parent pointers.

**Trace: insert 1, then 2, then 3 into an empty LLRB.**

1. `put(1)`: `insert(null, ...)` returns a RED node `1`; `put` sets `root.color = BLACK`. Tree: single black node `1`. 2-3 tree: `[1]`.
2. `put(2)`: at `h = 1`, `cmp > 0`, so `1.right` becomes a RED node `2`. Now `isRed(h.right) && !isRed(h.left)` is true, so `rotateLeft(1)`: `2` becomes the subtree root with color BLACK (inherited), `1` becomes `2`'s left child with color RED. Tree: `2` with red-left `1`. 2-3 tree: `[1 2]`.
3. `put(3)`: at `h = 2`, `cmp > 0`, so `2.right` becomes RED node `3`. Check rule 1: `isRed(right)` yes, but `isRed(left)` is also yes (`1` is red), so rule 1 does **not** apply. Rule 2: `h.left.left` is null, not red, so no. Rule 3: `isRed(left) && isRed(right)` yes, so `flipColors(2)`: `2` becomes RED, `1` and `3` become BLACK. Back in `put`, `root.color = BLACK`. Tree: `2` with black children `1` and `3`. 2-3 tree: `[2]` over `[1]` and `[3]`, exactly the split we expect.

### Example 11: the optional exercise (insert 7, 6, 5, 4, 3, 2, 1)

Insert `7, 6, 5, 4, 3, 2, 1` into an initially empty LLRB. Each insert goes to the far left, so you repeatedly trigger the double-left rule and then color flips. The end result is a **perfectly balanced** BST:

```
        4
      /   \
     2     6
    / \   / \
   1   3 5   7
```

This is the striking part: the same key sequence would produce a single Θ(N)-tall right-leaning chain in a naive BST. The slides link a demo and a video walkthrough of the solution.

### Example 12: the Rule Derivation worksheet at the end of the deck

The final three slides ("RULE 1 / 2 / 3 DERIVATION") walk through the same three rules once more as a fill-in exercise, on a tree rooted at `F`:

- **Rule 1**: add a key to a 2-node leaf in 2-3 world (so that leaf becomes a 3-node); in LLRB world the new node arrives as a red **right** child, so you **rotate left** at the node with the red right link, then redraw the whole tree with red edges dashed or labeled.
- **Rule 2**: insert one more key so that three items end up glued in the wrong shape (two consecutive left red links); you **rotate right** the top of that chain so the temporary 3-item node has one red child on each side.
- **Rule 3**: the shape is now a node with two red children, which a 2-3 tree would handle with `split`; here you call **`colorFlip`** on that node, then convert the resulting LLRB back into its equivalent 2-3 tree.

(The extracted slide text for these three slides has inconsistent key labels, for example `add(A)` in the prose versus `add(S)` in the diagram, so the specific letters are not reliable; the three rules and the order they are derived in are.)

---

## Common Pitfalls

1. **Thinking red/black is a property of nodes rather than links.** The color belongs to the link from a node to its parent. The code stores it in the child node purely for convenience. A "red node" means "I am glued to my parent."
2. **Thinking red links do something at runtime.** They are "a convenient fiction." `contains` on an LLRB is literally BST `contains` and never looks at a color.
3. **Getting the rotation direction backwards.** `rotateLeft(x)` is named for the node that goes **down and left**; the node that comes **up** is `x`'s old **right** child. Students frequently name the rotation after the node that rises.
4. **Forgetting to re-parent the orphaned subtree.** In `rotateLeft(G)`, `G.right` must be set to the rising node's old **left** subtree. Skipping this loses a subtree or breaks the ordering.
5. **Forgetting that the caller must re-link.** `rotateLeft` returns a new subtree root. If you call it and throw away the return value, the grandparent still points at the old node and the tree is corrupted.
6. **Attempting an undefined rotation.** `rotateRight(x)` requires `x` to have a left child; `rotateLeft(x)` requires a right child. `rotateRight(1)` on a node with only a right child is invalid.
7. **Using root-to-leaf instead of root-to-null for black balance.** The lecture flags this as a real historical error in the course. The tree `G` with red-left `B`, `B` with a single child `A`, and `G.right = X` can pass a root-to-leaf check yet have no corresponding 2-3 tree.
8. **Believing LLRBs are perfectly balanced.** They are only *height-balanced to within a constant factor*: height at most `2H + 1` where `H` is the 2-3 tree height, so O(log N), not necessarily perfect.
9. **Reporting the LLRB height as equal to the 2-3 tree height.** Each 3-node on the path adds a red link.
10. **Thinking you build a 2-3 tree and then convert it.** You never materialize the 2-3 tree. It exists only in your head as the correctness argument.
11. **Applying the three insert rules in the wrong order, or only at the insertion point.** They are applied bottom-up at every node on the path back to the root, in the order left-lean, double-left, flip. Skipping the upward pass misses cascades.
12. **Panicking at a momentary right-leaning red link or a node with two red links.** Those are the legal *temporary* representations of a 4-node mid-insert; they just must not survive the end of the operation.
13. **Thinking a rotation can fix a color-flip situation (or vice versa).** Task 4's fix uses **no** rotation: a color flip changes colors only, never shape. Conversely, rotations do not add or remove items from 2-3 nodes.
14. **Confusing `java.util.TreeMap` with an LLRB.** `TreeMap` is a red-black tree that is *not* left-leaning, corresponds to a **2-3-4** tree, and is *not* a 1-1 correspondence (glue links on either side are allowed).
15. **Claiming rotation-based rebalancing is cheap in general.** Rebalancing an arbitrary BST from scratch takes O(N) rotations. The LLRB's achievement is doing O(log N) repair work *per insertion* so you never need the O(N) pass.
16. **Forgetting `root.color = BLACK` after insert.** The root has no parent, so its incoming link cannot be a glue link. (This is also what quietly turns "the root became a temporary red node" into "the tree grew one level," matching how 2-3 trees grow at the root.)
17. **Assuming delete works symmetrically.** The lecture explicitly says LLRB delete is not covered; "LLRBs insertion is simple to implement (but delete is hard)."

---

## Likely Exam Points

### 1. Execute a given rotation

**Q.** Given the tree below, draw the result of `rotateLeft(6)`.

```
      6
    /   \
   1     8
    \     \
     4     13
          /  \
         9    17
```

**A.** `6`'s right child is `8`, so `8` rises to become the root and `6` becomes `8`'s left child. `8`'s old left child is null, so `6`'s right child becomes null.

```
        8
      /   \
     6     13
    /     /  \
   1     9    17
    \
     4
```

### 2. Identify the new parent / direction of movement

**Q.** If you `rotateLeft(x)`, which direction does `x` move and who becomes its parent?

**A.** `x` moves down and to the **left**, and its new parent is **`x`'s old right child**. (Mirror: `rotateRight(x)` sends `x` down and right, and its new parent is `x`'s old left child.)

### 3. Give a rotation sequence that balances a tree

**Q.** Give a sequence of rotations that balances this tree.

```
   1
    \
     3
    /
   2
```

**A.** `rotateRight(3)` giving `1 -> 2 -> 3` (all right links), then `rotateLeft(1)` giving root `2` with children `1` and `3`. Other correct sequences exist. Also know the fact: rotation preserves the search tree property, so any such sequence is "safe" by construction, and any BST can be balanced in O(N) rotations.

### 4. Convert a 2-3 tree to an LLRB (and back)

**Q.** Draw the LLRB corresponding to the 2-3 tree with root 3-node `[u w]` and children `[a s]`, `[v]`, `[x y]`.

**A.** Each 3-node `p q` becomes `q` with red-left child `p`:

```
         w
       */  \
      u     y
     / \  */
    s   v x
  */
  a
```

Reverse direction: contract every red link, merging the red child into its parent as the smaller item of a 2-item node.

### 5. Validity of a candidate LLRB

**Q.** Is the following a valid LLRB? `G` is the root; `G.left = B` via a **black** link; `B.left = A` via a **red** link; `B.right = C` via a black link; `G.right = X` via a black link.

**A.** **No.** Contract the red link: `A` and `B` form the 2-3 node `[A B]`, whose only child is `C`. A 3-node must have 3 children (or be a leaf), so the 2-3 tree is illegal. Equivalently, black-link counts to nulls differ: the path `G -> B -> A -> null` has 3 black links while `G -> X -> null` has 2.

### 6. LLRB height versus 2-3 tree height

**Q.** A 2-3 tree has height `H` (measured in links). What is the maximum possible height of the corresponding LLRB, and why?

**A.** **`2H + 1`**. A root-to-leaf path has `H` black links (one per 2-3 tree link) and up to `H + 1` red links (one per 3-node among the `H + 1` nodes on the path). Hence `H + (H + 1) = 2H + 1`, i.e. never more than about 2x. Since 2-3 trees are O(log N) tall, LLRBs are too.

Follow-up often asked: for the lecture's height-3 2-3 tree, the LLRB had height `3 + 2 = 5`.

### 7. Which repair rule applies?

**Q.** Name the violation and the fix for each situation: (a) a node `h` whose right link is red and whose left link is black; (b) a red node whose left child is also red; (c) a node with two red child links.

**A.**
(a) **Left Leaning Violation** -> `rotateLeft(h)`.
(b) **Incorrect 4 Node Violation** -> `rotateRight` the top node of the chain.
(c) **Temporary 4 Node** -> `colorFlip` that node (emulates 2-3 `split`); **no rotation**, and the shape is unchanged.

### 8. Full insertion trace with cascading

**Q.** Starting from the LLRB `B` (root, black) with black-left child `A` and black-right child `S`, where `S` has a red-left child `E`, insert `Z`. Show every step.

**A.**
1. `Z` is inserted as a **red right** child of `S`. Now `S` has red children on both sides: a temporary 4-node `[E S Z]`.
2. Rule 3 applies: `flip(S)`. The link `B -> S` becomes red; `S -> E` and `S -> Z` become black. 2-3 world: `split(E/S/Z)` promotes `S` into `B`, giving the 3-node `[B S]` over `A`, `E`, `Z`.
3. The cascade: at `B`, the red link now leans **right**, a Left Leaning Violation. Rule 1 applies: `rotateLeft(B)`.
4. Final tree: root `S`, red-left child `B` (whose children are `A` and `E`), right child `Z`. This is the correct LLRB drawing of `[B S]` over `A`, `E`, `Z`.

### 9. Small insert trace from scratch

**Q.** Insert `1`, `2`, `3` in that order into an empty LLRB. What is the final tree?

**A.** `2` as a black root with black children `1` and `3` (perfectly balanced). Steps: `1` becomes a black root; inserting `2` creates a red right link, fixed by `rotateLeft(1)` to give `2` with red-left `1`; inserting `3` creates red children on both sides of `2`, fixed by `flipColors(2)`, then `put` resets the root to black.

### 10. Runtimes

**Q.** State the worst-case runtimes of `contains` and `insert` on an LLRB, and justify them.

**A.** Both **Θ(log N)**. Height is O(log N) by the 2-3 correspondence plus the `2H + 1` bound, so `contains` is O(log N) BST search. `insert` is O(log N) to descend and create the node, plus O(1) rotation/flip work at each of O(log N) levels on the way back up, so O(log N) total.

### 11. New-link color, and why

**Q.** When inserting into an LLRB, should the new link be red or black? Why?

**A.** **Red.** In a 2-3 tree a new item always joins an existing leaf node first (2-node becomes 3-node, 3-node becomes a temporary 4-node); the tree does not immediately get taller. A red (glue) link is exactly the LLRB encoding of "this item joined an existing node." A black link would falsely claim a new 2-3 tree level.

### 12. Conceptual true/false

**Q.** True or false, with justification: (a) Rotation changes the set of keys in a BST. (b) A color flip can change the shape of an LLRB. (c) Every LLRB is a BST. (d) Java's `TreeMap` is an LLRB. (e) An LLRB is always perfectly balanced.

**A.** (a) **False**, rotation preserves keys and in-order order. (b) **False**, a flip only recolors, which is why Task 4 needs no rotation. (c) **True**, LLRBs *are* normal BSTs; colors are a fiction layered on top. (d) **False**, `TreeMap` is a red-black tree that is **not** left-leaning and corresponds to a 2-3-4 tree, not 1-1. (e) **False**, it is within a constant factor of balanced (height at most `2H + 1`), which is enough for O(log N).

### 13. Counting / combinatorics facts from the slides

**Q.** How many distinct BSTs are there on N items, and what bound did the lecture give on rotation distance between two BST configurations?

**A.** **Catalan(N)** distinct BSTs (the particular one you get depends on insertion order), and you can get from any configuration to any other in **`2n - 6`** rotations.

### 14. Why not just implement 2-3 trees?

**Q.** List the implementation difficulties of 2-3 trees that motivate LLRBs.

**A.** Maintaining different node types; interconverting between 2-nodes and 3-nodes (and temporary 4-nodes); walking back up the tree to split nodes. Plus performance problems in practice. LLRBs get the same asymptotics with a single node type and a short insert method.

---

## Summary

- 2-3 / B-trees give guaranteed balance but are ugly to implement: multiple node types, node interconversion, upward splitting. ("Beautiful algorithms are, unfortunately, not always the most useful.")
- **Tree rotation** is O(1) pointer surgery on a BST that changes shape while preserving the search tree property and in-order order. `rotateLeft(G)`: `G`'s right child rises, `G` becomes its left child, and the riser's old left subtree becomes `G`'s new right subtree. `rotateRight` is the mirror and the inverse.
- `rotateLeft(x)` sends `x` **down and left**; `x`'s new parent is its old **right** child. A rotation is invalid if the needed child does not exist. Rotations can raise or lower height.
- There are Catalan(N) BST shapes on N keys; rotations can reach any shape from any other (within `2n - 6` moves), and can rebalance any BST in O(N) moves, which is too slow to do per-operation.
- **Big idea:** maintain a 1-1 correspondence (isometry) between the BST and a 2-3 tree. Then balance is inherited for free from the 2-3 tree.
- 2-nodes map directly; a **3-node `p q`** becomes BST node `q` with a left **glue link** to `p`, drawn **red**. Glue nodes were rejected as inelegant and wasteful.
- The result is a **Left-Leaning Red-Black BST (LLRB)**: a perfectly normal BST. Red is a fiction; search is plain BST search.
- **Invariants:** no node has two red links; red links lean left; every root-to-**null** path has the same number of black links (perfect black balance). Root-to-**leaf** is the wrong invariant.
- **Height:** each 3-node adds one red link, so a 2-3 tree of height `H` yields an LLRB of height at most `H + (H + 1) = 2H + 1`, never more than about 2x. Therefore LLRBs are O(log N) tall.
- **Insertion:** insert as a normal BST with a **red** link, then on the way back up apply, in order: `rotateLeft` on a right-leaning red link (Left Leaning Violation); `rotateRight` on two consecutive left red links (Incorrect 4 Node Violation); `colorFlip` on a node with two red children (Temporary 4 Node, emulating `split`). Flips change colors only, never shape.
- Repairs can **cascade** upward, exactly like 2-3 splits; the recursive "return the new subtree root" idiom handles this automatically, and the root is forced black at the end.
- **Runtime:** `contains` O(log N), `insert` O(log N) (O(log N) descent plus O(log N) rotations/flips). Delete is not covered in this course.
- Inserting `7, 6, 5, 4, 3, 2, 1` into an empty LLRB, which would produce a degenerate chain in a naive BST, yields a perfectly balanced tree.
- **Landscape:** BSTs are simple but imbalance-prone; 2-3 trees are balanced but painful; LLRBs are easy to insert into and balanced via the 2-3 bijection; Java's `TreeMap` is a non-left-leaning red-black tree in (non-1-1) correspondence with a 2-3-4 tree, more complex but faster. Also out there: AVL, WAVL, splay trees, treaps, skip lists, and hashing (coming later).
