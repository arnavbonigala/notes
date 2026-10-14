<!-- Wed, Oct 14, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 21: Tree and Graph Traversals

This lecture closes out the tree material and opens the graph unit. It starts by recalling what a tree is (a set of nodes plus edges, with exactly one path between any two nodes) and what a *rooted* tree adds (a chosen root, so every non-root node has exactly one parent, and childless nodes are leaves), then observes that iterating over a tree is called *traversal* rather than iteration because, unlike a list, there is no single obvious order: level order plus the three depth-first orders (preorder, inorder, postorder) are all useful for different jobs, such as printing a directory listing (preorder) or summing file sizes in a folder (postorder). The lecture then generalizes: not every relationship is hierarchical (the Paris Metro and BART both contain cycles), so we drop the "exactly one path" constraint and get *graphs*, of which trees are a special case. After fixing vocabulary (simple graph, directed/undirected, cyclic/acyclic, path, simple path, cycle, connected) and surveying famous graph problems whose difficulty is wildly uneven (Euler tour: easy since 1873; Hamilton tour: no efficient algorithm known), the lecture motivates traversal through the s-t connectivity problem. A naive recursive `connected(s, t)` loops forever on a cycle; the fix is *marking* each vertex so it is visited at most once, which yields depth-first search. The lecture then extends DFS into `DepthFirstPaths`, which records an `edgeTo` array giving a path from the source to every reachable vertex, names the two DFS orders (DFS preorder = order of `dfs` calls, DFS postorder = order of `dfs` returns), notes BFS order as the graph analogue of level order, and ends with a challenge: invent BFS to compute shortest path lengths, to be solved next lecture.

(The posted slides are the Spring 2026 deck and carry a "not yet updated for Fall 2026" warning, so treat the announcement slides, homework deadlines, and instructor names as belonging to that semester, not necessarily to Fall 2026. The linked textbook chapter ("20. Tree Traversals and Graphs") served up only its title page, so everything below comes from the slides and code.)

---

## Key Concepts

### 1. Trees, re-stated carefully

A tree is **a set of nodes plus a set of edges connecting them, constrained so that there is exactly one path between any two nodes**. That one constraint does a lot of work: it simultaneously rules out cycles (two different routes between two nodes) and rules out disconnected pieces (zero routes). A *rooted* tree additionally designates one node as the root; every node except the root then has exactly one **parent**, namely the first node on the path from that node toward the root, and a node with no child is a **leaf**.

Two things are worth internalizing here. First, "rooted" is a choice we impose, not a property the node set has intrinsically: the same tree rooted at a different node gives different parent/child relationships. The slides make this point by drawing three different-looking pictures that all have A as root, B as a child of A, and C as a child of B. Second, trees in 61B are not only a data structure implementation detail. We have seen trees as node-based implementations (search trees, tries, heaps, disjoint sets), but trees are a general modeling tool: organization charts, phylogenetic trees and family lineages (with the asterisk that not all family lineages are trees, since intermarriage creates multiple paths), and decision/flowchart documents like the MOH Training Manual for Management of Malaria.

### 2. Why "traversal" and not "iteration"

For a list, iteration has an obvious meaning: front to back. For a tree there are many sensible orders, and each is useful for different things, so the operation gets its own name. The motivating example is a file system tree rooted at a folder `61b`, with subfolders `proj0` and `hw1`, and leaves like `planets.txt` (851 bytes), `hw1.md` (23433 bytes), `karplus-strong.png` (16180 bytes), and `GuitarHeroLite.java` (1251 bytes). Suppose you want the total size of every file under `61b`. You must visit every node, and *the order matters for how you can accumulate the answer*.

### 3. The four orders

Using the lecture's running binary search tree (reconstructed; the PDF diagram is lost):

```
        D
      /   \
     B     F
    / \   / \
   A   C E   G
```

- **Level order**: top to bottom, left to right, "like reading in English": `D B F A C E G`.
- **Preorder** (depth-first): visit the node, then traverse its children: `D B A C F E G`.
- **Inorder** (depth-first): traverse the left child, visit, then traverse the right child: `A B C D E F G`.
- **Postorder** (depth-first): traverse left, traverse right, then visit: `A C B E G F D`.

The three depth-first orders are all the same recursive walk over the tree; the only difference is **where in the recursive procedure the "visit" action sits** relative to the recursive calls. The lecture makes a vocabulary distinction that pays off repeatedly: **traversing a node is not the same as visiting it**. In `preOrder`, we traverse into `B`'s whole subtree, but we only "visit" (print, or do the action on) `B` once, at a specific moment. In a depth-first traversal, every node is traversed into three times in a binary tree (once from the parent, once when returning from each child), but visited exactly once.

"Depth first" is named for the rough idea that deep nodes (like `A`) get handled before shallow ones (like `F`). It is rough because preorder visits the root first; the precise statement is that an entire subtree is finished before the traversal moves to the next sibling subtree.

Note that the inorder sequence of a BST is sorted (`A B C D E F G` here). That is the BST invariant showing up as a traversal property, and it is why inorder is the natural iteration order for a search tree.

### 4. The human tracing trick

For hand-tracing on an exam, the lecture gives a visual trick (explicitly "for humans, not algorithms"): trace a path around the outside of the drawn tree, starting at the top and going counter-clockwise, and then

- **preorder**: visit each time you pass the **left** side of a node,
- **inorder**: visit each time you cross the **bottom** of a node,
- **postorder**: visit each time you cross the **right** side of a node.

One curve, three answers, read off at three different moments. The slide's example tree (reconstructed from the printed answer) is

```
            1
          /   \
         2     3
        / \     \
       4   5     6
          / \     \
         7   8     9
```

and its postorder is `4 7 8 5 2 9 6 3 1`, matching the slide's `478529631`.

### 5. What traversals are good for

Two concrete uses, both from the slides:

- **Preorder for printing a directory listing.** You want to print a folder's name before the things inside it, so the action happens before the recursive descent. The slide's example tree is rooted at `sc2APM`, with children `directOverlay`, `notes`, and `python`, and deeper nodes `directIO`, `directO.suo`, `directO.sln`, `printAPM.py`, `DXHookD3D11.cs`, `Injector.cs`.
- **Postorder for gathering file sizes.** To know a folder's total size you must first know the totals of all its children, so the action must happen *after* the recursive calls return. This is the structural reason postorder exists: it is the order in which you can compute a value that depends on your subtree.

### 6. From trees to graphs

Trees model strict hierarchy well, but many relationships are not hierarchical. A metro map (Paris) or the pre-pandemic BART map is not a tree because it has a **cycle** (for BART, exactly one: San Bruno, SFO, Millbrae), which is the same as saying there is more than one way to get from A to B.

So we relax the definition. **A graph is a set of nodes plus a set of zero or more edges, each connecting two nodes.** That is it: no "exactly one path" constraint. Consequently **every tree is a graph**, but not conversely. A **simple graph** forbids self-loops (length-1 loops) and parallel edges (two edges joining the same pair). In 61B, "graph" means "simple graph" unless stated otherwise.

Graphs come in flavors along independent axes: **directed vs undirected** (do edges have a direction?), **cyclic vs acyclic** (is there a cycle?), and with or without **labels/weights** on vertices or edges. The Paris Metro is undirected, connected, cyclic, and vertex-labeled (each line has a color). The slide's directed-graph example is an "is-a-type-of" ontology where `descent is-a-type-of movement`, and it is not a tree because there are two paths from `group_action` to `event`.

### 7. Graph problems vary enormously in difficulty

The lecture lists famous named problems: s-t path, connectivity, biconnectivity, shortest s-t path, cycle detection, Euler tour, Hamilton tour, planarity, isomorphism. The headline lesson is that **you often cannot tell how hard a graph problem is without deep consideration**. Euler tour (a cycle using every *edge* exactly once) has had an efficient O(number of edges) algorithm since 1873 and is covered in CS70. Hamilton tour (a cycle using every *vertex* exactly once) sounds like a trivial variant, yet after decades of study the best known algorithms are exponential time. The two problem statements differ by one word; their difficulty differs by a complexity class.

### 8. s-t connectivity, and why naive recursion fails

Problem: given source `s` and target `t`, is there a path from `s` to `t`? The obvious recursion is:

- if `s == t`, return true;
- otherwise, if `connected(v, t)` for any neighbor `v` of `s`, return true;
- return false.

This is wrong, and the failure is instructive. On the lecture's graph, `connected(0, 7)` asks `connected(1, 7)`, which (since 0 is a neighbor of 1) asks `connected(0, 7)` again: **infinite loop**, and in Java, a `StackOverflowError`. The structural cause is the thing that distinguishes graphs from trees: cycles, and the fact that undirected edges can be walked back along.

The fix is one idea, and it is the whole idea behind graph traversal: **mark vertices as you reach them and never recurse into a marked vertex.** Marking guarantees each vertex is visited at most once, which both terminates the recursion and bounds the work.

### 9. DFS is a technique, not one algorithm

Exploring a neighbor's *entire* subgraph before moving on to the next neighbor is **depth-first traversal / depth-first search**. On the lecture's graph, DFS from 0 explores all of 1's subgraph before it would ever use the edge 0-3 (in the alternately drawn example), and the slides stress a subtlety: it is entirely possible that 1's subgraph *includes* 3. That is still depth first, because we did not use the edge 0-3 until 1's subgraph was exhausted. DFS is defined by the order of exploration, not by which vertices end up where.

The lecture then shows DFS doing something strictly more than answering yes/no: `DepthFirstPaths` finds *a path* from `s` to every reachable vertex, by storing, for each newly discovered vertex `w`, the vertex `v` we came from in `edgeTo[w]`. This is the general pattern the summary slide emphasizes: **by performing actions or setting instance variables during a traversal, you solve problems**. The traversal is the skeleton; the action is the problem-specific flesh.

### 10. Tree orders vs graph orders

The last conceptual move is to line up the two vocabularies:

| Trees | Graphs (relative to a source `s`) |
|---|---|
| Preorder | **DFS preorder**: action *before* the DFS calls to neighbors; equals the order of `dfs` calls |
| Postorder | **DFS postorder**: action *after* the DFS calls to neighbors; equals the order of `dfs` returns |
| Level order | **BFS order**: act in order of distance from `s`; "breadth first", wide not deep |
| Inorder | no general analogue (inorder needs a notion of "left child" vs "right child") *(extra context)* |

For the lecture's graph with source 0, the three orders are `012543678` (DFS preorder), `347685210` (DFS postorder), and `0 | 1 | 2 4 | 5 3 | 6 8 | 7` (BFS order, grouped by distance from 0).

### 11. The cliffhanger

The closing challenge: given a graph, find the **length of the shortest path from `s` to all other vertices**. Hints: you need to visit vertices in BFS order, and you need some kind of data structure (DFS's recursion gives you a call stack for free; BFS will not be recursive). Solved next lecture.

---

## Definitions

- **Tree**: a set of nodes and a set of edges connecting them, such that there is exactly one path between any two nodes. (Equivalently, as the summary slide puts it: a graph with no cycles in which every vertex is connected.)
- **Rooted tree**: a tree in which one node has been chosen as the root.
- **Root**: the chosen distinguished node of a rooted tree; the only node with no parent.
- **Parent of node N**: the first node on the path from N to the root. Every node except the root has exactly one.
- **Child**: the inverse of parent; B is a child of A if A is B's parent.
- **Leaf**: a node with no children.
- **Traversal**: an order in which you visit (and act upon) the nodes/vertices of a tree or graph. The tree analogue of list iteration.
- **Visiting a node**: performing the action (printing, accumulating, setting a field) on it. Distinct from **traversing** a node, which means passing through it during the walk.
- **Level order**: traversal visiting nodes top to bottom, left to right.
- **Depth-first traversal**: a traversal that completely finishes one child's subtree/subgraph before starting the next. Three tree variants: preorder, inorder, postorder.
- **Preorder**: visit the node, then traverse its children.
- **Inorder**: traverse the left child, visit the node, then traverse the right child.
- **Postorder**: traverse the left child, traverse the right child, then visit the node.
- **Graph**: a set of nodes (vertices) and a set of zero or more edges, each of which connects two nodes.
- **Vertex (node)**: an element of a graph's node set.
- **Edge**: a pair of vertices, representing a connection between them.
- **Adjacent**: two vertices with an edge between them are adjacent (they are each other's neighbors).
- **Simple graph**: a graph with no self-loops (edges from a vertex to itself) and no parallel edges (two edges connecting the same pair of vertices). In 61B, "graph" means "simple graph" by default.
- **Directed graph**: a graph whose edges have a direction (edge from a to b is not the same as b to a).
- **Undirected graph**: a graph whose edges have no direction.
- **Labeled / weighted graph**: a graph whose vertices or edges carry labels (for example colors) or numeric weights.
- **Path**: a sequence of vertices connected by edges.
- **Simple path**: a path with no repeated vertices.
- **Cycle**: a path whose first and last vertices are the same.
- **Cyclic graph**: a graph containing a cycle. **Acyclic**: containing none.
- **Connected (two vertices)**: there exists a path between them.
- **Connected (graph)**: all vertices are connected to each other.
- **s-t connectivity problem**: given a source `s` and target `t`, is there a path from `s` to `t`?
- **s-t path problem**: as above, but produce the path.
- **Shortest s-t path**: what is the shortest path between `s` and `t`?
- **Connectivity (problem)**: is the whole graph connected?
- **Biconnectivity**: is there a vertex whose removal disconnects the graph?
- **Cycle detection**: does the graph contain any cycles?
- **Euler tour**: a cycle that uses every *edge* exactly once. Efficiently solvable in O(number of edges), known since 1873.
- **Hamilton tour**: a cycle that uses every *vertex* exactly once. No efficient algorithm known; best known are exponential time.
- **Planarity**: can the graph be drawn on paper with no crossing edges?
- **Isomorphism**: are two graphs the same graph in disguise?
- **Depth First Search (DFS) / Depth First Traversal**: a traversal that explores a neighbor's entire subgraph before moving on to the next neighbor, marking vertices so each is visited at most once.
- **Marking**: recording that a vertex has already been reached, so the traversal does not revisit it. Without it, cycles cause infinite recursion.
- **`edgeTo[w]`**: the vertex from which `w` was discovered during DFS. Following `edgeTo` backwards from a vertex to the source reconstructs a path.
- **DFS preorder**: graph traversal order in which the action happens before the recursive calls to neighbors; equal to the order of `dfs` calls.
- **DFS postorder**: graph traversal order in which the action happens after the recursive calls to neighbors; equal to the order of `dfs` returns.
- **BFS order**: graph traversal order by increasing distance from the source; the graph analogue of level order. BFS = breadth first search.

---

## Worked Examples

### Example 1: The three depth-first tree traversals

The lecture's code, lightly cleaned into compiling Java (the slides omit semicolons and return types in places):

```java
private void preOrder(BSTNode x) {
    if (x == null) { return; }
    System.out.print(x.key);     // visit
    preOrder(x.left);            // traverse left
    preOrder(x.right);           // traverse right
}

private void inOrder(BSTNode x) {
    if (x == null) { return; }
    inOrder(x.left);
    System.out.print(x.key);
    inOrder(x.right);
}

private void postOrder(BSTNode x) {
    if (x == null) { return; }
    postOrder(x.left);
    postOrder(x.right);
    System.out.print(x.key);
}
```

All three are byte-for-byte the same except for the position of one line. On the tree

```
        D
      /   \
     B     F
    / \   / \
   A   C E   G
```

they print `DBACFEG`, `ABCDEFG`, and `ACBEGFD` respectively.

**Step-by-step call stack trace of `preOrder(D)`,** following the lecture's slide-by-slide demo. Reason about this as a stack of frames: each frame holds its own local `x`, which points at one node object. Nothing is shared or mutated; the only "state" is the printed output and the stack itself.

| Step | Call stack (top at right) | Output so far | What just happened |
|---|---|---|---|
| 1 | `preOrder(D)` | (empty) | `x` points at D, not null |
| 2 | `preOrder(D)` | `D` | printed `x.key` |
| 3 | `preOrder(D)` → `preOrder(B)` | `D` | descended into `x.left` |
| 4 | `preOrder(D)` → `preOrder(B)` | `DB` | printed B |
| 5 | `preOrder(D)` → `preOrder(B)` → `preOrder(A)` | `DB` | descended to A |
| 6 | ... → `preOrder(A)` | `DBA` | printed A |
| 7 | ... → `preOrder(A)` | `DBA` | A's two children are `null`; both calls hit the base case and return immediately (the slides skip these "for brevity") |
| 8 | `preOrder(D)` → `preOrder(B)` | `DBA` | A's frame popped |
| 9 | `preOrder(D)` → `preOrder(B)` → `preOrder(C)` | `DBAC` | C printed, its null children return |
| 10 | `preOrder(D)` | `DBAC` | B's frame popped; B is done |
| 11 | `preOrder(D)` → `preOrder(F)` | `DBACF` | descended into `x.right`, printed F |
| 12 | ... → `preOrder(E)` | `DBACFE` | E's subtree done |
| 13 | ... → `preOrder(G)` | `DBACFEG` | G's subtree done |
| 14 | (empty) | `DBACFEG` | all frames popped |

Two observations to carry forward. First, the base case `if (x == null) return;` is what makes the recursion terminate; on a tree you need nothing else, because there is exactly one path to each node and therefore no way to come back. Second, each node is passed through three times but printed once, which is exactly the "traverse vs visit" distinction.

### Example 2: Postorder for computing total file size

The lecture's code (this is the one place the slides show a traversal that *returns* a value, and the slide's pseudocode is loose: it is declared like a `void` method but returns `int`, and it uses `x.children()` for a general tree rather than `left`/`right`):

```java
// Slide version, as real Java.
private int postOrder(Node x) {
    if (x == null) { return 0; }
    int total = 0;
    for (Node c : x.children()) {
        total += postOrder(c);   // recurse first ...
    }
    total += x.fileSize();       // ... then act
    return total;
}
```

Why this *must* be postorder: `total` cannot be computed until every child's recursive call has returned a number. The action (adding this node's own size, returning the sum) depends on the subtree results, so it is forced to come after the calls. Compare with the directory-*listing* example, where the action (print the name) depends on nothing, so it can come before the calls, which is what produces a readable top-down listing.

**Numbers from the slide's `sc2APM` example.** The diagram's exact leaf-to-folder attribution is partly unreadable in the PDF extraction, but the arithmetic chain the slides display is unambiguous and shows the bubbling-up clearly:

```
directIO  = 18381 (DXHookD3D11.cs) + 8798 (Injector.cs)          = 27179
directOverlay = 27179 (directIO) + 38912 + 881                   = 66972
sc2APM    = 66972 (directOverlay) + 874 (python subtree) + 324   = 68170
```

Reading this as an environment/stack picture: the deepest frames (the `.cs` files) return their own `fileSize()` with an empty `children()` loop. Their caller, the frame for `directIO`, has accumulated `18381` into its *own local* `total`, then `8798`, then adds `directIO`'s own size (0 for this folder) and returns `27179`. Each frame has a private `total`; there is no shared accumulator, no field being mutated. The totals flow upward purely through return values, which is why postorder with a return value is the clean way to write this. (If you instead wanted to write it with a shared mutable counter, you would pass around or close over one object and lose the compositional structure.)

### Example 3: s-t connectivity, broken then fixed

The graph used throughout the DFS demos, reconstructed from the adjacency checks the slides perform:

```
 s=0 --- 1 --- 4 --- 3
         |     |
         2 --- 5 --- 6 --- 7=t
               |
               8
```

Adjacency lists, in the order the demo checks them (the order matters for the answers):

```
0: 1
1: 0, 2, 4
2: 1, 5
3: 4
4: 1, 3, 5
5: 2, 4, 6, 8
6: 5, 7
7: 6
8: 5
```

Nine vertices, nine edges, exactly one cycle: `1 - 2 - 5 - 4 - 1`.

**The broken version:**

```java
// WRONG: infinite recursion on any graph with a cycle.
private boolean connected(int s, int t) {
    if (s == t) { return true; }
    for (int v : G.adj(s)) {
        if (connected(v, t)) { return true; }
    }
    return false;
}
```

Trace of `connected(0, 7)`: `0 != 7`, so we check neighbor 1 and call `connected(1, 7)`. `1 != 7`, so we check 1's first neighbor, which is 0, and call `connected(0, 7)`. The stack now grows `0, 1, 0, 1, 0, 1, ...` without bound. Note that each frame is a fresh frame with its own `s`, so nothing in the program state records that we have been at 0 before. That missing memory is precisely the bug.

**The fixed version:**

```java
private boolean[] marked;   // one array, shared by every frame

private boolean connected(int s, int t) {
    marked[s] = true;                       // remember we have been here
    if (s == t) { return true; }
    for (int v : G.adj(s)) {
        if (!marked[v] && connected(v, t)) {
            return true;
        }
    }
    return false;
}
```

The box-and-pointer intuition: `marked` is a single array object on the heap; every recursive frame holds a reference to that same array, so a write made deep in the recursion is visible to every frame, including frames that have not resumed yet. This is the opposite of the `total` variable in Example 2, which was per-frame. Mutating shared state is what buys termination.

**Full trace of `connected(0, 7)` on the graph above,** following the lecture's demo:

| Call stack | Action |
|---|---|
| `0` | mark 0. `0 == 7`? No. `isMarked(1)`? No → recurse |
| `0 → 1` | mark 1. No. neighbor 0 marked; neighbor 2 unmarked → recurse |
| `0 → 1 → 2` | mark 2. No. neighbor 1 marked; neighbor 5 unmarked → recurse |
| `0 → 1 → 2 → 5` | mark 5. No. neighbor 2 marked; neighbor 4 unmarked → recurse |
| `0 → 1 → 2 → 5 → 4` | mark 4. No. neighbor 1 marked; neighbor 3 unmarked → recurse |
| `0 → 1 → 2 → 5 → 4 → 3` | mark 3. No. neighbor 4 marked. **No more neighbors → return false** |
| `0 → 1 → 2 → 5 → 4` | `connected(3,7)` was false, keep going: neighbor 5 marked. No more neighbors → **return false** |
| `0 → 1 → 2 → 5` | `connected(4,7)` was false, keep going: neighbor 6 unmarked → recurse |
| `0 → 1 → 2 → 5 → 6` | mark 6. No. neighbor 5 marked; neighbor 7 unmarked → recurse |
| `0 → 1 → 2 → 5 → 6 → 7` | mark 7. `7 == 7`? **Yes, return true** |
| `0 → 1 → 2 → 5 → 6` | true → **return true** |
| `0 → 1 → 2 → 5` | true → return true |
| `0 → 1 → 2` | true → return true |
| `0 → 1` | true → return true |
| `0` | true → **return true** |

Notice the shape: we dove all the way down the branch `1, 2, 5, 4, 3`, dead-ended, unwound only as far as necessary (back to 5), and then tried 5's next neighbor. That is "go as deep as possible", and it also shows that DFS's answer is *a* path, not a short one: the found route `0-1-2-5-6-7` is fine here, but DFS in general happily returns a long path when a short one exists.

Also note that vertex 8 was never visited: DFS only touches vertices reachable from `s`, and it stopped early because it found `t`.

### Example 4: `DepthFirstPaths`

The slides' pseudocode:

```
dfs(v):
  - Mark v.
  - For each unmarked adjacent vertex w:
      - set edgeTo[w] = v
      - dfs(w)
```

As Java:

```java
public class DepthFirstPaths {
    private boolean[] marked;
    private int[] edgeTo;
    private int s;

    public DepthFirstPaths(Graph G, int s) {
        marked = new boolean[G.V()];
        edgeTo = new int[G.V()];
        this.s = s;
        dfs(G, s);
    }

    private void dfs(Graph G, int v) {
        marked[v] = true;
        for (int w : G.adj(v)) {
            if (!marked[w]) {
                edgeTo[w] = v;   // the action, done BEFORE recursing: DFS preorder
                dfs(G, w);
            }
        }
    }
}
```

Goal, in the lecture's words: find a path from `s` to every other reachable vertex, visiting each vertex at most once. Note there is no `t` and no early exit; this explores everything reachable.

**Trace of `dfs(0)` on the same graph,** with the final table from the slides:

| call stack | what happens | `edgeTo` set |
|---|---|---|
| `0` | mark 0; 1 unmarked | `edgeTo[1] = 0` |
| `0→1` | mark 1; 0 marked; 2 unmarked | `edgeTo[2] = 1` |
| `0→1→2` | mark 2; 1 marked; 5 unmarked | `edgeTo[5] = 2` |
| `0→1→2→5` | mark 5; 2 marked; 4 unmarked | `edgeTo[4] = 5` |
| `0→1→2→5→4` | mark 4; 1 marked; 3 unmarked | `edgeTo[3] = 4` |
| `0→1→2→5→4→3` | mark 3; 4 marked; no more neighbors, **return** | |
| `0→1→2→5→4` | 5 marked; no more neighbors, **return** | |
| `0→1→2→5` | 6 unmarked | `edgeTo[6] = 5` |
| `0→1→2→5→6` | mark 6; 5 marked; 7 unmarked | `edgeTo[7] = 6` |
| `0→1→2→5→6→7` | mark 7; 6 marked; **return** | |
| `0→1→2→5→6` | **return** | |
| `0→1→2→5` | 8 unmarked | `edgeTo[8] = 5` |
| `0→1→2→5→8` | mark 8; 5 marked; **return** | |
| `0→1→2→5` | done, **return** | |
| `0→1→2` | done, **return** | |
| `0→1` | 4 already marked; done, **return** | |
| `0` | done, **return** | |

Final state:

| # | marked | edgeTo |
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

- **Order of `dfs` calls (DFS preorder)**: `0 1 2 5 4 3 6 7 8`
- **Order of `dfs` returns (DFS postorder)**: `3 4 7 6 8 5 2 1 0`

The `edgeTo` array is a tree in disguise: it stores, for each vertex, its parent in the **DFS tree** rooted at `s`. The set of edges `{(edgeTo[v], v)}` forms a tree spanning exactly the reachable vertices, which is why one array of ints suffices to encode a path to every vertex at once.

**Reading a path out of `edgeTo`.** You walk *backwards* from the destination to the source and then reverse. For example, a path from 0 to 3: `edgeTo[3] = 4`, `edgeTo[4] = 5`, `edgeTo[5] = 2`, `edgeTo[2] = 1`, `edgeTo[1] = 0`, so the path is `0 → 1 → 2 → 5 → 4 → 3`.

Written out *(extra context: the slides show only `dfs` and the table; `hasPathTo`/`pathTo` are the natural client methods and appear in the Algorithms 4th edition code this lecture draws its figures from)*:

```java
public boolean hasPathTo(int v) {
    return marked[v];       // reachable iff DFS from s marked it
}

public Iterable<Integer> pathTo(int v) {
    if (!hasPathTo(v)) { return null; }
    Deque<Integer> path = new ArrayDeque<>();
    for (int x = v; x != s; x = edgeTo[x]) {
        path.push(x);       // pushing reverses the backwards walk
    }
    path.push(s);
    return path;            // iterating a Deque goes from most recent push
}
```

### Example 5: Same traversal, different placement (DFS postorder)

```java
private void dfsPostorder(Graph G, int v) {
    marked[v] = true;
    for (int w : G.adj(v)) {
        if (!marked[w]) {
            dfsPostorder(G, w);
        }
    }
    System.out.print(v);   // action AFTER the calls: DFS postorder
}
```

Running this from vertex 0 on the lecture's graph prints `347685210`. This is identical code to `DepthFirstPaths.dfs` except for which side of the loop the action sits on, exactly mirroring the preorder/postorder relationship in trees. The lecture's framing: tree traversals and graph traversals are the same family of ideas, with graphs needing the `marked` array because they can have cycles and multiple paths.

### Example 6: The poll, worked (reading `edgeTo` on an unknown graph)

Given the result of `DepthFirstPaths` on an unknown graph:

| # | marked | edgeTo |
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

- **Which vertices are reachable from 0?** Every marked one: `0, 1, 2, 3, 4, 5, 7`. Vertices 6 and 8 are unmarked, so no path from 0 reaches them.
- **A path from 0 to 2?** Walk backwards: the way we got to 2 is from `edgeTo[2] = 4`; the way we got to 4 is from `edgeTo[4] = 0`. So the path is **`0 → 4 → 2`** (choice C). Choice B (`0 → 4 → 7 → 2`) is wrong because `edgeTo[7] = 2`, meaning 7 was discovered *from* 2, not the other way round. Choice A would require an edge 0-2, which `edgeTo` gives no evidence for.

---

## Common Pitfalls

1. **Forgetting to mark, or marking too late.** This is *the* graph-traversal bug. Without marking, any cycle (or any undirected edge, since you can walk back along it) sends the recursion into an infinite loop and then a `StackOverflowError`. Mark a vertex at the top of its `dfs`, not after the loop.
2. **Checking `marked` in only one place.** The lecture's code both checks `!marked[w]` before recursing and marks on entry. Dropping the guard in the loop means you re-enter already-visited vertices.
3. **Assuming DFS finds shortest paths.** It does not. In the demo, DFS reaches 4 via `0-1-2-5-4` even though `0-1-4` exists, so `edgeTo[4] = 5`. Shortest paths need BFS, which is next lecture's topic.
4. **Thinking DFS preorder/postorder are unique.** They depend entirely on the order in which `G.adj(v)` hands back neighbors. The slides say "*One* valid DFS preorder for this graph: 012543678". If an exam gives you adjacency lists, use their order; if it gives only a picture, state the convention you used.
5. **Confusing "traverse" with "visit".** A depth-first traversal passes through a node several times but acts on it once. Exam answers that list a node more than once in a preorder are usually this confusion.
6. **Confusing DFS postorder with "reverse preorder".** For the lecture's graph, preorder is `012543678` and postorder is `347685210`, which is *not* the reverse of the preorder (`876345210`). They are different orders, not mirror images.
7. **Mixing up level order and preorder.** Both start at the root. For the `D` tree, level order is `DBFACEG` and preorder is `DBACFEG`. Level order is breadth-wise; preorder plunges.
8. **Getting the postorder of a tree wrong by one swap.** The poll's distractors (`DBACEFG`, `ACBFEGD`) are exactly the kinds of near-misses you produce by rushing. Use the counter-clockwise outline trick: for postorder, visit when you cross the **right** side of a node, and the root is always last.
9. **Reading `edgeTo` forwards.** `edgeTo[w] = v` means "we got to `w` from `v`", so paths are reconstructed by walking from the destination back to the source and reversing. Reading it forwards gives nonsense.
10. **Treating `edgeTo[s]` as meaningful.** The source has no parent; the table shows `-`. Your `pathTo` loop must terminate at `s` rather than following `edgeTo[s]`.
11. **Forgetting that an undirected edge appears in two adjacency lists.** This is why `isMarked(parent)? Yes` shows up constantly in the trace, and why the naive algorithm loops.
12. **Writing a postorder accumulator as `void`.** The file-size example only works because each call *returns* its subtree total. The slide's code is informal pseudocode (missing semicolons, missing `int` return type, and a couple of later slides mistype `edgeTo[4] = 5` as `edgeTo[3] = 4`); write real Java on exams and trust the tables over the slide prose.
13. **Assuming every tree-shaped thing is a tree.** Family lineages generally are not (two paths can connect two people), the "is-a-type-of" ontology on the slides is not (two paths from `group_action` to `event`), and BART is not (San Bruno / SFO / Millbrae form a cycle). Check the "exactly one path between any two nodes" condition.
14. **Assuming a graph must be connected or simple.** A graph may have *zero* edges, several components, and (unless we say "simple") self-loops and parallel edges. In 61B we assume simple by default, but not connected by default.
15. **Thinking "sounds similar" means "similarly hard".** Euler tour and Hamilton tour differ by one word and by a complexity class.
16. **Applying inorder to a non-binary tree.** Inorder needs exactly a left side and a right side; the general file-system tree with `x.children()` has no such split. *(extra context)*

---

## Likely Exam Points

### 1. Produce all four traversal orders for a given tree

**Q.** For the tree below, give the preorder, inorder, postorder, and level order traversals.

```
        D
      /   \
     B     F
    / \   / \
   A   C E   G
```

**A.** Preorder `DBACFEG`; inorder `ABCDEFG`; postorder `ACBEGFD`; level order `DBFACEG`. (Sanity checks: preorder starts with the root, postorder ends with the root, inorder of a BST is sorted, level order reads the drawing like English.)

### 2. Identify where in the recursion the "action" goes

**Q.** You want to print each folder's name *before* the names of everything inside it. Which traversal, and where does the print statement go relative to the recursive calls? What if instead you want to compute each folder's total size?

**A.** Printing a directory listing is **preorder**: the print comes before the recursive calls, because it depends on nothing. Computing total size is **postorder**: the `total += x.fileSize(); return total;` comes after the loop of recursive calls, because the node's value depends on the values returned by its children.

### 3. Hand-trace a tree traversal's call stack

**Q.** During `preOrder(D)` on the tree above, what is the call stack at the moment `A` is printed, and how many frames are on it?

**A.** `preOrder(D) → preOrder(B) → preOrder(A)`, three frames. Each frame holds its own local `x`; the output accumulates outside the frames.

### 4. Why the naive s-t algorithm is wrong, and the fix

**Q.** A student writes `connected(s, t)`: "if `s == t` return true; else return true if `connected(v, t)` for any neighbor `v` of `s`; else return false." Why does this fail, and what is the minimal fix?

**A.** On any graph with a cycle (or in fact any undirected edge) it recurses forever: `connected(0,7)` calls `connected(1,7)` which calls `connected(0,7)` again, because 0 is a neighbor of 1. Nothing records that 0 was already reached. The fix is to **mark** `s` on entry and only recurse into *unmarked* neighbors, so each vertex is visited at most once.

### 5. Fill in the `marked` / `edgeTo` table and give both DFS orders

**Q.** Run `DepthFirstPaths` from vertex 0 on

```
 0 --- 1 --- 4 --- 3
       |     |
       2 --- 5 --- 6 --- 7
             |
             8
```

with adjacency lists `0:[1]`, `1:[0,2,4]`, `2:[1,5]`, `3:[4]`, `4:[1,3,5]`, `5:[2,4,6,8]`, `6:[5,7]`, `7:[6]`, `8:[5]`. Give `edgeTo`, the DFS preorder, and the DFS postorder.

**A.** `edgeTo`: `1←0, 2←1, 5←2, 4←5, 3←4, 6←5, 7←6, 8←5` (0 has none). DFS preorder (order of calls) `0 1 2 5 4 3 6 7 8`. DFS postorder (order of returns) `3 4 7 6 8 5 2 1 0`.

### 6. Reconstruct a path from `edgeTo`

**Q.** Given `edgeTo = [-, 4, 4, 4, 0, 1, -, 2, -]` and `marked = [T,T,T,T,T,T,F,T,F]` after DFS from 0, which vertices are unreachable from 0, and what is the path from 0 to 7?

**A.** Unreachable: 6 and 8 (unmarked). Path to 7: `edgeTo[7]=2`, `edgeTo[2]=4`, `edgeTo[4]=0`, so `0 → 4 → 2 → 7`.

### 7. BFS order vs DFS order

**Q.** For the graph in point 5, give the BFS order from 0, grouped by distance.

**A.** `0 | 1 | 2, 4 | 5, 3 | 6, 8 | 7`, that is `0 1 2 4 5 3 6 8 7` as one of the valid orders (within a distance group the order depends on adjacency order). BFS is the graph analogue of level order: act in order of distance from the source.

### 8. Tree vs graph classification

**Q.** True or false, with justification: (a) every tree is a graph; (b) every graph is a tree; (c) a graph with 5 vertices and 0 edges is a tree; (d) the pre-pandemic BART map is a tree.

**A.** (a) True, a tree is just a graph with the extra "exactly one path between any two nodes" constraint. (b) False, graphs may have cycles or be disconnected. (c) False, there is no path between two distinct vertices, so the graph is not connected. (d) False, it has a cycle (San Bruno, SFO, Millbrae).

### 9. Graph vocabulary precision

**Q.** Define *path*, *simple path*, and *cycle*, and say what makes a graph *simple* and what makes it *connected*.

**A.** A path is a sequence of vertices connected by edges; a simple path is a path with no repeated vertices; a cycle is a path whose first and last vertices are the same. A graph is simple if it has no self-loops and no parallel edges. A graph is connected if every pair of vertices has a path between them.

### 10. Problem-difficulty trivia

**Q.** State the difference between an Euler tour and a Hamilton tour, and the known difficulty of each.

**A.** Euler tour: a cycle using every **edge** exactly once; efficient O(number of edges) algorithms have been known since 1873. Hamilton tour: a cycle using every **vertex** exactly once; no efficient algorithm is known, the best are exponential time. The moral is that you cannot judge a graph problem's difficulty from its statement.

### 11. The marked-set is reachability

**Q.** After `DepthFirstPaths` from `s` finishes, how do you answer "is there a path from `s` to `v`?" in constant time?

**A.** Check `marked[v]`. DFS marks exactly the vertices reachable from `s`, so `hasPathTo(v)` is just `marked[v]`.

### 12. Runtime of DFS *(extra context: not stated in these slides, but standard and commonly asked)*

**Q.** With adjacency lists, what is the running time of DFS over a graph with V vertices and E edges, and why?

**A.** O(V + E). Marking makes each vertex enter `dfs` at most once, and for each vertex we scan its adjacency list once, so the total adjacency work is proportional to the sum of degrees, which is 2E for an undirected graph.

### 13. Reconstructing a tree from traversals *(extra context: a classic exam variant built on this lecture's material)*

**Q.** A binary tree has preorder `D B A C F E G` and inorder `A B C D E F G`. What is its root, and what are its left and right subtrees?

**A.** Preorder's first element is the root, `D`. In the inorder, everything before `D` (`A B C`) is the left subtree and everything after (`E F G`) is the right subtree. Recursing with the next preorder elements gives `B` over `A` and `C`, and `F` over `E` and `G`, which is the lecture's tree.

---

## Summary

- A **tree** is nodes plus edges with **exactly one path between any two nodes**; a **rooted** tree names a root, giving every other node exactly one parent, with childless nodes as leaves.
- Iterating a tree is called **traversal**, because there are many useful orders: **level order** (top to bottom, left to right) plus the depth-first **preorder** (visit, then children), **inorder** (left, visit, right), and **postorder** (left, right, visit).
- For `D(B(A,C), F(E,G))`: preorder `DBACFEG`, inorder `ABCDEFG`, postorder `ACBEGFD`, level order `DBFACEG`. Inorder of a BST is sorted.
- The three depth-first traversals are the **same code** with the action moved relative to the recursive calls. Choose the order by what the action depends on: nothing → preorder (print a directory listing); the children's results → postorder (sum file sizes).
- Human tracing trick: walk the outline counter-clockwise from the top; visit on the left for preorder, the bottom for inorder, the right for postorder.
- **Traversing** a node (passing through it) is not **visiting** it (acting on it).
- A **graph** is nodes plus zero or more edges, with no path-uniqueness constraint. Every tree is a graph; a tree is exactly a connected acyclic graph. A **simple graph** has no self-loops and no parallel edges, and in 61B "graph" means "simple graph".
- Key vocabulary: directed, undirected, cyclic, acyclic, labels/weights, adjacent, path, simple path, cycle, connected (pair) and connected (graph).
- Famous problems: s-t path, connectivity, biconnectivity, shortest s-t path, cycle detection, Euler tour, Hamilton tour, planarity, isomorphism. Difficulty is not predictable from the statement: Euler tour is O(E) and solved in 1873; Hamilton tour has no known efficient algorithm.
- The naive recursive `connected(s, t)` loops forever on cycles. **Marking** vertices so each is visited at most once fixes it, and that is exactly **depth-first search**: explore one neighbor's entire subgraph before moving to the next.
- `DepthFirstPaths` adds `edgeTo[w] = v` when discovering `w` from `v`. The array encodes a DFS tree rooted at `s`; read it **backwards** from a destination to recover a path. `marked[v]` answers reachability.
- On the lecture's 9-vertex graph from source 0: DFS preorder `012543678` (order of calls), DFS postorder `347685210` (order of returns), BFS order `0 | 1 | 2 4 | 5 3 | 6 8 | 7`.
- Graph traversals parallel tree traversals: DFS preorder ↔ preorder, DFS postorder ↔ postorder, BFS ↔ level order. There is no general inorder analogue.
- DFS gives *a* path, not a shortest one. Shortest paths need BFS and some extra data structure, which is the challenge posed for next lecture.
- Universal pattern: pick a traversal, then **do an action or set an instance variable at the right moment** in it. That single pattern solves connectivity, path finding, file sizes, and much more.
