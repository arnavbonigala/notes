<!-- Fri, Nov 20, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 36: Sorting and Algorithmic Bounds

This lecture steps back from inventing new sorting algorithms and asks a harder question: how fast *could* any comparison sort possibly be? We define TUCS (The Ultimate Comparison Sort), the hypothetical asymptotically fastest comparison-based sorting algorithm, possibly not yet discovered, and we try to pin down its worst case runtime R(N) from both sides. The easy bounds come fast: R(N) is O(N log N) because mergesort already exists, and R(N) is Ω(N) because you have to at least look at every item. The rest of the lecture closes the gap on the lower bound using a game called Puppy, Cat, Dog: given N boxed animals and a scale that answers only yes/no comparison questions, how many questions must you ask in the worst case? Because there are N! possible arrangements and each yes/no question is one branch of a binary decision tree, you need a tree with N! leaves, hence at least lg(N!) levels. A short chain of math shows log(N!) is Θ(N log N). Finally, since sorting solves Puppy, Cat, Dog (sort the boxes, then read off the order), sorting inherits that Ω(N log N) lower bound. The punchline: mergesort and heapsort are not just good, they are asymptotically optimal, and no future human cleverness can beat Θ(N log N) comparisons in the worst case.

## Key Concepts

### Where we were: the sorts summary table

The lecture opens with the running table of sorts you have built up over the previous lectures:

| Sort | Memory | Time | Notes |
|---|---|---|---|
| Selection Sort | Θ(1) | Θ(N²) | |
| Heapsort | Θ(1) | Θ(N log N) | Bad caching (61C) |
| Mergesort | Θ(N) | Θ(N log N) | |
| Insertion Sort | Θ(1) | Θ(N²) | Θ(N) if almost sorted |
| Random Quicksort | Θ(log N) expected | Θ(N log N) expected | Fastest sort |

Two footnotes from the slides matter:

- Random quicksort's Θ(log N) memory comes from the cost of the computer tracking recursive calls, and it is an *expected* amount. The difference between log N and constant memory is trivial in practice.
- Stability is a property of sorts covered in discussion this week, and the slides explicitly say it is **in scope for exams**, even though this lecture does not develop it.

The question driving this lecture: several sorts hit Θ(N log N) worst case. Is that a coincidence, or is it a wall?

### TUCS: reasoning about an algorithm nobody has written

Define **TUCS** = The Ultimate Comparison Sort: the asymptotically fastest possible comparison sorting algorithm, possibly yet to be discovered. Let R(N) be its worst case runtime.

It feels strange to give asymptotic bounds on an algorithm whose code you have never seen, but the slides promise you can, and here is the trick: you bound it by reasoning about the *problem*, not the implementation.

**Upper bound, O(N log N).** TUCS is by definition at least as fast as every comparison sort. Mergesort is a comparison sort with Θ(N log N) worst case. So TUCS cannot be worse than Θ(N log N). Any algorithm you can actually exhibit gives you a ceiling on the best possible algorithm.

**First lower bound, Ω(1).** Trivially true, but useless: it just says the problem does not get easier as N grows.

**Better lower bound, Ω(N).** You have to at least look at every item. If you skipped an item entirely, an adversary could change that item to something that belongs elsewhere and your output would be wrong.

So after five minutes of thought, TUCS "lives" somewhere between Θ(N) and Θ(N log N). The lecture's goal is to squeeze that gap shut from the bottom.

The strength of what we are about to prove is worth sitting with. A lower bound of Ω(N log N) on TUCS means that **across the infinite space of all possible ideas any human might ever have for sorting using sequential comparisons, none has a worst case runtime better than Θ(N log N)**. This is not "we tried hard and failed." It is a mathematical impossibility result.

### Puppy, Cat, Dog: turning sorting into a guessing game

The lower bound argument goes through a detour. Suppose there is a puppy, a cat, and a dog, each inside an opaque soundproof box labeled A, B, and C. You cannot see or hear them. You have a scale, so you can ask yes/no questions of the form "is box a lighter than box b?" (The implicit ordering: puppy < cat < dog by weight.) Goal: figure out which animal is in which box.

Trace the small cases from the slides:

- Ask `a < b`? and `b < c`?
  - Yes, Yes → the weights increase left to right, so sorted order is abc → a: puppy, b: cat, c: dog.
  - No, No → weights decrease left to right, sorted order is cba → c: puppy, b: cat, a: dog.
  - **Yes, No** → ambiguous! Two answers survive: acb (a: puppy, c: cat, b: dog) and cab (c: puppy, a: cat, b: dog). In both, b is the dog (heaviest), but we do not know whether a or c is lighter.
  - To resolve it, ask a third question: `a < c`? If yes, order is acb; if no, order is cab.

This is the crucial observation: **two questions are not always enough for N = 3.** Some branches need three.

The full decision tree for N = 3 from the slides:

```
                        Is a < b?
                  No                Yes
             Is a < c?            Is b < c?
          No        Yes        No         Yes
      Is b < c?     bac    Is a < c?      abc
     No     Yes            No    Yes
    cba     bca            cab   acb
```

Six leaves, one per permutation of three items. Note that some root-to-leaf paths have length 2 (`bac` and `abc`) and some have length 3. The **height** of the tree, which is the worst case number of questions, is 3.

### Counting leaves: why the tree must be tall

Now scale up to N = 4, the "puppy, cat, dog, walrus" problem. How many questions do you need to **definitely** solve it, meaning your tree of questions has to work for *any* permutation?

The argument is a counting argument and it is the heart of the lecture:

1. There are 4! = 24 possible permutations of 4 distinct items. (For N = 3, 3! = 6, matching the 6 leaves above.)
2. Each permutation must be a distinct leaf in your decision tree. If two permutations shared a leaf, you would be unable to distinguish them, like the Yes/No ambiguity above.
3. Each question is yes/no, so the tree is **binary**: each internal node has at most 2 children.
4. A binary tree of height h has at most 2^h leaves. To get 24 leaves you need 2^h ≥ 24, so h ≥ lg(24) = 4.58..., and since h is an integer, h ≥ 5.

Answer: **5 questions**, and the slides show an optimal tree of height 5 for N = 4. (`lg` just means log base 2.)

Generalizing: for N items the decision tree needs N! leaves, so you need at least ⌈lg(N!)⌉ levels. The worst case question count is **Ω(log(N!))**.

A note on how hard the exact problem is: finding an *optimal* decision tree for general N is an **open problem in mathematics**. Optimal trees are known only for N = 1 through N = 22 (see OEIS A036604), and the best trees for N = 16, 17, and 18 were not discovered until 2023. We are not computing the exact answer, we are bounding it.

### Understanding log(N!)

Ω(log(N!)) is an honest answer but an unfamiliar one. How does log(N!) compare to N, N log N, N², 2^N? The lecture works this out in stages.

**Stage 1: N! ∈ Ω(2^N).** Write both out:

```
N! =  N * (N-1) * (N-2) * ... * 3 * 2 * 1     (N terms, almost all bigger than 2)
2^N = 2 *   2   *   2   * ... * 2 * 2 * 2     (N terms, all exactly 2)
```

Both products have N terms, and almost every term of N! is at least 2. So N! grows at least as fast as 2^N. Taking logs: log(N!) ∈ Ω(log(2^N)) = Ω(N).

That only gets us back to Ω(N), which we already knew. We need something tighter.

**Stage 2: N! ∈ Ω((N/2)^(N/2)).** Compare against a cleverer function. The slides use 10! as the concrete case:

```
10! = 10 * 9 * 8 * 7 * 6 * 5 * 4 * 3 * 2 * 1
5^5 =  5 * 5 * 5 * 5 * 5
```

The intuition: the **top half** of the factorial's terms (the N/2 largest ones, here 10, 9, 8, 7, 6) are each at least N/2 (here 5). So N! is at least the product of N/2 copies of N/2, which is (N/2)^(N/2). The remaining smaller terms only help. Therefore N! > (N/2)^(N/2) for large N, so N! ∈ Ω((N/2)^(N/2)).

**Stage 3: log(N!) ∈ Ω(N log N).** Take logs of the inequality from stage 2:

- N! > (N/2)^(N/2)
- log(N!) > log((N/2)^(N/2))
- Bring down the exponent: log(N!) > (N/2) · log(N/2)
- Discard the unnecessary constant 1/2: log(N!) ∈ Ω(N log(N/2))
- Since log(N/2) = log(N) - log(2): log(N!) ∈ Ω(N log N - N log 2)
- The N log 2 term is linear and dominated: log(N!) ∈ Ω(N log N)

In words: **log(N!) grows at least as quickly as N log N.**

**Stage 4: N log N ∈ Ω(log(N!)), the other direction.** The slides' proof is a direct term-by-term comparison:

- log(N!) = log(N) + log(N-1) + log(N-2) + ... + log(1)
- N log N = log(N) + log(N) + log(N) + ... + log(N)   (N copies)

Both sums have N terms, and each term of the second sum is at least as large as the corresponding term of the first. So N log N ∈ Ω(log(N!)).

**Conclusion: log(N!) ∈ Θ(N log N).** Each function is Ω of the other, which is exactly what two-sided bounding means. Informally, N log N ≥ log(N!) and log(N!) ≥ N log N, so N log N = log(N!) asymptotically. The slides link a Desmos plot (desmos.com/calculator/qorylsvuhx) showing the two curves tracking each other.

The slides explicitly say: if you did not follow every proof step, that is OK, come back later, and for the rest of today just remember **log(N!) ∈ Θ(N log N)**.

### Reduction: sorting solves Puppy, Cat, Dog

We have shown the Puppy-Cat-Dog problem requires Ω(N log N) comparisons in the worst case. Any algorithm solving Puppy-Cat-Dog with comparisons therefore requires Ω(N log N) *time* in the worst case, since each comparison costs at least a "step" of time on top of whatever else the algorithm is doing.

Now the connecting move. **Sorting solves Puppy, Cat, Dog.** Finding an optimal question sequence is hard, but who said we need optimal? For N = 4, just sort the boxes with any comparison sort (insertion, selection, whatever):

- Leftmost box is the puppy (lightest)
- Left middle box is the cat
- Right middle box is the dog
- Rightmost box is the walrus (heaviest)

A comparison sort only ever asks yes/no comparison questions, so running it gives you a valid (if not optimal) decision tree for Puppy-Cat-Dog. The slides show the insertion sort decision tree for N = 4: its height is 6, one more than the theoretical minimum of 5. Insertion sort is a legal but slightly suboptimal strategy.

**The contrapositive is the whole point.** Could there be a sorting algorithm using only Θ(N) comparisons in the worst case? No. If such an algorithm existed, we could use it to solve Puppy-Cat-Dog with only Θ(N) comparisons. But we proved Puppy-Cat-Dog needs Ω(N log N) comparisons. Contradiction. So no such sorting algorithm exists.

The slides name this technique: it is a **reduction**. Puppy-Cat-Dog reduces to SORT, so SORT is at least as "hard" as Puppy-Cat-Dog. Hardness flows upward through a reduction. (You will reduce problems to each other extensively in CS70, CS170, and CS172.)

### The sorting lower bound, assembled

Chaining everything:

- Puppy, cat, dog requires lg(N!) comparisons, so it is Ω(lg N!).
- TUCS can solve puppy, cat, dog, and thus takes Ω(lg N!) compares.
- lg(N!) ∈ Ω(N log N), because N! ∈ Ω((N/2)^(N/2)).
- Therefore TUCS is Ω(N log N).

Informally: TUCS ≥ puppy, cat, dog ≥ log N! ≥ N log N.

**Any comparison based sort requires at least order N log N comparisons in its worst case.** Combined with the O(N log N) upper bound from mergesort, R(N) ∈ Θ(N log N). The gap is closed.

### Optimality: what this means for the sorts you know

Return to the table. Mergesort and heapsort achieve Θ(N log N) worst case. We have now proved no comparison sort can do better. Therefore:

**Our best sorts have achieved absolute asymptotic optimality. It is mathematically impossible to sort using fewer comparisons.**

The caveat from the slides: randomized quicksort is only *probabilistically* optimal, since its Θ(N log N) is an expected bound. But the probability is extremely high for even modest N. The slides' framing: "Are you worried about quantum teleportation? Then don't worry about Quicksort."

(extra context: the bound applies only to *comparison* sorts, algorithms whose only access to the data is asking "is x < y?" Sorts that look at the internal structure of the keys, like the LSD and MSD radix sorts mentioned in the Sound of Sorting links, sidestep this bound entirely and are typically covered in the next lecture. Nothing in this lecture is contradicted by that; radix sorts simply are not comparison sorts, so the decision tree argument does not apply to them.)

### Warm-up callbacks

The two warm-up questions frame the lecture:

1. "Would you rather add or multiply two 3-digit numbers in your head?" Everyone says add. Why? Because addition and multiplication have genuinely different intrinsic difficulty, not just different implementations. That is the theme: some problems are harder than others in a way that no cleverness fixes.
2. "If you could only ask Yes/No questions, what is the least number of questions to always figure out a 6-digit lock combination?" This is exactly the decision tree counting argument in miniature. There are 10^6 = 1,000,000 combinations, you need a binary tree with 10^6 leaves, so you need at least ⌈lg(10^6)⌉ = 20 questions.

### Sound of Sorting

The lecture closes with the Sound of Sorting video (youtube.com/watch?v=kPRA0W1kECg), which sonifies sorting algorithms on 125 items: selection sort (0:00), insertion sort (0:09), quicksort (0:38), mergesort (1:05), heapsort (1:28), LSD sort (1:54), MSD sort (2:10), Shell's sort (3:37).

Questions the slides pose to ponder after class: how many items for selection sort? Why does insertion sort take longer / more compares than selection sort? At what timestamp does the first partition complete for quicksort? Could the size of the input to mergesort be a power of 2? What do the colors mean for heapsort?

## Definitions

**Comparison sort**: A sorting algorithm that makes decisions by comparing items, for example via Java's `compareTo` method. It has no access to the internal structure of the keys beyond the ordering relation.

**TUCS (The Ultimate Comparison Sort)**: The asymptotically fastest possible comparison sorting algorithm, possibly yet to be discovered. Its worst case runtime is written R(N).

**R(N)**: The worst case runtime of TUCS. Proved in this lecture to be Θ(N log N).

**Decision tree**: A binary tree representing a strategy of yes/no questions. Each internal node is a question, each edge is an answer (yes or no), and each leaf is a final conclusion (here, a specific permutation). The **height** of the tree equals the worst case number of questions asked.

**Puppy, Cat, Dog problem**: Given N items in opaque labeled boxes and a device that answers only yes/no comparison questions, determine which item is in which box. Equivalent to determining the permutation.

**lg**: Logarithm base 2. Used in the slides interchangeably with "log" in asymptotic contexts, since changing logarithm base only multiplies by a constant.

**Reduction**: Showing that problem A can be solved using an algorithm for problem B, which proves B is at least as hard as A. Here, Puppy-Cat-Dog reduces to SORT, so SORT inherits Puppy-Cat-Dog's lower bound.

**Lower bound (Ω)**: A guarantee that an algorithm (or every algorithm for a problem) takes *at least* this much. Informally, f ∈ Ω(g) can be read as f ≥ g asymptotically.

**Upper bound (O)**: A guarantee that an algorithm takes *at most* this much. Informally, f ∈ O(g) can be read as f ≤ g asymptotically.

**Θ**: Both Ω and O. If f ∈ Ω(g) and g ∈ Ω(f), then f ∈ Θ(g), meaning they grow at the same rate.

**Asymptotic optimality**: An algorithm is asymptotically optimal for a problem when its runtime matches the problem's proven lower bound. Mergesort and heapsort are asymptotically optimal comparison sorts.

**Stability**: A property of sorts explored in this week's discussion. The slides flag it as in scope for exams but do not define it in this lecture. (extra context: a sort is stable if items comparing equal retain their original relative order.)

## Worked Examples

### Example 1: counting the decision tree bound for N = 4

The slides ask: how many questions to definitely solve puppy, cat, dog, walrus? Answer 5, by counting. Here is that count as runnable code.

```java
public class DecisionTreeBound {
    public static long factorial(int n) {
        long result = 1;
        for (int i = 2; i <= n; i += 1) {
            result = result * i;
        }
        return result;
    }

    public static int minQuestions(int n) {
        long leaves = factorial(n);
        int height = 0;
        long capacity = 1;
        while (capacity < leaves) {
            capacity = capacity * 2;
            height += 1;
        }
        return height;
    }

    public static void main(String[] args) {
        for (int n = 1; n <= 6; n += 1) {
            System.out.println("N=" + n
                + " permutations=" + factorial(n)
                + " minQuestions=" + minQuestions(n));
        }
    }
}
```

Step by step through `minQuestions(4)`:

- `factorial(4)` loops i from 2 to 4: result becomes 2, then 6, then 24. `leaves = 24`.
- `capacity` starts at 1 (a tree of height 0 holds 1 leaf), `height = 0`.
- Iteration 1: 1 < 24, so capacity = 2, height = 1.
- Iteration 2: 2 < 24, so capacity = 4, height = 2.
- Iteration 3: 4 < 24, so capacity = 8, height = 3.
- Iteration 4: 8 < 24, so capacity = 16, height = 4.
- Iteration 5: 16 < 24, so capacity = 32, height = 5.
- Loop check: 32 ≥ 24, stop. Return 5.

This loop is literally computing ⌈lg(24)⌉ = ⌈4.58⌉ = 5 by repeated doubling. The variable `capacity` is tracking "how many leaves can a binary tree of this height hold," and `height` is the answer we want.

Output:

```
N=1 permutations=1 minQuestions=0
N=2 permutations=2 minQuestions=1
N=3 permutations=6 minQuestions=3
N=4 permutations=24 minQuestions=5
N=5 permutations=120 minQuestions=7
N=6 permutations=720 minQuestions=10
```

Sanity check against the slides: N = 3 gives 3, matching the N = 3 decision tree whose longest path was indeed 3 questions. N = 4 gives 5, matching the slide answer exactly.

Important caveat: this computes the *information-theoretic lower bound*, not necessarily an achievable height. For small N they happen to coincide, but as the slides note, finding genuinely optimal trees is an open problem beyond N = 22.

### Example 2: the N = 3 decision tree as explicit code

The lecture's N = 3 decision tree can be written out directly. This makes concrete what "a decision tree is a strategy" means.

```java
public class PuppyCatDog3 {
    public static String solve(int a, int b, int c) {
        if (a < b) {
            if (b < c) {
                return "abc";
            } else {
                if (a < c) {
                    return "acb";
                } else {
                    return "cab";
                }
            }
        } else {
            if (b < c) {
                return "bac";
            } else {
                if (a < c) {
                    return "bca";
                } else {
                    return "cba";
                }
            }
        }
    }
}
```

Reading this as a tree: the outermost `if (a < b)` is the root question. Each nested `if` is an internal node. Each `return` is a leaf. There are exactly six `return` statements, matching 3! = 6 permutations.

Trace `solve(5, 1, 9)` (so b is the puppy, a is the cat, c is the dog):

- `a < b` is 5 < 1, false, so take the else branch.
- `b < c` is 1 < 9, true, so return `"bac"`.
- Two comparisons used. This is one of the short paths.

Trace `solve(5, 9, 1)` (so c is the puppy, a is the cat, b is the dog):

- `a < b` is 5 < 9, true.
- `b < c` is 9 < 1, false, so take the else branch.
- `a < c` is 5 < 1, false, so return `"cab"`.
- Three comparisons used. This is one of the long paths, matching the ambiguity the slides highlighted.

The key structural fact: **no path through this method can be shorter than ⌈lg 6⌉ = 3 for every input**. You can make two paths short (abc and bac), but you cannot make all six short, because a binary tree of height 2 has at most 4 leaves and you need 6.

### Example 3: verifying log(N!) ∈ Θ(N log N) numerically

The proof is clean but abstract. Seeing the ratio of the two functions converge toward a constant makes it tangible.

```java
public class LogFactorial {
    public static double logFactorial(int n) {
        double total = 0.0;
        for (int i = 2; i <= n; i += 1) {
            total += Math.log(i) / Math.log(2);
        }
        return total;
    }

    public static void main(String[] args) {
        for (int n = 10; n <= 1000000; n *= 10) {
            double lgFact = logFactorial(n);
            double nLogN = n * (Math.log(n) / Math.log(2));
            System.out.printf("N=%-9d lg(N!)=%-14.1f N lg N=%-14.1f ratio=%.4f%n",
                n, lgFact, nLogN, lgFact / nLogN);
        }
    }
}
```

What this does and why:

- `logFactorial` computes lg(N!) by summing lg(i) for i from 2 to N, using the identity log(N!) = log(N) + log(N-1) + ... + log(1). This is exactly the decomposition the slides used in the N log N ∈ Ω(log(N!)) proof. Summing logs instead of computing N! directly avoids overflow: 21! already exceeds the range of a `long`.
- `Math.log(i) / Math.log(2)` converts natural log to log base 2 via the change-of-base formula.
- The loop multiplies n by 10 each time to sweep several orders of magnitude.

Output (approximate):

```
N=10        lg(N!)=21.8          N lg N=33.2          ratio=0.6558
N=100       lg(N!)=524.8         N lg N=664.4         ratio=0.7899
N=1000      lg(N!)=8529.4        N lg N=9965.8        ratio=0.8559
N=10000     lg(N!)=118458.1      N lg N=132877.1      ratio=0.8914
N=100000    lg(N!)=1516705.3     N lg N=1660964.0     ratio=0.9131
N=1000000   lg(N!)=18488878.0    N lg N=19931568.6    ratio=0.9276
```

Read this carefully. The ratio stays bounded between roughly 0.65 and 1.0 and creeps slowly upward toward 1. It never blows up and never collapses toward 0. That bounded-ratio behavior is precisely what Θ means: lg(N!) and N lg N differ by at most a constant factor, in both directions. This is the numerical shadow of the two-sided proof. The ratio is always below 1, matching the "N log N ≥ log(N!)" direction, and it never drops toward 0, matching the "log(N!) ≥ (asymptotically) N log N" direction.

### Example 4: the Ω((N/2)^(N/2)) comparison from the slides

The slides use 10! vs 5^5 to build intuition. Making it concrete:

```java
public class FactorialVsHalfPower {
    public static void main(String[] args) {
        for (int n = 4; n <= 20; n += 2) {
            double logFact = 0.0;
            for (int i = 2; i <= n; i += 1) {
                logFact += Math.log(i);
            }
            double logHalfPower = (n / 2.0) * Math.log(n / 2.0);
            System.out.printf("N=%-4d ln(N!)=%-10.2f ln((N/2)^(N/2))=%-10.2f N! bigger? %b%n",
                n, logFact, logHalfPower, logFact > logHalfPower);
        }
    }
}
```

Why compare in log space: for N = 20, 20! is about 2.4 × 10^18 and 10^10 is 10^10. Both are near or past the edges of comfortable integer arithmetic, and the comparison is cleaner on the logs since log is monotonically increasing, so `a > b` exactly when `log a > log b`.

For N = 10: ln(10!) = ln(3628800) ≈ 15.10, and ln(5^5) = 5 · ln(5) ≈ 8.05. So 10! > 5^5, confirming the slides' claim with the concrete numbers 3,628,800 vs 3,125.

The structural reason (not just the numbers): among the terms of 10! = 10·9·8·7·6·5·4·3·2·1, the top five terms 10, 9, 8, 7, 6 are each at least 5. Their product alone is at least 5^5. Everything remaining (5·4·3·2·1 = 120) multiplies on top. So N! ≥ (N/2)^(N/2) is not a numerical accident, it is a term-by-term domination argument.

### Example 5: insertion sort as a Puppy-Cat-Dog solver

The slides claim sorting solves Puppy-Cat-Dog. Here is insertion sort on boxes, returning the identification.

```java
public class SortSolvesPuppyCatDog {
    public static int[] identify(int[] weights) {
        int n = weights.length;
        int[] boxLabels = new int[n];
        int[] copy = new int[n];
        for (int i = 0; i < n; i += 1) {
            boxLabels[i] = i;
            copy[i] = weights[i];
        }
        for (int i = 1; i < n; i += 1) {
            int j = i;
            while (j > 0 && copy[j] < copy[j - 1]) {
                int tmpW = copy[j];
                copy[j] = copy[j - 1];
                copy[j - 1] = tmpW;
                int tmpL = boxLabels[j];
                boxLabels[j] = boxLabels[j - 1];
                boxLabels[j - 1] = tmpL;
                j -= 1;
            }
        }
        return boxLabels;
    }

    public static void main(String[] args) {
        int[] weights = {30, 10, 50, 20};
        int[] answer = identify(weights);
        String[] animals = {"puppy", "cat", "dog", "walrus"};
        for (int rank = 0; rank < answer.length; rank += 1) {
            System.out.println("Box " + answer[rank] + " holds the " + animals[rank]);
        }
    }
}
```

Step by step on `weights = {30, 10, 50, 20}`:

Thinking in environment / box-and-pointer terms: `weights` is a reference to a 4-slot array object on the heap. `identify` creates two *new* array objects, `copy` and `boxLabels`, and copies values in, so the caller's `weights` array is never mutated. `boxLabels[k]` always answers "which original box is currently sitting in position k." The two arrays are kept in lockstep: every swap of `copy` entries is mirrored by a swap of `boxLabels` entries, so the correspondence never breaks.

Initial state: `copy = [30, 10, 50, 20]`, `boxLabels = [0, 1, 2, 3]`.

- i = 1: j = 1, is `copy[1] < copy[0]`, 10 < 30, yes. Swap. `copy = [10, 30, 50, 20]`, `boxLabels = [1, 0, 2, 3]`. j becomes 0, loop exits on `j > 0`.
- i = 2: j = 2, is `copy[2] < copy[1]`, 50 < 30, no. No swaps.
- i = 3: j = 3, is `copy[3] < copy[2]`, 20 < 50, yes. Swap. `copy = [10, 30, 20, 50]`, `boxLabels = [1, 0, 3, 2]`. j = 2, is `copy[2] < copy[1]`, 20 < 30, yes. Swap. `copy = [10, 20, 30, 50]`, `boxLabels = [1, 3, 0, 2]`. j = 1, is `copy[1] < copy[0]`, 20 < 10, no. Stop.

Final: `boxLabels = [1, 3, 0, 2]`. Output:

```
Box 1 holds the puppy
Box 3 holds the cat
Box 0 holds the dog
Box 2 holds the walrus
```

Check against the input: box 1 had weight 10 (lightest, puppy), box 3 had 20 (cat), box 0 had 30 (dog), box 2 had 50 (heaviest, walrus). Correct.

Count the comparisons on this run: 1 + 1 + 2 = 4. But the *worst case* over all 24 permutations is 6, matching the slides' statement that insertion sort's decision tree for N = 4 has height 6, one above the theoretical minimum of 5. Insertion sort is a valid but slightly wasteful question-asking strategy.

This method is the reduction, made executable: a comparison sort, used as a Puppy-Cat-Dog solver.

### Example 6: why no Θ(N) comparison sort can exist

The contradiction argument, written as a thought experiment in code.

```java
public class ImpossibilityArgument {
    public static int[] magicLinearSort(int[] items) {
        throw new UnsupportedOperationException(
            "This cannot exist. See the proof below.");
    }
}
```

The reasoning, step by step:

1. Suppose `magicLinearSort` existed, making only c·N comparisons in the worst case for some constant c.
2. By Example 5's construction, we could wrap it to solve Puppy-Cat-Dog, since all it does is ask yes/no comparison questions.
3. Its decision tree would have height at most c·N, so at most 2^(c·N) leaves.
4. But we need N! distinct leaves to distinguish all N! permutations.
5. We proved N! ∈ Ω((N/2)^(N/2)), which grows faster than 2^(c·N) for any fixed c. So for large enough N, 2^(c·N) < N!.
6. Pigeonhole: with fewer leaves than permutations, at least two distinct permutations must land on the same leaf. The algorithm cannot tell them apart, so it outputs the wrong order for at least one of them.
7. Contradiction. `magicLinearSort` cannot be correct.

The pigeonhole step in step 6 is the engine of the whole proof. It is the same reason two questions could not resolve the N = 3 case: too few leaves, so some pair of permutations collides.

## Common Pitfalls

**Confusing the bound on TUCS with a bound on a specific algorithm.** The Ω(N log N) result is about the *problem*, not about any one implementation. Saying "mergesort is Ω(N log N)" is a different (and also true) claim from "every comparison sort is Ω(N log N) in the worst case." The second is the lecture's theorem and it is far stronger.

**Forgetting that the bound applies to comparison sorts only.** Every step of the argument assumes the algorithm's only way of distinguishing items is yes/no comparison questions. An algorithm that inspects key structure directly is not constrained by this decision tree, since its "questions" are not binary and its decision tree is not binary. The slides are explicit: "By comparison sort, I mean that it uses e.g. the `compareTo` method in Java to make decisions."

**Thinking lg(N!) is some exotic new function.** It is Θ(N log N). The lecture spends four slides proving exactly this so you can stop worrying about it. When a problem says "the answer is Ω(log(N!))," the expected simplification is Ω(N log N).

**Claiming the lower bound is about average case.** The entire argument is about the worst case. The decision tree's *height* is the worst case path. Average case comparison bounds exist (extra context: also Θ(N log N)) but are not what this lecture proves.

**Mixing up which direction a reduction proves hardness.** Puppy-Cat-Dog reduces to sorting (sorting solves Puppy-Cat-Dog), so **sorting inherits Puppy-Cat-Dog's lower bound**, making sorting at least as hard. Reversing this ("Puppy-Cat-Dog is at least as hard as sorting") does not follow from what we showed and is a common exam error.

**Using 2^h ≥ N! but forgetting the ceiling.** lg(24) = 4.58, so the answer is 5, not 4 and not 4.58. You cannot ask a fractional question. Always round *up*.

**Assuming the information-theoretic bound is always achievable.** ⌈lg(N!)⌉ is a lower bound on the height. For N = 1 through 22 optimal trees are known and sometimes match it, but finding them in general is an open mathematical problem, and the best trees for N = 16, 17, 18 were only found in 2023.

**Mistaking "optimal" for "fastest in practice."** Mergesort and heapsort are both asymptotically optimal, yet the slides note heapsort has bad caching behavior (a 61C topic) and randomized quicksort is labeled "fastest sort" in practice despite being only probabilistically optimal. Asymptotic optimality is about comparison counts, not wall-clock speed.

**Thinking Ω(1) and Ω(N) are wrong.** They are both correct lower bounds on R(N), just weak ones. Ω notation gives *a* lower bound, not *the* tightest one. An exam asking for "the best Ω bound you can give" wants the tightest true one.

**Dropping the N log 2 term incorrectly.** In the stage-3 derivation, log(N!) ∈ Ω(N log N - N log 2). It is tempting to worry about the subtraction. The N log 2 term is Θ(N), which is dominated by N log N, so the whole expression is Ω(N log N). Subtracting a lower-order term does not change the asymptotic class.

**Proving only one direction of a Θ claim.** The lecture deliberately proves both log(N!) ∈ Ω(N log N) and N log N ∈ Ω(log(N!)) before concluding Θ. Showing only one direction and writing Θ is a classic lost point. The clicker question on this slide (answers A, B, C, D) exists precisely to test whether you see that both directions were needed, and the answer is C, both A and B.

## Likely Exam Points

### 1. Computing the decision tree lower bound for a given N

**Practice question:** You have 6 animals in labeled boxes and a scale answering only yes/no questions. What is the minimum number of questions that always suffices, as a lower bound from the decision tree argument?

**Answer:** 6! = 720 permutations, so you need a binary tree with at least 720 leaves. A binary tree of height h holds at most 2^h leaves, so 2^h ≥ 720, giving h ≥ lg(720) = 9.49..., so h ≥ 10. The lower bound is **10 questions**. (Note this is a lower bound on the height; whether a tree of exactly height 10 exists for N = 6 is a separate question.)

### 2. Stating and justifying the sorting lower bound

**Practice question:** State the sorting lower bound theorem and give the three-step chain of reasoning that proves it.

**Answer:** Any comparison based sorting algorithm requires Ω(N log N) comparisons in the worst case. The chain: (1) Puppy-Cat-Dog on N items requires a decision tree with N! leaves, hence height at least lg(N!), so it is Ω(lg N!). (2) Any comparison sort solves Puppy-Cat-Dog (sort the boxes, read off the order), so any comparison sort's decision tree must also be Ω(lg N!) tall. (3) lg(N!) ∈ Ω(N log N), since N! ∈ Ω((N/2)^(N/2)). Combining: any comparison sort is Ω(N log N).

### 3. The log(N!) = Θ(N log N) proof, in either direction

**Practice question:** Prove that N log N ∈ Ω(log(N!)).

**Answer:** Expand both sides as sums. log(N!) = log(N) + log(N-1) + log(N-2) + ... + log(1), which has N terms. N log N = log(N) + log(N) + ... + log(N), also N terms. Comparing term by term, each term of the second sum is at least as large as the corresponding term of the first, since log is increasing and N ≥ N-k for all k ≥ 0. Therefore N log N ≥ log(N!) for all N, so N log N ∈ Ω(log(N!)).

### 4. Giving the best O and Ω bounds for TUCS, with justification

**Practice question:** Let R(N) be the worst case runtime of the asymptotically fastest possible comparison sort. Give the tightest O and Ω bounds you can, with a one-line justification each.

**Answer:** R(N) ∈ O(N log N), because mergesort is a comparison sort with Θ(N log N) worst case, and TUCS is by definition no worse than any existing comparison sort. R(N) ∈ Ω(N log N), by the decision tree argument: TUCS must distinguish N! permutations using binary comparisons, requiring at least lg(N!) ∈ Ω(N log N) of them. Together: R(N) ∈ Θ(N log N).

### 5. Recognizing which claims the lecture does and does not license

**Practice question:** True or false, with justification: "The sorting lower bound proves that no sorting algorithm can run faster than Θ(N log N)."

**Answer:** **False.** The bound applies only to *comparison* sorts, algorithms that learn about the data only by asking yes/no comparison questions. The decision tree argument relies on every question being binary, so it does not constrain algorithms that extract information from keys in other ways. The correct statement is: no *comparison* sort can beat Θ(N log N) comparisons in the worst case.

### 6. Insertion sort's decision tree vs the optimal tree

**Practice question:** For N = 4, the theoretical minimum decision tree height is 5, but insertion sort's decision tree has height 6. What does this tell us, and does it contradict the lower bound?

**Answer:** It tells us insertion sort is a valid but non-optimal strategy for Puppy-Cat-Dog: it always produces a correct answer, but in its worst case it asks one more question than strictly necessary. It does not contradict the lower bound at all. The lower bound says height ≥ 5; insertion sort's 6 satisfies that. The lower bound forbids trees *shorter* than 5, not trees taller than 5.

### 7. The reduction concept

**Practice question:** Explain what it means that "Puppy-Cat-Dog reduces to SORT" and what that lets us conclude.

**Answer:** It means any instance of Puppy-Cat-Dog can be solved by calling a sorting algorithm: sort the boxes by weight, then the k-th box in sorted order holds the k-th lightest animal. Because sorting is powerful enough to solve Puppy-Cat-Dog, sorting is **at least as hard** as Puppy-Cat-Dog. Therefore any lower bound on Puppy-Cat-Dog is also a lower bound on sorting. This is why the Ω(N log N) comparison bound transfers from the puzzle to the sorting problem.

### 8. Reading the sorts summary table

**Practice question:** Which sorts in the lecture's table are asymptotically optimal comparison sorts, and what caveat applies to one of them?

**Answer:** Mergesort (Θ(N log N) worst case) and heapsort (Θ(N log N) worst case) are asymptotically optimal: they match the proven Ω(N log N) lower bound. Randomized quicksort has Θ(N log N) *expected* time, so it is only probabilistically optimal, though with extremely high probability even for modest N. Selection sort and insertion sort at Θ(N²) are not optimal, though insertion sort achieves Θ(N) on nearly-sorted input, which does not violate the bound because the bound is about the worst case.

### 9. The warm-up lock combination question

**Practice question:** Using only yes/no questions, what is the minimum number of questions needed to always determine a 6-digit lock combination?

**Answer:** There are 10^6 = 1,000,000 possible combinations, so the decision tree needs 1,000,000 leaves. A binary tree of height h holds at most 2^h leaves, so 2^h ≥ 10^6, giving h ≥ lg(10^6) ≈ 19.93, so h ≥ **20 questions**. Each yes/no question extracts at most one bit of information, and a 6-digit combination carries about 19.93 bits.

### 10. Why N! ∈ Ω((N/2)^(N/2))

**Practice question:** Explain intuitively why N! ≥ (N/2)^(N/2) for large N.

**Answer:** Write N! = N · (N-1) · ... · 2 · 1. Consider only the largest N/2 factors, which are N, N-1, ..., down to roughly N/2. Each of these is at least N/2. The product of N/2 numbers each at least N/2 is at least (N/2)^(N/2). The remaining smaller factors are all at least 1, so they only increase the product. Therefore N! ≥ (N/2)^(N/2). The slides illustrate with N = 10: 10! = 3,628,800 and 5^5 = 3,125, so 10! is far larger.

## Summary

- **TUCS** is the asymptotically fastest possible comparison sort, possibly undiscovered; R(N) is its worst case runtime.
- Easy bounds: R(N) ∈ O(N log N) because mergesort exists, R(N) ∈ Ω(N) because you must look at every item. The gap between Θ(N) and Θ(N log N) is what the lecture closes.
- **Puppy, Cat, Dog**: identify N animals in opaque boxes using only yes/no comparison questions. For N = 3 the answer is 3 questions, for N = 4 it is 5.
- **Decision tree argument**: N items have N! permutations, each needing its own leaf. Yes/no questions make the tree binary, and a binary tree of height h holds at most 2^h leaves. So h ≥ ⌈lg(N!)⌉, meaning Puppy-Cat-Dog is Ω(log(N!)).
- **log(N!) ∈ Θ(N log N)**, proved in two directions:
  - log(N!) ∈ Ω(N log N), because N! ≥ (N/2)^(N/2), so log(N!) ≥ (N/2)log(N/2) ∈ Ω(N log N).
  - N log N ∈ Ω(log(N!)), because log(N!) = Σ log(k) is a sum of N terms each at most log(N), while N log N is a sum of N terms each exactly log(N).
- **Reduction**: sorting solves Puppy-Cat-Dog (sort the boxes, read off the order), so sorting is at least as hard, and inherits the Ω(N log N) lower bound.
- **The sorting lower bound**: any comparison based sort requires at least log(N!) ∈ Ω(N log N) comparisons in the worst case. Informally, TUCS ≥ puppy, cat, dog ≥ log N! ≥ N log N.
- Combined with the O(N log N) upper bound, **R(N) ∈ Θ(N log N)**. Mergesort and heapsort are **asymptotically optimal**: it is mathematically impossible to sort with fewer comparisons. Randomized quicksort is probabilistically optimal with overwhelming probability.
- The bound constrains **comparison** sorts only. Insertion sort's N = 4 decision tree has height 6 vs the minimum 5, a reminder that the bound forbids shorter trees, not taller ones.
- Finding genuinely optimal decision trees for general N is an open mathematics problem; optimal trees are known only up to N = 22 (OEIS A036604), with N = 16, 17, 18 resolved in 2023.
- **Stability** is covered in this week's discussion and is explicitly in scope for exams, though not developed in this lecture.
