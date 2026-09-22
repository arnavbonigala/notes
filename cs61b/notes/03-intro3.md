<!-- Mon, Aug 31, 2026 | sources: code (no transcript available) -->
# Lecture 3: Intro 3

## Overview

This lecture finishes the "intro" arc of CS 61B by (1) completing our tour of the three workhorse Java data structures: the `List`, the array, and the `Map`, and (2) pivoting from syntax to the single most important mental model in the course: how Java stores and copies data. We start with old-school raw `List` code, see why it forces you to write `Object x = L.get(0)` instead of `String x = L.get(0)`, and fix it with generic angle-bracket syntax (`List<String> L = new ArrayList<>()`). We then meet arrays (fixed size, single type, no methods, one field `length`, special `[]` syntax) and maps (`Map<String, Integer> m = new HashMap<>()`, with `put`/`get` replacing Python's `m["cat"]`). Then comes the conceptual heart: the Mystery of the Walrus. Why does `b = a; b.weight = 5;` change what `a` sees, while `y = x; x = 2;` leaves `y` alone? The answer is the **Golden Rule of Equals**: every `=` in Java copies the bits in the box on the right into the box on the left, and nothing more. Because reference variables hold a 64-bit *address* rather than the object itself, copying those bits copies the arrow, not the walrus. The same rule explains parameter passing: Java is always pass by value. Everything after this lecture (building our own lists) depends on getting this right, which is why the textbook warns about the Law of the Broken Futon: a half-understanding here will quietly collapse later.

---

## Key Concepts

### 1. Lists, old style vs. modern style

Java's `List` is the closest analogue to a Python list. Last lecture's code looked like this:

```java
List L = new ArrayList();
L.add("a");
L.add("b");
System.out.println(L.get(0));
```

This compiles and runs, but it is "roughly how Java looked before 2005." The problem shows up the moment you want to store the result:

```java
String x = L.get(0);  // won't compile: "Required type: String, Provided: Object"
```

Because you never told Java what the list *contains*, `get` can only promise to hand back an `Object`. In `ListDemo.java` the lecture code writes `Object x = L.get(0);`, which does compile, but leaves you holding something you can't do much with.

There is an obsolete fix called **casting**, which 61B explicitly does not teach.

The modern fix is **generics**: put the element type in angle brackets.

```java
List<String> L = new ArrayList<>();
L.add("a");
L.add("b");
String x = L.get(0);  // works great
```

Two syntax rules that are always true in CS 61B style code:

- On the **declaration** side, name the specific type: `List<String>`, `List<Integer>`.
- On the **instantiation** side, leave the brackets empty: `new ArrayList<>()`.

The textbook is blunt that the asymmetry has "subtle reasons" behind it and that you should treat it as an arbitrary choice to memorize.

Generics also change **behavior**, not just syntax. A Python list happily holds `"horse"` and `7` side by side. A `List<String>` accepts `"horse"` and rejects `7` at *compile time*. The textbook argues this restriction is a feature: "placing limitations on yourself as a programmer is a good thing. Freedom leads to complexity, and complexity is hard to fit in your brain." Static typing is one of Java's main tools for self-imposed restriction.

One wrinkle from the lecture code (`ListDemo.java`): generics only work with reference types, so you must write `List<Integer>`, not `List<int>`. The comment in the file notes that **Project Valhalla** is the ongoing effort to make `List<int>` possible someday, but "until valhalla, we must do `List<Integer>`."

### 2. Arrays: a more restricted list

Think of an array as a list with features removed:

- The **size is fixed at creation time** and can never change.
- All items must be the same type.
- Arrays have **no methods**.
- Access uses Python-like bracket syntax: `x[0]`.
- Arrays have exactly one "instance variable," `length`, accessed as `x.length` (no parentheses, because it is a field, not a method).

From `ArrayDemo.java`:

```java
String[] x = new String[5];
x[0] = "a";
x[1] = "b";
```

Three valid creation notations, all equally fine:

```java
x = new int[3];                 // length 3, all zeros by default
y = new int[]{1, 2, 3, 4, 5};   // length 5, given values
int[] z = {9, 10, 11, 12, 13};  // may omit `new` only when also declaring
```

Note that unlike a declared local variable (which has no default value), array **elements** are filled with a default value when the array is instantiated: `0` for `int`, and `null` for reference types like `String`. This is why `new String[5]` gives you five slots that are all `null` until you assign.

Why does Java have both, when arrays are strictly less capable? **Performance.** Those restrictions let arrays be faster to read and write and smaller in memory (the details are CS 61C material). And why do arrays get the pretty `[]` syntax while lists are verbose? Partly history (arrays ~1995, lists ~1998) and partly because arrays sit much closer to the underlying virtual machine. The textbook's framing: Python was built to be beautiful and flexible, Java was built for performance, so Java's original primitive was the low-level performant array rather than a flexible high-level list.

### 3. Maps

A **map** is a collection of key-value pairs where keys are unique. You already know it as a Python dictionary; theory people call it an "associative array"; Sedgewick and Wayne call it a "symbol table."

Useful intuition: a map **generalizes a list**. A list maps *integers* to data (`L[0]` gets the 0th item). A map maps *any key type* to data (`M["cat"]` gets the `"cat"`th item).

Python:

```python
census = {}
census["cat"] = 103
census["dog"] = 5
num_cats = census["cat"]
```

Java (from `MapDemo.java`, with the textbook's numbers):

```java
Map<String, Integer> m = new HashMap<>();
m.put("cat", 103);
m.put("dog", 5);
int numCats = m.get("cat");
```

Note the two type parameters: key type first, then value type. Java has no `m["cat"]` syntax for maps, so you use `put` and `get`. `HashMap` is only one implementation of `Map`; later in the course we will meet `TreeMap`, which does the same core job (`put` to store, `get` to retrieve) but with different performance characteristics and slightly different capabilities.

The course's stated philosophy here: lists and maps have "many many special purpose functions (e.g. `getOrDefault`)" that will not be covered exhaustively. 61B is not a Java class, so you are expected to discover API methods as you need them. "We're tossing you in the deep end a bit."

This is also the pivot point of the course: from here on we mostly stop learning new syntax and start building our own lists from scratch. But first we need to understand references.

### 4. The Mystery of the Walrus

Two snippets. Predict each.

```java
Walrus a = new Walrus(1000, 8.3);
Walrus b;
b = a;
b.weight = 5;
System.out.println(a);
System.out.println(b);
```

```java
int x = 5;
int y;
y = x;
x = 2;
System.out.println("x is: " + x);
System.out.println("y is: " + y);
```

In the first, the change through `b` **does** affect what you see through `a` (both print weight 5). In the second, changing `x` **does not** affect `y` (`x is: 2`, `y is: 5`). The hint given is that Java behaves the same as Python here.

The mystery is: why the difference? Two snippets, same shape (`declare, copy, mutate`), opposite outcomes. Resolving this requires understanding bits, boxes, and reference types.

### 5. Bits and types

Everything in memory is a sequence of ones and zeros. Some examples given:

- `72` is often stored as `01001000`
- `205.75` is often stored as `01000011 01001101 11000000 00000000`
- The letter `H` is often stored as `01001000` (the same bits as 72)
- `true` is often stored as `00000001`

61B will not teach *why* 205.75 gets those particular 32 bits; that is CS 61C. But notice the puzzle: `72` and `H` are the same bits. How does Java know which one it is looking at?

**Through types.**

```java
char c = 'H';
int x = c;
System.out.println(c);  // H
System.out.println(x);  // 72
```

Same (well, almost the same) bits, different interpretation, decided entirely by the declared type. (*Extra context:* "almost" because `char` is 16 bits and `int` is 32, so the bit patterns differ in width even though the numeric value matches.)

Java has exactly **8 primitive types**: `byte`, `short`, `int`, `long`, `float`, `double`, `boolean`, `char`. The textbook notes you will likely never use `short` or `float`.

### 6. Declaring a variable: the box metaphor

Picture memory as billions of bits, each with a unique address. When you declare a variable of some type, Java finds a contiguous block with exactly enough bits for that type: 32 bits for an `int`, 8 for a `byte`, 64 for a `double`. Call such a block a **box**.

Java also records, in an internal table, a mapping from the variable name to the location of the first bit of its box. So after

```java
int x;
double y;
```

you have a 32-bit box named `x` and a 64-bit box named `y`. Java might put `x` at bits 352 through 384 and `y` at bits 20800 through 20864.

Three important properties:

1. **You can never learn the address.** Java gives you no way to find out that `x` lives at bit 352. Memory addresses are below the level of abstraction Java exposes. C does expose them. This is a deliberate tradeoff: less control (you lose certain optimizations) in exchange for eliminating a large class of extremely nasty bugs. Quoting Knuth: "We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil." The textbook's analogy: you can't control your own heartbeat, which limits your optimization options but also prevents you from accidentally switching it off.

2. **There are no default values for declared variables.** Java writes nothing into the box on declaration. The compiler therefore refuses to let you *use* a variable before an `=` has filled its box.

3. When you assign, the box is filled with the specified bits. Drawing the actual binary is **box notation**.

Because raw binary is unreadable, we use **simplified box notation** for the rest of the course: draw the box, write the human-readable value inside (`-1431195969`, `567213.112`).

### 7. The Golden Rule of Equals (GRoE)

> When you write `y = x`, you are telling the Java interpreter to **copy the bits from `x` into `y`**.

That is the whole rule, and it is true for **any** assignment using `=` in Java. No exceptions.

Apply it to the `int` snippet: `y = x` copies the bits `5` into `y`'s box. `x = 2` then overwrites `x`'s box with `2`. `y`'s box was never touched again, so `y` is still 5. The two boxes were never connected; only bits were copied.

### 8. Reference types

Everything that is not one of the 8 primitives, **including arrays**, is a **reference type**.

**Object instantiation.** When you instantiate with `new`, Java allocates a box for each instance variable of the class and fills each with a default value. The constructor then usually (but not always) overwrites those boxes.

```java
public static class Walrus {
    public int weight;
    public double tuskSize;

    public Walrus(int w, double ts) {
        weight = w;
        tuskSize = ts;
    }
}
```

`new Walrus(1000, 8.3)` produces an object consisting of a 32-bit box and a 64-bit box, 96 bits total for our purposes. (Real Java adds per-object overhead, so it is somewhat more than 96, but we ignore overhead since we never touch it directly.) This walrus is **anonymous**: it exists, but no variable holds it.

**Reference variable declaration.** When you declare a variable of *any* reference type (Walrus, Dog, Planet, array, anything), Java allocates a box of exactly **64 bits**, regardless of the type.

This looks like a paradox: a Walrus needs 96+ bits, so how can a 64-bit box hold one? Resolution: **the 64-bit box does not contain the walrus, it contains the address of the walrus in memory.**

```java
Walrus someWalrus;
someWalrus = new Walrus(1000, 8.3);
```

Line 1 creates a 64-bit box. Line 2 creates the object and `new` **returns its address**; by the GRoE those address bits are copied into the `someWalrus` box. If the walrus lived starting at bit 5051956592385990207, that number (as 64 bits) is what sits in the box.

A reference variable may also hold the special value **`null`**, which corresponds to all zeros.

**Box and pointer notation.** Since a 64-bit address is unreadable, we simplify:

- an all-zero address is drawn as `null`
- a nonzero address is drawn as an **arrow** pointing at an object instantiation

This is "box and pointer" notation, and it is the drawing style you will use for the rest of 61B.

### 9. Resolving the Mystery

```java
Walrus a = new Walrus(1000, 8.3);
Walrus b;
b = a;
```

- After line 1: a 64-bit box `a` holding an arrow to a Walrus object whose `weight` box is 1000 and `tuskSize` box is 8.3.
- After line 2: an empty 64-bit box `b`. **Important: `b` is undefined, not null.** Declaration writes nothing.
- After line 3: the GRoE copies the bits in `a` into `b`. Visually, `b` copies the *arrow*. Now two boxes hold arrows to **one** Walrus.

So `b.weight = 5` follows `b`'s arrow to the single shared object and changes its `weight` box. When you then print `a`, you follow `a`'s arrow to that same object and see 5.

The textbook's punchline: "And that's it. There's no more complexity than this." Nothing special happens for objects. The rule is identical to the `int` case. The only difference is *what the bits mean*: for an `int` the bits are the number, for a reference the bits are an address.

### 10. Parameter passing is also just the GRoE

Passing a parameter to a function copies the bits, exactly like `=`. Copying the bits is called **pass by value**, and in Java we **always** pass by value.

```java
public static double average(double a, double b) {
    return (a + b) / 2;
}

public static void main(String[] args) {
    double x = 5.5;
    double y = 10.5;
    double avg = average(x, y);
}
```

`main` has boxes `x` (5.5) and `y` (10.5). When `average` is invoked, it gets its **own scope** with two brand-new boxes `a` and `b`, and the bits are copied in. If `average` assigns to `a`, `main`'s `x` is unaffected, because by the GRoE you are only refilling the box labeled `a`.

The subtlety worth internalizing: "always pass by value" is *also* true for objects. What gets copied is the 64-bit address. So the method's parameter box and the caller's variable box hold two separate copies of the same arrow. Reassigning the parameter box does nothing to the caller. **Following** the arrow and mutating the object *is* visible to the caller, because there is only one object.

### 11. Losing objects

Objects can be lost if you lose the bits holding their address. If the only copy of a particular Walrus's address is in `x`, then `x = null` permanently loses that Walrus. This is not necessarily bad: often you are done with an object and it is fine to throw away the reference. We will rely on this when we build lists.

### 12. The Law of the Broken Futon

Why spend a whole lecture on something that looks trivial, especially if you have Java experience? Because it is very easy to have a "half-cocked understanding" that is *just* good enough to write working code without real comprehension. That is survivable in the short term and fatal in the long term, once you are building linked structures where references are the entire point. (The name comes from a linked blog post about cognitive breaking points in math education.)

---

## Definitions

- **List**: A Java data structure holding an ordered sequence of items, resizable, with methods like `add` and `get`. The closest Java analogue to a Python list.
- **`ArrayList`**: A specific implementation of `List`. In 61B style, you declare the variable as `List<T>` and instantiate as `new ArrayList<>()`.
- **Generics / angle bracket syntax**: The `<Type>` notation that tells Java exactly what type a collection holds, e.g. `List<String>`. Specific type on the declaration side, empty `<>` on the instantiation side.
- **Raw type (old-school list)**: A collection declared without type parameters, e.g. `List L = new ArrayList()`. Its `get` returns `Object`, so `String x = L.get(0)` fails to compile.
- **Casting**: An obsolete pre-generics technique for recovering the specific type from an `Object`. Explicitly not taught in 61B.
- **Project Valhalla**: The (ongoing, per the lecture code comment) Java effort that would make generics over primitives, e.g. `List<int>`, possible. Until then, use `List<Integer>`.
- **Array**: A fixed-size, single-type reference-type container with no methods, bracket access syntax (`x[0]`), and exactly one field, `length`. More restricted than a list, and therefore faster and smaller.
- **`length`**: The one "instance variable" of an array; `x.length` is the number of slots, and it never changes.
- **Map**: A collection of key-value pairs with unique keys. Called a dictionary in Python, an associative array in theory, a symbol table by Sedgewick and Wayne. Accessed with `put(key, value)` and `get(key)`.
- **`HashMap`**: One implementation of `Map`. `TreeMap` is another, with different performance and capability tradeoffs.
- **Memory**: A vast collection of bits, each with a unique address, that stores all information in the computer.
- **Box**: A contiguous block of memory bits of exactly the size needed for one variable of a given type (32 bits for `int`, 64 for `double`, 8 for `byte`, 64 for any reference).
- **Box notation**: Drawing a variable as a box with its actual binary bits written inside.
- **Simplified box notation**: Drawing a variable as a box with its human-readable value written inside instead of binary. Used for the rest of the course.
- **Box and pointer notation**: Simplified box notation for reference variables, where an all-zero address is drawn as `null` and any nonzero address is drawn as an arrow to an object instantiation.
- **Primitive type**: One of Java's 8 built-in types whose bits *are* the value: `byte`, `short`, `int`, `long`, `float`, `double`, `boolean`, `char`.
- **Reference type**: Any type that is not primitive, including all classes and all arrays. A variable of reference type is a 64-bit box holding the *address* of an object, not the object itself.
- **Golden Rule of Equals (GRoE)**: Any assignment with `=` in Java copies the bits from the box on the right into the box on the left, and does nothing else. True for every assignment without exception.
- **`null`**: The special reference value corresponding to all-zero bits, meaning "this reference variable points at no object."
- **Undefined (vs. null)**: A declared-but-unassigned variable has nothing written in its box at all. `Walrus b;` leaves `b` undefined, which is not the same as `b = null`.
- **Instantiation**: Creating a new object with `new`, which allocates a box for each instance variable, fills each with a default value, runs the constructor (which usually overwrites them), and returns the object's address.
- **Anonymous object**: An object that has been created but whose address is not stored in any variable.
- **Pass by value**: The parameter-passing discipline in which the bits of the argument are copied into a new box in the callee's scope. Java always passes by value, including for references (where the copied bits are the address).
- **Scope**: The set of boxes belonging to a particular method invocation. Each invocation of a method gets its own fresh boxes for its parameters and locals.
- **Law of the Broken Futon**: The idea that a partial understanding lets you keep functioning for a while but sets you up for collapse later, motivating the careful treatment of references here.

---

## Worked Examples

### Example 1: `ListDemo.java`, why you need generics

```java
package lec3_intro3;

import java.util.ArrayList;
import java.util.List;

public class ListDemo {
    void main() {

        List L = new ArrayList();
        // project valhalla will make this possible
        // until valhalla, we must do List<Integer>

        L.add("a");
        L.add("b");
        Object x = L.get(0);
    }
}
```

Step by step:

1. The two `import` lines bring `ArrayList` and `List` into scope. Both live in `java.util`, so both must be imported; importing `List` alone is not enough to say `new ArrayList()`.
2. `List L = new ArrayList();` declares `L` as a **raw** list (no angle brackets) and points it at a new empty `ArrayList`. In box-and-pointer terms: a 64-bit box named `L` holding an arrow to a fresh, empty `ArrayList` object.
3. `L.add("a");` and `L.add("b");` follow the arrow and append to that one list object. Because the list is raw, Java has no idea the elements are `String`s; as far as the compiler knows they are just `Object`s.
4. `Object x = L.get(0);` compiles, because `get` on a raw list is typed to return `Object`. Writing `String x = L.get(0);` here would be a compile error: "Required type: String, Provided: Object."

The fix, per the textbook:

```java
List<String> L = new ArrayList<>();
L.add("a");
L.add("b");
String x = L.get(0);  // works great
```

Now `get` is known to return a `String`, and separately, `L.add(7)` would be rejected at compile time.

The comment in the file is about a *different* limitation of generics: you cannot write `List<int>`. Generics only work over reference types, so integers must go in a `List<Integer>`. Project Valhalla is the effort that may eventually relax this.

(*Extra context on the method signature:* the lecture files use an instance-style `void main()` rather than `public static void main(String[] args)`. This is a recent Java simplification for single-file programs, and the textbook uses both forms interchangeably. Treat it as a course convention and follow whatever the assignment skeletons use.)

### Example 2: `ArrayDemo.java`, declaring and filling an array

```java
package lec3_intro3;

public class ArrayDemo {
    void main() {
        String[] x = new String[5];
        x[0] = "a";
        x[1] = "b";
        //x[2] = "iuHFLIUWHFLK ... ";
    }
}
```

Step by step:

1. `String[] x` declares a reference variable. Arrays are reference types, so this is a 64-bit box, same as any other reference.
2. `new String[5]` creates an array object with 5 slots. Each slot is itself a reference (to a `String`), and each is filled with the default value `null`. `new` returns the array's address, and by the GRoE those bits are copied into `x`'s box.
3. `x[0] = "a";` follows `x`'s arrow to the array object and copies, into slot 0, the address of the string `"a"`. Same for slot 1.
4. Slots 2, 3, 4 remain `null`. `x.length` is 5 and will stay 5 forever; there is no way to grow this array.

Note there are no imports and no `new java.util.anything`: arrays are built into the language, not a library class.

The commented-out third line is just scratch text from lecture. Nothing conceptual is being demonstrated by the garbage string itself; the takeaway is only that any `String` value can go in any slot.

Contrast the three creation notations:

```java
x = new int[3];                 // 3 slots, all 0
y = new int[]{1, 2, 3, 4, 5};   // 5 slots with those values
int[] z = {9, 10, 11, 12, 13};  // `new` omitted, legal only because we also declare z here
```

The third form's shorthand only works at the point of declaration. `z = {1, 2, 3};` on its own line is not legal.

### Example 3: `MapDemo.java`, maps and the `null` return

```java
package lec3_intro3;

import java.util.HashMap;
import java.util.Map;

public class MapDemo {
    void main() {
        Map<String, Integer> m = new HashMap<>();
        // python would look like this
        // m["cat"] = 5;

        m.put("cat", 5);
        m.put("dog", 917);

        // in python m["cat"]
        IO.println(m.get("cat"));
        IO.println(m.get("aaron"));
    }
}
```

Step by step:

1. `Map<String, Integer> m = new HashMap<>();` declares `m` with two type parameters: keys are `String`, values are `Integer`. Note the declaration/instantiation rule in action: full types on the left, bare `<>` on the right. In box-and-pointer terms, a 64-bit box `m` with an arrow to a new empty `HashMap`.
2. `m.put("cat", 5);` stores the pair. Java has no `m["cat"] = 5` syntax, which is exactly what the Python comment is highlighting.
3. `m.put("dog", 917);` stores a second pair. Keys are unique, so a second `put("dog", ...)` would *replace* the old value rather than adding a duplicate key.
4. `IO.println(m.get("cat"));` prints `5`.
5. `IO.println(m.get("aaron"));` prints `null`. `"aaron"` was never `put`, and a `Map`'s `get` returns `null` for a missing key rather than throwing.

That last line is the interesting one, and it sets up a trap. Printing `null` is harmless. But this is not:

```java
int numCats = m.get("aaron");   // NullPointerException at runtime
```

The `Integer` returned is `null`, and unwrapping `null` into an `int` fails at runtime. This is also why the value type must be `Integer` and not `int`: a map's value slot holds a reference, and references can be `null`.

(*Extra context:* `IO.println` is a newer convenience for `System.out.println` in recent Java. The textbook's equivalent snippets use `System.out.println`. Use whichever your skeleton code uses; the behavior shown here is identical.)

The textbook's version of the same program, with the census numbers:

```java
Map<String, Integer> m = new HashMap<>();
m.put("cat", 103);
m.put("dog", 5);
int numCats = m.get("cat");   // 103
```

### Example 4: The Mystery of the Walrus, traced with box and pointer

```java
Walrus a = new Walrus(1000, 8.3);
Walrus b;
b = a;
b.weight = 5;
System.out.println(a);
System.out.println(b);
```

Given:

```java
public static class Walrus {
    public int weight;
    public double tuskSize;

    public Walrus(int w, double ts) {
        weight = w;
        tuskSize = ts;
    }

    public String toString() {
        return String.format("weight: %d, tusk size: %.2f", weight, tuskSize);
    }
}
```

Line 1, `Walrus a = new Walrus(1000, 8.3);`
`new` allocates an object with two boxes: a 32-bit `weight` and a 64-bit `tuskSize`, both filled with defaults, then the constructor writes 1000 and 8.3 into them. `new` returns the object's address. A 64-bit box named `a` is created and those address bits are copied in. Picture: `a` with an arrow to a Walrus containing `weight: 1000, tuskSize: 8.3`.

Line 2, `Walrus b;`
A 64-bit box named `b` is created and **nothing is written into it**. `b` is *undefined*, which is specifically not the same as `null`. Nothing is drawn inside the box.

Line 3, `b = a;`
GRoE: copy the bits from `a`'s box into `b`'s box. The bits are an address, so `b` now holds the same address. Picture: two boxes, `a` and `b`, with two arrows pointing at **the same single Walrus object**. Crucially, no second Walrus was created. `new` is the only thing that creates objects, and there is no `new` on this line.

Line 4, `b.weight = 5;`
Follow `b`'s arrow to the object, then assign into that object's `weight` box: it now holds 5. There is only one object, so this is simultaneously a change to what `a` points at.

Lines 5 and 6 both print `weight: 5, tusk size: 8.30`.

### Example 5: The primitive version, for contrast

```java
int x = 5;
int y;
y = x;
x = 2;
System.out.println("x is: " + x);  // x is: 2
System.out.println("y is: " + y);  // y is: 5
```

Line 1: a 32-bit box `x` holding 5.
Line 2: a 32-bit box `y`, empty (undefined).
Line 3: GRoE copies the bits from `x` into `y`. `y`'s box now holds 5. The boxes are separate; nothing links them.
Line 4: overwrite `x`'s box with 2. `y`'s box is untouched.

**The two examples use the exact same rule.** In both, `=` copied bits from one box into another. The only difference is the *meaning* of the bits. For an `int`, the bits are the value itself, so copying gives you an independent value. For a `Walrus`, the bits are an address, so copying gives you a second way to reach the same object. This is the entire mystery, resolved.

### Example 6: Parameter passing, `average`

```java
public static double average(double a, double b) {
    return (a + b) / 2;
}

public static void main(String[] args) {
    double x = 5.5;
    double y = 10.5;
    double avg = average(x, y);
}
```

After the first two lines of `main`: boxes `x` (5.5) and `y` (10.5) in `main`'s scope.

At the call to `average`: the method gets its **own scope** containing two new 64-bit boxes named `a` and `b`, and the bits of `x` and `y` are copied in. That copy is what "pass by value" means. `a` and `b` are not `x` and `y` with different names; they are different boxes that happen to hold the same bits.

Therefore if `average` executed `a = 0;`, `main`'s `x` would still be 5.5.

### Example 7: `PassByValueFigure`, the test-your-understanding exercise

```java
public class PassByValueFigure {
    public static void main(String[] args) {
        Walrus walrus = new Walrus(3500, 10.5);
        int x = 9;

        doStuff(walrus, x);
        System.out.println(walrus);
        System.out.println(x);
    }

    public static void doStuff(Walrus W, int x) {
        W.weight = W.weight - 100;
        x = x - 5;
    }
}
```

Question: does `doStuff` affect `walrus` and/or `x`? The hint is that the GRoE alone suffices.

Trace it:

1. `main` has a 64-bit box `walrus` with an arrow to a Walrus (`weight` 3500, `tuskSize` 10.5), and a 32-bit box `x` holding 9.
2. Calling `doStuff(walrus, x)` creates a new scope with a 64-bit box `W` and a 32-bit box `x`. Note that `doStuff`'s `x` is a **completely different box** from `main`'s `x` that merely shares a name. The bits are copied in: `W` gets a copy of the address (so `W` and `walrus` both arrow to the one Walrus), and `doStuff`'s `x` gets a copy of the value 9.
3. `W.weight = W.weight - 100;` follows `W`'s arrow to the shared object and writes 3400 into its `weight` box. This is a mutation of the object, not an assignment to `W`, so it is visible to `main`.
4. `x = x - 5;` writes 4 into `doStuff`'s own `x` box. `main`'s `x` box is untouched.
5. `doStuff` returns and its boxes disappear.

Output:

```
weight: 3400, tusk size: 10.50
9
```

So: `walrus` is affected (its object was mutated), `x` is not.

Compare a variant that changes the answer:

```java
public static void doStuff(Walrus W, int x) {
    W = new Walrus(1, 1.0);   // only refills W's own box
    x = x - 5;
}
```

Here nothing in `main` changes at all. `W = ...` copies a new address into `doStuff`'s `W` box; `main`'s `walrus` box still holds the old address and still shows `weight: 3500`. The distinction to internalize: **assigning to a parameter is invisible to the caller; mutating through a parameter is visible.**

### Example 8: Array instantiation as a reference type

```java
int[] x;
Planet[] planets;
```

Both create 64-bit boxes, because array variables are reference variables like any other. `x` can only hold the address of an `int` array, `planets` only the address of a `Planet` array.

```java
x = new int[]{0, 1, 2, 95, 4};
```

`new` creates 5 boxes of 32 bits each (holding 0, 1, 2, 95, 4) and returns the address of the overall array object, which the GRoE copies into `x`'s box. Picture: `x` with an arrow to a row of five int boxes.

Now consider losing an object:

```java
x = null;
```

If `x` held the only copy of that array's address, the array is permanently lost. Same for a Walrus. This is often exactly what you want, and we will use it deliberately when building lists.

---

## Common Pitfalls

- **Forgetting the angle brackets on the declaration, or writing them on the instantiation.** The rule is `List<String> L = new ArrayList<>();`. Not `List<String> L = new ArrayList<String>();` in 61B style, and definitely not `List L = new ArrayList<String>();`.
- **Writing `List<int>` or `Map<String, int>`.** Generics require reference types: use `Integer`. (This is what the Project Valhalla comment in `ListDemo.java` is about.)
- **Assigning a raw `get` result to a specific type.** `String x = L.get(0)` on a raw `List` is a compile error, not a runtime error. Either use generics (correct) or `Object x` (what the lecture code does to make it compile).
- **Reaching for casting.** It is the obsolete fix, and 61B does not teach it. Use generics.
- **`x.length()` on an array.** `length` is a field, not a method. Arrays have **no** methods. (Confusingly, `String` *does* use `length()`. Arrays do not.)
- **Expecting an array to grow.** Size is fixed at creation. If you need growth you need a `List`, or you need to allocate a new array and copy.
- **Trying `m["cat"]` in Java.** Maps use `put` and `get`. Brackets are for arrays only.
- **Assuming a missing key throws.** `m.get("aaron")` returns `null`. It only blows up if you then unwrap it into an `int` or call a method on it.
- **Thinking `b = a` copies the object.** It copies the bits, which for a reference means it copies the arrow. Only `new` creates objects.
- **Confusing undefined with null.** After `Walrus b;`, `b` is undefined: nothing was written into the box. Java will refuse to let you read it. `b = null` is a deliberate, different thing (all-zero bits).
- **Expecting declared variables to have default values.** They do not, and the compiler enforces this. Array *elements*, by contrast, are default-initialized on instantiation (`0`, `null`, `false`).
- **Believing Java is "pass by reference" for objects.** Java is **always** pass by value. The copied value for an object argument happens to be an address. This distinction is exactly what makes the `doStuff` exercise tricky.
- **Assuming a method that mutates an object's field must be reassigning the parameter, or vice versa.** `W.weight = 5` (visible to caller) and `W = new Walrus(...)` (invisible to caller) look similar and behave oppositely.
- **Shadowing confusion.** `doStuff`'s parameter `x` and `main`'s local `x` are unrelated boxes with the same label. Each method invocation has its own scope.
- **Trying to print or reason about actual memory addresses.** Java deliberately does not expose them. Draw arrows instead.
- **The Broken Futon trap:** being able to get code to compile and run without actually being able to draw the box-and-pointer diagram. If you cannot draw it, you do not yet know it.

---

## Likely Exam Points

### 1. Generic list syntax

Commonly tested as "fix this line" or "will this compile."

**Q:** Which of these compile, assuming `java.util.List` and `java.util.ArrayList` are imported?
```java
(a) List<String> a = new ArrayList<>();
(b) List<String> b = new ArrayList();      
(c) List c = new ArrayList();
(d) String s = c.get(0);                   // c from (c)
(e) List<int> e = new ArrayList<>();
```

**A:** (a) is correct 61B style. (b) compiles but is not 61B style (it uses a raw type on the right; expect it to be marked wrong on a style question). (c) compiles as a raw list. (d) does **not** compile: raw `get` returns `Object`, so you would need `Object s`. (e) does not compile: generics cannot take primitives, you need `List<Integer>`.

### 2. Arrays vs. lists

**Q:** Name three things an array cannot do that a `List` can, and state why Java keeps arrays anyway.

**A:** An array cannot change size after creation, has no methods (no `add`, no `get`), and cannot mix types. Java keeps arrays because the restrictions make them faster to read and write and smaller in memory. They also predate lists in the language (~1995 vs ~1998) and sit closer to the underlying virtual machine, which is why they get the special `[]` syntax.

### 3. Map basics and missing keys

**Q:** What does this print?
```java
Map<String, Integer> m = new HashMap<>();
m.put("cat", 5);
m.put("cat", 12);
System.out.println(m.get("cat"));
System.out.println(m.get("emu"));
```

**A:** `12`, then `null`. Keys are unique, so the second `put("cat", ...)` replaces the value rather than adding a second entry. A missing key's `get` returns `null`. Follow-up: `int n = m.get("emu");` would throw a NullPointerException at runtime, which is why the value type must be `Integer`.

### 4. The Golden Rule of Equals, stated

**Q:** State the GRoE and explain, using it alone, why `y = x; x = 2;` leaves `y` unchanged but `b = a; b.weight = 5;` changes what `a` shows.

**A:** The GRoE: any `=` copies the bits in the box on the right into the box on the left, and does nothing else. For `int x`, the bits *are* the number 5, so `y`'s box gets its own copy of 5 and later changes to `x`'s box cannot reach it. For `Walrus a`, the bits are the 64-bit *address* of an object, so `b`'s box gets a copy of that address and both boxes now point at one object. `b.weight = 5` is not an assignment to `b` at all; it follows the arrow and assigns into the shared object's `weight` box, which `a` also reaches.

### 5. Box and pointer diagrams

**Q:** Draw (describe) the state after:
```java
int[] a = {1, 2, 3};
int[] b = a;
b[0] = 7;
int[] c = {1, 2, 3};
```
What are `a[0]`, `b[0]`, `c[0]`?

**A:** `a` is a 64-bit box with an arrow to a 3-slot int array. `b = a` copies the arrow, so `b` points at the **same** array; there is one array with two arrows into it. `b[0] = 7` writes into that shared array. `c = {1, 2, 3}` uses a separate instantiation, so `c` arrows to a **different**, brand-new array. Results: `a[0]` is 7, `b[0]` is 7, `c[0]` is 1. Rule of thumb: count the `new`s (including the `{...}` shorthand) to count the objects.

### 6. Primitive vs. reference, and box sizes

**Q:** How many bits does Java allocate for each declaration, and which are reference variables?
```java
int x;  double d;  boolean flag;  Walrus w;  int[] arr;  String s;
```

**A:** `int x` gets 32 bits, `double d` gets 64. `boolean flag` is a primitive (the exact bit count is not something we care about in 61B; the textbook notes `true` is often stored as `00000001`). `Walrus w`, `int[] arr`, and `String s` are all **reference** variables and each gets exactly 64 bits regardless of type, because the box holds an address. Only `byte`, `short`, `int`, `long`, `float`, `double`, `boolean`, `char` are primitives; everything else, arrays included, is a reference type.

### 7. Pass by value (the classic)

**Q:** What does this print?
```java
public static void main(String[] args) {
    Walrus walrus = new Walrus(3500, 10.5);
    int x = 9;
    doStuff(walrus, x);
    System.out.println(walrus);
    System.out.println(x);
}
public static void doStuff(Walrus W, int x) {
    W.weight = W.weight - 100;
    x = x - 5;
}
```

**A:**
```
weight: 3400, tusk size: 10.50
9
```
`W` receives a copy of the address, so `W.weight = ...` mutates the one shared Walrus and `main` sees 3400. `doStuff`'s `x` is a separate box holding a copy of 9; setting it to 4 has no effect on `main`'s `x`. Java always passes by value.

### 8. Assign-to-parameter vs. mutate-through-parameter

**Q:** Modify `doStuff` above to `W = new Walrus(1, 1.0);` instead of `W.weight = W.weight - 100;`. Now what does `main` print for `walrus`?

**A:** `weight: 3500, tusk size: 10.50`, unchanged. The GRoE says `W = ...` only refills `doStuff`'s own `W` box with a new address. `main`'s `walrus` box still holds the original address. A method can mutate an object you hand it, but it can never change *which* object your variable points at.

### 9. Undefined vs. null

**Q:** Is `b` null after `Walrus b;`? Can you print it?

**A:** No. `b` is **undefined**: Java writes nothing into a box on declaration, so there are no default values for local variables. The compiler will reject any attempt to use `b` before an assignment fills it. `null` is different: it is the specific all-zeros value you get from `b = null`, drawn as `null` in box and pointer notation.

### 10. Same bits, different types

**Q:** What does this print, and what principle does it demonstrate?
```java
char c = 'H';
int x = c;
System.out.println(c);
System.out.println(x);
```

**A:** `H` then `72`. All data is just bits, and `H` and `72` are stored with essentially the same bit pattern (`01001000`). The **type** is what tells Java how to interpret those bits. This is why Java's static types matter beyond just catching errors.

### 11. Losing an object

**Q:** After the following, what has happened to the array?
```java
int[] x = new int[]{0, 1, 2, 95, 4};
x = null;
```

**A:** If `x` held the only copy of the array's address, the array is permanently lost: there is no longer any way to reach it. This is fine when you are intentionally done with an object, and we will exploit it when building lists.

---

## Summary

- **Lists:** old-school raw `List L = new ArrayList()` forces `Object x = L.get(0)`. Fix with generics: specific type when declaring (`List<String>`), empty `<>` when instantiating (`new ArrayList<>()`). Generics also restrict what can go in, which the course treats as a feature: static typing limits your choices, and limits keep complexity manageable. Generics take reference types only, so `List<Integer>`, not `List<int>` (Project Valhalla may change this someday).
- **Arrays:** fixed size set at creation, one type, no methods, bracket access, one field `x.length`. Three creation forms (`new int[3]`, `new int[]{...}`, `int[] z = {...}`). Elements get default values on instantiation. More restricted than lists, and therefore faster and smaller.
- **Maps:** unique keys mapped to values, Python's dict. `Map<String, Integer> m = new HashMap<>(); m.put(k, v); m.get(k);`. A map generalizes a list from integer keys to arbitrary keys. `get` on a missing key returns `null`. `HashMap` is one implementation; `TreeMap` comes later with different tradeoffs.
- **The course expects you to look up API methods yourself.** 61B is not a Java class.
- **Everything is bits.** Types are what tell Java how to interpret identical bit patterns (`'H'` and `72`). There are 8 primitive types; `short` and `float` you will likely never use.
- **Box metaphor:** declaring a variable reserves a box of exactly the right size (32 bits for `int`, 64 for `double`) and records a name-to-location mapping. Java never reveals the address. Declaration writes nothing, so there are no default values for variables and the compiler blocks use before assignment. Draw boxes with readable values (simplified box notation).
- **GRoE, the one rule:** every `=` copies the bits from the right box into the left box, and nothing more. No exceptions.
- **Reference types:** everything that is not one of the 8 primitives, arrays included. A reference variable is always a 64-bit box holding the **address** of an object, never the object. `new` allocates the instance-variable boxes, default-fills them, runs the constructor, and returns the address. `null` is all zeros; undefined is nothing at all.
- **Box and pointer notation:** `null` drawn as `null`, any other address drawn as an arrow to an object. Count the `new`s to count the objects.
- **Mystery of the Walrus resolved:** `b = a` copies the arrow, so two variables reach one object, and mutating through either is visible through both. The `int` case uses the identical rule; only the meaning of the bits differs.
- **Parameter passing is the GRoE too:** Java is **always** pass by value. Each call gets its own scope with fresh boxes and copied bits. Mutating an object through a parameter is visible to the caller; assigning to the parameter itself is not.
- **Objects are lost when the last copy of their address is gone** (`x = null`), which is often exactly what you want.
- **Do not settle for a half-understanding here** (Law of the Broken Futon). If you cannot draw the diagram, you do not know it yet, and everything from here on is built on it.
