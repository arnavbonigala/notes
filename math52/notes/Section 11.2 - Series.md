# Section 11.2 - Series

## Summary

This section turns sequences into sums. Given a sequence $(a_n)$, the infinite series $\sum_{k=1}^{\infty} a_k$ is defined not by adding infinitely many numbers directly, but as the limit of its sequence of partial sums $S_n = \sum_{k=1}^{n} a_k$. Convergence of the series means convergence of $(S_n)$ to a real number.

Three big results come out of this:

1. **The geometric series.** $\sum_{k=0}^{\infty} r^k = \dfrac{1}{1-r}$ exactly when $|r| < 1$, proved by finding a closed form for $S_n(r)$ and taking $n \to \infty$.
2. **Telescoping sums.** Partial fractions can collapse a partial sum down to a couple of surviving terms, after which the limit is easy.
3. **The Basic Divergence Test.** If $\sum a_n$ converges then $a_n \to 0$. Contrapositive: if $a_n \not\to 0$, the series diverges. The converse is false, and §11.3 will show it.

Series also obey linearity: convergent series can be scaled and added term by term.

## Definitions and theorems

### Definition 1 (Infinite Series)

Let $(a_n)$ be a sequence. Then

$$\sum_{k=1}^{\infty} a_k = a_1 + a_2 + a_3 + \cdots$$

is an **infinite series**.

### Definition 2 (Partial Sum)

Let $(a_n)$ be a sequence. Then

$$S_n = \sum_{k=1}^{n} a_k = a_1 + a_2 + \cdots + a_n$$

is the **$n$th partial sum** of the series.

A useful recursion follows immediately:

$$S_{n+1} = \sum_{k=1}^{n+1} a_k = \underbrace{a_1 + \cdots + a_n}_{S_n} + a_{n+1} = S_n + a_{n+1}.$$

### Definition 3 (Convergence)

If

$$\lim_{n \to \infty} S_n = S \in \mathbf{R}$$

then the series $\sum a_n$ **converges**. If $(S_n)$ diverges then the series $\sum a_n$ **diverges**.

Equivalently, we are requiring the partial sums to converge:

$$\lim_{n \to \infty} S_n = \lim_{n \to \infty} \sum_{k=1}^{n} a_k = \sum_{k=1}^{\infty} a_k.$$

### Remark on notation

1. $\sum a_n$ or $\sum a_k$ denotes the series.
2. $\displaystyle\sum_{n=1}^{\infty} a_n$ is the value of the series starting at $n = 1$.
3. $\displaystyle\sum_{n=2}^{\infty} a_n$ is the value of the series starting at $n = 2$.

The analog for functions: with $f : \mathbf{R} \to \mathbf{R}$, $f(x) = x^2$, the symbol $f$ is the name of the function while $f(x)$ is the value of the function at $x$. Using $f(x)$ to refer to the function $f$ is common practice in mathematics.

### Remark on the dummy index

The index of summation is a dummy variable:

$$\sum_{k=1}^{\infty} a_k = \sum_{n=1}^{\infty} a_n.$$

However,

$$\sum_{k=1}^{\infty} a_k \neq \sum_{n=1}^{\infty} a_k$$

since

$$\sum_{n=1}^{\infty} a_k = a_k + a_k + a_k + \cdots$$

while

$$\sum_{k=1}^{\infty} a_k = a_1 + a_2 + a_3 + \cdots.$$

You can think of $\displaystyle\sum_{n=1}^{\infty} a_n$ as integrating the sequence $(a_n)$ over $[1, \infty)$.

### Lemma 1

Let $r \in \mathbf{R}$. Then

$$\lim_{n \to \infty} r^n = \begin{cases} 1, & r = 1 \\ 0, & r \in (-1, 1). \end{cases}$$

For any other $r$, the sequence diverges.

Reasons: $\lim_{n\to\infty} 1^n = 1$, while $\lim_{n\to\infty} (-1)^n$ does not exist. If $|r| < 1$ then $r^n \to 0$ because we have exponential decay. If $|r| > 1$ then $|r|^n \to \infty$.

### Definition 4 (Geometric Series)

Let $r \in \mathbf{R}$. The **geometric series**

$$\sum_{k=0}^{\infty} r^k = 1 + r + r^2 + \cdots = \frac{1}{1-r}$$

exists only for $|r| < 1$.

### Remark ($0^0$ convention)

$$\sum_{k=0}^{\infty} 0^k = 1 + 0 + 0 + \cdots$$

means we are defining $0^0 = 1$ in our summation.

### Corollary 1 (Geometric series starting at $k = 1$)

$$\sum_{k=1}^{\infty} r^k = \frac{r}{1-r} \qquad \text{for } |r| < 1.$$

### Theorem 1 (Term test, necessary condition)

If $\sum a_n$ converges, then $a_n \to 0$.

### Corollary 2 (Basic Divergence Test, B.D.T.)

Let $(a_n)$ be a sequence. If

$$\lim_{n \to \infty} a_n \neq 0 \qquad \text{or} \qquad \lim_{n \to \infty} a_n \ \text{d.n.e.}$$

then $\sum a_n$ diverges.

### Warning (converse fails)

There exists a sequence $(a_n)$ such that

$$a_n \to 0 \quad \text{and} \quad \sum a_n \ \text{diverges}.$$

This will be shown explicitly in §11.3.

### Theorem 2 (Linearity)

If $\sum a_n$ and $\sum b_n$ exist then

$$\sum_{k=1}^{\infty} (c \cdot a_k + b_k) = c\sum_{k=1}^{\infty} a_k + \sum_{k=1}^{\infty} b_k$$

for any $c \in \mathbf{R}$.

## Methods

### Proof of the geometric series formula

Let

$$S_n(r) = \sum_{k=0}^{n} r^k = 1 + r + r^2 + \cdots + r^n.$$

Multiply by $r$ and subtract:

$$S_n(r) - r S_n(r) = \left(1 + r + r^2 + \cdots + r^n\right) - \left(r + r^2 + \cdots + r^{n+1}\right) = 1 - r^{n+1},$$

since every term from $r$ through $r^n$ cancels. Factor the left-hand side:

$$(1 - r) S_n(r) = 1 - r^{n+1},$$

$$S_n(r) = \frac{1 - r^{n+1}}{1 - r} \qquad \text{for } r \neq 1.$$

If $|r| < 1$ then $|r^{n+1}| \to 0$ as $n \to \infty$, so

$$\frac{1}{1-r} = \lim_{n \to \infty} S_n(r) = \sum_{k=0}^{\infty} r^k.$$

### Shifting the starting index

Remove the constant term from the geometric series. Since

$$\sum_{k=0}^{\infty} r^k = 1 + r + r^2 + \cdots, \qquad \sum_{k=1}^{\infty} r^k = r + r^2 + \cdots,$$

we get

$$\sum_{k=1}^{\infty} r^k = \left(\sum_{k=0}^{\infty} r^k\right) - 1 = \frac{1}{1-r} - 1 = \frac{1}{1-r} - \frac{1-r}{1-r} = \frac{r}{1-r}.$$

### Proof of Theorem 1

If $\sum_{k=1}^{\infty} a_k = S \in \mathbf{R}$, then $S_n \to S$ and $S_{n+1} \to S$ (a shifted convergent sequence has the same limit). Also

$$S_{n+1} - S_n = \sum_{k=1}^{n+1} a_k - \sum_{k=1}^{n} a_k = a_{n+1}.$$

Thus

$$S_{n+1} - S_n \to S - S = 0 \implies a_{n+1} \to 0.$$

This is a consistency condition: all convergent sums $\sum a_n$ must satisfy $a_n \to 0$.

### Using the B.D.T. on geometric series

The B.D.T. gives an immediate argument for why $\sum r^n$ diverges whenever $|r| \geq 1$: by Lemma 1, $\lim_{n\to\infty} r^n$ does not exist (or is nonzero) for $|r| \geq 1$, so the terms fail to go to $0$.

### Telescoping sums

When the term is a rational function of $k$ whose denominator factors, apply a partial fraction decomposition (PFD) exactly as with integration of rational functions. Write the partial sum $S_n$ with the split terms, list several terms explicitly, and cancel. What survives is a short expression in $n$, and then take $n \to \infty$.

### Repeating decimals

Write the repeating decimal as a geometric series in powers of $1/10$, $1/100$, and so on, sum it with Corollary 1, and add the pieces using Theorem 2.

## Worked examples

### Example 1

Find $\displaystyle\sum_{k=0}^{\infty} 2^{-k}$.

Since $2^{-k} = (1/2)^k$, this is a geometric sum with $r = 1/2$:

$$\sum_{k=0}^{\infty} \left(\frac{1}{2}\right)^k = \frac{1}{1 - 1/2} = \frac{1}{1/2} = 2.$$

The sum corresponds to the area of a sequence of boxes where each box is half the height of the last and the initial box has height $1$.

### Example 2

Find $\displaystyle\sum_{k=1}^{\infty} \frac{2}{(2k+1)(2k+3)}$.

Here

$$S_n = \sum_{k=1}^{n} \frac{2}{(2k+1)(2k+3)}.$$

Just like with integration of rational functions, apply a PFD:

$$\frac{2}{(2k+1)(2k+3)} = \frac{A}{2k+1} + \frac{B}{2k+3},$$

$$2 = A(2k+3) + B(2k+1).$$

If $k = -\tfrac{1}{2}$: the $B$ term has factor $2(-\tfrac12)+1 = 0$, so

$$2 = A(-1+3) = 2A \implies A = 1.$$

If $k = -\tfrac{3}{2}$: the $A$ term has factor $2(-\tfrac32)+3 = 0$, so

$$2 = B(-3+1) = -2B \implies B = -1.$$

Then

$$S_n = \sum_{k=1}^{n}\left(\frac{1}{2k+1} - \frac{1}{2k+3}\right)$$

$$= \left(\frac13 - \frac15\right) + \left(\frac15 - \frac17\right) + \left(\frac17 - \frac19\right) + \cdots + \left(\frac{1}{2n-1} - \frac{1}{2n+1}\right) + \left(\frac{1}{2n+1} - \frac{1}{2n+3}\right),$$

and notice all the terms that cancel. Every negative term $-\frac{1}{2k+3}$ is killed by the positive term $\frac{1}{2(k+1)+1} = \frac{1}{2k+3}$ of the next bracket. Only the first positive term and the last negative term survive:

$$S_n = \frac13 - \frac{1}{2n+3}.$$

We call this a **telescoping sum**. Now

$$S_n \to \frac13$$

since $\dfrac{1}{2n+3} \to 0$ as $n \to \infty$. This shows

$$\sum_{k=1}^{\infty} \frac{2}{(2k+1)(2k+3)} = \frac13.$$

### Example 3

Determine if $\displaystyle\sum n \sin\left(\frac1n\right)$ converges.

Here, substituting $h = 1/n$ so that $h \to 0^+$ as $n \to \infty$, and noting $n \sin(1/n) = \frac{\sin(1/n)}{1/n}$:

$$\lim_{n \to \infty} n \sin\left(\frac1n\right) = \lim_{h \to 0^+} \frac{\sin(h)}{h} = 1$$

by geometry or l'Hospital. The terms tend to $1 \neq 0$, thus

$$\sum n \sin\left(\frac1n\right)$$

diverges by the B.D.T.

### Example 4

Express $0.111111111111\ldots$ and $0.121212121212\ldots$ as ratios of integers.

Notice that

$$0.111111\ldots = \frac{1}{10} + \frac{1}{100} + \frac{1}{1{,}000} + \cdots = \sum_{k=1}^{\infty}\left(\frac{1}{10}\right)^k.$$

By Corollary 1 with $r = 1/10$:

$$= \frac{\frac{1}{10}}{1 - \frac{1}{10}} = \frac{\frac{1}{10}}{\frac{9}{10}} = \frac19.$$

And

$$0.010101\ldots = \frac{1}{100} + \frac{1}{10{,}000} + \frac{1}{1{,}000{,}000} + \cdots = \frac{1}{100} + \frac{1}{100^2} + \frac{1}{100^3} + \cdots = \sum_{k=1}^{\infty}\left(\frac{1}{100}\right)^k$$

$$= \frac{\frac{1}{100}}{1 - \frac{1}{100}} = \frac{\frac{1}{100}}{\frac{99}{100}} = \frac{1}{99}.$$

Hence

$$0.121212\ldots = 0.111111\ldots + 0.010101\ldots = \frac19 + \frac{1}{99} = \frac{11}{99} + \frac{1}{99} = \frac{12}{99} = \frac{4}{33}.$$

### Example 5

Find the values of $x$ such that $\displaystyle\sum_{k=0}^{\infty} e^{kx}$ converges.

For $k \in \mathbf{Z}$,

$$e^{kx} = (e^x)^k,$$

and $|e^x| < 1$ whenever $x < 0$. We have a convergent geometric sum with ratio $r = e^x$:

$$\sum_{k=0}^{\infty} e^{kx} = \sum_{k=0}^{\infty} (e^x)^k = \frac{1}{1 - e^x}$$

given $x < 0$. If $x \geq 0$, then $e^x \geq 1$ and the sum diverges.

## Common mistakes and tips

- **A series is a limit of partial sums, not a giant addition.** Every convergence claim ultimately comes back to $\lim_{n\to\infty} S_n$. If you can get a closed form for $S_n$, you are done.
- **$a_n \to 0$ does not prove convergence.** The Warning is explicit: there is a sequence with $a_n \to 0$ whose series still diverges (§11.3). The B.D.T. is a one-way test: it can only prove divergence.
- **The B.D.T. concluding "diverges" needs $\lim a_n \neq 0$ or the limit failing to exist.** If the limit is $0$, the test gives no information at all, not "converges".
- **Watch the starting index on geometric series.** $\sum_{k=0}^{\infty} r^k = \frac{1}{1-r}$ but $\sum_{k=1}^{\infty} r^k = \frac{r}{1-r}$. When in doubt, write out the first few terms or subtract the missing ones off, as in the derivation above.
- **Check $|r| < 1$ before applying the formula.** For $|r| \geq 1$ the formula is meaningless; the series diverges.
- **Get the ratio into the form $r^k$ first.** In Example 1 rewrite $2^{-k}$ as $(1/2)^k$; in Example 5 rewrite $e^{kx}$ as $(e^x)^k$. Then read off $r$.
- **The index is a dummy variable only when it actually appears in the summand.** $\sum_{n=1}^{\infty} a_k$ is a sum of a constant repeated, not the same object as $\sum_{k=1}^{\infty} a_k$.
- **On telescoping sums, write out enough terms to see the pattern,** including the last two brackets in terms of $n$. Guessing what cancels without writing the tail is how the surviving term gets lost.
- **PFD constants:** plug in the values of $k$ that zero out a factor. That isolates one unknown at a time.
- **$0^0 = 1$ inside these summations,** by the convention stated in the Remark, so that $\sum_{k=0}^\infty 0^k$ makes sense.

## Formula sheet

**Series and partial sums**

$$\sum_{k=1}^{\infty} a_k = a_1 + a_2 + a_3 + \cdots, \qquad S_n = \sum_{k=1}^{n} a_k = a_1 + \cdots + a_n$$

$$S_{n+1} = S_n + a_{n+1}, \qquad S_{n+1} - S_n = a_{n+1}$$

$$\sum_{k=1}^{\infty} a_k = \lim_{n \to \infty} S_n \quad \text{(converges if this limit is in } \mathbf{R}\text{)}$$

**Powers of $r$**

$$\lim_{n \to \infty} r^n = \begin{cases} 1, & r = 1 \\ 0, & |r| < 1 \end{cases} \qquad \text{diverges otherwise}$$

**Finite geometric sum**

$$\sum_{k=0}^{n} r^k = \frac{1 - r^{n+1}}{1 - r}, \qquad r \neq 1$$

**Infinite geometric series** (both require $|r| < 1$)

$$\sum_{k=0}^{\infty} r^k = \frac{1}{1-r}, \qquad \sum_{k=1}^{\infty} r^k = \frac{r}{1-r}$$

**Term test / Basic Divergence Test**

$$\sum a_n \text{ converges} \implies a_n \to 0$$

$$\lim_{n\to\infty} a_n \neq 0 \ \text{ or d.n.e.} \implies \sum a_n \text{ diverges}$$

$$a_n \to 0 \ \not\Longrightarrow \ \sum a_n \text{ converges}$$

**Linearity** (for convergent $\sum a_n$, $\sum b_n$, and $c \in \mathbf{R}$)

$$\sum_{k=1}^{\infty} (c \cdot a_k + b_k) = c\sum_{k=1}^{\infty} a_k + \sum_{k=1}^{\infty} b_k$$

**Telescoping template**

$$\sum_{k=1}^{n} \left(b_k - b_{k+1}\right) = b_1 - b_{n+1}$$

**Worked values from this section**

$$\sum_{k=0}^{\infty} 2^{-k} = 2, \qquad \sum_{k=1}^{\infty} \frac{2}{(2k+1)(2k+3)} = \frac13$$

$$0.\overline{1} = \frac19, \qquad 0.\overline{01} = \frac{1}{99}, \qquad 0.\overline{12} = \frac{4}{33}$$

$$\sum_{k=0}^{\infty} e^{kx} = \frac{1}{1 - e^x} \quad (x < 0), \qquad \text{diverges for } x \geq 0$$
