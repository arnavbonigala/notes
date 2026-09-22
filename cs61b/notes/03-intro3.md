<!-- Mon, Aug 31, 2026 | sources: slides + code + YouTube auto-transcript + textbook -->
# Lecture 3: Lists, Arrays, Maps, References

## Overview

This lecture finishes the "learning Java syntax" phase of CS 61B and then pivots to the conceptual model that everything afterwards depends on. The first half covers three fundamental data structures: modern (Java 5.0, angle-bracket) `List`s, which fix the old-school problem that you could not assign the result of `get` to a typed variable and which restrict every element to a single static type; arrays, a stripped-down, fixed-size, method-free, high-performance collection that gets Java's special `[]` syntax; and maps (`Map`/`HashMap`), Java's version of the Python dictionary, understood as a generalization of a list where the keys can be any type rather than just the integers 0 through length - 1. The second half opens the "Mystery of the Walrus": why mutating `b` changes `a` for objects but not for `int`s. The resolution is a memory model built from bits, boxes, and the Golden Rule of Equals (`b = a` copies all the bits from `a` into `b`, always, for every type), plus the fact that a reference variable's box holds a 64-bit address rather than the object itself. Parameter passing obeys the same rule (pass by value), and 2D arrays are just arrays whose entries hold addresses of other arrays.

---

## Key Concepts

### 1. Old-school lists and their limitation

Lecture 2 left us with code like:

```java
List L = new ArrayList();
L.add("a");
L.add("b");
```

Java has no bracket notation for lists, so retrieval uses a method: `L.get(0)`. Java is a deliberately verbose language that does not lean on syntactic sugar, so where Python writes `L[0]`, Java writes `L.get(0)`.

The problem shows up the moment you try to *store* the result:

```java
String x = L.get(0);  // won't compile
```

The error is `Required type: String, Provided: Object`. A raw `List` is, from the type checker's point of view, a list of `Object`, so `get` hands back an `Object`, and an `Object` does not fit in a `String` box. There was a fix in 2005-era Java called **casting**; Josh explicitly says he will not teach it because it is obsolete.

### 2. Java 5.0 lists: angle brackets

In 2005 Java introduced angle-bracket (generic) syntax:

```java
List<String> L = new ArrayList<>();
L.add("a");
L.add("b");
String x = L.get(0);  // works
```

Two rules that are always true in 61B style code:

- **Declaration side**: name a specific type, e.g. `List<String>`, `List<Integer>`.
- **Instantiation side**: empty diamond, `new ArrayList<>()`.

Historical note from the slides: until about 2011 you had to write `<String>` on *both* sides. IntelliJ will show a yellow squiggle if you write `new ArrayList()` with no diamond at all; the code still compiles and runs, but it is considered bad style.

A consequence: **all items in a generic list must have the same type.** `L.add(3)` on a `List<String>` is a compile error. In Python, `L = []; L.append("a"); L.append(3)` is fine.

Note from the demo: you cannot write `List<int>`. You must write `List<Integer>`. Josh mentions that Project Valhalla (in development for roughly nine years) may eventually make `List<int>` possible, but for now it is not.

### 3. Why restriction is a *good* thing

The lecture poses this as a discussion question. The answer given:

- Single-typed lists **restrict the set of choices you have to make as a programmer.**
- Freedom leads to complexity; complexity is hard to fit in your brain.
- Uniformity means code that operates over a list will not break on a surprise element (a student answer accepted in lecture: "more robust").
- Java has many features designed to let you restrict yourself. **This is a recurring theme of the course.**

### 4. Arrays

An array is a **restricted version of the list ADT**:

- Size must be declared when the array is created, and **can never change** (no extending, no shortening).
- All items must be the same type.
- **No methods.** There is no `x.indexOf(...)`, no `x.add(...)`.
- Access uses Python-like bracket syntax: `x[0]`.
- Exactly one "instance variable": `x.length` (no parentheses, it is not a method call). The official Java docs do not call it an instance variable, but Josh says it is reasonable to think of it that way.

```java
void main() {
   String[] x = new String[5]; // size 5
   x[0] = "a";
   x[1] = "b";
   System.out.println(x[0]);
}
```

There is no Python equivalent you have been taught (Python lists grow; arrays do not).

**Three valid creation notations:**

```java
x = new int[3];                 // length 3, all zeros by default
y = new int[]{1, 2, 3, 4, 5};   // length inferred as 5
int[] z = {9, 10, 11, 12, 13};  // can omit `new` ONLY when also declaring
```

The third form's restriction is real and was emphasized in lecture: `z = {1, 2, 3};` on its own line (without the declaration `int[] z`) will not compile. Josh says he does not know the reason for that restriction.

**Why does Java have both lists and arrays?** Arrays are more performant: reading and writing is faster and they use less memory. Arrays are compact in memory, whereas lists involve "a level of indirection" that cannot be fully explained yet. You learn the details in CS 61C.

**Why does Java favor arrays** (special bracket syntax, no imports needed)? Partly historical (arrays ~1995 predate `List` ~1998), partly because arrays are closer to the underlying virtual machine, and partly philosophy: Java was built for performance, whereas Python was built to be beautiful, simple, and elegant. A student's "it's legacy" answer was accepted as partially right, but the deeper reason is memory allocation and performance.

### 5. Maps

A **map** is a collection of key-value pairs where **each key is guaranteed to be unique**. Other names for the same idea:

- **Dictionary** (Python)
- **Associative array** (theoretical CS)
- **Symbol table** (the Sedgewick and Wayne Princeton *Algorithms* book; Josh jokes "only at Princeton")

The key intuition offered: **a map is a generalization of a list.** A list maps *integers* (specifically 0 through length - 1) to data, so `L[0]` gets the 0th item. A map maps *any key type at all* to data, so `census["cat"]` gets the `"cat"`th item.

Python:

```python
census = {}
census["cat"] = 103
census["dog"] = 5
num_cats = census["cat"]
```

Java:

```java
import java.util.HashMap;
import java.util.Map;

void main() {
   Map<String, Integer> m = new HashMap<>();
   m.put("cat", 103);
   m.put("dog", 5);
   int numCats = m.get("cat");
}
```

Notice you specify **two** types: what you are mapping *from* (the key) and what you are mapping *to* (the value). Bracket assignment (`m["cat"] = 5`) does not exist in Java; you use `put` and `get`.

`HashMap` is one implementation of the `Map` interface. `TreeMap` is another, coming later in the course (roughly seven weeks in); it does the same core job but with different performance characteristics and minor capability differences. This mirrors the `List`/`ArrayList` abstract-vs-specific distinction from Lecture 2.

From the live demo (`MapDemo.java`): `m.get("aaron")` on a key that was never put returns **`null`**. Josh's comment: "Was that a good idea? I don't know, but that's what it does."

Lists and Maps have many special-purpose methods (`getOrDefault`, `keySet`, etc.) that the course will not cover exhaustively. You are expected to discover them as needed, using IntelliJ's autocomplete (press `.` to see what's available). 61B is not a Java class.

### 6. The Mystery of the Walrus

Two poll questions, posed before any of the machinery was taught:

```java
Walrus a = new Walrus(1000, 8.3);
Walrus b;
b = a;
b.weight = 5;
System.out.println(a);
System.out.println(b);
```
Does the change to `b` affect `a`? **Yes.** Output:
```
weight: 5, tusk size: 8.30
weight: 5, tusk size: 8.30
```

```java
int x = 5;
int y;
y = x;
x = 2;
```
Does the change to `x` affect `y`? **No.** Output: `x is: 2`, `y is: 5`.

The class split roughly 50/50 on the first one. Josh's framing: in the space of all possible programming languages, *any* answer could be correct. The only thing that fixes the behavior is that humans designed Java's semantics that way and wrote an interpreter to match. Our job is to learn what the Java designers chose.

Crucially: saying "both walruses changed" is already wrong. **There was only ever one walrus.** `a` and `b` are two names (aliases) for the same object. As Josh put it: "this is walrus, his name's A, but his friends call him B."

### 7. Bits

All information in memory is a sequence of ones and zeros (you could design a machine with three states, or call them cats and dogs; digital computers happen to use two).

- 72 is often stored as `01001000`
- 205.75 (as a 64-bit double) as `01000011 01001101 11000000 00000000 ...`
- The letter `H` as `01001000`, **the same bits as 72**
- `true` as `00000001`

Since `H` and `72` are the same bits, how does Java know how to interpret them? **Through types.** The declared type of the variable tells the interpreter how to read the bits.

```java
char c = 'H';
int x = c;
System.out.println(c);  // H
System.out.println(x);  // 72
```

There are **8 primitive types**: `byte`, `short`, `int`, `long`, `float`, `double`, `boolean`, `char`. 61B will not cover the precise binary representations (that is 61C). Precise representations may vary from machine to machine. (Textbook aside: you will likely never use `short` and `float`.)

### 8. Declaring a variable, and the box model

When you declare a variable of a primitive type:

1. Your computer sets aside **exactly enough bits** to hold a thing of that type (`int` gets 32 bits, `double` gets 64 bits). Each primitive type has a fixed width.
2. Java creates an internal table mapping the variable *name* to the *location* of those bits. ("Java's got its messy filing cabinet and it knows, ah, that's where X is.")
3. Java writes **nothing** into the reserved box. There are no default values for local variables, and for safety Java **will not let you use an uninitialized variable** (your code will not compile). This is unlike C, where you can declare and print garbage.

Java gives you no way to learn the actual memory address. This is a tradeoff: less control and fewer optimizations, but it eliminates a large class of extremely nasty bugs. (Textbook analogy: you cannot directly control your heartbeat, which restricts optimization but prevents you from accidentally turning it off. Also quoted: Knuth's "premature optimization is the root of all evil.")

**Simplified box notation**: since raw binary is unreadable to humans, we draw the box with a human-readable symbol inside (`5`, `-1431195969`, `567213.112`) instead of 32 or 64 bits. We will use this for the rest of the course.

### 9. The Golden Rule of Equals (GRoE)

> **Given variables `y` and `x`: `y = x` copies all the bits from `x` into `y`.**

That is the whole rule. It is true for **any** assignment with `=` in Java, for **any** type: `int`, `double`, `char`, `Walrus`, array, anything. "Always, always, always every time you do equals, it's just taking a sequence of bits and putting them in a box. If you ever want to know what happens, it's always that."

This is a rule Josh made up as a teaching device, not official Java terminology, but it is precisely correct.

### 10. Reference types

The 8 primitives are one category. **Everything else, including arrays, is a reference type.**

**Object instantiation.** When you run `new Walrus(1000, 8.3)`:

1. Java allocates a box of bits for **each instance variable** of the class: 32 bits for `int weight`, 64 bits for `double tuskSize`, so about 96 bits total.
2. Java fills them with **default values** (0 for numerics, `null` for references). Note the contrast with local variable declaration, which has no defaults. Josh's colorful framing: you cannot let someone look at an unborn "astral walrus," so you give it form with zeros.
3. The constructor then usually (but not always) overwrites those boxes. `weight = w;` literally copies the 32 bits of `w` into the `weight` box, by the GRoE.

Real Java objects carry a little header overhead, so a `Walrus` is slightly more than 96 bits, but we ignore that since we never interact with it directly. (In general, an object's size is the sum of its parts plus a bit of overhead.)

4. `new` **returns the address** of the newly created object. Addresses in Java are **64 bits**. If the object lands at memory location 2384723423, `new` returns 2384723423.

Why an address rather than the object itself? Because the object might be huge (a million bits), and it would be wildly inconvenient to copy the whole thing around every time. This level of indirection is the point.

**Reference variable declaration.** When you declare a variable of *any* reference type:

- Java allocates exactly **64 bits**, no matter what kind of object, whether it is a `Walrus`, a `Planet`, a `Dog`, or a giant array.
- Those 64 bits are either all zeros (**`null`**) or the 64-bit address of a specific instance.

The "Walrus Paradox" (a `Walrus` needs 96 bits but the variable only has 64) dissolves once you realize the box holds the *address*, not the walrus. Josh's metaphor from lecture: declaring `Walrus someWalrus;` builds a **walrus house** (always exactly 64 bits); `new` gives you back a **walrus address**; and `=` puts the address in the house.

**Box and pointer notation.** Since 64-bit addresses are meaningless to humans, we draw:

- An all-zero address as the word **`null`**.
- Any non-zero address as an **arrow** pointing at the object instantiation.

### 11. Reference types obey the GRoE

```java
Walrus a;
a = new Walrus(1000, 8.3);
Walrus b;
b = a;
```

Step by step in box-and-pointer terms:

1. `Walrus a;` creates a 64-bit box labeled `a`, containing nothing usable.
2. `a = new Walrus(1000, 8.3);` creates a 96-bit walrus object somewhere in memory; `new` returns its address; the GRoE copies those 64 address bits into `a`. Draw: box `a` with an arrow to the walrus.
3. `Walrus b;` creates a second 64-bit box. **Important: `b` is currently *undefined*, not `null`.** The slides call this out explicitly.
4. `b = a;` copies the bits in `a` into `b`. Visually, we copy the *arrow*: `b` now points at the **same** walrus instance. No new walrus was created.

Therefore `b.weight = 5;` reaches through the one and only walrus, and printing `a` shows weight 5 too.

### 12. Parameter passing is also the GRoE

Passing parameters obeys exactly the same rule: **copy the bits into the new scope.** This is called **pass by value**, and Java is *always* pass by value.

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

`main` has its own boxes `x` (64 bits, holding 5.5) and `y` (64 bits, holding 10.5). When `average` is invoked, it gets its **own scope** with two brand-new boxes `a` and `b`, and the bits are copied in. If `average` modified `a`, `main`'s `x` would be unaffected, because the assignment would just refill `a`'s box.

Josh notes some people claim Java is pass by reference; he disagrees. It is pass by value. When you pass an object, the *value being copied is the address*, which is why the callee can still reach and mutate the same object.

### 13. 2D arrays

A 2D array is an **array of array addresses**.

```java
int[][] x = new int[4][6];
```

This creates an array `x` of length 4. Each of the 4 entries stores the *address* of an `int` array of length 6. So this statement actually creates **5** arrays total: the outer one and 4 inner ones.

```java
int[][] theMatrix;
theMatrix = new int[4][3];
theMatrix[0] = new int[]{6, 1, 2};
theMatrix[1] = new int[]{7, 1, 2};
theMatrix[2] = new int[]{1, 2, 3, 4};

int[] rowZero = theMatrix[0];
rowZero[1] = -5;
```

Line by line, per the slides:

- `new int[4][3]` creates four boxes, each able to hold an int-array reference, and **also** creates four arrays of 3 boxes each (all zeros by default). The four addresses go into the outer boxes.
- `theMatrix[2] = new int[]{1, 2, 3, 4};` creates a *new* array with four boxes holding 1, 2, 3, 4, and copies its address into `theMatrix` box #2. The original length-3 array that used to live there is now unreachable (lost).
- Rows need not be the same length; `theMatrix[2]` has length 4 while `theMatrix[0]` has length 3.
- `int[] rowZero = theMatrix[0];` copies an *address*, so `rowZero` and `theMatrix[0]` are the same array. `rowZero[1] = -5;` changes `theMatrix[0][1]` too.

**Literal syntax** works too:

```java
int[][] pascalsTriangle = new int[][]{{1}, {1, 1}, {1, 2, 1}, {1, 3, 3, 1}};
```

**Ragged vs. rectangular:**

```java
int[][] x;
int[][] matrix;
x = new int[4][];       // creates 1 total array; the 4 inner slots are null
matrix = new int[4][4]; // creates 5 total arrays
```

If you want every row the same width, specify both numbers. Then `matrix.length` is 4 and `matrix[i].length` is 4 for all `i`.

---

## Definitions

- **List (Java)**: a resizable, ordered collection with methods (`add`, `get`, ...). `List` is the abstract type; `ArrayList` is one concrete implementation.
- **Java 4.0 style list**: a raw `List L = new ArrayList();` with no angle brackets. Elements are treated as `Object` on retrieval, so `String x = L.get(0);` will not compile.
- **Java 5.0 style list / angle bracket (generic) syntax**: `List<String> L = new ArrayList<>();`. Specify the concrete type on the declaration side, use the empty diamond `<>` on the instantiation side.
- **Casting**: the pre-2005 workaround for retrieving from raw lists. Explicitly not taught in 61B; obsolete.
- **Array**: a restricted collection whose size is fixed at creation time, whose elements all share one type, which has no methods, which is accessed with `x[i]`, and which has exactly one "instance variable," `length`.
- **`x.length`**: the size of array `x`. A field, not a method; no parentheses.
- **Map**: a collection of key-value pairs in which each key is guaranteed to be unique. Called a dictionary in Python, an associative array in theoretical CS, a symbol table in Sedgewick and Wayne.
- **`HashMap`**: the most common concrete implementation of `Map`. `TreeMap` is another, covered later.
- **Primitive type**: one of Java's 8 built-in types (`byte`, `short`, `int`, `long`, `float`, `double`, `boolean`, `char`). A variable of a primitive type holds the data itself.
- **Reference type**: any type that is not one of the 8 primitives, including all classes and **all arrays**. A variable of a reference type holds a 64-bit address, or `null`.
- **Box notation**: drawing a variable as a labeled box containing the literal bits it holds.
- **Simplified box notation**: the same, but with human-readable symbols in the box instead of binary.
- **Box and pointer notation**: simplified box notation for references, where an all-zero address is drawn as `null` and any non-zero address is drawn as an arrow pointing at the object instance.
- **The Golden Rule of Equals (GRoE)**: `y = x` copies all the bits from `x` into `y`. True for every type and every assignment in Java. (A 61B coinage, not official Java terminology.)
- **Pass by value**: parameter passing that copies the bits of the argument into the parameter's new box in the callee's scope. Java is always pass by value.
- **`null`**: the special reference value corresponding to an address of all zeros.
- **Undefined vs. null**: a *declared but unassigned* reference variable is undefined (Java refuses to let you read it). A variable explicitly set to `null` holds a legitimate all-zeros address.
- **Instantiation**: creating an object with `new`, which allocates boxes for every instance variable, fills them with defaults, runs the constructor, and returns the 64-bit address of the object.
- **2D array**: an array whose entries hold the addresses of other arrays.

---

## Worked Examples

### Example 1: The old-school list failure

```java
import java.util.ArrayList;
import java.util.List;

void main() {
   List L = new ArrayList();
   L.add("a");
   L.add("b");
   String x = L.get(0);   // COMPILE ERROR
}
```

**What happens:** `L` is a raw `List`, so the compiler models it as holding `Object`s. `L.add("a")` is fine because a `String` is an `Object`. But `L.get(0)` has *static type* `Object`. The compiler sees `String x = <an Object>;` and refuses: "Required type: String, Provided: Object."

**Why it matters:** the compiler checks types before the program runs. It does not know or care that the thing at index 0 *happens* to be `"a"` at runtime. It only knows the declared types.

**The 61B fix:**

```java
List<String> L = new ArrayList<>();
L.add("a");
L.add("b");
String x = L.get(0);   // compiles
```

Now `get` has static type `String`. Note also that `L.add(3)` would now be a compile error, and that you cannot write `List<int>`: it must be `List<Integer>`.

### Example 2: The array demo from lecture (`ArrayDemo.java`)

```java
void main() {
    String[] x = new String[5];
    x[0] = "a";
    x[1] = "b";
}
```

**Step by step:**
1. `String[] x` declares a 64-bit reference variable that can hold the address of a `String` array.
2. `new String[5]` allocates 5 boxes, each 64 bits (each holds a `String` *reference*), fills each with the default `null`, and returns the address of the whole array object.
3. The GRoE copies that address into `x`. Box-and-pointer: `x` is a box with an arrow pointing at a row of 5 boxes, each containing `null`.
4. `x[0] = "a";` copies the address of the string `"a"` into box 0.
5. `x[2]`, `x[3]`, `x[4]` remain `null`. `x.length` is 5 and will be 5 forever; you cannot grow or shrink the array.

### Example 3: The map demo from lecture (`MapDemo.java`)

```java
import java.util.HashMap;
import java.util.Map;

void main() {
    Map<String, Integer> m = new HashMap<>();
    m.put("cat", 5);
    m.put("dog", 917);
    IO.println(m.get("cat"));    // 5
    IO.println(m.get("aaron"));  // null
}
```

**Step by step:**
1. `Map<String, Integer> m` declares a reference variable. Two type arguments: key type first (what you map *from*), value type second (what you map *to*).
2. `new HashMap<>()` creates a concrete hash map; its address is copied into `m`.
3. `m.put("cat", 5)` associates key `"cat"` with value `5`. In Python this would be `m["cat"] = 5`; Java has no bracket syntax here.
4. `m.get("cat")` returns `5`.
5. `m.get("aaron")` returns `null`, since that key was never put. This is a behavior to remember: a missing key does not throw, it gives `null`. (`getOrDefault` exists if you want a fallback, though the lecture only mentioned it in passing.)

### Example 4: Mystery of the Walrus, fully traced

```java
Walrus a = new Walrus(1000, 8.3);
Walrus b;
b = a;
b.weight = 5;
System.out.println(a);   // weight: 5, tusk size: 8.30
System.out.println(b);   // weight: 5, tusk size: 8.30
```

with

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

**Environment trace in words:**

1. `new Walrus(1000, 8.3)` allocates two boxes for the new object: `weight` (32 bits) and `tuskSize` (64 bits), both set to 0 by default. The constructor runs, copying the bits of `w` into `weight` and the bits of `ts` into `tuskSize` (both by the GRoE). `new` returns the object's 64-bit address.
2. `Walrus a = ...` creates a 64-bit box labeled `a` and copies that address in. Picture: box `a`, arrow to a walrus object containing `weight: 1000, tuskSize: 8.3`.
3. `Walrus b;` creates a second 64-bit box. It is **undefined**, not `null`. Java will not let you read it yet.
4. `b = a;` by the GRoE copies the 64 bits of `a` into `b`. Visually, `b` gets a copy of the arrow: both boxes point at the **same** object. **No second walrus exists.**
5. `b.weight = 5;` follows `b`'s arrow to the one object and writes 5 into its `weight` box.
6. Both print statements follow arrows to the same object, so both print weight 5.

### Example 5: The primitive counterpart

```java
int x = 5;
int y;
y = x;
x = 2;
System.out.println("x is: " + x);   // x is: 2
System.out.println("y is: " + y);   // y is: 5
```

**Trace:**
1. `int x = 5;` creates a 32-bit box labeled `x` holding the bits for 5. There is **no arrow** here, just a literal 5 in the box.
2. `int y;` creates a second 32-bit box, currently unusable.
3. `y = x;` copies the 32 bits from `x` into `y`. Now two independent boxes each hold 5.
4. `x = 2;` refills `x`'s box with the bits for 2. `y`'s box is untouched.

**The unifying point:** steps 3 in Examples 4 and 5 are *the same operation*. Both copy the bits. The observable difference comes entirely from *what the bits mean*: in Example 4 the bits are an address, so copying them creates an alias; in Example 5 the bits are a number, so copying them creates an independent copy.

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

**Trace:**
1. `main` has boxes `x` (64 bits, 5.5) and `y` (64 bits, 10.5).
2. Calling `average(x, y)` creates a **new scope** with two brand-new boxes named `a` and `b`. The bits of `x` are copied into `a`; the bits of `y` are copied into `b`.
3. `average` computes 8.0 and returns it. The returned bits are copied into `avg`.
4. If `average` had done `a = 0;`, `main`'s `x` would still be 5.5, because that assignment only refills `a`'s box.

### Example 7: `doStuff` (the "piano" poll, posed at the end of lecture)

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

**Answer: B, the walrus loses 100 lbs but `main`'s `x` does not change.**

**Why, using only the GRoE:**
- Passing `walrus` copies its **64-bit address** into `doStuff`'s parameter `W`. `W` and `main`'s `walrus` are two boxes holding the same address, so both arrows point at the same object. `W.weight = W.weight - 100;` follows the arrow and mutates the one shared object, so `main` sees weight 3400.
- Passing `x` copies its **32 bits of value** into `doStuff`'s local `x`. Those are two independent boxes that happen to hold the same number. `x = x - 5;` refills the *local* box with 4. `main`'s `x` is still 9.

Note that `doStuff`'s `x` shadows nothing dangerous; it is simply a different box in a different scope that happens to share a name.

### Example 8: 2D array trace

```java
int[][] theMatrix;
theMatrix = new int[4][3];
theMatrix[0] = new int[]{6, 1, 2};
theMatrix[1] = new int[]{7, 1, 2};
theMatrix[2] = new int[]{1, 2, 3, 4};

int[] rowZero = theMatrix[0];
rowZero[1] = -5;
```

**Trace:**
1. `int[][] theMatrix;` creates a 64-bit box that can hold the address of an int-array-array.
2. `new int[4][3]` creates the outer array of 4 boxes (each holding a reference), **and** four separate length-3 int arrays filled with zeros, and stores their four addresses in the outer boxes. Five objects total.
3. `theMatrix[0] = new int[]{6, 1, 2};` builds a brand-new length-3 array holding 6, 1, 2 and copies its address into outer box 0, overwriting the address of the all-zeros array that was there. That old array is now unreachable.
4. Same for `theMatrix[1]`.
5. `theMatrix[2] = new int[]{1, 2, 3, 4};` stores the address of a **length-4** array. Rows need not have equal length.
6. `int[] rowZero = theMatrix[0];` copies an *address* (GRoE). `rowZero` and `theMatrix[0]` are now aliases for the same array.
7. `rowZero[1] = -5;` writes -5 into that shared array, so `theMatrix[0]` is now `{6, -5, 2}`.

### Example 9: Bonus slide exercise

```java
int[][] x = {{1, 2, 3}, {4, 5, 6}, {7, 8, 9}};
int[][] z = new int[3][];
z[0] = x[0];
z[0][0] = -z[0][0];
```

**What is `x[0][0]` at the end? Answer: -1.**

- `new int[3][]` creates only the outer array of 3 reference boxes, each `null`. No inner arrays.
- `z[0] = x[0];` copies the *address* of `x`'s first row into `z[0]`. `z[0]` and `x[0]` are aliases for the same length-3 array.
- `z[0][0] = -z[0][0];` reads 1, negates it, writes -1 into the shared array.
- Therefore `x[0][0]` is now -1. (The answer choices on the slide were listed as "x: 1" twice and "Other"; the point is that the aliasing makes the naive answer wrong.)

---

## Common Pitfalls

1. **Writing `List<int>`.** Does not compile. Use `List<Integer>`. (Project Valhalla may change this someday; not yet.)
2. **Putting the type on the instantiation side in 61B style.** Write `new ArrayList<>()`, not `new ArrayList<String>()`. Writing `new ArrayList()` with nothing compiles but is bad style and IntelliJ will warn.
3. **Forgetting that a raw `List` gives back `Object`.** `String x = rawList.get(0);` will not compile, and casting is not the 61B answer.
4. **Calling `x.length()` on an array.** Arrays have **no methods**. It is `x.length`, a field, no parentheses. (Confusingly, `String` *does* have a `.length()` method. Arrays do not.) *(extra context: the `String` contrast is not stated in the lecture but is a classic source of confusion.)*
5. **Trying to grow or shrink an array.** Impossible. Size is fixed at creation. If you need a changing size, use a `List`.
6. **Using the brace literal without a declaration.** `int[] z = {9, 10};` is legal; `z = {9, 10};` on a later line is not. You must write `z = new int[]{9, 10};`.
7. **Assuming `b = a` copies an object.** It copies bits. For references those bits are an address, so you get a second name for the same object, not a clone.
8. **Confusing "undefined" with "null".** After `Walrus b;`, `b` is undefined and Java will not let you read it. That is different from `b = null;`, which is a legal all-zeros address.
9. **Believing Java is pass by reference.** It is always pass by value. Mutating an object through a parameter works because the *address* was copied, not because the variable was shared. Reassigning the parameter itself (`W = new Walrus(...)`) would not affect the caller.
10. **Forgetting that a map `get` on a missing key returns `null`.** It does not throw. `int numCats = m.get("aaron");` would then fail at runtime when unboxing `null`. *(extra context: the unboxing detail is beyond this lecture; the lecture only showed that the printed result is `null`.)*
11. **Miscounting arrays created by `new int[4][4]` vs `new int[4][]`.** The first creates 5 arrays; the second creates 1, with 4 `null` slots.
12. **Assuming 2D array rows must be the same length.** They need not be, unless you specify both dimensions at creation.
13. **Thinking a variable's declared size tells you the object's size.** Every reference variable is 64 bits regardless of how big the object is.

---

## Likely Exam Points

### 1. Box and pointer tracing after aliasing

**Q.** After the following code, what is printed?

```java
Walrus a = new Walrus(1000, 8.3);
Walrus b = a;
b = new Walrus(200, 2.0);
System.out.println(a.weight);
```

**A.** `1000`. The second line aliases `b` to the same walrus, but the third line copies a *new* address into `b`'s box, redirecting `b`'s arrow to a brand-new walrus. `a`'s box was never touched, so `a` still points at the original object with weight 1000. (Contrast with `b.weight = 200;`, which would have changed what `a` sees.)

### 2. Stating and applying the GRoE

**Q.** State the Golden Rule of Equals, and use it (and nothing else) to explain why mutating an object through a parameter is visible to the caller while reassigning an `int` parameter is not.

**A.** GRoE: `y = x` copies all the bits from `x` into `y`; this holds for every assignment and for parameter passing. For an object parameter, the bits copied are the 64-bit *address*, so the parameter box and the caller's box hold the same address and their arrows point at one shared object; writing through either arrow is visible to both. For an `int` parameter, the bits copied are the *value*, so the two 32-bit boxes are independent; refilling the callee's box leaves the caller's untouched.

### 3. The `doStuff` question

**Q.** Given the `doStuff` code in Worked Example 7, does the call affect `walrus`, `main`'s `x`, both, or neither?

**A.** `walrus` loses 100 lbs (final weight 3400); `main`'s `x` stays 9. The address of the walrus was copied, so the callee mutates the shared object. The value 9 was copied, so the callee's `x = x - 5` only modifies its own local box.

### 4. Primitive vs. reference types

**Q.** Which of these are reference types: `int`, `int[]`, `Integer`, `char`, `String`, `double`, `boolean[]`?

**A.** Reference types: `int[]`, `Integer`, `String`, `boolean[]`. Primitives: `int`, `char`, `double`. Remember the rule: there are exactly 8 primitives (`byte`, `short`, `int`, `long`, `float`, `double`, `boolean`, `char`), and **everything else, including all arrays, is a reference type.**

### 5. How many bits / how many objects

**Q.** After `Walrus[] herd = new Walrus[3];`, how many `Walrus` objects exist? How many bits does the variable `herd` occupy?

**A.** Zero `Walrus` objects exist; the array holds 3 reference slots, each filled with the default `null`. The variable `herd` is 64 bits, like every reference variable, since it holds only an address.

### 6. 2D array creation counting and aliasing

**Q.** How many total arrays are created by `int[][] a = new int[3][];` versus `int[][] b = new int[3][5];`? Then, given `int[] r = b[0]; r[0] = 42;`, what is `b[0][0]`?

**A.** `new int[3][]` creates **1** array (3 `null` slots). `new int[3][5]` creates **4** arrays (one outer of length 3, three inner of length 5, all zeros). `r = b[0]` copies an address, so `r` and `b[0]` alias the same array; `b[0][0]` is **42**.

### 7. Generic list type checking

**Q.** Which lines fail to compile?

```java
List<String> L = new ArrayList<>();   // (1)
L.add("hello");                        // (2)
L.add(7);                              // (3)
String s = L.get(0);                   // (4)
int n = L.get(0);                      // (5)
```

**A.** Lines (3) and (5). Line (3) adds an `int` to a `List<String>`. Line (5) tries to put a `String` into an `int` box. Lines (1), (2), (4) are fine, and (4) is exactly the thing that the old raw-`List` style could not do.

### 8. Map syntax and behavior

**Q.** Write Java equivalent to the Python below, then say what `m.get("bird")` returns.

```python
census = {}
census["cat"] = 103
census["dog"] = 5
num_cats = census["cat"]
```

**A.**
```java
Map<String, Integer> m = new HashMap<>();
m.put("cat", 103);
m.put("dog", 5);
int numCats = m.get("cat");
```
`m.get("bird")` returns `null`, since that key was never inserted.

### 9. Arrays vs. lists tradeoff (conceptual/short answer)

**Q.** Give one reason Java includes arrays even though lists have strictly more features, and one reason Java gives arrays special bracket syntax.

**A.** Arrays are more performant: reading and writing is faster and they use less memory, because they are compact and lack the level of indirection lists involve. Arrays get the special syntax partly for historical reasons (arrays ~1995 predate `List` ~1998) and partly because arrays are a lower-level primitive closer to Java's virtual machine, reflecting Java's performance-oriented design philosophy.

### 10. Why single-typed lists are a benefit

**Q.** Going from Java 4.0 to Java 5.0 lists lost the ability to store mixed types. Why is that a good thing?

**A.** It restricts the set of choices you have to make as a programmer. Freedom leads to complexity, and complexity is hard to fit in your head. You always know exactly what a list holds, so code operating over it cannot break on a surprise element. This "limit yourself on purpose" idea is a recurring theme of 61B.

---

## Summary

- **Modern lists** use angle brackets: `List<String> L = new ArrayList<>();`. Specific type on the declaration side, empty `<>` on the instantiation side. All elements must share one type. Retrieval is `L.get(i)`, not `L[i]`. Raw `List` gives back `Object` and cannot be assigned to a `String`; casting is the obsolete fix and is not taught here.
- **Restricting yourself is good.** Less freedom means less complexity, more robustness, and code that fits in your brain. Recurring 61B theme.
- **Arrays** are lists with fewer features: fixed size set at creation, one element type, no methods, bracket access, and exactly one field `x.length`. Three creation forms: `new int[3]`, `new int[]{...}`, and `int[] z = {...}` (the last only when declaring on the same line). They exist because they are faster and smaller.
- **Maps** are collections of unique key-value pairs (dictionary / associative array / symbol table). `Map<String, Integer> m = new HashMap<>(); m.put(k, v); m.get(k);`. A missing key gives `null`. A map generalizes a list from integer keys to arbitrary keys. `TreeMap` is an alternative implementation coming later.
- **All memory is bits**; types tell Java how to interpret them (`01001000` is both `72` and `'H'`). 8 primitives: `byte`, `short`, `int`, `long`, `float`, `double`, `boolean`, `char`.
- **Declaring a variable** reserves exactly enough bits, records the name-to-location mapping, and writes **nothing**. Java forbids reading uninitialized variables.
- **The Golden Rule of Equals:** `y = x` copies all the bits from `x` into `y`. Always. Every type. Every assignment.
- **Reference types** are everything but the 8 primitives, including arrays. A reference variable is always **64 bits** and holds either `null` (all zeros) or the address returned by `new`. `new` allocates boxes for each instance variable, zero-fills them, runs the constructor, and returns the address.
- **Box and pointer notation:** `null` for all-zero addresses, arrows for real ones. `b = a` copies the arrow, producing an alias, not a copy of the object. A declared-but-unassigned reference is **undefined, not null**.
- **Parameter passing is the GRoE too**: Java is always **pass by value**. Copying an address lets the callee mutate the shared object; copying a primitive value does not let the callee affect the caller's variable.
- **2D arrays are arrays of array addresses.** `new int[4][6]` creates 5 arrays; `new int[4][]` creates 1. Rows can have different lengths. Grabbing a row into an `int[]` variable copies an address, so mutations are shared.
