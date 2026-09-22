<!-- Fri, Sep 04, 2026 | sources: slides + code + YouTube auto-transcript + textbook -->
# Lecture 5: Testing

This lecture introduces "a new way" of knowing your code works: instead of relying on an instructor's autograder, you write your own tests. The vehicle for this is `Sort.sort(String[] x)`, a method that destructively sorts an array of strings using **selection sort**. The lecture deliberately writes tests *before* the code (test-driven development), starting with an ad hoc hand-rolled test (tedious, full of loops and print statements), then replacing it with the **Google Truth** assertion library plus JUnit's `@Test` annotation, which gamifies development with green checkmarks. Along the way Josh intentionally (and once accidentally) introduces bugs: a broken `swap` that loses a value, a `findSmallest` that returns a `String` when `swap` needs an index, a recursive `sort` with a missing base case, and, most instructively, a `findSmallest` that always scans the whole array instead of only the unsorted suffix. Each bug is caught by a unit test and diagnosed with the IntelliJ debugger using the key idea: **find the moment when reality diverges from expectation**. The lecture closes with testing philosophy: autograders vs. unit tests vs. integration testing, and why tests give you stability, confidence in individual units, and the freedom to refactor.

---

## Key Concepts

### 1. How do you know your code works?

In prior classes (61A, Data 88, E7), you knew your code worked because it passed an autograder or instructor-provided local tests. In the real world, no such oracle exists. Programmers (human or LLM) believe their code works because of tests **they wrote themselves**.

Two important caveats stated in lecture:

- Knowing your code is **completely** correct is usually impossible. Formal software verification works in limited circumstances, and for the small data structures in 61B you could in principle write full proofs, but for large systems the specification itself is fuzzy. There are also deep theoretical reasons (the halting problem and related results) why no single tool can verify all programs.
- Tests give **strong evidence**, not proof. Josh's example: Java's built-in binary search had a subtle bug that went unnoticed for roughly 15 years.

### 2. Why ad hoc testing doesn't scale

The natural shape of a test is: build an input, build the expected result, run the method, compare. The comparison is the painful part. You cannot write `input == expected` for arrays, because `==` compares the literal bits in the memory boxes, i.e. whether the two variables hold the same *address*, not whether the arrays hold the same values. So you write a loop, compare element by element with `.equals`, print a mismatch message, and return.

That loop-and-print boilerplate appears in essentially *every* test you'd ever write. Rewriting it by hand for every method is tedious and repetitive, and it's exactly the thing that discourages people from testing at all. We don't want to write boilerplate someone else has already written.

### 3. Unit tests and unit testing frameworks

A **unit test** tests one individual unit of source code (usually a single method) to determine whether it is fit for use. The analogy from lecture: when you build a car, an airplane, or a robot dog, you test each component separately, then assemble tested components into larger units and test those.

Frameworks do the boilerplate for you. Examples named in lecture: **JUnit** (the foundational, longest-standing one), **AssertJ** (more popular), and **Truth** (a Google library). 61B uses Truth, layered on JUnit. Josh's stated reason: at the time he switched away from raw JUnit, Truth had nicer error messages. (The course autograder itself is built on Truth + JUnit + custom `jh61b` libraries.)

The Truth syntax:

```java
import static com.google.common.truth.Truth.assertThat;
...
assertThat(input).isEqualTo(expected);
```

Read it as an English sentence: "assert that `input` is equal to `expected`."

**Convention that matters:** the thing you are *evaluating* (the actual result your code produced) always goes inside `assertThat(...)`. The property you want it to have goes in the chained call (`isEqualTo`, `isTrue`, `isNotNull`, ...). Writing `assertThat(expected).isEqualTo(input)` compiles and technically tests the same thing, but the error message comes out backwards and will confuse you.

### 4. The `@Test` annotation

Adding `@Test` above a method **and making the method non-static** unlocks IntelliJ's test runner:

```java
import org.junit.jupiter.api.Test;
...
@Test
public void testSort() { ... }
```

- A single green arrow next to a method means "run this one test."
- A double green arrow next to the class means "run all tests in this class."
- You no longer need a `main` method in your test class at all.

**What an annotation actually is:** `@Test` does nothing by itself. It is a marker. A separate program (the test runner) uses Java's reflection library to open the class, find every method carrying the `@Test` annotation, and run each one. Pseudocode from the slides:

```java
List<Method> L = getMethodsWithAnnotation(TestSort.class, @Test);
int numTests = L.size();
int numPassed = 0;
for (Method m : L) {
    result r = m.execute();
    if (r.passed == true)  { numPassed += 1; }
    if (r.passed == false) { System.out.println(r.message); }
}
System.out.println(numPassed + "/" + numTests + " passed!");
```

**Why non-static?** In more complex testing setups, `TestSort` may have instance variables that the test methods require (for example, an object constructed fresh before each test). The runner instantiates the test class, so the methods must be instance methods. Josh explicitly marked the deeper reasons as beyond the scope of 61B. (The textbook is even blunter: "The reason why the function has to be non-static is unclear.")

**A crucial practical benefit of `@Test` over a hand-written `main`:** if `main` calls `testSort(); testFindSmallest(); testSwap();` in sequence, the *entire* `main` terminates as soon as `testSort` throws an assertion failure, and the later tests never run. With `@Test`, each test runs independently and you see a full report: which passed, which failed, and how long each took.

### 5. Gamification

With `@Test`, IntelliJ turns debugging into a game: you get concrete mini-goals, progress summarized in the bottom left, and you win when every test has a green check. The textbook frames this psychologically: you have created a judge for your own code whose approval you can only earn by writing it correctly. This is genuinely motivating, and it's the same hook that makes autograders addictive, except now you control it.

### 6. Selection sort

The algorithm, for a list of N items:

1. Find the smallest item.
2. Move it to the front (by **swapping** it with the current front item).
3. Selection sort the remaining N-1 items, without touching the front item.

Why swap rather than insert-and-shift? Shifting everything over requires rewriting all the numbers; swapping touches only two positions. (Efficiency is deferred to a much later lecture.)

Lecture trace on `{6, 3, 7, 2, 8, 1}` (the `*` marks the smallest item in the unsorted region, which is about to be swapped forward):

```
6 3 7 2 8 1*
1 3 7 2* 8 6
1 2 7 3* 8 6
1 2 3 7 8 6*
1 2 3 6 8 7*
1 2 3 6 7 8
```

Correctness of selection sort can be proven formally using **invariants**, a concept the course returns to later. It is not proven here.

### 7. Comparing Strings in Java

`x[i] < smallest` does not compile: "operator `<` cannot be applied to `java.lang.String`." Java does not allow `<` on Strings.

The fix, found via a search engine and Stack Overflow: `str1.compareTo(str2)` returns

- a **negative** number if `str1` is lexicographically less than `str2`,
- **zero** if they are equal,
- a **positive** number if `str1` is greater.

"Lexicographically" means alphabetically (roughly; it compares character codes). So the idiom is:

```java
int cmp = x[i].compareTo(x[smallestIndex]);
if (cmp < 0) { smallestIndex = i; }
```

Cite your source with a `@source` tag in the Javadoc. This is a 61B course convention, not typical real-world practice.

### 8. Getting unstuck: search engines vs. LLMs

Josh's explicit guidance in this lecture:

- When stuck on something easily describable ("how do I compare strings in Java?"), search for it.
- **Do not paste your class code into an LLM and ask "what's wrong?"** This violates course policy, for two reasons: (a) it short-circuits the learning of debugging skills this class is trying to teach, and (b) it makes academic-integrity conversations messy, because an LLM asked to "help" will often quietly rewrite your code.
- Ask **pointed questions** you have some sense of the answer to, rather than "what's wrong."
- LLMs may give you the **wrong level of detail**. Josh's live example: the cheap Gemini result for "less than strings Java" omitted the crucial fact that `compareTo` returns negative/zero/positive. Frontier models (Opus, Fable) gave answers he judged better than the Stack Overflow post. He also retracted an older warning: he used to say LLMs produce subtly buggy or inefficient code, but says frontier models no longer do this at 61B-assignment scale.

### 9. Recursion over arrays: the private helper method with an extra parameter

The natural recursive step is "now selection sort the rest of the array." In Python you'd write `sort(x[1:])`. **Java has no sub-array references**: there is no way to get the address of the middle of an array. Passing `x[1:]` is simply not a thing.

The standard solution: write a **private helper method** with an extra parameter delineating which part of the array to consider.

```java
public static void sort(String[] x) {
    sort(x, 0);                 // sort everything, nothing grayed out
}

/** Sort x starting from position k, leaving the first k untouched. */
private static void sort(String[] x, int k) { ... }
```

The public method keeps its clean signature; the helper does the recursion. Conceptually, `k` is the boundary between the "grayed out, already sorted, don't touch" prefix and the unsorted suffix. This pattern (overloading a method with an index parameter) is extremely common whenever you want recursion on a data structure that isn't inherently recursive, like an array.

### 10. Debugging: find where reality diverges from expectation

The single most emphasized debugging idea in the lecture:

> Don't just step through the code hoping to see something weird. Find the moment when reality diverges from expectation.

The method is the scientific method. Before each step, **write down what you predict the state should be**. Then step and compare. The first step where your prediction fails is where the bug lives, and you now know the bug is in whatever just executed.

Two supporting techniques:

- **Put breakpoints where the action is.** In lecture, Josh chose to break inside `swap`, because that's the elemental operation that transforms the array. A breakpoint at the recursive call would have been less informative.
- **Debug at a higher level of abstraction: prefer "Step Over" to "Step Into."** Stepping over a whole function call and checking whether its result matches your expectation is much faster than crawling through every line. You already have unit tests giving you confidence in the units, so treat them as black boxes until the evidence points at one.

### 11. Testing philosophy: three correctness tools

**Tool #1: The autograder.** Benefits: it verifies correctness for you, saving tedious non-instructive work, and it gamifies the process with points. Downsides: autograders don't exist in the real world, and they build bad habits.

**Autograder Driven Development (ADD)** is the worst way to program in 61B: write the entire program, submit, get a wall of errors, then loop forever over {run autograder, sprinkle in print statements, poke at the code}. This workflow is slow and unsafe, and you're not in control of your workflow or your code. Note the nuance from the slides: **print statements are not inherently evil.** They're a weak tool, but they're very easy to use.

**Tool #2: Unit tests.** You write tests for each unit of your program. Benefits: confidence in each unit, so you can *depend* on them; less debugging time because you can isolate attention to one method; and writing the test forces you to clarify what the unit is supposed to do. The dependency picture from the slides:

```
        testSort
           |
          sort
         /    \
      swap   findSmallest
       |          |
   testSwap  testFindSmallest
```

Downsides: thorough tests take time, incomplete tests can give false confidence, and it's hard to test units that depend on other units (think `addFirst` in your `LinkedListDeque`).

**Test-Driven Development (TDD).** The process:

1. Identify a new feature.
2. Write a unit test for that feature.
3. Run the test. It should fail. (RED)
4. Write code that passes the test. (GREEN) The implementation is now certifiably good.
5. Optional: refactor to make it faster or cleaner, with the tests as a safety net.

TDD is **not required** in 61B, and you might hate it. But unit testing in general is definitely a good idea. TDD is the exact opposite of the autograder-with-print-statements workflow; what's best for you is probably somewhere in the middle.

**Tool #3: Integration testing.** Unit tests verify the pieces; integration tests verify the pieces work *together* (as in Project 0, testing a whole `ArrayDeque` rather than each method in isolation). JUnit can be used for this too. Challenges: tedious to do manually, hard to automate, and at a high level of abstraction it's easy to miss subtle or rare errors. 61B won't have you build full-scale integration tests.

**Summary rule from the textbook:** definitely write tests, but only when they might be useful.

### 12. Why tests help during development

Development is an incremental process with lots of task switching and on-the-fly design modification. Trying to hold everything in your head at once is a recipe for disaster. Tests provide scaffolding:

- **Confidence in basic units.** The parachute analogy: you could test a ripcord by boarding a plane, jumping out, and pulling it. Or you could just pull it on the ground. Don't use `sort` to test `findSmallest`.
- **Regression protection.** Later changes to a basic unit can't silently break it; every piece is under constant inspection, not just the overall program.
- **Focus.** You can context-switch to a suspect method, establish it's correct (or fix it), and switch back.
- **Safe refactoring.** In larger projects (61B Projects 4 and 5), code gets ugly and needs rewriting. Tests let you redesign without fear.

This applies to LLM developers too: tests provide the same stability and scaffolding for machine-written code.

---

## Definitions

- **Unit test:** a software testing method by which individual units of source code (typically a single method) are tested to determine whether they are fit for use.
- **Unit:** one piece of your program, usually a single method, that can be tested in isolation.
- **Ad hoc test:** a test written by hand from scratch, with your own comparison loop and print statements, without a testing framework. Correct but tedious and repetitive.
- **Unit testing framework:** a library that handles the boilerplate of comparing values and reporting failures. Examples: JUnit, AssertJ, Truth.
- **Truth:** a Google assertion library (built over JUnit) used in 61B, with the syntax `assertThat(actual).isEqualTo(expected)`.
- **JUnit:** the foundational Java testing framework; supplies the `@Test` annotation and the test runner. 61B uses JUnit 5 (`org.junit.jupiter.api.Test`).
- **Annotation (`@Test`):** a marker attached to a method that does nothing by itself. A runner uses the reflection library to find all annotated methods and execute them.
- **Reflection:** the Java capability that lets a program inspect the methods/annotations of a class at runtime; it is how the test runner discovers `@Test` methods.
- **Destructive method:** a method that modifies its argument in place rather than returning a new object. `Sort.sort(String[] x)` is destructive: it returns `void` and rearranges `x` itself.
- **`void`:** a return type meaning the method returns nothing.
- **Selection sort:** a sorting algorithm: repeatedly find the smallest item in the unsorted region, swap it to the front of that region, and recurse on the rest.
- **`compareTo`:** `str1.compareTo(str2)` returns a negative number if `str1 < str2` lexicographically, 0 if equal, and a positive number if `str1 > str2`.
- **Lexicographic order:** dictionary/alphabetical ordering of strings.
- **`==` on reference types:** compares the literal bits in the memory boxes, i.e. whether two variables hold the same address, not whether the contents are equal.
- **Private helper method (with index parameter):** an overloaded method taking an extra parameter (e.g. `int start` or `int k`) that delineates which portion of an array to operate on; used to enable recursion over arrays, since Java has no sub-array references.
- **Test-Driven Development (TDD):** a development process where you write a failing unit test for a feature first (RED), then write code to pass it (GREEN), then optionally refactor.
- **Integration testing:** testing that verifies multiple components interact correctly together, one level of abstraction above unit testing.
- **Autograder Driven Development (ADD):** the anti-pattern of writing an entire program, submitting to the autograder, and iterating via print statements and resubmissions.
- **Invariant:** a property that holds at every step of an algorithm; used to prove correctness (mentioned, developed in a later lecture).
- **`@source` tag:** a Javadoc comment tag used in 61B to cite where you got help (e.g. a Stack Overflow URL).

---

## Worked Examples

### Example 1: The ad hoc test (and why it's painful)

```java
public class TestSort {
    /** Tests the sort method of the Sort class. */
    public static void testSort() {
        String[] input    = {"CC", "BB", "DD", "AA"};
        String[] expected = {"AA", "BB", "CC", "DD"};
        Sort.sort(input);

        for (int i = 0; i < input.length; i += 1) {
            if (!input[i].equals(expected[i])) {
                System.out.println("Mismatch at position " + i +
                        ", expected: '" + expected[i] +
                        "', but got '" + input[i] + "'");
                return;
            }
        }
    }

    public static void main(String[] args) {
        testSort();
    }
}
```

**Step by step:**

1. `input` and `expected` are two **separate** array objects. In box-and-pointer terms, `input` is a variable holding the address of one 4-box array; `expected` holds the address of a different 4-box array. Each box holds an address pointing at a `String` object.
2. `Sort.sort(input)` passes a **copy of the address** into `sort` (Java is always pass-by-value; the value copied here is a reference). Because `sort` follows that address to reach the same array object, any rearranging it does is visible to `testSort` afterward. This is exactly what "destructive" means.
3. The loop walks positions `0..length-1` comparing with `.equals` (contents), not `==` (addresses). If you wrote `input == expected` you would compare two different addresses and always get `false`, even for identical contents.
4. On the first mismatch, print and `return`, so you see only the *first* problem.

With a do-nothing `Sort.sort`, this prints something like:

```
Mismatch at position 0, expected: 'AA', but got 'CC'
```

Getting an error here is a **good** thing: it proves your test actually exercises the code. The problem with this style is that the loop-plus-print block is boilerplate you'd rewrite for every single test.

### Example 2: The same test with Truth

```java
import static com.google.common.truth.Truth.assertThat;
import org.junit.jupiter.api.Test;

public class TestSort {
    @Test
    public void testSort() {
        String[] input    = {"hello", "whoa", "apple", "hola"};
        String[] expected = {"apple", "hello", "hola", "whoa"};

        // after I call this, input should be sorted
        Sort.sort(input);

        assertThat(input).isEqualTo(expected);
    }
}
```

**What changed and why it matters:**

- The entire comparison loop collapses to one line. Truth knows how to deep-compare arrays and produces a message naming the first differing element.
- `import static` means you can write `assertThat(...)` instead of `Truth.assertThat(...)`.
- `@Test` + non-static means no `main` method is needed; IntelliJ shows green run arrows and a pass/fail report.
- Running this against a do-nothing `sort` produces a failure reporting expected `[apple, hello, hola, whoa]` but got `[hello, whoa, apple, hola]`.

**Design note from lecture:** the initial in-class input had the alphabetically-first element already in position 0. Josh deliberately changed it so that position 0 must move. A test where nothing has to change isn't exercising the code hard. (Analogy from lecture: asking someone to prove their strength by handing them earbuds.)

### Example 3: `findSmallest`, version 1 (returns a String) and its two bugs

Start with a deliberately wrong stub so the test can fail:

```java
public static String findSmallest(String[] x) {
    return "potato";   // or "cow" in the live demo
}
```

Then the first real attempt:

```java
public static String findSmallest(String[] x) {
    String smallest = x[0];
    for (int i = 0; i < x.length; i += 1) {
        if (x[i] < smallest) {      // COMPILE ERROR
            smallest = x[i];
        }
    }
    return smallest;
}
```

This does not compile: "operator `<` cannot be applied to `java.lang.String`." After searching and finding `compareTo`:

```java
/** Returns the smallest string in x.
  * @source https://stackoverflow.com/questions/5153496 */
public static String findSmallest(String[] x) {
    String smallest = x[0];
    for (int i = 0; i < x.length; i += 1) {
        int cmp = x[i].compareTo(smallest);
        if (cmp < 0) {
            smallest = x[i];
        }
    }
    return smallest;
}
```

**Trace on `{"hello", "whoa", "apple", "hola"}`:**

- `smallest = "hello"`.
- `i = 0`: `"hello".compareTo("hello")` is 0, not `< 0`, no change.
- `i = 1`: `"whoa".compareTo("hello")` is positive (w > h), no change.
- `i = 2`: `"apple".compareTo("hello")` is negative (a < h), so `smallest = "apple"`.
- `i = 3`: `"hola".compareTo("apple")` is positive, no change.
- Returns `"apple"`. Correct.

**The live-demo bug:** Josh first wrote `smallest = input[0];` inside the loop body instead of `smallest = input[i];`. The test caught it immediately. Note that the loop starts at `i = 0` even though `smallest` is initialized to `x[0]`; the wasted first comparison is harmless.

Matching test:

```java
@Test
public void testFindSmallest() {
    String[] input = {"hello", "whoa", "apple", "hola"};
    // expected smallest string ALPHABETICALLY, because we are sorting alphabetically
    String expected = "apple";
    String actual = Sort.findSmallest(input);
    assertThat(actual).isEqualTo(expected);
}
```

The textbook adds a cautionary note: an early stub `return x[2];` accidentally returned the *correct* answer for the chosen input, so the test passed and gave false confidence. Josh says he made exactly this mistake unintentionally while recording. The fix was to change the stub to something definitely wrong so the test would fail as intended.

### Example 4: `swap`, the classic overwrite bug

The naive version:

```java
public static void swap(String[] x, int a, int b) {
    x[a] = x[b];
    x[b] = x[a];
}
```

**Trace on `{"hello", "whoa", "apple", "hola"}` with `a = 1, b = 3`:**

- Initially box 1 holds a reference to `"whoa"`, box 3 holds a reference to `"hola"`.
- `x[1] = x[3]`: box 1 now points at `"hola"`. **The reference to `"whoa"` is gone.** Array is `{"hello", "hola", "apple", "hola"}`.
- `x[3] = x[1]`: box 3 is assigned box 1's current contents, which is `"hola"` again. No change.
- Result: `{"hello", "hola", "apple", "hola"}`. The value `"whoa"` has been destroyed and `"hola"` duplicated.

In the debugger, Josh watched exactly this: after the first line he saw `hola` in both positions and said "so I end up with double Ola."

The fix is a temporary variable:

```java
public static void swap(String[] input, int a, int b) {
    String temp = input[a];
    input[a] = input[b];
    input[b] = temp;
}
```

`temp` holds the reference to `"whoa"` before box `a` is overwritten, so it can be deposited into box `b`. Note that no `String` objects are created, copied, or mutated; only the two boxes' addresses change.

The test:

```java
@Test
public void testSwap() {
    String[] input    = {"hello", "whoa", "apple", "hola"};
    String[] expected = {"hello", "hola", "apple", "whoa"};

    Sort.swap(input, 1, 3);
    assertThat(input).isEqualTo(expected);
}
```

Note that during the live demo Josh initially wrote the wrong `expected` array while distracted, illustrating that **tests themselves can have bugs**. He also noted that the aside `x[a], x[b] = x[b], x[a]` (Python tuple swap) simply does not exist in Java.

### Example 5: Discovering the design error in `findSmallest`

Trying to wire the pieces together:

```java
public static void sort(String[] x) {
    String smallest = findSmallest(x);
    swap(x, 0, smallest);   // ??? does not compile
}
```

The types don't fit. `findSmallest` returns a `String`; `swap` needs two `int` indices. This is a **design/abstraction error**, not a typo: `findSmallest` should have been returning the *index* of the smallest string all along.

The revision (three coordinated edits, shown one at a time in the demo):

```java
public static int findSmallest(String[] x) {
    int smallestIndex = 0;                                  // was String smallest = x[0]
    for (int i = 0; i < x.length; i += 1) {
        int cmp = x[i].compareTo(x[smallestIndex]);         // index into the array now
        if (cmp < 0) {
            smallestIndex = i;                              // store i, not x[i]
        }
    }
    return smallestIndex;
}
```

Because this is a non-trivial change to a building block, the test must be updated too:

```java
@Test
public void testFindSmallest() {
    String[] input = {"hello", "whoa", "apple", "hola"};
    int expected = 2;                       // "apple" lives at index 2
    int actual = Sort.findSmallest(input);
    assertThat(actual).isEqualTo(expected);
}
```

Re-running gives a green check, so you can return to `sort` with confidence that this unit still works. **This is the whole point of unit tests:** you context-switched away to fix `findSmallest`, verified it independently, and switched back, without ever having to reason about whether the overall sort's misbehavior implied something about `findSmallest`.

### Example 6: The recursion, the missing base case, and the second design error

First attempt at recursion (what you'd like to write, but can't):

```java
public static void sort(String[] x) {
    int smallest = findSmallest(x);
    swap(x, 0, smallest);
    // sort(x[1:]);   <- Would be nice, but not possible in Java!
}
```

The helper-method solution:

```java
public static void sort(String[] x) {
    sort(x, 0);
}

/** Sort x starting from position k, leaving the first k untouched. */
private static void sort(String[] x, int k) {
    int smallestIndex = findSmallest(x);   // BUG: ignores k
    swap(x, k, smallestIndex);
    sort(x, k + 1);                        // BUG at first: no base case
}
```

**Bug A: no base case.** The recursion runs forever past the end of the array and throws `ArrayIndexOutOfBoundsException: 4` from inside `swap`. Fix:

```java
if (k >= x.length) {
    return;
}
```

Read it as: "sort these 10 strings starting from position 11" means do nothing.

**Bug B: `findSmallest` scans the whole array.** After fixing the base case, the test still fails. Debugging session from lecture, on input `{"hello", "whoa", "apple", "hola"}`:

| step | reality | expectation |
|---|---|---|
| time 0 | `{hello, whoa, apple, hola}` | (start) |
| after swap 1 | `{apple, whoa, hello, hola}` | `{apple, whoa, hello, hola}` ✓ |
| after swap 2 | `{whoa, apple, hello, hola}` | `{apple, hello, whoa, hola}` ✗ |

The second swap is where **reality diverges from expectation**. Josh set a breakpoint inside `swap` (the elemental operation where the array actually changes), predicted the state before each hit, and stepped. Inspecting the stack frame at the failing moment showed `smallestIndex == 0`, i.e. `findSmallest` selected `"apple"`, which is at index 0 and is supposed to be **grayed out and off-limits**. `findSmallest` always looks at the entire array, never at just the suffix starting at `k`.

The fix, again applied test-first:

```java
@Test
public void testFindSmallest() {
    String[] input = {"hello", "whoa", "apple", "hola"};
    int expected = 3;                                // smallest from index 3 onward is "hola"
    int actual = Sort.findSmallest(input, 3);
    assertThat(actual).isEqualTo(expected);
}
```

```java
// @source https://stackoverflow.com/questions/5153496
public static int findSmallest(String[] input, int startingIndex) {
    int currentSmallest = startingIndex;                 // was 0
    for (int i = startingIndex; i < input.length; i += 1) {   // was i = 0
        int cmp = input[i].compareTo(input[currentSmallest]);
        if (cmp < 0) {
            currentSmallest = i;
        }
    }
    return currentSmallest;
}
```

Both the initialization and the loop start must change. Changing only one gives a subtly wrong method.

### Example 7: The complete, correct code (lecture version)

```java
package lec5_testing;

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

    // @source https://stackoverflow.com/questions/5153496
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

    public static void swap(String[] input, int a, int b) {
        String temp = input[a];
        input[a] = input[b];
        input[b] = temp;
    }
}
```

**Full trace on `{"hello", "whoa", "apple", "hola"}`** (the commented trace in the lecture's own source file):

- `sort(x)` calls `sort(x, 0)`.
- `k = 0`: `findSmallest(x, 0)` returns 2 (`"apple"`). `swap(x, 0, 2)` gives `{apple, whoa, hello, hola}`. Recurse with `k = 1`.
- `k = 1`: `findSmallest(x, 1)` scans indices 1..3 (`whoa, hello, hola`), returns 2 (`"hello"`). `swap(x, 1, 2)` gives `{apple, hello, whoa, hola}`. Recurse with `k = 2`.
- `k = 2`: `findSmallest(x, 2)` scans `whoa, hola`, returns 3 (`"hola"`). `swap(x, 2, 3)` gives `{apple, hello, hola, whoa}`. Recurse with `k = 3`.
- `k = 3`: `findSmallest(x, 3)` scans only `whoa`, returns 3. `swap(x, 3, 3)` is a no-op. Recurse with `k = 4`.
- `k = 4`: `4 >= 4`, return. The recursion unwinds; nothing happens on the way back up (this is tail recursion, all the work is done on the way down).
- Final: `{apple, hello, hola, whoa}`, matching `expected` in `testSort`. Green check.

**Note on the lecture's final `testFindSmallest`:** it tests only `findSmallest(input, 3) == 3`. That's a thin test, and a fuller one would also check `findSmallest(input, 0) == 2`. The textbook version does check two cases. (extra context: it is good practice to include a case where the answer is *not* simply `startingIndex`, since `return startingIndex;` would pass the single-case test.)

---

## Common Pitfalls

1. **Using `==` to compare arrays or Strings.** `==` compares addresses (the literal bits in the memory boxes). Use `.equals`, `java.util.Arrays.equals`, or a Truth assertion.

2. **Writing a stub that accidentally returns the right answer.** `return x[2];` happened to be correct for one input, so the test silently passed and gave false confidence. Always verify that your new test **fails** before you write the implementation. That's step 3 of TDD for a reason.

3. **Backwards `assertThat`.** `assertThat(expected).isEqualTo(input)` compiles but reports failures backwards. The *actual* value goes inside `assertThat`.

4. **Forgetting `@Test`, or leaving the method `static`.** In the live demo, Josh's `testSwap` silently did not run because he forgot the `@Test` annotation. You need `@Test` **and** a non-static method for the runner to pick it up.

5. **Running all tests from `main` in sequence.** The first assertion failure throws and kills the whole `main`; later tests never run. Use `@Test` so each test is run and reported independently.

6. **`x[i] < smallest` on Strings.** Doesn't compile. Use `compareTo` and check the sign of the result.

7. **Storing the value when you meant the index (or vice versa).** `smallest = x[i]` vs. `smallest = i`. Also `input[0]` vs. `input[i]` inside the loop, an error Josh made by accident in lecture.

8. **Trying to slice arrays.** `sort(x[1:])` is Python, not Java. There is no reference to the middle of an array. Use a private helper with an index parameter.

9. **Missing base case in recursion.** Produces `ArrayIndexOutOfBoundsException` from deep inside a helper. Always write the base case first.

10. **Half-fixing a method when you add a parameter.** Adding `int startingIndex` to `findSmallest` requires changing *both* `int currentSmallest = startingIndex` *and* `for (int i = startingIndex; ...)`. Changing one leaves a subtle bug.

11. **Designing helpers in isolation without thinking about how they'll be used.** `findSmallest` was wrong twice for this reason: first returning a `String` when the caller needed an index, then ignoring the sorted prefix. Iterating on design is normal; tests are what make it safe.

12. **Not updating tests after changing a method's contract.** When `findSmallest` changed from returning `String` to returning `int`, both the implementation and `testFindSmallest` had to change together.

13. **Choosing weak test inputs.** An input where element 0 is already in the right place barely exercises `sort`. Pick inputs that force movement.

14. **Aimless stepping in the debugger.** Don't step hoping something looks weird. Predict the state, then check, and find the first divergence. Prefer Step Over to Step Into.

15. **Pasting your assignment code into an LLM asking "what's wrong?"** Against 61B policy, and it robs you of the debugging practice the class is built around.

---

## Likely Exam Points

### 1. Trace selection sort

**Q:** Using the lecture's selection sort, list the array contents after each swap for `{"pear", "fig", "apple", "date"}`.

**A:**
- `k = 0`: `findSmallest(x, 0)` returns 2 (`"apple"`). Swap 0 and 2: `{apple, fig, pear, date}`.
- `k = 1`: `findSmallest(x, 1)` scans `fig, pear, date`, returns 3 (`"date"`). Swap 1 and 3: `{apple, date, pear, fig}`.
- `k = 2`: `findSmallest(x, 2)` scans `pear, fig`, returns 3 (`"fig"`). Swap 2 and 3: `{apple, date, fig, pear}`.
- `k = 3`: `findSmallest(x, 3)` returns 3. Swap 3 and 3, no change.
- `k = 4`: base case, return. Final: `{apple, date, fig, pear}`.

### 2. Why `==` fails for arrays

**Q:** A student writes `assertThat(input == expected).isTrue();` in `testSort` and it fails even though the arrays print identically. Why?

**A:** `==` on reference types compares the literal bits in the memory boxes, i.e. the addresses. `input` and `expected` are two distinct array objects at two distinct addresses, so `input == expected` is `false` regardless of contents. Use `assertThat(input).isEqualTo(expected)`, which deep-compares the contents (or `java.util.Arrays.equals`, or an element-by-element loop with `.equals`).

### 3. The buggy `swap`

**Q:** Given `String[] a = {"x", "y", "z"};` and
```java
public static void swap(String[] x, int i, int j) {
    x[i] = x[j];
    x[j] = x[i];
}
```
what is `a` after `swap(a, 0, 2)`? Fix the method.

**A:** `x[0] = x[2]` makes the array `{"z", "y", "z"}` and destroys the only reference stored in box 0. Then `x[2] = x[0]` assigns `"z"` to box 2, which already holds `"z"`. Result: `{"z", "y", "z"}`. Fix with a temporary variable:
```java
String temp = x[i];
x[i] = x[j];
x[j] = temp;
```

### 4. `compareTo` semantics

**Q:** What does `"apple".compareTo("hello")` return, and what does the sign mean? Why can't we write `"apple" < "hello"`?

**A:** It returns a negative number, because `"apple"` is lexicographically less than `"hello"`. Negative means the receiver is smaller, 0 means equal, positive means the receiver is larger. Java does not define `<` for `String` (it's only for primitive numeric types), so `"apple" < "hello"` is a compile error: "operator `<` cannot be applied to `java.lang.String`."

### 5. Recursion over arrays

**Q:** Why can't `public static void sort(String[] x)` recurse directly on the rest of the array? What's the standard fix?

**A:** Java has no sub-array references: you cannot take the address of the middle of an array, so there's no `x[1:]`. The standard fix is a private overloaded helper with an extra index parameter, `sort(String[] x, int k)`, that sorts only positions `k` and beyond, with the public method calling `sort(x, 0)`. This pattern is used generally for recursion on non-recursive data structures like arrays.

### 6. Find the missing base case

**Q:** The helper below throws `ArrayIndexOutOfBoundsException`. What is missing and why does the exception surface inside `swap`?

```java
private static void sort(String[] x, int k) {
    int smallestIndex = findSmallest(x, k);
    swap(x, k, smallestIndex);
    sort(x, k + 1);
}
```

**A:** There is no base case. The recursion continues with `k` equal to `x.length` and beyond, and `swap(x, k, ...)` indexes past the end of the array, which is where the exception is thrown (the *cause* is in `sort`, the *symptom* is in `swap`). Add at the top:
```java
if (k >= x.length) { return; }
```

### 7. The `findSmallest` scope bug

**Q:** With `findSmallest(String[] x)` that always scans the whole array, `sort({"hello","whoa","apple","hola"})` produces `{"whoa","apple","hello","hola"}`. Explain what goes wrong on the second recursive call.

**A:** After the first swap the array is `{apple, whoa, hello, hola}` and `k = 1`, so `"apple"` is supposed to be frozen. But `findSmallest` scans from index 0 and returns 0 (`"apple"`), so `swap(x, 1, 0)` moves the already-placed `"apple"` back out of position. The fix is to give `findSmallest` a `startingIndex` parameter, initializing `currentSmallest = startingIndex` and looping from `i = startingIndex`.

### 8. What `@Test` does / why non-static

**Q:** What does `@Test` do at runtime, and why must the annotated method be non-static?

**A:** `@Test` is an annotation, a marker that does nothing on its own. A separate test runner uses the reflection library to enumerate the methods of the class, find every method carrying the annotation, and execute each one, collecting pass/fail results. The method must be non-static because in more complex setups the test class may have instance variables (fixtures) that the tests need, so the runner instantiates the class and invokes instance methods. Adding `@Test` also means you don't need a `main`, and each test runs independently, so one failure doesn't prevent the rest from running.

### 9. TDD steps

**Q:** List the steps of Test-Driven Development. Is TDD required in 61B?

**A:** (1) Identify a new feature. (2) Write a unit test for it. (3) Run the test; it should fail (RED). (4) Write code that passes the test (GREEN). (5) Optional: refactor, using the passing tests as a safety net. TDD is **not** required in 61B, and you may not like it, but unit testing in general is definitely a good idea.

### 10. Unit vs. integration testing

**Q:** Distinguish unit testing from integration testing, with one benefit and one drawback of each.

**A:** A unit test exercises one unit of code (usually one method) in isolation. Benefit: confidence in individual pieces and fast, localized debugging. Drawback: thorough tests take time, incomplete tests give false confidence, and units that depend on other units are hard to test. Integration testing verifies that components interact correctly as a whole system (as in Project 0's `ArrayDeque` tests). Benefit: catches interaction bugs that unit tests miss. Drawback: tedious to do manually, hard to automate, and at a high level of abstraction subtle or rare errors are easy to miss.

### 11. Debugging methodology

**Q:** What is the key idea when using a debugger, and where should you set a breakpoint in `sort`?

**A:** Find the moment where **reality diverges from expectation**: before each step, predict the program state, then check. Don't step aimlessly hoping to notice something weird. Also prefer "Step Over" to "Step Into" so you compare whole function results against expectations rather than crawling line by line. In `sort`, set the breakpoint inside `swap`, since that's the elemental operation that actually transforms the array, letting you snapshot the array before and after each exchange.

### 12. Why writing the test first is useful

**Q:** Give two concrete benefits of having `testFindSmallest` while you are in the middle of writing `sort`.

**A:** (1) When `sort` misbehaves, you can context-switch, run `testFindSmallest` alone to establish whether `findSmallest` is correct, and switch back, rather than inferring the behavior of a unit from the behavior of the whole program (the parachute-ripcord analogy: pull it on the ground rather than jumping out of a plane). (2) When you refactor or change `findSmallest`'s contract (e.g. `String` to `int`, or adding a `startingIndex` parameter), the test immediately tells you whether the revised unit still works, so later changes to basic units can't silently break them.

---

## Summary

- **The new way:** you write your own tests. In the real world, no autograder exists; programmers (and LLMs) trust their code because of tests they wrote. Tests give strong evidence, never proof; full correctness is usually impossible to establish.
- **Ad hoc tests** (manual comparison loop + print statements) work but are tedious and repetitive; the same boilerplate appears in every test.
- **Truth** replaces all of it with `assertThat(actual).isEqualTo(expected)` after `import static com.google.common.truth.Truth.assertThat;`. Put the value you're evaluating inside `assertThat`.
- **`@Test` + non-static method** gives green run arrows, removes the need for `main`, runs every test independently (so one failure doesn't hide the rest), and gamifies development: you win when every test has a green check. `@Test` is just a marker; a runner uses reflection to find and execute annotated methods.
- **`==` compares addresses**, not contents. Use `.equals`, `Arrays.equals`, or a Truth assertion for arrays and Strings.
- **Selection sort:** find the smallest item, swap it to the front, recurse on the rest. Trace: `6 3 7 2 8 1` → `1 3 7 2 8 6` → `1 2 7 3 8 6` → `1 2 3 7 8 6` → `1 2 3 6 8 7` → `1 2 3 6 7 8`. Correctness provable via invariants (later lecture).
- **Strings compare with `compareTo`**, not `<`: negative if smaller, 0 if equal, positive if larger. Cite outside help with `@source`.
- **`swap` needs a temp variable**, or the first assignment destroys the value you were trying to save.
- **Java has no sub-array references**, so recursion on arrays uses a private helper with an extra index parameter: `sort(x)` calls `sort(x, 0)`; `sort(x, k)` sorts positions `k` onward and needs a base case `if (k >= x.length) return;`.
- **Three bugs, three lessons:** broken `swap` (caught by unit test + debugger), `findSmallest` returning a `String` when an index was needed (caught by type mismatch at the call site), and `findSmallest` scanning the whole array instead of the suffix (caught by the debugger, by finding where reality diverged from expectation).
- **Debugging method:** predict the state, step, compare, find the first divergence. Break where the action is. Prefer Step Over to Step Into.
- **Philosophy:** autograders are slow, unreal, and encourage Autograder Driven Development; unit tests give confidence in each unit, faster debugging, clearer specs, and safe refactoring; integration tests check that units work together. TDD (RED, GREEN, refactor) is optional in 61B; unit testing is not a bad idea ever. Write tests, but only when they might be useful.
- **Course policy:** do not paste your 61B code into an LLM asking what's wrong. Search for specific facts, ask pointed questions, and read answers carefully; cheap models may omit crucial details (e.g. what `compareTo`'s return value means).
