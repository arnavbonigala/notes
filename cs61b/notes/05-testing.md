<!-- Fri, Sep 4, 2026 | sources: code (no transcript available) -->
# Lecture 5: Testing

## Overview

This lecture is about how you decide, on your own, whether your code is correct, without an autograder. The running example is building a `Sort` class whose `sort(String[] x)` method destructively sorts an array of strings using **selection sort**, and the "new way" of the lecture is that we write the *test first* and the implementation second. Along the way we meet ad hoc testing (hand-rolled loops and print statements), the JUnit + Google Truth libraries (`@Test`, `assertThat(...).isEqualTo(...)`), the decomposition of `sort` into small testable units (`findSmallest`, `swap`, and a recursive `sort(x, k)` helper), and the repeated experience of a test catching a real bug: a `swap` without a temp variable, a missing recursive base case, a `findSmallest` that scans the whole array instead of just the unsorted suffix, and a `findSmallest` that returns a `String` when `swap` needs an `int` index. The chapter ends with testing philosophy: autograders vs. unit tests vs. integration tests, and Test-Driven Development (TDD) as a workflow.

## Key Concepts

### 1. Correctness is your job, not the autograder's

In a class you tend to gain confidence in your code by uploading it and waiting for the autograder's verdict. But an autograder is just code an instructor wrote, and it is "fundamentally not all that different from the code that you are writing." Our autograder is in fact JUnit plus some custom libraries. In the real world nobody hands you a benevolent third-party grader; programmers write their own tests. The lecture's central move is to hand you the judge: you build a thing whose approval you can only win by making the code correct.

The failure mode this is meant to cure has a name in the chapter: **Autograder Driven Development**, where you write everything, fix the compiler errors, submit, get errors, sprinkle in some print statements, change something, submit again, repeat. Your workflow is gated on someone else's server, and you are not really in control of your code.

### 2. Write the test before the code

The workflow demonstrated is:

1. Write `testSort()` first.
2. Write a blank or deliberately wrong `Sort.sort`.
3. Run the test and **confirm it fails**.
4. Now write code until the failure goes away.

Step 3 is not a formality. The chapter has an important cautionary moment: the dummy `findSmallest` returned `x[2]`, and for the chosen test input `{"rawr", "a", "zaza", "newway"}` with `expected = "zaza"`, the test *passed by accident*. A test that passes against a stub is not testing anything. The fix was to change the stub to return `x[3]` so that the test genuinely failed, and only then start implementing. (The chapter notes this accident really happened while recording the lecture video.)

A failing test is a good thing: it converts "make my program correct," which is vague and huge, into a concrete mini-puzzle, "make this specific red message go away." Many programmers find this almost addictive, and IntelliJ leans into it by showing green checkmarks per test.

### 3. `==` does not compare array contents

When you compare two objects, `==` compares "the literal bits in the memory boxes." For arrays, the box holds an **address**, so `input == expected` asks whether the two variables point at the *same array object*, not whether they hold equal elements. Two distinct arrays with identical contents will fail `==`.

This is exactly the box-and-pointer picture from earlier lectures: `String[] input` is a single 64-bit box holding an address; the array of four address-boxes lives elsewhere on the heap; each of those points at a `String` object. `==` on the top-level variables only inspects that one outer box.

So an ad hoc test must loop element by element (using `.equals` for the `String`s inside), or use `java.util.Arrays.equals`, or, best, hand the whole problem to a library.

### 4. Ad hoc testing, and why it does not scale

The first `testSort` builds an input, builds an `expected`, calls `Sort.sort(input)`, then loops comparing `input[i]` to `expected[i]` and prints the first mismatch before breaking. It works. It is also tedious: every new test means another loop, another print statement, another `break`, and a manual decision in `main` about which test to run.

There is also a real structural problem with ad hoc tests driven from `main`. If `main` calls `testSort(); testFindSmallest(); testSwap();` and `testSort` throws, the whole `main` terminates and `testSwap` never runs at all. That is why the chapter's intermediate versions keep editing `main` to call exactly one test. JUnit exists to fix this: each `@Test` method is run independently, and one failure does not suppress the others.

### 5. JUnit + Google Truth

Two libraries, two jobs:

- **JUnit** provides the `@Test` annotation and the runner. Marking a method `@Test` (and, per the chapter, making it **non-static**) makes IntelliJ show green run arrows: a single arrow to run that one test, a double arrow to run all tests in the class. The chapter is candid that the reason non-static is required is "unclear, though this probably has to do with things happening behind the scene." (Extra context: JUnit constructs a fresh instance of the test class for each test method, which is why test methods are instance methods.)
- **Google Truth** provides the assertion syntax, imported statically:

```java
import static com.google.common.truth.Truth.assertThat;
...
assertThat(actual).isEqualTo(expected);
```

Truth knows how to compare arrays element-wise and reports the first differing element, e.g. `arrays first differed at element [0]; expected:<[an]> but was:<[i]>`. That one line replaces the entire hand-written loop-and-print.

Note the argument order convention: **actual first, expected inside `isEqualTo`**. The lecture's own `TestSort.java` follows `assertThat(input).isEqualTo(expected)`. (The chapter's `testSwap` at one point writes `assertThat(expected).isEqualTo(input)`, which still detects the bug but reverses the roles and so labels the two values backwards in the failure message.)

The lecture code's imports are worth memorizing:

```java
import org.junit.jupiter.api.Test;                      // JUnit 5
import static com.google.common.truth.Truth.assertThat; // Truth
```

### 6. Selection sort

Three steps, stated recursively:

- Find the smallest item.
- Move it to the front.
- Selection sort the remaining N-1 items, without touching the front item.

"Move it to the front" has two implementations. You could insert at the front and slide everything over: `{6, 3, 7, 2, 8, 1}` becomes `{1, 6, 3, 7, 2, 8}`. Much more efficient is to **swap** the smallest with whatever is currently at the front: `{1, 3, 7, 2, 8, 6}`. The lecture uses swapping.

Full trace of `{6, 3, 7, 2, 8, 1}`:

| Step | Array | Smallest in suffix | Action |
|---|---|---|---|
| start | `{6, 3, 7, 2, 8, 1}` | `1` at index 5 | swap 0 and 5 |
| 1 | `{1, 3, 7, 2, 8, 6}` | `2` at index 3 | swap 1 and 3 |
| 2 | `{1, 2, 7, 3, 8, 6}` | `3` at index 3 | swap 2 and 3 |
| 3 | `{1, 2, 3, 7, 8, 6}` | `6` at index 5 | swap 3 and 5 |
| 4 | `{1, 2, 3, 6, 8, 7}` | `7` at index 5 | swap 4 and 5 |
| 5 | `{1, 2, 3, 6, 7, 8}` | done | |

The chapter mentions that correctness can be proved formally using **invariants** (from chapter 2.4), but does not do so.

### 7. Java has no sub-array references, hence recursive helper methods with an index parameter

The natural recursive call is "sort everything after the front," which in Python you would write as a slice, `sort(x[1:])`. Java has no such thing: "there is no such thing in Java as a reference to a sub-array." You cannot pass the address of the middle of an array.

The standard fix, and a pattern you will use constantly in 61B, is a **private helper method with an extra parameter that delineates the region of interest**:

```java
public static void sort(String[] x) {   // the public entry point
    sort(x, 0);                         // set up the initial call
}

public static void sort(String[] x, int k) {  // the recursive workhorse
    ...
}
```

These are **overloaded** methods: same name, different parameter lists, which Java resolves by the argument types at each call site. The public one exists to give callers a clean interface and to establish the correct starting value of `k`. The chapter calls this "quite common when trying to use recursion on a data structure that is not inherently recursive, e.g. arrays."

### 8. Strings compare with `compareTo`, not `<`

Writing `if (x[i] < smallest)` on `String`s yields the compile error `< cannot be applied to 'java.lang.String'`. Java's relational operators only work on primitives. The chapter models the realistic response: search the web ("less than strings Java"), find the Stack Overflow answer, and learn that `str1.compareTo(str2)` returns

- a negative number if `str1 < str2`,
- `0` if they are equal,
- a positive number if `str1 > str2`.

For a course, cite the source with a `@source` tag in the Javadoc (the lecture code has `// @source https://stackoverflow.com/questions/5153496`). The chapter notes this citation practice "is not a typical real world practice," it is a course rule.

### 9. Unit testing reduces cognitive load

The sharpest argument in the chapter: while writing `sort`, we discovered a bug in `findSmallest`. Because `testFindSmallest` already existed, we could **switch contexts**, fix and re-verify `findSmallest` in isolation, and switch back, instead of repeatedly calling `sort` and trying to infer from the overall output whether `findSmallest` was to blame.

The analogy: you could test a parachute ripcord by taking off, jumping out, and pulling it. Or you could just pull it on the ground.

Tests also make **refactoring** safe. If you rewrite `findSmallest` to be faster or more readable, the tests tell you whether you broke it.

### 10. Testing philosophy: three tools

**Autograder.** Pros: verifies correctness for you, saves the tedious non-instructive work of writing all your own tests, gamifies assessment with points. Cons: can backfire when students chase points that affect neither grade nor learning; does not exist in the real world; builds bad habits; your workflow is hindered by sporadic upload-and-wait cycles.

**JUnit unit tests.** A **unit** is a piece of your program, often a single method. Testing each unit gives you confidence in the pieces so you can depend on them, isolates debugging attention to one unit at a time, and **forces you to clarify what each unit is supposed to do**. Downsides: thorough tests take time; incomplete unit tests give false confidence; and it is hard to test units that depend on other units (the chapter's example: `addFirst` in your `LinkedListDeque`).

**Integration testing.** Verifies that components interact properly together, one level of abstraction above unit testing. JUnit can do this too. Downsides: tedious to do manually, challenging to automate, and at a high level of abstraction it is easy to miss subtle or rare errors.

**Test-Driven Development (TDD):**

1. Identify a new feature.
2. Write a unit test for that feature.
3. Run the test. It should fail.
4. Write code that passes the test.
5. Optional: refactor, now with tests as a safety net.

The chapter's verdict: TDD is **not required** in 61B and may not be your style, but unit testing in general is definitely a good idea. Summary slogan: **write tests, but only when they might be useful.**

## Definitions

- **Destructive method**: a method that modifies its argument in place rather than returning a new object. `Sort.sort(String[] x)` is destructive: it returns `void` and the caller observes the change through their own reference to the same array.
- **Ad hoc test**: a hand-written test using ordinary loops, conditionals, and print statements, with no testing framework.
- **Unit**: a single piece of a program, typically one method, that can be tested on its own.
- **Unit test**: a test that verifies the behavior of one unit in isolation.
- **Integration test**: a test that verifies that multiple components interact correctly together.
- **JUnit**: the testing framework providing the `@Test` annotation and a runner that executes each test method independently. The course autograder is built on JUnit.
- **`@Test`**: a JUnit annotation marking a method as a test. The method must be non-static for IntelliJ's green run arrows to appear.
- **Google Truth**: an assertion library providing the fluent `assertThat(actual).isEqualTo(expected)` syntax, including sensible element-wise comparison and failure messages for arrays.
- **`assertThat(x).isEqualTo(y)`**: passes silently if `x` equals `y`; otherwise throws an `AssertionError` describing the mismatch, which JUnit records as a test failure.
- **Test-Driven Development (TDD)**: a development process in which the unit test for a feature is written, and observed to fail, before the feature's code is written.
- **Autograder Driven Development**: the anti-pattern of writing all code, fixing compiler errors, submitting, and iterating blindly on autograder feedback with print statements.
- **Selection sort**: a sorting algorithm that repeatedly finds the smallest remaining item, swaps it to the front of the unsorted region, and recurses on the rest.
- **`compareTo`**: `str1.compareTo(str2)` returns a negative int if `str1` precedes `str2`, `0` if equal, a positive int if `str1` follows `str2`.
- **Overloading**: defining multiple methods with the same name but different parameter lists in the same class, e.g. `sort(String[])` and `sort(String[], int)`.
- **Private helper method**: a non-public method, often carrying extra bookkeeping parameters (like a start index), used to implement a clean public method.
- **`@source` tag**: a Javadoc/comment annotation citing external help used while writing a method. A 61B course convention, not standard industry practice.

## Worked Examples

### Example 1: the ad hoc test, and why the loop exists

```java
public class TestSort {
    /** Tests the sort method of the Sort class. */
    public static void testSort() {
        String[] input = {"i", "have", "an", "egg"};
        String[] expected = {"an", "egg", "have", "i"};
        Sort.sort(input);
        for (int i = 0; i < input.length; i += 1) {
            if (!input[i].equals(expected[i])) {
                System.out.println("Mismatch in position " + i + ", expected: "
                    + expected + ", but got: " + input[i] + ".");
                break;
            }
        }
    }

    public static void main(String[] args) {
        testSort();
    }
}
```

Step by step:

1. `input` and `expected` are two separate arrays. In box-and-pointer terms: `input` is one box holding an address, `expected` is another box holding a *different* address. Each points at its own four-element array of `String` references.
2. `Sort.sort(input)` passes a **copy of the address** in `input` (Java is always pass-by-value). The copy points at the same array, so the method's writes to `input[i]` are visible to the caller. That is what makes `sort` destructive.
3. The loop compares element by element with `.equals`, not `==`, because the elements are `String` objects.
4. On the first mismatch it prints and `break`s, so you see only the first failure, which is usually the most informative one.

Against an empty `Sort.sort`, this prints `Mismatch in position 0, expected: an, but got: i.` Getting an error is a **good** result: it proves the test can detect a broken implementation.

Two things to notice critically: the `.equals` at the element level is essential, and the printed `expected` in that message is the array *variable*, so it would actually print something like `[Ljava.lang.String;@2f92e0f4` rather than `"an"` (the chapter's shown output is idealized). That sloppiness is itself an argument for using a library.

### Example 2: the same test in Truth

```java
import static com.google.common.truth.Truth.assertThat;

public class TestSort {
   /** Tests the sort method of the Sort class. */
   public static void testSort() {
       String[] input = {"cows", "dwell", "above", "clouds"};
       String[] expected = {"above", "clouds", "cows", "dwell"};
       Sort.sort(input);

       assertThat(input).isEqualTo(expected);
   }
}
```

One line replaces the loop, the `if`, the `println`, and the `break`. `assertThat(input)` wraps the array in a Truth subject; `.isEqualTo(expected)` performs the comparison. Truth compares arrays by contents (not by reference), so this does the right thing where a bare `input == expected` would not. If they differ it throws an `AssertionError` reporting the first differing element.

### Example 3: the lecture's final `TestSort.java`

```java
package lec5_testing;

import org.junit.jupiter.api.Test;
import static com.google.common.truth.Truth.assertThat;

/** Evaluate that Sort.sort and its helper
 *  functions work correctly. */
public class TestSort {
    @Test
    public void testSort() {
        String[] input = {"hello", "whoa", "apple", "hola"};
        String[] expected = {"apple", "hello", "hola", "whoa"};

        // after i call this, input should be sorted
        Sort.sort(input);

        assertThat(input).isEqualTo(expected);
    }

    @Test
    public void testFindSmallest() {
        String[] input = {"hello", "whoa", "apple", "hola"};

        // expected smallest string ALPHABETICALLY
        // because we are sorting ALPHABETICALLY
        int expected = 3;
        int actual = Sort.findSmallest(input, 3);
        assertThat(actual).isEqualTo(expected);
    }

    @Test
    public void testSwap() {
        String[] input = {"hello", "whoa", "apple", "hola"};
        String[] expected = {"hello", "hola", "apple", "whoa"};

        Sort.swap(input, 1, 3);
        assertThat(input).isEqualTo(expected);
    }
}
```

Notes on each piece:

- **No `main` method.** JUnit's runner finds the `@Test` methods. Each runs independently, so a failure in `testSort` does not prevent `testSwap` from running. This is precisely the problem the ad hoc version had.
- Every test method is `public void` and **non-static**, which is what makes the green arrows appear.
- Each test follows the same three-part shape: set up `input` and `expected`, perform the action, assert.
- `testFindSmallest` calls `findSmallest(input, 3)`. Starting at index 3, the only candidate is `"hola"` itself, so the answer is index 3. This is a legitimate boundary case (the one-element suffix) but note it is a weak test on its own: it would also pass for an implementation that just returns `startingIndex`. Stronger companions would be `findSmallest(input, 0) == 2` (`"apple"`) and `findSmallest(input, 1) == 2`.
- `testSwap` swaps indices 1 and 3: `{"hello", "whoa", "apple", "hola"}` becomes `{"hello", "hola", "apple", "whoa"}`. Note `swap` returns `void`; the assertion is on `input`, the mutated array, which is the correct way to test a destructive method.

### Example 4: `findSmallest`, four drafts

**Draft 0 (stub).**

```java
public static String findSmallest(String[] x) {
    return x[2];
}
```

This accidentally passed `testFindSmallest` because the test's expected answer happened to be at index 2. Changing it to `return x[3];` produced a genuine failure, confirming the test actually works.

**Draft 1 (does not compile).**

```java
String smallest = x[0];
for (int i = 0; i < x.length; i += 1) {
    if (x[i] < smallest) { smallest = x[i]; }   // error
}
```

Compile error: `< cannot be applied to 'java.lang.String'`.

**Draft 2 (compiles, returns the wrong *kind* of thing).**

```java
/** Returns the smallest string in x.
  * @source Got help with string compares from https://goo.gl/a7yBU5. */
public static String findSmallest(String[] x) {
    String smallest = x[0];
    for (int i = 0; i < x.length; i += 1) {
        int cmp = x[i].compareTo(smallest);
        if (cmp < 0) { smallest = x[i]; }
    }
    return smallest;
}
```

Correct as written, but when we try to plug it into `sort` we hit a type mismatch:

```java
String smallest = findSmallest(x);
swap(x, 0, smallest);   // swap wants two ints!
```

`swap` needs *indices*, not values. The design lesson: **what a helper returns is determined by what its caller needs**, and you often only discover that when you try to connect the pieces. "Iterating on a design is part of the process of writing code."

**Draft 3 (returns an index).**

```java
public static int findSmallest(String[] x) {
    int smallestIndex = 0;
    for (int i = 0; i < x.length; i += 1) {
        int cmp = x[i].compareTo(x[smallestIndex]);
        if (cmp < 0) { smallestIndex = i; }
    }
    return smallestIndex;
}
```

Tests updated accordingly (`expected` becomes `2`, an `int`). This passes its test, but it will still break `sort`, because it always scans from index 0.

**Draft 4 (final, with `start`).**

```java
public static int findSmallest(String[] input, int startingIndex) {
    int currentSmallest = startingIndex;
    for (int i = startingIndex; i < input.length; i += 1) {
        int cmp = input[i].compareTo(input[currentSmallest]);
        if (cmp < 0) {
            currentSmallest = i;
        }
    }
    return currentSmallest;
}
```

Both the initial value and the loop bound move to `startingIndex`, so only the unsorted suffix is considered. Two details worth internalizing: `currentSmallest` starts at `startingIndex` (not `0`, and not some sentinel), and the comparison re-reads `input[currentSmallest]` each iteration rather than caching the string, which keeps index and value in sync automatically.

### Example 5: `swap`, buggy and fixed

Buggy:

```java
public static void swap(String[] x, int a, int b) {
    x[a] = x[b];
    x[b] = x[a];   // x[a] was already overwritten!
}
```

Trace with `x = {"i", "have", "an", "egg"}`, `a = 0`, `b = 2`:

- `x[0] = x[2]` makes the array `{"an", "have", "an", "egg"}`. The original `"i"` reference has been **lost**: nothing points at it anymore.
- `x[2] = x[0]` reads the *new* `x[0]`, which is `"an"`, so it writes `"an"` back. The array is still `{"an", "have", "an", "egg"}`.

Result: `"i"` is gone and `"an"` is duplicated. The test reports `arrays first differed in element [2]; expected:<[i]> but was:<[an]>`.

In box-and-pointer terms, the boxes `x[0]` and `x[2]` hold addresses. Assignment copies an address into a box and destroys whatever address was there. To exchange two boxes you need a third box.

Fixed (and this is the lecture's final version):

```java
public static void swap(String[] input, int a, int b) {
    String temp = input[a];
    input[a] = input[b];
    input[b] = temp;
}
```

`temp` saves the address in `input[a]` before it is clobbered. Trace: `temp = "i"`, then `x[0] = "an"` giving `{"an", "have", "an", "egg"}`, then `x[2] = temp` giving `{"an", "have", "i", "egg"}`. Correct.

### Example 6: the recursive `sort`, and its two bugs

**Attempt A (Python-style, does not compile).**

```java
public static void sort(String[] x) {
    int smallestIndex = findSmallest(x);
    swap(x, 0, smallestIndex);
    sort(x[1:]);   // no such thing in Java
}
```

**Attempt B (helper with `start`, but no base case).**

```java
private static void sort(String[] x, int start) {
   int smallestIndex = findSmallest(x);
   swap(x, start, smallestIndex);
   sort(x, start + 1);
}
```

Running `testSort` gives `java.lang.ArrayIndexOutOfBoundsException: 4 at Sort.swap(...)`. Debugging shows `start` reaching 4 on a length-4 array. The recursion never stops because there is no base case: it keeps incrementing `start` past the end.

**Attempt C (base case added, but `findSmallest` still scans from 0).**

```java
private static void sort(String[] x, int start) {
   if (start == x.length) { return; }
   int smallestIndex = findSmallest(x);   // still scans the whole array
   swap(x, start, smallestIndex);
   sort(x, start + 1);
}
```

New failure: `arrays first differed at element [0]; expected:<[an]> but was:<[have]>`. Debugging at a **high level of abstraction** (using `Step Over` rather than `Step Into`, so you compare whole function results against expectations, per Lab 3) pinpoints the culprit: when sorting the last 3 of 4 items with `x = {"an", "have", "i", "egg"}` and `start = 1`, `findSmallest` returns index 0 (`"an"`) instead of index 3 (`"egg"`). Then `swap(x, 1, 0)` drags the already-placed `"an"` back out of position.

This is the exact bug the lecture code's inline comment records:

```java
//0: hello, whoa, apple, hola
//1: apple, whoa, hello, hola
//2: apple, hello, whoa, hola
//R2: whoa, apple, hello, hola
```

Lines 0, 1, 2 are the correct behavior at successive levels; `R2` shows the wrong result you get at that level when `findSmallest` is allowed to look back at the already-sorted prefix.

**Final version (the lecture code).**

```java
public class Sort {
    /** Sorts the array of strings destructively. */
    public static void sort(String[] x) {
        sort(x, 0);
    }

    /** Sort x starting from position k, leaving the first k untouched */
    public static void sort(String[] x, int k) {
        if (k >= x.length) {
            return;
        }
        int smallestIndex = findSmallest(x, k);
        swap(x, k, smallestIndex);
        sort(x, k + 1);
    }
}
```

Two differences from the textbook's version are worth noting: the lecture code uses `k >= x.length` rather than `k == x.length` (defensive, and it also makes the empty-array case safe), and the lecture code leaves the helper `public static` rather than `private static` (the textbook recommends `private`).

**Full trace of `Sort.sort({"hello", "whoa", "apple", "hola"})`:**

| Call | Array on entry | `findSmallest(x, k)` | `swap` | Array on exit of that step |
|---|---|---|---|---|
| `sort(x, 0)` | `{hello, whoa, apple, hola}` | index 2 (`apple`) | swap 0,2 | `{apple, whoa, hello, hola}` |
| `sort(x, 1)` | `{apple, whoa, hello, hola}` | index 2 (`hello`) | swap 1,2 | `{apple, hello, whoa, hola}` |
| `sort(x, 2)` | `{apple, hello, whoa, hola}` | index 3 (`hola`) | swap 2,3 | `{apple, hello, hola, whoa}` |
| `sort(x, 3)` | `{apple, hello, hola, whoa}` | index 3 (`whoa`) | swap 3,3 (no-op) | unchanged |
| `sort(x, 4)` | `{apple, hello, hola, whoa}` | base case, returns | | |

Final: `{"apple", "hello", "hola", "whoa"}`, which is exactly `expected` in `testSort`. Alphabetically, `"apple" < "hello" < "hola" < "whoa"`; note `"hello"` before `"hola"` because at index 1 we compare `'e'` against `'o'`.

(Extra context: there are 4 calls that do work plus 1 base case, and the number of comparisons is 4 + 3 + 2 + 1 = 10; in general N(N+1)/2, so selection sort does roughly N²/2 comparisons. Running time analysis is a later lecture, not this one.)

## Common Pitfalls

- **Using `==` to compare arrays or strings.** `==` compares the bits in the boxes, i.e. addresses for objects. Use `.equals`, `java.util.Arrays.equals`, or a Truth assertion.
- **Never watching your test fail.** The `x[2]` stub passed by accident. If you write a test and it is green on the first run against an unimplemented method, you have learned nothing. Break the code deliberately and confirm the test goes red.
- **Swapping without a temp variable.** `x[a] = x[b]; x[b] = x[a];` loses one value and duplicates the other.
- **Forgetting the base case in recursion.** Produces `ArrayIndexOutOfBoundsException` (here) or `StackOverflowError`.
- **Helper methods that ignore the region parameter.** Adding `start` to `sort` but not to `findSmallest` was the subtlest bug in the lecture: every individual piece looked reasonable, but the interface between them was wrong.
- **Returning the wrong kind of value.** `findSmallest` returning a `String` when the caller needs an index. Design the helper's return type around its caller's needs.
- **Trying to slice arrays.** Java has no sub-array reference. Pass an index parameter instead.
- **Comparing `String`s with `<`.** Compile error. Use `compareTo`, and remember it returns an `int`, not a `boolean`, so you must compare it against 0.
- **Running all your ad hoc tests from one `main`.** The first failure kills the rest. Use `@Test`.
- **Making `@Test` methods `static`.** The green arrows will not appear; JUnit needs instance methods.
- **Forgetting the static import** `com.google.common.truth.Truth.assertThat`, or importing `org.junit.Test` (JUnit 4) instead of `org.junit.jupiter.api.Test` (JUnit 5, which the lecture code uses).
- **Asserting on the return value of a destructive method.** `Sort.sort` and `Sort.swap` return `void`. Assert on the mutated array.
- **Weak tests giving false confidence.** `testFindSmallest(input, 3) == 3` passes even for a stub that returns its argument. The chapter explicitly warns that incomplete unit tests give false confidence.
- **Reversing the assertion arguments.** `assertThat(expected).isEqualTo(actual)` still catches bugs but mislabels which value is which in the failure message.

## Likely Exam Points

**1. `==` vs. content equality for arrays**

*Q:* `String[] a = {"x", "y"}; String[] b = {"x", "y"};` What do `a == b` and `java.util.Arrays.equals(a, b)` evaluate to, and why?

*A:* `a == b` is `false`: the two variables hold different addresses, since two separate array objects were created, and `==` compares the bits in the boxes. `Arrays.equals(a, b)` is `true`: it walks the arrays and compares corresponding elements with `.equals`. This is exactly why the ad hoc `testSort` uses a loop instead of `==`.

**2. Trace selection sort**

*Q:* Show the array after each swap when selection-sorting `{"dog", "bee", "cat", "ant"}`.

*A:*
- `sort(x, 0)`: smallest in `[0, 4)` is `"ant"` at index 3; swap 0 and 3 to get `{ant, bee, cat, dog}`.
- `sort(x, 1)`: smallest in `[1, 4)` is `"bee"` at index 1; swap 1 and 1, no change.
- `sort(x, 2)`: smallest in `[2, 4)` is `"cat"` at index 2; swap 2 and 2, no change.
- `sort(x, 3)`: smallest in `[3, 4)` is index 3; swap 3 and 3, no change.
- `sort(x, 4)`: `4 >= 4`, base case, return.

Final: `{ant, bee, cat, dog}`.

**3. Find the bug in `swap`**

*Q:* Trace `swap(x, 0, 1)` on `{"a", "b"}` for the implementation `x[a] = x[b]; x[b] = x[a];`. What is the result and what is the fix?

*A:* `x[0] = x[1]` gives `{"b", "b"}`; the address of `"a"` is lost. `x[1] = x[0]` reads the already-overwritten `x[0]`, which is `"b"`, giving `{"b", "b"}`. The fix is a temporary variable: `String temp = x[a]; x[a] = x[b]; x[b] = temp;`.

**4. Why a private recursive helper with an index?**

*Q:* Why can't `sort(String[] x)` call itself recursively on the rest of the array directly? What is the standard fix?

*A:* Java has no sub-array references, so there is no way to pass "everything from index 1 onward" as an array (no slice notation like Python's `x[1:]`). The standard fix is an overloaded helper `sort(String[] x, int k)` that takes an index delineating the region to consider, with the public `sort(String[] x)` calling `sort(x, 0)` to set up the initial call.

**5. The missing base case**

*Q:* What happens if you remove the `if (k >= x.length) { return; }` from the lecture's `sort(String[], int)`?

*A:* The recursion never terminates on its own. `k` keeps incrementing past the last index, and the first out-of-range access throws an `ArrayIndexOutOfBoundsException` (the chapter shows it thrown from inside `swap`, with value 4 on a length-4 array).

**6. The `findSmallest` region bug**

*Q:* Suppose `sort(String[] x, int k)` is correct but `findSmallest(x)` ignores `k` and always scans from index 0. On `{"hello", "whoa", "apple", "hola"}`, what goes wrong at `k = 1`?

*A:* At `k = 1` the array is `{"apple", "whoa", "hello", "hola"}`. Scanning from 0, `findSmallest` returns index 0 (`"apple"`), so `swap(x, 1, 0)` yields `{"whoa", "apple", "hello", "hola"}`, dragging the already-placed `"apple"` out of position 0. The fix is to give `findSmallest` a `start` parameter and begin both `currentSmallest` and the loop at `start`.

**7. Write a JUnit + Truth test**

*Q:* Write a JUnit 5 test verifying that `Sort.findSmallest` returns index 2 for `{"hello", "whoa", "apple", "hola"}` starting at index 0.

*A:*

```java
import org.junit.jupiter.api.Test;
import static com.google.common.truth.Truth.assertThat;

public class TestSort {
    @Test
    public void testFindSmallestFromZero() {
        String[] input = {"hello", "whoa", "apple", "hola"};
        int expected = 2;
        int actual = Sort.findSmallest(input, 0);
        assertThat(actual).isEqualTo(expected);
    }
}
```

The method must be non-static and annotated `@Test`; `assertThat` takes the actual value, `isEqualTo` the expected.

**8. TDD steps**

*Q:* List the steps of Test-Driven Development, and state whether 61B requires it.

*A:* (1) Identify a new feature. (2) Write a unit test for it. (3) Run the test and see it fail. (4) Write code that passes the test. (5) Optionally refactor, with the tests as a safety net. TDD is **not required** in 61B, though unit testing in general is strongly recommended.

**9. Unit vs. integration testing**

*Q:* Distinguish unit testing from integration testing and give one drawback of each.

*A:* A unit test checks one piece of the program (often a single method) in isolation; an integration test checks that components work correctly together, at a higher level of abstraction. Drawback of unit testing: thorough tests take time and incomplete tests give false confidence; also hard when one unit depends on another. Drawback of integration testing: tedious to do manually, hard to automate, and easy to miss subtle or rare errors at that level of abstraction.

**10. `compareTo` semantics**

*Q:* What does `"apple".compareTo("hola")` return, sign-wise, and why can't we write `"apple" < "hola"`?

*A:* Negative, because `"apple"` precedes `"hola"` alphabetically. Java's `<` only applies to primitives, so applying it to `String` is a compile error: `< cannot be applied to 'java.lang.String'`.

**11. One `main`, many tests**

*Q:* If `main` calls `testSort(); testFindSmallest(); testSwap();` and `testSort` throws an `AssertionError`, what happens to the other two tests? How does JUnit avoid this?

*A:* `main` terminates immediately, so `testFindSmallest` and `testSwap` never run. JUnit runs each `@Test` method independently and reports each result separately, so one failure does not hide the others.

## Summary

- The lecture's "new way": **write the test first**, watch it fail, then write code until it passes.
- A test that passes against a stub is not a test. The `return x[2];` accident proves it; break the code on purpose to verify the test.
- `==` on objects compares addresses, not contents. Compare arrays with a loop plus `.equals`, `java.util.Arrays.equals`, or Truth.
- Ad hoc tests work but are tedious, and running them all from one `main` means the first failure hides the rest.
- JUnit supplies `@Test` (methods must be non-static for IntelliJ's green arrows: single arrow runs one test, double arrow runs all); Google Truth supplies `assertThat(actual).isEqualTo(expected)`.
- Lecture imports: `org.junit.jupiter.api.Test` and `static com.google.common.truth.Truth.assertThat`.
- Selection sort: find the smallest, swap it to the front, recurse on the rest.
- `String` comparison uses `compareTo` (negative / zero / positive), not `<`; cite outside help with `@source`.
- Java has no sub-array references, so recursion over an array uses a helper with an index parameter: public `sort(x)` calls `sort(x, 0)`.
- Final `Sort`: `sort(x, k)` has base case `k >= x.length`, calls `findSmallest(x, k)`, `swap(x, k, smallestIndex)`, then `sort(x, k + 1)`.
- Bugs the tests caught: `swap` without a temp variable, missing base case, and `findSmallest` scanning the whole array instead of the suffix from `k`.
- Debug at a high level of abstraction: `Step Over` more than `Step Into`, comparing whole function results against expectations.
- Unit tests reduce cognitive load (fix `findSmallest` without reasoning through `sort`), localize bugs, clarify each unit's contract, and make refactoring safe. Pull the ripcord on the ground.
- Three correctness tools: autograders (convenient, but unreal and habit-forming), unit tests (confidence per unit, but time-consuming and possibly incomplete), integration tests (catch interaction bugs, but hard to automate).
- TDD is optional in 61B; unit testing is not a bad idea ever. Write tests, but only when they might be useful.
