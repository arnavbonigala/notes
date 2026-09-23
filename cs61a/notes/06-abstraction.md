<!-- Wed, Sep 09, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 6: Abstraction

This lecture completes the higher-order function story from Monday and then steps back to ask what it means to *use* a function without knowing how it works. The first half is mechanics: lambda expressions (anonymous function values), how lambda functions get their parent frame (the frame in which the lambda expression is *evaluated*, which is fixed at creation time and never at call time), functions that return functions (`get_term`, `make_adder`, `cake`/`pie`), function currying (turning a two-argument function into a chain of one-argument functions), zero-argument functions, and how `return` finishes a call expression even from inside a `while` loop. The second half is the titular topic: *functional abstraction*, meaning that giving a computational process a name lets a caller rely on the function's domain, range, and behavior while ignoring its implementation entirely. That idea drives the lecture's advice on choosing names and its tour of the three kinds of errors (syntax, runtime, logical) and how to read a traceback. The unifying moral: a function is a value with an environment attached, and a good abstraction is one whose users never have to look inside it.

---

## Key Concepts

### 1. A lambda expression is an expression that evaluates to a function

Every other way of making a function so far has been a *statement* (`def`). A lambda expression is an expression, so it can appear anywhere a value can appear: as an operand in a call, on the right side of an assignment, inside a `return` statement, even as the operator of a call.

```python
lambda x: pow(1/2, x)
```

Read this out loud as: "a function with formal parameter `x` that returns the value of `pow(1/2, x)`." Three facts to internalize:

- **No `return` keyword.** The body is the expression, and its value is automatically the return value. Writing `lambda x: return x` is a syntax error.
- **The body must be a single expression.** No assignment statements, no `while`, no `if` statements (a conditional *expression* like `a if b else c` is fine, since that is an expression).
- **It has no intrinsic name.** `def square(x)` creates a function whose intrinsic name is `square`; a lambda function's intrinsic name is `<lambda>`. The name only exists so humans can inspect it.

Anything written with lambda can be rewritten with `def`, but not vice versa. `def` is strictly more expressive. So lambdas are for convenience and concision, typically when a function is small and is being passed straight into a higher-order function.

### 2. Lambda expressions are usually arguments to higher-order functions

The lecture's running example is `summation` from Monday:

```python
def summation(n, term):
    """Sum the first n terms of a sequence."""
    total, k = 0, 1
    while k <= n:
        total, k = total + term(k), k + 1
    return total
```

`term` is a formal parameter that gets bound to a *function*. Previously we had to write a named helper (`identity`, `cube`, `pi_term`) just to hand it over. With lambda, the function can be created at the call site:

```python
summation(5, lambda x: x)                # 1 + 2 + 3 + 4 + 5 = 15
summation(5, lambda x: pow(1/2, x))      # 1/2 + 1/4 + 1/8 + 1/16 + 1/32 = 31/32
```

The three "equivalent ways of writing this" from the slides:

```python
summation(5, lambda x: pow(1/2, x))     # lambda inline as an argument

term = lambda x: pow(1/2, x)            # lambda bound by assignment
summation(5, term)

def term(x):                            # plain def
    return pow(1/2, x)
summation(5, term)
```

All three produce the same answer. The middle one (`term = lambda ...`) is legal but stylistically discouraged in 61A: if you are going to give the function a name anyway, `def` is clearer and gives you a docstring and a useful intrinsic name.

### 3. A lambda function's parent is the frame in which the lambda expression is evaluated

This is the single most examined rule in the lecture. When any function value is created, it records a **parent frame**, and that parent is the frame that was current *at the moment of creation*. Calling the function later creates a new frame whose parent is that recorded frame, not the frame of whoever called it. Python is lexically (statically) scoped, not dynamically scoped.

Practical corollary from the transcript: "everything that is not indented at all is going to be evaluated in the global frame." So a lambda written at the top level has parent Global, even if the only place it can ever be *referred to* is inside some function's frame (because it was immediately passed in as an argument).

### 4. Returning functions, and what gets captured

```python
def get_term(common_ratio):
    def term(k):
        return pow(common_ratio, k)
    return term
```

`get_term` is a function factory. Each call creates a fresh `get_term` frame with its own binding for `common_ratio`, then creates a `term` function whose parent is *that* frame, then returns it. The returned `term` keeps a handle on the frame, so the value of `common_ratio` it will use is the one in its parent frame, looked up when `term` is eventually called. This is why `get_term(1/3)` produces a function that computes powers of 1/3 forever, regardless of what happens to any other variable named `common_ratio` anywhere else. `make_adder(n)` from the slides is the same pattern.

Note the subtlety in the wording: the function captures the *frame*, not a snapshot of the value. Name lookup happens at call time, walking from the new local frame to its parent to its parent's parent, up to Global.

### 5. Function currying

**Currying** converts a function that takes multiple arguments into a chain of functions that each take a single argument:

```python
pow(1/2, 5)            # normal two-argument call
curry(pow)(1/2)(5)     # curried: two successive one-argument calls
```

```python
def curry(f):
    def g(x):
        def h(y):
            return f(x, y)
        return h
    return g
```

Trace the call expression `curry(pow)(1/2)(5)` left to right, since call expressions associate leftward:
1. `curry(pow)` returns `g`, whose parent frame binds `f` to `pow`.
2. `g(1/2)` returns `h`, whose parent frame binds `x` to 1/2 (and whose grandparent binds `f`).
3. `h(5)` evaluates `f(x, y)` = `pow(1/2, 5)` = 0.03125.

Currying matters because higher-order functions like `summation` demand a *one-argument* function. `pow` takes two arguments, so it cannot be passed directly. `curry(pow)(3)` manufactures the one-argument function `lambda y: pow(3, y)` that `summation` needs.

### 6. Zero-argument functions

A lambda does not need parameters: `lambda: randint(1, 6)` is a function of no arguments. Since the body is evaluated fresh on each call, a zero-argument function is a way to *delay* a computation and to get a possibly different value each time you call it. This is exactly how a die is modeled in the Hog project: a die is a zero-argument function you call to get a roll. (extra context: the demo shown in lecture was along the lines of `def dice(n): return lambda: randint(1, n)`; the slide only says "Demo: Dice", so treat the specific code as illustrative rather than quoted.)

### 7. `return` completes the evaluation of a call expression

The transcript spends real time here. The mechanics:

- Evaluating a call expression for a user-defined function means executing the body in a *new* environment.
- You keep executing until you reach a `return` statement or fall off the end of the body (in which case the value is `None`).
- When `return` is reached, you switch back to the previous environment, and the call expression now has a value: the value of the return expression.
- **Only one `return` statement is ever executed per call.** Once it is reached, nothing else in the body runs, including the rest of a `while` loop.

That last point is why `return` is a legitimate way to break out of a loop.

### 8. Functional abstraction

Given:

```python
def square(x):
    return mul(x, x)

def sum_squares(x, y):
    return square(x) + square(y)
```

What does `sum_squares` actually need to know about `square`?

| Fact about `square` | Does `sum_squares` need it? | Why |
| --- | --- | --- |
| It takes exactly one argument | **Yes** | Otherwise it cannot call it correctly (its domain) |
| Its intrinsic name is `square` | **No** | Intrinsic names are only for human inspection; any function bound to the name `square` in the current environment works |
| It returns the square of its argument | **Yes** | You must know the behavior to use the abstraction |
| It computes that by calling `mul` | **No** | Implementation is the abstraction's private business |

So `square` could be `pow(x, 2)`, or a built-in, or something bizarre like summing `x` copies of `x`, and `sum_squares` remains correct. That is the payoff: you can replace an implementation without touching its clients. It also explains why higher-order functions work at all: `summation` only knows `term` takes one argument and returns a number.

### 9. Choosing names

Names do not affect correctness; they affect whether a human can read the program. Guidelines from lecture:

- Names should convey the **meaning or purpose** of the value, not its type. Document the type in the docstring instead.
- Function names typically convey an effect (`print_...`), a behavior (`triple`), or the value returned (`abs`).
- Prefer `rolled_one` over `true_false`; `dice` over `d`; `take_turn` over `play_helper` (name what it does, not who calls it); `num_rolls` over `my_int`.
- Avoid single letters that are hard to read in many fonts: lowercase `l`, capital `I`, capital `O` (confusable with 1 and 0). Prefer `k`, `i`, `m`.
- Short names are fine for generic quantities: `n` for a count, `f` for a wrapped function, `x` for any real number. Conventional letters (`i, j, k, m, n` for integers; `x, y, z` for reals; `f, g, h` for functions) are worth learning because programmers across languages use them.
- Long names are fine when they document: `average_age = sum(ages) / len(ages)` beats a comment plus a cryptic expression.
- **Don't repeat yourself.** If the same compound expression appears twice (say `sqrt(a*a + b*b)` in both a condition and an assignment), name it (`hypotenuse`) so a change happens in one place.
- **Don't build monster expressions.** Pull out a meaningful subpart (for example the `discriminant` of the quadratic formula) and name it, so each line is readable.

Writing programs is a creative act; these are guidelines, not laws, but conventions help everyone read each other's code.

### 10. Three kinds of errors, and tracebacks

| Kind | When detected | Example |
| --- | --- | --- |
| **Syntax error** | Before execution begins, while Python reads the file | Unbalanced parentheses, `1 /* 2`, a `+` at the start of an expression |
| **Runtime error** | By the interpreter while the program runs; produces a traceback | `TypeError`, `ZeroDivisionError`, `NameError` |
| **Logical / behavior error** | Not by Python at all; the program runs and does the wrong thing | Detected only by writing and running tests |

Two lessons about locating errors:

- A **syntax error** is reported at the first place the interpreter could *tell* something was wrong, which may be lines after the actual mistake. In the demo, an unclosed `abs(` parenthesis on one line caused the reported syntax error to land on a perfectly fine-looking `def` on line 7, because Python was still expecting the rest of the `abs` expression.
- A **traceback** reports the full chain of calls in progress when the error was detected, from the outermost expression down to the innermost frame. The last line names the error type and message; the lines above it show file and line for each active call. The line where Python *noticed* the problem is often not the line that needs fixing; it is a starting point for reasoning backward.

---

## Definitions

- **Lambda expression**: an expression that evaluates to a function, written `lambda <params>: <single expression>`. It creates a function with those formal parameters that returns the value of the body expression.
- **Anonymous function**: a function with no intrinsic name bound at creation. A lambda function's intrinsic name is `<lambda>`.
- **Higher-order function**: a function that takes a function as an argument and/or returns a function.
- **Formal parameter**: the name in a function's `def` or lambda header, which gets bound to an argument value in the new frame when the function is called. In `summation(n, term)`, `term` is a formal parameter that will be bound to a function.
- **Parent frame (of a function)**: the frame in which the function's `def` statement or lambda expression was evaluated. Recorded when the function is created and used as the parent of every frame created by calling it.
- **Frame**: an environment record created by a call, holding the bindings of that call's parameters and local names, plus a link to its parent frame.
- **Environment**: a sequence of frames, starting with the current (local) frame and ending with Global. Name lookup searches the frames in order.
- **Lexical (static) scoping**: the rule that a function's non-local names are resolved in the environment where the function was *defined*, not where it was called. (extra context: the term itself; the lecture teaches the rule without naming it.)
- **Intrinsic name**: the name a function carries with it from its `def` (or `<lambda>`), used in its `repr`, as opposed to any name currently bound to it in some frame.
- **Function currying**: converting a function that takes multiple arguments into a chain of functions that each take a single argument, so that `f(x, y)` becomes `curry(f)(x)(y)`.
- **Zero-argument function**: a function with no formal parameters, called as `f()`. Its body is re-evaluated on every call, which delays or repeats a computation.
- **Return statement**: a statement that completes the evaluation of a call expression by switching back to the previous environment and giving the call the value of the return expression. Only one return executes per call.
- **Functional abstraction**: giving a name to a computational process and then referring to that process as a whole, without regard for its implementation details. A correct client depends only on the abstraction's domain (how many arguments, of what kind), range (what it returns), and behavior.
- **Syntax error**: an error caused by badly formed program text, detected before execution begins.
- **Runtime error**: an error detected by the interpreter during execution, reported via a traceback.
- **Logical (behavior) error**: an error Python cannot detect; the program runs but computes the wrong thing. Found by testing.
- **Traceback**: a report describing which calls were in progress, and at which file and line, when a runtime error was detected.

---

## Worked Examples

### Example 1: `summation` with lambda, and the `pi_term` series

```python
def identity(k):
    return k

def cube(k):
    return pow(k, 3)

def summation(n, term):
    """Sum the first n terms of a sequence.

    >>> summation(5, cube)
    225
    """
    total, k = 0, 1
    while k <= n:
        total, k = total + term(k), k + 1
    return total

def pi_term(k):
    return 8 / (k * 4 - 3) / (k * 4 - 1)
```

(The slide text shows `k * 4  3` and `k * 4  1`; the minus signs were lost in PDF extraction. The reconstruction above is the standard 61A `pi_term`.)

The doctest: `summation(5, cube)` runs the loop with `k = 1, 2, 3, 4, 5`, accumulating `1 + 8 + 27 + 64 + 125 = 225`. Notice that `term(k)` inside the loop is a call to whatever function `term` is bound to, so `summation`'s body never mentions `cube`.

The lambda versions:

```python
>>> summation(5, lambda x: x)            # same as passing identity
15
>>> summation(5, lambda x: pow(1/2, x))  # geometric series, ratio 1/2
0.96875                                  # = 31/32
```

Why the simultaneous assignment `total, k = total + term(k), k + 1` matters: the right side is fully evaluated first using the *old* `k`, so `term(k)` uses the current term index before `k` is incremented. Writing it as two separate statements with `k` first would skip a term.

### Example 2: `06.py`, which `common_ratio` wins?

This is the heart of the lecture demo. Read the file top to bottom as a sequence of statements changing the Global frame.

```python
def cube(k):
    return pow(k, 3)

common_ratio = 1/2

def get_term(common_ratio):
    def term(k):
        return pow(common_ratio, k)
    return term

common_ratio = 1/3

term = get_term(common_ratio)

def summation(n, term):
    """Sum the first n terms of a sequence."""
    common_ratio = 1/4
    total, k = 0, 1
    while k <= n:
        total, k = total + term(k), k + 1
    return total

common_ratio = 1/5
print(summation(5, term))
```

Environment reasoning in words:

1. Global gets `cube`, then `common_ratio = 0.5`, then `get_term`, then `common_ratio` is rebound to `1/3` (the old 0.5 is simply forgotten; nothing points to it).
2. `get_term(common_ratio)` evaluates the operand to `1/3`, then creates frame **f1: get_term**, with parameter `common_ratio` bound to `1/3`. Note this parameter *shadows* the global name inside `get_term`.
3. Executing `def term(k)` inside that frame creates a function `term` whose **parent is f1**. `get_term` returns it. Global's `term` now points to that function, which still points back at f1, where `common_ratio` is `1/3`. Even though `get_term` has returned, f1 survives because a function refers to it.
4. `def summation(...)` is at the top level, so `summation`'s parent is Global.
5. `common_ratio = 1/5` rebinds the *global* name. The binding inside f1 is untouched.
6. `summation(5, term)` creates frame **f2: summation** with `n = 5` and `term` bound to our closure. Its first line sets a *local* `common_ratio = 1/4` in f2.
7. Inside the loop, `term(k)` creates a frame whose parent is **f1** (recorded at creation), not f2 and not Global. Looking up `common_ratio` finds no binding in the new local frame, so it goes to the parent f1 and finds `1/3`.

So the local `1/4` in `summation` is a decoy, and so is the global `1/5`. The sum is

```
(1/3)^1 + (1/3)^2 + (1/3)^3 + (1/3)^4 + (1/3)^5
  = 121/243
  ≈ 0.4979423868312757
```

**Contrast with the lambda variant linked from the slides**, which is the same program except the factory is replaced by a top-level lambda:

```python
common_ratio = 1/2
term = lambda x: pow(common_ratio, x)
common_ratio = 1/3
def summation(n, term):
    common_ratio = 1/4
    ...
common_ratio = 1/5
print(summation(5, term))
```

Now the lambda expression was evaluated at the top level, so `term`'s **parent is Global**. When `term(k)` is called from inside `summation`, `common_ratio` is looked up in Global *at call time*, where it is now `1/5`. Result: `1/5 + 1/25 + 1/125 + 1/625 + 1/3125 = 781/3125 = 0.24992`. Still not `1/4`, because `summation`'s local frame is never on `term`'s lookup path. The two programs differ only in where the function was created, and that changes the answer.

### Example 3: currying `pow` into `summation`

```python
def curry(f):
    def g(x):
        def h(y):
            return f(x, y)
        return h
    return g

print(summation(5, curry(pow)(3)))
```

Step by step:

1. `curry(pow)` creates a `curry` frame binding `f` to the built-in `pow`, defines `g` with that frame as parent, and returns `g`.
2. `(3)` calls `g` with `x = 3`. A `g` frame is created whose parent is the `curry` frame. It defines `h` with the `g` frame as parent and returns `h`.
3. `h` is now a one-argument function. Calling `h(y)` evaluates `f(x, y)`: `f` is found two frames up (the `curry` frame, the built-in `pow`), `x` is found one frame up (3), `y` is local.
4. `summation(5, h)` therefore sums `pow(3, 1) + pow(3, 2) + pow(3, 3) + pow(3, 4) + pow(3, 5)` = `3 + 9 + 27 + 81 + 243` = **363**.

Box-and-pointer style summary: `h` is a box pointing at its `g` frame, which points at its `curry` frame, which points at Global. Three frames stay alive to service one tiny function call.

The lambda rewrite (extra context: shown in past offerings, not in these slides): `curry = lambda f: lambda x: lambda y: f(x, y)`.

### Example 4: the lambda environment diagram from the transcript

```python
a = 1
def f(g):
    a = 2
    return lambda y: a * g(y)

f(lambda y: a + y)(a)
```

Work it as the lecture does, deciding each function's parent *before* drawing frames.

- The lambda on the last line, `lambda y: a + y`, is not indented at all, so it is evaluated in Global. Its **parent is Global**, therefore its `a` will be the global `a`. This is true even though the only name that will ever refer to it is `g` inside `f`.
- The lambda in `f`'s body is evaluated when `f` runs, so its **parent is the `f` frame**, therefore its `a` will be `f`'s local `a`, which is 2.
- The final `(a)` is also part of a global expression, so that `a` is the global 1.

Now the trace:

1. Global: `a = 1`, `f` bound to the `f` function (parent Global).
2. To evaluate `f(lambda y: a + y)(a)`, Python must first evaluate the operator `f(lambda y: a + y)`, which is itself a call expression.
3. Evaluate the operand: a new lambda function is created, parent Global. Call it `λ_add`.
4. Frame **f1: f** is created with `g` bound to `λ_add`. Body runs: `a = 2` is a *local* binding in f1. Then the lambda expression is evaluated in f1, creating `λ_mul` with parent f1, and returned.
5. The outer call is now `λ_mul(a)`. Look up `a` in Global: 1.
6. Frame **f2: λ** is created, parent f1, with `y = 1`. Body is `a * g(y)`. Look up `a`: not in f2, so go to parent f1, where `a` is 2. Look up `g`: not in f2, so go to f1, where `g` is `λ_add`.
7. Calling `λ_add(1)` creates frame **f3: λ**, parent **Global** (that is what `λ_add` recorded). Body `a + y`: `a` is not in f3, so look in Global and find 1; `y` is 1. Returns 2.
8. Back in f2: `2 * 2` = **4**.

The payoff sentence from lecture: "it's going to use the global `a` (1) instead of the local `a` (2)," because a function "gets its parent when it's created."

### Example 5: Lab 02 Q2, functions as values (cake / pie / snake)

```python
>>> def cake():
...     print('beets')
...     def pie():
...         print('sweets')
...         return 'cake'
...     return pie
...
>>> chocolate = cake()
beets
>>> chocolate
<function cake.<locals>.pie at ...>
>>> chocolate()
sweets
'cake'
>>> more_chocolate, more_cake = chocolate(), cake
sweets
>>> more_chocolate
'cake'
```

What happened:

- `cake()` runs the body, so `'beets'` is printed as a side effect, then `pie` is created (parent = the `cake` frame) and returned. `chocolate` is bound to that function value.
- Typing `chocolate` displays its `repr`. The intrinsic name is `pie`, and `cake.<locals>.pie` records that it was defined inside `cake`. No output from `print` because the function was never called.
- `chocolate()` calls `pie`: prints `sweets` (side effect), returns the string `'cake'`, which the interpreter displays with quotes.
- In the tuple assignment, `chocolate()` is evaluated (prints `sweets`, yields `'cake'`), and `cake` is evaluated as a *name*, not called, so `more_cake` is bound to the same `cake` function object. Distinguish carefully: `chocolate()` calls, `cake` does not.

```python
>>> def snake(x, y):
...     if cake == more_cake:
...         return chocolate
...     else:
...         return x + y
...
>>> snake(10, 20)
<function cake.<locals>.pie at ...>
>>> snake(10, 20)()
sweets
'cake'
>>> cake = 'cake'
>>> snake(10, 20)
30
```

Reasoning: `snake` looks up `cake`, `more_cake`, and `chocolate` in its parent, Global, *each time it is called*. Initially Global's `cake` and `more_cake` are the same function object, so the comparison is true and `snake` returns the `pie` function (displayed, not called). `snake(10, 20)()` calls what came back, printing `sweets` and returning `'cake'`. After `cake = 'cake'` rebinds the global name to a string, the comparison is false, so `snake` returns `10 + 20 = 30`. The function object itself never changed; only which name points to it did.

### Example 6: lambda and def are interchangeable in principle

```python
>>> (lambda f: lambda x: f(f(x)))(lambda y: y * y)(3)
81
```

The slide annotates the outer lambda as `twice` and the argument lambda as `square`. Evaluate left to right:

1. `(lambda f: lambda x: f(f(x)))` is a function; call it on `(lambda y: y * y)`, binding `f` to the squaring function. That returns `lambda x: f(f(x))`, whose parent frame holds `f`.
2. Call that on 3: `f(f(3))` = `f(9)` = 81.

The `def` rewrite is the same computation with names attached:

```python
>>> def twice(f):
...     def g(x):
...         return f(f(x))
...     return g
...
>>> def square(y):
...     return y * y
...
>>> twice(square)(3)
81
```

### Example 7: `return` inside a `while` loop

```python
def end(n, d):
    """Print the final digits of N in reverse order until D is found.

    >>> end(34567, 5)
    7
    6
    5
    """
    while n > 0:
        last, n = n % 10, n // 10
        print(last)
        if d == last:
            return None
```

Trace `end(34567, 5)`: `last, n = 7, 3456`, print 7, `5 != 7`, loop. `last, n = 6, 345`, print 6, loop. `last, n = 5, 34`, print 5, `d == last`, so `return None` ends the call immediately. The remaining digits 4 and 3 are never printed even though `n > 0` is still true. If `d` never appears, the loop runs out and the function falls off the end, also returning `None`.

### Example 8: `search`, `inverse`, and a square root by brute force

```python
def search(f):
    """Return the smallest non-negative integer x for which f(x) is a true value."""
    x = 0
    while True:
        if f(x):
            return x
        x += 1

def is_three(x):
    return x == 3

>>> search(is_three)
3
```

`while True` is a deliberately infinite loop; the only exit is the `return`. Note that `if f(x)` does not compare to `True`; it relies on truthiness. `0` is a false value and every other number is a true value, which the next example exploits.

```python
def square(x):
    return x * x

def positive(x):
    return max(0, square(x) - 100)

>>> search(positive)
11
```

`positive(x)` is 0 (false) for `x = 0..10` and 21 (true) at `x = 11`, so `search` returns 11, which is one more than the square root of 100. Generalizing that observation gives an inverse-function combinator:

```python
def inverse(f):
    """Return g(y) such that g(f(x)) -> x."""
    return lambda y: search(lambda x: f(x) == y)

sqrt = inverse(square)

>>> sqrt(256)
16
>>> sqrt(16)
4
```

Two lambdas, two different jobs: the outer one is the returned function `g`, with parent the `inverse` frame (so it can still see `f`); the inner one is created fresh on each call to `g`, with parent the `g` frame (so it can see both `f` and `y`). This `sqrt` only terminates for perfect squares, since it searches integers; the textbook's Newton's method section gives a real implementation.

Finally, the simplified `search`:

```python
def search(f):
    x = 0
    while not f(x):
        x += 1
    return x
```

Same behavior, less machinery. The lecture's advice: after you get something working, stare at it for a while and see whether the logic simplifies.

### Example 9: reading a traceback

```python
def f(x):
    return g(x - 1)

def g(y):
    return h(y) - h(1/y)

def h(z):
    z * z          # BUG: missing return

print(f(12))
```

Running this gives a `TypeError: unsupported operand type(s) for -: 'NoneType' and 'NoneType'`, reported at the line in `g`. Read the traceback bottom-up to get the error, and top-down to get the story: `print(f(12))` needed `f(12)`, which called `g(11)`, which called `h` twice and then tried to subtract. Both calls to `h` returned `None`, because a function that falls off the end of its body returns `None`. The reported line is line 5 in `g`, but the *fix* belongs in `h`: add `return`. Detection point and bug location are different things.

Now fix `h` and change the call to `print(f(1))`:

```
ZeroDivisionError: division by zero
```

reported at the same line in `g`, because `h(1/y)` with `y == 0` divides by zero. Again the reported line is not the culprit. The rest of the traceback is what saves you: it shows `y` became 0 because `x` was 1, which came from the top-level call `f(1)`, which is the thing that actually needs fixing.

Syntax error demo, for contrast: writing `1 /* y` is not a valid operator and Python refuses to run the file at all, pointing a caret at the offending character. And an unclosed `abs(` parenthesis was reported as a syntax error several lines later, at a `def`, because Python was still trying to finish the `abs` call expression and a `def` cannot appear inside one.

---

## Common Pitfalls

1. **Putting `return` in a lambda.** `lambda x: return x * x` is a syntax error. The body *is* the return expression.
2. **Trying to put a statement in a lambda body.** No assignments, loops, or `if`/`else` statements. `lambda x: x = 1` and `lambda x: while x: ...` are invalid. A conditional expression (`lambda x: 'pos' if x > 0 else 'neg'`) is legal because it is an expression.
3. **Thinking a lambda's parent is where it is *used*.** In `f(lambda y: a + y)`, the lambda's parent is the frame where the lambda expression was evaluated (Global if it is at the top level), even though it will only ever be reached through the name `g` inside `f`.
4. **Thinking a closure snapshots values.** `get_term`'s `term` does not copy `1/3`; it points at a frame. If that frame's binding changed later, the function would see the new value. What it will *never* see is a binding in a different frame, such as `summation`'s local `common_ratio`.
5. **Expecting dynamic scoping.** A caller's local variables are invisible to the function it calls unless the callee's parent chain includes that frame. `common_ratio = 1/4` inside `summation` affects nothing.
6. **Confusing `f` and `f()`.** `chocolate` displays a function; `chocolate()` calls it. In `more_chocolate, more_cake = chocolate(), cake`, one is called and one is not. Similarly `summation(5, cube)` passes the function, while `summation(5, cube(5))` passes a number and then crashes when `summation` tries to call it.
7. **Forgetting `return` in a helper.** The function silently returns `None`, and the error only surfaces later as a `TypeError` about `NoneType` in the *caller*.
8. **Forgetting the extra call in a curried chain.** `curry(pow)(1/2)` is still a function. Only `curry(pow)(1/2)(5)` produces a number. Passing `curry(pow)` to `summation` fails because `summation` calls `term(k)` once and gets a function back, then tries to add it.
9. **Confusing `print` with `return` in interactive output.** `chocolate()` shows `sweets` (printed, no quotes) and then `'cake'` (the displayed return value, with quotes). Two different mechanisms on two lines.
10. **Assuming the traceback's last line is the buggy line.** It is where the interpreter noticed the problem. Read the whole traceback.
11. **Assuming a syntax error is on the reported line.** It is the first place the mistake became detectable, which can be lines after an unclosed bracket.
12. **Naming by type or by caller.** `my_int` and `play_helper` tell a reader nothing useful. Say what the value represents (`num_rolls`) and what the function does (`take_turn`).
13. **Writing `f = lambda ...` in 61A code.** Legal, but style-penalized; use `def` when the function deserves a name.
14. **Splitting the simultaneous assignment in `summation`.** `total = total + term(k)` then `k = k + 1` works, but reversing the order silently skips the first term.

---

## Likely Exam Points

### 1. Evaluate a nested lambda call expression

**Q.** What does `(lambda x: lambda y: x + y)(3)(4)` evaluate to, and what does `(lambda x: lambda y: x + y)(3)` evaluate to?

**A.** The second is a *function* (displayed as `<function <lambda>.<locals>.<lambda> at ...>`); its parent frame binds `x` to 3. Applying it to 4 gives `7`. Call expressions associate left to right, so you finish one call before starting the next.

### 2. Which frame does a name come from?

**Q.** What does this print?

```python
a = 1
def f(g):
    a = 2
    return lambda y: a * g(y)
print(f(lambda y: a + y)(a))
```

**A.** `4`. The argument lambda is created in Global, so its `a` is 1 and `g(1)` returns `1 + 1 = 2`. The returned lambda was created inside `f`, so its `a` is `f`'s local 2, giving `2 * 2 = 4`. Rule: a function's parent is the frame in which its `def`/lambda was evaluated.

### 3. Closure capture versus global rebinding

**Q.** In `06.py`, three different values are assigned to the name `common_ratio` (`1/3` as the argument to `get_term`, `1/4` locally in `summation`, `1/5` in Global just before the call). Which one does `term(k)` use, and why?

**A.** `1/3`. `term` was defined inside `get_term`, so its parent is the `get_term` frame where the parameter `common_ratio` is bound to `1/3`. Name lookup goes local frame, then parent (`get_term` frame, found), and never consults `summation`'s frame (not on the chain) or Global (the search stopped earlier). The sum is `121/243 ≈ 0.4979`. If `term` had instead been a top-level `lambda x: pow(common_ratio, x)`, its parent would be Global and the answer would use `1/5`, giving `781/3125 = 0.24992`.

### 4. Write or complete `curry`

**Q.** Fill in the blanks so that `curry2(f)(x)(y)` equals `f(x, y)`, using only lambdas. Then give the value of `curry2(pow)(2)(10)`.

**A.**
```python
curry2 = lambda f: lambda x: lambda y: f(x, y)
```
`curry2(pow)(2)(10)` is `pow(2, 10)` = `1024`.

### 5. Use a curried function with a higher-order function

**Q.** What does `summation(5, curry(pow)(3))` print, using `curry` and `summation` from lecture?

**A.** `363`. `curry(pow)(3)` is a one-argument function equivalent to `lambda y: pow(3, y)`, so the sum is `3 + 9 + 27 + 81 + 243`.

### 6. Translate lambda code to `def` code (and confirm the value)

**Q.** Rewrite `(lambda f: lambda x: f(f(x)))(lambda y: y * y)(3)` using only `def` statements, and give the result.

**A.**
```python
def twice(f):
    def g(x):
        return f(f(x))
    return g

def square(y):
    return y * y

twice(square)(3)   # 81
```
Any program using lambdas can be rewritten with `def`; the reverse is not true, since a lambda body must be a single expression.

### 7. Function value versus function call (the cake/snake pattern)

**Q.** Using the lab's definitions, what are the outputs of `snake(10, 20)` and then, after `cake = 'cake'`, of `snake(10, 20)`?

**A.** The first displays `<function cake.<locals>.pie at ...>`, because Global's `cake` and `more_cake` are still the same function object, so `snake` returns `chocolate` (the `pie` function) without calling it. After `cake = 'cake'`, the comparison is false and `snake(10, 20)` returns `30`. Nothing about the function objects changed; only the global binding of the name `cake` did.

### 8. Trace `return` out of a loop

**Q.** What does `end(34567, 5)` print, and what does it return?

**A.** It prints `7`, `6`, `5` on separate lines and returns `None`. Reaching `return` ends the call immediately, so digits 4 and 3 are never printed even though the `while` condition is still true.

### 9. Write a higher-order search function

**Q.** Write `search(f)` returning the smallest non-negative integer `x` for which `f(x)` is a true value, in at most four lines.

**A.**
```python
def search(f):
    x = 0
    while not f(x):
        x += 1
    return x
```
Relies on truthiness: `0` and `False` are false values, any nonzero number is a true value. `search(positive)` returns `11` for `positive(x) = max(0, square(x) - 100)`.

### 10. What must a client know about an abstraction?

**Q.** `sum_squares(x, y)` calls `square`. Which of the following must `sum_squares` know: (a) that `square` takes one argument, (b) that `square`'s intrinsic name is `square`, (c) that `square` returns the square of its argument, (d) that `square` uses `mul`?

**A.** (a) and (c) only. You need the domain (so you can call it) and the behavior (so the result is right). The intrinsic name is only for human inspection, and the implementation is irrelevant: `square` could use `pow`, be built-in, or be something bizarre, and `sum_squares` still works.

### 11. Classify an error and locate the bug

**Q.** Given `def h(z): z * z` (no `return`), what error does `print(f(12))` raise for `f(x) = g(x-1)` and `g(y) = h(y) - h(1/y)`, and which line should you change?

**A.** A `TypeError` about unsupported operand types for `-` on `'NoneType'` and `'NoneType'`, a *runtime* error, reported at the subtraction line in `g`. The fix belongs in `h`: add `return`. The traceback shows where the interpreter noticed the problem, not necessarily where the bug is. (If instead you call `f(1)`, the same line raises `ZeroDivisionError`, and the fix belongs at the top-level call site that made `y` be 0.)

### 12. Naming critique

**Q.** Improve these names: `true_false`, `d`, `play_helper`, `my_int`. What is wrong with `l`, `I`, and `O` as names?

**A.** `rolled_one`, `dice`, `take_turn`, `num_rolls`. Names should say what a value represents or what a function does, not its type or who calls it. Lowercase `l`, capital `I`, and capital `O` are easily confused with the digits 1 and 0 in many fonts; prefer `k`, `i`, `m`, `n`.

### 13. Zero-argument functions

**Q.** What is the difference between `x = randint(1, 6)` and `d = lambda: randint(1, 6)`, and how do you get a roll from each?

**A.** The first evaluates the call once and binds a single number to `x`; reading `x` again gives the same number. The second binds a zero-argument function to `d`, delaying the computation, and each call `d()` re-evaluates the body and can give a different number. This is the Hog project's model of a die.

---

## Summary

- A **lambda expression** is an expression evaluating to a function: `lambda <params>: <one expression>`. No `return` keyword, no statements in the body, no intrinsic name (`<lambda>`).
- Lambdas can appear anywhere a value can: as an argument, in an assignment, in a `return`, or as the operator of a call. Anything written with lambda can be rewritten with `def`, but not conversely.
- **A function's parent frame is the frame in which its `def`/lambda was evaluated**, fixed at creation. Calling it later builds a frame whose parent is that recorded frame. Nothing about the caller's frame is visible unless it happens to be on that chain.
- Top-level (unindented) code is evaluated in Global, so a lambda written there has parent Global even if it is immediately passed into a function.
- Function factories (`get_term`, `make_adder`, `curry`, `inverse`) return functions that keep their defining frame alive, which is how `get_term(1/3)` produces a function that permanently uses `1/3`. In `06.py` the answer is `121/243`, unaffected by `common_ratio = 1/4` in `summation` or `1/5` in Global.
- **Currying** turns `f(x, y)` into `curry(f)(x)(y)`, a chain of one-argument functions. Useful for feeding multi-argument functions to HOFs that demand one argument: `summation(5, curry(pow)(3))` is `363`.
- **Zero-argument functions** (`lambda: ...`) delay a computation and re-evaluate on every call; that is how a die is modeled.
- A **`return` statement** completes the evaluation of a call expression, switches back to the previous environment, and ends the call, including out of a `while True` loop. Only one `return` runs per call; falling off the end returns `None`.
- `search(f)` plus truthiness (0 is false, other numbers are true) plus `inverse(f)` gives a brute-force square root, and simplifying the loop to `while not f(x)` shows the value of rereading working code.
- **Functional abstraction**: a client needs the abstraction's number and kind of arguments and its behavior; it does not need the intrinsic name or the implementation. This is what makes implementations swappable and HOFs possible.
- **Names are for people.** Convey purpose, not type; document types in the docstring; name repeated compound expressions; break up unreadable expressions; avoid `l`/`I`/`O`; short generic names and long documenting names are both fine in the right place.
- **Three kinds of errors**: syntax (before execution), runtime (traceback), logical (only tests find them). In both syntax errors and tracebacks, the reported line is where the interpreter *detected* the problem, which is often not the line to fix.
