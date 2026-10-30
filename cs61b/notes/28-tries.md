<!-- Fri, Oct 30, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 28: Tries

Tries are a set/map implementation specialized for **String** keys. The lecture begins from a broader theme: an abstract data type (ADT) like Set, Map, List, Priority Queue, or Disjoint Sets can be implemented many different ways, and sometimes you can beat the general-purpose implementations by exploiting known properties of your keys. If keys are single ASCII characters, you can just index into an array (a `DataIndexedCharMap`) for Θ(1) `get` and `put`. Tries generalize that idea to strings: store one character per node, share nodes between keys with common prefixes, and mark which nodes complete a real key. The result is `add` in Θ(L) and `contains` in O(L) where L is the key length, which is independent of the number of keys N. The real payoff, though, is not the asymptotic win on `get`/`add` but the string-specific operations tries support naturally: `longestPrefixOf`, `keysWithPrefix`, and `collect`, with `keysWithPrefix` being the engine behind autocomplete. The lecture closes by comparing three strategies for mapping a character to a child node (`DataIndexedCharMap`, hash table, BST), all Θ(1) given a fixed alphabet size R, but with very different memory costs.

---

## Key Concepts

### 1. ADTs vs. implementations, and why we keep adding implementations

The lecture opens by recapping the landscape so far. Each ADT has many implementations:

- **Set / Map**: LinkedList, resizing array, vanilla BST, LLRB, B-Trees (2-3 / 2-3-4), separate chaining hash table
- **Priority Queue**: heap, balanced tree, ordered linked list
- **List**: LinkedList, resizing array
- **Disjoint Sets**: Quick Find, Quick Union, Weighted QU, WQUPC

The textbook chapter frames all of these as attacks on the same **search problem**: data comes in, and we want to retrieve the parts we care about. What differs is *what you retrieve by*: index (List), key identity (Map/Set), key order (PQ), or integer-pair connectivity (Disjoint Sets). Because these are ADTs, we specify behavior, not implementation, and we routinely build one ADT on top of another (a PQ on a heap-ordered tree on a tree). That layering is exactly what happens inside a trie: a trie node's children are themselves stored in a map, and we get to pick which map.

The starting benchmark for Set:

| Implementation | `contains(x)` | `add(x)` |
| --- | --- | --- |
| Balanced BST | Θ(log N) | Θ(log N) |
| Resizing separate chaining hash table | Θ(1) assuming even spread | Θ(1) on average, assuming even spread |

Both are already fast. The lecture's thesis: **knowing something about the key type lets you do better still.**

### 2. Special case 1: keys are single characters

If every key is a single ASCII character, you do not need comparisons or hashing at all. A character *is* an array index. So just allocate an array of size R (R = number of possible characters, e.g. 128 for ASCII) and use the character's numeric value as the slot. `get` and `put` become single array accesses: Θ(1), and not "Θ(1) on average assuming even spread" like a hash table, but genuinely, unconditionally Θ(1) with no collisions possible.

This is the `DataIndexedCharMap`, and it is the key building block for the rest of the lecture.

### 3. Special case 2: keys are strings, so store one character per node

The trie idea, stated on the slides as two bullets:

- **Every node stores only one letter.**
- **Nodes can be shared by multiple keys.**

Walk through the lecture's running example, inserting `"sam"`, `"sad"`, `"sap"`, `"same"`, `"a"`, `"awls"`:

1. Insert `"sam"`: root gets a child `s`, which gets a child `a`, which gets a child `m`.
2. Insert `"sad"`: `s` and `a` already exist and are reused. Only `d` is new, as a second child of `a`. So `a` now has children `{d, m}`.
3. Insert `"sap"`: again `s`, `a` are shared; add `p`. Now `a` has children `{d, m, p}`.
4. Insert `"same"`: `s`, `a`, `m` are shared; add `e` as child of `m`.
5. Insert `"a"`: nothing in the `s` subtree helps, since `"a"` starts at the root. The root gains a *second* child `a` (distinct from the `a` under `s`).
6. Insert `"awls"`: shares the new top-level `a`, then adds the chain `w -> l -> s`.

The resulting shape (reconstructed from the slides):

```
            (root)
           /      \
          a        s
          |        |
          w        a
          |      / | \
          l     d  m  p
          |        |
          s        e
```

Compare the same six keys in a BST (`sad` at the root, with `awls` and `same` as children, and `a`, `sam`, `sap` below) and in a hash table (buckets 0-3 holding `sad`/`sam`, `awls`, `a`/`sap`, `same`). The BST and hash table store each string whole; the trie stores the *alphabet of positions*, decomposing each key digit by digit and sharing the overlap. The slides call a trie "a digit-by-digit set representation."

### 4. The marking problem: search hits vs. misses

Once nodes are shared, the tree alone no longer tells you which keys are in the set. In the picture above, is `"sa"` in the set? There is a path root -> `s` -> `a`, so a naive "did I find a path?" check would say yes, which is wrong. Likewise `"aw"` and `"awl"` are paths but not keys.

The fix: **each node carries a boolean flag** (drawn on the slides as blue vs. white nodes, and called `isKey` in code) saying "a key ends here." Only the nodes at the end of `a`, `awls`, `sad`, `sam`, `same`, `sap` are marked.

That gives two distinct ways a search can miss:

- **The final node is white** (the path exists but no key ends there): `contains("sa")` is false.
- **You fall off the tree** (no child for the next character): `contains("saq")` and `contains("sax")` are false.

And a hit is: you consume every character of the query *and* land on a marked node. `contains("sam")` and `contains("a")` are true.

### 5. Tries as maps

Nothing about the structure is set-specific. For a trie **map**, instead of a boolean at each node, store a value (or `null`/`None` for "not a key"). The slide's example map holds nodes spelling `by -> 4`, `sea -> 6`, `sells -> 1`, `she -> 0`, `shells -> 3`, `the -> 5`, `shore -> 7`, and so on, with the numeric value sitting on the node where the key ends. "e.g. maps `by` to 4."

### 6. Performance in terms of N vs. L

Given a trie with N keys: what are the `add` and `contains` runtimes? The lecture's answer is **Θ(1)** for both, because the work done is walking down one character at a time for the length of the key, and that has nothing to do with how many keys are stored. The runtimes are *independent of the number of keys*.

If instead you parameterize by L, the length of the key:

- `add`: **Θ(L)** (you must walk/create all L levels)
- `contains`: **O(L)** (big-O, not Theta, because you can bail out early by falling off the tree or by exhausting the query)

So the comparison table becomes:

| Implementation | Key type | `get(x)` | `add(x)` |
| --- | --- | --- | --- |
| Balanced BST | comparable | Θ(log N) | Θ(log N) |
| Resizing separate chaining hash table | hashable | Θ(1) assuming even spread | Θ(1) on average, assuming even spread |
| Data indexed array | chars | Θ(1) | Θ(1) |
| Tries (BST, hash table, or `DataIndexedCharMap`) | Strings | Θ(1) | Θ(1) |

The slides are explicit that these trie entries are "runtimes treating length of keys as a constant." That is the whole reason the N-based answer is Θ(1).

(extra context) Note also that a hash table's Θ(1) is not free of L either: hashing a string of length L costs Θ(L) to read the characters, and a collision requires `equals` comparisons that also scan characters. So the trie's advantage on `get`/`add` is genuine but modest, which is exactly why the lecture pivots to string operations as the real selling point.

### 7. Choosing how a node tracks its children

Every trie node needs a map from character to child node. The lecture's three natural choices:

| Strategy | Links per node | Time per child lookup | Notes |
| --- | --- | --- | --- |
| `DataIndexedCharMap` | 128 (all of R), mostly `null` | Θ(1) | Very fast, very memory hungry |
| Hash table | C, where C = number of actual children | O(R) worst case | Almost as fast, less memory |
| Balanced BST | C | O(log R) | A little slower than hash table, similar memory |

Two refinements the slides stress:

- **Cost per link is higher** in the BST and hash table versions (you store keys, pointers, bucket overhead, etc.), so "C links" is not an apples-to-apples comparison with "128 links."
- **Since R is fixed** (e.g. 128 for ASCII), all three are Θ(1) asymptotically. The O(log R) and O(R) factors are constants. The real distinction is practical memory and constant factors, and the lecture's advice is that you "would have to do computational experiments to see which is best for your application." The summary slide says the **hash table is probably the most natural** choice.

The fundamental problem with `DataIndexedCharMap` is stated plainly: **our arrays are sparse.** Real-world tries have nodes with one or two children, so 126 of 128 slots are wasted. The slides illustrate this with a node holding `a` whose child map is "128 links, with one used, and 127 equal to null."

### 8. The redundant-character observation

In the first implementation, each node stores a `char ch`. But to reach a node you must have gone through a specific slot in its parent's child map, and that slot already encodes the character. So **`ch` is redundant and can be removed** and "things will work fine." The lecture shows the stripped-down node with only `isKey` and `next`.

(extra context) This is only cleanly true for the `DataIndexedCharMap` and hash-table versions, where the character is the lookup key into the parent's map. A BST-based child map also stores the character as its BST key, so the character still lives somewhere, just not duplicated in the child node.

### 9. Prefix operations: the actual reason to use a trie

"Theoretical asymptotic speed improvement is nice. But the main appeal of tries is their ability to efficiently support string specific operations like prefix matching."

On the running example:

- `longestPrefixOf("sample")` returns `"sam"`: walk down `s`, `a`, `m`, then `p` has no child matching `l`... more precisely, walk as far as the query lets you and report the longest key-marked prefix encountered. Here `sam` is a key, so that is the answer.
- `keysWithPrefix("sa")` returns `[sad, sam, same, sap]`: descend to the node reached by `"sa"`, then collect everything below it.
- `collect()` returns `["a", "awls", "sad", "sam", "same", "sap"]`: all keys in the trie.

The crucial structural fact: **everything sharing a prefix lives in one subtree.** Finding that subtree costs O(length of prefix), and then the collection cost is proportional to the output size, not to N. A BST or hash table cannot do this (a hash table in particular scatters `sad` and `same` into unrelated buckets, so prefix queries degrade to scanning everything).

### 10. Autocomplete (marked Optional on the slides)

The naive approach: build a trie map from strings to values, where the value encodes "how important this string is" (e.g. Google's ranking). Billions of strings fit efficiently because they share nodes. On input `"hello"`, call `keysWithPrefix("hello")` and return the 10 highest-valued results.

Worked example with six keys: `buck: 10`, `sad: 12`, `smog: 5`, `spit: 15`, `spite: 20`, `spy: 7`. On input `"s"`, `keysWithPrefix("s")` gives `sad, smog, spit, spite, spy`, and the top three by value are `spit, spite, sad`.

**The major flaw**: short prefixes match enormous numbers of keys. "We are collecting billions of results only to keep 10! This is extremely inefficient."

**The fix**: each node stores its own value *and* the best (maximum) value anywhere in its subtree. Search then explores nodes **in order of `best`**, i.e. as a priority-queue-driven best-first search. In the slide's example the root's `best` is 20 (from `spite`), the `s` node's `best` is 20, and under it `p` has `best` 20 while `m` has `best` 5, so the search "considers `sp` before `sm`." You can **stop as soon as your top 3 matches are all better than the best remaining** candidate, pruning whole subtrees you never touch. The summary slide underlines the lesson: "Optimal implementation of Autocomplete involves use of a priority queue!"

**Radix trees**: you can further merge chains of redundant single-child nodes, so `b-u-c-k` collapses into a single `buck` node and `a-d` into `ad`. "This version of trie is known as a 'radix tree' or 'radix trie'." The lecture explicitly does not discuss it further.

### 11. Other domain-specific string structures

The closing generalization: "we can sometimes take special advantage of our key type to improve our sets and maps." Tries are one instance for strings; the slides name **suffix trees** and **DAWGs** as other string set/map structures, both explicitly out of scope for the course.

---

## Definitions

- **Search problem**: the general problem of taking in a stream of data and retrieving the information of interest. All of List, Map, Set, Priority Queue, and Disjoint Sets are solutions to different flavors of it.
- **Abstract data type (ADT)**: a specification of *behavior* (what operations exist and what they mean) with no commitment to *implementation*. Multiple implementations can satisfy the same ADT, and ADTs can be implemented in terms of other ADTs.
- **Abstraction**: being concerned only with the behavior of something, not its underlying implementation. Applied in layers: a PQ uses a heap-ordered tree, which uses a tree, and each layer ignores the internals of the one below.
- **`DataIndexedCharMap<V>`**: a map from characters to values implemented as an array of length R, where the character's numeric value is used directly as the array index. Θ(1) `get` and `put`.
- **R**: the number of possible characters in the alphabet, e.g. 128 for ASCII. Treated as a fixed constant.
- **L**: the length of a key (a string).
- **N**: the number of keys stored in the data structure.
- **C**: for a given trie node, the number of children it actually has.
- **Trie**: a set/map implementation for String keys in which every node stores a single character, nodes are shared among keys with common prefixes, and each node carries a flag (`isKey`) or value indicating whether a key terminates there. Short for **Retrieval Tree**. Inventor Edward Fredkin suggested pronouncing it "tree," but almost everyone says "try."
- **`isKey`**: the boolean on a trie node meaning "some key ends exactly here." Drawn as a blue (true) or white (false) node on the slides.
- **Search hit**: all characters of the query are matched by a path from the root, *and* the final node has `isKey == true`.
- **Search miss**: either the final node is white (`isKey == false`), or you fall off the tree because some node lacks the needed child.
- **`collect()`**: returns all keys in the trie.
- **`keysWithPrefix(s)`**: returns all keys in the trie having `s` as a prefix.
- **`longestPrefixOf(s)`**: returns the longest prefix of `s` that is a key in the trie.
- **Radix tree / radix trie**: a trie in which redundant single-child chains are merged into single multi-character nodes. Mentioned, not covered.
- **Suffix tree, DAWG**: other string-specialized set/map structures. Named only; out of scope.

---

## Worked Examples

### Example 1: `DataIndexedCharMap`

The lecture's fill-in-the-blank exercise, completed:

```java
public class DataIndexedCharMap<V> {
    private V[] items;

    public DataIndexedCharMap(int R) {
        items = (V[]) new Object[R];
    }

    public void put(char c, V val) {
        items[c] = val;
    }

    public V get(char c) {
        return items[c];
    }
}
```

**What it does, line by line.** The constructor takes R, the alphabet size, and allocates a backing array of that length. The cast `(V[]) new Object[R]` is the standard generic-array workaround in Java: you cannot write `new V[R]`, so you create an `Object[]` and cast it, accepting an unchecked-cast warning.

**Why `items[c]` works.** In Java a `char` silently widens to an `int` in an array-index position, and that `int` is the character's code point. `'a'` is 97, so `put('a', val)` writes to slot 97. There is no hashing, no comparison, and no collision: distinct characters are distinct indices by construction. That is why the slides can claim unconditional Θ(1) rather than hash-table-style "Θ(1) assuming even spread."

**Box-and-pointer picture in words.** A `DataIndexedCharMap` object holds one reference, `items`, pointing to a 128-slot array. Each slot is a reference, almost all `null`. `put('a', x)` overwrites slot 97 to point at `x`. Nothing else in the array is touched, and the array never resizes, because R is fixed up front.

### Example 2: the very basic trie node

```java
public class TrieSet {
    private static final int R = 128;  // ASCII
    private Node root;                 // root of trie

    private static class Node {
        private char ch;
        private boolean isKey;
        private DataIndexedCharMap<Node> next;

        private Node(char c, boolean b, int R) {
            ch = c;
            isKey = b;
            next = new DataIndexedCharMap<>(R);
        }
    }
}
```

**Reading the fields.** `ch` is the single letter at this node. `isKey` is the blue/white marker. `next` is the character-to-child map, and since the keys of that map are characters, a `DataIndexedCharMap` is the obvious choice.

**Box-and-pointer picture in words.** One `TrieSet` object holds `root`. Each `Node` holds a char, a boolean, and a reference to a `DataIndexedCharMap`, which in turn holds a reference to a 128-slot `Node[]`. So a node with one child is really *three* heap objects (the node, the map, the 128-element array) with 127 null slots. That is the memory complaint: for the six-key example trie, you are paying roughly (number of nodes) x 128 references.

**The redundancy observation.** Suppose you are standing at the node for `a` and you follow `next.get('w')` to reach a child. You already know that child's character is `'w'`, because you asked for slot `'w'`. So the `ch` field in the child is never needed to answer anything, and the lecture notes it "can [be] remove[d] from the representation and things will work fine," leaving:

```java
private static class Node {
    private boolean isKey;
    private DataIndexedCharMap<Node> next;

    private Node(char c, boolean b, int R) {
        isKey = b;
        next = new DataIndexedCharMap<>(R);
    }
}
```

(extra context) The slides leave the constructor's `char c` parameter in place even after deleting the field, which just means the argument is now ignored. In real code you would drop the parameter too.

(extra context) The lecture never writes out `add` and `contains`, but they follow directly from the structure:

```java
// (extra context) not in the lecture, but the natural completion
public boolean contains(String key) {
    Node curr = root;
    for (int i = 0; i < key.length(); i += 1) {
        char c = key.charAt(i);
        if (curr.next.get(c) == null) {
            return false;            // fell off the tree
        }
        curr = curr.next.get(c);
    }
    return curr.isKey;               // white node => miss
}

public void add(String key) {
    if (key == null || key.length() == 0) { return; }
    Node curr = root;
    for (int i = 0; i < key.length(); i += 1) {
        char c = key.charAt(i);
        if (curr.next.get(c) == null) {
            curr.next.put(c, new Node(c, false, R));
        }
        curr = curr.next.get(c);
    }
    curr.isKey = true;               // mark the terminal node blue
}
```

Note how the two miss cases from the lecture map onto two different `return false` / final-check sites, and how `add` only ever creates the nodes that do not already exist, which is precisely the node-sharing property.

### Example 3: `collect()`, the lecture's "challenging warmup exercise"

The lecture's pseudocode, verbatim in structure:

```
collect():
  - Create an empty list of results x.
  - For character c in root.next.keys():
      - Call colHelp(c, x, root.next.get(c)).
  - Return x.

colHelp(String s, List<String> x, Node n):
  - If n.isKey, then x.add(s).
  - For character c in n.next.keys():
      - Call colHelp(s + c, x, n.next.get(c))
```

In Java:

```java
public List<String> collect() {
    List<String> x = new ArrayList<>();
    for (char c : root.next.keys()) {
        colHelp("" + c, x, root.next.get(c));
    }
    return x;
}

private void colHelp(String s, List<String> x, Node n) {
    if (n.isKey) {
        x.add(s);
    }
    for (char c : n.next.keys()) {
        colHelp(s + c, x, n.next.get(c));
    }
}
```

**Why it is shaped this way.** The root is special: it corresponds to the empty string and holds no character of its own, so `collect` handles the root's children itself and hands each one a one-character string. From then on `colHelp` has an invariant worth stating explicitly: **`s` is exactly the string spelled by the path from the root down to `n`, inclusive.** Given that invariant, "is `s` a key?" is answered by `n.isKey`, and recursing into child `c` means the path now spells `s + c`.

**The lecture's step-by-step trace** on the six-key trie:

| Step | Call | `x` after the call's `isKey` check |
| --- | --- | --- |
| 1 | `colHelp("a", x, <a>)` | `["a"]` (the `a` node is blue) |
| 2 | `colHelp("aw", x, <w>)` | `["a"]` (white) |
| 3 | `colHelp("awl", x, <l>)` | `["a"]` (white) |
| 4 | `colHelp("awls", x, <s>)` | `["a", "awls"]` (blue) |
| 5 | recursion unwinds, then `colHelp("s", x, <s>)` | `["a", "awls"]` (white) |
| 6 | `colHelp("sa", x, <a>)` | `["a", "awls"]` (white) |
| 7 | ... descend into `d`, `m`, `e`, `p` ... | `["a", "awls", "sad", "sam", "same", "sap"]` |

Notice the order the slides show: the `a` subtree is fully exhausted before `"s"` is ever visited. This is a plain depth-first traversal, and because `DataIndexedCharMap` keys come out in character order, the result arrives sorted: `["a", "awls", "sad", "sam", "same", "sap"]`.

**Environment reasoning in words.** Each `colHelp` frame has its own local `s`, its own `n`, and its own loop variable `c`, but all frames share *the same* `x`, because `x` is passed by value as a reference to one `ArrayList` object on the heap. Mutating `x` in a deep frame is visible in the shallow one, which is why no return value is needed. Meanwhile `s + c` creates a *new* String each time; it does not mutate the caller's `s`, so when frame 4 returns, frame 3's `s` is still `"awl"`. That difference, shared mutable accumulator vs. fresh immutable path string, is the whole trick.

(extra context) The maximum stack depth is the length of the longest key plus one, and `s + c` allocates a new string of length up to L at every node, so the traversal does O(total characters) extra string work. Using a `StringBuilder` with push/pop would avoid that.

### Example 4: `keysWithPrefix`

The lecture's algorithm:

```
Algorithm:
  - Find the node α corresponding to the string (shown in pink).
  - Create an empty list x.
  - For character c in α.next.keys():
      - Call colHelp("sa" + c, x, α.next.get(c)).
```

In Java:

```java
public List<String> keysWithPrefix(String prefix) {
    Node alpha = root;
    for (int i = 0; i < prefix.length(); i += 1) {
        alpha = alpha.next.get(prefix.charAt(i));
        if (alpha == null) {
            return new ArrayList<>();   // (extra context) prefix not present at all
        }
    }
    List<String> x = new ArrayList<>();
    for (char c : alpha.next.keys()) {
        colHelp(prefix + c, x, alpha.next.get(c));
    }
    return x;
}
```

**Step by step for `keysWithPrefix("sa")`.**

1. Start at root, follow `'s'`, then `'a'`. Now `alpha` is the node spelling `"sa"`, the pink node on the slide.
2. `alpha.next.keys()` is `{d, m, p}`.
3. `colHelp("sad", x, <d>)`: `d` is blue, so add `"sad"`. No children, return.
4. `colHelp("sam", x, <m>)`: `m` is blue, add `"sam"`. Then recurse into `e`: `colHelp("same", x, <e>)`, which is blue, add `"same"`.
5. `colHelp("sap", x, <p>)`: blue, add `"sap"`.
6. Result: `[sad, sam, same, sap]`, matching the slide.

**Why this is the structural payoff.** Step 1 costs O(length of prefix). Steps 2-5 touch only the `"sa"` subtree. Nothing in the `a`/`awls` branch is examined at all. In a hash table, `sad` and `same` hash to unrelated buckets, so there is no subtree to descend into and you would have to inspect every key.

**One subtlety worth noticing.** This version does not report the prefix itself even if the prefix is a key. If the trie contained `"sa"`, `keysWithPrefix("sa")` as written above would omit it, because `alpha.isKey` is never checked. The lecture's example happens not to expose this since `"sa"` is white.

### Example 5: autocomplete with subtree-best values (Optional section)

Keys and values: `buck: 10`, `sad: 12`, `smog: 5`, `spit: 15`, `spite: 20`, `spy: 7`.

**Naive version.** `keysWithPrefix("s")` collects `sad, smog, spit, spite, spy`, then sorts/selects the top three by value: `spit (15)`, `spite (20)`, `sad (12)`, reported by the slide as `spit, spite, sad`. Fine for six keys, catastrophic for a billion.

**Augmented version.** Annotate every node with `value` (its own, possibly `None`) and `best` (the max value in its subtree). From the slide:

- root: `value = None`, `best = 20`
- `b`: `value = None`, `best = 10`; the chain `u`, `c`, `k` carries `best = 10`, with `k` holding `value = 10`
- `s`: `value = None`, `best = 20`
- under `s`: `a` has `best = 12`, `m` has `best = 5`, `p` has `best = 20`
- under `p`: `i` has `best = 20`, `y` has `value = 7, best = 7`
- under `i`: `t` has `value = 15, best = 20`, and its child `e` has `value = 20, best = 20`

**How the search runs.** Maintain a priority queue of nodes ordered by `best`, highest first. Pop the node with the greatest `best`, record it if it is a key, push its children. Because `p.best = 20 > m.best = 5`, the search "consider[s] 'sp' before 'sm'," and because it processes in descending `best` order, it can **stop as soon as the three results already found are all better than the `best` of every remaining queue entry.** The `smog` subtree (best 5) is never expanded once three results above 5 exist.

**Why the invariant is sound.** `best` is an upper bound on any value in a subtree, so if your current 3rd-best answer beats a node's `best`, nothing inside that node can displace it. This is exactly the pruning argument, and it is why the summary slide says the optimal autocomplete uses a priority queue.

---

## Common Pitfalls

1. **Forgetting `isKey` / assuming "path exists" means "key present."** This is the single biggest conceptual trap. `"sa"`, `"aw"`, and `"awl"` are all walkable paths in the example trie and none is a key. A `contains` that returns `true` upon consuming the last character without checking `isKey` is wrong.
2. **Handling only one kind of miss.** There are two: white final node, and falling off the tree. Code must cover both.
3. **Claiming "Θ(1) `add` and `contains`" without the caveat.** The Θ(1) is *in terms of N*. In terms of L it is Θ(L) for `add` and O(L) for `contains`. On an exam, say which variable you are measuring in.
4. **Writing Θ(L) for `contains`.** The lecture deliberately writes **O(L)**, because a miss can terminate early (fall off the tree after one character).
5. **Treating the trie as strictly better than a hash table for `get`/`add`.** The lecture's word is "slightly better." The hash table's Θ(1) is already very good, and the trie pays real constant factors (pointer chasing, one object per character). The reason to reach for a trie is prefix operations.
6. **Thinking `DataIndexedCharMap` is just bad.** It is the fastest of the three child-tracking strategies. Its problem is memory: 128 links per node, nearly all null, i.e. sparse arrays.
7. **Assuming the hash-table child map is asymptotically Θ(1) in the worst case.** The slide lists it as **O(R)**, because all R characters could collide into one bucket. It is only "Θ(1)" after you fold the fixed R into the constant.
8. **Comparing "C links" to "128 links" naively.** The slides warn that the **cost per link is higher** in BST and hash-table nodes.
9. **Keeping the `char ch` field and believing it is necessary.** The character is already encoded by which slot of the parent's map you came through.
10. **In `colHelp`, mutating a shared string instead of building fresh ones, or returning a new list per call.** The pattern is: one shared mutable accumulator `x`, and a fresh path string `s + c` per recursive call. Mixing these up gives either corrupted paths or lost results.
11. **Forgetting the root is the empty string.** The root stores no character, which is why `collect()` seeds the recursion with the root's children rather than calling `colHelp("", x, root)` with a character-bearing root.
12. **Naive autocomplete on a short prefix.** Collecting billions of keys to return 10 is "extremely inefficient." You need the `best`-value augmentation plus a priority queue.
13. **Confusing a radix tree with a trie.** A radix tree merges single-child chains (`buck` as one node). The lecture names it and explicitly does not cover it.
14. **Mispronouncing or misexplaining the name.** "Trie" is short for **Retrieval** Tree; Fredkin wanted "tree," everyone says "try."

---

## Likely Exam Points

### 1. Draw the trie after a given insertion sequence

**Q.** Insert `"sea"`, `"sell"`, `"she"`, `"shell"` into an initially empty trie. How many nodes (excluding the root) does it contain, and which are marked `isKey`?

**A.** The root gets one child `s`. Under `s`: children `e` and `h`. Under `e`: `a` (key, for `sea`) and `l`; under that `l`: `l` (key, for `sell`). Under `h`: `e` (key, for `she`), then `l`, then `l` (key, for `shell`). Counting: `s`, `e`, `a`, `l`, `l`, `h`, `e`, `l`, `l` = 9 nodes. Four are marked `isKey`, one per inserted key. Note `sea` and `sell` share `s`,`e` and `she`/`shell` share `s`,`h`,`e`, which is the node-sharing property.

### 2. Hit vs. miss classification

**Q.** In the lecture's trie holding `{sam, sad, sap, same, a, awls}`, classify each: `contains("same")`, `contains("awl")`, `contains("sade")`, `contains("a")`.

**A.** `"same"`: hit (path exists, final node blue). `"awl"`: miss because the final node is white (it is only a prefix of `awls`). `"sade"`: miss because we fall off the tree (the `d` node has no child `e`). `"a"`: hit (the top-level `a` node is blue).

### 3. Runtime of `add`/`contains`, in N and in L

**Q.** A trie stores N string keys over a fixed alphabet of size R, with maximum key length L. Give the runtime of `contains` in terms of N, and in terms of L, and explain the difference between Θ and O in your answers.

**A.** In terms of N: Θ(1), since the number of nodes visited depends only on the query's length, not on how many keys are stored. In terms of L: O(L), because you visit at most one node per character, but you may stop early by falling off the tree, so L is only an upper bound. `add` is Θ(L), since it must walk or create all L levels and cannot terminate early.

### 4. Child-tracking strategy comparison

**Q.** Fill in the table for a trie node with C children over an alphabet of size R.

| Strategy | Links stored per node | Child-lookup time |
| --- | --- | --- |
| `DataIndexedCharMap` | ? | ? |
| Hash table | ? | ? |
| BST | ? | ? |

**A.** `DataIndexedCharMap`: R links (e.g. 128), Θ(1). Hash table: C links, O(R) worst case. BST: C links, O(log R). Since R is a fixed constant, all three are Θ(1) asymptotically; the differences are memory (R vs. C links, but with a higher cost per link in the hash/BST cases) and constant factors. The lecture's practical pick: hash table, as "probably the most natural."

### 5. Why can't a hash table do `keysWithPrefix` efficiently?

**Q.** You have a set of N strings. Explain why a trie supports `keysWithPrefix(p)` far more efficiently than a hash set does, and give the cost structure for the trie.

**A.** A good hash function scatters similar strings into unrelated buckets, so `sad` and `same` have no positional relationship, and the only correct way to answer a prefix query is to examine all N keys: Θ(N) times the cost of a prefix comparison. In a trie, every key sharing prefix `p` lies in the single subtree rooted at the node reached by walking `p`. So the cost is O(|p|) to find that node plus work proportional to the size of that subtree (the output), independent of N.

### 6. Write or complete `collect` / `colHelp`

**Q.** The following is meant to collect all keys. What is wrong with it, and what would it output on the trie holding `{a, awls}`?

```java
private void colHelp(String s, List<String> x, Node n) {
    for (char c : n.next.keys()) {
        colHelp(s + c, x, n.next.get(c));
    }
    if (n.next.keys().isEmpty()) {
        x.add(s);
    }
}
```

**A.** It uses "has no children" as the test for "is a key" instead of checking `n.isKey`. That happens to work for leaves but fails for keys that are proper prefixes of other keys. On `{a, awls}` it outputs only `["awls"]`, dropping `"a"`, because the `a` node has a child `w` and so is never added. The correct test is `if (n.isKey) { x.add(s); }`, which is exactly why the `isKey` flag exists.

### 7. The `colHelp` invariant and recursion mechanics

**Q.** In `colHelp(String s, List<String> x, Node n)`, what is the relationship between `s` and `n`, and why does the method not need a return value?

**A.** Invariant: `s` is exactly the string spelled by the root-to-`n` path, including `n`'s own character. No return value is needed because every recursive frame receives a reference to the *same* `List` object `x` and mutates it in place; results accumulate in shared heap state. Meanwhile `s + c` builds a new immutable String per call, so a child's path does not disturb the parent's `s`.

### 8. `longestPrefixOf`

**Q.** On the trie holding `{sam, sad, sap, same, a, awls}`, what is `longestPrefixOf("sample")`? What about `longestPrefixOf("award")`?

**A.** `longestPrefixOf("sample")` is `"sam"`: walking `s`, `a`, `m` succeeds and `m` is a key; continuing to `p` fails (the `m` node's only child is `e`), so the last key seen, `sam`, is the answer. `longestPrefixOf("award")` is `"a"`: `a` is a key, then `w` matches, then `a` does not match (the `w` node's child is `l`), so the longest key-prefix found is `"a"`.

### 9. Efficient autocomplete

**Q.** Why is "call `keysWithPrefix(p)` then take the top 10 by value" a bad autocomplete implementation, and how does the lecture fix it?

**A.** For a short `p`, the prefix subtree can hold billions of keys, so you collect billions of results to keep 10. The fix: store at each node both its own value and `best`, the maximum value in its subtree; then search best-first using a priority queue ordered by `best`, and stop once the current top 10 all exceed the `best` of every unexpanded node. Since `best` upper-bounds everything in a subtree, such subtrees can be pruned safely. Optionally, merge redundant single-child chains into a radix tree for additional savings.

### 10. Terminology

**Q.** What is "trie" short for, and what is one other string-specialized set/map structure named in this lecture?

**A.** Retrieval Tree (pronounced "try" by almost everyone, though Fredkin preferred "tree"). Other named structures: suffix trees and DAWGs, both out of scope for the course.

---

## Summary

- There are many implementations of every ADT; knowing your **key type** sometimes lets you beat general-purpose structures. Tries are the specialization for **String** keys.
- **`DataIndexedCharMap`**: with single-character keys, a character *is* an array index, so `items[c] = val` / `return items[c]` gives unconditional Θ(1) `put`/`get` over an array of size R (128 for ASCII).
- **Trie structure**: one character per node, nodes shared across keys with common prefixes, and an `isKey` boolean (or a value, for maps) marking where a key ends.
- `isKey` is essential: without it, prefixes like `"sa"`, `"aw"`, `"awl"` would falsely appear to be members.
- **Two kinds of miss**: the final node is white, or you fall off the tree. A hit requires both consuming the whole query and landing on a marked node.
- **Runtimes**: in terms of N, `add` and `contains` are Θ(1), independent of the number of keys. In terms of L, `add` is Θ(L) and `contains` is O(L) (big-O because of early exits).
- **Child tracking, three options**: `DataIndexedCharMap` (R links per node, Θ(1), memory hungry and sparse), hash table (C links, O(R), almost as fast and the most natural choice), BST (C links, O(log R)). With R fixed, all three are Θ(1); cost per link is higher for the latter two.
- The `char ch` field in a node is **redundant**, since the parent's child map slot already encodes the character.
- `collect()` is a DFS carrying the invariant "`s` spells the root-to-`n` path," adding `s` when `n.isKey`, with one shared mutable result list and fresh `s + c` strings.
- `keysWithPrefix(p)` = descend to the node for `p` in O(|p|), then run `collect`'s helper from there. The cost is output-proportional, not N-proportional, because all keys sharing a prefix live in one subtree.
- `longestPrefixOf(s)` walks down `s` and reports the longest key-marked prefix reached.
- **Autocomplete** (Optional): naive `keysWithPrefix` + top-10 collects far too much. Store each node's own value plus the `best` value in its subtree, search best-first with a **priority queue**, and prune once the top results all beat the best remaining. Merging single-child chains gives a **radix tree** (not covered).
- Other string-specific structures exist (suffix trees, DAWGs) but are out of scope.
- Bottom line from the slides: **data structures interact in beautiful and important ways** (a trie's children live in a map; its autocomplete needs a priority queue).
