# MATH 52 — Section 11.1: Sequences

Stewart 8th ed. | UC Berkeley, Fall 2026

---

## 1. What a sequence is

**Definition #1 (Sequence).** A **sequence** $(a_n)$ is an ordered list of numbers of the form

$$(a_1, a_2, a_3, \ldots, a_n, a_{n+1}, \ldots).$$

The book writes $\{a_n\}$ for the same object. Both notations mean the same thing: a term for each index $n$.

**Examples from lecture.**

$$a_n = n$$

generates the natural numbers, and

$$(b_n) = (0, 1, -1, 2, -2, \ldots)$$

generates the integers.

**Order matters.** The sequence

$$(c_n) = (2, 1, 3, 4, \ldots)$$

is *not* equal to $(a_n) = (1,2,3,4,\ldots)$, since

$$1 = a_1 \neq c_1 = 2.$$

Two sequences are equal exactly when all of their terms match: $a_n = b_n$ for all $n$. A single index disagreeing is enough to make them different.

---

## 2. Two ways to define a sequence

### Explicit

A formula for $a_n$ directly in terms of $n$.

**Example #1.** List the first three terms of

$$a_n = \frac{1}{3^n} \quad \text{and} \quad b_n = \frac{(-1)^n n}{n^2+1}.$$

For $a_n$, substitute $n = 1, 2, 3$:

$$a_1 = \frac{1}{3^1} = \frac13, \qquad a_2 = \frac{1}{3^2} = \frac19, \qquad a_3 = \frac{1}{3^3} = \frac{1}{27}.$$

For $b_n$, track the sign from $(-1)^n$ separately from the size:

$$b_1 = \frac{(-1)^1 \cdot 1}{1^2+1} = \frac{-1}{1+1} = -\frac12,$$

$$b_2 = \frac{(-1)^2 \cdot 2}{2^2+1} = \frac{1 \cdot 2}{4+1} = \frac25,$$

$$b_3 = \frac{(-1)^3 \cdot 3}{3^2+1} = \frac{-1 \cdot 3}{9+1} = -\frac{3}{10}.$$

The factor $(-1)^n$ is negative on odd $n$ and positive on even $n$. That alternating sign shows up constantly in this chapter.

### Recursive / implicit

Each term is defined from earlier terms, plus a starting value.

**Example #2.** If

$$c_{n+1} = 2c_n + 1 \quad \text{and} \quad c_1 = 1,$$

find $c_3$.

You have to walk up from the seed, one index at a time:

$$c_1 = 1,$$
$$c_2 = 2c_1 + 1 = 2(1) + 1 = 2 + 1 = 3,$$
$$c_3 = 2c_2 + 1 = 2 \cdot 3 + 1 = 6 + 1 = 7.$$

In general this sequence has the closed form

$$c_n = 2^n - 1,$$

and the proof (induction) is a Math 55 topic. Check it against the terms found: $2^1 - 1 = 1$, $2^2 - 1 = 3$, $2^3 - 1 = 7$.

---

## 3. Convergence, informally

**Informal Definition #1 (Convergence).** Let $(a_n)$ be a sequence and $L \in \mathbf{R}$. If we can make the terms $a_n$ as close to $L$ as we like by taking $n$ sufficiently large, then $L$ is the **limit** of $(a_n)$. Write

$$\lim_{n \to \infty} a_n = L \quad \text{or} \quad a_n \to L,$$

and say $(a_n)$ **converges** to $L$.

The idea is borrowed directly from limits of functions. For a function,

$$\lim_{x \to \infty} f(x) = L$$

means $f(x) \approx L$ for all $x$ large. So

$$a_n \to L$$

means $a_n \approx L$ for all $n$ big. In broad strokes: the terms are clustering near $L$.

The lecture pairs two pictures. A continuous graph $y = f(x)$ oscillating with decaying amplitude toward a horizontal dashed line at height $L$, and the same behavior as isolated dots at integer inputs $n$. A sequence is the sampled version of that graph, so the limit question is the same question.

**Lecture example.** If

$$a_n = 3 + \frac{(-1)^n}{n},$$

then $a_n \to 3$. The perturbation $(-1)^n / n$ flips sign every term but shrinks in size, so the dots bounce above and below $3$ while closing in on it. Plotted for $n$ up to $15$, the points hug the dashed line $y = 3$ tighter and tighter.

Note what convergence does *not* require: the terms never have to equal $L$, and they do not have to approach from one side.

---

## 4. Divergence

**Definition #2 (Divergence).** Let $(a_n)$ be a sequence. If

$$\lim_{n \to \infty} a_n$$

does not exist, then $(a_n)$ **diverges**.

The lecture names two causes.

**Cause 1: infinitely many terms cluster near two (or more) distinct points in $\mathbf{R}$.**

$$(a_n) = (1, -1, 1, -1, \ldots).$$

Infinitely many terms sit at $1$ and infinitely many sit at $-1$. There is no single value the terms settle near, so no limit exists. The plot is two flat rows of dots at heights $1.0$ and $-1.0$.

**Cause 2: infinitely many terms grow without bound.**

$$a_n = n \quad \text{or} \quad a_n = (-1)^n n.$$

So infinitely many terms of $(|a_n|)$ grow large. For $a_n = (-1)^n n$ the plot fans outward, alternating sign with magnitude increasing roughly linearly, reaching about $\pm 15$ by $n = 15$. Unbounded terms cannot cluster near any finite $L$.

---

## 5. Why $\epsilon$ shows up: equality restated

Before the formal limit definition, the lecture reframes equality of real numbers in terms of arbitrarily small tolerances.

**Definition #3 (Equality).** Let $x, y \in \mathbf{R}$. The following are equivalent:

1. $x = y$
2. for all $\epsilon > 0$ we have $|x - y| < \epsilon$.

Think of $\epsilon$ as very small. The lecture works the concrete case $y = 7$ with $\epsilon = 10^{-k}$.

Assume

$$|x - 7| < 10^{-k} \qquad (*)$$

for all $k > 0$. Geometrically, $x$ lies inside the open interval $(7 - 10^{-k},\ 7 + 10^{-k})$ for every $k$, and that interval shrinks around $7$. This says $x$ and $7$ are infinitely close to each other: as $k \to \infty$,

$$|x - 7| \to 0.$$

Now suppose $x \neq 7$. Then there is actual distance between the two numbers,

$$|x - 7| > 0.$$

Since $|x-7|$ is a fixed positive number and $10^{-k} \to 0$, we can choose $k_0$ large enough that

$$|x - 7| \geq 10^{-k_0},$$

meaning $x$ sits outside the interval $(7 - 10^{-k_0},\ 7 + 10^{-k_0})$. That contradicts $(*)$, which demanded $x$ be inside every such interval. Thus $x = 7$.

The takeaway: "closer than every positive tolerance" is the same as "equal." That is exactly the machinery the limit definition borrows.

---

## 6. The formal limit definition

**Definition #4 (Limit).** Let $(a_n)$ be a sequence and $L \in \mathbf{R}$. If for every $\epsilon > 0$ we have an index $N \in \mathbf{N}$ such that

$$n > N \quad \text{implies} \quad |a_n - L| < \epsilon,$$

then $(a_n)$ converges to $L$.

Read it as a game. Someone hands you a tolerance $\epsilon$, however small. You must produce a cutoff index $N$ past which every single term is within $\epsilon$ of $L$. If you can always do this, the limit is $L$.

**Picture.** On a number line, mark the band $(L - \epsilon, L + \epsilon)$. Terms $a_{N+1}, a_{N+k}, \ldots$ all lie inside the band, while an early term like $a_5$ may sit outside it. The summary from lecture:

> For any $\epsilon > 0$, all but finitely many $a_n$'s are in the interval $(L - \epsilon, L + \epsilon)$, and it does not matter how small the interval is.

Two consequences worth holding onto:

- $N$ is allowed to depend on $\epsilon$. Smaller $\epsilon$ generally forces larger $N$.
- Finitely many badly behaved terms never affect convergence. Only the tail matters.

---

## 7. Limit laws

**Theorem #1 (Basic Limit Laws).** Assume $a_n \to a$ and $b_n \to b$. Then

1. $1/n \to 0$
2. $a_n + b_n \to a + b$
3. $k a_n \to ka$, for $k \in \mathbf{R}$
4. $a_n b_n \to ab$
5. $\dfrac{a_n}{b_n} \to \dfrac{a}{b}$ provided $b \neq 0$
6. $a_n \to 0$ if and only if $|a_n| \to 0$.

Law 1 is the workhorse for rational expressions. Law 6 is the tool for alternating sequences: it converts a sign-flipping sequence into a positive one, but note it only applies when the limit is $0$. It says nothing about $a_n \to L$ for $L \neq 0$, and indeed $|(-1)^n| \to 1$ while $(-1)^n$ has no limit.

### Example #3

Find

$$\lim_{n \to \infty} \sqrt{\frac{1 + 4n^2}{1 + n^2}}.$$

Reuse the Math 51 technique: multiply all terms by $n^{-p}$, where $p$ is the highest power of $n$ appearing in

$$\left(\frac{1 + 4n^2}{1 + n^2}\right)^{1/2} \quad \Rightarrow \quad p = 2.$$

Divide numerator and denominator of the inside fraction by $n^2$:

$$\frac{1 + 4n^2}{1 + n^2} = \frac{(1 + 4n^2)/n^2}{(1+n^2)/n^2} = \frac{1/n^2 + 4}{1/n^2 + 1}.$$

So

$$\left(\frac{1 + 4n^2}{1 + n^2}\right)^{1/2} = \left(\frac{1/n^2 + 4}{1/n^2 + 1}\right)^{1/2} \to \frac{4^{1/2}}{1} = 2.$$

By Law 1, $1/n \to 0$, and by Law 4 applied to $1/n \cdot 1/n$, $1/n^2 \to 0$. The numerator inside tends to $0 + 4 = 4$ and the denominator to $0 + 1 = 1$, so by Law 5 the fraction tends to $4$. The limit laws then give

$$\lim_{n \to \infty} \left(\frac{1 + 4n^2}{1 + n^2}\right)^{1/2} = 2.$$

---

## 8. Squeeze Theorem

**Theorem #2 (Squeeze Theorem).** Suppose

$$a_n \to L \quad \text{and} \quad c_n \to L.$$

If $(b_n)$ satisfies

$$a_n \leq b_n \leq c_n$$

for all $n \geq n_0$, then $b_n \to L$.

Note the hypothesis is only needed for $n \geq n_0$, not for every $n$. Again, tails are what count.

**Lecture example.** If

$$-\frac{1}{n^2} < b_n \leq \frac{1}{n},$$

then $b_n \to 0$ by the Squeeze Theorem, since both $-1/n^2 \to 0$ and $1/n \to 0$. The accompanying plot shows the upper bound $1/n$ in orange descending toward $0$ from above and the lower bound $-1/n^2$ in blue rising toward $0$ from below, with the gap between them closing.

The theorem works in a similar manner to the Comparison Test for improper integrals: control an unknown quantity by trapping it between two things you understand.

This is the standard move for alternating sequences. Combined with Law 6, bounding $|b_n|$ between $0$ and something that tends to $0$ forces $b_n \to 0$.

---

## 9. Continuous functions and limits

**Theorem #3.** If $a_n \to L$ and $f$ is continuous at $L$, then

$$\lim_{n \to \infty} f(a_n) = f(L).$$

Continuity lets you push the limit inside the function.

**Lecture example.** If $a_n \to 3$ and $f(x) = x^2 + e^x$, then

$$f(a_n) = a_n^2 + e^{a_n} \to 3^2 + e^3$$

as $n \to \infty$.

The condition is continuity **at $L$**, the limit point, not everywhere. This theorem is what justifies moving a limit through roots, exponentials, logs, and trig functions.

---

## 10. Monotone sequences

**Definition #5 (Increasing/Decreasing).** If

$$a_n < a_{n+1}$$

for all $n$, then $(a_n)$ is an **increasing** sequence. If

$$a_{n+1} < a_n$$

for all $n$, then $(a_n)$ is a **decreasing** sequence. A sequence is **monotone** if it is either increasing or decreasing.

The increasing plot in lecture rises steeply at first and then flattens as $n$ approaches $15$. Increasing does not mean growing without bound; it only means never stepping backward.

---

## 11. Bounded sequences

**Definition #6 (Bounded Above and Below).** If $M \in \mathbf{R}$ satisfies

$$a_n \leq M$$

for all $n$, then $(a_n)$ is **bounded above** by $M$. If $m$ satisfies

$$a_n \geq m$$

for all $n$, then $(a_n)$ is **bounded below** by $m$.

**Lecture example.** We have

$$\frac1n < 5$$

and

$$\frac{(-1)^n}{n} > -2$$

for all $n$. Bounds are not required to be tight. Any number that works counts.

**Definition #7 (Bounded).** If

$$m \leq a_n \leq M$$

for all $n \in \mathbf{N}$, then $(a_n)$ is a **bounded** sequence.

Bounded means bounded above *and* below at once.

---

## 12. Worked monotonicity examples

### Example #4

Determine whether the sequence is increasing, decreasing, or not monotone:

$$a_n = \frac{1 - n}{2 + n}.$$

First get $a_{n+1}$ by substituting $n + 1$ everywhere $n$ appears:

$$a_{n+1} = \frac{1 - (n+1)}{2 + (n+1)} = \frac{1 - n - 1}{n + 3} = \frac{-n}{3 + n}.$$

Now test the guess $a_n > a_{n+1}$:

$$\frac{1-n}{2+n} > \frac{-n}{3+n}.$$

Both $n + 3 > 0$ and $n + 2 > 0$, so cross-multiplying preserves the inequality direction:

$$(3+n)(1-n) > -n(2+n).$$

Expand the left side: $(3+n)(1-n) = 3 - 3n + n - n^2 = 3 - 2n - n^2$. Expand the right: $-n(2+n) = -2n - n^2$. So

$$3 - 2n - n^2 > -2n - n^2.$$

Add $2n + n^2$ to both sides:

$$3 > 0.$$

Since $3 > 0$ is true, and every step is reversible (the cross-multiplication was by positive quantities), it follows that $a_n > a_{n+1}$. This shows $(a_n)$ is **decreasing**.

The logic here runs backwards on purpose: start from the inequality you want, simplify to something obviously true, then note the steps reverse.

### Example #5

Determine whether the sequence is increasing, decreasing, or not monotone:

$$b_n = 3 - 2ne^{-n}.$$

Is the sequence bounded?

Instead of comparing $b_n$ to $b_{n+1}$ algebraically, pass to the continuous version and differentiate. Let

$$f(x) = 3 - 2xe^{-x}.$$

By the product rule on $2xe^{-x}$: the derivative is $2e^{-x} + 2x(-e^{-x}) = 2e^{-x} - 2xe^{-x}$. With the leading minus sign,

$$f'(x) = -\left(2e^{-x} - 2xe^{-x}\right) = 2xe^{-x} - 2e^{-x} = (2x - 2)e^{-x}.$$

Since $e^{-x} > 0$ always, the sign of $f'(x)$ is the sign of $2x - 2$. So $f'(x) > 0$ for $x > 1$, and we know $f$ and $b_n$ are increasing.

For the limit, as $n \to \infty$,

$$\frac{n}{e^n} \to 0$$

by L'Hospital (the quotient is $\infty/\infty$, and differentiating top and bottom gives $1/e^n \to 0$). Since $2ne^{-n} = 2 \cdot \frac{n}{e^n} \to 0$, we get

$$b_n = 3 - 2ne^{-n} \to 3 - 0 = 3.$$

So $b_n \to 3$ and is bounded below by $b_1$, and above by $3$. The sequence climbs toward $3$ without ever reaching it, and $b_1$ is the smallest term because the sequence is increasing.

The derivative method is usually faster than the $a_n$ vs. $a_{n+1}$ comparison when exponentials or logs are involved. The comparison method is better when the expression is a ratio of polynomials.

---

## 13. Monotone Convergence Theorem

**Theorem #4 (Monotone Convergence Theorem).** If $(a_n)$ is monotone and bounded, then $(a_n)$ converges.

From lecture: this result is mainly used for technical arguments in later sections. Intuitively, it makes sense. We are rising or falling to a limiting value.

The plot shows an increasing sequence of dots climbing and flattening against a dashed horizontal line at height $L$.

What makes this theorem powerful is that it gives convergence **without producing the limit**. You verify two structural facts, monotone and bounded, and convergence follows. Later sections use exactly this to prove a series converges when its value is unknown.

Both hypotheses are needed. $a_n = n$ is monotone but unbounded and diverges. $a_n = (-1)^n$ is bounded but not monotone and diverges.

---

## 14. Quick reference

| Concept | Statement |
|---|---|
| Sequence | Ordered list $(a_1, a_2, \ldots)$, written $(a_n)$ or $\{a_n\}$ |
| Equality | $a_n = b_n$ for all $n$ |
| Converges | $a_n \to L$: for every $\epsilon > 0$ there is $N$ with $n > N \Rightarrow \lvert a_n - L\rvert < \epsilon$ |
| Diverges | $\lim_{n\to\infty} a_n$ does not exist |
| Two causes of divergence | Clustering near two or more points; terms growing without bound |
| Limit laws | Sum, scalar, product, quotient ($b \neq 0$); $1/n \to 0$; $a_n \to 0 \iff \lvert a_n\rvert \to 0$ |
| Squeeze | $a_n \le b_n \le c_n$ for $n \ge n_0$, both ends $\to L$, then $b_n \to L$ |
| Continuity | $a_n \to L$, $f$ continuous at $L$ $\Rightarrow f(a_n) \to f(L)$ |
| Increasing / decreasing | $a_n < a_{n+1}$ / $a_{n+1} < a_n$ for all $n$; monotone if either |
| Bounded | $m \le a_n \le M$ for all $n$ |
| Monotone Convergence | Monotone and bounded $\Rightarrow$ converges |

## 15. Strategy checklist

**To compute a limit:**

1. Rational in $n$: divide top and bottom by $n^p$, the highest power present, then apply $1/n \to 0$ and the limit laws (Example #3).
2. Alternating sign: bound $\lvert a_n \rvert$ and use Law 6, or set up a Squeeze.
3. Trapped between two known sequences: Squeeze Theorem.
4. Outer continuous function applied to a convergent inner sequence: Theorem #3, push the limit inside.
5. Exponential or log against a polynomial: pass to $f(x)$ and use L'Hospital, as with $n/e^n \to 0$ in Example #5.

**To classify monotonicity:**

1. Ratio of polynomials: compute $a_{n+1}$, set up the inequality you suspect, cross-multiply by positive denominators, simplify to a true statement (Example #4).
2. Exponential or log present: let $f(x)$ be the continuous version and check the sign of $f'(x)$ (Example #5).

**To conclude convergence without a formula for the limit:** show monotone, show bounded, cite Theorem #4.
