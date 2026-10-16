<!-- Fri, Oct 16, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 22: Graph Traversals and Implementations

## Overview

This lecture moves from graph algorithms as pictures to graph algorithms as code. We adopt the Princeton `Graph` API (integer-numbered vertices, `addEdge`, `adj`, `V`, `E`), write two small "client" programs against it (`degree` and `print`), then implement depth first search as a class `DepthFirstPaths` that solves the all-paths problem from a single source `s` by filling in two arrays, `marked[]` and `edgeTo[]`. To talk about runtime at all, we need a concrete representation, so we survey three: the adjacency matrix, the edge set, and the adjacency list (the one we will use by default). With adjacency lists, both `print` and DFS cost Θ(V + E) and O(V + E) respectively, and we see carefully why DFS cannot be called O(E) and usually cannot be called Θ(V + E). We then solve the all-*shortest*-paths problem with breadth first search, which replaces recursion with an explicit queue (the "fringe") and additionally tracks `distTo[]`. Finally we redo the runtime analysis under an adjacency matrix, where everything degrades to O(V²), which is the reason adjacency lists are the default for the rest of the course. A closing note: on Project 4B/C/D you design your own graph API and may not import the Princeton library.

---

## Key Concepts

### 1. An API plus a concrete representation = an implementable graph

To implement anything like `DepthFirstPaths` we need two separate things:

- **An API**: the set of `Graph` methods, their signatures, and their behaviors. This defines how a *client programmer* must think about graphs.
- **A concrete data structure** that actually stores the graph underneath.

The slides draw this as the same split we already know for sets: the abstraction `Set` has concrete implementations `HashSet` and `TreeSet`. Likewise the abstraction `Graph` has concrete implementations (adjacency matrix, edge set, adjacency list). Our choices on both levels have "profound implications" on three things:

- runtime,
- memory usage,
- difficulty of implementing various graph algorithms.

That last point is easy to overlook. The API you pick decides what is easy and what is painful for a client to write, independent of speed.

### 2. Design decision #1: vertices are integers

The common convention (and the one the Princeton API takes) is to **number vertices 0 through V-1 and ignore labels entirely inside the graph**. If your real data has labels, you keep a side `Map<Label, Integer>` to translate.

The lecture's example: a graph on Austin, Dallas, Houston becomes vertices 0, 1, 2, with
```
Map<String, Integer>:  Austin: 0,  Dallas: 1,  Houston: 2
```
"Intended graph" versus "how you'd build it." The benefit is that every per-vertex piece of bookkeeping becomes an array indexed by vertex number: `marked[v]`, `edgeTo[v]`, `distTo[v]`. That is why DFS and BFS code below is so short.

### 3. The Princeton Graph API, and what it deliberately leaves out

```java
public class Graph {
    public Graph(int V):               // create empty graph with V vertices
    public void addEdge(int v, int w): // add an edge v-w
    Iterable<Integer> adj(int v):      // vertices adjacent to v
    int V():                           // number of vertices
    int E():                           // number of edges
    ...
}
```

Features (really, limitations) the lecture calls out explicitly:

- **The number of vertices must be specified in advance.** No `addVertex`.
- **No weights or labels** on vertices or edges.
- **No method for the degree of a vertex.** You must count by iterating `adj(v)` yourself.

The degree omission is the concrete illustration of "choice of API determines how the client has to think": something that could have been one method call becomes a five-line loop in every client that needs it.

### 4. Design pattern: decouple the graph type from the processing algorithm

This is the central software-design idea of the lecture. DFS is **not** a method on `Graph`. Instead:

1. Create a graph object.
2. Pass the graph to a graph-processing method or constructor in a **client class**.
3. Query that client class for the information you want.

```java
public class Paths {
    public Paths(Graph G, int s):      // find all paths from s in G
    boolean hasPathTo(int v):          // is there a path from s to v?
    Iterable<Integer> pathTo(int v):   // path from s to v (if any)
}
```

Usage:
```java
Paths P = new Paths(G, 0);
P.hasPathTo(3);   // true
P.pathTo(3);      // {0, 1, 4, 3}
```

Notice the shape: **all the real work happens in the constructor**, which runs the traversal once and stores the results in arrays. After that, `hasPathTo` and `pathTo` are cheap queries. The constructor's job is to leave the results "in an easy to query manner."

The lecture's author notes contrast this with a common student instinct: putting `dfs` directly inside the `Graph` class, which then forces you to thread mutable helper arguments (visited sets, parent maps) through helper methods. The client-class pattern gives those pieces of state a natural home as instance fields. (The slide deck flags this contrast as a pattern it wants to require on exams, so it is worth internalizing.)

### 5. The three representations

**Adjacency matrix.** A V-by-V matrix where entry `(v, w)` is 1 if there is an edge from `v` to `w`.
- Directed example on vertices 0, 1, 2 with edges 0→1, 0→2, 1→2:
  row 0 = `0 1 1`, row 1 = `0 0 1`, row 2 = `0 0 0`.
- For an **undirected** graph each edge appears **twice**, once at `(v, w)` and once at `(w, v)`. The matrix is symmetric. The lecture's phrasing: "Simplicity at the expense of space."
- Undirected example (path 0-1-2-3): row 0 = `0 1 0 0`, row 1 = `1 0 1 0`, row 2 = `0 1 0 1`, row 3 = `0 0 1 0`.

**Edge sets** (marked "Extra" in the deck). Just a collection of all the edges, for example `HashSet<Edge>` where each `Edge` is a pair of ints: `{(0,1), (0,2), (1,2)}`. Compact, but `adj(v)` requires scanning every edge.

**Adjacency lists.** An array of lists indexed by vertex number. For the triangle-ish graph above: index 0 holds `[1, 2]`, index 1 holds `[2]`, index 2 holds the empty list. This is "the most popular approach for representing graphs" and is **efficient when graphs are sparse** (not too many edges).

The deck frames this as the same menu we already met for trees: a tree can be stored with fixed child links (representation 1a) or as an array of keys (representation 3), where the array version "uses much less memory and operations will tend to be faster, but only works for complete trees." Same lesson: representation choice buys speed or generality, rarely both.

*(Extra context: a standard Java adjacency list is `List<Integer>[] adj` or `ArrayList<ArrayList<Integer>>`; in box-and-pointer terms it is one array of V references, each pointing to a separate list object on the heap. The adjacency matrix by contrast is one `boolean[V][V]`, which is one array of V references to V-length rows, so V² cells exist whether or not the edges do.)*

### 6. Why the runtime is Θ(V + E), and how to read that

For `print` on an adjacency list, the lecture walks through two different levels of analysis.

First, the crude bound. Iterating over `v`'s neighbors costs between Ω(1) and O(V), because `v`'s list can hold anywhere from 1 to V items. We do this for V vertices. That gives **best case Θ(V), worst case Θ(V²)**. Both of those are correct statements about specific families of graphs, but neither describes the code in general.

Then the sharp bound. Pick a cost model: count `v += 1` operations plus `println` calls.
- `v += 1` happens exactly **V** times (once per vertex in the outer loop).
- `println` happens exactly **2E** times, because in an undirected adjacency list every edge appears in two lists.

So the total is **Θ(V + E) in all cases**. The key realization is that although any individual `adj(v)` iteration varies wildly in length, the *sum* of all the iteration lengths is pinned down exactly: it is the total number of adjacency-list entries, which is 2E.

How to interpret a two-variable bound: "No matter what family of increasingly complex graphs we generate, as V and E grow, the runtime will always grow exactly as Θ(V + E)." The lecture gives two families:

- **Very sparse**: every vertex connected to its square (2-4, 3-9, 4-16, 5-25, ...). Then E is Θ(√V), so the runtime is Θ(V + √V) = **Θ(V)**.
- **Very dense**: every vertex connected to every other. Then E is Θ(V²), so the runtime is Θ(V + V²) = **Θ(V²)**.

Both are instances of the single claim Θ(V + E). Also: **Θ(V + E) is equivalent to Θ(max(V, E))**; both are technically correct and the sum is used more often. The deck adds an honest caveat: we never formally defined asymptotics on multiple variables, and it turns out to be somewhat poorly defined (it links Howell's note on the subject).

### 7. BFS: replacing recursion with a queue

DFS gets its order from the call stack. BFS needs the opposite order, so it keeps an explicit **queue** called the **fringe**.

The lecture first poses this as a challenge with hints: visit vertices in BFS order, you will need some kind of data structure, and **do not use recursion**. The answer:

- Initialize a queue with the starting vertex `s` and **mark** `s`.
- Repeat until the queue is empty:
  - Remove vertex `v` from the front of the queue.
  - For each unmarked neighbor `n` of `v`:
    - Mark `n`.
    - Set `edgeTo[n] = v` (and/or `distTo[n] = distTo[v] + 1`).
    - Add `n` to the end of the queue.

A queue supports enqueue (addLast) and dequeue (removeFirst). The deck's aside: "A queue is the opposite of a stack. Stack has push (addFirst) and pop (removeFirst)."

The invariant that makes BFS correct for shortest paths is noted during the demo: **the distance to all items on the queue is always k or k + 1 for some k.** The fringe is sorted by distance and spans at most two consecutive levels, so the first time you reach a vertex you have reached it by a shortest path. That is exactly why `distTo[n] = distTo[v] + 1` is safe to write once and never revise.

### 8. Traversal orders: graph traversals generalize tree traversals

Just as a tree has preorder, inorder, postorder, and level order, a graph has several traversal orders relative to a source:

- **DFS preorder**: the order of `dfs` *calls*. On the lecture's 9-vertex demo graph from source 0: `0 1 2 5 4 3 6 7 8`.
- **DFS postorder**: the order of `dfs` *returns*. Same graph: `3 4 7 6 8 5 2 1 0`.
- **BFS order**: act in order of distance from `s`. Same graph: `0 1 24 53 68 7` (grouped by distance level). BFS is "analogous to level order. Search is wide, not deep."

The tree example given for comparison: preorder `DBACFEG`, inorder `ABCDEFG`, postorder `ACBEGFD`, level order `DBFACEG`.

---

## Definitions

- **API (Application Programming Interface)**: for our purposes, the set of `Graph` methods including their signatures and behaviors. It defines how `Graph` client programmers must think.
- **Client (client class / client program)**: code that uses the `Graph` API without knowing its internals. `print`, `degree`, `DepthFirstPaths`, and `BreadthFirstPaths` are all clients.
- **Degree of a vertex v**: the number of edges incident to `v`. Not provided by the Princeton API; computed by counting `adj(v)`.
- **Adjacency matrix**: a V-by-V matrix where entry `(v, w)` is 1 (true) if edge `v-w` is present, 0 otherwise. For undirected graphs, each edge is represented twice, making the matrix symmetric.
- **Edge set**: a collection of all edges in the graph, for example `HashSet<Edge>` where each `Edge` is a pair of ints. (Marked "Extra" in the lecture.)
- **Adjacency list**: an array of lists indexed by vertex number, where list `v` contains the neighbors of `v`. Efficient for sparse graphs; the default representation in this course.
- **Sparse graph**: a graph without too many edges (the number of edges is much less than V for most vertices).
- **`marked[v]`**: boolean array; `marked[v]` is true if and only if `v` is connected to `s` (more precisely, if and only if the traversal has reached `v`).
- **`edgeTo[v]`**: int array; `edgeTo[v]` is the previous vertex on the path from `s` to `v`. The array of `edgeTo` values encodes a tree of paths rooted at `s`.
- **`distTo[v]`** (BFS only): int array; the number of edges on the shortest path from `s` to `v`.
- **Fringe**: the queue of discovered-but-not-yet-processed vertices in BFS.
- **Queue**: a list with two operations, enqueue (addLast) and dequeue (removeFirst). FIFO.
- **Stack**: a list with push (addFirst) and pop (removeFirst). LIFO; the opposite ordering discipline from a queue.
- **DFS preorder**: the sequence of vertices in the order their `dfs` calls begin.
- **DFS postorder**: the sequence of vertices in the order their `dfs` calls return.
- **BFS order**: the sequence of vertices in nondecreasing order of distance from the source.
- **s-t paths problem**: find a path from `s` to every reachable vertex. Solved by `DepthFirstPaths`.
- **s-t shortest paths problem**: find a *shortest* path from `s` to every reachable vertex. Solved by `BreadthFirstPaths`.

---

## Worked Examples

### Example 1: the `degree` client

```java
/** degree of vertex v in graph G */
public static int degree(Graph G, int v) {
    int degree = 0;
    for (int w : G.adj(v)) {
        degree += 1;
    }
    return degree;
}
```

**What it does.** `G.adj(v)` returns an `Iterable<Integer>`, so the enhanced for loop walks every neighbor of `v`. We do not care what `w` is, only how many there are, so the body just increments a counter.

**Why it looks like this.** The API has no degree method, so the only way to learn the degree is to count. On the lecture's small graph (vertices 0, 1, 2, 3 with edges 0-1, 0-3, 1-2), `degree(G, 1) = 2` because `adj(1)` yields `0` and `2`.

**Runtime.** Θ(degree(v)) with an adjacency list; Θ(V) with an adjacency matrix, since the matrix row must be scanned in full even if the vertex has one neighbor.

### Example 2: the `print` client

Challenge posed in lecture: write a client method `print` that prints out a graph, producing

```
$ java printDemo
0 - 1
0 - 3
1 - 0
1 - 2
2 - 1
3 - 0
```

Answer:

```java
public static void print(Graph G) {
    for (int v = 0; v < G.V(); v += 1) {
        for (int w : G.adj(v)) {
            System.out.println(v + "-" + w);
        }
    }
}
```

**Step by step.** The outer loop relies on the integer-vertex convention: vertices are exactly `0 .. G.V()-1`, so a plain `int` loop enumerates all of them. The inner loop prints one line per adjacency-list entry. For the graph with edges 0-1, 0-3, 1-2 the output above has 6 lines for 3 edges, which is the visible evidence that **each undirected edge appears twice**, once from each endpoint.

**Runtime with an adjacency list: Θ(V + E).** Cost model: `v += 1` operations plus `println` calls. The outer increment runs V times; the print runs 2E times. Note the trap: the "obvious" answers Θ(V) and Θ(V²) are the best and worst case over families of graphs, not the correct general bound.

**Runtime with an adjacency matrix: Θ(V²).** Here `G.adj(v)` must scan the whole row of length V to answer `hasNext()` and `next()`, even for a vertex with no neighbors. So we pay Θ(V) per vertex, V times. (For the matrix on 0-1-2-3, `G.adj(2)` returns an iterator whose `next()` yields `1`, then `3`, but the underlying code iterated through the entire length-4 row to find them.)

### Example 3: `DepthFirstPaths`, recursive implementation

```java
public class DepthFirstPaths {
    private boolean[] marked;   // marked[v] is true iff v connected to s
    private int[] edgeTo;       // edgeTo[v] is previous vertex on path from s to v
    private int s;

    public DepthFirstPaths(Graph G, int s) {
        ...                     // not shown: data structure initialization
        dfs(G, s);              // find vertices connected to s
    }

    private void dfs(Graph G, int v) {
        marked[v] = true;
        for (int w : G.adj(v)) {
            if (!marked[w]) {
                edgeTo[w] = v;
                dfs(G, w);
            }
        }
    }
    ...
}
```

**Line by line.**
- `marked[v] = true;` happens **first**, on entry. This is what stops the recursion from cycling forever: a vertex is claimed the instant we arrive.
- The loop considers each neighbor `w` of `v`. If `w` is already marked, we skip it entirely (no recursive call, no `edgeTo` write).
- `edgeTo[w] = v;` records "I reached `w` from `v`." We write it *before* recursing, which matters only for readability; what matters for correctness is that we write it exactly once, at the moment of discovery.
- `dfs(G, w);` goes deeper immediately. That immediacy is what makes this depth first.

**Environment / box-and-pointer reasoning in words.** `marked` and `edgeTo` are instance fields of the `DepthFirstPaths` object, so every frame of the recursion sees and mutates the *same two arrays*; there is one `boolean[]` box of length V and one `int[]` box of length V on the heap, and each `dfs` frame holds only a copy of the reference `G` plus the local ints `v` and `w`. Nothing needs to be returned, and no mutable helper argument needs to be threaded through, precisely because the state lives on the enclosing object rather than in the parameters. Each nested `dfs` call pushes a new frame, so **the call stack can grow to depth V** on a long path graph. *(Extra context: that is a real limitation of recursive DFS in Java, a deep graph can overflow the stack, which is one reason the iterative, explicit-stack version exists.)*

**Trace on the lecture's 9-vertex demo graph.** *(Reconstructed from the lecture's stated traversal orders and BFS table; the diagram itself was lost in PDF extraction. The edges are 0-1, 1-2, 1-4, 2-5, 4-5, 4-3, 5-6, 5-8, 6-7, which reproduce every order and every table value the slides report.)*

Starting `dfs(G, 0)` with neighbors visited in increasing order: mark 0, go to 1, mark 1, go to 2 (its smallest unmarked neighbor), mark 2, go to 5, mark 5, go to 4, mark 4, go to 3, mark 3. Vertex 3's only neighbor is 4, already marked, so `dfs(3)` **returns**. Back in 4: neighbor 5 is marked, so `dfs(4)` returns. Back in 5: visit 6, then from 6 visit 7, 7 returns, 6 returns, back in 5 visit 8, 8 returns, 5 returns, 2 returns, 1 returns, 0 returns.

- Call order (preorder): `0 1 2 5 4 3 6 7 8` ✓ matches the slide.
- Return order (postorder): `3 4 7 6 8 5 2 1 0` ✓ matches the slide.
- Resulting `edgeTo`: `edgeTo[1]=0, edgeTo[2]=1, edgeTo[5]=2, edgeTo[4]=5, edgeTo[3]=4, edgeTo[6]=5, edgeTo[7]=6, edgeTo[8]=5`.

Note that DFS found the path 0 → 1 → 2 → 5 → 4 → 3 to vertex 3, of length 5, while BFS will find 0 → 1 → 4 → 3, of length 3. **DFS gives you a path, not a shortest path.**

### Example 4: `pathTo` and `hasPathTo`

The lecture poses this as a "question to ponder" before answering:

```java
public Iterable<Integer> pathTo(int v) {
    if (!hasPathTo(v)) return null;
    List<Integer> path = new ArrayList<>();
    for (int x = v; x != s; x = edgeTo[x]) {
        path.add(x);
    }
    path.add(s);
    Collections.reverse(path);
    return path;
}

public boolean hasPathTo(int v) {
    return marked[v];
}
```

**What it does and why.** `hasPathTo` is a one-liner because the traversal already answered that question: `marked[v]` is true exactly when the search reached `v`.

`pathTo` **walks the `edgeTo` array backwards.** Think of `edgeTo` as a parent-pointer tree rooted at `s`: following `x = edgeTo[x]` repeatedly is climbing from `v` toward the root. The loop condition `x != s` stops before adding the source, which is why `path.add(s)` appears separately right after. Because we collected vertices from `v` up to `s`, the list is backwards, so `Collections.reverse(path)` flips it in place to read `s` first. Returning `null` when unreachable is the API's chosen contract.

Walking the example: `pathTo(3)` on the DFS result above collects 3, 4, 5, 2, 1 (stopping when `x` becomes 0), appends 0 to get `[3, 4, 5, 2, 1, 0]`, reverses to `[0, 1, 2, 5, 4, 3]`. On the lecture's other example, `P.pathTo(3)` returns `{0, 1, 4, 3}` for a graph where DFS happened to descend that way.

**Runtime.** Θ(length of the path), so O(V). *(Extra context: `Collections.reverse` mutates the `ArrayList` in place and returns `void`, so `return Collections.reverse(path);` would not compile.)*

### Example 5: runtime of the `DepthFirstPaths` constructor (adjacency list)

**Answer: O(V + E).**

Cost model: the number of `dfs` calls plus the number of `marked[w]` checks.

- **Each vertex is visited at most once**, because `marked[v] = true` is set on entry and we never recurse into a marked vertex. So there are no more than **V** `dfs` calls, giving O(V).
- **Each edge is considered at most twice**, once from each endpoint's adjacency list. Each consideration is a constant-time `marked[w]` check. So no more than **2E** checks, giving O(E).

Total O(V + E).

**The very hard follow-up: could we say the runtime is O(E)?**

The tempting argument: you can only visit a vertex if there is an edge to it, so the number of `dfs` calls is itself bounded above by E, and the V term is absorbed.

**The answer is no.** The constructor has to **create an all-false `marked` array**, and marking all vertices as false takes Θ(V) time regardless of how few edges exist. Consider a graph with 1,000,000 vertices and a source with no edges: there is one `dfs` call and zero edge checks, but the constructor still touched a million array slots. The deeper lesson the slide states: "Our cost model earlier (dfs calls + marked checks) does not provide a tight bound." A cost model is a choice, and if your chosen operations exclude real work, your conclusion can be wrong. Array initialization was outside the model.

**And why not Θ(V + E)?** Because the lower bound fails for disconnected graphs. If the source has no edges touching it, the traversal does Θ(V) work (initialization) and no edge work at all, so the runtime is not proportional to E. We can only tighten to Θ(V + E) when the graph is connected.

**Space: Θ(V).** We need arrays of length V to store `marked` and `edgeTo`, no matter how few edges there are.

### Example 6: BFS by hand on the demo graph

The lecture runs a long slide-by-slide demo with a table of `marked`, `edgeTo`, `distTo` and the fringe contents. Here is the full trace, source `s = 0`:

| step | dequeued `v` | newly marked (with `edgeTo`, `distTo`) | fringe after |
|---|---|---|---|
| init | - | 0 (`-`, 0) | `[0]` |
| 1 | 0 | 1 (`0`, 1) | `[1]` |
| 2 | 1 | 2 (`1`, 2), 4 (`1`, 2) | `[2, 4]` |
| 3 | 2 | 5 (`2`, 3) | `[4, 5]` |
| 4 | 4 | 3 (`4`, 3) | `[5, 3]` |
| 5 | 5 | 6 (`5`, 4), 8 (`5`, 4) | `[3, 6, 8]` |
| 6 | 3 | nothing to add | `[6, 8]` |
| 7 | 6 | 7 (`6`, 5) | `[8, 7]` |
| 8 | 8 | nothing to add | `[7]` |
| 9 | 7 | nothing to add | `[]` |

Final state, matching the last slide of the demo:

| # | marked | edgeTo | distTo |
|---|---|---|---|
| 0 | T | - | 0 |
| 1 | T | 0-1 | 1 |
| 2 | T | 1-2 | 2 |
| 3 | T | 4-3 | 3 |
| 4 | T | 1-4 | 2 |
| 5 | T | 2-5 | 3 |
| 6 | T | 5-6 | 4 |
| 7 | T | 6-7 | 5 |
| 8 | T | 5-8 | 4 |

Two things to notice. First, at step 5 the slide annotates: "distance to all items on queue is always k or k + 1 for some k. Here k = 3." Look at the fringe `[3, 6, 8]`: distances 3, 4, 4. The queue never holds three different levels. Second, "nothing to add" happens when every neighbor of the dequeued vertex is already marked (vertex 3's only neighbor is 4, already marked at step 4), which is the normal case near the end of a traversal, not an error.

The BFS visit order, grouped by level, is `0 | 1 | 2 4 | 5 3 | 6 8 | 7`, exactly the slide's `0 1 24 53 68 7`.

### Example 7: `BreadthFirstPaths` implementation

```java
public class BreadthFirstPaths {
    private boolean[] marked;   // marked[v] is true iff v connected to s
    private int[] edgeTo;       // edgeTo[v] is previous vertex on path from s to v
    ...

    private void bfs(Graph G, int s) {
        Queue<Integer> fringe = new Queue<Integer>();
        fringe.enqueue(s);
        marked[s] = true;
        while (!fringe.isEmpty()) {
            int v = fringe.dequeue();
            for (int w : G.adj(v)) {
                if (!marked[w]) {
                    fringe.enqueue(w);
                    marked[w] = true;
                    edgeTo[w] = v;
                }
            }
        }
    }
}
```

**Step by step.** Setup enqueues `s` and marks it. The `while` loop is the "repeat until fringe is empty" line of the algorithm. For each freshly dequeued vertex `v`, for each **unmarked** neighbor `w`, we do three things together: enqueue it, mark it, and set `edgeTo[w] = v`. The version that also tracks distance adds `distTo[w] = distTo[v] + 1` in the same block.

**Why the structure differs from DFS.** Structurally this is DFS with the call stack replaced by an explicit queue, and that single swap changes stack-like LIFO ordering into FIFO ordering, which is what makes the search go wide instead of deep. Everything else (the `marked` guard, the `edgeTo` bookkeeping, the `pathTo` reconstruction, which is identical and inherited in spirit from `DepthFirstPaths`) is unchanged.

**Why marking happens at enqueue time, not dequeue time.** This is the subtle correctness point. A vertex can be the neighbor of several vertices on the current level. If you only marked on dequeue, the same vertex could be enqueued several times before it is ever processed, which inflates the queue and can overwrite `edgeTo`/`distTo` with a worse value. Marking at enqueue guarantees each vertex enters the fringe exactly once, which is also what keeps the edge-check count at 2E.

**Runtime: O(V + E)** with an adjacency list, by the same cost model as DFS (O(V) `.next()` calls over vertices plus O(E) `marked[w]` checks). **Space: Θ(V)** for the arrays. *(Extra context: the fringe itself also holds at most V vertices, so it does not change the Θ(V) space bound.)*

### Example 8: runtime of `DepthFirstPaths` with an adjacency matrix

**Answer: O(V²).** The slide itself says "This is a lot to digest! If you are feeling lost on this problem, don't feel bad. But do work on trying to understand the ideas here!"

The reasoning: in the worst case we iterate over the neighbors of all vertices. We create at most V iterators (one per `dfs` call), and **each one takes Θ(V) time to iterate over**, because the underlying code must scan an entire matrix row to answer `next()` and `hasNext()`. V iterators times Θ(V) each is O(V²). Put differently: iterating over the entire adjacency matrix takes O(V²) time, and that is what the traversal ends up doing.

Note what changed and what did not. The *number* of vertex visits and edge discoveries is identical to the adjacency-list case. What changed is the cost of asking "who are `v`'s neighbors?", which went from "proportional to the answer" to "proportional to V."

---

## Summary table of graph problems

With adjacency lists (the default):

| Problem | Description | Solution | Efficiency (adj. list) |
|---|---|---|---|
| s-t paths | Find a path from `s` to every reachable vertex | `DepthFirstPaths.java` | O(V+E) time, Θ(V) space |
| s-t shortest paths | Find a *shortest* path from `s` to every reachable vertex | `BreadthFirstPaths.java` | O(V+E) time, Θ(V) space |

With adjacency matrices:

| Problem | Solution | Efficiency (adj. matrix) |
|---|---|---|
| s-t paths | `DepthFirstPaths.java` | O(V²) time, Θ(V) space |
| s-t shortest paths | `BreadthFirstPaths.java` | O(V²) time, Θ(V) space |

"For sparse graphs (number of edges much less than V for most vertices), this is terrible runtime. Thus, we'll always use adjacency-list unless otherwise stated."

---

## Common Pitfalls

1. **Answering Θ(V) or Θ(V²) for `print` on an adjacency list.** Those are the best and worst cases over graph families. The correct all-cases answer is Θ(V + E). The fix is to count total adjacency-list entries (2E) rather than multiplying "V vertices" by "up to V neighbors."

2. **Claiming DFS is O(E).** The number of `dfs` calls really is bounded by E, but the constructor must zero out a length-V `marked` array, which is Θ(V) unconditionally. Any cost model that ignores initialization will mislead you.

3. **Claiming DFS is Θ(V + E) without conditions.** The tight two-sided bound requires the graph to be connected. A graph where no edge touches the source does Θ(V) work and ignores all E edges.

4. **Forgetting that an undirected edge appears twice.** Twice in the adjacency lists (so 2E list entries, so `print` emits 2E lines) and twice in the adjacency matrix (so the matrix is symmetric). Every runtime count in this lecture relies on the factor of 2.

5. **Marking at dequeue time in BFS.** Marks must be set when a vertex is *enqueued*. Marking on dequeue lets the same vertex be queued multiple times and can corrupt `edgeTo`/`distTo`.

6. **Marking after the recursive call in DFS.** `marked[v] = true` must be the first statement of `dfs`, before the neighbor loop. Set it later and a cycle sends you into infinite recursion.

7. **Expecting DFS to produce shortest paths.** On the demo graph DFS reaches vertex 3 via a path of 5 edges; BFS reaches it in 3. DFS solves s-t paths, BFS solves s-t *shortest* paths (for unweighted graphs).

8. **Assuming a unique DFS/BFS order.** The order depends on the order `adj(v)` yields neighbors, which is a property of the representation, not of the graph. Exam answers usually say "visit neighbors in increasing numerical order" or similar; look for that instruction.

9. **Expecting an `addVertex` or `degree()` method.** The Princeton API fixes V at construction and has no degree method. Writing `G.degree(v)` on an exam is a correctness error against *this* API.

10. **Confusing vertex numbers with labels.** The graph stores only ints; the `Map<Label, Integer>` lives outside the graph. If a question hands you city names, you need the map.

11. **Putting `dfs` inside the `Graph` class.** This is the pattern the lecture is steering you away from. It forces awkward mutable helper parameters. Use the separate client class whose constructor runs the traversal and whose fields hold the results.

12. **Off-by-one in `pathTo`.** The loop condition is `x != s`, which excludes the source, hence the separate `path.add(s)`. Omit that line and the source is missing from the returned path. And remember `Collections.reverse` is void and in-place.

13. **Assuming `adj` is cheap.** On an adjacency matrix, `adj(v)` costs Θ(V) even if `v` is isolated. This is the single fact that turns O(V + E) into O(V²).

---

## Likely Exam Points

### 1. Runtime of a client loop under a stated representation

Commonly tested: given a nested loop over `G.V()` and `G.adj(v)`, give the order of growth under adjacency list versus adjacency matrix.

**Practice:** Give the order of growth of the running time of
```java
for (int v = 0; v < G.V(); v += 1)
    for (int w : G.adj(v))
        System.out.println(v + "-" + w);
```
under (a) an adjacency list, (b) an adjacency matrix.

**Answer:** (a) Θ(V + E) in all cases: V outer increments plus 2E prints. (b) Θ(V²): each of V calls to `adj(v)` must scan a length-V row.

### 2. Why not O(E) for DFS

**Practice:** A student claims `DepthFirstPaths`'s constructor is O(E) because you can only visit a vertex if an edge leads to it. Give the correct bound and explain the flaw in one sentence.

**Answer:** The correct bound is O(V + E). The flaw is that the constructor must initialize the length-V `marked` array to all false, which costs Θ(V) regardless of E, so the V term cannot be dropped. (The original cost model of "dfs calls plus marked checks" simply did not count initialization.)

### 3. Θ versus O for traversals

**Practice:** Can we say the `DepthFirstPaths` constructor runs in Θ(V + E) time? Justify.

**Answer:** Not in general; only if the graph is connected. Counterexample: a graph where no edge touches the source. Then DFS makes one call and zero edge checks, so the runtime is Θ(V) and is not proportional to E.

### 4. Fill in the `marked` / `edgeTo` / `distTo` table for BFS

Essentially guaranteed in some form: given a drawn graph and a source, simulate BFS, report the fringe at each step and the final arrays.

**Practice:** On the demo graph (edges 0-1, 1-2, 1-4, 2-5, 4-5, 4-3, 5-6, 5-8, 6-7), BFS from 0 with neighbors considered in increasing order. What are the fringe contents right after vertex 5 is processed, and what is `distTo[7]`?

**Answer:** Just before processing 5 the fringe is `[5, 3]`; dequeuing 5 marks 6 and 8 with `edgeTo = 5` and `distTo = 4`, so the fringe becomes `[3, 6, 8]`. `distTo[7] = 5`, with `edgeTo[7] = 6`.

### 5. Reconstruct a path from `edgeTo`

**Practice:** Given `s = 0` and `edgeTo = [-, 0, 1, 4, 1, 2, 5, 6, 5]` (index 0 unused), what does `pathTo(7)` return?

**Answer:** Walk backwards from 7: 7, `edgeTo[7]=6`, `edgeTo[6]=5`, `edgeTo[5]=2`, `edgeTo[2]=1`, `edgeTo[1]=0` which equals `s` so the loop stops. Collected `[7, 6, 5, 2, 1]`, add `s` to get `[7, 6, 5, 2, 1, 0]`, reverse: **`[0, 1, 2, 5, 6, 7]`**.

### 6. DFS preorder versus postorder versus BFS order

**Practice:** On the same demo graph from source 0 (increasing neighbor order), give the DFS preorder, the DFS postorder, and the BFS order.

**Answer:** Preorder (call order) `0 1 2 5 4 3 6 7 8`. Postorder (return order) `3 4 7 6 8 5 2 1 0`. BFS order `0, 1, 2, 4, 5, 3, 6, 8, 7`. Postorder is not simply the reverse of preorder, so both must be derived from the actual recursion.

### 7. Write a client against the API

**Practice:** Using only `V()`, `E()`, and `adj(v)`, write a method that returns the maximum degree in `G`.

**Answer:**
```java
public static int maxDegree(Graph G) {
    int max = 0;
    for (int v = 0; v < G.V(); v += 1) {
        int d = 0;
        for (int w : G.adj(v)) { d += 1; }
        if (d > max) { max = d; }
    }
    return max;
}
```
Runtime Θ(V + E) with adjacency lists, Θ(V²) with an adjacency matrix. The inner counting loop is needed because the API provides no `degree` method.

### 8. Converting a graph between representations

**Practice:** Give the adjacency matrix and the adjacency lists for the undirected graph with vertices 0, 1, 2, 3 and edges 0-1, 1-2, 2-3.

**Answer:** Matrix rows: `0: 0 1 0 0`, `1: 1 0 1 0`, `2: 0 1 0 1`, `3: 0 0 1 0` (symmetric, 6 ones for 3 edges). Lists: `0: [1]`, `1: [0, 2]`, `2: [1, 3]`, `3: [2]` (6 entries total, that is 2E).

### 9. Space bounds

**Practice:** What is the space complexity of `BreadthFirstPaths` (the client, not the graph), and why is it not sensitive to E?

**Answer:** Θ(V). It stores `marked`, `edgeTo`, and optionally `distTo`, each an array of length V, plus a fringe holding at most V vertices. No per-edge storage is kept, so E does not appear.

### 10. DFS versus BFS, choosing the right one

**Practice:** You need the fewest-hops route between two subway stations in an unweighted map. Which traversal, and what does it cost on an adjacency list?

**Answer:** BFS, because it visits vertices in order of distance from the source and so sets `distTo` to the true shortest distance on first discovery. O(V + E) time, Θ(V) space. DFS would find a route but not necessarily a shortest one.

### 11. Interpreting Θ(V + E)

**Practice:** Simplify Θ(V + E) for (a) a graph where each vertex connects only to its square, (b) a complete graph.

**Answer:** (a) E is Θ(√V), so Θ(V + √V) = Θ(V). (b) E is Θ(V²), so Θ(V + V²) = Θ(V²). Both are consistent with the single claim Θ(V + E), which is also expressible as Θ(max(V, E)).

---

## Project 4B/C/D Note (from the lecture)

- On Project 4B you must come up with a **design**. This is a totally new kind of task and it will be hard. If, while implementing 4C, you find your design was bad, that is fine; the point of 4B was to get you thinking ahead.
- You **cannot import the Princeton Graphs library**. You design your own graph API with just the operations your project needs.
- You **may not need a separate `Graph` class** at all. Whether you have one is up to you.
- (The slides, written for a Spring offering, remind students that 4C is due shortly after the break and to start early. Check the Fall 2026 calendar for the actual dates.)

---

## Summary

- **Two things are needed to implement graph algorithms**: an API (defines how clients think) and a concrete representation (defines runtime and memory). Our choices affect runtime, memory usage, and how hard algorithms are to write.
- **The Princeton API**: `Graph(int V)`, `addEdge(v, w)`, `adj(v)`, `V()`, `E()`. Vertices are ints 0..V-1; use a `Map<Label, Integer>` for labels. V is fixed at construction, no weights, no `degree` method.
- **Design pattern**: decouple the graph type from the processing algorithm. Build the graph, hand it to a client class constructor (`DepthFirstPaths`, `BreadthFirstPaths`) which does all the work and stores results in arrays, then query with `hasPathTo` / `pathTo`.
- **Three representations**: adjacency matrix (simple, symmetric for undirected, Θ(V²) space), edge set (extra), and adjacency list (array of lists indexed by vertex, best for sparse graphs, our default).
- **DFS** marks on entry, records `edgeTo[w] = v` at discovery, and recurses immediately; `pathTo` climbs the `edgeTo` parent pointers backwards, appends `s`, and reverses.
- **BFS** replaces recursion with a queue (the fringe): enqueue and mark `s`; repeatedly dequeue `v` and, for each unmarked neighbor `n`, mark it, set `edgeTo[n] = v` and `distTo[n] = distTo[v] + 1`, and enqueue it. Mark at enqueue time. Fringe distances are always k or k+1.
- **DFS gives a path; BFS gives a shortest path** (unweighted).
- **Runtimes with adjacency lists**: `print` is Θ(V + E) in all cases (V increments plus 2E prints); DFS and BFS are O(V + E) time and Θ(V) space.
- **Not O(E)**: initializing the length-V `marked` array is Θ(V) unconditionally. **Not Θ(V + E)** unless the graph is connected.
- **Runtimes with adjacency matrices**: `print`, DFS, and BFS all become Θ(V²) / O(V²), because each `adj(v)` scans a full row. Terrible for sparse graphs, so **use adjacency lists unless told otherwise**.
- **Traversal orders generalize tree traversals**: DFS preorder (call order), DFS postorder (return order), BFS order (by distance, analogous to level order).
- **Θ(V + E) is the same as Θ(max(V, E))**; multi-variable asymptotics were never formally defined in this course and are somewhat poorly defined in general.
