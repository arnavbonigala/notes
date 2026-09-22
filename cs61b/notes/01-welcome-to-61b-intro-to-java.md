<!-- Wed, Aug 26, 2026 | sources: slides + code + YouTube auto-transcript + your recording + textbook -->
# Lecture 1: Welcome to 61B, Intro to Java

## Overview

This first lecture does two things: it frames what CS 61B is about, and it introduces Java by translating three tiny Python programs into Java. The framing idea is **layers of abstraction**: in 61A you learned to *use* a list (`x = [3, 4, 5]; x.append(6)`) but never saw how one is built; in 61B you drop down a layer and build lists yourself, two radically different ways, over the first five weeks. The course is about writing code that runs efficiently (good algorithms and data structures) and writing code efficiently by hand (design, build, test, debug, using git, IntelliJ, JUnit/Truth, and command line tools). The language is Java, chosen because it is fast, extremely popular, and has features Python lacks: **static typing**, **arrays**, and **subtype polymorphism**. The technical core of the lecture is three demos (`HelloWorld`, `HelloNumbers`, `LargerDemo`) that surface Java's syntax rules (curly braces, semicolons, a `main` function) and, more importantly, the central idea that Java variables and functions have **declared types that never change and are checked by the compiler before the program ever runs**. The lecture closes with logistics (grading, phases, attendance, the LLM policy) and the IntelliJ workflow, with HW1 (set up your computer) due Friday.

---

## Key Concepts

### 1. Layers of abstraction, and where 61B sits

In 61A you wrote `x = [3, 4, 5]` and `x.append(6)` without ever asking what happens underneath. Josh's point: someone actually wrote that list. CPython's `listobject.c` is roughly 4,000 lines of real code sitting under the one-line abstraction you used. 61B moves you one layer down: you become the person who implements the list, not just the person who calls it.

He illustrated this with **Plato's allegory of the cave**: the people in the cave see shadows and believe they are the world; behind them, others manipulate the objects casting those shadows. The Python programmer using a list lives with the shadows; in 61B you turn around and see the machinery. And the layers keep going, 61C asks how the computer itself works.

The important nuance: you are not learning to build lists because your job will involve building lists (that work is long done). Lists and maps are **exemplars** of important solved problems. Working through them by hand builds habits of decomposition and reasoning that transfer to any system you build later.

### 2. What the course is actually about

Two goals, stated as a pair:

- **Writing code that runs efficiently**: good algorithms, good data structures.
- **Writing code efficiently by hand**: designing, building, testing, and debugging large programs, plus tooling (git, IntelliJ, JUnit/Truth, command line tools). The phrase "by hand" is deliberate this semester: you are learning the flow state of writing programs yourself, not orchestrating an agent.

Assumed background: object oriented programming, recursion, lists, maps, and trees. ("Maps" is the 61B word for what you called a Python dictionary. Trees are less vital since the course picks them up along the way; OOP is the main hurdle if you came from E7.)

AI workflows will not be covered in depth, though there may be an optional project or a couple of labs for the curious.

### 3. Why Java

- Runs much faster than Python.
- Has language features Python lacks: **static typing**, **arrays** (fixed length, unlike a list), **subtype polymorphism** (a tool for managing complexity, covered later).
- Extremely popular.

The language arc across the lower division: 61A uses Python, Scheme, SQL; 61B uses Java; 61C uses C and assembly.

### 4. Java's surface rules (from Hello World)

Three syntactic facts, discovered by watching the compiler complain:

1. **Curly braces `{ }` delimit the beginning and ending of things** (functions, loops, if-blocks), where Python uses indentation. A consequence: you can legally cram an entire Java program onto one line and it still works. Indentation in Java is for humans only.
2. **Statements end with a semicolon.**
3. **The code you want to run must be inside a function called `main`.** As a student put it in lecture, `main` is the **entry point**: it tells Java what is actually supposed to run.

A historical note from the live demo: Josh first typed a bare `IO.println(...)` and got `class, interface, annotation type, enum, record, method, or field expected`, then wrapped it in `public class HelloWorld`, then got `illegal start of type` until he added `void main()`. He then pointed out that in modern Java **the class is technically no longer required**, and the rest of the lecture's code has no class at all. (In older Java, and in code you may have seen before, everything had to live inside a class.)

`IO.println` is the course's printing call. "IO" is the input/output library; Java makes you say *whose* `println` it is. The older, more verbose `System.out.println` also works. `println` adds a newline; `IO.print` does not (in the demo, using `print` put all ten numbers on one line).

### 5. Declaration before use (from Hello Numbers)

In Python, `x = 0` creates `x` on the spot. In Java, the compiler rejected `x = 0` with `cannot find symbol: variable x`. Before you can use a Java variable, you must **declare** it: write its **type**, then its **name**, then a semicolon.

```java
int x;   // declaration
x = 0;   // assignment
```

You may combine the two into `int x = 0;`. The lecture code split them apart deliberately, for pedagogical reasons, to show that declaration and assignment are two separate operations. That distinction matters later when you reason about what a variable holds versus what box exists.

**Box-and-pointer intuition in words (building on lecture's framing):** think of `int x;` as the compiler setting aside a labeled box, sized and shaped for integers, with the label `x`. The box exists but contains nothing usable yet. `x = 0;` puts a 0 into that box. The box's *shape* (its type) is fixed at declaration time and can never be reshaped; only the contents can change, and only to something that fits an `int`-shaped box. This is exactly why `x = "horse";` fails: you cannot put a horse in an integer box.

### 6. Static typing: the heart of the lecture

> **Java is statically typed:** all variables, parameters, and methods have a declared type; that type can never change; expressions also have types (e.g. `larger(5, 10) + 3` has type `int`); and the compiler checks that all types in your program are compatible **before the program ever runs**.

The demo that makes this vivid: take the same badly typed operation and run it in both languages.

- **Python** (`hello_numbers.py`): the program prints `0 1 2 3 4 5 6 7 8 9`, and *then* crashes on `x = x + "horse"`. The type check happened **during** execution.
- **Java**: the program prints **nothing**. It never runs at all. The compiler refuses it.

Josh pushed the class past the surface answer ("one complains about converting, one about adding") to the fundamental one: **Python runs and then explodes.** That is dangerous, because if you ship an app to someone's phone, a lurking type error can crash on the user's device. In Java, the compiler analyzes the program first and gives it a stamp of approval saying all the types are fine.

Also note: `String x = "horse";` after `int x = 0;` also fails. You cannot redeclare `x` with a different type. The type is static, meaning unchanging.

### 7. Static typing: the tradeoff

**The good (Josh's answers):**
- Catches certain classes of errors, making debugging easier.
- Type errors can (almost) never occur on the end user's computer.
- Makes code easier to read and reason about. (Josh's industry anecdote: mixed-up parameter types were a major real-world problem, and after 61B you will go back to Python wishing you knew your arguments' types.)
- Code can run more efficiently, since there is no need for expensive runtime type checks. (More on why in 61C.)

**The bad:**
- Code is much more verbose.
- Code is less general. A `larger` that takes two `int`s cannot compare `5.5` or two strings; you would need a separate function for each type.

### 8. Defining functions in Java (from Larger)

- There is **no `def` keyword**. You write the **return type**, then the name, then the typed parameters.
- **Every parameter must have a declared type.**
- **The return value must have a declared type**, and a Java function returns only **one** value. Use `void` when there is nothing to return.
- If conditions must be in parentheses: `if (x > y) { ... }`, not `if x > y:`.
- The lecture notes there will be "alternate ways of defining functions later."

Josh's framing for the math-minded: a Java function feels more like a mathematical function, with an explicit domain and codomain. `int larger(int x, int y)` has domain "pairs of integers" and returns an integer.

### 9. Compilation (skipped this semester)

The slides contained a compilation section, and the Fall 2026 slide deck explicitly says: **"We skipped these slides. We'll return to them in a much later lecture."** For completeness, the content is:

Java separates compilation and interpretation into two steps: `Hello.java` goes through the compiler (`javac`) to produce `Hello.class`, which the interpreter (`java`) then executes. Why produce a `.class` file at all?
- It has been type checked, so distributed code is safer.
- `.class` files are simpler for a machine to execute, so distributed code is faster.
- Minor benefit: it protects intellectual property, since you need not give out source. (Though `.class` files are easily reversible into similar-looking Java files.)

The command line version (`javac HelloWorld.java`, then `java HelloWorld`) was described as "a very old school (but sometimes useful) way of interacting with Java code. We won't do this in 61B."

### 10. Workflow: IntelliJ

Three workflow styles were compared:
- **Text editor plus command line** (61A, CS88): write in the editor, run on the command line. This is what Josh used in the live demo, with Sublime Text.
- **Jupyter notebooks** (Data 8, E7): write and run in the same environment.
- **Integrated Development Environment** (61B): write and run in the same environment, plus a debugger, autocomplete, continuous syntax checking, decompilation from `.class` to `.java`, and more. The screenshot's highlighted feature: IntelliJ automatically and continuously detects syntax errors as you type.

**Admonition from the slides:** the expectation is that everyone uses IntelliJ. It is not strictly required, but staff will provide **no support** for other tools or workflows.

---

## Definitions

- **Abstraction layer**: a level of a system that you use through an interface without knowing its implementation. Using `x.append(6)` is one layer; implementing what `append` does is the layer below.
- **Statically typed language**: a language in which every variable, parameter, and method has a declared type, that type never changes, and the compiler verifies type compatibility before the program runs. Java is statically typed.
- **Dynamically typed language**: (contrast drawn in lecture; Josh called Python "duck typed") a language where type checks are performed *during* execution, so a variable can hold values of different types over its lifetime and type errors surface at runtime.
- **Declaration**: a statement introducing a variable by stating its type followed by its name, e.g. `int x;`. Required before a variable can be used in Java.
- **Type**: the kind of value a variable, parameter, expression, or return value holds, e.g. `int`, `String`. Expressions have types too: `larger(5, 10) + 3` has type `int`.
- **`main`**: the function in which the code you want to run must live; the program's **entry point**.
- **`void`**: the return type written when a function returns no value, as in `void main()`.
- **Semicolon**: the statement terminator in Java; every statement ends with one.
- **Curly braces `{ }`**: delimiters marking the beginning and ending of things (functions, loops, conditionals). Java's replacement for Python's significant indentation.
- **`IO.println`**: the course's output call, printing its argument followed by a newline. `IO` is the input/output library. `IO.print` omits the newline. `System.out.println` is the older-school equivalent.
- **Array**: a Java data structure with a **fixed length**, distinguishing it from a Python list. (Named in lecture as a reason to use Java; covered in detail later.)
- **Subtype polymorphism**: a Java feature described as a tool for managing complexity, absent in Python, to be covered later in the course.
- **Compiler** (`javac`): the tool that turns `.java` source into a type-checked `.class` file. (From the skipped section.)
- **Interpreter** (`java`): the tool that executes a `.class` file. (From the skipped section.)
- **IDE (Integrated Development Environment)**: a single program for both writing and running code, with a debugger, autocomplete, continuous syntax checking, and decompilation. 61B uses IntelliJ.

---

## Worked Examples

All three are from the lecture code repository: https://github.com/Berkeley-CS61B/lectureCode-fa26

### Example 1: Hello World

**Python (`hello_world.py`):**
```python
print("hello world!!")
```

**Java (`HelloWorld.java`):**
```java
void main() {
	IO.println("Hello World");
}
```

**Step by step, following the live demo's discovery process:**

1. Josh first typed just the printing line with nothing around it. The compiler said `class, interface, annotation type, enum, record, method, or field expected`. Meaning: a Java file cannot simply be a loose line of code that does something. It must contain a declaration of some kind.
2. He wrapped it in `public class HelloWorld { ... }`. Now the error became `illegal start of type`, because a bare statement still cannot float directly inside a class body.
3. He added `void main() { ... }` around the print. Now it runs.
4. He then noted that in modern Java the surrounding class is **technically not required**, which is why the file as posted is just `void main() { ... }`.

**Why each piece is there:**
- `void` is the return type: `main` gives nothing back.
- `main` is the entry point; the runtime looks for it to know what to execute.
- `{` and `}` mark where the function body begins and ends, since Java does not read your indentation.
- `IO.println` is a call into the input/output library. Java will not accept a bare `print`; you must say whose `println` this is.
- The `;` ends the statement.

Running it prints `Hello World` and a newline.

### Example 2: Hello Numbers

**Python (`hello_numbers.py`):**
```python
x = 0
while x < 10:
	print(x)
	x = x + 1

x = x + "horse"
```

**Java (`HelloNumbers.java`):**
```java
void main() {
	// before we can use a variable
	// in Java, we must declare it
	// we'll always do so by
	// saying its type and then its name 
	// and then a semi-colon
	int x;

	x = 0;
	while (x < 10) {
		IO.println(x);
		x = x + 1;		
	}
}
```

**Step by step, again following the compiler errors as Josh hit them:**

1. He transliterated the Python directly, adding braces and semicolons, and wrote `while x < 10`. Error: **`'(' expected`**. Java requires parentheses around the loop condition. (Josh's joke: "Python requires you to put parentheses, and the compiler is good about yelling at you about it," meaning Java requires them.) Fix: `while (x < 10)`.
2. Next error: **`cannot find symbol: variable x`**. In Python, `x = 0` both creates and assigns. In Java, `x` was never introduced, so the compiler does not know what `x` is. He noted the message is not especially helpful: it does not point you at "x on line 2, column 5."
3. Fix: **declare** `x` first. `int x;` sets up an integer-shaped box named `x`. Then `x = 0;` fills it. The program now runs and prints `0` through `9`, one per line.
4. Josh showed `IO.print` instead of `IO.println`: same values, but all on a single line, since `print` does not emit a newline.
5. He confirmed a student's question: yes, you can compress declaration and assignment into `int x = 0;`. He was splitting them only to show that they are two distinct processes.

**Environment / box reasoning in words:** trace the loop. After `int x;` there is an `int` box labeled `x`, uninitialized. `x = 0;` makes it hold 0. The condition `0 < 10` is true, so the body runs: print 0, then `x = x + 1` evaluates `0 + 1` to 1 and stores 1 back in the same box. This repeats. When `x` holds 10, `10 < 10` is false, the loop exits, and `main` ends. The box was never reshaped, only refilled, ten times.

**The type-failure experiment (the point of the whole example):**

```java
int x = 0;
while (x < 10) {
    IO.println(x);
    x = x + 1;
}
x = "horse";        // doesn't work: cannot assign a String to an int variable
String x = "horse"; // doesn't work: x is already declared, and its type cannot change
```

Run the Python analog and you see `0 1 2 3 4 5 6 7 8 9` printed, **then** a crash. Run the Java version and you see **nothing at all**, because the program never starts. That difference (crash during execution versus refusal before execution) is the lecture's central takeaway about static typing.

### Example 3: Larger

**Python (`larger.py`):**
```python
def larger(x, y):
	if x > y:
		return x
	return y

print(larger(5, 10))
```

**Java (`LargerDemo.java`):**
```java
int larger(int x, int y) {
	if (x > y) {
		return x;
	}	
	return y;
}

void main() {
	IO.println(larger(5, 10));
}
```

**Step by step:**

1. Josh first wrote `larger(x, y) { ... }` with no types at all. The compiler complained: Java needs not just names but **types**, for both the parameters and the return value.
2. He added parameter types: `larger(int x, int y)`. Still incomplete, since the return type is missing.
3. He added the return type on the front: `int larger(int x, int y)`. Now it compiles. Note there is **no `def`**; the return type takes that slot.
4. `if (x > y)` needs parentheses, as with `while`.
5. `main` calls `IO.println(larger(5, 10))`. (The slides show `larger(-5, 10)`; the posted code uses `larger(5, 10)`. Either way the answer is 10.)

**Tracing `larger(5, 10)`:** `x` holds 5, `y` holds 10, each in its own `int`-shaped box created for this call. `5 > 10` is false, so the `if` body is skipped and control reaches `return y`, returning 10. That 10 is the value of the expression `larger(5, 10)`, which `IO.println` then prints.

**The limitation demo:** in Python, `larger("a", "z")` happily returns `"z"`, because Python compares strings at runtime and nothing forbids it. In Java:

```java
IO.println(larger("a", "z"));   // does not compile
```

The compiler refuses, because `larger` takes two `int`s and you handed it two `String`s. Josh: "You can't take strings and put them in the integer box." **The code will crash before it even runs.** If you want string comparison, you must write a second `larger` that takes two `String`s. That verbosity and loss of generality is exactly the "bad" column of the static typing tradeoff.

**Expression types:** the slides make the point that expressions have types too. `larger(5, 10) + 3` has type `int`, so `String x = larger(5, 10) + 3;` fails to compile. The compiler can determine this purely by reading the code, without running anything.

---

## Common Pitfalls

- **Forgetting the semicolon.** Every statement needs one. IntelliJ flags this continuously, which is one reason the course insists on an IDE.
- **Relying on indentation for structure.** Java reads only braces. Well-indented code with wrong braces is wrong; badly indented code with right braces is right (Josh demonstrated that you can put an entire program on one line). Indent anyway, for the humans.
- **Writing `while x < 10` or `if x > y` without parentheses.** Java requires `while (x < 10)` and `if (x > y)`.
- **Using a variable without declaring it.** `x = 0;` alone gives `cannot find symbol`. And note Josh's warning: **Java compiler messages are often not very helpful**. `cannot find symbol` does not tell you that you forgot a declaration; you have to learn to read the errors.
- **Assuming a variable's type can change.** Neither `x = "horse";` nor `String x = "horse";` works after `int x = 0;`. The first is a type mismatch; the second is an illegal redeclaration. The type is fixed at declaration forever.
- **Expecting a runtime crash for type errors.** In Java there is no partial output before the failure. Nothing runs. If you are looking for "it printed some numbers and then died," you are thinking in Python.
- **Omitting the return type or the parameter types on a function.** There is no `def`; the return type occupies that position, and every parameter needs its own type.
- **Trying to return more than one value.** Java functions return exactly one value.
- **Forgetting `void`** when the function returns nothing. `main` is `void main()`, not `main()`.
- **Using `IO.print` when you wanted line breaks.** Use `IO.println`.
- **Writing bare `print` or `println`.** Java wants to know whose: `IO.println` (or the older `System.out.println`).
- **Assuming you still need `public class Whatever` around everything.** In modern Java you do not, and the lecture's code omits it. If you have prior Java experience, this may look wrong to you; it is not.

---

## Likely Exam Points

### 1. Static versus dynamic typing: when are type errors caught?

Commonly tested by asking you to predict program *output*, not just whether it errors.

**Practice question.** Two programs do the same thing. The Python version:
```python
x = 0
while x < 3:
    print(x)
    x = x + 1
x = x + "horse"
```
The Java version:
```java
void main() {
    int x = 0;
    while (x < 3) {
        IO.println(x);
        x = x + 1;
    }
    x = x + "horse";
}
```
What does each print?

**Answer.** Python prints `0`, `1`, `2`, and then crashes with a runtime type error, because Python checks types during execution. Java prints **nothing**: the compiler rejects the program before any code runs, so there is no partial output. This is the key difference, and the practical consequence is that a Java type error can (almost) never occur on an end user's machine.

### 2. Identifying and fixing Java syntax/type errors in a snippet

**Practice question.** Find all the errors:
```java
void main() {
    x = 0
    while x < 5 {
        IO.println(x)
        x = x + 1
    }
}
```

**Answer.** Four categories of problem:
1. `x` is never declared. Needs `int x = 0;` (or `int x;` then `x = 0;`).
2. Missing semicolons after `x = 0`, after `IO.println(x)`, and after `x = x + 1`.
3. The `while` condition needs parentheses: `while (x < 5)`.

Corrected:
```java
void main() {
    int x = 0;
    while (x < 5) {
        IO.println(x);
        x = x + 1;
    }
}
```

### 3. Writing a correct Java function signature

**Practice question.** Translate this Python function to Java:
```python
def smaller(a, b):
    if a < b:
        return a
    return b
```

**Answer.**
```java
int smaller(int a, int b) {
    if (a < b) {
        return a;
    }
    return b;
}
```
Points to hit: no `def`; the **return type** `int` comes first; **every parameter is individually typed** (`int a, int b`, not `int a, b`); the `if` condition is parenthesized; returns are semicolon-terminated.

### 4. Reasoning about expression types

**Practice question.** Given `int larger(int x, int y) { ... }`, does `String s = larger(5, 10) + 3;` compile? Why or why not?

**Answer.** No. `larger(5, 10)` has type `int`, so `larger(5, 10) + 3` also has type `int`. Assigning an `int` expression to a `String` variable is a type mismatch, and the compiler detects this statically, before running anything. The slides list this exact case.

### 5. The tradeoffs of static typing

**Practice question.** Give two advantages and two disadvantages of static typing.

**Answer.** Advantages: it catches certain error classes early, easing debugging; type errors essentially cannot reach the end user's computer; code is easier to read and reason about since types are documented in the source; code runs more efficiently because no expensive runtime type checks are needed. Disadvantages: code is more verbose; code is less general, for instance you would need a separate `larger` to handle `5.5` or two strings.

### 6. Declaration versus assignment

**Practice question.** What does `int x;` do, and how does it differ from `x = 0;`? Can they be combined?

**Answer.** `int x;` is a **declaration**: it introduces the name `x` and fixes its type as `int` permanently. `x = 0;` is an **assignment**: it stores the value 0 into the already-declared `x`. They can be combined as `int x = 0;`. The lecture separated them to emphasize they are two distinct processes.

### 7. Java structural rules

**Practice question.** Name the three reflections the lecture drew from Hello World.

**Answer.** (1) We use `{ }` to delineate the beginning and ending of things, rather than indentation. (2) Statements end with a semicolon. (3) The code we want to run must be inside a function called `main` (the entry point).

### 8. (extra context) Why the lecture's Java has no `public class`

**Practice question.** Older Java tutorials wrap everything in `public class HelloWorld { public static void main(String[] args) { ... } }`. The lecture's file is just `void main() { IO.println("Hello World"); }`. Is the lecture's version valid Java?

**Answer.** Yes. Josh noted explicitly that **until very recently** Java required the enclosing class, and that technically it is no longer needed; the lecture code from this point on omits it. Knowing both forms is useful since you will encounter the older style in the wild, and `System.out.println` is likewise the older-school equivalent of `IO.println`. *(Marked extra context: this practice framing goes slightly beyond what the lecture asked you to do with the fact.)*

---

## Summary

**Course framing**
- 61B moves you one layer of abstraction down: from *using* a list to *implementing* one. Two radically different list implementations over the first five weeks.
- The course is about writing code that runs efficiently (algorithms, data structures) **and** writing code efficiently by hand (design, build, test, debug; git, IntelliJ, JUnit/Truth, command line tools).
- Assumes OOP, recursion, lists, maps, and trees. AI workflows are not covered in depth.
- Language arc: 61A (Python, Scheme, SQL) → 61B (Java) → 61C (C, assembly).
- Java is used because it is fast, popular, and has static typing, arrays (fixed length), and subtype polymorphism.

**Java syntax**
- `{ }` delimit blocks; indentation is meaningless to the compiler.
- Statements end with `;`.
- Runnable code lives in `main`, the program's entry point.
- `if` and `while` conditions require parentheses.
- `IO.println` prints with a newline; `IO.print` does not. `System.out.println` is the older equivalent.
- The wrapping `public class` is no longer required in modern Java, and the lecture code omits it.

**Static typing (the core idea)**
- Variables must be **declared** before use: type, then name, then semicolon (`int x;`).
- Every variable, parameter, and method has a declared type, and that type **never changes**.
- Expressions have types too (`larger(5, 10) + 3` is an `int`).
- The compiler checks all types **before the program runs**. Java refuses to run a badly typed program at all; Python runs and then explodes mid-execution.
- Good: catches errors early, keeps type errors off users' machines, easier to read and reason about, runs faster (no runtime type checks).
- Bad: more verbose, less general (a second `larger` is needed for strings or doubles).

**Functions**
- No `def`. Write the return type, then the name, then fully typed parameters: `int larger(int x, int y)`.
- `void` means no return value. Java functions return exactly one value.

**Workflow and logistics**
- 61B uses **IntelliJ**; other tools are allowed but unsupported by staff.
- Compilation (`javac` producing a type-checked `.class` file, then `java` running it) was on the slides but **skipped**, to be revisited in a much later lecture.
- Three phases: weeks 1-5 Java and data structures (solo, very fast, two mini-projects, programming midterm), weeks 6-10 data structures (solo, one design project, theoretical midterm), weeks 11-15 algorithms and software engineering (paired final project, slower).
- Grading: 6150 points total; surveys/HWs/mini-projects/design projects are effort-based with 100% medians; exams target a 65% mean and are uncurved; the final can replace midterms. Attendance points depend on opting into discussion and/or lab; lecture attendance gives 6 exam recovery points each (max 150), helpful only if your exam score is under 70%.
- Late work: 5% off per 12 hours; capped point categories absorb small losses. Weekly surveys get no extensions (lowest 4 effectively dropped).
- **LLM policy:** use LLMs minimally. No IntelliJ LLM plugins, no Cursor or Claude Code. LLMs should write none of the code you turn in. Do not give assignment specs to LLMs. Avoid "where is the bug?" unless genuinely stuck. Using an LLM as a tutor is reasonable, but talking to staff and classmates is richer. Provenance tracking begins after HW1.
- **Action items:** HW1 (set up IntelliJ) due Friday 11:59 PM; start HW2 by Friday; Lab 1 is drop-in this week only; fill out the discussion/lab matching form; Bridge Section Mondays 5-7 PM if your foundations are shaky (B or lower in 61A/CS88, or coming from E7).
