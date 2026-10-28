<!-- Wed, Oct 28, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 27: Hashing II

## Overview

This lecture finishes the hash table story started in Lecture 26. Lecture 26 answered the two structural problems (how to turn data into an array index, and what to do about collisions); this lecture is about the *contract* that makes hash tables actually work. The central theme is that a hash table's correctness depends on an agreement between three things: `hashCode`, `equals`, and the key's state not changing. We use a `ColoredNumber` visualizer to build intuition about how different hash functions spread items across buckets (returning `0` gives no spread, sum-of-digits gives bad spread, returning `num` gives good spread, and Java's default memory-address hash code gives good spread too). That raises the "hard question": if the default hash code already spreads well, why write a custom one? The answer is that spread is not the point: *consistency with `equals`* is. If you override `equals` but not `hashCode`, `contains` can only find an object by luck (probability 1/M) and `add` silently creates duplicates. We then look at immutability (what makes a class immutable, `final`, defensive copying of arrays) and why mutable keys break hash tables: mutating a key changes its hash code, so the item is "lost" in a bucket the table will never search again. The lecture closes with a peek at the real `java.util.HashSet`/`HashMap` source: a `HashSet` is a `HashMap` with ignored values, the table starts at 16 buckets, resizes when N/M exceeds a load factor of 0.75, uses a bit-mixing reduce function, and converts overfull bins into red-black trees.

---

## Key Concepts

### 1. Recap: the two problems and their answers

A hash table is a way of storing N items in M buckets.

- **Problem 1: how do we translate data into an array index?** Data goes through a hash function (`hashCode()`) to produce a *hash code*, an `int` which may be negative or huge. We then *reduce* the hash code into the range `[0, M)`. The lecture's reduce step is `Math.floorMod(hashCode, M)`. Example from the slides: `"đậu hũ".hashCode()` is `-2108180664`, and `Math.floorMod(-2108180664, 4)` is `0`, so it lands in bucket 0. Another example: `"Josh Hug"` hashes to `-1057935820`, which also floorMods to `0` with 4 buckets, so both strings share bucket 0.
- **Problem 2: what if two items get the same index?** Each bucket holds a *separate chain* of items (a linked list), and we resize the array as N grows.

`Math.floorMod` rather than `%` matters: in Java, `-2108180664 % 4` can be negative, which is not a legal array index. `floorMod` always returns a non-negative result for a positive modulus.

### 2. The runtime table

| Operation | `contains(x)` | `add(x)` |
|---|---|---|
| Bushy BSTs | Θ(log N) | Θ(log N) |
| Separate chaining hash table, **no** resizing | Θ(N) | Θ(N) |
| Separate chaining hash table **with** resizing | Θ(1)† | Θ(1)\*† |

\* means "on average" (amortized, because of resizing). † means "assuming items are evenly spread".

Both daggers matter. Without resizing, M is fixed, so with N items the chains have average length N/M = Θ(N) and you get linear behavior. With resizing but a terrible hash function (say, everything returns 0), all N items pile into one bucket and you are again Θ(N). The Θ(1) claim is contingent on *both* resizing *and* good spread.

### 3. Spread: what good and bad hash functions look like

The visualizer inserts `ColoredNumber` objects (fields `private int num; private Color color;`) numbered 0 through 19 into a table with 6 buckets.

With the **default** `hashCode` (inherited from `Object`, based on the object's memory address):
- Different colored numbers end up in the same bin.
- The distribution is roughly even across bins.
- "Looks kinda like a bell curve... sometimes" and you sometimes see sequential items in the same bin: these are just the random fluctuations you get from throwing 20 balls into 6 bins.

The slides then classify hash functions by spread quality:
- **No spread:** `return 0;`. Every item goes to bucket 0, so the "hash table" degenerates into a single linked list.
- **Bad spread:** sum of the digits. For numbers 0 through 19, digit sums range only from 0 to 10, so the hash codes clump into a narrow range and the bins are unevenly loaded.
- **Good spread:** `return num;`. For inputs 0 through 19 with 6 buckets, this distributes items as evenly as possible (round-robin: 0, 6, 12, 18 in bucket 0; 1, 7, 13, 19 in bucket 1; and so on).
- **Good spread (default):** the memory address is effectively random, so items spread evenly.

The lecture also mentions "more exotic" hash functions you *could* define: leading digit, sum of digits, length of the number. The point is that `hashCode` is just a method and you may put any (deterministic) logic in it; the design question is only whether it spreads well and whether it respects `equals`.

### 4. The hard question: why write a custom `hashCode` at all?

If the default memory-address hash code gives *good spread*, why bother? Because **spread is not the only requirement.** The hash table's whole search strategy is: "to look for `x`, compute `x`'s bucket and search only that bucket." For that to find anything, every object that is `equals` to `x` must land in the *same* bucket as `x`. The memory-address hash code gives two distinct-but-equal objects *different* hash codes, so the table looks in the wrong place.

So: custom hash functions exist to keep `hashCode` consistent with a custom `equals`, not primarily to improve spread.

### 5. `contains` with the default hash code: the 1/M lottery

With the default (memory address) `hashCode`:

```java
HashSet<ColoredNumber> hs = new HashSet<>();
ColoredNumber x = new ColoredNumber(4);
hs.add(x);
IO.println(hs.contains(x));   // true

ColoredNumber y = new ColoredNumber(4);
IO.println(hs.contains(y));   // false
IO.println(x.equals(y));      // false (with the default equals)
```

`hs.contains(x)` is `true` because `x` is literally the same object: same memory address, same hash code, same bucket, and `equals` succeeds by reference.

`hs.contains(y)` is `false`. In the slides' concrete numbers: `x` hashed to `445051633`, which floorMods to bucket 1 with M = 4, so `x` sits in bucket 1. The new object `y` hashed to `1349277854`, which floorMods to bucket 2, which is empty, so `contains` returns `false` after searching nothing. The slide's caption is the whole idea: "Different memory address means different hash code!"

But notice it is not *guaranteed* false. A third object `z = new ColoredNumber(4)` hashed to `1775282465`, which floorMods to **bucket 1**, the right bucket. If `equals` compares `num`, then `contains(z)` returns `true`: "We could get lucky and look in the right bucket!"

**So with an overridden `equals` but a memory-address `hashCode`, `contains` on an equal-but-distinct object returns `true` with probability about 1/(number of buckets), and `false` otherwise.** It is not reliably broken, which is worse than reliably broken: it is a nondeterministic bug.

### 6. The `equals` method for `ColoredNumber`

```java
@Override
public boolean equals(Object o) {
    if (o instanceof ColoredNumber otherCn) {
        return this.num == otherCn.num;
    }
    return false;
}
```

Two `ColoredNumber`s are equal if they have the same `num` (color is ignored). This uses Java's *pattern matching for `instanceof`*: `o instanceof ColoredNumber otherCn` both tests the type and, if the test passes, binds the narrowed variable `otherCn` so you do not need a separate cast.

The guiding design rule from the slides: **equal objects should behave identically.** If `x.equals(y)`, then there should be no observable way to distinguish `x` from `y` through the class's public behavior, and `hashCode()` is public behavior, so `x.hashCode()` must equal `y.hashCode()`.

### 7. Duplicate values: overriding `equals` but not `hashCode`

Suppose we keep the `num`-based `equals` but do *not* override `hashCode`. Insert 0 through 19 into the 6-bucket table, then:

```java
ColoredNumber zero = new ColoredNumber(0);
hs.add(zero);   // does another zero appear?
```

The multiple-choice answer: **any of A, B, or C can happen.** The new `zero` has a fresh random memory address, so it lands in an essentially random bin:
- 1/6 chance it lands in the bin that already contains the original 0. There, `add` walks the chain, finds an item where `equals` returns true, and refuses to insert. No duplicate: "equals blocks it."
- 5/6 chance it lands in any other bin. Nothing in that bin is `equals` to it (nothing there has `num == 0`), so it gets added. **Now the set contains two "equal" zeroes**, which violates the entire meaning of a Set.

(In the slide's specific picture the original zero happened to be in bin 1, so the breakdown was "5/6ths chance: in bin 0, 2, 3, 4, or 5: duplicate! 1/6 chance: in bin 1, no duplicate!")

### 8. The takeaway rule

> **If your class overrides `equals`, you should also override `hashCode` in a consistent manner.**
> **If two objects are `equals`, they must always have the same `hashCode`.**

If you violate it, "everything breaks":
- `contains` cannot find objects (unless it gets lucky).
- `add` results in duplicates.

Note the direction of the implication. `equals` ⟹ same `hashCode` is required. The converse is *not*: unequal objects are allowed to share a hash code (that is just a collision, which separate chaining handles fine). This is why `return 0;` is a *legal but useless* `hashCode`: it never lies, it just destroys performance.

### 9. Immutable data types

> **An immutable data type is one for which an instance cannot change in any observable way after instantiation.**

Examples given: mutable: `ArrayDeque`, `Percolation`. Immutable: `Integer`, `String`, `Date`.

```java
public class Date {
    public final int month;
    public final int day;
    public final int year;
    private boolean contrived = true;
    public Date(int m, int d, int y) {
        month = m; day = d; year = y;
    }
}
```

The `final` keyword helps the compiler enforce immutability: a `final` variable may be assigned once (in the constructor or in an initializer) and never again.

Two crucial caveats from the slides:
- **`final` is not sufficient for immutability.** A `final` reference to a mutable object still lets you mutate the object it points to. (This is the `RocksBox` case below.)
- **`final` is not necessary for immutability.** The `private boolean contrived` field in `Date` is not final, but since nothing can observe or change it from outside, `Date` instances are still observably unchangeable. The definition is about *observable* change, not about field modifiers.

### 10. Mutable hash table keys: the lost item

You *can* create a `HashSet<List<Integer>>`. "Weird stuff happens" if you insert a `List` and then mutate it, because a list's `hashCode` is computed from its contents. Mutating the list changes its hash code, which changes the bucket the table *would* put it in, but the item physically stays in the old bucket. The item is now unfindable: it is in bucket A, but every future lookup computes bucket B.

> **Bottom line: never mutate an Object being used as a key.** You get incorrect results and the item gets lost.

This connects the two halves of the lecture: immutable types make safe keys precisely because their hash code can never go stale. `String` and `Integer` are immutable, which is a big part of why they are the default keys everyone uses.

### 11. A peek into Java's real `HashSet` / `HashMap`

Reading the OpenJDK source (`java/util/HashSet.java` and `java/util/HashMap.java`):

- `HashSet` **delegates all of its work to a `HashMap<K, Object>` and ignores the value.** A set is just a map whose values nobody looks at.
- The table **starts at size 16** and **doubles every time N exceeds the load factor, which defaults to 0.75.** So N/M is never more than 0.75. The printed trace from the slides:
  ```
  Resize occurred, N = 13,   hash table array size = 32
  Resize occurred, N = 25,   hash table array size = 64
  Resize occurred, N = 49,   hash table array size = 128
  Resize occurred, N = 97,   hash table array size = 256
  Resize occurred, N = 193,  hash table array size = 512
  Resize occurred, N = 385,  hash table array size = 1024
  Resize occurred, N = 769,  hash table array size = 2048
  Resize occurred, N = 1537, hash table array size = 4096
  ```
  Sanity check: 16 × 0.75 = 12, so the 13th item triggers the resize to 32. Then 32 × 0.75 = 24, so the 25th triggers 64, and so on.
- The **reduce function is not `floorMod`**. It is (as shown on the slide) along the lines of `(hc ^ (hc >>> 16)) & (M - 1)`. Two ideas are in play: `& (M - 1)` is a fast bitwise substitute for `% M` that works because M is always a power of 2, and `hc ^ (hc >>> 16)` XORs the high 16 bits of the hash code into the low 16 bits so that the high bits are not simply thrown away by the mask. The slide's example: `"đậu hũ".hashCode()` of `-2108180664` reduces to index 15 when M = 16. The slides note the real reduce function "is a bit complicated" and is explicitly offered as office-hours material rather than examinable detail.
- **Longest bucket:** a simulation holding the load factor at or below 0.75 ends up with a longest bucket of around ~5. The precise math is the "balls into bins" problem, derived in CS70, and it can be used to show that the true worst case runtime is "a bit worse than constant" rather than exactly constant.
- **Red-black tree bins:** if a bin gets too full, `HashMap` converts it from a linked list into a red-black tree, so lookup within an overpopulated bin is logarithmic rather than linear. The quoted source comment explains it "usually acts as a binned (bucketed) hash table, but when bins get too large, they are transformed into bins of TreeNodes, each structured similarly to those in `java.util.TreeMap`." Explicitly "well beyond the scope of our course."
- Accompanying quote: "The most useful algorithms are, unfortunately, not always the most beautiful." - Josh Hug.

### 12. FAQ: hash set vs. hash map vs. hash table

Straight from the slides:
- A **hash set** is an implementation of the **Set ADT** using the hash table as its engine.
- A **hash map** is an implementation of the **Map ADT** using the hash table as its engine.
- A **hash table** is a way of storing information, where you have M buckets storing N items; each item has a `hashCode` telling you which of the M buckets to put it in.

So "hash table" is the mechanism; "hash set" and "hash map" are the data types built on it.

---

## Definitions

- **Hash table:** a way of storing information in which M buckets store N items; each item's `hashCode` determines which bucket it goes in.
- **Hash set:** an implementation of the Set ADT whose engine is a hash table.
- **Hash map:** an implementation of the Map ADT whose engine is a hash table.
- **Hash function:** a method (`hashCode()` in Java) that converts data into an integer hash code.
- **Hash code:** the `int` returned by a hash function. May be negative, may be any `int` value.
- **Reduce (step):** converting a hash code into a valid array index in `[0, M)`. In lecture, `Math.floorMod(hashCode, M)`. In real `HashMap`, a bit-mixing-then-mask operation.
- **Bucket / bin:** one slot of the hash table's array.
- **Separate chaining:** collision resolution in which each bucket holds a chain (linked list) of all items that reduce to that index.
- **Load factor:** N/M, the number of items divided by the number of buckets. Java's `HashMap` default is 0.75, resizing when it is exceeded.
- **Default `hashCode` (Java):** the `hashCode` inherited from `Object`, based on the object's memory address; effectively random, so distinct objects almost always get distinct hash codes.
- **`hashCode`/`equals` contract:** if two objects are `equals`, they must always have the same `hashCode`. (The converse need not hold.)
- **Immutable data type:** a type for which an instance cannot change in any observable way after instantiation.
- **`final` variable:** a variable that may be assigned a value once (in the constructor or an initializer) and never changed afterward. Neither sufficient nor necessary for a class to be immutable.
- **Defensive copying:** storing a copy of a passed-in mutable object (for example, a new array copied with `System.arraycopy`) rather than the caller's reference, so the caller cannot mutate your internals.
- **Balls into bins problem:** the probabilistic question of how full the fullest bin gets when you throw N balls into M bins at random; it characterizes the longest bucket in a hash table. Derived in CS70.
- **No spread / bad spread / good spread:** informal lecture vocabulary for hash functions. No spread = all items to one bucket (`return 0`). Bad spread = items clump into few buckets (digit sum). Good spread = items roughly evenly distributed (`return num`, or the memory address).

---

## Worked Examples

### Example 1: Tracing `contains` with the default hash code

```java
HashSet<ColoredNumber> hs = new HashSet<>();
ColoredNumber x = new ColoredNumber(4);
hs.add(x);
IO.println(hs.contains(x));

ColoredNumber y = new ColoredNumber(4);
IO.println(hs.contains(y));
```

Assume the `num`-based `equals` above but **no** `hashCode` override, and (as in the slides) 4 buckets.

Step by step, reasoning about the heap in words:

1. `new HashSet<>()` creates a set whose underlying array has some number of buckets, all empty. (Real Java starts at 16; the slides draw 4, so follow the slides.)
2. `new ColoredNumber(4)` allocates a `ColoredNumber` object somewhere in the heap. `x` holds the address of that object. The object's `num` field is 4.
3. `hs.add(x)` calls `x.hashCode()`. Since `hashCode` is not overridden, this returns something derived from `x`'s memory address: `445051633` in the slides. Reduce: `Math.floorMod(445051633, 4) = 1`. The table walks bucket 1's chain (empty), finds nothing `equals` to `x`, and appends `x` to bucket 1.
4. `hs.contains(x)` calls `x.hashCode()` again: same object, same address, same `445051633`, same bucket 1. Walk bucket 1's chain: the first node holds a reference to the very same object. `equals` returns true (same `num`, trivially). **Prints `true`.**
5. `new ColoredNumber(4)` allocates a **second, different** object, also with `num == 4`. `y` holds a *different* address from `x`. In box-and-pointer terms there are now two separate boxes in the heap, each containing `num = 4`, pointed to by `x` and `y` respectively.
6. `hs.contains(y)` calls `y.hashCode()`, which reflects `y`'s address: `1349277854`. Reduce: `Math.floorMod(1349277854, 4) = 2`. The table looks **only** in bucket 2. Bucket 2 is empty. "Nothing at index 2!" `contains` returns false **without ever calling `equals`**. **Prints `false`.**

The critical insight: step 6 never even got the chance to compare the two objects. A broken `hashCode` means `equals` is never consulted, because the search goes to the wrong bucket.

Had the third object's hash code landed on bucket 1 instead (as `z`'s `1775282465` does, since `Math.floorMod(1775282465, 4) = 1`), the chain walk *would* have reached `x`, `equals` *would* have compared `num == num`, and `contains` would print `true`. Hence: `true` with probability 1/M, `false` otherwise.

### Example 2: Fixing it with a consistent `hashCode`

```java
public class ColoredNumber {
    private int num;
    private Color color;

    @Override
    public boolean equals(Object o) {
        if (o instanceof ColoredNumber otherCn) {
            return this.num == otherCn.num;
        }
        return false;
    }

    @Override
    public int hashCode() {
        return num;
    }
}
```

Now re-run Example 1. `x.hashCode()` is 4 and `y.hashCode()` is also 4, because both have `num == 4` and the hash code is computed from *state*, not *identity*. Both reduce to the same bucket. `hs.contains(y)` goes to that bucket, walks the chain, reaches `x`, calls `equals`, gets true, and prints **`true`**, deterministically, every run.

This also fixes the duplicate problem: `hs.add(new ColoredNumber(0))` on a set already containing a 0 always lands in 0's bucket, always finds the existing equal item, and always refuses to insert.

And `return num;` is exactly the "good spread" function from the visualizer: inserting 0 through 19 into 6 buckets distributes them round-robin, four items per bucket for buckets 0 and 1 and three or four elsewhere. Correct *and* fast.

Note that `color` appears in neither `equals` nor `hashCode`, and that is fine and consistent: two `ColoredNumber`s with the same `num` and different colors are `equals`, and they also share a hash code. The contract holds.

### Example 3: The zero hash code

```java
@Override
public int hashCode() {
    return 0;
}
```

What distribution do we expect? **Everything in bucket 0**, with all other buckets empty, because `Math.floorMod(0, M) = 0` for every item. The visualizer confirms this: one enormous chain in bin 0.

Important: this is **correct but slow.** It never violates the `hashCode`/`equals` contract (equal objects trivially have equal hash codes, since *all* objects do), so `contains` always finds what is there and `add` never creates duplicates. It just reduces the hash table to a single linked list, making `contains` and `add` Θ(N). This is the cleanest illustration that the contract is about *correctness* and spread is about *performance*, and they are separate concerns.

### Example 4: Which classes are immutable?

```java
public class Pebble {              public class Rock {
  public int weight;                 public final int weight;
  public Pebble() {                  public Rock (int w) {
    weight = 1;                        weight = w;
  }                                  }
}                                  }

public class RocksBox {            public class SecretRocksBox {
  public final Rock[] rocks;         private Rock[] rocks;
  public RocksBox (Rock[] rox) {     public SecretRocksBox(Rock[] rox) {
    rocks = rox;                       rocks = rox;
  }                                  }
}                                  }
```

**`Pebble`: mutable.** `weight` is public and non-final:
```java
Pebble p = new Pebble();
p.weight = 2;
```

**`Rock`: immutable.** `weight` is `final` and is an `int` (a primitive, so there is no inner object to mutate). No mutation possible. The slide footnote: "Unless you use the special 'Reflections' library which lets you disobey access modifiers."

**`RocksBox`: mutable.** This is the "`final` is not sufficient" case. `rocks` is a `final` *reference*, so you can never point `rocks` at a different array, but the array it points to is itself mutable, and it is public:
```java
Rock r1 = new Rock(10);
Rock r2 = new Rock(20);
Rock[] rox = {r1, r2};
RocksBox rb = new RocksBox(rox);
rb.rocks[1] = null;        // observable change!
```
In box-and-pointer terms: `final` locks the arrow, not the box the arrow points to.

**`SecretRocksBox`: mutable.** Here the field is `private`, so you cannot reach the array through `rb`. But the constructor stored the *caller's* reference, so the caller still holds an arrow into the box's internals:
```java
Rock r1 = new Rock(10);
Rock r2 = new Rock(20);
Rock[] rox = {r1, r2};
SecretRocksBox rb = new SecretRocksBox(rox);
rox[0] = new Rock(-999);   // mutates rb's internal array!
```
This is the classic *escaping reference* bug: `rocks` and `rox` are two names for one array.

**The fix: defensive copying.**
```java
public class SecretRocks {
   private Rock[] rocks;
   public SecretRocks(Rock[] rox) {
       rocks = new Rock[rox.length];
       System.arraycopy(rox, 0,
                        rocks, 0,
                        rox.length);
   }
}
```
Now the constructor allocates its *own* array and copies the contents in. After construction, `rox` and `rocks` point to two distinct arrays, so `rox[0] = new Rock(-999);` changes only the caller's array. "Example mutation fails!" This works here partly because `Rock` is itself immutable: copying the array of references is enough, since nobody can mutate the individual `Rock`s either. *(extra context: if the elements had been mutable, a shallow array copy would not have been enough; you would need a deep copy. The slides do not raise this, but it follows from the same reasoning.)*

**Trade-offs of immutability (from the slides):**
- *Advantage:* less to think about, which avoids bugs and makes debugging easier. The analogy given: an immutable class has buttons you can press and windows you can look inside, and the results are ALWAYS the same, no matter what.
- *Disadvantage:* you must create a new object any time anything changes. Example: `String` concatenation is slow, because `concat` cannot modify the existing `String` and must build a fresh one. (The slide shows `String` with `charAt(int i)`, `compareTo(String s)`, `concat(String s)`, `split(String r)`, none of which mutate the receiver.)

### Example 5: Mutating a key, step by step

```java
List<Integer> items = new ArrayList<>();
items.add(0);
items.add(1);
System.out.println(items.hashCode());        // 962

HashSet<List<Integer>> hs = new HashSet<>();
hs.add(items);
hs.add(List.of(2, 3));
items.add(7);
System.out.println(hs.contains(items));
```

Assume 4 buckets. The trace:

1. `items` is the `ArrayList` `[0, 1]`. Its `hashCode()` is **962**. (`List.hashCode` is defined from the elements' hash codes, so it is content-based, not address-based.)
2. `hs.add(items)` reduces 962: `962 % 4 = 2`. `items` is stored in **bucket 2**.
3. `hs.add(List.of(2, 3))`: `[2, 3]` has hash code **1026**, and `1026 % 4 = 2` as well. So `[2, 3]` also goes in **bucket 2**. This is an ordinary collision, and separate chaining handles it: bucket 2's chain now has two items. Nothing is wrong yet.
4. `items.add(7)` mutates the list to `[0, 1, 7]`. **The hash table is not notified and does nothing.** The object sitting in bucket 2's chain is the very same object `items` points to, so from the table's perspective its contents just silently changed underneath it. Its hash code is now **29829**.
5. `hs.contains(items)` computes `items.hashCode()` = 29829, reduces: `29829 % 4 = 1`, and looks in **bucket 1**. Bucket 1 is **empty**. Returns **`false`**, even though the object is physically sitting right there in bucket 2.

So the failing call is `contains(items)`. The item is not deleted, not corrupted, just *unreachable*: filed under 2, searched for under 1. It also now blocks nothing, so you could `add` an equal list and get a duplicate, and iteration would still show it, which makes the inconsistency visible and confusing.

Box-and-pointer view: bucket 2's chain node holds an arrow to the same heap object that `items` points to. Mutation changes that one shared object. The table's only record of "where this belongs" was the bucket index it computed at insertion time, and that index is now stale.

---

## Common Pitfalls

1. **Overriding `equals` without overriding `hashCode`.** The single biggest pitfall of the lecture. Symptoms are nondeterministic: `contains` fails most of the time but succeeds with probability 1/M, and `add` creates duplicates most of the time. Because it *sometimes* works, it is easy to miss in testing.
2. **Thinking `return 0;` is "wrong" in the correctness sense.** It is contract-compliant and gives correct results. It is only catastrophically slow (Θ(N) operations). Conversely, a memory-address `hashCode` combined with a custom `equals` has *great* spread and is *incorrect*. Spread and correctness are orthogonal.
3. **Getting the contract's direction backwards.** `equals` ⟹ equal hash codes is required. Equal hash codes ⟹ `equals` is *not* required and is not even desirable to insist on; collisions are normal and handled by chaining.
4. **Using `%` instead of `Math.floorMod`.** Hash codes are often negative (`"đậu hũ".hashCode()` is `-2108180664`), and `%` in Java can yield a negative result, which is not a legal index.
5. **Mutating an object after using it as a key.** The item becomes unfindable. Prefer immutable keys (`String`, `Integer`) or at minimum never touch a key after insertion.
6. **Assuming `final` makes a class immutable.** `RocksBox` has a `final` array field and is mutable, because `final` freezes the reference, not the referenced object's contents.
7. **Assuming `private` makes a class immutable.** `SecretRocksBox` has a `private` array field and is mutable, because the constructor stored the caller's reference, which the caller still holds. Immutability requires that *no* path of observable change exists, including paths through aliases created before or during construction.
8. **Assuming non-`final` fields make a class mutable.** `Date`'s `contrived` field is non-final, but it is private and unobservable, so `Date` is still immutable. The definition is about *observable* change.
9. **Forgetting the `† assuming items are evenly spread` caveat** when claiming hash tables are Θ(1). Also forgetting that **without resizing** a separate-chaining hash table is Θ(N), not Θ(1).
10. **Confusing hash set, hash map, and hash table.** The first two are ADT implementations; the third is the underlying storage technique.
11. **Including a field in `equals` but not `hashCode`, or vice versa, inconsistently.** Omitting `color` from *both* is fine. Including `color` in `hashCode` but not in `equals` breaks the contract, since two objects equal by `num` alone could then get different hash codes.

---

## Likely Exam Points

### 1. Runtime table with the daggers

**Q:** Give the runtime of `contains(x)` for (a) a bushy BST, (b) a separate-chaining hash table with no resizing, (c) a separate-chaining hash table with resizing. State any assumptions for (c).

**A:** (a) Θ(log N). (b) Θ(N). (c) Θ(1) on average, assuming items are evenly spread (and resizing keeps the load factor bounded by a constant). Without the even-spread assumption, (c) degrades toward Θ(N).

### 2. Predicting the output of `contains` with a broken `hashCode`

**Q:** `ColoredNumber` overrides `equals` to compare `num` but does not override `hashCode`. A `HashSet<ColoredNumber>` with M buckets contains `new ColoredNumber(4)`. What does `hs.contains(new ColoredNumber(4))` return?

**A:** `true` with probability about 1/M, `false` otherwise. The new object's memory-address hash code is effectively random, so it reduces to a random bucket; only if that happens to be the bucket holding the stored 4 will `equals` ever be consulted and succeed.

### 3. The duplicates question

**Q:** With the `num`-based `equals` and no `hashCode` override, a 6-bucket `HashSet` already contains the numbers 0 through 19. We call `hs.add(new ColoredNumber(0))`. Which can happen: (A) another 0 added to bin 0, (B) another 0 added to bin 1, (C) another 0 added to some other bin, (D) no duplicate?

**A:** All of A, B, C, and D are possible, because the new object's bin is effectively random. If it lands in the bin that already holds the original 0 (probability 1/6), `equals` blocks the insert and there is no duplicate; in the other 5/6 of cases nothing in the chosen bin is `equals` to it, so a duplicate 0 is created. (In the slide's picture the original 0 was in bin 1, giving 1/6 chance of no duplicate and 5/6 chance of a duplicate in bin 0, 2, 3, 4, or 5.)

### 4. State the `hashCode`/`equals` contract and what breaks

**Q:** State the rule relating `equals` and `hashCode`, and name the two concrete failure modes when it is violated.

**A:** If a class overrides `equals`, it must override `hashCode` consistently: if two objects are `equals`, they must always have the same `hashCode`. Violations cause (1) `contains` failing to find objects that are present, except by luck, and (2) `add` inserting duplicates of equal items. (The converse implication, equal hash codes implying `equals`, is not required.)

### 5. Immutability identification

**Q:** Is this class immutable? If not, exhibit a mutation.
```java
public class SecretRocksBox {
  private Rock[] rocks;
  public SecretRocksBox(Rock[] rox) { rocks = rox; }
}
```
**A:** Not immutable. The constructor aliases the caller's array, so:
```java
Rock[] rox = {new Rock(10), new Rock(20)};
SecretRocksBox rb = new SecretRocksBox(rox);
rox[0] = new Rock(-999);     // observably changes rb's contents
```
Fix by defensively copying: allocate `rocks = new Rock[rox.length]` and `System.arraycopy(rox, 0, rocks, 0, rox.length)`.

### 6. Is `final` sufficient / necessary for immutability?

**Q:** True or false, with justification: declaring all fields `final` makes a class immutable.

**A:** False. `final` is neither sufficient nor necessary. Not sufficient: `RocksBox` has `public final Rock[] rocks`, but `rb.rocks[1] = null;` is an observable change, because `final` freezes the reference, not the array's contents. Not necessary: `Date`'s `private boolean contrived` field is non-final, yet `Date` is immutable since that field is unobservable from outside.

### 7. The mutable-key trace

**Q:** With a 4-bucket `HashSet<List<Integer>>`, we add `[0, 1]` (hash code 962), then `[2, 3]` (hash code 1026), then call `items.add(7)` on the first list, making its hash code 29829. Which call fails and why?

**A:** `hs.contains(items)` returns `false`. At insertion, 962 % 4 = 2, so the list was stored in bucket 2. After mutation its hash code is 29829, and 29829 % 4 = 1, so `contains` searches bucket 1, which is empty. The object is still in bucket 2, but no lookup will ever go there for it: the item is lost. General rule: never mutate an object used as a key.

### 8. Reduce step and negative hash codes

**Q:** `"đậu hũ".hashCode()` is `-2108180664`. Which bucket does it land in with M = 4, and why do we use `Math.floorMod` instead of `%`?

**A:** `Math.floorMod(-2108180664, 4) = 0`, so bucket 0. We use `floorMod` because Java's `%` preserves the sign of the dividend, so `%` on a negative hash code can produce a negative value, which is not a valid array index.

### 9. Java `HashMap` resizing arithmetic

**Q:** Java's `HashMap` starts with 16 buckets and a load factor of 0.75. At what N does the first resize occur, and what is the new array size?

**A:** 16 × 0.75 = 12, so inserting the 13th item triggers a resize, doubling the array to 32. The sequence continues N = 25 → 64, N = 49 → 128, N = 97 → 256, and so on, keeping N/M at or below 0.75.

### 10. Hash set vs. hash map vs. hash table

**Q:** Distinguish the three terms.

**A:** A hash table is a storage mechanism: M buckets holding N items, with each item's `hashCode` determining its bucket. A hash set is an implementation of the Set ADT built on a hash table. A hash map is an implementation of the Map ADT built on a hash table. In the real JDK, `HashSet` is literally implemented as a `HashMap<K, Object>` whose values are ignored.

### 11. Spread classification

**Q:** Classify each as no spread, bad spread, or good spread for `ColoredNumber`s 0 through 19 in 6 buckets: (a) `return 0;`, (b) sum of the digits of `num`, (c) `return num;`, (d) the default `Object.hashCode()`.

**A:** (a) No spread (everything in bucket 0). (b) Bad spread (digit sums for 0 through 19 only range over 0 to 10, clumping items). (c) Good spread (items distribute round-robin across the 6 buckets). (d) Good spread, since the memory address is effectively random. But note (d) is only *good spread*, not *correct*: paired with a `num`-based `equals` it breaks the hash table.

### 12. The "hard question"

**Q:** If Java's default `hashCode` already achieves good spread, why write custom hash functions?

**A:** Because spread is not the only requirement. A hash table finds an item by computing its bucket and searching only that bucket, so every object that is `equals` to the query must hash to the same bucket. The default memory-address hash code gives distinct-but-equal objects different hash codes, so lookups go to the wrong bucket. Custom `hashCode` methods exist primarily to stay consistent with a custom `equals`.

---

## Summary

- **Recap:** a hash table converts data to a hash code via `hashCode()`, reduces it to an index (lecture: `Math.floorMod(hc, M)`), and stores the item in that bucket's separate chain; it resizes when the load factor N/M exceeds a constant.
- **Runtimes:** bushy BST Θ(log N) for `contains` and `add`; separate chaining with **no** resizing Θ(N); separate chaining **with** resizing Θ(1) on average, assuming items are evenly spread.
- **Spread quality:** `return 0` = no spread; digit sum = bad spread; `return num` = good spread; the default memory-address hash code = good spread (addresses are effectively random).
- **The hard question's answer:** we write custom `hashCode` methods not for spread but for *consistency with `equals`*, since the table only searches one bucket.
- **The contract:** if you override `equals`, override `hashCode` consistently. Equal objects must always have equal hash codes. (The reverse is not required: collisions are fine.)
- **Violating it breaks two things:** `contains` finds a present item only with probability ~1/M (it looks in the wrong bucket and never even calls `equals`), and `add` silently inserts duplicates (~(M-1)/M of the time), destroying the Set invariant.
- **Correctness and speed are separate axes:** `return 0` is correct but Θ(N); a memory-address hash code with a custom `equals` is fast but wrong.
- **Immutable** = an instance cannot change in any **observable** way after instantiation. `final` is **neither sufficient** (`RocksBox`: a final reference to a mutable array) **nor necessary** (`Date`'s private non-final `contrived` field).
- `private` alone is not enough either: `SecretRocksBox` leaks because its constructor aliases the caller's array. Fix with **defensive copying** (`System.arraycopy` into a fresh array).
- **Immutability trade-off:** fewer bugs and easier debugging, at the cost of allocating a new object on every change (hence slow `String` concatenation).
- **Never mutate a key.** Mutating a key changes its hash code; the item stays in its old bucket while lookups compute a new one, so it is lost. `[0, 1]` (hash 962, bucket 2) mutated to `[0, 1, 7]` (hash 29829, bucket 1) makes `contains` return false.
- **Real Java:** `HashSet` delegates to `HashMap<K, Object>` ignoring values; array starts at 16 and doubles when N/M exceeds 0.75; the reduce step is a bit-mix plus mask, roughly `(hc ^ (hc >>> 16)) & (M - 1)`; the longest bucket in simulation is around 5 (the "balls into bins" problem, analyzed in CS70, shows the true worst case is slightly worse than constant); and overfull bins are converted into red-black trees, which is beyond the scope of the course.
- **Vocabulary:** hash table = the storage mechanism; hash set = Set ADT on a hash table; hash map = Map ADT on a hash table.
