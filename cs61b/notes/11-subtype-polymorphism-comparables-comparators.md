<!-- Mon, Sep 21, 2026 | sources: slides + code + textbook (no transcript available) -->
# Lecture 11: Subtype Polymorphism, Comparables, Comparators

## Overview

This lecture digs deeper into inheritance by contrasting two ways of telling a general-purpose routine (like a `max` function) how to compare objects: **polymorphism** and **function passing**. Python supports both: it uses *operator overloading* (defining `__gt__`) to give a class a natural order, and *function passing* (the `key=` argument) to specify an alternate order. Java has neither operator overloading nor idiomatic function passing in this course, so it uses **subtype polymorphism** for both jobs: a class declares `implements Comparable<T>` and overrides `compareTo` to define its one natural order, and separate classes implement `Comparator<T>` with a `compare(T, T)` method to define any number of alternate orders. Along the way the lecture works through a compilation-error puzzle about *which file* fails to compile when an interface method or an `implements` clause is missing, and closes by carefully distinguishing Comparable-vs-Comparator from the superficially similar Iterable-vs-Iterator pair. Note the instructor's announcement: this lecture is **not in scope for midterm 1**, but the ideas return with TreeMaps, TreeSets, and Priority Queues.

---

## Key Concepts

### 1. Two ways to tell a routine how to compare

Imagine you are writing a generic `max` function. It must, at some point, ask "is `a` bigger than `b`?" There are two fundamentally different ways to supply that answer:

- **Polymorphism**: the objects themselves know how to be compared. `max` just says `item > max_value` (Python) or `items[i].compareTo(items[maxDex])` (Java) and the *object's own type* determines which code runs.
- **Function passing**: the caller hands `max` a separate function, and `max` calls it on the items. `max(doglist, key=name_len)`.

The lecture's two Python demo files are exactly these two designs side by side:

```python
# polymorphic_max_demo.py               # function_passing_max_demo.py
def get_the_max(x):                     def get_the_max(x, key):
    max_value = x[0]                        max_value = x[0]
    for item in x:                          for item in x:
        if item > max_value:                    if key(item) > key(max_value):
            max_value = item                        max_value = item
    return max_value                        return max_value
```

The left version works on `[1, 2, 3, 4, 5]` *and* on a list of `Dog`s, as long as `Dog` defines `__gt__`. The right version works on anything at all, as long as the caller supplies a key.

The lecture's thesis: **Python uses both freely; idiomatic Java relies much more heavily on polymorphism.** Java lambdas exist, but the course explicitly says: "We won't even discuss function passing in Java in our course, but it does exist."

### 2. Polymorphism, defined

Quoting the slide (which quotes Wikipedia): polymorphism is "the ability in programming to present the same programming interface for differing underlying forms." The `>` operator in Python is one interface presented for many underlying forms (ints, strings, Dogs). In Java, a method signature declared in an interface is one interface presented for many underlying implementing classes.

### 3. Python's flavor: operator overloading and duck typing

```python
class Dog:
    def __init__(self, name, size):
        self.name = name
        self.size = size
    def __gt__(self, other):
        return self.size > other.size

hadi = Dog("hadi", 30)
zora = Dog("zora", 44)
print(hadi > zora)   # uses the __gt__ function
```

Python has a *universal* `>` operator that any class may overload by defining `__gt__`. Python is **duck typed**: you never have to declare anywhere that a `Dog` is a comparable thing. If `__gt__` exists when `>` is evaluated, it works; if not, you get a runtime error. There is no compile-time contract.

### 4. Java's flavor: subtype polymorphism via interfaces

Java has no operator overloading. `d1 > d2` on two `Dog`s is simply not legal Java, ever. (The code comment in `CollectionsDogDemo.java` says it outright: "operator overloading in Java does not exist".)

So how do you say "a `Dog` is a thing that can be compared"? The lecture poses this as a question and answers it: **you must implement some interface.** This is the same move made earlier in the course when `SLList implements List61B`: an interface names a capability, and a class declares "I have that capability" with `implements`.

The slide's diagram analogy:

```
    List61B                 Comparable<Dog>
       ^                          ^
       |                          |
    SLList                       Dog
```

This mechanism is called **subtype polymorphism**, and the lecture breaks it into three parts:

1. A **supertype** (`Comparable`) specifies the capability (comparison).
2. A **subtype** (`Dog`) overrides the supertype's abstract method.
3. **Java decides what to do at runtime** based on the dynamic type of the object invoking the method.

That third point is the crux: when `Collections.max` runs `items[i].compareTo(...)`, the compiler only knows `items[i]` is a `Comparable`. The actual method body that runs is picked at runtime from the object's real class. This is dynamic method selection, and it is what lets one `max` implementation work for every class in the world that implements `Comparable`.

### 5. Why `Collections.max(dogs)` fails without `Comparable`

```java
List<Dog> dogs = new ArrayList<>();
dogs.add(new Dog("Grigometh", 200));
dogs.add(new Dog("Pelusa", 5));
dogs.add(new Dog("Clifford", 9000));
Dog maxDog = Collections.max(dogs);   // incomprehensible error message
```

The slides note the error message is incomprehensible, and the reason underneath is simple: **`max` doesn't know how to compare two `Dog` objects.** `Collections.max` is written once, for all types, and its body can only call `compareTo`. `Dog` (as given in `Dog.java`) has no `compareTo`, so the call cannot be type-checked. The fix is to give `Dog` the capability.

### 6. The `Comparable` interface itself

The lecture makes a point of the fact that "almost all of Java is written in Java," so you can just go read `Comparable.java` on GitHub. Its essential content:

```java
public interface Comparable<T> {
    /**
     * Compares this object with the specified object for order.
     * Returns a negative integer, zero, or a positive integer
     * as this object is less than, equal to, or greater than the
     * specified object.
     */
    public int compareTo(T o);
}
```

Two observations from the slides:

- The **contract is about sign, not magnitude**: negative means "this is less," zero means "equal," positive means "this is greater." Nothing promises `-1`/`0`/`1`.
- The real file is **142 lines, almost all documentation**. "This is not uncommon with really important parts of the library. Documentation is important."

### 7. Implementing `Comparable` on `Dog`

Two steps: add `implements Comparable<Dog>` to the class header, and override `compareTo`.

```java
public class Dog implements Comparable<Dog> {
    ...
    @Override
    public int compareTo(Dog uddaDog) {
        if (size > uddaDog.size) {
            return 1;
        }
        if (size < uddaDog.size) {
            return -1;
        }
        return 0;
    }
}
```

The lecture then shows the "better approach," which it calls "very common in Java":

```java
public class Dog implements Comparable<Dog> {
    ...
    @Override
    public int compareTo(Dog uddaDog) {
        return size - uddaDog.size;
    }
}
```

This works precisely because the contract only cares about sign. `200 - 5` is positive, which is all `max` needs to know.

With this in place, `Collections.max(dogs)` compiles and returns Clifford.

### 8. Natural order vs. alternate orders

The lecture introduces **natural order**: "the ordering implied by a Comparable's `compareTo` method." For `Dog` as defined, the natural order is by `size`, so the ordering is Pelusa (5), Grigometh (200), Clifford (9000).

But you often want a *different* order, for example alphabetically by name: Clifford, Grigometh, Pelusa. A class only gets **one** natural order, because it only gets one `compareTo`. So alternate orders need a different mechanism.

### 9. `Comparator`: comparison packaged as an object

In Python, the alternate order is supplied by function passing (`key=name_len`). In Java, we again use subtype polymorphism: we define a class whose *whole job* is to compare two `Dog`s, and we pass an **instance** of it.

```java
public interface Comparator<T> {
    int compare(T o1, T o2);
    ...
}
```

```java
public class NameComparator implements Comparator<Dog> {
    @Override
    public int compare(Dog a, Dog b) {
        return a.name.compareTo(b.name);
    }
}
```

Note the delegation: `String` already implements `Comparable<String>`, so `a.name.compareTo(b.name)` reuses the natural order of strings and returns a correctly-signed int. This is the standard way to write a `Comparator` over a field.

The slides note `Comparator` has "a LOT of default methods. We won't talk about them."

Usage:

```java
Dog maxNameDog = Collections.max(dogs, new Dog.NameComparator());
```

The slide annotates this explicitly: **"This second argument is an object of type `Comparator<Dog>`."** Python passes a function; Java passes an object that wraps a function.

### 10. Avoiding the awkward instantiation

The instructor finds `new Dog.NameComparator()` at every call site "awkward and aesthetically unpleasant." The fix shown is one pre-instantiated static reference, named in all caps by convention:

```java
public static NameComparator NAME_COMPARATOR = new NameComparator();
```

```java
Dog maxNameDog = Collections.max(dogs, Dog.NAME_COMPARATOR);
```

This is exactly the commented-out line in `CollectionsDogDemo.java`:
```java
//Dog maxByName = Collections.max(dogs, Dog.NAME_COMPARATOR);
```

Note the comparator is written as a **nested static class** of `Dog` in the quiz slides (`public static class NameComparator implements Comparator<Dog>`), which is why it is referred to as `Dog.NameComparator`.

### 11. Bonus (not tested): lambdas

Marked on the slides as a bonus: "We will not teach Java lambdas in 61B, nor will you be expected to learn them."

```java
Comparator<Dog> dc = (d1, d2) -> d1.name.compareTo(d2.name);
Dog maxNameDog = Collections.max(dogs, dc);
```

The lambda "defines and instantiates a Comparator object" in one line. Java does have function passing; it just isn't this course's idiom.

### 12. Comparable vs. Comparator, and why they are *not* like Iterable vs. Iterator

This is a deliberate warning on the slides, because the name pairs look parallel and are not.

| Interface | Meaning | Method |
|---|---|---|
| `Comparable<T>` | "I can be compared to another object." | `int compareTo(T other)` |
| `Comparator<T>` | "I can tell you how to compare two objects." | `int compare(T x1, T x2)` |
| `Iterable<T>` | "I can give you an iterator." | `Iterator<T> iterator()` |
| `Iterator<T>` | "I can feed you objects." | `boolean hasNext()`, `T next()` |

The Iterable/Iterator pair is a *factory* relationship: an Iterable hands you an Iterator. Comparable and Comparator have no such relationship at all. They are two independent, alternative answers to the same question ("how do I order these?"), one intrinsic and one extrinsic.

Structurally:
- `Comparable` is implemented **by the class being compared**. There can be only one such order per class.
- `Comparator` is implemented **by some other class**, comparing "extrinsically." You may have many: `NameComparator`, `SpeedComparator`, `SizeComparator`, all implementing `Comparator<Dog>`.

### 13. Style asides from the opening of lecture

Before the main topic, the lecture reviewed Project 1 style. These are not exam content but are graded style expectations (mandatory "starting a bit after the midterm"):

- **Make code obvious and easy to read.** It is fine to create variables that exist only to *name* things; the performance penalty is negligible or nonexistent. Compare:

```java
// student version                        // instructor version
public T removeFirst() {                  public T removeFirst() {
    if (!isEmpty()) {                         if (size == 0) {
        T replace = sentinel.next.item;           return null;
        sentinel.next = sentinel.next.next;   }
        sentinel.next.prev = sentinel;
        size--;                               Node oldFront = sentinel.next;
        return replace;                       Node newFront = sentinel.next.next;
    }
    return null;                              sentinel.next = newFront;
}                                             newFront.prev = sentinel;

                                              size -= 1;
                                              return oldFront.value;
                                          }
```

- **Good style**: consistent spacing, camelCase variable names in Java.
- **Project 2 spoiler**: do not repeat non-obvious modulus math everywhere.

```java
// repeated everywhere:
T removed = items[(nextLast - 1 + items.length) % items.length];
items[(nextLast - 1 + items.length) % items.length] = null;

// instead, one helper:
private int wrapIndex(int index) {
    return (index + items.length) % items.length;
}
T removed = items[wrapIndex(nextLast - 1)];
items[wrapIndex(nextLast - 1)] = null;
```

---

## Definitions

**Polymorphism**: "The ability in programming to present the same programming interface for differing underlying forms." One calling syntax, many possible underlying implementations.

**Subtype polymorphism**: The flavor of polymorphism used in Java, in which a supertype (interface or superclass) declares a capability as an abstract method, subtypes override it, and Java selects which implementation to run **at runtime** based on the type of the object invoking the method.

**Operator overloading**: Python's flavor of polymorphism for comparison, in which a universal operator (such as `>`) is given a class-specific meaning by defining a special method (such as `__gt__`). Java does not have operator overloading.

**Function passing**: Supplying behavior to a routine by handing it a function as an argument (for example Python's `key=` parameter to `max`). Java supports it via lambdas, but 61B does not teach it.

**Duck typing**: Python's approach, in which you do not have to declare whether a capability (such as `>`) is available; it is simply attempted when used.

**Interface**: The Java construct used to declare that a class has a certain capability. To give a class a capability in Java, "we must implement some interface."

**`Comparable<T>`**: The Java interface declaring that an object can compare *itself* to another object. Its single abstract method is `public int compareTo(T o)`, which returns a negative integer, zero, or a positive integer as this object is less than, equal to, or greater than the specified object.

**`compareTo(T o)`**: The `Comparable` method. Contractually only the **sign** of the return value matters.

**Natural order**: The ordering implied by a class's `compareTo` method. Each class has at most one. For `Dog`, it is by `size`.

**`Comparator<T>`**: The Java interface for objects "designed for comparing other objects," that is, comparing *extrinsically*. Its central abstract method is `int compare(T o1, T o2)`. A class may have many distinct Comparators, each specifying one order.

**`compare(T o1, T o2)`**: The `Comparator` method. Returns a negative, zero, or positive int as `o1` is less than, equal to, or greater than `o2`.

**`Collections.max(collection)`**: Library method returning the maximum element by natural order; requires the elements to be `Comparable`.

**`Collections.max(collection, comparator)`**: Overload taking a `Comparator<T>` object as its second argument, returning the maximum element by that comparator's order.

**`@Override`**: Annotation asserting that the tagged method overrides a method from a supertype. If it does not actually override anything, the file fails to compile.

**Lambda (Java)**: Syntax `(d1, d2) -> ...` that defines and instantiates an implementing object of a functional interface in one expression. Bonus material; not taught or tested in 61B.

---

## Worked Examples

### Example 1: The polymorphic `max` in Python

```python
def get_the_max(x):
    max_value = x[0]
    for item in x:
        if item > max_value:
            max_value = item
    return max_value

class Dog:
    def __init__(self, name, size):
        self.name = name
        self.size = size
    def __gt__(self, other):
        return self.size > other.size

max_value = get_the_max([1, 2, 3, 4, 5])

list_of_dogs = [Dog("Grigometh", 10),
                Dog("Pelusa", 5),
                Dog("Clifford", 9000)]
max_dog = get_the_max(list_of_dogs)
```

Step by step:

1. `get_the_max([1, 2, 3, 4, 5])`: `max_value` starts at `1`. Each `item > max_value` uses int comparison. Result: `5`.
2. `get_the_max(list_of_dogs)`: `max_value` starts as the Grigometh object. The loop evaluates `item > max_value` on two `Dog` objects. Python looks up `__gt__` on `Dog` and runs `self.size > other.size`.
   - Grigometh vs Grigometh: `10 > 10` is False.
   - Pelusa vs Grigometh: `5 > 10` is False.
   - Clifford vs Grigometh: `9000 > 10` is True, so `max_value` becomes Clifford.
3. Result: Clifford.

The point: **`get_the_max` was not modified at all** to handle Dogs. One interface (`>`), two underlying forms (int and Dog). That is polymorphism.

### Example 2: The function-passing `max` in Python

```python
def get_the_max(x, key):
    max_value = x[0]
    for item in x:
        if key(item) > key(max_value):
            max_value = item
    return max_value

def length_of_name(dog):
    return len(dog.name)

dogs = [Dog("Grigometh", 10),
        Dog("Pelusa", 5),
        Dog("Clifford", 9000)]

max_dog = get_the_max(dogs, length_of_name)
```

Step by step:

1. `key` is bound to the *function object* `length_of_name`. Nothing is called yet; the function itself was passed as a value.
2. `max_value` starts at Grigometh. `key(max_value)` is `len("Grigometh")` = 9.
3. Grigometh: `9 > 9` False. Pelusa: `len("Pelusa")` = 6, `6 > 9` False. Clifford: `len("Clifford")` = 8, `8 > 9` False.
4. Result: Grigometh.

Note this gives a *different* answer than Example 1 on the same list, because the ordering criterion changed from size to name length. Also note `Dog.__gt__` is never consulted here: `get_the_max` never uses `>` on Dogs, only on the ints returned by `key`.

The slide version uses `max(doglist, key=name_len)` with the built-in; the demo file writes the loop out so you can see where `key` gets called.

### Example 3: `Collections.max` fails, then works

The starting code, from `CollectionsDogDemo.java` and `Dog.java`:

```java
public class Dog {
    public String name;
    public int size;

    public Dog(String n, int s) {
        name = n;
        size = s;
    }
}
```

```java
List<Dog> dogs = new ArrayList<>();
dogs.add(new Dog("Grigometh", 200));
dogs.add(new Dog("Pelusa", 5));
dogs.add(new Dog("Clifford", 9000));
Dog maxDog = Collections.max(dogs);   // does not compile
```

**Why it fails.** Picture the memory: `dogs` is a reference to an `ArrayList` object, which holds three references, to three `Dog` objects, each with a `name` reference (to a `String`) and an `int size` stored directly in the box. Nothing in any of those `Dog` boxes, and nothing in the `Dog` class, provides a `compareTo` method. `Collections.max` is written generically and its body must do something like `items[i].compareTo(items[maxDex])`. Since `Dog` is not a subtype of `Comparable`, the compiler rejects the call. As the slide says, the error message is incomprehensible, but the cause is just: **max doesn't know how to compare two Dog objects.**

**The fix.** Declare the capability and implement it:

```java
public class Dog implements Comparable<Dog> {
    public String name;
    public int size;

    public Dog(String n, int s) {
        name = n;
        size = s;
    }

    @Override
    public int compareTo(Dog uddaDog) {
        return size - uddaDog.size;
    }
}
```

**Trace of `Collections.max(dogs)`** (as sketched by the quiz slides' pseudocode for `max`): the routine walks the list keeping `maxDex`. Conceptually:

1. `maxDex = 0` (Grigometh).
2. Compare Pelusa to Grigometh: `items[1].compareTo(items[0])` calls `Dog.compareTo`, computing `5 - 200 = -195`, negative, so Pelusa is smaller. `maxDex` unchanged.
3. Compare Clifford to Grigometh: `9000 - 200 = 8800`, positive, so `maxDex = 2`.
4. Returns the `Dog` at index 2, Clifford.

The crucial dynamic-dispatch moment is step 2 and 3: `max`'s code was compiled knowing only that it holds `Comparable` references. At runtime, the objects on the heap are `Dog`s, so `Dog`'s `compareTo` body executes. `Collections.max` was never recompiled or modified.

**On `size - uddaDog.size` vs. the if-chain.** Both satisfy the contract. The if-chain returns exactly `1`, `-1`, or `0`; subtraction returns any signed int. `Collections.max` only checks the sign, so both work. (extra context: subtraction can overflow for extreme int values, for example a very large positive minus a very large negative; the lecture does not raise this, and for dog sizes it is irrelevant, but `Integer.compare(size, uddaDog.size)` avoids it.)

### Example 4: Compilation Error Puzzle #1 (hugcode.com/piano)

Setup, with the three relevant files:

```java
public class DogLauncher {                     public class Dog
  public static void main(String[] args) {     implements Comparable<Dog> {
    ...                                          ...
    Dog[] dogs = new Dog[]{d1, d2, d3};          public int compareTo(Dog o) {
    System.out.println(Collections.max(dogs));     return this.size - o.size;
  }                                              }
}                                              }

public interface Collections {   // the max function
    ...
    int cmp = items[i].compareTo(items[maxDex]);
    ...
}
```

**Q: If we omit `compareTo()`, which file fails to compile?**
A. DogLauncher.java  B. Dog.java  C. Collections.java  D. Comparable.java

**Answer: B, Dog.java.**

Reasoning: `Dog` claims `implements Comparable<Dog>`, which is a promise to provide every abstract method of that interface. If `compareTo` is missing, `Dog` is a concrete class with an unimplemented abstract method, which is a compile error *in Dog.java*. The slide adds a parenthetical: "(And I suppose DogLauncher will fail as well since Dog.class doesn't exist)," that is, the downstream failure is a consequence, but the *primary* error is in `Dog.java`.

Why not the others? `Collections.java` compiles fine: its `items[i].compareTo(...)` type-checks against the `Comparable` interface, which is unchanged. `Comparable.java` is a library file, untouched.

### Example 5: Compilation Error Puzzle #2 (hugcode.com/tiger)

Same three files. **Q: If we omit `implements Comparable<Dog>`, which file fails to compile?**

**Answer: A, DogLauncher.java.**

Reasoning: `Dog` still has a perfectly legal method named `compareTo`; a class is allowed to have any method it likes. So `Dog.java` compiles. `Collections.java` compiles, since it is written against `Comparable`. But `DogLauncher` calls `Collections.max(dogs)` where `dogs` is a `Dog[]`, and `max` requires `Comparable`s. Since `Dog` no longer declares itself a `Comparable`, **the call site in DogLauncher is what fails**: "it tries to pass things that are not Comparable, and Collections expects Comparables."

**The `@Override` wrinkle.** The slide adds: "If we used `@Override`, Dog will not compile, because we have an override tag but we're not overriding (if we omit implements Comparable)." So the answer flips to `Dog.java` when the annotation is present. This is precisely what `@Override` is for: it converts a silent semantic mistake into a loud, local compile error.

The pair of puzzles together makes the general lesson: **the location of a compile error depends on which promise was broken.**
- Broke the promise "I implement this interface's methods" → error in the *implementing* class.
- Broke the promise "this object is of the required type" → error at the *call site*.

### Example 6: Building and using a `NameComparator`

```java
public class Dog implements Comparable<Dog> {
    public String name;
    public int size;

    public Dog(String n, int s) { name = n; size = s; }

    @Override
    public int compareTo(Dog uddaDog) {
        return size - uddaDog.size;
    }

    public static class NameComparator implements Comparator<Dog> {
        @Override
        public int compare(Dog a, Dog b) {
            return a.name.compareTo(b.name);
        }
    }

    public static NameComparator NAME_COMPARATOR = new NameComparator();
}
```

```java
List<Dog> dogs = new ArrayList<>();
dogs.add(new Dog("Grigometh", 200));
dogs.add(new Dog("Pelusa", 5));
dogs.add(new Dog("Clifford", 9000));

Dog maxDog     = Collections.max(dogs);                  // Clifford (size 9000)
Dog maxNameDog = Collections.max(dogs, Dog.NAME_COMPARATOR);  // Pelusa (name "Pelusa")
```

Step by step for `maxNameDog`:

1. `Dog.NAME_COMPARATOR` evaluates to a reference to a single `NameComparator` object that was created once, when the `Dog` class was loaded. In box-and-pointer terms this object carries no instance data at all; it is just a handle whose type tells Java which `compare` body to run.
2. `Collections.max(dogs, cmp)` walks the list, calling `cmp.compare(candidate, currentMax)`.
3. Grigometh vs Grigometh: `"Grigometh".compareTo("Grigometh")` = 0.
4. Pelusa vs Grigometh: `"Pelusa".compareTo("Grigometh")` is positive ("P" after "G"), so Pelusa becomes the max.
5. Clifford vs Pelusa: `"Clifford".compareTo("Pelusa")` is negative ("C" before "P"), so no change.
6. Result: Pelusa.

Note that `maxDog` and `maxNameDog` are different dogs from the *same list*, which is the entire motivation for Comparators.

Also note the inner `a.name.compareTo(b.name)`: this is `String`'s own `compareTo`, that is, we are using `String`'s Comparable natural order to build `Dog`'s alternate order.

**Side-by-side with Python** (from the slides):

```python
def name_len(dog):
    return len(dog.name)
max(dogs, key=name_len)     # Python: pass a function directly
```
```java
Collections.max(dogs, new NameComparator());  // Java: pass an object wrapping the function
```

The slide's annotation: "In Java we package our comparison function inside of a Comparator object. We rely on subtype polymorphism."

### Example 7: Comparator Quiz (hugcode.com/lemon)

```java
IO.println("Frank".compareTo("Zeke"));   // negative number

Dog a = new Dog("Frank", 1);
Dog b = new Dog("Zeke", 1);
Comparator<Dog> nc = new Dog.NameComparator();
System.out.println(nc.compare(a, b));
```
with
```java
public static class NameComparator implements Comparator<Dog> {
    @Override
    public int compare(Dog a, Dog b) {
        return a.name.compareTo(b.name);
    }
}
```

**Q: What is the output?** A. `+1`  B. Positive Number  C. `-1`  D. Negative Number  E. Zero

**Answer: D, a Negative Number.**

Step by step:

1. `nc` has static type `Comparator<Dog>` and dynamic type `NameComparator`. The call `nc.compare(a, b)` is legal because `compare` is declared in `Comparator`; the body that runs is `NameComparator`'s.
2. Inside, `a.name` is `"Frank"` and `b.name` is `"Zeke"`.
3. `"Frank".compareTo("Zeke")` returns a negative number because "Frank" is alphabetically less than "Zeke".
4. That value is returned unchanged and printed.

**Why D and not C.** This is the point of the quiz. `String.compareTo` promises only the *sign*, not the value `-1`. (extra context: for strings differing at their first character it actually returns the difference of the char codes, here `'F' - 'Z'` = -20, but you should never rely on the magnitude.) The same trap applies to `Dog.compareTo` returning `size - uddaDog.size`: an exam asking "what does `clifford.compareTo(pelusa)` return?" wants "a positive number," not "1," unless the if-chain version is shown.

Note also that `a` and `b` both have `size == 1`, so the *natural* order would call them equal; the comparator sees them as clearly different. Different orderings, same objects.

---

## Common Pitfalls

1. **Trying to use `>` on objects in Java.** There is no operator overloading in Java, period. `d1 > d2` for `Dog`s will never compile, no matter what methods you define. Only `compareTo`/`compare`.

2. **Assuming `compareTo` returns exactly -1, 0, or 1.** The contract is *negative, zero, positive*. Writing `if (a.compareTo(b) == 1)` is a bug; write `if (a.compareTo(b) > 0)`.

3. **Getting the direction backwards.** `a.compareTo(b)` is positive when **a is greater**. Similarly `compare(o1, o2)` is positive when **o1 is greater**. Writing `return uddaDog.size - size` silently reverses your ordering and `Collections.max` will return the minimum.

4. **Forgetting the type argument.** `implements Comparable` (raw) is not the same as `implements Comparable<Dog>`. With the raw form your method must take an `Object`, and `public int compareTo(Dog d)` will not override anything.

5. **Writing `compareTo` but forgetting `implements`.** The method exists and `Dog.java` compiles, but `Dog` is not a `Comparable`, so every call site that needs a `Comparable` breaks (Puzzle #2). Conversely, writing `implements` but forgetting the method breaks `Dog.java` itself (Puzzle #1).

6. **Omitting `@Override`.** Without it, a mis-typed or mis-named method silently fails to override and the problem surfaces far away as a confusing error. With it, you get an immediate, local error.

7. **Confusing `compareTo` and `compare` signatures.** `Comparable` has a **one**-argument `compareTo` (the other operand is `this`). `Comparator` has a **two**-argument `compare` (the comparator itself is not one of the things being compared).

8. **Expecting Comparator to be produced by Comparable.** It is tempting to pattern-match onto Iterable/Iterator, where `iterator()` hands you the Iterator. There is no such link here. Comparable and Comparator are independent alternatives.

9. **Thinking a class can have several natural orders.** One `compareTo` per class, so one natural order. Multiple orders require multiple Comparators.

10. **Passing the Comparator *class* instead of an *instance*.** `Collections.max(dogs, NameComparator)` is not legal; you need `new NameComparator()` or a pre-made static instance like `Dog.NAME_COMPARATOR`.

11. **Forgetting the comparator must be an object of type `Comparator<Dog>`.** A plain helper method like `static int byName(Dog a, Dog b)` cannot be handed to `Collections.max` in the style this course teaches.

12. **Style traps carried over from the Project 1/2 discussion**: repeating non-obvious expressions (like the wraparound modulus) instead of naming them in a helper, and avoiding intermediate variables for fear of a performance cost that is "negligible or non-existent."

---

## Likely Exam Points

> Reminder from the slides: **this lecture is explicitly not in scope for Midterm 1.** The material does return later (TreeMaps, TreeSets, Priority Queues), so these points are aimed at later exams.

### 1. Which file fails to compile?

**Q:** `Dog` declares `implements Comparable<Dog>` but has no `compareTo` method. `DogLauncher` calls `Collections.max(dogs)`. Which file fails to compile first, and why?

**A:** `Dog.java`. A concrete class that claims to implement an interface must define all of that interface's abstract methods; failing to do so is an error in the class itself. (`DogLauncher.java` will also fail downstream because `Dog.class` never gets produced.)

### 2. The mirror-image case

**Q:** `Dog` has a method `public int compareTo(Dog o)` but the class header is just `public class Dog`, with no `implements`. Which file fails, and how does the answer change if the method is tagged `@Override`?

**A:** Without `@Override`, `Dog.java` compiles fine (a class may define any method), and **`DogLauncher.java`** fails, since it passes non-`Comparable`s to `Collections.max`, which expects `Comparable`s. With `@Override` present, **`Dog.java`** fails instead: the annotation asserts an override that is not actually happening.

### 3. What does `compareTo` return?

**Q:** Given `compareTo` implemented as `return size - uddaDog.size;`, with Clifford (9000) and Pelusa (5), what does `clifford.compareTo(pelusa)` return?

**A:** A positive number (specifically 8895 for this implementation, but the contractually correct answer is "positive"). A caller may only rely on the sign: negative means less than, zero means equal, positive means greater than.

### 4. Comparator output sign

**Q:** With `NameComparator.compare` returning `a.name.compareTo(b.name)`, what does `nc.compare(new Dog("Frank", 1), new Dog("Zeke", 1))` print?

**A:** A negative number, because "Frank" is alphabetically less than "Zeke". Not necessarily `-1`, so "Negative Number" is the right multiple-choice answer over "-1".

### 5. Write a Comparator

**Q:** Write a `SizeComparator` for `Dog` that orders dogs by size, and show how to use it to find the largest dog in a `List<Dog> dogs`.

**A:**
```java
public static class SizeComparator implements Comparator<Dog> {
    @Override
    public int compare(Dog a, Dog b) {
        return a.size - b.size;
    }
}
...
Dog biggest = Collections.max(dogs, new Dog.SizeComparator());
```
This duplicates the natural order, which is fine; it shows that a Comparator can express any order, including one already available.

### 6. Comparable vs. Comparator: which and why?

**Q:** You have a `Student` class. You want students sorted by ID by default, but sometimes by GPA and sometimes by last name. What goes where?

**A:** Make `Student implements Comparable<Student>` with `compareTo` comparing IDs, since that is the single natural order. Write two separate classes implementing `Comparator<Student>`, one comparing GPA and one comparing last name, and pass instances of them when an alternate order is needed. Rationale: a class has only one `compareTo`, so it has only one natural order, but it can have arbitrarily many Comparators.

### 7. Comparable/Comparator vs. Iterable/Iterator

**Q:** Is the relationship between `Comparable` and `Comparator` analogous to that between `Iterable` and `Iterator`? Give the method signatures to justify your answer.

**A:** No. `Iterable` produces an `Iterator` (`Iterator<T> iterator()`), and the `Iterator` then feeds objects (`boolean hasNext()`, `T next()`), so they are a factory/product pair. `Comparable` (`int compareTo(T other)`, "I can be compared to another object") and `Comparator` (`int compare(T x1, T x2)`, "I can tell you how to compare two objects") are two independent mechanisms for ordering, one intrinsic and one extrinsic; neither produces the other.

### 8. Python vs. Java mechanism

**Q:** In Python, `max(doglist)` works if `Dog` defines `__gt__`, and `max(doglist, key=name_len)` works by passing a function. Name the mechanism each corresponds to, and state what Java uses in each case.

**A:** `__gt__` is **operator overloading**, a form of polymorphism; `key=name_len` is **function passing**. Java uses **subtype polymorphism** for both: `implements Comparable<Dog>` with `compareTo` for the natural order, and a class implementing `Comparator<Dog>` with `compare` (an *object* passed as the second argument to `Collections.max`) for alternate orders. Java has no operator overloading at all, and although Java lambdas allow function passing, idiomatic 61B Java does not use them.

### 9. Where does dynamic dispatch happen?

**Q:** `Collections.max` was compiled long before `Dog` was ever written. How can its `items[i].compareTo(items[maxDex])` call run `Dog`'s code?

**A:** Subtype polymorphism. `max`'s code is type-checked against the supertype `Comparable`, which guarantees a `compareTo` exists. At **runtime**, Java selects the method implementation based on the actual (dynamic) type of the invoking object, so `Dog`'s overriding `compareTo` body runs.

### 10. Naming convention for a shared comparator

**Q:** Rewrite `Collections.max(dogs, new Dog.NameComparator())` so a fresh comparator is not built at each call site.

**A:** Add `public static NameComparator NAME_COMPARATOR = new NameComparator();` to `Dog`, then call `Collections.max(dogs, Dog.NAME_COMPARATOR);`. The all-caps name is the usual convention for such a static constant.

---

## Summary

- **Polymorphism** = "the same programming interface for differing underlying forms." Two ways to tell a generic routine how to compare: polymorphism, or function passing.
- **Python** uses both: `__gt__` (operator overloading) for the intrinsic order, and `key=` functions (function passing) for alternate orders. Python is **duck typed**; no capability is ever declared.
- **Java** has **no operator overloading** and idiomatically does not use function passing (61B never teaches Java lambdas). It uses **subtype polymorphism** for both jobs. To give a class a capability in Java, **implement an interface**.
- **Subtype polymorphism**: a supertype declares the capability, a subtype overrides the abstract method, and Java picks the implementation **at runtime** from the object's actual type.
- **`Comparable<T>`**: "I can be compared to another object." One method, `int compareTo(T o)`, returning **negative / zero / positive** as `this` is less than / equal to / greater than `o`. Only the sign is contractual.
  - `Dog implements Comparable<Dog>` with `return size - uddaDog.size;` is the clean, idiomatic version, and it makes `Collections.max(dogs)` work.
- **Natural order** = the order implied by `compareTo`. Exactly one per class.
- **`Comparator<T>`**: "I can tell you how to compare two objects." Method `int compare(T o1, T o2)`. Implemented by a *separate* class, so a class can have many (`NameComparator`, `SizeComparator`, `SpeedComparator`, ...).
  - `Collections.max(dogs, new Dog.NameComparator())`; the second argument is an **object** of type `Comparator<Dog>`. A pre-made `public static NameComparator NAME_COMPARATOR` avoids repeated instantiation.
  - `NameComparator.compare` delegates to `String`'s own `compareTo`.
- **Compile-error rules**: missing `compareTo` while claiming `implements` breaks the **implementing class**; missing `implements` while having the method breaks the **call site** (unless `@Override` is present, which breaks the class instead). `@Override` turns silent mistakes into local compile errors.
- **Comparable/Comparator are not Iterable/Iterator.** Iterable *produces* an Iterator; Comparable and Comparator are independent alternatives (intrinsic vs. extrinsic ordering).
- **Style**: name intermediate values with variables, factor repeated non-obvious math (`wrapIndex`) into helpers, keep spacing consistent, use camelCase. Mandatory a bit after the midterm.
- **Scope**: not on Midterm 1, but essential background for TreeMaps, TreeSets, and Priority Queues.
