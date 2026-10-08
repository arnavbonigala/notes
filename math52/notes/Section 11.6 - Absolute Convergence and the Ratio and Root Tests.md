# Section 11.6 - Absolute Convergence and the Ratio and Root Tests

## Summary

This section splits convergent series into two kinds and then gives two new tests that work by comparison with a geometric series.

The split is between absolute and conditional convergence. A series $\sum a_n$ converges absolutely when the series of absolute values $\sum |a_n|$ converges, and conditionally when $\sum a_n$ converges but $\sum |a_n| = \infty$. Theorem #1 says absolute convergence is the stronger condition: it implies ordinary convergence. The two series in Example #1 show both cases, $\sum (-1)^{k+1} k^{-1/2}$ conditionally and $\sum (-1)^{k+1}(k^2 + \sqrt{k})^{-1}$ absolutely.

The distinction matters because of Riemann's Rearrangement Theorem. If a series converges conditionally, you can reorder its terms to converge to any real number you like. Example #2 recalls the §11.5 rearrangement of the alternating harmonic series: the same terms in a different order sum to $\tfrac{1}{2}\ln(2)$ instead of $\ln(2)$. Absolutely convergent series are immune to this, and reordering them never changes the sum.

The two new tests are the Root Test, built on $L = \lim |a_n|^{1/n}$, and the Ratio Test, built on $L = \lim |a_{n+1}|/|a_n|$. Both have the same verdict structure: $L < 1$ gives absolute convergence, $L > 1$ gives divergence, $L = 1$ gives nothing. Both work the same way underneath. If $|a_n|^{1/n} \approx L$ then $|a_n| \approx L^n$, so the tail of the series looks like a geometric series with ratio $L$, and the geometric series $\sum x^k = \frac{1}{1-x}$ converges exactly when $|x| < 1$. The proofs make this precise by picking $\varepsilon$ to be half the distance from $L$ to $1$ and running the Basic Comparison Test against $\sum (L+\varepsilon)^n$.

Alongside the tests the section collects the limits you need to run them: $n^{1/n} \to 1$, $|a|^{1/n} \to 1$, and $(1 + x/n)^n \to e^x$, plus the definition and recursion for factorials, $n! = n(n-1)!$ with $0! = 1$. The closing guidance is that the Root Test suits powers of $n$, the Ratio Test suits factorials and recursive terms, and neither test sees pure powers of $n$ like $\sum 1/n$ or $\sum 1/n^2$, where both return $L = 1$.

## Definitions and theorems

**Definition #1 (Absolute Convergence).** Let $(a_n)$ be a sequence. If
$$\sum |a_n|$$
converges then the series $\sum a_n$ **converges absolutely**.

**Definition #2 (Conditional Convergence).** Let $(a_n)$ be a sequence. If
$$\sum |a_n| = \infty$$
diverges but
$$\sum a_n$$
exists then the series $\sum a_n$ **converges conditionally**.

**Theorem #1.** Suppose $\sum a_n$ converges absolutely. Then $\sum a_n$ converges.

*Proof.* Notice that
$$0 \le |a_n| - a_n \le 2|a_n|,$$
since $a_n \le |a_n|$ gives the left inequality and $-a_n \le |a_n|$ gives the right. So if $\sum 2|a_n|$ converges, which it does when $\sum |a_n|$ converges, then $\sum (|a_n| - a_n)$ converges by the Basic Comparison Test. Then
$$\sum a_n = \sum |a_n| - \sum (|a_n| - a_n)$$
tells us $\sum a_n$ must converge because it is a difference of two convergent series. $\blacksquare$

**Theorem #2 (Riemann's Rearrangement Theorem).** Let $x \in \mathbf{R}$. Suppose
$$S = \sum_{k=1}^{\infty} a_k$$
converges conditionally. Then we can rearrange the terms in the series so that the sum of the reordered terms converges to $x$.

**Warning.** The reordering of the terms in an absolutely convergent series does not change the value of the sum.

**Definition (Geometric Series).** Let $x \in \mathbf{R}$. The **geometric series**
$$\sum_{k=0}^{\infty} x^k = \frac{1}{1-x}$$
exists only for $|x| < 1$.

**Theorem #3 (Root Test).** Let $(a_n)$ be a sequence and define
$$L = \lim_{n \to \infty} |a_n|^{1/n}.$$
Then

1. $L < 1$ implies $\sum a_n$ converges absolutely
2. $L > 1$ implies $\sum a_n$ diverges
3. $L = 1$ is inconclusive so use another test.

*Proof sketch.* For $L \in (0,1)$ take $\varepsilon = \frac{1-L}{2}$, half the distance from $L$ to $1$. Then
$$L + \varepsilon = L + \frac{1}{2} - \frac{L}{2} = \frac{L}{2} + \frac{1}{2} < 1$$
since $L < 1$. With $|a_n|^{1/n} \to L$, choose $n$ large where
$$\left||a_n|^{1/n} - L\right| < \varepsilon \implies |a_n|^{1/n} - L < \varepsilon \implies |a_n|^{1/n} < L + \varepsilon < 1 \implies |a_n| < (L+\varepsilon)^n.$$
By the Basic Comparison Test $\sum a_n$ converges absolutely, since $\sum (L+\varepsilon)^n$ is a convergent geometric series because $L + \varepsilon < 1$.

For $L > 1$ choose $\varepsilon = \frac{L-1}{2}$, so $1 < L - \varepsilon$. Then for $n$ large,
$$\left||a_n|^{1/n} - L\right| < \varepsilon \implies -\varepsilon < |a_n|^{1/n} - L < \varepsilon \implies 1 < L - \varepsilon < |a_n|^{1/n} \implies 1 < (L-\varepsilon)^n < |a_n|.$$
Clearly $(a_n)$ is not near $0$ for $n$ large and so the series diverges by the Basic Divergence Test. $\blacksquare$

**Theorem #4 (Useful Limits).**

1. $n^{1/n} \to 1$
2. $|a|^{1/n} \to 1$ for $a \ne 0$
3. $\left(1 + \dfrac{x}{n}\right)^n \to e^x$

*Why part 3 holds.* The number $e$ is such that
$$1 = \lim_{h \to 0}\frac{e^h - 1}{h},$$
but we can also define it by
$$\lim_{h \to 0}(1+h)^{1/h} = e.$$
With $n = \frac{1}{h}$,
$$e = \lim_{n \to \infty}\left(1 + \frac{1}{n}\right)^n = \lim_{h \to 0^+}(1+h)^{1/h}.$$
For $n = xm$ we have, as $n \to \infty$,
$$m \to \begin{cases} \infty, & x > 0 \\ -\infty, & x < 0. \end{cases}$$
If $x > 0$,
$$\lim_{n \to \infty}\left(1 + \frac{x}{n}\right)^n = \lim_{m \to \infty}\left(1 + \frac{1}{m}\right)^{xm} = \left(\lim_{m \to \infty}\left(1 + \frac{1}{m}\right)^m\right)^x = e^x.$$
If $x < 0$,
$$\lim_{n \to \infty}\left(1 + \frac{x}{n}\right)^n = \left(\lim_{m \to -\infty}\left(1 + \frac{1}{m}\right)^m\right)^x = e^x$$
since $e$ can be expressed by
$$e = \lim_{m \to -\infty}\left(1 + \frac{1}{m}\right)^m = \lim_{h \to 0^-}(1+h)^{1/h}.$$
We have shown $\left(1 + \frac{x}{n}\right)^n \to e^x$ for all $x \in \mathbf{R}$.

**Definition #3 (Factorial).** Let $n \in \mathbf{N}$. Then
$$n! = n \times (n-1) \times (n-2) \times \cdots \times 3 \times 2 \times 1 = n(n-1)!$$
defines the $n$th **factorial**. The first few values are
$$1! = 1, \quad 2! = 2 \cdot 1 = 2, \quad 3! = 3 \cdot 2 \cdot 1 = 6, \quad 4! = 4 \cdot 6 = 24, \quad 5! = 5 \cdot 24 = 120,$$
and we define $0! = 1$.

**Theorem #5 (Ratio Test).** Let $(a_n)$ be a sequence of non-zero numbers and define
$$L = \lim_{n \to \infty} \frac{|a_{n+1}|}{|a_n|}.$$
Then

1. $L < 1$ implies $\sum a_n$ converges absolutely
2. $L > 1$ implies $\sum a_n$ diverges
3. $L = 1$ is inconclusive so use another test.

*Proof sketch.* Modify the Root Test argument. If $L < 1$, choose $n$ large such that
$$\left|\left|\frac{a_{n+1}}{a_n}\right| - L\right| < \varepsilon = \frac{1-L}{2},$$
so
$$|a_{n+1}| < (L+\varepsilon)|a_n| \implies |a_{n+2}| < (L+\varepsilon)^2|a_n|$$
and therefore
$$\sum_{k=n}^{\infty}|a_k| \le |a_n|\sum_{k=0}^{\infty}(L+\varepsilon)^k.$$
With $|L + \varepsilon| < 1$, we have convergence of $\sum |a_k|$ by the Basic Comparison Test. $\blacksquare$

**Warning.** Sums of powers of $n$ will often not work with the Root or Ratio Test.

**Remark.**

- Root Test: good for powers of $n$
- Ratio Test: good for $n!$ or recursive terms
- The Root Test is stronger than the Ratio Test, it works for more series. (Take Math 104)

## Methods

**Deciding absolute vs. conditional.** Given a series with mixed signs:

1. Strip the signs and test $\sum |a_n|$ with the tools from §11.2 to §11.4, usually a $p$-series, a comparison, or the integral test.
2. If $\sum |a_n|$ converges, you are done: the series converges absolutely, and Theorem #1 gives ordinary convergence for free.
3. If $\sum |a_n|$ diverges, test $\sum a_n$ itself, usually with the Alternating Series Test. If that converges, the series converges conditionally.

**The Root Test procedure.** Form $|a_n|^{1/n}$, simplify it algebraically before taking the limit, and compare the limit $L$ to $1$. The simplification is where the work is: $\left(\frac{p(n)}{q(n)}\right)^n$ has $n$th root $\left|\frac{p(n)}{q(n)}\right|$, and $\frac{c^n}{n^k}$ has $n$th root $\frac{c}{(n^{1/n})^k}$.

**The Ratio Test procedure.** Write out $a_{n+1}$ by substituting $n+1$ everywhere $n$ appears, simplify factorials with $(n+1)! = (n+1)n!$, then form the quotient $\frac{a_{n+1}}{a_n}$ and cancel. Dividing by $a_n$ means multiplying by its reciprocal, so factorials cancel cleanly rather than needing to be expanded.

**Why the two tests work.** Both compare the tail of the series to a geometric series.

For the Root Test: $|a_n|^{1/n} \to L$ means $|a_n| \approx L^n$, so $\sum |a_n| \approx \sum L^n$. The series "looks like" a geometric sum for $n$ large.

For the Ratio Test: $\left|\frac{a_{n+1}}{a_n}\right| \approx L$ means $|a_{n+1}| \approx L|a_n|$, so $|a_{n+2}| \approx L|a_{n+1}| \approx L^2|a_n|$, and
$$\sum_{k=n}^{\infty} |a_k| \approx |a_n|\left(1 + L + L^2 + \cdots\right) = |a_n| \sum_{k=0}^{\infty} L^k.$$
If $L < 1$ we have convergence and if $L > 1$ we have divergence.

**The $\varepsilon$ trick in the proofs.** To turn "$\approx$" into an inequality, take $\varepsilon$ to be half the distance from $L$ to $1$, so the number $L + \varepsilon$ (or $L - \varepsilon$) sits strictly between them and the comparison geometric series has a ratio strictly on the right side of $1$. This is the standard move whenever you need a limit statement to give you a strict inequality for all large $n$.

**Evaluating $n^{1/n}$.** Write it as an exponential and use continuity:
$$n^{1/n} = e^{\log(n^{1/n})} = e^{\frac{1}{n}\log(n)},$$
then bound the exponent. For $n > 4$,
$$0 < \frac{1}{n}\log(n) \le \frac{\sqrt{n}}{n} = \frac{1}{\sqrt{n}},$$
so $\frac{1}{n}\log(n) \to 0$, and since $e^x$ is continuous on $\mathbf{R}$,
$$e^{\frac{1}{n}\log(n)} \to e^0 = 1.$$

**Where the tests fail.** For
$$\sum \frac{1}{n} \qquad \text{and} \qquad \sum \frac{1}{n^2}$$
the Root Test fails:
$$\left(\frac{1}{n}\right)^{1/n} \to 1 \qquad \text{and} \qquad \left(\frac{1}{n^2}\right)^{1/n} \to 1,$$
and the Ratio Test fails too:
$$\frac{\frac{1}{n+1}}{\frac{1}{n}} = \frac{n}{n+1} \to 1 \qquad \text{and} \qquad \frac{\frac{1}{(n+1)^2}}{\frac{1}{n^2}} = \frac{n^2}{(n+1)^2} \to 1.$$
Both series give $L = 1$ even though one diverges and one converges, which is exactly why $L = 1$ carries no information.

## Worked examples

### Example #1: conditional vs. absolute

The series
$$\sum_{k=1}^{\infty} \frac{(-1)^{k+1}}{\sqrt{k}}$$
converges conditionally while
$$\sum_{k=1}^{\infty} \frac{(-1)^{k+1}}{k^2 + \sqrt{k}}$$
converges absolutely.

**First series.** We know
$$b_k = \frac{1}{\sqrt{k}}$$
is decreasing to $0$ so
$$\sum_{k=1}^{\infty} (-1)^{k+1}\frac{1}{k^{1/2}}$$
converges by the Alternating Series Test. Now
$$\sum_{k=1}^{\infty} |b_k| = \sum_{k=1}^{\infty} \frac{1}{k^{1/2}} = \infty$$
is a divergent $p$-series since $p = 1/2 \le 1$. So
$$\sum_{k=1}^{\infty} (-1)^{k+1}\frac{1}{k^{1/2}}$$
converges conditionally.

**Second series.** Since
$$0 < k^2 < k^2 + k^{1/2},$$
taking reciprocals reverses the inequality:
$$0 < \frac{1}{k^2 + k^{1/2}} < \frac{1}{k^2}.$$
We know
$$\sum_{k=1}^{\infty} |a_k| = \sum_{k=1}^{\infty} \frac{1}{k^2 + k^{1/2}}$$
converges by the Basic Comparison Test because
$$\sum_{k=1}^{\infty} \frac{1}{k^2}$$
is a convergent $p$-series since $p = 2 > 1$. Hence
$$\sum_{k=1}^{\infty} (-1)^{k+1}\frac{1}{k^2 + k^{1/2}}$$
converges absolutely.

### Example #2: a rearrangement that changes the sum

We showed in §11.5:
$$\ln(2) = 1 - \frac{1}{2} + \frac{1}{3} - \frac{1}{4} + \frac{1}{5} - \frac{1}{6} + \cdots$$
$$\frac{1}{2}\ln(2) = \left(1 - \frac{1}{2}\right) - \frac{1}{4} + \left(\frac{1}{3} - \frac{1}{6}\right) - \frac{1}{8} + \left(\frac{1}{5} - \frac{1}{10}\right) - \frac{1}{12} + \cdots$$

The second line uses the same terms as the first, reordered. The alternating harmonic series converges conditionally, so Theorem #2 applies and the rearranged sum is free to land somewhere else, here at exactly half the original value.

### Example #3: Root Test with a power of $n$

Determine the convergence of
$$\sum \left(\frac{4n^2 - 3}{7n^2 + 6}\right)^n.$$

Let
$$a_n = \left(\frac{4n^2 - 3}{7n^2 + 6}\right)^n,$$
then the outer exponent $n$ cancels against the $n$th root:
$$|a_n|^{1/n} = \left|\frac{4n^2 - 3}{7n^2 + 6}\right| \to \frac{4}{7} < 1,$$
dividing numerator and denominator by $n^2$ to see the limit. Hence
$$\sum \left(\frac{4n^2 - 3}{7n^2 + 6}\right)^n$$
converges absolutely by the Root Test.

### Example #4: Root Test with $n^{1/n}$

Determine the convergence of
$$\sum \frac{2^n}{n^{10}}.$$

We have
$$a_n = \frac{2^n}{n^{10}}$$
giving
$$|a_n|^{1/n} = \frac{2}{n^{10/n}} = \frac{2}{(n^{1/n})^{10}},$$
using $(2^n)^{1/n} = 2$ and $(n^{10})^{1/n} = n^{10/n} = (n^{1/n})^{10}$.

What does $n^{1/n}$ do as $n \to \infty$?
$$n^{1/n} = e^{\log(n^{1/n})} = e^{\frac{1}{n}\log(n)}$$
and for $n > 4$,
$$0 < \frac{1}{n}\log(n) \le \frac{\sqrt{n}}{n} = \frac{1}{\sqrt{n}}.$$
Thus
$$\frac{1}{n}\log(n) \to 0 \implies e^{\frac{1}{n}\log(n)} \to e^0 = 1$$
since $e^x$ is continuous on $\mathbf{R}$, so $n^{1/n} \to 1$.

This shows
$$|a_n|^{1/n} = \frac{2}{(n^{1/n})^{10}} \to \frac{2}{1^{10}} = 2 = L > 1$$
and so
$$\sum \frac{2^n}{n^{10}}$$
diverges by the Root Test.

### Example #5: Ratio Test with a factorial

Determine the convergence of
$$\sum \frac{n^2}{n!}.$$

Here
$$a_n = \frac{n^2}{n!}$$
and $(n+1)! = (n+1)n!$ gives
$$a_{n+1} = \frac{(n+1)^2}{(n+1)!} = \frac{(n+1)^2}{(n+1)n!} = \frac{n+1}{n!},$$
cancelling one factor of $n+1$.

Next,
$$\frac{a_{n+1}}{a_n} = \frac{\frac{n+1}{n!}}{\frac{n^2}{n!}} = \frac{n+1}{n!} \cdot \frac{n!}{n^2} = \frac{n+1}{n^2} \to 0,$$
since the denominator grows one power faster than the numerator. With $L = 0 < 1$,
$$\sum \frac{n^2}{n!}$$
converges by the Ratio Test.

### Example #6: Ratio Test giving $e$

Determine the convergence of
$$\sum \frac{n^n}{n!}.$$

Here
$$a_n = \frac{n^n}{n!}$$
so
$$a_{n+1} = \frac{(n+1)^{n+1}}{(n+1)!} = \frac{(n+1)(n+1)^n}{(n+1)n!} = \frac{(n+1)^n}{n!},$$
splitting $(n+1)^{n+1} = (n+1)(n+1)^n$ and $(n+1)! = (n+1)n!$ so the factor $n+1$ cancels.

Notice
$$\frac{a_{n+1}}{a_n} = \frac{\frac{(n+1)^n}{n!}}{\frac{n^n}{n!}} = \frac{(n+1)^n}{n^n} = \left(\frac{n+1}{n}\right)^n = \left(1 + \frac{1}{n}\right)^n \to e$$
by Theorem #4 part 3 with $x = 1$, and $e > 1$. Thus
$$\sum \frac{n^n}{n!}$$
diverges by the Ratio Test.

## Common mistakes and tips

**$L = 1$ means you learned nothing.** It does not mean divergence and it does not mean convergence. Drop the test and go back to the comparison, integral, or alternating series tools. The $\sum 1/n$ against $\sum 1/n^2$ pair is the standard demonstration: identical $L = 1$, opposite behavior.

**Pick the test by the shape of the term.** An $n$th power anywhere in the term points to the Root Test, since the $n$th root cancels it. A factorial or a term defined from the previous one points to the Ratio Test, since the quotient collapses the factorial. A plain rational function of $n$ points to neither.

**$(n+1)! = (n+1)n!$ is the only factorial identity you need.** Never expand factorials out. Write $a_{n+1}$, pull the $(n+1)$ off the front of $(n+1)!$, and the $n!$ cancels against the one in $a_n$.

**Divide, do not subtract, when forming the ratio.** $\frac{a_{n+1}}{a_n}$ means multiplying $a_{n+1}$ by the reciprocal of $a_n$. Writing it as a compound fraction and flipping the bottom is the reliable way to avoid losing a factor.

**Absolute convergence is strictly stronger.** Theorem #1 runs one direction only. Absolute convergence implies convergence; convergence does not imply absolute convergence, and Example #1 gives the counterexample. So showing $\sum |a_n|$ diverges tells you nothing yet about $\sum a_n$.

**Do not trust rearrangement on conditionally convergent series.** Reordering terms is a safe algebraic step only when the series converges absolutely. For a conditionally convergent series any real number is reachable, so a rearranged sum carries no meaning until you have checked absolute convergence.

**The Ratio Test requires non-zero terms.** Theorem #5 is stated for a sequence of non-zero numbers, because $|a_n|$ sits in a denominator.

**Both tests detect divergence through the Basic Divergence Test.** When $L > 1$ the conclusion is not just that the comparison failed. The terms satisfy $|a_n| > 1$ for $n$ large, so $a_n \not\to 0$ and the series diverges outright.

**Series of powers of $n$ alone resist both tests.** The warning on the slides is worth taking literally. If the term is a ratio of polynomials in $n$, go to the $p$-series, limit comparison, or integral test instead of grinding through a root or ratio computation that will return $1$.

## Formula sheet

**Absolute convergence.** $\sum |a_n|$ converges $\implies$ $\sum a_n$ converges absolutely $\implies$ $\sum a_n$ converges.

**Conditional convergence.** $\sum |a_n| = \infty$ and $\sum a_n$ exists $\implies$ $\sum a_n$ converges conditionally.

**Root Test.** With
$$L = \lim_{n \to \infty} |a_n|^{1/n}:$$
$$L < 1 \implies \text{absolute convergence}, \qquad L > 1 \implies \text{divergence}, \qquad L = 1 \implies \text{inconclusive}.$$

**Ratio Test.** With $a_n \ne 0$ and
$$L = \lim_{n \to \infty} \frac{|a_{n+1}|}{|a_n|}:$$
$$L < 1 \implies \text{absolute convergence}, \qquad L > 1 \implies \text{divergence}, \qquad L = 1 \implies \text{inconclusive}.$$

**Geometric series.**
$$\sum_{k=0}^{\infty} x^k = \frac{1}{1-x}, \qquad |x| < 1.$$

**Useful limits.**
$$n^{1/n} \to 1, \qquad |a|^{1/n} \to 1 \ (a \ne 0), \qquad \left(1 + \frac{x}{n}\right)^n \to e^x.$$

**Definitions of $e$.**
$$1 = \lim_{h \to 0} \frac{e^h - 1}{h}, \qquad e = \lim_{h \to 0}(1+h)^{1/h} = \lim_{n \to \infty}\left(1 + \frac{1}{n}\right)^n.$$

**Factorials.**
$$n! = n(n-1)!, \qquad (n+1)! = (n+1)n!, \qquad 0! = 1,$$
$$1! = 1, \quad 2! = 2, \quad 3! = 6, \quad 4! = 24, \quad 5! = 120.$$

**Key inequality in Theorem #1.**
$$0 \le |a_n| - a_n \le 2|a_n|, \qquad \sum a_n = \sum |a_n| - \sum\left(|a_n| - a_n\right).$$

**$\varepsilon$ choices in the proofs.**
$$L < 1: \ \varepsilon = \frac{1-L}{2}, \quad L + \varepsilon = \frac{L}{2} + \frac{1}{2} < 1; \qquad L > 1: \ \varepsilon = \frac{L-1}{2}, \quad L - \varepsilon > 1.$$

**Test selection.** Powers of $n$ to the Root Test. Factorials or recursive terms to the Ratio Test. Ratios of polynomials to neither, both give $L = 1$.
