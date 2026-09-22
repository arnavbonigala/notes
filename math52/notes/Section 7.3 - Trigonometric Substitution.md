# Section 7.3 - Trigonometric Substitution

## Summary

Trigonometric substitution handles integrals that contain the expressions $a^2 - x^2$, $a^2 + x^2$, or $x^2 - a^2$, usually inside a square root or a denominator. You replace $x$ with $a\sin(t)$, $a\tan(t)$, or $a\sec(t)$. A Pythagorean identity then turns the sum or difference of squares into a single squared trig function, so the square root goes away. You integrate in $t$, then use a reference triangle to rewrite the answer in terms of $x$. If the quadratic is not already a sum or difference of squares, like $x^2 - 2x + 10$, complete the square first, shift with $u = x - k$, and then apply a trig substitution.

## Definitions and theorems

**Pythagorean identities used in this section**

$$
\cos^2(t) + \sin^2(t) = 1
$$

$$
1 + \tan^2(t) = \sec^2(t)
$$

**Double angle identities used in this section**

$$
\cos^2(t) = \frac{1}{2}\big(1 + \cos(2t)\big) = \frac{1 + \cos(2t)}{2}
$$

$$
\sin(2t) = 2\cos(t)\sin(t)
$$

**Right triangle definitions** (for an acute angle $t$)

$$
\cos(t) = \frac{\text{adjacent}}{\text{hypotenuse}}, \qquad \sin(t) = \frac{\text{opposite}}{\text{hypotenuse}}
$$

**Square root of a square**

$$
\sqrt{\cos^2(t)} = |\cos(t)|
$$

This equals $\cos(t)$ only when $\cos(t) \ge 0$, which holds for $t \in [-\pi/2, \pi/2]$.

**Integral of secant**

$$
\int \sec(x)\,dx = \ln|\sec(x) + \tan(x)| + C
$$

**Summary of the substitutions (Remark from the slides)**

1. $a^2 - x^2$: use $x = a\sin(t)$ and $\cos^2(t) + \sin^2(t) = 1$
2. $a^2 + x^2$: use $x = a\tan(t)$ and $1 + \tan^2(t) = \sec^2(t)$
3. $x^2 - a^2$: use $x = a\sec(t)$ and $1 + \tan^2(t) = \sec^2(t)$

## Methods

### Method 1: Choosing and carrying out a trig substitution

**When to use:** the integrand contains $a^2 - x^2$, $a^2 + x^2$, or $x^2 - a^2$, especially as $\sqrt{\cdots}$ or in a denominator.

1. **Identify the form and $a$.** For example $\sqrt{36 - x^2}$ is $a^2 - x^2$ with $a = 6$. $16 + x^2$ is $a^2 + x^2$ with $a = 4$. $x^2 - 49$ is $x^2 - a^2$ with $a = 7$.
2. **Pick the substitution and its $t$ interval.**

   | Form | Substitution | $dx$ | Interval for $t$ | Simplifies to |
   |---|---|---|---|---|
   | $a^2 - x^2$ | $x = a\sin(t)$ | $a\cos(t)\,dt$ | $[-\pi/2, \pi/2]$ | $a^2\cos^2(t)$ |
   | $a^2 + x^2$ | $x = a\tan(t)$ | $a\sec^2(t)\,dt$ | $(-\pi/2, \pi/2)$ | $a^2\sec^2(t)$ |
   | $x^2 - a^2$ | $x = a\sec(t)$ | $a\sec(t)\tan(t)\,dt$ | $(0, \pi/2)$ when $x > a$ | $a^2\tan^2(t)$ |

3. **Simplify the square root.** Use the identity, then drop the absolute value because of how you chose the interval for $t$. For example, $\sqrt{a^2 - x^2} = a|\cos(t)| = a\cos(t)$ because $\cos(t) \ge 0$ on $[-\pi/2, \pi/2]$.
4. **Substitute everything**, including $dx$, and integrate in $t$. You will often need $\cos^2(t) = \frac{1}{2}(1 + \cos(2t))$ or $\int \sec(t)\,dt$.
5. **Convert back to $x$.**
   - Solve the substitution for $t$: $t = \arcsin(x/a)$, $t = \arctan(x/a)$, and so on.
   - For any other trig function of $t$, draw a **reference triangle** from the substitution and read off the ratio you need.
   - Rewrite $\sin(2t)$ as $2\sin(t)\cos(t)$ first so that only functions of $t$ you can read from the triangle remain.
6. **Simplify.** Constants such as $-\ln(4)$ can be absorbed into the constant of integration.

### Method 2: Building a reference triangle

**When to use:** you know one trig ratio of $t$ in terms of $x$ and need another one.

1. Write the known ratio as a side ratio. For example $\sin(t) = \frac{x}{a}$ means opposite $= x$ and hypotenuse $= a$.
2. Use the Pythagorean theorem to get the third side.
3. Read the ratio you need off the triangle.

Triangles from the slides:

| Substitution | Opposite | Adjacent | Hypotenuse |
|---|---|---|---|
| $x = a\sin(t)$ | $x$ | $\sqrt{a^2 - x^2}$ | $a$ |
| $x = a\tan(t)$ | $x$ | $a$ | $\sqrt{a^2 + x^2}$ |
| $x = a\sec(t)$ | $\sqrt{x^2 - a^2}$ | $a$ | $x$ |

### Method 3: Complete the square, then substitute

**When to use:** the integrand has a quadratic $x^2 + bx + c$ that is not already a pure sum or difference of squares.

1. Write $x^2 + bx + c = (x - k)^2 + h$ by completing the square.
2. Let $u = x - k$, so $du = dx$.
3. The integral now has $u^2 + h$ in it. Apply Method 1 to $u$.
4. At the end, replace $u$ with $x - k$.

## Worked examples

### Example 1: Reading one trig ratio from another

**Problem.** If $\tan(t) = \dfrac{\sqrt{9 - x^2}}{x}$ for $x \in (0, 3)$, what is $\sin(t)$ in terms of $x$?

**Step 1: Draw the triangle.** Take $t$ to be the acute angle in a right triangle with

- adjacent side $x$
- opposite side $\sqrt{9 - x^2}$

so that $\tan(t) = \frac{\text{opposite}}{\text{adjacent}} = \frac{\sqrt{9-x^2}}{x}$ as required.

**Step 2: Find the hypotenuse.**

$$
\text{hypotenuse}^2 = x^2 + \left(\sqrt{9 - x^2}\right)^2 = x^2 + 9 - x^2 = 9
$$

so the hypotenuse is $3$.

**Step 3: Read off the ratios.** Using $\sin(t) = \frac{\text{opposite}}{\text{hypotenuse}}$ and $\cos(t) = \frac{\text{adjacent}}{\text{hypotenuse}}$:

$$
\sin(t) = \frac{\sqrt{9 - x^2}}{3}, \qquad \cos(t) = \frac{x}{3}, \qquad x \in (0, 3)
$$

**Step 4: The angle itself.** The same triangle describes $t$ in three equivalent ways:

$$
t = \arccos\left(\frac{x}{3}\right) = \arctan\left(\frac{\sqrt{9 - x^2}}{x}\right) = \arcsin\left(\frac{\sqrt{9 - x^2}}{3}\right)
$$

**Answer.** $\sin(t) = \dfrac{\sqrt{9 - x^2}}{3}$.

---

### Example 2: $a^2 - x^2$ with $a = 1$

**Problem.** Find $\displaystyle\int \sqrt{1 - x^2}\,dx$.

**Why trig substitution:** $y = \sqrt{1 - x^2}$ is a semicircle. Trig functions fit naturally because the curve is easy to describe using a central angle.

**Step 1: Substitute.** Let

$$
x = \sin(t), \qquad dx = \cos(t)\,dt, \qquad t \in [-\pi/2, \pi/2]
$$

**Step 2: Simplify the root.**

$$
(1^2 - x^2)^{1/2} = (1 - \sin^2(t))^{1/2} = (\cos^2(t))^{1/2} = |\cos(t)| = \cos(t)
$$

This uses $\cos^2(t) + \sin^2(t) = 1$, and $\cos(t) \ge 0$ for $t \in [-\pi/2, \pi/2]$.

**Step 3: Rewrite the integral.**

$$
\int (1 - x^2)^{1/2}\,dx = \int \cos(t)\cdot\cos(t)\,dt = \int \cos^2(t)\,dt
$$

**Step 4: Integrate using the half angle identity** $\cos^2(t) = \frac{1}{2}(1 + \cos(2t))$:

$$
\int (1 - x^2)^{1/2}\,dx = \frac{1}{2}\int \big(1 + \cos(2t)\big)\,dt = \frac{1}{2}t + \frac{1}{4}\sin(2t) + C
$$

The $\frac{1}{4}$ comes from $\int \cos(2t)\,dt = \frac{1}{2}\sin(2t)$, multiplied by the outside $\frac{1}{2}$.

**Step 5: Prepare to convert back.** From $x = \sin(t)$ we get $t = \arcsin(x)$. Using $\sin(2t) = 2\cos(t)\sin(t)$:

$$
\frac{1}{4}\sin(2t) = \frac{1}{4}\cdot 2\cos(t)\sin(t) = \frac{1}{2}\cos(t)\sin(t)
$$

so

$$
\int (1 - x^2)^{1/2}\,dx = \frac{1}{2}t + \frac{1}{2}\cos(t)\sin(t) + C
$$

**Step 6: Reference triangle for $\cos(t)$.** $\sin(t) = x = \frac{x}{1}$, so opposite $= x$ and hypotenuse $= 1$. The adjacent side is $\sqrt{1 - x^2}$. So

$$
\cos(t) = \sqrt{1 - x^2}
$$

**Step 7: Substitute back** ($t = \arcsin(x)$, $\sin(t) = x$, $\cos(t) = \sqrt{1-x^2}$):

$$
\int \sqrt{1 - x^2}\,dx = \frac{1}{2}\arcsin(x) + \frac{1}{2}x\sqrt{1 - x^2} + C
$$

---

### Example 3: $a^2 - x^2$ with $a = 6$

**Problem.** Find $\displaystyle\int \sqrt{36 - x^2}\,dx$.

**Step 1: Substitute.** $36 = 6^2$, so let

$$
x = 6\sin(t), \qquad dx = 6\cos(t)\,dt, \qquad t \in [-\pi/2, \pi/2]
$$

**Step 2: Simplify the root.**

$$
\begin{aligned}
(6^2 - x^2)^{1/2} &= \big(6^2 - (6\sin(t))^2\big)^{1/2} \\
&= \big(36(1 - \sin^2(t))\big)^{1/2} = 6(1 - \sin^2(t))^{1/2} \\
&= 6(\cos^2(t))^{1/2} \\
&= 6|\cos(t)| \\
&= 6\cos(t)
\end{aligned}
$$

This uses $\cos^2(t) + \sin^2(t) = 1$, and $\cos(t) \ge 0$ on $[-\pi/2, \pi/2]$.

**Step 3: Rewrite and integrate.**

$$
\begin{aligned}
\int (6^2 - x^2)^{1/2}\,dx &= \int 6\cos(t)\cdot 6\cos(t)\,dt \\
&= \int 36\cos^2(t)\,dt \\
&= 36\cdot\frac{1}{2}\int \big(1 + \cos(2t)\big)\,dt \\
&= 18t + \frac{18}{2}\sin(2t) + C
\end{aligned}
$$

This uses $\cos^2(t) = \frac{1 + \cos(2t)}{2}$. The $\frac{18}{2}$ comes from $18\int\cos(2t)\,dt = 18\cdot\frac{1}{2}\sin(2t)$.

**Step 4: Prepare to convert back.**

$$
x = 6\sin(t) \implies \frac{x}{6} = \sin(t) \implies t = \arcsin\left(\frac{x}{6}\right)
$$

With $\sin(2t) = 2\cos(t)\sin(t)$, the term $\frac{18}{2}\sin(2t) = 9\cdot 2\cos(t)\sin(t) = 18\cos(t)\sin(t)$, so

$$
\int (6^2 - x^2)^{1/2}\,dx = 18t + 18\cos(t)\sin(t) + C
$$

**Step 5: Reference triangle.** $\sin(t) = \frac{x}{6}$ gives opposite $= x$ and hypotenuse $= 6$. The adjacent side is $\sqrt{36 - x^2}$, so

$$
\cos(t) = \frac{\sqrt{36 - x^2}}{6}
$$

**Step 6: Substitute back.**

$$
\begin{aligned}
\int (6^2 - x^2)^{1/2}\,dx &= 18\arcsin\left(\frac{x}{6}\right) + 18\cdot\frac{\sqrt{36 - x^2}}{6}\cdot\frac{x}{6} + C \\
&= 18\arcsin\left(\frac{x}{6}\right) + \frac{x}{2}\sqrt{36 - x^2} + C
\end{aligned}
$$

since $\frac{18}{36} = \frac{1}{2}$.

**Step 7: Equivalent forms.** With a little algebra, factor $36$ out of the root:

$$
\frac{x}{2}\sqrt{36 - x^2} = \frac{x}{2}\sqrt{36\left(1 - \frac{x^2}{36}\right)} = \frac{x}{2}\cdot 6\sqrt{1 - \frac{x^2}{36}} = 3x\sqrt{1 - \frac{x^2}{36}}
$$

So watch out if your solution does not "look" like the "right" one. It may be the same answer in a different form.

---

### Example 4: $a^2 + x^2$ with $a = 9$

**Problem.** Find $\displaystyle\int \frac{1}{9^2 + x^2}\,dx$.

**Step 1: Substitute.** Let

$$
x = 9\tan(t), \qquad dx = 9\sec^2(t)\,dt, \qquad t \in (-\pi/2, \pi/2)
$$

**Step 2: Simplify the denominator.** Using $1 + \tan^2(t) = \sec^2(t)$:

$$
9^2 + (9\tan(t))^2 = 9^2\big(1 + \tan^2(t)\big) = 9^2\sec^2(t)
$$

**Step 3: Rewrite and integrate.**

$$
\begin{aligned}
\int \frac{1}{9^2 + x^2}\,dx &= \int \frac{1}{9^2\sec^2(t)}\cdot 9\sec^2(t)\,dt \\
&= \int \frac{1}{9}\,dt \\
&= \frac{1}{9}t + C
\end{aligned}
$$

The $\sec^2(t)$ cancels and $\frac{9}{81} = \frac{1}{9}$.

**Step 4: Convert back.**

$$
x = 9\tan(t) \implies \frac{x}{9} = \tan(t) \implies t = \arctan\left(\frac{x}{9}\right)
$$

**Answer.**

$$
\int \frac{1}{9^2 + x^2}\,dx = \frac{1}{9}\arctan\left(\frac{x}{9}\right) + C
$$

---

### Example 5: $a^2 + x^2$ under a root, $a = 4$

**Problem.** Find $\displaystyle\int \frac{1}{\sqrt{16 + x^2}}\,dx$.

**Step 1: Substitute.** Let

$$
x = 4\tan(t), \qquad dx = 4\sec^2(t)\,dt, \qquad t \in (-\pi/2, \pi/2)
$$

**Step 2: Simplify.**

$$
4^2 + (4\tan(t))^2 = 4^2\big(1 + \tan^2(t)\big) = 4^2\sec^2(t)
$$

so $(4^2 + x^2)^{1/2} = 4\sec(t)$. (On $(-\pi/2, \pi/2)$, $\cos(t) > 0$, so $\sec(t) > 0$ and no absolute value is needed.)

**Step 3: Rewrite.**

$$
\int \frac{1}{(4^2 + x^2)^{1/2}}\,dx = \int \frac{1}{4\sec(t)}\cdot 4\sec^2(t)\,dt = \int \sec(t)\,dt
$$

**Step 4: Derive $\int \sec(t)\,dt$.** Multiply by "1" in the form $\frac{\tan(t) + \sec(t)}{\tan(t) + \sec(t)}$ and let

$$
\begin{aligned}
u &= \tan(t) + \sec(t) \\
du &= \big(\sec^2(t) + \sec(t)\tan(t)\big)\,dt = \big(\tan(t) + \sec(t)\big)\sec(t)\,dt
\end{aligned}
$$

Then

$$
\int \sec(t)\,dt = \int \sec(t)\left(\frac{\tan(t) + \sec(t)}{\tan(t) + \sec(t)}\right)dt = \int \frac{1}{u}\,du = \ln|\tan(t) + \sec(t)| + C
$$

The numerator $\sec(t)(\tan(t) + \sec(t))\,dt$ is exactly $du$, and the denominator is $u$.

So

$$
\int \frac{1}{(16 + x^2)^{1/2}}\,dx = \ln|\tan(t) + \sec(t)| + C
$$

**Step 5: Reference triangle.** $x = 4\tan(t)$ gives $\tan(t) = \frac{x}{4}$: opposite $= x$, adjacent $= 4$, hypotenuse $= \sqrt{16 + x^2}$. So

$$
\sec(t) = \frac{\text{hypotenuse}}{\text{adjacent}} = \frac{\sqrt{16 + x^2}}{4}
$$

**Step 6: Substitute back.**

$$
\int \frac{1}{(16 + x^2)^{1/2}}\,dx = \ln\left|\frac{x}{4} + \frac{\sqrt{16 + x^2}}{4}\right| + C
$$

**Step 7: Simplify further.** Combine the fraction and use $\ln\frac{A}{B} = \ln A - \ln B$:

$$
\begin{aligned}
\int \frac{1}{(16 + x^2)^{1/2}}\,dx &= \ln\left|\frac{x + \sqrt{16 + x^2}}{4}\right| + C \\
&= \ln\left|x + \sqrt{16 + x^2}\right| - \ln(4) + C \\
&= \ln\left|x + \sqrt{16 + x^2}\right| + C'
\end{aligned}
$$

where $C' = C - \ln(4)$ is just another constant.

---

### Example 6: $x^2 - a^2$ with $a = 7$

**Problem.** Find $\displaystyle\int \frac{1}{\sqrt{x^2 - 49}}\,dx$ for $x > 7$.

**Step 1: Substitute.** Let

$$
x = 7\sec(t), \qquad dx = 7\sec(t)\tan(t)\,dt, \qquad t \in (0, \pi/2)
$$

**Step 2: Simplify the root.** Using $1 + \tan^2(t) = \sec^2(t)$, so $\sec^2(t) - 1 = \tan^2(t)$:

$$
7^2\sec^2(t) - 7^2 = 7^2\big(\sec^2(t) - 1\big) = 7^2\tan^2(t)
$$

and

$$
(x^2 - 49)^{1/2} = 7|\tan(t)| = 7\tan(t)
$$

because of our choice of $t$: $\tan(t) > 0$ on $(0, \pi/2)$.

**Step 3: Rewrite and integrate.**

$$
\begin{aligned}
\int \frac{1}{(x^2 - 49)^{1/2}}\,dx &= \int \frac{1}{7\tan(t)}\cdot 7\sec(t)\tan(t)\,dt \\
&= \int \sec(t)\,dt \\
&= \ln|\sec(t) + \tan(t)| + C
\end{aligned}
$$

**Step 4: Reference triangle.** $x = 7\sec(t)$ means $\cos(t) = \frac{7}{x}$: adjacent $= 7$, hypotenuse $= x$, opposite $= \sqrt{x^2 - 7^2}$. So

$$
\sec(t) = \frac{x}{7}, \qquad \tan(t) = \frac{\sqrt{x^2 - 7^2}}{7}
$$

**Step 5: Substitute back and simplify.**

$$
\begin{aligned}
\int \frac{1}{(x^2 - 49)^{1/2}}\,dx &= \ln\left|\frac{x}{7} + \frac{\sqrt{x^2 - 7^2}}{7}\right| + C \\
&= \ln\left|\frac{x + \sqrt{x^2 - 7^2}}{7}\right| + C \\
&= \ln\left|x + \sqrt{x^2 - 7^2}\right| + C
\end{aligned}
$$

In the last step the $-\ln(7)$ is absorbed into the constant $C$, the same way as in Example 5.

**About the restriction $x > 7$.** The integrand's domain is $(-\infty, -7) \cup (7, \infty)$. Choosing $x > 7$ tells us the signs of the sides in the reference triangle. If instead $x < -7$, we can choose $t$ in $(\pi, 3\pi/2)$ instead of $(0, \pi/2)$.

---

### Example 7: Completing the square

**Problem.** Express $x^2 - 2x + 10$ as $u^2 + h$ where $u = x - k$.

**Step 1: Match the $x$ terms.** The $-2x$ term comes from $(x - 1)^2$:

$$
(x - 1)^2 = x^2 - 2x + 1
$$

**Step 2: Fix the constant.** We need $10$, and $(x-1)^2$ gives $1$, so add $9$:

$$
(x - 1)^2 + 9 = x^2 - 2x + 1 + 9 = x^2 - 2x + 10
$$

**Answer.** $x^2 - 2x + 10 = u^2 + 9$ with $u = x - 1$, so $k = 1$ and $h = 9$.

---

### Example 8: Complete the square, then substitute

**Problem.** Find $\displaystyle\int \frac{1}{x^2 - 2x + 10}\,dx$.

**Step 1: Complete the square** (from Example 7): $x^2 - 2x + 10 = (x - 1)^2 + 9$.

**Step 2: Shift.** Let $u = x - 1$, $du = dx$. Then

$$
\int \frac{1}{x^2 - 2x + 10}\,dx = \int \frac{1}{(x - 1)^2 + 9}\,dx = \int \frac{1}{u^2 + 9}\,du
$$

**Step 3: Trig substitution.** This is $a^2 + u^2$ with $a = 3$. Let

$$
u = 3\tan(t), \qquad du = 3\sec^2(t)\,dt
$$

Then

$$
u^2 + 9 = 9\tan^2(t) + 9 = 9\big(\tan^2(t) + 1\big) = 9\sec^2(t)
$$

**Step 4: Integrate.**

$$
\begin{aligned}
\int \frac{1}{u^2 + 9}\,du &= \int \frac{1}{9\sec^2(t)}\cdot 3\sec^2(t)\,dt \\
&= \frac{1}{3}\int dt \\
&= \frac{1}{3}t + C
\end{aligned}
$$

**Step 5: Convert back to $u$, then to $x$.**

$$
u = 3\tan(t) \implies \frac{u}{3} = \tan(t) \implies t = \arctan\left(\frac{u}{3}\right)
$$

Since $u = x - 1$:

$$
\int \frac{1}{x^2 - 2x + 10}\,dx = \frac{1}{3}\arctan\left(\frac{x - 1}{3}\right) + C
$$

## Common mistakes and tips

- **Match the form to the substitution.** $a^2 - x^2 \to a\sin(t)$, $a^2 + x^2 \to a\tan(t)$, $x^2 - a^2 \to a\sec(t)$. Mixing these up means the identity won't collapse the expression.
- **Don't forget $dx$.** Substitute $dx$ too: $a\cos(t)\,dt$, $a\sec^2(t)\,dt$, or $a\sec(t)\tan(t)\,dt$. Often this factor is what cancels against the simplified root.
- **$\sqrt{\cos^2(t)} = |\cos(t)|$, not automatically $\cos(t)$.** You can drop the absolute value only because of the interval you picked for $t$. State the interval.
- **For $x^2 - a^2$, the sign of $x$ matters.** For $x > a$ use $t \in (0, \pi/2)$. For $x < -a$ the slides choose $t \in (\pi, 3\pi/2)$.
- **Rewrite $\sin(2t)$ before converting back.** Use $\sin(2t) = 2\sin(t)\cos(t)$ so you can read both factors from the reference triangle.
- **Always convert back to $x$.** An answer in $t$ is not finished. Get $t$ from the inverse trig function and every other ratio from the triangle.
- **Answers can look different and still be correct.** $\frac{x}{2}\sqrt{36 - x^2} = 3x\sqrt{1 - \frac{x^2}{36}}$. Also, $\ln\left|\frac{x + \sqrt{16+x^2}}{4}\right| + C$ and $\ln|x + \sqrt{16+x^2}| + C'$ differ only by a constant.
- **Constants get absorbed.** Terms like $-\ln(4)$ or $-\ln(7)$ can be folded into $C$.
- **Messy quadratic? Complete the square first.** Write $x^2 + bx + c = (x - k)^2 + h$, let $u = x - k$, then use a trig substitution.
- **Memorize $\int \sec(x)\,dx$.** It shows up in both the $a^2 + x^2$ and $x^2 - a^2$ cases.

## Formula sheet

**Substitutions**

| Expression | Substitution | $dx$ | $t$ interval | Identity | Result |
|---|---|---|---|---|---|
| $a^2 - x^2$ | $x = a\sin(t)$ | $a\cos(t)\,dt$ | $[-\pi/2, \pi/2]$ | $\cos^2 t + \sin^2 t = 1$ | $\sqrt{a^2 - x^2} = a\cos(t)$ |
| $a^2 + x^2$ | $x = a\tan(t)$ | $a\sec^2(t)\,dt$ | $(-\pi/2, \pi/2)$ | $1 + \tan^2 t = \sec^2 t$ | $a^2 + x^2 = a^2\sec^2(t)$ |
| $x^2 - a^2$ | $x = a\sec(t)$ | $a\sec(t)\tan(t)\,dt$ | $(0, \pi/2)$ for $x > a$ | $1 + \tan^2 t = \sec^2 t$ | $\sqrt{x^2 - a^2} = a\tan(t)$ |

**Identities**

$$
\cos^2(t) + \sin^2(t) = 1 \qquad 1 + \tan^2(t) = \sec^2(t)
$$

$$
\cos^2(t) = \frac{1 + \cos(2t)}{2} \qquad \sin(2t) = 2\sin(t)\cos(t)
$$

**Integrals**

$$
\int \sec(x)\,dx = \ln|\sec(x) + \tan(x)| + C
$$

$$
\int \cos^2(t)\,dt = \frac{1}{2}t + \frac{1}{4}\sin(2t) + C
$$

**Results from the examples**

$$
\int \sqrt{1 - x^2}\,dx = \frac{1}{2}\arcsin(x) + \frac{1}{2}x\sqrt{1 - x^2} + C
$$

$$
\int \sqrt{36 - x^2}\,dx = 18\arcsin\left(\frac{x}{6}\right) + \frac{x}{2}\sqrt{36 - x^2} + C
$$

$$
\int \frac{1}{9^2 + x^2}\,dx = \frac{1}{9}\arctan\left(\frac{x}{9}\right) + C
$$

$$
\int \frac{1}{\sqrt{16 + x^2}}\,dx = \ln\left|x + \sqrt{16 + x^2}\right| + C
$$

$$
\int \frac{1}{\sqrt{x^2 - 49}}\,dx = \ln\left|x + \sqrt{x^2 - 49}\right| + C \quad (x > 7)
$$

$$
\int \frac{1}{x^2 - 2x + 10}\,dx = \frac{1}{3}\arctan\left(\frac{x - 1}{3}\right) + C
$$

**Completing the square**

$$
x^2 - 2x + 10 = (x - 1)^2 + 9
$$
