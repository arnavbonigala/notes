# Section 7.5 - Strategy for Integration

## Summary

This section has no new integration technique. It is practice in choosing the right technique from the ones you already have: algebraic simplification, substitution, trigonometric substitution, integration by parts (including the tabular method), polynomial division, and partial fractions. Every example on the slides starts from the table of basic integration formulas below. The goal is to rewrite the integrand until it matches one of those formulas. Use this section when an integral does not say which method to use, as on an exam. The main skill is spotting the structure of the integrand and recognizing when your first attempt is making things harder so you can switch to something else.

## Definitions and theorems

### Table of Integration Formulas

These are the formulas as given on the slides. Constants of integration are omitted.

1. $\displaystyle \int x^n \, dx = \frac{x^{n+1}}{n+1} \quad (n \neq -1)$
2. $\displaystyle \int \frac{1}{x} \, dx = \ln|x|$
3. $\displaystyle \int e^x \, dx = e^x$
4. $\displaystyle \int b^x \, dx = \frac{b^x}{\ln b}$
5. $\displaystyle \int \sin x \, dx = -\cos x$
6. $\displaystyle \int \cos x \, dx = \sin x$
7. $\displaystyle \int \sec^2 x \, dx = \tan x$
8. $\displaystyle \int \csc^2 x \, dx = -\cot x$
9. $\displaystyle \int \sec x \tan x \, dx = \sec x$
10. $\displaystyle \int \csc x \cot x \, dx = -\csc x$
11. $\displaystyle \int \sec x \, dx = \ln|\sec x + \tan x|$
12. $\displaystyle \int \csc x \, dx = \ln|\csc x - \cot x|$
13. $\displaystyle \int \tan x \, dx = \ln|\sec x|$
14. $\displaystyle \int \cot x \, dx = \ln|\sin x|$
15. $\displaystyle \int \sinh x \, dx = \cosh x$
16. $\displaystyle \int \cosh x \, dx = \sinh x$
17. $\displaystyle \int \frac{dx}{x^2 + a^2} = \frac{1}{a}\tan^{-1}\left(\frac{x}{a}\right)$
18. $\displaystyle \int \frac{dx}{\sqrt{a^2 - x^2}} = \sin^{-1}\left(\frac{x}{a}\right), \quad a > 0$

\*19. $\displaystyle \int \frac{dx}{x^2 - a^2} = \frac{1}{2a}\ln\left|\frac{x-a}{x+a}\right|$

\*20. $\displaystyle \int \frac{dx}{\sqrt{x^2 \pm a^2}} = \ln\left|x + \sqrt{x^2 \pm a^2}\right|$

(Formulas 19 and 20 are starred on the slide.)

## Methods

These are the techniques the worked examples use, in roughly the order you should think of them.

### 1. Simplify the integrand first (Example 1)
**When:** the integrand is a power or product that can be expanded, or it contains trig functions that cancel or reduce with identities.

1. Expand products and powers.
2. Cancel where possible. For example, $\cos x \sec x = 1$.
3. Replace awkward terms with identities, such as $\cos^2 x = \tfrac12 + \tfrac12\cos(2x)$.
4. Integrate term by term using the table.

### 2. u-substitution (Examples 2, 3, 6)
**When:** part of the integrand is, up to a constant, the derivative of another part.

1. Choose $u$ as the "inside" expression, e.g. $u = x^3$ when $x^2\,dx$ appears.
2. Compute $du$ and solve for the piece of the integrand it replaces.
3. Rewrite the whole integral in $u$ and pull constants out.
4. Integrate, then substitute back.

A special case to always check: **is the numerator a constant multiple of the derivative of the denominator?** If it is, $\int \frac{f'(x)}{f(x)}\,dx = \ln|f(x)| + C$ and you don't need partial fractions (Example 3).

You can also substitute more than once. For example, $u = x^3$ followed by $w = u/2$ (Example 2).

### 3. Trigonometric substitution (Example 4)
**When:** the integrand contains $1 + x^2$ (or more generally $a^2 + x^2$) inside a root, especially when the simple substitution $u = 1 + x^2$ leads to something harder.

1. Let $x = \tan t$, $dx = \sec^2 t\,dt$, with $t \in (-\pi/2, \pi/2)$ so that $\sec t > 0$.
2. Use $1 + \tan^2 t = \sec^2 t$, so $\sqrt{1 + \tan^2 t} = \sec t$.
3. Finish the integral in $t$, which may take another substitution or integration by parts.
4. Convert back to $x$ with a right triangle: opposite $= x$, adjacent $= 1$, hypotenuse $= \sqrt{x^2+1}$.

### 4. Integration by parts (Examples 5, 7)
**When:** the integrand is a product such as polynomial times logarithm, or a single function whose derivative is simpler, such as $\ln(1+x^2)$ with $dv = dx$.

$$
\int u\,dv = uv - \int v\,du
$$

1. Choose $u$ to be the factor that gets simpler when you differentiate it (a logarithm is a good choice).
2. Choose $dv$ to be something you can integrate.
3. Check that $\int v\,du$ is easier than the original integral. If it is harder, switch methods (Example 6).

### 5. Tabular integration by parts (Examples 4, 6)
**When:** you have a polynomial $p(u)$ times $e^u$ (or anything you can integrate repeatedly).

1. Make a column of $u$'s: the polynomial and its successive derivatives, down to $0$.
2. Make a column of $dv$'s: $e^u$ and its successive antiderivatives (always $e^u$).
3. Connect each $u$ entry to the next-lower $dv$ entry with a diagonal, alternating signs $+, -, +, -, \dots$
4. Add up the signed diagonal products.

### 6. Polynomial division (Example 5)
**When:** a rational function has numerator degree $\ge$ denominator degree.

Divide to get polynomial $+$ proper fraction, then integrate each piece. For example, $\frac{2x^2}{1+x^2} = 2 - \frac{2}{1+x^2}$.

### 7. Partial fraction decomposition (Example 8)
**When:** you have a proper rational function whose denominator factors, and the numerator is not simply the derivative of the denominator.

1. Factor the denominator. A root $x = r$ means $(x - r)$ is a factor. Divide it out to find the rest.
2. Write the PFD form: $\frac{A}{\text{linear}}$ for each linear factor and $\frac{Bx + C}{\text{quadratic}}$ for each irreducible quadratic.
3. Clear denominators, then find the constants by plugging in convenient $x$ values and matching coefficients.
4. Integrate each piece. For an irreducible quadratic, split the numerator into (a multiple of the derivative of the denominator) $+$ (a constant), and complete the square for the constant part to get an arctangent.

## Worked examples

### Example 1
Find
$$
\int (\sec(x) - \cos(x))^2 \, dx.
$$

**Strategy:** expand and simplify before integrating.

Expand the square:
$$
(\sec x - \cos x)^2 = \sec^2 x - 2\cos x \sec x + \cos^2 x.
$$

Since $\sec x = \frac{1}{\cos x}$, the middle term is $2\cos x \sec x = 2$. Use the half-angle identity $\cos^2 x = \frac12 + \frac12\cos(2x)$:
$$
= \sec^2 x - 2 + \frac12 + \frac12\cos(2x) = \sec^2 x - \frac32 + \frac12\cos(2x).
$$

Integrate term by term. Formula 7 gives $\int \sec^2 x\,dx = \tan x$, and $\int \cos(2x)\,dx = \frac12\sin(2x)$:
$$
\int (\sec x - \cos x)^2 \, dx = \int \left(\sec^2 x - \frac32 + \frac12\cos(2x)\right) dx = \tan(x) - \frac32 x + \frac14\sin(2x) + C.
$$

### Example 2
Find
$$
\int \frac{x^2}{\sqrt{4 - x^6}} \, dx.
$$

**Strategy:** $x^6 = (x^3)^2$ and $x^2$ is, up to a constant, the derivative of $x^3$, so substitute.

Let $u = x^3$, $du = 3x^2\,dx$, so $x^2\,dx = \frac13 du$:
$$
\int \frac{x^2}{(4 - x^6)^{1/2}}\,dx = \frac13\int \frac{1}{(4 - u^2)^{1/2}}\,du.
$$

Factor 4 out of the root: $(4 - u^2)^{1/2} = \left(4\left(1 - (u/2)^2\right)\right)^{1/2} = 2\left(1 - (u/2)^2\right)^{1/2}$. So
$$
= \frac16\int \frac{1}{\left(1 - (u/2)^2\right)^{1/2}}\,du.
$$

Substitute again: $w = \frac{u}{2}$, $dw = \frac12 du$, so $du = 2\,dw$:
$$
\frac16\int \frac{1}{(1 - (u/2)^2)^{1/2}}\,du = \frac13\int \frac{1}{(1 - w^2)^{1/2}}\,dw = \frac13\arcsin(w) + C = \frac13\arcsin\left(\frac{u}{2}\right) + C.
$$

Substitute back $u = x^3$:
$$
\int \frac{x^2}{\sqrt{4 - x^6}}\,dx = \frac13\arcsin\left(\frac{x^3}{2}\right) + C.
$$

(Check: formula 18 with $a = 2$ gives $\int \frac{du}{\sqrt{4 - u^2}} = \sin^{-1}(u/2)$ directly, which matches.)

### Example 3
Find
$$
\int \frac{4 - 3x^2}{x(x^2 - 4)} \, dx.
$$

**Strategy:** before doing partial fractions, check whether the numerator is related to the derivative of the denominator.

Expand the denominator:
$$
\frac{4 - 3x^2}{x(x^2 - 4)} = \frac{4 - 3x^2}{x^3 - 4x}.
$$

Let $u = x^3 - 4x$, $du = (3x^2 - 4)\,dx$. The numerator is $4 - 3x^2 = -(3x^2 - 4)$, so $(4 - 3x^2)\,dx = -du$:
$$
\int \frac{4 - 3x^2}{x(x^2-4)}\,dx = -\int \frac{1}{u}\,du = -\ln|u| + C.
$$

Use $-\ln|u| = \ln|u^{-1}|$:
$$
= \ln\left|\frac{1}{u}\right| + C = \ln\left|\frac{1}{x^3 - 4x}\right| + C.
$$

No PFD needed.

### Example 4
Find
$$
\int x e^{\sqrt{1 + x^2}} \, dx.
$$

**First attempt:** $u = 1 + x^2$, $du = 2x\,dx$ gives
$$
\int x e^{\sqrt{1+x^2}}\,dx = \frac12\int e^{u^{1/2}}\,du.
$$
This needs another clever substitution and integration by parts to finish, so try something else.

**Better approach:** there is a $1 + x^2$ term, so use a trig substitution. Let
$$
x = \tan t, \qquad dx = \sec^2 t\,dt, \qquad t \in (-\pi/2, \pi/2),
$$
which makes $\sec t > 0$. Since $1 + \tan^2 t = \sec^2 t$, we get $(1 + \tan^2 t)^{1/2} = \sec t$:
$$
\int x e^{\sqrt{1+x^2}}\,dx = \int \tan t \, e^{(1 + \tan^2 t)^{1/2}} \sec^2 t\,dt = \int \tan t \, e^{\sec t}\sec^2 t\,dt.
$$

Now let $u = \sec t$, $du = \sec t \tan t\,dt$. Split $\tan t \sec^2 t\,dt = \sec t \cdot (\sec t \tan t\,dt) = u\,du$:
$$
\int \tan t\, e^{\sec t}\sec^2 t\,dt = \int u e^u\,du.
$$

Tabular integration by parts:

| sign | $u$'s | $dv$'s |
|---|---|---|
| $+$ | $u$ | $e^u$ |
| $-$ | $1$ | $e^u$ |
| | $0$ | $e^u$ |

$$
\int u e^u\,du = u e^u - e^u + C = (u - 1)e^u + C = (\sec t - 1)e^{\sec t} + C.
$$

**Back to $x$:** draw a right triangle with angle $t$, opposite side $x$, adjacent side $1$, hypotenuse $\sqrt{x^2 + 1}$. Then
$$
\tan t = x \implies \sec t = \frac{1}{\cos t} = (1 + x^2)^{1/2}.
$$

Finally,
$$
\int x e^{\sqrt{x^2+1}}\,dx = \left(\sqrt{x^2+1} - 1\right)e^{\sqrt{x^2+1}} + C.
$$

### Example 5
Find
$$
\int \ln(1 + x^2) \, dx.
$$

**Strategy:** a lone logarithm calls for integration by parts with $dv = dx$.

$$
u = \ln(1+x^2), \quad dv = dx, \qquad du = \frac{2x}{1+x^2}\,dx, \quad v = x.
$$

$$
\int \ln(1+x^2)\,dx = uv - \int v\,du = x\ln(1+x^2) - \int \frac{2x^2}{1+x^2}\,dx.
$$

The remaining integrand is a rational function with numerator degree $=$ denominator degree, so use polynomial division. $1 + x^2$ goes into $2x^2$ twice: $2(x^2 + 1) = 2x^2 + 2$, and subtracting leaves a remainder of $-2$. So
$$
2(1 + x^2) - 2 = 2x^2 \implies \frac{2x^2}{1+x^2} = 2 - \frac{2}{1+x^2}.
$$

Integrate, using formula 17 with $a = 1$:
$$
\int \frac{2x^2}{1+x^2}\,dx = \int \left(2 - \frac{2}{1+x^2}\right)dx = 2x - 2\arctan(x) + C.
$$

Therefore
$$
\int \ln(1+x^2)\,dx = x\ln(1+x^2) - 2x + 2\arctan(x) + C.
$$

### Example 6
Find
$$
\int x e^{\sqrt{x}} \, dx.
$$

**First attempt (integration by parts):**
$$
u = e^{x^{1/2}}, \quad dv = x\,dx, \qquad du = \frac{e^{x^{1/2}}}{2x^{1/2}}\,dx, \quad v = \frac12 x^2.
$$
(The slides note that even $\int e^{x^{1/2}}\,dx$ on its own needs integration by parts, so taking $dv = e^{\sqrt{x}}\,dx$ isn't easy either.) This gives
$$
\int x e^{x^{1/2}}\,dx = \frac12 x^2 e^{x^{1/2}} - \int \frac12 x^2 \cdot \frac{e^{x^{1/2}}}{2x^{1/2}}\,dx,
$$
and the new integral is **more complicated**. Abandon this approach.

**Instead, substitute:** $u = x^{1/2}$, $du = \frac{1}{2x^{1/2}}\,dx$, so $dx = 2x^{1/2}\,du = 2u\,du$. Also $x = u^2$:
$$
\int x e^{x^{1/2}}\,dx = \int u^2 e^u \cdot 2u\,du = 2\int u^3 e^u\,du.
$$

**Equivalent way to see it:** multiply by $\frac{2x^{1/2}}{2x^{1/2}} = 1$:
$$
x e^{x^{1/2}} = x e^{x^{1/2}} \cdot \frac{2x^{1/2}}{2x^{1/2}} = 2x^{3/2}e^{x^{1/2}} \cdot \frac{1}{2x^{1/2}},
$$
then with $u = x^{1/2}$, $du = \frac{1}{2x^{1/2}}\,dx$, and $x^{3/2} = u^3$:
$$
\int x e^{x^{1/2}}\,dx = 2\int x^{3/2} e^{x^{1/2}} \cdot \frac{1}{2x^{1/2}}\,dx = 2\int u^3 e^u\,du.
$$

**Tabular integration by parts:**

| sign | $u$'s | $dv$'s |
|---|---|---|
| $+$ | $u^3$ | $e^u$ |
| $-$ | $3u^2$ | $e^u$ |
| $+$ | $6u$ | $e^u$ |
| $-$ | $6$ | $e^u$ |
| | $0$ | $e^u$ |

$$
2\int u^3 e^u\,du = 2(u^3 - 3u^2 + 6u - 6)e^u + C.
$$

Substitute back $u = x^{1/2}$, so $u^3 = x^{3/2}$ and $u^2 = x$:
$$
\int x e^{\sqrt{x}}\,dx = 2\left(x^{3/2} - 3x + 6x^{1/2} - 6\right)e^{\sqrt{x}} + C.
$$

### Example 7
Find
$$
\int x^3 \ln(x) \, dx.
$$

**Strategy:** a polynomial times a logarithm calls for integration by parts, with $u$ = the logarithm.

$$
u = \ln x, \quad dv = x^3\,dx, \qquad du = \frac1x\,dx, \quad v = \frac14 x^4.
$$

$$
\int x^3\ln x\,dx = \frac14 x^4 \ln x - \int \frac14 x^4 \cdot \frac1x\,dx = \frac14 x^4\ln x - \frac14\int x^3\,dx.
$$

Since $\int x^3\,dx = \frac14 x^4$:
$$
\int x^3\ln(x)\,dx = \frac14 x^4\ln(x) - \frac1{16}x^4 + C.
$$

### Example 8
Find
$$
\int \frac{1}{x^3 + 1} \, dx.
$$

**Step 1: factor the denominator.** $x^3 + 1 = 0$ when $x = -1$, so $x + 1$ divides $x^3 + 1$, and
$$
x^3 + 1 = (x+1)(x^2 - x + 1).
$$
The quadratic $x^2 - x + 1$ is irreducible (discriminant $1 - 4 = -3 < 0$).

**Step 2: PFD form.**
$$
\frac{1}{x^3+1} = \frac{1}{(x+1)(x^2-x+1)} = \frac{A}{x+1} + \frac{Bx + C}{x^2 - x + 1}.
$$

Multiply through by $(x+1)(x^2 - x + 1)$:
$$
1 = A(x^2 - x + 1) + (Bx + C)(x + 1) = A(x^2 - x + 1) + B(x^2 + x) + C(x + 1).
$$

**Step 3: solve for the constants.**

- Let $x = -1$: $1 = A(1 + 1 + 1) = 3A$, so $A = \frac13$.
- $x^2$ terms: $0 = (A + B)x^2$, so $B = -A = -\frac13$.
- Constant terms: $1 = A + C$, so $C = \frac23$.

(Check with the $x$ terms: $-A + B + C = -\frac13 - \frac13 + \frac23 = 0$. This matches, since the left side has no $x$ term.)

So
$$
\frac{1}{x^3+1} = \frac13\cdot\frac{1}{x+1} + \frac13\cdot\frac{-x + 2}{x^2 - x + 1},
$$
$$
\int \frac{1}{x^3+1}\,dx = \frac13\int\frac{1}{x+1}\,dx + \frac13\int\frac{-x+2}{x^2-x+1}\,dx.
$$

The first piece is $\frac13\ln|x+1|$.

**Step 4: the quadratic piece.** This uses the same method as $\int \frac{-x+9}{x^2+2x+5}\,dx$ from §7.4. The derivative of $x^2 - x + 1$ is $2x - 1$, so split the numerator into a multiple of that plus a constant:
$$
-x + 2 = -\frac12(2x - 1) + \frac32.
$$
Then
$$
\int \frac{-x+2}{x^2-x+1}\,dx = -\frac12\int\frac{2x-1}{x^2-x+1}\,dx + \frac32\int\frac{1}{x^2-x+1}\,dx.
$$

The first integral is $\ln|x^2 - x + 1|$ (numerator is the derivative of the denominator). For the second, complete the square:
$$
x^2 - x + 1 = \left(x - \tfrac12\right)^2 + \tfrac34 = \left(x - \tfrac12\right)^2 + \left(\tfrac{\sqrt3}{2}\right)^2.
$$
By formula 17 with $a = \frac{\sqrt3}{2}$:
$$
\int \frac{dx}{\left(x - \frac12\right)^2 + \left(\frac{\sqrt3}{2}\right)^2} = \frac{2}{\sqrt3}\arctan\left(\frac{x - \frac12}{\sqrt3/2}\right) = \frac{2}{\sqrt3}\arctan\left(\frac{2x-1}{\sqrt3}\right).
$$

Putting it together and multiplying by $\frac13$:
$$
\frac13\int\frac{-x+2}{x^2-x+1}\,dx = \frac13\left(-\frac12\ln|x^2-x+1| + \frac32\cdot\frac{2}{\sqrt3}\arctan\left(\frac{2x-1}{\sqrt3}\right)\right) = -\frac16\ln|x^2-x+1| + \frac{\sqrt3}{3}\arctan\left(\frac{2x-1}{\sqrt3}\right) + C,
$$
using $\frac13 \cdot \frac{3}{\sqrt3} = \frac{1}{\sqrt3} = \frac{\sqrt3}{3}$.

**Answer:**
$$
\int\frac{1}{x^3+1}\,dx = \frac13\ln|x+1| - \frac16\ln|x^2-x+1| + \frac{\sqrt3}{3}\arctan\left(\frac{2x-1}{\sqrt3}\right) + C.
$$

## Common mistakes and tips

- **Simplify before you integrate.** In Example 1, expanding and using $\cos x \sec x = 1$ and $\cos^2 x = \frac12 + \frac12\cos 2x$ turns the problem into three table integrals.
- **Look for $f'/f$ before starting partial fractions.** In Example 3 the numerator is $-1$ times the derivative of the expanded denominator, so a single substitution finishes it. Expanding $x(x^2-4)$ to $x^3 - 4x$ is what makes this visible.
- **Drop an approach that makes things worse.** Examples 4 and 6 both show a first attempt ($u = 1 + x^2$, or parts with $u = e^{\sqrt x}$) that leaves a harder integral. Recognize this quickly and switch.
- **A $\sqrt{x}$ inside an exponential suggests $u = \sqrt{x}$.** Then $dx = 2u\,du$ and $x = u^2$, which turns the integrand into polynomial times $e^u$ (Example 6).
- **A $1 + x^2$ under a root suggests $x = \tan t$.** Restrict $t \in (-\pi/2, \pi/2)$ so that $\sqrt{1 + \tan^2 t} = \sec t$ with no absolute value.
- **Convert trig substitutions back to $x$** with a right triangle (Example 4). Don't leave the answer in $t$.
- **Choosing $u$ in integration by parts:** pick the logarithm as $u$ (Examples 5 and 7). For a lone $\ln(\cdot)$, use $dv = dx$.
- **Tabular method signs alternate** $+, -, +, -$ starting with $+$. Stop when the $u$ column reaches $0$.
- **Divide first when degrees are not proper.** $\frac{2x^2}{1+x^2}$ has equal degrees, so divide to get $2 - \frac{2}{1+x^2}$ (Example 5).
- **Track constants through substitutions.** In Example 2, the $\frac13$ from $du = 3x^2 dx$ and the $\frac12$ from factoring $\sqrt4$ combine to $\frac16$, and then $du = 2\,dw$ brings it back to $\frac13$.
- **Factoring cubics:** find a root $r$ by inspection, then $(x - r)$ is a factor (Example 8). Pairing plug-in values with matching coefficients is the fastest way to get the PFD constants.
- **Irreducible quadratic denominators:** split the numerator into (multiple of the derivative) $+$ constant, giving a $\ln$ term plus an $\arctan$ term after completing the square.
- **Always include $+C$** on indefinite integrals, and keep the absolute values in $\ln|\cdot|$.

## Formula sheet

Basic table (memorize all of these):

$$
\int x^n\,dx = \frac{x^{n+1}}{n+1}\ (n \ne -1) \qquad \int \frac1x\,dx = \ln|x| \qquad \int e^x\,dx = e^x \qquad \int b^x\,dx = \frac{b^x}{\ln b}
$$

$$
\int \sin x\,dx = -\cos x \qquad \int \cos x\,dx = \sin x \qquad \int \sec^2 x\,dx = \tan x \qquad \int \csc^2 x\,dx = -\cot x
$$

$$
\int \sec x\tan x\,dx = \sec x \qquad \int \csc x\cot x\,dx = -\csc x
$$

$$
\int \sec x\,dx = \ln|\sec x + \tan x| \qquad \int \csc x\,dx = \ln|\csc x - \cot x|
$$

$$
\int \tan x\,dx = \ln|\sec x| \qquad \int \cot x\,dx = \ln|\sin x| \qquad \int \sinh x\,dx = \cosh x \qquad \int \cosh x\,dx = \sinh x
$$

$$
\int \frac{dx}{x^2 + a^2} = \frac1a\tan^{-1}\left(\frac xa\right) \qquad \int \frac{dx}{\sqrt{a^2 - x^2}} = \sin^{-1}\left(\frac xa\right),\ a > 0
$$

$$
\int \frac{dx}{x^2 - a^2} = \frac{1}{2a}\ln\left|\frac{x-a}{x+a}\right| \qquad \int \frac{dx}{\sqrt{x^2 \pm a^2}} = \ln\left|x + \sqrt{x^2 \pm a^2}\right|
$$

Tools used in the examples:

$$
\int u\,dv = uv - \int v\,du
$$

$$
\int \frac{f'(x)}{f(x)}\,dx = \ln|f(x)| + C
$$

$$
\cos^2 x = \frac12 + \frac12\cos(2x) \qquad 1 + \tan^2 t = \sec^2 t \qquad \cos x \sec x = 1
$$

$$
\int u e^u\,du = (u - 1)e^u + C \qquad \int u^3 e^u\,du = (u^3 - 3u^2 + 6u - 6)e^u + C
$$

$$
x^3 + 1 = (x + 1)(x^2 - x + 1)
$$
