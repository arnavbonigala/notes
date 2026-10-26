<!-- Mon, Oct 26, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 26: Hashing I

This lecture builds the hash table from scratch, starting from a plain unordered linked-list set and asking how to beat the two limitations of search-tree sets: the requirement that items be `Comparable`, and the Θ(log N) ceiling on `contains` and `add`. The derivation proceeds in stages: split one list into several lists ("buckets") and use modulus to decide which bucket an item belongs to (`ArrayOfListsSet`, giving an M-fold speedup but still Θ(N)); then make the number of buckets M grow with the number of items N so that the average bucket length stays constant (`DynamicArrayOfListsSet`, giving Θ(1) average operations); then handle non-integer data by first converting it to an integer ("integerization"), treating a string as a number in some base; then confront the fact that Java `int`s are fixed-precision, so overflow makes uniqueness impossible and our integerization function becomes a *hash code*, a projection from a huge (possibly infinite) set onto a fixed finite set. The result is the hash table: `data → hashCode() → hash code → reduce (floorMod) → bucket index`. The lecture closes with Java's actual `String.hashCode()` (base 31, with overflow-induced collisions), `Math.floorMod` for negative hash codes, the two warnings (never store mutable objects in a `HashSet`/`HashMap`; never override `equals` without `hashCode`), and a careful accounting of performance including the Θ(N) cost of a resize.

## Key Concepts

### Why we want something other than a search tree

Every set/map implementation seen so far has a weakness:

| Implementation | `contains(x)` | `add(x)` | Notes |
|---|---|---|---|
| `ArraySet` | Θ(N) | Θ(N) | |
| BST | Θ(N) | Θ(N) | Random trees are Θ(log N) |
| 2-3 Tree | Θ(log N) | Θ(log N) | Beautiful idea, very hard to implement |
| LLRB | Θ(log N) | Θ(log N) | Maintains bijection with 2-3 tree, hard to implement |

Two complaints drive the lecture:

1. **Comparability.** Search trees need to ask "is X < Y?" That question is not meaningful for all types. Can we build a set that does not require items to be comparable?
2. **Speed.** Θ(log N) is already amazing (a billion items is height ~30), but can we do better?

The answer to both is yes.

### Multiple lists: the separate chaining idea

Start with a `LinkedListSet<Integer>` with no particular order: `contains` and `add` are Θ(N) because you must scan. Previous improvements (skip lists, BSTs and their cousins) both required *sorted* data. The new approach will not.

Key move: split the single list into two lists, one holding the even items and one holding the odd items. If items are evenly distributed, `add` and `contains` are now twice as fast, because you only scan the one list where the item could possibly be.

Generalize to 10 lists. Which list does an integer go in? The lecture's analogy: when you order from TP Tea on Snackpass, you find your drink by **looking at the last digit** of your order number, which is exactly **modulus by 10**.

This paradigm, where each slot in an array holds a list (a "chain") of items, is called **separate chaining**.

### Modulus as a reduction

Taking an integer mod M gives the remainder on division by M:
- `13 % 10 = 3`
- `7 % 10 = 7`
- `1000 % 10 = 0`

Think of this as a **reduction** from the space of all integers down to the integers in `[0, M)`. Modulus is described in the slides as the most natural and best reduction function.

With M = 10 and the data `[3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15]`, items land as:

```
0: 10
1: 11
2: 12
3: 3, 13
4: 4, 14
5: 5, 15
6: 6
7: 7
8: 8
9: 9
```

When a single bucket holds more than one item, we call that a **collision**. Collisions are not an error, they are just a slower bucket.

### `ArrayOfListsSet` and why it is still Θ(N)

Generalize to any number of buckets M. With N items **evenly distributed** (minimizing collisions), each bucket has length about N/M, giving up to an M-fold speedup: with M = 10, all operations are up to 10 times faster.

But asymptotically this is no win: Θ(N / 10) = Θ(N). A constant factor of 10 vanishes in the asymptotics.

### `DynamicArrayOfListsSet`: make M depend on N

The fix is to stop treating M as a constant. Let M be a constant multiple of N, i.e. M = Nk for a constant k. Since `contains` runs in Θ(N / M), substituting M = Nk gives:

$$\Theta(N/M) = \Theta(N/(Nk)) = \Theta(1/k)$$

which is constant. Larger k means faster `contains` (more buckets, shorter chains), which matches intuition, but larger k also means more memory. So we pick a k that is a happy medium.

Equivalently stated in terms of the **load factor** N/M: keep N/M below some constant, and when it exceeds that constant, **double M** and redistribute everything.

If N items are evenly distributed over M = Nk buckets, each list has length ~k, which is constant asymptotically, so operations are constant *on average*.

### Resizing, step by step (the lecture's k = 1.5 example)

Rule: when N/M ≥ 1.5, double M. Start with M = 4, N = 0.

| Operation | N | M | N/M | Table (bucket: contents) |
|---|---|---|---|---|
| start | 0 | 4 | 0 | all empty |
| `add(7)` | 1 | 4 | 0.25 | 3: 7 |
| `add(16)` | 2 | 4 | 0.5 | 0: 16 / 3: 7 |
| `add(3)` | 3 | 4 | 0.75 | 0: 16 / 3: 7, 3 |
| `add(11)` | 4 | 4 | 1.0 | 0: 16 / 3: 7, 3, 11 |
| `add(20)` | 5 | 4 | 1.25 | 0: 16, 20 / 3: 7, 3, 11 |
| `add(13)` | 6 | 4 | 1.5 | 0: 16, 20 / 1: 13 / 3: 7, 3, 11 → **resize** |

After doubling to M = 8, every item's index is recomputed with `% 8`:

```
0: 16        (16 % 8 = 0)
1:
2:
3: 3, 11     (3 % 8 = 3, 11 % 8 = 3)
4: 20        (20 % 8 = 4)
5: 13        (13 % 8 = 5)
6:
7: 7         (7 % 8 = 7)
```

N = 6, M = 8, N/M = 0.75. Notice that items **move**: 16 and 20 were both in bucket 0 under `% 4` but separate under `% 8`; 11 stays with 3. This is the crucial point: the bucket index is a function of both the item and the current table size, so resizing requires rehashing (or at least re-reducing) every item.

### Integerization: storing strings

Modulus is natural on integers. For other data we need a function `data → integer` first. The lecture calls this the **integerization function**.

**Bad first idea:** use the alphabet position of the first letter (a = 1, ..., z = 26), so "cat" → bucket 3. Problems the lecture lists:
- Terrible worst case: a dataset of words all starting with "a" puts everything in one bucket.
- Only the first 26 buckets ever get used.
- Some strings do not start with a letter at all, e.g. "1a2b3c".

**Better idea:** treat the string as a number in base 26 (assuming only lowercase English letters). Use *all* the digits by multiplying each by a power of 26:

- `cat` = (3 × 26²) + (1 × 26¹) + (20 × 26⁰) = 2074
- `bee` = (2 × 26²) + (5 × 26¹) + (5 × 26⁰) = 1487

This mirrors decimal: 7091 = (7 × 10³) + (0 × 10²) + (9 × 10¹) + (1 × 10⁰), using 10 digits 0 through 9. One difference: in decimal, `000` is the same as `0`, but as strings `aaa` differs from `a`. To deal with this we simply have no zero digit: `a` is 1, not 0.

**Uniqueness:** as long as the base is ≥ 26, this gives every lowercase English word a *unique* number. Using base 26, no other word maps to 1487.

### Beyond lowercase: ASCII and Unicode

Lowercase-only is too restrictive; we want "2pac" and "eGg!" too. Each character needs a number, and **ASCII** already provides one: every ASCII character gets a value from 0 to 127, with 33 through 126 being the printable ones. `char c = 'D'` is equivalent to `char c = 68`. (Fun fact from the slides: character comparison in Java uses these values, e.g. `'a'` = 97 < `'b'` = 98.)

The biggest printable ASCII value is 126, so base 126 gives unique values for all such strings:
- `bee` = (98 × 126²) + (101 × 126¹) + (101 × 126⁰) = 1,568,675
- `2pac` = (50 × 126³) + (112 × 126²) + (97 × 126¹) + (99 × 126⁰) = 101,809,233
- `eGg!` = (98 × 126³) + (71 × 126²) + (98 × 126¹) + (33 × 126⁰) = 203,178,213

Java `char`s go further: this encoding is **Unicode**, and the table is too big to list.
- `char c = '☂'` is equivalent to `char c = 9730`
- `char c = '鳌'` is equivalent to `char c = 40140`
- `char c = '혜'` is equivalent to `char c = 54812`

For Chinese characters the largest value is 40,959, so base 40,959 would be needed for uniqueness:
- `守门员` = (23432 × 40959²) + (38376 × 40959¹) + (21592 × 40959⁰) = 39,312,024,869,368

### Integer overflow kills uniqueness

Can we map every possible string to a unique integer? **No.** In Java there are only finitely many `int`s: exactly 4,294,967,296 of them. This is **fixed precision**. (Some languages such as Python support arbitrary precision, where the number of integers is limited by memory instead of by a fixed bit count.)

The number 39,312,024,869,368 does not exist as a Java `int`. The largest Java `int` is 2,147,483,647, and going past it **overflows**, wrapping around to the smallest, -2,147,483,648:

```java
int x = 2147483647;
System.out.println(x);      // 2147483647
System.out.println(x + 1);  // -2147483648
```

With base 126 we overflow on even short strings: `omens` in base 126 is 28,196,917,171, far above the maximum `int`, and `asciiToInt("omens")` actually returns -1,867,853,901 in Java.

So we cannot have uniqueness. We give it up.

### Hash codes

The official term for the number we compute is a **hash code**. Quoting Wolfram|Alpha via the slides, a hash code "projects a value from a set with many (or even an infinite number of) members to a value from a set with a fixed number of (fewer) members." Here the target set is the set of Java integers, of size 4,294,967,296. Our integerization function *is* a hash code precisely because the set it projects onto is fixed and finite.

A `DynamicArrayOfLists` with a finite integerization function is called a **hash table**. The pipeline:

```
data  --hash function (hashCode())-->  hash code  --reduce (floorMod)-->  index
```

for example `2348762878 % 10 = 8`. Java hash codes must lie in [-2,147,483,648, 2,147,483,647].

The version used in 61B is an array of lists, i.e. separate chaining, where each bucket is a separate chain of items. Many more exotic alternatives exist: linear probing, cuckoo hashing, using something other than linked lists for buckets, and so on.

### Hash tables in Java

Hash tables are the most popular implementation of sets and maps: great performance in practice, no need for items to be comparable, and implementations are often relatively simple. Python dictionaries are hash tables in disguise. In Java they are `java.util.HashMap` and `java.util.HashSet`.

How does a `HashMap` know an object's hash code? Not through an interface like `implements Hashable`. Instead, **every** object in Java has a `.hashCode()` method, because `Object` declares one:

- `String toString()`
- `boolean equals(Object obj)`
- `int hashCode()`  ← this is where Java implements hash codes
- `Class<?> getClass()`, `protected Object clone()`, `protected void finalize()`, `void notify()`, `void notifyAll()`, `void wait(...)` (the last several are not discussed or used in 61B)

The **default implementation of `hashCode` uses the memory address**.

Java's actual `String.hashCode()` uses **base 31** rather than 126 (cleaned-up version from the slides):

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

Because of overflow, distinct strings collide. Both `"守门员".hashCode()` and `"孡㜹ቘ஝஝俎".hashCode()` return 23,729,400. Other real values:

```java
System.out.println("a".hashCode());                 // 97
System.out.println("bee".hashCode());               // 97410
System.out.println("포옹".hashCode());               // 1732557
System.out.println("kamala lifefully".hashCode());  // 1732557
System.out.println("đậu hũ".hashCode());            // -2108180664
```

Note `"포옹"` and `"kamala lifefully"` collide outright. Also note `"bee".hashCode()` is 97410 in base 31, not the 1487 we computed in base 26 or the 1,568,675 we computed in base 126: the base and the character values both matter.

### Negative hash codes and `Math.floorMod`

Hash codes can be negative, so plain `%` is not safe for computing an index. Suppose an item's hash code is -1 and M = 4. Conceptually we want it to loop around to bucket 3, like negative indexing in Python. But in Java, `-1 % 4 = -1`, which would cause an index error. Use `Math.floorMod` instead:

```java
public class ModTest {
    public static void main(String[] args) {
        System.out.println(-1 % 4);               // -1
        System.out.println(Math.floorMod(-1, 4)); // 3
    }
}
```

So the Java pipeline is: data → `hashCode()` → hash code → `Math.floorMod(hashCode, buckets.length)` → index. For `"đậu hũ"`, `hashCode()` is -2,108,180,664 and `Math.floorMod(-2108180664, 4)` gives 0.

### Two important warnings

**Warning #1: never store objects that can change in a `HashSet` or `HashMap`.** Such objects are called *mutable*. You would never want a `HashSet<List<Integer>>`. If an object's variables change, its `hashCode` changes, so it reduces to a different bucket than the one it is sitting in, and items get lost.

**Warning #2: never override `equals` without also overriding `hashCode`.** This also leads to lost items and generally weird behavior. `HashMap` and `HashSet` use `equals` to decide whether an item already exists within a particular bucket, so the two methods must agree.

### Performance, carefully

**Hash table with no resizing: Θ(N).** N changes, M stays constant, so buckets contain Θ(N/M) items, and treating M as a constant this is Θ(N). The slides illustrate with four buckets each holding five names.

**Hash table with resizing: Θ(1) on average.** N changes and M changes with it, so buckets contain Θ(N/M) → Θ(1) items *if the distribution is even*.

Why "on average"? Two things can break it:
- A resize is needed: Θ(N).
- The distribution is not even: Θ(N) (one bucket could hold everything).

**What makes a resize Θ(N)?**

| Step | Cost |
|---|---|
| Double the number of buckets | Θ(M) |
| Iterate through M buckets | Θ(M) |
| Iterate through N items | Θ(N) |
| &nbsp;&nbsp;per item: recompute hash code and new index | Θ(1) |
| &nbsp;&nbsp;per item: re-insert into the new buckets | Θ(1)* |

\* Θ(1) assuming insertion at a known index of the linked list, such as head insertion.

Since M = Θ(N), total = Θ(M) + Θ(M) + Θ(N) × (Θ(1) + Θ(1)) = **Θ(N)**.

The head-insertion detail matters: to re-insert into a chain in Θ(1) you insert at the head, right after the sentinel, so you never traverse the chain. (The slides note that in reality Java uses a mix of tail insertion and red-black trees, which is the subject of Hashing II, and that the exact in-practice chaining mechanism is out of scope.)

**Even distribution is required, not optional.** Two tables can have the identical load factor N/M = 0.75 and wildly different performance: if all three items sit in one bucket, `contains(String x)` is Θ(N) because you traverse that linked list; if the three items sit in three different buckets, it is Θ(1). The load factor alone does not guarantee speed.

How do we get an even distribution? Both inputs to `index = hashcode % buckets.length` matter: the **hash code** and the **table size** (number of buckets).

### Final summary table from the slides

| | `contains(x)` | `add(x)` |
|---|---|---|
| Bushy BSTs | Θ(log N) | Θ(log N) |
| Separate chaining hash table, **no** resizing | Θ(N) | Θ(N)† |
| ... **with** resizing | Θ(1)† | Θ(1)\*† |

\* on average &nbsp;&nbsp; † assuming items are evenly spread

### Creating a good hash code (marked "extra" in the lecture)

Goal: a `hashCode` that spreads real data out nicely.
- **Example #1 (bad):** `return 0`. Everything collides.
- **Example #2 (bad):** return the first character, "cat" → 3. Terrible worst cases, few buckets used.
- **Example #3 (bad):** add the chars together. "ab" collides with "ba".
- **Example #4 (good):** treat the string as a base-B number.

The hashbrowns analogy: you do not make hashbrowns by chopping a potato into nice predictable segments. Adding up characters is not nearly "random" enough. Multiplying data by powers of some base scrambles everything together into a seemingly random integer. Writing a good `hashCode()` is tricky.

Java 8's `String.hashCode()` has two differences from our version:
1. It uses base **31** (a small base). Real hash codes do not care about uniqueness.
2. It **caches** the computed value so future `hashCode` calls are faster.

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

**Why 31 and not 126?** It seems like 126 is better, since ignoring overflow it gives unique representations for all ASCII strings. But overflow is a *particularly bad* problem for base 126. All of these return hash code 634199182:

- `"geocronite is the best thing on the earth."`
- `"flan is the best thing on the earth."`
- `"treachery is the best thing on the earth."`
- `"Brazil is the best thing on the earth."`

Any string ending in the same last 32 characters gets the same base-126 hash code, because 126³² = 126³³ = 126³⁴ = ... = 0 under overflow, so the upper characters are all multiplied by zero. (See CS61C for more.)

**Typical base: a small prime.**
- Prime: never even, which avoids the overflow-to-zero issue above, and lowers the chance of the resulting hash code having a bad relationship with the number of buckets.
- Small: cheaper to compute.

A full treatment of good hash codes is well beyond the scope of 61B.

**Hashing a collection.** Lists are a lot like strings, a collection of items each with its own `hashCode`:

```java
@Override
public int hashCode() {
    int hashCode = 1;
    for (Object o : this) {
        hashCode = hashCode * 31;        // elevate/smear the current hash code
        hashCode = hashCode + o.hashCode();  // add new item's hash code
    }
    return hashCode;
}
```

To save time, you may look at only the first few items: higher chance of collisions, but things still work.

**Hashing a recursive data structure** involves recursive computation, e.g. a binary tree with sentinel leaves:

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

### Open addressing / linear probing (marked "extra" in the lecture)

Instead of linked lists, an alternate and more exotic strategy is **open addressing**: the set is stored as an array of *items*, and the index tells you where to put the item. If the target location is already occupied, use a different location:
- **Linear probing:** use the next address, and if that is occupied, keep scanning one by one.
- **Quadratic probing:** try the next address, then 4 ahead, then 9 ahead, then 16 ahead, ...
- Many more possibilities exist (see the optional reading for the lecture, or CS170).

In 61B we use separate chaining with linked lists.

## Definitions

**Bucket:** one of the M slots of a hash table; in separate chaining, each bucket is a list that may hold any number of items.

**Separate chaining:** the hash table design in which the backing array holds M lists, each a separate "chain" of the items that reduced to that index. This is the version used in 61B.

**Collision:** the situation where two or more items land in the same bucket. Not an error, just a longer chain to scan.

**Reduction function:** a function mapping an integer into the valid index range [0, M). Modulus is the most natural and best reduction function; in Java use `Math.floorMod` so negative hash codes work.

**Integerization function:** a function converting arbitrary data into an integer representation, e.g. `lowerCaseToInt("cat") = 2074`.

**Hash code:** the integer produced by a hash function. Formally, it projects a value from a set with many (possibly infinitely many) members onto a value from a set with a fixed, smaller number of members. In Java, hash codes are `int`s, in [-2,147,483,648, 2,147,483,647], so there are exactly 4,294,967,296 possible hash codes.

**Hash function:** the function computing a hash code; in Java, the `hashCode()` method inherited from `Object`.

**Hash table:** a `DynamicArrayOfLists` with a finite integerization function. Data is converted by a hash function into a hash code, the hash code is reduced to a valid index, and the data is stored in the bucket at that index.

**Hash set:** an implementation of the Set ADT using a hash table as its engine (`java.util.HashSet`).

**Hash map:** an implementation of the Map ADT using a hash table as its engine (`java.util.HashMap`).

**Load factor:** N/M, i.e. items divided by buckets. The hash table resizes when the load factor exceeds some constant (usually 0.75 in the lecture's summary; the resizing walkthrough uses 1.5).

**Backing array:** the array of buckets. "Buckets are the same thing as the backing array."

**Fixed precision:** the property that a numeric type has a finite set of representable values, as Java's `int` does. Contrast arbitrary precision (e.g. Python), where the number of representable integers is limited by memory rather than by a fixed number of bits.

**Integer overflow:** what happens when an `int` computation exceeds 2,147,483,647: it wraps around to -2,147,483,648 and continues from there.

**ASCII:** the most basic character set used by most computers; each character is assigned a value between 0 and 127, with 33 through 126 printable.

**Unicode:** the larger character encoding Java `char`s support, covering other languages and symbols (e.g. `'☂'` = 9730).

**Mutable object:** an object whose variables can change. Must never be stored in a `HashSet` or `HashMap`.

**Open addressing (extra):** a non-chaining strategy where the table stores items directly and a collision is resolved by probing for a different index (linear probing, quadratic probing, cuckoo hashing, etc.).

## Worked Examples

### Example 1: `englishToInt`, base 26 integerization

The lecture's optional exercise is to write a function converting lowercase English strings to integers via powers of 26, with `a: 1`, `z: 26`, `aa: 27`, `bee: 1487`, `cat: 2074`.

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

**What `letterNum` does.** `s.charAt(i)` returns a `char`, which is assigned to an `int` variable `ithChar`. This is where the character silently becomes its numeric code point: `'c'` becomes 99. The guard rejects anything outside `'a'` through `'z'` by comparing code points (97 through 122). Then `ithChar - 'a' + 1` shifts the range so that `'a'` → 97 - 97 + 1 = 1 and `'z'` → 122 - 97 + 1 = 26.

**What `englishToInt` does: the loop is Horner's method.** Rather than computing powers of 26 explicitly, each iteration multiplies the accumulated value by 26 (shifting every previously-added digit one place left in base 26) and then adds the new digit. Tracing `englishToInt("cat")`:

| i | char | `letterNum` | `intRep` before | `intRep * 26` | `intRep` after |
|---|---|---|---|---|---|
| 0 | `'c'` | 3 | 0 | 0 | 3 |
| 1 | `'a'` | 1 | 3 | 78 | 79 |
| 2 | `'t'` | 20 | 79 | 2054 | 2074 |

Result 2074, matching (3 × 26²) + (1 × 26¹) + (20 × 26⁰). The reason this works: after processing "c" we hold 3; multiplying by 26 twice over the two remaining iterations turns that 3 into 3 × 26², exactly the place value it needs.

Tracing `englishToInt("bee")`: 0 → 2 → (2×26 + 5) = 57 → (57×26 + 5) = 1487. ✓

**Environment reasoning.** `intRep` and `ithChar` are local `int` variables, each a primitive box holding a value directly, not a reference. `s` is a reference to a `String` object on the heap; `charAt` does not copy the string, it just reads one character out of it. No new objects are created anywhere in either method except the exception on the error path, so this is pure arithmetic on primitives.

### Example 2: `asciiToInt`, base 126 integerization and overflow

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

Structurally identical to `englishToInt`, with two changes: the base is 126 instead of 26, and `s.charAt(i)` is used *directly* as the digit instead of being shifted into 1..26. The `char` is implicitly widened to `int`, so `'b'` contributes 98, not 2. This is what makes the function work on any string, including `"2pac"` and `"eGg!"`, since every character already has a numeric value.

Tracing `asciiToInt("bee")` (`'b'` = 98, `'e'` = 101):

| i | char value | `intRep * 126` | `intRep` after |
|---|---|---|---|
| 0 | 98 | 0 | 98 |
| 1 | 101 | 12,348 | 12,449 |
| 2 | 101 | 1,568,574 | 1,568,675 |

Result 1,568,675, matching (98 × 126²) + (101 × 126¹) + (101 × 126⁰). ✓

**Now the failure.** `asciiToInt("omens")`: mathematically this is 28,196,917,171, but Java's maximum `int` is 2,147,483,647. The multiplications overflow, and the function returns **-1,867,853,901** instead. Nothing in the code detects this; overflow in Java is silent wraparound. This is exactly why we must abandon uniqueness and accept hash codes: the base-126 "unique representation" is a mathematical fact about integers that Java's fixed-precision `int` cannot honor.

The pathology is specific and severe. Because 126 is even, repeated multiplication by 126 eventually drives the accumulated value's low bits all the way out of the 32-bit word: 126³² = 126³³ = 126³⁴ = ... = 0 in `int` arithmetic. Any character more than 32 positions from the end gets multiplied by zero and contributes nothing, which is why the four very different "... is the best thing on the earth." strings all hash to 634199182.

### Example 3: Java's real `String.hashCode()`

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

Same Horner loop, base 31. Tracing `"bee".hashCode()`:

| i | char value | `intRep * 31` | `intRep` after |
|---|---|---|---|
| 0 | 98 (`'b'`) | 0 | 98 |
| 1 | 101 (`'e'`) | 3,038 | 3,139 |
| 2 | 101 (`'e'`) | 97,309 | 97,410 |

Returns 97,410, which is exactly what `"bee".hashCode()` prints in Java. ✓

And `"a".hashCode()` is 0 × 31 + 97 = 97. ✓

**Why 31 beats 126 here.** 31 is prime, hence odd, so multiplying by 31 repeatedly never collapses to zero the way powers of 126 do. Every character keeps influencing the result no matter how long the string, which is what the "scramble the potato" intuition is about. Collisions still exist (`"포옹"` and `"kamala lifefully"` both give 1,732,557; `"守门员"` and `"孡㜹ቘ஝஝俎"` both give 23,729,400), and that is fine: a hash code need not be unique, only well spread.

The Java 8 version additionally caches:

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

The guard `h == 0 && this.length() > 0` means: recompute only if we have not already stored a nonzero value and the string is nonempty. Since `String` is immutable, the cached value can never go stale. (A mutable class could not safely cache like this, which connects back to Warning #1.)

### Example 4: `Math.floorMod` versus `%` on a negative hash code

```java
public class ModTest {
    public static void main(String[] args) {
        System.out.println(-1 % 4);               // -1
        System.out.println(Math.floorMod(-1, 4)); // 3
    }
}
```

Java's `%` takes the sign of the dividend, so `-1 % 4` is -1, and `buckets[-1]` throws. `Math.floorMod(-1, 4)` rounds toward negative infinity, producing a result with the sign of the divisor, so it gives 3, which is a legal index and matches the "loop back around" intuition (like negative indexing in Python).

This is not a rare edge case: `"đậu hũ".hashCode()` is -2,108,180,664, a perfectly ordinary string with a negative hash code. The correct reduction is:

```java
int index = Math.floorMod(item.hashCode(), buckets.length);
// Math.floorMod(-2108180664, 4) == 0
```

### Example 5: tracing an insertion end to end

Consider a 4-bucket Java-style hash table and `insert("đậu hũ")`:

1. **Hash.** Call `"đậu hũ".hashCode()`, which runs the base-31 Horner loop and returns -2,108,180,664.
2. **Reduce.** `Math.floorMod(-2108180664, 4)` = 0.
3. **Check for duplicates.** Walk the chain at bucket 0, calling `equals` on each item, because a set must not store duplicates. (This is exactly the dependency behind Warning #2: if `equals` says two objects are equal but `hashCode` sends them to different buckets, this check never even looks in the right place.)
4. **Insert.** Not found, so insert into bucket 0's list, at the head, which is Θ(1).
5. **Check the load factor.** If N/M now exceeds the threshold, resize.

With `"a"` (97), `"포옹"` (1,732,557), and `"kamala lifefully"` (1,732,557) also inserted into the same 4-bucket table, the slides show the resulting layout:

```
0: đậu hũ
1: a, 포옹, kamala lifefully
2: bee
3:
```

(`97 % 4 = 1`, `1732557 % 4 = 1`, `97410 % 4 = 2`.) Note `"포옹"` and `"kamala lifefully"` are in bucket 1 both because of a genuine hash-code collision and because 1,732,557 mod 4 = 1. These are two independent sources of collision: equal hash codes, and distinct hash codes that reduce to the same index.

### Example 6: hashing a collection, and why "ab" vs "ba" matters

```java
@Override
public int hashCode() {
    int hashCode = 1;
    for (Object o : this) {
        hashCode = hashCode * 31;              // elevate/smear the current hash code
        hashCode = hashCode + o.hashCode();    // add new item's hash code
    }
    return hashCode;
}
```

Same Horner structure, but the "digits" are now the elements' own hash codes rather than characters. The `* 31` step is what makes order matter: without it (just summing element hash codes) `[x, y]` and `[y, x]` would collide, exactly the "adding chars together is bad, 'ab' collides with 'ba'" failure from the hash-code-quality discussion. Starting at 1 rather than 0 means an empty collection and a collection of one zero-hash item do not both produce 0.

Note that the loop calls `o.hashCode()` on each element, so a list's hash code depends on its contents. **This is precisely why a `HashSet<List<Integer>>` is dangerous**: mutate the inner list and its hash code changes, so the outer set is now looking in the wrong bucket for it, and the item is effectively lost even though it is still physically stored. The lecture's shortcut (look at only the first few items to save time) trades collision rate for speed and still works correctly.

### Example 7: hashing a recursive structure

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

The base case is the sentinel leaf (`value == null` → 0). The recursive case combines three numbers with distinct powers of 31 as weights, so a value in the left subtree and the same value in the right subtree contribute differently, meaning mirror-image trees do not automatically collide. Reasoning about the call structure: each node's call triggers two child calls, so the whole tree is visited, and the hash code depends on every stored value and on the shape of the tree.

## Common Pitfalls

- **Thinking an M-fold speedup changes the asymptotics.** Θ(N/10) = Θ(N). A fixed number of buckets gives a real-world win but no asymptotic win. Only tying M to N (M = Nk) converts Θ(N/M) into Θ(1).
- **Quoting "hash tables are Θ(1)" without the conditions.** Θ(1) is *on average* and *assuming items are evenly spread*. A resize costs Θ(N), and a bad distribution costs Θ(N) because you traverse one long chain.
- **Thinking a good load factor alone implies good performance.** The lecture shows two tables with identical load factor N/M = 0.75 where one has Θ(N) `contains` and the other Θ(1). You also need even distribution.
- **Using `%` instead of `Math.floorMod`.** Hash codes are routinely negative, and `-1 % 4` is -1 in Java, which is an invalid index. This is the single most common implementation bug in this material.
- **Believing a hash code is unique.** Overflow makes uniqueness impossible for strings in Java. `"포옹"` and `"kamala lifefully"` share hash code 1,732,557. Buckets must therefore still compare items with `equals`.
- **Assuming equal hash codes are the only cause of collisions.** Two items also collide when distinct hash codes reduce to the same index mod M.
- **Forgetting that items move on resize.** The index depends on M, so doubling M requires re-reducing (and in Java, recomputing or reusing the hash code for) every item. In the k = 1.5 walkthrough, 16 and 20 share bucket 0 under `% 4` but split apart under `% 8`.
- **Storing mutable objects in a `HashSet`/`HashMap`.** If the object changes, its hash code changes and it is no longer findable in the bucket where it lives. Never make a `HashSet<List<Integer>>`.
- **Overriding `equals` without `hashCode`.** Two objects that are `equals` can then land in different buckets, so lookups fail. The reverse coupling also matters: the bucket scan uses `equals` to decide membership.
- **Picking a large even base like 126 "for uniqueness".** Overflow is far worse for base 126: 126³² = 0 in `int` arithmetic, so only the last 32 characters matter, producing massive collisions (the four "... best thing on the earth." strings all give 634199182). Prefer a small prime.
- **Writing a hash code that just sums the characters.** "ab" collides with "ba". Order information is lost without the multiply-by-base step.
- **Confusing hash table, hash set, and hash map.** The hash table is the storage mechanism (M buckets, N items, a `hashCode` deciding the bucket). A hash set is the Set ADT built on it; a hash map is the Map ADT built on it.
- **Assuming `hashCode()` comes from an interface.** It does not; it is inherited from `Object` by every class, and the default implementation uses the memory address.
- **Forgetting the no-zero-digit detail in base 26.** Because `aaa` must differ from `a`, letters run 1 through 26, not 0 through 25.

## Likely Exam Points

### 1. Drawing a hash table after a sequence of insertions, including a resize

**Q.** A hash table of integers (hash code = the integer itself) starts with M = 4 and resizes by doubling when N/M ≥ 1.5. Insert 7, 16, 3, 11, 20, 13 in that order. Draw the final table.

**A.** The insertion of 13 brings N = 6, M = 4, N/M = 1.5, triggering a resize to M = 8. Recompute every index with `% 8`:

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

N = 6, M = 8, N/M = 0.75. The key move to show your work on is that items must be re-reduced: 16 and 20 separate, while 3 and 11 stay together.

### 2. Computing a base-B integerization by hand

**Q.** Convert "bee" to an integer using the powers-of-26 strategy, where `a` = 1 and `e` = 5.

**A.** (2 × 26²) + (5 × 26¹) + (5 × 26⁰) = 1352 + 130 + 5 = **1487**. (Compare `cat` = (3 × 26²) + (1 × 26¹) + (20 × 26⁰) = 2074.)

### 3. Computing Java's real `String.hashCode()`

**Q.** What does `"bee".hashCode()` return in Java, and why is it not 1487?

**A.** 97,410. Java uses base **31**, not 26, and uses the characters' raw code points (`'b'` = 98, `'e'` = 101), not alphabet positions. The Horner trace is 98 → 98×31 + 101 = 3139 → 3139×31 + 101 = 97,410.

### 4. The negative-hash-code reduction

**Q.** An item's `hashCode()` returns -1 and the table has 4 buckets. What is `-1 % 4` in Java, which bucket should the item go in, and what should the code use?

**A.** `-1 % 4` is **-1** in Java, which would be an invalid array index. The item belongs in bucket **3** (wrapping around, like negative indexing in Python). Use `Math.floorMod(-1, 4)`, which returns 3.

### 5. Stating the runtimes with their conditions

**Q.** Fill in the worst-case / average runtimes for `contains(x)` and `add(x)`: bushy BST, separate-chaining hash table without resizing, separate-chaining hash table with resizing. State every assumption.

**A.**

| | `contains(x)` | `add(x)` |
|---|---|---|
| Bushy BSTs | Θ(log N) | Θ(log N) |
| Hash table, no resizing | Θ(N) | Θ(N)† |
| Hash table, with resizing | Θ(1)† | Θ(1)\*† |

\* on average &nbsp;&nbsp; † assuming items are evenly spread. Without resizing, M is constant so buckets hold Θ(N/M) = Θ(N) items. With resizing, M = Θ(N) so buckets hold Θ(N/M) = Θ(1) items, but a resize itself costs Θ(N) and an uneven distribution degrades to Θ(N).

### 6. Why resizing costs Θ(N)

**Q.** Break down the cost of a resize operation and justify the total.

**A.** Doubling the number of buckets is Θ(M); iterating over the M buckets is Θ(M); iterating over the N items is Θ(N); per item, recomputing the hash code and new index is Θ(1), and re-inserting is Θ(1) if you insert at a known position such as the head of the chain. Since M = Θ(N), the total is Θ(M) + Θ(M) + Θ(N)×(Θ(1) + Θ(1)) = **Θ(N)**.

### 7. Load factor is not sufficient for good performance

**Q.** Two separate-chaining tables both have N = 3, M = 4, load factor 0.75. Table A has all three items in bucket 3; table B has one item each in buckets 0, 2, 3. Compare `contains(x)`.

**A.** Table A is Θ(N): you must traverse the three-item linked list in bucket 3. Table B is Θ(1): each bucket holds at most one item. Identical load factors, different performance. Good performance requires an **even distribution**, which depends on both the hash code and the table size.

### 8. The two warnings

**Q.** Why must you never put a mutable object in a `HashSet`, and why must you never override `equals` without overriding `hashCode`?

**A.** (1) The bucket an item lives in is determined by its hash code at insertion time. If the object's variables change, its hash code changes, so future lookups reduce to a different bucket and the item is lost (e.g. `HashSet<List<Integer>>`). (2) `HashMap`/`HashSet` locate a candidate bucket from `hashCode` and then use `equals` to check membership within that bucket. If `equals` reports two objects equal but `hashCode` sends them to different buckets, the membership check never runs on the right chain, so items get lost and behavior is generally weird.

### 9. Why a small prime base

**Q.** Base 126 gives unique integers for all ASCII strings (ignoring overflow), while base 31 does not. Why does Java use 31?

**A.** Because overflow is not ignorable. 126 is even, and in 32-bit `int` arithmetic 126³² = 126³³ = ... = 0, so every character more than 32 positions from the end is multiplied by zero and contributes nothing. All four of "geocronite / flan / treachery / Brazil is the best thing on the earth." hash to 634199182. A prime base is never even, avoiding this collapse, and also lowers the chance of a bad relationship between the hash code and the number of buckets. Small keeps the computation cheap. Real hash codes do not need uniqueness, only good spread.

### 10. Identifying bad hash functions

**Q.** For each, say whether it is a good or bad `hashCode` for strings and why: (a) `return 0;` (b) return the first character's alphabet position (c) add all the characters together (d) treat the string as a base-B number.

**A.** (a) Bad: every item collides into one bucket, so operations are Θ(N). (b) Bad: terrible worst case (a dataset of words starting with "a"), only 26 buckets ever used, and strings like "1a2b3c" do not start with a letter. (c) Bad: loses ordering, so "ab" collides with "ba". (d) Good: multiplying by powers of a base scrambles all the data together into a seemingly random integer.

### 11. Terminology

**Q.** Distinguish hash table, hash set, and hash map.

**A.** A **hash table** is a way of storing information: M buckets holding N items, where each item's `hashCode` tells you which bucket to use. A **hash set** is an implementation of the Set ADT with a hash table as its engine. A **hash map** is an implementation of the Map ADT with a hash table as its engine.

### 12. Why hash tables answer the lecture's two motivating questions

**Q.** Name two advantages hash tables have over search-tree-based sets.

**A.** They do not require items to be comparable (no `<` needed, only `hashCode` and `equals`), and they achieve Θ(1) average operations rather than Θ(log N). The slides add that they have great performance in practice and are often relatively simple to implement, which is why they are the most popular set/map implementation (Python dictionaries are hash tables in disguise).

## Summary

- Search-tree sets require `Comparable` items and are capped at Θ(log N). Hash tables fix both.
- Splitting one list into M buckets ("separate chaining") and choosing a bucket by **modulus** gives an M-fold speedup, but Θ(N/M) = Θ(N) when M is constant.
- Making **M grow with N** (M = Nk) turns Θ(N/M) into Θ(1/k) = Θ(1). Larger k is faster but uses more memory.
- **Load factor** = N/M. When it exceeds a constant (the lecture uses 1.5 in the walkthrough, 0.75 in the summary), **double M** and re-reduce every item; items can and do change buckets.
- Non-integer data is first converted by an **integerization function**: treat a string as a base-B number via the Horner loop `intRep = intRep * B + digit`. Base ≥ 26 gives unique values for lowercase words mathematically.
- **ASCII** assigns 0..127 to characters (printable 33..126); Java `char`s go further via **Unicode**.
- Java `int`s are **fixed precision** (4,294,967,296 values, max 2,147,483,647, overflowing to -2,147,483,648), so unique integerization is impossible. The function becomes a **hash code**: a projection from a huge set onto a fixed finite one.
- A `DynamicArrayOfLists` plus a finite integerization function is a **hash table**: `data → hashCode() → hash code → reduce → index → bucket`.
- Every Java object has `hashCode()` inherited from `Object` (default: memory address); it is not an interface. `java.util.HashMap` and `java.util.HashSet` use it.
- Java's `String.hashCode()` is a base-**31** Horner loop over `charAt`, with caching in Java 8. Collisions happen (`"포옹"` and `"kamala lifefully"` → 1,732,557).
- Reduce with **`Math.floorMod(hashCode, M)`**, never `%`: `-1 % 4` is -1 in Java, but `Math.floorMod(-1, 4)` is 3.
- **Warning #1:** never store mutable objects in a `HashSet`/`HashMap`; a changed hash code loses the item. **Warning #2:** never override `equals` without `hashCode`; buckets use `equals` for membership.
- Performance: no resizing → Θ(N). With resizing → **Θ(1) on average, assuming even distribution**. A resize is Θ(N) (Θ(M) + Θ(M) + Θ(N) work, with M = Θ(N)); head insertion keeps per-item re-insertion Θ(1). Equal load factors do not imply equal performance.
- Good hash codes (extra): multiply by powers of a **small prime** base; `return 0`, first-character-only, and character-summing are all bad. Large even bases like 126 collapse under overflow (126³² = 0), making only the last 32 characters matter.
- Alternatives to chaining (extra): **open addressing** with linear or quadratic probing, cuckoo hashing, non-list buckets. 61B uses separate chaining.
