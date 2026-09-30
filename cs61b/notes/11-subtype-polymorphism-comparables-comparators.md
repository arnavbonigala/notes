<!-- Mon, Sep 21, 2026 | sources: slides + code + YouTube auto-transcript + textbook -->
# Lecture 11: Subtype Polymorphism, Comparables, Comparators

## Overview

This lecture contrasts two philosophies for making code flexible: **polymorphism** (one interface, many underlying forms) and **function passing** (handing a function to another function as data). Python uses both freely: it overloads operators via magic methods like `__gt__`, and it accepts key functions in higher-order functions like `max`. Idiomatic Java, by contrast, leans almost entirely on **subtype polymorphism**: a supertype (usually an interface) declares a capability, a subtype overrides the abstract method, and Java picks the right implementation at runtime based on the actual object. We apply this to the problem of comparing objects. `Collections.max(dogs)` fails to compile on a plain `Dog` class because Java has no idea how to order dogs, so we declare `Dog implements Comparable<Dog>` and override `compareTo`, giving dogs a **natural order** (by size). When we want an *alternate* order (by name), Java does not let us pass a key function; instead we package the comparison logic inside a `Comparator<Dog>` object (e.g. `NameComparator`) and hand that object to `Collections.max(dogs, comparator)`. Along the way the lecture digs into what it means for a specific *file* to fail to compile, which is a lesson about the scope of compile-time checking. Note: the instructor stated explicitly that this lecture is **not in scope for Midterm 1**, but the ideas matter later for TreeMaps, TreeSets, and priority queues.

---

## Key Concepts

### 1. Code readability (opening announcements, not really about polymorphism)

Before the main content, the lecture reviewed Project 1 and Project 2 code style.

**Name your intermediate values.** Compare a student `removeFirst` with the instructor's:

```java
// Student version
public T removeFirst() {
    if (!isEmpty()) {
        T replace = sentinel.next.item;
        sentinel.next = sentinel.next.next;
        sentinel.next.prev = sentinel;
        size--;
        return replace;
    }
    return null;
}

// Instructor version
public T removeFirst() {
    if (size == 0) {
        return null;
    }

    Node oldFront = sentinel.next;
    Node newFront = sentinel.next.next;

    sentinel.next = newFront;
    newFront.prev = sentinel;

    size -= 1;
    return oldFront.value;
}
```

The point: creating variables **purely to give names to things** makes the code narrative and obvious. You can *see* which node is being replaced. The worry that extra local variables cost performance is unfounded: the Java just-in-time compiler (covered around lecture 37) optimizes away unneeded temporaries, so the penalty is negligible or non-existent.

**Layer abstractions for yourself.** Project 2 student code often repeats non-obvious modulus math:

```java
T removed = items[(nextLast - 1 + items.length) % items.length];
items[(nextLast - 1 + items.length) % items.length] = null;
```

Instead, hide the arithmetic behind a helper:

```java
private int wrapIndex(int index) {
    return (index + items.length) % items.length;
}
```

```java
T removed = items[wrapIndex(nextLast - 1)];
items[wrapIndex(nextLast - 1)] = null;
```

Now when reading, you only think "the item just before `nextLast`, wrapping around," not "what does this modulus do again?"

**Style will be graded** starting a bit after the midterm: consistent spacing, camelCase variable naming.

### 2. Polymorphism, informally

Wikipedia's definition, quoted on the slides: *"the ability in programming to present the same programming interface for differing underlying forms."*

The instructor mapped this onto the Python example directly:
- The **interface** is the `>` symbol.
- The **differing underlying forms** are `Dog`, `String`, `int`, and so on. All of them can sit on either side of `>`.

The instructor was candid that "polymorphism" is a slippery word that you learn by seeing many examples, and that "subtype polymorphism" is jargon you will likely forget by the end of the semester. The *idea* is what matters.

### 3. Python's approach #1: operator overloading

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

When the Python interpreter sees `>`, it looks for a `__gt__` method on the left operand. `>` is a privileged, magic, built-in concept in Python: every class *may* overload it, and code like a generic `get_the_max` will just work on anything that defines `__gt__`:

```python
def get_the_max(x):
    max_value = x[0]
    for item in x:
        if item > max_value:
            max_value = item
    return max_value

max_value = get_the_max([1, 2, 3, 4, 5])          # works on ints
max_dog   = get_the_max(list_of_dogs)             # works on Dogs, via __gt__
```

This is **duck typing**: you never declare that `>` is available on `Dog`. Either it works at runtime or it does not.

### 4. Python's approach #2: function passing

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

Or with the builtin and a lambda:

```python
max(doglist, key=name_len)
max(doglist, key=lambda d: len(d.name))
```

Here the *function itself* is an argument. `max` is a higher-order function. This is the natural Python move when you want an order other than the default.

### 5. Java has no operator overloading

You cannot teach Java's `>` to work on `Dog`. As the demo code comments put it: *"operator overloading in Java does not exist."* So the Python trick of defining `__gt__` has no Java analogue. Something else must carry the capability.

### 6. Capabilities in Java come from interfaces

The lecture's central question: **"How do we specify that a class has a certain capability in Java?"** Answer: **we implement some interface.**

This is the same pattern already seen in the course: `SLList implements List61B`, a `WizardIterator implements Iterator`. Now: `Dog implements Comparable<Dog>`. The relationship is "is-a": a `Dog` **is-a** `Comparable<Dog>`, meaning "I can be compared to other Dogs."

### 7. `Comparable<T>`

Since almost all of Java is written in Java, you can just read `Comparable.java` in the JDK source. The file is about 142 lines, nearly all documentation (the instructor's aside: heavy documentation is normal and important for library code). Stripped down, it is one method:

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

The contract is what matters:
- **negative** if `this` is less than `o`
- **zero** if they are equal (in ordering)
- **positive** if `this` is greater than `o`

It returns an `int`, not a `boolean`. A student asked why. The answer: one `int` encodes all three outcomes at once. Python needs `__gt__`, `__lt__`, `__le__`, `__ge__`, and so on; Java needs one method. The caller can ask whichever question it wants by testing the sign.

The contract deliberately says *any* negative or *any* positive number, not exactly `-1`/`+1`. That looseness is what makes the short subtraction idiom legal.

**Note on the type parameter.** In `public interface Comparable<T>`, the `<T>` is *declaring* a type parameter. In `public class Dog implements Comparable<Dog>`, the `<Dog>` is *filling it in*. You are not declaring a generic there; you are choosing a specific type argument. So inside `Dog`, the method signature becomes `public int compareTo(Dog o)`, not `compareTo(T o)`.

### 8. `compareTo` vs `equals`: why one takes `Dog` and the other takes `Object`

A student noticed the asymmetry: `equals` is declared `equals(Object o)` but `compareTo` here is `compareTo(Dog o)`. The instructor's explanation: `equals` gets called between wildly different types by all kinds of library code, so it must be able to handle an arbitrary object and answer "no." Comparison, by convention, is between a type and itself: it is usually meaningless to ask whether a `Dog` is "less than" a `String`. You *could* write `implements Comparable<Object>` if you genuinely wanted to compare against anything, but that is not idiomatic.

### 9. Subtype polymorphism, named

The flavor of polymorphism used here is **subtype polymorphism**:
- A **supertype** (`Comparable`) specifies the capability.
- A **subtype** (`Dog`) overrides the supertype's abstract method.
- **Java decides at runtime** which implementation to invoke, based on the dynamic type of the object doing the invoking.

This is genuinely different from Python's operator overloading. In Python, `>` is a single universal, privileged operator baked into the language, overloadable by anything. In Java there is no privileged comparison operator; there is just an ordinary interface named `Comparable` that happens to be the convention. Nothing stops you from making your own interface for some other capability, `IsACousin` or whatever, and the mechanism is identical.

### 10. Natural order, and when it is not enough

**Natural order**: the ordering implied by a class's `compareTo`. For our `Dog`, natural order is by size:

```
Pelusa (5)  <  Grigometh (200)  <  Clifford (9000)
```

But you often want a different order, e.g. alphabetical by name:

```
Clifford  <  Grigometh  <  Pelusa
```

(Coincidentally exactly the reverse here.) The classic real-world example given: sorting a spreadsheet by name instead of by number. A class gets **only one** natural order, so alternate orders need a different mechanism.

### 11. `Comparator<T>`

Python would pass a key function. Java packages the comparison logic **inside an object**:

```java
public interface Comparator<T> {
    int compare(T o1, T o2);
    ...
}
```

Two key differences from `Comparable`:
1. `compare` takes **two** arguments, both external objects. `compareTo` takes one, comparing against `this`.
2. `Dog` does **not** implement `Comparator<Dog>`. Instead, a separate class implements it, and `Dog` (or anyone) can hand out instances of it.

`Comparator` is said to compare **extrinsically** (from the outside), while `Comparable` compares **intrinsically** (the object compares itself). You can have many `Comparator<Dog>` classes at once: `NameComparator`, `SpeedComparator`, `SizeComparator`.

The real `Comparator` interface has a lot of default methods; the lecture explicitly skips them.

### 12. Comparable/Comparator vs Iterable/Iterator

The names look parallel in English ("-able" and "-or") but the relationships are **not** analogous:

- `Comparable`: I can be compared to another object. `int compareTo(T other)`
- `Comparator`: I can tell you how to compare two objects. `int compare(T x1, T x2)`
- `Iterable`: I can **give you** an iterator. `Iterator<T> iterator()`
- `Iterator`: I can feed you objects. `boolean hasNext()`, `T next()`

An `Iterable` *produces* an `Iterator`. A `Comparable` does **not** produce a `Comparator`; it gives you the comparison answer directly.

---

## Definitions

**Polymorphism**: "The ability in programming to present the same programming interface for differing underlying forms" (Wikipedia, as quoted in lecture).

**Operator overloading**: Defining what a built-in operator (like `>`) means for your own type. A form of polymorphism. Available in Python (via magic methods like `__gt__`); **not available in Java**.

**Function passing**: Supplying a function as an argument to another function, e.g. Python's `max(doglist, key=name_len)`. Java has this (lambdas, streams) but it is not taught in CS 61B and is not idiomatic for comparison.

**Higher order function**: A function that takes a function as an argument (or returns one), e.g. Python's `max` with a `key` parameter. (extra context: this is the standard term for what `get_the_max(x, key)` is doing; the lecture demonstrated the idea without dwelling on the phrase.)

**Duck typing**: Python's approach in which you do not declare whether a capability like `>` is available; you just use it and find out at runtime.

**Subtype polymorphism**: The pattern where a supertype specifies a capability via an abstract method, a subtype overrides it, and Java selects the implementation at runtime based on the type of the invoking object.

**`Comparable<T>`**: Java interface with the single abstract method `int compareTo(T o)`, returning a negative integer, zero, or a positive integer as `this` is less than, equal to, or greater than `o`. Implementing it declares that your class has a natural order.

**`compareTo(T o)`**: The `Comparable` method. `this` is the object being compared; `o` is the other object.

**Natural order**: The ordering implied by a `Comparable`'s `compareTo` method. Exactly one per class.

**`Comparator<T>`**: Java interface whose core abstract method is `int compare(T o1, T o2)`. Used to define an ordering **extrinsically**, from outside the class being ordered. A class may have arbitrarily many associated `Comparator`s.

**`compare(T o1, T o2)`**: The `Comparator` method. Both arguments are external objects; returns negative/zero/positive under the same sign convention as `compareTo`.

**Intrinsic vs extrinsic comparison**: Intrinsic = the object compares itself to another (`Comparable`). Extrinsic = a separate object supplies the comparison rule (`Comparator`).

**Static nested class**: A nested class declared `static`, meaning it can be instantiated without an enclosing instance of the outer class. Needed for `NameComparator` inside `Dog`, since a `NameComparator` should not require a particular `Dog` to exist.

**`Collections.max(collection)`**: Returns the maximum element according to the elements' natural order. Requires the element type to be `Comparable`.

**`Collections.max(collection, comparator)`**: Returns the maximum element according to the given `Comparator`.

---

## Worked Examples

### Example 1: The failure that motivates everything

Start with the bare `Dog` class from lecture:

```java
package lec11_inheritance3;

public class Dog {
    public String name;
    public int size;

    public Dog(String n, int s) {
        name = n;
        size = s;
    }
}
```

And the demo:

```java
package lec11_inheritance3;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class CollectionsDogDemo {
    public static void main(String[] args) {
        List<Dog> dogs = new ArrayList<>();

        dogs.add(new Dog("Grigometh", 200));
        dogs.add(new Dog("Pelusa", 5));
        dogs.add(new Dog("Clifford", 9000));

        // operator overloading in Java does not exist

        Dog maxDog = Collections.max(dogs);
    }
}
```

**What happens:** this does not compile. The error message is, in the instructor's words, "incomprehensible": IntelliJ complains that `max` expects something like `Collection<? extends T>` where `T` is comparable, and you gave it `List<Dog>`.

**Why:** `Collections.max` must, at some point, ask "is element A bigger than element B?" Its only vocabulary for that is `compareTo`. A plain `Dog` has no `compareTo`, and more importantly has not *declared* that it can be compared. The compiler catches this before the program ever runs.

**Box-and-pointer / environment reasoning in words:** `dogs` is a variable holding a reference to an `ArrayList` object. That `ArrayList` holds an internal array of three references, pointing to three separate `Dog` objects on the heap. Each `Dog` object has two boxes inside it: a `name` box holding a reference to a `String` object, and a `size` box holding an `int` value directly (`200`, `5`, `9000`). Nothing in any of these boxes tells `Collections.max` which of the three to prefer. The missing information is not data, it is *behavior*, and in Java behavior arrives through a type's method table, which is why the fix is a type-level declaration.

### Example 2: Making `Dog` comparable

Two steps, as stated on the slides: add `implements Comparable<Dog>` to the class header, and override `compareTo`.

First, the verbose version written live in lecture:

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

**Reading it:** `this` is me, the dog on which `compareTo` was called. `uddaDog` ("the other dog") is the argument. `size` with no qualifier means `this.size`. If I am bigger, return a positive number; if smaller, negative; otherwise zero. That is exactly the interface contract.

A student proposed comparing with `equals`. The instructor's response: `==` is fine and `equals` is wrong here, because `size` is a primitive `int`, and `equals` does not apply to primitives at all.

Then the cleaner version, described on the slides as "very common in Java":

```java
public class Dog implements Comparable<Dog> {
    ...
    @Override
    public int compareTo(Dog uddaDog) {
        return size - uddaDog.size;
    }
}
```

**Why this works:** if my size is 30 and theirs is 20, `30 - 20 = 10`, a positive number: correct, I am greater. If my size is 5 and theirs is 9000, `5 - 9000 = -8995`, a big negative number: correct, I am smaller. Equal sizes give 0. This idiom is *only* legal because the contract permits any negative and any positive value, not specifically `-1` and `+1`.

Now the original demo compiles and runs:

```java
Dog maxDog = Collections.max(dogs);   // maxDog points at Clifford
```

**Stepping through what `Collections.max` does:** it walks the collection keeping a running "best so far." Conceptually it starts with Grigometh (200) as the best, then reaches Pelusa and evaluates `pelusa.compareTo(grigometh)`, which is `5 - 200 = -195`, negative, so Pelusa is smaller and the best stays Grigometh. Then it reaches Clifford: `clifford.compareTo(grigometh)` is `9000 - 200 = 8800`, positive, so the best becomes Clifford. After the loop, the `maxDog` variable holds a reference to the very same `Dog` object that is also referenced from inside the `ArrayList`: one object, two arrows pointing at it, no copy made. In the debugger the instructor showed exactly this: `maxDog` pointing at Clifford.

**The polymorphism payoff:** `Collections.max` was compiled years ago at Oracle, long before `Dog` existed. It contains a call like `items[i].compareTo(items[maxDex])`. At runtime, when that call fires on a `Dog` object, Java dispatches to `Dog`'s `compareTo`. The library code did not need to know anything about dogs. That is subtype polymorphism.

### Example 3: Compilation Error Puzzle #1 (`hugcode.com/piano`)

Set up four files:

```java
public class DogLauncher {                        public class Dog
  public static void main(String[] args) {          implements Comparable<Dog> {
    ...                                               ...
    Dog[] dogs = new Dog[]{d1, d2, d3};               public int compareTo(Dog o) {
    System.out.println(Collections.max(dogs));          return this.size - o.size;
  }                                                   }
}                                                     ...
                                                  }

public interface Collections {                    public interface Comparable<T> {
    // the max function                               public int compareTo(T o);
    ...                                           }
    int cmp = items[i].compareTo(items[maxDex]);
    ...
}
```

(These `Collections`/`Comparable` sketches are the lecture's simplified stand-ins for the real library code.)

**Q: If we omit `compareTo()` from `Dog`, which file fails to compile?**
A. DogLauncher.java  B. Dog.java  C. Collections.java  D. Comparable.java

**Answer: B (Dog.java).**

**Reasoning, step by step:**

1. **Rule out C and D immediately.** `Collections` and `Comparable` are part of Java. They were written and compiled at Oracle *before your code existed*. "Fails to compile" means "there is something wrong inside that piece of code." Nothing you write in `Dog.java`, no matter how broken, can retroactively make `Comparable.java` ill-formed. Compilation of each file asks: is this code good, do the types check, is the syntax right? Thumbs up or thumbs down, per file.

2. **Do not reach for a runtime mental model.** A tempting but wrong story is: "the program runs, `Collections.max` tries to call `compareTo`, and boom, the method is not there." In Java that is impossible. The entire point of compile-time checking is to catch this before anything runs.

3. **So what is wrong with `Dog`?** `Dog` declares `implements Comparable<Dog>` but does not supply all the abstract methods that interface requires. That is a broken promise, and the compiler rejects it. The instructor's analogy: "you're claiming to be in BTS, but you cannot dance."

4. **The pedantic addendum:** `DogLauncher` will also fail, since `Dog.class` never gets produced. But the *source* of the problem, the file with something actually wrong in it, is `Dog.java`. B is the best answer.

### Example 4: Compilation Error Puzzle #2 (`hugcode.com/tiger`)

Same four files. **Q: If we omit `implements Comparable<Dog>` (keeping the `compareTo` method), which file fails to compile?**

**Answer: A (DogLauncher.java)**, with an important caveat about `@Override`.

**Reasoning:**

1. **C and D are out for the same reason as before.**

2. **Is `Dog` itself broken?** Consider `Dog` in isolation: a class with fields, a constructor, and a method `public int compareTo(Dog o)`. Is that legal Java? **Yes.** It is perfectly fine to have a method named `compareTo` without implementing `Comparable`. The instructor's version of the analogy: "you can dance even if you're not in BTS." So `Dog` compiles.

3. **The `@Override` caveat.** If the `compareTo` method still carries an `@Override` annotation while the class no longer implements anything that declares `compareTo`, then `Dog` **does** fail to compile: you claimed to override something and you are not overriding anything. So B is also a defensible answer *if* you were accounting for `@Override`. This is why the lecture called B "correct if you were thinking about the override tag."

4. **What breaks is `DogLauncher`.** It calls `Collections.max(dogs)` where `dogs` is a `Dog[]`. `Collections.max` demands `Comparable` elements. `Dog` is not declared `Comparable`, so the call is a type error. And yes, `Dog` visibly *has* a `compareTo` method sitting right there, but the Java compiler is strict: **you have to declare that you are `Comparable`, otherwise you don't count.** Java does not do structural/duck typing on interfaces.

**Why this puzzle matters:** it clarifies the *scope* of the compiler. Compilation is a per-file verdict on whether that code obeys the language rules, given the declared types of everything it touches. It happens before execution, and library code is immune to your mistakes.

### Example 5: A `Comparator` for name order

Java's interface:

```java
public interface Comparator<T> {
    int compare(T o1, T o2);
    ...
}
```

Our implementation. Note the `static` keyword and why it must be there:

```java
import java.util.Comparator;

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

    // Must be static: a NameComparator can be instantiated
    // without any actual Dog being involved.
    public static class NameComparator implements Comparator<Dog> {
        @Override
        public int compare(Dog a, Dog b) {
            return a.name.compareTo(b.name);
        }
    }
}
```

**Why the body is just one line.** We cannot write `if (o1.name < o2.name)`: relational operators do not work on `String` in Java. But `String` itself `implements Comparable<String>`, so it already has a `compareTo`. We delegate: ask dog A's name to compare itself to dog B's name. The instructor called this "the laziest thing we could possibly do," approvingly. This is subtype polymorphism used twice in one line: our `Comparator` implements an interface, and inside it we invoke `String`'s implementation of another interface.

**Why `static` is required.** This was flagged in lecture as "the very edge of any Java syntax I want to talk about." A non-static nested class (like `Node` inside an `SLList`) is tied to a specific instance of its enclosing class: every `Node` belongs to some particular `SLList`. But a `NameComparator` is not *about* any one dog; it is a free-floating rule for comparing any two dogs. Marking it `static` means it can be instantiated without an enclosing `Dog` object, which is what we need.

Using it:

```java
List<Dog> dogs = new ArrayList<>();
dogs.add(new Dog("Grigometh", 200));
dogs.add(new Dog("Pelusa", 5));
dogs.add(new Dog("Clifford", 9000));

Dog maxNameDog = Collections.max(dogs, new Dog.NameComparator());
```

**What flows where:** the second argument is an **object** of type `Comparator<Dog>`, not a function. `Collections.max` receives a reference to a freshly allocated `NameComparator` object (which, note, has no instance fields at all: it exists purely to carry a method). Inside `max`, whenever a comparison is needed, it calls `comparator.compare(x, y)`, and dynamic dispatch routes that to *our* `compare`. The maximum by name here is Pelusa, since alphabetically Clifford < Grigometh < Pelusa.

**Side-by-side with Python:**

```python
def name_len(dog):
    return len(dog.name)

dogs = [Dog("Grigometh", 10), Dog("Pelusa", 5), Dog("Clifford", 9000)]
max(dogs, key=name_len)
```

Python passes the function directly. Java wraps the comparison logic inside an object and relies on subtype polymorphism. Same goal, opposite philosophy. (The Python version uses name *length* rather than name itself, so it is an analogy, not an exact translation.)

### Example 6: From a getter method to a static constant

In lecture the instructor first built an accessor:

```java
public static Comparator<Dog> getAComparator() {
    return new NameComparator();
}
```

```java
Dog maxNameDog = Collections.max(dogs, Dog.getAComparator());
```

This works but is not the common idiom, and calling `Collections.max(dogs, new Dog.NameComparator())` directly was described as "awkward and aesthetically unpleasant."

The idiomatic fix: a pre-instantiated static constant, named in ALL_CAPS per the usual convention for global constants:

```java
public static NameComparator NAME_COMPARATOR = new NameComparator();
```

```java
Dog maxNameDog = Collections.max(dogs, Dog.NAME_COMPARATOR);
```

**Why this is nicer:** the `Dog` class now has a comparator "glued to it" that anyone can grab at any time. When the `Dog` class is loaded, one `NameComparator` object is created, and exactly one ever needs to exist, since it is stateless. The instructor described this as "the most common way you'll see in somewhat older school but idiomatic Java code."

### Example 7: The Comparator quiz (`hugcode.com/lemon`)

```java
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

Options: A. `+1`  B. Positive Number  C. `-1`  D. Negative Number  E. Zero

**Answer: D, Negative Number.**

**Trace:** `nc` holds a reference to a `NameComparator` object. `nc.compare(a, b)` dispatches to our `compare`. Inside, `a.name` is the `String` `"Frank"` and `b.name` is `"Zeke"`. So the whole expression reduces to exactly:

```java
System.out.println("Frank".compareTo("Zeke"));
```

"Frank" is alphabetically before "Zeke", so by the `Comparable` contract this must be negative. **Can we say it is `-1`?** No. The contract promises only the *sign*. We do not know how `String.compareTo` is implemented internally. When the instructor actually ran it, the output was **`-20`** (note: `'F'` is 70 and `'Z'` is 90, and 70 - 90 = -20, so it is consistent with a character-difference implementation; the instructor said the precise mechanism is below our level of abstraction). The class was split roughly evenly between C and D, which is exactly the trap: **`-1` is a plausible-looking wrong answer; "some negative number" is the right one.**

### Example 8 (bonus, explicitly not examinable): lambdas

```java
Comparator<Dog> dc = (d1, d2) -> d1.name.compareTo(d2.name);
Dog maxNameDog = Collections.max(dogs, dc);
```

The lambda defines *and* instantiates a `Comparator` object in one line. The slides state: "We will not teach Java lambdas in 61B, nor will you be expected to learn them." Java also has streams and other function-passing machinery; the instructor mentioned these exist and invited curious students to explore, but they are out of scope.

---

## Common Pitfalls

1. **Thinking you can overload `>` in Java.** You cannot. There is no operator overloading in Java, full stop. If you want ordering behavior, you implement an interface.

2. **Writing `compareTo` but forgetting `implements Comparable<Dog>`.** The method existing is not enough. Java requires the *declaration*. Puzzle #2 is built entirely around this: the class compiles, but every caller that expects a `Comparable` rejects it.

3. **Writing `implements Comparable<Dog>` but forgetting `compareTo`.** The mirror-image error, and this one breaks `Dog.java` itself. You promised a capability and did not deliver.

4. **Assuming `compareTo` returns exactly `-1`, `0`, or `+1`.** The contract is only about the **sign**. `"Frank".compareTo("Zeke")` returned `-20`. Never write `if (x.compareTo(y) == -1)`; write `if (x.compareTo(y) < 0)`.

5. **Believing the compiler catches missing methods at runtime.** It does not; that is the whole point. "Fails to compile" is a per-file, pre-execution verdict.

6. **Believing your code can break `Collections.java` or `Comparable.java`.** Those were compiled before your code existed. They are always fine.

7. **Leaving `@Override` on a method that no longer overrides anything.** This is itself a compile error, and it is the reason Puzzle #2 has a subtle second answer.

8. **Forgetting `static` on a nested `Comparator` class.** A non-static nested class requires an enclosing instance. `new Dog.NameComparator()` will not work without `static`, since a comparator is not associated with any particular dog.

9. **Confusing `compareTo` and `compare` arities.** `compareTo(T other)` takes **one** argument (the other is `this`). `compare(T o1, T o2)` takes **two**.

10. **Assuming `Comparable` gives you a `Comparator`, by analogy with `Iterable` giving you an `Iterator`.** The names rhyme; the relationships do not match. A `Comparable` answers comparisons directly; it does not hand out comparator objects.

11. **Using `equals` or `==` carelessly inside `compareTo`.** In the lecture's `Dog`, `size` is a primitive `int`, so `equals` does not apply to it at all. Also, mixing an `equals` check into a size-only comparison muddies the semantics.

12. **Trying `o1.name < o2.name` in a name comparator.** Relational operators do not work on `String` in Java. Use `String`'s own `compareTo`.

13. **Repeating opaque expressions instead of naming them.** From the opening announcements: repeated modulus math, unnamed intermediate nodes. Naming things costs essentially nothing at runtime and buys a lot of readability.

---

## Likely Exam Points

> Reminder: the instructor stated this lecture is **not in scope for Midterm 1**. These points are for later exams and for the TreeMap/TreeSet/priority-queue material where comparison machinery reappears.

**1. Which file fails to compile when `compareTo` is missing?**

*Q:* `Dog implements Comparable<Dog>` but has no `compareTo` method. `DogLauncher` calls `Collections.max(dogs)`. Which file fails to compile, and why?

*A:* `Dog.java`. It declares that it implements `Comparable<Dog>` but does not supply the required abstract method, so the class itself is ill-formed. `DogLauncher.java` will also fail downstream, since `Dog.class` is never produced, but `Dog.java` is the file with the actual defect. `Collections.java` and `Comparable.java` are part of Java, compiled long before your code existed, and nothing you write can break them.

**2. Which file fails to compile when `implements Comparable<Dog>` is missing?**

*Q:* `Dog` has a method `public int compareTo(Dog o)` but does **not** declare `implements Comparable<Dog>`. Which file fails?

*A:* `DogLauncher.java`, because it passes non-`Comparable` objects to `Collections.max`, which requires `Comparable` elements. `Dog.java` on its own is legal: a class may have a method named `compareTo` without implementing any interface. **Caveat:** if the method still has `@Override` on it, then `Dog.java` also fails, because there is nothing being overridden.

**3. What does a `compareTo`/`compare` call return?**

*Q:* Given `NameComparator` as defined in lecture, what does `nc.compare(new Dog("Frank", 1), new Dog("Zeke", 1))` print?

*A:* Some **negative number** (the actual observed value was `-20`). Not necessarily `-1`. The `Comparable` contract only specifies the sign: negative if the first is less, zero if equal, positive if greater.

**4. Write a `compareTo` obeying the contract.**

*Q:* Give a one-line `compareTo` for `Dog` ordering by `size`, and explain why it is valid.

*A:*
```java
@Override
public int compareTo(Dog uddaDog) {
    return size - uddaDog.size;
}
```
Valid because the contract accepts *any* negative/positive value, not just `-1`/`+1`. Bigger `this` gives a positive difference, smaller gives negative, equal gives zero.

**5. Write a `Comparator`.**

*Q:* Add a `SizeComparator` to `Dog` that orders dogs by size, plus an idiomatic way to access it.

*A:*
```java
public static class SizeComparator implements Comparator<Dog> {
    @Override
    public int compare(Dog a, Dog b) {
        return a.size - b.size;
    }
}

public static SizeComparator SIZE_COMPARATOR = new SizeComparator();
```
Used as `Collections.max(dogs, Dog.SIZE_COMPARATOR);`. The class must be `static` so it can be instantiated without an enclosing `Dog` instance, and the ALL_CAPS static field is the conventional naming for such a constant.

**6. `Comparable` vs `Comparator`: state the difference.**

*Q:* In one sentence each, distinguish `Comparable` from `Comparator`, and say how many of each a class can have.

*A:* `Comparable` means "I can compare myself to another object of my type" via `int compareTo(T other)`; it defines the class's single natural order, so there is exactly one. `Comparator` means "I can tell you how to compare two objects" via `int compare(T x1, T x2)`; it is extrinsic, defined in a separate class, and a type may have arbitrarily many (name, size, speed, etc.).

**7. Comparable/Comparator vs Iterable/Iterator.**

*Q:* Is the relationship between `Comparable` and `Comparator` the same as between `Iterable` and `Iterator`? Explain.

*A:* No. An `Iterable` **produces** an `Iterator` (`Iterator<T> iterator()`), and the `Iterator` then feeds out objects (`hasNext`, `next`). A `Comparable` does **not** produce a `Comparator`; it answers comparison queries directly. The English parallel in the names is misleading.

**8. Python vs Java philosophy.**

*Q:* How does Python let you customize how `max` picks its answer, and how does Java do the equivalent?

*A:* Python has two options: overload `__gt__` so the universal `>` operator works on your type (operator overloading), or pass a `key` function to `max` (function passing). Java has no operator overloading, and idiomatic Java avoids explicit function passing; instead it uses **subtype polymorphism**: implement `Comparable` for the natural order, or package an alternate order inside a `Comparator` object and pass that object to `Collections.max`.

**9. Define subtype polymorphism.**

*Q:* What are the three ingredients of subtype polymorphism as described in lecture?

*A:* (i) A supertype (e.g. the `Comparable` interface) specifies a capability via an abstract method. (ii) A subtype (e.g. `Dog`) overrides that method. (iii) Java decides at **runtime**, based on the actual type of the invoking object, which implementation to execute.

**10. Why does `compareTo` return `int` rather than `boolean`?**

*Q:* Explain the design choice.

*A:* A single `int` encodes all three possible outcomes (less than, equal to, greater than) in one method. Python needs separate `__gt__`, `__lt__`, `__le__`, `__ge__` methods; Java gets by with one, and the caller tests the sign for whichever question it cares about.

**11. Why `compareTo(Dog)` but `equals(Object)`?**

*Q:* Explain the asymmetry.

*A:* `equals` is called by all kinds of library code between arbitrary, unrelated types, so it must accept any `Object` and be able to answer "not equal." Comparison is conventionally between a type and itself, since ordering unrelated types is usually meaningless. You *could* write `implements Comparable<Object>`, but it is not idiomatic.

---

## Summary

- **Polymorphism** = "the same programming interface for differing underlying forms." **Function passing** = handing a function to another function as an argument. Python uses both; idiomatic Java leans overwhelmingly on polymorphism.
- Python compares via **operator overloading** (`__gt__` backing the universal `>`) and via **key functions** (`max(doglist, key=name_len)`). Python is **duck typed**: no declaration required.
- **Java has no operator overloading.** To give a class a capability, you **implement an interface**.
- **`Comparable<T>`** has one method, `int compareTo(T o)`: negative / zero / positive as `this` is less than / equal to / greater than `o`. Implementing it defines the class's **natural order** (for `Dog`, size). There is exactly one natural order per class.
- The **sign** is all that is guaranteed, never a specific magnitude. This is why `return size - uddaDog.size;` is the common idiom, and why `"Frank".compareTo("Zeke")` returning `-20` is perfectly correct.
- Once `Dog implements Comparable<Dog>`, `Collections.max(dogs)` works. Library code written years before `Dog` existed calls `compareTo`, and Java dispatches to your implementation at **runtime**. That is **subtype polymorphism**: supertype specifies, subtype overrides, runtime dispatches.
- **Compilation is per-file and happens before execution.** A file "fails to compile" when something is wrong *inside it*. Your code can never break `Collections.java` or `Comparable.java`.
  - Missing `compareTo` with the `implements` clause present: **`Dog.java`** breaks (broken promise).
  - Missing `implements` with `compareTo` present: **`DogLauncher.java`** breaks (passing non-`Comparable`s); `Dog.java` is legal unless `@Override` remains, in which case it breaks too.
  - Java does not duck type interfaces: having the method is not enough, you must **declare** the interface.
- **`Comparator<T>`** has `int compare(T o1, T o2)` and defines an order **extrinsically**, from a separate class. A type may have many: `NameComparator`, `SizeComparator`, `SpeedComparator`.
- A nested `Comparator` class must be **`static`**, since it need not be tied to any instance of the enclosing class. Idiomatically, expose it as an ALL_CAPS static constant: `public static NameComparator NAME_COMPARATOR = new NameComparator();`, then `Collections.max(dogs, Dog.NAME_COMPARATOR)`.
- Implement name ordering by delegating to `String`'s own `compareTo`: `return a.name.compareTo(b.name);`. Relational operators do not work on `String`.
- **Comparable/Comparator is not analogous to Iterable/Iterator.** `Iterable` *gives you* an `Iterator`; `Comparable` does **not** give you a `Comparator`, it gives answers directly.
- Out of scope but mentioned: Java **lambdas** (`Comparator<Dog> dc = (d1, d2) -> d1.name.compareTo(d2.name);`) and streams. Not taught, not examined.
- Style takeaways from the announcements: name your intermediate values (the JIT compiler makes the cost negligible), hide repeated non-obvious arithmetic behind helpers like `wrapIndex`, and keep spacing and camelCase consistent. Style becomes mandatory a bit after the midterm.
- **This lecture is not in scope for Midterm 1**, but is important groundwork for TreeMaps, TreeSets, and priority queues later in the course.
