# Section 11.5 - Alternating Series

## Summary

This section handles series whose terms flip sign every step: $\sum (-1)^n b_n$ with $b_n \ge 0$. The main result is the Alternating Series Test (AST), which says that if the sizes $b_n$ decrease to $0$, the series converges. That is a much weaker requirement than what the comparison, integral, or $p$-series tests need, which is why series like the alternating harmonic series converge even though the harmonic series does not.

The proof idea runs through the alternating harmonic series: group the terms in pairs so the even partial sums $S_{2n}$ increase and stay bounded, apply the Monotone Convergence Theorem, then show the odd partial sums $S_{2n+1}$ have the same limit because the gap between them shrinks to $0$. The partial sums oscillate around the true sum $S$, with each one overshooting on one side and the next undershooting on the other.

That oscillation gives the second main result: the error estimate $|S - S_n| \le b_{n+1}$. Truncating an alternating series costs no more than the first term you dropped. This is the cleanest error bound in the whole chapter.

The section ends with Example #4, a rearrangement of the alternating harmonic series using the pattern "one positive, two negatives" that sums to $\tfrac{1}{2}\ln(2)$ instead of $\ln(2)$, even though it uses exactly the same terms. The reason is that $\sum (-1)^{n+1}\frac{1}{n}$ converges while $\sum \frac{1}{n}$ diverges, so the convergence depends on the order of the terms. That distinction becomes conditional vs. absolute convergence in the next section.

## Definitions and theorems

**Alternating series.** A series of the form
$$\sum (-1)^n b_n \qquad \text{or} \qquad \sum (-1)^{n+1} b_n$$
where $b_n \ge 0$. The factor $(-1)^n$ or $(-1)^{n+1}$ makes the terms switch sign; $b_n$ is the size of the $n$th term.

**Theorem #1 (Alternating Series Test).** Suppose $(b_n)$ satisfies
$$\lim_{n \to \infty} b_n = 0 \qquad \text{and} \qquad b_{n+1} \le b_n \text{ for all } n,$$
then
$$\sum (-1)^n b_n$$
converges.

So two conditions: the sizes go to $0$, and the sizes are non-increasing.

**Theorem #2 (Error Estimate for the AST).** Assume
$$S = \sum_{k=1}^{\infty} (-1)^{k+1} b_k$$
converges by the Alternating Series Test. Then
$$|S - S_n| \le b_{n+1}$$
where $S_n$ is the $n$th partial sum of $S$.

**Remark.** We will see later that
$$\sum_{k=1}^{\infty} \frac{(-1)^{k+1}}{k} = \ln(2).$$

**Tool used in the proof: the Monotone Convergence Theorem (MCT).** A sequence that is increasing and bounded above converges.

## Methods

**Applying the AST.** Write the series as $\sum (-1)^n b_n$ and read off $b_n$ as the absolute value of the term, with the sign factor stripped out. Then check:

1. $\lim_{n\to\infty} b_n = 0$.
2. $b_{n+1} \le b_n$, at least eventually.

If both hold, the series converges.

**Checking that $b_n$ decreases.** When $b_n$ comes from a formula, set $f(x)$ equal to that formula with $x$ in place of $n$ and show $f'(x) < 0$. The derivative only has to be negative past some point, since finitely many terms at the front do not affect convergence. This is what "eventually" means in Example #2: $f'(x) < 0$ for $x > e^{1/2}$ is enough.

**Checking that $b_n \to 0$.** Compute the limit directly, or use L'Hospital's rule when the expression is of type $\frac{\infty}{\infty}$ or $\frac{0}{0}$.

**Using the error estimate to pick $n$.** To guarantee accuracy within a tolerance $\varepsilon$, solve
$$b_{n+1} \le \varepsilon$$
for $n$. The smallest integer $n$ satisfying this gives a partial sum $S_n$ with $|S - S_n| \le \varepsilon$.

**The proof technique for the AST (as done for the alternating harmonic series).**

1. Look at the even partial sums $S_{2n}$ and group the terms in consecutive pairs. Each pair is non-negative because $b_n$ decreases, so $S_{2n}$ is increasing.
2. Regroup the same sum the other way to bound it above. For the alternating harmonic series the pairs $\left(\frac{1}{2k-1} - \frac{1}{2k}\right)$ sit in disjoint subintervals of $(0,1)$, so their total is at most $1$.
3. Increasing plus bounded above gives convergence of $S_{2n}$ by the MCT.
4. Relate the odd partial sums: $S_{2n+1} = S_{2n} + b_{2n+1}$. Since $b_{2n+1} \to 0$, the odd sums have the same limit.
5. Both subsequences converge to the same value, so $S_n$ converges.

## Worked examples

### Example #1 (Alternating Harmonic Series)

Show that the sum
$$\sum_{k=1}^{\infty} \frac{(-1)^{k+1}}{k}$$
exists.

Write $S = \sum_{k=1}^{\infty} \frac{(-1)^{k+1}}{k}$ and look at the even partial sums:
$$S_{2n} = \sum_{k=1}^{2n} \frac{(-1)^{k+1}}{k} = \left(1 - \frac{1}{2}\right) + \left(\frac{1}{3} - \frac{1}{4}\right) + \cdots + \left(\frac{1}{2n-1} - \frac{1}{2n}\right).$$

Each parenthesized block is positive, so going from $S_{2n}$ to $S_{2n+2}$ adds a positive amount and $(S_{2n})$ is increasing.

For the upper bound, read each block $\frac{1}{2k-1} - \frac{1}{2k}$ as the length of the interval $\left(\frac{1}{2k}, \frac{1}{2k-1}\right)$. These intervals are
$$\left(\tfrac{1}{2}, 1\right), \quad \left(\tfrac{1}{4}, \tfrac{1}{3}\right), \quad \left(\tfrac{1}{6}, \tfrac{1}{5}\right), \quad \ldots$$
which are disjoint subsets of $(0,1)$. The total of finitely many disjoint lengths inside $(0,1)$ is at most the length of $(0,1)$, so
$$S_{2n} \le 1.$$

Thus $(S_{2n})$ is bounded and increasing, so it converges by the MCT.

Now handle the odd partial sums. Adding the next term gives
$$S_{2n+1} = S_{2n} + \frac{1}{2n+1},$$
so
$$\lim_{n\to\infty} S_{2n+1} = \lim_{n\to\infty}\left(S_{2n} + \frac{1}{2n+1}\right) = \lim_{n\to\infty} S_{2n} + 0 = \lim_{n\to\infty} S_{2n}$$
since
$$\lim_{n\to\infty} \frac{1}{2n+1} = 0.$$

The even-indexed and odd-indexed partial sums converge to the same value, so the full sequence of partial sums converges. The Alternating Harmonic Series converges. The same argument generalizes to any $(b_n)$ decreasing to $0$, which is Theorem #1.

Later work shows the value is $\ln(2)$.

### Example #2

Determine if
$$\sum_{k=2}^{\infty} (-1)^k \frac{\ln(k)}{k^2}$$
converges.

Here $b_k = \frac{\ln(k)}{k^2}$. Set
$$f(x) = \frac{\ln(x)}{x^2}.$$

Differentiate with the quotient rule:
$$f'(x) = \frac{\left(\frac{1}{x}\right)x^2 - \ln(x)\cdot 2x}{x^4} = \frac{x - 2x\ln(x)}{x^4} = \frac{1 - 2\ln(x)}{x^3}.$$

For $x > 0$ the denominator $x^3$ is positive, so the sign of $f'(x)$ matches $1 - 2\ln(x)$. Then
$$1 - 2\ln(x) < 0 \iff \ln(x) > \tfrac{1}{2} \iff x > e^{1/2}.$$

So $f'(x) < 0$ for $x > e^{1/2}$, meaning $f$ is eventually decreasing and the terms $b_k$ eventually decrease.

For the limit, $\frac{\ln(x)}{x^2}$ is of type $\frac{\infty}{\infty}$, so by L'Hospital,
$$\lim_{x\to\infty} \frac{\ln(x)}{x^2} = \lim_{x\to\infty} \frac{1/x}{2x} = \lim_{x\to\infty}\frac{1}{2x^2} = 0,$$
so
$$\frac{\ln(x)}{x^2} \to 0.$$

Both conditions hold, so
$$\sum_{k=2}^{\infty} (-1)^k \frac{\ln(k)}{k^2}$$
converges by the Alternating Series test.

### Proof of Theorem #2 (Error Estimate)

Picture the partial sums: $S_1$ sits above, $S_2$ drops below, $S_3$ comes back up but below $S_1$, $S_4$ rises but above $S_2$. The sums zigzag inward toward $S$, with the odd sums above and the even sums below.

With $(S_{2n})$ increasing and converging to $S$,
$$S_{2n} \le S.$$

By a similar argument $(S_{2n+1})$ is decreasing, since $(b_n)$ is decreasing to $0$:
$$0 \le b_{2n+3} \le b_{2n+2} \implies 0 \le b_{2n+2} - b_{2n+3}.$$

Adding the next two terms to $S_{2n+1}$ (a negative one, then a positive one):
$$S_{2n+3} = S_{2n+1} - b_{2n+2} + b_{2n+3} = S_{2n+1} - \underbrace{(b_{2n+2} - b_{2n+3})}_{\text{non negative}},$$
so
$$S_{2n+3} \le S_{2n+1}.$$

A decreasing sequence converging to $S$ stays above $S$, giving the sandwich
$$S_{2n} \le S \le S_{2n+1}.$$

Subtract $S_{2n}$ from all three parts:
$$0 \le S - S_{2n} \le S_{2n+1} - S_{2n} = b_{2n+1}.$$

The last equality holds because $S_{2n+1} = S_{2n} + b_{2n+1}$. This is the bound at even indices.

Do a similar thing with $S_{2n+2}$ and $S_{2n+1}$:
$$S_{2n+2} \le S \le S_{2n+1},$$
but subtract $S_{2n+1}$ this time:
$$S_{2n+2} - S_{2n+1} \le S - S_{2n+1} \le 0.$$

Since $S - S_{2n+1}$ sits between a negative number and $0$, its absolute value is at most that of the left endpoint:
$$|S - S_{2n+1}| \le |S_{2n+2} - S_{2n+1}| = b_{2n+2}.$$

This is the bound at odd indices. Together, both cases give $|S - S_n| \le b_{n+1}$ for every $n$.

### Example #3

Choose $n \in \mathbf{N}$ such that
$$|S - S_n| \le 10^{-4}$$
where
$$S = \sum_{k=1}^{\infty} \frac{(-1)^{k+1}}{k^2}.$$

Here $b_k = \frac{1}{k^2}$, so $b_{n+1} = \frac{1}{(n+1)^2}$. By Theorem #2 it is enough to make $b_{n+1} \le 10^{-4}$. Solve
$$b_{n+1} = \frac{1}{(n+1)^2} \le 10^{-4}.$$

Take reciprocals, which flips the inequality:
$$10^{4} \le (n+1)^2.$$

Take square roots, legal since both sides are positive and $n+1 > 0$:
$$100 \le n+1,$$
so $n \ge 99$.

Then
$$S_{99} \approx 0.822518$$
and more advanced methods show
$$S = \frac{\pi^2}{12} \approx 0.822467.$$

Thus
$$|S_{99} - S| \approx 5.05 \cdot 10^{-5} < 10^{-4},$$
so the bound holds, with room to spare. The estimate $b_{n+1}$ is an upper bound on the error, not the exact error.

### Example #4

Compare
$$\ln(2) = 1 - \frac{1}{2} + \frac{1}{3} - \frac{1}{4} + \frac{1}{5} + \cdots$$
to
$$B = 1 - \frac{1}{2} - \frac{1}{4} + \frac{1}{3} - \frac{1}{6} - \frac{1}{8} + \frac{1}{5} - \frac{1}{10} - \cdots$$

First, what are the next terms in $B$?
$$B = 1 - \frac{1}{2} - \frac{1}{4} + \frac{1}{3} - \frac{1}{6} - \frac{1}{8} + \frac{1}{5} - \frac{1}{10} - \frac{1}{12} + \frac{1}{7} - \frac{1}{14} - \cdots$$

The pattern: in $B$ we add an odd denominator term, then subtract two even denominator terms, so the signs run
$$\text{"}+, -, -\text{"}$$

The blocks are $\left(\frac{1}{2m-1}, -\frac{1}{2(2m-1)}, -\frac{1}{4m-2+2}\right)$ in the sense that the odd term $\frac{1}{2m-1}$ is followed by $-\frac{1}{2(2m-1)}$ and $-\frac{1}{4m}$: for $m=1$ that is $1, -\frac12, -\frac14$; for $m=2$ it is $\frac13, -\frac16, -\frac18$; for $m=3$ it is $\frac15, -\frac{1}{10}, -\frac{1}{12}$; for $m=4$ it is $\frac17, -\frac{1}{14}, \ldots$ Every term of the alternating harmonic series appears exactly once.

Now evaluate $B$ by grouping each odd term with the first even term that follows it:
$$B = \left(1 - \frac{1}{2}\right) - \frac{1}{4} + \left(\frac{1}{3} - \frac{1}{6}\right) - \frac{1}{8} + \left(\frac{1}{5} - \frac{1}{10}\right) - \cdots$$

Each parenthesis simplifies by halving: $1 - \frac12 = \frac12$, and $\frac13 - \frac16 = \frac{2-1}{6} = \frac16$, and $\frac15 - \frac{1}{10} = \frac{2-1}{10} = \frac{1}{10}$. So
$$B = \frac{1}{2} - \frac{1}{4} + \frac{1}{6} - \frac{1}{8} + \frac{1}{10} - \frac{1}{12} + \cdots$$

Factor out $\frac12$ from every term:
$$B = \frac{1}{2}\left(1 - \frac{1}{2} + \frac{1}{3} - \frac{1}{4} + \frac{1}{5} - \frac{1}{6} + \cdots\right) = \frac{1}{2}\ln(2).$$

Yet both sums contain the exact same terms. Here is what drives the spooky behavior:

1. $\sum (-1)^{n+1}\frac{1}{n}$ converges.
2. $\sum \frac{1}{n}$ diverges.

The series converges only because of the cancellation between positive and negative terms, and the sizes alone sum to infinity. Rearranging shifts how much positive mass lands before how much negative mass, so the total changes. This is why the order of terms matters for an alternating series like this one.

## Common mistakes and tips

**Both AST conditions are required.** Checking only $b_n \to 0$ is not enough. The sizes must also be non-increasing. If $b_n \to 0$ but bounces up and down, the AST does not apply and you need another approach.

**"Eventually decreasing" is fine.** Example #2 only gets $f'(x) < 0$ for $x > e^{1/2}$. That is enough: finitely many terms at the start do not change whether a series converges. Say "eventually" explicitly when you write the argument.

**Strip the sign before identifying $b_n$.** In $\sum (-1)^k \frac{\ln(k)}{k^2}$ the sequence is $b_k = \frac{\ln(k)}{k^2}$, not $(-1)^k \frac{\ln(k)}{k^2}$. The AST is stated for $b_n \ge 0$.

**Use a derivative, not a feeling, to check decrease.** Set $f(x)$ from the formula for $b_n$ and compute $f'(x)$. Quotients like $\frac{\ln(x)}{x^2}$ increase first and then decrease, so eyeballing the first few terms misleads.

**The error estimate uses $b_{n+1}$, not $b_n$.** $|S - S_n| \le b_{n+1}$ is the size of the first term you left out. Setting $b_n \le \varepsilon$ instead of $b_{n+1} \le \varepsilon$ gives an off-by-one answer.

**$b_{n+1}$ is a bound, not the actual error.** In Example #3 the bound is $10^{-4}$ and the real error at $n = 99$ is about $5.05 \cdot 10^{-5}$, roughly half the bound. The bound is correct but conservative.

**Watch the inequality direction when taking reciprocals.** From $\frac{1}{(n+1)^2} \le 10^{-4}$ you get $10^{4} \le (n+1)^2$, with the inequality flipped.

**Convergence of an alternating series does not mean the sizes sum.** $\sum \frac{(-1)^{k+1}}{k}$ converges but $\sum \frac{1}{k}$ diverges. The convergence comes from cancellation.

**Order can change the sum.** Example #4 shows the same terms rearranged give $\frac12 \ln(2)$ instead of $\ln(2)$. Do not freely reshuffle terms in a series that converges only through cancellation.

**Pairing terms is a legitimate proof move inside a partial sum.** Grouping $\left(1 - \frac12\right) + \left(\frac13 - \frac14\right) + \cdots$ is safe because it is a finite sum. The danger in Example #4 is reordering across the whole infinite series, not regrouping a finite piece.

## Formula sheet

**Alternating series form**
$$\sum (-1)^n b_n, \qquad b_n \ge 0$$

**Theorem #1, Alternating Series Test.** If
$$\lim_{n \to \infty} b_n = 0 \quad \text{and} \quad b_{n+1} \le b_n \text{ for all } n,$$
then $\sum (-1)^n b_n$ converges.

**Theorem #2, Error Estimate for the AST.** If $S = \sum_{k=1}^{\infty} (-1)^{k+1} b_k$ converges by the AST, then
$$|S - S_n| \le b_{n+1}$$

**Choosing $n$ for a tolerance $\varepsilon$:** solve $b_{n+1} \le \varepsilon$.

**Partial sum recursion**
$$S_{n+1} = S_n + (-1)^{n+2} b_{n+1}$$
For the alternating harmonic series specifically, $S_{2n+1} = S_{2n} + \frac{1}{2n+1}$.

**Sandwich on the partial sums**
$$S_{2n} \le S \le S_{2n+1}$$

**Monotone Convergence Theorem.** An increasing sequence that is bounded above converges.

**Key sums from this section**
$$\sum_{k=1}^{\infty} \frac{(-1)^{k+1}}{k} = \ln(2)$$
$$\sum_{k=1}^{\infty} \frac{(-1)^{k+1}}{k^2} = \frac{\pi^2}{12} \approx 0.822467$$

**Alternating harmonic facts**
$$\sum (-1)^{n+1}\frac{1}{n} \text{ converges}, \qquad \sum \frac{1}{n} \text{ diverges}$$

**Derivative from Example #2**
$$f(x) = \frac{\ln(x)}{x^2} \implies f'(x) = \frac{1 - 2\ln(x)}{x^3} < 0 \text{ for } x > e^{1/2}$$
$$\lim_{x\to\infty}\frac{\ln(x)}{x^2} = 0 \text{ by L'Hospital}$$
