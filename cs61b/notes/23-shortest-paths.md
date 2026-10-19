<!-- Mon, Oct 19, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 23: Shortest Paths

This lecture moves from unweighted path finding (DFS/BFS from Lecture 22) to **shortest paths in edge-weighted graphs**. We start by showing why BFS is the wrong tool for a Google Maps style application: BFS minimizes the *number of edges*, not the *total weight*, so it can return a 330 m route when a 130 m route exists. We then define the goal precisely: the **Shortest Paths Tree (SPT)**, a tree with exactly `V - 1` edges that simultaneously encodes the shortest path from a single source `s` to every reachable vertex, represented in code by the familiar `distTo` and `edgeTo` arrays. The engine that builds it is **relaxation**: start with a wrong SPT and repeatedly ask "is this edge better than what I have?" (`relax(E)`), or relax all of a vertex's outgoing edges at once (`relax(v)`). **Dijkstra's algorithm** is then just BFS with the queue swapped for a priority queue: visit vertices in *closest-first* order, relaxing each vertex exactly once, which runs in `O(E log V)` time and `Θ(V)` space. The lecture closes with **A\*** (explicitly marked out of scope), which adds a heuristic estimate `h(v, goal)` to each priority so that a single-target search can stop early without exploring the whole graph, and notes that a bad heuristic can make A\* return a wrong answer.

---

## Key Concepts

### 1. Recap: DFS vs BFS for path finding

| Problem | Description | Solution | Efficiency (adj. list) |
|---|---|---|---|
| s-t paths | Find a path from `s` to every reachable vertex | `DepthFirstPaths.java` | `O(V+E)` time, `Θ(V)` space |
| s-t shortest paths | Find a *shortest* path (fewest edges) from `s` to every reachable vertex | `BreadthFirstPaths.java` | `O(V+E)` time, `Θ(V)` space |

Which is better? The lecture compares along four axes:

- **Correctness.** Both work on all graphs (both find *some* path to every reachable vertex).
- **Output quality.** BFS is a "2-for-1 deal": you get paths *and* those paths are guaranteed to have the fewest edges. DFS gives no such guarantee.
- **Time efficiency.** Very similar. Both consider every edge twice. Distinguishing them needs experiments or very careful analysis.
- **Space efficiency.** Neither dominates:
  - DFS is worse for **spindly** graphs: the call stack gets very deep, so the computer needs `Θ(V)` memory just to remember the recursive calls (a CS61C topic).
  - BFS is worse for absurdly **bushy** graphs: the queue gets huge. Example: a source connected to 999,999 other vertices means 999,999 vertices enqueued at once, so `Θ(V)` memory.
  - **In practice this comparison barely matters** for our implementations, because we already spend `Θ(V)` memory on the `distTo` and `edgeTo` arrays. (You could shrink this by storing them in a `Map` instead of an array, so only *reached* vertices occupy space.)

### 2. Why BFS does not work for Google Maps

BFS returns the path with the shortest *number of edges*, not the shortest *distance*. The lecture's map example: going from `s` to `t`, BFS finds a 2-hop route of length roughly 330 m, while the correct answer is a longer-in-hops route of roughly 130 m. A long road counts as one edge to BFS, exactly the same as a 10 m road.

The fix is to attach a number to each edge, its **weight** (length in miles, travel time, cost, etc.), and to minimize the **sum of weights** along the path. That changes the problem, and therefore the algorithm.

A small sanity example from the lecture (towns `A` through `G`, weights in miles): the best path from `A` to `F` is `A -> B -> E -> F` with total length 9, even though `A -> C -> F` touches only three towns, because `A -> C -> F` has total length `1 + 15 = 16`. **Fewest edges does not mean shortest.**

### 3. The goal: the Shortest Paths Tree

Two related problems:

- **Single source, single target:** shortest path from `s` to one specific `t`.
- **Single source, multiple targets:** shortest path from `s` to *every* other vertex.

Two structural observations drive everything:

1. **A shortest path never contains a cycle** (assuming non-negative weights). Deleting a cycle can only make the total weight smaller or equal, so there is always a simple shortest path.
2. **The full solution is always a tree.** Think of it as the union of the shortest paths to all vertices. If the shortest path to `G` goes through `E`, then the prefix of that path is itself a shortest path to `E`, so the paths share structure and overlap into a tree rooted at `s`.

**How many edges does the SPT have?** Exactly `V - 1` (given a connected graph where every vertex is reachable). Reason: every vertex has exactly **one** incoming edge in the tree (the last edge on its shortest path), except the source, which has none. The lecture's example has `V = 7` and therefore 6 SPT edges. Note that this is true regardless of how many edges `E` the graph has.

**Representation.** Identical to BFS from last lecture:

- `distTo[v]`: the best known total weight from `s` to `v` (starts at `0` for `s`, `∞` for everything else).
- `edgeTo[v]`: the vertex (or edge) that comes immediately *before* `v` on that best known path. Following `edgeTo` backwards from `v` reconstructs the path in reverse.

For the lecture's 7-vertex graph with `s = A`, the final answer is:

| v | `distTo[]` | `edgeTo[]` |
|---|---|---|
| A | 0.0 | - |
| B | 2.0 | A→B |
| C | 1.0 | A→C |
| D | 11.0 | G→D |
| E | 5.0 | B→E |
| F | 9.0 | E→F |
| G | 10.0 | E→G |

(The slide prints `A→B` in the `C` row; that is a typo for `A→C`, since `distTo[C] = 1.0` comes from the weight-1 edge `A→C`.)

Notice `distTo[D] = 11` via the long way around `A -> B -> E -> G -> D` (`2 + 3 + 5 + 1`), even though the direct edge `B→D` has weight 11 and `A -> B -> D` costs `2 + 11 = 13`. The SPT is not "mostly direct edges."

### 4. Relaxation: the universal improvement operation

The strategy shared by essentially all shortest-path algorithms:

> Start from an **incorrect** SPT, then make improvements until it is correct.

The improvement operation is **relaxation**.

**Edge relaxation.** `relax(E)`: *if edge `E` is better, use it instead.* Concretely, for edge `v → w` with weight `weight(v, w)`:

```
if (distTo[v] + weight(v, w) < distTo[w]) {
    distTo[w] = distTo[v] + weight(v, w);
    edgeTo[w] = v;
}
```

Read it as: "my current best route to `w` costs `distTo[w]`. Could I do better by going to `v` first and then taking this one edge?" If yes, overwrite both arrays. If no, do nothing. Relaxation is cheap, local (it looks at one edge and two array slots), and never makes `distTo` worse.

**Node relaxation.** `relax(v)`: relax *all* of `v`'s outgoing edges. This is the per-vertex unit of work in Dijkstra's.

### 5. SPT algorithms (all built from relaxation)

The lecture names three, and covers only one:

- **Bellman-Ford (1955):** relax all edges, in any order, and repeat `V` times.
- **Dijkstra's Algorithm (1956):** relax all nodes exactly once, in **closest-first** order.
- **Duan-Mao et al. (2025):** relax edges in batches.

### 6. Dijkstra's Algorithm: closest-first order

We have seen Depth First order (stack) and Breadth First order (queue). Dijkstra's introduces a third traversal order: **Closest First**, meaning in increasing order of `distTo`. The vague question in lecture, and its answer:

> To get Breadth First order we used a **Queue** as our fringe. To get Closest First order, what should the fringe be? A **priority queue**.

Pseudocode, as given:

```
add all vertices to PQ with infinite priority
// (except the source, whose distTo and priority are 0)
while (!pq.isEmpty()) {
    v = pq.removeMin();
    relax(v);            // i.e. relax all edges from v
}
```

Where relaxing an edge that improves `distTo[w]` must also **`changePriority(w, distTo[w])`** so the PQ keeps its closest-first ordering.

**The key invariant and why Dijkstra's is correct.** In Dijkstra's, relaxing an edge that points to an **already-visited** (already removed from the PQ) vertex **never works**. The lecture's argument by example: we are relaxing `E`, and the edge `E→C` points at `C`, which we already visited. But we visited `C` before `E`, which (because we remove in closest-first order) means `distTo[C] ≤ distTo[E]`. Since weights are non-negative, `distTo[E] + weight(E, C) ≥ distTo[E] ≥ distTo[C]`, so the edge cannot be an improvement. Concretely in the demo: `5 + 1 > 1`, so `C` is unchanged.

Said generally: **once a vertex is removed from the PQ, its `distTo` is final.** A vertex that is "white" (visited, no longer on the fringe) can never be improved again, assuming non-negative weights. That is exactly why the algorithm can get away with relaxing each node only once, and why it yields the correct SPT.

### 7. Runtime analysis

Assuming a binary-heap-based priority queue:

| Operation | # Operations | Cost per operation | Total cost |
|---|---|---|---|
| PQ `add` | `V` | `O(log V)` | `O(V log V)` |
| PQ `removeSmallest` | `V` | `O(log V)` | `O(V log V)` |
| PQ `changePriority` | `E` | `O(log V)` | `O(E log V)` |

Each vertex is added once and removed once. Each *edge* is examined once (when its source vertex is relaxed), and each examination triggers at most one `changePriority`.

Total: `O(V log V + V log V + E log V)`. For a connected graph `E ≥ V - 1`, so this simplifies to the headline result:

**Dijkstra's runtime: `O(E log V)`. Space: `Θ(V)`.**

### 8. A\* (marked "Extra, not in scope" on the slides)

**The problem with Dijkstra's for navigation.** Dijkstra's is single-source *multiple*-target. If you only want Denver to NYC, Dijkstra's will explore nearly everything within two thousand miles of Denver (including Utah and Montana) before it reaches NYC. Correct, but wasteful. We want it to head *east* first.

**The A\* idea.** Visit vertices in order of `d(source, v) + h(v, goal)`, where `h(v, goal)` is an **estimate** of the remaining distance from `v` to the goal. Compare to Dijkstra's, which orders by `d(source, v)` alone. In words: expand `v` if we already know the fastest way to reach `v` **and** we suspect `v` is on a fast route to the goal.

- `h` is an arbitrary **heuristic** ("using experience to learn and improve"). It does not have to be perfect.
- `h(v, goal)` **does not change** as the algorithm runs:
  ```java
  /** h(v, goal) DOES NOT CHANGE as algorithm runs. */
  public method h(v, goal) {
      return computeLineDistance(v.latLong, goal.latLong);
  }
  ```
  For maps, as-the-crow-flies (straight line) distance to the goal is the classic choice.
- `distTo` and `edgeTo` are still updated with **true** distances; only the *priorities* include `h`.
- We stop as soon as the goal is the next vertex to be dequeued.

**Consequences.** Not every vertex gets visited (that is the point), and the resulting tree is **not** a valid SPT for all vertices: in the lecture's A\* demo the path to `D` comes out as 13 instead of the true 11. That is fine, because we only asked about the path to `G`.

**Heuristic quality.**

- `h(v, goal) = 0` for all `v`: A\* degenerates into **Dijkstra's**.
- `h(v, goal) = 10000` (a constant) for all `v`: also just **Dijkstra's**, since adding the same constant to every priority does not change the ordering.
- `h` that *overestimates* for some vertices (for example, declaring Illinois and Indiana to be in the middle of nowhere with `h = 100000`): A\* becomes **incorrect**. It will dodge Illinois and fail to find the shortest path. It does not merely run slower.

**Admissibility and consistency (EXTRA: beyond course scope, per the slides).** For this version of A\* to be correct the heuristic should be:

- **Admissible:** `h(v, goal) ≤` the true distance from `v` to the goal (never overestimates).
- **Consistent:** for each neighbor `w` of `v`, `h(v, goal) ≤ dist(v, w) + h(w, goal)`, where `dist(v, w)` is the edge weight.

All consistent heuristics are admissible. These are *sufficient but not necessary* conditions: admissibility suffices for "A\* tree search," consistency suffices for "A\* graph search" (which is the version we saw). The slides state explicitly: you will not be expected to judge whether a heuristic is admissible or consistent unless the exam defines the terms. The takeaway you *are* responsible for: **the choice of heuristic matters, and a bad choice can make A\* give the wrong answer.** CS188 covers this properly.

**A\* vs Dijkstra's visualization.** `http://qiao.github.io/PathFinding.js/visual/`. Note that when all edge weights are equal, **Dijkstra's algorithm is just breadth first search**. The tool is also good for seeing the difference between the order in which nodes are *visited* and the nodes that end up *on* the shortest path: unless you are very lucky, vastly more nodes are visited than lie on the final path.

### 9. The full graph problems table

| Problem | Description | Solution | Efficiency |
|---|---|---|---|
| paths | Find a path from `s` to every reachable vertex | `DepthFirstPaths.java` | `O(V+E)` time, `Θ(V)` space |
| shortest paths | Fewest-edge path from `s` to every reachable vertex | `BreadthFirstPaths.java` | `O(V+E)` time, `Θ(V)` space |
| shortest weighted paths | Shortest path *considering weights* from `s` to every reachable vertex | `DijkstrasSP.java` | `O(E log V)` time, `Θ(V)` space |
| shortest weighted path (out of scope) | Shortest weighted path from `s` to one target | A\*: same as Dijkstra's, but with `h(v, goal)` added to each vertex's priority | Time depends on heuristic, `Θ(V)` space |

---

## Definitions

- **Edge weight:** a number attached to an edge representing its cost (distance, time, price). An **edge-weighted graph** is a graph whose edges carry weights.
- **Path weight / path length:** the *sum* of the weights of the edges along a path. In a weighted setting, "length" means this sum, not the number of edges.
- **Shortest path from `s` to `t`:** a path from `s` to `t` of minimum total weight. With non-negative weights it can always be taken to contain no cycles.
- **Shortest Paths Tree (SPT) of `G` rooted at `s`:** the tree formed by the union of shortest paths from `s` to every reachable vertex. For a connected graph with all vertices reachable, it has exactly `V - 1` edges, because every vertex except `s` has exactly one incoming edge.
- **`distTo[v]`:** the best known total weight of a path from the source to `v`. Initialized to `0` for the source and `∞` for all other vertices. At the end of Dijkstra's, it equals the true shortest-path weight.
- **`edgeTo[v]`:** the predecessor of `v` on the best known path from the source to `v`, i.e. the last edge of that path. Initialized to "none."
- **Edge relaxation, `relax(E)`:** "if edge `E` is better, use it instead." For `E = v → w`: if `distTo[v] + weight(v, w) < distTo[w]`, set `distTo[w] = distTo[v] + weight(v, w)` and `edgeTo[w] = v`.
- **Node relaxation, `relax(v)`:** relax all of `v`'s outgoing edges.
- **Fringe:** the data structure holding vertices that have been discovered but not yet processed. DFS: stack. BFS: queue. Dijkstra's/A\*: priority queue.
- **Closest First order:** visiting vertices in increasing order of their (final) `distTo` value from the source. The traversal order Dijkstra's produces.
- **Dijkstra's Algorithm (1956):** relax all nodes exactly once, in closest-first order, using a priority queue as the fringe, keyed on `distTo`.
- **Bellman-Ford (1955):** relax all edges in any order, repeated `V` times (named only; not covered).
- **Heuristic `h(v, goal)`:** a fixed, cheap-to-compute estimate of the distance from `v` to the goal. Does not change during the run.
- **A\* (graph search):** Dijkstra's with priorities `distTo[v] + h(v, goal)` instead of `distTo[v]`, terminating when the goal is dequeued. (Out of scope.)
- **Admissible heuristic (extra):** `h(v, goal)` never exceeds the true distance from `v` to the goal.
- **Consistent heuristic (extra):** for every edge `v → w`, `h(v, goal) ≤ weight(v, w) + h(w, goal)`. Consistency implies admissibility.

---

## Worked Examples

> **A note on reconstruction.** The slide diagrams were lost in the PDF extraction, so the graphs below were reconstructed from the surviving edge labels plus the numeric traces printed on the slides (`distTo` tables, "Vertex C unchanged since 5+1 > 1", the fringe contents, and the A\* `h` column). Every reconstructed edge is consistent with all of those printed numbers, including the lecture's claim that its heuristic is "the min weight edge out of each vertex." The graph is **directed**.

### Example 0: The lecture's demo graph, in Java

Vertices `A..G` map to `0..6` (the A\* slide says "s = 0, goal = 6", i.e. `A` to `G`).

```java
import java.util.ArrayList;
import java.util.List;

/** One directed, weighted edge. Immutable, so it is safe to share
 *  the same DirectedEdge object between the adjacency list and edgeTo. */
public class DirectedEdge {
    private final int from;
    private final int to;
    private final double weight;

    public DirectedEdge(int from, int to, double weight) {
        this.from = from;
        this.to = to;
        this.weight = weight;
    }

    public int from() { return from; }
    public int to() { return to; }
    public double weight() { return weight; }
}

/** Adjacency-list representation: one bucket of outgoing edges per vertex. */
public class EdgeWeightedDigraph {
    private final List<List<DirectedEdge>> adj;

    public EdgeWeightedDigraph(int V) {
        adj = new ArrayList<>(V);
        for (int v = 0; v < V; v += 1) {
            adj.add(new ArrayList<>());
        }
    }

    public int V() { return adj.size(); }

    public void addEdge(int v, int w, double weight) {
        adj.get(v).add(new DirectedEdge(v, w, weight));
    }

    /** Only the OUTGOING edges of v. This is what relax(v) iterates over. */
    public Iterable<DirectedEdge> adj(int v) { return adj.get(v); }
}
```

Building the lecture's graph:

```java
// A=0, B=1, C=2, D=3, E=4, F=5, G=6
EdgeWeightedDigraph G = new EdgeWeightedDigraph(7);
G.addEdge(0, 1, 2);    // A -> B, 2
G.addEdge(0, 2, 1);    // A -> C, 1
G.addEdge(1, 2, 5);    // B -> C, 5
G.addEdge(1, 3, 11);   // B -> D, 11
G.addEdge(1, 4, 3);    // B -> E, 3
G.addEdge(2, 5, 15);   // C -> F, 15
G.addEdge(3, 4, 2);    // D -> E, 2
G.addEdge(4, 2, 1);    // E -> C, 1
G.addEdge(4, 5, 4);    // E -> F, 4
G.addEdge(4, 6, 5);    // E -> G, 5
G.addEdge(6, 3, 1);    // G -> D, 1
G.addEdge(6, 5, 1);    // G -> F, 1
```

Adjacency list view (`V = 7`, `E = 12`):

```
A: B(2), C(1)
B: C(5), D(11), E(3)
C: F(15)
D: E(2)
E: C(1), F(4), G(5)
F: (no outgoing edges)
G: D(1), F(1)
```

**Box-and-pointer / memory picture in words.** `distTo` and `edgeTo` are two parallel arrays of length `V`, indexed by vertex number. `distTo` holds `double`s (primitives, stored directly in the array's boxes). `edgeTo` holds references to `DirectedEdge` objects (or just `int` predecessor indices), so each box either points at an edge object out on the heap or holds `null`/`-1`. The SPT is therefore not a separate tree object with child pointers at all: it is encoded entirely as `V` parent pointers sitting in one array. To walk from `v` back to `s` you repeatedly read `edgeTo[v]`, follow it to its `from()` vertex, and use that as the next index. Because every vertex has at most one `edgeTo` entry, this chain cannot branch, which is a second way to see that the SPT has at most `V - 1` edges.

### Example 1: Edge relaxation (the lecture's magenta-edge example)

Graph: `A→B` weight 1, `B→C` weight 5, `C→D` weight 3, `A→D` weight 100. We start from an **incorrect** SPT:

| v | `distTo[]` | `edgeTo[]` |
|---|---|---|
| A | 0.0 | - |
| B | 1.0 | A→B |
| C | 6.0 | B→C |
| D | 100.0 | A→D |

Now relax the edge `C→D`:

```java
private void relaxEdge(DirectedEdge e) {
    int v = e.from();          // C
    int w = e.to();            // D
    if (distTo[v] + e.weight() < distTo[w]) {   // 6.0 + 3.0 = 9.0 < 100.0  --> true
        distTo[w] = distTo[v] + e.weight();     // distTo[D] = 9.0
        edgeTo[w] = v;                          // edgeTo[D] = C
    }
}
```

**Step by step.** `distTo[C]` is 6, the edge weight is 3, so routing through `C` costs 9. That beats the current 100, so both arrays are overwritten: `distTo[D] = 9.0`, `edgeTo[D] = C→D`. The `A→D` edge of weight 100 is now simply unused (it is still in the graph, it is just not in the tree).

| v | `distTo[]` | `edgeTo[]` |
|---|---|---|
| A | 0.0 | - |
| B | 1.0 | A→B |
| C | 6.0 | B→C |
| D | **9.0** | **C→D** |

This shortest paths tree is now correct. Note what relaxation did *not* do: it did not look at the whole graph, recompute anything, or touch any vertex other than `D`. One edge, one comparison, two writes.

### Example 2: Node relaxation (the lecture's relax(E) exercise)

The lecture's second example starts from this incorrect SPT (source `A`) and asks what happens when we relax node **E**, which has `distTo[E] = 16.0`:

| v | `distTo[]` before | `edgeTo[]` before |
|---|---|---|
| A | 0.0 | - |
| B | 8.0 | A→B |
| C | 13.0 | B→C |
| D | 40.0 | B→D |
| E | 16.0 | C→E |
| F | 3.0 | A→F |
| G | ∞ | - |

The outgoing edges of `E` are `E→D` with weight 2 and `E→G` with weight 2. `relax(E)` relaxes each in turn:

```java
private void relax(int v) {
    for (DirectedEdge e : G.adj(v)) {
        relaxEdge(e);
    }
}
```

- `E→D`: `distTo[E] + 2 = 16 + 2 = 18 < 40`, so **improve**. `distTo[D] = 18.0`, `edgeTo[D] = E→D`. (We just discovered that the 32-weight shortcut `B→D` was a bad idea.)
- `E→G`: `distTo[E] + 2 = 18 < ∞`, so **improve**. `distTo[G] = 18.0`, `edgeTo[G] = E→G`. This is the first time `G` becomes reachable at all.

| v | `distTo[]` after | `edgeTo[]` after |
|---|---|---|
| A | 0.0 | - |
| B | 8.0 | A→B |
| C | 13.0 | B→C |
| D | **18.0** | **E→D** |
| E | 16.0 | C→E |
| F | 3.0 | A→F |
| G | **18.0** | **E→G** |

Two things worth internalizing: relaxing one node can improve several vertices at once, and an improvement can both *lower a finite value* (`D`: 40 to 18) and *discover a new vertex* (`G`: ∞ to 18).

### Example 3: `DijkstrasSP.java`

Here is the full algorithm. The PQ interface is the CS61B `ExtrinsicMinPQ` style (`add(item, priority)`, `removeSmallest()`, `changePriority(item, priority)`, `contains(item)`), which is what makes the slides' operation counts (`V` adds, `V` removes, `E` changePriorities) literal. *(extra context: the slides give only the four-line pseudocode and the name `DijkstrasSP.java`; the Java below is a faithful expansion of that pseudocode, not code shown in lecture.)*

```java
import java.util.ArrayDeque;
import java.util.Arrays;
import java.util.Deque;

public class DijkstrasSP {
    private final EdgeWeightedDigraph G;
    private final double[] distTo;
    private final DirectedEdge[] edgeTo;
    private final ExtrinsicMinPQ<Integer> fringe;

    public DijkstrasSP(EdgeWeightedDigraph G, int s) {
        this.G = G;
        distTo = new double[G.V()];
        edgeTo = new DirectedEdge[G.V()];
        Arrays.fill(distTo, Double.POSITIVE_INFINITY);
        distTo[s] = 0.0;

        // "add all vertices to PQ with infinite priority"
        // (the source's priority is 0, since distTo[s] == 0)
        fringe = new ArrayHeapMinPQ<>();
        for (int v = 0; v < G.V(); v += 1) {
            fringe.add(v, distTo[v]);
        }

        // "while (!pq.isEmpty()) { v = pq.removeMin(); relax(v); }"
        while (fringe.size() > 0) {
            int v = fringe.removeSmallest();
            relax(v);
        }
    }

    /** Node relaxation: relax every edge pointing OUT of v. */
    private void relax(int v) {
        for (DirectedEdge e : G.adj(v)) {
            int w = e.to();
            if (distTo[v] + e.weight() < distTo[w]) {
                distTo[w] = distTo[v] + e.weight();
                edgeTo[w] = e;
                // Keep the fringe ordered by the NEW distance.
                // w may already be visited (gone from the PQ), in which case
                // this branch is unreachable with non-negative weights.
                if (fringe.contains(w)) {
                    fringe.changePriority(w, distTo[w]);
                }
            }
        }
    }

    public double distTo(int v) {
        return distTo[v];
    }

    public boolean hasPathTo(int v) {
        return distTo[v] < Double.POSITIVE_INFINITY;
    }

    /** Follow edgeTo backwards, then reverse. */
    public Iterable<DirectedEdge> pathTo(int v) {
        if (!hasPathTo(v)) {
            return null;
        }
        Deque<DirectedEdge> path = new ArrayDeque<>();
        for (DirectedEdge e = edgeTo[v]; e != null; e = edgeTo[e.from()]) {
            path.push(e);   // push onto the front, so the path comes out s-first
        }
        return path;
    }
}
```

**What each piece is doing and why.**

- `Arrays.fill(distTo, POSITIVE_INFINITY)` plus `distTo[s] = 0.0` sets up the *initial incorrect SPT*: "I can reach the source for free and I believe nothing else is reachable." Every later step only improves it.
- Adding *all* vertices up front is why we need `changePriority`: a vertex sitting in the heap at priority ∞ must be re-keyed when we find a real route to it. This is also the direct source of the `Θ(V)` space bound.
- `removeSmallest()` is the only line that enforces **closest-first** order, and the correctness invariant depends on it: the vertex with the smallest tentative distance in the fringe already has its final `distTo`, because any other route to it would have to pass through some other fringe vertex with a larger or equal distance and then add a non-negative weight.
- `relax(v)` iterates only over **outgoing** edges. On an undirected graph you would iterate over all incident edges instead.
- `pathTo` is the pointer-chasing walk described above: `edgeTo[v]` gives the last edge, `edgeTo[e.from()]` gives the one before it, and so on until a `null` marks the source. Pushing onto a `Deque` front reverses the sequence for free.

### Example 4: Tracing Dijkstra's on the demo graph (`s = A`)

Start: all vertices in the fringe, `A` at 0 and everything else at ∞.

```
Fringe: [(A: 0), (B: ∞), (C: ∞), (D: ∞), (E: ∞), (F: ∞), (G: ∞)]
```

**Step 1, remove `A` (0).** Relax `A→B (2)` and `A→C (1)`.
`distTo[B] = 2, edgeTo[B] = A→B`; `distTo[C] = 1, edgeTo[C] = A→C`.
`Fringe: [(C: 1), (B: 2), (D: ∞), (E: ∞), (F: ∞), (G: ∞)]`

**Step 2, remove `C` (1).** `C`'s only outgoing edge is `C→F (15)`.
`distTo[F] = 16, edgeTo[F] = C→F`.
`Fringe: [(B: 2), (F: 16), (D: ∞), (E: ∞), (G: ∞)]`

**Step 3, remove `B` (2).** Relax `B→C (5)`, `B→D (11)`, `B→E (3)`.
- `C`: `2 + 5 = 7`, **not** `< 1`. Unchanged. (The slide calls this out: "Vertex C unchanged since 2+5 > 1.")
- `D`: `2 + 11 = 13 < ∞`. `distTo[D] = 13, edgeTo[D] = B→D`.
- `E`: `2 + 3 = 5 < ∞`. `distTo[E] = 5, edgeTo[E] = B→E`.

`Fringe: [(E: 5), (D: 13), (F: 16), (G: ∞)]`

**Step 4, remove `E` (5).** Relax `E→C (1)`, `E→F (4)`, `E→G (5)`.
- `C`: `5 + 1 = 6`, not `< 1`. Unchanged. This is the **invariant in action**: `C` was already visited, so no edge into it can ever help.
- `F`: `5 + 4 = 9 < 16`. `distTo[F] = 9, edgeTo[F] = E→F`. (A real improvement over the `C→F` route.)
- `G`: `5 + 5 = 10 < ∞`. `distTo[G] = 10, edgeTo[G] = E→G`.

`Fringe: [(F: 9), (G: 10), (D: 13)]`

**Step 5, remove `F` (9).** `F` has no outgoing edges. Nothing happens.
`Fringe: [(G: 10), (D: 13)]`

**Step 6, remove `G` (10).** Relax `G→D (1)` and `G→F (1)`.
- `D`: `10 + 1 = 11 < 13`. `distTo[D] = 11, edgeTo[D] = G→D`. The 11-weight direct edge `B→D` loses to a four-edge detour.
- `F`: `10 + 1 = 11`, not `< 9`. Unchanged (and `F` is already visited anyway).

`Fringe: [(D: 11)]`

**Step 7, remove `D` (11).** Relax `D→E (2)`: `11 + 2 = 13`, not `< 5`. Unchanged ("Vertex E unchanged since 11 + 2 > 5").

`Fringe: []`. Done. Final tables match the SPT table in the Key Concepts section.

**Removal order: `A, C, B, E, F, G, D`,** with distances `0, 1, 2, 5, 9, 10, 11`, monotonically increasing. That increasing sequence is the definition of closest-first order and a good self-check when you trace by hand. Observe also that `D` is removed *last* despite having a direct edge from `B`, and that the removal order is nothing like BFS order.

### Example 5: A\* on the same graph (out of scope, for contrast)

`s = A (0)`, `goal = G (6)`. The lecture's (arbitrary) heuristic is "the min weight edge out of each vertex":

| v | A | B | C | D | E | F | G |
|---|---|---|---|---|---|---|---|
| `h(v, goal)` | 1 | 3 | 15 | 2 | 1 | ∞ | 0 |

The only code change is in the priority:

```java
// Dijkstra's:  fringe.changePriority(w, distTo[w]);
// A*:          fringe.changePriority(w, distTo[w] + h(w, goal));
//
// distTo and edgeTo still store TRUE distances. Only the ordering changes.
while (fringe.size() > 0) {
    int v = fringe.removeSmallest();
    if (v == goal) {            // early exit: single target only
        break;
    }
    relax(v);                   // relax uses distTo[w] + h(w, goal) as priority
}
```

Trace:

1. Remove `A` (priority `0 + 1 = 1`). `distTo[B] = 2` with priority `2 + 3 = 5`; `distTo[C] = 1` with priority `1 + 15 = 16`.
   `Fringe: [(B: 5), (C: 16), (D: ∞), (E: ∞), (F: ∞), (G: ∞)]`
   Note that `C` is nearer than `B` in true distance, but A\* deprioritizes it: `C`'s only way onward is a weight-15 edge, so it looks like a dead end.
2. Remove `B` (5). `C`: `7 > 1`, no change. `distTo[D] = 13`, priority `13 + 2 = 15`. `distTo[E] = 5`, priority `5 + 1 = 6`.
   `Fringe: [(E: 6), (D: 15), (C: 16), (F: ∞), (G: ∞)]`
3. Remove `E` (6). `C`: `6 > 1`, no change. `distTo[F] = 9`, priority `9 + ∞ = ∞`. `distTo[G] = 10`, priority `10 + 0 = 10`.
   `Fringe: [(G: 10), (D: 15), (C: 16), (F: ∞)]`
4. The next vertex to be dequeued is `G`, our target, so **we are done.** Path: `A → B → E → G`, length 10.

**Observations from the slides:** not every vertex got visited (`C`, `D`, `F` were never dequeued), and the result is **not** a correct SPT for `A` (the recorded path to `D` costs 13, but the true shortest is 11). Both are acceptable, because A\* only promises the path to the single goal.

---

## Common Pitfalls

1. **Using BFS when edges have weights.** BFS minimizes hop count. The lecture's whole motivating example is a 2-hop 330 m route beating out a many-hop 130 m route. Conversely, remember the special case: if all weights are *equal*, Dijkstra's **is** BFS.
2. **Confusing "fewest edges" with "shortest."** `A -> C -> F` has 2 edges and length 16; `A -> B -> E -> F` has 3 edges and length 9. The SPT is a tree of minimum *weight* paths, and there is no bound on how many edges a shortest path uses beyond `V - 1`.
3. **Forgetting `distTo[s] = 0`.** If every entry including the source starts at ∞, every relaxation computes `∞ + weight` and nothing ever improves.
4. **Forgetting `changePriority` after a successful relaxation.** The `distTo` arrays will be updated but the heap will hand you vertices in the wrong order, breaking closest-first and therefore correctness.
5. **Relaxing a vertex more than once.** Dijkstra's relaxes each node **exactly once**. If you re-add visited vertices to the fringe you are doing extra work for nothing, since (with non-negative weights) a visited vertex can never be improved.
6. **Expecting the direct edge to win.** In the demo, `B→D` has weight 11 and still loses to `A -> B -> E -> G -> D` at total 11... specifically, the direct route `A -> B -> D` costs 13 and loses. Do not assume a vertex's SPT parent is whatever neighbor discovered it first.
7. **Thinking the SPT is unique.** With tied path weights there can be several valid SPTs. What *is* fixed is the number of edges (`V - 1`) and each `distTo` value.
8. **Thinking the SPT edge count depends on `E`.** It is always `V - 1` (all vertices reachable), whether the graph is sparse or complete.
9. **Assuming non-negative weights without saying so.** Both "shortest paths contain no cycles" and "visited vertices never improve" are stated in lecture *with the non-negative-weights caveat*. *(extra context: with a negative edge the invariant genuinely fails. Consider `A→B` weight 1, `A→C` weight 2, `C→B` weight -2. Dijkstra's dequeues `B` at 1 and finalizes it, missing the true distance of 0. That is what Bellman-Ford is for.)*
10. **Mis-stating the runtime.** It is `O(E log V)`, not `O(V log V)` or `O(E log E)`. The `E log V` term comes from `changePriority`, not from the removals.
11. **A\*: putting the heuristic into `distTo`.** `h` goes into the *priority* only. `distTo` and `edgeTo` must keep true distances, or the reported path lengths will be garbage.
12. **A\*: recomputing or updating `h` during the run.** The slide is emphatic: `h(v, goal)` does not change as the algorithm runs.
13. **A\*: assuming a bad heuristic only costs speed.** An overestimating heuristic can make A\* return a genuinely wrong path (the Illinois example). A zero or constant heuristic is the harmless case: it just reduces to Dijkstra's.
14. **A\*: treating its output as an SPT.** It is not. It is a single path, and other vertices' recorded distances may be wrong.

---

## Likely Exam Points

### 1. SPT edge count

**Q.** If `G` is a connected edge-weighted graph with `V` vertices and `E` edges, and every vertex is reachable from `s`, how many edges are in the Shortest Paths Tree of `G`? Does the answer depend on `E`?

**A.** Exactly `V - 1`, independent of `E`. Each vertex has exactly one incoming SPT edge (the last edge of its shortest path) except the source, which has none. For the lecture's graph with `V = 7`, that is 6 edges.

### 2. Order of removal from the priority queue

**Q.** Run Dijkstra's from `A` on the lecture's graph (`A: B(2), C(1); B: C(5), D(11), E(3); C: F(15); D: E(2); E: C(1), F(4), G(5); G: D(1), F(1)`). List the vertices in the order they are removed from the fringe, with the `distTo` value at removal.

**A.** `A(0), C(1), B(2), E(5), F(9), G(10), D(11)`. The values must be non-decreasing: that is the closest-first guarantee. (Note this differs from BFS order from `A`, which would be `A, B, C, D, E, ...`.)

### 3. Filling in `distTo` / `edgeTo` after a relaxation

**Q.** Given `distTo = {A:0, B:8, C:13, D:40, E:16, F:3, G:∞}` and `edgeTo = {B: A→B, C: B→C, D: B→D, E: C→E, F: A→F}`, with `E`'s outgoing edges being `E→D (2)` and `E→G (2)`, show both arrays after `relax(E)`.

**A.** `E→D`: `16 + 2 = 18 < 40`, so `distTo[D] = 18`, `edgeTo[D] = E→D`. `E→G`: `16 + 2 = 18 < ∞`, so `distTo[G] = 18`, `edgeTo[G] = E→G`. All other entries are unchanged. (`B→D` is now out of the tree.)

### 4. Runtime derivation

**Q.** State Dijkstra's runtime with a binary heap PQ and justify each term.

**A.** `V` `add` operations, `V` `removeSmallest` operations, and at most `E` `changePriority` operations, each `O(log V)`, giving `O(V log V + V log V + E log V)`. For a connected graph `E ≥ V - 1`, so this is `O(E log V)`. Space is `Θ(V)` for `distTo`, `edgeTo`, and the PQ. The `E` bound on `changePriority` holds because each edge is examined exactly once, when its source vertex is relaxed.

### 5. Can a finalized distance change?

**Q.** True or false: in Dijkstra's on a graph with non-negative weights, `distTo[v]` can change after `v` has been removed from the priority queue. Justify.

**A.** False. If `v` was removed before `w`, then `distTo[v] ≤ distTo[w]`, so any edge `w → v` offers `distTo[w] + weight(w, v) ≥ distTo[w] ≥ distTo[v]`, which is never a strict improvement. This is the lecture's "relaxing an arrow pointing to an already-visited node NEVER works" invariant, and it is why Dijkstra's is correct. (It is exactly this step that requires non-negative weights.)

### 6. BFS as a special case

**Q.** Under what condition is Dijkstra's algorithm equivalent to breadth first search?

**A.** When all edge weights are equal (in particular, all 1). Then "closest first" and "fewest edges first" are the same ordering, and the priority queue behaves like a FIFO queue.

### 7. Structure of a shortest path

**Q.** True or false, with non-negative weights: a shortest path may contain a cycle.

**A.** False (or more precisely, there is always a shortest path without one). Removing a cycle from a path leaves a valid path whose weight is no larger, so a simple path always achieves the minimum. This is why the solution to the single-source problem is a *tree*.

### 8. Relaxation semantics

**Q.** Define `relax(E)` and `relax(v)` in one line each.

**A.** `relax(E)`: if edge `E = v→w` is better, use it instead, i.e. if `distTo[v] + weight(v,w) < distTo[w]`, update `distTo[w]` and `edgeTo[w]`. `relax(v)`: relax all of `v`'s outgoing edges.

### 9. Fringe data structures

**Q.** DFS, BFS, and Dijkstra's are the same traversal skeleton with different fringes. Name the fringe for each and the resulting visit order.

**A.** DFS: stack (implicit, via recursion), depth first order. BFS: queue, breadth first order. Dijkstra's: priority queue keyed on `distTo`, closest first order.

### 10. A\* heuristic quality (flagged out of scope, but the slides define the terms, and they say you may be asked if the exam defines them)

**Q.** For A\* searching Denver to NYC, describe what happens if (a) `h(v, goal) = 0` for all `v`, (b) `h(v, goal) = 10000` for all `v`, (c) `h(Chicago, goal) = h(Indianapolis, goal) = 100000` while other estimates are accurate.

**A.** (a) A\* degenerates to Dijkstra's: priorities are just `distTo`. (b) Also Dijkstra's: adding the same constant to every priority does not change the relative ordering. (c) **Incorrect output.** The heuristic grossly overestimates (it is inadmissible and inconsistent), so A\* dodges Illinois and Indiana and may return a non-shortest route. A bad heuristic is a correctness problem, not just a speed problem.

### 11. Single-target behavior

**Q.** Why is Dijkstra's a poor choice for a single-source single-target navigation app, and what does A\* give up in exchange for speed?

**A.** Dijkstra's explores in uniform expanding rings, so going Denver to NYC it examines nearly everything within thousands of miles of Denver before reaching the target. A\* biases the search toward the goal using `h`, stopping when the goal is dequeued. In exchange, it no longer produces a correct SPT for the other vertices (in the demo, `distTo[D]` ends up at 13 instead of 11), and its correctness now depends on the heuristic being well behaved.

---

## Summary

- **BFS is the wrong algorithm for weighted graphs.** It minimizes number of edges, not total weight. Mapping example: BFS returns ~330 m where ~130 m exists.
- **DFS vs BFS recap:** both correct and both `O(V+E)` time; BFS additionally guarantees fewest edges; DFS is bad on spindly graphs (deep call stack), BFS is bad on bushy graphs (huge queue); both are `Θ(V)` space anyway because of `distTo` and `edgeTo`.
- **The goal is the Shortest Paths Tree (SPT)**, the union of the shortest paths from `s` to every vertex. With non-negative weights, shortest paths have no cycles, the solution is always a tree, and it has exactly **`V - 1` edges** (one incoming edge per vertex, none for the source).
- **Representation:** `distTo[v]` (best known weight, init `0` for `s` and `∞` elsewhere) and `edgeTo[v]` (predecessor). Following `edgeTo` backwards reconstructs the path.
- **Relaxation is the universal improvement step.** `relax(E)`: if the edge is better, use it. `relax(v)`: relax all outgoing edges of `v`. Algorithms start from an incorrect SPT and relax until it is correct (Bellman-Ford 1955, Dijkstra 1956, Duan-Mao et al. 2025).
- **Dijkstra's algorithm** = BFS with a **priority queue** fringe: add all vertices at priority ∞ (source at 0), then repeatedly `removeMin()` and relax that node. Each node is relaxed exactly once, in **closest-first** order.
- **Correctness invariant:** once a vertex leaves the PQ, its `distTo` is final, because any edge into an already-visited vertex satisfies `distTo[v] + weight ≥ distTo[v] ≥ distTo[w]` when weights are non-negative.
- **Runtime:** `V` adds + `V` removeSmallests + `E` changePriorities, each `O(log V)`, so **`O(E log V)`** time and `Θ(V)` space for a connected graph.
- **Demo takeaways:** removal order `A, C, B, E, F, G, D` with non-decreasing distances; a direct heavy edge (`B→D`, 11) can lose to a long light detour (`A -> B -> E -> G -> D`, 11 total); and relaxations into visited vertices (`2+5 > 1`, `5+1 > 1`, `11+2 > 5`) always fail.
- **A\* (out of scope):** order by `distTo[v] + h(v, goal)` and stop when the goal is dequeued. Faster for single-target queries, visits fewer vertices, but produces only one correct path, not an SPT. `h` is fixed during the run; `h = 0` or any constant reduces A\* to Dijkstra's; an overestimating heuristic can make A\* **wrong**. Admissibility and consistency are the sufficient conditions (CS188 material).
- If all edge weights are equal, **Dijkstra's is just BFS**.
