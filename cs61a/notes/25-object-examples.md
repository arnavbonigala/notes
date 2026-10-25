<!-- Fri, Oct 23, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 25: Object Examples

> The slide deck for this lecture is titled "Data Examples" and it is a practice/review lecture rather than a new-material lecture. It walks through examples of the three data abstractions you have been building with objects: linked lists built with the `Link` dataclass, Python lists under mutation (with environment diagrams, including two real midterm questions), and classes with class attributes. The YouTube video for the day also includes a review segment on list mutation in environment diagrams and a set of iterable/iterator exercises solved with comprehensions and built-in functions.

## Overview

This lecture is pure practice on object-based data. It has three movements. First, linked lists: how to *build* a linked list with recursion (the recursive call's return value becomes the `rest`), shown through `squares`, `low_squares`, and `slice_link`, which reimplements `s[i:j]` for linked lists because normal slice notation does not work on them. Second, list mutation: a careful review of which list operations mutate in place (`append`, `extend`, `insert`, `remove`, `pop`, item assignment, slice assignment, `+=`) versus which create new lists (`list(t)`, `t[:]`, `t + []`, `+`, slicing), and how to draw that in an environment diagram where arrows always point at *values*, never at names. That section includes the nested-list diagram `t[0].append(t[1:2])`, the classic `prefixes` aliasing bug, a recursive-with-mutation `fibs_rec`, and two past midterm environment-diagram problems (Fall 2022 MT2 Q2, Spring 2023 MT2 Q1). Third, classes: the `Transaction` example, which shows that a class attribute is one single value shared by the class and all of its instances, so `self.log.append(self)` mutates a shared list while `self.prior = list(self.log)` takes a private snapshot copy.

## Key Concepts

### 1. Building linked lists recursively: the recursive result is the `rest`

The data definition used all semester:

```python
type LinkedList[T] = Link[T] | tuple[()]

@dataclass
class Link[T]:
    first: T
    rest: LinkedList[T] = ()   # rest defaults to the empty linked list
```

A linked list is either a `Link` (which has a `first` value and a `rest` that is itself a linked list) or the empty tuple `()`, which stands for the empty linked list.

Because the `rest` of a `Link` is *the same kind of thing* as the whole structure, the natural way to build one is recursion. The slide states the shape directly:

```python
def some_function(...) -> LinkedList:
    ... Link(..., some_function(...)) ...
```

That is, you construct one `Link` whose `first` is the value you want at this position and whose `rest` is whatever the recursive call returns. The base case returns `()`. This is the mirror image of *processing* a linked list recursively (where you call the function on `s.rest`): when building, the call gives you the tail; when processing, the tail gives you the call.

### 2. Helper functions carry the state the recursion needs

Neither `squares` nor `low_squares` recurses on its own parameter. Both define an inner helper whose parameters are exactly the pieces of state the recursion has to track:

- `squares(n)` needs only "which number am I squaring next", so `f(i)`.
- `low_squares(n)` needs two things, which the slide labels explicitly: `i` = *what to square*, and `k` = *how many found so far*. The stopping condition is about `k` (we found enough), not about `i` (which keeps climbing until enough qualify).

This is a general pattern: when the stopping condition and the stepping variable are different quantities, you need a second parameter. Note also the third branch in `low_squares`: when a candidate fails the test, you return the recursive call *directly* without wrapping it in a `Link`. Returning the recursive result unchanged is how you skip an element.

### 3. Slicing a linked list: two independent counters

`s[i:j]` does not work on a linked list, so `slice_link(s, i, j)` reimplements it. The insight is that `i` and `j` have different jobs:

- `i` counts how many elements still need to be *skipped* before output starts.
- `j` counts how many elements remain *until the end* of the requested range.

So walking one element forward down the list decrements both (`i-1, j-1`), while *emitting* an element decrements only the remaining-length counter (`j-1`) because the skipping is already done (`i` is already 0). The two base cases are `j == 0` (the range is used up) and `s == ()` (the list ran out, which is why `slice_link(evens, 1, 100)` does not crash and just returns `(2 6)`).

The slide reconstructs the recursion for `slice_link(evens, 1, 2)` where `evens = Link(4, Link(2, Link(6)))`:

- `slice_link(evens, 1, 2)` returns whatever `slice_link(evens.rest, 0, 1)` returns (skip the 4),
- `slice_link(evens.rest, 0, 1)` links `2` to the result of `slice_link(evens.rest.rest, 0, 0)`,
- `slice_link(evens.rest.rest, 0, 0)` returns `()` because `j == 0`.

Result: a *new* `Link` containing 2 whose rest is `()`. The 6 is never copied, and the original `evens` is untouched: `slice_link` builds new `Link` objects rather than mutating.

### 4. Which list operations mutate, and which build new lists

This is the heart of the review segment. Mutating operations change an existing list object in place, so every name and every container that refers to that object sees the change:

| Mutates in place | Creates a new list |
| --- | --- |
| `s.append(x)` (adds one element) | `list(s)` |
| `s.extend(t)` (adds all elements of `t`) | `s[:]` (or any slice) |
| `s.insert(i, x)` | `s + t` |
| `s[i] = x` | `s + []` |
| `s[i:j] = t` (slice assignment) | `[x for x in s]` |
| `s.pop()` / `s.pop(i)` (removes and returns) | |
| `s.remove(x)` (removes first match) | |
| `s += t` | |

Two details from the demo that get tested constantly:

- `append`, `extend`, `insert`, and `remove` return `None`. The slide comment says it outright: `print(suits.append('x'))` prints `None`. Only `pop` returns something useful.
- `s += t` **mutates** the list `s` refers to (it is equivalent to `s.extend(t)`), while `s = s + t` creates a brand new list and rebinds `s`. If another name `t` is aliased to the same list, `+=` changes what `t` sees and `s = s + ...` does not.

### 5. Environment diagram discipline for lists

Three rules from the review:

1. **Arrows point to values, not names.** After `s.append(t)`, the new last element of `s` points at the list `[5, 6]`, *not* at the name `t`. The transcript is emphatic here: evaluating `t` produces a value, and it is that value that goes into the list. Consequently, rebinding `t = 0` afterward does not touch `s` at all.
2. **Lists that are built during evaluation but never named can be erased** (they are unreachable), for example the one-element list `[t]` in `a = s + [t]`.
3. **Mutation reaches through every path.** If the same inner list is reachable from `t`, from `a`, and from `b`, then `b[1][1] = 0` shows up as a change in all three. In the diagram you literally see one box change and three readings change.

### 6. Shallow copying

`list(t)`, `t[:]`, and `t + []` all make *a copy of `t`*, but only one level deep. For `t = [[1, 2], [3, 4]]`, the copy is a new outer list whose two elements are arrows to **the same two inner lists**. So mutating `t[0]` is visible through the copy. This is why the `prefixes` fix uses `list(so_far)` (a fresh outer list is exactly what is needed there, since the elements are numbers).

### 7. Mutation as an accumulation strategy, even inside recursion

`fibs_rec` shows that recursion and mutation compose: create one list in the outer function, have the recursive inner function `append` to it, and return the list at the end. The recursive function returns nothing meaningful; its purpose is its *effect* on the shared list. Because each `append` happens before the recursive call, `s[-2] + s[-1]` always reads the two most recently added values.

### 8. Class attributes are shared; instance attributes are per-instance

In `Transaction`, `log = []` sits in the class body, so there is exactly **one** list, no matter how many instances exist. The slide's wording: "A class attribute can be accessed from either an instance or its class. There is only one value for a class attribute, regardless of how many instances."

Critically:

- `self.log.append(self)` **looks up** `log` (not found on the instance, so found on the class) and then **mutates** that shared list. Every instance sees the growth.
- `self.prior = list(self.log)` **assigns** to `self.prior`, which creates an attribute on *this instance only*, holding a snapshot copy of the log as of construction time. The slide notes the equivalent spellings `list(type(self).log)` and `list(Transaction.log)`.
- Each instance therefore stores a list of the `Transaction` objects that existed before it. The diagram shows three instances each with `amount` and `prior`, and one class-level `log` list pointing at all of them.

The slide also annotates `self` as "always bound to some `Transaction` instance", which is why `self.amount` and `self.prior` are the per-instance data while `self.log` resolves upward to the class.

### 9. Doing data manipulation with comprehensions and built-ins (video segment)

The second half of the video is a set of four "do it in one or two lines" problems. The reusable techniques:

- **`min(map(abs, s))` is not `min(s, key=abs)`.** The first gives you the smallest absolute *value*; the second gives you the *element* whose absolute value is smallest. Pick based on what you need next.
- **To get indices, iterate over `range(len(s))`, not over `s`.** Then index with `s[i]` inside the comprehension.
- **A list comprehension can always be rewritten with `map`/`filter`**, but `filter` returns an iterator, so wrap it in `list(...)` if you need a list.
- **Adjacent pairs**: either index with `range(len(s) - 1)` and use `s[i] + s[i+1]`, or `zip(s[:-1], s[1:])` and unpack with `for a, b in ...`. The last index is deliberately excluded because it has no `i + 1`.
- **Dictionary comprehensions take a filter clause too**: `{key: value for d in range(10) if <keep d>}`. Use `any([...])` (or a precomputed list of last digits) to drop keys with no matching elements.
- **Aggregation functions** (`max`, `min`, `sum`, `any`, `all`, `len`, `s.count`) turn a comprehension of per-element answers into a single answer. "All counts above one" can be written as `all([... > 1 ...])` or `min([...]) > 1`.

## Definitions

- **Linked list**: a sequence represented as either the empty linked list `()` or a `Link` object holding a first value and a rest that is itself a linked list.
- **`Link`**: the dataclass with fields `first` (the element at this position) and `rest` (the rest of the linked list, defaulting to `()`).
- **Base case (when building)**: the branch of a recursive constructor that returns `()` instead of building another `Link`.
- **Mutation**: a change to the contents, length, or structure of an existing object, as opposed to rebinding a name to a different object.
- **Aliasing**: two or more names (or list elements) referring to the same object, so that mutation through one is visible through all.
- **Shallow copy**: a new list whose elements are the same objects as the original's elements; produced by `list(t)`, `t[:]`, or `t + []`.
- **Slice assignment** (`s[i:j] = t`): replaces the elements of `s` from `i` up to `j` with all elements of `t`, growing or shrinking `s` as needed (the replaced slice and the replacement need not have the same length).
- **`pop`**: removes and returns the last element (`s.pop()`) or the element at an index (`s.pop(i)`).
- **`remove`**: removes the first element equal to the argument and returns `None`.
- **`append` vs `extend`**: `append` adds its argument as a single element; `extend` adds each element of its argument.
- **Class attribute**: an attribute bound in the class body (or on the class object), of which there is exactly one value shared by the class and all instances, accessible as `Class.attr` or `instance.attr`.
- **Instance attribute**: an attribute bound on a particular object (typically via `self.name = ...`), with a separate value per instance.
- **Attribute lookup on an instance**: Python first looks for an instance attribute; if there is none, it looks on the class. (Assignment through an instance always creates/overwrites an *instance* attribute.)
- **Environment diagram**: a drawing of frames, bindings, and objects used to track which names refer to which values and how objects change over time.

## Worked Examples

### A. `squares`: build a linked list of `n` perfect squares

```python
def squares(n) -> LinkedList[int]:
    """A linked list of n positive perfect squares.

    >>> print(squares(10))
    (1 4 9 16 25 36 49 64 81 100)
    """
    def f(i):
        if i > n:
            return ()
        else:
            return Link(i*i, f(i+1))
    return f(1)
```

Step by step for `squares(3)`:

1. `f(1)`: `1 <= 3`, so return `Link(1, f(2))`. Python must evaluate `f(2)` before it can construct the `Link`.
2. `f(2)`: return `Link(4, f(3))`.
3. `f(3)`: return `Link(9, f(4))`.
4. `f(4)`: `4 > 3`, return `()`.
5. Now the `Link`s are built from the inside out: `Link(9, ())`, then `Link(4, Link(9, ()))`, then `Link(1, Link(4, Link(9, ())))`.

In box-and-pointer terms: three `Link` objects, each a two-field box. The first box holds `1` and an arrow to the second; the second holds `4` and an arrow to the third; the third holds `9` and `()`. `print` (via the `__str__` that `link.py` supplies) displays this as `(1 4 9)`.

Note that `f` closes over `n` from the enclosing `squares` frame, so `i > n` works without passing `n` along.

### B. `low_squares`: filter while building, with a count parameter

```python
def low_squares(n) -> LinkedList[int]:
    """A linked list of n positive perfect squares with ones digits below 5.

    >>> print(low_squares(10))
    (1 4 25 64 81 100 121 144 225 324)
    """
    def f(i, k):
        if k == n:
            return ()
        elif i * i % 10 <= 5:
            return Link(i*i, f(i+1, k+1))
        else:
            return f(i+1, k)
    return f(1, 0)
```

`i` is what to square; `k` is how many have been found.

Trace of `low_squares(3)`, i.e. `f(1, 0)`:

| call | `i*i` | `i*i % 10` | branch | action |
| --- | --- | --- | --- | --- |
| `f(1, 0)` | 1 | 1 | `<= 5` | `Link(1, f(2, 1))` |
| `f(2, 1)` | 4 | 4 | `<= 5` | `Link(4, f(3, 2))` |
| `f(3, 2)` | 9 | 9 | else | `f(4, 2)` (skip, nothing built, `k` unchanged) |
| `f(4, 2)` | 16 | 6 | else | `f(5, 2)` |
| `f(5, 2)` | 25 | 5 | `<= 5` | `Link(25, f(6, 3))` |
| `f(6, 3)` | - | - | `k == n` | `()` |

Result: `(1 4 25)`. Two things to notice. The termination test is `k == n`, so the recursion keeps going as long as it needs to in order to find enough qualifying squares (it is not bounded by `i`). And in the skip branch, you return the recursive call *unchanged*: no `Link` is created for a rejected value.

Also notice `i * i % 10 <= 5` uses the ones digit of the square, not of `i`. The docstring says "ones digit below 5", and `<= 5` is what the lecture wrote; it keeps 25 (ones digit 5) because `5 <= 5`, which is consistent with the expected output `(1 4 25 64 81 100 121 144 225 324)`.

### C. `slice_link`: slicing a linked list

```python
def slice_link(s: LinkedList, i, j) -> LinkedList:
    """Return a linked list containing elements from i:j.

    >>> evens = Link(4, Link(2, Link(6)))
    >>> print(slice_link(evens, 1, 100))
    (2 6)
    >>> print(slice_link(evens, 1, 2))
    (2)
    >>> print(slice_link(evens, 0, 2))
    (4 2)
    >>> slice_link(evens, 1, 1)
    ()
    """
    assert i >= 0 and j >= 0
    if j == 0 or s == ():
        return ()
    elif i == 0:
        return Link(s.first, slice_link(s.rest, 0, j-1))
    else:
        return slice_link(s.rest, i-1, j-1)
```

Trace `slice_link(evens, 1, 2)` on `evens = Link(4, Link(2, Link(6)))`:

1. `j` is 2 (not 0) and `s` is not `()`. `i` is 1, not 0, so take the else branch: return `slice_link(evens.rest, 0, 1)`. We moved past the `4` and decremented both counters.
2. `slice_link(Link(2, Link(6)), 0, 1)`: `j` is 1, `s` is nonempty, and now `i == 0`, so return `Link(2, slice_link(evens.rest.rest, 0, 0))`.
3. `slice_link(Link(6), 0, 0)`: `j == 0`, return `()`.
4. The answer is `Link(2, ())`, printed as `(2)`.

Why `slice_link(evens, 1, 1)` is `()`: the else branch gives `slice_link(evens.rest, 0, 0)`, and `j == 0` immediately returns `()`, matching Python's behavior for `s[1:1]`.

Why `slice_link(evens, 1, 100)` does not crash: after emitting `2` and `6`, the recursive call receives `s == ()` with `j` still large, and the `s == ()` base case catches it. Both base cases are needed, and because they are joined with `or`, the order of the two tests does not matter here.

A small equivalence: the slide wrote the second branch as `Link(s.first, slice_link(s.rest, i, j-1))` while the code file wrote `slice_link(s.rest, 0, j-1)`. In that branch `i` is already 0, so the two are identical.

Note also that no original `Link` is modified: `slice_link` allocates a fresh `Link` per emitted element and shares nothing but the element values.

### D. List mutation in environment diagrams (video review)

Each of these restarts from `s = [2, 3]` and `t = [5, 6]`.

**`append` adds one element:**

```python
s = [2, 3]
t = [5, 6]
s.append(t)
t = 0
# s is [2, 3, [5, 6]]   and   t is 0
```

Diagram reasoning: `s.append(t)` evaluates `t` to the list object `[5, 6]` and stores an arrow to *that object* as `s`'s third element. Rebinding `t = 0` changes only the global binding for `t`; the arrow from inside `s` still points at the list. `s` is a three-element list: two numbers and a list.

**`extend` adds all elements:**

```python
s = [2, 3]
t = [5, 6]
s.extend(t)
t[1] = 0
# s is [2, 3, 5, 6]   and   t is [5, 0]
```

Here `s` ends up containing only numbers (copied values, not an arrow to `t`'s list), so mutating `t` cannot affect `s`.

**`+` and slicing build new lists, but share their elements:**

```python
s = [2, 3]
t = [5, 6]
a = s + [t]
b = a[1:]
a[1] = 9
b[1][1] = 0
# s is [2, 3], t is [5, 0], a is [2, 9, [5, 0]], b is [3, [5, 0]]
```

Step by step:

1. `[t]` builds a temporary one-element list holding an arrow to `t`'s list. It is never named, so in a diagram you may erase it as unreachable.
2. `s + [t]` builds a *new* three-element list `[2, 3, <arrow to [5,6]>]` and names it `a`. `s` is unaffected.
3. `a[1:]` builds another new list `[3, <arrow to [5,6]>]` and names it `b`. Note that the inner arrow is *shared*: `a[2]`, `b[1]`, and `t` are three paths to one object.
4. `a[1] = 9` changes only `a`'s own second element.
5. `b[1][1] = 0` follows `b[1]` to the shared `[5, 6]` and sets its element 1 to `0`. Now you see the same `0` three ways: as `t[1]`, as `a[2][1]`, and as `b[1][1]`.

**`list` copies:**

```python
s = [2, 3]
t = list(s)
s[1] = 0
# s is [2, 0]   and   t is [2, 3]
```

**Slice assignment can change length:**

```python
s = [2, 3]
t = [5, 6]
s[0:0] = t     # insert all of t at the front: s is [5, 6, 2, 3]
s[3:] = t      # replace the length-1 slice [3] with [5, 6]: s is [5, 6, 2, 5, 6]
t[1] = 0
# s is [5, 6, 2, 5, 6]   and   t is [5, 0]
```

`s[0:0]` names an empty slice, so assigning a length-2 list there inserts two elements and shifts everything right. `s[3:]` names a length-1 slice, so assigning a length-2 list replaces one element with two. Because the elements copied in are numbers, the later `t[1] = 0` does not change `s`.

**Shrinking:**

```python
s = [2, 3]
t = [5, 6]
t = s.pop()
# s is [2], t is 3, and the list [5, 6] is now unreachable
```

```python
s = [2, 3]
t = [5, 6]
t.extend(t)    # t is [5, 6, 5, 6]
t.remove(5)    # removes the FIRST 5 only: t is [6, 5, 6]
```

(The video's audio cuts off mid-sentence on the `remove` example; the behavior stated on the slides is that `remove` removes the first match, which gives `[6, 5, 6]`.)

**The `suits` demo from the slides:**

```python
suits = ['coin', 'string', 'myriad', 'cup']
suits.pop()              # -> 'cup';    suits is ['coin', 'string', 'myriad']
suits.pop(1)             # -> 'string'; suits is ['coin', 'myriad']
suits.remove('myriad')   # -> None;     suits is ['coin']
suits.append('spade')                 # ['coin', 'spade']
suits.extend(['heart', 'club'])       # ['coin', 'spade', 'heart', 'club']
suits[0] = 'diamond'                  # ['diamond', 'spade', 'heart', 'club']
suits.insert(0, 'joker')              # ['joker', 'diamond', 'spade', 'heart', 'club']
print(suits.append('x'))              # prints None (append returns None)

s = [1, 2]
s.append([3, 4])         # s is [1, 2, [3, 4]]
s.extend([5, 6])         # s is [1, 2, [3, 4], 5, 6]

t = s
s += [7]                 # MUTATES: s and t are both [1, 2, [3, 4], 5, 6, 7]
s = s + [8]              # NEW LIST: s is [..., 7, 8]; t is still [..., 7]
```

### E. Lists in lists in environment diagrams

```python
t = [[1, 2], [3, 4]]
list(t)
t[0].append(t[1:2])
print(t)
# [[1, 2, [[3, 4]]], [3, 4]]
```

Reasoning:

- `t` is a two-element list whose elements are arrows to `[1, 2]` and `[3, 4]`.
- `list(t)` builds a shallow copy (a new outer list pointing at the same two inner lists), but the value is discarded. The slide uses it to label the three equivalent copy idioms: `list(t)`, `t[:]`, `t + []`.
- `t[1:2]` builds a *new* one-element list whose single element is an arrow to the same `[3, 4]` object. In diagram terms, this is a new box containing one arrow.
- `t[0].append(...)` mutates the inner list `[1, 2]` in place, making it `[1, 2, [[3, 4]]]`.
- `t` itself still has length 2 and its second element is still `[3, 4]`, so printing gives `[[1, 2, [[3, 4]]], [3, 4]]`. The doubled brackets come from the one-element slice wrapping the `[3, 4]` list.

### F. Fall 2022 Midterm 2, Question 2

```python
def of(us):
    def last(k):
        "The last k items of us"
        while k > 0:
            result.append(us.pop())
            k = k - 1
        return result
    return last

def surround(n, f):
    "n is the first and last item of f(2)"
    result = [n]
    result = f(2)
    result[0] = [n]
    return result.append(n)

result = [1]
surround(3, of([4, 5, 6]))
print(result)
# prints [[3], 6, 5, 3]
```

The trap is name resolution. Inside `last`, the name `result` is not local, and `last`'s parent frame is the `of` frame (`f1`), which binds only `us` and `last`. So `result` resolves all the way out to the **global** `result`, which is `[1]`. The local `result` in `surround` is irrelevant to `last`, even though `last` is called from `surround`: lexical scoping follows the parent frame of the function's definition, not the caller.

Step by step:

1. `of([4, 5, 6])` creates frame `f1` with `us` bound to the list `[4, 5, 6]`, defines `last` with `[parent=f1]`, and returns that function.
2. `surround(3, <last>)` creates frame `f2` with `n = 3`, `f = last`.
3. `result = [n]` binds a *local* `result` in `f2` to a new list `[3]`.
4. `result = f(2)` calls `last(2)` in frame `f3` with `k = 2`, parent `f1`.
   - First iteration: `us.pop()` removes and returns `6`; `result.append(6)` mutates the **global** list to `[1, 6]`. `k` becomes 1.
   - Second iteration: `us.pop()` returns `5`; global list becomes `[1, 6, 5]`. `k` becomes 0.
   - Returns the global list object `[1, 6, 5]`.
   - Side effect: `us` is now `[4]`.
5. Back in `surround`, the local `result` is rebound to that same global list object (they are now aliases).
6. `result[0] = [n]` replaces the `1` with a new one-element list `[3]`: the list is now `[[3], 6, 5]`.
7. `return result.append(n)` mutates it to `[[3], 6, 5, 3]` and returns `None` (so `surround` returns `None`).
8. `print(result)` in the global frame prints `[[3], 6, 5, 3]`.

Diagram notes matching the slide: `f3: last [parent=f1]`, `k` shown as `2` crossed out to `0`, `surround`'s return value `None`, and the global `result` list drawn as four slots with an arrow from slot 0 to a one-element list containing `3`.

### G. Spring 2023 Midterm 2, Question 1

```python
def chain(s):
    return [s[0], s[1:]]

silver = [2, chain([3, 4, 5])]
gold = [silver[0], silver[1].pop()]
silver[0] = 1
platinum = chain(chain([6, 7, 8]))
```

```
>>> silver
[1, [3]]
>>> gold
[2, [4, 5]]
>>> platinum
[6, [[7, 8]]]
```

Step by step:

1. `chain(s)` returns a new two-element list: the first element of `s`, and a new list that is the rest of `s` (`s[1:]` is a fresh list).
2. `chain([3, 4, 5])` returns `[3, [4, 5]]`, so `silver` is `[2, [3, [4, 5]]]`. The list `[4, 5]` is a distinct object reachable as `silver[1][1]`.
3. `gold = [silver[0], silver[1].pop()]`. Evaluating the elements left to right: `silver[0]` is `2`; `silver[1].pop()` mutates `silver[1]` from `[3, [4, 5]]` to `[3]` and returns the `[4, 5]` object. So `gold` is `[2, [4, 5]]` where `gold[1]` *is* that same list object, and `silver` is now `[2, [3]]`.
4. `silver[0] = 1` makes `silver` into `[1, [3]]`. Note `gold[0]` is still `2`: it holds a copy of the value `2`, not a reference to `silver`'s slot.
5. `chain([6, 7, 8])` returns `[6, [7, 8]]`. Passing that to `chain` again: `s[0]` is `6`, and `s[1:]` is a new one-element list whose single element is the `[7, 8]` list. So `platinum` is `[6, [[7, 8]]]`.

The transferable lessons: `pop` both mutates and returns, slicing creates a new outer list but shares inner objects, and assigning into one list never changes another list that merely copied a value out of it.

### H. `prefixes`: the aliasing bug and its fix

```python
def prefixes(s):
    """Return a list of all of the list prefixes of s.

    >>> prefixes([1, 2, 3])
    [[1], [1, 2], [1, 2, 3]]
    """
    so_far = []
    result = []
    for x in s:
        so_far.append(x)
        result.append(so_far)      # BUG
    return result
```

Running the buggy version gives:

```
>>> prefixes([1, 2, 3])
[[1, 2, 3], [1, 2, 3], [1, 2, 3]]
```

Why: `so_far` names one single list for the whole loop. `result.append(so_far)` stores an arrow to *that one object* three times. By the end, that object is `[1, 2, 3]`, and all three slots of `result` point at it, so you see the same contents three times. In a diagram, `result` has three arrows converging on one box.

The fix shown on the slide is to append a copy, taking a snapshot of the current state:

```python
        result.append(list(so_far))
```

Now each iteration stores a distinct list object: `[1]`, then `[1, 2]`, then `[1, 2, 3]`. (`so_far[:]` or `so_far + []` would work equally well.)

### I. `fibs_rec`: recursion plus mutation

```python
def fibs_rec(n):
    """Return a list of the first n fibonacci numbers.

    >>> fibs_rec(10)
    [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
    """
    s = []
    def f(i):
        if i < n:
            if i < 2:
                s.append(i)
            else:
                s.append(s[-2] + s[-1])
            f(i + 1)
    f(0)
    return s
```

Step by step for `fibs_rec(4)`:

- `s = []`, then `f(0)`: `0 < 4`, and `0 < 2`, so append `0`: `s` is `[0]`. Recurse with `f(1)`.
- `f(1)`: `1 < 4`, and `1 < 2`, so append `1`: `s` is `[0, 1]`. Recurse with `f(2)`.
- `f(2)`: `2 < 4` but not `< 2`, so append `s[-2] + s[-1]` = `0 + 1` = `1`: `s` is `[0, 1, 1]`. Recurse with `f(3)`.
- `f(3)`: append `1 + 1` = `2`: `s` is `[0, 1, 1, 2]`. Recurse with `f(4)`.
- `f(4)`: `4 < 4` is false, so the `if` body is skipped, `f` returns `None`, and the recursion unwinds doing nothing more.
- `fibs_rec` returns `s`, which is `[0, 1, 1, 2]`.

Key observations: `i` counts how many numbers have been appended so far (it doubles as the index of the element being added), `f` returns `None` and works purely by effect, `s` is a nonlocal name that `f` *mutates* (not assigns, so no `nonlocal` statement is needed), and `s[-2] + s[-1]` reads the two most recent appends. The blanks on the slide were `if i < n:` and `f(i + 1)` inside `f`, and `f(0)` before the `return`.

### J. `Transaction`: class attributes in practice

```python
class Transaction:
    """A logged transaction.

    >>> s = [20, -3, -4]
    >>> ts = [Transaction(x) for x in s]
    >>> ts[1].balance()
    17
    >>> ts[2].balance()
    13
    """
    log = []                                   # class attribute: one shared list

    def __init__(self, amount):
        self.amount = amount                   # instance attribute
        self.prior = list(self.log)            # a COPY: a list of Transactions
        self.log.append(self)                  # mutates the shared class attribute

    def balance(self):
        """The sum of amounts for this transaction and all prior transactions"""
        return self.amount + sum([t.amount for t in self.prior])
```

Walking through `ts = [Transaction(x) for x in s]`:

1. `Transaction(20)`: `self.amount = 20`. `self.log` finds no instance attribute, so it reads the class attribute `[]`, and `list([])` gives a fresh empty list, so `self.prior = []`. Then `self.log.append(self)` makes the class-level log `[t0]`.
2. `Transaction(-3)`: `self.amount = -3`. `self.prior = list(self.log)` is a new list `[t0]` (a snapshot, so later appends to `log` do not change it). Then the log becomes `[t0, t1]`.
3. `Transaction(-4)`: `self.amount = -4`, `self.prior` is a new list `[t0, t1]`, and the log becomes `[t0, t1, t2]`.

Then:

- `ts[1].balance()` is `-3 + sum([20])` = `17`.
- `ts[2].balance()` is `-4 + sum([20, -3])` = `-4 + 17` = `13`.

Diagram shape (from the slide): one `Transaction` class object holding `log`, which points to a list with arrows to each instance; three instance objects each holding `amount` and `prior`, where `prior` points to its own separate list of arrows back to earlier instances.

Two blanks that the slide filled in are worth memorizing:

- `self.prior = list(self.log)`, with the note that `list(type(self).log)` and `list(Transaction.log)` are equivalent. The copy matters: without `list(...)`, every instance would share the single growing log and `prior` would include transactions created *after* it (including itself).
- `return self.amount + sum([t.amount for t in self.prior])`: `prior` holds `Transaction` objects, not numbers, so you must pull `.amount` off each one.

### K. Iterable/iterator exercises (video segment)

**1. Indices of the elements with the smallest absolute value.**

```python
def min_abs_indices(s):
    """A list of the indices of the elements of s with the smallest absolute value.

    >>> min_abs_indices([-4, -3, -2, 3, 2, 4])
    [2, 4]
    >>> min_abs_indices([1, 2, 3, 4, 5])
    [0]
    """
    min_abs = min(map(abs, s))
    return [i for i in range(len(s)) if abs(s[i]) == min_abs]
```

Why each piece: `min(s)` would give the smallest signed value (`-4`), and `min(s, key=abs)` would give the *element* with the smallest absolute value (`-2`), not the value `2`. Mapping `abs` over `s` first and then taking `min` gives the number `2`. Then, because we want *indices*, the comprehension iterates over `range(len(s))` and indexes with `s[i]`. (Iterating `for x in s` and testing `abs(x) == min_abs` would give the elements `[-2, 2]` instead.)

The equivalent `filter` version from the video:

```python
    def f(i):
        return abs(s[i]) == min_abs
    return list(filter(f, range(len(s))))
```

`list(...)` is required because `filter` returns an iterator object, not a list.

**2. Largest sum of two adjacent elements.**

```python
def largest_adj_sum(s):
    """The largest sum of two adjacent elements in a list s (len(s) > 1).

    >>> largest_adj_sum([-4, -3, -2, 3, 2, 4])
    6
    >>> largest_adj_sum([-4, 3, -2, -3, 2, -4])
    1
    """
    return max([s[i] + s[i+1] for i in range(len(s) - 1)])
```

The sums for the first example are `-7, -5, 1, 5, 6`, so the answer is `6`. The range stops one short because the last element has no `i + 1` neighbor. In the second example all sums are small and the answer `1` comes from `3 + (-2)`, which shows that you cannot just take the last pair or assume positivity.

The `zip` version:

```python
    return max([a + b for a, b in zip(s[:-1], s[1:])])
```

`zip(s, s)` would pair each element with itself, which is useless. Dropping the last element from the first sequence and the first element from the second sequence lines up neighbors, so `zip` yields exactly the adjacent pairs `(-4, -3), (-3, -2), (-2, 3), (3, 2), (2, 4)`.

**3. Dictionary from digit to the elements ending in that digit.**

```python
def digit_dict(s):
    """Map each digit d to the list of elements in s that end with d.

    >>> digit_dict([5, 8, 13, 21, 34, 55, 89])
    {5: [5, 55], 8: [8], 3: [13], 1: [21], 4: [34], 9: [89]}
    """
    last_digits = [x % 10 for x in s]
    return {d: [x for x in s if x % 10 == d] for d in range(10) if d in last_digits}
```

(The exact input list is reconstructed from the video's description of the result: keys for 1, 3, 4, 5, 8, 9 with two elements, `5` and `55`, under key 5.)

The structure is a dictionary comprehension whose key is `d`, whose value is itself a list comprehension, and whose `for`/`if` clauses iterate `range(10)` and drop digits that nothing ends in (otherwise you would get entries like `2: []`). The video first wrote the filter as `if any([x % 10 == d for x in s])` and then factored out `last_digits` to keep only one gnarly expression instead of two.

**4. Does every element equal some other element?**

```python
def all_have_an_equal(s):
    """Does every element equal some other element in s?

    >>> all_have_an_equal([-4, -3, -2, 3, 2, 4])
    False
    >>> all_have_an_equal([4, 3, 2, 2, 3, 4])
    True
    """
    return min([s.count(x) for x in s]) > 1
```

The video built this up in three stages:

- Index-based: `s[:i] + s[i+1:]` is the whole list with only the element at index `i` removed (other equal values survive), so `s[i] in s[:i] + s[i+1:]` asks whether `s[i]` appears elsewhere. Then wrap it: `all([s[i] in s[:i] + s[i+1:] for i in range(len(s))])`.
- Counting by comprehension: `len([y for y in s if y == x])` counts copies of `x`, and `all([len([y for y in s if y == x]) > 1 for x in s])` checks every element.
- Simplest: `s.count(x)` is built in, so compute all the counts and check that the smallest is above 1. `min([...]) > 1` and `all([... > 1 ...])` express the same idea.

## Common Pitfalls

1. **Forgetting that the recursive call is the `rest`.** Writing `Link(f(i+1), i*i)` or returning `Link(i*i)` and losing the tail. The constructor order is `Link(first, rest)`.
2. **Wrapping a skipped element.** In `low_squares`, the non-matching branch must `return f(i+1, k)`, not `return Link(i*i, f(i+1, k))`. Building a `Link` is how you *keep* a value.
3. **Using the wrong counter to terminate.** `low_squares` stops on `k == n` (count found), not on `i > n`. Mixing these up gives too few results.
4. **Omitting a base case in `slice_link`.** Without `s == ()`, `slice_link(evens, 1, 100)` would try `().first` and crash. Without `j == 0`, the slice would never end.
5. **Decrementing the wrong counters.** When emitting an element, only `j` decreases; when skipping an element, both `i` and `j` decrease.
6. **Trying `s[i:j]` on a linked list.** Slice notation is a list/sequence operation; `Link` does not support it, which is the whole reason `slice_link` exists.
7. **Drawing an arrow to a name.** In an environment diagram, `s.append(t)` stores the *value* of `t`. If you draw the arrow to the binding `t` instead of to the list object, the later `t = 0` will wrongly seem to change `s`.
8. **Confusing `append` with `extend`.** `s.append([3, 4])` adds one element that is a list; `s.extend([3, 4])` adds two numbers.
9. **Using the return value of a mutating method.** `append`, `extend`, `insert`, and `remove` all return `None`. `x = s.append(3)` binds `x` to `None`. Only `pop` returns a useful value.
10. **Assuming `+=` and `= ... + ...` behave alike.** `s += [7]` mutates the existing list (visible through every alias); `s = s + [8]` creates a new list and leaves aliases pointing at the old one.
11. **Appending an accumulator instead of a copy** (the `prefixes` bug). If you keep mutating one list and store it repeatedly, every stored reference shows the final contents. Store `list(so_far)` to snapshot.
12. **Believing a copy is deep.** `list(t)` / `t[:]` / `t + []` copy only the outer list; inner lists remain shared, so `t[0].append(...)` is visible through the copy.
13. **Resolving a name by where a function was called instead of where it was defined.** In the Fall 2022 problem, `result` inside `last` is the global `result`, not `surround`'s local `result`.
14. **Forgetting that `pop` both removes and returns.** In the Spring 2023 problem, `silver[1].pop()` shrinks `silver[1]` *and* hands the removed list to `gold`.
15. **Assigning to a class attribute through `self`.** `self.log = ...` would create an instance attribute and leave the class attribute untouched; `self.log.append(...)` mutates the shared one. Know which you want.
16. **Sharing when you meant to snapshot.** Without `list(...)` in `self.prior = list(self.log)`, `prior` would alias the growing log, so every transaction would eventually count itself and all later transactions.
17. **Forgetting that a class attribute persists across all instances** (and across doctests in the same session), so a `log` style list keeps growing.
18. **`min(s, key=abs)` vs `min(map(abs, s))`.** The first returns an element, the second a magnitude.
19. **Iterating over `s` when you need indices.** Use `range(len(s))` and index inside the comprehension.
20. **Off-by-one on adjacent pairs.** Use `range(len(s) - 1)`, or `zip(s[:-1], s[1:])`.
21. **Leaving empty entries in a dictionary comprehension.** Add an `if` clause over the keys to drop digits with no matching elements.

## Likely Exam Points

**1. Fill in a recursive linked-list constructor.**

*Q:* Complete `evens(n)` so it returns a linked list of the first `n` positive even numbers, so `print(evens(4))` shows `(2 4 6 8)`.

```python
def evens(n) -> LinkedList[int]:
    def f(i):
        if ____________:
            return ()
        else:
            return ____________
    return f(1)
```

*A:* `if i > n: return ()` and `return Link(2*i, f(i+1))`. The recursive result becomes the `rest`, and the base case returns the empty linked list `()`.

**2. Counting-parameter recursion (the `low_squares` pattern).**

*Q:* Why does `low_squares` need two parameters `i` and `k`, and what would go wrong with only `i`?

*A:* `i` says which number to square next, while `k` says how many qualifying squares have been produced. The stopping condition is "we have found `n` of them" (`k == n`), which is not the same as "we have examined `n` candidates", because some candidates are skipped. With only `i` you could not tell when to stop; you would produce however many of the first `n` candidates happened to qualify, so `low_squares(10)` would return fewer than 10 squares.

**3. Trace `slice_link`.**

*Q:* With `evens = Link(4, Link(2, Link(6)))`, what does `print(slice_link(evens, 0, 2))` display, and how many new `Link` objects are created?

*A:* It displays `(4 2)`. `i == 0` so it emits `4` and calls `slice_link(evens.rest, 0, 1)`, which emits `2` and calls `slice_link(evens.rest.rest, 0, 0)`, which hits `j == 0` and returns `()`. Two new `Link` objects are created; the original `evens` is unchanged.

**4. WWPD on mutation and aliasing.**

*Q:* What are `s` and `t` at the end?

```python
s = [1, 2]
t = s
s.append([3, 4])
s += [5]
s = s + [6]
t[0] = 0
```

*A:* `s` is `[0, 2, [3, 4], 5, 6]` and `t` is `[0, 2, [3, 4], 5]`. `append` and `+=` mutate the shared list, so `t` sees them. `s = s + [6]` builds a new list and rebinds only `s`, so `t` keeps pointing at the original, which still has 4 elements. Finally `t[0] = 0` mutates that original list, and since `s`'s new list copied the arrow/value from slot 0 *before* the change... careful: `s + [6]` copied the value `1` into the new list at the time of the addition, so `s[0]` is still... Let's be precise: `s + [6]` was evaluated before `t[0] = 0`, so the new list's slot 0 holds `1`. The final answer is `s` is `[1, 2, [3, 4], 5, 6]` and `t` is `[0, 2, [3, 4], 5]`.

**5. Return value of a mutating method.**

*Q:* What does this print?

```python
s = [1, 2]
print(s.append(3), s.pop(), s)
```

*A:* `None 3 [1, 2]`. `append` returns `None` while mutating `s` to `[1, 2, 3]`; `pop()` then removes and returns `3`, leaving `[1, 2]`. (All three arguments are evaluated left to right before printing.)

**6. The `prefixes` aliasing bug.**

*Q:* The buggy `prefixes([1, 2, 3])` returns `[[1, 2, 3], [1, 2, 3], [1, 2, 3]]`. Explain in one sentence and give the one-line fix.

*A:* `result` holds three references to the single `so_far` list, which by the end contains all three elements, so all three entries display the same final contents; fix it by appending a copy: `result.append(list(so_far))` (or `so_far[:]`).

**7. Recursion that works by mutation.**

*Q:* Fill in the blanks of `fibs_rec` and state what `f` returns.

*A:* `if i < n:` as the outer condition inside `f`, `f(i + 1)` as the last line of `f`'s body, and `f(0)` before `return s`. `f` returns `None`; its effect is appending to the nonlocal list `s`. No `nonlocal` statement is needed because `s` is mutated, never reassigned.

**8. Environment diagram name resolution (Fall 2022 MT2 Q2 style).**

*Q:* In that problem, `surround` binds a local `result = [n]` before calling `f(2)`. Why does `last` not append to that local list?

*A:* Because `last` was defined inside `of`, its parent frame is the `of` frame, whose parent is the global frame. Lexical scoping means the lookup of `result` goes instance-of-definition upward (`last` frame, `of` frame, global frame) and finds the global `result = [1]`. The caller's local variables are never searched. The program prints `[[3], 6, 5, 3]`.

**9. `pop` inside a list literal (Spring 2023 MT2 Q1 style).**

*Q:* After `silver = [2, chain([3, 4, 5])]` and `gold = [silver[0], silver[1].pop()]`, what are `silver` and `gold`? (`chain(s)` returns `[s[0], s[1:]]`.)

*A:* `silver` is `[2, [3]]` and `gold` is `[2, [4, 5]]`. `chain([3, 4, 5])` gave `[3, [4, 5]]`; `pop()` removed its last element (the list `[4, 5]`) and returned it, so that object ends up inside `gold` while `silver[1]` shrinks to `[3]`. A later `silver[0] = 1` makes `silver` `[1, [3]]` and does not affect `gold[0]`, which holds the value `2`.

**10. Nested slice and append.**

*Q:* What does `print(t)` show?

```python
t = [[1, 2], [3, 4]]
t[0].append(t[1:2])
```

*A:* `[[1, 2, [[3, 4]]], [3, 4]]`. `t[1:2]` is a new one-element list holding a reference to `[3, 4]`, and that one-element list is appended into the inner list `[1, 2]`.

**11. Class attribute vs instance attribute.**

*Q:* Given the `Transaction` class, what is `len(Transaction.log)` after `ts = [Transaction(x) for x in [20, -3, -4]]`, and what is `ts[0].prior`?

*A:* `len(Transaction.log)` is `3`, since `log` is a single shared class attribute that each `__init__` appends to. `ts[0].prior` is `[]`, because `prior` was a copy of the log taken before any transaction had been logged.

**12. Why the copy in `self.prior = list(self.log)`.**

*Q:* What would `ts[1].balance()` return if `__init__` used `self.prior = self.log` instead?

*A:* `prior` would alias the shared log rather than snapshot it, so by the time `balance` runs, `ts[1].prior` would be `[t0, t1, t2]` and the answer would be `-3 + (20 + -3 + -4) = -10` instead of `17`. The copy is what makes `prior` mean "the transactions that existed before me".

**13. Comprehension/built-in one-liners.**

*Q:* Write a one-line expression for the largest sum of two adjacent elements of `s` (assume `len(s) > 1`), using `zip`.

*A:* `max([a + b for a, b in zip(s[:-1], s[1:])])`. Dropping the last element of the first sequence and the first element of the second lines up neighboring pairs.

**14. Indices vs elements.**

*Q:* What is the difference between `[x for x in s if abs(x) == min(map(abs, s))]` and `[i for i in range(len(s)) if abs(s[i]) == min(map(abs, s))]` for `s = [-4, -3, -2, 3, 2, 4]`?

*A:* The first gives the *elements* `[-2, 2]`; the second gives the *indices* `[2, 4]`. To produce indices you must iterate over `range(len(s))` and index into `s`.

**15. Filtering keys in a dictionary comprehension.**

*Q:* Why does `digit_dict` need the `if` clause on `d`?

*A:* Without it, every digit `0` through `9` would become a key, including digits that no element ends in, giving entries with empty lists (for example `2: []`). The `if d in last_digits` (equivalently `if any([x % 10 == d for x in s])`) keeps only digits that actually occur.

## Summary

- A linked list is `()` or a `Link(first, rest)` where `rest` is itself a linked list; build one recursively by letting the recursive call's return value be the `rest`, with `()` as the base case.
- `squares` needs one helper parameter (what to square); `low_squares` needs two (what to square, how many found), stops on the count, and *returns the recursive call unchanged* to skip a value.
- `slice_link(s, i, j)` reimplements `s[i:j]`: base cases `j == 0` or `s == ()`; when `i == 0` emit `s.first` and recurse with `j-1`; otherwise skip and recurse with `i-1, j-1`. It builds new `Link`s and never mutates the input.
- Mutating list operations: `append` (one element), `extend` (all elements), `insert`, `remove` (first match), `pop` (removes and returns), item assignment, slice assignment (can change length), and `+=`. All except `pop` return `None`.
- Non-mutating operations that build new lists: `list(t)`, `t[:]`, `t + []`, `+`, slicing, and comprehensions. These copies are shallow: inner lists stay shared.
- In environment diagrams, arrows point to values, not names; unnamed intermediate lists are unreachable and may be erased; a change to a shared object shows up through every path that reaches it.
- `s += t` mutates (aliases see it); `s = s + t` rebinds to a new list (aliases do not).
- The `prefixes` bug: appending the same accumulator repeatedly stores one object many times; append `list(so_far)` to snapshot each step.
- `fibs_rec` shows recursion plus mutation: an outer list, an inner recursive function that appends and returns `None`, and `s[-2] + s[-1]` to read the last two values.
- Fall 2022 MT2 Q2 lesson: names in a nested function resolve through its *definition* environment (parent frame), not the caller, so `last` appends to the global `result`; answer `[[3], 6, 5, 3]`.
- Spring 2023 MT2 Q1 lesson: `pop` mutates and returns, and slicing creates a new outer list that shares inner objects; `silver` is `[1, [3]]`, `gold` is `[2, [4, 5]]`, `platinum` is `[6, [[7, 8]]]`.
- A class attribute is one value shared by the class and all instances: `self.log.append(self)` mutates it, while `self.prior = list(self.log)` creates a per-instance snapshot copy. Assignment through `self` always creates an instance attribute.
- Data-manipulation toolkit: `min(map(abs, s))` for a magnitude, `range(len(s))` for indices, `range(len(s) - 1)` or `zip(s[:-1], s[1:])` for adjacent pairs, dictionary comprehensions with an `if` clause over keys, and aggregators (`max`, `min`, `sum`, `any`, `all`, `s.count`) to collapse a comprehension to one answer.
