<!-- Wed, Aug 26, 2026 | sources: code + YouTube auto-transcript + your recording -->
# Lecture 1: Intro 1

CS 61B opens by locating itself one layer of abstraction below CS 61A: in 61A you *used* Python lists, and in 61B you learn how something like a list is actually built. Josh Hug framed this with Plato's allegory of the cave (the programmer who only sees shadows versus the one who walks behind the screen to see the machinery), and noted that CPython's `listobject.c` is roughly 4,000 lines of code that someone had to write. The course has two goals: writing code that runs efficiently (good algorithms and data structures), and writing code efficiently by hand (design, build, test, and debug large programs using Git, IntelliJ, JUnit, and command line tools). The language is Java, chosen because it is faster than Python, has static typing, has arrays (fixed length, unlike lists), and has subtype polymorphism as a tool for managing complexity. The technical content of the day was a three-program tour comparing Java against Python side by side: `hello_world`, `hello_numbers`, and `larger`. Those three programs surface Java's defining early features: braces instead of indentation, semicolons as statement terminators, code living inside a `main` method, variables that must be declared with an explicit type, function definitions that declare parameter types and a return type, and, most importantly, the fact that Java checks types *before* the program runs rather than blowing up mid-execution the way Python does.

---

## Key Concepts

### 1. Peeling back a layer of abstraction

61A taught programming in layers of abstraction and handed you a `list`. Nobody showed you where the list came from. In 61B the first five weeks are spent building *two radically different ways of implementing a list*. The point is explicitly not that you will build lists professionally (that work is done). Lists and maps are **exemplars**: classic problems whose solutions build habits of mind (decomposition, seeing layers) that transfer to designing any system. If you continue as a CS major, 61C peels back another layer and asks how the computer itself works.

### 2. What the course is actually about

Two things, in Hug's own phrasing:

- **Code that runs efficiently**: good algorithms and good data structures. Asymptotic analysis, resizing arrays, tree structures, some graph theory, and P vs NP at the very end.
- **Code written efficiently, by hand**: getting into the programmer's flow state, and learning to design, build, test, and debug large programs with professional tooling (Git, IntelliJ, JUnit, command line tools).

The target outcome: the confident sense that you could build any software by hand if you chose to, so that even when LLMs generate most of your professional code, you understand what is happening under the hood.

### 3. Assumed background and the LLM policy

Assumed: object-oriented programming, recursion, lists, maps (what Python calls dictionaries), and trees. Trees are less vital since the course rebuilds them. Students from E7 will hit classes as the early hurdle and should use the bridge section.

On LLMs, the stated policy was strong and specific: use them minimally. No LLM plugins in IntelliJ, no Cursor or Claude Code for this class, no LLM-written code turned in no matter how small, do not feed assignment specs to an LLM for a reword (reading specs is part of the skill), and preferably do not paste code in and ask "where is the bug?" Using an LLM as a *tutor* ("I'm confused about hash tables, here's my thinking") is more reasonable, though talking to staff and fellow students is richer. The justification given was empirical: on a past fall paper midterm where students had to write code by hand, roughly 10% of students scored under 10%, attributed to shaky foundations propped up by LLM assistance. The analogy offered: doing LLM-assisted Chinese homework and then being asked to speak Chinese unaided.

### 4. Why Java instead of Python

| Reason | Detail from lecture |
|---|---|
| Speed | Java code runs faster than Python (the *why* is a 61C topic) |
| Static typing | Not taught in the predecessor courses; the central new idea today |
| Arrays | Fixed-length, unlike a Python list |
| Subtype polymorphism | A tool for managing complexity, covered later |
| Popularity | Less popular than Python, but still very widely used |

Java is old-fashioned (nobody has a recent Java tattoo), but it is a solid workhorse that keeps improving.

### 5. Java syntax basics (from `HelloWorld`)

Three structural facts, discovered by watching the compiler reject each intermediate attempt:

- **Braces, not indentation.** Python uses indentation to mark the beginning and end of blocks; Java uses `{` and `}`. A consequence: you can legally cram an entire Java program onto one line and it still works. Indentation in Java is purely for human readers.
- **Semicolons end statements.** Each statement such as `IO.println("Hello World");` ends in `;`.
- **Code must live inside a method, normally `main`.** A bare line of code at file level produces the error `class, interface, annotation type, enum, record, method, or field expected`. A student suggested the term **entry point** for `main`, which Hug accepted: `main` tells the system what is actually supposed to run.

A note on the class wrapper: historically you had to write `public class HelloWorld { ... }` around everything, and if you write only the class with no `main` you get `illegal start of type` style complaints until a method is added. Very recent Java allows a top-level `void main()` with no enclosing class, and the lecture code files use exactly that form, so the rest of the day's examples have no `class` line.

On printing: `IO.println` is the course's printing call. `IO` is the input/output library, and Java will not let you just say `println` on its own because it needs to know *whose* `println`. The older, more traditional spelling `System.out.println` also works and is fine. `print` versus `println`: `println` appends a newline, `print` does not, which is why an early run of the numbers program put everything on one line.

### 6. Variable declaration (from `HelloNumbers`)

In Java, before you can use a variable you must **declare** it. The declaration form used in this class is: **type, then name, then semicolon**.

```java
int x;
```

Skipping this yields `cannot find symbol` / `symbol: variable x`. Hug pointed out that this compiler message is not especially helpful: it does not tell you where the symbol should have been declared. Declaration and assignment are two separate processes conceptually, which is why the lecture wrote them on separate lines, but a student asked whether they could be combined and the answer is yes, `int x = 0;` is legal Java.

Also from this program: `while` in Java requires parentheses around its condition, `while (x < 10) { ... }`. Forgetting them gives a `'(' expected` error, and Hug noted the compiler is good about yelling at you for this one.

### 7. Static typing: the central idea of the lecture

Four properties, as summarized on the reflection slide:

1. Variables must be **declared** before use.
2. Variables have a **specific type**.
3. That type **cannot change**.
4. Types are **verified before the code runs**.

Property 4 is the one Hug pushed hardest on, and he made students find it themselves. Both `hello_numbers.py` and the Java equivalent contain a deliberate type error (`x = x + "horse"`). The Python version **prints 0 through 9 and then crashes**: it ran, produced output, and exploded partway through. The Java version **never runs at all**: the compiler analyzes the program, finds the bad type, and refuses to produce a running program. One student first answered "it's trying to convert it here and complaining about addition there," which is a real difference but not the fundamental one; the fundamental one is *when* the failure happens.

Why this matters beyond the classroom: if you ship an app to someone's phone, a Python-style latent type error can crash on the user's device. Java's compiler gives the program a stamp of approval on types before it ever leaves your machine. Python was described as a duck-typed language where "I could do whatever the hell I want" and make `x` a horse later.

**Benefits of static typing:** prevents whole classes of type bugs from reaching users; code runs faster (details in 61C); code is easier to read and reason about. Hug's industry anecdote: messed-up types of parameters being passed in was a significant real-world problem, and after this course you will miss knowing your argument types when you return to Python.

**Costs of static typing:** verbosity, and you may need more than one function to cover what one Python function covered. `larger` for `int`s will not accept strings, so supporting strings requires writing a second `larger` that takes two `String`s. When Hug asked for the downside, the student answer was "verbosity."

### 8. Function definitions in Java (from `LargerDemo`)

The Java function declaration rule: **return type, then name, then parenthesized parameters each with their own declared type**. There is no `def` keyword. If a function returns nothing, its return type is `void`, which is why `main` is written `void main()`.

Hug's framing: Java functions feel more like mathematical functions. `int larger(int x, int y)` has a domain of pairs of integers (the Cartesian product of the integers with themselves) and a codomain of integers, and the signature states this explicitly. Calling `larger("a", "z")` is rejected at compile time: you cannot put strings in an integer box, so the code crashes before it even runs. The Python `larger` happily handles strings and returns `"z"`.

### 9. Tooling

So far Hug has been editing in Sublime Text and running from the command line, but students will use an **IDE (integrated development environment)**, specifically IntelliJ: a heavier program where you edit and run code in one place. Homework 1 is setting this up.

---

## Definitions

- **Abstraction layer**: a level of description that hides the machinery beneath it. A Python programmer using `list` sits one layer above the C code that implements `list`; 61B moves you one layer down.
- **Declaration**: a statement that brings a variable into existence by stating its type and its name, for example `int x;`. Required in Java before any use of the variable.
- **Static typing**: the property that every variable and expression has a declared type, that the type never changes, and that the types are checked by the compiler before the program is executed.
- **Dynamic typing (duck typing)**: Python's approach, where a variable can hold a value of any type and can be rebound to a different type at any time; type errors surface only when the offending line actually executes.
- **Compiler**: the program that analyzes Java source before execution, reports type errors and syntax errors, and, if everything checks out, produces a runnable program. Failure at this stage means the program never runs.
- **`main`**: the method that serves as the program's **entry point**, the code that actually gets run when you run the program.
- **`void`**: the return type written for a function that returns no value.
- **Return type**: the type of the value a function hands back, written immediately before the function's name in its declaration.
- **`IO.println` / `IO.print`**: the course's output calls. `IO` stands for the input/output library. `println` appends a newline; `print` does not. `System.out.println` is the older equivalent of `IO.println`.
- **Semicolon**: the statement terminator in Java.
- **Curly braces `{ }`**: Java's delimiters for the beginning and end of a block, filling the role that indentation plays in Python.
- **Array**: (previewed only) a Java data structure like a list but with a fixed length.
- **Subtype polymorphism**: (previewed only) a Java feature for managing complexity, to be covered later.

---

## Worked Examples

### Example 1: Hello World, built up error by error

Python side:

```python
print("hello world!!")
```

Java side, as it was reached in class. Attempt 1, a bare statement in a file:

```java
IO.println("Hello World");
```

The compiler responds: `class, interface, annotation type, enum, record, method, or field expected`. Reading that message: Java expects the top level of a file to contain a *declaration* of something, not an *action*. A lone statement has no home.

Attempt 2, wrap it in a class (the historically required form):

```java
public class HelloWorld {
    IO.println("Hello World");
}
```

Still wrong: a statement cannot float directly inside a class body either. The fix is to put it inside a method.

Final form (lecture file `HelloWorld.java`, using the modern no-class style):

```java
void main() {
    IO.println("Hello World");
}
```

Step by step, what this does:

1. `void main()` declares a function named `main` that takes no arguments and returns nothing (`void`).
2. `{` opens the body of `main`.
3. `IO.println("Hello World");` asks the input/output library to print the string and then a newline. The semicolon ends the statement.
4. `}` closes the body.
5. Running the program invokes `main`, the entry point, which prints `Hello World`.

**Why it is shaped this way:** every piece of runnable code in Java lives inside a method, and `main` is where execution begins. Compare to Python, where the file itself is the program and `print(...)` at top level simply runs. Java trades that convenience for structure.

### Example 2: Hello Numbers, and the declaration requirement

Python (`hello_numbers.py`, ignoring the last line for a moment):

```python
x = 0
while x < 10:
	print(x)
	x = x + 1
```

Prints 0 through 9. Note: no declaration, no type, no parentheses, no braces, no semicolons. Indentation defines the loop body.

The Java translation was built by pasting the Python in and fixing what the compiler complained about, in this order:

1. Wrap everything in `void main() { ... }` and convert indentation into braces.
2. Add semicolons to the ends of statements.
3. `print` becomes `IO.print` / `IO.println`.
4. Compiler error `'(' expected` on the `while` line: Java requires parentheses around the loop condition, so `while x < 10` becomes `while (x < 10)`.
5. Compiler error `cannot find symbol, symbol: variable x`: `x` was never declared. There is no way to tell the compiler "x is on line 2, column 5"; you must actually declare it.

Final form (lecture file `HelloNumbers.java`):

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

Reasoning about it in environment/box terms, in words: the declaration `int x;` creates a box labelled `x` that is permanently stamped "holds an `int`" and nothing else. The stamp is applied at compile time and never comes off. `x = 0;` drops the value 0 into that box. Each loop iteration reads the box, prints its contents, computes contents-plus-one, and puts the result back in the same box. After the iteration where `x` becomes 10, the condition `10 < 10` is false and the loop exits. Output: the digits 0 through 9, one per line (with `println`; with `print` they all ran together on one line, which is exactly what happened in class before the fix).

### Example 3: The horse, or when type errors are caught

Python, with the full file:

```python
x = 0
while x < 10:
	print(x)
	x = x + 1

x = x + "horse"
```

Run it. Output:

```
0
1
2
3
4
5
6
7
8
9
Traceback (most recent call last):
  ...
TypeError: unsupported operand type(s) for +: 'int' and 'str'
```

The program **ran**, did nine useful things, and then died. (extra context: the exact traceback text above is the standard CPython message; the lecture only showed that it printed 0 through 9 and then crashed.)

Now the analogous Java, adding `x = x + "horse";` at the end of `main`:

```java
void main() {
	int x;
	x = 0;
	while (x < 10) {
		IO.println(x);
		x = x + 1;
	}
	x = x + "horse";   // rejected by the compiler
}
```

Output:

```
(nothing: the program never runs)
```

The compiler rejects the assignment because the right-hand side is a `String` and `x` is an `int`. Nothing is printed, not even the 0 through 9 that would have come first.

**The subtle, fundamental difference to internalize:** it is not that one language complains about conversion and the other about addition. It is **when** the complaint happens. Python detects the problem at the moment the bad line executes, which may be on a user's device, months after shipping, after the program has already done real work. Java detects it before the program can run at all. The compiler analyzes the whole program and, in Hug's phrase, gives it a stamp of approval that all the types are fine.

(A related smaller demo: writing `x = "horse";` after `int x;` also fails to compile. You cannot put a horse in an `int` box. In Python you can rebind `x` to a horse whenever you like.)

### Example 4: `larger`, and typing function signatures

Python (`larger.py`):

```python
def larger(x, y):
	if x > y:
		return x
	return y

print(larger(5, 10))
```

Prints `10`. And `larger("a", "z")` prints `z`, because `>` works on strings too. One function, any comparable types.

Java, first attempt, transliterated without types:

```java
larger(x, y) {
	if (x > y) {
		return x;
	}
	return y;
}
```

The compiler rejects this. Two things are missing: the **types of the parameters** and the **return type of the function**.

Final form (lecture file `LargerDemo.java`):

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

Step by step:

1. `int larger(int x, int y)` declares a function named `larger`. The leading `int` is the return type: calling this function produces an `int`. `int x` and `int y` declare two parameters, each an `int`. There is no `def`.
2. `if (x > y)` needs parentheses around the condition, same rule as `while`.
3. `return x;` hands the value back and exits the function immediately.
4. If the `if` did not fire, control falls through to `return y;`. No `else` is needed precisely because `return` exits.
5. `main` calls `larger(5, 10)`. Since `5 > 10` is false, the first `return` is skipped and `10` is returned. `IO.println` prints `10`.

**The math framing:** the signature `int larger(int x, int y)` states the function's domain (pairs of integers) and its range (integers) up front, which a Python `def` does not.

**The cost, demonstrated:**

```java
IO.println(larger("a", "z"));   // does not compile
```

The compiler refuses: you declared a function taking two `int`s and you handed it two `String`s. Supporting strings requires writing a second, separate `larger` that takes two `String` parameters and returns a `String`. Python needed only one function. This is the verbosity tax you pay for compile-time safety.

---

## Common Pitfalls

- **Forgetting to declare a variable.** Using `x` without `int x;` first gives `cannot find symbol: variable x`. The message does not tell you where to declare it, so learn to read it as "you never created this box."
- **Forgetting semicolons.** Java statements end in `;`. Indentation and line breaks are irrelevant to the compiler.
- **Using indentation to define blocks.** Pretty indentation with missing braces is a Java program that means something different (or does not compile). Braces alone delimit blocks. Conversely, a correctly braced program crammed onto one line works fine, which is exactly why indentation cannot be trusted as structure.
- **Omitting parentheses around `if` / `while` conditions.** `while x < 10` is a syntax error; `while (x < 10)` is correct.
- **Writing `print` or `println` bare.** Java needs to know whose `println`: `IO.println(...)` (or the older `System.out.println(...)`).
- **Confusing `print` with `println`.** `IO.print` in a loop puts all output on one line. This happened live in lecture.
- **Writing `def`.** There is no `def` keyword in Java. The function declaration begins with its return type.
- **Forgetting the return type, or forgetting `void`.** A function that returns nothing still needs `void` written out.
- **Expecting a Python-style crash.** Do not assume that because your program printed nothing, your logic is broken. If it failed to compile, it never ran. Read the compiler output before assuming the program executed.
- **Assuming one function covers all types.** An `int larger` will not accept `String`s, no matter how sensible `>` would be on them.
- **Expecting a variable to change type.** Once `x` is an `int`, it is an `int` for the rest of its life.

---

## Likely Exam Points

Note: this is the first lecture and its content is foundational rather than heavily examined on its own, but the following are the pieces that recur and that exams reliably lean on. Remember that exams in this course are **on paper**, so handwriting syntactically correct Java is itself the skill being tested.

**1. When are type errors caught, and why does it matter?**

> *Q: A Python program and an equivalent Java program each contain a statement that adds an integer to a string, placed after a loop that prints 0 through 9. Describe the observable difference in behavior when each is run, and explain why the Java behavior is preferable for shipped software.*
>
> A: The Python program runs, prints 0 through 9, and then raises a `TypeError` when it reaches the bad statement. The Java program does not run at all: the compiler detects the type mismatch before execution and refuses to produce a running program, so no output appears. Java's behavior is preferable for shipped software because a latent type error in Python may not surface until the program is running on a user's device, potentially long after release, whereas Java surfaces it on the developer's machine at compile time.

**2. Fix the broken Java.**

> *Q: Identify every error in the following and write the corrected version.*
> ```java
> void main() {
>     x = 0
>     while x < 10 {
>         print(x);
>         x = x + 1;
>     }
> }
> ```
>
> A: Four problems: (a) `x` is never declared, needs `int x;` (or `int x = 0;`); (b) `x = 0` is missing a semicolon; (c) the `while` condition needs parentheses; (d) `print` must be `IO.println` (or `System.out.println`). Corrected:
> ```java
> void main() {
>     int x = 0;
>     while (x < 10) {
>         IO.println(x);
>         x = x + 1;
>     }
> }
> ```

**3. Translate a Python function definition to Java.**

> *Q: Translate the following to Java.*
> ```python
> def larger(x, y):
> 	if x > y:
> 		return x
> 	return y
> ```
>
> A:
> ```java
> int larger(int x, int y) {
>     if (x > y) {
>         return x;
>     }
>     return y;
> }
> ```
> The key additions are the return type `int` before the name, the type `int` on each parameter, the parentheses around the `if` condition, the braces, and the semicolons. The `def` keyword is dropped.

**4. State the properties of static typing.**

> *Q: List the four properties of Java's type system as presented in lecture, and give one advantage and one disadvantage.*
>
> A: (1) Variables must be declared before use; (2) each variable has a specific type; (3) that type cannot change; (4) types are verified before the code runs. Advantage (any one of): type errors cannot reach the user's device, code runs faster, code is easier to read and reason about. Disadvantage: greater verbosity, and you may need multiple versions of a function to cover multiple types (for example a separate `larger` for `String`s).

**5. Why does this not compile?**

> *Q: Given `int larger(int x, int y)` as defined above, what happens when you write `IO.println(larger("a", "z"));` and why?*
>
> A: It fails to compile. `larger` is declared to take two `int` arguments, and `"a"` and `"z"` are `String`s. The type mismatch is caught by the compiler, so the program never runs. To handle strings you would need a separate function taking two `String` parameters.

**6. Structure of a Java program.**

> *Q: Why can a Java file not consist of just the single line `IO.println("Hello World");`? What is the minimal legal structure?*
>
> A: Java requires code to live inside a method rather than at the top level of a file; a bare statement produces an error along the lines of `class, interface, annotation type, enum, record, method, or field expected`. The minimal structure puts the statement inside `main`, the program's entry point: `void main() { IO.println("Hello World"); }`. Historically this also had to be wrapped in a class such as `public class HelloWorld { ... }`.

**7. Conceptual: what is 61B about?**

> *Q: In one or two sentences, what does it mean to say 61B moves you "one layer down" from 61A?*
>
> A: In 61A you used data structures like Python's `list` as black boxes provided to you. In 61B you learn how such structures are implemented, starting with two different ways of building a list, so that you understand and can reason about the machinery beneath the abstractions you use.

---

## Summary

- 61B goes one abstraction layer below 61A: you used `list`, now you learn how a list is built. The first five weeks cover two radically different list implementations, treated as exemplars for thinking about design and decomposition.
- Two course goals: code that **runs** efficiently (algorithms, data structures, asymptotics) and code written efficiently **by hand** (design, build, test, debug with Git, IntelliJ, JUnit, command line tools).
- Java is used because it is faster than Python, is statically typed, has fixed-length arrays, offers subtype polymorphism, and is widely used.
- Java syntax essentials: braces delimit blocks (not indentation), semicolons terminate statements, `if` and `while` conditions need parentheses, and runnable code lives inside a method, with `main` as the entry point.
- Printing is `IO.println` (newline) or `IO.print` (no newline); `System.out.println` is the older equivalent. `IO` is the input/output library.
- Variables must be declared before use: **type, then name, then semicolon**, as in `int x;`. Declaring and assigning on one line (`int x = 0;`) is legal.
- **Static typing** means: variables are declared, have a specific type, that type never changes, and types are checked **before the code runs**.
- The headline contrast: Python runs and *then* explodes on a type error; Java refuses to run at all. Catching errors at compile time keeps them off users' devices.
- Function declarations give a **return type** (or `void`), a name, and explicitly typed parameters: `int larger(int x, int y)`. There is no `def`. Java functions resemble math functions with a stated domain and range.
- The cost of static typing is verbosity and needing separate functions per type: `int larger` cannot take `String`s.
- Logistics touched on: three paper exams (mini midterm 1, midterm 2, final) with a clobber policy and an uncurved ~65% target mean; homeworks, four mini projects, two design projects, weekly surveys, opt-in discussion and lab attendance points, capped point categories, and lecture attendance yielding exam recovery points.
- LLM policy: use minimally, no LLM-written code turned in, no IDE plugins, no feeding specs to an LLM; tutor-style use is more acceptable but talking to humans is better.
- Homework 1 (set up IntelliJ) is due Friday; Lab 1 is a Java warm-up and is drop-in this week; the bridge section is recommended if you got a B or less in the prerequisite or came from E7.
- Practice, not innate talent, is what makes you good at this (the Dance Dance Revolution lesson): most of your learning happens in the ~200 hours outside lecture.
