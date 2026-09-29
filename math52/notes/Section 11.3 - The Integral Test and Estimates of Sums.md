# Section 11.3 - The Integral Test and Estimates of Sums

## Summary

Section 11.2 gave a test that can only prove divergence. This section gives the first real convergence test: compare a series to an improper integral.

The idea is a picture. If $f$ is positive, decreasing, and continuous on $[1, \infty)$, and $a_n = f(n)$, then the terms $a_n$ are exactly the heights of rectangles of width $1$ sitting under or over the graph of $f$. Summing those rectangles sandwiches $\int_1^\infty f(x)\, dx$ between two copies of the series, shifted by one term. The integral and the series then either both converge or both diverge.

Three payoffs:

1. **The Integral Test.** $\int_1^\infty f(x)\, dx$ converges if and only if $\sum a_n$ converges.
2. **The $p$-series test.** $\sum \frac{1}{n^p}$ converges exactly when $p > 1$. The case $p = 1$ is the harmonic series, which diverges even though $\frac{1}{n} \to 0$. That settles the question left open in §11.2: the converse of the Basic Divergence Test is false.
3. **Error estimates.** The same rectangle picture bounds the tail $R_n = S - S_n$ between two integrals. This turns a partial sum into an interval containing the exact sum, and the lower endpoint $L_n$ of that interval is a much better approximation than $S_n$ itself.

An important limitation: the Integral Test tells you *whether* a series converges. It does not tell you the value of the sum.

## Definitions and theorems

### Theorem 1 (The Integral Test)

Let $f$ be a positive, decreasing, continuous function on $[1, \infty)$ and define $(a_n)$ by

$$a_n = f(n).$$

Then the following are equivalent:

1. The improper integral

$$\int_1^\infty f(x)\, dx$$

converges.

2. The series $\sum a_n$ converges.

### Why the Integral Test works

Work on the interval $[1,2]$ first. Here $a_1 = f(1)$ and $a_2 = f(2)$. Since $f$ is decreasing, $f(2) \le f(x) \le f(1)$ for all $x \in [1,2]$, so the rectangle of size $1 \times a_1$ is bigger than the region under the curve and the rectangle of size $1 \times a_2$ is smaller. Since the base has length $1$, each rectangle's area equals its height, and

$$a_2 < \int_1^2 f(x)\, dx < a_1.$$

The same argument runs on every interval $[n, n+1]$:

$$a_{n+1} < \int_n^{n+1} f(x)\, dx < a_n.$$

Adding all of these up over $n = 1, 2, 3, \dots$ gives

$$\sum_{k=1}^\infty a_{k+1} = \sum_{k=2}^\infty a_k \le \int_1^\infty f(x)\, dx \le \sum_{k=1}^\infty a_k.$$

The reindexing in the first equality is just shifting the dummy variable: as $k$ runs over $1, 2, 3, \dots$, the index $k+1$ runs over $2, 3, 4, \dots$.

**Convergent integral gives convergent series.** The partial sums

$$S_n = \sum_{k=1}^n a_k$$

form an increasing sequence, since every $a_k > 0$ means $S_{n+1} = S_n + a_{n+1} > S_n$. If $\int_1^\infty f(x)\, dx$ exists, the left inequality above bounds $\sum_{k=2}^\infty a_k$ above by that finite number, so the increasing partial sums are bounded. The Monotone Convergence Theorem then says $\sum_{k=2}^\infty a_k$ converges, and adding back the single term $a_1$ gives that $(S_n)$ converges as well.

**Divergent integral gives divergent series.** If

$$\int_1^\infty f(x)\, dx = \infty,$$

then the right inequality says the larger term $\sum_{k=1}^\infty a_k$ also diverges to $\infty$.

### Definition 1 ($p$-series)

Let $p \in \mathbf{R}$. The sum

$$\sum \frac{1}{n^p}$$

is a **$p$-series**.

### Theorem 2 ($p$-series test)

The $p$-series

$$\sum \frac{1}{n^p}$$

converges if and only if $p > 1$.

### Definition 2 (Harmonic Series)

We call

$$\sum_{n=1}^\infty \frac{1}{n}$$

the **harmonic series**.

This is the $p$-series with $p = 1$, so by Theorem 2 it diverges.

### Theorem 3 (Integral Test Error Estimate)

Assume $f$ and $(a_n)$ satisfy the hypotheses of the Integral Test and

$$S = \sum_{k=1}^\infty a_k$$

converges. Then

$$\int_{n+1}^\infty f(x)\, dx < S - S_n < \int_n^\infty f(x)\, dx$$

where $S_n$ is the $n$th partial sum of $S$.

### Why the error estimate works

We want to approximate the tail $R_n$,

$$0 < R_n = S - S_n = \sum_{k=n+1}^\infty a_k = a_{n+1} + a_{n+2} + \cdots,$$

or how close the full sum $S$ is to the partial sum $S_n$. It is positive because every $a_k > 0$.

Place rectangles of width $1$ and heights $a_{n+1}, a_{n+2}, \dots$ starting at $x = n+1$. They sit under the curve, so their total area exceeds the area under the curve:

$$\int_{n+1}^\infty f(x)\, dx < \sum_{k=n+1}^\infty a_k.$$

Now shift the rectangles to the left so they start at $x = n$, and compare their area to $f$'s integral over $[n, \infty)$. Each rectangle's height is now the value of $f$ at its left endpoint, which by decreasingness is the largest value of $f$ on that subinterval, so the rectangles cover the region under the curve:

$$\sum_{k=n+1}^\infty a_k < \int_n^\infty f(x)\, dx.$$

Hence

$$\int_{n+1}^\infty f(x)\, dx < \sum_{k=n+1}^\infty a_k < \int_n^\infty f(x)\, dx,$$

which is Theorem 3 once the middle term is recognized as $R_n = S - S_n$.

### Remark (two equivalent forms)

The following are equivalent:

1. $$\int_{n+1}^\infty f(x)\, dx < S - S_n < \int_n^\infty f(x)\, dx$$

2. $$S_n + \int_{n+1}^\infty f(x)\, dx < S < S_n + \int_n^\infty f(x)\, dx$$

To pass from the first to the second, add $S_n$ to each of the three terms:

$$S_n + \int_{n+1}^\infty f(x)\, dx < S_n + \sum_{k=n+1}^\infty a_k < S_n + \int_n^\infty f(x)\, dx,$$

and the middle term is $S$ since $S = S_n + R_n$.

### The bounds $L_n$ and $U_n$

The term $S - S_n$ measures the size of the error between the exact sum and the partial sum. The terms

$$L_n = S_n + \int_{n+1}^\infty f(x)\, dx \qquad \text{and} \qquad U_n = S_n + \int_n^\infty f(x)\, dx$$

are altogether different objects: they are the endpoints of an interval trapping $S$. On the number line the order is

$$S_n < L_n < S < U_n.$$

We have

$$S \in (L_n, U_n)$$

and

$$S_n < L_n$$

because $\int_{n+1}^\infty f(x)\, dx > 0$. This means $L_n$ is a better approximation of $S$ than $S_n$ is: it starts at $S_n$ and moves toward $S$ without overshooting.

## Methods

### Applying the Integral Test

1. Write down the candidate function $f$ by replacing $n$ with $x$ in $a_n$.
2. Verify all three hypotheses on $[1, \infty)$ (or on $[N, \infty)$ for some $N$, since finitely many terms do not affect convergence):
   - **positive**: $f(x) > 0$,
   - **decreasing**: often read off directly, for instance $1/x^p$ decreases because $x^p$ is positive and increasing; otherwise check $f'(x) < 0$,
   - **continuous**: usually immediate for rational, exponential, or logarithmic expressions away from their singularities.
3. Evaluate $\int_1^\infty f(x)\, dx$ as a limit: $\displaystyle \lim_{b \to \infty} \int_1^b f(x)\, dx$.
4. A convergent integral means a convergent series; a divergent integral means a divergent series.
5. Do **not** report the value of the integral as the value of the sum.

### Recognizing a $p$-series

If the terms are $1/n^p$, skip the integral and quote Theorem 2: converges iff $p > 1$. This covers $\sum \frac{1}{n^2}$ (converges, $p = 2$), $\sum \frac{1}{n}$ (diverges, $p = 1$), $\sum \frac{1}{\sqrt{n}}$ (diverges, $p = 1/2$).

### Estimating a sum and its error

Given a convergent $\sum a_k$ meeting the Integral Test hypotheses:

1. Compute $S_n$ by direct addition for a manageable $n$.
2. Compute the two tail integrals $\int_{n+1}^\infty f$ and $\int_n^\infty f$.
3. Theorem 3 gives $\int_{n+1}^\infty f < S - S_n < \int_n^\infty f$, so the error of the approximation $S \approx S_n$ is at most $\int_n^\infty f(x)\, dx$.
4. For a sharper answer, form $L_n$ and $U_n$, report $S \approx L_n$, and bound the error by $U_n - L_n$.

### Choosing $n$ for a prescribed accuracy

To force the error of $S_n$ below a tolerance $\varepsilon$, it is enough to make the upper bound on the error smaller than $\varepsilon$:

$$\int_n^\infty f(x)\, dx < \varepsilon.$$

Evaluate the integral as a function of $n$, solve the resulting inequality for $n$, and pick any integer satisfying it.

## Worked examples

### Example 1: does $\sum \frac{1}{n^2}$ converge?

Let

$$f(x) = \frac{1}{x^2}$$

for $x > 0$. With $x^2$ positive and increasing, $f$ is positive and decreasing. Clearly $f$ is continuous.

Now

$$\int_1^\infty \frac{1}{x^2}\, dx = \lim_{b \to \infty} \left( -\frac{1}{x} \right) \Big|_{x=1}^{b} = \lim_{b \to \infty} \left( -\frac{1}{b} + 1 \right) = 1.$$

The antiderivative is the power rule: $\int x^{-2}\, dx = \frac{x^{-1}}{-1} = -\frac{1}{x}$. Evaluating at the endpoints gives $-\frac{1}{b} - \left(-\frac{1}{1}\right) = -\frac{1}{b} + 1$, and $\frac{1}{b} \to 0$.

This tells us

$$\sum \frac{1}{n^2}$$

converges by the Integral Test. However, we do not know the value of the sum. In particular, the sum is not $1$; the integral's value says nothing about the sum's value.

### Example 2: does $\sum \frac{1}{n}$ converge?

Let

$$f(x) = \frac{1}{x}$$

for $x > 0$. With $x$ positive and increasing, $f$ is positive and decreasing. Clearly $f$ is continuous.

Now

$$\int_1^\infty \frac{1}{x}\, dx = \lim_{b \to \infty} \ln|x| \Big|_{x=1}^{b} = \lim_{b \to \infty} \left( \ln(b) - \ln(1) \right) = \infty,$$

using $\ln(1) = 0$ and $\ln(b) \to \infty$.

This tells us

$$\sum \frac{1}{n}$$

diverges by the Integral Test. This happens even though

$$\lim_{n \to \infty} \frac{1}{n} = 0.$$

Thus the terms in the series do not tend to zero fast enough to ensure convergence. This is the promised counterexample to the converse of the Basic Divergence Test.

The rate at which the series grows is very slow:

$$\sum_{k=1}^{2^{64}} \frac{1}{k} \approx 45.$$

Over $10^{19}$ terms buy a partial sum of about $45$, and the series still runs off to infinity.

### Example 3: approximating $S = \sum_{k=1}^\infty \frac{1}{k^2}$ with $S_{10}$

**Part 1: approximate $S$ using $S_{10}$ and estimate the error.**

We have

$$S_{10} = \sum_{k=1}^{10} \frac{1}{k^2} \approx 1.5498.$$

The Integral Test Error Estimate gives

$$\int_{11}^\infty \frac{1}{x^2}\, dx < S - S_{10} < \int_{10}^\infty \frac{1}{x^2}\, dx.$$

Since

$$\int_n^\infty \frac{1}{x^2}\, dx = \lim_{b \to \infty} \left( -\frac{1}{x} \right) \Big|_{x=n}^{b} = \lim_{b \to \infty} \left( -\frac{1}{b} + \frac{1}{n} \right) = \frac{1}{n},$$

we have

$$\frac{1}{11} < S - S_{10} < \frac{1}{10}.$$

Thus

$$S \approx 1.5498$$

with an error less than $0.1$.

**Part 2: how many terms give an error less than $10^{-2}$?**

To obtain an error less than $10^{-2}$, it is enough to choose $n$ such that

$$\int_n^\infty \frac{1}{x^2}\, dx = \frac{1}{n} < \frac{1}{10^2}.$$

Taking reciprocals of both positive sides reverses the inequality, so this occurs when

$$10^2 < n.$$

Therefore, we can choose

$$n = 10^2 + 1 = 101$$

to get

$$S - S_{101} < 10^{-2}.$$

### Example 4: a better approximation of $S = \sum_{k=1}^\infty \frac{1}{k^2}$ using $L_{10}$

We have

$$L_{10} = S_{10} + \int_{11}^\infty \frac{1}{x^2}\, dx = S_{10} + \frac{1}{11} \approx 1.5498 + 0.0909 \approx 1.6407$$

and

$$U_{10} = S_{10} + \int_{10}^\infty \frac{1}{x^2}\, dx = S_{10} + \frac{1}{10} \approx 1.5498 + 0.1 \approx 1.6498.$$

Therefore,

$$S \in [1.6407, 1.6498].$$

In particular,

$$S \approx L_{10} \approx 1.6407$$

with error

$$0 < S - L_{10} < \underbrace{U_{10} - L_{10}}_{\text{length of } [L_{10},\, U_{10}]} = \frac{1}{10} - \frac{1}{11} = \frac{11 - 10}{110} = \frac{1}{110} \approx 0.0091 < 10^{-2}.$$

The lower bound holds because $S > L_{10}$, and the upper bound holds because both $S$ and $L_{10}$ lie in $[L_{10}, U_{10}]$, so their difference cannot exceed the interval's length.

Only using $S_{10}$ gave an error smaller than $0.1$. To obtain an error bound for $S_n$ comparable to the $0.0091$ bound for $L_{10}$, we need $n = 101$. Ten terms plus one integral beat one hundred and one terms.

### Remark

With some work, we can show

$$\sum_{k=1}^\infty \frac{1}{k^2} = \frac{\pi^2}{6}.$$

Numerically $\pi^2/6 \approx 1.6449$, which sits inside $[1.6407, 1.6498]$ as Example 4 predicts. The Integral Test never produces this value; it only certifies that some finite value exists.

## Common mistakes and tips

- **The integral's value is not the sum.** Example 1 has $\int_1^\infty x^{-2}\, dx = 1$ but $\sum 1/n^2 = \pi^2/6 \approx 1.6449$. The test detects convergence, nothing more.
- **Check all three hypotheses.** Positive, decreasing, continuous. If $f$ oscillates or increases somewhere, the rectangle comparison breaks and the theorem does not apply.
- **Decreasing eventually is enough.** Convergence is unaffected by finitely many terms, so if $f$ only becomes decreasing past some $x = N$, run the test on $[N, \infty)$.
- **$a_n \to 0$ proves nothing.** The harmonic series is the standard counterexample. The Basic Divergence Test from §11.2 can only conclude divergence, never convergence.
- **Know the $p = 1$ boundary cold.** $p > 1$ converges, $p \le 1$ diverges, and the borderline case $p = 1$ diverges. Watch for disguises: $\sum \frac{1}{\sqrt{n}}$ is $p = 1/2$ and diverges, $\sum \frac{1}{n\sqrt{n}}$ is $p = 3/2$ and converges.
- **Mind which endpoint goes with which bound.** The tail $R_n$ is bounded below by the integral from $n+1$ and above by the integral from $n$. The larger interval of integration gives the larger bound.
- **$L_n$ and $U_n$ are not errors.** They are estimates of $S$ itself. The error in $S \approx L_n$ is bounded by $U_n - L_n$, not by $L_n$.
- **Prefer $L_n$ over $S_n$.** When a problem asks for the best approximation from a given number of terms, adding $\int_{n+1}^\infty f$ costs one integral and typically buys an order of magnitude in accuracy.
- **Reciprocals flip inequalities.** From $\frac{1}{n} < \frac{1}{10^2}$ with both sides positive you get $n > 10^2$, and then you must pick an integer strictly larger, here $n = 101$.

## Formula sheet

**Integral Test.** $f$ positive, decreasing, continuous on $[1,\infty)$, $a_n = f(n)$:

$$\int_1^\infty f(x)\, dx \text{ converges} \iff \sum a_n \text{ converges}.$$

**Rectangle sandwich behind the test.**

$$a_{n+1} < \int_n^{n+1} f(x)\, dx < a_n, \qquad \sum_{k=2}^\infty a_k \le \int_1^\infty f(x)\, dx \le \sum_{k=1}^\infty a_k.$$

**$p$-series.**

$$\sum \frac{1}{n^p} \text{ converges} \iff p > 1.$$

**Harmonic series.**

$$\sum_{n=1}^\infty \frac{1}{n} \text{ diverges}, \qquad \sum_{k=1}^{2^{64}} \frac{1}{k} \approx 45.$$

**Tail / remainder.**

$$R_n = S - S_n = \sum_{k=n+1}^\infty a_k > 0, \qquad S = S_n + R_n.$$

**Integral Test Error Estimate.**

$$\int_{n+1}^\infty f(x)\, dx < S - S_n < \int_n^\infty f(x)\, dx$$

equivalently

$$S_n + \int_{n+1}^\infty f(x)\, dx < S < S_n + \int_n^\infty f(x)\, dx.$$

**Bounding interval.**

$$L_n = S_n + \int_{n+1}^\infty f(x)\, dx, \qquad U_n = S_n + \int_n^\infty f(x)\, dx, \qquad S_n < L_n < S < U_n.$$

$$0 < S - L_n < U_n - L_n.$$

**Useful integrals from this section.**

$$\int_n^\infty \frac{1}{x^2}\, dx = \frac{1}{n}, \qquad \int_1^\infty \frac{1}{x^2}\, dx = 1, \qquad \int_1^\infty \frac{1}{x}\, dx = \infty.$$

**Known value (stated, not proved here).**

$$\sum_{k=1}^\infty \frac{1}{k^2} = \frac{\pi^2}{6} \approx 1.6449.$$
