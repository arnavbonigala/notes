<!-- Wed, Sep 16, 2026 | sources: slides + code + YouTube auto-transcript + textbook -->
# Lecture 9: Interface and Implementation Inheritance

## Overview

This lecture solves a concrete annoyance: after building both `AList` (array-backed) and `SLList` (linked-list-backed) with identical method signatures, any library function written against one of them (like `WordUtils.longest`) has to be duplicated for the other, which is ugly, doesn't extend to future list classes, and is a maintenance hazard (fix a bug in one copy, forget the other). Java's answer borrows an idea from natural language: "dog" is a *hypernym* of "poodle" and "malamute", so washing instructions can be written once for dogs. In Java we express hypernym/hyponym ("is-a") relationships with the `interface` and `implements` keywords: we declare a reference type `List61B` listing only method *signatures* (what a list can do, not how), then declare `AList` and `SLList` as hyponyms via `implements`. This is **interface inheritance**, and it lets `longest(List61B<String> list)` work on any list, including ones not yet invented. Java also permits **implementation inheritance** via `default` methods, where the interface supplies actual code (e.g. `print()`) that subclasses inherit and may override; when a subclass overrides, the overriding version is the one that runs, even through a variable declared with the interface type. The lecture closes by naming this top-level "operations only" concept an **Abstract Data Type** (Deque, Stack, GrabBag, List, Set, Map), and by noting a scope change: dynamic method selection with static/dynamic types has been cut from 61B, and `extends` comes after the midterm.

---

## Key Concepts

### 1. The desire for generality (the motivating problem)

After adding an `insert` method, `AList<Item>` and `SLList<Blorp>` have *exactly the same method signatures*:

```
AList()                              SLList(), SLList(Blorp x)
void insert(Item x, int position)    void insert(Blorp item, int position)
void addFirst(Item x)                void addFirst(Blorp x)
void addLast(Item x)                 void addLast(Blorp x)
Item getFirst()                      Blorp getFirst()
Item getLast()                       Blorp getLast()
Item get(int i)                      Blorp get(int i)
int size()                           int size()
Item removeLast()                    Blorp removeLast()
```

Now write a library, `WordUtils.java`, with a method that finds the longest string in a list. Written against `SLList<String>`, it will not accept an `AList<String>`. To make it accept an `AList`, Josh pointed out in lecture that you change *exactly one thing*: the parameter type on the first line. The body is untouched. That's the tell-tale sign that the code doesn't actually care which list it is, only that it has `size()` and `get(int)`.

### 2. Method overloading is a bad fix here

Java permits two methods with the same name as long as their parameter lists differ, so you can literally write both:

```java
public static String longest(AList<String> list)  { ... }
public static String longest(SLList<String> list) { ... }
```

This compiles and works. The lecture's objections:

- The code is virtually identical. "Aesthetically gross."
- It won't work for future lists: invent a `QList` and you need a third copy.
- It's harder to maintain: find a bug, fix it in the `SLList` version, forget the `AList` version.

Josh noted that "maintenance" for code is a slightly odd word, since code doesn't wear out like a bicycle; maintenance happens because you later realize things you want to change.

### 3. Hypernyms and hyponyms

Natural languages already solved this. Instructions for washing a poodle and washing a malamute are identical except for the noun, so English doesn't write both: it writes instructions for washing a **dog**.

- **Hypernym**: the more general word. "dog" is a hypernym of "poodle", "malamute", "yorkie", "dachshund".
- **Hyponym**: the more specific word. "poodle" is a hyponym of "dog".

These are exact opposites (a one-to-one relationship, as stated in lecture), and they stack into a hierarchy: a dog *is-a* canine, a canine *is-a* carnivore, a carnivore *is-an* animal. The WordNet project built exactly such a graph of English nouns; the lecture mentioned it will resurface in Project 4.

The goal for the rest of the lecture: express hypernym/hyponym relationships in Java.

### 4. Reference types, and the two-step process

Java has eight primitive types (int, long, double, short, etc.), and you cannot add new ones. Everything else is a **reference type**: `Dog`, `Cat`, `AList`, `SLList`, `List61B`. A "type" is essentially the tag on a variable that says what its memory box is allowed to hold.

- **Step 1**: define a reference type for the hypernym: `List61B.java`, declared with `interface` instead of `class`.
- **Step 2**: declare that `SLList` and `AList` are hyponyms of that type, using `implements`.

### 5. `interface`: what, not how

An interface is like a class but more abstract: it is a **specification of what a List is able to do, not how to do it**.

Josh built `List61B` live by copying `SLList` and deleting everything implementation-specific:

- The private `Node` class: **gone**. Some lists (like `AList`) have no nodes.
- The constructors: **gone**. "Constructors have no interface; they are abstract and you can't instantiate them directly."
- `getLastNode()`: **gone**. That's a private detail of a linked-list implementation. Josh stressed that the job of whoever designs the interface is to find the abstractions *all* implementations can honestly support.
- Method bodies: **gone**, replaced by semicolons. Only the signatures survive.

The generic type parameter is just a placeholder local to each file: `List61B` uses `<Glatch>` in the lecture code, `SLList` uses `<Blorp>`, `AList` uses `<Item>`. They do not need to match.

### 6. `implements`: declaring the is-a relationship

```java
public class SLList<Blorp> implements List61B<Blorp> { ... }
public class AList<Item>  implements List61B<Item>  { ... }
```

This declares to the compiler (and to the world) that an `SLList` **is-a** `List61B`. Immediately, `WordUtils.longest` can be written once:

```java
public static String longest(List61B<String> list) { ... }
```

and it works on `SLList`, `AList`, `QList`, "and even lists that have not yet been invented."

**The contract is enforced.** In lecture, adding `implements List61B<Item>` to `AList` produced a compile error, because that version of `AList` lacked an `insert` method. Josh's analogy: you can claim to be a member of the Backstreet Boys, but if you can't spin, dance, and do everything a Backstreet Boy can do, people will see right through you. The compiler sees right through you too. Slide version: if `List61B` declares `public void proo();` and `AList` has no `proo()`, `AList` will not compile.

### 7. Overriding vs. overloading

These sound alike and are constantly confused, so nail the distinction:

- **Signature** = method name + number and types of parameters. (In lecture Josh added "as well as its return type," then admitted he couldn't recall whether the formal definition includes return type. The slide definition given in this course is: *the signature of a method is its name and the number and type of params*. Use the slide definition.)
- **Overriding**: a subclass has a method with the **exact same signature** as one in the superclass/superinterface. `AList.addLast(Item)` overrides `List61B.addLast(Item)`. `Pig.makeNoise()` overrides `Animal.makeNoise()`.
- **Overloading**: two methods with the **same name but different signatures**. `Dog.makeNoise(Dog x)` does not match `Animal.makeNoise()`, so it's overloading, not overriding. `Math.abs(int)` and `Math.abs(double)` are overloaded.

Josh's mnemonic framing: overloading is "a whole bunch of different things that all look kind of the same"; overriding is "my boss has one idea and I have another."

### 8. `@Override`

An optional annotation placed above an overriding method:

```java
@Override
public void addLast(Item x) { ... }
```

- It does **not** change behavior. Without it, you are still overriding.
- Its **only** effect is that the code won't compile if the method isn't actually overriding anything.
- **Main reason to use it: protects against typos.** Write `public void addLats(Item x)` or `public void pirnt()` and you've silently created a brand-new method nobody will ever call, while the inherited version keeps running. With `@Override`, that's a compile error.
- Secondary reason: it reminds the reader that this definition came from somewhere higher up in the hierarchy.
- In 61B, always mark overriding methods with `@Override`. Note from the lecture code walkthrough: you do **not** put `@Override` on private helper methods or constructors.

### 9. Interface inheritance

Specifying the capabilities of a subclass using `implements` is **interface inheritance**.

- *Interface*: the list of all method signatures.
- *Inheritance*: the subclass "inherits" the interface from the superclass.
- Specifies **what** the subclass can do, but not **how**.
- Subclasses must override **all** of these methods, or fail to compile.
- Relationships can be multi-generational: `Collection61B` → `List61B` → `AList`/`SLList`. (Interfaces drawn in white, classes in green on the slide; details deferred to a later lecture.)
- A class may implement more than one interface (asked in lecture: yes, you could be a `List61B` and an `Iterable` and a `StuffHolder` simultaneously).

### 10. "Is-a" and memory boxes

A memory box can only hold 64-bit addresses of the appropriate type. So `List61B<String> inputList` can only hold the address of a `List61B<String>`. In box-and-pointer terms: `WordUtils.longest(a1)` creates a new box named `inputList` typed `List61B<String>`, and copies into it the 64-bit address sitting in `a1`. That box points at an actual `AList` object out in the heap, which internally holds an `items` array reference and an `int size`. This is legal precisely because an `AList` **is-a** `List61B`. The declared type of the box restricts what addresses may be stored; the object at the far end of the arrow is still a full `AList`.

Likewise:

```java
List61B<String> someList = new SLList<>();
someList.addFirst("elk");
```

compiles and runs fine. An `SLList` object is created, its address is stored in `someList`, and `"elk"` is inserted into that `SLList`. (The tempting wrong answer, "it crashes because the interface doesn't implement `addFirst`", misunderstands what an interface is for: the interface *declares* `addFirst`, which is exactly what makes the call legal, and the real `SLList` object supplies the code.)

### 11. Why bother with the interface at all? (the Multiset question)

Approach one: just write `ArrayMultiset`. Approach two: write a `Multiset` interface plus an identical `ArrayMultiset` that `implements Multiset` with `@Override` tags. What does approach two buy you?

Answers rejected in lecture:
- "Faster" -- **no**, there is no impact on speed.
- "More efficient" -- **no**, there is no impact on efficiency.
- "More organized" -- maybe.

The actual point: **in case you ever want a different implementation of multiset.** Any future implementation (e.g. `LinkedListMultiset`) is guaranteed to have at least those operations. Josh added the honest caveat: if you truly will never have a second implementation, you may not need the interface for your purposes.

### 12. Implementation inheritance and `default` methods

Interface inheritance: subclass inherits signatures, but NOT implementation.
Implementation inheritance: subclass inherits signatures **AND** implementation.

Java allows the latter via the `default` keyword in an interface. Josh's analogy: interface inheritance is the boss telling you *what* you must be able to do; implementation inheritance is the boss also telling you *how*, like Chipotle training you to put nine beans on each burrito.

```java
default public void print() {
    for (int i = 0; i < size(); i += 1) {
        System.out.print(get(i) + " ");
    }
    System.out.println();
}
```

Note what the default method is allowed to use: it has **no instance variables** to work with, only the other methods the interface promises (`size()` and `get(i)`). Those methods may not have been written yet when you author the interface, but by the time anyone actually implements `List61B`, they exist. Also note: you write `size()` and `get(i)` bare, not `this.size()`; unlike Python, no explicit `self`/`this` is required (though `this.get(i)` is also legal).

Now `SLList` gets a working `print()` for free: search `SLList.java` and you will not find a `print` method, yet `someWords.print()` works.

### 13. Default methods can be inefficient for some implementations

Is that `print()` efficient?

- **Efficient for `AList`**: `get(i)` is a direct array index.
- **Inefficient for `SLList`**: `get(i)` has to walk from the front every single time. Printing a 100-element `SLList` walks 0 nodes, then 1, then 2, ... then 99. Josh's image: like a shuttle-run exercise where each trip goes farther and you keep returning to the start.

So the answer is **(b) efficient for AList, inefficient for SLList**. (64% of the class got this.)

### 14. Overriding a default method

If you don't like a default method, override it:

```java
public class SLList<Item> implements List61B<Item> {
    @Override
    public void print() {
        for (Node p = sentinel.next; p != null; p = p.next) {
            System.out.print(p.item + " ");
        }
        System.out.println();
    }
}
```

This walks each node once. A `while` loop version is equally fine; Josh noted students sometimes panic at the `for` loop form thinking something special is required, but it's just the same code.

**This is also the best argument for `@Override`.** Without it, a typo (`pirnt()`) leaves you with a method nobody calls, while the slow default keeps silently running: exactly the kind of bug that could sink a mission-critical application. With `@Override`, the compiler catches it.

**Which one runs?**

```java
List61B<String> someList = new SLList<>();
someList.addLast("elk");
someList.addLast("are");
someList.addLast("watching");
someList.print();
```

**`SLList.print()` runs**, the efficient one, not the default. Reason given in lecture: "since the object we're pointing at has overridden print, we'll use that overridden method." The declared type of the variable does not downgrade you to the default. Josh's phrasing: "whenever you override a method, in almost all normal circumstances the actual overridden method will get called."

### 15. Interface vs. implementation inheritance, editorially

- **Interface inheritance (what)**: lets you generalize code in a powerful, simple way. Josh: "a really beautiful tool... hard to abuse in a way that's super nasty."
- **Implementation inheritance (how)**: allows code reuse (`print()` written once in `List61B.java`), and gives subclass designers another dimension of control: decide whether or not to override each default. But it is a stronger form of inheritance and can lead people to reuse code "in Byzantine ways." The lecture promises a later lecture on how it can be abused.

### 16. Abstract Data Types

An **Abstract Data Type (ADT)** is defined only by its operations, not by its implementation.

- In class: `List61B` is the ADT; `AList` and `SLList` are implementations.
- In projects: `Deque` is the ADT; `ArrayDeque` and `LinkedListDeque` are implementations. Deque ADT operations: `addFirst(Item x)`, `addLast(Item x)`, `boolean isEmpty()`, `int size()`, `printDeque()`, `Item removeFirst()`, `Item removeLast()`, `Item get(int index)`.
- Java's syntax differentiates ADTs from implementations nicely: `List<Integer> L = new ArrayList<>();` names the abstraction on the left and the implementation on the right. Caveat: Java interfaces aren't *purely* abstract, since they can contain implementation details like default methods.
- Josh's broader point: in theoretical CS, the set of operations available to you defines the universe of tasks you can solve, and how efficiently.

**Other ADTs in the lecture:**

*Stack*: `push(int x)` puts x on top; `int pop()` removes and returns the top item. Linked list or array? **Both are about the same.** No resizing for linked lists, so a linked list is probably a little faster. (Josh left this as the closing/attendance question to be debriefed Friday.)

*GrabBag*: `insert(int x)`, `int remove()` (removes a random item), `int sample()` (samples a random item without removing), `int size()`. Linked list or array? The slide maps these to `insertBack()`, `getBack()`, `deleteBack()`, `get(int i)`. **(extra context)** The slides pose the question but the transcript ends before the debrief; the slide's mapping points at array, because `sample()` needs `get(int i)` at a random index, which is instant in an array and requires walking in a linked list.

*Java's built-in interfaces*: Lists, Sets, and Maps. Lists and Sets are subinterfaces of a more abstract interface called `Collection` (not really discussed in 61B). Maps are also known as associative arrays, associative lists (Lisp), symbol tables, or dictionaries (Python).

### 17. Changes to scope in 61B (important for exams)

61B used to teach **dynamic method selection**, relying on "static type" (a.k.a. compile-time type) and "dynamic type" (a.k.a. run-time type). Josh **cut this material**: students spent a great deal of time on something that isn't ultimately very important, and this is not a class about Java minutiae. TAs used to love writing tricky overload/override puzzle questions; he has banned them. (Reference for the curious: the Spring 2017 Bird/Falcon/`gulgate` midterm problem, and the Spring 2021 61B slides.)

What you **do** need: if multiple methods have different signatures, it's overloading; if it's an exact signature match from the lower-down class to its "boss", it's overriding.

**The `extends` keyword will be covered after the midterm and is NOT in scope for midterm 1.**

---

## Definitions

- **Hypernym**: the more general term in an is-a relationship. "Dog" is a hypernym of "poodle"; `List61B` is a hypernym of `AList` and `SLList`.
- **Hyponym**: the more specific term; the exact opposite relationship. "Poodle" is a hyponym of "dog"; `SLList` is a hyponym of `List61B`.
- **Reference type**: any type that is not one of Java's eight primitives. All classes and interfaces are reference types; you cannot define new primitive types.
- **Interface (Java keyword)**: a reference type declared with `interface` instead of `class`, specifying what a type is able to do, not how to do it. Contains method signatures, may contain `default` methods, and has no constructors.
- **Interface (general sense)**: the list of all method signatures of a class.
- **`implements`**: keyword declaring that a class is a hyponym of an interface, i.e. that it supplies all the interface's methods.
- **Signature**: a method's name together with the number and types of its parameters.
- **Overriding**: a subclass defines a method with the exact same signature as one in the superclass/superinterface.
- **Overloading**: two or more methods share a name but have different signatures.
- **`@Override`**: an optional annotation whose sole effect is a compile error if the annotated method does not actually override anything. Used to catch typos.
- **Interface inheritance**: inheriting method signatures but not implementations; specifies *what*, not *how*. Subclasses must override every method or fail to compile.
- **Implementation inheritance**: inheriting signatures *and* implementations; specifies *how* as well as *what*.
- **`default` (in an interface)**: keyword marking a method in an interface that comes with a body, which implementing classes inherit unless they override it.
- **Is-a relationship**: the relationship declared by `implements`; permits a variable of the supertype to hold a reference to an object of the subtype.
- **Abstract Data Type (ADT)**: a type defined only by its operations, not by its implementation (e.g. Deque, Stack, GrabBag, List, Set, Map).
- **Method overloading (restated for contrast)**: multiple methods, same name, different parameters, all coexisting in one class.

---

## Worked Examples

### Example 1: The duplication problem and its resolution

**Before.** `WordUtils.longest` written against `SLList`:

```java
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
```

Step by step: `maxDex` tracks the index of the longest string seen so far, starting at 0. Each iteration fetches the current champion (`list.get(maxDex)`) and the candidate (`list.get(i)`), compares lengths, and updates `maxDex` on a strict improvement. At the end, fetch and return the champion. (The slides note this is very inefficient, because of the repeated `get(maxDex)` inside the loop; don't worry about it for now.)

**Making it work on `AList` instead:** change *one token*, `SLList` → `AList` in the parameter type. Nothing else. This is the clue that the body only needs `size()` and `get(int)`.

**Making it work on both by overloading:** paste a second copy with the other parameter type. Legal Java, but now a bug fix in one copy must be mirrored in the other, and a future `QList` demands a third copy.

**After.** One method, written against the interface:

```java
public static String longest(List61B<String> list) { /* identical body */ }
```

and called as:

```java
AList<String> a = new AList<>();
a.addLast("egg");
a.addLast("boyz");
longest(a);         // legal: an AList is-a List61B
```

*Box-and-pointer reasoning:* `a` is a box of declared type `AList<String>` holding the address of an `AList` object on the heap. Calling `longest(a)` creates a parameter box named `list` of declared type `List61B<String>` and copies that same address into it. Two boxes of different declared types now point at the same single object. Nothing was copied or converted; only the *label on the box* differs. When `list.get(i)` executes, the code that runs is `AList`'s `get`, because the object at the end of the arrow is an `AList`.

### Example 2: Building `List61B`

```java
package lec9_inheritance1;

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

Reading it line by line: each declaration ends in a semicolon with no body, so it's a pure promise. There are no constructors, no `Node` inner class, no `getLastNode()` private helper, and no instance variables, because those are all "how", and none of them are shared by every possible list. `print()` is the one exception: `default` gives it a body, and that body is written entirely in terms of the interface's own promises (`size()` and `get(i)`), which is the only vocabulary available to it.

`<Glatch>` is a purely local placeholder. Josh renamed it from `Blorp` to `Glatch` live to prove the point: the implementing classes use `<Blorp>` and `<Item>` and nothing breaks.

### Example 3: `implements`, and the compile error that teaches the contract

```java
public class SLList<Blorp> implements List61B<Blorp> {
    private class Node { ... }          // no @Override: private helper class
    private Node sentinel;
    private int size;

    public SLList() { ... }             // no @Override: constructor
    public SLList(Blorp x) { ... }      // no @Override: constructor

    @Override
    public void insert(Blorp item, int position) { ... }
    @Override
    public void addFirst(Blorp x) { ... }
    @Override
    public void addLast(Blorp x) { ... }
    @Override
    public Blorp getFirst() { ... }

    private Node getLastNode() { ... }  // no @Override: not in the interface!

    @Override
    public Blorp getLast() { ... }
    @Override
    public Blorp get(int i) { ... }
    @Override
    public int size() { ... }
    @Override
    public Blorp removeLast() { ... }

    public void explode() { }           // extra method: fine, not in interface
}
```

Note carefully: `getLastNode()` is private and **not** in `List61B`, so adding `@Override` there is a compile error (this happened live: "Oh, that's yelling at me"). `explode()` is a public method not in the interface, which is perfectly allowed: a class may do *more* than its interface promises, just never less.

Now try it on the in-lecture `AList`, which lacked `insert`:

```java
public class AList<Item> implements List61B<Item> { ... }  // COMPILE ERROR
```

The compiler reports a missing `insert` method. `AList` claimed to be a `List61B` while being unable to do everything a `List61B` must do. The fix in the final lecture code is to actually write `insert`:

```java
/** Inserts item into given position. */
public void insert(Item x, int position) {
    Item[] newItems = (Item[]) new Object[items.length + 1];
    System.arraycopy(items, 0, newItems, 0, position);
    newItems[position] = x;
    System.arraycopy(items, position, newItems, position + 1, items.length - position);
    items = newItems;
}
```

This allocates a new array one slot bigger, copies the prefix `[0, position)`, drops `x` at `position`, copies the suffix, and swaps in the new array.

### Example 4: Overriding vs. overloading, side by side

```java
public interface Animal {
    public void makeNoise();
}

public class Pig implements Animal {
    public void makeNoise() {          // signature: makeNoise()  -> EXACT match
        System.out.print("oink");
    }
}                                       // Pig OVERRIDES makeNoise()

public class Dog implements Animal {
    public void makeNoise(Dog x) {     // signature: makeNoise(Dog) -> NO match
        ...
    }
}                                       // makeNoise is OVERLOADED
```

Why is `Dog` overloading? Because the parameter list differs: `makeNoise()` vs `makeNoise(Dog)`. Different signature, so no override happens. **(extra context)** As written, `Dog` would actually fail to compile, since it never supplies the no-argument `makeNoise()` that `Animal` demands, exactly the `AList`/`insert` situation.

And plain overloading with no inheritance involved:

```java
public class Math {
    public int    abs(int a)
    public double abs(double a)         // abs is OVERLOADED
}
```

### Example 5: Default `print()` vs. overridden `print()`

Trace `print()` on `SLList` containing `["hi", "aiouwhelfiauhweliuhf"]`, using the **default**:

1. `size()` returns 2, so the loop runs `i = 0, 1`.
2. `i = 0`: `get(0)` walks from `sentinel.next` 0 hops, returns `"hi"`. Print `hi, `.
3. `i = 1`: `get(1)` starts again at `sentinel.next` and walks 1 hop, returns the long string. Print it.
4. Newline.

For 2 elements that's 0 + 1 = 1 hop. For *N* elements it's 0 + 1 + ... + (N-1) hops, i.e. quadratic-ish restarting-from-scratch work. For `AList`, each `get(i)` is `items[i]`, a single array index, so the same loop is efficient.

Now `SLList`'s override:

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

`p` starts at the first real node (the sentinel holds no data) and advances exactly once per element, printing as it goes, stopping when `p` falls off the end at `null`. Total hops: *N*. The pointer never restarts.

*Box-and-pointer reasoning:* `p` is a local box holding a `Node` address. Each `p = p.next` overwrites that box with the address stored in the current node's `next` field, so `p` slides along the chain of nodes one link at a time, rather than being reset to `sentinel.next` on every element as `get(i)` does.

### Example 6: Which `print()` runs?

```java
public static void main(String[] args) {
    List61B<String> someList = new SLList<>();
    someList.addLast("elk");
    someList.addLast("are");
    someList.addLast("watching");
    someList.print();
}
```

`someList` is declared `List61B<String>`, so its box may hold any `List61B` address, and the `new SLList<>()` address qualifies. When `print()` is called, **`SLList.print()` runs**, the efficient overridden version. The object being pointed at has overridden `print`, so the overriding method wins; the declared type of the variable only governs *which methods you are allowed to call*, not which body executes. Output: `elk, are, watching, `.

Contrast: if you called `someList.explode()`, that would **not compile**, because `explode()` is not in the `List61B` interface, even though the actual object has one. **(extra context, though it follows directly from the "memory box holds a `List61B`" rule the lecture gave.)**

---

## Common Pitfalls

1. **Thinking overloading is the solution to generality.** It compiles, but it duplicates code, doesn't extend to future classes, and invites the fix-one-forget-the-other bug. Interface inheritance is the right tool.

2. **Confusing overriding with overloading.** Check the *signature*. Exact name + parameter match with a method in the supertype = overriding. Same name, different parameters = overloading. `makeNoise(Dog x)` does not override `makeNoise()`.

3. **Thinking `@Override` does something at runtime.** It doesn't. Its only effect is to make non-overriding code fail to compile. Behavior is identical with or without it.

4. **Putting `@Override` on the wrong things.** Constructors and private helper methods (`getLastNode()`) aren't overriding anything, and `@Override` there is a compile error.

5. **Forgetting that `implements` is a binding contract.** Miss even one interface method and the class will not compile. Conversely, having *extra* methods (`explode()`) is fine.

6. **Trying to put constructors in an interface.** Interfaces have no constructors; you cannot instantiate an interface directly.

7. **Believing an interface makes code faster or more memory-efficient.** It does neither. Explicitly rejected in lecture. The benefit is generality and the guarantee that future implementations supply the same operations.

8. **Assuming `List61B<String> someList = new SLList<>()` then `someList.addFirst("elk")` fails because "interfaces have no implementation."** It works: the interface declares `addFirst`, which makes the call legal, and the `SLList` object supplies the code.

9. **Assuming the declared type determines which method body runs.** It does not. If the object's class overrides the method, the override runs, even through an interface-typed variable.

10. **Assuming a `default` method is efficient for every implementation.** The default `print()` is fine for `AList` and bad for `SLList`, because `get(i)` has wildly different costs. Defaults are written in terms of interface methods whose cost the interface author cannot know.

11. **A typo'd override silently doing nothing.** `pirnt()` or `addLats()` compiles happily without `@Override`, and the slow (or wrong) inherited version keeps running. This is the whole reason for the annotation.

12. **Expecting generic type parameter names to match across files.** `<Glatch>`, `<Blorp>`, `<Item>` are local placeholders; they do not need to agree.

13. **Studying static/dynamic types and dynamic method selection for this course.** That material was cut. Also, `extends` is not in scope for midterm 1.

---

## Likely Exam Points

### 1. Identify overriding vs. overloading

**Q.** Given `public interface Animal { void makeNoise(); }`, classify each method in `class Cat implements Animal`: (i) `public void makeNoise()`, (ii) `public void makeNoise(int volume)`, (iii) `public void makeNoise(String s)`. Which are overrides?

**A.** Only (i) is an override: its signature `makeNoise()` exactly matches the interface's. (ii) and (iii) have different parameter lists, so they are overloads of the name `makeNoise` within `Cat`. Only (i) may carry `@Override`; putting `@Override` on (ii) or (iii) is a compile error.

### 2. Does it compile? (the `implements` contract)

**Q.** `List61B` declares eight methods plus a `default print()`. `QList implements List61B<Item>` and defines all eight methods plus `helper()`, but misspells one as `removeLastt`. Does `QList` compile? Would `@Override` have helped?

**A.** No, it does not compile: `QList` fails to supply `removeLast`, so it does not fulfill the interface contract. With `@Override` on `removeLastt` you'd additionally get a clear "does not override anything" error pointing straight at the typo, which is the second reason the annotation is recommended. `helper()` being an extra method is fine. Note `print()` does not need to be implemented, since `default` supplies it.

### 3. Which method body runs?

**Q.** `List61B` has `default public void print()` using `get(i)`. `SLList` overrides `print()` with a node-walking version; `AList` does not override it. What runs, and is it efficient?

```java
List61B<String> x = new SLList<>();
List61B<String> y = new AList<>();
x.print();
y.print();
```

**A.** `x.print()` runs `SLList.print()` (the override wins, regardless of the declared type `List61B`) and is efficient: one pointer hop per element. `y.print()` runs the inherited default and is also efficient, because `AList.get(i)` is a constant-time array index.

### 4. Legality of assignments and calls through an interface type

**Q.** With `SLList implements List61B` and `SLList` having a public `explode()` not declared in `List61B`, which lines compile?
```java
List61B<String> a = new SLList<>();   // (1)
SLList<String> b = new SLList<>();    // (2)
a.addFirst("elk");                    // (3)
a.explode();                          // (4)
b.explode();                          // (5)
```

**A.** (1) compiles: an `SLList` is-a `List61B`, so its address fits in a `List61B` box. (2) compiles trivially. (3) compiles: `addFirst` is declared in `List61B`. (4) does **not** compile: `explode()` is not part of the `List61B` interface, so it is not callable through a `List61B`-typed variable. (5) compiles.

### 5. Why bother with an interface? (conceptual short answer)

**Q.** You're told to implement a Multiset. What is the advantage of writing a `Multiset` interface and an `ArrayMultiset implements Multiset`, over just writing `ArrayMultiset` alone?

**A.** Not speed and not efficiency: there is zero runtime impact. The advantage is generality: if you later write a second implementation (e.g. `LinkedListMultiset`), the interface guarantees it offers at least the same operations, and all client code written against `Multiset` works unchanged on both. This is interface inheritance.

### 6. Default method efficiency analysis

**Q.** The default `print()` loops `i` from 0 to `size()-1` calling `get(i)`. Is it efficient for `AList`? For `SLList`? Explain.

**A.** Efficient for `AList`: `get(i)` is `items[i]`, immediate. Inefficient for `SLList`: `get(i)` restarts at `sentinel.next` and walks `i` links every call, so printing *N* items costs 0+1+2+...+(N-1) hops instead of *N*. Fix: override `print()` in `SLList` to walk the list once with a single moving `Node` pointer.

### 7. Fill in the interface

**Q.** Write the `List61B` interface declaration for a list supporting `addLast`, `get`, and `size`, plus a default `getLast()` implemented using only interface methods.

**A.**
```java
public interface List61B<Item> {
    public void addLast(Item x);
    public Item get(int i);
    public int size();

    default public Item getLast() {
        return get(size() - 1);
    }
}
```
Key points: no bodies on the non-default methods (semicolons only), no constructors, no instance variables, and the default method may only use the interface's own methods.

### 8. ADT operations and implementation choice

**Q.** The Stack ADT supports `push(int x)` and `int pop()`. Would a linked list or an array give faster overall performance? What about the GrabBag ADT (`insert`, `remove` a random item, `sample` a random item, `size`)?

**A.** Stack: **both are about the same**; a linked list avoids resizing so it may be slightly faster. GrabBag: `sample()` and `remove()` need to reach a *random* index, which an array does immediately via `get(int i)` while a linked list must walk to it, so an **array** is the better fit. (The GrabBag debrief was left for the following lecture; this reasoning follows from the slide's operation mapping. **(extra context)**)

### 9. Terminology recall

**Q.** Define interface inheritance and implementation inheritance, and give the 61B example of each.

**A.** Interface inheritance: the subclass inherits method *signatures* only, specifying what it can do but not how; it must override all of them or fail to compile. Example: `AList implements List61B` inheriting `addFirst`, `get`, `size`, etc. Implementation inheritance: the subclass inherits signatures *and* code. Example: `List61B`'s `default print()`, which `AList` gets for free.

### 10. Hypernym/hyponym mapping

**Q.** In the relationship `SLList implements List61B`, which is the hypernym and which the hyponym? Which corresponds to "dog" and which to "poodle"?

**A.** `List61B` is the hypernym (like "dog"); `SLList` is the hyponym (like "poodle"). The is-a arrow points upward: an `SLList` is-a `List61B`, just as a poodle is-a dog.

---

## Summary

- Duplicating `longest` for `AList` and `SLList` works via **method overloading** (same name, different signatures) but is gross, doesn't scale to new list types, and is a maintenance hazard.
- Natural languages solve this with **hypernyms** ("dog" over "poodle", "malamute"); the opposite relation is a **hyponym**. These form is-a hierarchies (dog → canine → carnivore → animal).
- Java expresses this in two steps: (1) define a reference type with `interface` (e.g. `List61B.java`), (2) mark hyponyms with `implements`.
- An **interface** specifies *what* a type can do, not *how*: signatures only, no constructors, no instance variables, no implementation-specific helpers like `getLastNode()`.
- Generic type parameter names (`<Item>`, `<Blorp>`, `<Glatch>`) are local placeholders and need not match across files.
- `implements` is an enforced contract: omit any declared method and the class will not compile. Extra methods are allowed.
- **Overriding** = exact same signature as the supertype's method. **Overloading** = same name, different signature. Signature = name + number and types of parameters.
- **`@Override`** changes nothing at runtime; its sole effect is a compile error when the method isn't actually overriding. Use it in 61B to catch typos like `addLats` / `pirnt`.
- **Interface inheritance**: inherit signatures only; generalizes code so `WordUtils.longest(List61B<String>)` works on any list, including ones not yet invented.
- **Implementation inheritance**: use `default` in an interface to supply a body subclasses inherit (e.g. `print()` written with `size()` and `get(i)`).
- A default method can be efficient for one implementation and terrible for another: `print()` is fast for `AList`, slow for `SLList`, because `get(i)` restarts the walk each time.
- Subclasses may **override defaults**; when they do, the overriding method runs, even through a variable declared with the interface type.
- A memory box typed `List61B<String>` can hold any `List61B` address because of the is-a relationship, but only interface-declared methods may be called through it.
- An interface has **no** effect on speed or memory; its payoff is generality and a guarantee for future implementations.
- An **Abstract Data Type** is defined only by its operations: Deque (`ArrayDeque`, `LinkedListDeque`), Stack, GrabBag, List, Set, Map. Java expresses this as `List<Integer> L = new ArrayList<>();`. Lists and Sets are subinterfaces of `Collection`; Maps go by many names (associative arrays, symbol tables, dictionaries).
- Stack: array and linked list perform about the same (linked list avoids resizing). GrabBag favors an array because random-index access is needed.
- Scope: static/dynamic types and dynamic method selection have been **cut** from 61B; `extends` comes after the midterm and is **not** on midterm 1.
