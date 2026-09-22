<!-- Fri, Aug 28, 2026 | sources: slides + code + YouTube auto-transcript + your recording + textbook -->
# Lecture 2: Defining and Using Classes, Lists

## Overview

This lecture is the transition from "I know how to write a program in Python" to "I know how to write a program in Java." It covers how to define your own type in Java by writing a class: declaring instance variables, writing a constructor, and writing instance methods, along with the terminology (declaration, instantiation, assignment, invocation, member, object) that the course will use for the rest of the semester. It then draws the central distinction of the day: static (class) members versus non-static (instance) members, where the one-sentence summary Josh gives is "static means there is no `this`." Along the way it clears up two pieces of Java ceremony: the `public` keyword (which, for now, makes no difference) and the old `public static void main(String[] args)` incantation, which Java 25 replaced with a plain `void main()` plus `IO.println`. The last third of the lecture writes the Java equivalent of a four-line Python list program, discovering along the way that `List` needs an import, that `List` is abstract and cannot be instantiated, and that you must choose a concrete implementation such as `ArrayList` or `LinkedList`. That discovery motivates the final idea: the difference between an abstract data type (`List`) and a concrete implementation (`ArrayList`, `LinkedList`, `Stack`, ...), and why multiple implementations of the same idea exist at all (performance and extra operations).

---

## Key Concepts

### 1. Java classes are Java's way of letting you define your own type

Just like Python, Java lets you define new types. The syntax is different, but the ideas line up almost one to one:

| Python | Java |
|---|---|
| `class Dog():` | `class Dog { ... }` |
| `def __init__(self, size):` | `Dog(int s) { ... }` (a constructor) |
| `def make_noise(self):` | `void makeNoise() { ... }` (an instance method) |
| attributes created on the fly via `self.size = size` | instance variables declared at the top of the class: `int size;` |
| `d = Dog()` | `Dog d = new Dog(...);` |
| `d.make_noise()` | `d.makeNoise();` |

The differences Josh drew out in class, from the students' own observations:

- **`new`**: Java requires the `new` keyword to create an object. Python does not.
- **No `self`**: Java never passes the receiving object in explicitly. Inside an instance method, the current object is implicitly available and can be named `this`, but you do not declare it as a parameter and usually do not need to write it.
- **Instance variables must be declared, up front, with types**: a Java class has a designated region at the top where every property of the object is listed. That list is fixed. You cannot attach a new field to an object at runtime the way Python lets you. Java is a strict language: the list of properties is fixed, types cannot change, and everything must be declared before use.
- **All code lives inside a method, and a program starts at `main`**: you cannot have loose statements sitting at the top level of a file the way Python does. When you run a Java program, Java looks for a method called `main` and runs it. Code outside of any method is, in Josh's words, orphaned, with no place to live.

### 2. A class is a blueprint; instances are objects

`Dog.java` is a blueprint that describes what all `Dog` objects are like. Every `Dog` object will have exactly one `int` called `size`, and will be able to `makeNoise()`, and will be able to do nothing else and have nothing else. So:

```java
Dog hugeDog = new Dog(150);
hugeDog.size = 5;        // fine, size is guaranteed to exist (and can be changed)
hugeDog.name = "frank";  // will NOT compile, name is not in the blueprint
```

Notice the failure mode: not a runtime crash, but a refusal to compile. As Josh put it, the wild thing about Java is that you avoid the trouble by not even letting the code run in the first place.

A related question from class: what if you declare an instance variable and never use it? It compiles and runs fine, and it defaults to zero (for an `int`). Default values get more attention in lecture 3.

### 3. Constructors are not methods

The top-of-class variable declarations say *what a Dog has*. The constructor says *how to build one*:

```java
Dog(int s) {
    size = s;
}
```

A constructor looks like a method (it has parameters, it has a body, it runs code) but it is not one: it has no return type, its name is the class name, and it exists to determine how the class is instantiated. The mental picture from lecture: the dog is coming down the assembly line, a `size` slot appears (initially undefined/zero), and the value passed in as `s` gets slotted in. It is the Java analogue of Python's `__init__`.

**Shadowing (raised by a student):** if you write the constructor as `Dog(int size) { size = size; }`, the parameter shadows the instance variable, and the line does nothing useful: it takes the parameter and assigns it to itself. In environment-diagram terms, there are two separate boxes, the local `size` (the parameter) and the object's `this.size`, and the plain name `size` refers to the local one. The fix is `this.size = size;`, which explicitly names the object's variable on the left. Lecture avoids the whole problem by naming the parameter `s`.

### 4. `this`

Inside an instance method or constructor, `this` refers to the current object, the specific dog whose method was invoked. From the `maxDog` example on the slides:

```java
Dog maxDog(Dog otherDog) {
    if (otherDog.size > this.size) {   // this. here is OPTIONAL
        return otherDog;
    }
    return this;                        // this here is REQUIRED
}
```

The distinction that the slides call out and Josh emphasized live: `this.size` can be shortened to `size`, because in that context the only `size` you could be talking about is your own. But `return this;` cannot be shortened. There is no bare `return;` that Java will interpret as "return myself"; you need a name for the current object, and that name is `this`.

### 5. Static versus instance members

This is the core conceptual content of the lecture.

- An **instance method** is an action taken by a *specific* object. It is invoked on an instance: `maya.makeNoise()`. It can see that instance's variables, because there is a "me."
- A **static method** (also called a class method) is an action taken by *the class itself*. It is invoked using the class name: `Dog.maxDog(d1, d2)`. There is no `this` in a static context, and therefore it cannot access instance variables directly. It can only reach instance variables through a specific instance handed to it, e.g. `d1.size`.

The lecture demonstrated this by writing `maxDog` twice, once each way:

- **Instance version:** `lilDog.maxDog(clifford)`, where one dog judges itself against another.
- **Static version:** `Dog.maxDog(lilDog, clifford)`, where, as Josh framed it, "the god of dogs" does the judging impartially, and no specific dog is involved.

For `Dog`, choosing between them is described in the lecture as a purely aesthetic decision: maybe you think an instance should do the comparing, maybe you think the class should. Both are fine.

**Why static methods exist at all:** some classes are never meaningfully instantiated. `Math` is the canonical example. Because `Math.round` and `Math.sqrt` are static, you write

```java
x = Math.round(5.6);
```

instead of the awkward

```java
Math m = new Math();
x = m.round(5.6);
```

`Math` is a bag of utility methods; there is nothing that a "Math object" would mean. That is where static really makes sense.

### 6. Static variables (and why to be careful with them)

A class can also have static variables, properties inherent to the class rather than to any one instance:

```java
class Dog {
    int size;
    static String binomen = "Canis familiaris";
    ...
}
```

Rules and advice from lecture:

- Always access static variables through the class name: `Dog.binomen`, not `maya.binomen`. Java technically permits the instance form, but the textbook calls it bad style, confusing, and "in my opinion an error by the Java designers."
- Even worse: `maya.binomen = "Vulpes vulpes";`. That is not setting Maya's scientific name, it is changing the scientific name for *every* dog, and the syntax actively hides that.
- **Strong recommendation: avoid static variables whose values change.** It becomes hard to keep track of which parts of your program read from and write to the shared variable, which leads to complicated code. Josh noted this becomes especially tempting around Project 5.

### 7. `public`

You will constantly see `public` in front of classes, variables, constructors, and methods in real-world Java. For this lecture it makes **no difference** whatsoever whether you include it; Josh mixed it in and out deliberately so students would see it does not matter yet. (Lecture 3 covers it properly.) The gloss he gave when asked: `public` means any class can use this thing, whereas omitting it means only classes in the same package can.

The lecture code file even ends with the comment: `// public means nothign today / sometimes I'm doing it, sometimes I'm not / sorry`.

### 8. Java before and after September 2025

Java 25 (September 2025) simplified the ceremony:

```java
// Pre-Java 25
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("hello world");
    }
}
```

```java
// Modern Java (25+), what this course uses
void main() {
    IO.println("hello world");
}
```

Three changes: all code no longer must be inside a class, `IO.println` replaces `System.out.println`, and `void main()` replaces `public static void main(String[] args)`. Many older 61B resources (including Fall 2025 materials) use the old style, so expect to see it and do not be surprised.

Decoding the old incantation, piece by piece:
- `public`: usable by any class.
- `static`: not associated with any instance, so no instantiation needed.
- `void`: returns nothing.
- `main`: the name Java looks for to start your program.
- `String[] args`: an array of strings supplied by the operating system, i.e. command line arguments. If you have a `Morph.java` and run `java Morph joshhug.jpg manuelsabin.jpg`, those two strings arrive in `args`. Declared beyond the scope of this class.

### 9. No imports needed between your own `.java` files

Unlike Python, you do not import code from other `.java` files in your own project. Java automatically scans a list of folders (including the current one) to see whether the desired class exists. So `DogInvestigator.java` can just say `new Dog(3)` with no import line at all.

### 10. Lists in Java (old-school style)

The Python program being translated:

```python
L = []
L.append("a")
L.append("b")
L.append("c")
print(L)     # ['a', 'b', 'c']
```

The lecture built the Java version by hitting each error in turn:

1. **`List L = new List();`** → "can't resolve symbol List." Its actual full name is `java.util.List`.
2. **`java.util.List L = new java.util.List();`** → resolves the name, but is ugly. You can instead write `import java.util.List;` at the top and then use the short name.
   - Crucially: **importing in Java is different from Python.** In Python, importing is what makes something available. In Java, importing only *shortens the name*. You could skip every import and write `java.util.List` and `java.util.ArrayList` in full everywhere, and it would work identically.
3. **`List L = new List();`** (with the import) → "List is abstract, cannot be instantiated." A `List` is an abstract notion; you must pick a *specific kind* of list.
4. **`List L = new ArrayList();`** (with both imports) → compiles.

Final program (the slide version; the lecture code file used integers and `IO.println`):

```java
import java.util.ArrayList;
import java.util.List;

void main() {
    List L = new ArrayList();
    L.add("a");
    L.add("b");
    L.add("c");
    System.out.println(L);   // [a, b, c]
}
```

Two things to notice: Java's append operation is called **`add`**, not `append`; and this code is deliberately written in a very old-school style (circa Java 4.0 / 5.0, roughly 2002-2004). Lecture 3 modernizes it (e.g. `List<String> L`).

You could write `ArrayList L = new ArrayList();` and it would work fine. Josh wrote `List L = new ArrayList();` on purpose, to set up a principle that will matter enormously in the coming weeks.

### 11. Abstract data types versus concrete implementations

In Python, a list is a list is a list; there is no syntactic distinction between the abstract idea of a list and an actual implementation, and there is a first-class default (`[]`). In Java there are many kinds of list, and the programmer must explicitly say which one they want.

- **`java.util.List` is an Abstract Data Type (ADT).** It is a guarantee: any `List` has at least the operations documented at the `List` API page (`add`, `get`, `isEmpty`, `indexOf`, `lastIndexOf`, ...).
- **`ArrayList`, `LinkedList`, `Stack`, `Vector`, `CopyOnWriteArrayList`, `RoleList`, `AttributeList`, ... are Concrete Implementations.** Their internal code may be radically different, but from above the abstraction boundary (that is, from the user's perspective) they all provide at least what `List` guarantees.

**Why bother having more than one implementation?** The lecture's two answers, arrived at from student suggestions about runtime and memory:

1. **Performance.** Different implementations are fast at different things. `LinkedList` removes its front item very quickly, even in a billion-element list. `ArrayList` is slow at this, because under the hood everything gets scooted over.
2. **Extra operations.** Some implementations offer more than the base guarantee, e.g. `Stack` adds `push` and `pop`.

The most common list in practice is `ArrayList`; `LinkedList` shows up sometimes. This distinction is the seed of a major theme of the first part of the course.

---

## Definitions

- **Class**: a blueprint describing a type: the variables its instances have and the methods they can run. Instances must obey the blueprint exactly.
- **Object**: an instance of any class. Created with `new`.
- **Instance variable (non-static variable)**: a variable declared inside a class, outside any method, which every instance of the class gets its own copy of. Must be declared in the class; cannot be added at runtime.
- **Static variable (class variable)**: a variable declared `static` inside a class; a property of the class itself rather than of any instance. Shared by all instances. Should be accessed via the class name.
- **Constructor**: a class-name-shaped, return-type-free block that determines how to instantiate the class, e.g. `Dog(int s) { size = s; }`. Similar to a method, but it is not a method. Analogous to Python's `__init__`.
- **Method**: a function that is part of a class. In Java, nearly all functions are inside classes, so "method" and "function" are nearly interchangeable.
- **Instance method (non-static method)**: a method invoked on a specific object, e.g. `maya.makeNoise()`. Has access to `this` and to instance variables.
- **Static method (class method)**: a method declared `static`, invoked using the class name, e.g. `Dog.maxDog(d1, d2)`. There is no `this`; it cannot access instance variables except via a specific instance passed to it.
- **`this`**: keyword referring to the current object inside an instance method or constructor. Does not exist in a static context.
- **Member**: any variable or method of a class. Accessed with dot notation.
- **Dot notation**: the `x.y` syntax meaning "the member `y` belonging to `x`"; indicates a hierarchy.
- **Declaration**: creating a variable of a given type without necessarily giving it a value, e.g. `Dog smallDog;`. The house, with no dog in it yet.
- **Instantiation**: creating an object, e.g. `new Dog(20);`. The dog, with no house.
- **Assignment**: putting an instantiated object into a declared variable, e.g. `smallDog = new Dog(5);`. Putting the dog in the house.
- **Invocation**: calling a method on an object or class, e.g. `hugeDog.makeNoise()`.
- **Client**: a class that uses another class. `DogInvestigator` is a client of `Dog`.
- **Shadowing**: when a local variable or parameter has the same name as an instance variable, so the plain name refers to the local one, hiding the instance variable.
- **List**: an ordered sequence of objects, often written as comma-separated values in brackets, e.g. `[3, 6, 9, 12, 15]`, supporting operations such as appending, retrieving by index, and removing by index or value.
- **Abstract Data Type (ADT)**: a specification of a set of guaranteed operations, without commitment to how they are implemented. `java.util.List` is an ADT.
- **Concrete Implementation**: an actual class implementing an ADT, e.g. `ArrayList`, `LinkedList`, `Stack`. Provides at least the ADT's guaranteed operations, plus possibly more.
- **`public`**: a keyword meaning the thing can be used by any class. For this lecture, including it or not makes no difference. (Full treatment in lecture 3.)
- **Overloading**: (mentioned in response to a student question about the two `maxDog` methods with the same name but different parameter lists) having multiple methods with the same name distinguished by their parameters. Josh accepted this term with a stated ~2% uncertainty about whether a static/non-static pair has a special name.

---

## Worked Examples

### Example 1: The `Dog` class, built up from nothing

This is the file written live in lecture (`Dog.java`), with the lecture's own comments:

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

Step by step:

1. `int size;` declares the single instance variable. Every `Dog` object will have exactly one `int` named `size`, no more and no less.
2. `Dog(int s) { size = s; }` is the constructor. When `new Dog(5)` runs, Java allocates a fresh `Dog` object whose `size` starts at the default `0`, runs the constructor body with `s` bound to `5`, and the assignment writes `5` into the object's `size` slot.
3. `void makeNoise()` is an instance method: no `static`, so it runs on behalf of a specific dog. Inside, `this.size` is that specific dog's size. The `this.` prefix here is optional; the code would behave identically with plain `size`.
4. `main` creates a dog of size 5 and invokes `makeNoise()` on it. Since `5 < 10`, it prints `yipyipyippyip`. With `new Dog(1000)` instead, we fall through to the `else` branch and get `aroooooooooooo`.

(Note on the `static` in `static void main()`: in lecture, IntelliJ refused to run a plain `void main()` inside a class that was in a package, so Josh added `static` to make it go. He explicitly told students not to worry about it; the modern `void main()` form is what the course uses, and `DogInvestigator`'s plain `void main()` worked fine.)

**Box-and-pointer / environment reasoning in words:** after `Dog d = new Dog(5);` there is a variable `d` in the frame for `main`. `d` does not contain a dog; it contains a reference (an arrow) pointing to a `Dog` object sitting out in memory. That object has one labeled box inside it, `size`, containing `5`. When you invoke `d.makeNoise()`, Java follows the arrow to the object, and inside the method `this` is a reference pointing at that same object, so `this.size` reads the `5`.

### Example 2: Declaration, instantiation, assignment, invocation (the terminology slide)

```java
void main() {
    Dog smallDog;                  // Declaration of a Dog variable
    new Dog(20);                   // Instantiation of a Dog object (and nothing else)
    smallDog = new Dog(5);         // Instantiation and Assignment
    Dog hugeDog = new Dog(150);    // Declaration, Instantiation, and Assignment
    smallDog.makeNoise();
    hugeDog.makeNoise();           // Invocation of the 150 lb Dog's makeNoise method
}
```

Line by line:

1. `Dog smallDog;` builds the house but no dog lives in it yet. There is a variable, of type `Dog`, holding nothing useful.
2. `new Dog(20);` builds a dog with no house. The object is created but no variable refers to it, so nothing can ever reach it again. (Student question: what is the point? Answer: pedagogy, you would not actually write this. Follow-up: what happens to that dog? It becomes garbage and is eventually reclaimed by the garbage collector. More on that much later in the course.)
3. `smallDog = new Dog(5);` does both: creates a dog and puts it in the house.
4. `Dog hugeDog = new Dog(150);` is the all-in-one form you will write nearly all the time.
5. `hugeDog.makeNoise()` is invocation. The dot means "a member of `hugeDog`."

Output: `smallDog` has size 5, so `yipyipyip!`; `hugeDog` has size 150, so `woof!` (or the lecture's `aroooooooooooo`).

### Example 3: A client class (`DogInvestigator`)

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

The `main` method does not have to live in `Dog`. A separate class can use `Dog` perfectly well, and that separate class is called a **client** of `Dog`. Note there is **no import**: Java finds `Dog` by scanning the allowed folders. (IntelliJ detail from lecture: each class with a `main` gets a little green run arrow, and whichever you click determines which `main` runs.)

### Example 4: `maxDog` as an instance method

```java
// return the larger of the dogs
public Dog maxDog(Dog otherDog) {
    if (otherDog.size > this.size) {
        return otherDog;
    }
    return this; // the only choice here is this
}
```

Trace of `lilDog.maxDog(clifford)`:

1. `this` points at the `lilDog` object (`size` 3). `otherDog` points at the `clifford` object (`size` 1000).
2. The condition `1000 > 3` is true, so we `return otherDog`, i.e. a reference to Clifford.
3. `bigger` now points to the exact same object as `clifford`. No copying happened; both names refer to one dog.
4. `bigger.makeNoise()` sees `size` 1000, which is not `< 10` and not `< 30`, so it prints the sonorous `aroooooooooooo`.

Note the asymmetry the lecture stressed: `this.size` could be written as just `size`, but `return this;` genuinely requires the keyword.

### Example 5: `maxDog` as a static method

```java
// static means there is no 'this'
// and the method must be invoked
// by calling Dog.maxDog
static Dog maxDog(Dog ep, Dog milo) {
    if (ep.size > milo.size) {
        return ep;
    }
    return milo;
}
```

Invoked as:

```java
Dog bigger = Dog.maxDog(lilDog, clifford);
bigger.makeNoise();
```

What changed and why:

- No `this` exists in this method. Writing `this.size` here is a compile error; IntelliJ reports that `this` does not make sense in a static context. Nor could you write `IO.println(this.size)`, because there is no dog doing it.
- Both dogs must be passed in explicitly as parameters, and both are reached by name: `ep.size`, `milo.size`. Static methods can still touch instance variables, but only through a specific instance handed to them.
- Invocation uses the class name, `Dog.maxDog(...)`, not an instance name.

Result is the same: `1000 > 3` is false in the `ep.size > milo.size` test when called as `Dog.maxDog(lilDog, clifford)` (since `ep` is `lilDog` with size 3 and `milo` is `clifford` with size 1000), so it returns `milo`, i.e. Clifford. Same output.

### Example 6: Mixing static and non-static members

```java
class Dog {
    int size;                                       // instance variable
    static String binomen = "Canis familiaris";     // static variable

    Dog(int s) {
        size = s;
    }

    static Dog maxDog(Dog d1, Dog d2) {             // static method
        if (d1.size > d2.size) { return d1; }
        return d2;
    }

    void makeNoise() {                              // instance method
        if (size < 10) {
            System.out.println("yipyipyip!");
        } else if (size < 30) {
            System.out.println("bark. bark.");
        } else {
            System.out.println("woof!");
        }
    }
}
```

Access rules on display here:
- `Dog.binomen` is correct; `maya.binomen` is legal but bad style; `maya.binomen = "Vulpes vulpes"` is worse still, because it silently changes the binomen for all dogs.
- `Dog.makeNoise()` does not work: a non-static member cannot be invoked using the class name.
- Inside `maxDog`, there is no bare `size`; you must go through `d1` or `d2`.

### Example 7: Building the list program by fixing errors

```java
package lec2_intro2;
// by using import, we can use the shorter name
// but importing isn't necessary to use a list
// you could just java.util.List;
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

The path to this code, error by error:

1. `List L = new List();` with no import → **"can't resolve symbol List."** Java does not know the short name `List` by default.
2. `java.util.List L = new java.util.List();` → the name now resolves (so we get past that error) but is verbose, and there is still a compilation error waiting.
3. Add `import java.util.List;` → the short name works. Remember: the import did **not** make `List` available; it was always available under its full name. The import only lets you abbreviate.
4. `List L = new List();` → **"List is abstract, cannot be instantiated."** You must pick a concrete kind of list.
5. `List L = new ArrayList();` (plus `import java.util.ArrayList;`) → compiles and runs, printing `[0, 1, 2]`.

Note the declared type is `List` while the created object is an `ArrayList`. Writing `ArrayList L = new ArrayList();` would also work; the `List` form is a deliberate setup for the abstraction-boundary theme coming later. Swapping in `new LinkedList()` on the right-hand side would also work, and the rest of the code would not change at all, which is exactly the point of an ADT.

---

## Common Pitfalls

1. **Forgetting `new`.** `Dog d = Dog(5);` is a Python habit. Java needs `Dog d = new Dog(5);`.
2. **Trying to add a field that is not in the blueprint.** `hugeDog.name = "frank";` will not compile. Java's property list is fixed at class-definition time.
3. **Forgetting to declare a type.** A student asked whether you could write `d = new Dog(5);` without saying `Dog`. You cannot: every variable in Java has a declared type, and in this class you always write it explicitly. (Josh mentioned there is another thing you can type instead, which he will not teach.)
4. **Shadowing in a constructor.** `Dog(int size) { size = size; }` compiles but does nothing useful. Either rename the parameter (`int s`) or write `this.size = size;`.
5. **Calling an instance method via the class name.** `Dog.makeNoise()` or `Human.consider(10)` is a compile error when the method is non-static.
6. **Calling a static method via an instance name.** `h.ponder(10)` compiles and runs, but it is confusing and, in Josh's opinion, arguably should have been a compile error. Do not do it.
7. **Using `this` (or an instance variable) inside a static method.** There is no `this` in a static context. Static methods can only touch instance variables through an instance parameter.
8. **Accessing or, worse, mutating a static variable through an instance.** `maya.binomen` is bad style; `maya.binomen = "..."` changes it for everything, which the syntax disguises.
9. **Mutable static variables generally.** Strongly discouraged: it becomes hard to track which parts of the program read and write them.
10. **Returning the wrong type.** Returning `this.size` (an `int`) from a method declared to return a `Dog` will not compile. Java is very strict about type checking.
11. **Expecting `append` on a Java list.** Java's method is `add`.
12. **Thinking a missing import means "not available."** In Java, imports only shorten names. Conversely, you do not import your own project's classes at all.
13. **Trying to instantiate `List`.** `new List()` fails: `List` is abstract. Pick `ArrayList`, `LinkedList`, etc.
14. **Writing loose statements outside a method.** Java code must live in a method; orphaned code has nowhere to live, and Java starts execution by finding `main`.

---

## Likely Exam Points

### 1. Static versus instance invocation (the in-class attendance question)

Given:

```java
class Human {
    int consider(int x) { ... }
    static int ponder(int y) { ... }
}
```

**Q:** Which of these usages is appropriate, and why?
```java
Human h = new Human();
h.consider(5);
Human.consider(10);
h.ponder(10);
Human.ponder(10);
```

**A:**
- `h.consider(5)`, fine. Instance method invoked on an instance.
- `Human.consider(10)`, **compile error.** `consider` is non-static, so it cannot be invoked through the class name.
- `h.ponder(10)`, **legal Java, but inappropriate/confusing.** You are asking a specific instance to run a method for which there is no specific instance. Josh argued it arguably should have been a compile error.
- `Human.ponder(10)`, fine. Static method invoked on the class.

Follow-up posed in lecture: if `h.consider(5)` returns 5000 and `Human.ponder(10)` returns 1000, what do `h.ponder(10)` and `Human.ponder(10)` return? Both return 1000: `ponder` is static, so which (if any) instance you write on the left makes no difference to its behavior.

### 2. Why can't a static method use instance variables?

**Q:** Explain in one sentence why the following does not compile.
```java
static void makeNoise() {
    if (size < 10) { IO.println("yipyipyip!"); }
}
```
**A:** Because `static` means there is no `this`: the method belongs to the class, not to any particular dog, so there is no "my" `size` to read. The method would have to take a `Dog` parameter and read `d.size`.

### 3. Instance versus static `maxDog`

**Q:** Rewrite the instance method `Dog maxDog(Dog otherDog)` as a static method, and show how each is invoked.
**A:**
```java
// instance
Dog maxDog(Dog otherDog) {
    if (otherDog.size > this.size) { return otherDog; }
    return this;
}
// invoked: lilDog.maxDog(clifford);

// static
static Dog maxDog(Dog d1, Dog d2) {
    if (d1.size > d2.size) { return d1; }
    return d2;
}
// invoked: Dog.maxDog(lilDog, clifford);
```
The static version needs both dogs as parameters and has no `this`; the instance version needs one parameter and gets the other dog from `this`.

### 4. Terminology labeling

**Q:** Label each line as declaration, instantiation, assignment, invocation, or some combination.
```java
Dog a;
new Dog(20);
a = new Dog(5);
Dog b = new Dog(150);
b.makeNoise();
```
**A:** declaration; instantiation; instantiation and assignment; declaration, instantiation, and assignment; invocation.

### 5. Constructor shadowing

**Q:** What is wrong with `Dog(int size) { size = size; }`, and what does `size` end up as after `new Dog(30)`?
**A:** The parameter `size` shadows the instance variable, so the assignment copies the parameter onto itself and the instance variable is never written. After `new Dog(30)` the object's `size` is still the default `0`. Fix: `this.size = size;`.

### 6. What the blueprint permits

**Q:** Given `class Dog { int size; ... }`, which of these lines fails, and how does it fail?
```java
Dog d = new Dog(150);
d.size = 5;
d.name = "frank";
```
**A:** The third line fails, and it fails at **compile time**, not run time, because `name` is not in the `Dog` blueprint and Java forbids adding instance variables at runtime. The second line is fine: you may change an existing instance variable.

### 7. Static variables

**Q:** Given `static String binomen = "Canis familiaris";` inside `Dog`, what is wrong with `maya.binomen = "Vulpes vulpes";`?
**A:** Two things. Stylistically, a static variable should be accessed by class name (`Dog.binomen`), not through an instance. Semantically, this does not give Maya a personal binomen; there is only one `binomen`, shared by the class, so this changes it for every `Dog`, while the syntax makes it look instance-specific.

### 8. `List` versus `ArrayList` / ADT versus concrete implementation

**Q:** Why does `List L = new List();` fail to compile, and give two reasons Java bothers to offer multiple implementations of `List`.
**A:** `List` is abstract: it specifies guaranteed operations but contains no actual implementation, so it cannot be instantiated. You must choose a concrete implementation such as `ArrayList` or `LinkedList`. Two reasons for multiple implementations: (1) **performance** differs (a `LinkedList` removes its front item quickly regardless of size, while an `ArrayList` is slow because elements must shift over), and (2) some implementations provide **extra operations** beyond the `List` guarantee (e.g. `Stack` adds `push` and `pop`).

### 9. Imports

**Q:** True or false: you must import `java.util.List` in order to use a `List`. Explain.
**A:** False. Imports in Java only shorten names. You can write `java.util.List L = new java.util.ArrayList();` with no imports at all. This differs from Python, where importing is what makes something available. (Separately, you never import your own project's classes; Java finds them by scanning folders.)

### 10. Old versus new `main`

**Q:** Explain each piece of `public static void main(String[] args)`.
**A:** `public`: usable by any class. `static`: belongs to the class, so no instantiation needed to run it. `void`: returns nothing. `main`: the name Java looks for to start the program. `String[] args`: array of command line arguments supplied by the operating system. In Java 25+ this course writes simply `void main()` with `IO.println`.

---

## Summary

- Java lets you define your own types with **classes**; a class is a **blueprint**, and each instance created with `new` is an **object**.
- **Instance variables** are declared at the top of the class, with types, and the list is fixed: you cannot attach new fields at runtime, and trying to do so is a compile error, not a crash.
- A **constructor** (`Dog(int s) { size = s; }`) is not a method; it says how to build an instance. It is Java's `__init__`. Beware **shadowing** if the parameter shares a name with the instance variable.
- **`this`** refers to the current object inside an instance method. `this.size` is usually optional; `return this;` is not.
- Terminology: **declaration** (the house), **instantiation** (the dog), **assignment** (dog into house), **invocation** (calling a method), **member** (any variable or method), **client** (a class that uses another class).
- **Static means there is no `this`.** Static methods are invoked on the class name, cannot see instance variables except through a passed-in instance, and exist for utility classes like `Math` that are never meaningfully instantiated. Instance methods are invoked on a specific object.
- A class can mix static and non-static members. **Static variables** are class-wide; access via the class name, and strongly avoid mutable ones.
- **`public`** makes no difference this lecture; lecture 3 covers access control.
- Java 25 replaced `public static void main(String[] args)` + `System.out.println` with `void main()` + `IO.println`; old 61B materials use the old style.
- You do **not** import your own `.java` files; Java scans folders. **Imports only shorten names**, unlike Python.
- Java's list append is **`add`**, not `append`. `new List()` fails because `List` is abstract; pick a concrete implementation, typically `ArrayList`.
- **`List` is an abstract data type**; `ArrayList`, `LinkedList`, `Stack`, etc. are **concrete implementations**. Multiple implementations exist for **performance** (e.g. `LinkedList` removes the front item fast, `ArrayList` does not) and for **extra operations** (e.g. `Stack`'s `push`/`pop`).
- Lecture 3 picks up with modern list syntax (e.g. `List<String> L`) and starts building our own lists.
