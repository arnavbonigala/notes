<!-- Fri, Sep 18, 2026 | sources: slides + code + YouTube auto-transcript + textbook -->
# Lecture 10: Iterators, Iterables, and Equals

## Overview

This lecture completes the "core Java" arc by taking a deliberately bare-bones data structure, `ArraySet` (a set backed by an array, with `add`, `contains`, and `size`), and upgrading it into something "industrial strength" by adding the features that Java programmers expect of any real collection. The two big themes are iteration and object methods. On the iteration side, we peel back the magic of the enhanced for loop (`for (int i : aset)`) and discover that it is literally shorthand for asking the object for an `Iterator`, then repeatedly calling `hasNext()` and `next()`; to make our own class work with that syntax we write a private nested iterator class and declare `implements Iterable<T>`. On the object-methods side, we look at the methods every class inherits from `Object`, and override two of them: `toString()`, so printing an `ArraySet` shows its contents instead of `ArraySet@75412c2f`, and `equals(Object)`, so that two sets with the same contents compare as equal. Along the way we nail down the distinction between `==` (compares the bits, i.e. reference identity) and `.equals` (semantic equality), the meaning of `this`, the modern `instanceof` pattern-matching syntax, and a few side notes: autoboxing/unboxing, and why string concatenation in a loop is slow (use `StringBuilder`).

---

## Key Concepts

### 1. The starting point: `ArraySet`

A `Set` is a collection with no duplicates. Adding an element that is already present has no effect, and the main query is "is this in the set?". The lecture showed the same idea side by side in Java and Python:

```java
ArraySet<String> S = new ArraySet<>();
S.add("Oakland");
S.add("Toronto");
S.add("Minneapolis");
S.add("Oakland");     // no effect
S.add("Taipei");
IO.println(S.contains("Oakland"));   // true
```

```python
s = set()
s.add("Oakland")
s.add("Toronto")
s.add("Minneapolis")
s.add("Oakland")      # no effect
s.add("Taipei")
print("Oakland" in s) # True
```

Our `ArraySet` is deliberately simple: it keeps a `T[] items` array and an `int size`, it does not implement any `Set` interface (for now), and it ignores resizing, so it will eventually run out of room. That is fine, we already know how to resize from the list lectures.

The key point about the basic implementation is that `contains` does a linear scan and uses `.equals`, not `==`:

```java
public boolean contains(T x) {
    for (int i = 0; i < size; i += 1) {
        if (items[i].equals(x)) { return true; }
    }
    return false;
}
```

`==` on references would only ask "is this the exact same object in memory?", which is almost never what we mean when we ask if a set contains a value. `add` is then just "if not already contained, put it at index `size` and bump `size`":

```java
public void add(T x) {
    if (!contains(x)) {
        items[size] = x;
        size += 1;
    }
}
```

The constructor contains a wart that is worth noting because you will hit it in your own code:

```java
items = (T[]) new Object[100];   // "unchecked cast" compiler warning, nothing we can do
```

Java will not let you write `new T[100]`, so we allocate an `Object[]` and cast. The compiler warns; we ignore it.

### 2. The enhanced for loop is not magic, it is shorthand

Java lets you write:

```java
Set<Integer> javaset = new HashSet<>();
javaset.add(5); javaset.add(23); javaset.add(42);
for (int i : javaset) {
    System.out.println(i);
}
```

But the same loop over our `ArraySet` fails to compile:

```
error: for-each not applicable to expression type
        for (int i : aset) {
                     ^
  required: array or java.lang.Iterable
  found:    ArraySet<Integer>
```

The reason is that `for (x : thing)` is **literally shorthand** for a `while` loop driven by an iterator. The compiler rewrites

```java
for (int x : javaset) {
    IO.println(x);
}
```

into (essentially)

```java
Iterator<Integer> seer = javaset.iterator();
while (seer.hasNext()) {
    int x = seer.next();
    IO.println(x);
}
```

The lecture called the left version "nice" iteration and the right version "ugly" iteration. They do exactly the same thing. The variable name `seer` is a joke about "one who sees": the iterator is an object that looks into the collection for you.

### 3. What an `Iterator` is and how `next` behaves

`Iterator<T>` is an interface with (for our purposes) two methods:

```java
public interface Iterator<T> {
    boolean hasNext();
    T next();
}
```

`hasNext()` answers "are there more values?" and `next()` does **two** jobs at once:

1. returns the value at the current position, and
2. advances the iterator's position.

That double duty is the single most important thing to internalize. Java could have designed the API with a separate `move()` or `getNext()`, but it did not: one call to `next()` both yields and advances. This is why calling `next()` twice when you meant to look at the same element twice is a bug, and why a loop body must usually save `next()` into a local variable.

The lecture's animation, described in words: imagine a wizard ("seer") born by `javaset.iterator()`, standing just before the first element of `5, 23, 42`.

| Call | Result | Wizard position afterwards |
|---|---|---|
| `seer.hasNext()` | `true` | at `5` |
| `seer.next()` | `5` (printed) | at `23` |
| `seer.hasNext()` | `true` | at `23` |
| `seer.next()` | `23` (printed) | at `42` |
| `seer.hasNext()` | `true` | at `42` |
| `seer.next()` | `42` (printed) | past the end |
| `seer.hasNext()` | `false` | past the end |

Output: `5`, `23`, `42`, then the loop ends.

An important honesty note from lecture: the wizard is a cartoon. In our real implementation the "wizard" is just an object whose only state is an `int`. There is nothing visible moving through the array.

(extra context) If you call `next()` when `hasNext()` is false, the standard library throws `NoSuchElementException`. The lecture explicitly said exception behavior here was out of scope for the day; our simple `ArraySetIterator` would instead read past `size` and return a stale or `null` slot.

### 4. Writing an iterator for `ArraySet`

To support ugly iteration we need two things:

1. an `iterator()` method on `ArraySet` that returns an `Iterator<T>`, and
2. some class implementing `Iterator<T>` with useful `hasNext()` and `next()`.

There is no `Iterator` object lying around that knows about our array, so we must build one. We write a **private nested class** (called `ArraySetIterator` in the slides, `MagicWizard` in the live-coded file, same thing):

```java
private class ArraySetIterator implements Iterator<T> {
    private int wizPos;                        // where the "wizard" is looking

    public ArraySetIterator() { wizPos = 0; }

    public boolean hasNext() { return wizPos < size; }

    public T next() {
        T returnItem = items[wizPos];
        wizPos += 1;
        return returnItem;
    }
}

public Iterator<T> iterator() {
    return new ArraySetIterator();
}
```

Two things to notice. First, `hasNext` and `next` refer to `size` and `items`, which are instance variables of the **outer** `ArraySet`, not of the iterator. A (non-static) nested class can see the enclosing object's fields, which is exactly why we nest it. Second, `wizPos` is the iterator's own state, so each call to `iterator()` produces a fresh, independent traversal starting at 0.

`hasNext` can be written with an `if`/`else` returning `true`/`false`, but `return wizPos < size;` is the same thing and cleaner.

### 5. `Iterable`: telling Java that you have an `iterator()` method

After writing `iterator()`, ugly iteration works but the enhanced for loop still fails with the same error. Why? Because Java does not go looking for a method named `iterator` by duck typing. As the lecture put it: in some languages the compiler would just check automatically, but in Java a hypernym/hyponym relationship must be declared with `implements`. The compiler needs a *type-level* guarantee that `ArraySet` has an `iterator()` method.

The interface that provides that guarantee is `Iterable`:

```java
public interface Iterable<T> {
    Iterator<T> iterator();
    // plus some default methods, not shown
}
```

This is an interface so trivial it is confusing (the lecture compared it to being asked for `lim_{x->inf} 4` in calculus: the answer is just 4). Its whole content is "I have an `iterator()` method." One line fixes everything:

```java
public class ArraySet<T> implements Iterable<T> {
    ...
    public Iterator<T> iterator() { return new ArraySetIterator(); }
}
```

Now `for (int i : aset)` compiles and runs.

**Recipe to support the enhanced for loop (memorize this):**
1. Add an `iterator()` method to your class that returns an `Iterator<T>`.
2. Make that returned `Iterator<T>` have useful `hasNext()` and `next()` methods.
3. Add `implements Iterable<T>` to your class declaration.

This is exactly what you do in the last part of Project 2.

### 6. How Java's own collections fit together

This is the same mechanism the built-in collections use:

```
Iterable<T>
    ^
    |  (extends)
Collection<E>
    ^
    |  (extends)
  Set<E>
```

```java
public interface Collection<E> extends Iterable<E> {
    public Iterator<E> iterator();
}
public interface Set<E> extends Collection<E> {
    public Iterator<E> iterator();
}
```

So `HashSet`, `TreeSet`, `ArrayList`, and friends all work with the enhanced for loop for exactly the reason our `ArraySet` now does. The `extends` keyword for interfaces was flagged as mostly beyond the scope of this class, but it is why a `Set` is also `Iterable`.

### 7. The compiler's checks (clicker question)

Given the desugared version:

```java
Set<Integer> javaset = new HashSet<Integer>();
Iterator<Integer> seer = javaset.iterator();
while (seer.hasNext()) { IO.println(seer.next()); }
```

which checks must the compiler perform? The answers are **A** (does the `Set` interface have an `iterator()` method?) and **D** (does the `Iterator` interface have `next`/`hasNext` methods?).

The logic: we only ever call `.iterator()` on the thing whose static type is `Set`, and we only ever call `.hasNext()`/`.next()` on the thing whose static type is `Iterator`. So B ("does `Set` have next/hasNext?") and C ("does `Iterator` have an `iterator` method?") are not checks the compiler needs. It is a trivia-flavored question, but the point is real: the compiler checks methods against the **static type** of the variable you call them on.

### 8. Autoboxing and unboxing

In the ugly loop we wrote `int i = aseer.next();` even though `next()` returns `Integer`. Java allows assignment in both directions:

```java
Integer I = 1;
int i = I;        // unboxing

int iii = 3;
Integer III = iii; // autoboxing
```

Converting `int` to `Integer` via `=` is **autoboxing**; the other direction is **unboxing**. Practical advice from lecture: use `int` 99% of the time. The only time you need `Integer` is when filling in a generic type parameter, e.g. `ArraySet<Integer>`, because generics cannot take primitives. Autoboxing/unboxing also costs a little time at runtime (real work happens on that line), though this is not a focus of 61B.

### 9. `Object` methods

Every class you write is a hyponym (subclass) of `Object`, so every object already has these methods whether you wrote them or not:

| Method | Status in 61B |
|---|---|
| `String toString()` | Covered today |
| `boolean equals(Object obj)` | Covered today |
| `int hashCode()` | Coming later in the course |
| `Class<?> getClass()` | Mentioned |
| `protected Object clone()`, `protected void finalize()`, `notify()`, `notifyAll()`, `wait(...)` | Not discussed or used in 61B |

### 10. `toString()`

`toString()` provides a string representation of an object. `System.out.println(Object x)` calls `x.toString()` (strictly, `println` calls `String.valueOf`, which calls `toString`). This is the Java analogue of Python's `__str__`/`__repr__`.

The `Object` default implementation is the class name, an `@`, and the hash code, which by default is derived from the object's memory location:

```java
ArraySet<Integer> aset = new ArraySet<>();
aset.add(5); aset.add(23); aset.add(42);
IO.println(aset);   // ArraySet@75412c2f
```

Useless. So we override it. The lecture's live-coded version, using the iterator we just built:

```java
@Override
public String toString() {
    String returnString = "{";
    for (T item : this) {          // works because ArraySet is Iterable
        returnString += item.toString();
        returnString += ", ";
    }
    returnString += "}";
    return returnString;
}
```

Things worth noticing:
- `for (T item : this)` iterates over the current object. This only compiles because we made `ArraySet` implement `Iterable`.
- Writing `returnString += item` (without `.toString()`) also works: in Java, `+` with a `String` on one side automatically calls `toString()` on the other operand. (If the element type had no meaningful `toString`, there is nothing you can do about it; you get whatever it provides.)
- The trailing comma before `}` is a cosmetic flaw you can fix if you want.
- `@Override` is optional but strongly recommended: if you typo the name (e.g. `toStr1ng`), `@Override` makes the compiler tell you that you are not actually overriding anything.

**Performance warning.** The `+=` version is slow. Intuition: Java strings are *immutable*, so adding even one character builds an entirely new string, copying everything. In a loop, that is a lot of copying. IntelliJ flags it with a yellow squiggle. The fast version uses `StringBuilder`, which is designed for a string "in progress" so that appending does not rebuild everything:

```java
@Override
public String toString() {
    StringBuilder returnSB = new StringBuilder("{");
    for (int i = 0; i < size; i += 1) {
        returnSB.append(items[i]);
        returnSB.append(", ");
    }
    returnSB.append("}");
    return returnSB.toString();
}
```

(Bonus, from the slides) The lazy but clean way, using `String.join`:

```java
@Override
public String toString() {
    List<String> listOfItems = new ArrayList<>();
    for (T x : this) {
        listOfItems.add(x.toString());
    }
    return "{" + String.join(", ", listOfItems) + "}";
}
```

This one also fixes the trailing-comma problem for free.

### 11. `==` versus `.equals`

`==` compares the **bits in the two memory boxes**. For primitives that means comparing the values. For references, the bits are addresses, so `==` means "do these two variables reference the same object?"

```java
Set<Integer> javaset  = Set.of(5, 23, 42);
Set<Integer> javaset2 = Set.of(5, 23, 42);
IO.println(javaset == javaset2);         // false
IO.println(javaset.equals(javaset2));    // true
```

Box-and-pointer reasoning in words: `javaset` and `javaset2` are two separate 64-bit boxes, each holding the address of a *different* set object out in the heap. Those two addresses differ, so `==` is false even though the two objects have identical contents. `.equals` looks inside the objects and compares contents, so it is true.

To test equality in the sense we usually mean:
- Use `.equals` for classes. You have to write a `.equals` method for your own classes; the default one is not what you want.
- For arrays, use `Arrays.equals` or `Arrays.deepEquals`.

The default implementation in `Object.java` is literally:

```java
public class Object {
    ...
    public boolean equals(Object obj) {
        return (this == obj);
    }
}
```

`this` here is just the 64-bit address of the current object, so the default `.equals` is exactly `==`. That is why:

```java
ArraySet<Integer> aset  = new ArraySet<>(); aset.add(5);  aset.add(23);  aset.add(42);
ArraySet<Integer> aset2 = new ArraySet<>(); aset2.add(5); aset2.add(23); aset2.add(42);
IO.println(aset.equals(aset2));   // false, until we override equals
```

### 12. `this`

`this` is a reference to the current object. From the Lecture 2 `Dog` example:

```java
public Dog maxDog(Dog uddaDog) {
    if (size > uddaDog.size) {
        return this;
    }
    return uddaDog;
}
```

You can also use `this` to access your own instance variables or methods. Unlike Python, where `self` is mandatory, `this` is optional in Java. These two are identical in behavior:

```java
public Dog maxDog(Dog o) {          public Dog maxDog(Dog o) {
    if (this.size > o.size) {           if (size > o.size) {
        return this;                        return this;
    }                                   }
    return o;                           return o;
}                                   }
```

The one case where `this` is **mandatory** is a name conflict between a parameter (or local) and an instance variable:

```java
public Dog(int size) { size = size; }        // does NOTHING (assigns parameter to itself)
public Dog(int size) { this.size = size; }   // works
public Dog(int s)    { size = s; }           // works
public Dog(int s)    { this.size = s; }      // works
```

### 13. `instanceof` with pattern matching

The signature we must override is `equals(Object o)`, not `equals(ArraySet o)`. Why `Object`? Because the method in `Object` takes an `Object`, and overriding requires the same signature; writing `equals(ArraySet o)` plus `@Override` is a compile error because you are not actually overriding anything (you would be *overloading*). Conceptually it is also right: you might want an `ArrayDeque` and a `LinkedListDeque` with the same contents to be equal, so the parameter type has to be general.

That generality creates a problem: inside the method, `o` has static type `Object`, so `o.size` and `o.items` do not compile. The modern (Java 16+) solution is `instanceof` with pattern matching:

```java
@Override
public boolean equals(Object o) {
    if (o instanceof Dog uddaDog) {
        return this.size == uddaDog.size;
    }
    return false;
}
```

`o instanceof Dog uddaDog` does **two** things at once:
1. it evaluates to `true` if `o` is pointing at a `Dog` (and `false` otherwise, including when `o` is `null`, so no separate null check is needed), and
2. if true, it binds `o` to a new variable `uddaDog` whose static type is `Dog`, usable in the body.

This is called **pattern matching**. The lecture described step 2 informally as "reincarnating" the object under a new name with a more specific type.

### 14. Historical note: old-school `equals`

Before Java 16 (released March 2021), `equals` methods were ugly, with manual null checks, `getClass()` comparison, and explicit casting:

```java
@Override // OLD SCHOOL APPROACH. NOT PREFERRED IN 61B.
public boolean equals(Object o) {
    if (o == null) { return false; }
    if (this == o) { return true; }                  // optimization
    if (this.getClass() != o.getClass()) { return false; }
    ArraySet<T> other = (ArraySet<T>) o;
    ...
}
```

You should avoid this style (explicit casting) in 61B. Recognize it if you see it in old exams or old code.

---

## Definitions

- **Set**: An abstract data type storing a collection of values with no duplicates; adding a value already present has no effect. Core operations here: `add(value)`, `contains(value)`, `size()`.
- **`ArraySet<T>`**: The class built in this lecture. A set backed by a `T[] items` array plus an `int size`; `contains` is a linear scan using `.equals`; `add` checks `contains` first; resizing is ignored.
- **Enhanced for loop (for-each loop)**: The syntax `for (Type x : collection) { ... }`. It is shorthand that the compiler expands into obtaining an `Iterator` from the collection and looping with `hasNext()`/`next()`. It requires the expression to be an array or a `java.lang.Iterable`.
- **`Iterator<T>`**: An interface whose implementors provide `boolean hasNext()` (are there more values?) and `T next()` (return the current value **and** advance). An object of this type is "one who sees" into a collection; the lecture nicknamed instances `seer` and the implementing class `MagicWizard`.
- **`hasNext()`**: Returns `true` if there are more values remaining to be returned by `next()`.
- **`next()`**: Returns the next value and simultaneously advances the iterator's position. Two jobs, one call.
- **`Iterable<T>`**: An interface whose entire content (aside from some default methods) is `Iterator<T> iterator();`. Declaring `implements Iterable<T>` is how you formally tell Java "I have an `iterator()` method," which is what the enhanced for loop requires.
- **`Collection<E>`**: A Java interface that `extends Iterable<E>`; `Set<E>` and `List<E>` extend `Collection<E>`. (Mostly out of scope for 61B, mentioned for orientation.)
- **`wizPos`**: In our `ArraySetIterator`, the private `int` instance variable recording which index the iterator is currently looking at. It is the iterator's only state.
- **Nested (inner) class, e.g. `private class ArraySetIterator`**: A class declared inside another class. A non-static nested class can access the enclosing object's instance variables (`items`, `size`), which is why the iterator can see the set's contents.
- **Autoboxing**: Automatic conversion from a primitive (e.g. `int`) to its wrapper object type (e.g. `Integer`) via assignment.
- **Unboxing**: The reverse, automatic conversion from `Integer` to `int`.
- **`Object`**: The universal superclass; every class is a hyponym of `Object` and inherits `toString()`, `equals(Object)`, `hashCode()`, `getClass()`, and others.
- **`toString()`**: An `Object` method returning a `String` representation of an object. Called automatically by `System.out.println(Object)` (via `String.valueOf`) and by string concatenation with `+`. Default implementation: class name, `@`, hash code (default hash code is derived from memory address).
- **`equals(Object obj)`**: An `Object` method for semantic equality. Default implementation is `return (this == obj);`, i.e. reference identity, which is usually not what you want.
- **`==`**: An operator comparing the bits in two memory boxes. For references this means "do they point at the same object?"
- **`this`**: A reference to the current object (in effect, its address). Optional when accessing your own members, mandatory to disambiguate an instance variable from a same-named parameter or local.
- **`instanceof` (with pattern matching)**: `o instanceof Dog d` returns `true` iff `o` references a `Dog` (false for `null`), and on the true branch binds `d` as a `Dog`-typed variable referring to the same object. Available since Java 16.
- **`StringBuilder`**: A mutable string-building class whose `append` operation is fast, unlike repeated `String` concatenation, which rebuilds the whole string each time because Strings are immutable.
- **Var arg (`Glerp... stuff`)**: A parameter that accepts any number of comma-separated arguments and is received as an array. (Bonus.)
- **`@Override`**: An annotation asserting that the method overrides a superclass/interface method. Optional, but it catches typos and wrong signatures at compile time.

---

## Worked Examples

### Example 1: Desugaring the enhanced for loop

```java
Set<Integer> javaset = new TreeSet<>();
javaset.add(5);
javaset.add(23);
javaset.add(42);

for (int i : javaset) {
    System.out.println(i);
}

// the code above is EXACTLY THE SAME as:
Iterator<Integer> seer = javaset.iterator();
while (seer.hasNext()) {
    int x = seer.next();
    System.out.println(x);
}
```

Step by step for the bottom (ugly) version:

1. `javaset.iterator()` asks the `TreeSet` to manufacture a fresh iterator object. A new object is created on the heap; `seer` is a box holding its address. That object has its own position state, initially "before the first element."
2. `seer.hasNext()` → `true` (there are 3 elements, we have consumed 0).
3. `seer.next()` → returns `5` **and** advances. Note it returns an `Integer`; assigning it to `int x` is unboxing.
4. Print `5`.
5. `hasNext()` → `true`; `next()` → `23`, advance; print `23`.
6. `hasNext()` → `true`; `next()` → `42`, advance; print `42`.
7. `hasNext()` → `false`; loop terminates.

Output: `5`, `23`, `42` (a `TreeSet` iterates in sorted order; a `HashSet` makes no ordering promise).

The top (nice) version produces the identical sequence of calls. The only difference is that the iterator variable is invisible to you, which is also why you cannot accidentally mess with it.

### Example 2: Building `ArraySet`'s iterator from scratch

Goal: make this work.

```java
ArraySet<Integer> aset = new ArraySet<>();
aset.add(5);
aset.add(23);
aset.add(42);

Iterator<Integer> aseer = aset.iterator();
while (aseer.hasNext()) {
    int i = aseer.next();
    IO.println(i);
}
```

Step 1: `aset.iterator()` does not exist, so the compiler errors. Add the method. But what should it return? We need "some sort of magic wizard who has a `next` and `hasNext` method," and no such class exists, so we must write one.

Step 2: Write the nested class and declare what it is.

```java
private class ArraySetIterator implements Iterator<T> {
```

`implements Iterator<T>` is a formal promise: "I have `hasNext()` and `next()`." Without it, the compiler will not let us return an `ArraySetIterator` where an `Iterator<T>` is expected, no matter what methods it happens to contain.

Step 3: What state does a wizard need? A position. An `int`:

```java
    private int wizPos;   // where the wizard is looking in our array
    public ArraySetIterator() { wizPos = 0; }
```

Step 4: `hasNext`. How do we know there is more stuff? If the position has not reached the number of items:

```java
    public boolean hasNext() { return wizPos < size; }
```

`size` here is the enclosing `ArraySet`'s field, visible because the class is nested and non-static.

Step 5: `next`. Two jobs: grab the item, then advance.

```java
    public T next() {
        T returnItem = items[wizPos];
        wizPos += 1;
        return returnItem;
    }
}
```

Note the order: save first, then increment, then return the saved value. If you incremented first you would return the wrong element; if you returned first the increment would never run.

Step 6: Hook it up.

```java
/** returns an iterator (a.k.a. seer) into ME */
public Iterator<T> iterator() {
    return new ArraySetIterator();
}
```

**Debugger trace (as shown in lecture):** stepping over `aset.iterator()` creates an object whose `wizPos` is `0`. Each loop iteration, `wizPos` visibly increments: 0 → 1 → 2 → 3. When `wizPos` is 3 and `size` is 3, `hasNext()` returns `false` and the loop exits. Printed output: `5`, `23`, `42`.

**Environment reasoning in words:** `aset` is a box holding the address of an `ArraySet` object. That object has two instance variables: `items` (holding the address of a 100-element `Object[]`, whose first three slots hold addresses of `Integer` objects 5, 23, 42) and `size` (holding `3`). `aseer` is a box holding the address of a separate `ArraySetIterator` object, whose only instance variable is `wizPos`. Because the iterator is a non-static nested class instance, it also carries a hidden link back to the enclosing `ArraySet`, which is how `items` and `size` resolve inside its methods. Calling `aset.iterator()` a second time creates a **second, independent** iterator object with its own `wizPos = 0`.

### Example 3: Making the enhanced for loop work

After Example 2, this still fails:

```java
for (int i : aset) { IO.println(i); }
// error: required: array or java.lang.Iterable; found: ArraySet<Integer>
```

Even though `ArraySet` *has* an `iterator()` method, Java will not search for it by name. The fix is a single declaration:

```java
public class ArraySet<T> implements Iterable<T> {
    ...
    public Iterator<T> iterator() { return new ArraySetIterator(); }
}
```

Now the type `ArraySet<Integer>` is a subtype of `Iterable<Integer>`, the compiler's requirement is satisfied, and the loop compiles and prints `5`, `23`, `42`. Note that `iterator()` must be `public` to satisfy the interface.

### Example 4: `toString` for `ArraySet`

Before overriding:

```java
IO.println(aset);      // ArraySet@75412c2f
```

`println(Object x)` calls `x.toString()`; the inherited `Object` version gives class name + `@` + hash code, and the default hash code is derived from the address.

After overriding (lecture's version, cleaned to use `StringBuilder` as in the code file):

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

Step by step on `{5, 23, 42}`:
1. Start the builder with `"["`.
2. `for (T x : this)` desugars to `this.iterator()` plus a `hasNext`/`next` loop, which only works because of `implements Iterable<T>`. So `toString` reuses the machinery from Examples 2 and 3.
3. Append `5`, then `,` → `"[5,"`. Appending a non-String calls its `toString()` automatically.
4. Append `23`, `,` → `"[5,23,"`.
5. Append `42`, `,` → `"[5,23,42,"`.
6. Append `"]"` → `"[5,23,42,]"`, and return `.toString()` of the builder.

There is a trailing comma; the lecture left it in, noting you can fix it if you care. The `String +=` version produces the same string but is slow, since each `+=` builds a brand-new immutable `String`.

A live-coding slip worth remembering: writing `returnString += "s"` inside `for (T s : this)` prints `sss`, because the literal string `"s"` is not the variable `s`. Concatenate the variable, not a quoted letter.

### Example 5: `equals` for `ArraySet`

First attempt, which fails:

```java
@Override
public boolean equals(ArraySet o) { ... }   // compiler error with @Override
```

This does not override `Object.equals(Object)`; it overloads it. `@Override` catches the mistake. Correct signature:

```java
@Override
public boolean equals(Object o) {
    if (o instanceof ArraySet oas) {
        // check sets are of the same size
        if (oas.size != this.size) {
            return false;
        }
        // check that all of MY items are in the other array set
        for (T x : this) {
            if (!oas.contains(x)) {
                return false;
            }
        }
        return true;
    }
    // o is not an ArraySet, so return false
    return false;
}
```

Step by step:
1. `o instanceof ArraySet oas` asks "is `o` pointing at an `ArraySet`?" If `o` is `null` or some unrelated type (say a `List`), this is `false` and we fall through to `return false`. No separate null check needed.
2. If true, `oas` is now an `ArraySet`-typed name for the same object, so `oas.size` and `oas.contains(...)` compile. (Without the pattern variable, `o.size` would be a compile error, since `o`'s static type is `Object`.)
3. Size check first: different sizes means not equal, and it is cheap.
4. Same size: iterate over **my** elements (using the enhanced for loop we enabled) and verify each one is in the other set. This is where `contains` on the other set does the real work.
5. If every one of my elements is in the other set **and** the sizes match, the sets are equal, so `return true`. (Equal size is what makes one-directional containment sufficient; if I had 3 elements all present in theirs and they also have 3, there is no room for an extra.)
6. Not an `ArraySet` at all: `return false`.

The polished, "pretty close to a standard `equals`" version from the slides adds one optimization:

```java
@Override
public boolean equals(Object other) {
    if (this == other) { return true; }   // doesn't affect correctness, saves time
                                          // if this and other are the same object
    if (other instanceof ArraySet otherSet) {
        if (this.size != otherSet.size) { return false; }
        for (T x : this) {
            if (!otherSet.contains(x)) {
                return false;
            }
        }
        return true;
    }
    return false;
}
```

`ArraySet` here is technically a raw type (no `<T>`); the slides say not to worry about it.

Running it:

```java
ArraySet<Integer> aset  = new ArraySet<>(); aset.add(5);  aset.add(23);  aset.add(42);
ArraySet<Integer> aset2 = new ArraySet<>(); aset2.add(5); aset2.add(23); aset2.add(42);
IO.println(aset.equals(aset2));   // true (was false before we overrode equals)
IO.println(aset == aset2);        // false: different objects
```

### Example 6 (Bonus): writing your own `.of`

Java's `Set.of(5, 23, 42)` is a static factory. You can write your own:

```java
public static <Glerp> ArraySet<Glerp> of(Glerp... stuff) {
    ArraySet<Glerp> returnSet = new ArraySet<Glerp>();
    for (Glerp x : stuff) {
        returnSet.add(x);
    }
    return returnSet;
}
```

`Glerp... stuff` is a **var arg**: callers write comma-separated values, and inside the method `stuff` is an array (hence the enhanced for loop over it works, arrays being for-each-compatible). `<Glerp>` before the return type declares a generic type parameter for the static method, which is needed because static methods cannot use the class's `T`.

### Example 7 (from the code file): the null policy

The live-coded `add` in `ArraySet.java` throws on null:

```java
public void add(T x) {
    if (x == null) {
        throw new IllegalArgumentException("can't add null");
    }
    if (contains(x)) {
        return;
    }
    items[size] = x;
    size += 1;
}
```

Why: `contains` calls `items[i].equals(x)`, and more generally allowing nulls forces null-handling everywhere. Throwing an `IllegalArgumentException` is one design choice; the summary slide explicitly says there are other ways to deal with nulls and that this choice was arguably bad. The important skill is recognizing that "what do I do about null?" is a design decision you must consciously make.

---

## Common Pitfalls

1. **Forgetting `implements Iterable<T>`.** You write a perfect `iterator()` method, ugly iteration works, and the enhanced for loop still refuses to compile with `required: array or java.lang.Iterable`. Java does not duck-type; you must declare the relationship.

2. **Forgetting to implement `Iterator<T>` on your nested class.** Having methods named `hasNext` and `next` is not enough; the class must declare `implements Iterator<T>` so it can be returned where an `Iterator<T>` is required.

3. **Making `iterator()` non-public.** Interface methods must be public. A package-private `Iterator<T> iterator()` will not satisfy `Iterable<T>`.

4. **Calling `next()` more than once per loop iteration.** Since `next()` advances, calling it twice skips an element. Save it: `T x = it.next();`.

5. **Getting the order wrong inside `next()`.** `wizPos += 1; return items[wizPos];` returns the wrong element, and `return items[wizPos]; wizPos += 1;` never advances (unreachable code). Save, advance, return.

6. **Writing `public boolean equals(ArraySet o)`.** This overloads instead of overriding, so `Object`-typed calls (including from library code) still get the identity-based default. `@Override` catches it at compile time; without `@Override` it silently "works" in some call sites and fails in others.

7. **Using `==` to compare objects.** `javaset == javaset2` is `false` for two distinct sets with identical contents. `==` compares bits (addresses) and only tells you whether it is the *same object*.

8. **Expecting the default `.equals` to compare contents.** `Object.equals` is literally `return (this == obj);`, so until you override it, `.equals` is just `==`.

9. **Using `.equals` or `==` on arrays.** Use `Arrays.equals` (or `Arrays.deepEquals` for nested arrays); array `.equals` is inherited identity comparison.

10. **Using `items[i] == x` inside `contains`.** This would only find the exact same object, not an equal value. The lecture is explicit that `contains` uses `items[i].equals(x)`.

11. **Building strings with `+=` in a loop.** Correct but slow, because Strings are immutable so every concatenation copies the whole thing. Use `StringBuilder` (IntelliJ will even offer the conversion).

12. **The `size = size;` constructor bug.** With a parameter named the same as the instance variable, `size = size;` assigns the parameter to itself and does nothing. Write `this.size = size;` or rename the parameter.

13. **Trailing separator in `toString`.** The straightforward loop produces `{5, 23, 42, }`. Not a correctness disaster, but exam graders and readers notice; `String.join` avoids it.

14. **Writing old-style `equals` with explicit casts and `getClass()` checks.** Obsolete since Java 16; prefer `instanceof` pattern matching. Recognize the old style, do not write it.

15. **Assuming `instanceof` needs a null guard.** It does not: `null instanceof Anything` is `false`, so the pattern-matching form handles null for free.

16. **Reaching for `Integer` when `int` will do.** Use `int` except when supplying a generic type argument, where primitives are not allowed.

---

## Likely Exam Points

### 1. Desugaring the enhanced for loop

*Commonly tested as: rewrite this for-each loop without for-each, or identify why a for-each loop fails to compile.*

**Q.** Rewrite `for (String s : myCollection) { System.out.println(s); }` without using the enhanced for loop. What must be true of `myCollection`'s type for the original to compile?

**A.**
```java
Iterator<String> it = myCollection.iterator();
while (it.hasNext()) {
    String s = it.next();
    System.out.println(s);
}
```
`myCollection` must be an array or its static type must be (a subtype of) `java.lang.Iterable`, i.e. the type must declare an `iterator()` method returning an `Iterator`.

### 2. Fill in the blanks of an iterator class

*Project 2 and discussion worksheet material; expect a skeleton with `hasNext`/`next` bodies blanked out.*

**Q.** Complete the iterator so that `for (T x : myArraySet)` visits `items[0] ... items[size - 1]` in order.

```java
private class ArraySetIterator implements Iterator<T> {
    private int wizPos;
    public ArraySetIterator() { ______ }
    public boolean hasNext()  { ______ }
    public T next()           { ______ }
}
```

**A.**
```java
public ArraySetIterator() { wizPos = 0; }
public boolean hasNext()  { return wizPos < size; }
public T next() {
    T returnItem = items[wizPos];
    wizPos += 1;
    return returnItem;
}
```

### 3. Why doesn't my for-each compile? (the three-step recipe)

**Q.** `MyList<T>` has a correct `public Iterator<T> iterator()` method and a correct nested `Iterator<T>` implementation, but `for (T x : myList)` gives `error: for-each not applicable to expression type`. What is missing and why?

**A.** The class declaration is missing `implements Iterable<T>`. In Java, having a method with the right name is not enough: hypernym/hyponym relationships must be declared explicitly with `implements`, and the compiler requires the static type to be `Iterable` before it will desugar the loop.

### 4. Compiler checks for the desugared loop (the clicker)

**Q.** For `Iterator<Integer> seer = javaset.iterator(); while (seer.hasNext()) { IO.println(seer.next()); }` with `Set<Integer> javaset`, which of the following does the compiler check? (A) `Set` has `iterator()`. (B) `Set` has `next`/`hasNext`. (C) `Iterator` has `iterator()`. (D) `Iterator` has `next`/`hasNext`.

**A.** A and D. The compiler checks methods against the static type of the receiver: `.iterator()` is called on a `Set`, and `.hasNext()`/`.next()` are called on an `Iterator`. B and C are not needed.

### 5. Trace `hasNext`/`next`

**Q.** An iterator over `{5, 23, 42}` is created. The following calls are made in order: `hasNext()`, `next()`, `next()`, `hasNext()`, `next()`, `hasNext()`. What does each return?

**A.** `true`, `5`, `23`, `true`, `42`, `false`. Key point: each `next()` both returns and advances, so two consecutive `next()` calls yield two different elements.

### 6. `==` versus `.equals`

**Q.** What does the following print, and why?
```java
Set<Integer> a = Set.of(5, 23, 42);
Set<Integer> b = Set.of(5, 23, 42);
System.out.println(a == b);
System.out.println(a.equals(b));
```

**A.** `false` then `true`. `a` and `b` are two boxes holding addresses of two distinct objects, and `==` compares those bits, so it is false. `Set`'s `.equals` compares contents semantically, so it is true.

### 7. The default `equals`

**Q.** You define `class Point { int x, y; }` and do not override `equals`. Two `Point`s both have `x = 1, y = 2`. What does `p1.equals(p2)` return and why?

**A.** `false`. The inherited `Object.equals` is `return (this == obj);`, i.e. reference identity, so distinct objects are never equal under it.

### 8. Write / debug an `equals` method

**Q.** What is wrong with the following, and fix it?
```java
@Override
public boolean equals(ArraySet o) {
    return this.size == o.size;
}
```

**A.** Two problems. (1) The parameter type must be `Object` to actually override `Object.equals(Object)`; as written it is an overload, and `@Override` makes it a compile error. (2) Comparing only sizes is wrong; equal sizes does not mean equal contents. Fixed:
```java
@Override
public boolean equals(Object o) {
    if (this == o) { return true; }
    if (o instanceof ArraySet otherSet) {
        if (this.size != otherSet.size) { return false; }
        for (T x : this) {
            if (!otherSet.contains(x)) { return false; }
        }
        return true;
    }
    return false;
}
```

### 9. `instanceof` pattern matching

**Q.** What two things does `if (o instanceof Dog uddaDog)` do? What happens if `o` is `null`?

**A.** It (1) evaluates to `true` exactly when `o` references a `Dog`, and (2) on the true branch introduces a new variable `uddaDog` of static type `Dog` bound to the same object, so you can call `Dog` methods and read `Dog` fields. If `o` is `null`, the expression is `false` and the body is skipped, so no separate null check is needed.

### 10. `toString` behavior

**Q.** `ArraySet` does not override `toString`. What does `System.out.println(aset)` print, and what is the mechanism?

**A.** Something like `ArraySet@75412c2f`: the class name, `@`, and the hash code (by default derived from the object's memory location). `println(Object x)` calls `String.valueOf(x)`, which calls `x.toString()`, and the inherited `Object.toString()` produces that format.

### 11. `this` and shadowed variables

**Q.** Why does the constructor `public Dog(int size) { size = size; }` fail to set the instance variable, and what are two fixes?

**A.** Inside the constructor, the parameter `size` shadows the instance variable, so `size = size;` assigns the parameter to itself and the field is untouched. Fixes: `this.size = size;`, or rename the parameter (`public Dog(int s) { size = s; }`).

### 12. String concatenation performance

**Q.** Why is building a `toString` with `returnString += item;` inside a loop considered slow, and what should you use instead?

**A.** Java Strings are immutable, so appending even one character allocates and copies an entirely new String. Doing this once per element makes the loop do far more copying than necessary. Use a `StringBuilder` and `append`, which is designed for an in-progress string, then call `.toString()` at the end.

### 13. Autoboxing/unboxing

**Q.** `next()` returns `Integer` but we wrote `int i = aseer.next();`. Is that legal? What is it called, and which should you prefer?

**A.** Legal. Converting `Integer` to `int` is unboxing (the reverse is autoboxing). Prefer `int` except when supplying a generic type argument such as `ArraySet<Integer>`, since generics cannot take primitives; boxing/unboxing also costs a little runtime.

---

## Summary

- We built `ArraySet<T>`: `T[] items` plus `int size`, with linear-scan `contains` (using `.equals`, not `==`), a `contains`-guarded `add`, and `size()`. Resizing was ignored; `add` throws `IllegalArgumentException` on `null` (one of several possible null policies, arguably not the best one).
- **The enhanced for loop is shorthand.** `for (T x : c)` is literally `Iterator<T> it = c.iterator(); while (it.hasNext()) { T x = it.next(); ... }`.
- **`Iterator<T>`** has `boolean hasNext()` and `T next()`. `next()` does two jobs: return the current value **and** advance.
- **To support the enhanced for loop, three steps:** (1) add a public `iterator()` returning an `Iterator<T>`; (2) make that iterator have useful `hasNext()`/`next()`; (3) add `implements Iterable<T>` to the class declaration.
- Our iterator was a private nested class whose only state was `wizPos`, starting at 0; `hasNext` is `wizPos < size`, `next` saves `items[wizPos]`, increments, returns the saved value. Being nested lets it see the enclosing set's `items` and `size`.
- **`Iterable<T>`** is the one-method interface `Iterator<T> iterator();`. Java requires the relationship be *declared*; it will not find an `iterator()` method by name. Java's own hierarchy: `Set` extends `Collection` extends `Iterable`.
- The compiler checks method calls against the **static type** of the receiver: `iterator()` against `Set`, `hasNext`/`next` against `Iterator`.
- **Autoboxing/unboxing** makes `int` and `Integer` interchangeable in assignments; prefer `int` except as a generic type argument.
- Every class inherits from **`Object`**: `toString()`, `equals(Object)`, `hashCode()` (later in the course), and others we will not use.
- **`toString()`** defaults to class name + `@` + hash code (address-derived). Override it; `println` and string `+` call it automatically. Beware `+=` in a loop (Strings are immutable): use `StringBuilder`, or `String.join` for a clean bonus version.
- **`==` compares the bits**; for references that means "same object." **`.equals` is for semantic equality**, but `Object`'s default `equals` is literally `return (this == obj);`, so you must override it. Use `Arrays.equals`/`Arrays.deepEquals` for arrays.
- **`equals` must take `Object`**, not your own type, or you are overloading rather than overriding. `@Override` catches this.
- **`instanceof` pattern matching** (`o instanceof ArraySet oas`) both tests the type (false for `null`) and binds a correctly typed variable. The pre-Java-16 style with null checks, `getClass()`, and explicit casts is obsolete: do not write it.
- A good `equals`: optional `this == other` fast path, `instanceof` check, size check, then verify every one of my elements is contained in the other.
- This was the last in-scope lecture for Mini-Midterm 1 (no cheat sheet, a reference sheet is provided); these topics are also what you need for the last part of Project 2.
