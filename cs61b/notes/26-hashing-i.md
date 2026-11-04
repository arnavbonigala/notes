<!-- Mon, Oct 26, 2026 | sources: slides + YouTube auto-transcript + textbook -->
# Lecture 26: Hashing I

## Overview

Up to this point, every Set/Map implementation we have built (ArraySet, BST, 2-3 tree, LLRB) has required items to be **comparable**, and the best of them topped out at Θ(log N) for `contains` and `add`. This lecture asks whether we can do better on both counts, and the answer is yes. The idea is to stop storing items in one long list and instead split them across M "buckets," using a cheap arithmetic rule (modulus) to compute which bucket an item belongs to. If items spread out evenly across buckets, each bucket holds about N/M items, giving an M-fold speedup; if we then let M grow with N (resizing when the load factor N/M exceeds a constant like 1.5), each bucket holds a constant number of items and operations become Θ(1) on average. To handle non-integer data such as Strings, we first "integerize" it by treating the characters as digits of a number in some base (base 26 for lowercase English, base 126 for ASCII, larger for Unicode), which overflows Java's fixed-precision `int` and therefore produces a value in a *fixed* finite range: that is exactly what makes it a **hash code**. The resulting data structure, an array of linked lists indexed by a reduced hash code, is a **hash table** with **separate chaining**, which is how `java.util.HashMap` and `HashSet` work and which relies on every Java object having a `.hashCode()` method.

---

## Key Concepts

### 1. Why search trees are not the end of the story

Our tree-based sets have two limitations:

- **They require comparability.** A BST needs to answer "is X < Y?" at every node. Not all types have a natural order, and inventing one is sometimes artificial.
- **Θ(log N), while excellent, may not be optimal.** The lecture is explicit that log N is already very good (1 billion items means height ~30, and log is exponentially better than linear), but constant time is better still, and "there is nothing better than constant time."

Hashing addresses both: it needs only a way to turn an object into an integer, not a way to order objects.

### 2. From one list to many lists (separate chaining)

Start with an unordered `LinkedListSet<Integer>`. Both `contains` and `add` are Θ(N) because you must scan from the front. We previously improved this two ways, both of which required **sorted order**: express lanes (skip lists, out of scope) and tree-shaped restructuring (BSTs and relatives).

The new idea needs no order at all. Split the single list into two: one for even items, one for odd. To check whether 7 is present, you only look in the odd list, so you have halved your work. Generalize: use 10 lists and send each integer to list `x % 10`, which is the same as "look at the last digit." The lecture's analogy is picking up a Snackpass order at TP Tea: you find your drink by reading the last digit of your order number.

Each list is called a **bucket**. When two or more items land in the same bucket, that is a **collision**. This whole paradigm, where each bucket is a chain (linked list) of items, is called **separate chaining**.

Example from lecture, inserting `[3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15]` with M = 10:

| Bucket | Contents |
|---|---|
| 0 | 10 |
| 1 | 11 |
| 2 | 12 |
| 3 | 3, 13 |
| 4 | 4, 14 |
| 5 | 5, 15 |
| 6 | 6 |
| 7 | 7 |
| 8 | 8 |
| 9 | 9 |

Most buckets hold one item; buckets 3, 4, 5 have collisions.

### 3. Modulus as a reduction function

Taking `x % M` gives the remainder of dividing x by M. It is a **reduction** from the infinite set of all integers down to the finite index range `[0, M)`:

```
All Integers  --% 10-->  [0, 10)
```

The lecture stresses how cheap and total this is: hand the class a number like 1,973,200,529 and everyone can instantly say "bucket 9," because only the last digit matters. Modulus is described as "the most natural and best reduction function."

### 4. The central picture: large set to small set

The transcript's framing, carried over from the warm-up: a hash function takes a set with very many (possibly infinitely many) members and maps it into a set with a fixed, smaller number of members. The warm-up asked how to map n-bit strings into 2 bins (take the least significant bit, take the most significant bit, take the parity, or trivially map everything to 0, which is legal but useless) and then into 2^k bins (take the first k bits, the last k bits, etc.). More bins preserve more information and cause fewer collisions, but cost more space. That trade-off is the whole lecture in miniature.

The lecture also notes the **pigeonhole principle** (CS 70): mapping a large set into a smaller one makes collisions unavoidable. The goal is never zero collisions, only *few* collisions. (The transcript also mentions that hash functions appear in many other guises, e.g. cryptographic hashes, SHA-256, Git commit hashes, pairwise-independent hashes, all unified by this same "big set to small set" picture; those are out of scope here.)

### 5. ArrayOfListsSet: an M-fold speedup that is still Θ(N)

Generalizing to M buckets gives `ArrayOfListsSet`. With N items **evenly distributed**, each bucket holds about N/M items, so operations are up to M times faster. With M = 10, everything is 10x faster.

But asymptotically this is nothing: Θ(N/10) = Θ(N), because 10 is a constant and Θ ignores constants. We need a smarter choice of M.

### 6. Choosing M: make it depend on N

Two extremes:

- **M = 10 (fixed):** constant-factor speedup only, still Θ(N).
- **M = number of possible integers:** every integer gets its own bucket, so `contains` is a single array lookup, Θ(1). But Java has exactly 4,294,967,296 ints, so you would need an array of over 4 billion slots. Far too much memory. The lecture's analogy: if your apartment is nothing but labeled baskets, you have no room left for anything else.

The happy medium: **let M depend on N.** Specifically pick M = Nk for a constant k. Since `contains` runs in Θ(N/M), substituting gives Θ(N/(Nk)) = Θ(1/k), which is constant.

Reading the bound: larger k means more buckets and faster `contains`, matching intuition. But larger k also means more memory. So pick a k that is a reasonable compromise. (The transcript also fielded student suggestions of M = √N, M = log N, and M = N^(1/3); these are not constant-factor-of-N and would not give Θ(1), though they would beat a fixed M.)

### 7. The load factor and resizing

Rather than recompute M on every insertion, we do what we did for `ArrayList`: keep M fixed, and when it gets too crowded, double it. The crowding measure is the **load factor**:

```
load factor = N / M = items / buckets
```

The lecture's worked rule is k = 1.5: when N/M ≥ 1.5, double M. (Java's `HashMap` typically uses 0.75, mentioned in the Summary slide.)

Crucially, doubling M changes the modulus, so **every item must be rehashed and reinserted**. An item that was in bucket `x % 4` now belongs in bucket `x % 8`. The transcript's analogy: it is a spring cleaning, you are changing your whole organization system, so everything must be re-filed. This resize is a Θ(N) operation, but it happens rarely enough to amortize away.

### 8. Storing non-integers: integerization

Modulus is defined on integers, so to store Strings we first convert them to integers.

**Bad first idea:** use the alphabet position of the first letter ("cat" → 3). Problems raised in lecture:
- Terrible worst case: a dataset of words all starting with "a" dumps everything into one bucket.
- Only 26 buckets ever get used, so we can never grow M.
- English letter frequencies are very uneven, so bucket "a" would be huge and bucket "z" nearly empty.
- Some strings do not start with a letter at all, e.g. "1a2b3c".

**Better idea:** treat the whole string as a number in base 26, which uses every character.

Review of decimal first: base 10 has 10 digits, and longer numbers are sequences of digits weighted by powers of 10:

```
7091₁₀ = (7 × 10³) + (0 × 10²) + (9 × 10¹) + (1 × 10⁰)
```

Do the same with letters, mapping a = 1, b = 2, ..., z = 26:

```
cat₂₆ = (3 × 26²) + (1 × 26¹) + (20 × 26⁰) = 2074
bee₂₆ = (2 × 26²) + (5 × 26¹) + (5 × 26⁰) = 1487
```

One difference from decimal: in decimal, `000` equals `0`, but the strings `"aaa"` and `"a"` are different. We avoid this by **having no zero digit**: `a` is 1, not 0. As long as the base is ≥ 26, this gives every lowercase English word a **unique** number.

### 9. ASCII, Unicode, and bigger bases

Lowercase-only is too restrictive; we want "2pac" and "eGg!" too. ASCII assigns every character a value in 0-127, with 33-126 printable. `char c = 'D'` is equivalent to `char c = 68`; lowercase `'a'` is 97, `'b'` is 98 (so Java's character comparison is just comparing these codes), `'!'` is 33, and the digit character `'2'` is 50.

Using 126 as the base covers all printable English text:

```
bee₁₂₆  = (98 × 126²) + (101 × 126¹) + (101 × 126⁰) = 1,568,675
2pac₁₂₆ = (50 × 126³) + (112 × 126²) + (97 × 126¹) + (99 × 126⁰) = 101,809,233
eGg!₁₂₆ = (98 × 126³) + (71 × 126²) + (98 × 126¹) + (33 × 126⁰) = 203,178,213
```

Java `char` also supports **Unicode**, which covers far more: `'☂'` is 9730, `'鳌'` is 40140, `'혜'` is 54812. For Chinese characters the largest value is 40,959, so a unique encoding of all Chinese strings needs base 40959:

```
守门员₄₀₉₅₉ = (23432 × 40959²) + (38376 × 40959¹) + (21592 × 40959⁰) = 39,312,024,869,368
```

### 10. Integer overflow, and why it is fine

That number does not exist as a Java `int`. Java integers are **fixed precision**: there are exactly 4,294,967,296 of them, from -2,147,483,648 to 2,147,483,647. Going past the max **overflows** and wraps to the min: the number after 2,147,483,647 is -2,147,483,648. (Some languages, e.g. Python, support arbitrary precision instead, limited only by memory.)

Overflow strikes even short strings: `omens₁₂₆` = 28,196,917,171 mathematically, but `asciiToInt("omens")` returns -1,867,853,901 in Java.

So we **cannot** map every possible string to a unique integer. The lecture's point is that this is acceptable: we were going to reduce into M buckets anyway, so losing information is already part of the plan. Java's wraparound is just "a built-in bucket that gives you the modulus whether you like it or not." All we need is a map from a big set to a small set.

### 11. Hash codes and the hash table

The official name for the integer we compute is a **hash code**. Our integerization function qualifies as a hash code precisely because its target set is *fixed* in size (the 4,294,967,296 Java ints).

A DynamicArrayOfLists with a finite integerization function is a **hash table**:

1. `data` → **hash function** (`hashCode()`) → **hash code** in [-2147483648, 2147483647]
2. hash code → **reduce** (modulus / `Math.floorMod`) → **index** in [0, M)
3. store data in the bucket at that index

Example from the slides: `"抱抱".hashCode()` is 1034854400, and `1034854400 % 10 = 0`, so it goes in bucket 0 alongside "hug", "포옹", and "守门员".

Many other hash table flavors exist (linear probing, quadratic probing, cuckoo hashing, buckets that are not linked lists). In 61B we use separate chaining.

### 12. Hash tables in Java

Hash tables are the most popular Set/Map implementation: great performance in practice, no comparability requirement, relatively simple to implement. Python dictionaries are hash tables in disguise. Java provides `java.util.HashMap` and `java.util.HashSet`.

How does a `HashMap` get an object's hash code? **Not** via an interface like "implements Hashable." Instead, `hashCode()` is a method on `Object`, so *every* Java object has one:

```java
String toString()
boolean equals(Object obj)
int hashCode()          // this is where Java implements hash codes
Class<?> getClass()
protected Object clone()
protected void finalize()
void notify() / notifyAll() / wait(...)   // not discussed in 61B
```

The default `hashCode()` implementation uses the object's **memory address**.

Java's actual String `hashCode` uses **base 31**, not 126:

```java
public int hashCode(String s) {
    int intRep = 0;
    for (int i = 0; i < s.length(); i += 1) {
        intRep = intRep * 31;
        intRep = intRep + s.charAt(i);
    }
    return intRep;
}
```

Because of overflow, distinct strings can collide: both `"守门员".hashCode()` and `"孡㜹ቘ஝஝俎".hashCode()` return 23,729,400. Real examples printed in lecture:

```
"a"                 →  97
"bee"               →  97410
"포옹"              →  1732557
"kamala lifefully"  →  1732557       // collision!
"đậu hũ"            →  -2108180664
```

### 13. Negative hash codes and `Math.floorMod`

Hash codes can be negative, as `"đậu hũ"` shows. Conceptually, a hash code of -1 with M = 4 should wrap around to index 3, like negative indexing in Python. But Java's `%` does not do that:

```java
System.out.println(-1 % 4);                  // -1   → ArrayIndexOutOfBoundsException!
System.out.println(Math.floorMod(-1, 4));    // 3    → correct
```

So the reduce step in Java must use `Math.floorMod(hashCode, M)`, not `%`. This is the "caveat" the slides promised to revisit.

### 14. Performance

**Hash table with no resizing: Θ(N).** N changes but M stays constant, so buckets hold Θ(N/M) = Θ(N) items even with perfect distribution (treating M as a constant: if M = 4, Θ(N/4) = Θ(N)).

**Hash table with resizing: Θ(1) on average.** M grows with N, so buckets hold Θ(N/M) = Θ(1) items under even distribution.

Why "on average," and why the caveats?
- A **resize** costs Θ(N).
- An **uneven distribution** degrades to Θ(N): if every item lands in one bucket, you traverse a linked list of length N.

Breakdown of the Θ(N) resize cost:

| Step | Cost |
|---|---|
| Allocate double the buckets | Θ(M) |
| Iterate through M buckets | Θ(M) |
| Iterate through N items | Θ(N) |
| ...per item: recompute hash code and new index | Θ(1) |
| ...per item: re-insert into new buckets | Θ(1) (with head insertion) |

Since M = Θ(N): Θ(M) + Θ(M) + Θ(N)·(Θ(1) + Θ(1)) = **Θ(N)**.

The per-item Θ(1) re-insertion assumes insertion at a known position in the chain, e.g. **head insertion** into a sentinel-headed linked list: point the new node's `next` at the sentinel's old `next`, then point the sentinel at the new node. No traversal, so Θ(1). (The slides note that in practice Java uses a mix of tail insertion and red-black trees, which is Hashing II material, and that you may ignore the real-world details here.)

**Even distribution matters independently of load factor.** Two tables can both have load factor N/M = 0.75 and have wildly different performance:

```
Bad:                            Good:
0                               0  "Isabel"
1                               1
2                               2  "Ryan"
3  "Marcus" → "Ryan" → "Isabel" 3  "Marcus"
```

Left is Θ(N) for `contains`, right is Θ(1). How do we get an even distribution? Since `index = hashCode % buckets.length`, it depends on both the **hash code** and the **table size (number of buckets)**.

Summary table:

| Implementation | `contains(x)` | `add(x)` |
|---|---|---|
| Bushy BSTs (2-3, LLRB) | Θ(log N) | Θ(log N) |
| Separate chaining hash table, **no** resizing | Θ(N) † | Θ(N) † |
| Separate chaining hash table, **with** resizing | Θ(1) † | Θ(1) * † |

\* on average &nbsp;&nbsp; † assuming items are evenly spread

---

## Definitions

- **Bucket:** one slot of the hash table's backing array; in separate chaining it holds a list of zero or more items. "Buckets" and "the backing array" refer to the same thing.
- **Collision:** when two or more distinct items are assigned to the same bucket. Unavoidable in general, by the pigeonhole principle.
- **Separate chaining:** the hash table disambiguation strategy used in 61B, where each bucket is a separate chain (linked list) of items.
- **Reduction function:** the function mapping an integer (hash code) to a valid bucket index in [0, M). Modulus is the most natural and best choice; in Java use `Math.floorMod`.
- **Integerization function:** a function converting arbitrary data into an integer representation, e.g. `lowerCaseToInt`, `englishToInt`, `asciiToInt`.
- **Hash code:** an integer produced by projecting a value from a set with many (or infinitely many) members into a set with a fixed, smaller number of members. In Java, hash codes are `int`s, so they lie in [-2,147,483,648, 2,147,483,647].
- **Hash function:** the function that computes a hash code from data; in Java, the `hashCode()` method, which every `Object` has.
- **Hash table:** a DynamicArrayOfLists combined with a finite integerization function. Data is hashed to a hash code, the hash code is reduced to an index, and the data is stored in the bucket at that index.
- **N:** the number of items in the hash table.
- **M:** the number of buckets in the hash table.
- **Load factor:** N/M, items divided by buckets. The table resizes when this exceeds a chosen constant (1.5 in the lecture's running example, 0.75 in the Summary slide and in Java's `HashMap`).
- **Integer overflow:** in a fixed-precision integer type, exceeding the maximum value wraps around to the minimum value. In Java, `2147483647 + 1 == -2147483648`.
- **Fixed precision:** an integer representation using a fixed number of bits, hence a finite set of representable values (contrast with arbitrary precision, as in Python, where the limit is memory).
- **Hash set:** an implementation of the Set ADT using a hash table as its engine.
- **Hash map:** an implementation of the Map ADT using a hash table as its engine.
- **Open addressing (extra):** an alternative to separate chaining where the set is stored as a plain array of items and a collision is resolved by probing for a different index (linear probing: scan forward one at a time; quadratic probing: try +4, +9, +16, ...).

---

## Worked Examples

### Example 1: Resizing walkthrough (the lecture's own example, k = 1.5)

Rule: when N/M ≥ 1.5, double M. Start with M = 4, N = 0.

```
add(7):   7 % 4 = 3
          N = 1, M = 4, N/M = 0.25
          bucket 0:,      bucket 1:,      bucket 2:,      bucket 3: 7

add(16):  16 % 4 = 0
          N = 2, M = 4, N/M = 0.5
          bucket 0: 16     bucket 1:,      bucket 2:,      bucket 3: 7

add(3):   3 % 4 = 3                                    (collision with 7)
          N = 3, M = 4, N/M = 0.75
          bucket 0: 16     bucket 1:,      bucket 2:,      bucket 3: 7, 3

add(11):  11 % 4 = 3     (4 goes into 11 twice → 8, remainder 3)
          N = 4, M = 4, N/M = 1.0
          bucket 0: 16     bucket 1:,      bucket 2:,      bucket 3: 7, 3, 11

add(20):  20 % 4 = 0
          N = 5, M = 4, N/M = 1.25
          bucket 0: 16, 20 bucket 1:,      bucket 2:,      bucket 3: 7, 3, 11

add(13):  13 % 4 = 1
          N = 6, M = 4, N/M = 1.5  ← threshold reached, RESIZE
          bucket 0: 16, 20 bucket 1: 13     bucket 2:,      bucket 3: 7, 3, 11
```

Now double M from 4 to 8 and **rehash every item** against the new modulus:

- 16 % 8 = 0
- 20 % 8 = 4
- 13 % 8 = 5
- 7 % 8 = 7
- 3 % 8 = 3
- 11 % 8 = 3 (8 goes into 11 once, remainder 3)

```
N = 6, M = 8, N/M = 0.75
bucket 0: 16
bucket 1: , 
bucket 2: , 
bucket 3: 3, 11
bucket 4: 20
bucket 5: 13
bucket 6: , 
bucket 7: 7
```

The load factor dropped back to 0.75, and the longest chain shrank from 3 to 2. The common wrong answer, flagged in the transcript as splitting the room, is putting 11 in bucket 7 because the array's last index is 7; the modulus is the **array length** (8), not the largest index, and `11 % 8 = 3`.

### Example 2: `englishToInt`, base 26 integerization

```java
/** Converts ith character of String to a letter number.
  * e.g. 'a' -> 1, 'b' -> 2, 'z' -> 26 */
public static int letterNum(String s, int i) {
    int ithChar = s.charAt(i);
    if ((ithChar < 'a') || (ithChar > 'z'))
        { throw new IllegalArgumentException(); }
    return ithChar - 'a' + 1;
}

public static int englishToInt(String s) {
    int intRep = 0;
    for (int i = 0; i < s.length(); i += 1) {
        intRep = intRep * 26;
        intRep = intRep + letterNum(s, i);
    }
    return intRep;
}
```

**What it does, line by line.** `letterNum` reads character `i` into an `int`. Because a `char` holds its Unicode/ASCII code, `s.charAt(i)` widens silently to that number: `'c'` becomes 99. The guard rejects anything outside `'a'`..`'z'` (i.e. 97..122). Then `ithChar - 'a' + 1` is 99 - 97 + 1 = 3, converting the raw code into the 1-indexed alphabet position. The `+ 1` is what removes zero from our digit set so that `"a"` and `"aa"` differ.

`englishToInt` is Horner's method. Each iteration does two things: `intRep * 26` shifts every digit already accumulated one place left in base 26, and `+ letterNum(s, i)` drops the new digit into the ones place.

**Trace of `englishToInt("cat")`:**

| i | char | letterNum | `intRep * 26` | `intRep` after add |
|---|---|---|---|---|
| start | | | | 0 |
| 0 | 'c' | 3 | 0 × 26 = 0 | 0 + 3 = **3** |
| 1 | 'a' | 1 | 3 × 26 = 78 | 78 + 1 = **79** |
| 2 | 't' | 20 | 79 × 26 = 2054 | 2054 + 20 = **2074** |

Result 2074, matching `(3 × 26²) + (1 × 26¹) + (20 × 26⁰)`. The multiply-then-add loop is exactly the expanded polynomial: `c` was multiplied by 26 twice (once per subsequent character), `a` once, `t` zero times.

Lecture's expected outputs: `a` → 1, `z` → 26, `aa` → 27, `bee` → 1487, `cat` → 2074. (`dog` and `potato` were left as exercises.)

### Example 3: `asciiToInt`, base 126, and overflow

```java
public static int asciiToInt(String s) {
    int intRep = 0;
    for (int i = 0; i < s.length(); i += 1) {
        intRep = intRep * 126;
        intRep = intRep + s.charAt(i);
    }
    return intRep;
}
```

Structurally identical to `englishToInt`, with two changes: base 126 instead of 26, and it uses the raw `s.charAt(i)` code as the digit rather than remapping to 1-26. That means it accepts any character, so `"2pac"` and `"eGg!"` work, and it no longer needs the `letterNum` guard.

**Why it overflows.** `"omens"` as a base-126 number is 28,196,917,171, far beyond `Integer.MAX_VALUE` = 2,147,483,647. Each `intRep * 126` silently wraps, so the function returns -1,867,853,901. The value is a perfectly usable hash code; it is just not unique and not the mathematical value.

### Example 4: Negative hash code reduction

```java
public class ModTest {
  public static void main(String[] args) {
    System.out.println(-1 % 4);                 // prints -1
    System.out.println(Math.floorMod(-1, 4));   // prints 3
  }
}
```

Suppose an item's hash code is -1 and M = 4. The conceptual answer is bucket 3 (wrapping backward from the end). Java's `%` returns -1, which as an array index throws an exception. `Math.floorMod(-1, 4)` returns 3. Concretely, for `"đậu hũ"` with hash code -2108180664, `Math.floorMod(-2108180664, 4)` gives index 0.

### Example 5: Java's real String `hashCode` (cached, base 31) (slide material, under "extra")

```java
@Override
public int hashCode() {
    int h = cachedHashValue;
    if (h == 0 && this.length() > 0) {
        for (int i = 0; i < this.length(); i++) {
            h = 31 * h + this.charAt(i);
        }
        cachedHashValue = h;
    }
    return h;
}
```

Two differences from our `asciiToInt`. First, base **31** rather than 126. Second, the result is **cached** in an instance field, so repeated `hashCode()` calls on the same String are Θ(1) after the first. The `h == 0 && length() > 0` condition means "not yet computed," which is why an empty string short-circuits to 0.

### Example 6: Hashing a collection (extra slide material)

```java
@Override
public int hashCode() {
    int hashCode = 1;
    for (Object o : this) {
        hashCode = hashCode * 31;      // elevate/smear the current hash code
        hashCode = hashCode + o.hashCode();   // add new item's hash code
    }
    return hashCode;
}
```

A List is like a String: a collection of items each with its own hash code. Same Horner pattern, with `o.hashCode()` in place of a character code. Starting at 1 rather than 0 distinguishes some empty/leading cases. To save time, you can hash only the first few items: more collisions, but still correct.

### Example 7: Hashing a recursive data structure (extra slide material)

```java
@Override
public int hashCode() {
    if (this.value == null) {
        return 0;
    }
    return this.value.hashCode() +
    31 * this.left.hashCode() +
    31 * 31 * this.right.hashCode();
}
```

For a binary tree with sentinel leaves, the base case is a null-valued sentinel returning 0. Otherwise combine this node's value hash with the recursively computed left and right subtree hashes, weighted by different powers of 31 so that swapping the left and right subtrees produces a different hash code.

---

## Common Pitfalls

1. **Using `%` instead of `Math.floorMod` on hash codes.** Java's `%` returns a negative result for negative operands, and hash codes are frequently negative. `-1 % 4 == -1` produces an index error.

2. **Reducing by the largest index instead of the array length.** After doubling M from 4 to 8, the modulus is 8, not 7. `11 % 8 = 3`, not 7. This is the exact point where the lecture poll split the room.

3. **Forgetting to rehash on resize.** Doubling the array is not enough. Every item's index depends on M, so every item must be rehashed and reinserted into the new backing array.

4. **Thinking a constant number of buckets gives constant time.** Θ(N/10) = Θ(N). A fixed M buys only a constant-factor speedup. You need M to grow with N.

5. **Thinking a good load factor alone guarantees Θ(1).** Two tables with identical load factor 0.75 can be Θ(N) and Θ(1) respectively. The Θ(1) bound also requires **even distribution**, which depends on the quality of the hash code.

6. **Expecting unique hash codes.** By the pigeonhole principle plus integer overflow, collisions are mathematically unavoidable: `"포옹"` and `"kamala lifefully"` both hash to 1732557, and `"守门员"` collides with `"孡㜹ቘ஝஝俎"`.

7. **Storing mutable objects in a `HashSet`/`HashMap`.** (Warning #1 from lecture.) If an object's instance variables change, its `hashCode` changes, so it now belongs in a different bucket than the one it is sitting in, and the item gets lost. Never make a `HashSet<List<Integer>>`.

8. **Overriding `equals` without overriding `hashCode`.** (Warning #2 from lecture.) HashMaps and HashSets use `hashCode` to find the bucket and `equals` to check whether an item is already in that bucket. Overriding only one leads to lost items and generally weird behavior.

9. **Confusing hash set / hash map / hash table.** Per the FAQ: a hash set is a Set ADT implementation, a hash map is a Map ADT implementation, and a hash table is the underlying storage mechanism (M buckets holding N items) that both use.

10. **Assuming adversarial data behaves like random data.** The transcript is explicit: if an attacker knows your hashing scheme with M = 4, they can insert 0, 4, 8, 16, 20, 24, 28, ... and funnel everything into bucket 0, degrading you to Θ(N).

11. **Assuming a bigger base is always a better hash code.** (Extra, from the Creating a Good Hash Code section.) Base 126 is *worse* than base 31 because of overflow behavior; see the next section.

---

## Likely Exam Points

### 1. Trace a hash table through insertions and a resize

**Q.** A separate-chaining hash table starts with M = 4 and resizes (doubling M) when N/M ≥ 1.5. Items hash to themselves. Insert 7, 16, 3, 11, 20, 13 in that order. Show the final table.

**A.** After `add(13)`, N = 6 and N/M = 6/4 = 1.5, triggering a resize to M = 8. Rehash all six items mod 8: 16 → 0, 3 → 3, 11 → 3, 20 → 4, 13 → 5, 7 → 7.

```
0: 16
1:
2:
3: 3, 11
4: 20
5: 13
6:
7: 7
```
N = 6, M = 8, N/M = 0.75.

---

### 2. Runtime of hash table operations under stated assumptions

**Q.** Give the worst-case runtime of `contains(x)` for a separate-chaining hash table (a) with no resizing, (b) with resizing and even distribution, (c) with resizing but all items hashing to the same bucket.

**A.** (a) Θ(N): M is constant, so buckets hold Θ(N/M) = Θ(N) items. (b) Θ(1): M = Θ(N), so buckets hold Θ(N/M) = Θ(1) items. (c) Θ(N): one bucket is a linked list of length N, and traversing it is linear. Even distribution is a required assumption for the Θ(1) bound, and `add` is additionally only Θ(1) *on average* because resizes cost Θ(N).

---

### 3. Why resizing costs Θ(N)

**Q.** Break down the cost of a resize operation and justify the total.

**A.** Allocating the doubled bucket array is Θ(M); iterating the M old buckets is Θ(M); iterating the N items is Θ(N); per item, recomputing the hash code and new index is Θ(1), and re-inserting is Θ(1) assuming insertion at a known position such as head insertion into a sentinel-headed list. Total: Θ(M) + Θ(M) + Θ(N)·(Θ(1) + Θ(1)). Since M = Θ(N), this is **Θ(N)**.

---

### 4. Base-B integerization by hand

**Q.** Using the lecture's scheme (a = 1, ..., z = 26, base 26), convert `"bee"` to an integer.

**A.** `b` = 2, `e` = 5, `e` = 5, so `bee₂₆ = (2 × 26²) + (5 × 26¹) + (5 × 26⁰) = 1352 + 130 + 5 = 1487`.

---

### 5. Horner's-method trace

**Q.** Trace `englishToInt("bee")` and give `intRep` after each loop iteration.

**A.** Start `intRep = 0`. i = 0: `0 * 26 = 0`, `+ 2` → **2**. i = 1: `2 * 26 = 52`, `+ 5` → **57**. i = 2: `57 * 26 = 1482`, `+ 5` → **1487**. Return 1487.

---

### 6. Integer overflow

**Q.** What does `System.out.println(2147483647 + 1)` print in Java, and why does this matter for hashing?

**A.** It prints `-2147483648`. Java ints are fixed precision with exactly 4,294,967,296 values; exceeding the maximum wraps to the minimum. It matters because a base-126 (or larger) integerization of even a short string like `"omens"` exceeds the int range, so distinct strings can collide. This is acceptable: it is exactly what makes the function a hash code (a map onto a *fixed*-size set), and we reduce into M buckets anyway.

---

### 7. Negative hash codes

**Q.** An object's `hashCode()` returns -2108180664 and the table has M = 4 buckets. Which bucket does it go in, and what is the wrong way to compute this?

**A.** `Math.floorMod(-2108180664, 4) = 0`, so bucket 0. The wrong way is `-2108180664 % 4`, which in Java yields a nonpositive result (here 0, but for a hash code of -1 it yields -1), causing array index errors in general. Always use `Math.floorMod` for the reduce step.

---

### 8. Load factor is not sufficient for Θ(1)

**Q.** Table A has buckets `[ , , , (Marcus→Ryan→Isabel)]` and Table B has `[Isabel, , Ryan, Marcus]`. Both have M = 4, N = 3, load factor 0.75. Are their `contains` runtimes the same?

**A.** No. Table B is Θ(1) because every bucket holds a constant number of items. Table A is Θ(N) because `contains("Isabel")` must traverse a chain of length N. Equal load factors do not imply equal performance; even distribution is what matters, and that depends on the hash code and the table size (`index = hashCode % buckets.length`).

---

### 9. The two HashMap warnings

**Q.** Why should you never store mutable objects in a `HashSet`? Why must you override `hashCode` whenever you override `equals`?

**A.** If a stored object's variables change, its `hashCode` changes, so it is now in the wrong bucket; lookups compute the new bucket and never find it, so the item is effectively lost. Separately, hash tables use `hashCode` to pick the bucket and `equals` to test membership within that bucket; overriding `equals` so that two logically equal objects compare equal, without making them produce equal hash codes, sends them to different buckets and the `equals` check is never even reached.

---

### 10. Why hashing can beat Θ(log N) at all

**Q.** Our BST-based sets achieve Θ(log N). Name the two limitations of search-tree sets that hash tables address.

**A.** (i) Search trees require items to be **comparable** (you must be able to ask "is X < Y?"), which is not true of all types; hash tables need only an integer hash code. (ii) Search trees are Θ(log N); hash tables achieve Θ(1) average time by keeping the number of items per bucket constant.

---

### 11. Choosing M (extra, from the derivation)

**Q.** Why is setting M = N^(1/2) insufficient for Θ(1) operations, and what choice of M works?

**A.** `contains` runs in Θ(N/M). With M = √N that is Θ(N/√N) = Θ(√N), which grows with N. Setting M = Nk for a constant k gives Θ(N/(Nk)) = Θ(1/k) = Θ(1). Larger k means faster `contains` but more memory, so a moderate k is chosen (equivalently, resize whenever N/M exceeds a constant like 1.5 or 0.75).

---

### 12. Choosing a hash base (extra section)

**Q.** Why does Java's String `hashCode` use base 31 rather than base 126, even though base 126 gives unique values for all ASCII strings ignoring overflow?

**A.** Overflow makes base 126 catastrophic: 126³² = 126³³ = ... = 0 under int arithmetic, so all characters beyond the last 32 get multiplied by zero. Any two strings sharing their last 32 characters collide, e.g. `"geocronite is the best thing on the earth."`, `"flan is the best thing on the earth."`, `"treachery is the best thing on the earth."`, and `"Brazil is the best thing on the earth."` all hash to 634199182. A small **prime** base avoids this: being odd (never even) sidesteps that overflow collapse, it is less likely to have a bad relationship with the number of buckets, and being small keeps it cheap to compute. The lecture's hashbrowns analogy: you do not make hashbrowns by chopping a potato into neat predictable segments; multiplying by powers of a prime base scrambles the data together into a seemingly random integer, whereas just adding characters is not random enough (`"ab"` collides with `"ba"`).

---

### 13. Bad hash code recognition (extra section)

**Q.** For each, say whether it is a good or bad `hashCode` and why: (a) `return 0;` (b) return the first character of the word; (c) add all the characters together; (d) treat the string as a base-B number.

**A.** (a) Bad: every item lands in one bucket, so the table degenerates to a single linked list, Θ(N). (b) Bad: only the first 26 buckets ever get used, English first letters are very unevenly distributed, and words starting with the same letter all collide. (c) Bad: `"ab"` and `"ba"` collide, since addition discards position information. (d) Good: position is preserved via powers of the base, so data gets scrambled across all buckets.

---

## Summary

- Search-tree sets require **comparable** items and give Θ(log N). Hashing requires only an integer hash code and gives Θ(1) average time.
- Split a single list into **M buckets** (separate chaining) and route each item with a **reduction function**, normally modulus. With even distribution, buckets hold ~N/M items, an M-fold speedup.
- A **fixed** M is not enough asymptotically: Θ(N/10) = Θ(N). Setting M = Nk for constant k gives Θ(1/k) = Θ(1).
- Implement this by resizing: when the **load factor** N/M exceeds a constant (1.5 in lecture, 0.75 typically), double M and **rehash every item**, since indices depend on M.
- To store non-integers, apply an **integerization function** first: treat the data as a number in some base. Base 26 for lowercase English (a = 1, not 0, so `"a"` ≠ `"aaa"`), base 126 for ASCII, base 40959 for Chinese characters.
- `cat₂₆ = (3 × 26²) + (1 × 26¹) + (20 × 26⁰) = 2074`; `bee₂₆ = 1487`. Horner's method (`intRep = intRep * B + digit`) computes this in one pass.
- Java ints are **fixed precision** (4,294,967,296 values, max 2,147,483,647, which overflows to -2,147,483,648), so unique integerization of all strings is impossible. That is fine: a map onto a fixed-size set is precisely a **hash code**.
- A **hash table** = DynamicArrayOfLists + finite integerization. Pipeline: `data → hashCode() → hash code → reduce → index → bucket`.
- Every Java `Object` has `hashCode()` (default: memory address); Java's String version uses base 31 and caches the result. Collisions are real: `"포옹"` and `"kamala lifefully"` both give 1732557.
- Hash codes can be negative, and Java's `%` preserves the sign, so reduce with **`Math.floorMod(hashCode, M)`**.
- Performance: no resizing → Θ(N); with resizing and even spread → Θ(1) `contains`, Θ(1) average `add`. Resizing costs Θ(N); an uneven distribution also degrades to Θ(N). Equal load factors do **not** imply equal performance.
- Two standing warnings: never store **mutable** objects in a `HashSet`/`HashMap`, and never override `equals` without also overriding `hashCode`.
- Vocabulary: a *hash set* implements the Set ADT, a *hash map* implements the Map ADT, and a *hash table* is the underlying M-buckets-holding-N-items mechanism.
- (Extra) Good hash codes use a **small prime** base to scramble data and avoid overflow collapse; alternatives to separate chaining include linear probing, quadratic probing, and cuckoo hashing, but 61B uses separate chaining.
