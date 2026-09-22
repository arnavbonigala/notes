<!-- Mon, Aug 31, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 3: Control

This lecture is about **control**: how Python decides *which* statements to execute and *how many times*. It opens with a review of the environment model from Lecture 2 (def statements, call expressions, frames, environments, and name lookup), then draws a sharp line between **printing** and **returning** (and the role of `None` and side effects), then introduces the two control statements that will power the rest of the course: **conditional statements** (`if` / `elif` / `else`, governed by the idea of a *boolean context* and Python's true/false values) and **while statements** (repetition, governed by a condition that must eventually become false). It closes with a real algorithm built from these pieces: printing the prime factorization of a positive integer by repeatedly finding and dividing out the smallest factor greater than 1. The unifying theme is that every compound statement has a precise, mechanical **execution rule**, and if you follow the rule line by line you can always predict exactly what a program does.

---

## Key Concepts

### 1. Review: environments give names meaning

A **def statement** creates a function and binds a name to it in the current frame. A **call expression** applies a function to arguments. To apply a user-defined function:

1. Create a new frame.
2. Bind the function's formal parameters to the argument values in that frame.
3. Execute the body of the function in that new environment.

An **environment** is a *sequence* of frames, not a single frame. Before any function call, the only environment is the global frame alone. Once you call a user-defined function, you get a two-frame environment: the new local frame, followed by the global frame.

In the lecture's `square(square(3))` example, there is **one** function but **two** frames (`f1` for the call on 3, `f2` for the call on 9), because each call creates its own frame. That diagram contains **three** environments:

- the global frame alone,
- `f1` then global,
- `f2` then global.

No environment contains all three frames. Starting from any frame you recover its whole environment by following **parent** links; the global frame has no parent, so it is always last.

**Name lookup rule:** a name evaluates to the value bound to it in the **earliest frame of the current environment** in which that name is found. Evaluating `mul(x, x)` inside `f2`: `x` is found immediately in `f2` (so `x` is 9); `mul` is *not* in `f2`, so we continue to the global frame and find it there.

The punchline: **names have no meaning without an environment**, and the same name can mean different things in different environments. The lecture's deliberately confusing example:

```python
from operator import mul

def square(square):
    return mul(square, square)

square(4)   # 16
```

This works because the call expression `square(4)` is *not indented*, so its operator is evaluated in the **global** frame, where `square` is the function. The body `mul(square, square)` **is** indented, so it is always evaluated in an environment starting with the new local frame, where `square` is bound to 4. The local binding is found first, so the global binding is never reached from inside the body. (Do not write code like this; the point is that indentation tells you which environment a line is evaluated in.)

### 2. Statements, compound statements, clauses, and suites

A **statement** is executed by the interpreter to perform an action (bind a name, define a function, print something). Statements can span multiple lines. A **compound statement** looks like:

```
<header>:
    <statement>
    <statement>
<separating header>:
    <statement>
    <statement>
```

- A **clause** is one header plus the indented statements that follow it.
- Those indented statements are the **suite** of the clause.
- The **first header determines the statement's type** (so you can always tell what kind of statement you are looking at).
- The header of a clause **controls** the suite that follows it.

`def` statements are compound statements; their suite is called the **body** of the function.

**Execution rule for a suite:** execute the first statement; then, *unless directed otherwise*, execute the rest. The "unless directed otherwise" is exactly where `return`, and later control constructs, come in.

### 3. Print versus return, and `None`

This is the most commonly misunderstood distinction for new 61A students, so the lecture attacks it head-on.

- **Returning** hands a value back to the code that called the function, so it can be used in a larger expression.
- **Printing** is a **side effect**: text appears on the screen. `print` itself returns `None`.
- A function whose body ends without executing a `return` statement returns `None` by default. `None` is never displayed by the interactive interpreter.

Contrast:

```python
def triple(x):
    return 3 * x      # versus print(3 * x)
```

With `return`, `triple(2) + triple(3)` is 15. With `print` instead, `triple(2)` prints 6 and evaluates to `None`, so `triple(2) + triple(3)` prints 6 and 9 and then fails, because you cannot add `None` to `None`. *(extra context: the exact error is `TypeError: unsupported operand type(s) for +: 'NoneType' and 'NoneType'`.)*

Two more vocabulary items that go with this:

- A **pure function** just computes a return value from its arguments (e.g. `abs`, `pow`).
- A **non-pure function** has a side effect in addition to (or instead of) returning a value (e.g. `print`). Side effects happen **every time the function is called**, so calling a non-pure function twice makes the side effect happen twice.

### 4. Conditional statements

A conditional statement contains suites that **may or may not be evaluated**.

**Execution rule for conditional statements:** consider each clause in order. Evaluate the header's expression. If it is a **true value**, execute the suite for that clause and **skip all the remaining clauses**. (An `else` header has no expression; its suite runs if no earlier header was true.)

Consequence: within a single `if / elif / ... / else` statement, **at most one suite is ever executed**.

**Syntax:** exactly one `if` clause first, then zero or more `elif` clauses, then zero or one `else` clause, which must come last.

The critical distinction is between **one statement with several clauses** and **several separate statements**:

| Code | x = 10 | x = 1 | x = -1 |
|---|---|---|---|
| `if x > 2: print('big')` <br> `if x > 0: print('positive')` (two separate statements) | `big` `positive` | `positive` | (nothing) |
| `if x > 2: print('big')` <br> `elif x > 0: print('less big')` (one statement, two clauses) | `big` | `less big` | (nothing) |
| `if x > 2: print('big')` <br> `elif x > 0: print('less big')` <br> `else: print('not pos')` (one statement, three clauses) | `big` | `less big` | `not pos` |

With two separate `if` statements and `x = 10`, **both** bodies run. With `elif`, only the first matching body runs.

### 5. Boolean contexts, false values, and boolean operators

A **boolean context** is a place in Python code where you write an expression but *all that matters* is whether the value is true or false, not what the value is. The header expression of an `if` clause, an `elif` clause, and a `while` clause are all boolean contexts. (An `else` header has no boolean context.)

**False values:** `False`, `0`, `None`, `''` (the empty string). There are more to come later in the course.
**True values:** everything else.

**Comparison operators:** `>`, `<`, `>=`, `<=`, `==`, `!=`. Remember `==` is a comparison; `=` is assignment.

**Boolean operators:** `or`, `and`, `not`.

*(extra context, since the slide had a live demo whose details are not in the transcript: `and` and `or` **short-circuit** and evaluate to one of their operand values, not necessarily to `True`/`False`. `x and y` evaluates `x`; if it is a false value, that value is the result and `y` is never evaluated; otherwise the result is `y`. `x or y` evaluates `x`; if it is a true value, that value is the result; otherwise the result is `y`. `not x` always returns `True` or `False`.)*

### 6. While statements and iteration

**Iteration** means repeating things. A `while` statement repeats its suite as long as a condition is true.

**Execution rule for while statements:**

1. Evaluate the header's expression.
2. If it is a true value, execute the **whole** suite, then return to step 1.

Three considerations from the slides:

- **How many separate names do you need, and what does each one mean?** Naming is the design work in writing a loop. Typically you need names initialized before the loop, at least one of which is updated inside the loop.
- **The while condition must eventually become a false value** for the statement to terminate, unless there is a `return` inside the body. Something appearing in the condition must change in the body.
- **Once the condition is evaluated, the entire body is executed.** The condition is *not* rechecked in the middle of the suite. This is the source of most while-loop confusion.

---

## Definitions

- **Statement:** a piece of code executed by the interpreter to perform an action, such as binding a name to a value or defining a function.
- **Compound statement:** a statement spanning multiple lines, made of one or more clauses, each a header plus an indented suite.
- **Header:** the first line of a clause, ending in a colon; it controls the suite that follows. The first header of a compound statement determines the statement's type.
- **Clause:** a single header together with its indented statements.
- **Suite:** the sequence of indented statements controlled by a header.
- **Body (of a function):** the suite of a `def` statement.
- **Execute a suite:** execute its first statement, and then, unless directed otherwise, execute the rest, in order.
- **Conditional statement:** a compound statement whose clauses' suites may or may not be executed; at most one suite of a given conditional statement is executed.
- **Boolean context:** a position in code where an expression is written but only its truth or falsehood matters (for example the header expression of `if`, `elif`, or `while`).
- **False value:** a value that is false in a boolean context: `False`, `0`, `None`, `''` (more later).
- **True value:** any value that is not a false value.
- **Comparison operator:** `>`, `<`, `>=`, `<=`, `==`, `!=`.
- **Boolean operator:** `or`, `and`, `not`.
- **While statement:** a compound statement whose suite is executed repeatedly, as long as its header expression evaluates to a true value, checked before each iteration.
- **Iteration:** repeating a computation multiple times.
- **`None`:** the value representing "nothing"; it is the default return value of a function whose body finishes without a `return` statement, it is the return value of `print`, and it is never displayed by the interactive interpreter.
- **Side effect:** an observable effect of calling a function other than returning a value (for example printing to the screen).
- **Pure function:** a function whose only effect is to return a value computed from its arguments.
- **Non-pure function:** a function with side effects, such as `print`.
- **Environment:** a sequence of frames, used to give meaning to names. A name evaluates to the value bound to it in the earliest frame of the current environment in which that name is found.
- **Frame:** a binding of names to values, created when a function is called; it has a parent (the global frame has none).
- **Prime factors of a positive integer n:** primes whose product is n (for example 12 = 2 \* 2 \* 3).
- **Floor division (`//`):** integer division that discards the remainder; `858 // 2` is 429.

---

## Worked Examples

### Example 1: the four `h` functions (printing versus returning)

```python
def f(x):
    return x + 1

# (A)                 (B)                    (C)                          (D)
def h(x):             def h(x):              def h(x):                    def h(x):
    f(x)                  print(f(x))            return print(f(x))           return f(x)
```

Evaluating `h(3)` in each case:

| | (A) | (B) | (C) | (D) |
|---|---|---|---|---|
| Prints | (nothing) | `4` | `4` | (nothing) |
| Returns | `None` | `None` | `None` | `4` |

Step by step:

- **(A)** A new frame binds `x` to 3. The body calls `f(3)`, which returns 4. That value is simply discarded, since it is not printed, not returned, and not bound to any name. The body ends with no `return`, so the **default return value `None`** comes back. The interpreter displays nothing, because `None` is never displayed.
- **(B)** `f(3)` returns 4, then `print(4)` writes `4` to the screen. `print`'s own return value (`None`) is discarded. The body ends without `return`, so again `None` is returned by default. So: prints 4, returns `None`.
- **(C)** Same printing happens, but now we explicitly return **the return value of `print(f(x))`**, which is `None`. Prints 4, returns `None`. (A) and (B) return `None` by *default*; (C) returns `None` because `print` returned it. Same observable result, different reason.
- **(D)** `f(3)` returns 4 and `h` returns it. Nothing is printed, but the interactive interpreter **displays** `4` because the expression `h(3)` has value 4. This looks identical to (B) at the prompt, which is exactly the trap: only (D) gives you a value you can compute with, as in `h(3) + 1`.

### Example 2: print then return, and why the number of calls matters

Task: implement `h(x)` that first **prints**, then **returns**, the value of `f(x)`.

```python
# (A) wrong: returns None, not f(x)
def h(x):
    return print(f(x))

# (B) works for pure f: calls f twice
def h(x):
    print(f(x))
    return f(x)

# (C) works in general: calls f once
def h(x):
    y = f(x)
    print(y)
    return y
```

(A) is wrong: it returns `print`'s value, `None`.

(B) and (C) both print `f(x)` and then return `f(x)`, so for a pure `f` like `f(x) = x + 1` they behave identically:

```
>>> h(2)
3
3
```

(the first `3` is printed by `print`, the second `3` is the interpreter displaying the returned value).

The poll question: **for what `f` do (B) and (C) differ?** Answer: any **non-pure** `f`, that is, one with side effects, because (B) calls `f` **twice** and (C) calls it **once**. The lecture's concrete choice is `f = print`:

```python
>>> f = print
>>> h(2)      # version (B)
2
None
2
>>> h(2)      # version (C)
2
None
```

Tracing (B) with `f = print`: `f(2)` prints `2` and returns `None`; then `print(None)` prints `None`; then `return f(x)` calls `print(2)` again, printing `2` a second time and returning `None`. Tracing (C): `y = f(2)` prints `2` once and binds `y` to `None`; `print(y)` prints `None`; `return y` returns `None` without calling `f` again. The side effect happens twice in the version that calls `f` twice. Moral: bind the result to a name and reuse it, both to avoid recomputation and to avoid duplicating side effects.

### Example 3: `noisy` and evaluation order

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

To evaluate `noisy(noisy(2) + noisy(3))`, Python evaluates the operator and then the operand subexpressions, left to right, before applying the outer function:

1. `noisy(2)`: new frame binds `x` to 2, prints `NOISY 2`, returns 3.
2. `noisy(3)`: new frame binds `x` to 3, prints `NOISY 3`, returns 4.
3. The operand `3 + 4` evaluates to 7.
4. `noisy(7)`: new frame binds `x` to 7, prints `NOISY 7`, returns 8.
5. The interpreter displays `8`.

Three separate frames are created from one function. The printed lines record the order in which the frames were created; the final `8` is the displayed return value, not printed output.

### Example 4: absolute value with a conditional statement

```python
def absolute_value(x):
    """Return the absolute value of x."""
    if x < 0:
        return -x
    elif x == 0:
        return 0
    else:
        return x
```

```
>>> absolute_value(-2)
2
>>> absolute_value(0)
0
>>> absolute_value(3)
3
```

The body is **one statement with three clauses**: three headers (`if`, `elif`, `else`) and three suites, each a single `return` statement. Applying the execution rule to `absolute_value(-2)`: create a frame binding `x` to -2; evaluate `x < 0`, which is `True`, a true value, so execute that suite (`return -x`, giving 2) and **skip the remaining clauses**. For `absolute_value(3)`: `x < 0` is `False`, so skip that suite; `x == 0` is `False`, so skip that suite; the `else` clause has no condition, so its suite runs and returns 3.

Two boolean contexts appear here, one in each of the `if` and `elif` headers. `else` has none. (Python has a built-in `abs`; this is purely for illustration.)

### Example 5: the while loop demo (summing 1, 2, 3)

```python
i, total = 0, 0
while i < 3:
    i = i + 1
    total = total + i
```

Trace, following the execution rule exactly:

| Step | `i` before | `i < 3`? | body executes | `i` after | `total` after |
|---|---|---|---|---|---|
| start | 0 | | | 0 | 0 |
| iter 1 | 0 | true | `i = 0 + 1`, then `total = 0 + 1` | 1 | 1 |
| iter 2 | 1 | true | `i = 1 + 1`, then `total = 1 + 2` | 2 | 3 |
| iter 3 | 2 | true | `i = 2 + 1`, then `total = 3 + 3` | 3 | 6 |
| end | 3 | false | (not executed) | 3 | 6 |

In environment terms: everything happens in the **global frame**, whose bindings for `i` and `total` are repeatedly rebound. Notice in iteration 1 that when `i + 1` is evaluated, `i` is still 0, which is why the new binding is 1; assignment evaluates the right-hand side first, then rebinds the name.

The crucial detail the lecture emphasizes: during iteration 3, after `i = i + 1` makes `i` equal to 3, Python **does not** go back and recheck `i < 3`. The suite is executed to completion, so `total = total + i` runs with `i` equal to 3 and gives 6. Only then is the header re-evaluated: `3 < 3` is `False`, a false value, so the suite is not executed and we do not return to step 1. The statement finishes with `i` bound to 3 and `total` bound to 6.

### Example 6: prime factorization

```python
def prime_factors(n):
    """Print the prime factors of positive integer n
       in non-decreasing order.

    >>> prime_factors(8)
    2
    2
    2
    >>> prime_factors(858)
    2
    3
    11
    13
    """
    while n > 1:
        k = smallest_factor(n)
        print(k)
        n = n // k

def smallest_factor(n):
    """Return the smallest factor of n greater than 1."""
    k = 2
    while n % k != 0:
        k = k + 1
    return k
```

**The idea:** each positive integer n has a set of prime factors whose product is n (8 = 2 \* 2 \* 2, 9 = 3 \* 3, 10 = 2 \* 5, 11 = 11, 12 = 2 \* 2 \* 3). The approach is: find the **smallest** factor of n greater than 1, print it, divide n by it, and repeat with the smaller number. The lecture's picture of 858:

```
858 = 2 * 429 = 2 * 3 * 143 = 2 * 3 * 11 * 13
```

**Why the smallest factor greater than 1 is guaranteed prime:** if the smallest such factor `k` were composite, it would itself have a factor `j` with `1 < j < k`, and `j` would also divide `n`, contradicting that `k` is smallest. So we never need a separate primality test. *(extra context: that justification is the reason this simple loop is correct, and it is a common follow-up question.)*

**`smallest_factor`, step by step.** Two names are needed: `n` (the number, unchanged) and `k` (the candidate divisor, which changes). `k` starts at 2. The condition `n % k != 0` asks "does `k` fail to divide `n`?"; while that is a true value, increment `k`. The loop must terminate because `n` itself divides `n`, so `k` cannot grow past `n`. When the loop exits, `n % k == 0`, so `k` is returned as the smallest factor greater than 1. Note the pattern: the name in the condition (`k`) is exactly the name updated in the body.

**`prime_factors(858)`, step by step.** The loop condition is `n > 1`.

| Iteration | `n` at top | `k = smallest_factor(n)` | printed | `n = n // k` |
|---|---|---|---|---|
| 1 | 858 | 2 (since 858 % 2 == 0 immediately) | `2` | 429 |
| 2 | 429 | 3 (2 fails, 3 divides) | `3` | 143 |
| 3 | 143 | 11 (k tries 2..10, all fail) | `11` | 13 |
| 4 | 13 | 13 (13 is prime, so k climbs to 13) | `13` | 1 |
| exit | 1 | | | `1 > 1` is false, done |

Frames view: each call to `smallest_factor` creates its own frame with its own `n` and `k`, which disappears when the call returns. The `n` inside `smallest_factor` is a *different* binding from the `n` inside `prime_factors`; they live in different frames, and reassigning `n` in `prime_factors` cannot affect the other. Both frames have the global frame as parent, which is where the names `prime_factors` and `smallest_factor` are found.

Termination argument: `k >= 2` always, so `n // k` is strictly smaller than `n`, so `n` decreases every iteration and eventually reaches 1.

Also notice `prime_factors` **prints and returns `None`**, tying back to the first half of the lecture: `prime_factors(8)` cannot be used in an arithmetic expression. Its doctests show printed output, not a return value.

---

## Common Pitfalls

1. **Confusing printing with returning.** A function whose body only calls `print` returns `None`, so its "result" cannot be used in a larger expression. At the interactive prompt, printing 4 and returning 4 look identical; they are completely different.
2. **Thinking `return print(...)` returns the printed value.** It returns `None`, because that is what `print` returns.
3. **Expecting output when a function returns `None`.** The interactive interpreter never displays `None`, so a function that computes but never prints or returns appears to do nothing.
4. **Forgetting `return` and then wondering why you got `None`.** A body that finishes without executing a `return` statement returns `None` by default.
5. **Calling a non-pure function twice by accident.** Repeating `f(x)` in two places calls it twice and so duplicates its side effects (and its cost). Bind it to a name once.
6. **Using separate `if` statements when you meant `elif`.** With `x = 10`, two separate `if`s print both `big` and `positive`; an `if`/`elif` prints only `big`. Within one conditional statement, at most one suite ever runs.
7. **Putting `elif` or `else` before the end, or using `elif` without a preceding `if`.** The order is: exactly one `if`, then zero or more `elif`, then at most one `else`, last.
8. **Writing `=` where `==` belongs** in a header, or comparing with `=` in a boolean context.
9. **Assuming only `True`/`False` matter in a boolean context.** `0`, `''`, and `None` are false values; `-1`, `'0'`, and `'False'` are all true values.
10. **Expecting the `while` condition to be checked mid-body.** Once the header is true, the **entire** suite runs. In the summing example, `total = total + i` runs even when `i` has just become 3.
11. **Infinite loops from forgetting to update the name in the condition.** If nothing in the header expression changes (and there is no `return` in the body), the loop never ends.
12. **Using `/` instead of `//`** in `n = n // k`. True division produces a float (429.0), which breaks the integer reasoning and the `%` tests that follow.
13. **Assuming the local `n` in `smallest_factor` is the same as the `n` in `prime_factors`.** They are separate bindings in separate frames.
14. **Looking up names in the wrong frame.** A name is found in the **earliest** frame of the current environment that has it, so a local binding shadows a global one (the `def square(square)` example).

---

## Likely Exam Points

### 1. What does it print versus what does it return

**Q.** Given `def f(x): return x + 1`, what does the interactive interpreter display, and what does `h` return, for `def h(x): print(f(x))` when you type `h(3) + 1`?

**A.** It first prints `4` (from the `print` inside `h`), then raises a `TypeError`, because `h(3)` evaluates to `None` and `None + 1` is not allowed. `h` prints 4 and returns `None`.

### 2. Non-pure functions and repeated calls

**Q.** With `f = print`, what is displayed by `h(2)` for `def h(x): print(f(x)); return f(x)` versus `def h(x): y = f(x); print(y); return y`?

**A.** First version displays `2`, `None`, `2` and returns `None` (nothing extra displayed for `None`). Second displays `2`, `None` and returns `None`. The first calls `f` twice, so its side effect happens twice.

### 3. Order of side effects in nested calls

**Q.** With `noisy` as defined in lecture, what is the full output of `noisy(noisy(2) + noisy(3))`?

**A.**
```
NOISY 2
NOISY 3
NOISY 7
8
```
Operands are evaluated left to right before the outer call is applied; the final `8` is the displayed return value.

### 4. `elif` versus separate `if` statements

**Q.** For `x = 10`, what does each of these print?
```python
# (i)
if x > 2:
    print('big')
if x > 0:
    print('positive')

# (ii)
if x > 2:
    print('big')
elif x > 0:
    print('less big')
```

**A.** (i) prints `big` then `positive` (two independent statements, both conditions true). (ii) prints only `big` (one statement; once a clause's suite runs, remaining clauses are skipped).

### 5. True and false values

**Q.** Which of the following print `yes`? `if 0:`, `if ''`, `if None:`, `if -1:`, `if '0':`, `if 'False':`

**A.** Only `-1`, `'0'`, and `'False'` are true values, so only those three print `yes`. `0`, `''`, and `None` are false values.

### 6. While loop tracing, especially "the whole body runs"

**Q.** What are `i` and `total` after this code, and why is `total` not 3?
```python
i, total = 0, 0
while i < 3:
    i = i + 1
    total = total + i
```

**A.** `i` is 3 and `total` is 6. On the last iteration the condition `2 < 3` was true, so the **entire** suite ran: `i` became 3, and then `total = total + i` added 3, giving 6. The condition is only rechecked after the whole suite finishes.

### 7. Tracing `smallest_factor`

**Q.** How many times is the body of the `while` loop in `smallest_factor(143)` executed, and what is returned?

**A.** `k` starts at 2 and is incremented while `143 % k != 0`. It fails for k = 2 through 10 (9 increments), and succeeds at k = 11, so the body runs 9 times and 11 is returned.

### 8. Prime factorization output

**Q.** What does `prime_factors(12)` display? What does it return?

**A.** It prints
```
2
2
3
```
and returns `None`, since it has no `return` statement.

### 9. Why no primality test is needed

**Q.** `prime_factors` prints `smallest_factor(n)` without checking whether it is prime. Why is it guaranteed prime?

**A.** If the smallest factor `k > 1` of `n` were composite, it would have a factor `j` with `1 < j < k`, and `j` would divide `n` too, contradicting minimality of `k`.

### 10. Loop termination

**Q.** Why does the `while n > 1` loop in `prime_factors` always terminate for a positive integer `n`?

**A.** `smallest_factor(n)` returns some `k >= 2`, so `n // k` is strictly less than `n`. Since `n` strictly decreases each iteration and stays a positive integer, it must eventually reach 1, making the condition false.

### 11. Environments and name lookup

**Q.** In `def square(square): return mul(square, square)` followed by `square(4)`, why is the result 16 rather than an error?

**A.** The call expression `square(4)` is at top level, so its operator is looked up in the global frame, where `square` is the function. The body is evaluated in the new local frame, where `square` is bound to 4, and because lookup uses the **earliest** frame of the current environment, the body sees 4. So `mul(4, 4)` is 16.

### 12. Counting frames and environments

**Q.** For `square(square(3))`, how many `square` functions, how many frames, and how many distinct environments appear in the diagram?

**A.** One function, two local frames (`f1` for the call on 3, `f2` for the call on 9) plus the global frame, and three environments: global alone; `f1` then global; `f2` then global. No environment contains both `f1` and `f2`.

---

## Summary

- **Review:** calling a user-defined function creates a frame, binds formal parameters to arguments, and executes the body in that new environment. An environment is a sequence of frames; a name means the value bound in the **earliest** frame of the current environment containing it. Indentation tells you which environment a line runs in.
- **Compound statements** consist of clauses; a clause is a **header** plus its indented **suite**. The first header determines the statement type. To execute a suite: execute the first statement, then, unless directed otherwise, the rest.
- **Printing is a side effect; returning gives a value to the caller.** `print` returns `None`, and a function body that finishes without `return` returns `None`. `None` is never displayed by the interpreter.
- `return print(...)` returns `None`. Only a real `return` of a computed value lets a caller use the result in a larger expression.
- **Non-pure functions** have side effects that repeat with every call, so calling `f(x)` twice can produce different behavior than computing it once and naming the result.
- **Conditional statement rule:** consider clauses in order; evaluate each header expression; on the first true value, execute that suite and skip the remaining clauses. At most one suite runs. Syntax: one `if`, then zero or more `elif`, then at most one `else`, last.
- **Boolean contexts** care only about truth. False values so far: `False`, `0`, `None`, `''`. Everything else is a true value. Comparison operators: `>`, `<`, `>=`, `<=`, `==`, `!=`. Boolean operators: `or`, `and`, `not`.
- **While statement rule:** evaluate the header expression; if it is a true value, execute the **whole** suite and go back to step 1. The condition is never rechecked mid-body.
- To write a correct loop: decide **what names you need and what each means**, initialize them before the loop, and make sure something in the condition changes so the condition eventually becomes false (or the body returns).
- **Prime factorization:** repeatedly print the smallest factor greater than 1 and divide `n` by it (`n = n // k`) until `n` is 1. The smallest factor greater than 1 is always prime, and `n` strictly decreases, so the loop terminates. `smallest_factor` uses a second loop that increments `k` from 2 while `n % k != 0`.
