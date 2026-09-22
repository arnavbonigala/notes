<!-- Mon, Sep 21, 2026 | sources: code (no transcript available) -->
# Lecture 11: Inheritance 3

## Overview

This lecture is about the last big piece of the inheritance toolkit: using interfaces to let *one* piece of code work on *many* types, and specifically how Java expresses "comparison" as an interface rather than as an operator. The lecture starts in Python, where a single `get_the_max` function works on a list of ints and on a list of `Dog`s because Python lets a class overload `>` via `__gt__`, and where you can pass a *function* (a `key`) into another function to change the notion of "biggest". Java has neither operator overloading nor (in 61B scope) first-class functions, so the same flexibility is achieved with two interfaces: `Comparable<T>`, which a class implements to define its one natural ordering via `compareTo`, and `Comparator<T>`, a separate object that packages an *alternative* ordering in a `compare` method and is passed as an argument. The lecture's running example is a `Dog` class with `name` and `size`, made comparable by size so that `Collections.max(dogs)` works, plus a `NameComparator` so that `Collections.max(dogs, Dog.NAME_COMPARATOR)` can rank dogs alphabetically instead. The course textbook chapter paired with this part of the course (Chapter 11, "Inheritance III: Iterators, Object Methods") covers the closely related material of `List`/`Set`, throwing exceptions, `Iterable`/`Iterator`, and the `Object` methods `toString()` and `equals()`; that material is included below in Part 2 so these notes cover everything the chapter assigns.

> **Note on sources.** No transcript or slide text was available for this lecture, so Part 1 is reconstructed from the lecture code files (`Dog.java`, `CollectionsDogDemo.java`, `polymorphic_max_demo.py`, `function_passing_max_demo.py`, `pick_random.py`), including the commented-out lines in those files, which show where the live coding was headed. Part 2 follows the assigned textbook chapter (which the book itself says "corresponds to Lecture 10", so it sits one lecture behind the code). Anything I add that is not directly supported by the provided material is marked **(extra context)**.

---

## Key Concepts

### 1. Why "max" is easy in Python and awkward in Java

Here is the Python version from `polymorphic_max_demo.py`:

```python
def get_the_max(x):
    max_value = x[0]
    for item in x:
        if item > max_value:
            max_value = item
    return max_value
```

This one function works on `[1, 2, 3, 4, 5]` and on a list of `Dog` objects. It works on `Dog`s only because `Dog` defines `__gt__`:

```python
class Dog:
    def __init__(self, name, size):
        self.name = name
        self.size = size

    def __gt__(self, other):
        return self.size > other.size
```

In Python, `item > max_value` is really a call to `item.__gt__(max_value)`. Python looks up `__gt__` on the actual runtime object, so `>` means "compare ints" for ints and "compare sizes" for `Dog`s. This is **operator overloading**: the class gets to redefine what an operator means for its instances.

Java deliberately does not have operator overloading. The comment in `CollectionsDogDemo.java` says it outright:

```java
// operator overloading in Java does not exist
```

In Java, `>` is defined only for primitive numeric types. If you write `if (item > maxValue)` where `item` is a `Dog`, the compiler rejects it: there is no meaning for `>` on reference types, and you cannot give it one. So the Python trick is unavailable, and we need a different mechanism to write a single `max` that works on many types.

### 2. The Java answer: put comparison behind an interface

Java's replacement for operator overloading is an ordinary **method** declared in an **interface**. If every "comparable" class promises to have a method with a known name and signature, then generic code can call that method and rely on **dynamic method selection** to run the right implementation.

Java provides this interface in the standard library:

```java
public interface Comparable<T> {
    int compareTo(T o);
}
```

The contract of `compareTo` is numeric, not boolean:

- `this.compareTo(o) < 0` means `this` is **less than** `o`
- `this.compareTo(o) == 0` means they are **equal** in this ordering
- `this.compareTo(o) > 0` means `this` is **greater than** `o`

Only the *sign* matters. The magnitude is unspecified. Returning a number instead of a boolean is what lets one method express all three outcomes, which is what sorting and searching code needs.

The intuition to hold onto: `implements Comparable<Dog>` is a promise in the type system. It says "a `Dog` knows how to rank itself against another `Dog`". Any library method that only needs that promise (max, min, sort, binary search, `TreeSet`) can then accept `Dog`s without ever having heard of dogs.

### 3. `Collections.max` and what it demands of you

`CollectionsDogDemo.java` is the payoff:

```java
List<Dog> dogs = new ArrayList<>();
dogs.add(new Dog("Grigometh", 200));
dogs.add(new Dog("Pelusa", 5));
dogs.add(new Dog("Clifford", 9000));

Dog maxDog = Collections.max(dogs);
```

`java.util.Collections` is a class of static utility methods that operate on collections. `Collections.max(collection)` walks the collection and calls `compareTo` on the elements. Its type signature requires the element type to be `Comparable`, so this line **does not compile** with the `Dog.java` exactly as shipped in the lecture folder (that `Dog` has only `name`, `size`, and a constructor). The live-coding step is to add the interface:

```java
public class Dog implements Comparable<Dog> {
    ...
    @Override
    public int compareTo(Dog otherDog) {
        return this.size - otherDog.size;
    }
}
```

Once that exists, `Collections.max(dogs)` compiles and returns `Clifford` (size 9000). Notice how much work the interface saved: we wrote zero lines of max-finding logic and we never edited `Collections`.

This is the core inheritance idea of the lecture. `Collections.max` has a **static type** view of the world (it sees things only as `Comparable`), while at runtime the **dynamic type** `Dog` supplies the actual `compareTo` body. Interface inheritance gives the compiler its guarantee; dynamic method selection gives the program its behavior.

### 4. One natural ordering is not enough: `Comparator`

`Comparable` gives a class exactly **one** ordering, its "natural order". But `Dog` has two sensible orderings (by size and by name), and the lecture's Python file shows the flexible alternative directly. In `function_passing_max_demo.py`:

```python
def get_the_max(x, key):
    max_value = x[0]
    for item in x:
        if key(item) > key(max_value):
            max_value = item
    return max_value

def length_of_name(dog):
    return len(dog.name)

max_dog = get_the_max(dogs, length_of_name)
```

Here `length_of_name` is passed *as a value*. Python functions are first-class objects, so "how to rank" can be handed in at the call site. Java (in 61B scope) cannot pass a bare method as an argument. The Java workaround is to pass an **object whose job is to hold that method**. That object's type is:

```java
public interface Comparator<T> {
    int compare(T o1, T o2);
}
```

Read the two interfaces side by side and the difference is clear:

| | `Comparable<T>` | `Comparator<T>` |
|---|---|---|
| Method | `int compareTo(T other)` | `int compare(T a, T b)` |
| Arguments | one (compares to `this`) | two (compares two outsiders) |
| Who implements it | the class being ordered | a separate, usually tiny, helper class |
| How many per class | one natural order | as many as you like |
| Package | `java.lang` (no import) | `java.util` (must import) |

The lecture's `Dog.java` already has `import java.util.Comparator;` at the top even though the shipped code never uses it, which is the tell that the live-coded version adds a comparator. And `CollectionsDogDemo.java` has the target line commented out:

```java
//Dog maxByName = Collections.max(dogs, Dog.NAME_COMPARATOR);
```

That line tells you three things about the intended design: the comparator lives inside `Dog`, it is reachable as a `static` member (accessed on the class, not on an instance), and it is passed as the second argument to a two-argument overload of `Collections.max`.

### 5. Why the comparator is a private static nested class exposed as a constant

The idiomatic 61B pattern is:

```java
private static class NameComparator implements Comparator<Dog> {
    public int compare(Dog d1, Dog d2) {
        return d1.name.compareTo(d2.name);
    }
}

public static final Comparator<Dog> NAME_COMPARATOR = new NameComparator();
```

Reasoning about each piece:

- **Nested inside `Dog`**: the comparator is conceptually part of `Dog`'s public story, and nesting keeps the file count down.
- **`static` nested class**: a `NameComparator` does not need a particular enclosing `Dog` instance to do its work, so it should not hold a hidden reference to one. (Contrast with `ArraySetIterator` in Part 2, which is a non-static inner class precisely *because* it needs the enclosing set's `items` and `size`.)
- **`private` class, `public static final` field**: callers do not need the class name, only an object that can compare dogs. Handing out the interface type `Comparator<Dog>` hides the implementation and lets you swap it later. Callers write `Dog.NAME_COMPARATOR`, never `new Dog.NameComparator()`.
- **`d1.name.compareTo(d2.name)`**: `String` already implements `Comparable<String>`, so we delegate rather than reimplement lexicographic comparison. Reusing an existing `compareTo` inside a `compare` is extremely common.

With that in place, `Collections.max(dogs, Dog.NAME_COMPARATOR)` returns `Pelusa`, since "Pelusa" > "Grigometh" > "Clifford" lexicographically.

### 6. Callbacks, and the shape of the idea

The general pattern in both languages is a **callback** (also called a higher order function in Python): general-purpose code (`get_the_max`, `Collections.max`) calls back into code you supply (`length_of_name`, `NAME_COMPARATOR.compare`) to make the one decision it cannot make itself. Python packages the callback as a function; Java packages it as an object implementing an interface. Same idea, different container.

The lecture code notes the modern Java shortcut but explicitly puts it out of scope:

```java
// Not in scope for 61B but you can also use a lambda
// expression in Java to define a Comparator rather than a
// separate class. Lambdas in Java are done using ->
//
// Comparator<Dog> dc = (d1, d2) -> d1.name.compareTo(d2.name);
```

Treat lambdas as "nice to know, do not use on the exam unless told you may".

### 7. `pick_random.py` and dynamic typing

```python
import random

def pick_random(list):
   index = random.randint(0, len(list) - 1)
   return list[index]

x = [1, 2, 3, 4, 5]
print(pick_random(x))
```

This tiny file makes the contrast concrete: in Python, a helper works on a list of anything with no type annotations at all, because types are checked (if at all) at runtime. Java gets the same reuse only by being explicit about it, either with generics (`static <T> T pickRandom(List<T> items)`) or with interfaces. The recurring theme of the lecture is that Java's compile-time checking costs you ceremony (`implements Comparable<Dog>`, a whole `NameComparator` class) and buys you guarantees the compiler can enforce before you run anything. *(The interpretation of this file's role is a reconstruction from context, since no slide text was available.)*

---

## Definitions

**Operator overloading**: giving an operator such as `>` or `+` a class-specific meaning by defining a special method (Python's `__gt__`, `__add__`, etc.). Python supports it; **Java does not**, which is why Java needs comparison interfaces.

**Higher order function / callback**: a function or object passed into other code so that the other code can call it to customize behavior. Python's `key=length_of_name` argument is one; a Java `Comparator` object is the Java equivalent.

**`Comparable<T>`** (in `java.lang`): interface with the single abstract method `int compareTo(T o)`. A class implements it to declare its one **natural ordering**. Returns negative if `this` is less than `o`, zero if equal, positive if greater.

**Natural order**: the ordering defined by a class's own `compareTo`. For `Dog` in this lecture, order by `size`.

**`Comparator<T>`** (in `java.util`): interface with the method `int compare(T o1, T o2)`, used to define an ordering *outside* the class being ordered, so that a type can have many orderings. Same sign convention as `compareTo`.

**`Collections`**: the `java.util` utility class of static methods on collections. `Collections.max(list)` uses natural order; `Collections.max(list, comparator)` uses the supplied comparator.

**Static nested class**: a class declared inside another with the `static` keyword. It has no reference to an enclosing instance, which suits helper types such as comparators.

**Inner (non-static nested) class**: a class declared inside another without `static`. Each instance is tied to an enclosing instance and can read its fields directly, which suits iterators.

**Dynamic method selection**: at runtime, an overridden/implemented instance method call is dispatched using the object's dynamic type, not the static type of the variable. This is what makes `Collections.max` run *Dog's* `compareTo`.

**Lambda expression** (mentioned, out of scope for 61B): compact Java syntax such as `(d1, d2) -> d1.name.compareTo(d2.name)` for creating an object implementing a one-method interface.

*Textbook chapter terms:*

**`Set`**: a collection of unique elements with no notion of order or indexing. `HashSet` is an implementation.

**`List`**: an ordered collection allowing duplicates and index access. `ArrayList` is an implementation.

**Exception**: an object that, when thrown, halts normal control flow. Thrown with `throw new ExceptionObject(args)`.

**`Iterable<T>`**: interface with `Iterator<T> iterator()`. A class must implement it to be usable in an enhanced for loop.

**`Iterator<T>`**: interface with `boolean hasNext()` and `T next()`. The object that actually steps through an iterable.

**`toString()`**: `Object` method returning a string representation; implicitly called by `System.out.println` and by string concatenation. The default implementation prints the class name and a hexadecimal memory-derived value.

**`equals(Object o)`**: `Object` method for value equality; by default behaves like `==`. Meant to be overridden.

**`==`**: checks whether two boxes hold the same bits. For primitives, equal values; for references, the same address (the same object).

---

## Worked Examples

### Example 1: Python's polymorphic max, traced

```python
def get_the_max(x):
    max_value = x[0]
    for item in x:
        if item > max_value:
            max_value = item
    return max_value

list_of_dogs = [Dog("Grigometh", 10),
                Dog("Pelusa", 5),
                Dog("Clifford", 9000)]

max_dog = get_the_max(list_of_dogs)
```

Step by step:

1. `max_value = x[0]`, so `max_value` now refers to the same `Dog` object as `list_of_dogs[0]`, the Grigometh object. In box-and-pointer terms, two names point at one box; nothing is copied.
2. Iteration 1: `item` is Grigometh. `item > max_value` calls `Grigometh.__gt__(Grigometh)`, evaluating `10 > 10`, which is `False`. No change.
3. Iteration 2: `item` is Pelusa. `5 > 10` is `False`. No change.
4. Iteration 3: `item` is Clifford. `9000 > 10` is `True`, so `max_value` is repointed at the Clifford object.
5. Returns the Clifford object.

Then `print(f"The max dog is: {max_dog}")` prints something like `The max dog is: <__main__.Dog object at 0x7f...>`, because `Dog` defines `__gt__` but no `__str__`. That is exactly the motivation for `toString()` in Part 2: the default representation shows identity, not content.

The essential observation: `get_the_max` never mentions `Dog`. It works because the objects it receives happen to support `>`. Swap in `[1, 2, 3, 4, 5]` and the same body runs integer comparison instead.

### Example 2: the same function with a key (callback)

```python
def get_the_max(x, key):
    max_value = x[0]
    for item in x:
        if key(item) > key(max_value):
            max_value = item
    return max_value

def length_of_name(dog):
    return len(dog.name)

dogs = [Dog("Grigometh", 10), Dog("Pelusa", 5), Dog("Clifford", 9000)]
max_dog = get_the_max(dogs, length_of_name)
```

Now `get_the_max` compares `key(item)` values, which are plain ints, so `Dog.__gt__` is never used at all. Name lengths are Grigometh = 9, Pelusa = 6, Clifford = 8, so the answer is **Grigometh**, a different dog than the size-based version picked. The comparison rule has moved out of the class and into an argument, which is precisely the flexibility `Comparator` provides in Java.

(Two harmless quirks in the lecture file: `length_of_name` is defined twice, the second definition simply replacing the identical first, and `Dog.__gt__` is present but unused in this version.)

### Example 3: making `Dog` comparable in Java

Starting point, as shipped:

```java
package lec11_inheritance3;

import java.util.Comparator;

public class Dog {
    public String name;
    public int size;

    public Dog(String n, int s) {
        name = n;
        size = s;
    }
}
```

Completed version (the live-coding target, reconstructed from the demo's commented code):

```java
package lec11_inheritance3;

import java.util.Comparator;

public class Dog implements Comparable<Dog> {
    public String name;
    public int size;

    public Dog(String n, int s) {
        name = n;
        size = s;
    }

    @Override
    public int compareTo(Dog otherDog) {
        return this.size - otherDog.size;
    }

    /** Orders Dogs alphabetically by name instead of by size. */
    private static class NameComparator implements Comparator<Dog> {
        public int compare(Dog d1, Dog d2) {
            return d1.name.compareTo(d2.name);
        }
    }

    public static final Comparator<Dog> NAME_COMPARATOR = new NameComparator();
}
```

Why each line matters:

- `implements Comparable<Dog>`: the generic parameter is `Dog`, so `compareTo` takes a `Dog` and the compiler checks the argument type for us. Without the type parameter (raw `Comparable`), `compareTo` would take an `Object` and you would have to cast, risking a `ClassCastException` at runtime **(extra context)**.
- `return this.size - otherDog.size;`: negative when `this` is smaller, zero when equal, positive when larger. Exactly the contract. *(Caution, extra context: subtraction can overflow for extreme `int` values, so `Integer.compare(this.size, otherDog.size)` is the safe general form. For dog sizes it is fine, and the subtraction form is the one 61B normally writes.)*
- `@Override`: not required, but it makes the compiler verify that you really are implementing the interface method. If you typo the name or the parameter type, you get a compile error instead of a silent bug.

### Example 4: running `CollectionsDogDemo`

```java
List<Dog> dogs = new ArrayList<>();

dogs.add(new Dog("Grigometh", 200));
dogs.add(new Dog("Pelusa", 5));
dogs.add(new Dog("Clifford", 9000));

Dog maxDog = Collections.max(dogs);
Dog maxByName = Collections.max(dogs, Dog.NAME_COMPARATOR);
```

Memory reasoning, in words: `dogs` is a local variable holding the address of an `ArrayList` object. That `ArrayList` holds (among other fields) an array whose slots hold addresses of three separate `Dog` objects. Each `Dog` object holds an `int` `size` directly in its box and an address pointing at a `String` object for `name`. `Collections.max` returns one of those addresses, copied into `maxDog`. So `maxDog` and `dogs.get(2)` are the *same* object: mutating `maxDog.size` would change what you see through the list.

The static type of `dogs` is `List<Dog>`, an interface, while its dynamic type is `ArrayList<Dog>`. `Collections.max` sees only an even weaker view (`Collection<? extends T>` where `T` is `Comparable`), yet the calls it makes land in `ArrayList`'s iterator and in `Dog.compareTo`. That is inheritance doing its job in both directions: interfaces for the compiler, dynamic dispatch for the runtime.

Results:
- `Collections.max(dogs)` compares by `size`: 200 vs 5 vs 9000, so `maxDog` is **Clifford**.
- `Collections.max(dogs, Dog.NAME_COMPARATOR)` compares by `name`: "Clifford" < "Grigometh" < "Pelusa", so `maxByName` is **Pelusa**.

If you delete `implements Comparable<Dog>` from `Dog`, the first call fails **at compile time** with a message about `Dog` not being within the bound of the type variable. This is a favorite exam distinction: it is a compile error, not a runtime exception.

### Example 5: writing the generic max yourself (reconstruction of the standard 61B build-up)

Before reaching for the library, the classic 61B progression is worth being able to reproduce, because exams ask for it. *(This build-up is standard 61B material for this topic and a reasonable reconstruction of the live-coding path, but it is not literally in the provided files, so treat the exact form as mine.)*

Attempt 1, a max that works for `Dog` only:

```java
public static Dog maxDog(Dog[] dogs) {
    Dog maxDog = dogs[0];
    for (Dog d : dogs) {
        if (d.size > maxDog.size) {
            maxDog = d;
        }
    }
    return maxDog;
}
```

This works but must be rewritten for `Cat`, `Student`, and every other class: no reuse at all.

Attempt 2, define your own comparison interface:

```java
public interface OurComparable {
    int compareTo(Object o);
}
```

```java
public class Maximizer {
    public static OurComparable max(OurComparable[] items) {
        int maxDex = 0;
        for (int i = 0; i < items.length; i += 1) {
            if (items[i].compareTo(items[maxDex]) > 0) {
                maxDex = i;
            }
        }
        return items[maxDex];
    }
}
```

One `max`, usable by any class that implements `OurComparable`. The costs are visible in the signature: the parameter and return type are `Object`/`OurComparable`, so callers must cast the result back to `Dog`, and `compareTo` must cast its argument, which can blow up at runtime.

Attempt 3, use Java's generic `Comparable<T>` instead, which removes both casts. In modern Java you can write the generic method directly:

```java
public static <T extends Comparable<T>> T getTheMax(List<T> items) {
    T maxValue = items.get(0);
    for (T item : items) {
        if (item.compareTo(maxValue) > 0) {
            maxValue = item;
        }
    }
    return maxValue;
}
```

Read `<T extends Comparable<T>>` as "for any type `T` that knows how to compare itself to another `T`". This is the direct Java translation of the Python `get_the_max`, and `Collections.max` is essentially this method already written for you.

And the comparator version, the translation of the Python `key` version:

```java
public static <T> T getTheMax(List<T> items, Comparator<T> comp) {
    T maxValue = items.get(0);
    for (T item : items) {
        if (comp.compare(item, maxValue) > 0) {
            maxValue = item;
        }
    }
    return maxValue;
}
```

Note that `T` needs no bound here: the ordering knowledge lives entirely in `comp`.

---

## Part 2: Textbook Chapter 11 (Lists and Sets, Exceptions, Iterators, Object Methods)

The assigned reading builds an `ArraySet` and uses it to introduce four things. These show up constantly on exams and in projects, so they are covered in full here.

### 2.1 `List` and `Set` in real Java

`List` and `Set` are **interfaces** in `java.util`, so you cannot instantiate them; you instantiate an implementation:

```java
import java.util.List;
import java.util.ArrayList;
import java.util.Set;
import java.util.HashSet;

List<Integer> L = new ArrayList<>();
L.add(5);
L.add(10);
System.out.println(L);          // [5, 10]

Set<String> s = new HashSet<>();
s.add("Tokyo");
s.add("Lagos");
System.out.println(s.contains("Tokyo"));  // true
```

Without the import you must use the canonical name: `java.util.List<Integer> L = new java.util.ArrayList<>();`.

A `Set` holds **unique** elements with **no order** and no indexing. Adding a duplicate does nothing. Consequently, adding the same items to a `Set` and a `List` always leaves the set with size less than or equal to the list's. Python's equivalent is `set()` with the `in` keyword instead of a `contains` method.

### 2.2 `ArraySet`, and throwing exceptions

`ArraySet` supports `add(value)`, `contains(value)`, and `size()`. The naive `contains` has a bug:

```java
public boolean contains(T x) {
    for (int i = 0; i < size; i += 1) {
        if (items[i].equals(x)) {   // NullPointerException if items[i] is null
            return true;
        }
    }
    return false;
}
```

If `null` was added, `items[i].equals(x)` is `null.equals(x)` and you get a `NullPointerException` from deep inside the class, with a message that tells the user nothing about what they did wrong.

Exceptions are objects, and you throw them yourself with `throw new ExceptionObject(args)` (Python's `raise`). The fix:

```java
/* Associates the specified value with the specified key in this map.
   Throws an IllegalArgumentException if the key is null. */
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

You still crash, so why is this better? Two reasons the chapter gives: (1) you control *where* the flow stops, so the failure happens at the moment of the mistake rather than later; (2) the exception type and message are informative to the person using your class. Better still would be not crashing at all: either silently refuse `null`, or make `contains` handle `items[i] == null`. Whichever you choose, document it, because the caller needs to know what to expect.

### 2.3 Iteration: what the enhanced for loop really is

This works:

```java
for (String city : s) { ... }   // s is a HashSet
```

This does not, until we do some work:

```java
for (String city : arraySet) { ... }
```

The enhanced for loop is syntactic sugar. The compiler rewrites it into:

```java
Iterator<String> seer = s.iterator();
while (seer.hasNext()) {
    String city = seer.next();
    ...
}
```

So compiling it requires two things, checked against **static types**:

1. Does the static type of `s` have an `iterator()` method? Yes, because `Set`/`List` extend `Iterable` (via `Collection`), and `Iterable<T>` declares `Iterator<T> iterator();`.
2. Does the static type of `seer` have `hasNext()` and `next()`? Yes, because `Iterator<T>` declares `boolean hasNext();` and `T next();`.

`next()` does two jobs at once: it returns the next element **and** advances the iterator, so each element is visited once.

To make `ArraySet` iterable, write an iterator class and a factory method:

```java
public Iterator<T> iterator() {
    return new ArraySetIterator();
}

private class ArraySetIterator implements Iterator<T> {
    private int wizPos;

    public ArraySetIterator() {
        wizPos = 0;
    }

    public boolean hasNext() {
        return wizPos < size;
    }

    public T next() {
        T returnItem = items[wizPos];
        wizPos += 1;
        return returnItem;
    }
}
```

`ArraySetIterator` is a **non-static inner class**, which is what lets `hasNext` say `size` and `next` say `items` with no qualification: each iterator instance is attached to the enclosing `ArraySet`. The state of the iteration is one `int`, `wizPos`, living in the iterator object, not in the set. That is why you can have several independent iterators over the same set at once: each `iterator()` call returns a fresh object with its own `wizPos` at 0.

Finally declare `public class ArraySet<T> implements Iterable<T>`, and the enhanced for loop compiles:

```java
ArraySet<Integer> aset = new ArraySet<>();
aset.add(5);
aset.add(23);
aset.add(42);
for (int i : aset) {
    System.out.println(i);
}
```

Two contract questions the chapter answers explicitly:

- **What if `next()` is called when `hasNext()` is false?** Officially undefined, but the common convention is to throw `NoSuchElementException`.
- **Is `hasNext()` always called before `next()`?** No. A caller who knows the length may skip it, so `next()` cannot assume it. You can always call `hasNext()` from inside `next()` if you need to.

Remember the division of labor: **Iterable** is the thing you can iterate over (it hands out iterators); **Iterator** is the machine that does the stepping.

### 2.4 `toString()`

Every class inherits from `Object`, which provides `toString()`, `equals(Object)`, `getClass()`, `hashCode()`, `clone()`, `finalize()`, `notify()`, `notifyAll()`, and the `wait()` overloads. This chapter focuses on the first two.

`System.out.println(dog)` is really:

```java
String s = dog.toString();
System.out.println(s);
```

`Object`'s default `toString()` yields a hexadecimal, memory-derived string, which is why unprinted custom classes look like gibberish (and exactly mirrors the Python `<Dog object at 0x...>` output from Example 1). `ArrayList` and Java arrays override it, which is why lists print nicely.

A first attempt for `ArraySet`:

```java
public String toString() {
    String returnString = "{";
    for (int i = 0; i < size; i += 1) {
        returnString += items[i];
        returnString += ", ";
    }
    returnString += "}";
    return returnString;
}
```

This is correct-ish but naive: Java `String`s are immutable, so `returnString += ...` builds an **entirely new string** each time, copying everything already there. The cost is linear in the current length, so the total is quadratic. The chapter's bonus arithmetic: if one character costs 1 second, building `{1, 2, 3, 4, 5}` costs `1 + 2 + 3 + 4 + 5 + 6 + 7` seconds, because each step re-copies the prefix.

The fix is `StringBuilder`, a mutable string object you append to in place:

```java
public String toString() {
    StringBuilder returnSB = new StringBuilder("{");
    for (int i = 0; i < size - 1; i += 1) {
        returnSB.append(items[i].toString());
        returnSB.append(", ");
    }
    returnSB.append(items[size - 1]);
    returnSB.append("}");
    return returnSB.toString();
}
```

Note the loop stops at `size - 1` and the last element is appended separately, so there is no trailing comma. *(Extra context: this version throws or misbehaves on an empty set, since `items[size - 1]` is `items[-1]`. Guard `size == 0` if asked to write a fully robust version.)*

### 2.5 `==` versus `equals`

`==` compares the contents of the two boxes. For primitives that means the values; for references that means the addresses, so `==` asks "are these literally the same object?".

```java
public class Doge {
   public int age;
   public String name;

   public Doge(int age, String name) {
      this.age = age;
      this.name = name;
   }

   public static void main(String[] args) {
      int x = 5;
      int y = 5;
      int z = 6;

      Doge fido = new Doge(5, "Fido");
      Doge doggo = new Doge(6, "Doggo");
      Doge fidoTwin = new Doge(5, "Fido");
      Doge fidoRealTwin = fido;
   }
}
```

Box-and-pointer reasoning: `x`, `y`, `z` are boxes holding `5`, `5`, `6` directly. `fido`, `doggo`, `fidoTwin` each hold the address of a **different** newly constructed object; three separate `new` calls make three separate objects, even though two of them have identical contents. `fidoRealTwin = fido` copies the address, so those two boxes hold the same arrow.

Results:

| Expression | Value | Why |
|---|---|---|
| `x == y` | `true` | same primitive value |
| `x == z` | `false` | 5 is not 6 |
| `fido == doggo` | `false` | different objects |
| `fido == fidoTwin` | `false` | different objects, identical contents |
| `fido == fidoRealTwin` | `true` | same address copied |

`fido == fidoTwin` being false is the problem: it is silly for two identical dogs to be unequal, and it makes tests useless, since an expected `ArrayList` built with `new` would never be `==` to the actual one. Hence `equals`.

`Object.equals(Object o)` behaves like `==` by default, but you can override it to define equality however your class needs. For `ArraySet`, two sets are equal if they contain the same elements:

```java
@Override
public boolean equals(Object other) {
    if (this == other) {
        return true;
    }
    if (other == null) {
        return false;
    }
    if (other.getClass() != this.getClass()) {
        return false;
    }
    ArraySet<T> o = (ArraySet<T>) other;
    if (o.size() != this.size()) {
        return false;
    }
    for (T item : this) {
        if (!o.contains(item)) {
            return false;
        }
    }
    return true;
}
```

Walking through the guards, in order and for a reason:

1. `this == other`: a fast path. If it is literally the same object, it is equal, and we skip the whole loop.
2. `other == null`: `x.equals(null)` must be `false`, never a crash.
3. `other.getClass() != this.getClass()`: ensures `aset.equals("fish")` is `false` rather than a `ClassCastException` on the next line.
4. Only now is the cast `(ArraySet<T>) other` safe.
5. Size check first, then containment. Same size plus "every element of `this` is in `o`" implies the sets are equal. Note the loop `for (T item : this)` only works because we made `ArraySet` iterable, which is a nice illustration of the chapter's pieces composing.

With the demo main method: `aset.equals(aset2)` is `true`, `aset.equals(null)` is `false`, `aset.equals("fish")` is `false`, `aset.equals(aset)` is `true`.

**Rules for `equals` in Java**:
1. It must be an **equivalence relation**: reflexive (`x.equals(x)`), symmetric (`x.equals(y)` iff `y.equals(x)`), transitive (`x.equals(y)` and `y.equals(z)` implies `x.equals(z)`).
2. It must take an **`Object`** argument, otherwise you are overloading, not overriding.
3. It must be **consistent**: if `x` and `y` are unchanged, repeated calls give the same answer.
4. `x.equals(null)` must always be **false**.

*(Extra context: Java also expects that if `x.equals(y)` then `x.hashCode() == y.hashCode()`, which matters once you put your objects in a `HashSet` or `HashMap`. That is covered later in the course.)*

---

## Common Pitfalls

**Comparison (Part 1)**

1. **Writing `if (dog1 > dog2)` in Java.** Compile error. There is no operator overloading; you must call `compareTo` or `compare` and check the sign.
2. **Making `compareTo` return a boolean.** The contract requires `int`, and a boolean cannot express three outcomes. `return this.size > other.size;` does not compile as a `compareTo`.
3. **Testing `compareTo(...) == 1` instead of `> 0`.** Only the sign is specified; an implementation may legally return 47 or -9000.
4. **Confusing the two interfaces.** `compareTo` takes **one** argument and lives in the class being ordered; `compare` takes **two** and lives in a separate comparator class. Writing `public int compare(Dog other)` inside a `Comparator` silently fails to implement the interface and produces a "does not override abstract method" error.
5. **Forgetting the `java.util.Comparator` import.** `Comparable` is in `java.lang` (automatic), `Comparator` is not.
6. **Forgetting `implements Comparable<Dog>`.** `Collections.max(dogs)` then fails at **compile time**, not with a runtime exception.
7. **Instantiating the comparator wrongly.** With a `private static class NameComparator`, outside code cannot say `new Dog.NameComparator()`. Expose a `public static final Comparator<Dog> NAME_COMPARATOR` (or a `public static Comparator<Dog> getNameComparator()`) instead.
8. **Making the comparator non-static when it does not need enclosing state**, or making an iterator static when it does. Ask: does this helper need a specific enclosing instance?
9. **Assuming the natural order and a comparator agree.** In this very lecture they disagree: max by size is Clifford, max by name is Pelusa.
10. **Using a lambda on an exam.** The lecture explicitly flags lambdas as out of 61B scope.

**Textbook material (Part 2)**

11. **`public boolean equals(Dog other)`.** This **overloads**, it does not override, so library code calling `equals(Object)` gets the default identity behavior and your method is silently ignored. Always write `equals(Object)` and always annotate with `@Override`.
12. **Casting before checking the class.** Reorder the guards and `aset.equals("fish")` throws `ClassCastException` instead of returning `false`.
13. **Forgetting the `null` guard,** producing a `NullPointerException` where `false` was required.
14. **Building strings with `+=` in a loop.** Correct but quadratic. Use `StringBuilder`.
15. **Trying to instantiate an interface**: `new List<>()` or `new Set<>()` do not compile.
16. **Confusing `Iterable` and `Iterator`.** `Iterable` has `iterator()`; `Iterator` has `hasNext()`/`next()`. A class that implements only `Iterator` cannot be used in an enhanced for loop.
17. **Storing iteration state in the collection** instead of in the iterator, which breaks nested or concurrent loops over the same object.
18. **Having `hasNext()` consume an element,** or having `next()` fail to advance. `next()` must both return and advance; `hasNext()` must have no side effects.
19. **Assuming `hasNext()` is always called first.** It is not guaranteed.
20. **Returning the same iterator object from every `iterator()` call.** Each call must return a fresh iterator positioned at the start.

---

## Likely Exam Points

**1. `Comparable` versus `Comparator`, by signature**

*Question.* Fill in both blanks. `Dog` should be ordered by `size` naturally, and a `SizeComparator` should order dogs by size as well.

```java
public class Dog implements ________ {
    public int size;
    public int compareTo(______ o) { ... }
}
public class SizeComparator implements ________ {
    public int compare(______, ______) { ... }
}
```

*Answer.* `implements Comparable<Dog>` with `public int compareTo(Dog o) { return this.size - o.size; }`, and `implements Comparator<Dog>` with `public int compare(Dog d1, Dog d2) { return d1.size - d2.size; }`. Key discriminators: one argument versus two, and `this` participating versus not.

**2. Does it compile?**

*Question.* Given the lecture's original `Dog` (no `Comparable`), what happens with `Dog maxDog = Collections.max(dogs);`?

*Answer.* A **compile-time error**. `Collections.max(Collection)` requires the element type to be `Comparable`, and `Dog` is not, so the compiler rejects the call before the program ever runs. Adding `implements Comparable<Dog>` with a `compareTo` fixes it. (The alternative fix is to call the two-argument overload with a comparator.)

**3. Predicting the result of max under two orderings**

*Question.* With `Dog("Grigometh", 200)`, `Dog("Pelusa", 5)`, `Dog("Clifford", 9000)`, and a `Dog.compareTo` comparing `size` plus a `NAME_COMPARATOR` comparing `name`, what do `Collections.max(dogs)` and `Collections.max(dogs, Dog.NAME_COMPARATOR)` return?

*Answer.* `Clifford` (largest size, 9000) and `Pelusa` (lexicographically largest name, since 'P' > 'G' > 'C'). The point being tested is that natural order and comparator order are independent.

**4. Writing a comparator from a description**

*Question.* Write a `Comparator<Dog>` that orders dogs by name **length**, shortest first, matching the Python `length_of_name` key.

*Answer.*

```java
public static class NameLengthComparator implements Comparator<Dog> {
    public int compare(Dog d1, Dog d2) {
        return d1.name.length() - d2.name.length();
    }
}
```

Passed to `Collections.max`, this returns Grigometh (9 characters).

**5. Why can't Java just use `>`?**

*Question.* In one or two sentences, explain why Python's `get_the_max` works on a list of `Dog`s but the identical Java loop does not.

*Answer.* Python lets a class overload the `>` operator by defining `__gt__`, and it looks that method up on the object at runtime. Java has no operator overloading: `>` is defined only on primitives and cannot be given meaning for reference types, so comparison must be expressed as a method (`compareTo` or `compare`) declared in an interface, and reuse comes from dynamic method selection on that interface.

**6. Static type versus dynamic type in a `Comparable` call**

*Question.* `Comparable<Dog> c = new Dog("Pelusa", 5); c.compareTo(otherDog);` Which class's `compareTo` runs, and why is the call legal?

*Answer.* It is legal because the **static type** `Comparable<Dog>` declares `compareTo(Dog)`, so the compiler is satisfied. `Dog`'s implementation runs because method selection at runtime uses the **dynamic type**, `Dog`. Note that `c.name` would **not** compile, since `Comparable` has no `name` field.

**7. Enhanced for loop desugaring**

*Question.* Rewrite `for (String city : s) { System.out.println(city); }` without the enhanced for loop, and state which interfaces `s` and the helper variable must satisfy.

*Answer.*

```java
Iterator<String> seer = s.iterator();
while (seer.hasNext()) {
    String city = seer.next();
    System.out.println(city);
}
```

The static type of `s` must be (or extend) `Iterable<String>`, supplying `iterator()`; the static type of `seer` must be `Iterator<String>`, supplying `hasNext()` and `next()`.

**8. Making a class iterable**

*Question.* What exactly must you add to `ArraySet` so `for (int i : aset)` compiles?

*Answer.* Declare `public class ArraySet<T> implements Iterable<T>`, add `public Iterator<T> iterator() { return new ArraySetIterator(); }`, and write a nested `ArraySetIterator implements Iterator<T>` with a position field, `hasNext()` returning `wizPos < size`, and `next()` returning `items[wizPos]` after incrementing. Merely implementing `Iterator` on `ArraySet` itself is not enough for the enhanced for loop.

**9. Overriding `equals` correctly**

*Question.* A student writes `public boolean equals(ArraySet<T> other) { ... }`. What is wrong, and how would you catch it?

*Answer.* It **overloads** rather than overrides, because `Object.equals` takes an `Object`. Code that calls `equals` through an `Object` reference (such as `ArrayList.contains`) will get the default identity comparison. Adding `@Override` makes the compiler flag the mistake immediately.

**10. `==` versus `equals` trace**

*Question.* With `Doge fido = new Doge(5, "Fido"); Doge fidoTwin = new Doge(5, "Fido"); Doge fidoRealTwin = fido;`, evaluate `fido == fidoTwin` and `fido == fidoRealTwin`.

*Answer.* `false` and `true`. Two `new` calls create two distinct objects regardless of identical field values; assignment copies the address, so `fidoRealTwin` points at the very same object.

**11. `toString` efficiency**

*Question.* Why is repeated `returnString += items[i]` slow, and what is the fix?

*Answer.* Java `String`s are immutable, so each `+=` allocates a brand-new string and copies the entire existing contents, making the total work quadratic in the output length. Use `StringBuilder`, which is mutable and appends in place.

**12. Throwing exceptions**

*Question.* `ArraySet.add(null)` used to crash inside `contains` with a `NullPointerException`. Show the improved behavior and say why an exception you throw yourself is better than one that happens to you.

*Answer.* `if (x == null) { throw new IllegalArgumentException("can't add null"); }` at the top of `add`. It stops the program at the point of the actual mistake, under your control, with a type and message that tell the caller what they did wrong, rather than surfacing an opaque failure from deep inside the implementation.

**13. Iterator with a twist (textbook metacognitive exercise)**

*Question.* Modify `ArraySetIterator` to take a `Comparator<T>` and a reference item `ref`, returning only items greater than `ref`.

*Answer.* Store `ref` and `comp` as fields set in the constructor, and have `next()` skip forward while `comp.compare(returnItem, ref) <= 0`:

```java
public T next() {
    T returnItem = items[pos];
    while (comp.compare(returnItem, ref) <= 0) {
        pos += 1;
        returnItem = items[pos];
    }
    pos += 1;
    return returnItem;
}
```

*(Extra context: the textbook's posted solution omits the final `pos += 1` and does not make `hasNext()` account for skipped items, so it can loop forever or run off the array. A fully correct version advances `pos` past the returned item and has `hasNext()` scan ahead for the next qualifying element. Worth understanding, since exam graders like exactly this kind of boundary reasoning.)*

---

## Summary

- Python can write one polymorphic `get_the_max` because classes overload `>` via `__gt__` and because functions are first-class values that can be passed in as a `key`.
- **Java has no operator overloading.** `>` works on primitives only, so comparison must be a method declared in an interface.
- **`Comparable<T>`** (`java.lang`): `int compareTo(T o)`. Implemented by the class itself; defines its single **natural order**. Returns negative / zero / positive; only the sign matters.
- **`Comparator<T>`** (`java.util`, needs an import): `int compare(T a, T b)`. A separate object, so a type can have arbitrarily many orderings, and the ordering can be chosen at the call site. This is Java's stand-in for passing a function.
- The lecture's `Dog` implements `Comparable<Dog>` comparing `size`, and exposes a `NameComparator` as `public static final Comparator<Dog> NAME_COMPARATOR`. The comparator is a **static** nested class because it needs no enclosing `Dog`.
- `Collections.max(dogs)` returns Clifford (size 9000); `Collections.max(dogs, Dog.NAME_COMPARATOR)` returns Pelusa. The library code is written once and works on any type, via interfaces at compile time plus dynamic method selection at runtime.
- Omitting `implements Comparable<Dog>` makes `Collections.max(dogs)` a **compile-time** error, not a runtime one.
- Lambdas (`(d1, d2) -> ...`) can replace a comparator class but are explicitly out of scope for 61B.
- `List` and `Set` are interfaces; instantiate `ArrayList` / `HashSet`. Sets hold unique, unordered elements.
- Throw your own exceptions with `throw new IllegalArgumentException("...")` to stop the program where the mistake actually is, with a useful type and message. Document the behavior.
- The enhanced for loop desugars to `iterator()`, `hasNext()`, `next()`. **`Iterable`** supplies `iterator()`; **`Iterator`** supplies `hasNext()` and `next()`, and `next()` both returns and advances.
- To iterate your own class: `implements Iterable<T>`, return a fresh nested iterator from `iterator()`, and keep the position in the iterator (an inner class, so it can read the enclosing `items` and `size`).
- Every class inherits from `Object`. Override **`toString()`** for readable printing (use `StringBuilder`, since `+=` on strings is quadratic) and **`equals(Object)`** for value equality.
- A correct `equals` checks `this == other`, then `other == null`, then `getClass()`, then casts, then compares contents; it must be reflexive, symmetric, transitive, consistent, and false for `null`, and must take an `Object` (use `@Override` to prove it).
- `==` compares box contents: values for primitives, addresses for references. Two separately constructed objects with identical fields are never `==`.
