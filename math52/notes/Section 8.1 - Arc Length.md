# Section 8.1 - Arc Length

## Summary

This section answers one question: how long is the curve $y = f(x)$ between $x = a$ and $x = b$? You approximate the curve with straight line segments, use the Pythagorean theorem and the Mean Value Theorem on each segment, and let the number of segments go to infinity. The result is an integral, $L = \int_a^b \sqrt{1 + (f'(x))^2}\,dx$. Use it whenever a problem asks for the length of a graph or the distance traveled along a path given as $y = f(x)$, and $f'$ is continuous. Exact answers are rare, because few functions make $\sqrt{1 + (f'(x))^2}$ easy to integrate. The ones that work usually simplify to a power of $(1 + x)$ or to a perfect square.

## Definitions and theorems

**The problem.** We want to measure the length of the curve

$$
\{(x, f(x)) \mid x \in [a, b]\}
$$

where $f$ is a "smooth" function on $[a, b]$.

**Mean Value Theorem.** If $f$ is a differentiable function on the interval $[a, b]$, then there exists a number $c$ between $a$ and $b$ such that

$$
f'(c) = \frac{f(b) - f(a)}{b - a}
$$

or, equivalently,

$$
f(b) - f(a) = f'(c)(b - a).
$$

Interpretation from the slides: the secant line through $(a, f(a))$ and $(b, f(b))$ has slope $\frac{f(b) - f(a)}{b - a}$, and it is parallel to some tangent line of $f$. That tangent line is $y = f'(c)(x - c) + f(c)$. The slope of the secant line is $f$'s average rate of change over $[a, b]$.

**Theorem #1 (Arc length).** Let $f'$ be continuous on $[a, b]$. Then the length of the curve from $(a, f(a))$ to $(b, f(b))$ is given by

$$
L = \int_a^b \sqrt{1 + (f'(x))^2}\,dx.
$$

We call $L$ the **arc length** of $f$ over the interval $[a, b]$.

**Definition #1 (Arc length function).** Let $f'$ be continuous on $[a, b]$. Then

$$
s(x) = \int_a^x \sqrt{1 + (f'(t))^2}\,dt
$$

for $x \in [a, b]$ is the **arc length function** of $f$.

**Properties of $s(x)$.** By the Fundamental Theorem of Calculus,

$$
s'(x) = \sqrt{1 + (f'(x))^2}.
$$

Since $(f'(x))^2 \ge 0$, we have $1 + (f'(x))^2 \ge 1$, so

$$
s'(x) \ge 1.
$$

The only time $s'(x) = 1$ is when $f'(x) = 0$. The slides list two ways to read this:

1. $f$ is constant along an interval, so the curve is flat there.
2. $f'$, the slope of the curve, is 0.

### Derivation of the formula (from the slides)

1. Assume the curve is piecewise linear, so it can be approximated by straight segments.
2. Partition $[a, b]$ with step size $\Delta x$:
   $$
   x_k = a + k\,\Delta x, \qquad I_k = [x_{k-1}, x_k], \qquad k = 0, 1, \ldots, n.
   $$
3. Estimate the length of the curve over $I_k$ by the length $L_k$ of the segment joining $(x_{k-1}, f(x_{k-1}))$ and $(x_k, f(x_k))$.
4. Take $k = 1$. The segment $L_1$ is the hypotenuse of a right triangle with
   - width $\Delta x$
   - height $\Delta y_1 = f(x_1) - f(x_0)$.
5. By the Pythagorean theorem:
   $$
   \begin{aligned}
   L_1^2 &= (\Delta x)^2 + (\Delta y_1)^2 \\
   &= (\Delta x)^2\left(1 + \left(\frac{\Delta y_1}{\Delta x}\right)^2\right) \\
   &= (\Delta x)^2\left(1 + \left(\frac{f(x_1) - f(x_0)}{x_1 - x_0}\right)^2\right).
   \end{aligned}
   $$
   The second line factors out $(\Delta x)^2$. The third line uses $\Delta x = x_1 - x_0$.
6. By the MVT there is some $\bar{x}_1 \in [x_0, x_1]$ with
   $$
   \frac{f(x_1) - f(x_0)}{x_1 - x_0} = f'(\bar{x}_1).
   $$
   Substituting and taking the square root (since $\Delta x > 0$):
   $$
   L_1 = \left(1 + (f'(\bar{x}_1))^2\right)^{1/2}\Delta x.
   $$
7. Add up all the approximations:
   $$
   L \approx \sum_{k=1}^n L_k = \sum_{k=1}^n \left(1 + (f'(\bar{x}_k))^2\right)^{1/2}\Delta x.
   $$
8. This is a Riemann sum. Let $n \to \infty$:
   $$
   L = \int_a^b \left(1 + (f'(x))^2\right)^{1/2}dx.
   $$

## Methods

### Method 1: Compute an arc length exactly

**When to use:** you are asked for the length of $y = f(x)$ on $[a, b]$, $f'$ is continuous, and $1 + (f'(x))^2$ simplifies nicely.

1. Compute $f'(x)$.
2. Compute $(f'(x))^2$ and simplify fully.
3. Form $1 + (f'(x))^2$ and look for a simplification:
   - it may be a simple linear expression, like $1 + x$ (Example 1)
   - it may be a perfect square, like $\left(\frac{e^x}{2} + \frac{e^{-x}}{2}\right)^2$ (Example 2)
4. Take the square root. For a perfect square, check that the base is nonnegative on $[a, b]$ so the square root simply removes the square.
5. Integrate from $a$ to $b$ and evaluate.

### Method 2: Set up an arc length integral for an applied problem

**When to use:** a word problem describes a path $y = f(x)$ and asks for the distance traveled along it.

1. Find the interval of $x$ values. Use the physical conditions, such as "hits the ground" meaning $y = 0$.
2. Pick the root that makes physical sense, for example the one where the object moves forward.
3. Compute $\frac{dy}{dx}$ and write $L = \int_a^b \left(1 + \left(\frac{dy}{dx}\right)^2\right)^{1/2}dx$.
4. Evaluate with a known antiderivative (trig substitution) or numerically.

### Method 3: Arc length function

**When to use:** you need the length from a fixed starting point $a$ out to a variable point $x$, or the rate at which arc length grows.

1. Write $s(x) = \int_a^x \sqrt{1 + (f'(t))^2}\,dt$. Use a dummy variable $t$ inside.
2. For the rate of change, apply FTC: $s'(x) = \sqrt{1 + (f'(x))^2}$.

## Worked examples

### Example 1

**Problem.** Find the arc length of $y = \frac{2}{3}x^{3/2}$ over $[0, 3]$.

**Step 1: Derivative.** By the power rule,
$$
f'(x) = \frac{2}{3}\cdot\frac{3}{2}x^{1/2} = x^{1/2}.
$$

**Step 2: Square and add 1.** $(f'(x))^2 = (x^{1/2})^2 = x$ for $x \ge 0$, so
$$
\left(1 + (f'(x))^2\right)^{1/2} = (1 + x)^{1/2}.
$$

**Step 3: Integrate.** An antiderivative of $(1 + x)^{1/2}$ is $\frac{2}{3}(1 + x)^{3/2}$, by the power rule with $u = 1 + x$ and $du = dx$.
$$
\begin{aligned}
L &= \int_0^3 (1 + x)^{1/2}\,dx \\
&= \frac{2}{3}(1 + x)^{3/2}\Big|_{x=0}^{3} \\
&= \frac{2}{3}\left(4^{3/2} - 1^{3/2}\right) \\
&= \frac{2}{3}\left(2^3 - 1\right) \\
&= \frac{2}{3}\cdot 7 \\
&= \frac{14}{3}.
\end{aligned}
$$

Here $4^{3/2} = (4^{1/2})^3 = 2^3 = 8$.

**Answer:** $L = \dfrac{14}{3}$.

### Example 2

**Problem.** Find the arc length of $y = \frac{e^x}{2} + \frac{e^{-x}}{2}$ over $[-\ln(2), \ln(2)]$.

**Step 1: Derivative.** Since $\frac{d}{dx}e^{-x} = -e^{-x}$,
$$
f'(x) = \frac{e^x}{2} - \frac{e^{-x}}{2}.
$$

**Step 2: Square.**
$$
\begin{aligned}
(f'(x))^2 &= \left(\frac{e^x}{2} - \frac{e^{-x}}{2}\right)\left(\frac{e^x}{2} - \frac{e^{-x}}{2}\right) \\
&= \frac{e^{2x}}{4} - 2\,\frac{e^x e^{-x}}{4} + \frac{e^{-2x}}{4} \\
&= \left(\frac{e^x}{2}\right)^2 - \frac{1}{2} + \left(\frac{e^{-x}}{2}\right)^2.
\end{aligned}
$$
The middle term simplifies because $e^x e^{-x} = e^0 = 1$, so $-2\cdot\frac{1}{4} = -\frac{1}{2}$.

**Step 3: Add 1.** Adding 1 changes $-\frac{1}{2}$ into $+\frac{1}{2}$:
$$
\begin{aligned}
1 + (f'(x))^2 &= \left(\frac{e^x}{2}\right)^2 + \frac{1}{2} + \left(\frac{e^{-x}}{2}\right)^2 \\
&= \left(\frac{e^x}{2} + \frac{e^{-x}}{2}\right)^2.
\end{aligned}
$$
This is a perfect square. Expanding $\left(\frac{e^x}{2} + \frac{e^{-x}}{2}\right)^2$ gives the same three terms with the middle term $+2\cdot\frac{1}{4} = +\frac{1}{2}$.

**Step 4: Square root.** Since $\frac{e^x}{2} + \frac{e^{-x}}{2} > 0$ for all $x$,
$$
\left(1 + (f'(x))^2\right)^{1/2} = \frac{e^x}{2} + \frac{e^{-x}}{2}.
$$
The slides call this "how convenient..."

**Step 5: Integrate.** An antiderivative of $\frac{e^x}{2} + \frac{e^{-x}}{2}$ is $\frac{e^x}{2} - \frac{e^{-x}}{2}$.
$$
\int_{-\ln(2)}^{\ln(2)} \frac{e^x}{2} + \frac{e^{-x}}{2}\,dx = \frac{e^x}{2} - \frac{e^{-x}}{2}\Big|_{x=-\ln(2)}^{\ln(2)}.
$$

Evaluate using $e^{\ln 2} = 2$ and $e^{-\ln 2} = \frac{1}{2}$:
- At $x = \ln(2)$: $\frac{1}{2}\left(2 - \frac{1}{2}\right) = \frac{1}{2}\cdot\frac{3}{2} = \frac{3}{4}$.
- At $x = -\ln(2)$: $\frac{1}{2}\left(\frac{1}{2} - 2\right) = \frac{1}{2}\cdot\left(-\frac{3}{2}\right) = -\frac{3}{4}$.

$$
= \frac{1}{2}\left(2 - \frac{1}{2}\right) - \frac{1}{2}\left(\frac{1}{2} - 2\right) = \frac{3}{4} - \left(-\frac{3}{4}\right) = \frac{3}{2}.
$$

**Answer:** $L = \dfrac{3}{2}$.

### Example 3

**Problem.** A hawk flying at 15 m/s at an altitude of 180 m accidentally drops its prey. The path of the falling prey is described by the equation
$$
y = 180 - \frac{x^2}{45}
$$
until it hits the ground, where $y$ is its height above the ground and $x$ is the horizontal distance traveled in meters. Find the integral that measures the distance traveled by the prey from the time it is dropped until the time it hits the ground.

**Step 1: Find where it lands.** The prey hits the ground when $y = 0$:
$$
\begin{aligned}
0 &= 180 - \frac{x^2}{45} \\
x^2 &= 180 \cdot 45 = 8100 \\
x &= \pm\sqrt{8100} = \pm 90.
\end{aligned}
$$

**Step 2: Choose the root.** The prey is moving forward, so take $x = 90$. It is dropped at $x = 0$, so the interval is $[0, 90]$.

**Step 3: Derivative.**
$$
\frac{dy}{dx} = -\frac{2x}{45}, \qquad \left(\frac{dy}{dx}\right)^2 = \frac{4x^2}{45^2}.
$$

**Step 4: Set up the integral.**
$$
L = \int_0^{90}\left(1 + \left(\frac{dy}{dx}\right)^2\right)^{1/2}dx = \int_0^{90}\left(1 + \frac{4x^2}{45^2}\right)^{1/2}dx \approx 209 \text{ m}.
$$

**Step 5: Evaluate.** The slides give this antiderivative, which comes from a trig substitution:
$$
\int (1 + kx^2)^{1/2}\,dx = \frac{1}{2}x\sqrt{1 + kx^2} + \frac{1}{2\sqrt{k}}\log\left(\sqrt{1 + kx^2} + \sqrt{k}\,x\right) + C.
$$
Here $\log$ is the natural log. Use $k = \frac{4}{45^2} = \frac{4}{2025}$, so $\sqrt{k} = \frac{2}{45}$ and $\frac{1}{2\sqrt{k}} = \frac{45}{4}$.

- At $x = 90$: $kx^2 = \frac{4 \cdot 8100}{2025} = 16$, so $\sqrt{1 + kx^2} = \sqrt{17}$ and $\sqrt{k}\,x = \frac{2}{45}\cdot 90 = 4$. The value is
  $$
  \frac{1}{2}(90)\sqrt{17} + \frac{45}{4}\ln\left(\sqrt{17} + 4\right) = 45\sqrt{17} + \frac{45}{4}\ln\left(4 + \sqrt{17}\right).
  $$
- At $x = 0$: $0 + \frac{45}{4}\ln(1) = 0$.

So
$$
L = 45\sqrt{17} + \frac{45}{4}\ln\left(4 + \sqrt{17}\right) \approx 185.5 + 23.6 \approx 209 \text{ m}.
$$

**Answer:** $L = \displaystyle\int_0^{90}\left(1 + \frac{4x^2}{45^2}\right)^{1/2}dx \approx 209$ m.

## Common mistakes and tips

- **Square the derivative, not the function.** The integrand is $\sqrt{1 + (f'(x))^2}$. Do not use $f(x)$ or $1 + f'(x)$.
- **Simplify $(f'(x))^2$ completely before adding 1.** In Example 2 the perfect square only shows up after you notice $e^x e^{-x} = 1$.
- **Look for a perfect square.** If $(f'(x))^2$ looks like $A^2 - \frac{1}{2} + B^2$ with $AB = \frac{1}{4}$, then adding 1 gives $(A + B)^2$.
- **Check the sign before removing a square root.** $\sqrt{u^2} = |u|$. In Example 2 you can drop the absolute value because $\frac{e^x}{2} + \frac{e^{-x}}{2} > 0$.
- **The hypothesis matters.** Theorem #1 needs $f'$ continuous on $[a, b]$.
- **Find the limits from context in word problems.** "Hits the ground" means $y = 0$. Throw out the root that doesn't make physical sense (Example 3 uses $x = 90$, not $-90$).
- **Expect hard integrals.** Most arc length integrals have no nice antiderivative. If the question only asks for the integral, as in Example 3, set it up and stop.
- **Sanity check with $s'(x) \ge 1$.** Arc length over $[a, b]$ is always at least $b - a$, with equality only when the curve is flat ($f' = 0$). In Example 1, $\frac{14}{3} \approx 4.67 \ge 3$. In Example 2, $\frac{3}{2} = 1.5 \ge 2\ln 2 \approx 1.39$.
- **Use a dummy variable in $s(x)$.** Write $\int_a^x \sqrt{1 + (f'(t))^2}\,dt$, not $dx$ with $x$ as the upper limit.

## Formula sheet

**Arc length** ($f'$ continuous on $[a, b]$):
$$
L = \int_a^b \sqrt{1 + (f'(x))^2}\,dx = \int_a^b \left(1 + \left(\frac{dy}{dx}\right)^2\right)^{1/2}dx
$$

**Arc length function:**
$$
s(x) = \int_a^x \sqrt{1 + (f'(t))^2}\,dt, \qquad s'(x) = \sqrt{1 + (f'(x))^2} \ge 1
$$

**Segment length** (from the derivation):
$$
L_k^2 = (\Delta x)^2 + (\Delta y_k)^2, \qquad L_k = \left(1 + (f'(\bar{x}_k))^2\right)^{1/2}\Delta x
$$

**Mean Value Theorem:**
$$
f'(c) = \frac{f(b) - f(a)}{b - a} \quad\Longleftrightarrow\quad f(b) - f(a) = f'(c)(b - a)
$$

**Useful antiderivative** (trig substitution, from Example 3):
$$
\int (1 + kx^2)^{1/2}\,dx = \frac{1}{2}x\sqrt{1 + kx^2} + \frac{1}{2\sqrt{k}}\ln\left(\sqrt{1 + kx^2} + \sqrt{k}\,x\right) + C
$$

**Perfect square identity** (from Example 2):
$$
1 + \left(\frac{e^x}{2} - \frac{e^{-x}}{2}\right)^2 = \left(\frac{e^x}{2} + \frac{e^{-x}}{2}\right)^2
$$
