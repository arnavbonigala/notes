<!-- Fri, Oct 09, 2026 | sources: slides + code + YouTube auto-transcript -->
# Lecture 19: Classes

## Overview

This lecture introduces object-oriented programming (OOP), the dominant paradigm for organizing large programs today. The core idea builds directly on data abstraction and mutation from earlier lectures, but takes one step further: instead of keeping information in one place and separately writing functions that operate on it, we *bundle* related data together with the behavior that operates on that data. The unit of bundling is the **object**, which has its own **local state** (attributes) and knows how to manage that state through **methods** (functions specific to that kind of object). A **class** statement creates a new user-defined type and describes how all of its **instances** behave. We cover the mechanics: `class` statements, the special `__init__` constructor method, the `self` parameter and how dot notation automatically supplies it, attribute access and assignment with dot expressions, object identity (`is` / `is not`), and the `@dataclass` decorator as shorthand. The running examples are a bank `Account` class (in the slides/transcript) and a `Student` class with `discussion_points`, `mt2_points`, and `a_plus_points` (in the lecture code).

---

## Key Concepts

### Why OOP exists

Programs get large and complex. Humans can only hold so much complexity in mind at once, so we break programs into small, modular components that can be written, tested, and reasoned about individually. Data abstraction already did part of this job: it let us separate *how* data is represented from *how* it is used.

OOP extends this by bundling together:
1. **Multiple pieces of information** (like data abstraction did), and
2. **Related behavior** (new).

So an object representing a date not only stores everything needed to represent that date, it can also *do* date-like things, such as computing what day of the week it falls on.

### The metaphor: a program as interacting objects

Don't think of your program as one long list of data plus a pile of functions that operate on it. Think of it as a collection of objects that interact with each other, where **each object keeps track of its own information**.

Consequences of this view:

- Each object has its own **local state**, which may change over time.
- Mutation is often involved in OOP (though not required). Crucially, when mutation happens, it does not happen in some global list holding all program information. The thing changing **belongs to a particular object**.
- Each object should **know how to manage its own local state**: it keeps track of what changes are allowed.
- You interact with an object through **methods**: functions specific to that kind of object.
- Several objects can be **instances** of a common **class**, which is a type of object. You can have many lists that all behave the same way.
- Different classes may relate to each other.

### Classes, instances, methods

Python (and many other languages) has built-in syntax and vocabulary to support this metaphor:

- A **class** defines how objects of a particular type behave.
- An **object is an instance of a class**. Every object has a type, which is its class, and once an object is created it has that class for all time.
- A **method** is a function called on an object using a dot expression.

The difference between a function and a method: a function is typically defined in the global frame and can be called on any value. A method is specific to a particular object. You can't call `append` on just anything, but you can invoke it on a list `s` because `s` knows how to append:

```python
>>> s = [1, 2]
>>> s.append(5)       # append is the method, invoked on s with argument 5
>>> s
[1, 2, 5]
```

### `list` is a class, not a function

We have been calling `list` as though it were a function, but it is actually a class. Calling it on an iterable creates a new list object:

```python
>>> s = list(range(3))
>>> s
[0, 1, 2]
>>> type(s)
<class 'list'>
>>> type(s) is list
True
```

This is a general truth in Python: **if you have a class and you call it like a function, you create a new instance of that class.**

Note that `type(s)` is not the list itself, it is the *class* that describes all possible behavior related to lists. The `list` class defines everything about how lists work: the methods, how `+` and `*` work on lists, how item assignment and lookup work. It is up to the `list` class to know that the first element is element `0` rather than element `1`.

### A class statement creates a user-defined type

Now we move from using built-in classes to defining our own. A `class` statement creates a new type, and a class **describes the behavior of its instances**.

The driving idea for the `Student` class: all students have a `name` and a midterm score `mt2_points`, so the `Student` class should add those attributes to **each newly created instance**.

A class statement has these parts:
1. The keyword `class`, then the name of the class.
2. A definition of what happens when a new instance is created: the special method `__init__`.
3. Method definitions providing behavior.

### Object construction and `__init__`

When you call a class, two things happen:

1. A **new instance of that class is created**. It knows it is (say) an `Account` or a `Student`, but it is a **blank slate** in terms of attributes. It has no attributes yet.
2. The class's `__init__` method is called with the new object as its **first argument** (always named `self`), along with any additional arguments provided in the call expression.

So for the bank account example from the transcript:

```python
class Account:
    def __init__(self, account_holder):
        self.balance = 0
        self.holder = account_holder

    def deposit(self, amount):
        self.balance = self.balance + amount
        return self.balance

    def withdraw(self, amount):
        if amount > self.balance:
            return 'Insufficient funds'
        self.balance = self.balance - amount
        return self.balance
```

```python
>>> a = Account('John')
```

Step by step: an empty instance is created; `__init__` is called with that instance bound to `self` and `'John'` bound to `account_holder`. Since `self` refers to the new instance, `self.balance = 0` **adds** a `balance` attribute bound to `0` to the instance. Then `self.holder = account_holder` adds a `holder` attribute. Now the instance has two attributes and is ready to be used as an account. One call expression did all of that work.

`__init__` is sometimes called a **constructor method**, vocabulary that is shared with other programming languages.

Two details worth noticing:

- `__init__` is always written with those double underscores on both sides. That is the specific name Python looks for when you create a new instance.
- **The name of the attribute can differ from the parameter name used inside `__init__`.** Above, the parameter is `account_holder` but the attribute is `holder`. (In the `Student` class the names happen to match, which is also fine.)

### Attributes are accessed and modified with dot expressions

```python
>>> a = Account('John')
>>> a.holder
'John'
>>> a.balance
0
>>> a.balance = 12        # attribute assignment statement
>>> a.balance
12
```

Rules from this lecture:

- The value of an existing attribute **can be changed at any time**.
- An attribute **can be assigned a value even if that attribute did not exist before**. New attributes can be added at any time in Python.
- The value of an attribute can be **another object**, not just a number.

The transcript's example of all three at once:

```python
>>> b = Account('Jack')
>>> b.balance = 20
>>> a.backup = b          # brand-new attribute, whose value is another Account
>>> a.backup.balance
20
```

`a.backup` refers to `b`, so we now have two different ways of referring to the same account: `b` or `a.backup`.

A `balance` is an example of **local state**: it is particular to the `a` object, and it is not something you could refer to without a dot expression.

### Object identity

Because objects' attributes can change (objects can mutate), it matters whether you have two distinct objects with the same attributes or two names for the same object.

**Every object that is an instance of a user-defined class has a unique identity.** `a` and `b` are different accounts because `Account` was called twice, and every call to `Account` creates a new instance. There is only one `Account` class but many instances.

The identity operators `is` and `is not` test whether two expressions evaluate to the **same object**:

```python
>>> a is a
True
>>> a is not b
True        # different objects even if balance and holder matched
>>> c = a
>>> c is a
True        # assignment binds a new name; it does not create a new object
```

### Method definition and how `self` gets supplied

Invoking a method means using a dot expression to call one of the methods defined in the class. **All invoked methods have access to the object via the `self` parameter**, so they can all access and manipulate the object's attributes. This is the mechanism by which an object manages its own local state: the class of the object describes how the state changes, for example how the balance gets updated.

The key asymmetry:

```python
    def deposit(self, amount):      # defined with TWO parameters
        self.balance = self.balance + amount
        return self.balance
```
```python
>>> tom_account = Account('Tom')
>>> tom_account.deposit(100)        # called with ONE argument
100
```

**Dot notation automatically supplies the first argument to a method.** It is as if `tom_account` is assigned to `self`. The method is invoked with one argument, but `tom_account` is also passed in and bound to `self`.

Note also: every time you want to refer to `balance` inside a method, you must use a dot expression (`self.balance`), because it is an attribute of the object, not a local variable.

Methods can have arbitrarily complicated behavior: you can write any function you want. Maybe `withdraw` first checks that you have enough money and, if not, returns `'Insufficient funds'` without updating the balance. A method does not need a return statement; without one it returns `None` but can still change attributes.

### Why methods instead of plain functions?

The transcript asks this directly. Answer: **different types of objects might deposit in different ways.** `deposit` as written is specific to the `Account` class. Some other class, like an investment fund, might deposit with different behavior and different code, so you define a separate `deposit` method in that other class. Which one runs depends on the **type of the object** you invoke `deposit` on.

### The `@dataclass` decorator

If all you want is a class whose instances carry a fixed set of named attributes, the `@dataclass` decorator simplifies the class statement by generating `__init__` for you from the annotated attribute names:

```python
from dataclasses import dataclass

@dataclass
class Student:
    name: str
    mt2_points: int
```
```python
>>> eve = Student("Eve", 26)
```

You write the attribute names and their types, and you get a constructor that takes them in order. (In the lecture code this version is named `SimpleStudent` so it does not collide with the full `Student` class.)

But a dataclass alone gives you data, not behavior. A class can also **describe the behavior of its instances**: all students share a `total_points` method and an `assign_mt2_grade` method, which is what the full hand-written `Student` class provides.

---

## Definitions

- **Object-oriented programming (OOP)**: a paradigm for organizing programs in which computation is expressed as a collection of objects that interact with each other, each keeping track of its own information and behavior; the dominant paradigm in use today.
- **Object**: a bundle of data (local state) together with related behavior; represents something in the world.
- **Class**: a type of object; it defines how objects of that particular type behave. A `class` statement creates a new user-defined type.
- **Instance**: an object of a particular class. An object is an instance of a class, and once created it has that class for all time.
- **Attribute**: a named value belonging to an object, accessed and modified using dot expressions (e.g. `a.balance`, `bao.mt2_points`).
- **Attribute assignment statement**: a statement of the form `<object>.<name> = <value>`, which changes an existing attribute or creates a new one.
- **Method**: a function defined in a class statement, called on an object using a dot expression; it is specific to that kind of object.
- **Local state**: the information particular to one object, not referable without a dot expression; may change over time, and the object's class describes how it is allowed to change.
- **`__init__`**: the special method name for the function that constructs an instance. It is called automatically with the newly created (attribute-free) object as its first argument, plus any arguments from the call expression. Also called the **constructor method**.
- **`self`**: the conventional name for the first parameter of a method; it is bound to the instance of the class on which the method was invoked.
- **Dot expression**: an expression of the form `<expression>.<name>`, used to look up an attribute or method of the object that `<expression>` evaluates to.
- **Identity operators `is` / `is not`**: operators that test whether two expressions evaluate to the *same* object (not merely objects with equal attributes).
- **`@dataclass`**: a decorator (from the `dataclasses` module) that simplifies a class statement by generating `__init__` from annotated attribute declarations.

---

## Worked Examples

### Example 1: The `Student` class

```python
class Student:
    def __init__(self, name, in_mega):
        self.name = name
        self.discussion_points = 0
        if in_mega:
            self.discussion_points = 10
        self.mt2_points = 0
        self.a_plus_points = 0

    def total_points(self):
        return self.discussion_points + self.mt2_points

    def assign_mt2_grade(self, points):
        self.mt2_points = min(20, points)
        if points == 26:
            self.a_plus_points += 5
```

Each piece:

- `__init__` is the special method name for the function that constructs a `Student` instance.
- `self` is the instance of the `Student` class on which the method was invoked.
- `total_points` and `assign_mt2_grade` are **methods**, functions defined in the class statement.
- `name`, `discussion_points`, `mt2_points`, `a_plus_points` are **attributes**.

Notice that `__init__` sets `discussion_points` to `0`, then **overwrites** it with `10` if `in_mega` is true. Attribute assignment can change an existing attribute at any time, so this two-step pattern is fine.

Now trace the doctest from the lecture code:

```python
>>> bao = Student('Bao', False)
>>> eve = Student('Eve', False)
```

Calling the class `Student` creates a new, attribute-free instance, then calls `__init__` with that instance bound to `self`, `'Bao'` bound to `name`, and `False` bound to `in_mega`. Inside, four attribute assignments add attributes to the instance: `name = 'Bao'`, `discussion_points = 0` (the `if` body is skipped since `in_mega` is false), `mt2_points = 0`, `a_plus_points = 0`. The second call does the same for Eve. **Two separate calls, so two distinct instances**, each with its own attributes. `bao is not eve`.

In box-and-pointer / environment terms, described in words: the name `bao` in the global frame points to an object tagged with the class `Student`; that object holds its own four name-to-value bindings. The name `eve` points to a *different* `Student` object with its own four bindings. The function values for `__init__`, `total_points`, and `assign_mt2_grade` live with the class, not duplicated inside each instance; both instances share that one class.

```python
>>> bao.mt2_points
0
```
A dot expression looks up the `mt2_points` attribute on the `bao` instance.

```python
>>> bao.total_points()
0
```
`bao.total_points` is a method lookup; calling it with **no explicit arguments** still binds `self` to `bao`, because dot notation automatically supplies the first argument. Inside, `self.discussion_points + self.mt2_points` evaluates to `0 + 0`.

```python
>>> bao.assign_mt2_grade(26)
```
`self` is bound to `bao`, `points` to `26`. First `self.mt2_points = min(20, 26)` sets `bao.mt2_points` to `20` (the cap). Then since `points == 26` is true, `self.a_plus_points += 5` makes `bao.a_plus_points` become `5`. There is no return statement, so the call returns `None` and the interpreter prints nothing, but the object **was mutated**.

```python
>>> bao.mt2_points
20
>>> eve.mt2_points
0
```
This pair is the whole point about local state and identity: mutating `bao` leaves `eve` untouched, because `mt2_points` is local to each instance. (The slide also shows `bao.total_points()` returning `20` after the grade assignment, since `discussion_points` is still `0`.)

(extra context) The slide's second panel shows `>>> bao = Student('Bao', in_mega=False)`, which is the same call written with a keyword argument rather than positionally.

### Example 2: Discussion question, midterm retakes

Fill in `retake_mt2`, which should replace the MT2 grade with `points` if `points` is higher than the existing MT2 grade, capping `mt2_points` at 20 even if `points` is higher.

```python
    def retake_mt2(self, points):
        """ Replaces MT2 grade with points, if points is higher than the existing MT2 grade.

        Caps mt2_grade at 20, even if points is higher.

        >>> s = Student('Bao', False)
        >>> s.assign_mt2_grade(8)
        >>> s.mt2_points
        8
        >>> s.retake_mt2(26)
        >>> s.mt2_points
        20
        """
        if points > self.mt2_points:
            self.mt2_points = min(points, 20)
```

Reasoning:

- The condition must compare the new score against the **current stored attribute**, so it needs a dot expression: `points > self.mt2_points`. Writing `points > mt2_points` would be a `NameError`, since `mt2_points` is not a local variable.
- The body must both apply the cap and store the result back onto the instance: `self.mt2_points = min(points, 20)`.
- Why `min` inside the `if` rather than just `self.mt2_points = points`? Because the spec caps at 20. With `s.mt2_points == 8` and `points == 26`, the condition `26 > 8` is true and the assignment stores `min(26, 20) == 20`.
- Note the ordering matters for correctness in the other direction too: calling `s.retake_mt2(5)` when `s.mt2_points` is `8` leaves the attribute at `8`, since `5 > 8` is false and nothing is assigned.
- A subtle point: `retake_mt2` does **not** award `a_plus_points`, unlike `assign_mt2_grade`. The two methods encode different policies even though they touch the same attribute. That is exactly "the object's class describes how its state is allowed to change."

### Example 3: Discussion question, creating many students

Write `create_students(regular, mega)`, taking a list of names in regular discussion and a list of names in mega discussion, returning a dictionary mapping each name to a new `Student` with that name.

```python
def create_students(regular: list[str], mega: list[str]) -> dict[str, Student]:
    """
    >>> s = create_students(["Daniel", "Darren", "Peyton"], ["Terry", "Tejas"])
    >>> s["Darren"].discussion_points
    0
    >>> s["Terry"].discussion_points
    10
    """
    result = {name: Student(name, False) for name in regular}
    for name in mega:
        result[name] = Student(name, True)
    return result
```

Step by step:

1. The dictionary comprehension iterates over `regular` and, for each name, **calls the class** `Student(name, False)`. Each call creates a separate instance, so the three regular students are three distinct objects with `discussion_points == 0`.
2. The `for` loop adds one new key-value pair per mega student, each with `in_mega=True`, so `__init__` runs the `if` branch and sets `discussion_points = 10`.
3. `return result` returns the dictionary.

Reading the doctest with the type annotation in mind: `s` is a **dictionary**, `s["Darren"]` is a **`Student` instance**, and `s["Darren"].discussion_points` is a dot expression looking up an attribute on that instance. Chained subscript-then-dot is the thing to get comfortable reading here.

In box-and-pointer terms: one dictionary object holds five keys; each value slot points to a distinct `Student` object. No two keys share a `Student`, because `Student(...)` was called five separate times.

The lecture code also notes an alternative one-liner:

```python
    return {name: Student(name, name not in regular) for name in regular + mega}
```

This concatenates both lists and decides the `in_mega` flag by testing membership in `regular`. (extra context) These two versions differ if a name appears in both lists: the original overwrites with `in_mega=True`, while the alternative produces `in_mega=False` for a name present in `regular`.

### Example 4: The `@dataclass` version

```python
from __future__ import annotations
from dataclasses import dataclass

@dataclass
class SimpleStudent:
    name: str
    mt2_points: int
```
```python
>>> eve = SimpleStudent("Eve", 26)
>>> eve.name
'Eve'
>>> eve.mt2_points
26
```

No `__init__` was written. The decorator generated one from the two annotated attribute declarations, in the order they appear. Note the contrast with the hand-written `Student`: `SimpleStudent` takes `mt2_points` directly as a constructor argument with no cap and no `discussion_points`/`a_plus_points`, and it has no methods, so it carries data but no behavior.

### Example 5: Identity vs. equal-looking objects

```python
>>> a = Account('John')
>>> b = Account('John')
>>> a is b
False
>>> c = a
>>> c is a
True
>>> c.deposit(100)
100
>>> a.balance
100
```

`a` and `b` were produced by two separate calls, so they are two instances with unique identities even with identical attribute values. `c = a` binds a new name to the *same* object, so depositing through `c` changes what `a.balance` reports.

---

## Common Pitfalls

- **Forgetting `self` in method definitions.** `def total_points():` then `bao.total_points()` raises a `TypeError` about taking 0 positional arguments but 1 being given, because dot notation supplies the instance.
- **Forgetting the dot when referring to an attribute inside a method.** Writing `return discussion_points + mt2_points` or `if points > mt2_points:` gives a `NameError`. Attributes are only reachable through a dot expression on `self`.
- **Passing `self` explicitly at the call site.** Write `bao.assign_mt2_grade(26)`, not `bao.assign_mt2_grade(bao, 26)`. The method is defined with two parameters but invoked with one argument.
- **Confusing the class with an instance.** `Student` is one class; each call `Student(...)` makes a new instance. `type(bao) is Student`, and `bao` is not `Student`.
- **Assuming mutating one instance affects others.** `bao.assign_mt2_grade(26)` leaves `eve.mt2_points` at `0`. Attributes are local state per object.
- **Expecting a method to return something when it has no return statement.** `assign_mt2_grade` returns `None`; it communicates by mutating. `>>> bao.assign_mt2_grade(26)` prints nothing. Writing `x = bao.assign_mt2_grade(26)` binds `x` to `None`.
- **Confusing "same attributes" with "same object."** Two accounts with equal balances and holders are still different objects; `is` is about identity, not matching contents.
- **Thinking assignment copies an object.** `c = a` creates no new object; it creates a second name for the one object.
- **Misspelling `__init__`** (one underscore, `__int__`, `init`). Then it is never called automatically and your instances have no attributes, producing an `AttributeError` on first use.
- **Writing `min(20, points)` vs. forgetting the cap.** In `assign_mt2_grade` and `retake_mt2`, the cap must be applied at the moment of assignment, otherwise `mt2_points` can exceed 20.
- **Returning the comprehension too early in `create_students`.** The mega students must be added before returning; a common slip is to `return` inside the loop, which yields only part of the dictionary.
- **Assuming attributes must be numbers or strings.** An attribute's value can be another object, including another instance of the same class (`a.backup = b`).

---

## Likely Exam Points

### 1. Tracing attribute mutation across two instances

**Q.** Given the `Student` class above:
```python
>>> a = Student('A', True)
>>> b = Student('B', False)
>>> a.assign_mt2_grade(26)
>>> b.assign_mt2_grade(18)
>>> a.total_points()
>>> b.total_points()
>>> a.a_plus_points
>>> b.a_plus_points
```
What are the four displayed values?

**A.** `30`, `18`, `5`, `0`.
`a` is in mega, so `discussion_points = 10`; `assign_mt2_grade(26)` sets `mt2_points = min(20, 26) = 20` and, since `points == 26`, bumps `a_plus_points` to `5`. So `a.total_points()` is `10 + 20 = 30`. `b` has `discussion_points = 0` and `mt2_points = min(20, 18) = 18`, so `18`; `18 != 26`, so `b.a_plus_points` stays `0`. Mutating `a` never touched `b`.

### 2. `self` and the argument-count asymmetry

**Q.** Why does `def deposit(self, amount)` take two parameters while `a.deposit(15)` passes one argument? What would `Account.deposit(a, 15)` do?

**A.** Dot notation automatically supplies the object as the first argument, so `a` is bound to `self` and `15` to `amount`. (extra context) `Account.deposit(a, 15)` looks up the plain function on the class and passes both arguments explicitly, with the same effect.

### 3. Fill in a method that reads and updates an attribute

**Q.** Complete `retake_mt2`, which replaces the MT2 grade with `points` only if `points` is higher than the existing grade, capping at 20.

**A.**
```python
        if points > self.mt2_points:
            self.mt2_points = min(points, 20)
```
Both blanks need `self.` because `mt2_points` is an attribute, not a local name. The cap goes inside the assignment so a 26 becomes a stored 20.

### 4. Identity operators

**Q.** What does each line display?
```python
>>> a = Account('John')
>>> b = Account('John')
>>> c = a
>>> a is b
>>> a is c
>>> a.holder == b.holder
```

**A.** `False`, `True`, `True`. Two calls to `Account` make two distinct instances, so `a is b` is `False` even with identical holders. `c = a` binds a new name to the same object, so `a is c` is `True`. The holders are equal strings, so the `==` comparison is `True`, which is exactly why `is` and `==` must not be confused.

### 5. Constructing many objects in a data structure

**Q.** Complete `create_students` so that regular students get `discussion_points == 0` and mega students get `10`.

**A.**
```python
    result = {name: Student(name, False) for name in regular}
    for name in mega:
        result[name] = Student(name, True)
    return result
```
Each `Student(...)` call creates a distinct instance; the dictionary maps names (keys) to those instances (values). Reading `students["Terry"].discussion_points`: subscript the dictionary to get a `Student`, then use a dot expression for its attribute.

### 6. What `__init__` does and when it runs

**Q.** After `a = Account('Alan')`, how many steps occurred and what did each do? Could you add an attribute `a.nickname` that `__init__` never mentions?

**A.** Two steps: (1) a new instance of `Account` is created with **no attributes** (a blank slate that knows it is an `Account`); (2) `__init__` is called with that instance as its first argument (`self`) plus `'Alan'`, and its attribute assignments add `balance` and `holder` to the instance. Yes, `a.nickname = 'Al'` works: in Python an attribute can be assigned even if it did not exist before, and new attributes can be added at any time.

### 7. Methods with no return value

**Q.** What does `print(bao.assign_mt2_grade(26))` display, and did anything change?

**A.** It displays `None`, because `assign_mt2_grade` has no return statement. The object did change: `bao.mt2_points` is now `20` and `bao.a_plus_points` increased by 5. A method without a return statement can still mutate the object.

### 8. `@dataclass` vs. a hand-written class

**Q.** Why can't `@dataclass` replace the full `Student` class here?

**A.** `@dataclass` generates `__init__` from annotated attribute names, which saves writing a constructor, but it does not supply behavior. The hand-written `Student` needs an `in_mega` parameter whose effect is conditional logic (`discussion_points` of 0 or 10), extra attributes not passed in (`a_plus_points`), and methods (`total_points`, `assign_mt2_grade`, `retake_mt2`). A class describes the behavior of its instances, not just their data.

### 9. Attributes whose values are objects

**Q.** What is displayed?
```python
>>> a, b = Account('John'), Account('Jack')
>>> b.balance = 20
>>> a.backup = b
>>> a.backup.balance
>>> a.backup is b
```

**A.** `20` and `True`. `backup` is a brand-new attribute whose value is another `Account` object, so `a.backup` and `b` are two different ways of referring to the same account. `a.backup.balance` chains two dot expressions: look up `backup` on `a` to get the account, then `balance` on that.

---

## Summary

- **OOP** organizes programs as collections of interacting objects, each tracking its own information and behavior; it builds on data abstraction and mutation and is the dominant paradigm today.
- Each object has **local state** that may change over time; mutation belongs to a particular object rather than to a global structure, and each object knows how to manage its own state.
- A **class** defines how objects of a type behave; an **object** is an **instance** of a class and keeps that class for all time; a **method** is a function called on an object with a dot expression and is specific to that kind of object.
- Built-in types are classes too: `list` is a class, and calling a class like a function creates a new instance (`type(list(range(3))) is list`).
- A `class` statement creates a new user-defined type. **Methods are functions defined in a class statement.**
- Calling a class does two things: creates an attribute-free instance, then calls **`__init__`** (the constructor method) with that instance as the first argument (`self`) plus the call's arguments. Attribute names may differ from `__init__` parameter names.
- **`self`** is the instance on which the method was invoked. **Dot notation automatically supplies the first argument**, so a method defined with two parameters is invoked with one.
- Attributes are accessed and modified with **dot expressions**; existing attributes can be changed at any time, new attributes can be added at any time, and an attribute's value may be another object.
- Inside a method, every reference to an attribute needs `self.`; methods may return anything or nothing (returning `None` while still mutating).
- Every instance of a user-defined class has a **unique identity**; `is` and `is not` test for the same object, and assignment (`c = a`) binds a new name without creating a new object.
- Methods rather than plain functions, because different classes can define different behavior under the same method name, and the object's type selects which runs.
- **`@dataclass`** generates `__init__` from annotated attributes, a shorthand for data-only classes; behavior still requires writing methods.
- Lecture examples: `Student` with `__init__`, `total_points`, `assign_mt2_grade`, `retake_mt2` (`if points > self.mt2_points: self.mt2_points = min(points, 20)`), and `create_students` building a `dict[str, Student]` via comprehension plus a loop.
