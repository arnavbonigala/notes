<!-- Wed, Oct 14, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 21: Inheritance

## Overview

This lecture finishes the attribute-lookup material from the previous lecture with two warm-up exercises (the `Place` class, which uses a mutable *class* attribute to chain instances together, and the `Friend` class, where methods mutate *other* objects passed in as arguments), then introduces **inheritance**: a way of relating two classes when one is a specialized version of another. Writing `class CheckingAccount(Account):` says "a checking account *is an* account", and the subclass only needs to write down what is *different*: a lower `interest`, a `withdraw_fee`, and an overriding `withdraw` method that defers to `Account.withdraw` (or `super().withdraw`) rather than copy-pasting its logic. The key mechanical idea is that base class attributes are **not copied** into the subclass; instead, attribute *lookup* walks up the chain (instance, then class, then base class, then its base class, ...). The lecture then covers design guidance (don't repeat yourself, prefer looking attributes up on the instance so instances and further subclasses can specialize them, and use inheritance for *is-a* relationships but **composition** for *has-a* relationships, as in a `Bank` that holds a list of accounts), multiple inheritance (`AsSeenOnTVAccount(CheckingAccount, SavingsAccount)`), and ends with two "what would Python do?" stress tests of attribute lookup (`Where's Waldo` and the `A`/`B`/`C` example).

---

## Key Concepts

### 1. Review: the two kinds of attribute, and the asymmetry of assignment

An **instance attribute** lives in one object; a **class attribute** is shared by the class and all of its instances (and all of its subclasses' instances).

Lookup and assignment are **not** symmetric, and this is the single most tested idea in this part of the course:

- **Lookup** `obj.x`: first check the instance's own attributes. If not found there, look in the instance's class (and then that class's base class, and so on). So lookup *falls back* to the class.
- **Assignment** `obj.x = v`: **always** creates or overwrites an attribute *on that instance*. It never, ever modifies the class. So assignment does *not* fall back to the class.
- To change a class attribute you must assign to a class: `Place.last = self`, or `type(self).last = self`.

This is why the lecture notes on the `Place` slide say it is "OK to write `self.last` or `type(self).last`" for *reading* `Place.last`, but it is "not ok to write `self.last`" for *writing* it: `self.last = self` would quietly stash a per-instance `last` and leave the shared class attribute alone.

The same trap hides inside augmented assignment. `other.talked_to += 1` means `other.talked_to = other.talked_to + 1`: the right side uses lookup (which may find a class attribute), and the left side always writes an instance attribute.

### 2. Inheritance: specialization without duplication

Not every class exists in isolation. The most common use for inheritance is two similar classes that differ in their **degree of specialization**: the specialized class has all the attributes of the general class, plus some special-case behavior. Syntax:

```python
class CheckingAccount(Account):   # CheckingAccount is a subclass; Account is the base class
    ...
```

Conceptually the subclass **shares** all the attributes of its base class. It may **override** some of them to change behavior slightly; anything not overridden stays the same. So when you write a subclass, you write down **only what is different**.

### 3. How lookup actually works (nothing is copied)

> "Base class attributes aren't copied into subclasses!"

To look up a name **in a class**:

1. If the name is an attribute in the class, return its value.
2. Otherwise, look up the name in the base class, if there is one (and this is recursive: the base class's own lookup will consult *its* base class).
3. If no class in the chain has it, `AttributeError`.

To look up a name on an **instance**: check the instance's own attributes first, and only then run the class lookup above starting at `type(instance)`. If the result is a function found via a class, you get a **bound method** (with `self` already attached).

Tracing `ch = CheckingAccount('Tom')`:

| Expression | Where the name is found | Result |
|---|---|---|
| `CheckingAccount('Tom')` | `__init__` not in `CheckingAccount`, found in `Account` | sets `holder='Tom'`, `balance=0` |
| `ch.interest` | not an instance attr; found in `CheckingAccount` | `0.01` |
| `ch.deposit(20)` | not an instance attr; not in `CheckingAccount`; found in `Account` | `20` |
| `ch.withdraw(5)` | found in `CheckingAccount` (overrides `Account.withdraw`) | `14` |

### 4. Overridden attributes are still reachable through the class object

Overriding means giving an attribute in the subclass the **same name** as one in the base class. The base class version is not destroyed: the class object still has it. Two ways to reach it from inside `CheckingAccount.withdraw`:

```python
return Account.withdraw(self, amount + self.withdraw_fee)   # explicit self required
return super().withdraw(amount + self.withdraw_fee)         # super() supplies self for you
```

Looking up `withdraw` on the **class** `Account` gives a plain function, so you must pass `self` yourself. `super()` inside a method starts the lookup *after* the current class in the inheritance chain and gives you a bound method, so you do not pass `self`.

Why defer instead of copy-pasting `Account.withdraw`'s body and adding `amount += 1`? Because the copy "will be correct for a while", but later changes to `Account.withdraw` would not be reflected in the subclass. Don't repeat yourself.

### 5. Prefer looking attributes up on the instance

Inside `CheckingAccount.withdraw` there were three ways to get the fee:

- `amount + 1`: hard-coded, can never be changed.
- `amount + CheckingAccount.withdraw_fee`: finds the class attribute, but ignores the possibility that a particular account, or a further subclass, has its own fee.
- `amount + self.withdraw_fee`: **best**. If the instance has its own `withdraw_fee`, use it; otherwise fall back through the class chain. This is what makes specialization by instance attribute or by further subclassing possible.

So the general rule from this lecture: use a *class* name when you deliberately want a specific (overridden) version, as in `Account.withdraw`, and use `self.` whenever you want the most specialized version available.

### 6. Inheritance (is-a) vs composition (has-a)

Object-oriented programming works best when the code follows the real-world metaphor.

- **Inheritance** represents **is-a**: a `CheckingAccount` *is a* kind of `Account`, so it inherits from `Account` and thereby has all of an account's attributes.
- **Composition** represents **has-a**: one object has another object as an attribute. A `Bank` *has a* list of accounts. A bank does not inherit from `Account`, and an account does not inherit from `Bank`; they are related only by one holding the other.

### 7. Multiple inheritance

A class may have more than one base class: `class AsSeenOnTVAccount(CheckingAccount, SavingsAccount):`. Lookup then has to consider several chains. The lecture's rule of thumb is all you need:

> There is a particular order in which we look at all of these, and all you need to know is that you look at the subclass before you look at the base class.

For `AsSeenOnTVAccount` the practical order is: the instance, then `AsSeenOnTVAccount`, then `CheckingAccount`, then `SavingsAccount`, then `Account`. So `withdraw` comes from `CheckingAccount` and `deposit` comes from `SavingsAccount`.

*(extra context)* The exact algorithm Python uses is called the **method resolution order** (MRO, computed by C3 linearization), visible as `AsSeenOnTVAccount.__mro__`. CS 61A only expects "left to right, subclass before base class".

---

## Definitions

- **Inheritance**: a mechanism for relating classes, in which a subclass shares the attributes of its base class(es) and may override some of them. Expressed as `class Sub(Base):`.
- **Base class (superclass / parent class)**: the class named in parentheses in a `class` statement; the class whose attributes are inherited.
- **Subclass (child class)**: the class created by a `class` statement that names a base class. "A subclass *is a* specialized kind of its base class."
- **Attribute**: a name-value pair stored on an object or a class.
- **Instance attribute**: an attribute stored on a single object. Created by any assignment of the form `obj.name = value` (typically `self.name = value` in `__init__`).
- **Class attribute**: an attribute stored on a class, shared by the class, its instances, and its subclasses and their instances. Created by an assignment in the class body, or by `ClassName.name = value`.
- **Attribute lookup (on an instance)**: check the instance's own attributes; if absent, look the name up in its class, following base classes recursively; a function found this way becomes a bound method.
- **Attribute lookup (on a class)**: if the name is an attribute of the class, return its value; otherwise look it up in the base class, if there is one.
- **Overriding**: defining in a subclass an attribute with the same name as one in a base class, so that lookups starting at the subclass find the new one. The base class version remains accessible through the base class object (for example `Account.withdraw`).
- **Bound method**: a function found by looking up a name on an *instance* through its class, packaged together with that instance so that `self` is supplied automatically. Looking the same name up on the *class* gives the plain function, which requires an explicit `self` argument.
- **`super()`**: inside a method, returns a proxy that performs attribute lookup starting *after* the current class in the inheritance order, returning bound methods (so `self` is not passed explicitly).
- **Composition**: a design relationship in which one object holds another object as an attribute; models *has-a*.
- **Multiple inheritance**: a class with more than one base class.
- **Method resolution order**: the fixed order in which classes are searched during attribute lookup; subclasses always precede their base classes.

---

## Worked Examples

### Example 1: `Place`, mutating a class attribute

```python
class Place:
    """
    >>> places = [Place(x*2) for x in range(10)]
    >>> places[2].print_history()
    4
    2
    0
    """
    last = None

    def __init__(self, n):
        self.name = n
        self.then = Place.last     # the Place created just before this one
        Place.last = self          # this Place becomes the newest one

    def print_history(self):
        print(self.name)
        if self.then is not None:
            Place.print_history(self.then)   # same as self.then.print_history()

    def __str__(self):
        return f"(Place: name: {self.name}, then: {self.then})"
```

**Step by step.** Executing the `class` statement creates a `Place` class object with attributes `last: None`, `__init__`, `print_history`, and `__str__`.

`Place(0)`: a new instance is created and `__init__` runs with `self` bound to it and `n = 0`.
1. `self.name = 0` creates an instance attribute.
2. `self.then = Place.last` *reads* the class attribute, which is still `None`, and stores `None` as an instance attribute. (Reading `self.last` or `type(self).last` here would give the same `None`, which is why the slide marks those as acceptable.)
3. `Place.last = self` *writes* the class attribute, repointing it at this new instance.

`Place(2)`: `self.then` is read as the `Place(0)` instance, then `Place.last` is repointed at `Place(2)`. And so on.

**Box-and-pointer picture in words.** There is exactly one `Place` class object, holding one `last` arrow. Each instance holds a `name` and a `then` arrow. Instance `Place(4)`'s `then` points at `Place(2)`, whose `then` points at `Place(0)`, whose `then` is `None`. So the instances form a **linked list in reverse creation order**, and `Place.last` is a pointer to the head (the most recently created place). The list comprehension creates places named 0, 2, 4, ..., 18.

`places[2]` is the place named 4, so `print_history` prints `4`, then recurses on `then` to print `2`, then `0`, then stops at `None`.

Note `Place.print_history(self.then)` in the body: looking the method up on the *class* yields a plain function, so `self.then` is passed explicitly as `self`. It is exactly equivalent to `self.then.print_history()`, which looks the name up on an instance and gets a bound method.

**The "no `places` variable" exercise.** To print all the places without using `places`:

```python
>>> Place.last.print_history()
18
16
14
12
10
8
6
4
2
0
```

because `Place.last` is the head of the chain.

**Why `self.last = self` is wrong.** If `__init__` ended with `self.last = self`, every instance would get its *own* `last` attribute pointing at itself, the class attribute `Place.last` would stay `None` forever, and every new place's `then` would be `None`, so `print_history` would print only one name.

**The unfinished exercise in the code file** (left blank in lecture; the intended fill-in is):

```python
def add_country_to_all(country: str):
    def add_country(place: Place):
        if place:
            place.country = country          # instance attribute on each place
            add_country(place.then)          # walk down the chain
    add_country(Place.last)                  # start from the newest place
```

*(extra context: this reconstruction of the blanks is mine; the posted file leaves them empty. Note that `place.country = country` creates a brand new instance attribute, demonstrating again that attribute assignment always targets the instance. Also, `from __future__ import annotations` at the top of the file is what lets the annotation `place: Place` refer to `Place` inside its own definition.)*

### Example 2: `Friend`, methods that mutate their arguments

```python
class Friend:
    """
    >>> john = Friend('John')
    >>> kay = Friend('Kay')
    >>> josh = Friend('Josh')
    >>> john.talk(kay)
    >>> josh.talk(kay)
    >>> john.talked_to
    0
    >>> kay.talked_to
    2
    >>> kay.is_bestie(josh)
    True
    >>> kay.is_bestie(john)
    False
    """

    def __init__(self, name: str):
        self.name = name
        self.just_talked = None
        self.talked_to = 0          # how many times someone else talked to this Friend

    def talk(self, other):
        self.just_talked = other
        other.just_talked = self
        other.talked_to += 1

    def is_bestie(self, other):
        return self.just_talked == other and other.just_talked == self

    def has_bestie(self):
        return self.just_talked and self.is_bestie(self.just_talked)
```

**Step by step.** `talked_to` counts how many times *someone else* talked to this friend, so it is incremented on `other`, not on `self`. This is the whole point of the exercise: a method may read and write the attributes of any object it can reach, not only `self`.

- `john.talk(kay)`: `self` is john, `other` is kay. john's `just_talked` becomes kay; kay's `just_talked` becomes john; kay's `talked_to` goes 0 to 1.
- `josh.talk(kay)`: josh's `just_talked` becomes kay; kay's `just_talked` is **overwritten** to josh; kay's `talked_to` goes 1 to 2.
- `john.talked_to` is still `0`: nobody ever called `talk` with john as `other`.
- `kay.is_bestie(josh)`: kay's `just_talked` is josh, and josh's `just_talked` is kay, so `True`.
- `kay.is_bestie(john)`: kay's `just_talked` is josh, not john, so the first conjunct is false and `False` is returned (short-circuiting means john's attributes are never consulted).

`has_bestie` guards with `self.just_talked` first so that a friend who has never talked to anyone (whose `just_talked` is `None`) returns a false value instead of comparing `None` against things. Note that `has_bestie` returns `None` rather than `False` in that case, since `and` returns its first false operand.

*(extra context)* `==` on these objects uses the default behavior, identity comparison, because `Friend` does not define `__eq__`. Using `is` would be equivalent and arguably clearer here.

### Example 3: `Account` and `CheckingAccount`

```python
class Account:
    """An account has a balance and a holder.

    >>> a = Account('John')
    >>> a.holder
    'John'
    >>> a.deposit(100)
    100
    >>> a.withdraw(90)
    10
    >>> a.withdraw(90)
    'Insufficient funds'
    >>> a.balance
    10
    >>> a.interest
    0.02
    """

    interest = 0.02                        # a class attribute

    def __init__(self, account_holder):
        self.holder = account_holder
        self.balance = 0

    def deposit(self, amount):
        self.balance = self.balance + amount
        return self.balance

    def withdraw(self, amount):
        if amount > self.balance:
            return 'Insufficient funds'
        self.balance = self.balance - amount
        return self.balance


class CheckingAccount(Account):
    """A bank account that charges for withdrawals.

    >>> ch = CheckingAccount('Jack')
    >>> ch.balance = 20
    >>> ch.withdraw(5)
    14
    >>> ch.interest
    0.01
    """

    withdraw_fee = 1
    interest = 0.01

    def withdraw(self, amount):
        return Account.withdraw(self, amount + self.withdraw_fee)
```

**Step by step for `ch.withdraw(5)` with a balance of 20:**

1. `ch.withdraw` is a dot expression. Look for `withdraw` among `ch`'s instance attributes: not there (`ch` has only `holder` and `balance`). Look in `type(ch)`, which is `CheckingAccount`: found. We get a bound method, so `self` is `ch`.
2. `amount` is 5. `self.withdraw_fee`: not an instance attribute, found in `CheckingAccount` as `1`. Total is 6.
3. `Account.withdraw` is looked up on the **class** `Account`, so it is a plain function and we pass `self` explicitly: `Account.withdraw(ch, 6)`.
4. Inside, `6 > 20` is false, so `self.balance = 20 - 6 = 14` (an instance attribute assignment on `ch`), and `14` is returned.

The slide's alternative, `super().withdraw(amount + self.withdraw_fee)`, does the same thing without naming `Account` and without passing `self`. (The posted code file lists both `return` statements one after the other, so the second is dead code; it is there to show the alternative, not to run.)

Live demo recap: `a = Account('John')` and `b = CheckingAccount('Jack')` both start at balance 0 and both use the *same* `Account.deposit` function, so depositing 100 into each gives 100. But `a.withdraw(10)` leaves 90 while `b.withdraw(10)` leaves 89, because the extra dollar of fee is charged.

Also note what `CheckingAccount` did **not** have to write: `__init__`, `deposit`, and `balance`/`holder` handling all come from `Account` unchanged.

### Example 4: `Bank`, composition rather than inheritance

Reconstructed from the live demo described in lecture *(extra context: this exact code is not in the posted `21.py`; the structure and behavior follow the lecture narration closely)*:

```python
class Bank:
    """A bank has accounts.

    >>> bank = Bank()
    >>> john = bank.open_account('John', 10)
    >>> jack = bank.open_account('Jack', 5, CheckingAccount)
    >>> john.interest
    0.02
    >>> jack.interest
    0.01
    >>> bank.pay_interest()
    >>> john.balance
    10.2
    >>> bank.too_big_to_fail()
    True
    """

    def __init__(self):
        self.accounts = []

    def open_account(self, holder, amount, kind=Account):
        account = kind(holder)
        account.deposit(amount)
        self.accounts.append(account)
        return account

    def pay_interest(self):
        for a in self.accounts:
            a.deposit(a.balance * a.interest)

    def too_big_to_fail(self):
        return len(self.accounts) > 1
```

**Why there is no inheritance here.** A bank *has* accounts; it is not a kind of account, and an account is not a kind of bank. So the relationship is modeled by an attribute (`self.accounts`), which is composition.

**Things to notice.**
- `kind` is a parameter whose *value is a class*: `kind(holder)` calls either `Account` or `CheckingAccount`. Classes are first-class values, so they can be passed as arguments and given as default values.
- `pay_interest` writes `a.interest`, not `Account.interest`. Because the lookup starts at the instance, each account automatically gets *its own* rate: John's regular account pays 0.02 (10 becomes 10.2) while Jack's checking account pays 0.01. This is the "look up attributes on instances whenever possible" guideline paying off: `pay_interest` needs no knowledge of what kinds of accounts exist.

### Example 5: Multiple inheritance, `AsSeenOnTVAccount`

```python
class SavingsAccount(Account):
    deposit_fee = 2

    def deposit(self, amount):
        return Account.deposit(self, amount - self.deposit_fee)


class AsSeenOnTVAccount(CheckingAccount, SavingsAccount):
    def __init__(self, account_holder):
        self.holder = account_holder
        self.balance = 1       # A free dollar!
```

*(extra context: the bodies of these two classes are reconstructed from the lecture narration; they are not in the posted `21.py`.)*

The sales pitch: a 1% interest rate, a $1 withdrawal fee, a $2 deposit fee, and a free dollar for opening the account.

**Step by step for `such_a_deal = AsSeenOnTVAccount('Jack')`:**

- `balance` is `1` as an instance attribute, set by `AsSeenOnTVAccount.__init__` (which overrides `Account.__init__`).
- `such_a_deal.deposit(20)`: `deposit` is not an instance attribute and not in `AsSeenOnTVAccount`. Search the bases left to right: `CheckingAccount` has no `deposit`, `SavingsAccount` does. So `SavingsAccount.deposit` runs: `Account.deposit(self, 20 - 2)` adds 18 to the balance of 1, giving **19**.
- `such_a_deal.withdraw(5)`: found in `CheckingAccount` (the first base searched), so `Account.withdraw(self, 5 + 1)` runs: 19 minus 6 is **13**.
- `such_a_deal.interest`: not in the instance, not in `AsSeenOnTVAccount`, found in `CheckingAccount` as **0.01**.

The lesson: several classes in the chain define `withdraw` and `deposit`, but the order is unambiguous, and the only rule you need is that a subclass is searched before its base classes.

### Example 6: Three attributes

```python
class A:
    x, y, z = 0, 1, 2

    def f(self):
        return [self.x, self.y, self.z]

class B(A):
    x = 6
    def __init__(self):
        self.z = 'A'
```

```
>>> A().f()
[0, 1, 2]
>>> B().f()
[6, 1, 'A']
```

**Step by step.** The `A` class object has class attributes `x: 0`, `y: 1`, `z: 2`, and `f`. The `B` class object has base class `A` and one class attribute `x: 6`, plus `__init__`. Note that `y` and `z` are **not copied** into `B`.

`A()` creates a blank instance (`A` has no `__init__`, so no instance attributes). `f` is found in `A`, so `self.x`, `self.y`, `self.z` all fall through to `A`'s class attributes: `[0, 1, 2]`.

`B()` creates an instance and runs `B.__init__`, which gives it one instance attribute, `z: 'A'`. Then `f` is looked up: not in the instance, not in `B`, found in `A`. Inside `f`, with `self` bound to the `B` instance:

- `self.x`: no instance `x`; `B` has `x = 6`, so `6` (`B` overrides `A`'s `x = 0`).
- `self.y`: no instance `y`; no `y` in `B`; found in `A` as `1`.
- `self.z`: the instance has `z = 'A'`, so the instance attribute wins over `A.z = 2`.

Result: `[6, 1, 'A']`. All three names are resolved at three different levels of the lookup chain, which is the whole point of the exercise.

### Example 7: Where's Waldo

```python
class Exhibit():
    z = '!'
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def get_name(self):
        return self.x + self.y + self.z

class FerrisWheel(Exhibit):
    z = 'do'
    def __init__(self, x):
        self.x = x
        self.y = 'l'

class Carousel(Exhibit):
    x = 'Wa'
```

Warm-ups:

```
>>> Exhibit('a', 'b').get_name()
'ab!'
>>> FerrisWheel('c').get_name()
'cldo'
>>> Carousel('d', 'e').get_name()
'de!'
```

- `Exhibit('a', 'b')`: instance attrs `x='a'`, `y='b'`; `z` falls through to `Exhibit.z = '!'`.
- `FerrisWheel('c')`: its own `__init__` sets `x='c'`, `y='l'`; `get_name` is inherited from `Exhibit`, but `self.z` finds `FerrisWheel.z = 'do'` before reaching `Exhibit.z`, so `'c' + 'l' + 'do'`.
- `Carousel('d', 'e')`: `Carousel` has no `__init__`, so `Exhibit.__init__` runs and sets instance attrs `x='d'`, `y='e'`; the *instance* `x` shadows the class attribute `Carousel.x = 'Wa'`, and `z` comes from `Exhibit`, giving `'de!'`.

**The puzzle**: write an expression with no quotes and no `+` that evaluates to `'Waldo'`.

```python
>>> FerrisWheel(Carousel.x).get_name()
'Waldo'
```

`Carousel.x` is a lookup on the **class object**, which finds the class attribute `'Wa'` (no instance exists, so nothing shadows it). That becomes the `FerrisWheel`'s `x`; `y` is `'l'`; `z` is `FerrisWheel.z = 'do'`. Concatenating gives `'Waldo'`.

*(extra context: the docstring in the posted file writes the expected output as `Waldo` without quotes, which would actually fail as a doctest since `get_name` returns a string whose repr includes quotes. The value is the string `'Waldo'`.)*

### Example 8: The attribute-lookup stress test (`A`, `B`, `C`)

Reconstructed from the lecture's walkthrough (the slide diagram is not in the extracted text):

```python
class A:
    z = -1
    def f(self, x):
        return B(x - 1)

class B(A):
    n = 4
    def __init__(self, y):
        if y:
            self.z = self.f(y)
        else:
            self.z = C(y + 1)

class C(B):
    def f(self, x):
        return x

a = A()
b = B(1)
b.n = 5
```

**What the three `class` statements build.** A class is an ordinary value that gets bound to a name in the global frame.

- `A`: class attributes `z: -1` and `f` (the function returning `B(x - 1)`). All the functions here have parent frame global.
- `B`: base class `A`, class attributes `n: 4` and `__init__`. `B` "forever knows" that `A` is its base class.
- `C`: base class `B`, one class attribute `f` (the identity-style function returning `x`), which **overrides** `A.f` for instances of `C`. Since `B` inherits from `A`, a `C` instance can reach everything in `A` except names that are overridden on the way.

**What the three assignment statements build.**

- `a = A()`: `A` has no `__init__`, so the new instance is a blank slate with no instance attributes.
- `b = B(1)`: see the trace below. The result is a `B` instance with one instance attribute `z`.
- `b.n = 5`: look up `b` in the current environment, then **add an instance attribute** `n: 5` to that one object. This has no effect on `B` or on `A`; `B.n` is still 4.

**Tracing `B(1)`.** `__init__` runs with `self` bound to the new `B` instance and `y = 1`. `1` is a true value, so `self.z = self.f(1)`. Which `f`? The instance has none, `B` has none, so `A.f` is used: it returns `B(1 - 1)`, that is `B(0)`.

Tracing that nested `B(0)`: `y = 0` is false, so `self.z = C(0 + 1) = C(1)`.

Tracing `C(1)`: `C` has no `__init__`, so lookup goes to `B.__init__` with `y = 1`. True, so `self.z = self.f(1)`, and now `self` is a **`C`** instance, so `f` is found in `C` and returns `1`. So this `C` instance has `z = 1`.

Unwinding: the inner `B(0)` instance has `z` = that `C` instance. The outer instance `b` has `z` = that inner `B` instance.

**The questions.**

```
>>> C(2).n
4
>>> a.z == C.z
True
>>> a.z == b.z
False
```

- `C(2).n`: constructing the `C` instance runs `B.__init__` with `y = 2`, which sets `self.z = self.f(2)`, and `C.f` returns `2`, so that instance has `z = 2`. But the instance is only used to look up `n`: not an instance attribute, not in `C`, found in `B` as **`4`**. (`b.n = 5` changed one object, not the class. And `C(2).z` would have been `2`.)
- `a.z == C.z`: `a` has no instance `z`, so lookup finds `A.z = -1`. `C.z` is a lookup on the class `C`: no `z` in `C`, none in `B`, found in `A` as `-1`. Same attribute reached two very different ways, so `True`.
- `a.z == b.z`: `-1` on the left; `b.z` is an instance attribute holding a `B` instance on the right. `False`.

**Which of `b.z`, `b.z.z`, `b.z.z.z`, `b.z.z.z.z` evaluates to an integer?**

- `b.z` is a `B` instance.
- `b.z.z` is a `C` instance.
- `b.z.z.z` is `1`, an integer. ✅
- `b.z.z.z.z` would be `(1).z`, which is an `AttributeError`.

---

## Common Pitfalls

1. **Thinking `self.attr = v` can change a class attribute.** It never does. It creates or overwrites an attribute on the instance, which then *shadows* the class attribute for that object only. Use `ClassName.attr = v` (or `type(self).attr = v`) to change the shared one. In the `Place` problem, `self.last = self` instead of `Place.last = self` breaks everything.
2. **Forgetting that `+=` writes to the instance.** `self.count += 1` reads through the chain but writes an instance attribute, so a shared counter stored as a class attribute will silently stop being shared.
3. **Believing base class attributes are copied into subclasses.** They are not. If you add an attribute to the base class *after* defining the subclass, the subclass sees it immediately, because lookup happens at access time.
4. **Forgetting `self` when calling a base class method through the class.** `Account.withdraw(amount)` is wrong; it must be `Account.withdraw(self, amount)`. Conversely, `super().withdraw(self, amount)` is wrong; `super()` already binds `self`.
5. **Hard-coding a class name where `self` belongs.** `CheckingAccount.withdraw_fee` works but blocks specialization by instance attribute or subclass; prefer `self.withdraw_fee`. The flip side: do *not* write `self.withdraw(...)` where you meant `Account.withdraw(self, ...)`, since inside an overriding method that is infinite recursion.
6. **Using inheritance for a has-a relationship.** A `Bank` that inherits from `Account` is a design error. Ask "is a bank a kind of account?" If not, use an attribute.
7. **Mixing up `obj.attr` and `Class.attr` in puzzles.** `Carousel('d','e').get_name()` is `'de!'` but `Carousel.x` is `'Wa'`: the instance's own `x` shadows the class's. Always ask explicitly: does the instance have this attribute?
8. **Assuming a method defined in a base class uses the base class's other attributes.** `A.f` called on a `B` instance still resolves `self.x` starting from the `B` instance, which is why `B().f()` gives `[6, 1, 'A']`, not `[0, 1, 2]`.
9. **Losing track of which object `self` is during a nested construction.** In the `B`/`C` trace, the same `B.__init__` body resolves `self.f` differently depending on whether `self` is a `B` or a `C` instance. Write down which instance `self` names at each step.
10. **Reading an inherited mutable class attribute and mutating it.** *(extra context, not stated in this lecture)* `self.some_list.append(x)` on an inherited list class attribute mutates the shared object for every instance and subclass, which is usually not what you want. Assignment shadows; mutation does not.

---

## Likely Exam Points

### 1. Fill in a class so that doctests pass, using a class attribute as shared state

This is exactly the `Place` discussion question, and it recurs in some form on nearly every exam.

**Practice.** Fill in the blanks so the doctests pass:

```python
class Counter:
    """
    >>> a, b = Counter(), Counter()
    >>> a.id, b.id
    (0, 1)
    >>> Counter.made
    2
    """
    made = 0
    def __init__(self):
        self.id = ________
        ________ = ________
```

**Answer.** `self.id = Counter.made` (reading `self.made` or `type(self).made` would also work) and `Counter.made = Counter.made + 1`. Writing `self.made = self.made + 1` would give each instance its own copy, leave `Counter.made` at 0, and make every `id` be 0.

### 2. Predict the value of an expression that requires tracing lookup through instance, subclass, and base class

**Practice.** Given:

```python
class P:
    v = 1
    def go(self):
        return self.v

class Q(P):
    v = 2

q = Q()
q.v = 3
```
What are `q.go()`, `Q.v`, `P.v`, and `Q().go()`?

**Answer.** `q.go()` is `3` (instance attribute shadows everything; `go` itself comes from `P`). `Q.v` is `2`, `P.v` is `1` (assigning `q.v` changed only the object `q`). `Q().go()` is `2` (fresh instance has no `v`, so `Q`'s class attribute is found before `P`'s).

### 3. Write a subclass that overrides a method and defers to the base class

**Practice.** Write `MinimumBalanceAccount(Account)` whose `withdraw` refuses any withdrawal that would leave the balance below `self.minimum`, which defaults to 10, and otherwise behaves exactly like `Account.withdraw`. Return `'Insufficient funds'` when refusing.

**Answer.**

```python
class MinimumBalanceAccount(Account):
    minimum = 10

    def withdraw(self, amount):
        if self.balance - amount < self.minimum:
            return 'Insufficient funds'
        return super().withdraw(amount)       # or Account.withdraw(self, amount)
```

Key points graders look for: `self.minimum` (not `MinimumBalanceAccount.minimum` or a hard-coded 10), no copy-pasting of `Account.withdraw`'s body, and either `super().withdraw(amount)` or `Account.withdraw(self, amount)` with `self` passed exactly once.

### 4. `Class.method(instance, ...)` vs `instance.method(...)`

**Practice.** For the lecture's `Place`, are `self.then.print_history()` and `Place.print_history(self.then)` equivalent? What about `Place.print_history()`?

**Answer.** The first two are equivalent: looking the method up on the instance produces a bound method that supplies `self.then` as `self`, while looking it up on the class produces a plain function that needs `self` passed explicitly. `Place.print_history()` is a `TypeError`: the required `self` argument is missing.

### 5. Multiple inheritance lookup order

**Practice.** With `class AsSeenOnTVAccount(CheckingAccount, SavingsAccount)` as in lecture, and `acct = AsSeenOnTVAccount('Jack')`, what are `acct.balance` after `acct.deposit(20)` and then `acct.withdraw(5)`, and what is `acct.interest`?

**Answer.** Start at `1`. `deposit` is found in `SavingsAccount` (`CheckingAccount` has none), so `20 - 2 = 18` is added, giving `19`. `withdraw` is found in `CheckingAccount`, so `5 + 1 = 6` is removed, giving `13`. `interest` is found in `CheckingAccount`, so `0.01`. The rule: instance, then the class, then the base classes left to right, subclass always before base class.

### 6. Inheritance vs composition (short answer / design)

**Practice.** For each pair, say whether inheritance or composition is appropriate: (a) `Dog` and `Animal`; (b) `Car` and `Engine`; (c) `SavingsAccount` and `Account`; (d) `Bank` and `Account`.

**Answer.** (a) inheritance, a dog *is an* animal. (b) composition, a car *has an* engine. (c) inheritance, a savings account *is an* account. (d) composition, a bank *has* accounts (stored as a list attribute).

### 7. Attribute assignment on an instance does not affect the class (and vice versa)

**Practice.** Using the lecture's `A`/`B`/`C` example, after `b = B(1)` and `b.n = 5`, what are `b.n`, `B.n`, and `C(2).n`?

**Answer.** `5`, `4`, and `4`. The assignment added an instance attribute to the single object `b`; `B`'s class attribute `n: 4` is untouched, and any other instance (including a `C` instance, which reaches `n` through `C` to `B`) still sees `4`.

### 8. Chained dot expressions over linked objects

**Practice.** In the `A`/`B`/`C` example, which of `b.z`, `b.z.z`, `b.z.z.z`, `b.z.z.z.z` is an integer, and why is the last one an error?

**Answer.** `b.z.z.z` is `1`. `b.z` is a `B` instance (built by `A.f` as `B(0)`), `b.z.z` is a `C` instance (built in the `else` branch as `C(1)`), and that `C` instance's `z` is `1` because `C.f` is the overriding identity function. `b.z.z.z.z` evaluates `(1).z`, and integers have no attribute `z`, so it raises `AttributeError`.

---

## Summary

- Attribute **lookup** falls back from instance to class to base class to base class's base class; attribute **assignment** on an object always writes an instance attribute and never touches a class.
- To change shared state, assign to the class: `Place.last = self`, not `self.last = self`. Reading is fine either way (`Place.last`, `self.last`, `type(self).last`).
- Methods can read and mutate objects other than `self`, as in `Friend.talk` setting `other.just_talked` and bumping `other.talked_to`.
- `class Sub(Base):` declares that `Sub` **is a** specialized `Base`. The subclass writes down only what differs; everything else is inherited.
- Base class attributes are **not copied** into subclasses. To look up a name in a class: return it if present, otherwise look in the base class, recursively.
- **Overriding** means reusing a name in the subclass. The base version stays reachable: `Account.withdraw(self, amt)` (plain function, pass `self`) or `super().withdraw(amt)` (bound, do not pass `self`).
- Design guidance: don't repeat yourself (defer to the base class instead of copy-pasting), keep overridden attributes reachable through class objects, and look attributes up on the **instance** (`self.withdraw_fee`, `a.interest`) so individual instances and further subclasses can specialize.
- Inheritance models **is-a**; **composition** (holding another object as an attribute) models **has-a**. A `Bank` has a list of accounts, so `Bank` does not inherit from `Account`. Classes are first-class values, so a bank can take a `kind` parameter and call `kind(holder)`.
- **Multiple inheritance** is allowed (`AsSeenOnTVAccount(CheckingAccount, SavingsAccount)` gets `withdraw` from `CheckingAccount`, `deposit` from `SavingsAccount`, `interest` 0.01, and its own `__init__`). The only rule you need: subclasses are searched before base classes, left to right.
- Puzzles like `FerrisWheel(Carousel.x).get_name()` giving `'Waldo'`, `B().f()` giving `[6, 1, 'A']`, and the `A`/`B`/`C` trace all reduce to the same discipline: name the object, ask whether it has the attribute itself, and if not walk up the chain one class at a time.
