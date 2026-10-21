<!-- Wed, Oct 21, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 24: Minimum Spanning Trees

This lecture shifts from shortest paths (one source, cheapest route to everything) to a source-free problem: given an undirected, edge-weighted graph, find the cheapest set of edges that keeps every vertex connected. That object is a *minimum spanning tree* (MST). The central theoretical tool is the **cut property**: for any way of splitting the vertices into two non-empty groups, the minimum-weight edge crossing between them must belong to the MST. From that single fact you get a generic MST algorithm (repeatedly find a cut with no MST edges crossing it, add the cheapest crossing edge, stop at V-1 edges), and both algorithms in this lecture are just clever ways of choosing those cuts. **Prim's algorithm** grows one tree outward from an arbitrary start vertex, always absorbing the cheapest edge leaving the tree, and is implementationally almost identical to Dijkstra's (O(E log V) with a binary-heap priority queue). **Kruskal's algorithm** instead sorts all edges and greedily adds any edge that does not create a cycle, using a weighted quick union with path compression to test for cycles, giving O(E log E), or O(E log* V) if the edges arrive pre-sorted. The lecture also carefully separates MSTs from shortest paths trees: an SPT depends on a start vertex, an MST does not, and there are graphs where no vertex's SPT is an MST.

---

## Key Concepts

### Warm-up: cycle detection in an undirected graph

Before MSTs, the lecture warms up with: given an undirected graph, does it contain a cycle? Two approaches, both reusing machinery from earlier in the course.

**Approach 1: DFS.** Start a DFS from an arbitrary vertex and keep going until you reach a vertex that is already marked. If you reach a marked vertex, you have found a second route to it, i.e. a cycle.

The subtlety: in an *undirected* graph, every edge looks like a 2-cycle. If you walk A to B, then from B you immediately see A, which is marked, and you would wrongly declare a cycle. The fix is simple: **do not count the node you came from.** Track the parent of each DFS call and skip it.

Runtime: O(V + E) in the worst case. The slides note a tighter bound of O(V): if the graph has more than V-1 edges it must already contain a cycle, so you can stop after examining at most V edges. Hence the number of edges you actually inspect is O(V), and O(V + E) collapses to O(V).

*(extra context)* Precisely: a connected acyclic graph on V vertices has exactly V-1 edges, so once the count of examined edges reaches V you can return `true` immediately without looking further.

**Approach 2: Disjoint sets (`WeightedQuickUnionUF`).** Walk through the edges one at a time. For each edge v-w:
- if `connected(v, w)` is already true, then v and w were joined by some earlier path, so adding this edge closes a cycle: return `true`;
- otherwise `union(v, w)` and continue.

Runtime: O(V + E·α(V)) with path compression, where α is the inverse Ackermann function from the disjoint sets lecture. By the same edge-counting argument, this simplifies to O(V·α(V)).

This second approach is worth internalizing, because **it is exactly the cycle test inside Kruskal's algorithm.**

### Spanning trees

Given an undirected graph G, a subgraph T is a **spanning tree** when three things hold:

1. T is connected,
2. T is acyclic (1 and 2 together make it a *tree*),
3. T includes all of G's vertices (this makes it *spanning*).

A **minimum spanning tree** is a spanning tree of minimum total weight (sum of its edge weights). The motivating example on the slides: a network of power lines connecting a set of buildings. You need every building reachable (spanning, connected), you gain nothing from redundant loops (acyclic), and you want to minimize total cable laid (minimum weight).

Two consequences worth keeping in your head:
- A spanning tree on V vertices always has exactly **V-1 edges.** This is why every algorithm below has "repeat until V-1 edges" as its stopping condition.
- A spanning tree exists only if G is connected. *(extra context)* If G is disconnected you instead get a minimum spanning *forest*, one tree per connected component; the lecture's algorithms both naturally produce this if you let them run out of usable edges.

A common error is to call a subgraph a spanning tree when it misses a vertex, or when it is a tree but leaves the graph disconnected. The "Which are Spanning Trees?" poll on the slides is testing exactly this: check all three properties, every time.

### Applications

The slides mention old-school handwriting recognition and medical imaging (for example, characterizing the arrangement of nuclei in cancer cells), and point at Eppstein's MST applications page (`http://www.ics.uci.edu/~eppstein/gina/mst.html`) for more.

### MST vs. shortest paths tree (SPT)

*(The slides mark this section as "covered in the web videos, but we won't cover them live." It is still fair game conceptually, and it is a classic exam trap, so study it.)*

The structural difference:

- A **shortest paths tree depends on the start vertex**, because its job is to tell you how to get from *that source* to everything else. Change the source, change the tree.
- An **MST has no source.** It is a property of the graph alone.

So the two are different kinds of object. Nonetheless, an MST sometimes *happens* to coincide with the SPT of some particular vertex.

The lecture's small example is a 4-vertex graph with edges A-B (weight 2), B-C (weight 2), C-D (weight 2), and A-D (weight 3). The MST takes the three weight-2 edges (total 6) and drops A-D. Is that tree an SPT?

- From **D**: it does not work. In the tree, D must reach A via D-C-B-A for cost 6, but the direct edge A-D costs 3. An SPT rooted at D would use A-D.
- From **B**: it does work, and the SPT from B is exactly the MST.
- From A, from C: those SPTs include the A-D edge, so they differ from the MST.

The deeper point is captured by the second example: the "ladder" graph with two weight-2 rungs in the middle and weight-1 edges elsewhere (two hub vertices labeled 0 on the far left and far right, with parallel top and bottom chains). Here the answer is **no vertex's SPT is the MST**:

- the MST needs only **2** of the weight-2 edges,
- but any SPT includes **at least 3** of the weight-2 edges.

Why, intuitively: the MST only has to connect the two halves of the ladder, and crossing the weight-2 middle region cheaply once (plus whatever is forced) suffices. An SPT must give *every* vertex its own shortest route back to the source, and vertices on the far side of the middle each demand an individually short crossing, which forces extra weight-2 edges into the tree. So minimizing *total* weight and minimizing *each individual distance to a source* are genuinely different objectives.

### The cut property (the key tool)

Two definitions first:

- A **cut** is an assignment of a graph's nodes to two non-empty sets.
- A **crossing edge** is an edge connecting a node in one set to a node in the other.

> **Cut property:** Given any cut, the minimum-weight crossing edge is in the MST.

For the rest of the lecture, **edge weights are assumed unique.** This makes "the MST" well defined and makes "the minimum crossing edge" unambiguous.

**Worked instance from the slides.** Take the standard 8-vertex weighted graph with edge list:

```
0-7 0.16    5-7 0.28    4-5 0.35    6-2 0.40
2-3 0.17    1-3 0.29    1-2 0.36    3-6 0.52
1-7 0.19    1-5 0.32    4-7 0.37    6-0 0.58
0-2 0.26    2-7 0.34    0-4 0.38    6-4 0.93
```

Consider the cut that puts {2, 3, 5, 6} on one side and {0, 1, 4, 7} on the other. The crossing edges are those with exactly one endpoint in {2,3,5,6}: 0-2 (0.26), 5-7 (0.28), 1-3 (0.29), 1-5 (0.32), 2-7 (0.34), 1-2 (0.36), 4-5 (0.35), 6-2 is internal (both 6 and 2 are inside, so not crossing), 2-3 internal, 3-6 internal, 6-0 (0.58), 6-4 (0.93), 0-4 is internal to the other side, 0-7 internal, 1-7 internal, 4-7 internal.

The minimum crossing edge is **0-2 at 0.26**, so **0-2 must be part of the MST.** Note how to do this mechanically: scan the edge list in increasing weight order and return the first edge whose two endpoints land on opposite sides of the cut. Since the list is already sorted, 0-7 (both outside) fails, 2-3 (both inside) fails, 1-7 (both outside) fails, 0-2 succeeds. Done.

**Proof of the cut property (exchange argument).** Let X be the MST, and suppose the minimum crossing edge e is *not* in X.

1. X is a spanning tree, so it already connects e's two endpoints. Adding e to X therefore creates a cycle.
2. That cycle starts on one side of the cut and returns, so some *other* edge f in X on that cycle must also be a crossing edge.
3. Since e is the *minimum* crossing edge and weights are unique, weight(e) < weight(f). Remove f and add e: you still have a spanning tree (removing an edge from a cycle cannot disconnect anything), and its total weight is strictly lower.
4. That contradicts X being minimum. Contradiction, so e is in the MST. ∎

Note where uniqueness is used: step 3 needs a *strict* inequality. *(extra context)* With ties, the honest statement is that the minimum crossing edge is in *some* MST, and both Prim's and Kruskal's still produce a valid MST; they just may produce different ones depending on tie-breaking.

### The generic MST algorithm

The cut property immediately yields a template:

```
Start with no edges in the MST.
  - Find a cut that has no crossing edges currently in the MST.
  - Add the smallest crossing edge of that cut to the MST.
  - Repeat until V-1 edges.
```

The "no crossing edges in the MST yet" condition matters: it guarantees the cheapest crossing edge is a *new* edge you have not already committed to, and that adding it cannot create a cycle.

The slides are blunt about the catch: this works, but **we need some way of finding such a cut, and picking randomly is not a good idea.** Prim's and Kruskal's are two systematic answers to "which cut?", and that is precisely why each one's correctness argument is just "it is a special case of the generic algorithm."

### Prim's algorithm

**Conceptual version:**

```
Start from some arbitrary start node.
  - Repeatedly add the shortest edge that has exactly one node inside
    the MST under construction.
  - Repeat until V-1 edges.
```

**Why it works, via the cut property.** At every step, define the cut as:
- one side: everything currently connected to the start vertex (the tree so far, "white" vertices on the slides),
- other side: everything not yet connected to the start ("green" vertices).

No MST edge crosses this cut by construction (every committed edge has both endpoints in the tree). Prim's always adds the smallest-weight edge connecting white to green, which is exactly the minimum crossing edge of that cut. By the cut property it belongs to the MST. So Prim's is a special case of the generic algorithm.

**Why the naive implementation is slow.** The conceptual version says "look at all edges leaving the tree and pick the smallest." Re-scanning all of those edges from scratch on every one of the V-1 iterations is wasteful: most of them were already examined and already lost. The lecture's example graph for this (start vertex A):

```
A-B 2, A-C 1, B-C 3 (or similar middle structure), B-D 11, B-E 5,
C-E 1, C-F 15, D-E 2, D-G 1, E-F 4, E-G 3, F-G 1
```

(The exact edge set is partly obscured by the PDF extraction; the edge weights visible on the slide are 11, 1, 2, 2, 3, 5, 2, 3, 1, 1, 4, 1, 15. Use the structure, not the precise layout.)

**Optimized version:** maintain a priority queue of vertices, keyed by the *best known edge weight connecting that vertex to the tree*. The slides describe it as "very similar to Dijkstra's," and that is the single most useful thing to remember.

### Prim's vs. Dijkstra's

> Prim's and Dijkstra's algorithms are exactly the same, except Dijkstra's considers "distance from the source" and Prim's considers "distance from the tree."

Side by side:

| | Dijkstra's | Prim's |
|---|---|---|
| Visit order | vertices in order of distance from the source | vertices in order of distance from the MST under construction |
| Relaxation test | is this edge better *based on distance to source*? i.e. `distTo[v] + e.weight() < distTo[w]` | is this edge better *based on distance to tree*? i.e. `e.weight() < distTo[w]` |
| `distTo[w]` means | length of best known path from s to w | weight of best known single edge from the tree to w |
| Output | shortest paths tree from s | minimum spanning tree |

Everything else (the fringe as a priority queue of vertices, `delMin`, `marked[]`, `edgeTo[]`, `decreasePriority`) is identical. In Dijkstra's the relaxation accumulates path length; in Prim's it does not accumulate at all, because "distance to the tree" is just the weight of one edge.

### Kruskal's algorithm

```
Initially mark all edges gray.
  - Consider edges in increasing order of weight.
  - Add edge to MST (mark black) unless doing so creates a cycle.
  - Repeat until V-1 edges.
```

**Why it works, via the cut property.** Suppose Kruskal's is about to add edge e = v-w. Define the cut as:
- side 1: all vertices currently connected to v (in the partial MST),
- side 2: everything else (which contains w, otherwise e would create a cycle and we would skip it).

Then:
- **No crossing edge is already black**, because any black crossing edge would have put w's component on side 1.
- **No crossing edge has lower weight than e**, because we process edges in increasing weight order, so every cheaper edge has already been considered and either added or rejected.

So e is the minimum crossing edge of this cut, and by the cut property it is in the MST. Again, a special case of the generic algorithm.

**Basic vs. optimized Kruskal's.** The slides reference separate demos for each. The basic version needs a way to answer "would adding v-w create a cycle?" The optimized version answers it with a **weighted quick union with path compression**, exactly as in the warm-up problem: v-w creates a cycle iff `connected(v, w)` is already true. Each accepted edge triggers a `union`.

### Prim's vs. Kruskal's (the visual intuition)

The slides show side-by-side animations. The qualitative difference:

- **Prim's** always has *one* connected tree, which grows outward from the start vertex like a blob. At every moment the partial result is a single connected subtree.
- **Kruskal's** scatters edges all over the graph, maintaining a *forest* of many disconnected fragments that gradually merge. Only at the very end does it become one tree.

Both end at the same MST (with unique weights).

### Runtimes

**Prim's**, assuming a binary-heap-based priority queue with O(log V) operations:

| | # Operations | Cost per operation | Total cost |
|---|---|---|---|
| PQ add | V | O(log V) | O(V log V) |
| PQ delMin | V | O(log V) | O(V log V) |
| PQ decreasePriority | O(E) | O(log V) | O(E log V) |

Overall: O(V log V + V log V + E log V). Assuming E > V, this is **O(E log V)**, same as Dijkstra's.

Reasoning for the counts: each vertex is inserted exactly once and deleted exactly once, so V of each. `decreasePriority` can fire at most once per edge examination, and across the whole run each edge is examined from each endpoint, so O(E) times.

**Kruskal's**, with a `MinPQ<Edge>` and `WeightedQuickUnionPC`:

| Operation | Number of times | Time per operation | Total time |
|---|---|---|---|
| Insert | E | O(log E) | O(E log E) |
| Delete minimum | O(E) | O(log E) | O(E log E) |
| union | O(V) | O(log* V) | O(V log* V) |
| isConnected | O(E) | O(log* V) | O(E log* V) |

Overall: **O(E log E)**.

Two important notes from the slides:

- **Note 1 (pre-sorted edges):** if you already have a sorted list of edges instead of a PQ, you just iterate through it in O(E) time, so the total becomes O(E + V log\* V + E log\* V) = **O(E log\* V)**. This is asymptotically better than O(E log V).
- **Note 2 (E log E vs. E log V):** E < V², so log E < log V² = 2 log V, hence **O(E log E) = O(E log V)**. So although Kruskal's with an unsorted edge set is slower than Prim's in practice on worst-case input, it is *not asymptotically slower*.
- *(Fun fact from the slides)* The E-insertions step can be done in O(E) via "bottom-up heapification," covered in the HeapSort lecture.

**Summary table:**

| Problem | Algorithm | Runtime (if E > V) | Notes |
|---|---|---|---|
| Shortest Paths | Dijkstra's | O(E log V) | Fails for negative weight edges. |
| MST | Prim's | O(E log V) | Analogous to Dijkstra's. |
| MST | Kruskal's | O(E log E) | Uses WQUPC. |
| MST | Kruskal's with pre-sorted edges | O(E log* V) | Uses WQUPC. |

**Can we do better than O(E log V)?** Yes. From the (clearly labeled bonus / "170 spoiler") history slide:

| Year | Worst case | Discovered by |
|---|---|---|
| 1975 | E log log V | Yao |
| 1984 | E log* V | Fredman-Tarjan |
| 1986 | E log(log* V) | Gabow-Galil-Spencer-Tarjan |
| 1997 | E α(V) log α(V) | Chazelle |
| 2000 | E α(V) | Chazelle |
| 2002 | optimal | Pettie-Ramachandran |
| ??? | E ??? | ??? |

A linear-time comparison-based MST algorithm is still an open problem: Pettie-Ramachandran gives an algorithm that is provably optimal, but nobody knows what that optimum's asymptotic form actually is.

---

## Definitions

- **Spanning tree:** Given an undirected graph G, a subgraph T that is (1) connected, (2) acyclic, and (3) includes all of G's vertices. Properties (1) and (2) make it a tree; (3) makes it spanning.
- **Minimum spanning tree (MST):** A spanning tree of minimum total weight, where total weight is the sum of the weights of its edges.
- **Cut:** An assignment of a graph's nodes to two non-empty sets.
- **Crossing edge:** An edge that connects a node in one set of a cut to a node in the other set.
- **Cut property:** Given any cut, the minimum-weight crossing edge is in the MST. (Stated under the assumption of unique edge weights.)
- **Prim's algorithm:** Starting from an arbitrary vertex, repeatedly add the shortest edge with exactly one endpoint inside the MST under construction, until V-1 edges have been added.
- **Kruskal's algorithm:** Consider edges in increasing order of weight, adding each to the MST unless doing so creates a cycle, until V-1 edges have been added.
- **Shortest paths tree (SPT):** A tree encoding shortest paths from one designated source vertex to every other vertex. Depends on the choice of source; an MST does not.
- **Scan (a vertex):** To consider all of a vertex's outgoing edges (terminology from the Prim's pseudocode).
- **Relaxation (in Prim's):** Updating `distTo[w]` and `edgeTo[w]` when an edge offers a cheaper connection from the current tree to w. Contrast with Dijkstra's, where relaxation measures distance to the source.
- **α(V):** The inverse Ackermann function, from the disjoint sets lecture; grows extraordinarily slowly.
- **log\* V:** The iterated logarithm, the WQU-with-path-compression bound used in the Kruskal's analysis.

---

## Worked Examples

### Example 1: Cycle detection with disjoint sets (the warm-up, Approach 2)

This is the lecture's Approach 2 written out. It is worth coding because it *is* the heart of Kruskal's.

```java
public static boolean hasCycle(EdgeWeightedGraph G) {
    WeightedQuickUnionUF uf = new WeightedQuickUnionUF(G.V());
    for (Edge e : G.edges()) {
        int v = e.from();
        int w = e.to();
        if (uf.connected(v, w)) {
            return true;      // v and w were already joined: this edge closes a cycle
        }
        uf.union(v, w);       // otherwise merge their components and keep going
    }
    return false;
}
```

**Step by step on the warm-up graph** (edges A-B, A-C, B-C, B-D, D-E, E-G, E-F, with A through G mapped to 0 through 6):

- Start: seven singleton sets, `{A} {B} {C} {D} {E} {F} {G}`. In box-and-pointer terms, the WQU's `parent` array has every slot pointing at itself (each entry stores its own index, or a negative size sentinel depending on the implementation).
- Edge A-B: `connected(A, B)` is false (different roots). `union` makes one root point at the other. Sets: `{A,B} {C} {D} {E} {F} {G}`.
- Edge A-C: false. Union. `{A,B,C} {D} {E} {F} {G}`.
- Edge B-C: `connected(B, C)`: follow B's parent pointers up to the root, follow C's parent pointers up to the root, **same root**. Return `true`: cycle found (A-B-C-A).

**Why it works:** the disjoint sets structure answers "are these two already in the same connected component?" An edge whose endpoints are already connected adds a second route between them, which is the definition of a cycle.

**Runtime:** O(V + E·α(V)) with path compression, which tightens to O(V·α(V)) because you can bail out after examining at most V edges (more than V-1 edges in a graph forces a cycle).

### Example 2: Prim's algorithm pseudocode (from the slides)

```java
public class PrimMST {
    public PrimMST(EdgeWeightedGraph G) {
        edgeTo = new Edge[G.V()];
        distTo = new double[G.V()];
        marked = new boolean[G.V()];
        fringe = new SpecialPQ<Double>(G.V());

        distTo[s] = 0.0;
        setDistancesToInfinityExceptS(s);
        insertAllVertices(fringe);

        /* Get vertices in order of distance from tree. */
        while (!fringe.isEmpty()) {
            int v = fringe.delMin();
            scan(G, v);
        }
    }

    private void scan(EdgeWeightedGraph G, int v) {
        marked[v] = true;
        for (Edge e : G.adj(v)) {
            int w = e.other(v);
            if (marked[w]) { continue; }
            if (e.weight() < distTo[w]) {
                distTo[w] = e.weight();
                edgeTo[w] = e;
                pq.decreasePriority(w, distTo[w]);
            }
        }
    }
}
```

**Reading it line by line:**

- `edgeTo[w]`: the cheapest known edge connecting w to the tree. At termination, the V-1 non-null entries of `edgeTo` *are* the MST.
- `distTo[w]`: the weight of that edge, i.e. w's current best known distance to the tree. Initialized to infinity for every vertex except s, which gets 0.0 so that `delMin` pulls it out first.
- `marked[w]`: true once w has been removed from the fringe and scanned, meaning w is committed to the tree.
- `fringe`: the slide annotation is important: *"Fringe is ordered by distTo tree. Must be a specialPQ like Dijkstra's."* It needs `decreasePriority`, which a plain `MinPQ` does not support, hence `SpecialPQ`. All V vertices are inserted up front.
- The main loop: `delMin()` gets the unvisited vertex closest to the tree. `scan` means "consider all of a vertex's outgoing edges."
- Inside `scan`: mark v (it is closest, so add it to the MST), then for each neighbor w, skip w if it is already in the MST (`continue`), otherwise test `e.weight() < distTo[w]`. If this edge is a better way to attach w to the tree, update `distTo[w]`, record `edgeTo[w] = e`, and bubble w up in the fringe with `decreasePriority`.

**The one line to stare at** is `if (e.weight() < distTo[w])`. In Dijkstra's this would be `if (distTo[v] + e.weight() < distTo[w])`. That single difference is the whole difference between SPT and MST: Prim's compares a bare edge weight ("how cheaply can w attach to the tree?"), Dijkstra's compares an accumulated path length ("how cheaply can w be reached from s?").

**Environment / box-and-pointer reasoning in words.** Think of three parallel arrays indexed by vertex number, laid out side by side. `distTo` is a row of doubles, all `Infinity` except `distTo[s] = 0.0`. `marked` is a row of `false`. `edgeTo` is a row of `null` references. The fringe holds V vertex ids whose ordering is determined by reading into `distTo`.

Running on the lecture's example graph starting at A:
1. All of A, B, C, D, E, F, G sit in the fringe. `distTo[A] = 0.0`, everything else `Infinity`.
2. `delMin` returns A. `marked[A] = true`. Scan A's edges: A-C with weight 1 beats `distTo[C] = Infinity`, so `distTo[C] = 1` and `edgeTo[C]` now points at the A-C `Edge` object (the array slot is overwritten to reference that object; no copy is made, the `Edge` lives on the heap and both the graph's adjacency list and `edgeTo[C]` point at the same one). Similarly A-B with weight 2 sets `distTo[B] = 2`, `edgeTo[B]` to the A-B edge.
3. `delMin` returns C (`distTo[C] = 1`, the smallest). `marked[C] = true`. Scan C: for edge C-E of weight 1, `1 < distTo[E] = Infinity`, so `distTo[E] = 1`, `edgeTo[E]` points at C-E. For C-B, if its weight exceeds the current `distTo[B] = 2`, the test fails and **nothing is written**. That failed test is the efficiency win over naive Prim's: a losing edge is dismissed in constant time and never looked at again.
4. Continue: `delMin` next yields E, then its neighbors get relaxed, and so on, until the fringe empties after V `delMin` calls and `edgeTo` holds V-1 edges forming the MST.

Key invariant, stated on the slide: **the fringe must be ordered by current best known distance from the tree.** This is why `decreasePriority` exists; mutating `distTo[w]` without telling the PQ would corrupt the heap ordering.

### Example 3: Kruskal's algorithm pseudocode (from the slides)

```java
public class KruskalMST {
    private List<Edge> mst = new ArrayList<Edge>();

    public KruskalMST(EdgeWeightedGraph G) {
        MinPQ<Edge> pq = new MinPQ<Edge>();
        for (Edge e : G.edges()) {
            pq.insert(e);
        }
        WeightedQuickUnionPC uf =
                 new WeightedQuickUnionPC(G.V());
        while (!pq.isEmpty() && mst.size() < G.V() - 1) {
            Edge e = pq.delMin();
            int v = e.from();
            int w = e.to();
            if (!uf.connected(v, w)) {
                uf.union(v, w);
                mst.add(e);
            }
        }
    }
}
```

**Reading it line by line:**

- `mst`: an `ArrayList<Edge>` accumulating the answer. Note that Kruskal's naturally produces a *list of edges*, whereas Prim's produces an `edgeTo` array indexed by vertex.
- `MinPQ<Edge>`: all E edges are inserted. Note this PQ is over **edges**, keyed by weight, not over vertices. `delMin` therefore yields edges in increasing order of weight, which is exactly the "consider edges in increasing order of weight" step. A plain `MinPQ` suffices; no `decreasePriority` is needed, because edge weights never change.
- `WeightedQuickUnionPC uf = new WeightedQuickUnionPC(G.V())`: V singleton components, one per vertex. This is the cycle detector from Example 1.
- Loop condition `!pq.isEmpty() && mst.size() < G.V() - 1`: stop early once V-1 edges are in (the tree is complete) or once edges run out. *(extra context)* The `!pq.isEmpty()` half is what keeps the loop safe on a disconnected graph, where you can never reach V-1 edges and instead end with a minimum spanning forest.
- `if (!uf.connected(v, w))`: "would adding this edge create a cycle?" If v and w are in different components, no cycle, so `union` them and commit the edge. If they are already connected, the edge is silently discarded.

**Step by step** on the 8-vertex graph from the cut property example (edges already listed in increasing order):

1. `0-7 (0.16)`: 0 and 7 in different components. Add. Components: `{0,7}` plus singletons. MST size 1.
2. `2-3 (0.17)`: different. Add. `{0,7} {2,3}`. Size 2.
3. `1-7 (0.19)`: 1 and 7 differ. Add, `union` merges. `{0,1,7} {2,3}`. Size 3.
4. `0-2 (0.26)`: 0 is in `{0,1,7}`, 2 is in `{2,3}`. Different. Add. `{0,1,2,3,7}`. Size 4. (Notice: this is the same edge the cut property analysis forced into the MST.)
5. `5-7 (0.28)`: different. Add. `{0,1,2,3,5,7}`. Size 5.
6. `1-3 (0.29)`: `connected(1, 3)` is **true** (both in the big component). Skip, it would create a cycle.
7. `1-5 (0.32)`: connected. Skip.
8. `2-7 (0.34)`: connected. Skip.
9. `4-5 (0.35)`: 4 is still a singleton. Different. Add. `{0,1,2,3,4,5,7}`. Size 6.
10. `1-2 (0.36)`, `4-7 (0.37)`, `0-4 (0.38)`: all connected. Skip, skip, skip.
11. `6-2 (0.40)`: 6 is a singleton. Add. `{0,...,7}`. Size 7 = V-1. **Loop exits.**

MST edges: 0-7, 2-3, 1-7, 0-2, 5-7, 4-5, 6-2. Total weight 0.16+0.17+0.19+0.26+0.28+0.35+0.40 = 1.81. Observe the Kruskal's signature: the partial result was a scattered forest (`{0,7}` and `{2,3}` were separate islands for several steps) that only became one tree at the very end.

### Example 4: Finding an MST by hand on the 4-vertex example

Graph: A-B (2), B-C (2), C-D (2), A-D (3).

Via Kruskal's: the three weight-2 edges come first. A-B: add. B-C: add. C-D: add, now 3 = V-1 edges, stop. A-D (3) is never even considered. MST weight = 6.

Via Prim's from A: fringe gives A first. Scan A: `distTo[B] = 2`, `distTo[D] = 3`. `delMin` gives B (2). Scan B: C gets `distTo[C] = 2`. `delMin` gives C (2, tie with nothing smaller). Scan C: edge C-D has weight 2, and `2 < distTo[D] = 3`, so `distTo[D]` drops to 2 and `edgeTo[D]` is overwritten from A-D to C-D. `delMin` gives D. Result: A-B, B-C, C-D, weight 6.

That overwrite of `edgeTo[D]` is worth noticing: Prim's tentatively planned to attach D via A-D and later found a better attachment. This is relaxation in action.

---

## Common Pitfalls

1. **Forgetting the "don't count the parent" fix in undirected DFS cycle detection.** Every undirected edge trivially looks like a cycle of length 2. Track the vertex you came from and skip it.

2. **Using Dijkstra's relaxation condition in Prim's.** Writing `distTo[v] + e.weight() < distTo[w]` turns Prim's back into Dijkstra's and gives you an SPT, not an MST. Prim's condition is `e.weight() < distTo[w]`: no accumulation.

3. **Thinking an MST is an SPT, or that an SPT is an MST.** An SPT depends on a start vertex; an MST does not. The ladder graph on the slides is a counterexample where *no* vertex's SPT is the MST. They can coincide (vertex B in the 4-cycle example), but that is a coincidence, not a rule.

4. **Misapplying the cut property's quantifier.** The property says: for *every* cut, the minimum crossing edge is in the MST. It does **not** say the minimum edge incident to a vertex set is the only MST edge crossing it, and it does **not** say the *maximum* crossing edge is excluded in general. Also note that an internal edge (both endpoints on the same side) is not a crossing edge at all, so it is irrelevant to that cut.

5. **Forgetting that a cut requires both sets to be non-empty.** "All vertices on one side" is not a cut and has no crossing edges.

6. **Checking only two of the three spanning tree properties.** Connected, acyclic, *and* includes all vertices. A subgraph can be a perfectly good tree while omitting a vertex, in which case it is not spanning.

7. **Using a plain `MinPQ` for Prim's fringe.** Prim's needs `decreasePriority` because a vertex's key changes as cheaper connecting edges are discovered. The slides call this a `SpecialPQ`, same as Dijkstra's. Kruskal's, by contrast, is fine with a plain `MinPQ<Edge>` since edge weights never change.

8. **Mixing up what is in the priority queue.** Prim's PQ holds **vertices** keyed by distance to the tree (size V). Kruskal's PQ holds **edges** keyed by weight (size E).

9. **Claiming Kruskal's is asymptotically slower than Prim's.** It is not: E < V², so O(E log E) = O(E log V). It can be slower in practice on unsorted input, but the asymptotics match.

10. **Forgetting the pre-sorted optimization and what it buys.** With pre-sorted edges Kruskal's is O(E log\* V), beating O(E log V). The savings come from skipping the PQ entirely, not from a faster union-find.

11. **Assuming unique MSTs in general.** The lecture assumes unique edge weights specifically so that "the MST" is well defined. With ties, multiple distinct MSTs can exist (all with the same total weight).

12. **Dropping the `mst.size() < G.V() - 1` early-stop or the `!pq.isEmpty()` guard in Kruskal's.** Without the second guard a disconnected graph loops forever or crashes on an empty PQ.

---

## Likely Exam Points

### 1. Apply the cut property to a given cut

**Q:** Using the edge list from lecture (0-7 0.16, 2-3 0.17, 1-7 0.19, 0-2 0.26, 5-7 0.28, 1-3 0.29, 1-5 0.32, 2-7 0.34, 4-5 0.35, 1-2 0.36, 4-7 0.37, 0-4 0.38, 6-2 0.40, 3-6 0.52, 6-0 0.58, 6-4 0.93), which edge is the minimum-weight edge crossing the cut {2, 3, 5, 6}, and what can you conclude?

**A:** Scan in increasing weight order and take the first edge with exactly one endpoint in {2,3,5,6}. 0-7: both outside, no. 2-3: both inside, no. 1-7: both outside, no. 0-2: 0 outside, 2 inside, **yes**. So the answer is **0-2 (weight 0.26)**, and by the cut property it **must be part of the MST**.

### 2. Prove or state the cut property

**Q:** State the cut property and sketch its proof.

**A:** *Statement:* given any cut of the graph, the minimum-weight crossing edge is in the MST (assuming unique edge weights). *Proof:* let X be the MST and suppose the minimum crossing edge e is not in X. Since X spans the graph, adding e to X creates a cycle. That cycle crosses the cut and must come back, so it contains another crossing edge f in X. Because e is the minimum crossing edge, weight(e) < weight(f). Swapping (remove f, add e) yields a spanning tree of strictly lower weight, contradicting the minimality of X. Hence e is in X.

### 3. Prim's vs. Dijkstra's: the one-line difference

**Q:** You are handed working Dijkstra's code. What is the minimal change that turns it into Prim's, and why does that change work?

**A:** Change the relaxation condition from `distTo[v] + e.weight() < distTo[w]` to `e.weight() < distTo[w]` (and set the initial source distance to 0 as before). Dijkstra's keys each vertex by distance from the *source*, so it accumulates path weight; Prim's keys each vertex by distance from the *tree under construction*, which is just the weight of a single connecting edge. Consequently Dijkstra's visits vertices in order of distance from the source and Prim's visits them in order of distance from the MST under construction.

### 4. Trace Kruskal's and/or Prim's

**Q:** On the graph A-B (2), B-C (2), C-D (2), A-D (3), give the MST and its weight. Then say which vertex's SPT, if any, equals that MST.

**A:** MST = {A-B, B-C, C-D}, weight 6 (A-D is excluded). The SPT from **B** equals this MST. It fails for D (D would reach A directly via A-D at cost 3 rather than 6 around the tree), and the SPTs from A and C also include A-D.

### 5. Is the MST ever an SPT? (the ladder graph)

**Q:** For the "ladder" graph with two weight-2 rungs in the middle and weight-1 edges everywhere else, is there a vertex whose SPT equals the MST? Justify.

**A:** **No.** The MST includes only **2** of the weight-2 edges, but any SPT includes at **least 3** of them. Since the edge sets differ, no SPT can equal the MST. The general reason: an MST minimizes total weight, while an SPT must minimize each vertex's individual distance to the source, and those objectives force different edges here.

### 6. Runtime of Prim's with a binary heap

**Q:** Give the runtime of Prim's, assuming all PQ operations are O(log V), and justify each term.

**A:** V inserts at O(log V) = O(V log V); V `delMin`s at O(log V) = O(V log V); O(E) `decreasePriority` calls at O(log V) = O(E log V). Total O(V log V + V log V + E log V), which for E > V is **O(E log V)**, the same as Dijkstra's. The `decreasePriority` count is O(E) because each edge can trigger at most a constant number of successful relaxations across the run.

### 7. Runtime of Kruskal's, including both variants

**Q:** Give Kruskal's runtime with a `MinPQ`, and with a pre-sorted edge list. Is Kruskal's asymptotically slower than Prim's?

**A:** With a `MinPQ`: E inserts and O(E) `delMin`s at O(log E) each, O(V) `union`s and O(E) `connected`s at O(log\* V) each, total **O(E log E)**. With pre-sorted edges you skip the PQ and iterate in O(E), giving O(E + V log\* V + E log\* V) = **O(E log\* V)**. Kruskal's is **not** asymptotically slower than Prim's: E < V², so log E < 2 log V, hence O(E log E) = O(E log V). It can be slower in practice on worst-case unsorted input.

### 8. Identify valid spanning trees

**Q:** Which of the three subgraphs shown is a spanning tree? What do you check?

**A:** Check all three properties: connected, acyclic, includes every vertex of G. A subgraph with a cycle fails, a subgraph that leaves some vertex unreachable fails (either disconnected or not spanning), and only one that satisfies all three qualifies. A quick sanity check: a spanning tree of a V-vertex graph has exactly V-1 edges, so any candidate with a different edge count is immediately out.

### 9. Why does Prim's/Kruskal's work?

**Q:** Explain why Kruskal's algorithm produces an MST, referring to the cut property.

**A:** It is a special case of the generic MST algorithm. When Kruskal's adds edge e = v-w, take the cut whose side 1 is all vertices currently connected to v and whose side 2 is everything else (containing w). No crossing edge is already in the MST under construction, because such an edge would have merged w's component into side 1. No crossing edge has lower weight than e, because edges are considered in increasing weight order and all lighter edges were already handled. So e is the minimum crossing edge of that cut and belongs to the MST by the cut property. (For Prim's, the cut is "everything connected to the start vertex" versus "everything else," and Prim's by definition adds the lightest edge crossing it.)

### 10. Cycle detection (the warm-up)

**Q:** Give two ways to test an undirected graph for a cycle, with runtimes.

**A:** (a) DFS from an arbitrary vertex until you encounter a marked vertex, taking care not to count the vertex you came from: O(V + E), tightenable to O(V) since you need examine at most V edges. (b) Walk the edges with a `WeightedQuickUnionUF`: for each edge v-w, if `connected(v, w)` return true, else `union(v, w)`. O(V + E·α(V)) with path compression, tightenable to O(V·α(V)).

---

## Summary

- A **spanning tree** is a subgraph that is connected, acyclic, and includes all vertices; an **MST** is one of minimum total weight. A spanning tree on V vertices has exactly V-1 edges.
- **Warm-up (cycle detection in an undirected graph):** DFS until you hit a marked vertex, ignoring the vertex you came from, in O(V + E) (tightenable to O(V)); or disjoint sets, adding each edge unless its endpoints are already `connected`, in O(V + E·α(V)) (tightenable to O(V·α(V))). The second method is the cycle test inside Kruskal's.
- **MST is not an SPT.** An SPT depends on a start vertex; an MST has no source. Sometimes an MST coincides with the SPT of a particular vertex (vertex B in the 4-cycle example), and sometimes no vertex works at all (the ladder graph, where the MST uses 2 weight-2 edges but every SPT uses at least 3).
- A **cut** assigns the vertices to two non-empty sets; a **crossing edge** has one endpoint in each. **Cut property:** the minimum-weight crossing edge of any cut is in the MST. Proved by an exchange argument: adding the minimum crossing edge to the MST creates a cycle containing another crossing edge of larger weight, and swapping them gives a lighter spanning tree, a contradiction. Assumes unique edge weights.
- **Generic MST algorithm:** repeatedly find a cut with no MST edges crossing it, add its smallest crossing edge, stop at V-1 edges. The hard part is choosing the cut.
- **Prim's:** from an arbitrary start, repeatedly add the shortest edge with exactly one endpoint in the tree so far, until V-1 edges. Correct because the cut is (connected to start) vs. (not connected to start). The optimized version is Dijkstra's with the relaxation condition changed from `distTo[v] + e.weight() < distTo[w]` to `e.weight() < distTo[w]`; it needs a PQ of **vertices** supporting `decreasePriority`. **O(E log V)** for E > V.
- **Kruskal's:** consider edges in increasing weight order, adding each unless it creates a cycle, until V-1 edges. Correct because the cut is (connected to v) vs. (everything else): no crossing edge is black and none is lighter. Uses a `MinPQ` of **edges** plus `WeightedQuickUnionPC` for the cycle test. **O(E log E)**, or **O(E log\* V)** with pre-sorted edges.
- Prim's maintains one growing connected tree; Kruskal's maintains a scattered forest that merges into a tree only at the end. Both produce the same MST under unique weights.
- Since E < V², O(E log E) = O(E log V), so Kruskal's is not *asymptotically* slower than Prim's even though it can lose in practice on unsorted input.
- **Reference table:** Dijkstra's O(E log V) (fails on negative weights); Prim's O(E log V); Kruskal's O(E log E); Kruskal's with pre-sorted edges O(E log\* V).
- *(Bonus, CS 170 territory)* Faster MST algorithms exist (Yao 1975 through Chazelle's O(E α(V)) in 2000), and Pettie-Ramachandran 2002 is provably optimal, though the optimal asymptotic form is still unknown.
