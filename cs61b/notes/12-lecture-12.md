<!-- Wed, Sep 23, 2026 | sources: slides + textbook (no transcript available) -->
# Lecture 12: Lecture 12

This lecture (Wed, Sep 23) is a **midterm practice / exam skills session** rather than a new-material lecture: the slide deck is the "Midterm Practice" deck (carried over from Spring 2026, and explicitly banner-labeled "WARNING, THESE SLIDES HAVE NOT YET BEEN UPDATED FOR FALL 2026"), run by the course staff as a live PrairieLearn work session with two coding demos (`LinkedListDeque61B.add(...)` at an index, and "Rotate to Front"). The slides cover the general problem-solving workflow for a CBTF exam (read, test, implement, test, repeat), how the autograder scoring penalty works, how to run and debug JUnit tests in VS Code, and what to test for on a deque problem. The accompanying textbook chapter for this lecture is **Chapter 12: Asymptotics I**, which is the first real content chapter on *execution cost*: it introduces the `dup1` / `dup2` duplicate-finding example, three failed-or-tedious ways to characterize runtime (wall-clock timing, exact operation counts, symbolic operation counts), the four simplifications (worst case, cost model, drop low-order terms, drop constants), the idea of *order of growth*, and the simplified analysis process that lets you skip building a count table entirely. These notes cover both halves: the exam-practice content from the slides and the asymptotics content from the textbook chapter.

---

## Key Concepts

### Part A: Exam practice and workflow (from the slides)

**1. The general workflow.** The deck states the loop twice, once at the start and once again after the first demo, which is a strong hint that staff consider it the single most important takeaway:

1. Read the problem carefully
2. Create test cases
3. Write implementation
4. Test code
5. Repeat (steps 2 through 4 repeat as necessary)

The ordering matters. Tests come *before* implementation. The reason given on the "Testing" slide is that writing a test forces you to decide what the method is supposed to do before you decide how it does it. If you cannot write the test, you have not understood the spec, and writing code at that point is guessing.

**2. The autograder is not your test suite.** The slide has a header graphic reading "PRESSING THIS BUTTON TWICE LOSES POINTS," and the body says: only submit to the autograder once you have tested your code, because **submitting more than once reduces your overall score by 25% each time after the first run**. The stated pedagogical goal is "for you to be comfortable enough with writing your own tests and using them as your source of truth." On an exam like this, your local JUnit tests are the ground truth; the autograder is a final confirmation, not a debugging tool.

**3. The exam logistics (Spring 2026 slide values).** Midterm 1 runs Monday through Thursday in the CBTF, the exam is **uncurved** and there is **no cheat sheet**, and students should familiarize themselves with the "CBTF Getting Started for Students" page. CBTF HW 4 is four problems on PrairieLearn "in the exact same format as the exam," which is the intended dress rehearsal. *(Note: the specific dates on the slide, 2/23 through 2/26, and the HW 4 due date of 2/20, are the Spring 2026 dates. Check the Fall 2026 calendar for the actual dates.)*

**4. VS Code as the exam environment.** The CBTF exam runs in VS Code, not IntelliJ. The slides' framing is "VS Code... just another IntelliJ!" with these concrete mechanics:
- Run a single test with the **green play button next to each test**.
- Pass/fail results appear in the **Test Results** panel.
- Output from `System.out.println` appears in the **Debug Console**, not in Test Results. This is the detail people lose time on: if your print statements "disappear," you are looking at the wrong panel.
- The **NullAnnotation popup can be ignored**; it is not an error in your code.
- HW 4 Task 1 plus the VS Code Guide are the pointers for getting set up.

**5. The problem type: a deque with an indexed `add`.** `Deque61B.java` and `LinkedListDeque61B.java` "look just like Project 1." The new method is `add()`, which is "similar to `addFirst()` and `addLast()`, but with a specific index," so it "allows us to add anywhere in the list." Conceptually, `addFirst` and `addLast` are the two special cases of a general "insert at position" operation, and the general version is the same pointer surgery with a walk to find the insertion point first.

**6. What to test for on this problem.** The slides give an explicit checklist:
- Adding to the **front**, the **middle**, and the **end**.
- **Size** must be correctly updated (a very common silent bug: the list looks right but `size()` is stale).
- **Edge cases**, but the spec lets you "assume that `idx` is always within `[0, size)`," so you do not need to defend against out-of-range indices.

**7. How to test for it.** Two techniques, both from the slides:
- Use `toList()` to iterate over the deque and produce a plain `List` you can inspect.
- Compare against an expected list with `isEqualTo()`, or use `containsExactly(...).inOrder()` from the Truth assertion library.

The `.inOrder()` part matters: `containsExactly` without it checks membership regardless of order, which would pass on a deque that has the right elements in the wrong positions, exactly the bug an indexed `add` is likely to have.

### Part B: Asymptotics I (from the textbook chapter)

**8. Two kinds of cost.** Everything in CS 61B up to this point has been about **programming cost**: how long it takes you to write the code, how readable it is, how maintainable it is (the chapter stresses that most real cost is maintenance and scalability, not initial development). From here on the course is about **execution cost**, which splits into **time complexity** (how long the program takes to run) and **space complexity** (how much memory it needs).

**9. The motivating problem.** Given a **sorted** array, decide whether it contains a duplicate:

```java
List<Integer> example = [-3, -1, 2, 4, 4, 8, 10, 12];
```

The naive algorithm compares every pair of elements. The better algorithm exploits sortedness: duplicates in a sorted array must be adjacent, so you only compare each element to its immediate neighbor. Intuitively the naive one does a lot of redundant work, but "a lot more" is not a measurement. The whole chapter is about making that comparison precise.

**10. Technique 1: measure seconds.** Time the program with a stopwatch, Unix `time`, or the Princeton standard library `Stopwatch` class. The result is that as input size grows, `dup1` takes longer and longer while `dup2` stays roughly flat. This is easy to understand and easy to do, but it fails the goals: it can take a very long time to run, and running times vary by machine, compiler, and input data. It is simple but not mathematically rigorous, and the machine/compiler/input variation can obscure the real relationship between the two algorithms.

**11. Technique 2A: count operations at a fixed N.** Instead of timing, count how many operations each algorithm performs at some fixed input size (say N = 10000). This fixes **machine independence**: the operation counts do not change when you switch laptops. But it is tedious to count every operation, the chosen N is arbitrary, and it still tells you nothing about actual elapsed time.

**12. Technique 2B: symbolic counts.** Count operations as a *function of N* rather than at one fixed N. This is the first technique that shows how the algorithm **scales**, which is the thing we actually care about. It is also the most tedious of all, and still does not give wall-clock time.

**13. Asymptotic behavior.** What we really care about is behavior "for very large values of N," because the algorithms that matter are the ones that survive contact with billions of particles, billions of social network users, or billions of bytes of video. Algorithms that scale well have better asymptotic runtime behavior. This reframing is what makes the simplifications legitimate: we are not being sloppy, we are deliberately discarding information that does not affect large-N behavior.

**14. The four simplifications.** These turn an unmanageable count table into a single expression.

- **Simplification 1: consider only the worst case.** The worst case is where the most interesting effects appear, so ignore best and average cases. Concretely: in each table row, keep only the upper end of the range.
- **Simplification 2: restrict attention to one operation.** Pick a single representative operation to act as a proxy for the whole runtime. That choice is called the **cost model**. For `dup1`, good choices are the increment, the `<`, the `==`, or the array accesses, because each of these happens once per unit of "real work." Bad choices are `j = i + 1` and `i = 0`, because they do not track the loop body's work (`i = 0` happens exactly once no matter how big N is, so it can never reveal scaling).
- **Simplification 3: eliminate low-order terms.** In (N² + 3N + 2)/2, the 3N and the 2 are irrelevant for large N.
- **Simplification 4: eliminate multiplicative constants.** 3N², N²/2, and N² are "all in the family/shape of N²." The constants have no real meaning anyway, since by picking one representative operation we already threw away information about the others.

**15. Why the dominant term wins (the calculus intuition).** Suppose an algorithm has counts `<` = 100N² + 3N, `>` = N³ + 1, `&&` = 5000. Which term controls the runtime? Let `<` cost α nanoseconds, `>` cost β, and `&&` cost γ. The total is roughly α(100N² + 3N) + β(N³ + 1) + 5000γ nanoseconds. As N grows without bound, the N³ term dwarfs everything else no matter how large α is relative to β. The answer is **cubic**. This is the argument that licenses Simplifications 3 and 4: constants and low-order terms can delay the crossover, but they cannot prevent it.

**16. Order of growth, Θ, and O.** Given code, express its runtime as a function R(N), where N is a property of the input (usually its size). We do not want the exact R(N); we want its **order of growth**. Notation: Θ(f(N)) means "the same order of growth as f," while O(f(N)) can be roughly read as "grows no faster than f," that is, "less than or equal to." So 2N is in Θ(N), and also in O(N) and O(N²), but not in Θ(N²) and not in O(1).

**17. The simplified analysis process.** The painful process is: build an exact table of all operation counts, then apply the four simplifications. The chapter proposes skipping the table:

1. Choose a cost model (the representative operation).
2. Find the order of growth of that operation's count, either by
   - making an exact count and discarding the unnecessary pieces, or
   - using intuition and inspection, which is a skill that comes with practice.

Then: if the chosen operation takes constant time, R(N) ∈ Θ(f(N)) where C(N) ∈ Θ(f(N)).

---

## Definitions

- **Programming cost**: the human cost of software, comprising development time, readability, modifiability, and maintainability. The chapter emphasizes that much of a program's total cost is maintenance and scalability, not initial development.
- **Execution cost**: the machine cost of running the software, comprising time complexity and space complexity.
- **Time complexity**: how much time a program takes to execute, as a function of input size.
- **Space complexity**: how much memory a program requires, as a function of input size.
- **Naive algorithm (for duplicate detection)**: compare every pair of elements. Implemented as `dup1`.
- **Better algorithm (for duplicate detection on a sorted array)**: compare each element only to its immediate neighbor. Implemented as `dup2`.
- **Symbolic count**: an operation count expressed as a function of the input size N rather than as a number for one particular N.
- **Asymptotic behavior**: the behavior of a function for very large values of N.
- **Order of growth**: the "shape" of a function after discarding low-order terms and multiplicative constants, for example N, N², N³.
- **Cost model**: the single representative operation chosen to stand in for the whole runtime during analysis.
- **R(N)**: the runtime of a piece of code expressed as a function of N, a property of the input (usually its size).
- **C(N)**: the count of how many times the chosen representative operation occurs, as a function of N.
- **Θ(f(N))** ("Theta"): the set of functions with the *same* order of growth as f(N).
- **O(f(N))** ("Big-O"): roughly, the set of functions whose order of growth is *less than or equal to* f(N).
- **Worst case**: the input of a given size that maximizes the operation count. The case we usually (though not always) analyze.
- **`add(...)` on a deque (slides)**: like `addFirst` and `addLast`, but at a specified index, allowing insertion anywhere in the list. The spec permits assuming `idx` is always within `[0, size)`.
- **`toList()`**: a deque method that iterates over the structure and returns its contents as a `List`, used in tests to inspect the whole deque at once.
- **`containsExactly(...).inOrder()`**: a Truth assertion that the collection holds exactly the given elements, in exactly the given order.

---

## Worked Examples

### Example 1: `dup1`, the naive algorithm

```java
// Naive algorithm: compare everything
public static boolean dup1(int[] A) {
  for (int i = 0; i < A.length; i += 1) {
    for (int j = i + 1; j < A.length; j += 1) {
      if (A[i] == A[j]) {
         return true;
      }
    }
  }
  return false;
}
```

**What it does.** The outer loop fixes one element `A[i]`. The inner loop starts at `j = i + 1` (not at 0, which avoids comparing an element to itself and avoids re-checking pairs in both orders) and compares `A[i]` to every later element. If any pair matches, return `true` immediately. If the loops finish, no duplicate exists, so return `false`.

**Technique 2A / 2B, the count table.** For N = A.length:

| Operation | Symbolic Count | Count (for N = 10000) |
| --- | --- | --- |
| `i = 0` | 1 | 1 |
| `j = i+1` | 1 to N | 1 to 10000 |
| `<` | 2 to (N² + 3N + 2)/2 | 2 to 50,015,001 |
| `+= 1` | 0 to (N² + N)/2 | 0 to 50,005,000 |
| `==` | 1 to (N² - N)/2 | 1 to 49,995,000 |
| array accesses | 2 to N² - N | 2 to 99,990,000 |

Reading the table row by row: `i = 0` runs exactly once, at the very start of the call, regardless of N. `j = i + 1` runs once in the best case, which is when `A[0] == A[1]` and the method returns on the first comparison (convince yourself: the inner loop is entered once, so `j` is initialized once, and we return before the outer loop can advance). In the worst case `j` is initialized once per value of `i`, which is N times. As you go further down the table the exact counts get progressively harder to compute, which is exactly the chapter's point: this is not a sustainable way to analyze code, and the exact numbers turn out not to matter.

### Example 2: `dup2`, the better algorithm

```java
// Better algorithm: compare only neighbors
public static boolean dup2(int[] A) {
  for (int i = 0; i < A.length - 1; i += 1) {
    if (A[i] == A[i + 1]) {
      return true;
    }
  }
  return false;
}
```

**What it does.** Since the array is sorted, equal values must sit next to each other. So a single pass comparing `A[i]` to `A[i + 1]` is enough. The loop bound is `A.length - 1` so that `A[i + 1]` never runs off the end.

**Why the precondition matters.** `dup2` is only correct because the input is sorted. Run it on `[3, 1, 3]` and it returns `false` even though 3 appears twice. The efficiency gain is bought entirely with an assumption about the input.

**The count table.** (This is the checkpoint exercise from section 12.3.)

| Operation | Symbolic Count | Count (for N = 10000) |
| --- | --- | --- |
| `i = 0` | 1 | 1 |
| `i += 1` | 0 to N | 0 to 10,000 |
| `<` | 0 to N - 1 | 0 to 9,999 |
| `==` | 1 to N - 1 | 1 to 9,999 |
| array accesses | 2 to 2N - 2 | 2 to 19,998 |

The textbook prints the second row with the label `j = i+1`, carried over from the `dup1` table; `dup2` has no `j`, so the row is the loop increment. The chapter explicitly says "It's okay if you were slightly off, you want *rough* estimates," and a couple of the rows here are off by one from a strict count. That tolerance is the point: precision in the low-order terms is wasted effort.

**Comparing the two tables.** Three readings, each better than the last:
1. `dup2` uses fewer operations to accomplish the same goal.
2. `dup2` **scales** much better in the worst case: (N² + 3N + 2)/2 versus N - 1.
3. Best of all: **parabolas always grow faster than lines.** That statement is true independent of constants, hardware, and language, which is why it is the one worth keeping.

**Applying the four simplifications to `dup2`.** Take the worst case (upper ends only), pick a cost model, say array accesses, whose worst-case count is 2N - 2, drop the low-order term -2, drop the multiplicative constant 2, and you get **order of growth N**. Picking `<`, `==`, or the loop increment instead gives the same answer, which is a good sanity check that the cost model choice was reasonable.

### Example 3: `dup1` by exact count, without building a table

```java
int N = A.length;
for (int i = 0; i < N; i += 1)
   for (int j = i + 1; j < N; j += 1)
      if (A[i] == A[j])
         return true;
return false;
```

**Cost model:** the number of `==` operations.

**Step-by-step.** When `i = 0`, `j` runs from 1 to N-1, so the inner loop body executes N-1 times. When `i = 1`, `j` runs from 2 to N-1, so N-2 times. When `i = 2`, N-3 times. And so on down to 1, then 0. Summing:

```
cost = 1 + 2 + 3 + ... + (N-2) + (N-1) = N(N-1)/2
```

Expanding gives (N² - N)/2, which matches the `==` row of the `dup1` table exactly. Drop the -N/2 (low-order term) and the 1/2 (multiplicative constant) and the worst-case order of growth is **N²**.

### Example 4: `dup1` by geometric argument

Instead of summing the series, picture the pairs `(i, j)` with `j > i` as cells in an N by N grid. The comparisons occupy the strictly-upper-triangular half. That region is a right triangle with side length N - 1, and the area of a triangle with side length s is proportional to s². So the count is on the order of (N-1)², whose order of growth is **N²**. Same answer, no algebra.

The chapter is candid that this is "definitely not something that is immediately obvious. It takes time and practice to see these patterns." It is worth learning because it generalizes: a triple-nested loop with `i < j < k` carves out a tetrahedron, giving N³.

### Example 5: a loop that looks quadratic but is not

```java
for (int i = 0; i < N; i++) {
    int j = 0;
    while (j < N) {
        j = N;
    }
}
```

**Step-by-step.** Fix an iteration of the outer loop. `j` starts at 0. If N > 0, the condition `j < N` is true, so the body runs once and sets `j = N`. Now `j < N` is false and the while loop exits. The inner loop body therefore executes **exactly once**, not N times, regardless of N.

So the total work is: outer loop runs N times, constant work each. The runtime is **linear**, Θ(N).

The lesson is that nesting depth is not the answer. You have to read what the inner loop actually does.

### Example 6 (extra context): the LLD indexed `add` demo

The slides show this as a live demo without printing code, so the following is a reconstruction consistent with a Project 1 style circular-sentinel doubly linked list and the stated spec ("assume that `idx` is always within `[0, size)`"). Check the actual signature in the provided `Deque61B.java` on the exam.

```java
@Override
public void add(int index, T item) {
    // Walk to the node currently sitting at position `index`.
    Node p = sentinel.next;
    for (int i = 0; i < index; i += 1) {
        p = p.next;
    }
    // Splice a new node in directly before p.
    Node newNode = new Node(p.prev, item, p);
    p.prev.next = newNode;
    p.prev = newNode;
    size += 1;
}
```

**Box-and-pointer reasoning in words.** Picture the list as a ring of boxes, with the sentinel box at the top of the ring and `sentinel.next` pointing at the first real element. Starting `p` at `sentinel.next` and advancing it `index` times leaves `p` on the element currently at position `index`. Inserting "before `p`" is what makes position `index` belong to the new item and shifts everything from the old `p` onward one slot to the right.

The splice itself touches four arrows, two of which the constructor sets:
1. `newNode.prev` points back at `p.prev` (set by the constructor).
2. `newNode.next` points forward at `p` (set by the constructor).
3. `p.prev.next`, the forward arrow of the node before the insertion point, is redirected from `p` to `newNode`.
4. `p.prev`, the backward arrow of `p`, is redirected from the old predecessor to `newNode`.

**Order matters.** Line 3 uses `p.prev` to find the predecessor. If you wrote `p.prev = newNode;` first, you would destroy the pointer you still need and then set `newNode.next = newNode`, producing a self-loop and orphaning the front of the list. Capture the neighbor in the constructor (or in a local variable) before you overwrite anything.

**Why the sentinel makes `index == 0` free.** With a circular sentinel, `p = sentinel.next` and `p.prev == sentinel`, so the exact same four lines insert at the front with no special case. That is the payoff of the sentinel design from Project 1, and it is also why the slides describe `add` as just a generalized `addFirst`/`addLast`.

**Do not forget `size += 1`.** The slides call this out explicitly under "What should we test for": "Checking size (must correctly update size)." A list with correct contents and a stale `size` passes a `toList()` check and still fails.

### Example 7 (extra context): tests for the indexed `add`, following the slides' checklist

```java
import static com.google.common.truth.Truth.assertThat;
import org.junit.jupiter.api.Test;

public class LinkedListDeque61BTest {

    @Test
    public void addToFront() {
        Deque61B<Integer> d = new LinkedListDeque61B<>();
        d.addLast(1);
        d.addLast(2);
        d.addLast(3);
        d.add(0, 99);
        assertThat(d.toList()).containsExactly(99, 1, 2, 3).inOrder();
        assertThat(d.size()).isEqualTo(4);
    }

    @Test
    public void addToMiddle() {
        Deque61B<Integer> d = new LinkedListDeque61B<>();
        d.addLast(1);
        d.addLast(2);
        d.addLast(3);
        d.add(1, 99);
        assertThat(d.toList()).containsExactly(1, 99, 2, 3).inOrder();
        assertThat(d.size()).isEqualTo(4);
    }

    @Test
    public void addToEnd() {
        Deque61B<Integer> d = new LinkedListDeque61B<>();
        d.addLast(1);
        d.addLast(2);
        d.addLast(3);
        d.add(2, 99);
        assertThat(d.toList()).containsExactly(1, 2, 99, 3).inOrder();
        assertThat(d.size()).isEqualTo(4);
    }
}
```

Three tests, one per bullet on the "Testing" slide (front, middle, end), each also asserting on `size()`. Note the `.inOrder()` on every `containsExactly`: without it, a buggy `add` that inserts 99 at the wrong position still passes all three. You can equally write `assertThat(d.toList()).isEqualTo(List.of(1, 99, 2, 3));`, which the slides list as the alternative; `isEqualTo` on a `List` is order-sensitive by definition.

Run each with the green play button beside the method name, read pass/fail in **Test Results**, and read any `System.out.println` output in the **Debug Console**.

### Example 8 (extra context): "Rotate to Front"

The slides list this as a second demo with no code shown, so this is a reconstruction of the standard form of the problem: move the element at a given index to the front of the deque, preserving the relative order of everything else.

```java
public void rotateToFront(int index) {
    // Find the node at `index`.
    Node p = sentinel.next;
    for (int i = 0; i < index; i += 1) {
        p = p.next;
    }
    // Unlink p from where it currently is.
    p.prev.next = p.next;
    p.next.prev = p.prev;
    // Relink p directly after the sentinel.
    p.prev = sentinel;
    p.next = sentinel.next;
    sentinel.next.prev = p;
    sentinel.next = p;
}
```

**Reasoning in words.** Removal and insertion are separate phases and must not be interleaved. In the removal phase, the two neighbors of `p` are stitched to each other, which is why both `p.prev` and `p.next` are read before either is overwritten. After those two lines, `p` is detached from the ring but still reachable through the local variable `p`, and the ring is a valid, shorter list. In the insertion phase, `p` is spliced between `sentinel` and the current first element, again touching four arrows.

`size` does not change, because one removal and one insertion cancel out. Also note the order of the last two lines: `sentinel.next.prev = p;` must run before `sentinel.next = p;`, otherwise `sentinel.next` already equals `p` and you set `p.prev = p`.

**Worth testing:** `index == 0` (the element is already at the front, and the code should leave the list unchanged rather than corrupting it), the last index, a single-element deque, and the order of the remaining elements after the rotation.

**(extra context) Asymptotic tie-in.** The walk to position `index` is the dominant cost, so `rotateToFront` and the indexed `add` are both Θ(N) in the worst case on a linked list, even though the splice itself is constant time. Choosing "pointer reassignments" as your cost model here would be a *bad* cost model choice, since that count is constant and would falsely suggest constant runtime. The right cost model is the number of `p = p.next` steps.

---

## Common Pitfalls

**On the exam mechanics (slides):**

- **Submitting to the autograder to find out whether your code works.** Each submission after the first costs 25% of your score on that problem. Write and run local tests first; the slides are explicit that your own tests should be your "source of truth."
- **Looking for print output in the Test Results panel.** `System.out.println` output goes to the **Debug Console** in VS Code.
- **Treating the NullAnnotation popup as a compile error.** It can be ignored.
- **Writing code before writing tests.** The workflow is deliberately test-first, and the slide repeats the workflow after the first demo to reinforce it.
- **Defending against indices outside `[0, size)`.** The spec says you may assume the index is in range. Time spent on that is wasted.
- **Using `containsExactly(...)` without `.inOrder()`.** It passes on correctly-populated but wrongly-ordered deques, which is exactly the failure mode of an indexed insert.
- **Forgetting `size += 1`.** Very easy to miss and invisible to a contents-only test.

**On pointer manipulation (extra context, but directly relevant to both demos):**

- **Overwriting a pointer you still need.** Always capture or use a node's neighbors before you reassign its `prev`/`next`.
- **Forgetting that every splice touches four arrows** in a doubly linked list, not two.
- **Special-casing the front or back** when a circular sentinel already handles them uniformly. Extra special cases are extra bugs.

**On asymptotics (textbook):**

- **Assuming nesting depth equals the exponent.** The Chapter 12.7 example with `j = N` inside the while loop is quadratic-looking and linear in reality.
- **Assuming "input of size 2N takes 4x as long" applies at small N.** For a Θ(N²) function, going from N to 2N does roughly quadruple the time *by definition of asymptotics*, but going from 100 to 200 need not, because 100 may be too small for asymptotic behavior to have kicked in. Asymptotics describe very large inputs only.
- **Keeping constants "because they might matter."** They cannot change the order of growth. 100N² is still Θ(N²) and is still beaten by N³ for large enough N.
- **Picking a bad cost model.** `i = 0` executes exactly once regardless of N, so counting it yields Θ(1) for every loop in existence. The representative operation must scale with the work being done.
- **Confusing Θ with O.** Θ is "same order of growth"; O is "no faster than." 2N is in O(N²) and this is a true, if weak, statement. It is *not* in Θ(N²).
- **Reporting a best case when the question wants a worst case.** `dup1` returns after one comparison if `A[0] == A[1]`, so its best case is Θ(1). Unless told otherwise, report the worst case.
- **Forgetting that `dup2` requires a sorted input.** The speedup is a consequence of a precondition, not free.

---

## Likely Exam Points

**1. Order of growth of nested loops.**

*Q:* Give the worst-case order of growth of the runtime of the following, in terms of N.
```java
for (int i = 0; i < N; i += 1)
   for (int j = i + 1; j < N; j += 1)
      if (A[i] == A[j])
         return true;
return false;
```
*A:* Θ(N²). Using `==` as the cost model, the inner body runs (N-1) + (N-2) + ... + 1 = N(N-1)/2 = (N² - N)/2 times in the worst case. Drop the low-order term and the constant to get N².

**2. Loops where the inner loop does not actually run N times.**

*Q:* What is the runtime of the following in terms of N?
```java
for (int i = 0; i < N; i++) {
    int j = 0;
    while (j < N) {
        j = N;
    }
}
```
*A:* Linear, Θ(N). The inner loop immediately sets `j = N` on its first iteration, so it runs exactly once per outer iteration. Only the outer loop's N iterations matter.

**3. Θ versus O membership.**

*Q:* Let f(N) = 2N. Which of the following are true? f ∈ Θ(1), f ∈ Θ(N), f ∈ Θ(N²), f ∈ O(1), f ∈ O(N), f ∈ O(N²).
*A:* True: **f ∈ Θ(N)**, **f ∈ O(N)**, **f ∈ O(N²)**. False: Θ(1), Θ(N²), O(1). Θ means the same order of growth; O means roughly "less than or equal to" that order of growth, so a linear function is O of anything linear or larger.

**4. Scaling predictions and when they apply.**

*Q:* True or false, and justify: (a) If f(N) ∈ Θ(N²), running f on input size 2N instead of N takes roughly 4 times as long. (b) If f(N) ∈ Θ(N²), running f on input size 200 instead of 100 takes roughly 4 times as long.
*A:* (a) **True**, this is what the definition of asymptotics gives you. (b) **False**. 100 may be too small an input for the asymptotic behavior to have set in; low-order terms and constants can still dominate at that scale.

**5. Identifying the dominant term.**

*Q:* An algorithm performs 100N² + 3N less-than operations, N³ + 1 greater-than operations, and 5000 `&&` operations. What is the order of growth of its runtime?
*A:* **Θ(N³)**, cubic. Even if `<` were far more expensive per operation than `>`, the total time α(100N² + 3N) + β(N³ + 1) + 5000γ is eventually dominated by the βN³ term as N grows. The constant 100 delays the crossover but does not change the asymptotics.

**6. Choosing a valid cost model.**

*Q:* For `dup1`, which of these are reasonable cost models and which are not: `i = 0`, `j = i + 1`, `<`, `==`, array accesses?
*A:* **Reasonable:** `<`, `==`, array accesses, and the increment. Each of these tracks the work the loops actually do and gives order of growth N². **Not reasonable:** `i = 0`, which runs exactly once regardless of N, and `j = i + 1`, which only tracks the outer loop (at most N times) and so would understate the runtime as linear.

**7. Why asymptotics rather than a stopwatch.**

*Q:* Give three reasons to prefer asymptotic analysis over empirically timing your code with something like the `Stopwatch` class.
*A:* (1) Different computers run at different speeds depending on architecture, hardware, even room temperature, so the same code can produce very different times. (2) It may not be feasible to test on extremely large inputs; the run could take too long. (3) Asymptotics are language-agnostic, whereas empirical times vary hugely across languages (C is typically 10 to 400 times faster than Python for the same algorithm). (4) The worst case may occur only for inputs that are hard to construct or measure empirically.

**8. Reading a symbolic count table.**

*Q:* Apply the four simplifications to the `dup2` table below and state the order of growth.

| Operation | Symbolic Count |
| --- | --- |
| `i = 0` | 1 |
| increment | 0 to N |
| `<` | 0 to N - 1 |
| `==` | 1 to N - 1 |
| array accesses | 2 to 2N - 2 |

*A:* Take the worst case (upper ends), choose array accesses as the cost model giving 2N - 2, drop the -2, drop the 2, and get **Θ(N)**. Choosing `<`, `==`, or the increment gives the same Θ(N), so any of them is an acceptable answer. `i = 0` is not, since it is constant.

**9. Best case versus worst case.**

*Q:* What is the best-case order of growth of `dup1`, and on what input does it occur?
*A:* **Θ(1)**. If `A[0] == A[1]`, the very first `==` comparison returns `true`, so a constant number of operations run regardless of N. This is exactly why we standardize on the worst case: best-case analysis makes almost every algorithm look identical.

**10. Writing a test for an indexed deque `add` (coding, slides).**

*Q:* You are given `LinkedListDeque61B<T>` with `addFirst`, `addLast`, `size`, `toList`, and a new `add(int index, T item)` where `idx` is guaranteed in `[0, size)`. Write a JUnit test that would catch an implementation that inserts *after* position `index` instead of before it.
*A:* Insert into the middle of a known list and assert on exact order:
```java
@Test
public void addInsertsBeforeIndex() {
    Deque61B<Integer> d = new LinkedListDeque61B<>();
    d.addLast(1);
    d.addLast(2);
    d.addLast(3);
    d.add(1, 99);
    assertThat(d.toList()).containsExactly(1, 99, 2, 3).inOrder();
    assertThat(d.size()).isEqualTo(4);
}
```
The off-by-one bug produces `[1, 2, 99, 3]`, which `.inOrder()` rejects. Dropping `.inOrder()` would let the bug through, since both lists contain exactly the same four elements.

**11. Autograder policy.**

*Q:* You finish a CBTF problem and want to check your work. What should you do first, and what does an extra submission cost?
*A:* Run your **own local tests** first, using the green play button and reading Test Results and the Debug Console. Only submit once you are confident. Each submission after the first **reduces your overall score by 25%**.

---

## Summary

**Slides (midterm practice session):**
- The workflow, stated twice for emphasis: read the problem carefully, create test cases, write the implementation, test the code, repeat.
- Tests come before implementation; test-driven development also checks that you understood the spec before you started coding.
- Your own tests are the source of truth, not the autograder. Every autograder submission after the first costs 25% of your score.
- Midterm 1 is uncurved, in the CBTF, with no cheat sheet. CBTF HW 4 is in the exact same format as the exam. (Dates on the slides are the Spring 2026 dates; confirm Fall 2026 dates on the course calendar.)
- VS Code is the exam environment: green play button to run a test, Test Results for pass/fail, Debug Console for print output, NullAnnotation popup ignorable.
- The demo problem, `Deque61B` / `LinkedListDeque61B` with an indexed `add`, is Project 1 code plus one method; `add` generalizes `addFirst` and `addLast` to an arbitrary index, assuming `idx` is in `[0, size)`.
- Test adding to front, middle, and end; always assert on `size()`; use `toList()` plus `isEqualTo()` or `containsExactly(...).inOrder()`.
- The second demo was "Rotate to Front" (same pointer-manipulation skill set).

**Textbook Chapter 12, Asymptotics I:**
- The course pivots from programming cost (development, readability, maintainability) to execution cost (time and space complexity).
- Running example: detecting duplicates in a sorted array. `dup1` compares all pairs; `dup2` compares only neighbors and is correct only because the input is sorted.
- Timing in seconds is simple but machine-, compiler-, and input-dependent, and slow. Exact counts at a fixed N are machine-independent but tedious and use an arbitrary N. Symbolic counts reveal scaling but are the most tedious of all.
- We care about asymptotic behavior, meaning behavior at very large N, because that is what determines whether an algorithm survives billions of inputs.
- Four simplifications: consider only the worst case, restrict attention to one representative operation (the cost model), eliminate low-order terms, eliminate multiplicative constants.
- What survives is the **order of growth**. Parabolas always grow faster than lines, independent of constants and hardware.
- Simplified process: pick a cost model, find the order of growth of C(N) by exact count or by inspection, and if that operation is constant-time then R(N) ∈ Θ(f(N)).
- `dup1` is Θ(N²) in the worst case (via 1 + 2 + ... + (N-1) = N(N-1)/2, or via the area of a right triangle of side N-1). `dup2` is Θ(N).
- Θ means "same order of growth"; O means roughly "no faster than." 2N is in Θ(N), O(N), and O(N²).
- Asymptotic predictions (doubling N quadruples a Θ(N²) runtime) hold for large N only, not necessarily at N = 100.
