<!-- Wed, Oct 14, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 21: Tree and Graph Traversals

## Overview

This lecture formalizes two of the most important abstractions in the course and then connects them. It starts by pinning down the definition of a tree (a set of nodes plus a set of edges such that there is exactly one path between any two nodes) and observes that trees show up everywhere: inside data structures we have already built (BSTs, heaps, tries, disjoint sets) and outside of them (org charts, family lineages, phylogenetic trees, field manuals). Because a tree has structure in more than one dimension, "iterating" over it is not one thing: there are several natural orders, called traversals, and the lecture presents four of them (level order, plus the three depth first orders: preorder, inorder, postorder), shows the tiny recursive code for each, and gives real uses (inorder to print a BST in sorted order, preorder to print an indented directory listing, postorder to total up file sizes, since a folder's size depends on its children's sizes being known first). The lecture then relaxes the tree constraint to get graphs (a set of nodes plus zero or more edges, each connecting two nodes), introduces graph vocabulary (directed/undirected, cyclic/acyclic, path, simple path, cycle, adjacent, connected) and a catalogue of famous graph problems whose difficulties are wildly different (Euler tour is solvable in time proportional to the number of edges and has been since 1873; Hamilton tour has no known efficient algorithm). Finally, it motivates graph traversal with the s-t connectivity problem: the obvious recursive algorithm loops forever on a cyclic graph, and the fix (mark each vertex when you first reach it, and only recurse into unmarked neighbors) *is* depth first search. The same skeleton with one extra array (`edgeTo`) becomes `DepthFirstPaths`, which records a path from the source to every reachable vertex, and the lecture closes with the observation that graphs have traversal orders too (DFS preorder, DFS postorder, BFS) and a challenge to invent BFS to get *shortest* paths.

---

## Key Concepts

### 1. Trees, defined precisely

A tree is:

- a set of **nodes**, and
- a set of **edges** connecting those nodes,
- with the constraint that **there is exactly one path between any two nodes**.

That one constraint is secretly doing two jobs, which the lecture emphasized:

1. **At least one** path between any two nodes means the structure is **connected** (no detached islands).
2. **At most one** path between any two nodes means there are **no cycles** (no two different ways to get from A to B).

So a structure fails to be a tree if it has a cycle (two routes between some pair of nodes) *or* if it is disconnected (zero routes between some pair). Both failure modes appeared in the lecture's green/pink picture: pink structures were either cyclic or disconnected.

A **rooted tree** is a tree where we have *chosen* one node to be the root. Josh's intuition: any node of any tree can be the root, and once you pick it, you "pick the tree up by that node and everything dangles below it." Once rooted, every node except the root has exactly one **parent** (the first node on the path from that node to the root), and a node with no children is a **leaf**. Note the consequence: the same underlying tree has many different rooted versions, and preorder/inorder/postorder are defined only *after* you choose a root (and, for inorder, an ordering of children).

### 2. Traversal is the tree analogue of iteration, but there is more than one of them

For a list there are only two natural iteration orders, front-to-back and back-to-front. For a tree there are roughly four natural orders (plus their reverses). What one might call "tree iteration" is conventionally called **tree traversal**, and each order is useful for different tasks.

The lecture's motivating example: you have a folder `61b` containing `proj0` (which contains `audio`, and `data`, which contains `planets.txt` at 851 bytes) and `hw1` (which contains `synthesizer`, `spec` with `hw1.md` at 23433 bytes and `karplus-strong.png` at 16180 bytes, and `GuitarHeroLite.java` at 1251 bytes). If you want the total size of all files under `61b`, you must walk the whole tree, and the *order* in which you walk it determines whether the computation is natural or awkward.

### 3. "Traversing" a node vs "visiting" a node

This distinction is subtle but it is the whole point of having three depth first orders. In all three, the recursion **traverses** the same nodes in the same shape; what differs is *when* you **visit** (that is, take the action: print the key, add a file size, set `edgeTo`). The code for all three is identical except for the position of one line:

```java
// Preorder: visit, then traverse children
preOrder(BSTNode x) {
    if (x == null) return;
    print(x.key);
    preOrder(x.left);
    preOrder(x.right);
}

// Inorder: traverse left, visit, traverse right
inOrder(BSTNode x) {
    if (x == null) return;
    inOrder(x.left);
    print(x.key);
    inOrder(x.right);
}

// Postorder: traverse left, traverse right, then visit
postOrder(BSTNode x) {
    if (x == null) return;
    postOrder(x.left);
    postOrder(x.right);
    print(x.key);
}
```

The lecture's running tree (a BST):

```
            D
          /   \
         B     F
        / \   / \
       A   C E   G
```

- **Level order**: `DBFACEG` (top to bottom, left to right, "like reading in English")
- **Preorder**: `DBACFEG`
- **Inorder**: `ABCDEFG`
- **Postorder**: `ACBEGFD`

Two things to notice. First, the inorder traversal of a BST prints the keys in sorted order, which is why it is called "inorder." Second, postorder is *not* the reverse of preorder (`DBACFEG` reversed is `GEFCABD`, which was a wrong answer choice on the poll; the correct postorder is `ACBEGFD`).

### 4. The depth first intuition, and the human visual trick

The rough idea shared by all three depth first traversals: traverse **deep** nodes (like `A`) before **shallow** ones (like `F`). Level order does the opposite, going level by level.

For humans doing this by hand (not an algorithm, just a trick), trace a loop around the drawing of the tree starting at the top and going counterclockwise, hugging the nodes:

- **Preorder**: visit each node when you pass its **left** side.
- **Inorder**: visit each node when you cross its **bottom**.
- **Postorder**: visit each node when you cross its **right** side.

The postorder case makes intuitive sense: by the time your pencil reaches a node's right side, you have already been all the way around both of its subtrees. The lecture demonstrated this on a nine node numbered tree and the class called out the numbers as the pencil crossed each right side. (The slide text gives the resulting string as `478529631`; the corresponding diagram did not survive the PDF extraction, so do not over-trust that exact string, but do trust the trick.)

### 5. Why each traversal is useful

- **Inorder**: build a `TreeSet`/`TreeMap` on a BST, then print its contents in sorted order. "If I gave you data and you used a BST to build out a bunch of stuff into a tree, this is the code you use to print out the items in order."
- **Preorder**: print an indented directory listing. Visit (print the name) *before* recursing, and add one tab of indentation per level of depth, and you get exactly the familiar nested listing (`sc2APM`, then indented `directOverlay`, then further indented `directIO`, then `DXHookD3D11.cs`). The action must come first, because a folder's name must be printed before its contents.
- **Postorder**: total up file sizes. A folder's size is the sum of its children's sizes plus its own, so you *cannot* act on a folder until all of its children have been fully processed. That is precisely "visit after the recursive calls."
- **Level order**: natural "reading" order; the graph analogue of level order is BFS, which is how you get shortest paths.

### 6. From trees to graphs: dropping the constraint

Trees are excellent for **strict hierarchy**, but not every relationship is hierarchical. The Paris Metro map was the counterexample: no station is "the boss of" another station, and a natural consequence is that the map **contains cycles** (more than one way to get from A to B).

A **graph** is:

- a set of nodes (also called **vertices**), and
- a set of **zero or more** edges, each of which connects two nodes.

That is the tree definition with the "exactly one path" constraint deleted. Important consequence: **every tree is a graph**, but not every graph is a tree. A graph is a tree exactly when it is connected and acyclic.

Example from lecture: the pre-pandemic BART map is *not* a tree, because it contains exactly one cycle (San Bruno, SFO, Millbrae). Remove that and it would be a tree.

### 7. Simple graphs: the 61B default

A **simple graph** has:

- no edge connecting a vertex to itself (no self loops, no "length 1 loops"), and
- no two edges connecting the same pair of vertices (no parallel edges).

In 61B, unless explicitly stated otherwise, **"graph" means "simple graph."**

### 8. Graph flavors and vocabulary

Graphs can be **directed** (edges have a direction, good for one-way streets or "is-a-type-of" relationships) or **undirected** (symmetric, "you and I are friends"); **cyclic** (there is at least one cycle) or **acyclic**; and vertices or edges may carry **labels** or **weights**.

Characterizing the lecture's two real examples:

- **Paris Metro**: undirected, connected, cyclic (so not a tree), vertex-labeled (each station has a color).
- **WordNet** (used in project 4): directed, and not a tree, because there are two distinct paths from `group_action` to `event`. Edges capture "is-a-type-of" (descent is-a-type-of movement, movement is-a-type-of change, and so on).

### 9. Graph problems look alike and are not alike

A sampler of standard named problems:

| Problem | Question |
|---|---|
| s-t Path | Is there a path between vertices s and t? |
| Connectivity | Is the graph connected (a path between all pairs)? |
| Biconnectivity | Is there a vertex whose removal disconnects the graph? |
| Shortest s-t Path | What is the shortest path between s and t? |
| Cycle Detection | Does the graph contain any cycles? |
| Euler Tour | Is there a cycle that uses every **edge** exactly once? |
| Hamilton Tour | Is there a cycle that uses every **vertex** exactly once? |
| Planarity | Can the graph be drawn with no crossing edges? |
| Isomorphism | Are two graphs the same graph in disguise? |

Euler tour and Hamilton tour are one word apart, yet: an efficient Euler tour algorithm running in time proportional to the number of edges has been known since **1873** and is covered in CS70, while despite decades of intense study **no efficient Hamilton tour algorithm is known** and the best known algorithms take exponential time. Josh's framing: an efficient Hamilton tour algorithm would be a math result "of unbelievable magnitude." The lesson for you: **you often cannot tell how hard a graph problem is without deep consideration**, so do not assume a plausible-sounding graph problem is easy.

The common tool for attacking almost all of these is **traversal**.

### 10. s-t connectivity, the infinite loop, and the fix

Problem: given a source `s` and target `t`, is there a path between them?

First attempt, a natural recursion:

> `connected(s, t)`:
> - Does `s == t`? If so, return `true`.
> - Otherwise, check `connected(v, t)` for all neighbors `v` of `s`. If any is true, return `true`.
> - Return `false`.

This is **wrong**: it can loop forever. On the lecture's graph, `connected(0, 7)` asks `connected(1, 7)`, which (because `0` is a neighbor of `1`) asks `connected(0, 7)`, which asks `connected(1, 7)`, forever. "I'm not seven, are you? No, I'm not seven, are you?"

The fix, suggested from the audience, is to leave breadcrumbs:

> `connected(s, t)`:
> - **Mark `s`.**
> - Does `s == t`? If so, return `true`.
> - Otherwise, if `connected(v, t)` for any **unmarked** neighbor `v` of `s`, return `true`.
> - Return `false`.

Same idea as before, but each vertex is visited at most once, so the recursion must terminate. Marking is what makes graph traversal work at all, and it is the single biggest structural difference from tree traversal.

### 11. Depth first search

The pattern "explore a neighbor's **entire** subgraph before moving on to the next neighbor" is **depth first traversal**, or **depth first search (DFS)**. Called depth first because you go as deep as possible before backing up. On the lecture's graph, DFS from `0` explores all of `1`'s subgraph before ever using another edge out of `0`.

A clarification the lecture made explicitly: it is entirely possible for a neighbor's "subgraph" to loop back around and include a vertex that the source is also directly adjacent to. That does not make the traversal any less depth first, because the direct edge from the source is still not used until the first subgraph has been fully explored.

The lecture's intuition pump was xkcd 761: you are getting ready for a date, you branch into "medical emergency," then "snake bite," then "which snakes are most dangerous," then "the venom research is inconsistent, I'll build a spreadsheet," and your date arrives while you are still deep in one subtree and have not touched the other branches. That is DFS, including its pathology.

### 12. DFS is a skeleton, not a single algorithm

`connected` answers a yes/no question. The same skeleton plus one array answers a much richer question: **find a path from `s` to every reachable vertex, visiting each vertex at most once.** That algorithm is `DepthFirstPaths`, and the only change is that whenever you discover a new vertex `w` from vertex `v`, you record `edgeTo[w] = v`, meaning "the way you get to `w` is through me." At the end you hold two arrays: `marked`, which tells you everything reachable, and `edgeTo`, which is a compressed encoding of a path to every reachable vertex (follow `edgeTo` backwards from any vertex until you reach `s`).

This is the lecture's big takeaway about traversals: **by performing actions or setting instance variables during a traversal, you solve problems.** The traversal is the chassis; the action is the algorithm.

One subtlety worth internalizing from the `connected` demo: the recursive call `connected(3, 7)` returned `false` even though `3` and `7` really are connected in the graph. That is not a bug. The meaning of the recursive return value is relative to the current `marked` array: "within the still-unexplored part of the graph reachable from me, is `t` there?" Only the top-level return value answers the original question.

### 13. Graph traversal orders

Just as trees have preorder, inorder, postorder, and level order, graphs have multiple traversal orders **relative to a given source**:

- **DFS preorder**: take the action *before* the recursive DFS calls to neighbors. Equivalent to the order of `dfs` **calls**. For the lecture's graph from source `0`: `012543678`.
- **DFS postorder**: take the action *after* the recursive calls. Equivalent to the order of `dfs` **returns**. For the same graph: `347685210`.
- **BFS order**: act in order of **distance from `s`**. Analogous to level order; "search is wide, not deep." For the same graph: `0`, `1`, `2`/`4`, `5`/`3`, `6`/`8`, `7`, written on the slide as `0 1 24 53 68 7`.

Note there is no graph analogue of *inorder*, because a general graph vertex does not have a "left half" and a "right half" of its neighbors.

Also note the phrasing "**one valid** DFS preorder for this graph." Which DFS order you get depends on the order in which you iterate over each vertex's neighbors, so a graph has many valid DFS preorders.

### 14. The closing challenge: invent BFS

Given a graph with source `s`, find the **length of the shortest path** from `s` to every other vertex. Hints given: you need to visit vertices in BFS order, and you will need some kind of data structure. Solution next lecture. (extra context: the data structure is a **queue**, and the reason DFS cannot be patched to do this is visible right in the demo: `DepthFirstPaths` records `edgeTo[4] = 5`, giving the path `0 -> 1 -> 2 -> 5 -> 4`, even though `0 -> 1 -> 4` exists and is shorter.)

---

## Definitions

- **Tree**: A set of nodes together with a set of edges connecting those nodes, subject to the constraint that there is exactly one path between any two nodes. Equivalently (extra context): connected and acyclic.
- **Rooted tree**: A tree in which one node has been designated the root.
- **Root**: The designated node of a rooted tree; the only node with no parent.
- **Parent (of node N)**: In a rooted tree, the first node on the path from N to the root. Every node except the root has exactly one parent.
- **Child**: The inverse relation of parent.
- **Leaf**: A node with no children.
- **Tree traversal**: An order in which the nodes of a tree are visited; the tree analogue of iteration over a list.
- **Traverse (a node)**: To have the recursion reach/handle that node. Distinct from visiting it.
- **Visit (a node)**: To take the action associated with the traversal at that node (printing its key, adding its file size, setting `edgeTo`, and so on).
- **Level order traversal**: Visit nodes top to bottom, left to right, like reading English. For the lecture's tree: `DBFACEG`.
- **Depth first traversal**: A traversal that traverses deep nodes before shallow ones. Three types: preorder, inorder, postorder.
- **Preorder traversal**: Visit a node, then traverse its children. `DBACFEG`.
- **Inorder traversal**: Traverse the left child, visit the node, then traverse the right child. `ABCDEFG`. On a BST this yields sorted order.
- **Postorder traversal**: Traverse the left child, traverse the right child, then visit the node. `ACBEGFD`.
- **Graph**: A set of nodes plus a set of zero or more edges, each of which connects two nodes.
- **Vertex (plural vertices)**: A node of a graph.
- **Edge**: A pair of vertices, representing a connection between them.
- **Adjacent**: Two vertices are adjacent if there is an edge between them.
- **Simple graph**: A graph with no self loops (no edge from a vertex to itself) and no parallel edges (no two edges connecting the same pair of vertices). In 61B, "graph" means "simple graph" unless stated otherwise.
- **Directed graph**: A graph whose edges have a direction (edge from a to b is not the same as b to a).
- **Undirected graph**: A graph whose edges have no direction.
- **Label / weight**: Optional extra data attached to a vertex or an edge.
- **Path**: A sequence of vertices connected by edges.
- **Simple path**: A path with no repeated vertices.
- **Cycle**: A path whose first and last vertices are the same.
- **Cyclic graph**: A graph containing at least one cycle. One cycle is enough.
- **Acyclic graph**: A graph with no cycles.
- **Connected (two vertices)**: There exists a path between them.
- **Connected graph**: Every pair of vertices is connected.
- **Connected component**: A collection of vertices that are all connected to one another (mentioned in lecture discussion, not on the slides).
- **s-t connectivity problem**: Given a source vertex `s` and a target vertex `t`, is there a path between `s` and `t`?
- **Shortest s-t path problem**: What is the shortest path between `s` and `t`?
- **Biconnectivity**: Is there a vertex whose removal disconnects the graph?
- **Cycle detection**: Does the graph contain any cycles?
- **Euler tour**: A cycle that uses every **edge** exactly once. Efficiently findable (time proportional to the number of edges, known since 1873).
- **Hamilton tour**: A cycle that uses every **vertex** exactly once. No known efficient algorithm; best known are exponential time.
- **Planarity**: Can the graph be drawn on paper with no crossing edges?
- **Isomorphism**: Are two graphs the same graph in disguise?
- **Depth first search / depth first traversal (DFS)**: A graph traversal that explores a neighbor's entire subgraph before moving on to the next neighbor, using marking to avoid revisiting vertices.
- **Marking**: Recording that a vertex has already been reached, so it is never visited twice. This is what prevents infinite recursion on cyclic graphs.
- **DFS preorder**: Graph traversal order in which the action happens before the DFS calls to neighbors; equivalently, the order of `dfs` calls.
- **DFS postorder**: Graph traversal order in which the action happens after the DFS calls to neighbors; equivalently, the order of `dfs` returns.
- **BFS (breadth first search) order**: Graph traversal order by increasing distance from the source; the graph analogue of level order.
- **`marked[]`**: An array of booleans, initially all `false`, where `marked[v]` is `true` once `v` has been reached.
- **`edgeTo[]`**: An array of ints where `edgeTo[w]` is the vertex from which `w` was discovered, that is, the second-to-last vertex on the recorded path from the source to `w`.

---

## Worked Examples

### Example 1: Preorder traversal, traced with the call stack

```java
public class BST<Key extends Comparable<Key>> {
    private class BSTNode {
        Key key;
        BSTNode left, right;
    }

    private BSTNode root;

    public void printPreOrder() {
        preOrder(root);
    }

    private void preOrder(BSTNode x) {
        if (x == null) return;
        System.out.print(x.key);   // visit
        preOrder(x.left);          // traverse left
        preOrder(x.right);         // traverse right
    }
}
```

Tree:

```
            D
          /   \
         B     F
        / \   / \
       A   C E   G
```

Step by step on `preOrder(D)`, following the lecture's call stack animation:

1. Call stack: `preOrder(D)`. `x` is not null, so print `D`. Output: `D`.
2. Execute `preOrder(x.left)`, which pushes a new frame `preOrder(B)`. Call stack: `preOrder(D) -> preOrder(B)`.
3. In `preOrder(B)`: not null, print `B`. Output: `DB`.
4. `preOrder(B)` calls `preOrder(A)`. Call stack: `preOrder(D) -> preOrder(B) -> preOrder(A)`. Print `A`. Output: `DBA`.
5. `preOrder(A)` calls `preOrder(null)` for its left child, which immediately returns at the base case (the lecture skipped drawing these for brevity). Then `preOrder(null)` for its right child, which also returns. `preOrder(A)`'s body is finished, so its frame pops and control returns to whoever called it, which is `preOrder(B)`.
6. Back in `preOrder(B)`, the next statement is `preOrder(x.right)`, so `preOrder(C)` is pushed. It prints `C` and recurses twice into `null`. Output: `DBAC`. `preOrder(C)` pops back to `preOrder(B)`, whose body is now done, so it pops back to `preOrder(D)`.
7. Back in `preOrder(D)`, the next statement is `preOrder(x.right)`, so `preOrder(F)` is pushed. Print `F`. Output: `DBACF`.
8. `preOrder(F)` calls `preOrder(E)` (prints `E`), then `preOrder(G)` (prints `G`). Output: `DBACFEG`.
9. All frames pop. Final output: `DBACFEG`.

**Environment/box-and-pointer reasoning in words.** Each frame on the call stack has its own local `x`, holding a *reference* (an arrow) to one node object on the heap. There is exactly one `D` node object, one `B` node object, and so on; the recursion does not copy nodes, it only copies arrows. When `preOrder(B)` executes `preOrder(x.left)`, Java evaluates `x.left` (following `B`'s left arrow to the `A` object) and passes that arrow as the new frame's `x`. The null check at the top is doing the work of noticing that a node's `left` or `right` arrow points at nothing, which is the base case that terminates every branch of the recursion. Nothing is shared or mutated here, which is why pure traversal is so simple; contrast this with Example 4, where frames share mutable arrays.

Why this is "preorder": the `print` statement is positioned before both recursive calls, so a node is visited the first time the recursion arrives at it.

### Example 2: Inorder and postorder, same code with the visit moved

```java
private void inOrder(BSTNode x) {
    if (x == null) return;
    inOrder(x.left);
    System.out.print(x.key);
    inOrder(x.right);
}

private void postOrder(BSTNode x) {
    if (x == null) return;
    postOrder(x.left);
    postOrder(x.right);
    System.out.print(x.key);
}
```

**Inorder trace (narrated as in lecture).** Start at `D`. `D` is not null, but we do not print yet, because we have not traversed `D`'s left subtree. Go left to `B`. Same story, do not print. Go left to `A`. Go left from `A`, hit `null`, return immediately. Now `A`'s left is done, so print `A`. Go right from `A`, `null`, return. Pop back to `B`: `B`'s left subtree is finished, so print `B`. Now go right to `C`: trivially finish `C`'s left, print `C`, trivially finish `C`'s right, return. Pop back to `B`, done, pop to `D`: `D`'s entire left subtree has been traversed (`A`, `B`, `C` all printed), so print `D`. Then recurse right into `F`, which by the same pattern prints `E`, then `F`, then `G`. Output: `ABCDEFG`, which is sorted, which is why it is called *inorder*.

**Postorder reasoning (the poll question).** The key question at each node is: "have I finished *both* of my children?" `A` is the first node for which both children (both `null`) are finished, so `A` prints first. After `A` we return to `B`, but `B` cannot print, because `C` has not been handled. So `C` prints second. Now `B`'s children are both done, so `B` prints third. Partial output: `ACB`. Symmetrically on the right: `E`, then `G`, then `F`, giving `ACBEGF`. Finally `D`, whose two children are now complete. Full output: `ACBEGFD`.

Poll options and answer: the correct choice was **4. ACBEGFD**. Option 2 (`GFEDCBA`) and option 3 (`GEFCABD`, which is preorder reversed) are the classic traps.

### Example 3: Postorder traversal to total up file sizes

The lecture's code (slide pseudocode, lightly cleaned up; note it uses the name `BSTNode` even though these nodes have an arbitrary number of children, and `postOrder` here returns an `int` rather than printing):

```java
int postOrder(BSTNode x) {
    if (x == null) return 0;
    int total = 0;
    for (BSTNode c : x.children()) {
        total += postOrder(c);
    }
    total += x.fileSize();
    return total;
}
```

**What it does and why postorder is forced.** The "visit" action here is not a print, it is the line `total += x.fileSize()` together with returning `total` to the caller. That action must come *after* the loop over children, because a folder's total is defined in terms of its children's totals. The structural requirement of the problem dictates the traversal order: you physically cannot know a folder's size until every descendant has reported in. This is the general signature of a postorder problem: the value at a node is an aggregate of the values at its children.

Trace on the lecture's `sc2APM` directory tree (a real tool Josh wrote to monitor an opponent's actions-per-minute in StarCraft 2):

```
sc2APM
├── directOverlay
│   ├── directIO
│   │   ├── DXHookD3D11.cs   18381
│   │   └── Injector.cs       8798
│   ├── directO.suo            881
│   └── directO.sln            874
├── notes
└── python
    └── printAPM.py
```

1. The recursion descends to the deepest interesting node first. Leaf files are trivial: `postOrder(DXHookD3D11.cs)` loops over zero children, adds `18381`, returns `18381`. Similarly `Injector.cs` returns `8798`.
2. `directIO` is the first *interesting* node to finish, because both of its children have reported. Its `total` is `18381 + 8798 = 27179`, plus its own `fileSize()` (0 for a folder), so it returns `27179`. The lecture explicitly called out `27179` appearing on `directIO` at this point.
3. `directOverlay` cannot finish until `directIO`, `directO.suo`, and `directO.sln` have all returned. Once they have, it sums them with its own contribution and returns `66972`.
4. `notes` returns `324`; `python` returns `874`.
5. At the root, `sc2APM` sums its three children: `66972 + 324 + 874 = 68170`, the grand total shown on the slide.

(The extracted slide text garbles a couple of the intermediate per-file numbers inside `directOverlay`, so the exact split of `66972` among its children is not fully recoverable from the provided material; the root total `68170 = 66972 + 324 + 874` and the `directIO` subtotal `27179 = 18381 + 8798` are solid.)

Contrast with the preorder version of the same walk: if instead of summing you print `x.name()` with one tab per level *before* recursing, you get the indented directory listing (`sc2APM`, then `directOverlay`, then `directIO`, then `DXHookD3D11.cs`, then `Injector.cs`, then `directO.suo`, ...). Same traversal shape, different placement of the action, completely different application.

### Example 4: s-t connectivity, broken version then fixed version

The lecture's graph (adjacency recovered from the demo steps, with neighbors considered in increasing numeric order):

```
vertex : neighbors
  0    : 1
  1    : 0, 2, 4
  2    : 1, 5
  3    : 4
  4    : 1, 3, 5
  5    : 2, 4, 6, 8
  6    : 5, 7
  7    : 6
  8    : 5
```

Broken version, in Java (extra context: the `Graph` class with `G.V()` and `G.adj(v)` is the API used in later lectures; the lecture itself stated this algorithm in English):

```java
// WRONG: infinite recursion on any graph with a cycle,
// and even on a single undirected edge.
public boolean connectedBroken(Graph G, int s, int t) {
    if (s == t) return true;
    for (int v : G.adj(s)) {
        if (connectedBroken(G, v, t)) return true;
    }
    return false;
}
```

Why it breaks, exactly as traced in lecture for `connectedBroken(G, 0, 7)`:

- `connectedBroken(0, 7)`: `0 != 7`, so recurse on neighbor `1`.
- `connectedBroken(1, 7)`: `1 != 7`, and `0` is a neighbor of `1`, so recurse on `0`.
- `connectedBroken(0, 7)` again. The stack grows forever and you get a `StackOverflowError`.

Note that this does not even require a cycle in the drawing: in an undirected graph, a single edge `0-1` already provides the two-step loop, because each endpoint is a neighbor of the other.

Fixed version, which is DFS:

```java
public class Connectivity {
    private Graph G;
    private boolean[] marked;

    public Connectivity(Graph G) {
        this.G = G;
        marked = new boolean[G.V()];   // all false initially
    }

    public boolean connected(int s, int t) {
        marked[s] = true;
        if (s == t) return true;
        for (int v : G.adj(s)) {
            if (!marked[v]) {
                if (connected(v, t)) return true;
            }
        }
        return false;
    }
}
```

Full trace of `connected(0, 7)`, following the lecture's slide-by-slide demo (`*` = currently executing vertex):

| Call stack | What happens |
|---|---|
| `0` | `mark(0)`. `0 == 7`? No. `isMarked(1)`? No, so recurse. |
| `0 -> 1` | `mark(1)`. `1 == 7`? No. `isMarked(0)`? Yes, skip. `isMarked(2)`? No, recurse. |
| `0 -> 1 -> 2` | `mark(2)`. `2 == 7`? No. `isMarked(1)`? Yes, skip. `isMarked(5)`? No, recurse. |
| `0 -> 1 -> 2 -> 5` | `mark(5)`. `5 == 7`? No. `isMarked(2)`? Yes. `isMarked(4)`? No, recurse. |
| `0 -> 1 -> 2 -> 5 -> 4` | `mark(4)`. `4 == 7`? No. `isMarked(1)`? Yes. `isMarked(3)`? No, recurse. |
| `0 -> 1 -> 2 -> 5 -> 4 -> 3` | `mark(3)`. `3 == 7`? No. `isMarked(4)`? Yes. No more neighbors, **return false**. |
| `0 -> 1 -> 2 -> 5 -> 4` | Got `false` from `3`, so keep checking: `isMarked(5)`? Yes. No more neighbors, **return false**. |
| `0 -> 1 -> 2 -> 5` | Got `false` from `4`, so keep checking: `isMarked(6)`? No, recurse. |
| `0 -> 1 -> 2 -> 5 -> 6` | `mark(6)`. `6 == 7`? No. `isMarked(5)`? Yes. `isMarked(7)`? No, recurse. |
| `0 -> 1 -> 2 -> 5 -> 6 -> 7` | `mark(7)`. `7 == 7`? **Yes, return true.** |
| `0 -> 1 -> 2 -> 5 -> 6` | Got `true`, **return true**. |
| `0 -> 1 -> 2 -> 5` | Got `true`, **return true**. Vertex `8` is never explored. |
| `0 -> 1 -> 2` | Got `true`, return `true`. |
| `0 -> 1` | Got `true`, return `true`. Vertex `4` never re-examined. |
| `0` | Got `true`, return `true`. Overall answer: `true`. |

**Environment/box-and-pointer reasoning in words.** Every recursive frame has its own copies of the `int` parameters `s` and `t` (ints are copied by value into the new frame). But there is exactly **one** `boolean[] marked` object on the heap, and every frame reaches the same array through the shared instance variable. That shared mutable state is the entire mechanism: a write performed deep in the recursion (`marked[7] = true`) is instantly visible to every frame above it, which is what guarantees no vertex is ever expanded twice. If you instead gave each frame its own copy of `marked`, the algorithm would be back to looping forever. A corollary worth remembering: `marked` is *not* reset between the recursive calls, so this object can only answer one `connected` query correctly before `marked` must be cleared.

**The subtle point about `connected(3, 7)` returning `false`.** Vertex `3` really is connected to `7` in the graph, yet its recursive call returned `false`. That is correct behavior: the return value means "is `t` reachable from me *through currently unmarked vertices*." By the time `3` runs, its only neighbor `4` is already marked. Only the outermost return value answers the original question.

### Example 5: DepthFirstPaths

Goal: find a path from `s` to *every* reachable vertex, visiting each vertex at most once. Think of it as an oracle you build once and then query for directions.

Algorithm as stated in lecture:

> `dfs(v)`:
> - Mark `v`.
> - For each unmarked adjacent vertex `w`:
>   - set `edgeTo[w] = v`
>   - `dfs(w)`

In Java:

```java
public class DepthFirstPaths {
    private boolean[] marked;   // marked[v]: is v reachable from s?
    private int[] edgeTo;       // edgeTo[v]: previous vertex on path s -> v
    private int s;

    public DepthFirstPaths(Graph G, int s) {
        this.s = s;
        marked = new boolean[G.V()];
        edgeTo = new int[G.V()];
        dfs(G, s);
    }

    private void dfs(Graph G, int v) {
        marked[v] = true;
        for (int w : G.adj(v)) {
            if (!marked[w]) {
                edgeTo[w] = v;   // the action: "the way to get to w is through me"
                dfs(G, w);
            }
        }
    }

    public boolean hasPathTo(int v) {
        return marked[v];
    }

    // (extra context) The lecture showed how to read edgeTo backwards by hand
    // but did not give this code; it appears in the standard implementation.
    public Iterable<Integer> pathTo(int v) {
        if (!hasPathTo(v)) return null;
        Deque<Integer> path = new ArrayDeque<>();
        for (int x = v; x != s; x = edgeTo[x]) {
            path.push(x);        // push so the path comes out s-first
        }
        path.push(s);
        return path;
    }
}
```

Trace of `new DepthFirstPaths(G, 0)` on the same graph, matching the lecture's table animation:

| `dfs` call | Action taken |
|---|---|
| `dfs(0)` | `marked[0] = true`. `1` unmarked: `edgeTo[1] = 0`, recurse. |
| `dfs(1)` | `marked[1] = true`. `0` marked, skip. `2` unmarked: `edgeTo[2] = 1`, recurse. |
| `dfs(2)` | `marked[2] = true`. `1` marked. `5` unmarked: `edgeTo[5] = 2`, recurse. |
| `dfs(5)` | `marked[5] = true`. `2` marked. `4` unmarked: `edgeTo[4] = 5`, recurse. |
| `dfs(4)` | `marked[4] = true`. `1` marked. `3` unmarked: `edgeTo[3] = 4`, recurse. |
| `dfs(3)` | `marked[3] = true`. `4` marked. No more neighbors, **return**. |
| back in `dfs(4)` | `5` marked. No more neighbors, **return**. |
| back in `dfs(5)` | `6` unmarked: `edgeTo[6] = 5`, recurse. |
| `dfs(6)` | `marked[6] = true`. `5` marked. `7` unmarked: `edgeTo[7] = 6`, recurse. |
| `dfs(7)` | `marked[7] = true`. `6` marked. No more neighbors, **return**. |
| back in `dfs(6)` | No more neighbors, **return**. |
| back in `dfs(5)` | `8` unmarked: `edgeTo[8] = 5`, recurse. |
| `dfs(8)` | `marked[8] = true`. `5` marked. **Return**. |
| back in `dfs(5)` | No more neighbors, **return**. |
| back in `dfs(2)` | No more neighbors, **return**. |
| back in `dfs(1)` | `4` marked now, skip. **Return**. |
| back in `dfs(0)` | No more neighbors, **return**. Done. |

Final state:

| v | `marked` | `edgeTo` |
|---|---|---|
| 0 | T | - |
| 1 | T | 0 |
| 2 | T | 1 |
| 3 | T | 4 |
| 4 | T | 5 |
| 5 | T | 2 |
| 6 | T | 5 |
| 7 | T | 6 |
| 8 | T | 5 |

Orders produced:

- **DFS preorder** (order of `dfs` calls, which is the order `edgeTo` entries were written): `0 1 2 5 4 3 6 7 8`
- **DFS postorder** (order of `dfs` returns): `3 4 7 6 8 5 2 1 0`

Two differences from `connected` that the lecture pointed out explicitly:

1. There is no early exit. We do not stop when we reach `7`, because the goal is paths to *everything*, so vertex `8` is explored here even though the connectivity version skipped it.
2. The action (`edgeTo[w] = v`) happens *before* the recursive call, which is why this is "DFS preorder." If you instead moved a `print(v)` to after the loop, you would get DFS postorder.

**How to read a path out of `edgeTo`.** To get a path from `0` to `3`: `edgeTo[3] = 4`, `edgeTo[4] = 5`, `edgeTo[5] = 2`, `edgeTo[2] = 1`, `edgeTo[1] = 0`, and `0` is the source, so stop. Reverse the collected vertices: `0 -> 1 -> 2 -> 5 -> 4 -> 3`. Notice that this path is *not* shortest (`0 -> 1 -> 4 -> 3` is shorter), because DFS committed to `4` via `5` before it ever reconsidered the edge `1-4`. This is exactly the gap BFS will close.

### Example 6: The in-lecture poll, reading `edgeTo` backwards

`DepthFirstPaths` was run on some unknown graph from source `0`, producing:

| v | `marked` | `edgeTo` |
|---|---|---|
| 0 | T | - |
| 1 | T | 4 |
| 2 | T | 4 |
| 3 | T | 4 |
| 4 | T | 0 |
| 5 | T | 1 |
| 6 | F | - |
| 7 | T | 2 |
| 8 | F | - |

**Which vertices are reachable from 0?** Everything with `marked == true`: `0, 1, 2, 3, 4, 5, 7`. Vertices `6` and `8` are **not** reachable.

**What is a path from 0 to 2?** Follow `edgeTo` backwards from `2`: `edgeTo[2] = 4`, `edgeTo[4] = 0`, and `0` is the source. Reversing gives `0 -> 4 -> 2`, which is **answer C**. The lecture's one-line summary: "the way we get to `2` is from `4`; the way we get to `4` is from `0`."

Watch out for the distractors: `0 -> 2` (answer A) assumes an edge that `edgeTo` does not claim; `0 -> 4 -> 7 -> 2` and `0 -> 7 -> 4 -> 2` try to involve `7`, but `edgeTo[7] = 2` means `7` was discovered *from* `2`, which is the opposite direction.

---

## Common Pitfalls

1. **Confusing "traversing" with "visiting."** All three depth first traversals traverse the identical set of nodes in the identical recursive shape. Only the position of the action line differs. If an exam asks you to produce an order, locate the action, not the recursion.
2. **Thinking postorder is reversed preorder.** For the lecture's tree, preorder reversed is `GEFCABD`, but the true postorder is `ACBEGFD`. (extra context: reverse postorder equals preorder only in special cases, such as a path-shaped tree; in general these are genuinely different orders.)
3. **Dropping the `if (x == null) return;` base case.** Every one of these methods recurses into `x.left` and `x.right` unconditionally, so the null check is the only thing stopping a `NullPointerException` at every leaf.
4. **Forgetting to mark, in graph traversal.** This is the error the whole second half of the lecture is built around. Without marking, an undirected edge `0-1` alone produces infinite mutual recursion.
5. **Checking `marked` but never setting it (or setting it too late).** `marked[v] = true` must be the first thing `dfs(v)` does. If you defer it, a vertex can be pushed onto the stack more than once before it is ever marked.
6. **Assuming a recursive `false` from `connected` means "not connected."** `connected(3, 7)` returned `false` in the demo even though `3` and `7` are connected. The recursive value is relative to the current `marked` array. Only the top-level answer is meaningful.
7. **Reusing a `marked` array across queries.** After one `connected(s, t)` call, `marked` is dirty. A second query on the same object will give wrong answers unless `marked` is reset.
8. **Assuming DFS gives shortest paths.** The demo's own output has `edgeTo[4] = 5`, encoding `0 -> 1 -> 2 -> 5 -> 4` when `0 -> 1 -> 4` exists. DFS gives *a* path, not *the shortest* path. That is what BFS is for.
9. **Reading `edgeTo` forwards.** `edgeTo[w]` is the vertex you came *from*, so a path must be reconstructed backwards from the target and then reversed. `edgeTo[7] = 2` means `2 -> 7`, not `7 -> 2`.
10. **Treating "the" DFS order as unique.** A graph has many valid DFS preorders; the one you get depends on the order you iterate neighbors. The slides deliberately say "**one valid** DFS preorder for this graph."
11. **Thinking "acyclic" alone makes a tree.** A tree must be acyclic *and* connected. The lecture's pink non-tree examples included a disconnected acyclic structure. (extra context: a disconnected acyclic graph is called a forest.)
12. **Forgetting that every tree is a graph.** The implication runs tree => graph, not the other way. Statements like "all graphs are trees" are false; "all trees are graphs" is true.
13. **Assuming self loops and parallel edges are allowed.** In 61B, "graph" means "simple graph" by default, so both are excluded unless a problem says otherwise.
14. **Assuming a simply-stated graph problem is easy.** Euler tour and Hamilton tour differ by one word (edge vs vertex) and differ by an exponential in known difficulty. Do not guess at difficulty.
15. **Trying to get level order / BFS out of plain recursion.** The three depth first orders fall out of recursion for free; level order and BFS do not, which is exactly why the closing challenge says you will need a data structure.
16. **(extra context) Very deep recursion.** Recursive DFS uses one stack frame per vertex on the current path, so on a graph with a very long path it can overflow the call stack. Real implementations sometimes use an explicit stack.

---

## Likely Exam Points

**1. Produce all four traversal orders of a given binary tree.**

*Q:* For the tree with root `D`, left child `B` (children `A`, `C`) and right child `F` (children `E`, `G`), give the preorder, inorder, postorder, and level order traversals.

*A:* Preorder `DBACFEG`; inorder `ABCDEFG`; postorder `ACBEGFD`; level order `DBFACEG`.

**2. Given mangled traversal code, state the order it prints.**

*Q:* What order does this print on the same tree?

```java
void mystery(BSTNode x) {
    if (x == null) return;
    mystery(x.right);
    System.out.print(x.key);
    mystery(x.left);
}
```

*A:* `GFEDCBA`. It is an inorder traversal with the children swapped, so it is reverse inorder, and since the tree is a BST, that prints the keys in descending order.

**3. Choose the right traversal for a task.**

*Q:* You want (a) a sorted listing of all keys in a BST, (b) an indented directory listing, (c) the total size of all files under a folder. Which traversal for each, and why for (c)?

*A:* (a) inorder, (b) preorder, (c) postorder, because a folder's total is a function of its children's totals, so the action at a node must come after all recursive calls have returned.

**4. Is this graph a tree?**

*Q:* The pre-pandemic BART map is connected and has exactly one cycle (San Bruno, SFO, Millbrae). Is it a tree? What is the minimum number of edges you would have to delete to make it one?

*A:* Not a tree, because a tree requires exactly one path between every pair of vertices and a cycle gives two paths between vertices on it. Deleting one edge of that single cycle suffices, so the answer is 1 edge.

**5. Terminology precision.**

*Q:* Define "simple graph," "simple path," and "cyclic graph," and state which of these a graph with a single self loop at vertex `v` violates.

*A:* A simple graph has no self loops and no parallel edges. A simple path is a path with no repeated vertices. A cyclic graph is one containing at least one cycle. A graph with a self loop at `v` is not simple (it has a self loop) and is cyclic (the self loop is a path whose first and last vertices are the same).

**6. Why marking is required.**

*Q:* State concretely what goes wrong with `connected(s, t)` if you omit marking, using an undirected graph with just the edge `0-1` and the query `connected(0, 1)` replaced by `connected(0, 5)`.

*A:* `connected(0, 5)` finds `0 != 5` and recurses on `1`; `connected(1, 5)` finds `1 != 5` and recurses on its neighbor `0`; this alternates forever, so the program never terminates (in Java, a `StackOverflowError`). Marking each vertex on entry and only recursing into unmarked neighbors bounds the number of calls by the number of vertices.

**7. DFS preorder and postorder of a graph given neighbor order.**

*Q:* For the lecture's graph (adjacency lists in increasing numeric order, source `0`), give the DFS preorder and DFS postorder. Also state what each corresponds to operationally.

*A:* DFS preorder `0 1 2 5 4 3 6 7 8`, which is the order of `dfs` *calls* (action before recursing). DFS postorder `3 4 7 6 8 5 2 1 0`, which is the order of `dfs` *returns* (action after recursing).

**8. BFS order and distance.**

*Q:* For the same graph and source `0`, give a valid BFS order and the distance from `0` to vertex `7`.

*A:* `0, 1, {2, 4}, {5, 3}, {6, 8}, 7` (the slide writes this as `0 1 24 53 68 7`); vertices inside braces are at equal distance and may appear in either order. Distance from `0` to `7` is 5, via `0 -> 1 -> 2 -> 5 -> 6 -> 7` (or `0 -> 1 -> 4 -> 5 -> 6 -> 7`).

**9. Fill in or read off the `marked` and `edgeTo` tables.**

*Q:* After `DepthFirstPaths` from source `0`, you are given `edgeTo[1]=4, edgeTo[2]=4, edgeTo[3]=4, edgeTo[4]=0, edgeTo[5]=1, edgeTo[7]=2`, with `marked[6] = marked[8] = false`. Which vertices are unreachable from `0`, and what path does the table encode from `0` to `5`?

*A:* `6` and `8` are unreachable. Path to `5`: `edgeTo[5]=1`, `edgeTo[1]=4`, `edgeTo[4]=0`, so `0 -> 4 -> 1 -> 5`.

**10. DFS vs shortest paths.**

*Q:* In the lecture's `DepthFirstPaths` run from `0`, the recorded path to vertex `4` is `0 -> 1 -> 2 -> 5 -> 4`, but the graph contains the edge `1-4`. Is the algorithm buggy? Explain.

*A:* No. DFS guarantees *a* path to every reachable vertex, not a shortest one. When `dfs(1)` ran, it iterated neighbors in order and recursed into `2` first; that recursion reached `4` (through `5`) and marked it, so by the time `dfs(1)` examined `4`, it was already marked and no shorter `edgeTo` was recorded. Getting shortest paths requires visiting vertices in BFS order instead.

**11. Euler vs Hamilton difficulty.**

*Q:* One of these two has a known algorithm running in time proportional to the number of edges, and one has no known efficient algorithm. Which is which: a cycle using every edge exactly once, or a cycle using every vertex exactly once?

*A:* Every edge exactly once is the **Euler tour**, efficiently solvable (since 1873, covered in CS70, time proportional to the number of edges). Every vertex exactly once is the **Hamilton tour**, for which the best known algorithms are exponential time.

**12. Conceptual: what is a traversal good for?**

*Q:* In one sentence, how do you turn a graph traversal into an algorithm?

*A:* By performing actions or setting instance variables at the right moment during the traversal (before the recursive calls for preorder, after for postorder), which is how the same DFS skeleton solves s-t connectivity (`marked`) and path finding (`edgeTo`).

**13. (extra context, commonly asked) Can you reconstruct a tree from its traversals?**

*Q:* Given only the postorder traversal of a binary tree, can you recover the tree?

*A:* No. A single traversal is not enough; many distinct trees share a postorder. (The lecture raised this as an open-ended puzzle and said that one traversal is definitely not enough. extra context: inorder together with either preorder or postorder *does* uniquely determine a binary tree with distinct keys, whereas preorder plus postorder alone does not in general, since a root with one left child and a root with one right child give the same pair of strings.)

---

## Summary

- A **tree** is a set of nodes plus edges with exactly one path between any two nodes, which is equivalent to being connected *and* acyclic. A **rooted tree** designates one node as the root; every other node then has exactly one parent, and childless nodes are leaves.
- Trees appear inside data structures (BSTs, heaps, tries, disjoint sets) and outside them (org charts, family lineages, phylogenetic trees, procedural manuals).
- "Iterating" a tree is called **traversal**, and unlike a list there are several natural orders, each useful for different things.
- The four traversals covered, on the tree rooted at `D` with children `B` (`A`, `C`) and `F` (`E`, `G`):
  - **Level order** `DBFACEG`, **preorder** `DBACFEG`, **inorder** `ABCDEFG`, **postorder** `ACBEGFD`.
- The three depth first traversals are the *same recursion* with the visit (the action) moved: before both calls (pre), between them (in), after both (post). **Traversing** a node is not the same as **visiting** it.
- Human trick: trace counterclockwise around the drawing and visit at a node's left (pre), bottom (in), or right (post).
- Uses: inorder prints a BST in sorted order; preorder prints an indented directory listing; postorder aggregates values upward (total file sizes), because a parent's value needs its children's values first.
- A **graph** is nodes plus zero or more edges, each connecting two nodes: the tree definition with the one-path constraint removed. Every tree is a graph; not every graph is a tree (cycles and/or disconnection are allowed).
- A **simple graph** has no self loops and no parallel edges. In 61B, "graph" means "simple graph."
- Vocabulary: directed/undirected, cyclic/acyclic, vertex, edge, adjacent, path, simple path, cycle, connected (two vertices), connected (whole graph), labels/weights.
- Examples: Paris Metro (undirected, connected, cyclic, vertex-labeled); pre-pandemic BART (one cycle, so not a tree); WordNet (directed, not a tree, two paths from `group_action` to `event`).
- Graph problems are easy to state and vary enormously in difficulty: Euler tour (every edge once) is efficiently solvable since 1873; Hamilton tour (every vertex once) has only exponential-time known algorithms.
- The common tool for essentially all graph problems is **traversal**.
- The naive recursion for **s-t connectivity** loops forever on cycles. The fix is to **mark** each vertex on arrival and recurse only into unmarked neighbors. That fix *is* **depth first search**: explore a neighbor's entire subgraph before moving on.
- Marking works because all recursive frames share one `marked` array on the heap; a deep write is visible to every frame above.
- Recursive return values inside `connected` are relative to the current `marked` array, so an inner `false` does not mean "not connected"; only the top-level answer does.
- **`DepthFirstPaths`** is the same skeleton with one added action, `edgeTo[w] = v`, meaning "the way to reach `w` is through me." At the end, `marked` tells you what is reachable and `edgeTo` encodes a path to each reachable vertex, read backwards from the target and reversed.
- Graphs have traversal orders too, relative to a source: **DFS preorder** (order of calls, `012543678`), **DFS postorder** (order of returns, `347685210`), and **BFS order** (by distance from the source, `0 1 24 53 68 7`). There are many valid DFS orders depending on neighbor iteration order; there is no graph analogue of inorder.
- DFS gives *a* path, not a shortest path. Getting shortest paths requires visiting in BFS order, which needs an extra data structure (the closing challenge; resolved next lecture).
