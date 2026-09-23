<!-- Fri, Sep 04, 2026 | sources: slides + YouTube auto-transcript -->
# Lecture 5: Environments

## Overview

This lecture cashes in on the environment machinery introduced earlier: the whole reason environments and environment diagrams exist is to explain exactly how higher-order functions work. After warming up with two Fall 2022 Midterm 1 review problems (a nested `print`/`None`/`or` expression, and an iterative `classify` function for perfect/abundant/deficient numbers), the lecture shows that the existing diagram rules already handle functions passed as arguments (`apply_twice`) with no changes at all. It then introduces the genuinely new idea: nested `def` statements create functions whose **parent frame is not the global frame**, which is how a returned function like `adder` can still "remember" the `n` from the `make_adder` call that created it. From there we get a precise, hand-drawable recipe for environment diagrams (what to write when a function is defined, and what to write when one is called), the notion of **local scope** (why `f`'s `y` is invisible inside a separately defined `g`), function **composition** (`compose1`), **lambda expressions** (expressions that evaluate to functions, differing from `def` essentially only in the intrinsic name), and **currying** (turning a multi-argument function into a chain of one-argument higher-order functions).

---

## Key Concepts

### 1. Higher-order functions need no new diagram rules

A **higher-order function** is a function that takes a function as an argument, returns a function, or both. The lecture's headline point: *the environment diagram rules you already know already handle this case.* A function value is just a value, like `3` or `'hello'`. When you pass `square` as an argument, you bind the formal parameter to that function value in the new frame, exactly as you would bind a parameter to a number.

### 2. Name lookup follows the environment, frame by frame

An **environment** is a *sequence* of frames, starting with some local frame and ending with the global frame. To look up a name:

1. Look in the first frame of the current environment.
2. If not found, look in its parent frame.
3. Keep following parents until you reach the global frame.
4. The **first** binding you find is the value you use. If you reach the end without finding it, you get a `NameError`.

This single rule explains everything else in the lecture.

### 3. Every function has a parent frame; every frame has a parent frame

The four bullets the lecture calls "the key points":

- Every user-defined function has a **parent frame** (often, but not always, global).
- The parent of a function is **the frame in which it was defined**.
- Every local frame has a **parent frame** (often global).
- The parent of a frame is **the parent of the function called**.

Read those in order and they form a chain: when a `def` executes, the function records whichever frame is current *at definition time*. Later, when that function is called, the new frame copies that recorded parent. The frame's parent then determines the environment in which the body is evaluated. The lecture's own framing: "here's an explanation for why we have parents of functions in the first place: it's so that when we call those functions, we can write down the correct parent for the frame. Now, why do we need parents for frames? Well, that tells us how to find the current environment."

### 4. Nested `def` is where the parent stops being global

In every earlier example, functions were defined at the top level, so their parent was the global frame and every environment had length 2 (local, then global). A `def` inside another function's body executes while a *local* frame is current, so the inner function's parent is that local frame. Environments now have length 3 or more.

This is the mechanism behind "a function that has data inside it." `make_adder(3)` creates a frame `f1` with `n` bound to `3`, defines `adder` with `[parent=f1]`, and returns `adder`. Even after `make_adder` returns, frame `f1` does not disappear: the returned function still points at it, so calling `add_three(4)` creates a frame whose parent is `f1`, and `n` is found there.

### 5. Returning brings a value from the local frame back to the caller

A `return` statement "brings information from a local frame back into the frame that was the current frame when we called this function in the first place." Evaluation resumes exactly at the call expression that was being evaluated. In `add_three = make_adder(3)`, the returned function value becomes the value of the call expression, and ordinary assignment binds it to `add_three`. This is the same function object under a second name: you can give multiple names to the same function.

### 6. Local scope

Formal parameters (and any names assigned in a function body) have **local scope**: they are visible only in that function's body and in the bodies of functions *nested inside* it. They are not visible in some other function that merely happens to be *called* from it. The distinction is definition location, not call location.

### 7. Function composition

`compose1(f, g)` returns a new function `h` where `h(x)` is `f(g(x))`: apply `g` first, then `f`. The point of the example is generality: the same `compose1` builds `squiple` (triple then square), `tripare` (square then triple), and `squadder` (add 2, then square), including composing with a function that was itself produced by `make_adder`.

### 8. Lambda expressions

A **lambda expression** is an expression that *evaluates to a function*. The motivation in lecture: `square = x * x` does not define a function, it evaluates `x * x` right now and binds the resulting number. You need something on the right-hand side of `=` that evaluates to a function, and that is what `lambda x: x * x` gives you.

Rules and limits:
- Syntax: `lambda <formal parameters>: <return expression>`.
- **No `return` keyword.** You write the return expression directly after the colon. This is a quirk of the syntax.
- The body is limited to a single expression: lambdas "always create simple functions that do nothing but evaluate a single expression."
- Lambdas cannot contain statements. If you want a `while` inside a function body, you must use `def`.
- Lambdas are not especially common in Python (a `def` is more usual), but they are fundamental in other languages, and languages that lacked them are adding them.

### 9. Lambda vs `def`: the face-off

They are almost exactly the same:
- Both create a function with the same domain, range, and behavior.
- Both set the function's parent to the frame in which it was defined.
- In these examples, both end up binding the function to the name `square`.

The one difference: **only the `def` statement gives the function an intrinsic name.** A lambda first creates a nameless function; the assignment statement is a separate step that binds it. In `def`, creation and binding happen together as one effect of the statement.

In environment diagrams this shows up as: a lambda's function value is written with the Greek letter λ where `def`'s would say `square`, and the frame created when you call it is titled λ rather than `square`. The lecture calls this "tiny, really insignificant" for evaluation purposes: the names that matter for looking things up are the ones bound in frames, not the intrinsic name.

### 10. Currying

**Currying** is transforming a multi-argument function into a single-argument higher-order function that returns a function taking the rest of the arguments. `add(x, y)` takes both arguments at once; `make_adder(n)(k)` takes them one at a time via two call expressions. `curry2` expresses that relationship in general, converting any two-argument `f` into the `make_adder`-shaped version.

Historical note from lecture: currying was discovered by Moses Schönfinkel and later rediscovered and popularized by Haskell Curry, so "some people think it should be called Schönfinkeling."

### 11. Review material: `print`, `None`, and `or`

- **False values in Python** (so far): `False`, `0`, `''`, `None`. (The slide adds "more to come.")
- To evaluate `<left> or <right>`: evaluate `<left>`; if the result is a true value `v`, the whole expression is `v`; otherwise the expression is the value of `<right>`.
- `print` returns `None`, and that `None` gets passed to enclosing `print` calls, which is why nested prints produce cascading `None`s.

---

## Definitions

- **Higher-order function**: a function that takes another function as an argument value, or returns a function as a return value, or both.
- **Frame**: a binding structure created by a function call, holding bindings from names to values, labeled (`f1`, `f2`, ...) and titled with the name of the function called.
- **Global frame**: the frame in which top-level statements execute; the last frame of every environment.
- **Environment**: a sequence of frames, starting with a local frame (or the global frame) and continuing through parents to the global frame.
- **Current environment**: the environment in which the expression currently being evaluated is evaluated; it begins with the current frame.
- **Parent of a function**: the frame that was current when the function was defined (created).
- **Parent of a frame**: the parent of the function that was called to create that frame.
- **Local frame**: the frame created by applying a user-defined function.
- **Formal parameter**: a name in a function's parameter list, bound to an argument value in the new frame when the function is called.
- **Local scope**: the region in which a formal parameter (or other local name) is accessible, namely the body of the function in which it is bound, including bodies of functions nested within it.
- **Applying a user-defined function** (three steps): create a new frame; bind the formal parameters to the arguments; execute the body.
- **Lambda expression**: an expression that evaluates to a function, of the form `lambda <parameters>: <return expression>`, with no `return` keyword and a single-expression body.
- **Intrinsic name**: the name stored in the function value itself and shown when you display the function; `def` supplies one, lambda's is `<lambda>`.
- **Function composition**: combining functions `f` and `g` into a new function `h` where `h(x) = f(g(x))`.
- **Currying**: transforming a multi-argument function into a single-argument higher-order function that returns a function taking the remaining arguments.
- **Perfect / abundant / deficient number**: a positive integer `n` whose proper factors (factors of `n` below `n`) sum to exactly `n` / to more than `n` / to less than `n`.
- **False values (so far)**: `False`, `0`, `''`, `None`.

---

## Worked Examples

### Example 1: The nested `print` expression (Fall 2022 Midterm 1, Q1c)

```python
s = "Knock"
print(print(print(s, s) or print("Who's There?")), "Who?")
```

Work strictly inside out.

1. **Innermost**: `print(s, s)` prints `Knock Knock` and returns `None`.
2. **The `or`**: the left operand is now `None`, which is a false value, so `or` evaluates its right operand: `print("Who's There?")`. That prints `Who's There?` and returns `None`. Since we took the right branch, the value of the whole `or` expression is that `None`.
3. **Middle print**: that `None` is passed to `print(...)`, so it prints the line `None` and itself returns `None`.
4. **Outermost print**: it receives that `None` plus the string `"Who?"`, so it prints `None Who?`.

Output, in order:

```
Knock Knock
Who's There?
None
None Who?
```

**Why:** each `print` call both produces output *and* returns `None`, and a `None` return value fed into an outer `print` becomes visible output. The `or` short-circuits only on a true left value; `None` is false, so both prints ran.

### Example 2: `classify` (Fall 2022 Midterm 1, Q3)

Two poll questions framed the design:
- What do we need to compute? **(c) sum the factors less than n.**
- What should we iterate over? **(b) every integer less than n.**

Filling in the skeleton from the slides:

```python
def classify(n):
    """Return whether n > 1 is 'deficient', 'perfect', or 'abundant'.
    >>> classify(6)   # Proper factors 1, 2 and 3 sum to exactly 6.
    'perfect'
    >>> classify(24)  # Proper factors 1, 2, 3, 4, 6, 8, and 12 sum to 36.
    'abundant'
    >>> classify(23)  # Proper factor 1 sums to 1.
    'deficient'
    """
    total, k = 0, 1
    while k < n:
        if n % k == 0:
            total = total + k
        k = k + 1
    if total == n:
        return 'perfect'
    elif total < n:
        return 'deficient'
    else:
        return 'abundant'
```

**Step by step on `classify(6)`:** `total` starts at 0 and `k` at 1. `k = 1`: `6 % 1 == 0`, so `total` becomes 1. `k = 2`: divides, `total` becomes 3. `k = 3`: divides, `total` becomes 6. `k = 4`: `6 % 4 == 2`, skip. `k = 5`: skip. `k = 6`: loop condition `k < n` fails, so we stop *before* counting `n` itself, which is exactly what "proper factors" requires. `total == n`, so return `'perfect'`.

**Why the loop bound matters:** using `k <= n` would add `n` to `total` and make every number look abundant. The two blanks in the `if`/`elif` chain follow the definitions directly: equal means perfect, less means deficient, otherwise abundant.

### Example 3: `apply_twice` (functions as arguments)

```python
def apply_twice(f, x):
    return f(f(x))

def square(x):
    return x * x

result = apply_twice(square, 2)
```

**Environment reasoning in words:**

1. The two `def` statements execute in the global frame. Each creates a function value whose parent is the global frame, and binds the name (`apply_twice`, `square`) to it in the global frame. **Nothing has been squared yet.**
2. `apply_twice(square, 2)` is a call expression. Evaluate the operator `apply_twice` (a function), then the operands `square` (a function value, not a call) and `2`.
3. Apply the user-defined function: create frame `f1` titled `apply_twice`, parent global (copied from the function). Bind `f` to the `square` function and `x` to `2`. Execute the body, `return f(f(x))`.
4. To evaluate `f(f(x))`, evaluate the operator `f` and the operand expression `f(x)`. Looking up `f` in the current environment (`f1`, then global): it is found in `f1`, bound to `square`.
5. Inner `f(x)`: call `square` on `2` in a new frame `f2` (parent global, since `square`'s parent is global), giving return value `4`.
6. Outer call: `f(4)` calls `square` on `4` in frame `f3`, return value `16`.
7. `apply_twice` returns `16`, and the global assignment binds `result` to `16`.

**Why it matters:** no new rule was used. `f` is just a name bound to a value that happens to be a function, found by the ordinary lookup rule.

### Example 4: `make_adder` (nested `def`, returned function)

```python
def make_adder(n):
    def adder(k):
        return k + n
    return adder

three_more_than = make_adder(3)
result = three_more_than(4)
```

(The lecture also uses `add_three = make_adder(3); print(add_three(4))`, the same structure.)

**Step by step:**

1. `def make_adder` in the global frame: creates `func make_adder(n) [parent=Global]`, binds `make_adder` in the global frame. Its body is **not** executed.
2. `make_adder(3)`: create frame `f1` titled `make_adder`, `[parent=Global]`, with `n` bound to `3`. Execute the body.
3. First body statement is `def adder(k)`. The current frame is `f1`, so this creates `func adder(k) [parent=f1]` and binds the name `adder` **in f1**, not in the global frame. At this moment there is no way to refer to `adder` from the global frame.
4. `return adder`: look up `adder` in the current environment (`f1`, then global), find the function value in `f1`, and return it. Control goes back to the call expression `make_adder(3)` in the global frame, whose value is now that `adder` function.
5. Assignment binds `three_more_than` to that same function value. One function, two names (`adder` inside `f1`, `three_more_than` in the global frame).
6. `three_more_than(4)`: the function's parent is `f1`, so the new frame `f2` is titled `adder`, `[parent=f1]`, with `k` bound to `4`.
7. Execute `return k + n` in the environment `f2` -> `f1` -> Global. Look up `k`: found in `f2`, it is `4`. Look up `n`: not in `f2`, follow the parent to `f1`, found, it is `3`. So the result is `7`, bound to `result`.

**Why:** this is exactly where the "parent of a function is where it was defined" rule earns its keep. `f1` survives the return of `make_adder` because a live function value points at it.

### Example 5: "Wrong `make_adder()`?" variants

Which of these do not print `7` for `add_three = make_adder(3); print(add_three(4))`?

```python
# (A)                     # (B)                     # (C)                     # (D)
def make_adder(n):        def make_adder(n):        def make_adder(n):        def make_adder(n):
    k = 5                     n = 5
    def adder(k):             def adder(k):             def adder(k):             def adder(k):
                                                            k = 5
        return n + k              return n + k              return n + k              return n + k
    return adder              return adder              return adder              return adder

add_three = make_adder(3) add_three = make_adder(3) add_three = make_adder(3) add_three = make_adder(3)
                                                                                add_five = make_adder(5)
print(add_three(4))       print(add_three(4))       print(add_three(4))       print(add_three(4))
```

Reason each one out with the lookup rule:

- **(A)** `k = 5` is bound in the `make_adder` frame `f1`. When `adder(4)` runs, `k` is a formal parameter bound to `4` in the *adder* frame, which is the first frame searched, so the `k` in `f1` is shadowed and never consulted. `n` is still `3`. Prints **7** (correct).
- **(B)** `n = 5` rebinds `n` **in f1, before `adder` is called**. `adder`'s body looks up `n` in `f1` at call time and finds `5`. Prints **9** (incorrect).
- **(C)** `k = 5` inside `adder`'s body rebinds the local `k` to `5`, discarding the argument `4`, before `return n + k`. Prints **8** (incorrect).
- **(D)** Adding `add_five = make_adder(5)` creates a *second, separate* `make_adder` frame with its own `n`. `add_three` still points at the first frame. Prints **7** (correct).

Two further variants on a later slide, asking which produces an incorrect `add_three`:

```python
# (E)                             # (F)
def make_adder(n):                def make_adder(n):
    def adder(k):                     def adder(k):
        return n + k                      return n + k
    n = 5                             return adder
    return adder
                                  add_three = make_adder(3)
add_three = make_adder(3)         n = 5
print(add_three(4))               print(add_three(4))
```

- **(E)** is incorrect: `n = 5` executes in `f1` before returning, and since `adder` looks up `n` in `f1` at call time (not at definition time), it finds `5` and prints **9**. Note that it does not matter that the `def adder` came first; what matters is the value of `n` in `f1` when `adder`'s body runs.
- **(F)** is fine: `n = 5` happens in the **global frame**. `adder`'s environment is `adder frame` -> `f1` -> Global, and `n` is found in `f1` (bound to `3`) before the search ever reaches the global frame. Prints **7**.

The hint on the slide, "what changes in this environment diagram?", is pointing at exactly this: which frame does the assignment touch, and is that frame on the lookup path before some other binding of the same name?

A final variant, un-nesting the definition entirely:

```python
def make_adder(n):
    return adder

def adder(k):
    return n + k

add_three = make_adder(3)
print(add_three(4))
```

Here `adder` is defined in the *global* frame, so its parent is global. Calling `add_three(4)` creates a frame with parent Global, and looking up `n` searches that frame, then global: `n` is nowhere to be found, so this is a `NameError`. The `n` in `make_adder`'s frame is not on the lookup path.

### Example 6: Local scope and the `NameError`

```python
def f(x, y):
    return g(x)

def g(a):
    return a + y

f(1, 2)
```

**What happens:** `def f` and `def g` both execute in the global frame, so both functions have parent Global. `f(1, 2)` creates a frame with `x = 1`, `y = 2`, parent Global. Its body calls `g(x)`, creating a frame with `a = 1` and **parent Global** (copied from `g`, which was defined globally). Evaluating `a + y` in the environment (`g` frame, then Global): `a` is `1`; `y` is not in the `g` frame and not in the global frame, so Python raises an error that `y` is not defined.

**Why, in the lecture's words:** "this frame is not in the current environment." The frame for `f` exists, and it does contain `y`, but it is not on `g`'s lookup path because `g` was not *defined* inside `f`. The environment created by calling a top-level function (no `def` within a `def`) consists of one local frame followed by the global frame, full stop.

**Contrast with `make_adder`:** there, `adder`'s body *could* refer to `n` because `adder` was nested inside `make_adder`, making the `make_adder` frame its parent.

### Example 7: `compose1` and the two-environment diagram

```python
def square(x):
    return x * x

def triple(x):
    return 3 * x

def compose1(f, g):
    def h(x):
        return f(g(x))
    return h

squiple = compose1(square, triple)   # squiple(5) -> 3*5 = 15, 15*15 = 225
tripare = compose1(triple, square)   # tripare(5) -> 5*5 = 25, 3*25 = 75
```

Note the order: in `compose1(f, g)`, `g` is applied **first**.

Now the big example, composing with a function that was itself just produced:

```python
def make_adder(n):
    def adder(k):
        return k + n
    return adder

squadder = compose1(square, make_adder(2))
squadder(3)      # 3 + 2 = 5, then 5 * 5 = 25
```

The lecture notes you can write the whole thing as one call expression: `compose1(square, make_adder(2))(3)`.

**Step by step:**

1. Evaluate the operator `compose1` (easy, found in global) and the operands. `square` is easy. `make_adder(2)` is itself a call expression and requires work: create frame `f1` titled `make_adder`, `n = 2`, define `adder` with `[parent=f1]`, return it.
2. Call `compose1` with `f` bound to `square` and `g` bound to that returned `adder` function. This creates frame `f2` titled `compose1`, parent Global.
3. In `f2`, `def h(x)` creates `func h(x) [parent=f2]`, bound to `h` in `f2`. `compose1` returns `h`.
4. Call `h(3)`: frame `f3` titled `h`, `[parent=f2]`, `x = 3`. Body is `return f(g(x))`.
5. Evaluate `g(x)` first. Look up `f`, `g`, `x` in the environment `f3` -> `f2` -> Global: `x` is in `f3`; `f` and `g` are in `f2`.
6. `g` is the `adder` function, so calling it creates frame `f4` titled `adder`, `[parent=f1]`, with `k = 3`. Evaluating `k + n` uses the environment `f4` -> `f1` -> Global: `k = 3` in `f4`, `n = 2` in `f1`. Returns `5`.
7. Now call `f(5)`, that is `square(5)`, returning `25`. `h` returns `25`.

**Why this example is the climax:** there are **two different environments, each of length 3**, active in this one computation: `f3 -> f2 -> Global` for `h`'s body, and `f4 -> f1 -> Global` for `adder`'s body. They have different middle frames because `h` and `adder` were defined in different places. Together they produce `25`.

### Example 8: Lambda expressions

```python
>>> x = 10
>>> square = x * x       # NOT a function
>>> square
100
```

`x * x` is evaluated immediately to `100`, and `100` is what gets bound. To bind a *function*, the right-hand side must evaluate to a function:

```python
>>> square = lambda x: x * x
>>> square
<function <lambda> at ...>
>>> square(4)
16
```

Read `lambda x: x * x` as "a function with formal parameter `x` that returns the value of `x * x`." You can also apply a lambda immediately, with the lambda expression as the operator of a call expression:

```python
(lambda x: x * x)(4)     # 16
```

**Environment diagram difference:** with `square = lambda x: x * x` followed by `square(4)`, the global frame binds `square` to a function value whose intrinsic name is λ, and the call creates a frame titled λ with `x = 4`, returning `16`. With `def square(x): return x * x` followed by `square(4)`, the function value is named `square` and the frame is titled `square`. The names that actually matter for evaluation are the ones bound *in frames* (here, `square` in the global frame and `x` in the local frame); the intrinsic name just helps you keep track of which frame is which.

### Example 9: Currying

```python
def make_adder(n):
    return lambda k: n + k      # same effect as the nested def version

make_adder(2)(3)                # 5: two calls are required to get a number
```

Contrast with a plain two-argument function:

```python
from operator import add
add(2, 3)                       # 5: one call
```

`curry2` expresses the general relationship between these two shapes:

```python
def curry2(f):
    def g(x):
        def h(y):
            return f(x, y)
        return h
    return g

m = curry2(add)
add_three = m(3)
add_three(4)        # 7
```

The same thing as nested lambdas:

```python
curry2 = lambda f: lambda x: lambda y: f(x, y)
curry2(add)(2)(3)   # 5
```

**Why it works:** `curry2(add)` returns `g` with parent the `curry2` frame (where `f` is `add`). `g(3)` returns `h` with parent the `g` frame (where `x` is `3`). Calling `h(4)` evaluates `f(x, y)` in an environment where `y = 4` comes from `h`'s frame, `x = 3` from `g`'s frame, and `f = add` from `curry2`'s frame: a four-frame lookup chain ending at global.

---

## Common Pitfalls

- **Thinking a function's parent is where it was *called*.** It is where it was *defined*. This is the single most common source of wrong diagrams.
- **Thinking the frame for a returned function's call has the *caller's* frame as parent.** The frame's parent is copied from the function value, not from wherever the call happens to appear.
- **Assuming a local frame disappears after `return`.** It persists as long as some function value (or frame) still names it as parent. That is why `make_adder`'s `n` survives.
- **Believing names are captured at definition time.** The body looks names up *when it runs*. Variant (E) prints `9` precisely because `n` was reassigned in `f1` after `adder` was defined but before it was called.
- **Forgetting that a parameter shadows an outer name of the same name.** Variant (A) is correct because `adder`'s own `k` is found first.
- **Expecting a called function to see the caller's locals.** Example 6 raises a `NameError` for `y`. Only *nesting* creates access, not calling.
- **Writing `return` inside a lambda.** There is no `return` keyword in a lambda; you write the return expression directly after the colon.
- **Trying to put a statement (like `while` or an assignment) in a lambda body.** Lambdas hold a single expression; use `def`.
- **Confusing the two `compose1` orders.** In `f(g(x))`, `g` runs first. `squiple = compose1(square, triple)` triples then squares.
- **Writing `k <= n` in the `classify` loop**, which counts `n` as its own proper factor and makes everything abundant.
- **Forgetting that `print` returns `None`**, so nested prints print extra `None`s.
- **Forgetting that `or` returns a *value*, not a boolean.** `None or print(...)` evaluates the right side and yields its value.
- **Labeling function parents with a frame's title rather than its label.** Use `f1`, `f2`, ... because there can be several frames all titled `make_adder`, and each needs a unique label.

---

## Likely Exam Points

### 1. Drawing an environment diagram with a nested `def`

**Q.** For the code below, what is the parent of the frame created by the call `add_two(10)`, and what does the program print?

```python
def make_adder(n):
    def adder(k):
        return k + n
    return adder

add_two = make_adder(2)
add_two = make_adder(7)
print(add_two(10))
```

**A.** Two separate `make_adder` frames are created, call them `f1` (`n = 2`) and `f2` (`n = 7`). The second assignment rebinds `add_two` to the second `adder`, whose parent is `f2`. So the call frame's parent is `f2`, `n` is found as `7`, and the program prints `17`. The first `adder` and `f1` are simply no longer reachable by name.

### 2. "Which version breaks `make_adder`?"

**Q.** Does this print `7`? Explain using the lookup rule.

```python
def make_adder(n):
    def adder(k):
        return n + k
    n = n + 1
    return adder

add_three = make_adder(3)
print(add_three(4))
```

**A.** No, it prints `8`. `n = n + 1` rebinds `n` to `4` in the `make_adder` frame before `adder` is ever called. `adder`'s body looks up `n` at call time, following its parent to that frame, and finds `4`. `4 + 4 = 8`. (Same logic as slide variants B and E.)

### 3. Local scope / `NameError`

**Q.** What happens when you run this?

```python
def outer(a):
    return helper(a)

def helper(b):
    return a + b

outer(5)
```

**A.** A `NameError`: `a` is not defined. `helper` was defined in the global frame, so its parent is Global and its environment is (helper frame, Global). The `outer` frame containing `a` is not on that path. Calling a function does not grant access to the caller's locals; only nesting does.

### 4. Higher-order function evaluation order

**Q.** What does `apply_twice(make_adder(3), 4)` return, given the standard `apply_twice(f, x): return f(f(x))`?

**A.** `10`. `make_adder(3)` is evaluated as an operand *before* `apply_twice` is applied, producing an adder function with `n = 3`. Then `f(f(4))` is `adder(adder(4))` = `adder(7)` = `10`. Both calls to `adder` create frames whose parent is the same `make_adder` frame.

### 5. Lambda vs `def`

**Q.** Name every way in which `square = lambda x: x * x` differs from `def square(x): return x * x`.

**A.** Essentially one way: only the `def` statement gives the function an intrinsic name (the lambda's is `<lambda>`, which shows up when you display the function and as the frame title in a diagram). Both create a function with the same domain, range, and behavior, both set the parent to the frame of definition, and both bind it to the name `square`, though lambda does so in two steps (create nameless function, then assign) while `def` does both at once. Separately, a lambda's body is limited to a single expression and cannot contain statements.

### 6. Lambda syntax and evaluation

**Q.** What is the value of `(lambda x: lambda y: x * y)(3)(4)`, and what is the value of `lambda x: x * x` by itself?

**A.** `12`: the outer lambda applied to `3` returns a function that multiplies its argument by `3`; applying that to `4` gives `12`. The expression `lambda x: x * x` on its own evaluates to a function value (nothing is called, nothing is printed as a number).

### 7. Composition order

**Q.** Given `compose1(f, g)` returning `h(x) = f(g(x))`, what does `compose1(triple, square)(5)` return, with `square(x) = x * x` and `triple(x) = 3 * x`?

**A.** `75`. `g` is `square`, applied first: `5 * 5 = 25`. Then `f` is `triple`: `3 * 25 = 75`. (This is the lecture's `tripare`.) Compare `compose1(square, triple)(5)`, the `squiple`, which gives `(3*5)**2 = 225`.

### 8. Currying

**Q.** Write `curry2` using only lambda expressions, and use it to make a function that adds `10`.

**A.**

```python
curry2 = lambda f: lambda x: lambda y: f(x, y)
add_ten = curry2(add)(10)
add_ten(5)      # 15
```

Currying turns a two-argument function into a one-argument function returning a one-argument function.

### 9. `print` / `None` / `or`

**Q.** What does `print(print(1) or 2)` display?

**A.** Two lines: `1`, then `2`. The inner `print(1)` displays `1` and returns `None`; `None` is false, so `or` evaluates to `2`; the outer print displays `2`.

### 10. Iterative accumulation (`classify`-style)

**Q.** Fill in the loop condition and the `elif` for `classify` and say why the condition cannot be `k <= n`.

**A.** `while k < n:` and `elif total < n:`. Using `k <= n` would include `n` itself as a factor, adding `n` to `total` and making every `n > 1` classify as abundant, since proper factors exclude `n`.

---

## Summary

- A **higher-order function** takes and/or returns functions; the existing environment diagram rules handle it with **no changes**, because function values are just values.
- An **environment is a sequence of frames**. Name lookup checks the first frame, then its parent, then its parent, up to the global frame, and uses the **first** binding found; failing to find it is a `NameError`.
- **When a function is defined:** create a function value `func <name>(<params>) [parent=<label>]` whose parent is the **current frame**, and bind `<name>` to it in the current frame.
- **When a function is called:** (1) add a local frame titled with the function's name, (2) copy the function's parent onto the frame, (3) bind formal parameters to arguments, then execute the body in the environment starting at that frame.
- The four key rules: every user-defined function has a parent frame; **the parent of a function is the frame in which it was defined**; every local frame has a parent frame; **the parent of a frame is the parent of the function called**.
- **Nested `def`** is where parents stop being the global frame, producing environments of length 3 or more, and is how a returned function "remembers" data (`make_adder`'s `n` lives in a frame that survives the return).
- Bodies look names up **when they run**, not when they are defined, so reassigning a name in the enclosing frame before the call changes the result (variants B, C, E print `9`, `8`, `9`; A, D, F print `7`).
- **Local scope**: a called function sees only its own frame and its *definition* ancestors, never the caller's locals; that is why `f`'s `y` is invisible inside a top-level `g`.
- **`compose1(f, g)`** returns `h(x) = f(g(x))`, applying `g` first; composing `square` with `make_adder(2)` yields two distinct 3-frame environments active at once.
- A **lambda expression** evaluates to a function: `lambda <params>: <return expression>`, no `return` keyword, single expression only, no statements. It is nearly identical to `def` except that only `def` supplies an **intrinsic name**.
- **Currying** turns an `n`-argument function into a chain of one-argument higher-order functions: `curry2 = lambda f: lambda x: lambda y: f(x, y)`, so `curry2(add)(3)(4)` is `7`, mirroring `make_adder(3)(4)`.
- Review: false values so far are `False`, `0`, `''`, `None`; `<left> or <right>` returns `<left>`'s value if true, else `<right>`'s value; `print` returns `None`, which becomes visible output when passed to an outer `print`.
