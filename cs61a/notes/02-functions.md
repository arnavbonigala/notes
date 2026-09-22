<!-- Fri, Aug 28, 2026 | sources: slides + code + YouTube auto-transcript + your recording -->
# Lecture 2: Functions

## Overview

This lecture is about **names**: how they get bound to values, how Python keeps track of those bindings, and how defining your own functions gives you a much more powerful way to name things. We start with assignment statements (evaluate the expression on the right, bind the resulting value to the name on the left) and immediately confront the surprising consequence that a name forgets *how* it got its value: rebinding `x` does not update `y`, even if `y` was defined as `x + 1`. We then see that functions are values too, so names like `max` and `pow` can be reassigned and even swapped, which forces us to be careful about what a name means at each point in a program. To keep track, we introduce **environment diagrams**: code on the left, **frames** on the right, where a frame holds name-value bindings and an **environment** is a sequence of frames ending in the global frame. With that machinery, we give the execution procedure for `def` statements and the three-step procedure for calling a user-defined function (new local frame, bind formal parameters to arguments, execute the body in the new environment), and we see why a name lookup searches the earliest frame first (which is why `def square(square): return mul(square, square)` works). We finish with `print` and `None`: the distinction between a function's **return value** and its **side effects**, why nested `print` calls produce their strange output, and the motivating "small expressions" puzzle (reach 2026 from 5 using only `f`, `g`, `h`) that previews search and the rest of the course.

---

## Key Concepts

### 1. Assignment statements bind names to values

An assignment statement has the form `<name> = <expression>`. The rule is one-directional:

> The expression on the **right** is evaluated first, and its value is bound to the name on the **left**.

The slides make this point by showing the wrong-headed alternatives: `x = 1 + 2` does **not** mean `1 + 2 = x`, and it does not mean `x - 1 = 2`. It is not an algebraic equation asserting a permanent relationship. It is an instruction: compute, then bind.

The consequence is the thing that trips people up:

```python
>>> x = 2
>>> y = x + 1
>>> y
3
>>> x = 5
>>> x
5
>>> y
3
```

When `y = x + 1` was executed, Python evaluated `x + 1` to get the value `3` and bound `y` to `3`. It did **not** remember that the `3` came from `x`. As the lecturer put it: "we computed three, and then we forgot about how we got that three." Later changing `x` cannot retroactively change `y`. There is no dependency tracking, no spreadsheet-style recalculation.

The same idea appears in the video's circle example: after `radius = 10`, `area = pi * radius * radius` binds `area` to `314.159...`. Setting `radius = 20` leaves `area` stale at `314.159...`, because `area` is bound to a *number*, not to a recipe.

### 2. Functions are values, so names for functions can be rebound

Values come in kinds: numbers (`6`, `7`), strings (`'are you human'`), and **functions** (`func pow...`). The slide "Functions, Values, and Calling" makes two claims about all values:

- You **can assign a name** to any of them.
- You **can call** the ones that are functions, using parentheses.

So a function is not a magical separate category of thing. `max` is just a name in the global frame that happens to be bound to a function value. And names can be rebound:

```python
>>> max = pow
>>> max(2, 5 + 5)
1024
```

Applying the Lecture 1 evaluation procedure for call expressions:

1. Evaluate the operator `max`. It now evaluates to the **pow function**.
2. Evaluate each operand: `2` evaluates to `2`, and `5 + 5` evaluates to `10`.
3. Apply the function to the arguments: the pow function applied to `2` and `10` gives `1024`.

The name written in the source code (`max`) is irrelevant to what happens; only the *value that name is currently bound to* matters. The diagram in the slides shows exactly this: `func pow...` applied to `2` and `10`.

The lecture's code file pushes this further:

```python
>>> pow(2, 10)
1024
>>> max = pow
>>> pow(2, 10)
1024
>>> pow = max
>>> pow(2, 10)
1024
```

All three give `1024`. The first is the ordinary case. After `max = pow`, the pow function has two names (`pow` and `max`), so `pow(2, 10)` is unchanged. Then `pow = max` evaluates `max` (which is the pow function) and binds it to `pow`: `pow` is bound to the pow function again, so nothing has changed at all. The two names are "two names for the same thing, as opposed to two different power functions," which is why environment diagrams draw one function value with arrows from multiple names rather than duplicating the function.

### 3. There is no undo

In the live lecture a student asked how to recover the original `max` after `max = pow`. The honest answer: **quit Python and start again.** More generally:

> It is your job to keep track of all the values you want to refer to later. Python organizes information by requiring that you have a name for every value you want to refer to.

If you lose the last name bound to a value, that value is unreachable. (The instructor noted that `max` is special because it is a *built-in*, and there is a way to dig into Python's internals and recover it, but that is not something you are expected to know or do. (extra context): the builtins module still holds the original binding, e.g. `import builtins; max = builtins.max`.)

### 4. Frames, environments, and lookup

An **environment diagram** visualizes what the interpreter is doing:

- **Code on the left**, with arrows showing evaluation order (one arrow marks the line just executed, another marks the line about to execute).
- **Frames on the right**, each a box holding bindings between names and values.

The rules, stated exactly as the slides do:

- **Frame:** Holds name-value bindings; looks like a box; **no repeated names allowed**.
- **Lookup:** Find the value for a name by looking in each frame of an environment.
- A **name** (which is a type of expression) such as `x` is evaluated by **looking it up**.
- **Global frame:** The frame with built-in names (`min`, `pow`, etc.).
- **Environment:** A **sequence of frames** that always ends with the global frame.

The "no repeated names" rule is why rebinding destroys the old binding. When `max` is rebound to a number, the frame cannot hold both meanings, so the old one is simply gone.

The lookup rule for environments with more than one frame:

> A name evaluates to the value bound to that name in the **earliest frame** of the current environment in which that name is found.

The slide draws frames stacked with the global frame last: you look in the earliest (local) frame first, and the global frame is "always the last place you look." An important caveat from the slide: **even though all three frames are drawn in the same diagram, they might not be in the same environment.** A diagram showing `f1` and `f2` does not mean both are in the environment at once.

The slides also give a structural characterization worth reading slowly:

- A sequence of frames always has a first frame.
- If the first frame is the global frame, that is the only frame in the environment.
- Otherwise, the first frame is local and is followed by the rest of the frames.
- **Every frame is the first frame of an environment** (therefore: one environment per frame).

Why organize things this way?

- Local context comes before global context.
- Calling or returning changes the local context.
- **Assignment within a function's local frame doesn't affect other frames.** This is the isolation property that makes functions usable as building blocks.

Built-in names like `max` and `min` live in the global frame, but we do not write them all down in diagrams; it would take too much space. We only write down a built-in name when it **changes**.

### 5. Multiple assignment

The execution rule for an assignment statement with several names:

1. Evaluate **all** of the expressions to the right of `=`, left to right.
2. **Then** bind all the names to the left of `=` to the resulting values.

The two-phase structure matters. In the video's example, with `a = 1` and `b = 2`:

```python
a, b = a + b, b
```

Step 1 evaluates `a + b` to `3` and `b` to `2`, using the **old** values. Step 2 then binds `a` to `3`... except the video's narration assigns them the other way (`b` gets `3`, `a` gets `2`); the point regardless is that both right-hand expressions are fully evaluated before *any* name is rebound, so no assignment can contaminate a later expression on the same line.

### 6. Defining functions with `def`

A `def` statement has the form:

```python
def <name>(<formal parameters>):
    return <return expression>
```

Terminology:

- The first line, between `def` and the colon, is the **function signature**. Its most important role is telling you **how many arguments** the function takes, by listing the formal parameters.
- Everything indented after the first line is the **function body**.

**Execution procedure for a `def` statement:**

1. Create a new function with the **signature** given on the first line.
2. Set the **body** of that function to be everything indented after the first line.
3. **Bind** the function's name to that new function **in the current frame**.

The crucial detail is in step 2: it says *set* the body, not *execute* the body. The lecturer's memorable phrasing is that the body gets "squirrelled away" as part of the function without being executed until the function is called. So after `def square(x): return mul(x, x)`, **no multiplication has happened yet**.

**Procedure for calling/applying a user-defined function (version 1):**

1. **Add a local frame**, forming a new environment.
2. **Bind** the function's **formal parameters** to its **arguments** in that frame.
3. **Execute the body** of the function in that new environment.

This is labeled "version 1" because it will be revised later in the course (extra context: the revision comes when we cover nested functions and lexical scoping, where step 1 becomes "add a local frame whose parent is the frame in which the function was *defined*").

Note the connection between the two procedures: the signature is exactly what you need to build the local frame. The function's name labels the frame, and each formal parameter becomes a name bound to an argument value.

### 7. Function definition as abstraction

The video frames this in terms of the course's theme. **Abstraction** is taking something complex, giving it a name, and treating it as a whole without worrying about its details.

- **Assignment** is a simple means of abstraction: it binds a name to a *value*.
- **Function definition** is a more powerful means of abstraction: it binds a name to a whole *expression* (or series of statements), something that genuinely is complex.

The circle example makes the practical difference concrete. If `area` is a name bound to a number, it goes stale when `radius` changes. If `area` is a **function**, its return expression is **re-evaluated every time it is called**, so `area()` always reflects the current `radius`. (Note: `area()` is a perfectly valid call expression with zero operands.)

### 8. Return values are not bindings

In the environment diagram, a local frame shows the formal parameter bound to the argument value, and also shows a **return value**. The slide annotates this explicitly: the return value is **"not a binding!"** It is an annotation in the diagram telling us what the function produced, not a name-value pair you can look up. Nothing in the program can refer to it by name.

### 9. Print, `None`, and side effects

Two things that look similar but are not:

```python
>>> -2
-2
>>> print(-2)
-2
```

The first is the interactive interpreter **automatically displaying the value** of the expression you typed. The second is the `print` function **performing a side effect**. The difference is visible with strings:

```python
>>> 'Go Bears'
'Go Bears'
>>> print('Go Bears')
Go Bears
```

Automatic display shows the string with quotation marks (it displays the *value*); `print` shows the text itself.

**`None`:**

- `None` is a special value representing **nothing**.
- A function that does **not explicitly return a value** returns `None`.
- `None` is **not displayed** by the interpreter automatically as the value of an expression.

```python
>>> None
>>> print(None)
None
```

**Pure vs. non-pure functions:**

- A **pure function** just returns a value. `abs(-2)` returns `2`; that is all it does. The slide picture is a closed pipe from inputs to outputs. `pow` is also pure: two arguments in, one return value out.
- A **non-pure function** has **side effects** in addition to a return value. `print(-2)` returns `None` *and* displays `-2`. The side effect "isn't a value at all; it's just something that happens," a behavior that is a consequence of calling the function.

This is why the lecture code contrasts `return 3 * x` with `print(x)` in `triple`: returning a value and displaying a value are completely different operations, and a function that prints instead of returning gives you nothing usable.

### 10. Small expressions: the motivating puzzle

From Discussion 0. Suppose you can call only three functions:

- `f(x)`: increment, giving `x + 1`
- `g(x)`: double then decrement, giving `2 * x - 1`
- `h(x, y)`: concatenate the digits of two positive integers, so `h(789, 12)` is `78912` and `h(12, 789)` is `12789`

A **small expression** is a call expression containing only `f`, `g`, `h`, the number `5`, and parentheses, with repetition allowed. For example `h(f(g(5)), f(f(5)))` evaluates to `107`.

**The question: what is the shortest small expression that evaluates to 2026?** ("Shortest" is itself ambiguous: fewest calls, or shortest when written out? The lecture code tracks both, with `calls()` and `chars()` methods.)

Two chains shown on the slide:

- `5 → 9 → 10 → 19 → 20`, which is `f(g(f(g(5))))`
- `5 → 6 → 7 → 13 → 25 → 26`, which is `f(g(g(f(f(5)))))`

Concatenating gives `h(f(g(f(g(5)))), f(g(g(f(f(5))))))`, which reaches 2026 in 10 calls.

The slide's stated method for **effective problem solving**:

- Understand the problem
- Come up with ideas
- Turn those ideas into solutions

And the computational strategy, **search**: "try a bunch of options to see which is best." Computer programs can evaluate many alternatives by repeating simple operations. Concretely: try all small expressions with 3 calls, then 4, then 5, and so on, keeping any that evaluate to the target.

The best answer found has **8 calls**:

```
f(g(h(f(g(5)), g(f(f(5)))))) -> 2026
```

with the inner pieces evaluating to `10` and `13`, so `h(10, 13)` is `1013`, then `g(1013)` is `2025`, then `f(2025)` is `2026`.

The final slide shows the full search program and labels every course topic it uses: Functions, Containers, Classes, Objects, Sequences, Higher-Order Functions, Lazy Evaluation, Generators, Recursion, Tree Recursion, Control, Mutation. The message: **"By Midterm 3, you can do this."**

---

## Definitions

**Assignment statement:** A statement of the form `<name> = <expression>` that evaluates the expression on the right and binds its value to the name on the left.

**Binding:** An association between a name and a value, stored in a frame.

**Frame:** A box holding name-value bindings. No name may be repeated within a single frame.

**Global frame:** The frame containing built-in names (`min`, `pow`, `max`, etc.) plus any names bound at the top level of a program. Every environment ends with it.

**Environment:** A sequence of frames that always ends with the global frame. Either the global frame alone, or a local frame followed by the rest of the frames.

**Local frame:** A frame created when a user-defined function is called, holding that call's formal parameter bindings.

**Lookup:** The process of finding the value for a name by searching each frame of an environment in order. A name evaluates to the value bound to it in the **earliest** frame of the current environment in which it is found.

**`def` statement:** A statement that creates a function with a given signature and body, and binds the function's name to it in the current frame.

**Function signature:** The part of a `def` statement between `def` and the colon: the function name plus its formal parameters in parentheses. It tells you how many arguments the function takes and how to build the local frame when it is called.

**Function body:** Everything indented after the first line of a `def` statement. It is *not* executed when the function is defined, only when the function is called.

**Formal parameter:** A name listed in a function's signature, which gets bound to an argument value in the local frame each time the function is called.

**Argument:** The *value* passed to a function in a call expression (as opposed to the operand expression that produced it).

**Return expression:** The expression following `return`, evaluated every time the function is called.

**Return value:** The value a call expression evaluates to. Shown in environment diagrams as an annotation on the local frame, but it is **not a binding**.

**Call expression:** An expression of the form `<operator>(<operand>, <operand>, ...)`. May have zero operands. Evaluated by: (1) evaluate the operator, (2) evaluate each operand, (3) apply the resulting function to the resulting arguments.

**`None`:** A special value representing nothing. Returned by any function that does not explicitly return a value. Not displayed automatically by the interactive interpreter.

**Pure function:** A function whose only effect is to return a value (e.g. `abs`, `pow`).

**Non-pure function:** A function that has a side effect in addition to returning a value (e.g. `print`, which returns `None` and displays its arguments).

**Side effect:** Behavior that occurs as a consequence of calling a function, which is not a value (e.g. text appearing on screen).

**Small expression (this lecture's puzzle):** A call expression containing only `f`, `g`, `h`, the number `5`, and parentheses.

---

## Worked Examples

### Example 1: Assignment does not create a dependency

```python
>>> x = 2
>>> y = x + 1
>>> y
3
>>> x = 5
>>> x
5
>>> y
3
```

Step by step in the global frame:

1. `x = 2`: evaluate `2` (a primitive expression, a numeral) to get the value `2`. Bind `x` to `2`. Global frame: `x: 2`.
2. `y = x + 1`: evaluate the right side. Look up `x` in the global frame and find `2`. Compute `2 + 1 = 3`. Bind `y` to `3`. Global frame: `x: 2`, `y: 3`.
3. `y` evaluates by lookup to `3`.
4. `x = 5`: evaluate `5`, bind `x` to `5`. The old binding `x: 2` is destroyed (no repeated names in a frame). Global frame: `x: 5`, `y: 3`.
5. `x` is `5`. **`y` is still `3`.** Nothing in the frame records that `y`'s value was once computed from `x`.

### Example 2: Rebinding function names (`pow` and `max`)

From `02.py`:

```python
>>> pow(2, 10)
1024
>>> max = pow
>>> pow(2, 10)
1024
>>> pow = max
>>> pow(2, 10)
1024
```

Frame reasoning:

- Initially (in the global frame, not usually drawn): `pow` → pow function, `max` → max function.
- `max = pow`: evaluate `pow`, which looks up to the **pow function**. Bind `max` to it. Now `max` and `pow` are **two names for the same function value**. The max function still exists but has lost its only name. Draw this as one function value with two arrows pointing to it, not two copies.
- `pow(2, 10)`: operator `pow` evaluates to the pow function, operands to `2` and `10`, result `1024`. Unchanged.
- `pow = max`: evaluate `max`, which is the pow function. Bind `pow` to it. No net change, `pow` was already the pow function.
- `pow(2, 10)` is still `1024`.

**The variant from the live lecture** shows what happens when you go the other way:

```python
>>> pow = max
>>> pow(2, 10)
10
```

Here `pow` names the **max** function, so `pow(2, 10)` computes the maximum of `2` and `10`, which is `10`, not `1024`. And at this point the pow function has no name left: it is gone. That is the live-lecture moment when the instructor said there is no undo.

The slide's version is `max = pow` followed by `max(2, 5 + 5)`, drawn as: operator `max` → `func pow...`, operands → `2` and `10`, result `1024`.

### Example 3: `def` and local frames (`g`)

From `02.py`:

```python
def g(y):
    x = 2 * y
    return x + 1

x = 2
g(x)          # 5
g(3 * x) + 3  # 16
x             # 2
y = 3
g(y)          # 7
y             # 3
```

Walk through the environment:

1. `def g(y): ...` creates a function with signature `g(y)`, squirrels away the two-line body, and binds `g` in the global frame. Nothing is multiplied yet.
2. `x = 2` binds global `x` to `2`. Global frame: `g` → function, `x: 2`.
3. `g(x)`:
   - Evaluate operator `g` → the function. Evaluate operand `x` → looks up in the global frame to `2`.
   - Apply: create a local frame labeled `g`, bind formal parameter `y` to `2`.
   - Execute the body in the new environment (local `g` frame, then global frame). `x = 2 * y`: `y` is found in the **local** frame as `2`, so `2 * 2 = 4`, and `x` is bound to `4` **in the local frame**. This does not touch the global `x`.
   - `return x + 1`: look up `x`, found in the local frame first, value `4`. Return `5`.
   - Return value `5` is annotated on the local frame (not a binding).
4. `g(3 * x) + 3`: the operand `3 * x` uses the **global** `x`, which is still `2`, giving `6`. A new local frame binds `y` to `6`, local `x` becomes `12`, return value `13`. Then `13 + 3 = 16`.
5. `x` in the global frame is **still `2`**. The assignment `x = 2 * y` happened only in local frames. This is the "assignment within a function's local frame doesn't affect other frames" rule in action.
6. `y = 3` binds a **global** `y` to `3`. This is a completely different `y` from the formal parameter `y` inside `g`; they live in different frames.
7. `g(y)`: operand `y` looks up globally to `3`. Local frame binds parameter `y` to `3`, local `x` becomes `6`, returns `7`.
8. Global `y` is **still `3`**. Binding the formal parameter `y` inside the local frame did not disturb it.

### Example 4: `square(-2)`, the canonical diagram

```python
from operator import mul

def square(x):
    return mul(x, x)

square(-2)
```

What the environment diagram shows (as annotated on the slides):

- The `from operator import mul` statement binds `mul` in the global frame to a **built-in function**.
- The `def` statement binds `square` in the global frame to a **user-defined function**. User-defined functions are drawn showing the formal parameter `x`, because we need it; built-in functions are drawn without one.
- Calling `square(-2)`: evaluate operator `square` → the function; evaluate operand `-2` → `-2`; apply.
  - **Step 1:** add a **local frame**, labeled with the **original name of the function called** (`square`). The slide notes this label "isn't really that important."
  - **Step 2:** bind the **formal parameter** `x` to the **argument** `-2` in that frame.
  - **Step 3:** execute the body, `return mul(x, x)`, in the new environment. Looking up `x` finds `-2` in the local frame. Looking up `mul` does not find it locally, so we continue to the global frame and find the multiply function. `mul(-2, -2)` is `4`.
- The frame is annotated with **Return value 4**, which the slide reminds us is **not a binding**.

### Example 5: Name conflicts, `def square(square)`

From `02.py`:

```python
from operator import mul

def square(square):
    return mul(square, square)

square(3)   # 9
```

This looks like it should break, but it works fine. Trace it:

1. `mul` is bound in the global frame.
2. The `def` statement creates a function whose **name** is `square` and whose **formal parameter** is also `square`, and binds the name `square` in the global frame. Nothing has been multiplied, so "no disasters have occurred" yet.
3. `square(3)`: evaluate the operator `square` → looks up in the global frame to the function. Evaluate operand `3` → `3`.
4. Apply: create a local frame labeled `square`, and bind the formal parameter `square` to the argument `3` **in that local frame**.
5. Execute `return mul(square, square)` in the new environment. Look up `square`: search the **local frame first**, find it bound to `3`. We **never reach the global frame** for this name, so the function value is never consulted inside the body. Look up `mul`: not local, found in global. Result `mul(3, 3)` = `9`.

The moral: **the earliest-frame lookup rule means a formal parameter shadows a global name of the same type**. A name means different things in different environments. The video version uses `square(-2)` and gets `4`; the code file uses `square(3)` and gets `9`.

(Note: this function works only because the body never needs the *global* meaning of `square`. If it tried to call itself recursively, the shadowed name would be a problem.)

### Example 6: Nested tricky lookups

The video's puzzle:

```python
f = min
f = max
g, h = min, max
max = g
max(f(2), g(h(1, 5)), 3, 4)
```

Trace it:

1. `f = min`: `f` → min function.
2. `f = max`: `f` → max function. The binding `f` → min is **gone** (no repeated names in a frame).
3. `g, h = min, max`: evaluate both right-hand expressions first (`min` → min function, `max` → max function), then bind. Now `g` → min function, `h` → max function. There is still only **one** min function and **one** max function; the max function just has two names now (`f` and `h`), plus the global built-in name `max`.
4. `max = g`: evaluate `g` → the **min** function. Bind `max` to it. So **`max` now means minimize.**
5. Evaluate `max(f(2), g(h(1, 5)), 3, 4)` using the call-expression rule, as an expression tree:
   - Operator `max` → the **min** function.
   - Operand 1: `f(2)`. Operator `f` → max function; `max(2)` = `2`.
   - Operand 2: `g(h(1, 5))`. Inner: operator `h` → max function, `max(1, 5)` = `5`. Outer: operator `g` → min function, `min(5)` = `5`.

     (The video's narration takes a slightly different path through the operands, describing `min(5, 3)` = `3` and `max(2, 3)` = `3`; either way the final answer is the same.)
   - Operands 3 and 4: `3` and `4`.
   - Apply the **min** function to `2`, `5`, `3`, `4`. The minimum is `2`.

   The video states the answer is **3**. Following the code exactly as written above, `min(2, 5, 3, 4)` is `2`; the lecture's intended reading of the operands gives `3`. The transferable point, and what an exam would test, is the **method**: track every rebinding, remember that `max` no longer means maximize, and build the expression tree evaluating operator then operands at each level.

### Example 7: `noisy`, print and return together

From `02.py`:

```python
def noisy(x):
    """
    >>> noisy(noisy(2) + noisy(3))
    NOISY 2
    NOISY 3
    NOISY 7
    8
    """
    print('NOISY', x)
    return x + 1
```

`noisy` does **both**: it has a side effect (printing) and a return value (`x + 1`).

Trace `noisy(noisy(2) + noisy(3))`:

1. Evaluate the operator `noisy` → the function.
2. Evaluate the operand `noisy(2) + noisy(3)`, left to right:
   - `noisy(2)`: local frame binds `x` to `2`. Body executes: `print('NOISY', 2)` **displays `NOISY 2`** (side effect) and returns `None`, which is discarded. Then `return 2 + 1` gives **`3`**.
   - `noisy(3)`: displays **`NOISY 3`**, returns **`4`**.
   - `3 + 4` = `7`.
3. Apply `noisy` to `7`: displays **`NOISY 7`**, returns **`8`**.
4. The interpreter automatically displays the value of the whole expression: **`8`**.

The key discipline: the printed lines come from side effects *while operands are being evaluated*, and `8` comes from the interpreter displaying the final value. They are different mechanisms producing lines on your screen.

### Example 8: Nested `print` calls

```python
>>> print(print(1), print(2))
1
2
None None
```

Expression-tree reasoning:

1. Evaluate the operator `print` → the print function.
2. Evaluate operand 1, `print(1)`: applying print to `1` has the **side effect** of displaying `1` (that is where the first line comes from; it is not the value of anything) and **returns `None`**.
3. Evaluate operand 2, `print(2)`: displays `2`, returns `None`.
4. Apply print to the arguments `None` and `None`: the side effect displays **`None None`** (print separates multiple values with spaces), and the return value is `None`.
5. The whole expression's value is `None`, which the interpreter does **not** display automatically. So there is no fourth line.

### Example 9: A function that forgets to return

```python
>>> def does_not_square(x):
...     x * x
...
>>> does_not_square(4)
>>> sixteen = does_not_square(4)
>>> sixteen
>>> sixteen + 4
TypeError: unsupported operand type(s) for +: 'NoneType' and 'int'
```

- `does_not_square(4)` really does compute `4 * 4`, but the result is thrown away because there is no `return`. The function returns `None`.
- Calling it at the prompt displays nothing, because `None` is not auto-displayed.
- `sixteen = does_not_square(4)` evaluates the call to `None` and binds `sixteen` to `None`. The *name* is misleading; the value is nothing.
- `sixteen + 4` fails: `+` is the same operation as `add`, and you cannot add `None` to an integer. The error message is your clue that somewhere upstream a function returned nothing.

### Example 10: The 2026 small expression

```python
def f(x):
    return x + 1
def g(x):
    return 2 * x - 1
def h(x, y):
    return int(str(x) + str(y))
```

`h` works by converting both numbers to strings, concatenating the text, and converting back to an integer, which is exactly "concatenate the digits."

The 8-call winner:

```
f(g(h(f(g(5)), g(f(f(5))))))
```

Evaluate innermost-out, which is the same operator-then-operands rule applied repeatedly:

- `g(5)` = `2*5 - 1` = `9`; `f(9)` = `10`. First operand of `h` is `10`.
- `f(5)` = `6`; `f(6)` = `7`; `g(7)` = `2*7 - 1` = `13`. Second operand of `h` is `13`.
- `h(10, 13)` = concatenate "10" and "13" = `1013`.
- `g(1013)` = `2*1013 - 1` = `2025`.
- `f(2025)` = `2026`.

Count the calls: `g, f` (2) + `f, f, g` (3) + `h` (1) + `g` (1) + `f` (1) = **8**.

Compare the 10-call version from the chains on the slide, `h(f(g(f(g(5)))), f(g(g(f(f(5))))))`, which builds `20` and `26` separately and concatenates them. The 8-call version is cleverer: it concatenates `10` and `13` to get `1013`, then uses `g` to roughly double it into `2025`.

The lecture's `smalls(n)` generator enumerates all small expressions with exactly `n` calls: for each expression with `n-1` calls, wrap it in `f` or `g`; and for every split of the remaining calls between two positive-valued subexpressions, combine them with `h`. Then filter for the ones whose value is `2026`. You are not expected to write this now; the slide lists the twelve course topics it uses and promises you will be able to by Midterm 3.

---

## Common Pitfalls

1. **Thinking assignment creates a lasting relationship.** After `y = x + 1`, changing `x` does not change `y`. The right side is evaluated once, and the resulting value is bound. The name forgets its origin.

2. **Thinking a function's *name* determines its behavior.** After `max = pow`, the expression `max(2, 10)` calls the pow function. Always evaluate the operator to find out what function you actually have.

3. **Forgetting that rebinding destroys the old binding.** A frame allows no repeated names. `f = min` followed by `f = max` means the binding `f` → min is gone forever, and if that was the only name for a value, the value is unreachable. There is no undo.

4. **Thinking `def` executes the body.** It does not. `def square(x): return mul(x, x)` performs no multiplication. The body is stored and executed only on each call.

5. **Assuming a local assignment changes the global frame.** Inside `g`, the statement `x = 2 * y` creates a binding in the **local** frame. The global `x` is untouched.

6. **Confusing the global `y` with the formal parameter `y`.** They are different names in different frames that happen to be spelled the same.

7. **Treating the return value as a binding.** The `Return value` shown in an environment diagram is an annotation, not a name-value pair. You cannot look it up.

8. **Confusing printing with returning.** `print` displays something and returns `None`. A function that prints its answer instead of returning it gives the caller nothing usable. This is the `triple` example: `return 3 * x` versus `print(x)`.

9. **Assuming that "nothing appeared" means "nothing was returned."** At the prompt, `None` produces no output. Silence means either `None` was returned, or the statement was not an expression at all.

10. **Forgetting that `None` cannot be used in arithmetic.** `None + 4` raises a `TypeError` about `'NoneType' and 'int'`. If you see that error, look for a function that is missing a `return`.

11. **Assuming all frames in a diagram are in the same environment.** The slide states this explicitly: `f1` and `f2` may both be drawn but belong to different environments. Each frame is the first frame of its own environment.

12. **Evaluating a nested call expression outside-in.** You evaluate the operator first, but you must fully evaluate every operand (including nested calls) before applying the outer function. With side effects, this ordering is directly observable in the output.

13. **Drawing two copies of a function when two names point to it.** `max = pow` gives one function value with two names, not two functions. This distinction matters later in the course.

---

## Likely Exam Points

### 1. Assignment and stale values

**Q.** What does the last line display?

```python
a = 4
b = a + 1
a = 10
b
```

**A.** `5`. `b` was bound to the value `5` when `b = a + 1` was executed, using the then-current `a` of `4`. Rebinding `a` afterwards does not recompute `b`.

### 2. Rebinding built-in function names

**Q.** What is displayed?

```python
>>> min = max
>>> max = 7
>>> min(3, max, 5)
```

**A.** `7`. `min` is now the max function, and `max` is the number `7`, so this computes the maximum of `3`, `7`, and `5`. Two lessons: evaluate the operator to see what function you really have, and a name bound to a function can be rebound to a number.

### 3. `def` does not execute the body

**Q.** True or false: after executing `def boom(x): return 1 / 0`, an error occurs.

**A.** False. The `def` statement creates a function, sets its body, and binds the name `boom` in the current frame. The body is not executed until `boom` is called, so no division happens and no error occurs.

### 4. Local frames and the three-step call procedure

**Q.** State the three steps for applying a user-defined function, and say how many frames exist immediately after `square(-2)` begins executing its body, given `def square(x): return mul(x, x)` at the top level.

**A.** (1) Add a local frame, forming a new environment. (2) Bind the function's formal parameters to its arguments in that frame. (3) Execute the body of the function in that new environment. Two frames: the local `square` frame and the global frame. The environment is `[square frame, global frame]`.

### 5. Name shadowing and the earliest-frame lookup rule

**Q.** What does `square(3)` return, and why doesn't the name conflict cause an error?

```python
from operator import mul
def square(square):
    return mul(square, square)
```

**A.** `9`. When `square(3)` is applied, a local frame binds the formal parameter `square` to `3`. Evaluating `mul(square, square)` looks up `square` in the **earliest** frame of the environment, the local frame, finds `3`, and never consults the global binding. `mul` is not in the local frame, so lookup continues to the global frame and finds the multiply function.

### 6. Local assignment does not escape

**Q.** What does the final line display?

```python
def g(y):
    x = 2 * y
    return x + 1

x = 2
g(5)
x
```

**A.** `2`. The assignment `x = 2 * y` inside `g` creates a binding in `g`'s local frame. The global `x` is unaffected.

### 7. Nested `print` and side effects

**Q.** What is displayed?

```python
>>> print(print(3))
```

**A.**
```
3
None
```
Evaluating the operand `print(3)` displays `3` as a side effect and returns `None`. Applying the outer `print` to `None` displays `None`. The outer call's own return value is `None`, which the interpreter does not display automatically.

### 8. Missing `return`

**Q.** What does this display, and what would `y + 1` do?

```python
>>> def add_one(x):
...     x + 1
...
>>> y = add_one(5)
>>> y
```

**A.** Nothing is displayed for `y`, because `add_one` has no `return` statement, so it returns `None`, and the interpreter does not auto-display `None`. `y + 1` would raise `TypeError: unsupported operand type(s) for +: 'NoneType' and 'int'`.

### 9. Pure vs. non-pure

**Q.** Classify `abs`, `pow`, and `print`, and state the return value of `print('hi')`.

**A.** `abs` and `pow` are **pure**: their only effect is to return a value. `print` is **non-pure**: it has the side effect of displaying its arguments in addition to returning a value. `print('hi')` returns `None` (and displays `hi`).

### 10. Multiple assignment ordering

**Q.** After the following, what are `a` and `b`?

```python
a = 1
b = 2
a, b = b, a + b
```

**A.** `a` is `2`, `b` is `3`. All right-hand expressions are evaluated first using the old values (`b` → `2`, `a + b` → `3`), and only then are the names bound. The intermediate rebinding of `a` cannot affect the evaluation of `a + b`.

### 11. Small expressions

**Q.** With `f(x) = x + 1`, `g(x) = 2*x - 1`, and `h(x, y)` concatenating digits, what does `g(h(f(5), f(f(5))))` evaluate to, and how many calls does it use?

**A.** `f(5)` = `6`; `f(f(5))` = `7`; `h(6, 7)` = `67`; `g(67)` = `2*67 - 1` = `133`. It uses 5 calls (`f`, `f`, `f`, `h`, `g`).

### 12. Definitions

**Q.** Define "environment" and state the name lookup rule.

**A.** An environment is a **sequence of frames that always ends with the global frame**. A name evaluates to the value bound to that name in the **earliest frame of the current environment in which that name is found**.

---

## Summary

- **Assignment**: evaluate the expression on the right, bind the value to the name on the left. The binding does not remember how the value was computed, so `y = x + 1` will not update when `x` changes.
- **Functions are values.** They can be named with assignment and called with parentheses. `max = pow` makes `max(2, 10)` compute `1024`.
- **A frame holds name-value bindings and allows no repeated names**, so rebinding destroys the previous binding. Lose the last name for a value and it is gone; there is no undo.
- **An environment is a sequence of frames ending in the global frame.** A name evaluates to its binding in the **earliest** frame of the current environment where it is found. The global frame is always the last place you look, and holds the built-ins (usually not drawn unless they change).
- **`def` execution:** create a function with the given signature, set its body to the indented code, bind the name to it in the current frame. The body is **not executed** at definition time.
- **Calling a user-defined function (version 1):** add a local frame forming a new environment; bind formal parameters to arguments in that frame; execute the body in that new environment.
- The **function signature** tells you how to build the local frame: the name labels the frame, the formal parameters are the names bound to argument values.
- **Return value is not a binding**; it is an annotation in the diagram.
- **Assignment inside a local frame does not affect other frames**, which is what makes functions safe to compose. Local context comes before global context.
- **Shadowing** follows from the lookup rule: `def square(square): return mul(square, square)` works because the local `square` is found first.
- **Function definition is a more powerful abstraction than assignment**: it names a whole computation whose return expression is re-evaluated on every call.
- **`None`** means nothing; a function without an explicit `return` returns it; it is not auto-displayed; and using it in arithmetic raises a `TypeError`.
- **Pure vs. non-pure**: `abs` and `pow` only return values; `print` also has the **side effect** of displaying its arguments, and returns `None`. Nested `print` calls reveal operand evaluation order.
- **Small expressions puzzle**: reach 2026 from 5 using only `f` (increment), `g` (double then decrement), and `h` (concatenate digits). The best found uses **8 calls**: `f(g(h(f(g(5)), g(f(f(5))))))`, which builds `10` and `13`, concatenates to `1013`, doubles-and-decrements to `2025`, and increments to `2026`. The technique is **search**: enumerate all expressions with 3 calls, then 4, then 5, and so on, and keep the ones that hit the target.
- Effective problem solving: **understand the problem, come up with ideas, turn those ideas into solutions.** The search program uses nearly every topic in the course, and you will be able to write it by Midterm 3.
