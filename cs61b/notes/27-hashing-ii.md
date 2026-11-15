<!-- Wed, Oct 28, 2026 | sources: slides + code + YouTube auto-transcript + textbook -->
# Lecture 27: Hashing II

This lecture closes out hash tables by digging into the two halves of the hashing pipeline (the hash code and the reduce step) and showing exactly what breaks when the details are wrong. After recapping that a hash table gives Θ(1) average `contains` and `add` (assuming items spread evenly), we use a visualizer over `ColoredNumber` objects to see how different `hashCode` implementations change the distribution of items across buckets: returning `0` gives no spread, summing digits gives a bad spread, returning `num` or the default memory-address-based code gives a good spread. The core lesson is the contract between `hashCode` and `equals`: `hashCode` decides which bucket you look in, `equals` decides whether the item is in that bucket, and if you override one without the other, `contains` fails (except by luck, with probability 1/M) and `add` creates duplicates. We then look at immutability (what makes a class immutable, and why `final` is neither necessary nor sufficient) and the related disaster of mutating an object while it is being used as a hash table key, which silently loses the item. We finish with a peek at Java's real `HashSet`/`HashMap`: `HashSet` delegates to a `HashMap`, the table starts at 16 buckets and doubles when N/M exceeds a load factor of 0.75, the reduce step is `(hc ^ (hc >>> 16)) & (M - 1)`, the longest bucket in a load-factor-0.75 simulation is around 5, and overfull bins are converted into red-black trees.

## Key Concepts

### The two-stage pipeline, restated

Every hash table operation starts with the same mapping:

```
data  --hashCode()-->  hash code (an int)  --reduce-->  valid array index
```

Concretely from the slides: the string `đậu hũ` has `hashCode()` `-2108180664`, and `Math.floorMod(-2108180664, 4)` gives `0`, so it goes in bucket 0. Two separate problems live here:

- **Problem 1:** how do we turn arbitrary data into an array index? Answer: a hash function produces an `int`, then we reduce (mod) it into range.
- **Problem 2:** what if two items land on the same index? Answer: each bucket is a *separate chain* (a linked list) of items, and we resize when the load factor N/M exceeds some constant.

The lecturer stressed two mental pictures worth memorizing: hashing is a **mapping from a large (possibly infinite) set into a small fixed set**, and because it is *just a mapping*, there is no searching, no branching, no tree descent. That is the entire source of its speed. Separate chaining is an implementation detail for handling the collisions that mapping down to a small set necessarily creates.

### Runtimes

| | `contains(x)` | `add(x)` |
|---|---|---|
| Bushy (balanced) BST | Θ(log N) | Θ(log N) |
| Separate chaining hash table, **no** resizing | Θ(N) | Θ(N) |
| Separate chaining hash table **with** resizing | Θ(1)†  | Θ(1)\*† |

\* "on average" (resizing is occasionally expensive, amortized away, exactly like doubling an array).
† assuming items are spread evenly.

Both asterisks matter. Without resizing, buckets grow without bound and you degenerate back to linear scanning of a linked list. Without even spread, one bucket holds everything and you again degenerate to a linked list. The whole lecture is about keeping that second assumption true.

### What makes a hash code "good"

The lecture ranked candidate hash codes for a `ColoredNumber` (which has an `int num` and a `Color color`):

- **No spread:** `return 0;`. Extremely fast, deterministic, legal. Everything lands in bucket 0, so you have reinvented the linked list and `contains` is Θ(N).
- **Random:** `return new Random().nextInt();`. Lovely even distribution, zero bucket pileup, and completely useless: the same input hashes to a different bucket each call, so you can never find anything again. It violates determinism, which is non-negotiable.
- **Memory address (Java's default):** deterministic (tied to the specific object) *and* effectively random-looking, so you get good spread. This is the correct answer to the slide's multiple choice: (C) good spread.
- **Bad spread:** sum of the digits. Better than `0`, but clumps into a small range of values, so the distribution is lumpy.
- **Good spread:** `return this.num;` when the numbers themselves are well spread.

The lecturer added an important caveat beyond the slides: `return this.num` is only a good hash code *if the data is actually spread out*. If `num` were, say, "number of meals per day," values would cluster at 2 and 3, so almost everything would land in two buckets. **The quality of a hash code depends on the distribution of your data, not just on the code.** The goal is always "looks pseudo-random, but is deterministic."

He also showed empirically, by deleting the warm-up loop `for (int i = 0; i < 170; i += 1) { new Object().hashCode(); }` in `HashTableVisualizer`, that Java's default `hashCode` is *not* very random early in its sequence: bucket 3 was consistently near-empty and bucket 2 consistently full. Java's default hash code is good enough for large data sets but is not a cryptographic, collision-resistant hash. He tied this to a cautionary story about the RAND random number generator, where research results turned out to be artifacts of the generator rather than of the data. (The specific company/history anecdote is the lecturer's aside, not examinable material.)

### The hard question: why write custom hash functions at all?

If the default memory-address hash code already gives good spread, why bother? The answer is the rest of the lecture: **the default hash code is keyed to object identity, and that is usually not the notion of sameness you want.** Once you decide two distinct objects with the same `num` should count as "the same item," you must override `equals`, and the moment you override `equals` you are forced to override `hashCode` too, or the table breaks.

### hashCode and equals: the contract

This is the single most important takeaway from the lecture:

- **`hashCode` determines which bucket an item goes in.**
- **`equals` determines whether the item is in that bucket.**

They must be kept in sync. Formally: *if two objects are `equals`, they must always have the same `hashCode`.* The lecturer phrased it as "you cannot let hashCode and equals work asynchronously" and "you can't change one without changing the other." Java's own `hashCode` documentation says the same thing.

Why it breaks, in both directions:

1. **Override `equals`, not `hashCode`.** Two objects that your `equals` calls equal land in different buckets (because their memory addresses differ). So `contains` looks in the wrong bucket and returns `false` even though an equal object is present. And `add` happily inserts a duplicate, because the duplicate lands in some other bucket where `equals` never gets a chance to block it. From the slides: adding a second `ColoredNumber(0)` to a table with 6 buckets gives a 5/6 chance of a duplicate and only a 1/6 chance that it lands in the same bucket where `equals` catches it.
2. **Override `hashCode`, not `equals`.** Now two equal-in-spirit objects reliably land in the *same* bucket, but `contains` still walks that bucket calling `equals`, which still compares memory addresses, so it still returns `false`. The lecturer demonstrated exactly this live: with `hashCode` returning `num` but `equals` not overridden, `hs.contains(y)` printed `false` even though `x` and `y` were in the same bin.

Only when both are overridden consistently does everything print `true`.

### The general principle behind equals

The slides state it as: **equal objects should behave identically.** If `x.equals(y)`, then every observable thing about them (including `hashCode()`) should agree. Overriding `equals` is a promise about behavior, and `hashCode` is part of the behavior you are promising.

### Immutability

An **immutable** data type is one whose instance cannot change in any observable way after instantiation. Mutable: `ArrayDeque`, `Percolation`. Immutable: `Integer`, `String`, `Date` (as written in the slides' example).

The `final` keyword helps the compiler enforce immutability: a `final` variable may be assigned once (in the constructor or initializer) and never again. But the slides make a sharp point: **`final` is neither sufficient nor necessary for a class to be immutable.** It is not sufficient because a `final` reference to a mutable object (an array, say) still lets you mutate what it points to. It is not necessary because you could achieve immutability through other means (private fields with no mutators).

Trade-offs:

- **Advantage:** less to think about. Fewer bugs, easier debugging. The slides' analogy: an immutable class is a box with buttons to press and windows to look through, and the results are *always* the same no matter what happened before.
- **Disadvantage:** any change requires creating a new object. This is why Java `String` concatenation in a loop is slow.

### Mutable keys: the silent failure

You *can* build a `HashSet<List<Integer>>`. Weird things happen if you insert a `List` and then mutate it. Concretely from the slides, with 4 buckets:

1. `items = [0, 1]` has `hashCode()` `962`. `962 % 4 == 2`, so it lands in bucket 2.
2. Add `List.of(2, 3)`, whose `hashCode()` is `1026`. `1026 % 4 == 2`, so it also lands in bucket 2 (a collision, handled by the chain).
3. Now `items.add(7)`, so `items` is `[0, 1, 7]`, whose `hashCode()` is `29829`.
4. `hs.contains(items)` now computes `29829 % 4 == 1` and looks in bucket 1, which is **empty**. Returns `false`.

The item is still physically sitting in bucket 2, but the table can never find it again, because its *current* hash code no longer matches the bucket it was filed under. Bottom line from the slides: **never mutate an object being used as a key.** This is the same failure mode as the `hashCode`/`equals` desync, viewed through time instead of through two objects.

### A peek into Java's real HashSet

From reading the OpenJDK source (`java/util/HashSet.java` and `java/util/HashMap.java`):

- `HashSet` **delegates all of its work to a `HashMap<K, Object>`** and ignores the value.
- The hash table **starts at size 16**, and **doubles every time N exceeds the load factor**, which **defaults to 0.75**.
- The reduce function is not a plain mod. It is `(hc ^ (hc >>> 16)) & (M - 1)`. For M = 16, the slide shows `đậu hũ`'s hash code `-2108180664` reducing to index `15`. The `& (M - 1)` works as a mod because M is always a power of two; the `^ (hc >>> 16)` mixes the high bits down into the low bits so that high-bit information is not discarded by the mask. (The lecture called this "a bit complicated" and deferred details to office hours; the explanation of *why* the XOR-shift is there is extra context.)
- Observed resize points (printed using the reflection library): N = 13 → array size 32, N = 25 → 64, N = 49 → 128, N = 97 → 256, N = 193 → 512, N = 385 → 1024, N = 769 → 2048, N = 1537 → 4096. N/M is never more than 0.75.
- In a simulation keeping load factor at or below 0.75, the **longest bucket ends up around 5**. CS70 derives this precisely as the "balls into bins" problem, and that math shows the worst case runtime is a bit worse than constant.
- If a bin gets too full, **it is converted into a red-black tree** of `TreeNode`s, structured like `java.util.TreeMap`, for faster lookup when overpopulated. Well beyond the scope of the course. As Josh Hug puts it: "The most useful algorithms are, unfortunately, not always the most beautiful."

### Terminology (the FAQ slide)

- A **hash set** is an implementation of the Set ADT using a hash table as its engine.
- A **hash map** is an implementation of the Map ADT using a hash table as its engine.
- A **hash table** is the underlying storage scheme: M buckets holding N items, where each item's `hashCode` tells you which of the M buckets it goes in.

## Definitions

- **Hash table:** a storage scheme with M buckets holding N items, where each item's hash code determines which bucket it occupies.
- **Hash set:** an implementation of the Set ADT whose engine is a hash table.
- **Hash map:** an implementation of the Map ADT whose engine is a hash table.
- **Hash function / `hashCode()`:** a function mapping arbitrary data to an `int`. Must be deterministic; should spread values out.
- **Hash code:** the `int` returned by the hash function, before reduction. Can be negative.
- **Reduce (step):** converting a hash code into a valid array index, e.g. `Math.floorMod(hashCode, M)`, or in real Java `(hc ^ (hc >>> 16)) & (M - 1)`.
- **Bucket / bin:** one slot of the hash table's backing array.
- **Separate chaining:** the collision-resolution strategy where each bucket holds a chain (linked list) of all items hashing to that index.
- **Collision:** two items reducing to the same bucket index.
- **Load factor:** N/M, the number of items divided by the number of buckets. Java's `HashMap` default maximum is 0.75.
- **Resizing:** growing the bucket array (Java doubles it) when the load factor exceeds the threshold, to keep chains short.
- **`equals` contract (as used here):** equal objects should behave identically; in particular, if `a.equals(b)` then `a.hashCode() == b.hashCode()`.
- **Immutable data type:** a type for which an instance cannot change in any observable way after instantiation.
- **`final` variable:** a variable that may be assigned a value once (in the constructor or an initializer) and can never change afterwards. Neither sufficient nor necessary for immutability.
- **Defensive copy:** constructing your own internal copy of a passed-in mutable object (e.g. via `System.arraycopy`) so external code cannot mutate your state.
- **Balls into bins problem:** the probabilistic question of how full the fullest bin gets when N balls are thrown into M bins at random; used to analyze the longest chain. (Covered in CS70.)

## Worked Examples

### 1. The `ColoredNumber` class and the visualizer

The object being inserted into the table:

```java
public class ColoredNumber {
    private static final List<Color> colors =
            List.of(Color.WHITE, Color.ORANGE, Color.CYAN, Color.GREEN, Color.YELLOW);

    public int num;
    public Color color;

    public ColoredNumber(int n) {
        num = n;
        color = getRandomColor();
    }
    // draw(...) and getRandomColor() omitted
}
```

Note what is *absent*: there is no `equals` and no `hashCode`. So both come from `Object`: `equals` is reference equality (same memory address) and `hashCode` is the default identity hash code.

The visualizer inserts `ColoredNumber`s 0 through 19 into a table with M = 6 buckets:

```java
public class HashTableVisualizer {
    List<ColoredNumber> cns = new ArrayList<>();
    int N = 20; // 0, 1, 2, ... 19
    int M = 6;

    private void addAndVisualize(int n) {
        ColoredNumber cn = new ColoredNumber(n);
        IO.println("%d, %d".formatted(cn.hashCode(), Math.floorMod(cn.hashCode(), M)));
        cns.add(cn);
        visualize(cns, M, scale);
        StdDraw.pause(pauseTime);
    }
```

and the bucket for each item is computed exactly as in the slides' diagram:

```java
int bucketNumber = Math.floorMod(s.hashCode(), M);
```

`Math.floorMod` rather than `%` because hash codes are frequently negative and `%` in Java can return a negative result, which would be an invalid array index.

**Run 1 observations (default `hashCode`):** different-colored numbers share bins; the distribution is roughly even across bins (chains of about 2 to 5); sequential items occasionally land in the same bin. Each run looks different. The reason: the default `hashCode` is effectively (pseudo-)random yet deterministic per object, because it is derived from the object's identity (we can pretend it is the memory address).

A subtlety in the visualizer worth understanding, because it is where the duplicate-detection behavior comes from:

```java
/** Simulate the deduplicating effect of a hash set */
private static List<ColoredNumber> preprocess(List<ColoredNumber> oomages) {
    List<ColoredNumber> newList = new ArrayList<>();
    for (ColoredNumber element : oomages) {
        if (newList.contains(element)) {
            newList.remove(element);
        }
        newList.add(element);
    }
    return newList;
}
```

`ArrayList.contains` uses `equals`. So whether a re-inserted item appears as a duplicate in the drawing depends entirely on whether `ColoredNumber` overrides `equals`, which is exactly the mechanism the lecture is probing.

### 2. The zero hash code

```java
@Override
public int hashCode() {
    return 0;
}
```

Every item's hash code is `0`, and `Math.floorMod(0, 6) == 0`, so every single item lands in bucket 0. The resulting picture is one enormous chain in bin 0 and five empty bins. Searching for, say, "white 5" means walking the entire chain. We are back to a linked list with Θ(N) `contains`.

On the `@Override` annotation: the lecturer asked whether it is required. It is **not** required for the override to take effect, but it is good practice. It documents intent, and if you typo the method name (say `hashcode()`), the compiler complains that you are not overriding anything from the superclass instead of silently defining a brand-new method that Java never calls. It reduces the number of ways to shoot yourself in the foot.

### 3. Designing a hash code to produce a given distribution

Target distribution: bin 0 holds 0, 6, 12, 18; bin 1 holds 1, 7, 13, 19; bin 2 holds 2, 8, 14; and so on.

The class answer:

```java
@Override
public int hashCode() {
    return num % 6;
}
```

Each bucket then holds all numbers congruent to its index mod 6: bin 0 gets multiples of 6, bin 1 gets multiples of 6 plus 1, bin 2 gets multiples of 6 plus 2, etc.

Then the follow-up: what if we drop the `% 6`?

```java
@Override
public int hashCode() {
    return num;
}
```

The distribution is **identical**. Why? Because the pipeline already applies a reduce step: the visualizer computes `Math.floorMod(cn.hashCode(), M)` with M = 6. Taking mod 6 inside `hashCode` and then mod 6 again in the reduce step is the same as taking mod 6 once (`(num % 6) % 6 == num % 6`). Keep the picture in mind: `hashCode` converts data to an `int`, and the *table* does the modding. You do not need to mod yourself. Someone suggested `num % M`; M is not in scope inside `ColoredNumber`, and anyway the table's own reduce handles it.

Other hash codes you could legally define (the lecture lists them as exercises): leading digit, sum of the digits, length of the number. All of them are valid hash codes. They are not equally good.

### 4. `ContainsDemo`: the default hash code, no overrides

```java
public class ContainsDemo {
    public static void main(String[] args) {
        HashSet<ColoredNumber> hs = new HashSet<>();
        ColoredNumber x = new ColoredNumber(10);
        hs.add(x);

        IO.println(hs.contains(x)); // true

        ColoredNumber y = new ColoredNumber(10);
        IO.println(hs.contains(y)); // false

        IO.println(x.equals(y));    // false
    }
}
```

Step by step (the slides use `4` rather than `10`; the behavior is the same):

1. `new ColoredNumber(4)` is created; call its hash code `445051633`. `Math.floorMod(445051633, 4) == 1`, so `x` is stored in bucket 1.
2. `hs.contains(x)`: hashes the *same object*, gets the *same* hash code, reduces to bucket 1, walks bucket 1, finds an entry, and `x.equals(x)` is trivially true under reference equality. Prints **true**.
3. `ColoredNumber y = new ColoredNumber(4)` creates a **different object at a different memory address**. Box-and-pointer view: `x` and `y` are two separate boxes on the heap, each with `num = 4`, and the two local variables point at different boxes. Different address means a different default hash code, say `1349277854`, which reduces to `Math.floorMod(1349277854, 4) == 2`.
4. `hs.contains(y)` looks in bucket **2**, which is empty. Prints **false**.
5. `x.equals(y)` uses `Object.equals`, which is `x == y`, which is false because they are different boxes. Prints **false**.

The slides also show the lucky variant: a third object `z = new ColoredNumber(4)` with hash code `1775282465` reduces to bucket 1, so `contains(z)` *would* walk the correct bucket. But even then it only returns true if `equals` says so. With no `equals` override it still fails. With `equals` overridden but `hashCode` not, you get the headline result: **`contains` returns `true` with probability 1/(number of buckets), and `false` otherwise.** A deterministic algorithm that is probably wrong, which, as the lecturer said, is quite strange.

### 5. Overriding `equals` only

```java
@Override
public boolean equals(Object o) {
    if (o instanceof ColoredNumber otherCn) {
        return this.num == otherCn.num;
    }
    return false;
}
```

Two points of Java mechanics the lecturer walked through live. First, the parameter must be `Object`, not `ColoredNumber`, or you are *overloading* rather than overriding and the collections framework will never call your method. Second, you cannot write `o.num` directly, because the compiler only knows `o` is an `Object` and `Object` has no `num` field. The pattern-matching `instanceof` (`o instanceof ColoredNumber otherCn`) both checks the type and binds a correctly-typed variable `otherCn` in one step. Anything that is not a `ColoredNumber` returns `false`.

Now run the same `contains` test, with `equals` overridden but `hashCode` left as the default:

```java
HashSet<ColoredNumber> hs = new HashSet<>();
ColoredNumber x = new ColoredNumber(10);
hs.add(x);
IO.println(hs.contains(x)); // true
ColoredNumber y = new ColoredNumber(10);
IO.println(hs.contains(y)); // false  (true only with probability 1/M)
IO.println(x.equals(y));    // true
```

So we have the bizarre state of affairs that `x.equals(y)` is `true`, yet the set containing `x` reports that it does not contain `y`. The reason is structural, not logical: `equals` is never consulted, because the lookup went to the wrong bucket.

**And the duplicate problem.** With the same setup (slide question), adding a second `ColoredNumber(0)` to a 6-bucket table holding 0 through 19:

- 5/6 chance: the new zero's random identity hash lands it in bin 1, 2, 3, 4, or 5 (any bin other than the one holding the original zero), where no equal item exists, so it is inserted. **Duplicate.**
- 1/6 chance: it lands in the bin holding the original zero, `equals` fires, and insertion is blocked. **No duplicate.**

A set that sometimes contains two equal elements is not a set.

### 6. Overriding both, consistently

```java
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
```

Now `x` and `y` with `num = 10` have the same hash code, so they reduce to the same bucket; `contains(y)` walks that bucket and `equals` confirms a match. The lecture's demo prints `true`, `true`, `true`. The two methods agree on what "the same item" means: `hashCode` routes equal items to the same bucket, `equals` recognizes them once there.

(Extra context: a `hashCode` based on `num` while `equals` is based on `num` is consistent. The reverse-direction requirement does *not* hold: unequal objects are allowed to share a hash code, that is just a collision, and collisions are fine. Only "equal implies same hash code" is required. The commented-out variant in `ColoredNumber.java` where `equals` compares `color` while `hashCode` returns `num % 6` would be inconsistent: two `ColoredNumber`s could be equal by color yet hash to different buckets.)

### 7. Immutability: which classes are immutable?

```java
public class Pebble {
    public int weight;
    public Pebble() { weight = 1; }
}
```

**Mutable.** `weight` is a non-final public field:

```java
Pebble p = new Pebble();
p.weight = 2;
```

```java
public class Rock {
    public final int weight;
    public Rock(int w) { weight = w; }
}
```

**Immutable.** `weight` is `final` and an `int` (a primitive, so there is no referenced object to tamper with). No mutation possible, unless you use the Reflections library to disobey access modifiers.

```java
public class RocksBox {
    public final Rock[] rocks;
    public RocksBox(Rock[] rox) { rocks = rox; }
}
```

**Mutable**, and this is the instructive case. `rocks` is `final`, so you cannot reassign the *reference*, but you can absolutely change what it points at:

```java
Rock r1 = new Rock(10);
Rock r2 = new Rock(20);
Rock[] rox = {r1, r2};
RocksBox rb = new RocksBox(rox);
rb.rocks[1] = null;   // mutation!
```

Box-and-pointer reasoning: `rb.rocks` is a pointer to an array object. `final` freezes the arrow, not the array it points to.

```java
public class SecretRocksBox {
    private Rock[] rocks;
    public SecretRocksBox(Rock[] rox) { rocks = rox; }
}
```

**Mutable**, and even more subtly. The field is `private`, so no outside code can touch `rb.rocks`. But the constructor *aliased* the caller's array: `rocks` and the caller's `rox` point at the **same** array object. So the caller retains a handle on your internals:

```java
Rock r1 = new Rock(10);
Rock r2 = new Rock(20);
Rock[] rox = {r1, r2};
SecretRocksBox rb = new SecretRocksBox(rox);
rox[0] = new Rock(-999);   // observably changes rb's state
```

**The fix: a defensive copy.**

```java
public class SecretRocks {
    private Rock[] rocks;
    public SecretRocks(Rock[] rox) {
        rocks = new Rock[rox.length];
        System.arraycopy(rox, 0, rocks, 0, rox.length);
    }
}
```

Now `rocks` points to our own private array. Mutating the caller's `rox` afterwards has no effect, so the example mutation fails. This is also the concrete demonstration that `final` is neither sufficient (`RocksBox`) nor necessary (`SecretRocks` has no `final` and is immutable) for immutability.

### 8. `MutableKeysDemo`: losing an item forever

```java
public class MutableKeysDemo {
    public static void main() {
        List<Integer> items = new ArrayList<>();
        items.add(1);
        items.add(1);
        IO.println(items.hashCode());

        HashSet<List<Integer>> hs = new HashSet<>();
        hs.add(items);
        hs.add(List.of(2, 3));
        IO.println(hs.contains(items));   // true

        items.add(7);
        IO.println(hs.contains(items));   // false
        IO.println(items.hashCode());     // changed!
    }
}
```

The slides use `[0, 1]` and a 4-bucket table, so trace that version:

1. `items = [0, 1]`. `items.hashCode()` is `962`. Note that `List`'s `hashCode` is defined by *contents*, not identity, which is precisely why this is dangerous: content changes, hash changes.
2. `hs.add(items)`: `962 % 4 == 2`, so a reference to the list object is stored in bucket 2.
3. `hs.add(List.of(2, 3))`: hash code `1026`, and `1026 % 4 == 2`, so it joins the chain in bucket 2. Fine, that is an ordinary collision.
4. `items.add(7)`. In box-and-pointer terms **nothing in the table moved**: bucket 2's chain still points at the very same list object. But that object's contents are now `[0, 1, 7]`, so its `hashCode()` is `29829`.
5. `hs.contains(items)`: the table recomputes `items.hashCode()`, gets `29829`, reduces `29829 % 4 == 1`, and looks in bucket **1**, which is empty. Returns **false**.

The list is in the set and unreachable from the set. The index under which it was filed is stale. Bottom line: **never mutate an object being used as a key.** You get incorrect results and the item gets lost.

(Extra context: this is why immutable types such as `String` and `Integer` make excellent hash keys, and why Java's `record` types are a convenient way to get a value class with sensible `equals`/`hashCode` and no mutators.)

## Common Pitfalls

- **Overriding `equals` without `hashCode`.** The single most common bug this lecture exists to prevent. Symptoms: `contains` returns `false` for items that are present, and `add` creates duplicates. Think "5/6 chance of a duplicate."
- **Overriding `hashCode` without `equals`.** Items route to the right bucket, but `equals` still compares memory addresses, so lookups still fail. Both must change together.
- **Writing `public boolean equals(ColoredNumber o)`.** That is an overload, not an override. Java's collections call `equals(Object)`, so your method is never invoked and you get silent reference-equality behavior. Use `@Override` and it won't compile, which is the point of the annotation.
- **Accessing `o.num` inside `equals` without a type check.** Will not compile; `Object` has no `num`. Use the pattern-matching `instanceof`. Also remember to return `false` for non-matching types rather than throwing.
- **Mutating an object after using it as a key.** Silently loses the item. Also applies to putting a mutable object in a `HashSet` and handing a reference to someone else who might mutate it.
- **Using `%` instead of `Math.floorMod` to reduce.** Hash codes are often negative, and Java's `%` preserves the sign, so you can get a negative array index and an `ArrayIndexOutOfBoundsException`.
- **Modding inside `hashCode`.** Harmless but redundant: the table reduces for you. Worse, hard-coding `% 6` inside `hashCode` ties your class to a table size it should know nothing about, and the table resizes.
- **Thinking `final` makes a class immutable.** A `final` reference to a mutable object (array, `ArrayList`) is still mutable through that reference. Conversely, immutability can be achieved without `final`.
- **Aliasing a constructor parameter into a private field** and calling it encapsulated. The caller still holds a reference. Make a defensive copy.
- **Believing a hash table is unconditionally Θ(1).** It is Θ(1) *on average* and *assuming even spread*, which depends on resizing and on a decent hash function matched to your data's distribution.
- **Assuming a "reasonable-looking" hash code gives good spread.** `return num` is good for 0..19 and terrible for "meals per day." Think about the actual data distribution.
- **Trusting a pseudo-random source to be random over short runs.** The lecturer's live demo showed Java's identity hash codes looking distinctly non-uniform for the first ~170 objects.

## Likely Exam Points

### 1. The hashCode/equals contract

**Q:** You override `equals` on a class so two instances are equal when their `name` fields match, but you do not override `hashCode`. You add one such object to a `HashSet` with 8 buckets, then call `contains` with a *different but `equals`-matching* object. What is the probability the call returns `true`, and why?

**A:** 1/8. The default `hashCode` is identity-based, so the second object's hash code is unrelated to the first object's. `contains` reduces that hash code and examines exactly one bucket. With 8 buckets it lands in the bucket holding the original item with probability 1/8, and only then does `equals` get a chance to report a match. Otherwise it examines the wrong (likely empty) bucket and returns `false`. The rule: if two objects are `equals`, they must have the same `hashCode`.

### 2. Duplicates from an inconsistent override

**Q:** A 6-bucket `HashSet<ColoredNumber>` already contains `ColoredNumber`s 0 through 19. `equals` compares `num`; `hashCode` is not overridden. You call `hs.add(new ColoredNumber(0))`. Which outcomes are possible and with what probabilities?

**A:** The new object's identity hash sends it to a uniformly random bin. With probability 1/6 it lands in the bin already holding the original 0, where `equals` detects the duplicate and nothing is added. With probability 5/6 it lands in one of the other five bins and is inserted, so the set now contains two elements that are `equals` to each other. A set is not supposed to be able to do that.

### 3. Identifying a hash code from a distribution (or vice versa)

**Q:** `ColoredNumber`s 0 through 19 are inserted into a table with M = 6. Bin 0 contains 0, 6, 12, 18; bin 1 contains 1, 7, 13, 19; bin 2 contains 2, 8, 14; bin 3 contains 3, 9, 15; bin 4 contains 4, 10, 16; bin 5 contains 5, 11, 17. Give two different `hashCode` implementations consistent with this picture, and explain why both work.

**A:** `return num;` and `return num % 6;`. Both work because the table applies its own reduce step `Math.floorMod(hashCode(), 6)`. With `return num`, bucket = `num % 6` directly. With `return num % 6`, bucket = `(num % 6) % 6`, which equals `num % 6`. Modding inside `hashCode` is redundant when the table mods with the same M.

### 4. Runtime table

**Q:** Fill in worst-case/average runtimes for `contains` and `add` for: a bushy BST; a separate-chaining hash table with no resizing; a separate-chaining hash table with resizing. State all assumptions.

**A:** Bushy BST: Θ(log N) for both. Hash table with no resizing: Θ(N) for both, because N/M grows without bound so chains become length Θ(N). Hash table with resizing: Θ(1) for `contains` and Θ(1) amortized/average for `add`, both assuming items are spread evenly. The `add` average is because resizing occasionally costs Θ(N) but is amortized across insertions, exactly like array doubling.

### 5. Mutable keys

**Q:** With a 4-bucket table: `List<Integer> a = new ArrayList<>(List.of(0, 1));` (`hashCode()` = 962) is added to a `HashSet`. Then `a.add(7)`, making `a.hashCode()` = 29829. What does `hs.contains(a)` return, and where is the list actually stored?

**A:** It returns `false`. The list was filed in bucket `962 % 4 == 2` and the table still holds a reference to it there, but `contains` recomputes the hash of the *current* contents, gets `29829 % 4 == 1`, and inspects bucket 1, which is empty. The object is in the set but permanently unreachable through the set. Never mutate an object used as a key.

### 6. Immutability

**Q:** Is the following class immutable? Justify, and if not, fix it.

```java
public class Roster {
    private final String[] names;
    public Roster(String[] ns) { names = ns; }
    public String[] getNames() { return names; }
}
```

**A:** Not immutable, for two independent reasons. (1) The constructor aliases the caller's array, so the caller can mutate `names` from outside via its own reference. (2) `getNames` hands out a reference to the internal array, so any client can write into it. `final` only prevents reassigning the `names` pointer, not mutating the array it points at. Fix both ends with defensive copies: in the constructor, `names = new String[ns.length]; System.arraycopy(ns, 0, names, 0, ns.length);` and in the getter, return a copy rather than `names`. (The `String` elements themselves are already immutable, so copying the array suffices.)

### 7. Evaluating candidate hash codes

**Q:** Rank these `hashCode` implementations for an object with an `int num`, from worst to best spread, and say which one is outright invalid as a hash code: (a) `return 0;` (b) `return new Random().nextInt();` (c) sum of the digits of `num` (d) `return num;`

**A:** (b) is **invalid**: it is not deterministic, so the same object hashes to a different bucket on every call and can never be found again (even though its distribution is beautifully uniform). Among the valid ones, worst to best spread: (a) no spread, everything in bucket 0, degenerating to a linked list; (c) bad spread, digit sums cluster into a narrow range; (d) good spread, *provided `num` values are themselves well spread*. If `num` were clustered (e.g. values only ever 2 or 3), (d) would also be a poor choice.

### 8. Java's HashMap internals

**Q:** Java's `HashMap` starts with 16 buckets and a load factor of 0.75. At what value of N does it first resize, and to what size? What is special about keeping M a power of two?

**A:** It resizes when N exceeds 0.75 × 16 = 12, i.e. at N = 13, doubling the array to 32. (Subsequent resizes: N = 25 → 64, N = 49 → 128, and so on.) Keeping M a power of two lets the reduce step use the bitmask `& (M - 1)` instead of a slower `%`. The accompanying `hc ^ (hc >>> 16)` mixes high bits into the low bits first, so that the mask (which keeps only low bits) does not throw away all the information in the high half of the hash code. (The explanation of why the XOR-shift is needed is extra context; the lecture noted only that the reduce function "is a bit complicated.")

### 9. The `@Override` annotation

**Q:** Is `@Override` required to override `hashCode`? What does it buy you?

**A:** Not required. It is a compile-time check: if your method does not actually override anything in a supertype (a typo like `hashcode()`, or a wrong parameter type like `equals(ColoredNumber)`), the compiler reports an error instead of silently creating a new, never-called method. It also documents intent. Use it.

## Summary

- Hashing is a **deterministic mapping** from a large set into a small fixed set of M buckets: `data → hashCode() → int → reduce → index`. There is no searching involved, which is why it is fast.
- Separate chaining resolves collisions; resizing when N/M exceeds a constant keeps chains short. Result: Θ(1) `contains` and Θ(1) average `add`, **assuming even spread**. Without resizing or without even spread, you degenerate to Θ(N).
- A hash code must be **deterministic** (so `new Random().nextInt()` is disqualified despite perfect spread) and should **spread items evenly**. `return 0` is no spread, digit-sum is bad spread, `return num` and the identity-based default are good spread.
- Whether a hash code is good depends on your **data's distribution**, not the code alone.
- **`hashCode` picks the bucket; `equals` decides membership within the bucket.** They must stay in sync: if two objects are `equals`, they must have equal hash codes.
- Override `equals` without `hashCode` → `contains` fails except with probability 1/M, and `add` creates duplicates (5/6 chance in a 6-bucket table). Override `hashCode` without `equals` → lookups still fail, because `equals` still compares addresses.
- Write `equals(Object o)` with a pattern-matching `instanceof` and `@Override`; `equals(MyType o)` is an overload Java will never call.
- An **immutable** type cannot change observably after construction. `final` is neither sufficient (a `final` array reference still lets you mutate the array) nor necessary. Fix aliasing with **defensive copies** (`System.arraycopy`).
- Immutability trades performance (new object per change, so `String` concatenation is slow) for a large reduction in reasoning burden and bugs.
- **Never mutate an object used as a key.** Its hash code changes, the table keeps looking in the new bucket, and the item becomes unreachable (`[0,1]` at bucket 2 becomes `[0,1,7]` hashing to bucket 1, which is empty).
- Java's real implementation: `HashSet` delegates to `HashMap<K, Object>`; table starts at 16 and doubles when N/M > 0.75; reduce is `(hc ^ (hc >>> 16)) & (M - 1)`; longest chain at load factor 0.75 is about 5 (the balls-into-bins problem, CS70); overfull bins become red-black trees.
- Terminology: a **hash table** is the storage scheme (M buckets, N items); a **hash set** is the Set ADT built on it; a **hash map** is the Map ADT built on it.
