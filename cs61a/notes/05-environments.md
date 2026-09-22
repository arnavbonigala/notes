<!-- Fri, Sep 04, 2026 | sources: slides (no transcript available) -->
# Lecture 5: Environments

This lecture warms up with two review problems from a past Midterm 1 (a `print`/`None`/`or` expression-evaluation puzzle and an iteration problem, "It's Perfect"), then spends most of its time on the central topic of the course's first third: **environment diagrams**, especially for higher-order functions. The big idea is that a name in Python has no meaning on its own: it only has meaning *relative to an environment*, which is a sequence of frames. Every user-defined function value carries a **parent frame** (the frame in which the `def` statement was executed), and every call creates a **local frame** whose parent is copied from the function being called. Once you internalize those two rules plus "look up a name by searching the current frame, then its parent, then its parent's parent, ... up to Global," you can predict exactly what `make_adder`-style code does, including all the subtly broken variants shown in the "Wrong `make_adder()`?" polls. A student quote from the Fall 2024 final survey is included on the slides to make the point bluntly: environment diagrams are extremely important, and time spent understanding them pays off for the rest of the course.

---

## Key Concepts

### 1. `print` returns `None`; printing and returning are different things

`print` is a function whose *effect* is to display text and whose *return value* is always `None`. That distinction is the entire trick behind the midterm problem on the slides. When you nest `print` calls, the inner call displays something and then hands `None` to the outer call, which displays `None` and then hands another `None` outward.

It helps to say the two roles out loud when tracing:

- **Effect (side effect):** text appears on the screen, in the order the calls actually happen.
- **Value:** what the call expression evaluates to, which for `print` is always `None`.

Also remember that the interactive interpreter does not display a `None` result, but `print(None)` does display the text `None`. In the midterm question every `None` you see in the output came from an explicit `print` of a `None` value.

### 2. False values and short-circuiting `or`

The lecture lists Python's false values so far:

```
False, 0, '', None      (more to come)
```

and gives the precise evaluation rule for `or`:

> To evaluate the expression `<left> or <right>`:
> 1. Evaluate the subexpression `<left>`.
> 2. If the result is a true value `v`, then the expression evaluates to `v`.
> 3. Otherwise, the expression evaluates to the value of the subexpression `<right>`.

Two things are worth noticing. First, `or` is **not** a call expression: it does not evaluate both operands up front. If the left operand is a true value, the right operand is never evaluated at all, so any side effects on the right (like a `print`) never happen. Second, `or` evaluates to one of its *operand values*, not to `True` or `False`. `3 or 4` is `3`, not `True`; `None or 'hi'` is `'hi'`. In the midterm expression, `print(s, s)` returns `None`, which is a false value, so the right operand `print("Who's There?")` does get evaluated (and does print).

### 3. The accumulation pattern in iteration

"It's Perfect" is the standard `while`-loop accumulation shape: initialize an accumulator and a counter, loop over a range of candidate values, conditionally update the accumulator, then advance the counter. The two poll questions on the slides isolate the two design decisions:

- **What do we need to compute?** (c) *sum* the factors less than `n`.
- **What should we iterate over?** (b) *every integer less than `n`* (then test each one for divisibility).

That second answer is the important one conceptually: you cannot directly "iterate over the factors," because you do not have them yet. You iterate over all candidates and use `n % k == 0` as a filter. This "iterate over everything, filter with an `if`" pattern is the workhorse of every counting/summing problem in CS 61A.

### 4. Environments: frames, bindings, and lookup

An **environment** is a sequence of frames. A **frame** is a table of **bindings** from names to values. Executing an assignment statement (`x = 5`), a `def` statement, or binding a formal parameter to an argument all create bindings *in the current frame*.

When Python evaluates a name, it does **not** magically know which one you mean: it searches the current frame first, then the current frame's parent frame, then that frame's parent, and so on up to the Global frame. The first binding found wins. (extra context) If no frame in the chain has a binding for that name, Python raises a `NameError`.

This search order is what makes **shadowing** work: an inner binding for `k` hides an outer binding for `k`, because the search stops as soon as it finds one.

### 5. Functions as values: passing them in and returning them out

Two separate slides make two separate points about higher-order functions:

- **`apply_twice`:** a name can be bound to a *functional argument*. When you call `apply_twice(square, 2)`, the formal parameter `f` is bound to the `square` function value in the new local frame, so `f(f(x))` calls `square` twice.
- **`make_adder`:** a `def` statement can appear *inside* another function's body, and the resulting function value can be *returned*. The returned function outlives the call that created it, and it still remembers the frame it was defined in.

### 6. The parent rules (the heart of the lecture)

The lecture states four bullets that together determine everything:

- Every user-defined function has a **parent frame** (often Global).
- **The parent of a function is the frame in which it was defined.**
- Every local frame has a **parent frame** (often Global).
- **The parent of a frame is the parent of the function called.**

Read those two bolded sentences as a pair. The first says: parents are fixed at `def` time, by *where the code is written and executed*, not by who calls it later. The second says: at call time you do not invent a parent, you *copy* the one recorded on the function value.

(extra context) This is called **lexical scoping** (or static scoping): the meaning of a free name in a function body is determined by the textual context in which the function was defined. The alternative, *dynamic scoping*, would look the name up in the caller's frame, and Python does not do that.

### 7. The mechanical recipe for drawing diagrams

Straight from the slides:

**When a function is defined:**
1. Create a function value labeled `func <name>(<formal parameters>) [parent=<label>]`.
2. Its parent is the **current frame**.
3. Bind `<name>` to the function value **in the current frame**.

**When a function is called:**
1. Add a local frame, titled with the `<name>` of the function being called.
2. Copy the parent of the function to the local frame: `[parent=<label>]`.
3. Bind the `<formal parameters>` to the arguments in the local frame.

(extra context) Two small conventions that Python Tutor and 61A exams use: the frame is titled with the *intrinsic name* of the function (the name in its `func` label), not with the name you happened to call it by, and frames are numbered `f1`, `f2`, `f3`, ... in creation order. Built-in functions like `print` do not get frames.

### 8. Lookup is deferred until the body actually runs

This is the subtlety that variants (C), (E), and the final slide are testing. A function body is **not** evaluated when the function is defined. The names inside it are resolved only when the function is **called**. So if the value bound to `n` in the parent frame changes between definition time and call time, the function sees the *new* value. The function remembers a **frame**, not a snapshot of values.

---

## Definitions

- **`None`:** the value representing "nothing"; the return value of any function (such as `print`) that has no explicit `return` of a value. It is a false value.
- **Side effect:** an observable consequence of evaluating an expression other than producing a value, such as displaying text. `print`'s side effect is displaying; its value is `None`.
- **False values (so far):** `False`, `0`, `''`, `None`. Everything else so far is a true value.
- **`<left> or <right>`:** evaluate `<left>`; if its value `v` is a true value, the expression evaluates to `v` and `<right>` is never evaluated; otherwise the expression evaluates to the value of `<right>`.
- **Short-circuiting:** the property of `and`/`or` that the right operand may never be evaluated, so its side effects may never occur.
- **Proper factor of `n`:** a factor of `n` that is strictly less than `n`.
- **Perfect / abundant / deficient number:** a positive integer `n` whose proper factors sum to exactly `n` / to more than `n` / to less than `n`, respectively.
- **Frame:** a table of bindings from names to values, created by a function call (a *local frame*) or existing from the start (the *Global frame*).
- **Binding:** an association in a frame between a name and a value.
- **Environment:** a sequence of frames, starting with a local frame (or the Global frame) and following parent links up to Global. Names are looked up in this sequence, first frame first.
- **Global frame:** the frame in which top-level statements execute; the root of every environment.
- **Parent frame of a function:** the frame in which the function's `def` statement was executed. Recorded on the function value as `[parent=<label>]`.
- **Parent frame of a local frame:** the parent of the function that was called to create that frame (copied, not recomputed).
- **Formal parameter:** a name listed in a function's `def` header; bound to an argument value in the new local frame at call time.
- **Higher-order function:** a function that takes a function as an argument or returns a function as a value (`apply_twice` does the former, `make_adder` the latter).
- **Shadowing:** an inner binding of a name hiding an outer binding of the same name, because lookup stops at the first match.
- **Lexical (static) scope:** (extra context) the rule that a function's free names are resolved in the environment where the function was *defined*, not where it was *called*.
- **Closure:** (extra context) the combination of a function value together with its parent frame. `make_adder(3)` returns a closure over a frame in which `n` is 3.

---

## Worked Examples

### Example 1: The nested `print` puzzle (Fall 2022 MT1 Q1(c))

```python
s = "Knock"
print(print(print(s, s) or print("Who's There?")), "Who?")
```

Trace it from the inside out, keeping "what gets displayed" and "what value comes back" in separate columns.

1. The outermost `print` is a call expression, so Python evaluates its operator (`print`) and then its operands **left to right**. The first operand is the whole expression `print(print(s, s) or print("Who's There?"))`; the second operand is the string `"Who?"`.
2. To evaluate that first operand, Python must first evaluate *its* argument: `print(s, s) or print("Who's There?")`.
3. `or` evaluates its left operand first: `print(s, s)`. This **displays** `Knock Knock` and **returns** `None`.
4. `None` is a false value, so `or` does not stop. It evaluates its right operand: `print("Who's There?")`. This **displays** `Who's There?` and **returns** `None`.
5. The `or` expression evaluates to that right-hand value: `None`.
6. Now the middle `print(None)` runs. It **displays** `None` and **returns** `None`.
7. Finally the outer call is `print(None, "Who?")`. It **displays** `None Who?` and returns `None`, which is discarded at the top level.

Output:

```
Knock Knock
Who's There?
None
None Who?
```

The slide's annotation "This `None` is passed to `print!`" is marking step 6: the `None` produced by the `or` is not silently ignored, it becomes an argument and therefore gets displayed.

### Example 2: `classify` (Fall 2022 MT1 Q3, "It's Perfect")

The skeleton from the slide, with the blanks filled in:

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

Why each blank is what it is:

- **`while k < n`:** we iterate over every integer below `n`. Starting at `k = 1` and stopping before `n` gives exactly the candidate proper factors. Using `k <= n` would include `n` itself, which is a factor of `n` but not a *proper* factor, and would make every number "abundant or perfect" incorrectly.
- **`if n % k == 0`:** the divisibility filter. `k` is a factor of `n` exactly when `n % k` is `0`.
- **`total = total + k`:** the accumulation step, only executed for actual factors.
- **`k = k + 1`:** the counter update. It sits *outside* the `if` so that every candidate is advanced past, factor or not. Putting it inside the `if` would loop forever the first time a non-factor appeared.
- **`elif total < n`:** we have already handled `total == n`, so the remaining split is "less than" (deficient) versus everything else (abundant).

Tracing `classify(6)`: `k` takes values 1, 2, 3, 4, 5. Factors found: 1, 2, 3, so `total` becomes 1, then 3, then 6, and stays 6. Loop ends with `k == 6`. `total == n`, so `'perfect'`.

Tracing `classify(23)`: 23 is prime, so only `k = 1` passes the filter and `total` ends at 1. `1 < 23`, so `'deficient'`.

### Example 3: Functions as arguments, `apply_twice`

```python
def apply_twice(f, x):
    return f(f(x))

def square(x):
    return x * x

result = apply_twice(square, 2)
```

Environment reasoning, in words:

1. **Global frame.** The first `def` creates a function value `func apply_twice(f, x) [parent=Global]` and binds the name `apply_twice` to it in Global. The second `def` creates `func square(x) [parent=Global]` and binds `square` to it in Global. Think of both names in Global as arrows pointing to function values drawn off to the right.
2. **The call `apply_twice(square, 2)`.** Python evaluates the operator `apply_twice` (a function value) and the operands: `square` evaluates to the `square` function value, `2` evaluates to `2`. Then, applying a user-defined function:
   - Create a new frame `f1: apply_twice`, with `[parent=Global]` copied from the function value.
   - Bind the formal parameters in `f1`: `f` points to the same `square` function value that Global's `square` points to (two arrows, one function), and `x` is `2`.
   - Execute the body: `return f(f(x))`.
3. **Inner call `f(x)`.** Look up `f` in `f1`: found, it is `square`. Look up `x` in `f1`: found, it is `2`. Create `f2: square [parent=Global]`, bind `x` to `2`, return `2 * 2` = `4`.
4. **Outer call `f(4)`.** Create `f3: square [parent=Global]`, bind `x` to `4`, return `16`.
5. `f1` returns `16`, and back in Global, `result` is bound to `16`.

Two points to notice. First, frame `f2` has its own `x` bound to `2`; that does not disturb `f1`'s `x`, because they are bindings in different frames. Second, `f2` and `f3` have parent `Global`, not `f1`, because `square` was defined in Global. The call chain and the parent chain are different things.

### Example 4: Nested `def`, `make_adder`

```python
def make_adder(n):
    def adder(k):
        return k + n
    return adder

three_more_than = make_adder(3)
result = three_more_than(4)
```

1. **Global:** `def make_adder` creates `func make_adder(n) [parent=Global]` and binds `make_adder` in Global.
2. **Call `make_adder(3)`:** create `f1: make_adder [parent=Global]`, bind `n` to `3`.
3. **Inside `f1`, the `def adder` statement executes.** This is a definition, so apply the definition rule: create a function value `func adder(k) [parent=f1]` (the current frame is `f1`, so `f1` is the parent), and bind the name `adder` **in `f1`**. Nothing inside `adder`'s body is evaluated yet.
4. **`return adder`:** look up `adder` in `f1`, found; `f1` returns that function value.
5. Back in Global, `three_more_than` is bound to that same function value. Note that `three_more_than` and `f1`'s `adder` are two names pointing at one function, and that function still records `[parent=f1]`. This is why `f1` cannot be forgotten.
6. **Call `three_more_than(4)`:** the function value's intrinsic name is `adder`, so create frame `f2: adder`, and copy the parent from the function value: `[parent=f1]`, **not** Global. Bind `k` to `4` in `f2`.
7. **Execute `return k + n`:** look up `k`, found in `f2` as `4`. Look up `n`: not in `f2`, so follow the parent link to `f1`, found as `3`. Return `7`.
8. `result` is bound to `7` in Global.

The whole magic of `make_adder` is step 6: the local frame's parent is `f1`, which is how the returned function "remembers" `n`.

### Example 5: "Wrong `make_adder()`?" variants (A) through (D)

The poll asks: **which of these do not print 7?**

**(A) An unused local `k` in the outer function:**

```python
def make_adder(n):
    k = 5
    def adder(k):
        return n + k
    return adder

add_three = make_adder(3)
print(add_three(4))
```

`f1: make_adder` has `n = 3` and `k = 5`. Calling `add_three(4)` creates `f2: adder [parent=f1]` with `k = 4`. Evaluating `n + k`: `n` is not in `f2`, so we follow the parent to `f1` and get `3`; `k` **is** in `f2`, so lookup stops there and gets `4`. The `k = 5` in `f1` is shadowed and never consulted. **Prints 7.** Correct.

**(B) Rebinding `n` in the outer function before the `def`:**

```python
def make_adder(n):
    n = 5
    def adder(k):
        return n + k
    return adder

add_three = make_adder(3)
print(add_three(4))
```

`f1` starts with `n = 3`, then the assignment `n = 5` rebinds `n` **in `f1`** to `5`. When `adder` later looks up `n` in its parent `f1`, it finds `5`. **Prints 9.** Broken.

**(C) Rebinding the parameter inside the inner function:**

```python
def make_adder(n):
    def adder(k):
        k = 5
        return n + k
    return adder

add_three = make_adder(3)
print(add_three(4))
```

`f2: adder` binds `k` to `4` at call time, but the very first statement of the body rebinds `k` to `5` in `f2`, throwing away the argument. `n + k` is `3 + 5`. **Prints 8.** Broken.

**(D) Creating a second adder:**

```python
def make_adder(n):
    def adder(k):
        return n + k
    return adder

add_three = make_adder(3)
add_five = make_adder(5)
print(add_three(4))
```

Each call to `make_adder` creates a **separate** frame: `f1` with `n = 3` and `f2` with `n = 5`, and each `def adder` creates a **separate** function value, one with `[parent=f1]` and one with `[parent=f2]`. `add_three` points to the first, so calling it creates a frame with parent `f1`, where `n` is `3`. The second call did not disturb the first. **Prints 7.** Correct.

**Answer: (B) and (C) do not print 7.**

### Example 6: "Wrong `make_adder()`?" variants (E) and (F)

The poll asks: **which results in an incorrect `add_three`, so prints a different result?** The slide hints: "what changes in this environment diagram?"

**(E) Rebinding `n` in the outer frame *after* the `def`:**

```python
def make_adder(n):
    def adder(k):
        return n + k
    n = 5
    return adder

add_three = make_adder(3)
print(add_three(4))
```

It is tempting to think `adder` was "already defined" with `n = 3` and so is safe. It is not. Defining `adder` does not evaluate `n`; it only records `[parent=f1]`. Then `n = 5` changes the binding *inside `f1`*, the very frame `adder` points to. At call time, `adder` looks up `n` in `f1` and finds `5`. **Prints 9.** Broken. This is exactly what the hint is pointing at: the arrow from `adder` to `f1` does not change, but the *contents* of `f1` do.

**(F) Assigning `n` in the Global frame after the call:**

```python
def make_adder(n):
    def adder(k):
        return n + k
    return adder

add_three = make_adder(3)
n = 5
print(add_three(4))
```

Here `n = 5` creates a binding in **Global**, not in `f1`. Calling `add_three(4)` creates `f2: adder [parent=f1]`; looking up `n` searches `f2` (not found), then `f1` (found, `3`), and stops before ever reaching Global. The Global `n` is shadowed by `f1`'s `n`. **Prints 7.** Correct.

**Answer: (E) is the broken one.** The lesson is that what matters is not *when* in the text a name is reassigned, but *which frame* the reassignment happens in.

### Example 7: Moving the inner `def` out to Global

The final slide's code:

```python
def make_adder(n):
    return adder

def adder(k):
    return n + k

add_three = make_adder(3)
print(add_three(4))
```

Apply the definition rule to each `def`. Both execute in the Global frame, so we get `func make_adder(n) [parent=Global]` and `func adder(k) [parent=Global]`. Notice `adder`'s parent is now **Global**, not any `make_adder` frame.

1. `make_adder(3)` creates `f1: make_adder [parent=Global]` with `n = 3`. Its body is `return adder`: `adder` is not in `f1`, so lookup follows the parent to Global and finds the global `adder` function value. `f1` returns it, and `add_three` is bound to it.
2. `add_three(4)` creates `f2: adder`, and copies the parent from the function value: `[parent=Global]`. Bind `k = 4`.
3. Execute `n + k`. Look up `n`: not in `f2`; follow parent to Global: **not there either** (nothing at top level binds `n`). There are no more frames.
4. Result: `NameError: name 'n' is not defined`. The `n = 3` in `f1` is completely unreachable from `f2`, because `f2`'s parent is Global, and `f1` is not on that chain at all.

This is the cleanest demonstration of the lecture's rule: the parent comes from *where the function was defined*, not from *who called it* or *whose frame was active at the time*. Even though `adder` was called "from inside" the world created by `make_adder`, its environment has nothing to do with `f1`.

---

## Common Pitfalls

- **Confusing printing with returning.** `print(...)` displays and returns `None`. A function whose body only prints returns `None`, so using its result in arithmetic raises a `TypeError`, and nesting it in another `print` displays `None`.
- **Forgetting that `None` is a false value.** In the midterm expression, students often assume the `or` stops after the first `print`. It does not, because `print` returned `None`.
- **Thinking `or` produces a boolean.** It produces one of its operand values. `0 or 'hi'` is `'hi'`, and `5 or crash()` is `5` with `crash()` never called.
- **Assuming both operands of `or` are always evaluated.** They are not; side effects on the right can be skipped entirely.
- **Putting the counter update inside the `if` in an accumulation loop.** In `classify`, moving `k = k + 1` inside `if n % k == 0` causes an infinite loop on the first non-factor.
- **Using `k <= n` instead of `k < n` for proper factors.** That includes `n` itself and breaks every classification.
- **Making a call frame's parent be the calling frame.** It is the parent *of the function being called*, copied from the function value. Example 7 shows exactly how badly this goes wrong.
- **Treating a returned function as a snapshot of values.** It holds a link to a frame, so later changes to bindings in that frame (variant (E)) are visible to it.
- **Missing shadowing.** In variant (A), the inner parameter `k` shadows the outer `k = 5`; in variant (F), `f1`'s `n = 3` shadows the Global `n = 5`. Lookup always stops at the first frame that has the name.
- **Believing an assignment in one frame affects a same-named binding in another.** Variant (B)/(E) changes `f1`'s `n` (which matters); variant (F) changes Global's `n` (which does not).
- **Forgetting that a `def` inside a function body binds the name in the *local* frame, not in Global.** After `make_adder` returns, the name `adder` does not exist in Global unless you bind it there.
- **Forgetting that each call creates a fresh frame.** Variant (D): `make_adder(5)` does not overwrite anything used by `add_three`.
- **Evaluating a function body at definition time.** Nothing inside a `def` body runs until the function is called, which is why undefined names inside a body do not cause errors until then (Example 7).

---

## Likely Exam Points

### 1. Trace a nested `print` / `or` expression and give exact output

**Practice:** What does the following display?

```python
x = 0
print(print(x) or print('hi') or 'done')
```

**Answer:**
```
0
hi
done
```
`print(x)` displays `0` and returns `None` (false), so `or` moves on. `print('hi')` displays `hi` and returns `None` (false), so `or` moves on again and the whole `or` expression evaluates to `'done'`, which the outer `print` displays. (Note: the outer `print` displays `done` without quotes.)

### 2. Short-circuiting and false values

**Practice:** What is the value of `'' or 0 or [] is not needed; use: '' or 0 or 'x'`, and how many times is `f` called in `f(1) or f(0)` if `f` returns its argument?

**Answer:** `'' or 0 or 'x'` evaluates to `'x'` (both `''` and `0` are false values, so evaluation continues to the last operand, and `or` returns that operand's value even though it is the last one). For `f(1) or f(0)`: `f(1)` returns `1`, a true value, so `or` stops immediately and returns `1`. `f` is called **once**.

### 3. Fill-in-the-blank iteration with an accumulator

**Practice:** Fill in the blanks so that `count_factors(n)` returns the number of factors of `n` (including `n` itself), for `n > 0`.

```python
def count_factors(n):
    count, k = 0, 1
    while ______:
        if ______:
            ______
        k = k + 1
    return count
```

**Answer:** `while k <= n:`, `if n % k == 0:`, `count = count + 1`. Note the `<=` here, unlike `classify`, because `n` itself counts as a factor. The structure is identical to `classify`: iterate over every candidate, filter with `%`, update the accumulator, advance the counter outside the `if`.

### 4. Draw the environment diagram for a higher-order function

**Practice:** For the code below, list every frame created, its parent, and its bindings, and give the final value of `result`.

```python
def compose(f, g):
    def h(x):
        return f(g(x))
    return h

def inc(x):
    return x + 1

result = compose(inc, inc)(5)
```

**Answer:**
- Global: `compose` to `func compose(f, g) [parent=Global]`, `inc` to `func inc(x) [parent=Global]`, `result` to `7`.
- `f1: compose [parent=Global]`, with `f` and `g` both pointing to the `inc` function value, and `h` to `func h(x) [parent=f1]`. Returns that `h`.
- `f2: h [parent=f1]`, with `x = 5`. Body evaluates `g(5)` first.
- `f3: inc [parent=Global]`, `x = 5`, returns `6`.
- `f4: inc [parent=Global]`, `x = 6`, returns `7`.
- `f2` returns `7`, so `result` is `7`.

The key checkpoints: `h`'s parent is `f1` (where it was defined), while `inc`'s frames have parent Global (where `inc` was defined), and `f2` does **not** have parent Global.

### 5. "Which variant is broken, and what does it print instead?"

**Practice:** What does this print, and why?

```python
def make_mul(n):
    def mul(k):
        return n * k
    n = n + 1
    return mul

print(make_mul(3)(4))
```

**Answer:** `16`. The `def mul` records `[parent=f1]` but evaluates nothing. Then `n = n + 1` rebinds `n` in `f1` to `4`. When `mul(4)` runs, its frame has parent `f1`, so `n` looks up to `4` and the body returns `4 * 4 = 16`. This is variant (E)'s lesson: reassigning in the parent frame after the `def` still affects the returned function.

### 6. Parent frame determined by definition site, not call site

**Practice:** What happens when this runs?

```python
def outer(m):
    return helper

def helper(j):
    return m * j

print(outer(2)(3))
```

**Answer:** `NameError: name 'm' is not defined`. `helper` was defined in Global, so its parent is Global and its call frame's parent is Global. `m` exists only in `outer`'s local frame, which is not on `helper`'s parent chain. Note that `outer(2)` itself succeeds (it finds the global `helper` by searching `f1`, then Global); the error happens only when the body of `helper` runs.

### 7. Shadowing and which frame an assignment targets

**Practice:** What does this print?

```python
x = 10
def f(x):
    def g():
        return x
    x = 20
    return g
print(f(1)())
```

**Answer:** `20`. `g`'s parent is `f`'s frame. `x` is bound to `1` there at call time, then rebound to `20` by the assignment in `f`'s body. `g` looks up `x` in its parent frame and finds `20`, never reaching the Global `x = 10`, which is shadowed.

### 8. Multiple independent closures from one factory

**Practice:** What does this print?

```python
def make_adder(n):
    def adder(k):
        return n + k
    return adder

a = make_adder(1)
b = make_adder(100)
print(a(1), b(1))
```

**Answer:** `2 101`. Each call to `make_adder` makes its own frame (`f1` with `n = 1`, `f2` with `n = 100`) and its own `adder` function value with a different parent. They do not interfere.

---

## Summary

- `print` displays as a side effect and **returns `None`**; `None` is a false value, so nested prints inside `or` chains keep going and end up displaying `None`.
- Python's false values so far: `False`, `0`, `''`, `None`.
- `<left> or <right>`: evaluate `<left>`; if it is a true value `v`, the expression is `v` and `<right>` is never evaluated; otherwise the expression is the value of `<right>`.
- Midterm output for the Knock-Knock expression: `Knock Knock`, `Who's There?`, `None`, `None Who?`.
- Accumulation pattern (`classify`): initialize `total, k = 0, 1`; loop `while k < n`; filter with `if n % k == 0`; accumulate; advance `k` **outside** the `if`. Iterate over every integer below `n`, not over "the factors."
- An **environment** is a sequence of frames; a name is looked up in the current frame, then its parent, then up the chain to Global; the first match wins (this is shadowing).
- **Definition rule:** a `def` creates `func <name>(<params>) [parent=<current frame>]` and binds `<name>` in the current frame. The body is not evaluated yet.
- **Call rule:** create a local frame titled with the function's name, **copy** the function's parent into the frame, and bind formal parameters to arguments.
- The parent of a function is where it was **defined**; the parent of a frame is the parent of the **function called**, never the caller's frame.
- Functions are values: they can be passed in (`apply_twice(square, 2)` gives `16`) and returned out (`make_adder(3)` gives a function that adds 3).
- A returned function remembers a **frame**, not a snapshot, so later reassignments *in that frame* change its behavior: variant (E), `n = 5` after the `def`, prints 9.
- Variants that break `make_adder`: **(B)** rebinding `n` in the outer frame before the `def` (prints 9), **(C)** rebinding `k` inside `adder` (prints 8), **(E)** rebinding `n` in the outer frame after the `def` (prints 9). Variants **(A)**, **(D)**, and **(F)** all still print 7, because of shadowing, fresh frames per call, and shadowing again, respectively.
- Moving the inner `def` to Global breaks the closure entirely: `adder`'s parent becomes Global, `n` is unreachable, and the call raises `NameError`.
- Environment diagrams are the single highest-leverage skill in this part of the course; practice by predicting the diagram before running Python Tutor, then checking.
