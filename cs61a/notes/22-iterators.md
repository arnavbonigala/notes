<!-- Fri, Oct 16, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 22: Iterators

## Overview

This lecture introduces the **iterator** as a way to represent sequential data *implicitly*: instead of holding all of a sequence's elements in a container, an iterator bundles together a sequence together with a *position* in that sequence, and hands you one element at a time when you ask with `next`. The built-in `iter` turns an *iterable* (a list, tuple, string, range, dictionary, dictionary view, map object, and so on) into an iterator, and `next` advances it, raising `StopIteration` when there is nothing left. Because iterators are mutable objects that remember where they are, two iterators over the same container are independent, but two *names* for the same iterator share a position, and a `for` statement over an iterator consumes it. On top of this interface Python provides built-in functions that return iterators and compute **lazily** (`map`, `filter`, `zip`, `reversed`), meaning work is performed only when a value is actually requested, which matters both for efficiency and for short-circuiting functions like `all` and `any`. The lecture closes with the practical motivation for iterators (data abstraction, carrying a position around, guaranteeing each element is processed once, as illustrated by a shuffled Blackjack deck) plus list-mutation practice, since iterator questions on exams are usually mixed with shared-structure questions.

## Key Concepts

### 1. Containers, iterables, and iterators

A **container** can provide an iterator that gives access to its elements in order. There are two built-in functions that make up the whole interface:

- `iter(iterable)` returns a *new* iterator over the elements of an iterable value.
- `next(iterator)` returns the next element of the iterator, and as a side effect advances it.

Think of an iterator as **a position in some sequence**. It knows the contents of the underlying sequence and it has a marker for "what's next". It gives you access to the element at that position and everything after it, and nothing before it.

Critically, `next` only works on *iterators*, not on iterables in general:

```python
>>> s = [[1, 2], 3, 4, 5]
>>> next(s)
TypeError: 'list' object is not an iterator
```

A list is iterable but is not itself an iterator: it has no position. You must call `iter` first.

### 2. Iterators are mutable; calling `next` mutates them

All iterators are mutable objects. When you call `next` on one, the object itself changes to point at the following element. This leads to the single most-tested distinction in this lecture:

- **Two iterators over the same container are independent.** `t = iter(s)` and `u = iter(s)` are two separate markers into the same list. Advancing `t` does not move `u`.
- **Two names for the same iterator are not independent.** `w = u` binds `w` to the *same object*, so `next(w)` advances `u` too.

From the slides:

```python
>>> s = [3, 4, 5]
>>> t = iter(s)
>>> next(t)
3
>>> next(t)
4
>>> u = iter(s)      # a second, fresh marker into the same list
>>> next(u)
3
>>> next(t)          # t did not forget where it was
5
>>> w = u            # w and u are the same iterator object
>>> next(w)
4
```

In box-and-pointer terms: there is exactly one list object `[3, 4, 5]`. `t` and `u` point to two different iterator objects, each of which points at that same list and stores its own index. `w` and `u` point at *one* iterator object, so the index they share advances once per `next` call from either name.

### 3. Exhaustion and `StopIteration`

Once an iterator has passed the last element, asking for more raises a special exception, `StopIteration`. That is how you (and Python's `for` statement) detect the end:

```python
>>> t = iter([[1, 2], 3, 4, 5])
>>> next(t)
[1, 2]
>>> next(t)
3
>>> list(t)     # collects only what is LEFT
[4, 5]
>>> next(t)
StopIteration
```

Note that `list(t)` does not restart from the beginning. It drains the remaining elements, which also leaves `t` exhausted.

### 4. Viewing an iterator's contents

To see everything an iterator will produce, place the elements into a new container: `list(...)`, `tuple(...)`, or `sorted(...)` (which returns a sorted list of all elements). Doing so **consumes** the iterator, so a second `list(...)` call gives `[]`.

### 5. Iterating over dictionaries

A dictionary and its *views* are all iterable values:

- `iter(d)` and `iter(d.keys())` both iterate over the keys.
- `iter(d.values())` iterates over the values.
- `iter(d.items())` iterates over `(key, value)` tuples.

There is only one dictionary; the keys/values/items views are different *views* of that same dictionary, each of which you can get an iterator for.

**Order:** items appear in the order in which they were added (insertion order). This is true of recent Python versions (from Python 3.6 onward). In older versions dictionaries were "unordered collections of key-value pairs" and the order was arbitrary, so the lecture notes that in real code you should keep in mind which Python version your users run.

```python
>>> d = {'one': 1, 'two': 2, 'three': 3}
>>> d['zero'] = 0
>>> k = iter(d)        # same as iter(d.keys())
>>> next(k)
'one'
>>> next(k)
'two'
```

**Mutation during iteration:** if the dictionary **changes size** while you hold an iterator over it, that iterator becomes invalid and you get an error: `dictionary changed size during iteration`. You must build new iterators. But if you only **change the value** bound to an existing key, the structure (the set of keys) has not changed, so you may continue iterating.

### 6. `for` statements and iterators

A `for` statement works over iterable values, and it also works over iterators themselves. But there is a difference in reusability:

- Over an **iterable** such as a `range` or a list: each `for` statement starts from the beginning, every time, because the container itself has no position.
- Over an **iterator**: the `for` statement advances the marker all the way to the end. A second `for` over the same iterator prints nothing, because the iterator is exhausted.

```python
>>> r = range(3, 6)
>>> for i in r:    print(i)     # 3 4 5
>>> for i in r:    print(i)     # 3 4 5 again

>>> ri = iter(r)
>>> next(ri)                    # 3, marker advances
>>> for i in ri:   print(i)     # 4 5 only
>>> for i in ri:   print(i)     # nothing: ri is exhausted
```

The lecture code makes the equivalence explicit: a `for` statement over an iterator is just a `next` loop that catches `StopIteration`.

```python
def loop(t):
    for x in t:
        print(x)

def loop_iter(t):
    while True:
        try:
            print(next(t))
        except StopIteration:
            return
```

*(extra context)* The `for` statement actually calls `iter` on whatever it is given; for an iterator, `iter` returns that same iterator object, which is exactly why the marker is shared and the iterator gets consumed.

### 7. Lazy computation and the built-in iterator functions

Many built-in Python sequence operations return **iterators that compute results lazily**. Lazy computation means a result is computed only when it is requested: the interpreter gets ready to compute whatever is desired, but does not actually compute it until the value is going to be used.

| Function | Produces an iterator over |
|---|---|
| `map(func, iterable)` | `func(x)` for `x` in `iterable` |
| `filter(func, iterable)` | each `x` in `iterable` for which `func(x)` is a true value |
| `zip(first_iter, second_iter)` | co-indexed `(x, y)` pairs |
| `reversed(sequence)` | the elements of a sequence in reverse order |

The contrast with a list comprehension is the cleanest demonstration:

```python
def cube(k):
    print("cubing", k)
    return pow(k, 3)
```

```python
>>> x = [cube(k) for k in range(1, 6)]     >>> x = map(cube, range(1, 6))
cubing 1                                    (nothing is printed!)
cubing 2
cubing 3                                   >>> x
cubing 4                                   <map object at 0x10a002f20>
cubing 5
>>> x
[1, 8, 27, 64, 125]
```

The list comprehension is **eager**: it calls `cube` five times immediately and builds a list. `map` is **lazy**: it builds a map object (an iterator) and calls `cube` only as elements are demanded.

### 8. Laziness composes

Lazy iterators can be chained, and the chain still does only the work required:

```python
>>> m = map(double, range(3, 7))            # nothing doubled yet
>>> f = lambda y: y >= 10
>>> t = filter(f, m)                        # still nothing doubled
>>> next(t)
*** 3 => 6 ***      # 6 fails the filter
*** 4 => 8 ***      # 8 fails the filter
*** 5 => 10 ***     # 10 passes
10
>>> list(t)
*** 6 => 12 ***
[12]
```

Exactly as much work as needed to produce the next value, and no more. If you call `list(...)` on the whole chain from scratch, you force all the work (doubling 3, 4, 5, and 6) and get `[10, 12]`.

### 9. `all` and `any` short-circuit

```
all(s: iterable) -> bool   # True if all values in s are true values
                           # returns as soon as a false value is reached
any(s: iterable) -> bool   # True if at least one value in s is a true value
                           # returns as soon as a true value is reached
```

Combine short-circuiting with laziness and you get the lecture's puzzle:

```python
>>> x = all(map(print, range(-3, 3)))
-3
```

Why only `-3`?

1. `print(-3)` displays `-3` and returns `None`.
2. `None` is a false value.
3. `all` returns `False` as soon as it sees a false value, so `all([None, ...])` is `False` no matter what follows.
4. The map iterator therefore **never needs to advance past `-3`**, so `print` is never called on `-2`, `-1`, `0`, `1`, `2`.

Same idea with the noisier `double`:

```python
>>> all(map(double, range(-3, 3)))
*** -3 => -6 ***
*** -2 => -4 ***
*** -1 => -2 ***
*** 0 => 0 ***
False
```

Here `-6`, `-4`, `-2` are all true values (nonzero), so `all` keeps going; `double(0)` returns `0`, which is false, so `all` stops immediately and returns `False`. Elements `1` and `2` are never doubled.

### 10. `zip` details

- Returns an iterator over co-indexed tuples: `zip([1, 2], [3, 4])` yields `(1, 3)` then `(2, 4)`.
- If one input is longer, `zip` **only iterates over matches and skips the extras**: `zip([1, 2], [3, 4, 5])` yields `(1, 3), (2, 4)` and the `5` is lost.
- You can zip more than two iterables: `zip([1, 2], [3, 4, 5], [6, 7])` yields triples of co-indexed elements, here `(1, 3, 6)` and `(2, 4, 7)`.

### 11. Why use iterators at all?

Three reasons from the lecture:

1. **Data abstraction.** Code that only uses `next` (or a `for` statement, or occasionally `iter`) makes very few assumptions about the data representation. Switching from a list to a tuple, a map object, a `dict_keys` object, or some other format does not require rewriting that code, so other people can use your code on their data.
2. **A sequence plus a position in one object.** This is convenient to pass around, because the position travels with it. It is a way to ensure each element is processed **only once**: whoever calls `next` advances the iterator for everyone else, so a given value is returned by `next` exactly once.
3. **Restricting operations.** Handing out an iterator instead of a list limits what a function can do to the sequence: it can only request the next value, not change the sequence.

The Blackjack demo is the payoff for reason 2 and 3: `shuffle` builds a list of all 52 cards, calls `random.shuffle`, and returns `iter(deck)`. Everyone (player, dealer) draws by calling `next(deck)`, which guarantees **no repeated cards**: you keep drawing off the top until you run out. Dealing the initial cards is just four `next(deck)` calls, in order (player card, dealer up card, player card, dealer hole card), and the player's strategy function sees only the dealer's up card, not the hole card.

## Definitions

- **Iterable value:** any value that can be passed to the built-in `iter` to produce an iterator. Lists, tuples, strings, ranges, dictionaries, dictionary views (`keys`, `values`, `items`), and iterators themselves are all iterable.
- **Iterator:** a mutable object returned from `iter` that can be passed to the built-in `next`. It represents a position within a sequence and provides access to the element at that position and everything after it.
- **`iter(iterable)`:** built-in function returning a new iterator over the elements of an iterable value.
- **`next(iterator)`:** built-in function returning the next element in an iterator and advancing the iterator's position.
- **`StopIteration`:** the exception raised by `next` when an iterator has no more elements. It is how code detects the end of an iterator.
- **Container:** a value that holds elements and can provide an iterator over them in some order.
- **Lazy computation:** a computation whose result is only performed when it is requested. Built-in functions like `map`, `filter`, `zip`, and `reversed` return iterators that compute lazily.
- **`map(func, iterable)`:** returns an iterator over `func(x)` for each `x` in `iterable`.
- **`filter(func, iterable)`:** returns an iterator over the `x` in `iterable` for which `func(x)` is a true value.
- **`zip(first_iter, second_iter, ...)`:** returns an iterator over co-indexed tuples, stopping when the shortest input is exhausted.
- **`reversed(sequence)`:** returns an iterator over the elements of a sequence in reverse order.
- **`all(s)`:** returns `True` if every value in iterable `s` is a true value; returns `False` as soon as a false value is reached.
- **`any(s)`:** returns `True` if at least one value in `s` is a true value; returns `True` as soon as a true value is reached.
- **Dictionary view:** an iterable object (`d.keys()`, `d.values()`, `d.items()`) providing access to part of a dictionary's contents in insertion order.
- **Map object / filter object / range iterator / list reverse iterator:** the concrete iterator types returned by `map`, `filter`, `iter(range(...))`, and `reversed(list)`. They display as e.g. `<map object at 0x...>`, not as lists.

## Worked Examples

### Example 1: The iterator demo (independent markers, `list`, `StopIteration`)

```python
>>> s = [[1, 2], 3, 4, 5]
>>> next(s)
TypeError: 'list' object is not an iterator
>>> t = iter(s)
>>> next(t)
[1, 2]
>>> next(t)
3
>>> u = iter(s)
>>> next(u)
[1, 2]
>>> list(t)
[4, 5]
>>> next(t)
StopIteration
```

Step by step:

1. `next(s)` fails: `s` is a list, which is iterable but has no position, so it is not an iterator.
2. `t = iter(s)` creates an iterator object whose marker sits before index 0 of `s`.
3. First `next(t)` returns the element at index 0, which is the *sub-list* `[1, 2]` (the actual list object inside `s`, not a copy), and advances the marker.
4. Second `next(t)` returns `3`.
5. `u = iter(s)` creates a **second, independent** iterator starting at the beginning. `next(u)` therefore returns `[1, 2]` again; `t` is unaffected.
6. `list(t)` drains the rest of `t`, which is `[4, 5]`. Note it does not include `[1, 2]` or `3`, which were already consumed.
7. `next(t)` now raises `StopIteration`, since `list(t)` ran `t` to exhaustion.

### Example 2: Discussion question (aliasing an iterator)

```python
a = [1, 2, 3]
b = [a, 4]
c = iter(a)
d = c
print(next(c))
print(next(d))
print(b)
```

Output:

```
1
2
[[1, 2, 3], 4]
```

Reasoning in box-and-pointer terms:

- There is one list object `[1, 2, 3]`, bound to `a`. `b` is a new list whose first element is a *pointer to that same list object*, so `b` displays as `[[1, 2, 3], 4]`.
- `c = iter(a)` makes one iterator object over `a`, with marker at index 0.
- `d = c` does **not** create a new iterator. `c` and `d` are two names for the same object, so they share one marker.
- `next(c)` returns `1` and advances the shared marker to index 1.
- `next(d)` therefore returns `2`, not `1`.
- `print(b)` shows `[[1, 2, 3], 4]`: **iterating does not mutate the underlying list.** Advancing an iterator changes the iterator, never the container. Nothing was removed from `a`.

The common wrong answer is `[[3], 4]` or `[[2, 3], 4]`, from imagining `next` "pops" elements. It does not.

### Example 3: Re-implementing `summation` three ways

```python
def cube(k):
    return pow(k, 3)

def summation(n, term):
    """Sum the first n terms of a sequence.

    >>> summation(5, cube)
    225
    """
    total, k = 0, 1
    while k <= n:
        total = total + term(k)
        k = k + 1
    return total
```

The sum for `summation(5, cube)` is `1 + 8 + 27 + 64 + 125 = 225`.

With a list comprehension:

```python
def summation_list_comprehension(n, term):
    return sum([term(k) for k in range(1, n + 1)])
```

With `map`:

```python
def summation_map(n, term):
    return sum(map(term, range(1, n + 1)))
```

Both replace the explicit accumulation loop. Points worth internalizing:

- `range(1, n + 1)` gives `1` through `n` inclusive, matching the `while k <= n` loop.
- The list comprehension builds an actual list of `n` values in memory before `sum` sees any of them.
- `map(term, ...)` builds no list: `sum` pulls values one at a time from the map iterator, calling `term` once per value on demand. Same answer, but no intermediate list.
- `sum` itself is **eager** in the sense that it must consume the whole iterator to produce a number, so with `cube` printing, both versions ultimately print all five lines. The difference is *when* and *whether a list exists*, not the final answer.

### Example 4: `map` is lazy (the `double` demo)

```python
def double(x):
    print('***', x, '=>', 2*x, '***')
    return 2*x
```

```python
>>> bcd = ['b', 'c', 'd']
>>> [x.upper() for x in bcd]
['B', 'C', 'D']
>>> caps = map(lambda x: x.upper(), bcd)
>>> next(caps)
'B'
>>> next(caps)
'C'
>>> s = range(3, 7)
>>> doubled = map(double, s)
>>> next(doubled)
*** 3 => 6 ***
6
>>> next(doubled)
*** 4 => 8 ***
8
>>> list(doubled)
*** 5 => 10 ***
*** 6 => 12 ***
[10, 12]
```

Step by step:

1. The list comprehension produces a real list `['B', 'C', 'D']` immediately.
2. `map(lambda x: x.upper(), bcd)` produces a map object. Nothing is uppercased yet. Each `next` call applies the lambda to exactly one element.
3. `doubled = map(double, s)` prints **nothing**: `double` has not been called even once.
4. The first `next(doubled)` calls `double(3)` at that moment, printing one line and returning `6`.
5. The second `next(doubled)` calls `double(4)`.
6. `list(doubled)` drains the rest, so `double(5)` and `double(6)` run now, in that order, and the resulting list `[10, 12]` contains only the remaining values. `6` and `8` are gone: they were already consumed.

### Example 5: The `average` trap (consuming an iterator twice)

```python
def average(s):
    """Return the average of values in a list.

    >>> average([3, 4, 5, 6])
    4.5
    >>> average(map(lambda x: x * x, [3, 4, 5, 6]))
    ZeroDivisionError: division by zero
    """
    return sum(s) / len(list(s))
```

- With a **list** argument, `sum(s)` is `18` and `len(list(s))` is `4`, giving `4.5`. A list can be traversed as many times as you like.
- With a **map object**, Python evaluates the operands left to right: `sum(s)` drains the iterator (computing `9 + 16 + 25 + 36 = 86`). By the time `list(s)` is evaluated, `s` is exhausted, so `list(s)` is `[]` and `len(list(s))` is `0`. Dividing by zero raises `ZeroDivisionError`.

The lesson: a function that traverses its argument more than once cannot accept an arbitrary iterator. Either require a sequence, or convert once up front (`s = list(s)`).

### Example 6: `all` with a list comprehension vs. laziness (`is_leafy`)

```python
from tree import *

def is_leafy(t) -> bool:
    "Return true if all of t's branches are leaves."
    return all([is_leaf(b) for b in branches(t)])
```

This returns `True` when every branch of `t` is a leaf (and vacuously `True` for a leaf `t`, since `branches(t)` is empty and `all([])` is `True`).

Note the interaction with short-circuiting: because the argument is a **list comprehension**, `is_leaf` is called on *every* branch before `all` runs at all. The list is built eagerly; `all`'s early exit saves nothing here. *(extra context)* Dropping the brackets, `all(is_leaf(b) for b in branches(t))`, would make the argument lazy so that `all` could stop at the first non-leaf branch. The bracketed version is still correct, just not short-circuiting.

### Example 7: Palindrome (why `reversed` needs care)

Goal: return whether `s` reads the same forward and backward, for any kind of sequence (lists, strings, and so on). `[3, 1, 4, 1, 3]` is a palindrome; `[3, 1, 4, 1, 5]` is not.

The tempting but **wrong** version:

```python
def palindrome(s):
    return s == reversed(s)      # always False!
```

Why it fails: `reversed(s)` is a `list_reverseiterator` object, not a sequence. Comparing a list to an iterator with `==` compares object identity/type, so the result is `False` for every input. Never compare a container to an iterator.

```python
>>> t = [1, 2, 3]
>>> reversed(t)
<list_reverseiterator object at 0x...>
>>> list(reversed(t))
[3, 2, 1]
```

Also be careful: `list(r)` twice on the same reverse iterator gives `[3, 2, 1]` then `[]`, because the first call used it up.

Fix 1, compare list to list:

```python
def palindrome(s):
    return list(s) == list(reversed(s))
```

Fix 2, zip the sequence with its own reverse and check that every co-indexed pair is equal:

```python
def palindrome(s):
    return all([a == b for a, b in zip(s, reversed(s))])
```

For `[1, 2, 3]`, `zip(s, reversed(s))` yields `(1, 3), (2, 2), (3, 1)`. For a palindrome every pair would have equal components. The lecture's take: both are fine, but the `zip` version generalizes better, since you can swap `a == b` for a looser comparison (approximate equality, case-insensitive matching, and so on).

### Example 8: List practice (shared structure)

```python
s = [2, [3, 4], 5]
s.append([6, 7])
s.extend([8, 9])
t = []
t.extend(s)
s[3].append(s[1].pop())
print(t)
```

Step by step:

1. `s` is `[2, [3, 4], 5]`, where element 1 is a pointer to a nested list object.
2. `s.append([6, 7])` adds **one** element, the list `[6, 7]`: `s` is `[2, [3, 4], 5, [6, 7]]`.
3. `s.extend([8, 9])` adds the *elements* of `[8, 9]`: `s` is `[2, [3, 4], 5, [6, 7], 8, 9]`.
4. `t.extend(s)` makes `t` a new list with the **same six elements**, which means `t[1]` and `s[1]` point to the same nested list, and `t[3]` and `s[3]` point to the same `[6, 7]`. This is a shallow copy.
5. `s[1].pop()` removes and returns `4` from the nested list, so that list is now `[3]`. `s[3].append(4)` makes `[6, 7]` into `[6, 7, 4]`.
6. Because `t` shares those same nested list objects, the mutations are visible through `t`.

Output: `[2, [3], 5, [6, 7, 4], 8, 9]`

### Example 9: Spring 2023 Midterm 2, Question 1

```python
def chain(s):
    return [s[0], s[1:]]

silver = [2, chain([3, 4, 5])]
gold = [silver[0], silver[1].pop()]
silver[0] = 1
platinum = chain(chain([6, 7, 8]))
```

Reminder: `s.pop()` removes and returns the last item of list `s`.

Step by step:

1. `chain([3, 4, 5])`: inside the frame for `chain`, `s[0]` is `3` and `s[1:]` is a **new** list `[4, 5]` (slicing always builds a new list). It returns `[3, [4, 5]]`. So `silver` is `[2, [3, [4, 5]]]`.
2. `gold = [silver[0], silver[1].pop()]`, evaluated left to right:
   - `silver[0]` is the integer `2`.
   - `silver[1]` is the list `[3, [4, 5]]`; `.pop()` removes its last element and returns it, which is the list `[4, 5]`. As a side effect `silver[1]` becomes `[3]`.
   - So `gold` is `[2, [4, 5]]`, where `gold[1]` is the very same list object that used to be inside `silver`.
3. `silver[0] = 1` changes `silver` to `[1, [3]]`. `gold[0]` is unaffected: `gold[0]` holds the integer value `2` that was copied in at step 2, not a pointer into `silver`.
4. `platinum = chain(chain([6, 7, 8]))`:
   - Inner call: `chain([6, 7, 8])` returns `[6, [7, 8]]`.
   - Outer call with `s = [6, [7, 8]]`: `s[0]` is `6`, and `s[1:]` is a new one-element list containing `[7, 8]`, that is `[[7, 8]]`.
   - Result: `[6, [[7, 8]]]`.

```python
>>> silver
[1, [3]]
>>> gold
[2, [4, 5]]
>>> platinum
[6, [[7, 8]]]
```

The two takeaways: `pop` mutates the list it is called on while handing the removed object to a new place (so structure is *shared*, not copied), and slicing creates a new list, so `s[1:]` in `chain` never aliases the argument's spine.

## Common Pitfalls

1. **Calling `next` on a list, string, or range.** Those are iterables, not iterators: `TypeError: 'list' object is not an iterator`. Call `iter` first.
2. **Thinking `d = c` makes a second iterator.** Assignment copies a reference, not the iterator. Only `iter(...)` makes a fresh marker.
3. **Thinking `next` mutates the underlying container.** Iterating never removes elements from the list or dictionary. Only the iterator changes.
4. **Forgetting that `list(it)` returns only what remains**, and exhausts `it` in the process. Calling `list(it)` twice yields the full remainder, then `[]`.
5. **Using an iterator in two `for` statements.** The first loop drains it; the second loop body never runs. (Iterating a *container* twice is fine.)
6. **Traversing an iterator argument more than once in a function**, as in `sum(s) / len(list(s))`. See the `average` `ZeroDivisionError`.
7. **Comparing a container to an iterator with `==`.** `s == reversed(s)` is always `False`, and `[1, 2, 3] == iter([1, 2, 3])` is `False` too. Convert both sides with `list`.
8. **Expecting `map`/`filter`/`zip`/`reversed` to print a list in the REPL.** They display as `<map object at 0x...>` and compute lazily. Nothing has been computed yet at the moment they are created.
9. **Expecting side effects (prints) to happen when `map` is called.** They happen when values are requested, and may never happen for elements that are never requested (as with `all(map(print, range(-3, 3)))`).
10. **Forgetting that `None`, `0`, `''`, `[]`, and `{}` are false values** when reasoning about `all`/`any`. `all(map(print, ...))` hinges on `print` returning `None`.
11. **Mutating a dictionary's size while iterating over it** (or over its keys/values/items views): `dictionary changed size during iteration`. Reassigning an existing key's value is safe.
12. **Assuming `zip` pads.** It stops at the shortest input and silently drops extras.
13. **Relying on dictionary ordering in older Pythons.** Insertion order is guaranteed only from Python 3.6 onward.

## Likely Exam Points

### 1. Trace `iter`/`next` with aliased vs. independent iterators

**Practice.** What does this print?

```python
s = [4, 5, 6]
a = iter(s)
b = iter(s)
c = b
print(next(a), next(b), next(c), next(a))
print(s)
```

**Answer.** `4 4 5 5`, then `[4, 5, 6]`.
`a` and `b` are independent iterators over the same list, so each starts at `4`. `c` is the same object as `b`, so `next(c)` continues where `b` left off and gives `5`. Then `next(a)` gives `a`'s second element, `5`. The list itself is never modified, so `s` is unchanged.

### 2. "What is printed?" with lazy `map` and a printing function

**Practice.** Given

```python
def f(x):
    print('f', x)
    return x > 2
```

what is displayed by the following?

```python
>>> m = map(f, [1, 2, 3, 4])
>>> next(m)
>>> any(m)
```

**Answer.**
`m = map(f, ...)` prints nothing.
`next(m)` prints `f 1` and the REPL displays `False`.
`any(m)` resumes where `m` left off: it prints `f 2` (returns `False`, keep going), then `f 3` (returns `True`), and `any` short-circuits immediately, returning `True`. `f 4` is never printed. So the display is:

```
f 1
False
f 2
f 3
True
```

### 3. `all`/`any` short-circuiting with side effects

**Practice.** What is printed by `x = any(map(print, range(3)))`, and what is `x`?

**Answer.** It prints `0`, `1`, `2` (all three lines), and `x` is `False`. `print` returns `None` every time, which is false, so `any` never finds a true value and must consume the entire iterator. Contrast with `all(map(print, ...))`, which stops after the very first element.

### 4. Filling in a blank with `map`, `filter`, `zip`, `sum`, `all`, `any`

**Practice.** Fill in the blank so that `count_evens` returns the number of even values in iterable `s`, using `filter` and no `for`/`while` statements.

```python
def count_evens(s):
    return len(list(_______________________________))
```

**Answer.** `filter(lambda x: x % 2 == 0, s)`
`filter` returns a lazy iterator, and `list` forces it so `len` can measure it. (`sum(map(lambda x: x % 2 == 0, s))` would also work, since `True` counts as `1`.)

### 5. Why an iterator-consuming function breaks

**Practice.** The function below is supposed to return `True` if `s` contains at least two values greater than `10`. It works for lists but fails for `map` objects. Why, and how would you fix it?

```python
def two_big(s):
    return sum([1 for x in s if x > 10]) >= 2
```

**Answer.** As written, this one actually works for a map object, since `s` is traversed only once. The failure mode appears whenever a function traverses `s` twice, for instance `return len([x for x in s if x > 10]) >= 2 and len(list(s)) > 0`. The general rule: an iterator can be consumed only once, so a function that needs multiple passes must first do `s = list(s)` (or require a sequence). The exam version of this question is usually the lecture's `average`: `sum(s) / len(list(s))` raises `ZeroDivisionError` on an iterator because `sum` exhausts `s` before `list(s)` runs.

### 6. `for` over an iterator consumes it

**Practice.** What does this print?

```python
r = iter(range(4))
for i in r:
    if i == 1:
        break
print(list(r))
```

**Answer.** `[2, 3]`. The loop consumed `0` and `1` (the `break` happens after `1` was already produced), so the remainder of the iterator is `2` and `3`.

### 7. Dictionary iteration and mutation

**Practice.** Which of these raise an error?

```python
d = {'a': 1, 'b': 2}
k = iter(d)
next(k)
d['a'] = 100      # (i)
next(k)
d['c'] = 3        # (ii)
next(k)
```

**Answer.** Line (i) is fine: changing the value for an existing key does not change the dictionary's size or structure, so `k` remains valid and the following `next(k)` returns `'b'`. Line (ii) adds a key, changing the size, so the `next(k)` after it raises `RuntimeError: dictionary changed size during iteration`. You would have to build a new iterator.

### 8. Reverse/zip-based palindrome style questions

**Practice.** Why does `def pal(s): return s == reversed(s)` always return `False`, and give a one-line correct version.

**Answer.** `reversed(s)` is an iterator object, and a list (or string) is never `==` to an iterator object, so the comparison is always `False`. Correct: `return list(s) == list(reversed(s))`, or `return all([a == b for a, b in zip(s, reversed(s))])`.

### 9. Mixed list mutation / box-and-pointer (always on the exam alongside iterators)

**Practice.** What are `x` and `y` after the following?

```python
x = [1, [2, 3]]
y = [x[0], x[1]]
x[0] = 9
x[1].append(4)
```

**Answer.** `x` is `[9, [2, 3, 4]]` and `y` is `[1, [2, 3, 4]]`.
`y[0]` copied the *value* `1`, so rebinding `x[0]` does not affect `y`. But `y[1]` and `x[1]` point to the same nested list object, so appending through `x` is visible through `y`.

### 10. Conceptual: why pass an iterator?

**Practice.** Give two reasons the Blackjack demo represents the deck as `iter(shuffled_cards)` rather than as a list.

**Answer.** (a) It bundles the sequence with the current position, so every function that draws a card advances the shared position and no card can be dealt twice. (b) It restricts the operations available to code that receives the deck: that code can only request the next card, not inspect, reorder, or modify the deck. (A third reason from lecture: code written against the iterator interface works regardless of how the underlying data is represented.)

## Summary

- An **iterator** represents sequential data implicitly: it is a sequence plus a position. `iter(iterable)` makes one; `next(iterator)` returns the next element and advances the position.
- `next` works only on iterators. Calling `next` on a list raises `TypeError: 'list' object is not an iterator`.
- Iterators are **mutable**. Two `iter` calls on one container give independent markers; plain assignment (`d = c`) gives two names for one marker.
- Iterating never mutates the underlying container.
- When an iterator is exhausted, `next` raises **`StopIteration`**. A `for` statement is exactly a `next` loop that catches it.
- `list`, `tuple`, and `sorted` collect the **remaining** elements of an iterator and exhaust it. A second collection call yields nothing.
- A `for` over a container can be repeated; a `for` over an iterator consumes it so a second loop sees nothing.
- Dictionaries, plus their `keys()`, `values()`, and `items()` views, are all iterable, in **insertion order** (Python 3.6+). `iter(d)` iterates keys. Changing a dictionary's **size** during iteration invalidates its iterators; changing a value does not.
- `map`, `filter`, `zip`, and `reversed` return iterators that compute **lazily**: no work and no side effects until a value is requested. A list comprehension is eager by contrast.
- `all` and `any` **short-circuit**, which combined with laziness means elements past the deciding one are never computed. Hence `all(map(print, range(-3, 3)))` prints only `-3` (because `print` returns the false value `None`).
- Never compare a container to an iterator with `==`; convert with `list` first.
- A function that traverses its argument twice (like `sum(s) / len(list(s))`) is broken for iterator inputs.
- `zip` truncates to the shortest input and accepts more than two iterables.
- Use iterators for data abstraction (code depends only on `iter`/`next`), to carry a position along with a sequence (each element processed exactly once, as with a shuffled Blackjack deck), and to restrict what callers can do to a sequence.
- `summation` can be rewritten as `sum([term(k) for k in range(1, n + 1)])` or `sum(map(term, range(1, n + 1)))`.
