<!-- Fri, Oct 16, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 22: Graph Traversals and Implementations

This lecture takes the graph traversal *ideas* from Lecture 21 and turns them into real, runnable Java. We start by reviewing DFS on a concrete demo graph, naming the three natural graph traversal orders (DFS preorder, DFS postorder, BFS order) by analogy to tree traversals. We then introduce the Princeton `Graph` API, which numbers vertices `0` through `V-1` and exposes only `addEdge`, `adj`, `V`, and `E`, and we see how that minimal API shapes client code (`degree`, `print`). The central design pattern is to decouple the graph from the algorithm: a separate class such as `DepthFirstPaths` or `BreadthFirstPaths` takes a graph plus a source in its constructor, precomputes `marked` and `edgeTo` arrays, and then answers queries like `hasPathTo(v)` and `pathTo(v)` in constant or output-proportional time without ever touching the graph again. We look at two concrete representations (adjacency matrix and adjacency list, plus edge sets as an aside), and we do careful asymptotic analysis: with an adjacency list, `print`, DFS, and BFS are all Θ(V + E) / O(V + E) time with Θ(V) space, while with an adjacency matrix they degrade to Θ(V²) / O(V²). Finally, BFS is presented as the shortest-path (fewest edges) traversal, implemented with an explicit queue called the *fringe* rather than recursion.

---

## Key Concepts

### 1. Reviewing DFS: what `marked` and `edgeTo` actually record

The lecture re-ran the DFS demo from Lecture 21 on this graph (source `s = 0`):

```
                    3
   0 --- 1 ------- 4 --- 6
   s      \        |      \
            2 --- 5 --- 7
                   \
                    8
```

(Reconstructed from the slide layout and the transcript walkthrough: the demo graph has vertices 0 through 8, with 0 adjacent to 1; 1 adjacent to 0, 2, 4; 2 adjacent to 1, 5; 4 adjacent to 1, 3, 5; 5 adjacent to 2, 4, 6, 8; 6 adjacent to 5, 7.)

DFS maintains two pieces of bookkeeping:

- **`marked[v]`**: `true` once we have *visited* `v`. This is what prevents infinite looping in a cyclic graph, and it is also exactly the answer to "is `v` reachable from `s`?"
- **`edgeTo[v]`**: the vertex we *came from* when we first reached `v`. Josh emphasized the reading of this: `edgeTo[1] = 0` means "one way to get to 1 is from 0."

The crucial insight is that `edgeTo` is a **parent pointer array encoding a tree** (the DFS tree / BFS tree). You can reconstruct a full path from `s` to `v` by following `edgeTo` backwards from `v` until you hit `s`, then reversing. In the transcript Josh walked an example where `edgeTo[2] = 4` and `edgeTo[4] = 0`, giving the path `0 -> 4 -> 2`, and stressed that **you never need to look at the graph again to do this**: the arrays alone contain the answer.

### 2. Graph traversal orders, by analogy to tree traversals

Trees have preorder, inorder, postorder, and level order. Graphs (from a source) have the analogous family:

- **DFS preorder**: the order in which `dfs(G, v)` is *called*. On the demo graph: `0 1 2 5 4 3 6 7 8`. This is the graph analogue of tree preorder ("visit the node as soon as you see it").
- **DFS postorder**: the order in which `dfs` calls *finish* / return. On the demo graph: `3 4 7 6 8 5 2 1 0`. The transcript walked through this carefully: vertex 3 is the first to be fully done (it runs out of unmarked neighbors), then 4 (both its remaining edges considered), then we are *not* done with 5 yet because 8 is unexplored, so 6 then 7, then back up: 7 done, 6 done, 8 done, 5 done, 2, 1, 0. Josh noted this order "is actually genuinely useful" for later algorithms, even though it looks like abstract trivia now.
- **BFS order**: visit in order of *distance from `s`*. On the demo graph: `0`, then `1`, then `{2, 4}` (both distance 2), then `{5, 3}` (distance 3), then `{6, 8}` (distance 4), then `7` (distance 5). The slide writes one such order as `0 1 24 53 68 7`, where the grouped digits are tied in distance and can appear in either order. This is the analogue of **level order**: "search is wide, not deep."

A student asked whether re-entering a vertex (e.g. bouncing back to 4 after finishing 3) should appear again in the preorder. Answer: no, DFS preorder is defined by the order in which calls are *initially made*. Josh mused that the "every time the vertex is active" sequence would be a different, unnamed order (he jokingly called it the "DFS active vertex order").

### 3. Why BFS needs a queue, not recursion

Josh's framing: in DFS, the data structure tracking "what to do next" is **implicit**, it is the Java call stack of recursive `dfs` invocations. For BFS we need an **explicit** data structure, because recursion naturally dives deep and BFS must stay shallow.

A **queue** is "the opposite of a stack": stack is `push`/`addFirst` + `pop`/`removeFirst` (LIFO), queue is `enqueue`/`addLast` + `dequeue`/`removeFirst` (FIFO). You already built a deque in Projects 1 and 2, so you have the machinery.

The queue in a traversal is called the **fringe**: "fringe" is an English word for the edge of a thing, and the fringe holds exactly the frontier of the region explored so far, the vertices discovered but not yet expanded.

The FIFO discipline is what guarantees correctness. Josh's memorable version: the fringe is "Taiwan style," a strictly orderly line where nobody cuts in. Because new vertices are only ever appended at the back, there is no way for a far vertex (like 3) to get ahead of nearer vertices (like 2, 4, 5), so vertices are dequeued in nondecreasing order of distance.

### 4. BFS gives *shortest* paths; DFS does not

This is the practical payoff. In the demo, BFS found `distTo[4] = 2` with path `0 -> 1 -> 4`. DFS on the same graph reached 4 only after `0 -> 1 -> 2 -> 5 -> 4`, a path of length 4. Josh's analogy: driving directions. "You don't want to find some depth-first direction somewhere; you want to radiate out from your source."

### 5. The fringe invariant

A property Josh explicitly flagged "for future reference": at any moment during BFS, every vertex on the fringe has distance either **k or k+1** for some single k. On the slide where the fringe is `[3, 6, 8]`, k = 3 (3 has distance 3; 6 and 8 have distance 4). This makes intuitive sense: you are always expanding at the current frontier, so the queue straddles at most two consecutive levels. (This invariant is exactly why FIFO suffices for unweighted shortest paths, and it is what breaks when edges get weights, which is next lecture's topic.)

### 6. API vs. concrete implementation

To implement a graph algorithm you need two separate things:

1. **An API**: the method signatures and behaviors. This "defines how Graph client programmers must think."
2. **A concrete data structure** underneath.

The slide draws the analogy to `Set` (the API/interface) with `HashSet` and `TreeSet` as concrete implementations. For graphs the concrete choices will be adjacency matrix, edge set, or adjacency list.

Both choices have "profound implications" on:
- Runtime
- Memory usage
- Difficulty of implementing various graph algorithms

That last one matters for your own coding time, not just the machine's: with the Princeton API, getting a vertex's degree takes a whole helper method with a loop, because no `degree` method exists.

### 7. Graph API decision #1: integer vertices

The Princeton convention is to **number vertices `0` through `V-1`** and use the integer everywhere internally, ignoring labels. If your real data has labels (city names), you keep a `Map<Label, Integer>` on the side:

```
Intended graph:  Austin --- Dallas,  Austin --- Houston
Numbering:       Austin: 0, Dallas: 1, Houston: 2
Map<String, Integer>: {"Austin" -> 0, "Dallas" -> 1, "Houston" -> 2}
```

Integer vertices are what make the `boolean[] marked` and `int[] edgeTo` arrays possible: you index directly by vertex number in constant time. (extra context: if vertices were arbitrary objects you would need `HashMap<Vertex, Boolean>` instead, which still works but costs hashing on every access.)

### 8. The Princeton `Graph` API and its deliberate limitations

```java
public class Graph {
    public Graph(int V):               // Create empty graph with V vertices
    public void addEdge(int v, int w): // add an edge v-w
    Iterable<Integer> adj(int v):      // vertices adjacent to v
    int V():                           // number of vertices
    int E():                           // number of edges
    ...
}
```

Features (really, restrictions) called out in lecture:
- **Number of vertices must be specified in advance**, like the weighted quick union object from the disjoint sets lecture. You cannot grow the graph.
- **No weights or labels** on nodes or edges.
- **No `degree(v)` method.**

`adj(v)` returns an `Iterable<Integer>`, not a `List<Integer>`. Josh stressed this: "you don't even know if it's a list or what, but it can be iterated over." That is good API design, it hides the representation, which is precisely why the same client code works over an adjacency list or an adjacency matrix with wildly different runtimes.

Important for your own work: this API is **one choice among many**. Lab 9 uses a different API. On Project 4B you design your own, and you **may not import the Princeton Graphs library**, because the point is to design an API with just the operations your project needs. You may not even need a separate `Graph` class at all; that is your call.

### 9. The "decouple type from processing algorithm" pattern

This is the single most important structural takeaway of the lecture, and the slide deck's TODO note shows the instructors specifically wanted to contrast it with the naive student approach of stuffing DFS into the `Graph` class itself (which then forces you to thread mutable helper arguments everywhere).

The pattern:
- Create a graph object.
- Pass the graph to a graph-processing method (or **constructor**) in a separate **client class**.
- Query the client class for information.

```java
public class Paths {
    public Paths(Graph G, int s):       // Find all paths from s
    boolean hasPathTo(int v):           // is there a path from s to v?
    Iterable<Integer> pathTo(int v):    // path from s to v (if any)
}
```

Josh's extended metaphor: the `Paths` object is a bodhisattva. On construction it "peruses every possible facet of the graph," gaining enlightenment about what is reachable and how, memorizes it in `marked` and `edgeTo`, and thereafter answers queries purely "from its own internal wisdom," never consulting the graph again.

```java
Paths P = new Paths(G, 0);
P.hasPathTo(3);  // returns true
P.pathTo(3);     // returns {0, 1, 4, 3}
```

The payoff: all the expensive work happens once, in the constructor. `hasPathTo` is then a single array read.

### 10. Three concrete graph representations

**Representation 1: Adjacency matrix.** A `V x V` 2D array of booleans (or 0/1 ints). `matrix[v][w] == 1` means there is an edge from `v` to `w`.

Directed example (edges 0->1, 0->2, 1->2):
```
      t: 0  1  2
s: 0     0  1  1
   1     0  0  1
   2     0  0  0
```

Undirected example (path 0-1, 1-2, 2-3):
```
      w: 0  1  2  3
v: 0     0  1  0  0
   1     1  0  1  0
   2     0  1  0  1
   3     0  0  1  0
```

For an undirected graph **each edge is represented twice**, giving a symmetric matrix: "simplicity at the expense of space."

The killer problem: to answer `adj(v)` you must scan the entire row. `G.adj(2)` on the matrix above returns an iterator yielding 1 then 3, but the underlying code "has to iterate through the entire array to handle `next()` and `hasNext()` calls," so iterating over one vertex's neighbors is **Θ(V)** regardless of how few neighbors it has.

**Representation 2 (marked "Extra" on the slide): Edge sets.** A single collection of all edges, e.g. `HashSet<Edge>` where each `Edge` is a pair of ints: `{(0,1), (0,2), (1,2)}`. Compact, but terrible for `adj(v)`, since you must filter the whole set.

**Representation 3: Adjacency lists.** Maintain an **array of lists indexed by vertex number**:
```
0 -> [1, 2]
1 -> [2]
2 -> []
```
This is described on the slide as the "most popular approach for representing graphs," efficient when graphs are **sparse** (not too many edges). Josh's motivating example: a graph of all cities in America. Almost none of the possible edges exist, so a matrix would be "mainly zeros," which is a waste of space. With an adjacency list, "if I want to know who my neighbors are, I just get the list back immediately," instead of asking "are you 0's neighbor? are you 0's neighbor?" V times.

Bottom line from the lecture: **always assume adjacency list unless otherwise stated.**

### 11. The Θ(V + E) runtime and how to interpret it

This was the hardest part of the lecture, and Josh said so repeatedly. The analysis has a specific shape worth internalizing.

For the `print` client on an **adjacency list**:
- Runtime to iterate over `v`'s neighbors: the list can hold between 0 and V items, so this is **Ω(1), O(V)**, not Θ of anything.
- Number of vertices considered: **V**.

Naively multiplying gives best case Θ(V) (no edges at all: the outer loop dominates) and worst case Θ(V²) (every vertex adjacent to every other). Since best and worst differ, there is no single Θ in terms of V alone.

The resolution: **state the answer in terms of both V and E.** Pick the cost model = (number of `v += 1` operations) + (number of `println` calls). Then:
- `v += 1` happens exactly **V** times.
- `println` happens exactly **2E** times (each undirected edge is printed once from each endpoint).

So in **all cases** the runtime is **Θ(V + E)**.

How to interpret Θ(V + E): no matter what *family* of increasingly complex graphs you generate, as V and E grow the runtime grows exactly as Θ(V + E). Two sanity checks from the slides:
- **Very sparse family**: every vertex connected to its square (2-4, 3-9, 4-16, 5-25, ...). Then E is Θ(√V), and Θ(V + √V) = **Θ(V)**. The prints are irrelevant next to the outer loop.
- **Very dense family**: every vertex connected to every other. Then E is Θ(V²), and Θ(V + V²) = **Θ(V²)**.

Both specialize correctly, which is exactly why the two-variable form is the right answer.

Also: **Θ(V + E) is equivalent to Θ(max(V, E))**. Both are technically correct; the sum is conventional. And a candid caveat from the slides: "We never formally defined asymptotics on multiple variables, and it turns out to be somewhat poorly defined" (reference: `https://people.cs.ksu.edu/~rhowell/asymptotic.pdf`).

### 12. Why DFS/BFS are O(V + E), and why not O(E)

For `DepthFirstPaths` with an adjacency list, cost model = (number of `dfs` calls) + (number of `marked[w]` checks):
- Each vertex is visited **at most once**: once `marked[v]` is true, `dfs` is never called on it again. So at most V `dfs` calls, O(V).
- Each edge is considered **at most twice**: once from each endpoint. (For reachable edges, exactly twice.) So at most 2E `marked` checks, O(E).

Total: **O(V + E)**.

The "very hard question" from the slides: could we just say **O(E)**? The tempting argument: you can only reach a vertex via an edge, so the number of `dfs` calls is bounded by E, so the V term is redundant.

**No.** The reason is slightly pedantic but real: the constructor has to **create and initialize the all-false `marked` array**, which takes **Θ(V)** time all by itself. A graph with zero edges still costs Θ(V), not constant. The lesson stated on the slide: "our cost model earlier (dfs calls + marked checks) does not provide a tight bound," because it omitted array initialization.

Two further precision points from the summary table:
- We say **O**(V + E), not Θ(V + E), unless the graph is **connected**. Counterexample: a graph where no edges touch the source, so DFS explores nothing and only the Θ(V) initialization happens.
- **Space is Θ(V)**: you need arrays of length V (`marked`, `edgeTo`, and for BFS `distTo`) regardless of how much of the graph you actually explore.

### 13. Representation changes the runtime

With an **adjacency matrix**, the same `print` code is **Θ(V²)**: iterating over one vertex's neighbors is Θ(V) (full row scan), and we do it V times. Note the E term vanishes entirely, the runtime does not care how many edges there are.

Likewise `DepthFirstPaths` becomes **O(V²)**: we create at most V iterators, each taking Θ(V) to run to completion. "Essentially, iterating over the entire adjacency matrix takes O(V²) time."

| Problem | Description | Solution | Adj. list | Adj. matrix |
|---|---|---|---|---|
| s-t paths | Find a path from s to every reachable vertex | `DepthFirstPaths.java` | O(V+E) time, Θ(V) space | O(V²) time, Θ(V) space |
| s-t shortest paths | Find a *shortest* path from s to every reachable vertex | `BreadthFirstPaths.java` | O(V+E) time, Θ(V) space | O(V²) time, Θ(V) space |

For sparse graphs this is a disaster, hence: **always use adjacency list unless otherwise stated.**

---

## Definitions

- **API (Application Programming Interface)**: for our purposes, the set of a class's methods including their signatures and behaviors. It defines how client programmers must think, and constrains what is easy or hard (or impossible) to write as a client.
- **Client / client class**: a class or method that *uses* the Graph API rather than implementing it. `print`, `degree`, `DepthFirstPaths`, and `BreadthFirstPaths` are all clients.
- **Degree of a vertex v**: the number of edges connected to `v`. Not provided by the Princeton API; must be computed by iterating over `adj(v)`.
- **Adjacency matrix**: a V x V 2D array where entry `[v][w]` indicates whether edge v-w exists. Undirected graphs yield a symmetric matrix with each edge stored twice.
- **Edge set**: a single collection of all edges, each edge being a pair of vertices, e.g. `HashSet<Edge>`. (Labeled "Extra" in the slides.)
- **Adjacency list**: an array (or list) of lists indexed by vertex number, where entry `v` holds a list of `v`'s neighbors. The most popular representation in practice, and efficient for sparse graphs.
- **Sparse graph**: a graph with relatively few edges, so that most entries of the adjacency matrix would be zero.
- **Dense graph**: a graph where E grows quickly with V, e.g. every vertex connected to every other, giving E = Θ(V²).
- **`marked[v]`**: boolean array; `marked[v]` is true if and only if `v` is connected to (reachable from) the source `s`. Also serves as the visited-set that prevents revisiting.
- **`edgeTo[v]`**: int array; the previous vertex on the path from `s` to `v`. Collectively a parent-pointer encoding of the traversal tree.
- **`distTo[v]`**: int array used by BFS; the number of edges on the shortest path from `s` to `v`. Maintained as `distTo[n] = distTo[v] + 1`.
- **DFS preorder**: the order in which `dfs` is *called* on vertices. Analogous to tree preorder.
- **DFS postorder**: the order in which `dfs` calls *return* / complete. Analogous to tree postorder.
- **BFS order**: an order that visits vertices in nondecreasing order of distance (number of edges) from the source. Analogous to tree level order.
- **Queue**: a list with two operations, `enqueue` (a.k.a. `addLast`) and `dequeue` (a.k.a. `removeFirst`). FIFO. The opposite of a stack, which has `push` (`addFirst`) and `pop` (`removeFirst`).
- **Fringe**: the queue used in BFS, holding the frontier of discovered-but-not-yet-expanded vertices.
- **Breadth First Search (BFS)**: initialize a fringe (queue) with source `s` and mark it; repeat until the fringe is empty: remove vertex `v` from the front, and for each unmarked neighbor `n` of `v`, mark `n`, set `edgeTo[n] = v` (and optionally `distTo[n] = distTo[v] + 1`), and add `n` to the end of the fringe.
- **Θ(V + E)**: equivalently Θ(max(V, E)); a runtime that grows with whichever of the vertex count or edge count dominates.

---

## Worked Examples

### Example 1: `degree`, a minimal client

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

**What it does.** `G.adj(v)` hands back an `Iterable<Integer>`. The enhanced for loop calls `iterator()` on it, then repeatedly `hasNext()`/`next()`, binding each neighbor to `w`. We ignore `w` entirely and just count iterations.

**Why it is written this way.** The API has no `degree` method, so counting by iteration is the only option. This is the lecture's concrete illustration that "getting the degree of a vertex requires many lines of code with this choice of API."

**Trace on the 4-vertex graph** (edges 0-1, 0-3, 1-2):
```
   0
 3   1
   2
```
`degree(G, 1)`: `adj(1)` yields `0`, then `2`. Loop body runs twice. Returns **2**. Correct, vertex 1 touches edges 1-0 and 1-2.

**Runtime.** Adjacency list: Θ(degree(v)), between Ω(1) and O(V). Adjacency matrix: Θ(V) always, since the iterator must scan the whole row.

### Example 2: `print`, and the subtlety in its runtime

```java
public static void print(Graph G) {
    for (int v = 0; v < G.V(); v += 1) {
        for (int w : G.adj(v)) {
            System.out.println(v + "-" + w);
        }
    }
}
```

Output on the graph above:
```
$ java printDemo
0 - 1
0 - 3
1 - 0
1 - 2
2 - 1
3 - 0
```

**Step by step.** The outer loop exploits the integer-vertex convention: vertices are exactly `0 .. V()-1`, so a plain `int` counter enumerates all of them, no iterator over vertices needed. For each `v`, the inner loop walks `v`'s neighbor list and prints the pair.

**Why each edge appears twice.** The graph is undirected, so edge 0-1 lives in both `adj(0)` and `adj(1)`. Hence six lines of output for three edges. This is not a bug; it falls straight out of the representation, and it is exactly the `2E` in the cost model.

**Runtime, carefully.**
- Cost model: `v += 1` operations plus `println` calls.
- `v += 1`: exactly V times.
- `println`: exactly 2E times.
- Total: **Θ(V + E)** in all cases, with best case Θ(V) and worst case Θ(V²) when re-expressed in V alone.

Swap in an adjacency matrix and the same source code becomes **Θ(V²)**: each `adj(v)` iterator costs Θ(V) to exhaust, times V vertices. Identical client code, different asymptotics, purely from the hidden representation. This is the lecture's whole point about API vs. implementation in one example.

### Example 3: `DepthFirstPaths`, recursive

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

**Structure.** All the work happens in the **constructor**, which calls the private recursive helper. The results land in instance variables, so later queries are cheap. The private `dfs` taking `(G, v)` keeps the recursion clean: no mutable arrays threaded through parameters, because `marked` and `edgeTo` are fields of the enclosing object. This is the pattern the instructor notes say should be "a required pattern," as opposed to putting DFS inside `Graph` itself and passing mutable helper arguments around.

**Order of operations inside `dfs`.** Note that `marked[v] = true` comes **first**, before any recursion. If you marked after the loop, a cycle would send you into infinite recursion. Note also that `edgeTo[w] = v` is set **before** `dfs(G, w)`, at the moment of discovery, which is precisely why `edgeTo[w]` records the *discoverer* of `w`.

**Environment / call-stack reasoning (in words).** Calling `new DepthFirstPaths(G, 0)` creates one object on the heap holding references to a `boolean[]` of length V (all false) and an `int[]` of length V, plus the int `s = 0`. The constructor frame calls `dfs(G, 0)`, pushing a frame whose locals are `G`, `v = 0`, and the hidden iterator over `adj(0)`. That frame sets `marked[0] = true` in the shared heap array, then on finding unmarked neighbor 1 writes `edgeTo[1] = 0` and pushes a *new* `dfs` frame with `v = 1`. The old frame stays alive on the stack with its iterator position preserved, which is exactly what makes DFS resume correctly after returning. At the deepest point of the demo graph the stack holds frames for 0, 1, 2, 5, 4, 3 simultaneously. Every frame points at the same two arrays, so the marks one frame writes are immediately visible to all the others: the arrays are the shared mutable state, the stack is the implicit "what to do next" structure.

**Trace on the demo graph from source 0** (preorder, i.e. the order of `dfs` calls): `0, 1, 2, 5, 4, 3, 6, 7, 8`. The frames return in postorder: `3, 4, 7, 6, 8, 5, 2, 1, 0`.

### Example 4: `pathTo` and `hasPathTo`

```java
public class DepthFirstPaths {
    private boolean[] marked;
    private int[] edgeTo;
    private int s;
    ...
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
}
```

**`hasPathTo`** is the lecture's "see it or you don't" moment (only about a third of the class got it on the poll). The answer is a single array read: **a vertex is reachable from `s` exactly when DFS marked it.** No graph access, no search, Θ(1).

**`pathTo`** walks the parent pointers. Reading the loop: start `x` at the destination `v`; while `x` is not the source, append `x` and hop to its discoverer `edgeTo[x]`. The loop deliberately stops *before* adding `s` (since `edgeTo[s]` is meaningless), so `s` is appended manually afterward. At that point `path` holds the route backwards, destination first, so `Collections.reverse` flips it into source-to-destination order.

**Trace.** Suppose after DFS we have `edgeTo[2] = 4`, `edgeTo[4] = 0`, `s = 0`, and we call `pathTo(2)`:
- `marked[2]` is true, so continue.
- `x = 2`: `2 != 0`, add 2. `path = [2]`. `x = edgeTo[2] = 4`.
- `x = 4`: `4 != 0`, add 4. `path = [2, 4]`. `x = edgeTo[4] = 0`.
- `x = 0`: loop condition `x != s` fails, exit.
- `path.add(0)` gives `[2, 4, 0]`; reverse gives `[0, 4, 2]`.

Returned path: `0 -> 4 -> 2`. Josh's emphasis: "we can't even see the graph when I do this." The two arrays are sufficient.

**Guard clause.** The `if (!hasPathTo(v)) return null;` matters. Without it, an unreachable `v` would have `edgeTo[v]` at its default `0`, and the loop would either terminate with a bogus path or spin forever.

**Runtime.** `hasPathTo` is Θ(1). `pathTo` is Θ(length of path), bounded by O(V).

### Example 5: `BreadthFirstPaths`, hand-traced

Algorithm restated from the slides:
- Initialize the fringe (a queue) with source `s` and mark it.
- Repeat until the fringe is empty:
  - Remove vertex `v` from the front of the fringe.
  - For each unmarked neighbor `n` of `v`: mark `n`, add `n` to the fringe, set `edgeTo[n] = v`, set `distTo[n] = distTo[v] + 1`.

Full trace on the demo graph with `s = 0` (`#`, `marked`, `edgeTo`, `distTo`, and the fringe after each step):

| Step | Action | Fringe after | New assignments |
|---|---|---|---|
| 0 | init | `[0]` | `marked[0]=T`, `distTo[0]=0` |
| 1 | dequeue 0 | `[1]` | `marked[1]=T`, `edgeTo[1]=0`, `distTo[1]=1` |
| 2 | dequeue 1 | `[2, 4]` | `marked[2]=T, edgeTo[2]=1, distTo[2]=2`; `marked[4]=T, edgeTo[4]=1, distTo[4]=2` |
| 3 | dequeue 2 | `[4, 5]` | `marked[5]=T`, `edgeTo[5]=2`, `distTo[5]=3` |
| 4 | dequeue 4 | `[5, 3]` | `marked[3]=T`, `edgeTo[3]=4`, `distTo[3]=3` (edge 4-5 considered, but 5 already marked, so nothing happens) |
| 5 | dequeue 5 | `[3, 6, 8]` | `marked[6]=T, edgeTo[6]=5, distTo[6]=4`; `marked[8]=T, edgeTo[8]=5, distTo[8]=4` |
| 6 | dequeue 3 | `[6, 8]` | nothing to add |
| 7 | dequeue 6 | `[8, 7]` | `marked[7]=T`, `edgeTo[7]=6`, `distTo[7]=5` |
| 8 | dequeue 8 | `[7]` | nothing to add |
| 9 | dequeue 7 | `[]` | nothing to add |

Final state:

| # | marked | edgeTo | distTo |
|---|---|---|---|
| 0 | T | - | 0 |
| 1 | T | 0 | 1 |
| 2 | T | 1 | 2 |
| 3 | T | 4 | 3 |
| 4 | T | 1 | 2 |
| 5 | T | 2 | 3 |
| 6 | T | 5 | 4 |
| 7 | T | 6 | 5 |
| 8 | T | 5 | 4 |

Things to notice in this trace:

- **Step 2 is the DFS/BFS divergence.** DFS at vertex 1 would dive into 2 and never look at edge 1-4 until much later. BFS marks **both** 2 and 4 immediately, in the same iteration. That is why `distTo[4] = 2` here versus a path of length 4 under DFS.
- **Step 4 shows a considered-but-unchanged edge.** From vertex 4 we examine both 3 and 5. Vertex 5 is already marked, so nothing changes, but the edge was still *considered*, and that consideration is a `marked[w]` check that counts toward the E term in the runtime.
- **Step 5 confirms the fringe invariant.** Fringe `[3, 6, 8]` has distances `3, 4, 4`: k and k+1 with k = 3.
- **Marking at enqueue time, not dequeue time.** A vertex is marked the moment it is added. If you marked only on dequeue, a vertex with two discoverers could be enqueued twice and you would lose both the shortest-distance guarantee and the linear runtime.

The slide also shows the final `edgeTo` column rendered as edges (`0-1`, `1-2`, `4-3`, `1-4`, `2-5`, `5-6`, `6-7`, `5-8`), which collectively form the **BFS tree**, the union of shortest paths from `s`.

### Example 6: `BreadthFirstPaths` in Java

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

**Reading it against DFS.** Structurally this is the DFS code with the recursive call replaced by an enqueue, and the implicit call stack replaced by the explicit `fringe`. The `marked`/`edgeTo` bookkeeping is identical; only the *order* in which pending work is retrieved differs (FIFO instead of LIFO). That is the whole difference between shortest paths and arbitrary paths.

(extra context: swapping `Queue` for a stack gives an iterative DFS, though the visit order differs slightly from the recursive version. Not covered in this lecture.)

To also track distances, add an `int[] distTo` field and `distTo[w] = distTo[v] + 1;` inside the `if`, as the demo did. You will write this on Lab 9.

**Runtime.** Same cost model, same answer: **O(V + E)**. The slide phrases it as "O(V) `.next()` calls and O(E) `marked[w]` checks" (each vertex is enqueued and dequeued at most once; each edge is considered at most twice). Space **Θ(V)**.

**Note on `Queue`.** `Queue` here is the Princeton library's concrete class with `enqueue`/`dequeue`/`isEmpty`. (extra context: in the Java standard library `Queue` is an *interface*; you would write `Queue<Integer> fringe = new ArrayDeque<>();` and use `add`/`remove` or `offer`/`poll`. Worth knowing since you cannot import the Princeton library on Project 4.)

---

## Common Pitfalls

1. **Putting DFS inside your `Graph` class.** The instructor notes call this out explicitly as the student approach to avoid. If `dfs` is a method on `Graph`, you end up passing mutable `marked`/`edgeTo` arrays through every recursive call. Use the decoupled pattern: a separate `Paths`-style class whose constructor does the traversal and whose fields hold the results.

2. **Marking too late in DFS.** `marked[v] = true` must be the first line of `dfs`. Marking after the neighbor loop means a cycle sends you into infinite recursion.

3. **Marking on dequeue instead of enqueue in BFS.** If you mark only when a vertex comes off the fringe, the same vertex can be enqueued multiple times by different neighbors. This breaks the O(V + E) bound and can produce wrong `distTo`/`edgeTo` values.

4. **Using a stack (or recursion) and expecting shortest paths.** DFS found vertex 4 at effective distance 4 in the demo; BFS found it at 2. Only the FIFO fringe gives shortest paths in an unweighted graph.

5. **Forgetting that each undirected edge appears twice.** In an adjacency list, edge v-w sits in both `adj(v)` and `adj(w)`. This is why `print` emits 2E lines and why the DFS cost model says "at most 2E `marked` checks." Students who count E instead of 2E usually still get the right Θ, but they misread demo output.

6. **Saying O(E) for DFS/BFS.** The constructor must allocate and zero a length-V `marked` array, which is Θ(V) even on an edgeless graph. The V term cannot be dropped.

7. **Saying Θ(V + E) for DFS/BFS without qualification.** Θ requires the traversal actually to reach things. If the source is isolated, DFS does Θ(V) initialization work and nothing else. Use **O(V + E)** unless you are told the graph is connected.

8. **Assuming `adj` is cheap regardless of representation.** `adj(v)` on an adjacency matrix is Θ(V) even if `v` has no neighbors at all, because the iterator must scan the row. The identical client code is Θ(V + E) on a list and Θ(V²) on a matrix.

9. **Forgetting the `hasPathTo` guard in `pathTo`.** `edgeTo` entries for unreachable vertices are uninitialized (default 0), so the backtracking loop can produce nonsense or never terminate.

10. **Off-by-one in `pathTo`.** The loop condition is `x != s`, so the source is never added inside the loop. You must `path.add(s)` afterward, and then reverse. Skipping either step gives a path missing its start or running backwards.

11. **Confusing DFS preorder with the "active vertex" sequence.** Preorder is the sequence of `dfs` *calls*, each vertex appearing exactly once. Bouncing back up to vertex 4 after finishing 3 does not re-add 4 to the preorder.

12. **Assuming vertices have labels.** The Princeton API numbers them `0 .. V-1`. Labels require a side `Map<Label, Integer>`. Also, V must be fixed at construction, you cannot add vertices later.

13. **Importing the Princeton Graphs library on Project 4.** Explicitly forbidden. Design your own API with only the operations you need.

---

## Likely Exam Points

### 1. Compute DFS preorder, DFS postorder, and BFS order from a drawn graph

Standard warm-up question, usually with a stated tie-breaking rule (e.g. "visit smaller-numbered neighbors first").

**Practice.** On the demo graph with `s = 0` (adjacency: 0:{1}, 1:{0,2,4}, 2:{1,5}, 4:{1,3,5}, 5:{2,4,6,8}, 6:{5,7}), give the DFS preorder, the DFS postorder, and one valid BFS order.

**Answer.** Preorder (order of `dfs` calls): `0 1 2 5 4 3 6 7 8`. Postorder (order of returns): `3 4 7 6 8 5 2 1 0`. BFS order: `0, 1, {2,4}, {5,3}, {6,8}, 7`, e.g. `0 1 2 4 5 3 6 8 7`, where tied vertices may appear in either order.

### 2. Fill in the `marked`, `edgeTo`, `distTo`, and fringe state at a given point in a BFS

Essentially the lecture demo as an exam table. The in-lecture poll was exactly this ("Which vertex? What `edgeTo` and `distTo` values change?").

**Practice.** In the trace above, the fringe is `[4, 5]`. Which vertex is dequeued next, and which table entries change?

**Answer.** Vertex **4** is dequeued (it is at the front, and equivalently it is the nearer of 4 and 5 to the source). Its neighbors 3 and 5 are both considered. Vertex 5 is already marked, so nothing happens there. Vertex 3 changes: `marked[3] = T`, `edgeTo[3] = 4`, `distTo[3] = distTo[4] + 1 = 3`. The fringe becomes `[5, 3]`.

### 3. Recover a path from an `edgeTo` array alone

Tests whether you understand `edgeTo` as parent pointers.

**Practice.** Given `s = 0` and `edgeTo = [-, 0, 4, 4, 0, 2, 5, 6, 5]` (index = vertex), what does `pathTo(2)` return?

**Answer.** Start at 2: `edgeTo[2] = 4`, `edgeTo[4] = 0 = s`. Backwards list `[2, 4, 0]`, reversed: **`0 -> 4 -> 2`**, i.e. `{0, 4, 2}`. No graph needed.

### 4. Runtime of a graph client under a stated representation

The single most heavily tested idea in this lecture, and it comes in both directions (list and matrix).

**Practice.** What is the order of growth of `print` if the graph uses (a) an adjacency list, (b) an adjacency matrix? (Options from lecture: Θ(V), Θ(V+E), Θ(V²), Θ(V·E).)

**Answer.** (a) **Θ(V + E)**: V iterations of the outer loop plus 2E `println` calls. In terms of V alone it ranges from best case Θ(V) to worst case Θ(V²), which is why the two-variable form is the right answer. (b) **Θ(V²)**: exhausting each row iterator costs Θ(V) regardless of degree, times V vertices. The E term disappears entirely.

### 5. Runtime of `DepthFirstPaths` / `BreadthFirstPaths`, with the right bound symbol

**Practice.** Give a tight O bound on the `DepthFirstPaths` constructor with (a) adjacency list and (b) adjacency matrix. Why can we not say O(E) in case (a)? Why not Θ(V + E)?

**Answer.** (a) **O(V + E)**: at most V `dfs` calls (each vertex visited at most once thanks to `marked`) and at most 2E `marked[w]` checks (each edge considered at most twice). (b) **O(V²)**: at most V iterators created, each Θ(V) to exhaust, so we essentially scan the whole matrix. Not O(E), because the constructor must allocate and initialize the all-false length-V `marked` array, which is Θ(V) even when E = 0. Not Θ(V + E) in general, because if the source is isolated, DFS explores nothing and the cost is just Θ(V); you need the graph to be connected for the Θ claim.

### 6. Space complexity

Easy points, frequently paired with the time question.

**Practice.** What is the space complexity of `DepthFirstPaths` and `BreadthFirstPaths` (excluding the graph itself)?

**Answer.** **Θ(V)** for both: arrays of length V (`marked`, `edgeTo`, plus `distTo` for BFS). BFS's fringe also holds at most V vertices, so it does not change the bound. Note Θ, not O: the arrays are allocated in full regardless of how much of the graph is explored.

### 7. Interpreting Θ(V + E) on a family of graphs

**Practice.** A family of graphs connects every vertex to its square (2-4, 3-9, 4-16, ...). What does a Θ(V + E) algorithm simplify to? What about a complete graph?

**Answer.** The square family has E = Θ(√V), so Θ(V + √V) = **Θ(V)**. A complete graph has E = Θ(V²), so Θ(V + V²) = **Θ(V²)**. Also recall Θ(V + E) = Θ(max(V, E)); the sum is just the conventional way to write it.

### 8. DFS vs. BFS: which to use

**Practice.** You want the fewest-turns driving route in an unweighted road graph. DFS or BFS, and why?

**Answer.** **BFS.** It dequeues vertices in nondecreasing order of distance from the source, so the first time a vertex is marked it is reached by a shortest path, and `distTo`/`edgeTo` record that path. DFS gives *some* path, not a shortest one: in the demo it reached vertex 4 via a 4-edge route while BFS found a 2-edge route.

### 9. Why BFS cannot use recursion / what the fringe is

**Practice.** Why does BFS require an explicit data structure while DFS does not, and what invariant does the queue maintain?

**Answer.** DFS's pending work is tracked implicitly by the **call stack** (LIFO), which naturally dives deep. BFS needs FIFO order, so it keeps an explicit **queue** (the fringe). Because vertices are only ever appended at the back, a farther vertex can never jump ahead of a nearer one. The invariant: at all times every vertex on the fringe has distance **k or k+1** for some single k.

### 10. API design consequences

**Practice.** The Princeton API has no `degree(int v)` method. What does a client have to do, and what does this cost under each representation?

**Answer.** Iterate over `G.adj(v)` and count. Adjacency list: Θ(degree(v)), i.e. Ω(1) and O(V). Adjacency matrix: Θ(V) always, because the iterator scans the entire row. This is the lecture's illustration that the choice of API determines "how client programmers must think," and can also affect client runtime and memory.

### 11. Identify or construct a representation

**Practice.** For the undirected path graph 0-1, 1-2, 2-3, write the adjacency matrix and the adjacency lists. Which uses less space, and when does the answer flip?

**Answer.** Matrix (symmetric, each edge twice):
```
    0  1  2  3
0   0  1  0  0
1   1  0  1  0
2   0  1  0  1
3   0  0  1  0
```
Lists: `0:[1]`, `1:[0,2]`, `2:[1,3]`, `3:[2]`. The matrix always uses Θ(V²) space; lists use Θ(V + E). For **sparse** graphs lists win decisively (and make `adj` fast); only for very **dense** graphs (E approaching V²) does the matrix become competitive, and it remains simpler for O(1) edge-existence queries. (extra context: the O(1) `hasEdge` query is the matrix's genuine advantage; the lecture did not discuss edge-existence queries, only neighbor iteration.)

---

## Summary

- **Two orthogonal choices**: the graph **API** (what methods exist) and the **concrete representation** (what data structure is underneath). Both profoundly affect runtime, memory, and how hard algorithms are to write.
- **Princeton API**: `Graph(int V)`, `addEdge(int v, int w)`, `Iterable<Integer> adj(int v)`, `int V()`, `int E()`. Vertices are integers `0 .. V-1`; V is fixed up front; no weights or labels; no `degree` method. Labeled data needs a side `Map<Label, Integer>`.
- **Design pattern**: decouple the graph type from the processing algorithm. Build the graph, pass it (plus a source) to a client class constructor that precomputes everything into fields, then query the client. Do **not** put DFS inside `Graph`.
- **DFS** (`DepthFirstPaths`): recursive; `marked[v] = true` first, then for each unmarked neighbor `w` set `edgeTo[w] = v` and recurse. `hasPathTo(v)` is just `marked[v]`; `pathTo(v)` follows `edgeTo` backwards from `v` to `s` and reverses. The call stack is the implicit fringe.
- **BFS** (`BreadthFirstPaths`): iterative with an explicit **queue** (the fringe). Enqueue and mark `s`; while nonempty, dequeue `v` and for each unmarked neighbor `w`: enqueue, mark, set `edgeTo[w] = v` (and `distTo[w] = distTo[v] + 1`). FIFO order gives **shortest** (fewest-edge) paths; DFS does not.
- **Mark at discovery (enqueue) time**, not at dequeue time.
- **Traversal orders**: DFS preorder = order of calls (`0 1 2 5 4 3 6 7 8` on the demo graph); DFS postorder = order of returns (`3 4 7 6 8 5 2 1 0`); BFS order = by distance from `s` (`0, 1, {2,4}, {5,3}, {6,8}, 7`). Analogous to tree preorder/postorder/level order.
- **Fringe invariant**: every vertex on the BFS queue has distance k or k+1 for a single k.
- **Representations**: adjacency matrix (V x V booleans, each undirected edge stored twice, `adj` costs Θ(V)); edge set (marked Extra); **adjacency list** (array of neighbor lists, most popular, efficient for sparse graphs, `adj` returns the list directly).
- **Runtimes with adjacency list**: `print` is Θ(V + E); DFS and BFS are **O(V + E)** time, **Θ(V)** space. Cost model: V traversal calls plus 2E `marked` checks.
- **Runtimes with adjacency matrix**: `print` is Θ(V²); DFS and BFS are **O(V²)**, terrible for sparse graphs. Default to adjacency list unless told otherwise.
- **Bound precision**: not O(E) (the all-false `marked` array costs Θ(V)); not Θ(V + E) unless the graph is connected (an isolated source means only the Θ(V) initialization runs). Θ(V + E) = Θ(max(V, E)); multi-variable asymptotics were never formally defined in this course.
- **Project 4B/C/D**: you design your own graph API, you may **not** import the Princeton Graphs library, and you may not even need a separate `Graph` class. Expect your first design to be imperfect; that is the intended learning experience. Start 4C early.
