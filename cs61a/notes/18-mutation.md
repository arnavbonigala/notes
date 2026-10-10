<!-- Wed, Oct 07, 2026 | sources: slides + YouTube auto-transcript -->
# Lecture 18: Mutation

## Overview

Up to this point in CS 61A, every value we built was effectively frozen: a compound object such as a rational number or a tree was "nothing more than the totality of its pieces," so the only way to get a different value was to compute a new one. This lecture introduces the big new idea that **an object can change its value over time**. Lists and dictionaries are *mutable*: the same object can hold different contents at different moments, and every name bound to that object sees the change. From this one idea a long chain of consequences follows: the distinction between *identity* (`is`) and *equality* (`==`), the fact that a function can change its caller's data as a side effect, the fact that the same expression can evaluate to two different values without any name being reassigned, the trap of mutable default arguments, and finally the ability to write *mutable functions* whose behavior changes across calls because they keep **persistent local state** inside a mutable value (the `make_withdraw_list` bank account). The lecture also showed how mutation interacts with recursion (why we write `p = p + [t.label]` instead of `p.append(...)`, and how a one-element list `n = [0]` can accumulate a count across recursive calls), worked a past-midterm environment diagram question, and closed with f-strings and string interpolation.

---

## Key Concepts

### 1. The same object can change: the story of card suits

The lecture motivated mutation with the history of playing-card suits. Cards are believed to have been invented in Asia with three suits (coin, string, myriad) representing denominations of money. As cards travelled to Europe, suits were lost and replaced, and by the modern American/French deck we have hearts, diamonds, spades, clubs.

The point is that we can model this history with **one list that changes**, rather than a sequence of new lists:

```python
>>> suits = ['coin', 'string', 'myriad']
>>> original_suits = suits        # a second name for the SAME list
>>> suits.pop()                   # 'myriad' lost on the way to Europe
'myriad'
>>> suits.remove('string')        # 'string' lost too
>>> suits.append('cup')           # Spanish deck gains cups
>>> suits.extend(['sword', 'club'])
>>> suits[2] = 'spade'            # Italian: "espada" -> spade
>>> suits[0:2] = ['heart', 'diamond']   # French: coin & cup -> heart & diamond
>>> suits
['heart', 'diamond', 'spade', 'club']
>>> original_suits                # never reassigned, but changed anyway!
['heart', 'diamond', 'spade', 'club']
```

`original_suits` was bound way back at the top and never reassigned, yet it shows the modern suits. The reason: `original_suits` and `suits` were never two lists, they were **two names for the same object**. This is the central lesson:

> **All names that refer to an object are affected by a mutation of that object.**

The lecture's human analogy: bind the name `same_person` to a baby emoji; when the baby grows up, `same_person` is still bound to the same person, but the person has changed. In the real world we would not call growing up a "mutation," but in computer science *mutation* means any change to an object.

Only objects of **mutable types** can change value. So far we have seen two: **lists** and **dictionaries**.

### 2. Creating a new list vs. modifying an existing list

This distinction is the heart of the lecture, and it is the single most testable idea. The slides laid out both columns side by side.

**Create a new list** (the original is untouched):

| Operation | Example |
| --- | --- |
| List literal | `s = [1, 2, 3]` |
| List constructor | `t = list(s)` |
| List comprehension | `u = [x for x in s]` |
| `+` | `v = s + t` |
| Slicing | `w = t[1:]` |

**Modify an existing list** (no new list object; every alias sees the change):

| Operation | Effect |
| --- | --- |
| `s.append(4)` | adds `4` as one new element at the end |
| `s[1] = 'hi'` | element assignment: changes index 1 |
| `s.extend(t)` | adds *each element* of `t` to `s`; modifies `s` but **not** `t` |
| `s += t` | **same as `s.extend(t)`**, i.e. it mutates `s` |
| `s.remove(2)` | removes the *first* occurrence of the value `2` |
| `s.pop()` | removes **and returns** the last item |
| `s.pop(1)` | removes and returns the item at index 1 |

Two traps hide in this table:

- `append` adds **one** element even if that element is itself a list; `extend` adds **many**.
- `s = s + t` and `s += t` look interchangeable but are not. `s + t` builds a brand new list and rebinds the local name `s` to it, leaving the old list untouched for all other names. `s += t` mutates the list in place, so every other name bound to that list changes too.

### 3. Aliasing and nested lists

```python
s = [[1, 2], 3]
t = s[0]        # t and s[0] are the SAME list object
s[0][1] = 4     # mutates that shared list
s.append(t)     # s now contains that same list object twice
t.append(5)     # visible through s[0] AND s[2]
print(s)        # [[1, 4, 5], 3, [1, 4, 5]]
```

Box-and-pointer reasoning in words: the global frame has one arrow for `s` pointing at a two-box list; box 0 of that list holds an arrow to a second list `[1, 2]`; `t` is a *third* arrow to that same second list. Writing `s[0][1] = 4` follows `s`'s arrow, then box 0's arrow, and overwrites box 1 with `4`. `s.append(t)` grows `s` to three boxes and puts a *copy of the arrow* (not a copy of the list) in box 2. So after `t.append(5)`, both `s[0]` and `s[2]` display the longer list, because there is only one list there. Note also that `print(s)` shows the contents twice even though there is only one object.

### 4. Mutation can happen inside a function call

A function can change any mutable object in its scope, whether it was passed in as an argument or just referred to by name:

```python
>>> four = [1, 2, 3, 4]
>>> len(four)
4
>>> mystery(four)
>>> len(four)
2
```

Three different `mystery` definitions produce this behavior:

```python
def mystery(s):
    s.pop()
    s.pop()

def mystery(s):
    s[2:] = []      # replace the slice from index 2 onward with nothing

def mystery():       # takes no arguments at all!
    four.pop()
    four.pop()
```

The argument-taking versions work because calling `mystery(four)` creates a new frame in which the parameter `s` is bound to **the same list object** that `four` is bound to, not a copy. The no-argument version works because `four` lives in the global frame, and any function body can look up names in the global frame.

The deeper consequence: when you pass a mutable value to a function, you lose the guarantee that it will be the same afterward. The lecture framed this as the trade-off of mutation.

> **Upside of mutation:** you can model things that change over time.
> **Downside:** after you assign something and then use it, you no longer know what will be there, because something in between could have changed it.

### 5. Immutability: tuples (and numbers, and strings)

Tuples are **immutable**. There is no way to change their contents:

```python
>>> s = (1, 2, 3)
>>> s.append(4)       # AttributeError: no append; the operation does not exist
>>> s[1] = 5          # TypeError: tuples do not support element assignment
```

Immutability of the *object* is not the same as permanence of the *name*. You may freely build a new tuple and rebind:

```python
>>> s + (4,)          # evaluates to a new tuple; s itself is unchanged
(1, 2, 3, 4)
>>> s                 # still (1, 2, 3)
(1, 2, 3)
>>> s = s + (4,)      # NOW s is different, because we reassigned the name
```

**How do you actually tell whether an object changed?** The lecture's technique: **use two names**. If a second name that you never touch still shows the old value, nothing was mutated.

```python
>>> x = 2
>>> y = x + 1         # y is 3
>>> x = 5             # numbers are immutable: assigning 5 to x does not
>>> y                 # turn the number 2 into the number 5
3
```

```python
>>> s = (1, 2, 3)
>>> t = s + (4,)      # t is (1, 2, 3, 4)
>>> s = ...           # whatever you do to s
>>> t                 # t is unchanged; the only way to change t is to reassign t
(1, 2, 3, 4)
```

Contrast with lists:

```python
>>> a = [1, 2, 3]
>>> b = [a, 4]        # b contains a, it does not copy a
>>> a.append(9)
>>> b                 # b "changed" without ever being reassigned
[[1, 2, 3, 9], 4]
```

So: **if you assign a name to an immutable value, you are guaranteed to get back exactly what you assigned, forever, no matter what functions you call.** With a mutable value you get no such guarantee.

### 6. Why the value of an expression can change

Two independent reasons an expression such as `x + x` can give different answers at two points in a program:

1. **Name change (does not require mutability).** If `x` is reassigned in between, `x + x` changes. This was true from day one of the course, even with numbers.
2. **Object mutation (new in this lecture).** Even with **no reassignment of `x`**, if the object `x` refers to is mutated, `x + x` can change. For example if `x` is a list and we append to it.

Also important: **an immutable sequence can appear to change if it contains a mutable element.**

```python
>>> t = ([1, 2], 3)
>>> t[0].append(5)
>>> t
([1, 2, 5], 3)
```

The tuple's *composition* did not change: it is the same tuple holding arrows to the same two objects. But one of those objects is mutable and was changed, so the tuple looks different when printed.

### 7. Identity vs. equality: "the same" vs. "equal"

Before mutation, a compound object was nothing but the sum of its parts. With mutation, "a compound data object such as a list has an **identity** in addition to the pieces of which it is composed." A list can still be *the same list* even after its contents change.

The **identity operator** is `is` (and its negation `is not`):

- `exp0 is exp1` is `True` if both expressions evaluate to **the same object**.
- `exp0 == exp1` is `True` if both expressions evaluate to **equal values**.

**Identical objects are always equal, but equal values need not be identical.**

Case A: same object, two names.

```python
>>> a = [10]
>>> b = a
>>> a == b, a is b
(True, True)
>>> a.append(20)
>>> a == b            # still equal: a is [10, 20] and b IS a
True
```

Case B: equal but not the same.

```python
>>> a = [10]
>>> b = [10]
>>> a == b, a is b
(True, False)
>>> b.append(20)
>>> a, b
([10], [10, 20])
>>> a == b            # no longer equal, and we never reassigned a or b
False
```

They happened to have the same contents, so they were equal, but they were **never the same**, and when one changed the other did not. Continuing:

```python
>>> c = b             # c really IS b, two names for one thing
>>> c.pop()
20
>>> b                 # changed
[10]
>>> a                 # untouched
[10]
```

### 8. Mutable default arguments are dangerous

> **A default argument value is part of the function value itself, evaluated once when the `def` is executed, not generated anew on every call.**

```python
def f(s=[]):
    s.append(5)
    return len(s)
```

```python
>>> f()
1
>>> f()
2
>>> f()
3
```

Each call with no argument binds `s` to **the very same list** stored in the function object. Appending mutates it, and that mutation is still there on the next call. A function that looks completely simple has behavior that is different every time. This is something to watch out for as soon as you learn about mutation. (The lecture told this example twice, once appending `5` and once appending `3`; the mechanism is identical either way.)

### 9. Mutable values let us build mutable functions and local state

A **mutable function** is a function whose behavior varies over time: the same argument can produce different results on different calls. The lecture's example is a bank account.

```python
>>> withdraw = make_withdraw_list(100)
>>> withdraw(25)
75
>>> withdraw(25)
50
>>> withdraw(60)
'Insufficient funds'
>>> withdraw(15)
35
```

Here `withdraw` *is* the bank account: the argument is the amount to withdraw and the return value is the remaining balance. A second withdrawal of the same amount gives a different return value. This is **not a pure function**: something changes as a side effect of calling it. The information that changes (the balance) is **state**, and it is **local to the function**, because it is not written down anywhere else in the program.

Where is the balance stored? Inside a **mutable list referenced from the parent frame of `withdraw`**:

```python
def make_withdraw_list(balance):
    b = [balance]
    def withdraw(amount):
        if amount > b[0]:
            return 'Insufficient funds'
        b[0] = b[0] - amount
        return b[0]
    return withdraw
```

Key points the lecture stressed:

- `withdraw` checks `b[0]`, **not** `balance`. The parameter `balance` keeps its original value forever (it is just a number), so it would always say 100.
- `withdraw` never executes an assignment statement for the name `b`. **Assignment statements like `b = ...` always bind in the local frame** (Python does have `nonlocal` and `global` statements that change this, but this lecture does not use them). What `withdraw` does instead is **element assignment** on the object `b` refers to. The name `b` keeps pointing at the same list; the *contents* of that list change over time.
- This is how persistent local state is achieved: the list survives between calls because `withdraw`'s parent frame survives as long as `withdraw` does.

Environment diagram in words: calling `make_withdraw_list(100)` creates frame `f1` with `balance` bound to `100` and `b` bound to a one-element list containing `100`. Inside `f1` we define `withdraw`, a function value whose **parent is `f1`**, so its body can both read and mutate `b`. `make_withdraw_list` returns `withdraw`, which the global frame binds to the name `withdraw`. Calling `withdraw(25)` creates a new frame whose parent is `f1`; it looks up `b` in `f1`, finds `b[0]` is `100`, confirms `25 <= 100`, and performs element assignment so the list now contains `75`. The next call finds `75` already in the list, because it is the same list.

The takeaway: we have bundled **behavior together with data**, and unlike every earlier example in the course, the data changes every time we call the function. That is a natural consequence of having mutable values in the language.

### 10. Mutation and recursion

**(a) Why `p = p + [t.label]` and not `p.append(t.label)`?** From the previous lecture's `print_big`:

```python
def print_big(t, n):
    """Print the paths in t that have a sum larger or equal to n."""
    def extend_and_maybe_print(t, p):
        p = p + [t.label]          # NEW list each time
        if is_leaf(t):
            if sum(p) >= n:
                print(p)
        else:
            for b in t.branches:
                extend_and_maybe_print(b, p)
    extend_and_maybe_print(t, [])
```

Each recursive call needs **its own** path. `p = p + [t.label]` creates a brand-new list and rebinds the *local* name `p`, so the caller's `p` is untouched and each sibling branch starts from the same unextended path. By contrast `p.append(t.label)` and `p += [t.label]` both **mutate the single shared list**, so the first branch's labels would still be sitting in `p` when the second branch ran, and every printed path would be wrong.

**(b) Using a mutable value to accumulate across recursive calls (`bigs`).** Implement `bigs`, which takes a `Tree` instance `t` with integer labels and returns the number of nodes whose labels are larger than any label of their ancestors.

```python
def bigs(t):
    """Return the number of nodes in t that are larger than all their ancestors."""
    n = [0]
    def f(a, x):
        if a.label > x:            # x is the largest ancestor label so far
            n[0] += 1              # increment the shared total
        for b in a.branches:
            f(b, max(a.label, x))  # pass down the new max
    f(t, t.label - 1)              # root label is always larger than its "ancestors"
    return n[0]
```

Two mutation ideas appear here:

- `n = [0]` is a one-element list used as a **counter shared by every recursive call**. `n[0] += 1` is element assignment on a mutable object, so it works from inside `f` without any `nonlocal` statement. If `n` were the integer `0`, writing `n += 1` inside `f` would be an assignment to a local name and would fail.
- `max(a.label, x)` threads the largest-ancestor-so-far down the tree as an ordinary (immutable) argument, no mutation needed.
- The clever initialization `f(t, t.label - 1)` guarantees the root counts, since any integer is greater than itself minus one.

**(c) Building lists with `append` (`sums`).** Return all lists summing to `n` using positive numbers up to `m` with no adjacent repeats.

```python
def sums(n, m):
    """Return lists that sum to n containing positive numbers up to m that
    have no adjacent repeats, for n > 0 and m > 0.

    >>> sums(5, 1)
    []
    >>> sums(5, 2)
    [[2, 1, 2]]
    >>> sums(5, 3)
    [[1, 3, 1], [2, 1, 2], [2, 3], [3, 2]]
    >>> sums(5, 5)
    [[1, 3, 1], [1, 4], [2, 1, 2], [2, 3], [3, 2], [4, 1], [5]]
    >>> sums(6, 3)
    [[1, 2, 1, 2], [1, 2, 3], [1, 3, 2], [2, 1, 2, 1], [2, 1, 3], [2, 3, 1], [3, 1, 2], [3, 2, 1]]
    """
    result = []
    for k in range(1, min(m + 1, n)):   # k is the first number of the list
        for rest in sums(n - k, m):     # all ways to sum to n - k
            if rest[0] != k:            # no adjacent repeats
                result.append([k] + rest)
    if n <= m:
        result.append([n])
    return result
```

The recursive idea, in the slides' own words: "Start with a 1. What are all of the ways to build the rest of the list, now that we've used a 1? Figure out how to sum to 4."

- `min(m + 1, n)` as the upper bound does double duty: `k` must be at most `m`, and `k` must be strictly less than `n` so that the recursive subproblem `sums(n - k, m)` has a positive target.
- The base-ish case `if n <= m: result.append([n])` handles the single-element list `[n]`.
- Critically, we build with **`[k] + rest`, which creates a new list**, and append that to `result`. We do **not** mutate `rest`. Each `rest` came back from a recursive call and may be reused across several iterations of the `k` loop; mutating it would corrupt all the other answers that share it. The accompanying demo deliberately did the wrong thing (looping over `result` while appending the *same* list objects back into it after mutating them) to show how quickly aliasing turns a list of results into a tangle of duplicate arrows to the same object.

### 11. f-strings and string interpolation

**String interpolation** means evaluating a string literal that contains expressions, so that those expressions are evaluated and become part of the string. It is used whenever you want a string built from some fixed text plus the result of evaluating an expression. You could already do this with concatenation:

```python
>>> pi = 3.14159
>>> print('pi starts with ' + str(pi) + '...')
pi starts with 3.14159...
```

f-strings (note the `f` before the quotation mark) are the **preferred method of string interpolation in Python**, and are both faster and more concise:

```python
>>> f'pi starts with {pi}...'
'pi starts with 3.14159...'
>>> print(f'pi starts with {pi}...')
pi starts with 3.14159...
```

Rules the lecture highlighted:

- An f-string evaluates to an ordinary string, so it can be embedded in a larger expression.
- Text inside `{ }` is treated as a **Python expression and evaluated**; text outside is literal.
- You need the `f`. Without it you just get a regular string containing literal curly braces: `'2 + 2 = {2 + 2}'`.
- The sub-expressions can be **arbitrary** and are evaluated **in the current environment**:

```python
>>> f'2 + 2 = {2 + 2}'
'2 + 2 = 4'
>>> f'{abs(-4)}'
'4'
>>> abs = float          # rebinding abs changes the result
>>> f'{abs(-4)}'
'-4.0'
```

Anything you could build with concatenation you can build with interpolation.

- The result contains the **`str` string** of each sub-expression's value, not the `repr` string. If an object `half` has different `str` and `repr`, then `f'half of a half is {half * half}'` shows the `str` form. If you want the `repr`, call `repr(...)` explicitly inside the braces.
- **Sub-expressions are evaluated like anywhere else in Python, so they may have side effects**, left to right. This is the mutation connection:

```python
>>> s = [9, 8, 7]
>>> f'because {s.pop()} {s.pop()} {s}'
'because 7 8 [9]'
```

The first `s.pop()` returns `7`, the second returns `8`, and by the time `{s}` is evaluated only `[9]` is left.

- Detailed restrictions on quotes and braces inside f-strings are **not required content** in the course.

---

## Definitions

- **Mutation**: any change to an object. In computer science this covers all changes to an object's contents, whether or not the change would be called a "mutation" in everyday life.
- **Mutable type**: a type whose objects can change in value. Seen so far: `list`, `dict`.
- **Immutable type**: a type whose objects can never change in value. Seen so far: `int`, `float`, `str`, `tuple`.
- **Alias**: a second name (or a second slot in some container) bound to the same object. All aliases of an object see every mutation of it.
- **Element assignment**: a statement of the form `s[i] = v` (or `d[key] = v`), which mutates the existing container rather than creating a new one.
- **Slice assignment**: a statement of the form `s[i:j] = other_sequence`, which replaces that slice of `s` in place. `s[2:] = []` deletes everything from index 2 onward.
- **Identity** (`is` / `is not`): `exp0 is exp1` evaluates to `True` if and only if both expressions evaluate to **the same object**.
- **Equality** (`==`): `exp0 == exp1` evaluates to `True` if both expressions evaluate to **equal values**. Identical objects are always equal; equal objects are not necessarily identical.
- **Identity of a compound object**: the property that makes a list "the same list" even after its contents change. With mutation, a compound object is more than the totality of its pieces.
- **Pure function**: a function whose only effect is to return a value. A function that mutates an object is **not** pure, because it has a side effect.
- **Side effect**: any change a function makes beyond returning its value, such as mutating an argument or printing.
- **Mutable function**: a function whose behavior varies over time, so that the same argument can give different return values on different calls.
- **Local state**: information that belongs to a particular function, is not recorded anywhere else in the program, and determines that function's behavior.
- **Persistent local state**: local state that survives across multiple calls to a function, so that information accumulates between calls. Achieved here by storing a mutable value (a one-element list) in the function's parent frame.
- **Default argument value**: the value given in a `def` for a parameter. It is evaluated once and becomes **part of the function value**, not regenerated per call. A mutable default argument therefore persists its mutations across calls.
- **String interpolation**: evaluating a string literal that contains expressions, so that the expressions' values become part of the resulting string.
- **f-string**: a string literal prefixed with `f` whose `{ }`-delimited sub-expressions are evaluated in the current environment, with the `str` of each value inserted into the result.

---

## Worked Examples

### Example 1: list mutation in a loop (the reversal that is not a reversal)

```python
s = [3, 5, 7, 9, 11]
for i in range(len(s)):
    s[i] = s[-(i + 1)]
```

Step by step, remembering that each assignment mutates `s` immediately, so later reads see earlier writes:

- `i = 0`: `s[-1]` is `11`. `s` becomes `[11, 5, 7, 9, 11]`.
- `i = 1`: `s[-2]` is `9`. `s` becomes `[11, 9, 7, 9, 11]`.
- `i = 2`: `s[-3]` is `7`. `s` becomes `[11, 9, 7, 9, 11]` (no change).
- `i = 3`: `s[-4]` is now `9` (already overwritten!). `s` stays `[11, 9, 7, 9, 11]`.
- `i = 4`: `s[-5]` is now `11`. `s` stays `[11, 9, 7, 9, 11]`.

Final: `[11, 9, 7, 9, 11]`. A naive reader expects a reversal, but because the list is mutated in place, the second half of the loop reads values the first half already clobbered. This is exactly the hazard mutation introduces.

### Example 2: nested list aliasing

```python
s = [[1, 2], 3]
t = s[0]
s[0][1] = 4
s.append(t)
t.append(5)
print(s)
```

- `s = [[1, 2], 3]`: one outer list with two boxes. Box 0 holds an arrow to an inner list `[1, 2]`.
- `t = s[0]`: `t` is a second arrow to that same inner list. No copy was made.
- `s[0][1] = 4`: follow `s`, then box 0, then write box 1. The inner list is now `[1, 4]`. Both `s[0]` and `t` show this.
- `s.append(t)`: the outer list grows to three boxes; box 2 holds an arrow to the **same** inner list. The inner list now has two arrows from inside `s` plus `t`.
- `t.append(5)`: the inner list becomes `[1, 4, 5]`.
- `print(s)` outputs `[[1, 4, 5], 3, [1, 4, 5]]`. Printed twice, but there is only one such list in memory. Note `s[0] is s[2]` is `True`.

### Example 3: dictionary mutation

```python
>>> numerals = {'I': 1, 'V': 5, 'X': 10}
>>> numerals
{'I': 1, 'V': 5, 'X': 10}
>>> numerals['X']
10
>>> numerals['X'] = 11          # change the value bound to an existing key
>>> numerals['X']
11
>>> numerals
{'I': 1, 'V': 5, 'X': 11}       # the old binding X -> 10 is gone completely
>>> numerals['L'] = 50          # element assignment also ADDS new entries
>>> numerals['L']
50
>>> numerals.pop('X')           # pop takes a KEY and removes that key-value pair
11
>>> numerals['X']
KeyError: 'X'
```

Note the contrast with lists: `list.pop` takes an **index** (and defaults to the last item), while `dict.pop` takes a **key**.

### Example 4: a function that shortens its argument

```python
>>> four = [1, 2, 3, 4]
>>> len(four)
4
>>> def mystery(s):
...     s.pop()
...     s.pop()
>>> mystery(four)
>>> len(four)
2
>>> four
[1, 2]
```

Environment reasoning: the global frame binds `four` to a four-box list. Calling `mystery(four)` opens frame `f1` with parameter `s` bound to **the same list object** (the arrow is copied, not the list). `s.pop()` mutates that one list, so `four`'s value is forever changed. `four` is not bound to anything different than before; it is just that the list itself is different now.

A variant with the same effect, using slice assignment:

```python
def mystery(s):
    s[2:] = []      # replace everything from index 2 on with no elements at all
```

And a variant that takes no arguments, reaching into the global frame:

```python
def mystery():
    four.pop()
    four.pop()
```

### Example 5: the mutable default argument

```python
def f(s=[]):
    s.append(5)
    return len(s)
```

- The `def` statement is executed once. The empty list is created once and stored **inside the function value**.
- `f()` binds `s` to that list, appends `5` so it is `[5]`, returns `1`.
- `f()` again binds `s` to **that same list**, appends `5` so it is `[5, 5]`, returns `2`.
- `f()` again returns `3`.

Each call sees the mutations of all previous calls, so a function that looks pure gives a different answer every time.

### Example 6: `make_withdraw_list`, persistent local state

```python
def make_withdraw_list(balance):
    b = [balance]
    def withdraw(amount):
        if amount > b[0]:
            return 'Insufficient funds'
        b[0] = b[0] - amount
        return b[0]
    return withdraw

>>> withdraw = make_withdraw_list(100)
>>> withdraw(25)
75
>>> withdraw(25)
50
>>> withdraw(60)
'Insufficient funds'
>>> withdraw(15)
35
```

Trace, frame by frame:

1. `make_withdraw_list(100)` creates frame `f1` (parent: Global) with `balance = 100`. The body creates a list `[100]` and binds `b` to it in `f1`. It defines `withdraw`, a function value whose **parent is `f1`**. It returns `withdraw`, which Global binds to the name `withdraw`.
2. `withdraw(25)` creates frame `f2` with parent `f1` and `amount = 25`. Looking up `b` fails in `f2`, succeeds in `f1`. `b[0]` is `100`, and `25 > 100` is false, so we execute `b[0] = b[0] - amount`, i.e. element assignment writing `75` into box 0 of the list in `f1`. Return `75`.
3. `withdraw(25)` creates `f3`, also with parent `f1`, and finds `b[0]` is already `75`, because it is **the same list**. Writes `50`, returns `50`.
4. `withdraw(60)`: `60 > 50`, so return `'Insufficient funds'` without changing anything.
5. `withdraw(15)`: writes `35`, returns `35`.

The crucial subtlety: `withdraw` never reassigns any name in `f1`. Assignment statements like `b = ...` always bind in the local frame (Python's `nonlocal` and `global` statements can change that, but we are not using them). What `withdraw` changes is the **value of the object `b` refers to**. The name `b`, and the identity of the list, never change; only its contents do.

Also note that `withdraw` checks `b[0]`, not `balance`. The parameter `balance` still holds `100` forever, since numbers are immutable and nothing reassigns it.

### Example 7: environment diagram practice (Spring 2023 Midterm 2, Question 1)

```python
def chain(s):
    return [s[0], s[1:]]

silver = [2, chain([3, 4, 5])]
gold = [silver[0], silver[1].pop()]
silver[0] = 1
platinum = chain(chain([6, 7, 8]))
```

Reminder: `s.pop()` **removes and returns** the last item in list `s`.

Line by line:

1. `chain([3, 4, 5])` is called. Inside, `s[0]` is `3` and `s[1:]` is a **new** list `[4, 5]` (slicing creates a new list). It returns a new list `[3, [4, 5]]`. So `silver` is bound to `[2, [3, [4, 5]]]`. The expression `silver[1]` evaluates to the list `[3, [4, 5]]`.
2. `gold = [silver[0], silver[1].pop()]`. Evaluate left to right: `silver[0]` is `2`. Then `silver[1].pop()` mutates the list `[3, [4, 5]]` by removing and returning its last element, which is the list `[4, 5]`. So `gold` is `[2, [4, 5]]`, and as a **side effect** `silver` is now `[2, [3]]`. Note that `gold[1]` is the very same `[4, 5]` object that used to be inside `silver`.
3. `silver[0] = 1`. Element assignment on the outer list of `silver`, so `silver` is `[1, [3]]`. `gold[0]` is unaffected: it holds the number `2`, which was copied as a value binding, not an arrow into `silver`.
4. `platinum = chain(chain([6, 7, 8]))`. The inner call returns `[6, [7, 8]]`. The outer call receives that list, takes `s[0]` which is `6`, and `s[1:]` which is a new one-element list whose single element is **the same `[7, 8]` object**. It returns `[6, [[7, 8]]]`.

Final values:

```python
>>> silver
[1, [3]]
>>> gold
[2, [4, 5]]
>>> platinum
[6, [[7, 8]]]
```

What a drawn diagram would show: the global frame has `chain`, `silver`, `gold`, `platinum`. Three frames for the three `chain` calls, each with parent Global, each with a parameter `s` and a return value. Six list objects in all, with one `[4, 5]` shared between `gold` and nothing else (it was detached from `silver` by the `pop`), and one `[7, 8]` reachable only through `platinum[1][0]`.

### Example 8: identity vs. equality in sequence

```python
>>> a = [10]
>>> b = [10]
>>> a == b
True
>>> a is b
False
>>> b.append(20)
>>> a == b
False
>>> c = b
>>> c is b
True
>>> c.pop()
20
>>> b
[10]
>>> a
[10]
>>> a == b, a is b
(True, False)
```

At the end `a` and `b` are equal again, but they are still not the same list, so a future change to either one will not be reflected in the other.

---

## Common Pitfalls

1. **Thinking `s = s + t` and `s += t` are the same.** `+` builds a new list and rebinds the local name; `+=` is `extend` and mutates in place, so every alias changes. On an exam, if a function parameter is mutated with `+=`, the caller's list changes too.
2. **Assuming assignment copies.** `t = s` makes a second name for one list. To get a copy you need `list(s)`, `s[:]`, or a comprehension, and even then it is a **shallow** copy: nested lists are still shared.
3. **Forgetting that `pop` returns while `append` and `extend` return `None`.** `t = s.append(4)` binds `t` to `None`. Likewise `s = s.append(4)` destroys your list reference.
4. **Confusing `append` with `extend`.** `s.append([1, 2])` adds one element (a list); `s.extend([1, 2])` adds two elements.
5. **Confusing `remove` (by value, first occurrence only) with `pop` (by index, default last).**
6. **Confusing `list.pop(i)` (index) with `dict.pop(key)` (key).**
7. **Mutating a list while iterating over it.** The demo that looped `for s in result:` while doing `result.append(s)` shows how aliases pile up and the loop sees objects it just changed.
8. **Using `p.append(...)` or `p += [...]` to extend a path in a tree recursion.** Sibling branches share the list, so each branch contaminates the next. Use `p = p + [...]` to build a fresh list per call.
9. **Writing `n += 1` on an integer inside a nested helper to accumulate a count.** Assignment binds locally, so this raises an error (or silently creates a local). The lecture's solution is a one-element list `n = [0]` plus `n[0] += 1`, which is element assignment on a shared mutable object.
10. **Mutable default arguments.** `def f(s=[])` shares one list across all calls. Treat any mutable default as a bug unless you deliberately want shared state.
11. **Believing a tuple's contents can never appear to change.** A tuple holding a list looks different after the list is mutated, even though the tuple itself is unchanged.
12. **Using `is` where you mean `==`, or vice versa.** Two separately built lists with identical contents are `==` but not `is`.
13. **Assuming the parameter name still has the original value inside a closure.** In `make_withdraw_list`, `balance` stays `100` forever; only `b[0]` tracks the current balance.
14. **Forgetting the `f` on an f-string.** Without it the braces are just characters.
15. **Forgetting that f-string sub-expressions have side effects and are evaluated left to right.** `f'{s.pop()} {s.pop()} {s}'` mutates `s` twice before printing it.

---

## Likely Exam Points

### 1. Tracing aliasing through nested list mutation

**Q.** What does this print?

```python
a = [1, [2, 3]]
b = a[1]
a.append(b)
b[0] = 9
a[1] = 'x'
print(a, b)
```

**A.** `b` is an alias for the inner list `[2, 3]`. `a.append(b)` makes `a` three boxes long, with box 2 arrowing to that same inner list. `b[0] = 9` mutates the inner list to `[9, 3]`, visible through `a[1]` and `a[2]`. Then `a[1] = 'x'` overwrites box 1 of the **outer** list with a string; this does not touch the inner list, which is still reachable through `a[2]` and `b`. Output: `[1, 'x', [9, 3]] [9, 3]`.

### 2. `+` vs. `+=` inside a function

**Q.** What are the final values of `s` in each case?

```python
def g1(t):
    t = t + [4]

def g2(t):
    t += [4]

s1 = [1, 2, 3]; g1(s1)
s2 = [1, 2, 3]; g2(s2)
```

**A.** `s1` is `[1, 2, 3]`. `t + [4]` builds a new list and rebinds only the local name `t`, which disappears when the frame does. `s2` is `[1, 2, 3, 4]`, because `+=` on a list is `extend`, which mutates the object that `s2` also refers to.

### 3. `is` vs. `==`

**Q.** Give the value of each expression.

```python
a = [1, 2]
b = [1, 2]
c = a
```
`a == b`, `a is b`, `a is c`, `a == c`, and after `c.append(3)`, `a == b` and `a`.

**A.** `a == b` is `True` (equal contents). `a is b` is `False` (two separate objects). `a is c` is `True`. `a == c` is `True`. After `c.append(3)`, `a` is `[1, 2, 3]` because `c` is `a`, so `a == b` is now `False`.

### 4. Mutable default argument

**Q.** What does the third call return?

```python
def collect(x, acc=[]):
    acc.append(x)
    return acc

collect(1)
collect(2)
print(collect(3))
```

**A.** `[1, 2, 3]`. The default list is created once when the `def` runs and is shared across all calls that omit `acc`, so each call appends to the same list. (And all three calls returned that same object, so earlier return values also show `[1, 2, 3]`.)

### 5. Persistent local state with a mutable value

**Q.** Write `make_counter()` that returns a function of no arguments returning 1, then 2, then 3, and so on, without using `nonlocal`.

**A.**

```python
def make_counter():
    count = [0]
    def counter():
        count[0] += 1
        return count[0]
    return counter
```

The list `count` lives in the parent frame of `counter` and survives between calls. Element assignment mutates it, so no assignment to a non-local **name** is needed. Two separate calls to `make_counter()` produce two independent counters, because each call creates a new frame with a new list.

### 6. Environment diagram with `pop` and slicing (the `chain` style question)

**Q.** Using `def chain(s): return [s[0], s[1:]]`, what is `platinum` after `platinum = chain(chain([6, 7, 8]))`, and is `platinum[1][0]` the same object as the list created by the inner call's slice?

**A.** `platinum` is `[6, [[7, 8]]]`. The inner call returns `[6, [7, 8]]` where `[7, 8]` came from a slice. The outer call slices that two-element list from index 1, producing a **new** one-element list whose single element is an arrow to the **same** `[7, 8]` object. So yes, `platinum[1][0]` is that same list.

### 7. Why a tree recursion must not mutate its path

**Q.** In `print_big`, what goes wrong if `p = p + [t.label]` is replaced with `p.append(t.label)`?

**A.** All recursive calls would share one list. After descending into the first branch, the labels from that branch would still be in `p` when the second branch ran, so paths would be printed with extra labels from unrelated branches, and `sum(p)` would be wrong. `p = p + [t.label]` creates a fresh list per call and rebinds only the local `p`, keeping each path independent.

### 8. Accumulating with a one-element list (`bigs`)

**Q.** In `bigs`, why is `n = [0]` used instead of `n = 0`, and why is the helper called as `f(t, t.label - 1)`?

**A.** `n = [0]` gives a mutable object that the nested helper can update with `n[0] += 1`; writing `n += 1` with an integer would be an assignment to a local name inside `f` and would not update the outer `n`. Calling `f(t, t.label - 1)` makes the second argument (the largest ancestor label so far) smaller than the root's label, so the root is always counted, which is correct since the root has no ancestors.

### 9. Immutability test with two names

**Q.** Does this change `t`?

```python
s = (1, 2, 3)
t = (s, 4)
s = s + (5,)
print(t)
```

**A.** No: `(1, 2, 3)`, `4`, so `t` prints as `((1, 2, 3), 4)`. `s + (5,)` builds a new tuple and rebinds the name `s`; the tuple inside `t` is the original immutable object. If `s` had been a list and we had called `s.append(5)`, `t` would print as `([1, 2, 3, 5], 4)`.

### 10. f-string with side effects

**Q.** What does this evaluate to?

```python
s = [9, 8, 7]
f'because {s.pop()} {s.pop()} {s}'
```

**A.** `'because 7 8 [9]'`. Sub-expressions are evaluated in order: the first `pop` returns `7` (leaving `[9, 8]`), the second returns `8` (leaving `[9]`), then `{s}` inserts `str([9])`.

### 11. f-string uses `str`, not `repr`

**Q.** If an object `half` has `str(half) == '1/2'` and `repr(half) == 'Fraction(1, 2)'`, what is `f'{half}'`? How would you get the repr?

**A.** `'1/2'`, because f-strings insert the `str` of each value. Write `f'{repr(half)}'` to get `'Fraction(1, 2)'`.

---

## Summary

- **The big new idea:** an object can change its value over time. The same list can hold different contents as computation proceeds.
- **Only mutable types can change.** Lists and dictionaries are mutable; numbers, strings, and tuples are immutable.
- **All names that refer to an object are affected by a mutation of it.** Two names for one list are not two lists, as the card-suits story showed.
- **Know which operations create a new list** (literal, `list(s)`, comprehension, `+`, slicing) **and which mutate an existing one** (`append`, element assignment, `extend`, `+=`, `remove`, `pop`, slice assignment). `s += t` is `s.extend(t)`, not `s = s + t`.
- **Functions can mutate objects in their scope**, whether passed as arguments or found in an enclosing or global frame. Such a function is not pure; it has a side effect.
- **Immutability is a guarantee:** assign a name to an immutable value and it will still be that value no matter what functions run in between. Mutable values offer no such guarantee. The way to test whether something mutated is to use a **second name** and see whether it changed.
- **An immutable sequence can appear to change if it contains a mutable element.** The tuple is the same tuple holding the same objects; one of those objects changed.
- **An expression's value can change for two reasons:** a name was rebound (true since day one), or an object was mutated (new, and does not require any reassignment).
- **Identity vs. equality:** `is` tests whether two expressions evaluate to the same object; `==` tests whether they evaluate to equal values. Identical implies equal, not the reverse. Equal-but-not-identical lists diverge the moment one is mutated.
- **With mutation, a compound object has an identity beyond the sum of its parts.** A list is still the same list after its contents change.
- **Mutable default arguments are dangerous:** the default value is part of the function value, created once at `def` time, so mutations persist across calls.
- **Mutable values enable mutable functions and persistent local state.** `make_withdraw_list` stores the balance in a one-element list in `withdraw`'s parent frame; `withdraw` never rebinds a non-local name, it performs element assignment on the same list. This bundles behavior with changing data.
- **In recursion, build new lists rather than mutating shared ones** (`p = p + [t.label]`, `[k] + rest`), except when you deliberately want a shared accumulator (`n = [0]` with `n[0] += 1`).
- **f-strings** are the preferred string interpolation in Python: `{ }` holds an arbitrary expression evaluated in the current environment, the `str` of its value is inserted, and side effects happen left to right.
