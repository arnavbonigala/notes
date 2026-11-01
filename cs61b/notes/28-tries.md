<!-- Fri, Oct 30, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 28: Tries

## Overview

This lecture closes out hashing with a discussion of mutability, then introduces a new way to implement sets and maps: the **trie** (short for "retrieval tree," pronounced "try"). The big-picture message is that balanced BSTs and hash tables work for *arbitrary* data, but when we know something extra about our keys we can sometimes do better. If keys are single ASCII characters, we can just use an array indexed by the character's ASCII value (a `DataIndexedCharMap`), giving Θ(1) operations. If keys are *strings*, we can use a trie: a tree where every node holds a single character, nodes are shared between keys with common prefixes, and a boolean flag marks which nodes end a real key. Treating key length L as a constant, `add` and `contains` run in Θ(1) and O(1) time respectively, independent of the number of keys N. We look at three ways to track a node's children (`DataIndexedCharMap`, hash table, BST) and the memory/speed tradeoffs among them. The real payoff, though, is not the asymptotics: it is that tries efficiently support **string-specific operations** like `longestPrefixOf` and `keysWithPrefix`, the latter being the heart of autocomplete. The lecture ends on the theme that data structures compose: an optimal autocomplete combines a trie with a priority queue.

---

## Key Concepts

### 1. Why another set/map implementation at all?

By this point in the course we have many implementations of the same abstract data types:

| ADT | Implementations seen |
|---|---|
| Set / Map | LinkedList, resizing array, vanilla BST, LLRB, B-Trees (2-3 / 2-3-4), separate chaining hash table |
| Priority Queue | Heap, balanced tree, ordered linked list |
| List | LinkedList, resizing array |
| Disjoint Sets | Quick Find, Quick Union, Weighted QU, WQUPC |

The lecturer described this as a kind of allegory of the cave: the ADT is the shadow we interact with, and the implementation is the machinery behind it. We already have a Θ(1) implementation of sets (hash tables), which seems like the best you could dream of, so why keep going?

| | key type | `get(x)` / `contains(x)` | `add(x)` |
|---|---|---|---|
| Balanced BST | comparable | Θ(log N) | Θ(log N) |
| Resizing separate chaining hash table | hashable | Θ(1) (assuming even spread) | Θ(1) on average (assuming even spread) |

The answer is twofold. First, hash table Θ(1) comes with caveats (even spread, amortization, good hash function). Second, and more importantly, constant time is not the only thing we want. If we **know something about our keys**, we can get both unconditional constant time *and extra useful operations*. The tradeoff is roughly "constant time plus extra utility, in exchange for only working on a restricted key type."

### 2. Special case 1: character keys and the `DataIndexedCharMap`

Suppose we know every key is a single ASCII character (`'a'`, `'g'`, `'!'`). ASCII is itself a deterministic, perfect "hash function": it is a fixed standard saying `'a'` is 97, `'b'` is 98, `'c'` is 99, and so on. So we need no fancy hash code at all. Just make an array of length R (R = 128 for ASCII) and use the character as the index.

| | key type | `get(x)` | `add(x)` |
|---|---|---|---|
| data indexed array | chars | Θ(1) | Θ(1) |

This is simple, fast, and completely collision-free. It is the building block the first trie implementation uses. Note R is a parameter: 128 for ASCII, much larger for Unicode, smaller if your alphabet is smaller.

### 3. Special case 2: string keys, and the key trie idea

Now suppose keys are strings. Consider the set containing `"sam"`, `"sad"`, `"sap"`, `"same"`, `"a"`, `"awls"`. A BST would store whole strings compared lexicographically; a hash table would scatter whole strings into buckets:

```
BST (lexicographic)              Hash Table (buckets)
        sad                      0: sad, sam
       /   \                     1: awls
    awls   same                  2: a, sap
    /  \    /  \                 3: same
   a   sam     sap
```

A trie takes a completely different view. **Two key ideas:**

- **Every node stores only one letter.**
- **Nodes can be shared by multiple keys.**

Adding `"sam"` and `"sad"` gives a chain `s -> a` that branches to `m` and `d`. Then:

- `"sap"`: reuse `s -> a`, add child `p` to the `a` node.
- `"same"`: reuse `s -> a -> m`, add child `e` to the `m` node.
- `"a"`: shares nothing with the `s` branch, so it becomes a second child of the (empty) root.
- `"awls"`: hangs off that new `a` as `a -> w -> l -> s`.

The resulting trie (root is an invisible/empty node with children `a` and `s`):

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

This is "a digit-by-digit set representation": we break each word into its characters and let the tree structure encode shared prefixes. This matches how languages like English actually work, with shared roots and prefixes.

### 4. The problem of ending-ness: `isKey`

The structure above has a bug. Does it contain `"sa"`? Walking `s` then `a` lands on a real node, so a naive answer would be "yes," but `"sa"` was never inserted. Same for `"aw"` and `"awl"`.

**Fix:** mark each node with a boolean saying "a key ends here." The slides draw these as **blue** nodes (key ends here) versus **white** nodes (just a waypoint). In code this is the field `isKey`.

```
            (root)
           /      \
      [a:blue]   [s]
          |        |
         [w]      [a]
          |      / | \
         [l] [d:blue] [m] [p:blue]
          |            |
      [s:blue]    [e:blue]   ... (m is also blue, since "sam" is a key)
```

Note carefully that a node can be blue even if it has blue descendants (`m` for `"sam"` and `e` for `"same"` are both keys), and a node can be white even if blue nodes lie below it (the `a` under `s`, since `"sa"` is not a key, and the `w`/`l` under the top `a`).

### 5. Search hits and misses

For the set `{"sam", "sad", "sap", "same", "a", "awls"}`:

- `contains("sam")`: **true**, we land on a blue node.
- `contains("sa")`: **false**, we land on a white node.
- `contains("a")`: **true**, blue node.
- `contains("saq")`: **false**, we *fell off the tree* (the `a` node has no `q` child).

So there are exactly **two ways to miss**:

1. The final node is white (`isKey == false`).
2. We fall off the tree, i.e. some character along the way has no corresponding child link.

A hit requires both that the whole path exists *and* that the final node is a key.

### 6. Tries as maps

Tries can be maps, not just sets. Instead of a boolean `isKey`, each node stores a value (or `null` / "no value" if no key ends there). The slides show a trie map over keys including `by -> 4`, `sell -> 1`, `sea -> 6`, `she -> 0`, `shells -> 3`, `shore -> 7`, `the -> 5`. The general principle, stated in lecture: any set implementation can be turned into a map by storing a key-value pair where you previously stored just a key.

### 7. Trie performance in terms of N

Given a trie with N keys:

- `add` runtime: **Θ(1)**
- `contains` runtime: **Θ(1)**

This surprises people, and it did in lecture (answers were scattered across all options). The reason: the work done is determined entirely by the characters of the key being looked up. We follow one predetermined path, one character at a time. The depth we reach is bounded by the length of the key, not by how many keys are in the structure. Adding a million more words does not make `contains("sam")` slower.

Stated more precisely in terms of **L, the length of the key**:

- `add`: **Θ(L)** (we must walk/create all L levels)
- `contains`: **O(L)** (we may bail out early by falling off the tree)

Since string lengths are treated as a constant (words in a dictionary are short and do not grow with N), both are Θ(1) in N.

| | key type | `get(x)` | `add(x)` |
|---|---|---|---|
| Balanced BST | comparable | Θ(log N) | Θ(log N) |
| Resizing separate chaining hash table | hashable | Θ(1), even spread | Θ(1) avg, even spread |
| data indexed array | chars | Θ(1) | Θ(1) |
| Tries | Strings | Θ(1) | Θ(1) |

A point emphasized in lecture: the hash table's Θ(1) has caveats, while the trie's constant time is unconditional (no hash function quality assumption, no even-spread assumption).

### 8. Three strategies for tracking children

The question raised by students in lecture: given a node, how do we find the child labeled `'p'` quickly? If we did a linear scan over children, we would pay a search cost at every level. Three answers:

**(a) `DataIndexedCharMap`:** every node holds an array of R = 128 links, one per possible character, almost all `null`. Looking up child `'w'` means `next.get('w')`, which reads index 119. Very fast (Θ(1)), but memory hungry: "128 links, with one used, and 127 equal to null." The fundamental problem is that **our arrays are sparse**.

**(b) Hash table:** each node stores a hash table from character to child node. Almost as fast, uses much less memory because it stores only C links where C is the number of actual children.

**(c) Balanced BST:** each node stores a BST keyed by character. A little slower than the hash table, similar memory to the hash table. Also C links per node.

Memory and time comparison from the slides:

| Strategy | links per node | child lookup |
|---|---|---|
| `DataIndexedCharMap` | 128 (i.e. R) | Θ(1) |
| BST | C (number of children) | O(log R) |
| Hash Table | C (number of children) | O(R) |

Two caveats the slides make explicitly: the **cost per link is higher** in the BST and hash table versions (there is per-entry overhead beyond the bare pointer), and since **R is fixed** (e.g. 128), all three child-lookup costs can be thought of as Θ(1). So all three choices are fine; the hash table is "probably the most natural," and you would need computational experiments to pick the best for a specific application.

### 9. The 128-null-links debate

Is the `DataIndexedCharMap` version a memory disaster? The lecture gave both sides:

- **No:** 128 is a *constant*. ASCII never grows. So the per-node space is O(1) and the total is O(number of nodes), asymptotically fine.
- **Yes:** it is still extremely wasteful in practice. Real words use a handful of distinct next-characters, so you are allocating 128 slots to use maybe 3. This is the motivation for switching to hash tables or BSTs.

### 10. String-specific operations: the real reason to use tries

This is the punchline. The lecture opened with a warm-up: how do you find all words with a given prefix, say `"ente"`? The best idea with the tools we had was a double loop: for every word in the dictionary, compare it character by character against the prefix. That is linear in the number of words, and it is painful because many words are *almost* matches (`"intent"` vs `"enter"`) so you do real character work before rejecting them. Crucially, **none of our existing data structures (BST, hash table) give a clean way to do this.** Hashing destroys prefix structure entirely.

Tries make this natural because **all keys sharing a prefix live in the same subtree**. The two headline operations:

- `longestPrefixOf("sample")` -> `"sam"` (the longest key in the trie that is a prefix of the argument)
- `keysWithPrefix("sa")` -> `["sad", "sam", "same", "sap"]`

### 11. Collecting keys and `keysWithPrefix`

**`collect()`**, returning all keys (for our running example: `["a", "awls", "sad", "sam", "same", "sap"]`), as given in the slides:

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

The invariant: when `colHelp(s, x, n)` is called, `s` is exactly the string spelled out by the path from the root down to `n` (inclusive). So `n.isKey` means `s` is a key.

**`keysWithPrefix(prefix)`** is then a tiny modification:

```
- Find the node alpha corresponding to the prefix string.
- Create an empty list x.
- For character c in alpha.next.keys():
    - Call colHelp(prefix + c, x, alpha.next.get(c)).
```

The only structural difference from `collect()` is that we first walk down to the prefix node instead of starting at the root, and we seed the accumulated string with the prefix instead of the empty string. (One subtlety worth noting: as written in the slides this starts from `alpha`'s children, so if the prefix is itself a key it would not be added by this loop; a complete implementation also checks `alpha.isKey`. *(extra context)*)

### 12. Autocomplete (marked "Extra" / optional in the slides)

Google's "how are..." dropdown is a `keysWithPrefix` problem. The straightforward approach:

- Build a trie-based **map** from strings to values, where the value represents how important the string is (e.g. popularity).
- Billions of strings can be stored efficiently because they share nodes.
- When the user types `"hello"`, call `keysWithPrefix("hello")` and return the 10 highest-valued results.

Worked example: strings with values `buck: 10`, `sad: 12`, `smog: 5`, `spit: 15`, `spite: 20`, `spy: 7`. If the user types `"s"`, `keysWithPrefix("s")` gives `sad, smog, spit, spite, spy`, and the top three by value are `spit, spite, sad`.

**The major flaw:** for a short prefix, `keysWithPrefix` returns an enormous number of results. You collect billions of matches only to keep 10. Extremely inefficient.

**Improvement:** have each node store its own value *and* the best (maximum) value in its subtree. Then search nodes in order of "best," using a priority queue. In the example, the root's `s` child has `best = 20`, the `sp` subtree has `best = 20` while the `sm` subtree has `best = 5`, so we explore `sp` before `sm`. **We can stop as soon as the top 3 matches found so far are all better than the best remaining candidate.** This is where the "data structures interact in beautiful and important ways" theme lands: optimal autocomplete needs a trie *and* a priority queue.

**Further optimization (mentioned, not discussed):** merge redundant chains of single-child nodes, e.g. collapse `b-u-c-k` into a single `buck` node. This variant is called a **radix tree** or **radix trie**.

### 13. Other domain-specific string structures

More generally, we can exploit known key types to improve sets and maps. Other string set/map structures exist (**Suffix Trees**, **DAWG**) but were explicitly not covered.

### 14. Hashing wrap-up (first part of lecture)

Before tries, the lecture finished the hashing material:

**`hashCode` and `equals` must stay synchronized.** If you override `equals`, you must override `hashCode`. The reasoning: `equals` declares which objects should land in the same bucket; `hashCode` determines which bucket they actually land in. The default `hashCode` is essentially based on memory address, so two objects you *consider* equal would get different addresses and land in different buckets, breaking `contains`. Changing one without the other desynchronizes them in either direction.

**Mutability and hashing.** An object is mutable if its state can change after construction. An `int` is not mutable (3 is always 3). A `String` is not mutable: concatenation creates a brand-new string rather than modifying in place (which is also why repeated string concatenation is slow). The `final` keyword helps the compiler enforce immutability.

The mutability examples discussed:

```java
public class Pebble {
    public int weight;
    public Pebble(int w) { weight = w; }
}
// MUTABLE: Pebble p = new Pebble(1); p.weight = 2;
```

Making the field `final` blocks that mutation, so `p.weight = 2` no longer compiles: now immutable.

But `final` on a *reference* is weaker. A `final` array field still permits mutating the array's contents:

```java
// rocks is final, but:
stones[1] = null;   // still allowed, the array contents changed
```

And even making the field `private final` is not airtight: if a reference to the array was captured elsewhere (e.g. the caller kept the array they passed to the constructor), they can still mutate it afterward. "You really can't be sure that there aren't any back doors."

**Why mutable keys are bad in a hash table:** hashing is a deterministic function of the object's contents telling you which bucket to look in. Worked example from lecture:

- A list `[0, 1]` hashes to 962. With 4 buckets, `962 % 4 = 2`, so it goes in bucket 2.
- A list `[2, 3]` hashes to 1026, also landing in bucket 2. Fine so far, both live in bucket 2.
- Now mutate the first list: `items.add(7)`, making it `[0, 1, 7]`. Its hash code is now something different, say 2929829, and `2929829 % 4 = 1`.
- Query `contains([0,1,7])`: we compute hash 2929829, go to bucket **1**, find it empty, and report **false**. But the object *is* in the table, sitting in bucket 2. We looked in the wrong place.

Mutating a key after insertion makes it unfindable. If you do find it, that is a coincidence (the new hash happened to land in the same bucket).

**Java's actual `HashMap` (aside):** you can read the JDK source on GitHub. Java buckets are not always linked lists: if a bucket gets too full, Java replaces the linked list with a **red-black tree** so items in that bucket are easier to find. This ties to Knuth's observation that the most beautiful algorithms are not always the most useful, and the converse Josh quote that the most useful algorithms are not always the most beautiful. Sometimes you combine data structures to get the best of both worlds, at the cost of a clumsier description.

---

## Definitions

- **Search problem:** given a stream of data, retrieve the information of interest. All the data structures in the course solve different flavors of this problem (retrieve by index, by key identity, by key order, etc.). *(from textbook)*
- **Abstract Data Type (ADT):** a specification of *behavior* only, not implementation. Multiple implementations can satisfy the same ADT, and one ADT is often implemented using another. *(from textbook)*
- **Abstraction:** being concerned only with the behavior of something and not its underlying implementation. Applied in *layers*: a PQ may use a heap-ordered tree, which uses a tree, and each layer ignores the layer below's internals. *(from textbook)*
- **`DataIndexedCharMap<V>`:** a map from `char` to `V` implemented as an array of length R (the alphabet size, e.g. 128 for ASCII), where a character's own integer code is used directly as the array index. Θ(1) `get` and `put`.
- **R:** the number of possible characters in the alphabet, e.g. 128 for ASCII. Treated as a fixed constant.
- **Trie:** short for **Retrieval Tree**. A tree-based set/map implementation for string keys in which each node stores a single character, nodes are shared between keys with common prefixes, and each node carries a flag (or value) indicating whether a key ends at that node. Edward Fredkin, the inventor, suggested it be pronounced "tree," but almost everyone pronounces it "try."
- **`isKey`:** the boolean field on a trie node that is `true` exactly when the string spelled by the root-to-node path is a key in the set. Drawn as a blue node in the slides; `false` nodes are white.
- **Search hit:** a `contains` call that ends on a node with `isKey == true`.
- **Search miss:** a `contains` call that either (a) ends on a node with `isKey == false`, or (b) "falls off the tree" because some character has no corresponding child link.
- **L:** the length of the key (string) being operated on. Trie `add` is Θ(L); `contains` is O(L).
- **`collect()`:** returns all keys stored in the trie.
- **`keysWithPrefix(p)`:** returns all keys in the trie that begin with the string `p`. E.g. `keysWithPrefix("sa")` is `["sad", "sam", "same", "sap"]`.
- **`longestPrefixOf(s)`:** returns the longest key in the trie that is a prefix of `s`. E.g. `longestPrefixOf("sample")` is `"sam"`.
- **Autocomplete problem:** given a typed prefix, return the top-k most "important" stored strings having that prefix.
- **Radix tree / radix trie:** a trie variant in which redundant chains of single-child nodes are merged into one node (e.g. `b-u-c-k` becomes `buck`). Mentioned but not discussed.
- **Mutable:** an object is mutable if its state can be changed after construction. `int` and `String` are immutable in Java; a `Pebble` with a public non-`final` field is mutable.
- **Suffix Tree / DAWG:** other string-specialized set/map data structures, named but explicitly out of scope.

---

## Worked Examples

### Example 1: `DataIndexedCharMap`

The lecture's fill-in-the-blank code, completed:

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

**Step by step.** The constructor takes R, the alphabet size, and allocates an array of that length. Note the cast `(V[]) new Object[R]`: Java does not let you write `new V[R]` because of generic type erasure, so we allocate an `Object[]` and cast. This generates an unchecked-cast warning, which is expected and unavoidable here. *(extra context: the erasure explanation is standard Java background, not spelled out in this lecture.)*

In `put`, the expression `items[c]` silently widens the `char c` to an `int`. For `c == 'a'`, that is 97, so `put('a', val)` writes to slot 97. In `get`, the same conversion means we read slot 97 back. There is no hashing, no collision handling, no comparison: ASCII *is* the index function, and it is injective, so two different characters can never collide.

**Box-and-pointer reasoning.** After `new DataIndexedCharMap<String>(128)`, the object on the heap has one field `items` pointing to a 128-slot array, every slot `null`. After `put('a', "hello")`, slot 97 points at the string `"hello"`; the other 127 slots are still `null`. This is exactly the sparseness that will later motivate replacing the array with a hash table inside trie nodes.

**Runtime.** `get` and `put` are Θ(1): one array index computation and one memory access each.

### Example 2: The very basic trie implementation

From the slides:

```java
public class TrieSet {
    private static final int R = 128;   // ASCII
    private Node root;                  // root of trie

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

**Step by step.** `TrieSet` holds a single `root`. The root is an empty/invisible node: it stores no meaningful character, it exists only so that the first characters of all keys can be its children.

Each `Node` stores three things:
- `ch`: the character this node represents.
- `isKey`: whether a key ends here (blue vs white).
- `next`: the map from character to child node. Since we know keys are characters, we use a `DataIndexedCharMap`.

**Environment / box-and-pointer reasoning for `"a"` and `"awls"`.** Consider the node for the character `'a'` at the top level. Its box has `ch = 'a'`, `isKey = true` (because `"a"` is a key), and `next` pointing at a `DataIndexedCharMap`. That map's internal array has 128 slots. Exactly one is non-null: slot **119**, because `'w'` is ASCII 119. Slot 119 points to a `Node` with `ch = 'w'`, `isKey = false`, and its own 128-slot array whose only non-null slot is 108 (`'l'`). That `'l'` node is white and points on to an `'s'` node with `isKey = true`. So the chain `a -> w -> l -> s` is four nodes, four 128-slot arrays, and 512 pointer slots of which 3 are used plus the terminal node's 128 all-null.

**The redundancy observation.** The lecture points out that `ch` is actually **redundant**: if a node sits in slot 97 of its parent's `DataIndexedCharMap`, that node *is* the `'a'` child, by construction. Storing `ch = 'a'` repeats information the array index already encodes. So we can delete it:

```java
private static class Node {
    private boolean isKey;
    private DataIndexedCharMap<Node> next;

    private Node(boolean b, int R) {
        isKey = b;
        next = new DataIndexedCharMap<>(R);
    }
}
```

Caveat *(extra context)*: `ch` is only redundant because the parent's child map is keyed by character. If you ever iterate over a node's children in a representation that does not hand you the character alongside the child, you would need it back.

### Example 3: `contains` on a trie

The lecture described `contains` in words rather than writing it out; here is the direct translation. *(extra context: the code is mine, the algorithm is exactly the lecture's "walk the path, check the color, two ways to miss.")*

```java
public boolean contains(String key) {
    Node curr = root;
    for (int i = 0; i < key.length(); i += 1) {
        char c = key.charAt(i);
        curr = curr.next.get(c);
        if (curr == null) {
            return false;        // miss type 2: fell off the tree
        }
    }
    return curr.isKey;           // miss type 1 if false; hit if true
}
```

**Trace `contains("sam")`** on our trie:
1. `curr = root`. Read `'s'`. `root.next.get('s')` reads slot 115, non-null, so `curr` is the `s` node.
2. Read `'a'`. `curr.next.get('a')` reads slot 97, non-null, so `curr` is the `a`-under-`s` node. This node is **white**, but we do not care: we are not finished reading the key.
3. Read `'m'`. Slot 109 is non-null, so `curr` is the `m` node.
4. Loop ends. Return `curr.isKey`, which is `true`. **Hit.**

**Trace `contains("sa")`:** steps 1 and 2 as above, loop ends on the `a` node, return `a.isKey` which is `false`. **Miss, white node.** This is why the student's suggestion to "make the last letter special" was necessary.

**Trace `contains("saq")`:** steps 1 and 2 as above. Read `'q'` (ASCII 113). The `a` node's array has non-null entries only at `d`, `m`, `p`, so slot 113 is `null`. `curr == null`, return `false` immediately. **Miss, fell off the tree.** Note we returned after 3 character reads without ever looking at `isKey`; this early exit is why `contains` is O(L) rather than Θ(L).

**Why the runtime is independent of N.** Every iteration does one Θ(1) array index. The number of iterations is at most `key.length()`. Nothing in the loop depends on how many keys the trie holds. Insert a million more words and the trace above is character-for-character identical.

### Example 4: `add` on a trie

```java
public void add(String key) {
    Node curr = root;
    for (int i = 0; i < key.length(); i += 1) {
        char c = key.charAt(i);
        if (curr.next.get(c) == null) {
            curr.next.put(c, new Node(false, R));
        }
        curr = curr.next.get(c);
    }
    curr.isKey = true;
}
```

*(extra context: code mine; matches the lecture's conceptual description of insertion.)*

**Trace `add("same")`** into a trie already containing `"sam"` and `"sad"`:
1. `curr = root`. `'s'` child exists, so no allocation; descend. **Node shared with the existing keys.**
2. `'a'` child exists; descend. Shared again.
3. `'m'` child exists; descend. Shared again. Note we do *not* touch `m.isKey`, so `"sam"` remains a key.
4. `'e'` child does **not** exist. Allocate a new `Node(false, R)`, store it in slot 101 of `m`'s map, descend into it.
5. Loop ends. Set `e.isKey = true`.

Net effect: exactly one new node for a 4-character key, because 3 characters of prefix were already present. This node sharing is the whole reason a trie can "store billions of strings efficiently."

**Trace `add("a")`** into that trie: the root has no `'a'` child yet, so we allocate one, descend, loop ends immediately, set `isKey = true`. This is a second top-level branch, which is exactly the lecture's answer to "where should I put A?": off to the side, sharing nothing with the `s` branch. But it pays off immediately, because `add("awls")` then reuses it.

**Runtime.** Θ(L): we always do all L iterations (no early exit, since missing links get created), each Θ(1).

### Example 5: `collect()`, traced

Using the slide algorithm, rendered in Java:

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

**Trace on the running trie** (the slides step through exactly this):

| Call stack | `x` after |
|---|---|
| `colHelp("a", x, a)` -> `a.isKey` is true, add | `["a"]` |
| `colHelp("aw", x, w)` -> white, recurse | `["a"]` |
| `colHelp("awl", x, l)` -> white, recurse | `["a"]` |
| `colHelp("awls", x, s)` -> blue, add | `["a", "awls"]` |
| returns all the way up, next root child | |
| `colHelp("s", x, s)` -> white, recurse | `["a", "awls"]` |
| `colHelp("sa", x, a)` -> white, recurse over `d`, `m`, `p` | `["a", "awls"]` |
| ... `sad`, `sam`, `same`, `sap` collected | `["a", "awls", "sad", "sam", "same", "sap"]` |

**The crucial invariant:** the first argument `s` is always the string spelled out from the root down to and including `n`. The recursive call passes `s + c`, which extends the path string by exactly the character of the child we are descending into. This is why `n.isKey` is a valid test for "`s` is a key": the node's flag and the accumulated string always refer to the same path.

**Why it is a depth-first traversal.** `colHelp` fully explores one child's subtree before moving to the next, which is why we get `"a"` then all of `"awls"`'s branch before touching `s` at all. The results come out in alphabetical order here only because `DataIndexedCharMap.keys()` iterates the array in index order, which for ASCII letters is alphabetical order. *(extra context: with a hash-table child map the iteration order would be arbitrary.)*

### Example 6: `keysWithPrefix`

The slide algorithm, in Java:

```java
public List<String> keysWithPrefix(String prefix) {
    Node alpha = root;
    for (int i = 0; i < prefix.length(); i += 1) {
        alpha = alpha.next.get(prefix.charAt(i));
        if (alpha == null) {
            return new ArrayList<>();   // no keys with this prefix
        }
    }
    List<String> x = new ArrayList<>();
    if (alpha.isKey) {                  // in case the prefix itself is a key
        x.add(prefix);
    }
    for (char c : alpha.next.keys()) {
        colHelp(prefix + c, x, alpha.next.get(c));
    }
    return x;
}
```

**Trace `keysWithPrefix("sa")`:**
1. Walk `'s'` then `'a'`, landing on the `a`-under-`s` node. This is the node the slides mark in pink, call it α. Cost: 2 steps, i.e. O(length of prefix).
2. α is white, so the prefix itself is not added.
3. Iterate α's children: `d`, `m`, `p`.
   - `colHelp("sad", x, d)`: blue, add `"sad"`. No children.
   - `colHelp("sam", x, m)`: blue, add `"sam"`. Then `colHelp("same", x, e)`: blue, add `"same"`.
   - `colHelp("sap", x, p)`: blue, add `"sap"`.
4. Return `["sad", "sam", "same", "sap"]`.

**Why this beats the warm-up double loop.** The double loop touches *every* word in the dictionary and does character comparisons on each. This algorithm touches only the nodes in α's subtree, which contains *exactly and only* the keys with that prefix. The work is O(length of prefix) to find α, plus work proportional to the size of the answer. Words like `"intent"` that merely look similar to `"enter"` are never examined, because they diverge at the very first character and live in a completely different subtree.

### Example 7: Mutable key breaks a hash set (lecture's trace)

```java
List<Integer> items = new ArrayList<>();
items.add(0);
items.add(1);
// items.hashCode() == 962 (say); 962 % 4 == 2  -> bucket 2

Set<List<Integer>> s = new HashSet<>();
s.add(items);          // lands in bucket 2

items.add(7);          // MUTATION: items is now [0, 1, 7]
// items.hashCode() is now, say, 2929829; 2929829 % 4 == 1

s.contains(items);     // computes hash -> bucket 1 -> empty -> false (!)
```

**What went wrong.** The object was *filed* under bucket 2 using its old hash code. The lookup computes the *new* hash code and searches bucket 1. The object is physically still in the table, in bucket 2, but it is unreachable by lookup. The data structure's correctness depended on the key's hash code never changing, and mutation broke that. `String` is immutable precisely so it is safe as a hash key.

---

## Common Pitfalls

1. **Forgetting `isKey` entirely.** Without it, `contains("sa")` returns `true` for the trie built from `{"sam","sad","sap","same"}`. Path existence is *not* key membership. Every trie exam problem hinges on this.

2. **Thinking a white node means no keys below it, or that a blue node means nothing below it.** Both are wrong. `m` is blue (`"sam"`) and has a blue descendant `e` (`"same"`). The `a` under `s` is white but has four keys beneath it. The flag describes only the node's own path string.

3. **Thinking the invisible root stores a character.** It does not. It is a pure dispatch point so that multiple distinct first characters can coexist. There is no character at the root and `root.isKey` corresponds to the empty string. *(extra context: whether you allow the empty string as a key is an implementation decision.)*

4. **Saying trie operations are Θ(log N) or Θ(N).** Scattered answers on this in lecture. Trie runtime is driven by key length L, not key count N, so it is Θ(1) in N. Confusing the two parameters is the single most common error.

5. **Writing Θ(L) for `contains`.** It is **O(L)**, because falling off the tree ends the search early. `add` is **Θ(L)** because it always walks the full key.

6. **Claiming `DataIndexedCharMap` nodes are asymptotically wasteful.** 128 is a constant, so per-node space is O(1). It is *practically* wasteful (127 of 128 links null), not asymptotically catastrophic. Say "wasteful in practice, constant asymptotically."

7. **Mixing up R and the child count C.** `DataIndexedCharMap` stores R links per node regardless of how many children exist. BST and hash table store C links, where C is the actual number of children. Also note the slides' explicit caveat that cost *per link* is higher for BST and hash table.

8. **Getting the BST vs hash table child-lookup costs backwards.** The slides say BST is O(log R) and hash table is O(R) (worst case, when everything collides into one bucket). Since R is a fixed constant, all three are Θ(1) in practice.

9. **Omitting the `alpha.isKey` check in `keysWithPrefix`.** If you call `keysWithPrefix("a")` on our trie, `"a"` itself is a key, and the children loop alone would never emit it.

10. **Breaking the string-accumulator invariant in `colHelp`.** Passing `s` instead of `s + c`, or adding `c` at the wrong point, desynchronizes the accumulated string from the node, and you report wrong keys. The accumulated string must always equal the root-to-`n` path.

11. **Doing naive autocomplete.** Calling `keysWithPrefix("a")` on a billion-string trie to keep 10 results is "extremely inefficient." You need the best-value-in-subtree augmentation plus a priority queue, with pruning when the current top-k all beat the best remaining candidate.

12. **Overriding `equals` without `hashCode`** (or the reverse). They must stay synchronized: `equals` says which things belong in the same bucket, `hashCode` says which bucket they go to.

13. **Believing `final` makes an object immutable.** `final` on a field prevents reassigning the field. It does not prevent mutating the object the field points at (array contents, list elements), and it does not prevent an outside party who captured the reference earlier from mutating it.

14. **Pronouncing/spelling.** "Trie" (pronounced "try"), short for **Retrieval Tree**. Fredkin wanted "tree"; nobody complied.

---

## Likely Exam Points

### 1. Build a trie from a list of strings, mark the key nodes

Classic. Expect to draw a trie and to be asked which nodes are "blue."

**Q:** Insert `"cat"`, `"car"`, `"card"`, `"cab"`, `"do"` into an initially empty trie. How many nodes does it contain (excluding the root), and which of them have `isKey == true`?

**A:** Nodes: `c, a, t, r, d, b` for the `cat/car/card/cab` branch and `d, o` for `"do"`. That is `c`, `a`, `t`, `r`, `d`, `b`, `d`, `o` = **8 nodes**. The `c -> a` prefix is shared by four keys; `c -> a -> r` is shared by `"car"` and `"card"`. The key nodes are: the `t` (for `"cat"`), the `r` (for `"car"`), the `d` under `r` (for `"card"`), the `b` (for `"cab"`), and the `o` (for `"do"`) = **5 key nodes**, which equals the number of inserted keys, as it must.

### 2. Hit vs miss classification

**Q:** For the trie holding `{"sam","sad","sap","same","a","awls"}`, classify each of `contains("awl")`, `contains("same")`, `contains("sax")`, `contains("awls")` as a hit, a white-node miss, or a fell-off-the-tree miss.

**A:** `contains("awl")`: white-node miss (the `l` node exists but `isKey` is false). `contains("same")`: hit. `contains("sax")`: fell-off-the-tree miss (the `a` under `s` has children `d`, `m`, `p` but no `x`). `contains("awls")`: hit.

### 3. Runtime of `add` and `contains`, in N and in L

**Q:** A trie contains N string keys drawn from an alphabet of size R. Give the runtime of `contains(key)` in terms of N, and in terms of L = `key.length()`. Why do these differ in form?

**A:** In terms of N: **Θ(1)**, because the work depends only on the characters of `key`, not on how many keys are stored. In terms of L: **O(L)**, one Θ(1) child lookup per character, with early termination if we fall off the tree. `add` is **Θ(L)** (always walks all L characters, no early exit) and **Θ(1)** in N. The two forms differ because they parameterize different things: the trie's cost is a function of the key, and treating L as constant collapses it to Θ(1) in N.

### 4. Child-tracking strategy tradeoffs

**Q:** Name the three child-tracking strategies from lecture. For each, state the number of links stored per node and the child-lookup cost. Which is most memory efficient, and which is "most natural"?

**A:**

| Strategy | links/node | lookup |
|---|---|---|
| `DataIndexedCharMap` | R (e.g. 128) | Θ(1) |
| Balanced BST | C (number of children) | O(log R) |
| Hash Table | C (number of children) | O(R) |

BST and hash table are far more memory efficient since real nodes have few children, though cost per link is higher. Since R is fixed, all three lookups are effectively Θ(1). The slides call the **hash table** the most natural choice, and note you would run computational experiments to choose for a specific application.

### 5. Write or complete `collect` / `keysWithPrefix`

**Q:** Why does `colHelp` take the accumulated string `s` as a parameter rather than reconstructing it from the node? What goes wrong if the recursive call passes `s` instead of `s + c`?

**A:** Nodes have no parent pointers, so a node cannot reconstruct its own root-path. Threading `s` down maintains the invariant "`s` is the string spelled by the root-to-`n` path," which is exactly what makes `n.isKey -> x.add(s)` correct. If you pass `s` unchanged, the invariant breaks: at depth 3 under the `a` branch you would still hold `"a"`, so `colHelp` would add `"a"` once per blue descendant and never produce `"awls"`.

### 6. `longestPrefixOf` and `keysWithPrefix` outputs

**Q:** For the trie holding `{"sam","sad","sap","same","a","awls"}`, give `longestPrefixOf("sample")`, `keysWithPrefix("sa")`, and `keysWithPrefix("aw")`.

**A:** `longestPrefixOf("sample")` = `"sam"`: we walk `s-a-m` (`"sam"` is a key, remember it), then `p`, at which point we fall off, so the last key seen wins. `keysWithPrefix("sa")` = `["sad","sam","same","sap"]`. `keysWithPrefix("aw")` = `["awls"]` (the `w` node is white, so `"aw"` itself is not included).

### 7. Why tries beat hash tables and BSTs for prefix queries

**Q:** You need to find all dictionary words beginning with `"ente"`. Explain in one or two sentences why a hash set is a poor choice and a trie is a good one.

**A:** A hash set scatters whole strings into buckets based on their full hash code, so strings with a shared prefix have no spatial relationship in the table; you must scan all N words and compare each against the prefix. A trie stores the prefix once as a path, and *all* keys with that prefix live in exactly one subtree, so the query costs O(prefix length) to locate the subtree plus work proportional to the number of answers.

### 8. Efficient autocomplete (extra topic, lower probability)

**Q:** What is the flaw in "call `keysWithPrefix(p)` and return the 10 highest-valued results," and how does the lecture's improvement fix it?

**A:** For a short prefix, `keysWithPrefix` collects potentially billions of results just to keep 10. The fix: store at each node both its own value and the **best value in its subtree**, then explore nodes in decreasing order of "best" using a priority queue, stopping as soon as the current top 10 all exceed the best remaining candidate's value. This prunes entire subtrees (e.g. exploring the `sp` subtree with best 20 before the `sm` subtree with best 5). A further refinement is to merge single-child chains, giving a **radix trie**.

### 9. The redundant `ch` field

**Q:** The initial `Node` class stores `private char ch`. Why is this field redundant, and under what condition does that argument hold?

**A:** A child sits in the slot of its parent's child map keyed by its own character, so the character is already determined by *where* the node is stored: the node at index 97 of a parent's `DataIndexedCharMap` must be the `'a'` child. The argument holds as long as you always reach a node through a parent's character-keyed map; it fails if you need a node to report its own letter without that context.

### 10. Mutability and hashing (from the first part of lecture)

**Q:** Explain why mutable objects make poor hash table keys, using a concrete sequence of operations.

**A:** The bucket index is computed from the key's hash code at insertion time. If the key then mutates, its hash code changes, so a later lookup computes a different bucket and searches the wrong place. Concretely: `List l = [0,1]` with `hashCode()` 962 goes to bucket `962 % 4 = 2`. After `l.add(7)`, `hashCode()` becomes 2929829 and `2929829 % 4 = 1`, so `contains(l)` inspects bucket 1, finds it empty, and returns `false` even though `l` is sitting in bucket 2. Also relevant: if you override `equals`, you must override `hashCode` to match, since `equals` determines which objects *should* share a bucket while `hashCode` determines which bucket they *do* land in.

### 11. Where the trie fits in the ADT/implementation picture

**Q:** Is a trie an ADT or an implementation? What ADT(s) does it implement, and what restriction does it impose?

**A:** An **implementation**, not an ADT. It implements **Set** and **Map**, like a BST or hash table does. The restriction is on the key type: keys must be **strings** (more generally, sequences over a fixed alphabet). In exchange for that restriction we get unconditional constant-time `add`/`contains` in N and efficient prefix operations.

---

## Summary

- **Theme:** balanced BSTs and hash tables work on arbitrary keys; if you know something extra about your keys you can sometimes do better. This lecture exploits "keys are characters" and "keys are strings."
- **`DataIndexedCharMap`:** keys are single ASCII chars, so use the char's own code as an array index into an array of size R (128 for ASCII). Θ(1) `get` and `put`, no hashing needed, no collisions possible.
- **Trie** = **Retrieval Tree**, pronounced "try" (Fredkin wanted "tree"). For string keys: every node stores **one letter**, nodes are **shared** between keys with common prefixes, and the root is an empty dispatch node.
- **`isKey` (blue vs white)** distinguishes "a key ends here" from "just a waypoint." Without it you cannot tell `"sa"` (not a key) from `"sam"` (a key).
- **Two ways to miss:** final node is white, or you fall off the tree (no child link for the next character).
- **Runtimes:** independent of N. `add` is **Θ(L)**, `contains` is **O(L)**, where L is key length. Treating L as constant: both **Θ(1)**, with no even-spread or hash-quality caveats (unlike hash tables).
- **Three child-tracking strategies:** `DataIndexedCharMap` (R = 128 links/node, Θ(1), memory hungry), **Hash Table** (C links/node, O(R) worst case, most natural choice), **Balanced BST** (C links/node, O(log R), slightly slower). Cost per link is higher for the latter two. R is fixed, so all three are effectively Θ(1).
- The `ch` field in `Node` is **redundant** and can be dropped, since position in the parent's child map already encodes the character.
- **Tries can be maps**, not just sets: store a value at each node instead of a boolean.
- **The real payoff is string operations:** `longestPrefixOf("sample")` = `"sam"`; `keysWithPrefix("sa")` = `["sad","sam","same","sap"]`. All keys with a given prefix live in one subtree, which no BST or hash table gives you.
- **`collect` / `colHelp`** recursion threads the accumulated root-to-node string as a parameter, maintaining the invariant that `s` spells the path to `n`, so `n.isKey` means `s` is a key. `keysWithPrefix` is `collect` started at the prefix node with the prefix as the seed string.
- **Autocomplete (extra):** naive `keysWithPrefix` collects billions to keep 10. Fix: store best-subtree-value per node, explore in order of "best" with a priority queue, prune when the top-k beat the best remaining. Merging single-child chains yields a **radix trie**.
- **Other domain-specific string structures** exist (Suffix Trees, DAWGs) but are out of scope.
- **Hashing wrap-up:** `hashCode` and `equals` must stay synchronized; mutable keys break hash tables because the bucket index was computed from the old hash code; `final` prevents field reassignment, not deep mutation; Java's real `HashMap` upgrades overfull buckets from linked lists to red-black trees.
- **Bottom line:** data structures interact in beautiful and important ways.
