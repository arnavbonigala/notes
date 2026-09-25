<!-- Fri, Sep 25, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 13: Asymptotics I

This lecture is the pivot point of CS 61B: up to now we have mostly cared about *programming cost* (how long code takes to write, read, and maintain), and from here to the end of the course we care about *execution cost* (how much time and memory a program uses when it runs). The core question is: as the size of the input grows, what happens to the runtime? Rather than trying to compute an exact runtime (which depends on the machine, the compiler, the data, and a hundred other things), we characterize the *order of growth* of the runtime: we ignore low-order terms and ignore multiplicative constants, so that `8N`, `N`, and `N + 500` all collapse to the same answer, "grows like N." The lecture gives a mechanical (and deliberately tedious) recipe (count every operation in terms of N), then shows the shortcut that makes it practical (pick one representative operation as a *cost model* and count only that). Finally, it formalizes "order of growth" as **Big-Theta** (Θ), an "equals"-like statement that pins a function between two constant multiples of a simpler function for large N, and introduces **Big O** as the "less than or equal" variant used for upper bounds. The punchline, stressed on the slides as "extremely important," is that for very large N the highest-order term dominates no matter what the hardware-dependent constants are.

---

## Key Concepts

### Two flavors of efficiency

The slides open with the line "An engineer will do for a dime what any fool will do for a dollar," then split efficiency in two:

- **Programming cost**: how long it takes to *develop* the program, and how easy it is to read, modify, and maintain. The slides emphasize this is more important than you might think, because the majority of software cost is maintenance, not development. This is what the course has focused on so far (interfaces, inheritance, generics, testing) and will revisit later.
- **Execution cost**: how much *time* the program takes to execute and how much *memory* it requires. This is the subject from today until the end of the course.

Asymptotics is the tool for reasoning about execution cost in a machine-independent way.

### "How long does this take?" is the wrong question

Consider the lecture's running example:

```java
public static void countEvens(int[] numbers) {
    int evens = 0;
    for (int i = 0; i < numbers.length; i++) {
        if (numbers[i] % 2 == 0) {
            evens += 1;
        }
    }
    IO.println("Number of evens: " + evens);
}
```

If someone asks "how long does `countEvens` take to run?", any honest answer has to admit:

- Runtime depends on **how fast the computer is**. A 2005 laptop and a 2026 server give wildly different numbers for identical code.
- Runtime depends on the **size of the array**. There is no single number; there is a *function* of the input size.
- The code does some fixed amount of work **for each item in the array**, so the runtime "grows like N," where N is the length of `numbers`.

That last statement is the useful one. It is true on any machine, in any decade, for any array contents. That is what makes order of growth the right abstraction: we throw away exactly the parts of the answer that depend on things we do not control, and keep the part that describes the algorithm.

### Defining order of growth

**Order of growth answers: as N (the size of the input) grows, what happens to the runtime of the algorithm?**

Two simplifications make this precise and usable:

1. **Focus on behavior as N gets large, so ignore low-order terms.** In `countEvens`, the time to initialize `evens` and the time to print the result are *constant*: they do not change when the array gets bigger. As N grows, that fixed overhead becomes negligible compared to the N units of loop work. Terms that grow more slowly get swamped.

2. **Ignore multiplicative constants.** `8N` grows in the same *way* that `N` does: double N and both double. The factor of 8 might come from the loop body doing eight machine operations instead of one, or from a slower CPU, neither of which tells you anything about the algorithm. So `8N`, `N/2`, and `500N` all have order of growth `N`.

Applying both to `countEvens`: **order of growth is N**, which has the concrete consequence that **if N doubles, runtime doubles**.

### Counting operations (the tedious approach)

The mechanical way to find order of growth is to count how many times each operation could execute, in terms of N (where N is `numbers.length`):

| operation | count |
|---|---|
| `evens = 0` | 1 |
| `i = 0` | 1 |
| `i < numbers.length` | N + 1 |
| `i++` | N |
| `% 2` | N |
| equals (`==`) | N |
| `evens += 1` | 0 to N |
| `IO.println` | 1 |

A few things to notice about this table, since each one is a spot where students slip:

- The loop *condition* runs **N + 1** times, not N: it is checked once before each of the N iterations, plus one final time that fails and exits the loop.
- `i++` runs N times (once at the end of each iteration).
- `evens += 1` is a **range, 0 to N**, because it only runs when the current element is even. The count depends on the *contents* of the array, not just its size. An all-odd array gives 0; an all-even array gives N. (This is the seed of "best case vs worst case," which the summary slide gestures at by noting we often, but not always, consider the worst case count.)
- Every entry is either a constant or something that grows like N. The whole table is "a bunch of constants plus a constant number of things proportional to N," which is why the answer is N.

### Cost models: the shortcut

Building that whole table for every program would be unbearable. The fix: **choose a representative operation as your cost model, and use the count of that one operation as the order of growth.** The slides label all of the table's operations "all reasonable cost models" precisely because every one of them (once you strip constants and low-order terms) gives the same answer: N.

Choosing `==` as the cost model for `countEvens`: it runs N times, so **order of growth is N**. One row of the table, same conclusion, a fraction of the work.

The reason this is legitimate is spelled out in the slide the deck flags as "Extremely important point. Make sure you understand it!" Suppose an algorithm's operation counts are:

| operation | count |
|---|---|
| less than (`<`) | 100N² + 3N |
| greater than (`>`) | 2N³ + 1 |
| and (`&&`) | 5,000 |

Let `<` take α nanoseconds, `>` take β nanoseconds, and `&&` take γ nanoseconds on whatever machine you have. Total time is:

```
α(100N² + 3N) + β(2N³ + 1) + 5000γ  nanoseconds
```

For very large N, the `2βN³` term dwarfs all the others **regardless of the values of α, β, and γ**. The hardware constants cannot rescue a cubic term, and cannot sink one either. So the order of growth is **N³**, and you could have gotten there by looking only at the `>` row. Picking the operation that occurs most often (the "representative" one) is enough.

### Why scaling matters

In most real settings we care only about asymptotic behavior, that is, what happens for very large N. The slides list the motivating cases:

- Simulation of billions of interacting particles.
- A social network with billions of users.
- Logging of billions of transactions.
- Encoding of billions of bytes of video data.

Algorithms that scale well (whose runtime curves "look like lines") have better asymptotic behavior than algorithms that scale poorly ("look like parabolas"). The lecture's illustration: suppose `glorp1` takes 2N² operations and `glorp2` takes 500N operations to glorpify N items. For *small* N, `glorp1` is faster (2N² < 500N whenever N < 250). But as the dataset grows, the parabolic algorithm falls farther and farther behind, and the gap keeps widening. The constant 500 buys `glorp2` nothing in the long run; the exponent is what matters.

The deck also shows a table from Kleinberg & Tardos of runtimes for various orders of growth, with the note that the effect is **dramatic**, and often determines whether a problem can be solved *at all* rather than merely how fast it is solved.

### Formalizing: Big-Theta

Given a function Q(N), apply the two simplifications (drop low-order terms, drop multiplicative constants) to get its order of growth. Example: Q(N) = 3N³ + N² has order of growth N³.

The lecture's exercise table, with answers:

| function R(N) | order of growth |
|---|---|
| N³ + 3N⁴ | N⁴ |
| 1/N + N³ | N³ |
| 1/N + 5 | 1 |
| Ne^N + N | Ne^N |
| 40 sin(N) + 4N² | N² |

Worth pausing on three of these:

- **1/N + N³ → N³**: the 1/N term *shrinks* toward 0 as N grows, so it is about as low-order as a term can be.
- **1/N + 5 → 1**: nothing grows here at all. The function tends to the constant 5, and "constant" is written as order of growth **1**, not 5 (constants are dropped).
- **40 sin(N) + 4N² → N²**: sin(N) has range [-1, 1], so `40 sin(N)` is trapped between -40 and 40 forever. A bounded wiggle is low-order compared to N².

**Big-Theta notation** is just a symbol for this. If R(N) has order of growth f(N), we write **R(N) ∈ Θ(f(N))**. So:

- N³ + 3N⁴ ∈ Θ(N⁴)
- 1/N + N³ ∈ Θ(N³)
- 1/N + 5 ∈ Θ(1)
- Ne^N + N ∈ Θ(Ne^N)
- 40 sin(N) + 4N² ∈ Θ(N²)

The formal definition: **R(N) ∈ Θ(f(N))** means there exist positive constants k₁ and k₂ such that

```
k1 * f(N)  <=  R(N)  <=  k2 * f(N)
```

for all values of N greater than some N₀ (i.e. for very large N). In words: R(N) is sandwiched between two constant multiples of f(N), eventually and forever. The "for all N greater than some N₀" clause is what lets us ignore small-N misbehavior (like `glorp1` beating `glorp2` below N = 250, or `40 sin(N) + 4N²` dipping around near the origin).

The lecture's worked instances of the definition:

- **40 sin(N) + 4N² ∈ Θ(N²)** with f(N) = N², k₁ = 3, k₂ = 5, since `3N² <= 40 sin(N) + 4N² <= 5N²` for large enough N (the ±40 wiggle is eventually buried by the N² slack on either side).
- The **countEvens-then-countDuplicates** example: runtime is c₁N + c₂N², and `c2*N² <= c1*N + c2*N² <= (c1 + c2)*N²` for N ≥ 1, giving Θ(N²).
- The challenge problem: R(N) = (4N² + 3N·ln(N)) / 2. Answer: **f(N) = N², k₁ = 1, k₂ = 3**. (R(N) = 2N² + 1.5N·ln(N); the N·ln(N) term grows more slowly than N², so R is eventually between 1·N² and 3·N².)

The lecture is emphatic that this formalism **does not change how you analyze code at all**. You do not go hunting for k₁ and k₂ when analyzing a loop. The only difference is that you write the Θ symbol anywhere you previously wrote "order of growth."

### Big O

Where Big-Theta can informally be thought of as "equals," **Big O can informally be thought of as "less than or equal."** Big O gives an *upper bound* on the order of growth.

All of the following are true simultaneously:

- N³ + 3N⁴ ∈ Θ(N⁴)
- N³ + 3N⁴ ∈ O(N⁴)
- N³ + 3N⁴ ∈ O(N⁶)
- N³ + 3N⁴ ∈ O(N!)
- N³ + 3N⁴ ∈ O(N^(N!))

The formal definition drops the lower bound: **R(N) ∈ O(f(N))** means there exists a positive constant k₂ such that

```
R(N)  <=  k2 * f(N)
```

for all values of N greater than some N₀. Example from the slides: 40 sin(N) + 4N² ∈ O(N⁴) with R(N) = 40 sin(N) + 4N², f(N) = N⁴, and k₂ = 1.

Thinking of these as **families** helps:

| | Informal meaning | Family | Family members |
|---|---|---|---|
| **Big Theta** Θ(f(N)) | Order of growth **is** f(N). | Θ(N²) | N²/2, 2N², N² + 38N + N |
| **Big O** O(f(N)) | Order of growth is **less than or equal to** f(N). | O(N²) | N²/2, 2N², lg(N) |

Note that `lg(N)` belongs to the O(N²) family but *not* the Θ(N²) family: it is bounded above by N² but grows far more slowly, so it fails the lower bound k₁N² ≤ lg(N). Θ is the more informative statement; O is weaker but sometimes all you can honestly say. The slides promise we will see why Big O is practically useful in the upcoming Disjoint Sets lecture.

---

## Definitions

**Programming cost**: The cost of developing and maintaining software: how long it takes to write, and how easy it is to read, modify, and maintain. The majority of software cost is maintenance, not development.

**Execution cost**: The cost of running the program: how much time it takes to execute and how much memory it requires.

**N**: A property of the input to a function, often the size of the input (for `countEvens`, the length of the array `numbers`). All runtime analysis is expressed in terms of N.

**R(N)**: The runtime of a code snippet expressed as a function of N.

**Order of growth**: The answer to "as N grows, what happens to the runtime of the algorithm?", obtained by taking R(N) and (a) ignoring low-order terms and (b) ignoring multiplicative constants. Example: 3N³ + N² has order of growth N³.

**Low-order term**: A term in R(N) that grows more slowly than the dominant term (including constants and terms that shrink, like 1/N, or that stay bounded, like 40 sin(N)). Low-order terms become negligible as N grows and are discarded.

**Multiplicative constant**: A constant factor multiplying a term. Discarded, because 8N grows in the same way that N does.

**Cost model**: A single representative operation chosen to stand in for total work; the count of that operation, as a function of N, is used as the order of growth. Terminology credited to *Algorithms, 4th edition* by Sedgewick and Wayne.

**C(N)**: The count of how many times the chosen representative operation occurs, as a function of N.

**Asymptotic behavior**: The behavior of a function for very large N; what we almost always care about in practice.

**Big-Theta, R(N) ∈ Θ(f(N))**: Means there exist positive constants k₁ and k₂ such that k₁·f(N) ≤ R(N) ≤ k₂·f(N) for all N greater than some N₀. Informally, "the order of growth of R(N) *is* f(N)"; it behaves like "equals."

**Big O, R(N) ∈ O(f(N))**: Means there exists a positive constant k₂ such that R(N) ≤ k₂·f(N) for all N greater than some N₀. Informally, "the order of growth of R(N) is *less than or equal to* f(N)"; an upper bound.

**N₀**: The threshold beyond which the Θ or O inequalities must hold. Its existence is what lets us ignore small-N behavior entirely.

---

## Worked Examples

### Example 1: `countEvens`, order of growth N

```java
public static void countEvens(int[] numbers) {
    int evens = 0;
    for (int i = 0; i < numbers.length; i++) {
        if (numbers[i] % 2 == 0) {
            evens += 1;
        }
    }
    IO.println("Number of evens: " + evens);
}
```

**What it does:** walks the array once, testing each element for evenness with `% 2 == 0`, incrementing a counter when the test passes, and printing the total.

**Step by step.** Let N = `numbers.length`.

1. Set up: `evens = 0` and `i = 0` each happen exactly once. These are constant work, independent of N.
2. The loop condition `i < numbers.length` is evaluated N + 1 times: once before each of the N iterations, and once more (returning false) to terminate.
3. Per iteration: one array access `numbers[i]`, one `% 2`, one `== 0`, one `i++`. Each therefore happens N times in total.
4. The body `evens += 1` executes somewhere between 0 and N times, depending on the array's contents. This is the only operation whose count depends on the *values* in the array rather than just its length.
5. `IO.println` runs once.

**Pointer reasoning in words (extra context: the lecture does not draw boxes here, but connecting to earlier lectures is useful).** `numbers` is a reference variable holding the address of an array object on the heap; passing it to `countEvens` copies the *address*, not the N elements, so the call itself is constant work no matter how large the array is. `numbers[i]` follows that reference and reads the i-th slot, which is a constant-time operation. `numbers.length` reads a field of the array object, also constant time, so the loop condition does not secretly hide a scan.

**Total:** a handful of constants plus a constant number of N-proportional terms, i.e. something of the form c₁N + c₂. Ignore low-order terms (the constant setup and print), ignore multiplicative constants (c₁), and you get **order of growth N**, that is, **Θ(N)**. Concretely: if N doubles, runtime doubles.

**Via cost model:** pick `==` as the representative operation. C(N) = N. C(N) ∈ Θ(N). Done.

---

### Example 2: `count1` vs `count2`, both Θ(N)

```java
public static void count1(int[] numbers) {
    int evens = 0;
    int odds = 0;
    for (int i = 0; i < numbers.length; i++) {
        if (numbers[i] % 2 == 0) {
            evens += 1;
        } else {
            odds += 1;
        }
    }
    IO.println("Number of evens: " + evens);
    IO.println("Number of odds: " + odds);
}
```

```java
public static void count2(int[] numbers) {
    int evens = 0;
    for (int i = 0; i < numbers.length; i++) {
        if (numbers[i] % 2 == 0) {
            evens += 1;
        }
    }
    int odds = 0;
    for (int i = 0; i < numbers.length; i++) {
        if (numbers[i] % 2 != 0) {
            odds += 1;
        }
    }
    IO.println("Number of evens: " + evens);
    IO.println("Number of odds: " + odds);
}
```

**What they do:** both count evens and odds in the array. `count1` does it in a single pass with an if/else; `count2` does it in two separate passes, one per parity.

**Step by step.**

- `count1`: one loop of N iterations. Each iteration does one parity test and exactly one increment (either the `if` branch or the `else` branch fires, never both, never neither). So roughly cN operations.
- `count2`: two loops, each of N iterations. The first does N parity tests; the second does another N parity tests. So roughly 2cN operations, about twice the work of `count1`.

**Both have order of growth N, i.e. Θ(N).** The factor of 2 separating them is exactly the kind of multiplicative constant we throw away. `count2` genuinely does more work, and on a real machine will be measurably slower, but it *scales* identically: doubling N doubles the runtime of both. Asymptotics is a statement about scaling, not about which of two programs wins a stopwatch race at a fixed N.

---

### Example 3: `countDuplicates`, order of growth N²

```java
public static void countDuplicates(int[] a) {
    int duplicates = 0;
    for (int i = 0; i < a.length; i++) {
        for (int j = i + 1; j < a.length; j++) {
            if (a[i] == a[j]) {
                duplicates += 1;
            }
        }
    }
    IO.println("Duplicates: " + duplicates);
}
```

**What it does:** examines every *unordered pair* of distinct positions (i, j) with j > i, and counts how many pairs hold equal values. Starting the inner loop at `j = i + 1` (rather than 0) avoids comparing an element to itself and avoids checking each pair twice.

**The intuitive approach:** ask "how many pairs do we check?" and use `==` as the cost model.

**The triangle picture (reconstructed from the slide's grid, N = 6).** Imagine a 6x6 grid with i as the row and j as the column. A `==` happens at cell (i, j) only when j > i, so the marks fill the strictly-upper-triangular region:

```
        j = 0   1    2    3    4    5
i = 0           ==   ==   ==   ==   ==     (5 comparisons)
i = 1                ==   ==   ==   ==     (4)
i = 2                     ==   ==   ==     (3)
i = 3                          ==   ==     (2)
i = 4                               ==     (1)
i = 5                                      (0)
```

For general N: when i = 0 the inner loop runs N - 1 times, when i = 1 it runs N - 2 times, and so on down to 0.

**Two ways to total it.**

*Geometric (fast and good enough):* the marks form a right triangle of legs about N by N, so the count is about the area of that triangle, **~N² / 2**. Halving is a multiplicative constant, so the **order of growth is N²**.

*Exact (the slide's summation):* the count is

```
(N - 1) + (N - 2) + ... + 3 + 2 + 1  =  N(N - 1) / 2
```

The slide justifies this by the classic pairing argument: lay the sum out in a rectangle of dimensions N by N and observe that the terms fill exactly half of it, giving (N - 1)/2 pairs of terms each summing to N, hence N(N - 1)/2. Expanding: N²/2 - N/2. Drop the low-order -N/2 term and the multiplicative 1/2, and the **order of growth is N², i.e. Θ(N²)**.

**Consequence:** if N doubles, runtime roughly *quadruples*.

---

### Example 4: `countZerps`, Θ(N²) with a mystery helper

```java
public static void countZerps(int[] a) {
    int zerps = 0;
    for (int i = 0; i < a.length; i++) {
        for (int j = i + 1; j < a.length; j++) {
            if (isZerp(a[i], a[j])) {
                zerps += 1;
            }
        }
    }
    IO.println("Zerps: " + zerps);
}
```

**What it does:** structurally identical to `countDuplicates`, except the pair test `a[i] == a[j]` has been replaced by a call to an unknown method `isZerp`.

**Step by step.** The loop structure is unchanged, so `isZerp` is called N(N - 1)/2 times, which is Θ(N²) calls. If each call costs some constant amount of time, the total is (constant) x Θ(N²), and the constant is discarded.

**Order of growth is N², assuming that `isZerp`'s runtime depends only on its two parameters and not on N.** That caveat is the entire lesson of this example. `isZerp` takes two `int`s, so there is nothing of size N inside it to scan, and its cost is a constant. But if a helper's runtime did grow with N (say, it took the array and searched it), you could no longer pull its cost out as a constant, and the answer would change. **You cannot analyze a method that calls a helper without knowing the helper's runtime.**

---

### Example 5: `count`, sequential composition

```java
public static void count(int[] numbers) {
    countEvens(numbers);
    countDuplicates(numbers);
}
```

**What it does:** calls the Θ(N) method, then the Θ(N²) method, on the same array.

**Step by step.** The two calls run one after another, so their costs *add*: R(N) = c₁N + c₂N² for some positive constants c₁ and c₂. Now apply the rules: c₁N is a low-order term next to c₂N², so **drop this low-order term; it doesn't matter as N gets big.** Then drop c₂.

**Order of growth is N², i.e. Θ(N²).**

The slides verify this against the formal Big-Theta definition, with f(N) = N²:

```
c2*N²  <=  c1*N + c2*N²  <=  (c1 + c2)*N²
```

Left inequality: adding the nonnegative c₁N only increases the value. Right inequality: for N ≥ 1 we have N ≤ N², so c₁N + c₂N² ≤ c₁N² + c₂N² = (c₁ + c₂)N². With k₁ = c₂, k₂ = c₁ + c₂, and N₀ = 1, the definition is satisfied.

**General rule this illustrates:** when two code blocks run in sequence, the total order of growth is the *larger* of the two. Only when they are *nested* do the costs multiply.

---

### Example 6: applying the formal definition to 40 sin(N) + 4N²

**Claim:** 40 sin(N) + 4N² ∈ Θ(N²), with k₁ = 3 and k₂ = 5.

**Reasoning.** Because sin(N) has range [-1, 1], the term `40 sin(N)` never leaves the interval [-40, 40] no matter how large N gets. So:

```
4N² - 40  <=  40 sin(N) + 4N²  <=  4N² + 40
```

For the lower bound, we need 3N² ≤ 4N² - 40, i.e. N² ≥ 40, which holds for all N ≥ 7. For the upper bound, we need 4N² + 40 ≤ 5N², i.e. N² ≥ 40 again. So taking N₀ = 7 works, and the definition is satisfied with k₁ = 3, k₂ = 5. (Extra context: the specific value N₀ = 7 is worked out here; the slides only state the constants k₁ = 3 and k₂ = 5 and note the bound holds for large N.)

The same function is also in **O(N⁴)** with k₂ = 1, since 40 sin(N) + 4N² ≤ N⁴ for large N. That is a true but much weaker statement: an upper bound need not be tight.

---

### Example 7: the "extremely important" operation-count argument

Given:

| operation | count |
|---|---|
| less than (`<`) | 100N² + 3N |
| greater than (`>`) | 2N³ + 1 |
| and (`&&`) | 5,000 |

**Step by step.**

1. Assign unknown per-operation costs: α ns for `<`, β ns for `>`, γ ns for `&&`.
2. Total runtime = α(100N² + 3N) + β(2N³ + 1) + 5000γ nanoseconds.
3. Expand and identify the dominant term: 2βN³ + 100αN² + 3αN + (β + 5000γ).
4. As N grows large, 2βN³ overwhelms every other term **regardless of the values of α, β, and γ**, because a cubic eventually beats any constant multiple of a quadratic.

**Order of growth is N³, i.e. Θ(N³).** Note the count 100N² has a huge leading constant and still loses. This is why the cost-model shortcut works: find the operation with the fastest-growing count and ignore the rest.

---

## Common Pitfalls

- **Answering the runtime question with a number of seconds.** "It takes 3 ms" is not an answer about an algorithm; it is a fact about one machine, one input, and one day. Answer with a function of N.

- **Keeping multiplicative constants.** Writing Θ(N²/2) for `countDuplicates` or Θ(2N) for `count2`. Constants are dropped: those are Θ(N²) and Θ(N). Writing Θ(5) for a constant-time operation is the same mistake; the answer is **Θ(1)**.

- **Keeping low-order terms.** Θ(N² + N) should be written Θ(N²). Θ(N³ + 3N⁴) should be Θ(N⁴). The `count` example is the canonical trap: Θ(N) + Θ(N²) is Θ(N²), not Θ(N + N²).

- **Being fooled by which term is written first.** In N³ + 3N⁴ the *first* term is not the dominant one, and 3N⁴ has the larger coefficient *and* the larger exponent. Always compare growth rates, not position or coefficient size.

- **Thinking a bounded or shrinking term matters.** `40 sin(N)` is stuck in [-40, 40] forever, and `1/N` shrinks toward zero. Both are low-order next to any growing term. And 1/N + 5 is Θ(1), not Θ(1/N).

- **Forgetting the +1 in loop condition counts.** `i < numbers.length` runs N + 1 times, not N. (It does not change the order of growth here, but it is exactly the kind of thing an exam asks for in an exact-count table.)

- **Assuming the loop body always executes.** `evens += 1` runs between 0 and N times because it depends on the array's *contents*. Counts that depend on values, not just size, are why we distinguish best case from worst case.

- **Assuming nested loops are automatically N².** In `countDuplicates` the inner loop starts at `i + 1`, not 0, so the number of comparisons is N(N - 1)/2, not N². It happens to still be Θ(N²) here, but the reasoning matters: the triangle is half the square, and half is just a constant factor. Other loop structures (for instance an inner loop that runs a fixed number of times) will not be quadratic at all.

- **Adding when you should multiply, or vice versa.** Sequential code blocks: take the max of the two orders of growth. Nested loops: multiply. `count` calls two methods sequentially, so N and N² give N².

- **Ignoring the cost of helper methods.** `countZerps` is Θ(N²) only because `isZerp`'s runtime depends on its two `int` parameters, not on N. Always state this assumption or check the helper.

- **Hunting for k₁ and k₂ when analyzing code.** Big-Theta's formal definition exists to make "order of growth" precise, but using Θ "does not change the way we analyze code at all." You do not produce constants when asked for the runtime of a loop; you just write Θ(...).

- **Treating Big O as if it were Big Theta.** N³ + 3N⁴ ∈ O(N!) is a perfectly *true* statement, just useless. O is "≤", so it can be arbitrarily loose. Conversely, being in O(N²) does not mean a function grows like N²: lg(N) ∈ O(N²) but lg(N) ∉ Θ(N²).

- **Assuming the asymptotically better algorithm is always faster.** For small N, `glorp1` (2N²) beats `glorp2` (500N). Asymptotics is a statement about large N, which is exactly why the formal definitions all say "for all N greater than some N₀."

- **Notation slips.** It is R(N) ∈ Θ(f(N)), with set membership: Θ(N²) is a *family* of functions (N²/2, 2N², N² + 38N + N are all members). Also, write Θ(N²), not Θ(N^2 operations) or Θ(N² time); the notation already describes growth.

---

## Likely Exam Points

### 1. Give the order of growth of a code snippet

**Q:** What is the runtime of the following, in Θ notation, in terms of N = `a.length`?

```java
public static void countDuplicates(int[] a) {
    int duplicates = 0;
    for (int i = 0; i < a.length; i++) {
        for (int j = i + 1; j < a.length; j++) {
            if (a[i] == a[j]) {
                duplicates += 1;
            }
        }
    }
    IO.println("Duplicates: " + duplicates);
}
```

**A:** Θ(N²). Using `==` as the cost model, the number of comparisons is (N - 1) + (N - 2) + ... + 1 = N(N - 1)/2 = N²/2 - N/2. Drop the low-order -N/2 term and the constant 1/2 to get N². Equivalently: the pairs checked form a right triangle with legs of about N, and its area is ~N²/2.

---

### 2. Simplify a function to its order of growth

**Q:** Give the order of growth of each: (a) N³ + 3N⁴, (b) 1/N + N³, (c) 1/N + 5, (d) Ne^N + N, (e) 40 sin(N) + 4N².

**A:** (a) N⁴, (b) N³, (c) 1, (d) Ne^N, (e) N². For (c), nothing grows, so the answer is the constant class Θ(1). For (e), 40 sin(N) is bounded in [-40, 40] and therefore low-order compared to 4N².

---

### 3. Exact operation counts in a table

**Q:** For `countEvens` on an array of length N, how many times is the loop condition `i < numbers.length` evaluated, and how many times does `evens += 1` execute?

**A:** The condition is evaluated **N + 1** times (once before each of the N iterations, plus the final failing check). `evens += 1` executes **between 0 and N times**, depending on how many elements of the array are even; its count depends on the array's contents, not just its length.

---

### 4. Sequential vs nested composition

**Q:** `countEvens` is Θ(N) and `countDuplicates` is Θ(N²). What is the runtime of `count`, which calls `countEvens(numbers)` then `countDuplicates(numbers)`?

**A:** Θ(N²). Sequential calls add: R(N) = c₁N + c₂N², and c₁N is a low-order term that is dropped. Formally, c₂N² ≤ c₁N + c₂N² ≤ (c₁ + c₂)N² for N ≥ 1, so with k₁ = c₂ and k₂ = c₁ + c₂ the Big-Theta definition is satisfied for f(N) = N².

---

### 5. Constant factors do not change the class

**Q:** `count1` counts evens and odds in one pass; `count2` does it in two separate passes over the same array. Which has the better order of growth?

**A:** Neither; both are Θ(N). `count2` does roughly twice the work of `count1`, but a factor of 2 is a multiplicative constant and is discarded. Both double their runtime when N doubles.

---

### 6. State the formal definition of Big-Theta and verify it

**Q:** State what R(N) ∈ Θ(f(N)) means, then verify that 40 sin(N) + 4N² ∈ Θ(N²).

**A:** R(N) ∈ Θ(f(N)) means there exist positive constants k₁ and k₂ such that k₁·f(N) ≤ R(N) ≤ k₂·f(N) for all N greater than some N₀. For the example, take f(N) = N², k₁ = 3, k₂ = 5: since sin(N) ∈ [-1, 1], we have 4N² - 40 ≤ 40 sin(N) + 4N² ≤ 4N² + 40, and 3N² ≤ 4N² - 40 together with 4N² + 40 ≤ 5N² both hold once N² ≥ 40 (so N₀ = 7 works).

---

### 7. Find f, k₁, k₂ for a given R(N)

**Q:** Suppose R(N) = (4N² + 3N·ln(N)) / 2. Find a simple f(N) and corresponding k₁ and k₂.

**A:** f(N) = N², k₁ = 1, k₂ = 3. R(N) = 2N² + 1.5N·ln(N); since N·ln(N) grows more slowly than N², R(N) is eventually squeezed between 1·N² and 3·N².

---

### 8. Big O vs Big Theta

**Q:** Which of the following are true for R(N) = N³ + 3N⁴? (i) R ∈ Θ(N⁴) (ii) R ∈ Θ(N⁶) (iii) R ∈ O(N⁶) (iv) R ∈ O(N!) (v) R ∈ O(N³)

**A:** (i), (iii), and (iv) are true. (ii) is false: Θ requires a matching lower bound, and N³ + 3N⁴ grows strictly more slowly than N⁶. (iv) is true but extremely loose; O is only an upper bound ("less than or equal"). (v) is false: R grows faster than N³, so no constant k₂ can make R(N) ≤ k₂N³ hold for all large N.

---

### 9. Is membership in O(f) enough to describe growth?

**Q:** A classmate says "lg(N) ∈ O(N²), so lg(N) and N² grow at the same rate." What is wrong?

**A:** Big O is an upper bound only, so O(N²) contains functions that grow much more slowly than N², including lg(N). "Same rate" would require lg(N) ∈ Θ(N²), which fails: no positive k₁ satisfies k₁N² ≤ lg(N) for all large N. Θ is the "equals"-like statement; O is "less than or equal."

---

### 10. Reasoning from an operation-count table

**Q:** A method's operation counts are: `<` occurs 100N² + 3N times, `>` occurs 2N³ + 1 times, `&&` occurs 5,000 times. What is the order of growth, and why do the per-operation hardware costs not matter?

**A:** Θ(N³). If `<`, `>`, `&&` cost α, β, γ nanoseconds respectively, total time is α(100N² + 3N) + β(2N³ + 1) + 5000γ. For very large N the 2βN³ term dominates all others regardless of α, β, and γ, because no constant factor can let a quadratic overtake a cubic. Constants affect the absolute time, never the order of growth.

---

### 11. Helper method assumptions

**Q:** What is the runtime of `countZerps`, and what assumption does your answer depend on?

**A:** Θ(N²), assuming `isZerp`'s runtime depends only on its two `int` parameters and not on N (i.e. it is constant time). The loop structure makes N(N - 1)/2 calls; if each call is constant time, that constant is dropped. If `isZerp`'s cost grew with N, the total would be larger.

---

### 12. Small N vs large N

**Q:** `glorp1` takes 2N² operations and `glorp2` takes 500N. Which is faster, and does that contradict the claim that `glorp2` has better asymptotic behavior?

**A:** For small N (specifically N < 250), `glorp1` is faster. There is no contradiction: asymptotic claims describe behavior for large N, which is exactly why the Θ and O definitions require the inequalities to hold only "for all N greater than some N₀." As the dataset grows, the parabolic `glorp1` falls farther and farther behind.

---

## Summary

- Efficiency has two flavors: **programming cost** (development and, mostly, maintenance) and **execution cost** (time and memory). From here on the course is about execution cost.
- Given a code snippet, express its runtime as a function **R(N)**, where N is a property of the input, usually its size.
- We rarely want R(N) exactly; we want its **order of growth**: as N grows, what happens to the runtime?
- Two simplifications produce the order of growth: **ignore low-order terms** (constants, bounded terms like 40 sin(N), shrinking terms like 1/N) and **ignore multiplicative constants** (8N grows like N).
- The tedious method is to count every operation in a table. Useful gotchas: loop conditions run **N + 1** times, and body counts can be ranges (0 to N) when they depend on input *values*.
- The practical method (not universal) is a **cost model**: pick one representative operation, let **C(N)** be its count, and find f(N) with C(N) ∈ Θ(f(N)). Often, but not always, use the worst-case count.
- This shortcut is valid because per-operation hardware costs α, β, γ can never let a lower-order term overtake a higher-order one for large N. ("Extremely important point.")
- **Composition rules:** sequential blocks add, so take the larger order of growth (Θ(N) then Θ(N²) is Θ(N²)); nested loops multiply. A nested loop starting at `j = i + 1` gives N(N - 1)/2 ≈ N²/2 pairs, which is still Θ(N²).
- **Big-Theta:** R(N) ∈ Θ(f(N)) iff there exist positive k₁, k₂ with k₁f(N) ≤ R(N) ≤ k₂f(N) for all N > N₀. Informally "equals"; it *is* order of growth in formal clothing, and using it changes nothing about how you analyze code.
- **Big O:** R(N) ∈ O(f(N)) iff there exists positive k₂ with R(N) ≤ k₂f(N) for all N > N₀. Informally "less than or equal"; an upper bound that may be arbitrarily loose (N³ + 3N⁴ ∈ O(N!)). Θ(N²) and O(N²) are *families*; lg(N) is in the second but not the first.
- Scaling matters enormously at real-world sizes (billions of users, particles, transactions, bytes) and often determines whether a problem is solvable at all, not merely how quickly.
- Canonical results from this lecture: `countEvens` ∈ Θ(N), `count1` and `count2` ∈ Θ(N), `countDuplicates` ∈ Θ(N²), `countZerps` ∈ Θ(N²) (given constant-time `isZerp`), `count` ∈ Θ(N²).
- Big O's practical usefulness is previewed for the upcoming **Disjoint Sets** lecture.
