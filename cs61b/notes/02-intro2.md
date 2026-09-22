<!-- Fri, Aug 28, 2026 | sources: code + your recording -->
# Lecture 2: Intro 2

This lecture is about **defining your own classes in Java**, built live in IntelliJ around a running `Dog` example. We start by writing a class with an instance variable (`size`), a constructor (which is *not* a method: it says how to make Dogs), and an instance method (`makeNoise`) whose behavior depends on the object's own state. Along the way we compare Java to Python at every step: Java requires `new` to instantiate, requires every variable to have a declared type, requires the full list of properties to be declared up front and never changed, and never passes `self` explicitly (Java gives you `this` implicitly). We then pin down precise terminology (declaration, instantiation, assignment, invocation), and draw the central distinction of the day: **static means there is no `this`**, so static methods are invoked on the class (`Dog.maxDog(a, b)`) and cannot touch instance variables, while instance methods are invoked on an object (`d.makeNoise()`) and can. The lecture closes with a warm-up for Lecture 3: Java 4 style lists, showing that `import` in Java only shortens a name (unlike Python, where it makes something available), that `List` is abstract so you cannot instantiate it directly, and that Java deliberately offers many list implementations (`ArrayList`, `LinkedList`, `Stack`, ...) with different performance and operations.

---

## Key Concepts

### 1. A class is a blueprint for objects

In Java, as in Python, a class lets you define your own type. `Dog.java` declares that a type called `Dog` exists. The class file says, authoritatively: *all Dog objects are this way*. They have a `size`, they can `makeNoise()`, they cannot do anything else, and they always have exactly those properties.

When you instantiate the class, you get a **Dog object** (an *instance*). The blueprint analogy is the one the lecture used: the class is the plan, each instance is a house built from it.

### 2. Instance variables are declared at the top, and the list is fixed

Every Java class declares its properties up front, in a dedicated region at the top of the class body:

```java
class Dog {
    int size;
    ...
}
```

This is one of the biggest departures from Python. In Python you can staple a new attribute onto an object whenever you feel like it. In Java the rules are strict: **the list of properties is fixed at compile time**. You can change the *value* of `size` on a dog you already made, but you can never give that dog a `name` if `Dog` has no `name` field. The lecture's phrasing: try it, and "the universe explodes", except it does not actually explode, because it will not even compile. That is the characteristic Java move: catch the mistake before the program ever runs.

Instructor's framing: **Java is a strict language, Python is more freemium.**

If you declare an instance variable and never use it (say `int numLegs;`), the code still compiles and runs, and the field defaults to `0`. Defaults get covered in Lecture 3.

### 3. Constructors: how to make a Dog

```java
Dog(int s) {
    size = s;
}
```

This is **not a method**. It is a **constructor**: it says how to make Dogs. Notice what is missing compared to Python's `__init__`:

- No `self` parameter. Java never makes you pass the receiver explicitly.
- No return type, and the name matches the class name exactly.
- You write `size = s`, not `self.size = s`. Java figures out from context whose `size` you mean.

You *may* write `this.size = s` and it does exactly the same thing. You do not have to.

Mental model from lecture: when the dog is about to be born, a `size` variable appears (initially undefined / zero-ish), and then the value you passed in gets slotted into it. Constructors can do far more complicated work than a single assignment, and later in the course they will.

### 4. `this` and shadowing

`this` refers to the current object. In `makeNoise`, `this.size` and plain `size` mean the same thing.

The exception is when a parameter has the same name as an instance variable. A student asked what happens if the constructor parameter were also named `size`:

```java
Dog(int size) {
    size = size;   // does NOTHING useful
}
```

Here the local parameter **shadows** the instance variable. `size = size` just takes the local and assigns it to itself. The instance variable is never touched. The fix is to disambiguate:

```java
this.size = size;  // works
```

Think of it in 61A environment-diagram terms: there are two separate variables in play, the local `size` (in the constructor's frame) and `this.size` (a field inside the object). The assignment copies the local's value into the object's field only if you name the target explicitly with `this.`.

The lecture's own habit is to sidestep the issue entirely by not letting the names collide (hence the parameter `s`).

### 5. Instance methods: behavior that depends on the object's own state

```java
void makeNoise() {
    if (this.size < 10) {
        IO.println("yipyipyippyip");
    } else if (this.size < 30) {
        IO.println("bark");
    } else {
        IO.println("aroooooooooooo");
    }
}
```

The motivating idea (and the reason for the two dog videos in lecture): we write code to represent things in the world, and a small dog and a huge dog make different noises. An **instance method** means *this specific dog does this thing*, and it can read that specific dog's instance variables.

`void` means the method returns nothing. That was a student question answered directly: you use `void` when the method does not return anything.

### 6. Terminology: method vs function

A function that lives inside a class is called a **method**. In Java, because essentially all code lives inside classes, the words "method" and "function" are near-interchangeable. Methods come in two flavors: **instance methods** and **static methods** (the latter is roughly analogous to a class method / static method in Python).

### 7. Where code lives, and `main`

You cannot have loose code floating outside a class in Java. When you run a Java program, Java finds the method called `main` and runs it. Code outside a class is orphaned: it has no place to live, and Java will not know what it means.

Important currency note from the lecture: **since Java 25 (September 2025)**, the rules relaxed. You write `void main()` rather than the old `public static void main(String[] args)`, you use `IO.println` instead of `System.out.println`, and some code no longer strictly has to be in a class. Because the lecture code uses `package lec2_intro2;`, the newest top-level-code relaxations did not apply in the demo. Old 61B material and old Java code will look like the verbose form; both work.

In IntelliJ, when multiple classes have a `main`, the little green arrow next to a given `main` runs that one.

### 8. Any class can use any other class, and imports are only shorthand

`DogInvestigator` uses `Dog` with no import at all. Java automatically scans a list of allowed folders, including the current one, and finds the class. **You do not need to import code from other files in your project.**

Even for built-in library classes, `import` is *not* required. It only lets you use a short name:

```java
java.util.List L = new java.util.ArrayList();  // legal, verbose
```
versus
```java
import java.util.List;
import java.util.ArrayList;
...
List L = new ArrayList();  // same thing, readable
```

The subtle but important contrast: **in Python, importing actually makes something available. In Java, importing only shortens the name.**

### 9. Static vs non-static: "static means there is no `this`"

This is the headline idea of the lecture, and it was stated as a slogan worth memorizing:

> **Static means there is no `this`. And the method must be invoked by calling `Dog.maxDog`.**

Two ways to write "give me the bigger of two dogs":

- **Instance method version**: one dog judges itself against another. `lilDog.maxDog(clifford)`. Inside, `this` is `lilDog` and the parameter is the other dog.
- **Static version**: no specific dog does the judging, the *idea of Dog* does. `Dog.maxDog(ep, milo)`. Inside, there is no `this` at all, so you cannot write `this.size`; IntelliJ will flag it as not making sense in a static context.

The lecture's philosophical gloss: with the static version, you can think of it as "the god of dogs" doing the judging, rather than one dog judging itself against another. Which you prefer here is a pure aesthetic call; neither is more correct.

**Static methods cannot access instance variables**, because there is no instance and therefore no identity.

### 10. Why static methods exist at all

Some classes do not make sense to instantiate. `Math.round` in Java is static for exactly this reason. If it were an instance method, you would have to construct a `Math` object and say something like `m.round(x)`, which is awkward. Instead, `Math` is a class full of utility methods invoked on the class name. That is a case where static genuinely, obviously makes sense.

### 11. Static variables

You can also put a variable on the class itself. All dogs share the scientific name *Canis familiaris*, so:

```java
static String binomen = "Canis familiaris";  // one copy, shared by all Dogs
```

Two pieces of advice given:

1. **Access static variables through the class name** (`Dog.binomen`), not through an instance (`maya.binomen`). Asking Maya for her scientific name is weird: that is not really Maya's property.
2. **Avoid static variables whose values change.** Writing `maya.binomen = "..."` would change it for *all* dogs, which is confusing and leads to tangled code. This will become tempting in Project 5 especially; resist it.

A class can freely mix static and non-static members. That is fine.

### 12. `public` means nothing today

Sometimes the lecture writes `public`, sometimes it does not. For now it does not matter. The one-sentence answer given when pressed: `public` means any class can use this, and omitting it means only classes in the same package can. Lecture 3 says more.

### 13. Lists in Java (warm-up for Lecture 3)

A **list** is an ordered sequence of objects. Independent of any language, lists support operations like appending an item, retrieving item *i*, and removing an item.

Building the Java version live surfaced three obstacles, each instructive:

1. **`List` is not a known name by default.** Its real name is `java.util.List`. Fix: use the full name, or `import java.util.List;`.
2. **You cannot do `new List()`.** `List` is *abstract*: it is an abstract notion of what a list can do, not a concrete thing you can build. (Abstraction gets proper treatment in a couple of weeks.) Fix: pick a concrete implementation, e.g. `new ArrayList()`.
3. Same import shorthand applies to `ArrayList`.

The important design point: **Java has many kinds of list**, and the programmer must choose. `ArrayList`, `LinkedList`, `Stack`, `Vector`, `CopyOnWriteArrayList`, and others are all types of list. In Python you rarely think about this; a list is a list is a list, and there is a concise first-class default syntax for it. Java has no such default.

Why have multiple implementations? Two reasons drawn out from the class:

- **Performance differs.** Removing the front item of a `LinkedList` is fast, even for a billion-element list. Doing the same to an `ArrayList` is slow, because everything after it has to be shifted.
- **Operations differ.** `Stack`, for example, also has `push` and `pop`.

You can peek at the `List` documentation to see the operations lists support (`isEmpty`, `indexOf`, `lastIndexOf`, and so on) and the list of implementations.

Note that `List L = new ArrayList();` and `ArrayList L = new ArrayList();` both work. The lecture deliberately used the first form to start highlighting a principle (declared type vs actual object type) that is central to the first part of the course. This was flagged as Java 4 style lists (circa 2002); Lecture 3 shows how lists are written today.

---

## Definitions

- **Class**: a definition of a new type; a blueprint specifying exactly which properties and behaviors every instance of that type has.
- **Instance / object**: a particular thing created from a class. A `Dog` created by `new Dog(5)` is a Dog object.
- **Instance variable**: a property declared at the top of a class. Every instance gets exactly these properties and only these properties.
- **Constructor**: a special block, named after the class and with no return type, that says how to create instances of the class. It is *not* a method.
- **Instance method**: a method invoked on a particular object; it has a `this` and can read and write that object's instance variables.
- **Static method**: a method invoked on the class itself (`Dog.maxDog(...)`). There is no `this`, so it cannot access instance variables.
- **Static variable**: a variable belonging to the class rather than to any instance, shared by all instances.
- **Method**: a function that is part of a class. In Java, nearly interchangeable with "function", since almost all code lives inside classes.
- **`this`**: a reference to the current object, available inside instance methods and constructors, never inside static methods.
- **Shadowing**: when a local variable or parameter has the same name as an instance variable, so the name inside the method refers to the local one rather than the field.
- **Declaration**: creating a variable and stating its type, with no object yet. `Dog smallDog;` builds the house; no dog lives there yet.
- **Instantiation**: creating an actual object with `new`. `new Dog(20);` makes a dog but does not put it anywhere.
- **Assignment**: putting an instantiated object into a declared variable. The dog moves into the house.
- **Invocation**: calling a method on something, e.g. `hugeDog.makeNoise()`. The dot generally indicates hierarchy: `makeNoise` is part of the `hugeDog` object.
- **`void`**: the return type used when a method returns nothing.
- **`public`**: an access modifier meaning any class can use this member; without it, only classes in the same package can. Treat it as "means nothing" for now.
- **List**: an ordered sequence of objects, supporting operations such as append, retrieve at index, and remove.
- **Abstract (as applied to `List`)**: a notion you cannot instantiate directly; you must choose a concrete implementation such as `ArrayList`.
- **Garbage collection**: when you create an object and nothing uses it, Java eventually notices nobody is using it and reclaims it. (Mentioned in passing; details later.)
- **Overloading**: two methods in the same class sharing a name but differing in parameters, as with the two `maxDog` methods. (The instructor called this overloading with a hedge about the static/non-static pair; see Pitfalls.)

---

## Worked Examples

### Example 1: The complete `Dog` class

```java
package lec2_intro2;

class Dog {
    // let's list off the properties of a Dog
    int size;

    // this is not a method
    // this is a constructor
    // it says how to make Dogs
    Dog(int s) {
        size = s;
    }

    void makeNoise() {
        if (this.size < 10) {
            IO.println("yipyipyippyip");
        } else if (this.size < 30) {
            IO.println("bark");
        } else {
            IO.println("aroooooooooooo");
        }
    }

    static void main() {
        Dog d = new Dog(5);
        d.makeNoise();
    }
    ...
}
```

Step by step through `main()`:

1. `Dog d` **declares** a variable `d` of declared type `Dog`. At this moment there is no Dog object at all: just a labeled box that is allowed to hold a Dog.
2. `new Dog(5)` **instantiates**. Java allocates a new Dog object with an instance variable `size`. Control enters the constructor `Dog(int s)` with `s` bound to `5`. The line `size = s` writes `5` into the new object's `size` field. The constructor finishes and the new object is handed back.
3. `d = ...` **assigns**: `d` now refers to that object. In box-and-pointer terms, the box labeled `d` holds an arrow pointing at the Dog object, and the Dog object contains a box labeled `size` holding `5`. `d` does not contain the dog; it points at it.
4. `d.makeNoise()` **invokes** the instance method on that object. Inside, `this` is the object `d` points to, so `this.size` is `5`. `5 < 10` is true, so it prints `yipyipyippyip`.

Change the `5` to a larger number and the branch changes: `20` prints `bark`, `1000` prints `aroooooooooooo`.

The `package lec2_intro2;` line at the top exists only because the course code folder has many subfolders (`lec1_intro1`, `lec2_intro2`, ...) and IntelliJ complains otherwise. Do not read anything into it.

Note the `static` on `main`. The lecture ran into a live-coding hiccup where `void main()` would not run in this context, and added `static` because it is known to work. Under Java 25 both forms are generally acceptable; the interaction with packages was what got in the way.

### Example 2: Declaration, instantiation, assignment, invocation, separated out

Reconstructing the terminology slide from the lecture narration:

```java
Dog smallDog;              // declaration only: a house, no dog
new Dog(20);               // instantiation only: a dog with nowhere to live
smallDog = new Dog(5);     // instantiation AND assignment: dog moves in
Dog hugeDog = new Dog(150); // declaration, instantiation, AND assignment, all in one
hugeDog.makeNoise();       // invocation
```

Line 2 is pedagogy only: a Dog is created, nothing refers to it, and Java's garbage collector eventually reclaims it. You would not write that in real code.

Line 4 is what you will write most of the time: declare, instantiate, and assign on a single line.

### Example 3: `maxDog` as an instance method

```java
// return the larger of the dogs
public Dog maxDog(Dog otherDog) {
    if (otherDog.size > this.size) {
        return otherDog;
    }
    return this; // the only choice here is this
}
```

Used from another class entirely:

```java
package lec2_intro2;

class DogInvestigator {
    void main() {
        Dog lilDog = new Dog(3);
        Dog clifford = new Dog(1000);

        Dog bigger = lilDog.maxDog(clifford);
        bigger.makeNoise();
    }
}
```

Walking through:

1. `lilDog` points at a Dog with `size == 3`. `clifford` points at a Dog with `size == 1000` (named after Clifford the Big Red Dog).
2. `lilDog.maxDog(clifford)`: we are asking `lilDog` to judge itself against another dog. Inside the method, `this` is the `lilDog` object and `otherDog` is the `clifford` object. These are two arrows pointing at two distinct Dog objects.
3. `otherDog.size > this.size` is `1000 > 3`, true, so we `return otherDog`, i.e. a reference to Clifford.
4. `bigger` is assigned that reference. Now `bigger` and `clifford` are two variables pointing at **the same object**, not copies.
5. `bigger.makeNoise()` prints `aroooooooooooo`, since `1000 >= 30`.

Two details worth noticing:

- `return this;` in the fall-through branch is mandatory in spirit: you cannot write a bare `return;` and expect Java to infer "yourself". You must name the thing you are returning, and "the only choice here is `this`".
- The `this.` in `this.size` is *not* required (plain `size` works, since the only size you could mean is your own). It is written for readability.
- Note there is no import in `DogInvestigator` for `Dog`: same package/folder, so Java just finds it.

### Example 4: `maxDog` as a static method

```java
// static means there is no 'this'
// and the method must invoked
// by calling Dog.maxDog
static Dog maxDog(Dog ep, Dog milo) {
    if (ep.size > milo.size) {
        return ep;
    }
    return milo;
}
```

Invoked as `Dog.maxDog(someDog, someOtherDog)`.

Reasoning through it:

- No specific dog is doing the judging. The *class* is. So there is no `this` in scope, and writing `this.size` inside this method is a compile error: "this doesn't make sense in a static context". Likewise you could never print `this.size` here, because there isn't a dog.
- Both dogs must come in as parameters, since neither is the receiver. Both `ep` and `milo` are references to Dog objects; comparing `ep.size > milo.size` reads the `size` field out of each object.
- Ties go to `milo` (the `>` is strict, so equal sizes fall through to `return milo`).

The two `maxDog` methods coexist in the same class, distinguished by their parameter lists (one Dog vs two Dogs).

### Example 5: Static vs instance invocation (the attendance question)

The in-class question used a `Human` class with a static method `ponder` and a non-static method `consider`, asking which invocations are appropriate. Reconstructing the reasoning given:

| Invocation | Verdict |
|---|---|
| `Human.consider()` | **Does not work.** `consider` is non-static; there is no instance for it to act on, so this will not compile. |
| `h.ponder()` (instance calling a static method) | **Compiles, but confusing.** You are asking a specific instance to call `ponder`, but no specific instance is doing the pondering. The lecture argued this arguably *should* have been a compile error. Avoid it. |
| `Human.ponder()` | **Fine.** Static method invoked on the class. |
| `h.consider()` | **Fine.** Instance method invoked on an instance. |

The rule to carry away: **invoke static members on the class name, instance members on an instance.**

### Example 6: The list demo, built up from broken to working

The final working code:

```java
package lec2_intro2;
// by using import, we can use the shorter name
// but importing isn't necessary to use a list
// you could jsut java.util.List;
// UNLIKE PYTHON where importing actually makes something available
import java.util.List;
import java.util.ArrayList;

public class ListDemo {
    void main() {
        List L = new ArrayList();
        L.add(0);
        L.add(1);
        L.add(2);
        IO.println(L);
    }
}
```

The path taken to get there:

1. Start with `List L = new List();` plus `L.add(...)` and a print. IntelliJ: **cannot resolve symbol `List`**. Java does not know the bare name `List`.
2. Spell it out: `java.util.List L = new java.util.List();`. Closer, but ugly, and there is still an error.
3. Add `import java.util.List;` so the short name `List` works. This is pure shorthand: it does not "make `List` available", it makes the name shorter.
4. Remaining error: **you cannot make a `List`**. `List` is abstract: an abstract notion of a list, not a concrete one. Choose a concrete implementation: `new java.util.ArrayList()`, then add `import java.util.ArrayList;` to shorten that too.
5. Run. Output is the list contents, `[0, 1, 2]`.

The Python counterpart is `L = []` followed by `L.append(...)`, with no import, no `new`, no type declaration, and no choice of implementation. The Java version forces three decisions Python hides: the declared type, the concrete implementation, and the `new`.

Also note: writing `ArrayList L = new ArrayList();` would work perfectly well here. The lecture chose `List L = new ArrayList();` deliberately, to begin separating "what type the variable is declared as" from "what kind of object is actually in it". That distinction is a pedagogical throughline for the first part of the course.

---

## Common Pitfalls

1. **Thinking a constructor is a method.** It is not. It has no return type, its name must match the class, and its job is to say how instances get made.

2. **Writing `self` in Java.** Java never passes the receiver explicitly. Do not put `self` (or `this`) in your parameter list.

3. **`size = size` in a constructor.** If the parameter shares the instance variable's name, this silently does nothing: it assigns the local to itself. Use `this.size = size;`, or avoid the collision by naming the parameter something else.

4. **Forgetting `new`.** In Python, `d = Dog(5)`. In Java you must say `new Dog(5)`. Making a dog is an explicit act.

5. **Omitting the declared type.** Every Java variable must have a declared type. `myDog = new Dog(20);` with no type is not legal Java. Even where it "seems obvious", Java semantics require the formal declaration. (The lecture mentioned there is another thing you can type there but declined to teach it: *(extra context)* this is `var`, local type inference. This class wants explicit types.)

6. **Trying to add a property that is not in the blueprint.** A `Dog` with no `name` field can never have a name. This does not blow up at runtime, it fails to compile.

7. **Using `this` inside a static method.** There is no `this` in a static context. Any reference to instance variables from a static method is a compile error.

8. **Calling a static method through an instance.** `h.ponder()` compiles but is misleading; use `Human.ponder()`. Correspondingly, calling an instance method on the class name (`Human.consider()`) simply does not work.

9. **Mutating static variables through an instance.** `maya.binomen = "..."` changes it for every dog. Access statics via the class name, and avoid static variables whose values change.

10. **Assuming Java `import` behaves like Python `import`.** It does not. Java imports are name shorthand only; the class was reachable all along under its full name. And for classes in your own project/folder, no import is needed at all.

11. **`new List()`.** `List` is abstract. Instantiate a concrete implementation like `ArrayList`.

12. **Putting code outside a class.** Java runs your program by finding `main`. Loose top-level code is orphaned. (Java 25 relaxed some of this, but not in the packaged context used in lecture.)

13. **Creating an object and never storing it.** `new Dog(20);` on its own line makes a dog nothing refers to; it gets garbage collected. Harmless, pointless.

14. **Over-trusting the "overloading" label on the two `maxDog` methods.** The instructor answered "yes, I guess so" with a self-declared 2% chance of being wrong about the static/non-static pair specifically. Know the concept; do not build an exam answer on that hedge alone.

---

## Likely Exam Points

### 1. Static vs instance: which invocations compile?

**Practice.** Given `class Human { static void ponder() {...} void consider() {...} }` and `Human h = new Human();`, classify each: (a) `Human.consider();` (b) `h.ponder();` (c) `Human.ponder();` (d) `h.consider();`

**Answer.** (a) does not compile: an instance method needs an instance. (b) compiles but is poor style and arguably should be an error, since no particular instance is pondering. (c) correct. (d) correct.

### 2. "Static means there is no `this`"

**Practice.** Why does the following fail to compile?
```java
static Dog maxDog(Dog ep, Dog milo) {
    if (ep.size > this.size) { return ep; }
    return milo;
}
```

**Answer.** `maxDog` is static, so it is invoked on the class, not on any particular Dog. There is no receiving object, hence no `this`, hence no instance variables to reach. Compare against `milo.size` instead.

### 3. Shadowing in a constructor

**Practice.** What is `d.size` after `Dog d = new Dog(7);` if the constructor is `Dog(int size) { size = size; }`?

**Answer.** `0`. The parameter `size` shadows the instance variable, so `size = size` assigns the parameter to itself and the field is never written; `int` fields default to `0`. Fix with `this.size = size;`.

### 4. Terminology: declaration / instantiation / assignment / invocation

**Practice.** Label each line: `Dog a;` / `new Dog(20);` / `a = new Dog(5);` / `Dog b = new Dog(150);` / `b.makeNoise();`

**Answer.** Declaration; instantiation; instantiation plus assignment; declaration plus instantiation plus assignment; invocation.

### 5. Instance-method `maxDog` tracing

**Practice.** With `Dog lilDog = new Dog(3); Dog clifford = new Dog(1000); Dog bigger = lilDog.maxDog(clifford); bigger.makeNoise();`, what prints, and what does `bigger` refer to?

**Answer.** Prints `aroooooooooooo`. Inside `maxDog`, `this` is `lilDog` (size 3) and `otherDog` is `clifford` (size 1000); `1000 > 3` so `otherDog` is returned. `bigger` and `clifford` now point at the same object (no copy is made), and since `1000 >= 30` the `else` branch of `makeNoise` runs.

### 6. `makeNoise` branch selection

**Practice.** What does each print: `new Dog(5).makeNoise()`, `new Dog(10).makeNoise()`, `new Dog(30).makeNoise()`?

**Answer.** `yipyipyippyip` (5 < 10); `bark` (10 is not < 10, but is < 30); `aroooooooooooo` (30 is not < 30). Watch the strict inequalities at the boundaries.

### 7. Java imports vs Python imports

**Practice.** True or false: without `import java.util.List;`, you cannot use a `List` in Java.

**Answer.** False. You can write `java.util.List L = new java.util.ArrayList();`. The import only lets you use the shorter name. This is unlike Python, where importing actually makes something available.

### 8. Why `List` cannot be instantiated

**Practice.** Why does `List L = new List();` fail, and what is the fix?

**Answer.** `List` is abstract: it describes what a list can do, without being a concrete list. Instantiate an implementation, e.g. `List L = new ArrayList();`.

### 9. Why Java has many list implementations

**Practice.** Give two reasons Java offers `ArrayList`, `LinkedList`, `Stack`, and others rather than one list type.

**Answer.** (1) Performance: removing the front element of a `LinkedList` is fast regardless of length, while an `ArrayList` must shift everything; memory use also differs. (2) Extra operations: `Stack` adds `push` and `pop`.

### 10. Static variables and good style

**Practice.** `Dog` has `static String binomen = "Canis familiaris";`. What is wrong with `maya.binomen = "Vulpes vulpes";`?

**Answer.** Two things. It accesses a static member through an instance, which obscures that the variable belongs to the class; and because there is one shared copy, it changes the scientific name for *every* Dog. Prefer `Dog.binomen`, and avoid mutable static variables generally.

### 11. What is fixed vs changeable about an object

**Practice.** After `Dog d = new Dog(150);`, can you do `d.size = 5;`? Can you do `d.name = "Maya";`?

**Answer.** `d.size = 5;` is fine: values of declared fields can change. `d.name = "Maya";` will not compile if `Dog` declares no `name` field: the list of properties is fixed by the class and cannot be extended at runtime, unlike Python.

---

## Summary

- A **class** defines a type and acts as a **blueprint**: every instance has exactly the declared instance variables and exactly the declared methods, no more.
- **Instance variables** are declared at the top of the class. The list is fixed at compile time; you can change values, never add properties. Unused fields default (e.g. `int` to `0`).
- A **constructor** (class name, no return type, no `self`) says how to make instances. `Dog(int s) { size = s; }`.
- `this` refers to the current object and is usually optional; it becomes mandatory when a parameter **shadows** a field (`this.size = size;`) or when returning the receiver (`return this;`).
- **Instance methods** act on a specific object and can read its fields; invoke as `d.makeNoise()`.
- **Static means there is no `this`.** Static methods are invoked on the class (`Dog.maxDog(a, b)`), take everything they need as parameters, and cannot touch instance variables.
- Static exists because some classes should not be instantiated: `Math.round` is the canonical example. In `Dog`, instance vs static `maxDog` is an aesthetic choice.
- **Static variables** are shared by all instances; access them via the class name and avoid mutating them.
- Vocabulary: **declaration** (a house), **instantiation** (`new`, a dog), **assignment** (dog moves in), **invocation** (`obj.method()`).
- Java requires a **declared type** on every variable, requires `new`, runs by finding `main`, and rejects loose top-level code.
- `public` does not matter yet (it means "any class can use this"; without it, same package only).
- Classes in your own project need **no import**. Java `import` only shortens names, unlike Python where import makes something available.
- `List` is **abstract**: use `List L = new ArrayList();`. Java deliberately has many list implementations, differing in performance, memory, and available operations.
- Unreferenced objects are reclaimed by the **garbage collector**.
- Next time (Lecture 3): modern Java lists, field defaults, more on `public`, and starting to build our own list.
