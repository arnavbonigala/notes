<!-- Fri, Sep 18, 2026 | sources: code (no transcript available) -->
# Lecture 10: Inheritance 2

## Overview

This lecture is about the second half of Java's inheritance story: how we use interfaces not just to describe "is-a" relationships, but to plug our own classes into machinery that Java (or someone else) already wrote. Two threads run through it. The first thread, from the lecture code (`ArraySet.java`, `IteratorDemo.java`), is **making our own data structure behave like a built-in one**: implementing `Iterable<T>` so that the enhanced for loop (`for (int i : aset)`) works, writing the `Iterator` object that actually walks the array, and overriding the `Object` methods `toString()` and `equals()` so that printing and comparing our set do something sensible. The second thread, from the textbook chapter, is **comparison and generic library code**: Python compares objects with operator overloading (`__gt__`) and handles alternate orders with function passing (`key=...`), while Java does both with **subtype polymorphism** instead, packaging "how to compare" into a `Comparable` implementation (one natural order, intrinsic) or a `Comparator` object (many orders, extrinsic). The unifying idea is that in Java, capability is declared by name with `implements`, not inferred from having the right methods, and once you declare it, library code like `Collections.max` or the for-each loop will call back into *your* overridden methods at runtime.

---

## Key Concepts

### 1. Subtype polymorphism: the supertype specifies, the subtype supplies

Polymorphism is, per the textbook's Wikipedia definition, "the ability in programming to present the same programming interface for differing underlying forms."

In **subtype polymorphism**, that shared interface is a supertype:

- A supertype (`Comparable`, `Iterable`, `Object`) declares a capability as a method signature.
- A subtype (`Dog`, `ArraySet`) overrides that method with its own implementation.
- At runtime, Java picks which implementation to run based on the **dynamic type** of the object that invoked the method.

This is the mechanism behind everything in this lecture. `Collections.max` was compiled years before your `Dog` class existed, but it can still sort your dogs, because it only ever says `a.compareTo(b)` and dynamic method selection does the rest.

### 2. Java is nominally typed; Python is duck typed

The comments in `ArraySet.java` hammer this point:

> "In some programming languages, it would just check automatically, but in Java, hypernym/hyponym relationships require the use of the `implements` keyword."

Writing an `iterator()` method in `ArraySet` is **not enough** to make `for (int i : aset)` compile. Java checks the *declared* type relationship, not the presence of the right methods. You must literally write `implements Iterable<T>` so the compiler knows `ArraySet` is-a `Iterable`. In Python you would just define `__iter__` and everything would work, because Python only checks at runtime whether the method happens to exist ("if it walks like a duck...").

The same story applies to comparison. Python's `get_the_max` works on any type because `>` dispatches to `__gt__` at runtime and nobody ever declared anything. Java's `Collections.max(dogs)` refuses to compile unless `Dog` formally `implements Comparable<Dog>`, and the error message is the famously awful "no instance(s) of type variable(s) T exist so that Dog conforms to Comparable<? super T>".

### 3. `Iterable` and `Iterator` are two different interfaces

This is the most commonly confused pair in the lecture. Their (simplified) definitions:

```java
public interface Iterable<T> {
    Iterator<T> iterator();
}

public interface Iterator<T> {
    boolean hasNext();
    T next();
}
```

- **`Iterable`** means "you can ask me for an iterator." It is the thing you loop *over*. `ArraySet` implements this.
- **`Iterator`** means "I am a cursor with a current position, and I can tell you if there's more and hand you the next item." It is the helper object doing the walking. `ArraySet.MagicWizard` implements this.

The separation matters: the collection holds the data, the iterator holds the *position*. You can have three iterators walking the same `ArraySet` at once, each with its own `wizPos`, because each is a separate object.

### 4. The enhanced for loop is syntactic sugar

`IteratorDemo.java` makes this explicit. These two snippets are, in the lecture's words, "EXACTLY THE SAME":

```java
for (int i : javaset) {
    System.out.println(i);
}
```

```java
Iterator<Integer> seer = javaset.iterator();
while (seer.hasNext()) {
    int x = seer.next();
    System.out.println(x);
}
```

The compiler literally rewrites the first into the second. That is why `implements Iterable<T>` is the "magic ingredient so that `:` works properly": the desugared form calls `.iterator()`, and the compiler will only emit that call if the static type is known to be `Iterable`.

### 5. Overriding `Object`'s methods: `toString` and `equals`

Every class in Java implicitly extends `Object`, which provides default implementations:

- `toString()` returns something like `lec10_inheritance2.ArraySet@2f92e0f4` (class name, `@`, hex hash code). Useless for debugging.
- `equals(Object o)` returns `this == o`, i.e. pure reference equality. Two distinct `ArraySet` objects holding identical contents would be "not equal."

`System.out.println(aset)` implicitly calls `aset.toString()`, so overriding `toString` immediately improves every print statement. Overriding `equals` is what makes `aset.equals(aset2)` return `true` for two separately-built sets with the same contents.

Critically, `equals` must take an **`Object`** parameter to actually override. `public boolean equals(ArraySet o)` would be an *overload*, a brand new unrelated method, and library code (which only knows about `Object`) would never call it.

### 6. `instanceof` pattern matching

```java
if (o instanceof ArraySet otherArraySet) {
```

The lecture comments describe this precisely: `instanceof` here does two things.

1. It returns `true` if `o`'s dynamic type is `ArraySet` (or a subtype).
2. It "reincarnates" `o` under a new name, `otherArraySet`, whose **static type is `ArraySet`**, so you can call `ArraySet` methods on it.

Without the pattern variable you would have to write the old two-step dance:

```java
if (o instanceof ArraySet) {
    ArraySet otherArraySet = (ArraySet) o;   // explicit cast
    ...
}
```

The pattern variable is scoped to where the check is known to have succeeded (the body of the `if`), so it is both shorter and safer.

### 7. Comparable: one intrinsic "natural order"

```java
public interface Comparable<T> {
    int compareTo(T o);
}
```

`compareTo` returns a negative integer, zero, or a positive integer as `this` is less than, equal to, or greater than the argument. Note the contract is about the **sign**, not the magnitude.

```java
public class Dog implements Comparable<Dog> {
    @Override
    public int compareTo(Dog uddaDog) {
        return this.size - uddaDog.size;
    }
}
```

The subtraction trick is idiomatic and short. The ordering it defines is the class's **natural order**, and there can be exactly one of them, because there is exactly one `compareTo` method.

### 8. Comparator: many extrinsic orders

```java
public interface Comparator<T> {
    int compare(T o1, T o2);
}
```

Note the shape difference: `compare` is a **two-argument** method living in a *separate* class, not a one-argument method living in `Dog`. That is exactly what lets you have many of them.

```java
public static class NameComparator implements Comparator<Dog> {
    @Override
    public int compare(Dog a, Dog b) {
        return a.name.compareTo(b.name);
    }
}
```

This is Java's answer to Python's `key=` function. Python passes a function; Java wraps the function in an object and passes the object. Same idea, different mechanism: **function passing** versus **subtype polymorphism**.

A small ergonomic fix from the textbook: since a `NameComparator` has no state, you never need more than one, so stash a single instance as a constant:

```java
public class Dog {
    public static final Comparator<Dog> NAME_COMPARATOR = new NameComparator();
}
```

Now callers write `Collections.max(dogs, Dog.NAME_COMPARATOR)` instead of `new Dog.NameComparator()`. The textbook explicitly leaves it to you whether this is actually an improvement.

### 9. Generic static methods and type bounds

To write library functions like `max` yourself, you need generics on the *method*, not the class:

```java
public static <T> T pickRandom(T[] x)
```

Read as: "I am declaring a public static function that works on objects of type T, it returns a T, it is called `pickRandom`, and it takes an array of Ts as input."

Why not just make the *class* generic? Because `public class RandomPicker<T>` forces you to instantiate a `RandomPicker<String>` object just to pin down `T`, and you cannot call a static method through an instance anyway. Making the class generic and the method non-static works but is awkward. The generic static method is the right tool.

A bonus convenience: when calling a generic static method, you do **not** write the type argument. `RandomPicker.pickRandom(x)` is enough; `T` is inferred from the argument.

When the method body needs to *do* something with `T`, a plain `<T>` is not enough, because as far as the compiler is concerned `T` could be anything, and `Object` has no `compareTo`. The fix is a **type bound**:

```java
public static <T extends Comparable<T>> T max(T[] items)
```

Read as: "...and additionally, T has to implement `Comparable`." Passing an array of non-`Comparable` objects is now a **compile-time** error, not a runtime surprise.

Note the keyword: `extends` is used for type bounds even when the bound is an interface you would normally `implement`. That is just Java's syntax.

### 10. Generics do not work with primitives

`pickRandom` and `max` work on `String[]`, `Dog[]`, `Integer[]`, but not `int[]`, `double[]`, or `char[]`. Generic type parameters can only be bound to reference types. There is no way around it, which is why the real Java library ships separate overloads: `Arrays.sort(int[])`, `Arrays.sort(double[])`, `Arrays.sort(float[])`, and so on. (The textbook mentions Project Valhalla as the possible eventual fix.)

---

## Definitions

- **Polymorphism**: "The ability in programming to present the same programming interface for differing underlying forms" (Wikipedia, as quoted in the textbook).
- **Subtype polymorphism**: Polymorphism achieved by having a supertype declare a capability and subtypes override it, with the implementation chosen at runtime by the object's dynamic type.
- **Operator overloading**: Defining what built-in operators mean for your type. Python does this via dunder methods like `__gt__`. Java does **not** have operator overloading.
- **Function passing**: Passing a function itself as an argument (Python's `key=name_len`). Java code typically does not do this for ordering; it passes a `Comparator` object instead.
- **Duck typing**: A type system in which an object's usability is determined by whether it happens to have the needed methods, checked at runtime. Python is duck typed; Java is not.
- **`implements`**: The Java keyword that formally declares a hyponym/hypernym (is-a) relationship between a class and an interface. Required; Java will not infer it from method presence.
- **`Iterable<T>`**: The interface declaring a single method `Iterator<T> iterator()`. Implementing it is what enables the enhanced for loop over your class.
- **`Iterator<T>`**: The interface declaring `boolean hasNext()` and `T next()`. An object that tracks a position within a collection and doles out elements one at a time.
- **Enhanced for loop (for-each)**: `for (T x : iterable)`, syntactic sugar that the compiler rewrites into a call to `iterator()` plus a `while (hasNext()) { ... next() ... }` loop.
- **`toString()`**: The `Object` method returning a `String` representation, called implicitly by `System.out.println` and by string concatenation. Default is `ClassName@hashcode`.
- **`equals(Object o)`**: The `Object` method defining logical equality. Default implementation is reference equality (`this == o`). Must take an `Object` parameter to override rather than overload.
- **`instanceof` pattern matching**: The form `o instanceof Type name`, which tests the dynamic type and, on success, binds `name` as a variable of that type, avoiding an explicit cast.
- **`Comparable<T>`**: Interface with `int compareTo(T o)`. Implemented **by** the class being compared; defines its single natural order.
- **Natural order**: The ordering implied by a class's own `compareTo` method.
- **`Comparator<T>`**: Interface with `int compare(T o1, T o2)`. Implemented by a **separate** class; defines an alternate, extrinsic order. Many may exist per type.
- **Generic static method**: A static method with its own type parameters declared before the return type, e.g. `public static <T> T pickRandom(T[] x)`. Type arguments are inferred at the call site.
- **Type bound**: A constraint on a generic type parameter, written `<T extends SomeType>`, restricting what may be substituted for `T` and telling the compiler which methods `T` is guaranteed to have.
- **Inner class**: A non-static nested class (like `MagicWizard`). Each instance is tied to an enclosing instance and can directly access its fields.

---

## Worked Examples

### Example 1: Why `for (int i : aset)` does not compile at first

The lecture starts with `ArraySet` lacking `implements Iterable<T>`:

```java
public class ArraySet<T> {
    private T[] items;
    private int size;
    // contains, add, size ...
}
```

```java
ArraySet<Integer> aset = new ArraySet<>();
aset.add(5);
aset.add(23);
aset.add(42);

for (int i : aset) {     // compile error
    System.out.println(i);
}
```

Step by step:

1. The compiler sees a for-each loop and tries to desugar it. To do that it must emit `aset.iterator()`.
2. It looks at the **static type** of `aset`, which is `ArraySet<Integer>`.
3. It asks: does `ArraySet` advertise, through its declared supertypes, that it has an `iterator()` method? No. There is no `implements Iterable<T>`.
4. Compile error. As the lecture comment puts it: "JAVA is unhappy right now, because it does not know that ArraySets have an iterator method."

Even if we had *written* an `iterator()` method, this would still fail without the `implements` clause, because Java checks the declared relationship, not the method list. (This is the nominal-vs-duck-typing point in action.)

### Example 2: The `Iterator` we have to write (`MagicWizard`)

```java
private class MagicWizard implements Iterator<T> {
    // this is where the wizard is looking in our array
    private int wizPos;

    MagicWizard() {
        wizPos = 0;
    }

    public boolean hasNext() {
        return (wizPos < size);
    }

    public T next() {
        T itemToReturn = items[wizPos];
        wizPos += 1;
        return itemToReturn;
    }
}

public Iterator<T> iterator() {
    return new MagicWizard();
}
```

Things to notice:

- `MagicWizard` is a **non-static inner class**. It refers to `size` and `items` with no qualification. Those are the *enclosing `ArraySet`'s* fields. In box-and-pointer terms: a `MagicWizard` box contains an `int wizPos` slot **and** a hidden reference back to the `ArraySet` that created it. When `hasNext()` reads `size`, it follows that hidden arrow to the outer object's `size` field. If `MagicWizard` were declared `static`, this hidden arrow would not exist and the code would not compile.
- `wizPos` is the *cursor*. It starts at 0 and advances one step per `next()` call. `hasNext()` is true exactly while the cursor is still inside the filled region `[0, size)`.
- `next()` does two things in order: grab `items[wizPos]` into a local, then advance. Returning `items[wizPos]` *after* incrementing would skip the first element and run off the end.
- `iterator()` returns a **fresh** `MagicWizard` each call, so each loop starts at position 0.

### Example 3: Tracing the desugared loop

With `implements Iterable<T>` in place, this:

```java
for (int i : aset) {
    System.out.println(i);
}
```

becomes, conceptually:

```java
Iterator<Integer> it = aset.iterator();
while (it.hasNext()) {
    int i = it.next();      // Integer auto-unboxed to int
    System.out.println(i);
}
```

Trace with `aset` holding `[5, 23, 42]`, `size == 3`:

| Step | `wizPos` before | `hasNext()` | `next()` returns | `wizPos` after |
|---|---|---|---|---|
| 1 | 0 | `0 < 3` true | `items[0]` = 5 | 1 |
| 2 | 1 | `1 < 3` true | `items[1]` = 23 | 2 |
| 3 | 2 | `2 < 3` true | `items[2]` = 42 | 3 |
| 4 | 3 | `3 < 3` false | (loop exits) | 3 |

Output: `5`, `23`, `42`.

`IteratorDemo.java` shows the same equivalence on a library type:

```java
Set<Integer> javaset = new TreeSet<>();
javaset.add(5); javaset.add(23); javaset.add(42);
for (int i : javaset) { System.out.println(i); }

// exactly the same as:
Iterator<Integer> seer = javaset.iterator();
while (seer.hasNext()) {
    int x = seer.next();
    System.out.println(x);
}
```

The program prints `5 23 42` twice. Note the static type is `Set<Integer>` while the dynamic type is `TreeSet<Integer>`: the for-each loop is legal because `Set` extends `Iterable`, and the iterator you actually get back is `TreeSet`'s, chosen by dynamic method selection. (extra context: `TreeSet` keeps elements sorted, which is why the output happens to be in ascending order.)

### Example 4: `toString`, and why it uses the iterator

```java
@Override
public String toString() {
    StringBuilder returnString = new StringBuilder("[");
    for (T x : this) {
        returnString.append(x);
        returnString.append(",");
    }
    returnString.append("]");
    return returnString.toString();
}
```

Step by step:

1. `for (T x : this)` iterates over the set itself. This works only because `ArraySet implements Iterable<T>`, so `toString` is a *client* of the iterator we just wrote. Nice payoff: once `iterator()` exists, other methods get to use the clean loop syntax.
2. `StringBuilder` accumulates the pieces. `append(x)` calls `x.toString()` under the hood.
3. `System.out.println(aset)` implicitly calls `aset.toString()`, printing `[5,23,42,]`.

Note the trailing comma before `]`, which is a cosmetic wart in the lecture version. (extra context: a common fix is to append the comma *before* each element except the first, or to use `String.join`.)

(extra context: the reason `StringBuilder` is used instead of `returnString += x` is performance. Repeated `+=` on a `String` builds a new `String` each time, making the loop quadratic in total length. This is discussed in detail later in the course.)

### Example 5: `equals`, line by line

```java
@Override
public boolean equals(Object o) {
    if (o instanceof ArraySet otherArraySet) {
        if (this.size == otherArraySet.size) {
            for (T x : this) {
                if (!otherArraySet.contains(x)) {
                    return false;
                }
            }
        } else {
            return false;
        }
        return true;
    }
    return false;
}
```

1. **Parameter type is `Object`.** This is what makes it a genuine override of `Object.equals`. The `@Override` annotation asks the compiler to verify that; if you slipped and wrote `equals(ArraySet o)`, `@Override` would trigger a compile error, which is exactly why you should always write it.
2. **`o instanceof ArraySet otherArraySet`** checks that `o` really is an `ArraySet` and simultaneously introduces `otherArraySet` with static type `ArraySet`. Without this, `otherArraySet.size` and `otherArraySet.contains(x)` would not compile, because `Object` has neither.
3. **Size check first.** Two sets with different sizes cannot be equal, and checking size is O(1), so bail out early.
4. **Containment check.** For a set, "same contents" means every element of `this` is in the other. Combined with the equal-size check (and the fact that `add` refuses duplicates), one-directional containment is enough. Each `contains` call is a linear scan, so this loop is O(n²) overall (extra context).
5. **Fall-through returns `true`** once the loop finishes without finding a missing element.
6. **Non-`ArraySet` argument returns `false`**, which is why the commented-out `aset.equals(List.of(1, 2, 3))` in `main` would simply be `false` rather than a crash.

Running `main`:

```java
System.out.println(aset);              // [5,23,42,]
ArraySet<Integer> aset2 = new ArraySet<>();
aset2.add(5); aset2.add(23); aset2.add(42);
IO.println(aset.equals(aset2));        // true
```

In box-and-pointer terms: `aset` and `aset2` are two different boxes at two different addresses, each with its own 100-element array. `aset == aset2` would be `false`. `aset.equals(aset2)` is `true` because our override looks at contents, not addresses.

(extra context: `IO.println` is the newer `java.lang.IO` convenience method available in recent JDKs. It behaves like `System.out.println` here.)

### Example 6: `Collections.max` on `Dog` via `Comparable`

```java
List<Dog> dogs = new ArrayList<>();
dogs.add(new Dog("Grigometh", 200));
dogs.add(new Dog("Pelusa", 5));
dogs.add(new Dog("Clifford", 9000));
Dog maxDog = Collections.max(dogs);
```

Before `Dog implements Comparable<Dog>`, this fails to compile with "no instance(s) of type variable(s) T exist so that Dog conforms to Comparable<? super T>". The fix:

```java
public class Dog implements Comparable<Dog> {
    @Override
    public int compareTo(Dog uddaDog) {
        return this.size - uddaDog.size;
    }
}
```

What happens at runtime: `Collections.max` walks the list holding a "best so far." It calls `candidate.compareTo(best)`. Its own code has no idea what a `Dog` is; the static type there is just some `T extends Comparable<...>`. Dynamic method selection routes each call to `Dog.compareTo`. Since `9000 - 200 > 0`, Clifford ends up as the max.

The longer explicit version is equivalent but wordier:

```java
@Override
public int compareTo(Dog uddaDog) {
    if (size > uddaDog.size) { return 1; }
    if (size < uddaDog.size) { return -1; }
    return 0;
}
```

### Example 7: `Comparator` for an alternate order

Python's version passes a function:

```python
def get_the_max(x, key):
    max_value = x[0]
    for item in x:
        if key(item) > key(max_value):
            max_value = item
    return max_value

def name_len(dog):
    return len(dog.name)

max_dog = get_the_max(doglist, name_len)
```

Java's version passes an object:

```java
public static class NameComparator implements Comparator<Dog> {
    @Override
    public int compare(Dog a, Dog b) {
        return a.name.compareTo(b.name);
    }
}
```

```java
Dog maxNameDog = Collections.max(dogs, new Dog.NameComparator());
// or, with the static constant:
Dog maxNameDog = Collections.max(dogs, Dog.NAME_COMPARATOR);
```

Now walk the textbook's check-your-understanding question:

```java
Dog a = new Dog("Frank", 1);
Dog b = new Dog("Zeke", 1);
Comparator<Dog> nc = new Dog.NameComparator();
System.out.println(nc.compare(a, b));
```

1. `nc.compare(a, b)` dispatches to `NameComparator.compare`.
2. That returns `"Frank".compareTo("Zeke")`.
3. `String.compareTo` compares character by character; the first characters differ, so it returns `'F' - 'Z'` = `70 - 90` = **-20**, a **negative** number, meaning Frank comes before Zeke alphabetically.
4. The `size` fields (both 1) are irrelevant here. The `Comparator` completely overrides the natural order.

Note that `nc`'s static type is `Comparator<Dog>` while its dynamic type is `NameComparator`: the usual interface-variable pattern.

(extra context, textbook "bonus": the same comparator can be written as a lambda, `Comparator<Dog> dc = (a, b) -> a.name.compareTo(b.name);`. The textbook explicitly says lambdas are not expected knowledge in this class.)

### Example 8: Building `max` ourselves, with a type bound

First attempt, which does not compile:

```java
public class Maximizer {
    public static <T> T max(T[] items) {
        T maxItem = items[0];
        for (int i = 0; i < items.length; i += 1) {
            int cmp = items[i].compareTo(maxItem);   // error: T has no compareTo
            if (cmp > 0) {
                maxItem = items[i];
            }
        }
        return maxItem;
    }
}
```

The compiler treats an unbounded `T` as (essentially) `Object`, and `Object` has no `compareTo`. Add a type bound:

```java
public class Maximizer {
    public static <T extends Comparable<T>> T max(T[] items) {
        T maxItem = items[0];
        for (int i = 0; i < items.length; i += 1) {
            int cmp = items[i].compareTo(maxItem);
            if (cmp > 0) {
                maxItem = items[i];
            }
        }
        return maxItem;
    }
}
```

Now the compiler knows every `T` implements `Comparable<T>`, so `compareTo` is legal. Calling `Maximizer.max(someNonComparableArray)` is a compile-time error rather than a runtime failure.

Call site, with no type argument needed:

```java
Dog[] dogs = {new Dog("Grigometh", 200), new Dog("Pelusa", 5), new Dog("Clifford", 9000)};
Dog biggest = Maximizer.max(dogs);   // T inferred as Dog
```

The industrial-strength signature the textbook shows is:

```java
public static <T extends Comparable<? super T>> T max(T[] items)
```

The `? super T` allows `T` to inherit its `compareTo` from a superclass (for example, comparing `Corgi[]` when only `Dog` implements `Comparable<Dog>`). The textbook says you will not need to wrestle with this much outside the end of Project 1B.

---

## Common Pitfalls

1. **Writing `iterator()` without `implements Iterable<T>`.** The for-each loop still will not compile. Java needs the declared relationship, not just the method.

2. **Confusing `Iterable` and `Iterator`.** `Iterable` has `iterator()`. `Iterator` has `hasNext()` and `next()`. A very common exam trap is asking which interface a given class should implement, or which methods a class must provide.

3. **Having `ArraySet` itself implement `Iterator`.** Tempting, but wrong: the position state would live in the collection, so you could only ever loop once, and nested loops over the same set would break. Keep the cursor in a separate object.

4. **Getting the order wrong inside `next()`.** Advance *after* reading. `wizPos += 1; return items[wizPos];` skips element 0 and reads one past the end.

5. **`hasNext()` using `items.length` instead of `size`.** `items` has capacity 100 but only `size` slots are filled. Using `items.length` would iterate over 97 `null`s.

6. **Making the iterator a `static` nested class.** It then has no enclosing instance, so `size` and `items` are not in scope.

7. **`equals(ArraySet o)` instead of `equals(Object o)`.** This is an overload, not an override. Your method silently never gets called by library code. Always write `@Override` so the compiler catches it.

8. **Forgetting the `instanceof` guard in `equals`.** Casting blindly throws `ClassCastException` when someone passes an unrelated object. `equals` must return `false`, not crash.

9. **Using `==` where `equals` is meant.** `aset == aset2` compares addresses. For objects, `==` is true only if both variables point to the exact same box.

10. **Mixing up `compareTo` and `compare`.** `Comparable` has one-argument `compareTo` and lives inside the class being ordered. `Comparator` has two-argument `compare` and lives outside.

11. **Assuming `compareTo` returns exactly -1, 0, or 1.** The contract is only about the sign. `"Frank".compareTo("Zeke")` is -20. Never write `if (a.compareTo(b) == 1)`.

12. **Thinking Java has operator overloading.** It does not. You cannot make `>` work on `Dog`. (extra context: `+` on `String` is a special case baked into the language, not something you can define for your own types.)

13. **Trying to make a generic method generic via the class.** `public class RandomPicker<T> { public static T pickRandom(T[] x) }` does not compile, because a static method cannot use the class's type parameter. Put `<T>` on the method.

14. **Writing the type argument when calling a generic static method.** Just call `Maximizer.max(dogs)`; the type is inferred.

15. **Passing primitive arrays to generic methods.** `int[]` is not `T[]`. Use `Integer[]`, or write a primitive-specific overload the way the real Java library does.

16. **Writing `<T implements Comparable<T>>`.** The keyword for type bounds is always `extends`, even for interfaces.

---

## Likely Exam Points

### 1. Desugaring the enhanced for loop

**Q:** Rewrite `for (String s : myList) { System.out.println(s); }` without using the enhanced for loop, assuming `myList` has static type `List<String>`.

**A:**
```java
Iterator<String> it = myList.iterator();
while (it.hasNext()) {
    String s = it.next();
    System.out.println(s);
}
```

### 2. Which interface, which methods

**Q:** You want `for (Card c : deck)` to work, where `Deck` is your own class. What must `Deck` declare, and what method(s) must it define? What must the helper object declare and define?

**A:** `Deck` must declare `implements Iterable<Card>` and define `public Iterator<Card> iterator()`. The helper (often a private inner class) must declare `implements Iterator<Card>` and define `public boolean hasNext()` and `public Card next()`.

### 3. Tracing an iterator

**Q:** Suppose `hasNext()` in `MagicWizard` were changed to `return (wizPos <= size);`. What happens when looping over an `ArraySet<Integer>` containing 5, 23, 42?

**A:** It prints 5, 23, 42, and then `null` (position 3, an unfilled slot). If the loop variable is declared `int` rather than `Integer`, auto-unboxing the `null` throws a `NullPointerException` on the fourth iteration.

### 4. Override vs overload on `equals`

**Q:** A student writes `public boolean equals(ArraySet o) { ... }` in `ArraySet`. The code compiles. Does `Object o = aset2; aset.equals(o);` call it? Why?

**A:** No. Overload resolution happens at **compile time** using static types. The static type of `o` is `Object`, so the compiler selects the inherited `Object.equals(Object)`, which does reference equality and returns `false`. Adding `@Override` would have caught the mistake at compile time, since `equals(ArraySet)` overrides nothing.

### 5. `instanceof` pattern matching

**Q:** What two things does `if (o instanceof ArraySet otherArraySet)` accomplish?

**A:** (1) It evaluates to `true` exactly when `o`'s dynamic type is `ArraySet` (or a subtype). (2) On success it binds `otherArraySet`, a variable whose static type is `ArraySet`, to `o`, so you can call `ArraySet` methods without an explicit cast.

### 6. Comparable vs Comparator

**Q:** You want to sort `Dog`s by size by default, but also sometimes by name, sometimes by age. Which interfaces do you use and where do they live?

**A:** `Dog implements Comparable<Dog>` with `compareTo` returning `this.size - other.size` defines the single natural order. For the alternates, write two separate classes, `NameComparator implements Comparator<Dog>` and `AgeComparator implements Comparator<Dog>`, each with a two-argument `compare`. `Comparable` gives one intrinsic order; `Comparator` gives arbitrarily many extrinsic ones.

### 7. Sign of a comparison

**Q:** Given `Dog a = new Dog("Frank", 1); Dog b = new Dog("Zeke", 1);` and a `NameComparator nc`, is `nc.compare(a, b)` positive, negative, or zero?

**A:** Negative (specifically -20, since `String.compareTo` returns `'F' - 'Z'`). The equal sizes are irrelevant because the comparator only looks at names.

### 8. Python vs Java mechanisms

**Q:** Fill in the table for how each language achieves each comparison.

**A:**

| | Natural order | Alternate order |
|---|---|---|
| Python | Operator overloading (`__gt__`) | Function passing (`key=name_len`) |
| Java | Subtype polymorphism (`Comparable.compareTo`) | Subtype polymorphism (`Comparator.compare`) |

Java uses subtype polymorphism for both; Python uses two different mechanisms.

### 9. Generic static method syntax

**Q:** Write the signature of a public static method named `smallest` that takes an array of `T` and returns a `T`, where `T` must be comparable to itself. Explain each piece.

**A:** `public static <T extends Comparable<T>> T smallest(T[] items)`. Reading left to right: `public static` is the access/staticness, `<T extends Comparable<T>>` declares a type parameter `T` constrained to implement `Comparable<T>`, the next `T` is the return type, `smallest` is the name, and `T[] items` is the parameter.

### 10. Why the type bound is needed

**Q:** `public static <T> T max(T[] items)` fails to compile at the line `items[i].compareTo(maxItem)`. Why, and what is the minimal fix?

**A:** With an unbounded `T`, the compiler can only assume `T` supports `Object`'s methods, and `Object` has no `compareTo`. The fix is a type bound: change `<T>` to `<T extends Comparable<T>>`.

### 11. Generics and primitives

**Q:** Does `Maximizer.max(new int[]{3, 1, 4})` compile? What about `Maximizer.max(new Integer[]{3, 1, 4})`?

**A:** The `int[]` version does not compile; generic type parameters cannot be bound to primitive types. The `Integer[]` version compiles fine, since `Integer` is a reference type implementing `Comparable<Integer>`.

### 12. Static vs dynamic type recall

**Q:** In `Comparator<Dog> nc = new Dog.NameComparator();`, what are the static and dynamic types of `nc`, and which `compare` runs on `nc.compare(a, b)`?

**A:** Static type `Comparator<Dog>` (what the compiler uses to check legality); dynamic type `NameComparator` (what actually exists at runtime). `NameComparator.compare` runs, by dynamic method selection.

---

## Summary

- **Java is nominally typed.** Having the right methods is not enough; you must declare `implements` so the compiler knows the is-a relationship. Python, being duck typed, does not require this.
- **`Iterable` vs `Iterator`.** `Iterable<T>` declares `iterator()`; it is the thing you loop over. `Iterator<T>` declares `hasNext()` and `next()`; it is the cursor object that tracks position.
- **The for-each loop is sugar.** `for (T x : c)` compiles into `Iterator<T> it = c.iterator(); while (it.hasNext()) { T x = it.next(); ... }`. That is why `implements Iterable<T>` is the "magic ingredient."
- **`ArraySet`'s iterator** is a private inner class (`MagicWizard`) holding an `int wizPos` plus an implicit reference to the enclosing set, which is how it reads `items` and `size` directly. `hasNext()` is `wizPos < size`; `next()` reads then advances.
- **Override `toString()`** to get useful printing instead of `ClassName@hashcode`. `System.out.println(obj)` calls it implicitly. `ArraySet.toString` itself loops with for-each, using the iterator it just gained.
- **Override `equals(Object o)`**, never `equals(ArraySet o)`. Use `@Override` so the compiler verifies you actually overrode. Guard with `instanceof`, return `false` for unrelated types, then compare contents.
- **`instanceof` pattern matching** (`o instanceof ArraySet other`) both tests the dynamic type and binds a correctly typed variable, eliminating the explicit cast.
- **`Comparable<T>`**: one-argument `compareTo`, implemented by the class itself, defines the single **natural order**. `return this.size - other.size;` is the idiomatic short form. Only the **sign** of the result is meaningful.
- **`Comparator<T>`**: two-argument `compare`, implemented by a separate class, defines an alternate **extrinsic** order. Many per type. Optionally exposed as a `public static final` constant like `Dog.NAME_COMPARATOR`.
- **Python vs Java**: Python uses operator overloading for natural order and function passing for alternate orders; Java uses **subtype polymorphism** for both.
- **`Collections.max(list)`** uses the natural order; **`Collections.max(list, comparator)`** uses the supplied one. Both work on your classes only because dynamic method selection calls back into your overrides.
- **Generic static methods**: put `<T>` before the return type (`public static <T> T pickRandom(T[] x)`). Do not make the class generic just to serve a static method. Type arguments are inferred at the call site.
- **Type bounds**: `<T extends Comparable<T>>` tells the compiler what `T` can do, turning a runtime hazard into a compile-time check. Use `extends` even for interfaces. The industrial version is `<T extends Comparable<? super T>>`.
- **Generics never work with primitives.** Use wrapper types, or write per-primitive overloads as the Java library does.
