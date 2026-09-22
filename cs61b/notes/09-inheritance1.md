<!-- Wed, Sep 16, 2026 | sources: code + YouTube auto-transcript -->
# Lecture 9: Inheritance 1

## Overview

This lecture solves a concrete annoyance: after building `AList` and `SLList` in previous lectures, we have two classes that do nearly the same thing, and any utility method (like `WordUtils.longest`) has to be duplicated once per list class. The fix is to express, in Java, the same idea that natural languages express with hypernyms ("dog" covers poodle and malamute): define a general reference type `List61B` using the `interface` keyword, and declare `AList` and `SLList` as hyponyms of it with `implements`. Doing so gives us **interface inheritance**, where the interface specifies *what* a class can do (method signatures) without saying *how*, so a single `longest(List61B<String> list)` works on any list, including lists that have not been invented yet. The lecture then introduces `default` methods, which are **implementation inheritance** (the superclass dictates *how*, not just *what*), shows that subclasses may override a default (`SLList.print` overrides the slow default because `get(i)` on a linked list is expensive), covers the `@Override` annotation and the overload/override distinction, and closes with the idea of an **abstract data type** (ADT) plus a teaser question about implementing a stack.

---

## Key Concepts

### 1. The motivating problem: duplicated code

We built two list classes that are "almost redundant." Consider a library class `WordUtils` with a method that finds the longest string in a list:

```java
public static String longest(SLList<String> list) { ... }
```

If a user wants to call it on an `AList`, the only change needed is the parameter type on the first line. That's telling: the body is identical. So one "solution" is to write both versions:

```java
public static String longest(SLList<String> list) { ... }
public static String longest(AList<String>  list) { ... }   // identical body
```

Java allows this (it is **overloading**, see below), but the lecture drew out three problems from the class:

1. **Duplicated, virtually identical code** (Hug: "I just find it displeasing").
2. **It does not extend to future lists.** Invent an `XList` tomorrow and you must write a third copy.
3. **Harder to maintain.** Fix a bug or change the approach in one copy and you must remember to change every other copy. Code does not "wear out" like a bicycle, but you do keep wanting to change it later, and that is where maintenance cost shows up.

One side point Hug corrected on the spot: `AList` is not "built in" and `SLList` "ours"; we wrote both, so they are on even footing.

### 2. Hypernyms and hyponyms: the English analogy

Washing a poodle and washing a malamute have essentially the same seven steps (brush first, lukewarm water, calm voice, shampoo, rinse, air dry, reward). English does not publish one set of instructions per breed. Instead we say **dog**, because *dog* is a **hypernym** of poodle, malamute, yorkie, schnauzer. The inverse relation is **hyponym**: poodle is a hyponym of dog. The two are exact one-to-one opposites (hyper = above, hypo = below).

These relations form a hierarchy: malamute → dog → canine → carnivore → animal. The **WordNet** project built exactly such a graph for English nouns; it is largely defunct now, but we will see it again in Project 4.

> Note on transcript wording: the auto-captions garble this at one point ("list is a hyponym of s-list"). The correct reading given the definitions is: `List61B` is the **hypernym**, and `SLList` and `AList` are its **hyponyms**.

Goal for the day: express hypernym/hyponym relationships in Java.

### 3. Reference types

Java has exactly **eight primitive types** (int, long, double, short, etc.), and you cannot add new ones. **Everything else is a reference type**: `Dog`, `Cat`, `AList`, `SLList`, `List61B`, any type you make up. A type is, informally, the tag on a variable (equivalently, on a memory box).

Interfaces create new reference types. That is why you can write `List61B<String> x = ...;` even though `List61B` can never itself be instantiated.

### 4. The two-step recipe

**Step 1: define the reference type** with the `interface` keyword instead of `class`. In IntelliJ, "New Java Class → Interface" simply writes `interface` instead of `class` in the generated file.

**Step 2: declare the hyponyms** with `implements`:

```java
public class SLList<Blorp> implements List61B<Blorp> { ... }
```

Building `List61B` was done live by copying `SLList` and **stripping out everything implementation-specific**:

- The private `Node` class: **deleted**. Some lists have no nodes (an `AList` certainly doesn't).
- **Constructors: not allowed in an interface.** As the code comment records: "constructors have no interface / they are abstract and you can't instantiate them directly."
- `getLastNode()`: **deleted**. It is a private detail of a linked-list implementation, and returns a `Node`, which only linked lists have.
- Every remaining method body: **deleted and replaced with a semicolon**.
- Instance variables (`sentinel`, `size` the field): gone. An interface has no state to speak of. (Hug hedged live about whether interfaces can hold any variables; the 61B-relevant point is you cannot put ordinary instance variables in one, so a default method can only use the interface's own methods.)

The result is the "papier-mâché sculpture of a dog": it has legs and a shape, but you cannot tell what the nose looks like.

Choosing what goes in the interface is a design responsibility. When a student pointed out that linked lists and array lists behave differently, Hug's answer was: it is the interface designer's job to find the **common abstractions everyone can rely on**. That is precisely why `getLastNode` is not in `List61B`.

### 5. The interface is a contract the compiler enforces

Live demo: adding `implements List61B<T>` to `AList` produced a red squiggle, because the in-class `AList` lacked an `insert` method. The compiler's complaint is "you claim to be a `List61B`, but you cannot do everything a `List61B` is supposed to do." Hug's analogy: claiming to be in the Backstreet Boys without the ability to spin, dance, and so forth. People (and compilers) see right through you.

(In the posted lecture code, `AList` *does* have `insert`, so it compiles. Hug said he would add the missing methods when pushing the repo, and indeed the `AList.java` above has `insert`, `addFirst`, `getFirst`, and `get`.)

### 6. Generic type parameters are per-file placeholders

`List61B<Glatch>`, `SLList<Blorp>`, `AList<Item>`: the names do **not** have to match. Each is just a placeholder local to its own file. `SLList<Blorp> implements List61B<Blorp>` reads "whatever `Blorp` is for me, that is what I plug into `List61B`." Renaming `Glatch` to anything else in `List61B.java` changes nothing elsewhere.

### 7. Overloading vs overriding

- **Overloading:** two or more methods with the **same name but different signatures** (different types and/or numbers of parameters) in the same class. The two `longest` methods were overloads. `Math.abs(int)` and `Math.abs(double)` are overloads. Mnemonic from lecture: "a whole bunch of different things that all look kind of the same."
- **Overriding:** a subclass (hyponym) defines a method with **the same signature** as the superclass/interface. Mnemonic: "my boss has one idea and I have another."

Lecture's example pair:

```java
interface Animal { void makeNoise(); }

class Pig implements Animal {
    public void makeNoise() { ... }      // OVERRIDES Animal's makeNoise
}

class Dog implements Animal {
    public void makeNoise(Dog x) { ... } // OVERLOADS: signature does not match
}
```

The rule Hug wants you to leave with: **exact signature match with the "boss" → overriding; different signature → overloading.**

Reassuring meta-note: TAs used to love writing devious overload/override puzzles, and Hug has **banned** them. Know the distinction, do not get bogged down.

### 8. The `@Override` annotation

`@Override` is **optional** and **changes nothing at runtime**. If you override without writing it, you are still overriding. So why use it?

1. **Typo protection.** Write `@Override public void addLats(...)` and you get a compile error instead of a silently dead method. This is the real payoff: if you meant to replace a slow `default` implementation and misspell the name, without `@Override` your "fast" method is never called and the slow default runs in production.
2. **A reminder to the reader**: this method "isn't just yours, it's somebody else's and you're overriding them."

Where **not** to put it: private helper methods (`getLastNode`), constructors, and any method not present in the supertype. Live, Hug added `@Override` to a method the interface did not declare and the compiler rejected it. Course guidance: **in 61B, annotate every override.**

### 9. Interface inheritance

> **Interface inheritance**: the subclass inherits the *interface* (the list of method signatures) from the superclass, specifying **what** the subclass can do but **not how**.

Why it is useful:

- You can write general code like `WordUtils.longest` that works on `SLList`, `AList`, and any list not yet invented.
- Subclasses **must** implement all the methods, or the code does not compile (the `AList` demo).

Hierarchies are allowed: you could define `Collection61B` above `List61B`, and a class may implement several interfaces at once (a student asked about being both a `List61B` and an `Iterable`, or some `StuffHolder`; both are fine in Java).

Terminology caveat from lecture: Hug will often say "superclass" and "subclass" even when the top thing is an interface. Technically you might say "super-interface," but everyone is lazy about this.

### 10. Assigning a subtype into a supertype box

A memory box of type `List61B` can only hold **addresses of `List61B`s**. So this might look worrisome:

```java
List61B<String> someList = new SLList<>();
WordUtils.longest(someAList);   // passing an AList into a List61B parameter
```

Java says: fine, because an `SLList` **is a** `List61B` (and likewise an `AList`). And calling `someList.addFirst(...)` through that box works, because `addFirst` is in the `List61B` interface. Hug likes writing `List<X> x = new ArrayList<>()` for exactly this reason: it makes you think about the list abstractly. (He also notes it would have been perfectly fine to just write `SLList` on the left.)

### 11. Implementation inheritance and `default` methods

Twelve years ago you could not put code in an interface. Now you can, via `default` methods:

```java
default public void print() { ... }
```

Contrast:
- `public void print();` → every list **must** write its own `print`.
- `default public void print() {...}` → every list **gets** this `print` for free, unless it overrides.

This is **implementation inheritance**: the boss tells you not just *what* to do but *how*. Hug's running analogy: Chipotle trains you to put nine beans on a burrito. You *can* stick it to the man and put thirty, but the default is nine.

Writing the default body is constrained in an instructive way: **you have no instance variables to work with, only the interface's own methods.** So the default `print` is built from `size()` and `get(i)`:

```java
default public void print() {
    for (int i = 0; i < size(); i += 1) {
        IO.print(get(i) + ", ");
    }
    IO.println();
}
```

Note `size()` and `get(i)` have no bodies yet when you write this; once somebody implements the interface, they will exist. Also note there is **no `this.`** required (unlike Python's explicit `self`); `this.get(i)` would be legal but is unnecessary.

Yes, this prints a trailing comma. Hug: "that's okay."

### 12. Why the default `print` is bad for `SLList` (the attendance-adjacent question)

Poll answer, 64% correct: **fast for `AList`, slow for `SLList`.**

`AList.get(i)` is `items[i]`, a direct array index. `SLList.get(i)` must walk from the sentinel. So the default `print` on an `SLList` walks 0 nodes, then 1, then 2, ..., re-traversing from the front every single time. Calling `get(50)` walks the whole front of the list, then `get(51)` walks it again. (Hug's image: those shuttle-run exercises where each rep goes farther and you keep coming back.)

So `SLList` overrides it, keeping a pointer that advances once per item:

```java
/** RAGE AGAINST THE CHIPOTLE MACHINE */
@Override
public void print() {
    Node p = sentinel.next;
    while (p != null) {
        IO.print(p.item + ", ");
        p = p.next;
    }
    IO.println();
}
```

A student suggested recursion; Hug agreed it would work but chose iteration as less confusing live. He also noted the slides version uses a `for` loop, which is the same code, and that people sometimes panic seeing a `for` loop there as if it were forbidden. It isn't; you can write whatever you want inside your override.

### 13. Which `print` actually runs?

```java
List61B<String> someList = new SLList<>();
someList.addFirst("elk");
someList.print();
```

Even though the **variable type** is `List61B` (whose default `print` is the slow one), the **efficient overridden `SLList.print` is what runs**. Hug's phrasing: "whenever you override a method, in almost all normal circumstances, the actual overridden method will get called." This is the behavior you want: otherwise you would have to worry about your code's performance changing based on which type you declared the variable as.

### 14. Interface inheritance vs implementation inheritance: a value judgment

Hug is explicit about his opinion:

- **Interface inheritance**: "a really beautiful tool." Generalizes code nicely and is **hard to abuse** in a nasty way.
- **Implementation inheritance**: a **stronger** form of inheritance, since it dictates how. It gives you an extra choice (do I override the default or not?) and it "can be very easily abused," tempting novices *and* experts into reusing code in "Byzantine ways." More on this in a much later lecture.

### 15. Abstract data types (ADTs)

The top-level "what, not how" type has a name: an **abstract data type**. We have already built several: `List61B` (with `AList` and `SLList` as concrete implementations), and the two Deque types from the projects (`ArrayDeque` and `LinkedListDeque`).

Why this matters beyond Java: in theoretical CS, the **set of operations available to you defines the universe of tasks you can solve**, and how efficiently you can solve them in terms of those operations.

**Closing question (to be debriefed Friday):** Consider the ADT `Stack` with `push` (put on top) and `pop` (remove the top). Push 6, push 2, then pop removes the 2. Would you implement a stack with a **linked list** or an **array**?

---

## Definitions

- **Primitive type:** one of Java's eight built-in types (int, long, double, short, and so on). You cannot create new ones.
- **Reference type:** any type that is not primitive. Every class and interface you or anyone else defines (`Dog`, `AList`, `List61B`) is a reference type. A reference-type variable holds an address.
- **Hypernym:** a word whose meaning encompasses more specific words. *Dog* is a hypernym of *poodle*. In Java, `List61B` is the hypernym of `SLList` and `AList`.
- **Hyponym:** the exact inverse of hypernym; the more specific word. *Poodle* is a hyponym of *dog*. Hypernym/hyponym is a strict one-to-one inverse relationship.
- **Interface (keyword):** "like a class, but it's more abstract." It defines a reference type and lists what its implementers can do. It cannot have constructors and cannot be instantiated directly.
- **`implements`:** the keyword declaring that a class is a hyponym of an interface, and promising to provide every method the interface specifies.
- **Method signature:** the name of the method plus the types and number of its parameters. (Hug added "as well as its return type," then admitted uncertainty about whether the formal definition includes the return type. *(extra context)* Formally in Java, the signature is name plus parameter types only; the return type is not part of it, which is why you cannot overload on return type alone. For 61B purposes, use Hug's practical rule: exact match → override, different parameters → overload.)
- **Interface (of a class):** the list of all method signatures a class offers.
- **Overloading:** having two or more methods with the same name but **different signatures**. Legal and common (`Math.abs`).
- **Overriding:** a subclass/implementing class defining a method with **the same signature** as one in its superclass/interface.
- **`@Override` annotation:** an optional tag placed above an overriding method. It does not change runtime behavior; it causes a compile error if the method does not actually override anything, protecting against typos and signaling intent.
- **Interface inheritance:** the subclass inherits the superclass's interface (its method signatures). Specifies **what** the subclass can do, not how. Implementers must override every method or the code will not compile.
- **Implementation inheritance:** the subclass inherits actual code from the superclass. Specifies **how** to do something by default. In interfaces, achieved with `default` methods.
- **`default` method:** a method in an interface that has a body. Implementing classes get it automatically, and may override it.
- **Subclass / superclass:** the hyponym / hypernym in a Java inheritance relationship. Lecture uses "superclass" loosely even when the supertype is an interface.
- **Abstract data type (ADT):** a type defined solely by the operations it supports, not by how those operations are implemented. Examples so far: `List61B`, `Deque`, `Stack`.
- **Multiset:** a set that is allowed to contain duplicates (used as the design-question example).
- **Stack:** an ADT with `push` (add to top) and `pop` (remove and return the top).

---

## Worked Examples

### Example 1: The duplication problem in `WordUtils`

```java
public class WordUtils {
    public static String longest(SLList<String> list) {
        int maxDex = 0;
        for (int i = 0; i < list.size(); i += 1) {
            String longestString = list.get(maxDex);
            String thisString = list.get(i);
            if (thisString.length() > longestString.length()) {
                maxDex = i;
            }
        }
        return list.get(maxDex);
    }
}
```

**Step by step:** `maxDex` tracks the index of the longest string seen so far, starting at 0. Each iteration fetches the current champion (`list.get(maxDex)`) and the candidate (`list.get(i)`), compares `.length()`, and promotes the candidate's index on a strict improvement. After the loop, return the item at `maxDex`. This is the standard "running max" pattern you've seen many times, and Hug notes it is somewhat inefficient (it refetches the champion every iteration) but that is not the point here.

**The observation:** to make this work on an `AList`, the *only* change is the parameter type on line 2. The body is untouched. Writing both versions gives you two methods named `longest` with different signatures, which is legal **overloading**, but duplicative, non-extensible, and a maintenance hazard.

### Example 2: Defining the interface

```java
/** An interface is like a class, but it's more abstract. */
public interface List61B<Glatch> {
    // constructors have no interface
    // they are abstract and you can't instantiate them directly
    public void insert(Glatch item, int position);
    public void addFirst(Glatch x);
    public void addLast(Glatch x);
    public Glatch getFirst();
    public Glatch getLast();
    public Glatch get(int i);
    public int size();
    public Glatch removeLast();

    // how to do it by default
    default public void print() {
        for (int i = 0; i < size(); i += 1) {
            IO.print(get(i) + ", ");
        }
        IO.println();
    }
}
```

**What was kept and why:** the eight signatures are the operations *every* list can meaningfully support. **What was thrown away and why:** the `Node` inner class (array lists have no nodes), the constructors (interfaces have none), `getLastNode()` (returns a `Node`, so it is linked-list-specific and private), all method bodies, and all instance variables. `explode()` was also left out (it was a mid-lecture demo from last time, "not for us").

**Why the default `print` is written with `size()` and `get(i)`:** inside the interface there are no fields, so the only tools available are the interface's own abstract methods. This is a nice illustration of programming purely against an abstraction.

### Example 3: Implementing the interface

```java
public class SLList<Blorp> implements List61B<Blorp> {
    private class Node { ... }       // no @Override: private, not in interface
    private Node sentinel;
    private int size;

    public SLList() { ... }          // no @Override: constructors never override

    @Override
    public void insert(Blorp item, int position) { ... }

    @Override
    public void addFirst(Blorp x) { ... }
    // ... etc for addLast, getFirst, getLast, get, size, removeLast
}
```

Reading `class SLList<Blorp> implements List61B<Blorp>`: "`SLList` is a `List61B`." The compiler now checks that all eight signatures are present. Note again `Blorp` vs `Glatch`: the names differ across files and that is fine, since each is a placeholder scoped to its own file.

**The failure case (live demo):** adding `implements List61B<T>` to a version of `AList` that lacked `insert` gave a compile error. The class claimed a capability it did not have.

### Example 4: The payoff, one general method

```java
public static String longest(List61B<String> list) { ... }   // one version, forever
```

```java
void main() {
    SLList<String> someWords = new SLList<>();
    someWords.addLast("hi");
    someWords.addLast("aiouwhelfiauhweliuhf");
    IO.println(longest(someWords));   // prints the long one
    someWords.print();
}
```

**Environment / box-and-pointer reasoning in words:** `someWords` is a memory box tagged `SLList<String>` holding the address of an `SLList` object on the heap (that object holds a `sentinel` pointer and a `size` int; the sentinel points to a chain of `Node`s). When `longest(someWords)` is called, a new box named `list` is created in `longest`'s frame, tagged `List61B<String>`. We are copying an `SLList` address into a `List61B`-tagged box. Java permits this because `SLList` **is a** `List61B`. Both boxes now point at the *same* `SLList` object (no copy of the list is made). Inside `longest`, the compiler will only let you call methods declared in `List61B` through that box, but the calls dispatch to `SLList`'s actual implementations on the object being pointed at.

### Example 5: Overriding a default for efficiency

The default `print` on an `SLList` of n items does `get(0)`, `get(1)`, ..., `get(n-1)`, and each `get(i)` restarts at the sentinel and walks i links. Total work grows like 0 + 1 + 2 + ... + (n-1). On an `AList`, each `get(i)` is `items[i]`, constant work, so the default is perfectly good there.

`SLList` therefore rages against the Chipotle machine:

```java
/** RAGE AGAINST THE CHIPOTLE MACHINE */
@Override
public void print() {
    Node p = sentinel.next;
    while (p != null) {
        IO.print(p.item + ", ");
        p = p.next;
    }
    IO.println();
}
```

**Trace:** `p` starts at the first real node (`sentinel.next`, skipping the sentinel, which holds `null`). Each iteration prints `p.item`, then advances `p` one link. The loop ends when `p` falls off the end (`null`). Each node is visited exactly once, so the whole print is one single pass instead of n restarts. `IO.println()` at the end just terminates the line.

**Dynamic dispatch check:**

```java
List61B<String> someList = new SLList<>();
someList.addFirst("hello");
someList.print();     // runs SLList's fast print, NOT the slow default
```

Even though the box is tagged `List61B`, the object is an `SLList`, and the overridden method wins.

### Example 6: The multiset design question (in-class poll)

> You want a multiset (a set allowing duplicates). Approach 1: just write `ArrayMultiset`. Approach 2: write a `Multiset` interface, then write `ArrayMultiset implements Multiset`. Why bother with Approach 2?

Answers Hug explicitly **rejected** from the room:
- "Faster" → **no impact on speed.**
- "More efficient" → **no impact on efficiency.**
- "More organized" → "maybe, maybe."

The answer he was after: **in case you ever want a different kind of multiset.** Creating the `Multiset` interface is interface inheritance, so any future multiset implementation (say `LinkedListMultiset`) is guaranteed to have at least the specified operations, and any code written against `Multiset` works with all of them. Important caveat he added: if you are genuinely never going to have another implementation, you may reasonably skip the interface. Generality is a judgment call, not a reflex.

---

## Common Pitfalls

- **Thinking `@Override` does something at runtime.** It does not. It only makes your code fail to compile when you got the name or signature wrong. "Literally the only thing that the override tag does is it makes your code not compile sometimes."
- **Forgetting `@Override` when overriding a `default` method.** This is the dangerous one. Misspell `print` as `pirnt` and you silently keep the slow default in a mission-critical path, with no error anywhere.
- **Putting `@Override` on things that do not override**: private helpers, constructors, or methods absent from the interface. That is a compile error (as demonstrated live).
- **Confusing overload with override.** Different parameter list → overload (a brand-new method that happens to share a name). Exact signature match with the supertype → override.
- **Expecting generic type parameter names to match across files.** `SLList<Blorp> implements List61B<Glatch>`'s names are unrelated placeholders. There is no rule that they agree.
- **Trying to put a constructor in an interface.** Not allowed. Interfaces cannot be instantiated.
- **Putting implementation-specific methods in the interface.** `getLastNode()` returns a `Node`; array-backed lists have no nodes. The interface must contain only the common abstraction.
- **Declaring `implements` without providing every method.** Compile error. You are "fronting."
- **Assuming the default implementation is always good enough.** The default `print` is fine for `AList` and bad for `SLList`. Inherited implementations are not automatically appropriate for every implementer.
- **Assuming the static (declared) type determines which method body runs.** It does not; the overriding method on the actual object runs.
- **Thinking interfaces make code faster.** They are about generality and maintainability, not performance.

---

## Likely Exam Points

### 1. Overload vs override identification

**Q.** Given `interface Animal { void makeNoise(); }`, classify `makeNoise` in each:
```java
class Pig implements Animal { public void makeNoise() {...} }
class Dog implements Animal { public void makeNoise(Dog x) {...} }
```

**A.** `Pig.makeNoise()` has the exact same signature as the interface's method, so it **overrides**. `Dog.makeNoise(Dog x)` takes a parameter the interface method does not, so the signature does not match: it does **not** override; it **overloads** the name `makeNoise`. (Bonus consequence: `Dog` therefore fails to implement `Animal` and will not compile until a no-argument `makeNoise` is added.)

### 2. Does it compile? (the interface contract)

**Q.** You add `implements List61B<Item>` to an `AList` that has `addLast`, `getLast`, `get`, `size`, and `removeLast`, but no `insert`, `addFirst`, or `getFirst`. What happens?

**A.** Compile error. Implementing an interface obligates you to provide **every** method it declares. The compiler reports the missing ones (`insert`, `addFirst`, `getFirst`). You must either implement them or make the class abstract. *(extra context: the "abstract" escape hatch was not covered in this lecture.)*

### 3. Which method body actually runs?

**Q.**
```java
List61B<String> L = new SLList<>();
L.addLast("a");
L.addLast("b");
L.print();
```
Does the default `List61B.print` or `SLList.print` run, and why?

**A.** `SLList.print` runs. The variable's declared type (`List61B`) controls which method names you are *allowed* to call, but the object's actual type (`SLList`) controls which implementation executes. Since `SLList` overrides `print`, its version wins, which is exactly what you want since the default would be slow for a linked list.

### 4. Why is the default `print` slow for `SLList`?

**Q.** Explain concisely.

**A.** The default loops `i` from 0 to `size() - 1` calling `get(i)`. `SLList.get(i)` must start from the sentinel and walk i links every time, so printing re-traverses the front of the list once per item (work proportional to 0 + 1 + ... + n-1). `AList.get(i)` is a constant-time array index, so the same default is fast there.

### 5. Purpose of `@Override`

**Q.** Your code works. What do you gain by adding `@Override`?

**A.** Nothing at runtime. You gain (a) a compile-time typo check: if you misname or mis-sign the method so it does not actually override, the compiler errors rather than leaving a method nobody calls (and the superclass's default silently running instead), and (b) readability: it flags to the reader that this method replaces a supertype's version. In 61B you are expected to write it on every override.

### 6. Interface inheritance vs implementation inheritance

**Q.** Define each and give a one-line example from `List61B`.

**A.** *Interface inheritance*: the subclass inherits method signatures only, specifying what it can do, not how. Example: `public Item getLast();` in `List61B`, every implementer must write its own body. *Implementation inheritance*: the subclass inherits actual code. Example: `default public void print() {...}` in `List61B`, every implementer gets that body for free unless it overrides. Hug's stance: interface inheritance is hard to abuse; implementation inheritance is stronger and easily abused.

### 7. Why define an interface at all? (the multiset question)

**Q.** Is writing a `Multiset` interface before `ArrayMultiset` faster, more efficient, or neither? What is the actual benefit?

**A.** Neither faster nor more efficient: there is no performance impact. The benefit is generality and maintenance: any future implementation (e.g. `LinkedListMultiset`) is guaranteed to support at least the specified operations, and client code written against `Multiset` works with all implementations without modification. If you will truly only ever have one implementation, the interface may not be worth it.

### 8. What can and cannot go in an interface?

**Q.** Which of these belong in `List61B`: a constructor; `private Node getLastNode()`; `public int size();`; `private Node sentinel;`; `default public void print() {...}`?

**A.** Belongs: `public int size();` and the `default print`. Does not belong: constructors (interfaces have none and cannot be instantiated), `getLastNode` (private, and `Node` is a linked-list-only implementation detail), and `sentinel` (an implementation-specific instance variable). *(extra context: Java does permit `public static final` constants in interfaces, and since Java 9 private methods too, but Hug explicitly put those details out of 61B scope.)*

### 9. Subtype assignment

**Q.** Which compile?
```java
List61B<String> a = new SLList<>();
SLList<String>  b = new SLList<>();
List61B<String> c = new List61B<>();
SLList<String>  d = someList61BVariable;
```

**A.** `a` compiles (an `SLList` is a `List61B`, so its address fits in a `List61B` box). `b` compiles trivially. `c` does **not**: you cannot instantiate an interface. `d` does **not** without a cast: not every `List61B` is an `SLList`. *(extra context: casting was not covered in this lecture.)*

### 10. Hypernym/hyponym vocabulary

**Q.** In `SLList implements List61B`, which is the hypernym and which the hyponym?

**A.** `List61B` is the hypernym (the more general type, like "dog"); `SLList` is the hyponym (the specific type, like "poodle"). The relationship is exactly inverse, and these hierarchies can be multi-level (`Collection61B` → `List61B` → `SLList`).

---

## Summary

- Two near-identical classes (`AList`, `SLList`) forced duplicate utility methods; duplication is displeasing, does not extend to future lists, and is a maintenance burden.
- English solves this with **hypernyms** (dog covers poodle and malamute); Java solves it with interfaces. **Hyponym** is the exact inverse. WordNet built such a hierarchy for English nouns (returns in Project 4).
- Java has 8 **primitive types** (fixed) and unlimited **reference types**; interfaces define new reference types.
- Two-step recipe: (1) define a reference type with `interface`, (2) declare hyponyms with `implements`.
- Building `List61B` meant stripping `SLList` down to bare signatures: no `Node` class, no constructors, no `getLastNode`, no fields, no bodies. Choosing the right common abstraction is the interface designer's job.
- `implements` is a compiler-enforced contract: miss a method and you do not compile.
- Generic parameter names (`Glatch`, `Blorp`, `Item`) are per-file placeholders and need not match.
- **Overloading** = same name, different signature. **Overriding** = same signature as the supertype. Devious overload/override exam puzzles are banned in this course.
- **`@Override`** is optional and runtime-neutral, but catches typos at compile time and documents intent. Use it on every override in 61B; never on private helpers or constructors.
- **Interface inheritance**: inherit *what* (signatures). Enables one general `longest(List61B<String>)` that works on `SLList`, `AList`, and lists not yet invented.
- A `List61B`-typed box can hold an `SLList` or `AList` address, because each **is a** `List61B`.
- **Implementation inheritance** via `default` methods: inherit *how*. The default `print` uses only `size()` and `get(i)` because interfaces have no fields.
- The default `print` is fast for `AList` (array indexing) but slow for `SLList` (each `get` re-walks from the sentinel), so `SLList` overrides it with a single-pass node walk.
- When a method is overridden, the **overriding** version runs, even through a supertype-typed variable.
- Hug's take: interface inheritance is beautiful and hard to abuse; implementation inheritance is stronger and easily abused (more in a later lecture).
- An **abstract data type** specifies operations, not implementation: `List61B`, `Deque`, `Stack`. In theory CS, the available operations define what you can solve and how fast.
- Open question for Friday: implement a `Stack` (push/pop) with a linked list or an array?
