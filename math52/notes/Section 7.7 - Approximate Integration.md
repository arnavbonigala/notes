# Section 7.7 - Approximate Integration

## Summary

This section estimates a definite integral $\int_a^b f(x)\,dx$ numerically. You split $[a,b]$ into $n$ equal pieces and add up simple areas. The rectangle methods (left, right, and midpoint Riemann sums) treat $f$ as constant on each piece. The Trapezoidal Rule treats $f$ as linear on each piece, and Simpson's Rule treats $f$ as quadratic on each pair of pieces. The error bound theorems use a bound $K$ on $f''$ or $f^{(4)}$ to say how far off the Midpoint, Trapezoidal, and Simpson approximations can be. Use these methods when an antiderivative is hard or impossible to find, or when you only know $f$ at evenly spaced points. For the same $n$, Simpson's Rule is usually far more accurate than the others.

## Definitions and theorems

### The problem

We want to approximate

$$
\int_a^b f(x)\,dx.
$$

### Definition #1 (Uniform Partition)

Let $[a,b] \subset \mathbf{R}$, $n \in \mathbf{N}$ and

$$
\Delta x = \frac{b-a}{n}.
$$

Then

$$
P = \{x_0, x_1, \dots, x_n\}
$$

where

$$
x_k = a + k\Delta x = x_{k-1} + \Delta x
$$

is a **uniform partition** of $[a,b]$ with **step-size** $\Delta x$.

Note that $x_0 = a$ and $x_n = a + n\Delta x = b$.

### Definition #2 (Left Riemann Sum)

Assume $[a,b]$ is an interval, $\Delta x = (b-a)/n$, and $f$ is integrable on $[a,b]$. Then

$$
L_n = \sum_{k=1}^{n} f(x_{k-1})\,\Delta x = \big(f(x_0) + f(x_1) + \dots + f(x_{n-1})\big)\Delta x
$$

is a **left Riemann sum** approximation of $\int_a^b f$.

The height of each rectangle is the value of $f$ at the **left** endpoint of its subinterval.

### Definition #3 (Right Riemann Sum)

Assume $[a,b]$ is an interval, $\Delta x = (b-a)/n$, and $f$ is integrable on $[a,b]$. Then

$$
R_n = \sum_{k=1}^{n} f(x_k)\,\Delta x = \big(f(x_1) + \dots + f(x_n)\big)\Delta x
$$

is a **right Riemann sum** approximation of $\int_a^b f$.

The height of each rectangle is the value of $f$ at the **right** endpoint of its subinterval.

### Remark (left vs. right)

$$
L_n - R_n = \big(f(x_0) - f(x_n)\big)\Delta x = \big(f(a) - f(b)\big)\Delta x
$$

and

$$
\int_a^b f(x)\,dx \approx L_n \approx R_n.
$$

Why: $L_n$ and $R_n$ share the terms $f(x_1), \dots, f(x_{n-1})$, so those cancel when you subtract. What remains is $f(x_0)$ from $L_n$ and $f(x_n)$ from $R_n$.

### Definition #4 (Midpoint Riemann Sum)

Assume $[a,b]$ is an interval, $\Delta x = (b-a)/n$, and $f$ is integrable on $[a,b]$. Then

$$
M_n = \sum_{k=1}^{n} f(\overline{x}_k)\,\Delta x = \big(f(\overline{x}_1) + \dots + f(\overline{x}_n)\big)\Delta x
$$

where

$$
\overline{x}_k = x_{k-1} + \Delta x/2 = \frac{x_{k-1} + x_k}{2}
$$

is a **midpoint Riemann sum** approximation of $\int_a^b f$.

The midpoints $\overline{x}_k$ are **new** points. The midpoint sum does **not** reuse the positions from the left/right sums.

### Definition #5 (Trapezoidal Rule)

Assume $[a,b]$ is an interval, $\Delta x = (b-a)/n$, and $f$ is integrable on $[a,b]$. Then

$$
T_n = \left(\frac{f(x_0)}{2} + \sum_{k=1}^{n-1} f(x_k) + \frac{f(x_n)}{2}\right)\Delta x
$$

$$
= \big(f(x_0) + 2f(x_1) + 2f(x_2) + \dots + 2f(x_{n-1}) + f(x_n)\big)\frac{\Delta x}{2}
$$

is a **trapezoidal rule** approximation of $\int_a^b f$.

The Trapezoidal Rule **does** reuse the positions from the left/right sums.

### Remark (Trapezoidal Rule vs. Riemann sums)

Naturally,

$$
\int_a^b f(x)\,dx \approx T_n
$$

and

$$
T_n = \frac{L_n + R_n}{2},
$$

but $T_n$ is **not** a Riemann sum.

Why $T_n = \frac{L_n + R_n}{2}$: the interior points $x_1, \dots, x_{n-1}$ appear in both sums, while $x_0$ appears only in $L_n$ and $x_n$ only in $R_n$. So

$$
L_n + R_n = \big(f(x_0) + 2f(x_1) + \dots + 2f(x_{n-1}) + f(x_n)\big)\Delta x.
$$

Dividing by 2 gives the second formula for $T_n$.

We are not estimating the area with rectangles, so the Trapezoidal Rule is a different kind of approximation. On each subinterval it replaces the curve with the straight line segment from $(x_{k-1}, f(x_{k-1}))$ to $(x_k, f(x_k))$.

### Exactness and over/underestimates

- Left/right Riemann sums are **exact** if $f$ is constant over the partition.
- The Trapezoidal Rule is **exact** if $f$ is piecewise linear over the partition.

If $f(x) \ge 0$ on $[a,b]$:

- **Left sum:** overestimates if $f' < 0$ ($f$ decreasing). Each height is taken at the left end, where $f$ is largest on that subinterval.
- **Right sum:** overestimates if $f' > 0$ ($f$ increasing). Each height is taken at the right end, where $f$ is largest on that subinterval.
- **Trapezoidal:** overestimates if $f'' > 0$ ($f$ concave up). The chords of a concave up curve lie above the curve.

### The simple case $n = 1$ ($\Delta x = b - a$)

- Left sum: $f(x_0)\Delta x$
- Right sum: $f(x_1)\Delta x$
- Midpoint sum: $f\left(x_0 + \dfrac{\Delta x}{2}\right)\Delta x$
- Trapezoidal sum: $\dfrac{1}{2}\big(f(x_0) + f(x_1)\big)\Delta x$

The trapezoidal formula is the area of a trapezoid with parallel sides $f(x_0)$ and $f(x_1)$ and width $\Delta x$. With $f(x_0) > f(x_1)$ as drawn on the slide, split it into a rectangle of height $f(x_1)$ and a triangle of height $f(x_0) - f(x_1)$. Both have width $\Delta x$:

$$
\underbrace{f(x_1)\Delta x}_{\text{rectangle}} + \underbrace{\frac{1}{2}\big(f(x_0) - f(x_1)\big)\Delta x}_{\text{triangle}} = \frac{1}{2}f(x_0)\Delta x + \frac{1}{2}f(x_1)\Delta x = \frac{1}{2}\big(f(x_0) + f(x_1)\big)\Delta x.
$$

### Definition #6 (Error Terms)

Assume $M_n$ and $T_n$ are the midpoint and trapezoidal rule approximations of $\int_a^b f$. Then

$$
E_M(n) = \int_a^b f(x)\,dx - M_n \quad \text{and} \quad E_T(n) = \int_a^b f(x)\,dx - T_n
$$

are the **error terms** of the approximations.

Sign convention: error = exact minus approximation. A **negative** error means the approximation is an **overestimate**. A **positive** error means it is an **underestimate**.

### Theorem #1 (Midpoint and Trapezoidal error bounds)

Let $f''$ be continuous on $[a,b]$ and assume $M_n$ and $T_n$ are midpoint and trapezoidal rule approximations of $\int_a^b f$. Then

$$
|E_M(n)| \le \frac{K(b-a)^3}{24n^2} = \frac{K(b-a)}{24}(\Delta x)^2
$$

and

$$
|E_T(n)| \le \frac{K(b-a)^3}{12n^2} = \frac{K(b-a)}{12}(\Delta x)^2
$$

where $|f''(x)| \le K$ for all $x \in [a,b]$.

The two forms agree because $\Delta x = \frac{b-a}{n}$, so $(b-a)(\Delta x)^2 = \frac{(b-a)^3}{n^2}$.

### Definition #7 (Simpson's Rule)

Assume $[a,b]$ is an interval, $n$ is **even**, $\Delta x = (b-a)/n$, and $f$ is integrable on $[a,b]$. Then

$$
S_n = \big(f(x_0) + 4f(x_1) + 2f(x_2) + 4f(x_3) + 2f(x_4) + \dots + 4f(x_{n-3}) + 2f(x_{n-2}) + 4f(x_{n-1}) + f(x_n)\big)\frac{\Delta x}{3}
$$

is a **Simpson's rule** approximation of $\int_a^b f$.

The Trapezoidal Rule approximates $f$ with line segments. Simpson's Rule approximates $f$ with a **quadratic polynomial**: a parabola through three consecutive partition points on each pair of subintervals.

### Why Simpson's Rule uses weights $1, 4, 1$

Consider Simpson's Rule over the partition $P = \{-\Delta x, 0, \Delta x\}$, so $x_0 = -\Delta x$, $x_1 = 0$, $x_2 = \Delta x$. Let

$$
f(x) \approx p(x) = A_2x^2 + A_1x + A_0
$$

with $f(x_0) = p(x_0)$, $f(x_1) = p(x_1)$, $f(x_2) = p(x_2)$. We expect

$$
\int_{x_0}^{x_2} p(x)\,dx \approx \int_{x_0}^{x_2} f(x)\,dx
$$

because $f$ and $p$ are equal at the partition points.

**Step 1: integrate $p$.** Split it into an even part and an odd part:

$$
\int_{x_0}^{x_2} p(x)\,dx = \int_{-\Delta x}^{\Delta x} (A_2x^2 + A_1x + A_0)\,dx = \int_{-\Delta x}^{\Delta x} \underbrace{(A_2x^2 + A_0)}_{\text{even function}}\,dx + \int_{-\Delta x}^{\Delta x} \underbrace{A_1x}_{\text{odd function}}\,dx.
$$

An odd function integrates to 0 over a symmetric interval. An even function integrates to twice its integral over $[0, \Delta x]$:

$$
= 2\int_0^{\Delta x} (A_2x^2 + A_0)\,dx = \left[\frac{2}{3}A_2x^3 + 2A_0x\right]_{x=0}^{\Delta x} = \frac{2}{3}A_2(\Delta x)^3 + 2A_0\Delta x = \frac{\Delta x}{3}\left(2A_2(\Delta x)^2 + 6A_0\right).
$$

**Step 2: take the weighted sum of the values of $f$.**

$$
f(x_0) = p(-\Delta x) = A_2(\Delta x)^2 - A_1\Delta x + A_0
$$

$$
4f(x_1) = 4p(0) = 4A_0
$$

$$
f(x_2) = p(\Delta x) = A_2(\Delta x)^2 + A_1\Delta x + A_0
$$

Adding these, the $A_1\Delta x$ terms cancel:

$$
f(x_0) + 4f(x_1) + f(x_2) = 2A_2(\Delta x)^2 + 6A_0.
$$

**Step 3: compare.** This sum matches the expression in parentheses from Step 1, so

$$
\frac{1}{3}\big(f(x_0) + 4f(x_1) + f(x_2)\big)\Delta x = \int_{x_0}^{x_2} p(x)\,dx \approx \int_{x_0}^{x_2} f(x)\,dx.
$$

We have evaluated the integral of $p$ as a weighted sum of $f(x_0)$, $f(x_1)$, and $f(x_2)$. Translations preserve area, so this also holds for any three uniformly spaced points $x_0, x_1, x_2$ in $\mathbf{R}$.

### Theorem #2 (Simpson error bound)

Let $f^{(4)}$ be continuous on $[a,b]$ and assume $S_n$ is a Simpson's rule approximation of $\int_a^b f$. Then

$$
E_S(n) = \int_a^b f(x)\,dx - S_n
$$

satisfies

$$
|E_S(n)| \le \frac{K(b-a)^5}{180n^4} = \frac{K(b-a)}{180}(\Delta x)^4
$$

where $|f^{(4)}(x)| \le K$ for all $x \in [a,b]$.

Simpson's Rule is exact for polynomials of degree 3 or less, because their fourth derivative is 0 and so $K = 0$. The method itself is designed to integrate quadratics exactly.

## Methods

### Method 1: Build the uniform partition

1. Compute $\Delta x = \dfrac{b-a}{n}$.
2. List $x_k = a + k\Delta x$ for $k = 0, 1, \dots, n$. Each point is the previous one plus $\Delta x$.
3. For the midpoint sum, also list $\overline{x}_k = x_{k-1} + \Delta x/2$ for $k = 1, \dots, n$.

Slide example: $a = 1$ and $b = 3$, so $\Delta x = \frac{3-1}{n} = \frac{2}{n}$.

| $n$ | $\Delta x$ | $P$ |
|---|---|---|
| 2 | $1$ | $\{1, 2, 3\}$ |
| 4 | $1/2$ | $\{1, 3/2, 2, 5/2, 3\}$ |
| 8 | $1/4$ | $\{1, 5/4, 3/2, 7/4, 2, 9/4, 5/2, 11/4, 3\}$ |

### Method 2: Left, right, and midpoint sums

Use these for a quick rectangle estimate, or when a problem asks for $L_n$, $R_n$, or $M_n$ by name.

1. Build the partition (Method 1).
2. Evaluate $f$ at the correct points:
   - $L_n$: $x_0, \dots, x_{n-1}$ (every point except the last).
   - $R_n$: $x_1, \dots, x_n$ (every point except the first).
   - $M_n$: the midpoints $\overline{x}_1, \dots, \overline{x}_n$ (new points).
3. Add the values and multiply by $\Delta x$.

With $a = 1$, $b = 3$, $n = 4$, $\Delta x = 1/2$:

$$
L_4 = \left(f(1) + f(3/2) + f(2) + f(5/2)\right)\tfrac{1}{2}
$$

$$
R_4 = \left(f(3/2) + f(2) + f(5/2) + f(3)\right)\tfrac{1}{2}
$$

$$
M_4 = \left(f(5/4) + f(7/4) + f(9/4) + f(11/4)\right)\tfrac{1}{2}
$$

### Method 3: Trapezoidal Rule

Use this when you have values of $f$ at the partition points and want a better estimate than a left or right sum. It is exact for piecewise linear $f$.

1. Build the partition (Method 1).
2. Evaluate $f$ at every partition point $x_0, \dots, x_n$.
3. Give the endpoints weight 1 and every interior point weight 2: $1, 2, 2, \dots, 2, 1$.
4. Add and multiply by $\dfrac{\Delta x}{2}$.

With $a = 1$, $b = 3$, $n = 4$, $\Delta x = 1/2$:

$$
T_4 = \left(f(1) + 2f(3/2) + 2f(2) + 2f(5/2) + f(3)\right)\tfrac{1}{4}.
$$

Shortcut if you already have both sums: $T_n = \dfrac{L_n + R_n}{2}$.

### Method 4: Simpson's Rule

Use this when you want high accuracy for a given $n$. It requires $n$ to be **even**.

1. Check that $n$ is even, then build the partition (Method 1).
2. Evaluate $f$ at every partition point $x_0, \dots, x_n$.
3. Use the weights $1, 4, 2, 4, 2, \dots, 2, 4, 1$. The endpoints get 1, odd-indexed points get 4, and even-indexed interior points get 2.
4. Add and multiply by $\dfrac{\Delta x}{3}$.

The pattern comes from applying the $n = 2$ formula to each pair of subintervals. For $n = 2$:

$$
S_2 = \big(f(x_0) + 4f(x_1) + f(x_2)\big)\frac{\Delta x}{3}.
$$

For $n = 4$, apply that formula on $[x_0, x_2]$ and on $[x_2, x_4]$:

$$
S_4 = \big(f(x_0) + 4f(x_1) + f(x_2)\big)\frac{\Delta x}{3} + \big(f(x_2) + 4f(x_3) + f(x_4)\big)\frac{\Delta x}{3}
$$

$$
= \big(f(x_0) + 4f(x_1) + 2f(x_2) + 4f(x_3) + f(x_4)\big)\frac{\Delta x}{3}.
$$

The shared point $x_2$ shows up in both pieces, so each even-indexed interior point gets weight 2.

For comparison, here are the other methods with $n = 2$ and $\Delta x = \frac{b-a}{2}$:

- left: $\big(f(x_0) + f(x_1)\big)\Delta x$
- right: $\big(f(x_1) + f(x_2)\big)\Delta x$
- midpoint: $\left(f\left(x_0 + \frac{\Delta x}{2}\right) + f\left(x_1 + \frac{\Delta x}{2}\right)\right)\Delta x$
- trapezoidal: $T_2 = \big(f(x_0) + f(x_1)\big)\frac{\Delta x}{2} + \big(f(x_1) + f(x_2)\big)\frac{\Delta x}{2} = \big(f(x_0) + 2f(x_1) + f(x_2)\big)\frac{\Delta x}{2}$

### Method 5: Bounding the error

Use this when a problem asks you to "estimate the error" of an approximation.

1. Pick the theorem. $E_M$ and $E_T$ use $f''$ (Theorem #1). $E_S$ uses $f^{(4)}$ (Theorem #2).
2. Compute the derivative you need.
3. Find $K$, an upper bound for $|f''(x)|$ or $|f^{(4)}(x)|$ on all of $[a,b]$. If that derivative is positive and increasing on $[a,b]$, its largest value is at $x = b$.
4. Compute $\Delta x = \frac{b-a}{n}$ and plug it into the bound:
   - Midpoint: $\dfrac{K(b-a)}{24}(\Delta x)^2$
   - Trapezoidal: $\dfrac{K(b-a)}{12}(\Delta x)^2$
   - Simpson: $\dfrac{K(b-a)}{180}(\Delta x)^4$
5. The result is an upper bound on the size of the error. The actual error can be smaller.

### Method 6: Deciding between overestimate and underestimate

For $f \ge 0$ on $[a,b]$:

- $f$ decreasing: $L_n$ overestimates.
- $f$ increasing: $R_n$ overestimates.
- $f$ concave up: $T_n$ overestimates.

If you know the exact value, you can also compute the error term directly. A negative error means an overestimate and a positive error means an underestimate.

## Worked examples

### Example #1: Compute $T_2$, $M_2$, and $T_4$ for $\int_0^2 e^x\,dx$

**Setup.** $a = 0$, $b = 2$, $f(x) = e^x$.

**$T_2$.** With $n = 2$, $\Delta x = \frac{2-0}{2} = 1$, so the partition is $\{0, 1, 2\}$. The trapezoidal weights are $1, 2, 1$ and the multiplier is $\frac{\Delta x}{2} = \frac{1}{2}$:

$$
T_2 = \left(e^0 + 2e^1 + e^2\right)\frac{1}{2} \approx \big(1 + 5.43656 + 7.38906\big)\frac{1}{2} = \frac{13.82562}{2} \approx 6.91281.
$$

**$M_2$.** Still $\Delta x = 1$. The midpoints are $\overline{x}_1 = 0 + \frac{1}{2} = \frac{1}{2}$ and $\overline{x}_2 = 1 + \frac{1}{2} = \frac{3}{2}$:

$$
M_2 = \left(e^{1/2} + e^{3/2}\right)\cdot 1 \approx 1.64872 + 4.48169 \approx 6.13041.
$$

**$T_4$.** With $n = 4$, $\Delta x = \frac{2}{4} = \frac{1}{2}$, so the partition is $\{0, \frac12, 1, \frac32, 2\}$. The weights are $1, 2, 2, 2, 1$ and the multiplier is $\frac{\Delta x}{2} = \frac{1}{4}$:

$$
T_4 = \left(e^0 + 2e^{1/2} + 2e^1 + 2e^{3/2} + e^2\right)\frac{1}{4}
$$

$$
\approx \big(1 + 3.29744 + 5.43656 + 8.96338 + 7.38906\big)\frac{1}{4} = \frac{26.08644}{4} \approx 6.52161.
$$

**Exact value for comparison.**

$$
\int_0^2 e^x\,dx = e^x\Big|_0^2 = e^2 - 1 \approx 6.38906.
$$

**Errors (Definition #6).**

$$
E_T(2) \approx 6.38906 - 6.91281 = -0.52375
$$

$$
E_M(2) \approx 6.38906 - 6.13041 = 0.25865
$$

$E_T(2) < 0$, so the Trapezoidal Rule gives an **overestimate**. That fits the concavity rule: $e^x$ is positive and concave up ($f'' = e^x > 0$), so the chords lie above the curve. $E_M(2) > 0$, so here the Midpoint Rule gives an **underestimate**.

### Example #2: Estimate $E_T(10)$ and $E_S(6)$ for $\int_0^2 e^x\,dx$

**Find $K$ for the trapezoidal bound.** $f''(x) = e^x$. On $[0,2]$, $f''$ is increasing and positive, so its largest value is at $x = 2$:

$$
|f''(x)| \le e^2 = K.
$$

**Trapezoidal bound for $n = 10$.**

$$
\Delta x = \frac{2 - 0}{10} = \frac{1}{5}.
$$

By Theorem #1,

$$
|E_T(10)| \le \frac{e^2(2-0)}{12}\left(\frac{1}{5}\right)^2 = \frac{2e^2}{12}\cdot\frac{1}{25} = \frac{2e^2}{300} = \frac{1}{150}e^2 \approx 0.05.
$$

**Find $K$ for the Simpson bound.** $f^{(4)}(x) = e^x$, which is also increasing on $[0,2]$, so

$$
|f^{(4)}(x)| \le e^2 = K.
$$

**Simpson bound for $n = 6$.** Simpson's Rule applies because 6 is even.

$$
\Delta x = \frac{2 - 0}{6} = \frac{1}{3}.
$$

By Theorem #2,

$$
|E_S(6)| \le \frac{e^2(2-0)}{180}\left(\frac{1}{3}\right)^4 = \frac{2e^2}{180}\cdot\frac{1}{81} = \frac{2e^2}{14580} = \frac{1}{7290}e^2 \approx 0.001.
$$

**Check.** The actual errors are

$$
|E_S(6)| \approx 0.0004, \qquad |E_T(10)| \approx 0.02.
$$

Both are below their bounds, so the estimates are reasonable. Simpson's Rule with 6 subintervals is more accurate than the Trapezoidal Rule with 10.

### Example #3: Estimate $E_S(4)$ and $E_S(6)$ for $\int_0^4 x^3\,dx$

**Derivatives.** $f(x) = x^3$, $f'(x) = 3x^2$, $f''(x) = 6x$, $f'''(x) = 6$, $f^{(4)}(x) = 0$.

Since $f^{(4)}(x) = 0$ on $[0,4]$, we can take $K = 0$. By Theorem #2,

$$
|E_S(n)| \le \frac{0\cdot(4-0)^5}{180n^4} = 0.
$$

So there is no error for any even $n \ge 2$. That is,

$$
|E_S(2k)| \le 0 \quad \text{for } k \ge 1,
$$

so $E_S(4) = 0$ and $E_S(6) = 0$.

**Check with $n = 4$.** $\Delta x = 1$, the partition is $\{0,1,2,3,4\}$, and the weights are $1,4,2,4,1$:

$$
S_4 = \big(0 + 4(1) + 2(8) + 4(27) + 64\big)\frac{1}{3} = \frac{0 + 4 + 16 + 108 + 64}{3} = \frac{192}{3} = 64.
$$

The exact value is also 64: $\int_0^4 x^3\,dx = \frac{x^4}{4}\Big|_0^4 = \frac{256}{4} = 64$.

**Takeaway.** Simpson's Rule is exact for polynomials of degree 3 or less, even though the method is designed to integrate quadratics exactly.

### Example #4: Find $|E_T(8)|$ and $|E_S(8)|$ for $\int_0^2 e^x\,dx$

The slides build up a table of the actual error sizes as $n$ doubles:

| $n$ | $\lvert E_T(n)\rvert$ | $\lvert E_S(n)\rvert$ |
|---:|---:|---:|
| 2 | $5 \times 10^{-1}$ | $3 \times 10^{-2}$ |
| 4 | $1 \times 10^{-1}$ | $2 \times 10^{-3}$ |
| 8 | $3 \times 10^{-2}$ | $1 \times 10^{-4}$ |
| 16 | $8 \times 10^{-3}$ | $9 \times 10^{-6}$ |
| 32 | $2 \times 10^{-3}$ | $5 \times 10^{-7}$ |
| 64 | $5 \times 10^{-4}$ | $3 \times 10^{-8}$ |
| 128 | $1 \times 10^{-4}$ | $2 \times 10^{-9}$ |
| 256 | $3 \times 10^{-5}$ | $1 \times 10^{-10}$ |
| 512 | $8 \times 10^{-6}$ | $8 \times 10^{-12}$ |

**Answer.** $|E_T(8)| \approx 3 \times 10^{-2}$ and $|E_S(8)| \approx 1 \times 10^{-4}$.

**Checking the first row.** From Example #1, $|E_T(2)| \approx 0.52375 \approx 5 \times 10^{-1}$. For Simpson with $n = 2$ and $\Delta x = 1$:

$$
S_2 = \left(e^0 + 4e^1 + e^2\right)\frac{1}{3} \approx \frac{1 + 10.87313 + 7.38906}{3} = \frac{19.26219}{3} \approx 6.42073,
$$

so $|E_S(2)| \approx |6.38906 - 6.42073| \approx 0.0317 \approx 3 \times 10^{-2}$.

**Reading the table.** Each time $n$ doubles, $\Delta x$ is cut in half.

- $|E_T|$ shrinks by about a factor of 4 each time. This matches the $(\Delta x)^2$ in Theorem #1, since $(1/2)^2 = 1/4$.
- $|E_S|$ shrinks by about a factor of 16 each time. This matches the $(\Delta x)^4$ in Theorem #2, since $(1/2)^4 = 1/16$.

## Common mistakes and tips

- **Midpoints are new points.** For $M_n$, evaluate $f$ at $\overline{x}_k = \frac{x_{k-1} + x_k}{2}$ instead of at the partition points. On $[1,3]$ with $n = 4$, use $5/4, 7/4, 9/4, 11/4$ and not $1, 3/2, 2, 5/2$.
- **Use the right multiplier.** Rectangle sums use $\Delta x$, the Trapezoidal Rule uses $\frac{\Delta x}{2}$, and Simpson's Rule uses $\frac{\Delta x}{3}$.
- **Get the weights right.** Trapezoidal: $1, 2, 2, \dots, 2, 1$. Simpson: $1, 4, 2, 4, \dots, 2, 4, 1$. In Simpson's Rule the weight next to each endpoint is always 4.
- **Simpson's Rule needs $n$ even.** It works on pairs of subintervals, so $S_5$ does not exist.
- **$T_n$ is not a Riemann sum,** even though $T_n = \frac{L_n + R_n}{2}$. It adds areas of trapezoids instead of rectangles.
- **Error sign convention.** $E = \text{exact} - \text{approximation}$, so a negative error means an overestimate.
- **The over/underestimate rules assume $f \ge 0$** on $[a,b]$. Left overestimates when $f$ is decreasing, right overestimates when $f$ is increasing, and trapezoidal overestimates when $f$ is concave up.
- **$K$ must bound the absolute value over the whole interval.** Find where $|f''|$ or $|f^{(4)}|$ is largest on $[a,b]$. For a positive, increasing derivative, that happens at the right endpoint.
- **Use the right derivative.** The Midpoint and Trapezoidal bounds use $f''$. The Simpson bound uses $f^{(4)}$.
- **An error bound is a ceiling on the error.** In Example #2 the bound on $|E_T(10)|$ is about $0.05$, while the actual error is about $0.02$.
- **Simpson's Rule is exact for polynomials of degree 3 or less.** If $f^{(4)} = 0$, then $K = 0$ and $E_S(n) = 0$ for every even $n$.
- **Accuracy grows quickly with $n$.** Doubling $n$ cuts the Trapezoidal error by about 4 and the Simpson error by about 16.

## Formula sheet

**Partition**

$$
\Delta x = \frac{b-a}{n}, \qquad x_k = a + k\Delta x, \qquad \overline{x}_k = \frac{x_{k-1} + x_k}{2}
$$

**Left, right, and midpoint sums**

$$
L_n = \big(f(x_0) + f(x_1) + \dots + f(x_{n-1})\big)\Delta x
$$

$$
R_n = \big(f(x_1) + \dots + f(x_n)\big)\Delta x
$$

$$
M_n = \big(f(\overline{x}_1) + \dots + f(\overline{x}_n)\big)\Delta x
$$

$$
L_n - R_n = \big(f(a) - f(b)\big)\Delta x
$$

**Trapezoidal Rule**

$$
T_n = \big(f(x_0) + 2f(x_1) + 2f(x_2) + \dots + 2f(x_{n-1}) + f(x_n)\big)\frac{\Delta x}{2} = \frac{L_n + R_n}{2}
$$

**Simpson's Rule ($n$ even)**

$$
S_n = \big(f(x_0) + 4f(x_1) + 2f(x_2) + 4f(x_3) + \dots + 2f(x_{n-2}) + 4f(x_{n-1}) + f(x_n)\big)\frac{\Delta x}{3}
$$

**Error terms**

$$
E_M(n) = \int_a^b f\,dx - M_n, \qquad E_T(n) = \int_a^b f\,dx - T_n, \qquad E_S(n) = \int_a^b f\,dx - S_n
$$

**Error bounds**

$$
|E_M(n)| \le \frac{K(b-a)^3}{24n^2} = \frac{K(b-a)}{24}(\Delta x)^2, \qquad |f''(x)| \le K \text{ on } [a,b]
$$

$$
|E_T(n)| \le \frac{K(b-a)^3}{12n^2} = \frac{K(b-a)}{12}(\Delta x)^2, \qquad |f''(x)| \le K \text{ on } [a,b]
$$

$$
|E_S(n)| \le \frac{K(b-a)^5}{180n^4} = \frac{K(b-a)}{180}(\Delta x)^4, \qquad |f^{(4)}(x)| \le K \text{ on } [a,b]
$$

**Exactness**

- Left/right sums: exact when $f$ is constant over the partition.
- Trapezoidal Rule: exact when $f$ is piecewise linear over the partition.
- Simpson's Rule: exact for polynomials of degree 3 or less.
