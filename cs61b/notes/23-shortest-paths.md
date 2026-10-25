<!-- Mon, Oct 19, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 23: Shortest Paths

This lecture extends the graph traversal work from Lecture 22 (DFS and BFS) to **edge-weighted graphs**, where each edge carries a cost (miles, meters, travel time). BFS finds the path with the *fewest edges*, which is not the same as the path with the *smallest total weight*, so BFS is the wrong tool for a Google Maps style application. The lecture first builds intuition for the goal object, the **Shortest Paths Tree (SPT)**, showing that the set of shortest paths from a single source always forms a tree with exactly `V - 1` edges. It then introduces the improvement operation that shortest-path algorithms are built from, **edge relaxation** (and its generalization, **node relaxation**), and uses it to construct **Dijkstra's algorithm**: visit vertices in "closest first" order using a priority queue as the fringe, relaxing each vertex's outgoing edges exactly once, for `O(E log V)` time on a connected graph. The lecture closes with **A\***, explicitly marked as extra and out of scope, which adds a heuristic estimate `h(v, goal)` to each vertex's priority so that a single-target search can avoid exploring the whole graph.

---

## Key Concepts

### 1. Why BFS is not enough

Lecture 22 established two path-finding tools:

| Problem | Solution | Efficiency (adj. list) |
|---|---|---|
| s-t paths: find a path from `s` to every reachable vertex | `DepthFirstPaths.java` | `O(V+E)` time, `Θ(V)` space |
| s-t shortest paths: find a shortest path (fewest edges) from `s` to every reachable vertex | `BreadthFirstPaths.java` | `O(V+E)` time, `Θ(V)` space |

Comparing them:

- **Correctness**: both work on all graphs.
- **Output quality**: BFS is a "2-for-1 deal", you get paths *and* those paths have the fewest edges.
- **Time efficiency**: essentially the same, both consider every edge twice. Distinguishing them needs experiments or very careful constant-factor analysis, which big-O hides.
- **Space efficiency**: DFS is worse on *spindly* graphs, because the call stack gets very deep and the computer needs `Θ(V)` memory to remember the recursive calls (covered more in CS 61C). BFS is worse on absurdly *bushy* graphs: with 1,000,000 vertices all connected to one vertex, 999,999 get enqueued at once, so the queue takes `Θ(V)` memory. Note that in our implementations we spend `Θ(V)` anyway for the `distTo` and `edgeTo` arrays (you could optimize by using a map instead of an array).

The real problem with BFS appears with the Hearst Mining Memorial Building / fountain example from lecture. Represent walkways as edges and intersections as vertices. BFS finds `s -> t` in 2 edge hops, but those 2 hops are a 260 m edge plus a 70 m edge, giving **~330 m**. The route a human would actually walk (straight past the fountain) takes more edges but totals only **~130 m**.

> BFS returns the path with the shortest *number of edges*, not necessarily the shortest path. We need an algorithm that accounts for edge distances, also known as **edge weights**.

**The naive fix and why it fails.** A student in lecture proposed subdividing: replace a weight-260 edge with 260 unit-weight edges and unit vertices, then run plain BFS. This *works*, but it is terrible: it blows up `V` and `E` enormously (what if a weight were a million?), and since BFS is `O(V + E)`, the runtime explodes with it. We want to treat weights as *numbers*, not as graph structure.

### 2. The warm-up: greedy is not automatically correct

The opening example (towns A through G, edge weights in miles):

```
                          D
                  11    /   \ 1
          B ----------      \
       2/   \3         2      G
  A ---      --- E ----------/
    \1       5/    \4   5
     C --------------  F
            15
```
(Reconstructed from the slides: edges `A-B` = 2, `A-C` = 1, `B-C` = 5, `B-D` = 11, `B-E` = 3, `C-F` = 15, `E-D` = 2, `E-F` = 4, `E-G` = 5, `G-D` = 1, `F-G` = 1.)

Finding the best path from A to F: the **greedy** first move is `A -> C` (cost 1, cheaper than `A -> B` at cost 2). But `C`'s only way onward is a 15-mile mountain crossing, so `A -> C -> F` costs **16**. The actual best path is `A -> B -> E -> F` = 2 + 3 + 4 = **9 miles**, even though it visits more towns. This is a counterexample proving that a purely greedy "take the cheapest next edge" strategy does not always work.

### 3. The optimal substructure insight

The instructor's key warm-up question: suppose you are told the shortest path from A to G goes through F. Do you already know what that path is?

Yes, *provided you already know the best path to F*. Why would you ever use a worse route to F and then continue to G? Any path to G through F decomposes into (path to F) + (F to G), and you minimize the total by minimizing the first part. Symmetrically, if there were a shorter path to E than `A -> B -> E`, then gluing `E -> F` onto it would give a shorter path to F, contradicting what we already computed.

This is why the solution has such clean structure. You might fear that the shortest paths to all seven vertices would be a tangled mess of overlapping routes needing the graph redrawn over and over. Instead, shortest paths to different targets *share prefixes*, and the whole solution collapses into a tree.

### 4. Single Source Single Target vs. Single Source (all targets)

**Single source, single target** (A to F only):

| v | distTo[] | edgeTo[] |
|---|---|---|
| A | 0.0 | - |
| B | 2.0 | A→B |
| C | - | - |
| D | - | - |
| E | 5.0 | B→E |
| F | 9.0 | E→F |
| G | - | - |

Observation: assuming non-negative weights, the solution is always a path **with no cycles**. (A cycle of non-negative weight can only add cost, so removing it never hurts.)

**Single source, all targets** (A to everything):

| v | distTo[] | edgeTo[] |
|---|---|---|
| A | 0.0 | - |
| B | 2.0 | A→B |
| C | 1.0 | A→B *(as printed on the slide; the edge is actually A→C, see Pitfalls)* |
| D | 11.0 | G→D |
| E | 5.0 | B→E |
| F | 9.0 | E→F |
| G | 10.0 | E→G |

Observation: the solution is always a **tree**, the union of the shortest paths to all vertices.

Reading the table: `distTo[D] = 11` with `edgeTo[D] = G→D` means the shortest route to D is "shortest route to G, then the edge G→D", i.e. `A -> B -> E -> G -> D` = 2 + 3 + 5 + 1 = 11. Note that the direct `B -> D` edge of weight 11 would give 2 + 11 = 13, worse.

### 5. SPT edge count

> If `G` is a connected edge-weighted graph with `V` vertices and `E` edges, how many edges are in the SPT (assuming every vertex is reachable)?

**Always `V - 1`.** For each vertex there is exactly one incoming edge (except the source, which has none). That is exactly what `edgeTo[]` records: one entry per vertex, one of which is empty. In the 7-vertex example, the SPT has 6 edges. Note `E` does not appear in the answer at all.

### 6. Edge relaxation

Many shortest-path algorithms share one approach:

- Start from an **incorrect** SPT.
- Make improvements until the SPT is correct.

The common improvement operation is **edge relaxation**: `relax(E)` means "if edge `E` is better, use it instead."

Lecture example. Graph: `A->B` weight 1, `B->C` weight 5, `A->D` weight 100, `C->D` weight 3. Starting (wrong) SPT:

| v | distTo[] | edgeTo[] |
|---|---|---|
| A | 0.0 | - |
| B | 1.0 | A→B |
| C | 6.0 | B→C |
| D | 100.0 | A→D |

Relax the magenta edge `C->D`: the candidate distance is `distTo[C] + weight(C,D) = 6 + 3 = 9`, which beats the current `distTo[D] = 100`. So we update:

| v | distTo[] | edgeTo[] |
|---|---|---|
| A | 0.0 | - |
| B | 1.0 | A→B |
| C | 6.0 | B→C |
| D | **9.0** | **C→D** |

This SPT is now correct.

### 7. Node relaxation

**`relax(v)` means: relax all of `v`'s outgoing edges.**

Lecture example (relaxing node E). Before:

| v | distTo[] | edgeTo[] |
|---|---|---|
| A | 0.0 | - |
| B | 8.0 | A→B |
| C | 13.0 | B→C |
| D | 40.0 | B→D |
| E | 16.0 | C→E |
| F | 3.0 | A→F |
| G | ∞ | - |

E has outgoing edges `E->D` of weight 2 and `E->G` of weight 2. Check each:

- `E->D`: `16 + 2 = 18 < 40`, so improve. `distTo[D] = 18`, `edgeTo[D] = E→D`.
- `E->G`: `16 + 2 = 18 < ∞`, so improve. `distTo[G] = 18`, `edgeTo[G] = E→G`.

After:

| v | distTo[] | edgeTo[] |
|---|---|---|
| A | 0.0 | - |
| B | 8.0 | A→B |
| C | 13.0 | B→C |
| D | **18.0** | **E→D** |
| E | 16.0 | C→E |
| F | 3.0 | A→F |
| G | **18.0** | **E→G** |

Node relaxation is the "expand the band outward from one vertex" step that Dijkstra's repeats.

### 8. The family of SPT algorithms

All of these use relaxation; they differ in the *order* of relaxations:

- **Bellman-Ford (1955)**: relax all edges (in any order), repeat `V` times. Slower than Dijkstra's but handles negative edge weights.
- **Dijkstra's Algorithm (1956)**: relax all nodes exactly once, in **closest first** order. Does *not* work with negative edge weights.
- **The Duan-Mao et al. Algorithm (2025)**: relax edges in batches. A recent breakthrough that beats Dijkstra's, which had long been thought optimal.

Only Dijkstra's is covered in this course.

### 9. Dijkstra's Algorithm: closest first order

We have seen Depth First order (stack / recursion) and Breadth First order (queue). Dijkstra's introduces a third: **Closest First**, visiting vertices in increasing order of their distance from the source, building the SPT as you go.

The instructor's intuition: imagine growing a circle (a "band") outward from A, and ask "what is the next-nearest thing I can reach?" From A, that is C at distance 1 (even though B is also a direct neighbor). Then B at 2. Then, crucially, the next thing is **not** necessarily a neighbor of A: it is E at distance 5, reached as `A -> B -> E` = 2 + 3. Then F at 9 (`A->B->E->F`, beating `A->C->F` at 16), then G at 10, then D at 11. We expand to the nearest unreached vertex *over the whole current frontier*, not just over neighbors of the source.

**The fringe data structure.** For Breadth First order the fringe was a Queue. For Closest First order, the fringe is a **priority queue**: it lets us always pull out whatever currently has the highest priority (smallest known distance), rather than simply first-come-first-served.

Pseudocode from the slides:

```
add all vertices to PQ with infinite priority
while (!pq.isEmpty()) {
    v = pq.removeMin();
    relax(v);  // i.e. relax all edges from v
}
```

The source is given priority 0 so it comes out first. Note this requires a PQ operation we had not previously put in our priority-queue ADT: **changePriority** (also called decrease-key), used when relaxing an edge improves a fringe vertex's distance.

### 10. The invariant that makes Dijkstra's correct

> In Dijkstra's algorithm, relaxing an edge that points to an already-visited (white) node **NEVER** works.

Why, by example from the demo: we are relaxing E's edges and one of them points to C. But we already visited C before E. That means C is *closer* to the source than E is. So it is impossible for a path going through E to reach C more cheaply. **This invariant is why Dijkstra's yields the correct SPT.**

The slide states the general version: with non-negative weights, it is impossible for any inactive vertex (white, already removed, not on the fringe) to be improved. The lecturer's phrasing: "if it's white, we've already expanded to that, we're already beyond the boundary, so it's not a shortcut if we're already there."

This is exactly where **non-negative weights** matter. If a negative edge existed, a later, farther vertex could lead back to an already-finalized vertex at lower cost, and the invariant collapses.

### 11. Runtime analysis

Assuming a binary-heap-based PQ, where every operation costs `O(log V)`:

| | # Operations | Cost per operation | Total cost |
|---|---|---|---|
| PQ add | `V` | `O(log V)` | `O(V log V)` |
| PQ removeSmallest | `V` | `O(log V)` | `O(V log V)` |
| PQ changePriority | `E` | `O(log V)` | `O(E log V)` |

Overall: `O(V log V + V log V + E log V)`. For a connected graph, `E >= V - 1`, so the `E log V` term dominates and we write simply:

> **Dijkstra's: `O(E log V)` time, `Θ(V)` space.**

Each vertex is added once and removed once (hence `V` of each). Each *edge* can trigger at most one `changePriority`, hence `E` of those.

### 12. A\* (EXTRA: explicitly marked out of scope in the slides)

**The problem with Dijkstra's for a single target.** Dijkstra's expands in all directions equally. Searching for a route from Denver to NYC, it will explore every place within nearly two thousand miles of Denver before it even locates NYC. When we have a single target, that is enormously wasteful: we want to explore eastwards first.

**The A\* idea.** Visit vertices in order of `d(source, v) + h(v, goal)`, where `h(v, goal)` is an *estimate* of the remaining distance from `v` to the goal. Compare with Dijkstra's, which only considers `d(source, v)`. In words: look at a location `v` if we already know the fastest way to reach `v`, **and** we suspect `v` is also on the fastest way to NYC accounting for the time to get to `v`. Henderson may be farther from Denver than Englewood, but it is probably better overall for getting to NYC.

**The heuristic.** `h` is arbitrary and does not have to be perfect ("heuristic: using experience to learn and improve"). For maps, a natural choice is as-the-crow-flies distance:

```java
/** h(v, goal) DOES NOT CHANGE as algorithm runs. */
public method h(v, goal) {
    return computeLineDistance(v.latLong, goal.latLong);
}
```

Note the comment: `h` is a fixed function of the vertex and the goal; it is not updated as the algorithm runs.

**The A\* demo** used the same 7-vertex graph with goal = G and `h` set to the minimum-weight edge out of each vertex (`h(A)=1, h(B)=3, h(C)=15, h(D)=2, h(E)=1, h(F)=∞, h(G)=0`). Key observations when it terminated (which it did as soon as the goal reached the front of the fringe):

- **Not every vertex got visited** (that is the whole point).
- The result is **not a shortest paths tree** for A. The recorded path to D was suboptimal (13 via B, not 11 via G). That is fine, because we only care about the path to G.

**Impact of heuristic quality:**

- `h(v, goal) = 0` for all `v`: we just get Dijkstra's algorithm.
- `h(v, goal) = 10000` for all `v`: we *also* just get Dijkstra's algorithm (a constant added to every priority does not change the ordering).
- `h(Chicago, goal) = h(Indianapolis, goal) = 100000` because "the midwest is in the middle of nowhere": the algorithm becomes **incorrect**. It will fail to find the shortest path by dodging Illinois. It does not merely run slower, it returns a wrong answer.

**(EXTRA, beyond course scope, per the slides.)** For A\* to be correct, the heuristic must be:

- **Admissible**: `h(v, NYC) <= ` true distance from `v` to NYC (never overestimates).
- **Consistent**: for each neighbor `w` of `v`, `h(v, NYC) <= dist(v, w) + h(w, NYC)`, where `dist(v, w)` is the weight of edge `v -> w`.

All consistent heuristics are admissible. If the heuristic is admissible, A\* *tree search* yields the shortest path; if consistent, A\* *graph search* (our version) yields the shortest path. These are sufficient but not necessary conditions. The slide is explicit: "You will not be expected to tell us whether a given heuristic is admissible or consistent unless we define these terms on an exam." The midwest heuristic above was both inadmissible and inconsistent. This topic belongs to CS 188.

**Visualization tool:** `http://qiao.github.io/PathFinding.js/visual/`. Note that if all edge weights are equal, Dijkstra's is just breadth first search. The tool is good for seeing the distinction between the order in which nodes are *visited* and the order in which they appear *on the shortest path*; unless you are really lucky, vastly more nodes are visited than lie on the shortest path.

### 13. The unifying idea

The lecturer's closing point: do not think of data structures and algorithms as separate subjects. DFS, BFS, and Dijkstra's look like three distinct algorithms, but they are all doing the same thing: pushing out a frontier. The only difference is the fringe data structure.

| Algorithm | Fringe |
|---|---|
| DFS | Stack (often implicit, via the recursive call stack) |
| BFS | Queue |
| Dijkstra's | Priority Queue |
| A\* (extra) | Priority Queue, with `h(v, goal)` added to each priority |

---

## Definitions

- **Edge-weighted graph**: a graph in which each edge carries a numeric weight (length, cost, travel time). Contrast with the unweighted graphs of Lecture 22, where every edge implicitly costs 1.
- **Edge weight**: the number attached to an edge, representing the distance or cost of traversing it.
- **Shortest path (weighted)**: a path from `s` to `t` minimizing the *sum of the edge weights* along it, not the number of edges.
- **Single Source Single Target Shortest Paths**: find a shortest path from source vertex `s` to one specific target vertex `t`. With non-negative weights the answer is always a path with no cycles.
- **Single Source Shortest Paths**: find shortest paths from source `s` to *every* other vertex. The answer is always a tree.
- **Shortest Paths Tree (SPT)**: the union of the shortest paths from a single source to all reachable vertices. For a connected graph with all vertices reachable, it has exactly `V - 1` edges, because every vertex except the source has exactly one incoming edge.
- **`distTo[v]`**: the best known total weight of a path from the source to `v`. Initialized to `∞` for every vertex except the source, which is `0`.
- **`edgeTo[v]`**: the last edge on the best known path to `v`, i.e. the edge that entered `v`. Together, the `edgeTo` entries encode the SPT.
- **Edge relaxation, `relax(E)`**: "if edge `E` is better, use it instead." Concretely, for edge `v -> w` of weight `d`, if `distTo[v] + d < distTo[w]`, set `distTo[w] = distTo[v] + d` and `edgeTo[w] = v -> w`.
- **Node relaxation, `relax(v)`**: relax all of `v`'s outgoing edges.
- **Fringe**: the collection of discovered-but-not-yet-visited vertices awaiting processing. A stack for DFS, a queue for BFS, a priority queue for Dijkstra's.
- **Closest First order**: visiting vertices in increasing order of their (final) distance from the source. The visitation order Dijkstra's achieves.
- **Dijkstra's Algorithm (1956)**: relax all nodes exactly once in closest first order, using a priority queue as the fringe. `O(E log V)` time, `Θ(V)` space. Requires non-negative edge weights.
- **Bellman-Ford (1955)**: relax all edges in any order, repeat `V` times. Slower than Dijkstra's but works with negative edge weights.
- **`changePriority` (decrease-key)**: priority queue operation that changes an item's priority in place. Needed by Dijkstra's when a relaxation improves a fringe vertex's distance.
- **Greedy algorithm**: an algorithm that takes the best option immediately in front of it. Sometimes provably sufficient, sometimes wrong (as in the A→C→F counterexample).
- **Heuristic** *(A\*, extra)*: `h(v, goal)`, an arbitrary estimate of the remaining distance from `v` to the goal. Fixed for the duration of the algorithm.
- **A\*** *(extra, out of scope)*: like Dijkstra's, but vertices are prioritized by `d(source, v) + h(v, goal)`. Aimed at the single-target problem.
- **Admissible heuristic** *(extra, beyond scope)*: `h(v, goal) <=` the true distance from `v` to the goal; never overestimates.
- **Consistent heuristic** *(extra, beyond scope)*: for each neighbor `w` of `v`, `h(v, goal) <= dist(v, w) + h(w, goal)`. All consistent heuristics are admissible.

---

## Worked Examples

### Example 1: `relax` as Java code

The slides express relaxation in words and diagrams. Here is the corresponding code, which is the single most important thing to be able to write from memory.

```java
/**
 * Edge relaxation. Given the edge v -> w with weight d, improve our best
 * known route to w if going "through v" is cheaper than what we have.
 */
private void relaxEdge(int v, int w, double d) {
    if (distTo[v] + d < distTo[w]) {   // is this edge better?
        distTo[w] = distTo[v] + d;     // record the new, smaller distance
        edgeTo[w] = v;                 // record that we arrived at w from v
        if (fringe.contains(w)) {
            fringe.changePriority(w, distTo[w]);  // move w up the line
        }
    }
    // otherwise: do nothing at all. Relaxation never makes distTo larger.
}

/** Node relaxation: relax all of v's outgoing edges. */
private void relaxNode(int v) {
    for (Edge e : graph.adj(v)) {
        relaxEdge(v, e.to(), e.weight());
    }
}
```

Step by step on the lecture's edge-relaxation example (`distTo[C] = 6`, `distTo[D] = 100`, edge `C -> D` of weight 3):

1. `relaxEdge(C, D, 3)` evaluates the guard: `distTo[C] + 3 = 6 + 3 = 9`.
2. Compare with `distTo[D] = 100`. Since `9 < 100`, the edge **is** better.
3. Write `distTo[D] = 9`. The old value 100 is gone.
4. Write `edgeTo[D] = C`. Think of `edgeTo` as a box-and-pointer structure: `edgeTo[D]` used to point back at `A`, and we have just redirected that arrow to point at `C`. The SPT changed shape: D's parent switched from A to C. No new objects were allocated; we just repointed one reference.
5. `distTo[D]` is now consistent with `distTo[C]`, so the tree is correct.

Why the guard is `<` and not `<=`: using `<` means ties leave the existing edge in place. This answers the student's question from lecture ("what if there are two shortest paths to the same vertex?"). You may break ties arbitrarily; `<` just happens to break them in favor of whichever edge was found first.

### Example 2: Dijkstra's, in code

Faithful to the lecture pseudocode (`add all vertices to PQ with infinite priority; while not empty, removeMin and relax`):

```java
public class DijkstrasSP {
    private double[] distTo;
    private int[] edgeTo;
    private PriorityQueue<Integer> fringe;  // min-PQ keyed on distTo

    public DijkstrasSP(EdgeWeightedGraph G, int s) {
        distTo = new double[G.V()];
        edgeTo = new int[G.V()];

        // Everything starts at infinity; then we can only improve.
        for (int v = 0; v < G.V(); v += 1) {
            distTo[v] = Double.POSITIVE_INFINITY;
            edgeTo[v] = -1;
        }
        distTo[s] = 0.0;

        // "add all vertices to PQ with infinite priority"
        fringe = new PriorityQueue<>();
        for (int v = 0; v < G.V(); v += 1) {
            fringe.add(v, distTo[v]);   // s enters with priority 0
        }

        while (!fringe.isEmpty()) {
            int v = fringe.removeMin();  // closest remaining vertex
            relaxNode(v);                // relax all edges out of v
        }
    }
}
```

Three details worth internalizing:

- **Why initialize to infinity?** As the lecturer put it: we set everything infinitely far away, then the algorithm only ever improves. Infinity is the identity element for "take the minimum", so the first real path discovered always wins the comparison in `relaxEdge`.
- **Why does the source come out first?** It is the only vertex with finite (zero) priority, so `removeMin` returns it immediately.
- **Why is each vertex relaxed exactly once?** Because `removeMin` removes it from the fringe permanently. Once removed, it is "white" and the invariant guarantees it can never be improved.

### Example 3: Full trace of Dijkstra's on the lecture graph

Source `s = A`. Edges: `A-B`(2), `A-C`(1), `B-C`(5), `B-D`(11), `B-E`(3), `C-F`(15), `E-D`(2), `E-F`(4), `E-G`(5), `F-G`(1), `G-D`(1).

**Initialization.** All `distTo` are `∞` except `distTo[A] = 0`; all `edgeTo` empty.

```
Fringe: [(A: 0), (B: ∞), (C: ∞), (D: ∞), (E: ∞), (F: ∞), (G: ∞)]
```

**Step 1: remove A (dist 0), relax A's edges.**
- `A->C`, weight 1: `0 + 1 = 1 < ∞`. Set `distTo[C] = 1`, `edgeTo[C] = A`.
- `A->B`, weight 2: `0 + 2 = 2 < ∞`. Set `distTo[B] = 2`, `edgeTo[B] = A`.

| Node | distTo | edgeTo |
|---|---|---|
| A | 0 | - |
| B | 2 | A |
| C | 1 | A |
| D, E, F, G | ∞ | - |

```
Fringe: [(C: 1), (B: 2), (D: ∞), (E: ∞), (F: ∞), (G: ∞)]
```

**Step 2: remove C (dist 1), relax C's edges.**
- `C->F`, weight 15: `1 + 15 = 16 < ∞`. Set `distTo[F] = 16`, `edgeTo[F] = C`.
- Note the candidate is **16, not 15**: it is `distTo[C] + weight`, not just the weight.

```
Fringe: [(B: 2), (F: 16), (D: ∞), (E: ∞), (G: ∞)]
```

**Step 3: remove B (dist 2), relax B's edges.**
- `B->E`, weight 3: `2 + 3 = 5 < ∞`. Set `distTo[E] = 5`, `edgeTo[E] = B`.
- `B->D`, weight 11: `2 + 11 = 13 < ∞`. Set `distTo[D] = 13`, `edgeTo[D] = B`.
- `B->C`, weight 5: `2 + 5 = 7`, but `distTo[C] = 1`. **Vertex C unchanged since 2 + 5 > 1.** C is already white (visited), so this relaxation was guaranteed to fail.

| Node | distTo | edgeTo |
|---|---|---|
| A | 0 | - |
| B | 2 | A |
| C | 1 | A |
| D | 13 | B |
| E | 5 | B |
| F | 16 | C |
| G | ∞ | - |

```
Fringe: [(E: 5), (D: 13), (F: 16), (G: ∞)]
```

**Step 4: remove E (dist 5), relax E's edges.**
- `E->F`, weight 4: `5 + 4 = 9 < 16`. Improve! `distTo[F] = 9`, `edgeTo[F] = E`. This is the moment the mountain-crossing route is abandoned.
- `E->G`, weight 5: `5 + 5 = 10 < ∞`. Set `distTo[G] = 10`, `edgeTo[G] = E`.
- `E->D`, weight 2: `5 + 2 = 7 < 13`. Improve to `distTo[D] = 7`, `edgeTo[D] = E`.
- `E->C`: **unchanged since 5 + 1 > 1.**

*(The slides' own trace keeps D at 13 at this point and later updates it to 11 via G. Following the slide's numbers exactly, D is 13 here; the slide's final answer is `distTo[D] = 11, edgeTo[D] = G`. Treat the slide table as authoritative for the final answer; the point of the step is the mechanics of relaxation. (extra context): with the edge weights as drawn, the `E->D` edge of weight 2 would in fact give D a distance of 7, so the printed trace and the printed final table are not fully consistent with each other. Exam questions will give you an unambiguous graph.)*

```
Fringe: [(F: 9), (G: 10), (D: 13)]
```

**Step 5: remove F (dist 9), relax F's edges.** `F->G`, weight 1: `9 + 1 = 10`, not `< 10`, so no change (a tie; leave it alone).

```
Fringe: [(G: 10), (D: 13)]
```

**Step 6: remove G (dist 10), relax G's edges.** `G->D`, weight 1: `10 + 1 = 11 < 13`. Improve! `distTo[D] = 11`, `edgeTo[D] = G`.

```
Fringe: [(D: 11)]
```

**Step 7: remove D (dist 11), relax D's edges.** `D->E`, weight 2: `11 + 2 = 13`, but `distTo[E] = 5`. **Vertex E unchanged since 11 + 2 > 5.** As the slide notes: with non-negative weights, it is impossible for any inactive vertex to be improved.

```
Fringe: []     <-- done
```

**Final result:**

| v | distTo[] | edgeTo[] |
|---|---|---|
| A | 0.0 | - |
| B | 2.0 | A→B |
| C | 1.0 | A→C |
| D | 11.0 | G→D |
| E | 5.0 | B→E |
| F | 9.0 | E→F |
| G | 10.0 | E→G |

Six edges for seven vertices, exactly `V - 1`, as predicted.

**Reading a path out of `edgeTo`.** To recover the path to D, walk the `edgeTo` pointers backwards: `D <- G <- E <- B <- A`, then reverse: `A -> B -> E -> G -> D`. Verify: 2 + 3 + 5 + 1 = 11. This is the same backtracking you did for BFS in Lecture 22; only the contents of the arrays differ.

### Example 4: A\* trace (extra, out of scope)

Same graph, goal = G, with `h(A)=1, h(B)=3, h(C)=15, h(D)=2, h(E)=1, h(F)=∞, h(G)=0`.

Priorities become `distTo[v] + h(v, G)`:

- After relaxing A: `distTo[B] = 2` so B's priority is `2 + 3 = 5`; `distTo[C] = 1` so C's priority is `1 + 15 = 16`. Fringe: `[(B: 5), (C: 16), ...]`. **B comes out before C**, the opposite of Dijkstra's, because C's enormous heuristic marks it as a dead end.
- Remove B, relax: `distTo[E] = 5` (priority `5 + 1 = 6`), `distTo[D] = 13` (priority `13 + 2 = 15`). Fringe: `[(E: 6), (D: 15), (C: 16), (F: ∞)]`.
- Remove E, relax: `distTo[F] = 9`, `distTo[G] = 10` (priority `10 + 0 = 10`). Fringe: `[(G: 10), (D: 15), (C: 16), (F: ∞)]`.
- The next vertex to be dequeued is the target G, **so we are done**.

Observations: C and F were never visited; `distTo[D] = 13` is suboptimal (the true answer is 11), which is acceptable because only the path to G was ever requested.

---

## Common Pitfalls

1. **Thinking BFS gives weighted shortest paths.** BFS minimizes the *number of edges*. Those are the same only when every edge has equal weight. (Correspondingly: Dijkstra's on an equal-weight graph degenerates to BFS.)

2. **Relaxing with the edge weight instead of `distTo[v] + weight`.** In Step 2 above, F's new distance is 16, not 15. The candidate is always "distance to the tail, plus the edge."

3. **Thinking the SPT edge count depends on `E`.** It is always `V - 1` for a connected graph with all vertices reachable, regardless of how many edges the graph has.

4. **Confusing the SPT with the MST.** The lecturer caught himself saying "minimal spanning tree" and corrected to "shortest path tree." They are different objects, built by different algorithms, optimizing different things. The SPT depends on the source; the MST does not. (extra context: MSTs are the next lecture's topic.)

5. **Confusing visitation order with path order.** Dijkstra's visits vastly more vertices than appear on any one shortest path. Which vertex you pop next is not the same as which vertex lies on the answer.

6. **Applying Dijkstra's to graphs with negative edge weights.** The "already-visited vertices can never be improved" invariant depends on non-negativity. With a negative edge, Dijkstra's can finalize a vertex too early and return a wrong answer. Use Bellman-Ford instead.

7. **Forgetting `changePriority`.** The pseudocode inserts every vertex up front at infinite priority and then *updates* priorities during relaxation. If your PQ cannot decrease a key, your runtime and your implementation both change. This operation is the source of the `E log V` term.

8. **Re-relaxing a vertex after it is removed.** Each vertex is relaxed *exactly once*, when it is popped. Relaxing it again is wasted work (and on exam traces, leads to wrong tables).

9. **Trying to make BFS work by subdividing weighted edges into unit edges.** It is correct but catastrophically wasteful in both time and memory, since the runtime is `O(V + E)` and you have just made `V` and `E` enormous.

10. **The slide's `edgeTo[C] = A→B` typo.** On the single-source table, the lecture slide lists `edgeTo[C]` as `A→B` with `distTo[C] = 1.0`. The actual edge is `A→C` (weight 1). Do not be thrown by this; the slides carry a banner warning that they have not yet been updated for Fall 2026.

11. **Assuming a bad A\* heuristic merely slows things down.** The midwest example shows that an inadmissible heuristic can make A\* return the **wrong** answer, not just a slower correct one.

12. **Expecting A\* to produce an SPT.** It does not. It produces one path to one goal, and the other `distTo` entries may be suboptimal or never filled in.

---

## Likely Exam Points

### 1. Trace Dijkstra's and fill in `distTo` / `edgeTo` / the fringe

The single most heavily tested item. Expect to be asked for the state after a specific number of `removeMin` operations.

**Q.** Using the lecture graph (`A-B`=2, `A-C`=1, `B-C`=5, `B-D`=11, `B-E`=3, `C-F`=15, `E-F`=4, `E-G`=5, `F-G`=1, `G-D`=1), run Dijkstra's from A. After removing exactly three vertices from the fringe, what is `distTo[F]` and what is in the fringe?

**A.** The three removed are A (0), C (1), B (2). `distTo[F] = 16` with `edgeTo[F] = C`, since only `A->C->F` has been discovered so far. The fringe is `[(E: 5), (D: 13), (F: 16), (G: ∞)]`.

### 2. SPT edge count

**Q.** A connected edge-weighted graph has `V = 100` vertices and `E = 4000` edges, with every vertex reachable from the source. How many edges are in the SPT?

**A.** 99, i.e. `V - 1`. Every vertex except the source has exactly one incoming edge in the tree. `E` is irrelevant.

### 3. Dijkstra's runtime, including where each term comes from

**Q.** State Dijkstra's runtime with a binary-heap PQ on a connected graph, and justify each term.

**A.** `O(E log V)`. There are `V` `add` operations and `V` `removeSmallest` operations, each `O(log V)`, giving `O(V log V)` twice; and at most `E` `changePriority` operations at `O(log V)` each, giving `O(E log V)`. Since `E >= V - 1` for a connected graph, the `E log V` term dominates. Space is `Θ(V)` for `distTo` and `edgeTo`.

### 4. Why relaxing into an already-visited vertex always fails

**Q.** During Dijkstra's from A, we pop E (with `distTo[E] = 5`) and consider the edge `E -> C` of weight 1, where C was popped earlier with `distTo[C] = 1`. Without computing anything, explain why this relaxation cannot succeed.

**A.** C was removed from the fringe before E, so C's final distance is at most E's: `distTo[C] <= distTo[E]`. Since edge weights are non-negative, any route through E to C costs `distTo[E] + w(E,C) >= distTo[E] >= distTo[C]`. So the candidate can never be strictly smaller. (Arithmetic check: `5 + 1 = 6 > 1`.) This invariant is the correctness argument for Dijkstra's.

### 5. Why Dijkstra's needs non-negative weights

**Q.** Does Dijkstra's always work on graphs with negative edge weights? What should you use instead?

**A.** No. The correctness invariant ("a vertex already removed from the fringe can never be improved") relies on non-negative weights; a negative edge can make a later-discovered vertex offer a cheaper route back to an already-finalized vertex. Use Bellman-Ford (relax all edges, repeat `V` times), which is slower but handles negative weights.

### 6. Edge/node relaxation mechanics

**Q.** `distTo[B] = 8`, `distTo[D] = 40` with `edgeTo[D] = B`, `distTo[E] = 16`, and there is an edge `E -> D` of weight 2. What happens when we relax node E?

**A.** We check all of E's outgoing edges. For `E -> D`: `distTo[E] + 2 = 18 < 40`, so we improve, setting `distTo[D] = 18` and `edgeTo[D] = E→D`. (Each of E's other outgoing edges is tested the same way; this is exactly the lecture's node-relaxation example.)

### 7. BFS vs. Dijkstra's vs. weighted shortest paths

**Q.** Give a graph where BFS from `s` returns a path to `t` that is not a weighted shortest path, and explain why.

**A.** The lecture's Hearst Mining example: a 2-edge route of 260 m + 70 m = 330 m versus a many-edge route totaling ~130 m. BFS minimizes edge count, which ignores weights entirely. Any graph where a short hop-count route uses expensive edges works.

### 8. Reconstructing a path from `edgeTo`

**Q.** Given `edgeTo[D] = G`, `edgeTo[G] = E`, `edgeTo[E] = B`, `edgeTo[B] = A`, `edgeTo[A] = -`, what is the shortest path from A to D?

**A.** Walk backwards from D: `D, G, E, B, A`, then reverse: `A -> B -> E -> G -> D`. Its length is `distTo[D]`.

### 9. Fringe data structures across traversals

**Q.** Fill in the fringe data structure for DFS, BFS, and Dijkstra's.

**A.** DFS: stack (usually implicit via recursion on the call stack). BFS: queue. Dijkstra's: priority queue. All three are the same frontier-expansion algorithm with a different fringe.

### 10. A\* heuristic behavior (extra, flagged out of scope; include only if the exam defines the terms)

**Q.** What happens to A\* if `h(v, goal) = 0` for every `v`? What if `h(v, goal) = 10000` for every `v`?

**A.** Both reduce to Dijkstra's algorithm. With `h = 0` the priority is just `distTo`. With a constant `h`, every priority is shifted by the same amount, so the relative ordering, and therefore the visitation order, is unchanged.

---

## Summary

- **BFS finds fewest-edge paths, not shortest-weight paths.** On an edge-weighted graph (Google Maps style) that gives wrong routes, e.g. 330 m instead of 130 m. Subdividing edges into unit edges "works" but blows up `V` and `E`.
- **Single source, single target**: with non-negative weights, the answer is always a simple path (no cycles).
- **Single source, all targets**: the answer is always a **Shortest Paths Tree**, with exactly **`V - 1` edges** (one incoming edge per vertex, none for the source). Shortest paths share prefixes, which is why the solution is a tree and not a tangle.
- **Represent the answer** with two arrays: `distTo[v]` (best known total weight) and `edgeTo[v]` (last edge into `v`).
- **Edge relaxation** `relax(E)`: if `distTo[v] + weight < distTo[w]`, update `distTo[w]` and `edgeTo[w]`. **Node relaxation** `relax(v)`: relax all of `v`'s outgoing edges.
- **General recipe**: start from an incorrect SPT, relax until correct. Bellman-Ford (1955) relaxes all edges `V` times, handles negative weights. Dijkstra's (1956) relaxes each node once in closest-first order. Duan-Mao et al. (2025) relaxes in batches.
- **Dijkstra's**: add all vertices to a PQ with infinite priority (source at 0); repeatedly `removeMin` and relax that node. The PQ is the fringe that produces **Closest First** order.
- **Correctness invariant**: with non-negative weights, relaxing into an already-removed (white) vertex can never succeed, because that vertex was already closer. This fails with negative weights, which is why Dijkstra's requires non-negative weights.
- **Runtime**: `V` adds + `V` removeMins + `E` changePrioritys, each `O(log V)` with a binary heap, giving `O(E log V)` on a connected graph, `Θ(V)` space.
- **DFS / BFS / Dijkstra's are the same frontier algorithm** with a stack / queue / priority queue respectively.
- **A\* (extra, out of scope)**: prioritize by `distTo[v] + h(v, goal)` to attack the single-target problem. Visits far fewer vertices, does not produce a valid SPT, and a bad heuristic makes it *wrong*, not just slow. Constant or zero heuristics reduce it to Dijkstra's. Admissibility and consistency (CS 188 topics) are sufficient conditions for correctness.

| Problem | Solution | Efficiency |
|---|---|---|
| paths | `DepthFirstPaths.java` | `O(V+E)` time, `Θ(V)` space |
| shortest paths (fewest edges) | `BreadthFirstPaths.java` | `O(V+E)` time, `Θ(V)` space |
| shortest weighted paths, all targets | `DijkstrasSP.java` | `O(E log V)` time, `Θ(V)` space |
| shortest weighted path, single target *(out of scope)* | A\*: Dijkstra's with `h(v, goal)` added to each priority | depends on heuristic; `Θ(V)` space |
