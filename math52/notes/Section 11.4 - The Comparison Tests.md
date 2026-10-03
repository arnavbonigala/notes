# Section 11.4 - The Comparison Tests

## Summary

This section gives two tests for deciding whether a series of positive terms converges, both built on the same idea: compare the series you care about to a series whose behavior you already know (a $p$-series or a geometric series).

The starting point is a dichotomy. For nonnegative terms there is no oscillation to worry about, so a series of nonnegative terms either converges to a finite number or diverges to $\infty$. Nothing else can happen. That is Lemma #1, and it is what makes comparison arguments work: once you trap the partial sums below a finite bound, convergence follows from the Monotone Convergence Theorem.

The **Basic Comparison Test (BCT)** compares term by term. If $0 < a_n \le b_n$, then the bigger series dragging to $\infty$ follows from the smaller one doing so, and the smaller series being finite follows from the bigger one being finite. The weakness is that you must produce an actual inequality, and sometimes the natural inequality points the wrong way (Example #3 shows this).

The **Limit Comparison Test (LCT)** fixes that. Instead of an inequality you take the limit $c = \lim_{n\to\infty} a_n/b_n$. If $c$ is positive and finite, the two series are essentially constant multiples of each other for large $n$, so they live or die together. The LCT is proved from the BCT by using the definition of limit with $\varepsilon = c/2$ to sandwich $a_k$ between $\tfrac{c}{2}b_k$ and $\tfrac{3c}{2}b_k$.

A practical remark appears here too: the hypotheses only need to hold for $n$ large. The same relaxation applies to the Integral Test, where $f$ only needs to be decreasing eventually. Finitely many terms never change convergence.

## Definitions and theorems

### Lemma #1

Let $(a_n)$ be a sequence of nonnegative numbers. Then one of the following holds:

1. the series $\sum a_n$ converges to a finite number, or
2. the series $\sum a_n$ diverges to infinity.

### Theorem #1 (Basic Comparison Test)

Let $(a_n)$ and $(b_n)$ be sequences such that

$$0 < a_n \le b_n$$

for all $n \in \mathbf{N}$. Then

$$\sum a_n = \infty \quad \text{implies} \quad \sum b_n = \infty$$

and

$$\sum b_n \in \mathbf{R} \quad \text{implies} \quad \sum a_n \in \mathbf{R}.$$

In words: if the smaller series blows up, so does the larger one. If the larger series is finite, so is the smaller one.

### Why the BCT is true

This runs the same argument used for the Integral Test. Set

$$S_n = \sum_{k=1}^{n} a_k.$$

Since $a_k > 0$, the sequence $(S_n)$ is increasing. If $(S_n)$ is bounded, the partial sums converge by the Monotone Convergence Theorem. Otherwise $S_n \to \infty$.

Now suppose $0 < a_k \le b_k$ and set

$$T_n = \sum_{k=1}^{n} b_k.$$

Summing the inequality $a_k \le b_k$ over $k = 1, \dots, n$ gives

$$0 < S_n \le T_n.$$

If $T_n \to T$, then since $(T_n)$ is increasing we have $T_n \le T$ for every $n$, hence $S_n \le T$. So $(S_n)$ is bounded and converges. Naturally, in the other direction,

$$S_n \to \infty \implies T_n \to \infty,$$

since $T_n \ge S_n$.

One caution: the MCT is an existence result. It tells you $S = \sum_{k=1}^{\infty} a_k$ exists; it does not tell you the value of $S$.

### Remark (relaxing the hypotheses)

The condition

$$0 < a_n \le b_n \text{ for all } n$$

in the Basic Comparison Test can be relaxed to hold for all $n$-large.

The condition

$$f \text{ decreasing on } [1, \infty)$$

in the Integral Test can be relaxed to $f$ decreasing for all $x$-large.

### Theorem #2 (Limit Comparison Test)

Let $(a_n)$ and $(b_n)$ be sequences of positive numbers where

$$c = \lim_{n\to\infty} \frac{a_n}{b_n}$$

exists. If $c > 0$ and finite, then

$$\sum a_n \quad \text{and} \quad \sum b_n$$

either both converge or both diverge.

### Why the LCT is true

This is a special application of the Basic Comparison Test. To use the test you need a good guess as to what series to compare it to.

**Idea.** With $c > 0$,

$$0 < \frac{a_k}{b_k} \approx c
\implies 0 < a_k \approx c \cdot b_k
\implies \sum a_k \approx \sum c \cdot b_k,$$

so the series are multiples of each other. They should both converge or diverge together.

**Formally.** With $\dfrac{a_k}{b_k} \to c > 0$, apply the definition of the limit with $\varepsilon = c/2$: choose $N \in \mathbf{N}$ such that

$$\left| \frac{a_k}{b_k} - c \right| < \frac{c}{2}$$

for any $k > N$. Unpacking the absolute value,

$$-\frac{c}{2} < \frac{a_k}{b_k} - c < \frac{c}{2},$$

and adding $c$ throughout,

$$\frac{c}{2} < \frac{a_k}{b_k} < \frac{3c}{2}.$$

Multiplying through by $b_k > 0$,

$$0 < \frac{c}{2} b_k < a_k < \frac{3c}{2} b_k.$$

Summing from $k = N+1$ onward,

$$\frac{c}{2} \sum_{k=N+1}^{\infty} b_k \le \sum_{k=N+1}^{\infty} a_k \le \frac{3c}{2} \sum_{k=N+1}^{\infty} b_k.$$

If $\sum a_k$ exists, then so must $\frac{c}{2}\sum b_k$ by the BCT, since $\frac{c}{2}b_k < a_k$. This also means $\sum b_k$ exists.

If $\sum b_k$ exists, then so must $\frac{3c}{2}\sum b_k$, and then $\sum a_k$ by the BCT, since $a_k < \frac{3c}{2}b_k$.

## Methods

**Choosing a comparison series.** Look at the dominant behavior of $a_k$ for $k$ large and strip away everything that does not matter. Lower-order terms in a polynomial, bounded factors, and additive constants all go. What survives is almost always a $p$-series $\sum 1/k^p$ or a geometric series $\sum r^k$, and you already know when those converge.

**When to use the BCT.** Use it when a clean inequality is available in the direction you need:

- To prove **divergence**, bound your series **below** by a known divergent series.
- To prove **convergence**, bound your series **above** by a known convergent series.

Standard moves for building the inequality:

- Dropping a positive term from the denominator makes the fraction larger.
- Adding a positive term to the denominator makes the fraction smaller.
- Replacing a numerator with something larger makes the fraction larger.
- Growth comparisons such as $0 < \ln(x) \le x^{1/2}$ for $x$ large (this follows from the MVT) let you trade a logarithm for a power.

**When to use the LCT.** Use it when the natural comparison points the wrong way, or when writing the inequality is more work than it is worth. Pick $b_k$ as the simplified version of $a_k$, compute

$$c = \lim_{k\to\infty} \frac{a_k}{b_k},$$

check $0 < c < \infty$, and then conclude that $\sum a_k$ and $\sum b_k$ share a fate.

**Computing the ratio limit.** Simplify the complex fraction first by multiplying by the reciprocal, then divide numerator and denominator by the dominant term. L'Hospital's Rule handles leftovers like $k/e^{2k} \to 0$. For exponentials, factoring the dominant exponential out of numerator and denominator is cleaner than L'Hospital on the whole quotient.

## Worked examples

### Example #1

Determine if

$$\sum_{k=2}^{\infty} \frac{1}{\sqrt{k} - 1}$$

converges.

For all $k \in \mathbf{N}$ with $k > 2$,

$$0 < \sqrt{k} - 1 < \sqrt{k}.$$

Taking reciprocals reverses the inequality between the two positive quantities:

$$0 < \frac{1}{\sqrt{k}} < \frac{1}{\sqrt{k} - 1}.$$

Choose

$$a_k = \frac{1}{\sqrt{k}} \quad \text{and} \quad b_k = \frac{1}{\sqrt{k}-1}.$$

The $p$-series with $p = 1/2$ diverges to $\infty$. So

$$\sum_{k=2}^{\infty} \frac{1}{\sqrt{k}} = \infty \implies \sum_{k=2}^{\infty} \frac{1}{\sqrt{k}-1} = \infty$$

by the Basic Comparison Test (BCT).

### Example #2

Determine if

$$\sum_{k=2}^{\infty} \frac{\ln(k)}{k^2}$$

converges.

From the MVT,

$$0 < \ln(x) \le x^{1/2} \quad \text{on } [4, \infty).$$

For $k > 3$, dividing that inequality by $k^2 > 0$,

$$0 < \frac{\ln(k)}{k^2} \le \frac{k^{1/2}}{k^2} = \frac{1}{k^{3/2}},$$

where the last step uses $k^{1/2}/k^2 = k^{1/2 - 2} = k^{-3/2}$.

Choose

$$a_k = \frac{\ln(k)}{k^2} \quad \text{and} \quad b_k = \frac{1}{k^{3/2}},$$

then

$$\sum_{k=1}^{\infty} \frac{1}{k^{3/2}}$$

is a convergent $p$-series since $p = 3/2 > 1$. Thus

$$\sum_{k=2}^{\infty} \frac{\ln(k)}{k^2}$$

converges by the BCT.

### Example #3

Determine if

$$\sum_{k=1}^{\infty} \frac{k}{k^2 + 2k + 3}$$

converges.

**First attempt (BCT, and why it fails).** Here

$$0 < k^2 < k^2 + 2k + 3$$

and so taking reciprocals and multiplying by $k > 0$,

$$\frac{k}{k^2 + 2k + 3} < \frac{k}{k^2} = \frac{1}{k}.$$

Bounding our sum above by the divergent Harmonic series does not help. An upper bound by a divergent series tells you nothing.

**Why we expect divergence.** Now

$$\frac{k}{k^2 + 2k + 3} \approx \frac{1}{k} \quad \text{for } k\text{-large}$$

suggests our sum diverges since it "looks like" the divergent Harmonic series.

We could argue

$$k^2 + 2k + 3 < 2k^2$$

to show

$$\frac{1}{2k} < \frac{k}{k^2 + 2k + 3}$$

for $k$-large, but this takes work. (Dividing $k$ by the smaller denominator $2k^2$ gives the smaller fraction $k/(2k^2) = 1/(2k)$, and $\sum 1/(2k)$ diverges, which would finish the job by the BCT.)

**The LCT version.** Take

$$a_k = \frac{k}{k^2 + 2k + 3} \approx \frac{1}{k} = b_k$$

and compute

$$\frac{\dfrac{k}{k^2+2k+3}}{\dfrac{1}{k}} = \frac{k}{k^2+2k+3} \cdot k = \frac{k^2}{k^2+2k+3}.$$

Dividing numerator and denominator by $k^2$,

$$\frac{k^2}{k^2+2k+3} = \frac{1}{1 + \dfrac{2}{k} + \dfrac{3}{k^2}} \to 1$$

as $k \to \infty$, since $2/k \to 0$ and $3/k^2 \to 0$.

This tells us $c = 1$. Hence

$$\sum_{k=1}^{\infty} \frac{k}{k^2+2k+3} = \infty$$

by the Limit Comparison Test (LCT) since $\sum \frac{1}{k} = \infty$.

### Example #4

Determine if

$$\sum_{k=1}^{\infty} \frac{e^{2k} + k}{e^{5k} - k^2}$$

converges.

Notice that factoring the dominant exponential out of the numerator and the denominator gives

$$a_k = \frac{e^{2k} + k}{e^{5k} - k^2}
= \frac{e^{2k}\left(1 + \dfrac{k}{e^{2k}}\right)}{e^{5k}\left(1 - \dfrac{k^2}{e^{5k}}\right)}
= \frac{1}{e^{3k}} \cdot \frac{1 + \dfrac{k}{e^{2k}}}{1 - \dfrac{k^2}{e^{5k}}},$$

using $e^{2k}/e^{5k} = e^{-3k}$.

And notice

$$\frac{k}{e^{2k}} \to 0 \quad \text{and} \quad \frac{k^2}{e^{5k}} \to 0$$

by L'Hospital. This yields

$$a_k \approx \frac{1}{e^{3k}} = b_k$$

since

$$\frac{a_k}{b_k} = \frac{\dfrac{e^{2k}+k}{e^{5k}-k^2}}{\dfrac{1}{e^{3k}}} \to 1.$$

(Concretely, $a_k/b_k$ equals the second factor above, $\dfrac{1 + k e^{-2k}}{1 - k^2 e^{-5k}}$, which tends to $\dfrac{1+0}{1-0} = 1$.)

Now $0 < 1/e^3 < 1$ and so

$$\sum_{k=1}^{\infty} b_k = \sum_{k=1}^{\infty} \left(\frac{1}{e^3}\right)^k$$

converges since we have a Geometric Series.

We have

$$\sum_{k=1}^{\infty} \frac{e^{2k}+k}{e^{5k}-k^2}$$

converges by the Limit Comparison Test.

### Example #5

Determine if

$$\sum_{k=1}^{\infty} \frac{\sin(1/k)}{k}$$

converges.

The picture to keep in mind is $y = \sin(x)$ on $[-\pi, \pi]$ together with the line through the origin of slope $1$: the two agree closely for $x$ near $0$.

For $1/k$ "small",

$$\sin\left(\frac{1}{k}\right) \approx \frac{1}{k} \implies \frac{1}{k}\sin\left(\frac{1}{k}\right) \approx \frac{1}{k^2}$$

so let

$$a_k = \frac{1}{k}\sin\left(\frac{1}{k}\right) \quad \text{and} \quad b_k = \frac{1}{k^2}.$$

Now

$$\frac{a_k}{b_k} = \frac{\dfrac{1}{k}\sin\left(\dfrac{1}{k}\right)}{\dfrac{1}{k^2}}
= \frac{1}{k}\sin\left(\frac{1}{k}\right) \cdot k^2
= \frac{\sin\left(\dfrac{1}{k}\right)}{\dfrac{1}{k}} \to 1$$

as $k \to \infty$ since

$$\lim_{x\to 0} \frac{\sin(x)}{x} = 1$$

by L'Hospital's Rule or special trig limits in §3.3. (As $k \to \infty$, the argument $x = 1/k \to 0$.)

Then

$$\sum_{k=1}^{\infty} \frac{1}{k}\sin\left(\frac{1}{k}\right)$$

converges by the LCT with $\sum \dfrac{1}{k^2}$. A similar LCT argument shows

$$\sum_{k=1}^{\infty} \sin\left(\frac{1}{k}\right) = \infty$$

since

$$\sum_{k=1}^{\infty} \frac{1}{k} = \infty.$$

## Common mistakes and tips

**Comparing in the useless direction.** This is the trap in Example #3. An upper bound by a divergent series says nothing, and a lower bound by a convergent series says nothing. Only two of the four combinations are informative. Before you write the inequality, decide which conclusion you want and make sure the inequality points that way.

**Forgetting positivity.** Both tests require positive (or at least nonnegative) terms. Lemma #1 is what rules out oscillation, and it is stated for nonnegative sequences. Neither test as stated here applies to a series with terms that change sign.

**Thinking you need the inequality for every $n$.** The Remark is explicit: $0 < a_n \le b_n$ only has to hold for $n$-large. Example #1 starts the inequality at $k > 2$ and Example #2 at $k > 3$, and neither is a problem. The same relaxation holds for the Integral Test's decreasing hypothesis.

**Reading a convergence proof as a value.** The MCT is an existence result. Showing $\sum a_k$ converges by comparison tells you the sum is finite, not what it equals.

**Mishandling the LCT ratio.** The ratio is $a_n/b_n$ with the series you are testing on top. Simplify the complex fraction by multiplying by the reciprocal of $b_k$ before taking the limit, then divide through by the dominant term.

**Ignoring the $c > 0$ and finite requirement.** The theorem as stated requires $c$ to exist and satisfy $0 < c < \infty$. If $c = 0$ or $c = \infty$, this version of the test gives no conclusion.

**Choosing a comparison series you do not actually know.** The whole point is to land on a $p$-series or a geometric series. If your $b_k$ is as hard as $a_k$, you have not made progress.

**Exponentials.** Factor out the dominant exponential rather than attacking the whole quotient with L'Hospital. In Example #4, the real work is recognizing that $e^{2k}/e^{5k} = e^{-3k}$ makes the comparison series geometric with ratio $1/e^3$.

## Formula sheet

**Lemma #1.** For $(a_n)$ nonnegative, $\sum a_n$ converges to a finite number or diverges to $\infty$.

**Basic Comparison Test.** If $0 < a_n \le b_n$ for all $n$ (or all $n$-large), then

$$\sum a_n = \infty \implies \sum b_n = \infty,
\qquad
\sum b_n \in \mathbf{R} \implies \sum a_n \in \mathbf{R}.$$

**Limit Comparison Test.** For $(a_n), (b_n)$ positive with

$$c = \lim_{n\to\infty}\frac{a_n}{b_n}$$

existing and $0 < c < \infty$, the series $\sum a_n$ and $\sum b_n$ both converge or both diverge.

**Partial sums.**

$$S_n = \sum_{k=1}^{n} a_k, \qquad T_n = \sum_{k=1}^{n} b_k, \qquad 0 < a_k \le b_k \implies 0 < S_n \le T_n.$$

**Key inequality inside the LCT proof.** For $k > N$,

$$0 < \frac{c}{2}b_k < a_k < \frac{3c}{2}b_k.$$

**Comparison series used here.**

$$\sum_{k=1}^{\infty} \frac{1}{k^p} \text{ converges when } p > 1 \text{ and diverges when } p \le 1,
\qquad
\sum_{k=1}^{\infty} r^k \text{ converges when } 0 < r < 1.$$

**Growth facts used here.**

$$0 < \ln(x) \le x^{1/2} \text{ on } [4,\infty),
\qquad
\frac{k}{e^{2k}} \to 0,
\qquad
\frac{k^2}{e^{5k}} \to 0,
\qquad
\lim_{x\to 0}\frac{\sin(x)}{x} = 1.$$

**Results established in the examples.**

$$\sum_{k=2}^{\infty}\frac{1}{\sqrt{k}-1} = \infty,
\qquad
\sum_{k=2}^{\infty}\frac{\ln(k)}{k^2} \text{ converges},
\qquad
\sum_{k=1}^{\infty}\frac{k}{k^2+2k+3} = \infty,$$

$$\sum_{k=1}^{\infty}\frac{e^{2k}+k}{e^{5k}-k^2} \text{ converges},
\qquad
\sum_{k=1}^{\infty}\frac{\sin(1/k)}{k} \text{ converges},
\qquad
\sum_{k=1}^{\infty}\sin\left(\frac{1}{k}\right) = \infty.$$
