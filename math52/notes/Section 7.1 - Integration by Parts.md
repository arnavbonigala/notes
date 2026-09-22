# Section 7.1 - Integration by Parts

## Summary

Integration by parts is the integration rule that comes from the Product Rule for derivatives. It rewrites $\int u\,dv$ as $uv - \int v\,du$, which trades an integral you can't do directly for one you can. Use it when the integrand is a product of two kinds of functions, such as a polynomial times $e^{kx}$, $\sin(kx)$, $\cos(kx)$, or $\ln(x)$, or $e^{kx}$ times a sine or cosine. It also works on single functions with no easy antiderivative, like $\ln(x)$ and $\ln^2(x)$, if you write them as the function times $1$. When you would have to apply integration by parts several times in a row, the tabular method keeps the work organized.

## Definitions and theorems

**Definition #1 (Real numbers).** The set of all **real numbers** is denoted by $\mathbf{R}$. If $x$ is a real number we express this as $x \in \mathbf{R}$.

**Definition #2.** We also have the sets of **natural numbers**, **integers**, and **rational numbers**:

$$
\mathbf{N} = \{1, 2, 3, \dots\}
$$

$$
\mathbf{Z} = \{0, \pm 1, \pm 2, \pm 3, \dots\}
$$

$$
\mathbf{Q} = \{m/n \mid m \in \mathbf{Z} \text{ and } n \in \mathbf{N}\}
$$

Observe

$$
\mathbf{N} \subset \mathbf{Z} \subset \mathbf{Q} \subset \mathbf{R}.
$$

For example, $-3 \in \mathbf{Z}$, $-\frac{3}{2} \in \mathbf{Q}$, and $\sqrt{2} \in \mathbf{R}$, but $-3, -\frac{3}{2}, \sqrt{2} \notin \mathbf{N}$.

**Definition #3 (Antiderivative).** Let $F$ and $f$ be functions on an interval $I$. If $F'(x) = f(x)$ on $I$, then $F$ is an **antiderivative** of $f$. We denote this as

$$
\int f(x)\,dx = F(x).
$$

**Theorem #1.** Let $F$ be any antiderivative of the function $f$ on an interval $I$. Then all of $f$'s antiderivatives are of the form

$$
F(x) + C
$$

where $C \in \mathbf{R}$.

For example, $\int e^x\,dx = e^x + C$ and $\int x^5\,dx = \frac{1}{6}x^6 + C$, where $C \in \mathbf{R}$.

**Theorem #2 (Fundamental Theorem of Calculus).** Let $f$ be continuous on the interval $[a,b]$ and define $g$ by

$$
g(x) = \int_a^x f(t)\,dt.
$$

Then

1. $g$ is continuous on $[a,b]$ and $g'$ exists on $(a,b)$, where $g'(x) = f(x)$.
2. $g$ is an antiderivative of $f$ on $[a,b]$.
3. We have
$$
F(b) - F(a) = \int_a^b f(x)\,dx
$$
where $F$ is any antiderivative of $f$ on $[a,b]$.

**Remark (shorthand).**

$$
F(b) - F(a) = F(x)\Big]_a^b = \Big[F(x)\Big]_a^b = F(x)\Big|_{x=a}^{b}
$$

**Theorem #3 (Chain Rule).** Let $g$ be differentiable at $x$ and $f$ be differentiable at $g(x)$. Then

$$
\frac{d}{dx} f(g(x)) = f'(g(x)) \cdot g'(x).
$$

**Theorem #4 (Integration by Substitution).** Let $u = g(x)$ be a differentiable function whose range is an interval $I$. If $f$ is continuous on $I$ then

$$
\int f(g(x)) \cdot g'(x)\,dx = \int f(u)\,du.
$$

**Theorem #5 (Integration by Substitution for Definite Integrals).** Let $g'$ be continuous on the interval $[a,b]$. If $f$ is continuous on the range of $u = g(x)$ then

$$
\int_a^b f(g(x)) \cdot g'(x)\,dx = \int_{g(a)}^{g(b)} f(u)\,du.
$$

**Theorem #6 (Product Rule).** Let $u$ and $v$ be differentiable functions. Then

$$
\frac{d}{dx}\big(u(x) \cdot v(x)\big) = u'(x) \cdot v(x) + u(x) \cdot v'(x).
$$

**Derivation of integration by parts from the Product Rule.** By the Product Rule,

$$
\frac{d}{dx}\big(f(x)g(x)\big) = f'(x)g(x) + f(x)g'(x).
$$

Integrate both sides. The left side is the integral of a derivative, so it gives back $f(x)g(x)$:

$$
f(x)g(x) = \int f'(x)g(x)\,dx + \int f(x)g'(x)\,dx.
$$

Subtract $\int f'(x)g(x)\,dx$ from both sides:

$$
\int f(x)g'(x)\,dx = f(x)g(x) - \int f'(x)g(x)\,dx.
$$

Now let

$$
u = f(x), \quad v = g(x), \quad du = f'(x)\,dx, \quad dv = g'(x)\,dx.
$$

Then

$$
\int u\,dv = u \cdot v - \int v\,du.
$$

**Theorem #7 (Integration by Parts).** Let $u$ and $v$ be differentiable functions. Then

$$
\int u\,dv = u \cdot v - \int v\,du.
$$

**Theorem #8 (Integration by Parts for Definite Integrals).** For $u'$ and $v'$ continuous,

$$
\int_a^b u(x) \cdot v'(x)\,dx = u(x) \cdot v(x)\Big|_{x=a}^{b} - \int_a^b v(x) \cdot u'(x)\,dx.
$$

## Methods

### Method 1: Integration by parts (one application)

**When to use:** the integrand is a product $u\,dv$ where $u$ gets simpler when you differentiate it and $dv$ is easy to integrate.

1. Split the integrand into $u$ and $dv$. Everything, including $dx$, has to go into one of the two.
2. Differentiate $u$ to get $du$. Integrate $dv$ to get $v$. You can leave off the constant when you find $v$.
3. Substitute into $\int u\,dv = uv - \int v\,du$.
4. Evaluate the new integral $\int v\,du$ and add $+C$.

The slides organize the pieces in a table:

$$
\begin{array}{cc}
u\text{'s} & dv\text{'s} \\
\hline
u & dv \\
du & v
\end{array}
$$

The diagonal from $u$ down to $v$ carries a $+$ sign and gives the term $uv$. The bottom row, from $du$ across to $v$, carries a $-$ sign and gives $-\int v\,du$.

### Method 2: Choosing $u$ and $dv$ (rules from the slides)

| Integrand | Choose $u$ | Choose $dv$ | Rule |
|---|---|---|---|
| $p(x)\ln(x)$, $p$ a polynomial | $\ln(x)$ | $p(x)\,dx$ | Differentiate the logarithm |
| $p(x)e^{kx}$ | $p(x)$ | $e^{kx}\,dx$ | Differentiate the polynomial |
| $p(x)\sin(kx)$ or $p(x)\cos(kx)$ | $p(x)$ | $\sin(kx)\,dx$ or $\cos(kx)\,dx$ | Differentiate the polynomial |
| $e^{kx}\sin(mx)$ or $e^{kx}\cos(nx)$ | either one | the other one | Apply tabular integration by parts twice, then solve |
| A single function with no product, like $\ln(x)$ | the function | $1 \cdot dx$ | Create the product with $1$ |

### Method 3: Tabular integration by parts

**When to use:** $u$ is a polynomial that reaches $0$ after repeated differentiation, so plain integration by parts would have to be applied again and again (Examples 6, 7, 8).

1. Make two columns: $u$'s on the left, $dv$'s on the right.
2. In the left column, write $u$ and then differentiate it repeatedly until you reach $0$.
3. In the right column, write the function from $dv$ and then integrate repeatedly, one row for each row on the left.
4. Draw diagonals from each left entry down to the next row on the right. Label them $+, -, +, -, \dots$, starting with $+$.
5. The answer is the sum of the signed diagonal products, plus $C$.

### Method 4: Tabular integration by parts that loops back (solve for $I$)

**When to use:** integrals like $\int e^{kx}\sin(mx)\,dx$ or $\int e^{kx}\cos(nx)\,dx$, where neither factor ever differentiates to $0$ (Example 9).

1. Name the original integral $I$.
2. Build the table for two rows of differentiation (three rows total).
3. Take the signed diagonal products as usual. The bottom row, read straight across with its sign, is $\int v\,du$. It is a constant multiple of $I$.
4. Write the equation $I = (\text{diagonal terms}) + (\text{multiple of } I)$.
5. Move the $I$ terms to one side, add $C$, and solve for $I$.

### Method 5: Definite integrals

**When to use:** the integral has limits.

Find the antiderivative with integration by parts and then evaluate it with the Fundamental Theorem of Calculus (Example 3). Or use Theorem #8 directly, evaluating the $uv$ term from $a$ to $b$.

### Method 6: Reduction formula for $\int \ln^n(x)\,dx$

**When to use:** powers of $\ln(x)$ (Example 10). Set $u = \ln^n(x)$ and $dv = dx$ to get

$$
\int \ln^n(x)\,dx = x\ln^n(x) - n\int \ln^{n-1}(x)\,dx.
$$

Each application lowers the power of $\ln(x)$ by one.

## Worked examples

### Review example (Chain Rule versus substitution)

**Find $\int -2xe^{-x^2}\,dx$.**

*By the Chain Rule.* Differentiate $e^{-x^2}$:

$$
\frac{d}{dx} e^{-x^2} = e^{-x^2} \frac{d}{dx}(-x^2) = e^{-x^2}(-2x).
$$

$e^{-x^2}$ is an antiderivative of $-2xe^{-x^2}$, so

$$
\int -2xe^{-x^2}\,dx = e^{-x^2} + C.
$$

*By substitution.* Let $u = -x^2$, so $du = -2x\,dx$. The integrand is exactly $e^{u}\,du$:

$$
\int -2xe^{-x^2}\,dx = \int e^u\,du = e^u + C = e^{-x^2} + C.
$$

Both methods give the same answer.

### Example #1

**Find $\displaystyle\int \frac{x}{x-3}\,dx$.**

Let $u = x - 3$, so $du = dx$. Solve the substitution for $x$: $x = u + 3$. Then

$$
\int \frac{x}{x-3}\,dx = \int \frac{u+3}{u}\,du.
$$

Split the fraction: $\frac{u+3}{u} = \frac{u}{u} + \frac{3}{u} = 1 + \frac{3}{u}$. So

$$
= \int \left(1 + \frac{3}{u}\right) du = u + 3\ln\lvert u\rvert + C.
$$

Back substitute $u = x - 3$:

$$
\int \frac{x}{x-3}\,dx = x - 3 + 3\ln\lvert x-3\rvert + C = x + 3\ln\lvert x-3\rvert + C
$$

where $C \in \mathbf{R}$. The $-3$ gets absorbed into the constant, since $C - 3$ is just another arbitrary real constant. The slides write this as "$C - 3$" $= C$.

### Example #2

**Find $\displaystyle\int 3xe^x\,dx$.**

Choose the polynomial as $u$:

$$
u = 3x, \qquad dv = e^x\,dx
$$

$$
du = 3\,dx, \qquad v = e^x
$$

By integration by parts:

$$
\int 3xe^x\,dx = \int u\,dv = uv - \int v\,du = 3xe^x - \int 3e^x\,dx.
$$

Since $\int 3e^x\,dx = 3e^x + C$,

$$
= 3xe^x - 3e^x + C = 3(x-1)e^x + C.
$$

The last step factors out $3e^x$.

### Example #3

**Find $\displaystyle\int_0^1 3xe^x\,dx$.**

Example 2 showed that $3(x-1)e^x$ is an antiderivative of $3xe^x$. By the Fundamental Theorem of Calculus:

$$
\int_0^1 3xe^x\,dx = 3(x-1)e^x\Big|_{x=0}^{1}
$$

$$
= 3 \cdot (1-1) \cdot e^1 - 3 \cdot (0-1) \cdot e^0
$$

$$
= 3 \cdot 0 \cdot e - 3 \cdot (-1) \cdot 1 = 0 + 3 = 3.
$$

Geometrically, $3$ is the area of the region under the curve $y = 3xe^x$ from $x = 0$ to $x = 1$. The curve is above the $x$-axis there, since $3xe^x \ge 0$ for $x \ge 0$.

### Example #4

**Find $\displaystyle\int \ln(x)\,dx$.**

There is no product of functions, so create one by writing the integrand as $\ln(x) \cdot 1$:

$$
u = \ln(x), \qquad dv = 1 \cdot dx
$$

$$
du = \frac{1}{x}\,dx, \qquad v = x
$$

Then

$$
\int \ln(x) \cdot 1\,dx = \ln(x) \cdot x - \int x \cdot \frac{1}{x}\,dx.
$$

Simplify $x \cdot \frac{1}{x} = 1$:

$$
= \ln(x) \cdot x - \int 1\,dx = \ln(x) \cdot x - x + C.
$$

### Example #5

**Find $\displaystyle\int x^5\ln(x)\,dx$.**

This has the form $p(x)\ln(x)$, so differentiate the logarithm:

$$
u = \ln(x), \qquad dv = x^5\,dx
$$

$$
du = \frac{1}{x}\,dx, \qquad v = \frac{x^6}{6}
$$

By integration by parts:

$$
\int x^5\ln(x)\,dx = \int u\,dv = uv - \int v\,du = \ln(x) \cdot \frac{x^6}{6} - \int \frac{x^6}{6} \cdot \frac{1}{x}\,dx.
$$

Simplify $\frac{x^6}{6} \cdot \frac{1}{x} = \frac{x^5}{6}$:

$$
= \frac{x^6}{6}\ln(x) - \int \frac{x^5}{6}\,dx.
$$

Since $\int \frac{x^5}{6}\,dx = \frac{1}{6} \cdot \frac{x^6}{6} = \frac{x^6}{36}$,

$$
= \frac{x^6}{6}\ln(x) - \frac{x^6}{36} + C
$$

where $C \in \mathbf{R}$.

### Example #6

**Find $\displaystyle\int x^2e^{2x}\,dx$.**

This has the form $p(x)e^{kx}$, so differentiate the polynomial:

$$
u = x^2, \qquad dv = e^{2x}\,dx
$$

$$
du = 2x\,dx, \qquad v = \frac{1}{2}e^{2x}
$$

By integration by parts:

$$
\int x^2e^{2x}\,dx = uv - \int v\,du = x^2 \cdot \frac{1}{2}e^{2x} - \int 2x \cdot \frac{1}{2}e^{2x}\,dx = \frac{1}{2}x^2e^{2x} - \int xe^{2x}\,dx.
$$

The remaining integral $\int xe^{2x}\,dx$ needs another round of integration by parts. Rather than apply the process again and risk confusion, use a table. Differentiate $x^2$ down to $0$ on the left. Integrate $e^{2x}$ repeatedly on the right, and each integration multiplies the coefficient by $\frac{1}{2}$:

| Sign | $u$'s | $dv$'s |
|---|---|---|
| $+$ | $x^2$ | $e^{2x}$ |
| $-$ | $2x$ | $\frac{1}{2}e^{2x}$ |
| $+$ | $2$ | $\frac{1}{4}e^{2x}$ |
|  | $0$ | $\frac{1}{8}e^{2x}$ |

Each sign multiplies the $u$ in its row by the $dv$ entry one row down. Adding the diagonal products:

$$
\int x^2e^{2x}\,dx = \left(x^2 \cdot \frac{1}{2} - 2x \cdot \frac{1}{4} + 2 \cdot \frac{1}{8}\right)e^{2x} + C
$$

$$
= \left(\frac{x^2}{2} - \frac{x}{2} + \frac{1}{4}\right)e^{2x} + C
$$

where $C \in \mathbf{R}$. The last row contributes nothing because its $u$ entry is $0$.

### Example #7

**Find $\displaystyle\int x^5e^x\,dx$.**

Use the table. Differentiate $x^5$ until you reach $0$. The antiderivative of $e^x$ is $e^x$, so the right column is $e^x$ in every row.

| Sign | $u$'s | $dv$'s |
|---|---|---|
| $+$ | $x^5$ | $e^x$ |
| $-$ | $5x^4$ | $e^x$ |
| $+$ | $20x^3$ | $e^x$ |
| $-$ | $60x^2$ | $e^x$ |
| $+$ | $120x$ | $e^x$ |
| $-$ | $120$ | $e^x$ |
|  | $0$ | $e^x$ |

The derivatives are $5x^4$, then $5 \cdot 4x^3 = 20x^3$, then $20 \cdot 3x^2 = 60x^2$, then $60 \cdot 2x = 120x$, then $120$, then $0$. Adding the signed diagonal products and factoring out $e^x$:

$$
\int x^5e^x\,dx = \left(x^5 - 5x^4 + 20x^3 - 60x^2 + 120x - 120\right)e^x + C
$$

where $C \in \mathbf{R}$.

### Example #8

**Find $\displaystyle\int x^2\sin(x)\,dx$.**

This has the form $p(x)\sin(kx)$, so differentiate the polynomial:

$$
u = x^2, \qquad dv = \sin(x)\,dx
$$

$$
du = 2x\,dx, \qquad v = -\cos(x)
$$

By integration by parts:

$$
\int x^2\sin(x)\,dx = uv - \int v\,du = x^2(-\cos(x)) - \int 2x \cdot (-\cos(x))\,dx = -x^2\cos(x) + \int 2x\cos(x)\,dx.
$$

The integral $\int 2x\cos(x)\,dx$ needs more integration by parts, so use a table instead. On the right, integrate repeatedly: $\int \sin(x)\,dx = -\cos(x)$, $\int -\cos(x)\,dx = -\sin(x)$, $\int -\sin(x)\,dx = \cos(x)$.

| Sign | $u$'s | $dv$'s |
|---|---|---|
| $+$ | $x^2$ | $\sin(x)$ |
| $-$ | $2x$ | $-\cos(x)$ |
| $+$ | $2$ | $-\sin(x)$ |
|  | $0$ | $\cos(x)$ |

Thus

$$
\int x^2\sin(x)\,dx = x^2(-\cos(x)) - 2x(-\sin(x)) + 2\cos(x) + C
$$

$$
= -x^2\cos(x) + 2x\sin(x) + 2\cos(x) + C
$$

$$
= (-x^2 + 2)\cos(x) + 2x\sin(x) + C
$$

where $C \in \mathbf{R}$.

### Example #9

**Find $\displaystyle\int \sin(x)e^{2x}\,dx$.**

Neither factor differentiates to $0$. The plan is to apply integration by parts twice and generate a multiple of the original integral $\int u\,dv$. Call the original integral $I$:

$$
I = \int \sin(x)e^{2x}\,dx.
$$

Build the table with $u = \sin(x)$ and $dv = e^{2x}\,dx$. Differentiate twice on the left and integrate twice on the right:

| Sign | $u$'s | $dv$'s |
|---|---|---|
| $+$ | $\sin(x)$ | $e^{2x}$ |
| $-$ | $\cos(x)$ | $\frac{1}{2}e^{2x}$ |
| $+$ (across) | $-\sin(x)$ | $\frac{1}{4}e^{2x}$ |

The first two signs are diagonals as usual. The bottom row, read straight across with a $+$ sign, represents the leftover integral $\int v\,du$, which here is $\int (-\sin(x)) \cdot \frac{1}{4}e^{2x}\,dx$. So

$$
I = \sin(x) \cdot \frac{1}{2}e^{2x} - \cos(x) \cdot \frac{1}{4}e^{2x} + \int (-\sin(x)) \cdot \frac{1}{4}e^{2x}\,dx
$$

$$
= \left(\frac{1}{2}\sin(x) - \frac{1}{4}\cos(x)\right)e^{2x} - \frac{1}{4}\int \sin(x)e^{2x}\,dx.
$$

The integral on the right is $I$ again:

$$
I = \left(\frac{1}{2}\sin(x) - \frac{1}{4}\cos(x)\right)e^{2x} - \frac{1}{4}I.
$$

Solve for $I$. Add $\frac{1}{4}I$ to both sides. Now that no integral is left on the right, attach the constant $C$:

$$
I + \frac{1}{4}I = \left(\frac{1}{2}\sin(x) - \frac{1}{4}\cos(x)\right)e^{2x} + C
$$

$$
\frac{5}{4}I = \left(\frac{1}{2}\sin(x) - \frac{1}{4}\cos(x)\right)e^{2x} + C.
$$

Multiply both sides by $\frac{4}{5}$. The constant becomes $\frac{4}{5}C$, which is still an arbitrary constant, so call it $C_1$:

$$
I = \frac{4}{5}\left(\frac{1}{2}\sin(x) - \frac{1}{4}\cos(x)\right)e^{2x} + C_1.
$$

Distribute: $\frac{4}{5} \cdot \frac{1}{2} = \frac{2}{5}$ and $\frac{4}{5} \cdot \frac{1}{4} = \frac{1}{5}$. Hence

$$
\int \sin(x)e^{2x}\,dx = \left(\frac{2}{5}\sin(x) - \frac{1}{5}\cos(x)\right)e^{2x} + C_1.
$$

### Example #10

**Find $\displaystyle\int \ln^2(x)\,dx = \int \ln(x) \cdot \ln(x)\,dx$.**

As in Example 4, there is no convenient product, so let $dv = dx$:

$$
u = \ln^2(x), \qquad v = x
$$

$$
du = 2\frac{\ln(x)}{x}\,dx, \qquad dv = dx
$$

Here $du$ comes from the Chain Rule: $\frac{d}{dx}\ln^2(x) = 2\ln(x) \cdot \frac{1}{x}$.

By integration by parts:

$$
\int \ln^2(x)\,dx = x\ln^2(x) - \int x \cdot 2\frac{\ln(x)}{x}\,dx = x\ln^2(x) - 2\int \ln(x)\,dx.
$$

The integral $\int \ln(x)\,dx = \ln(x) \cdot x - x + C$ was found earlier in Example 4. Substituting:

$$
= x\ln^2(x) - 2\big(\ln(x) \cdot x - x\big) + C
$$

$$
= x\ln^2(x) - 2\ln(x) \cdot x + 2x + C.
$$

The same steps with $u = \ln^n(x)$ give the general formula:

$$
\int \ln^n(x)\,dx = x\ln^n(x) - n\int \ln^{n-1}(x)\,dx.
$$

## Common mistakes and tips

- **Picking $u$ backwards.** For $p(x)\ln(x)$, differentiate the logarithm ($u = \ln(x)$). For $p(x)e^{kx}$, $p(x)\sin(kx)$, and $p(x)\cos(kx)$, differentiate the polynomial ($u = p(x)$). If you swap them, the new integral gets harder.
- **Thinking you need a visible product.** $\ln(x)$ and $\ln^2(x)$ don't look like products, but writing them as the function times $1$ with $dv = dx$ makes integration by parts work (Examples 4 and 10).
- **Losing the $\frac{1}{k}$ factor.** $\int e^{kx}\,dx = \frac{1}{k}e^{kx}$. In Example 6, the right column goes $e^{2x}, \frac{1}{2}e^{2x}, \frac{1}{4}e^{2x}, \frac{1}{8}e^{2x}$.
- **Sign errors in the table.** Signs alternate $+, -, +, -, \dots$, starting with $+$ on the first diagonal. Signs inside the entries, like $-\cos(x)$ and $-\sin(x)$ in Example 8, are separate from these and must be multiplied in as well.
- **Wrong product in the table.** Each diagonal pairs a $u$ entry with the $dv$ entry **one row below**. Never multiply entries in the same row, except for the bottom row in a loop-back problem like Example 9, where that row means $\int v\,du$.
- **Stopping the table at the right place.** For a polynomial $u$, keep differentiating until you reach $0$. For $e^{kx}\sin(mx)$ or $e^{kx}\cos(nx)$, stop after two differentiations and solve for $I$.
- **Mixing types in the loop-back case.** In $\int e^{kx}\sin(mx)\,dx$ it does not matter which factor is $u$ or $dv$. What matters is choosing the same type of term every time you integrate by parts.
- **The constant in solve-for-$I$ problems.** Add $C$ once the last integral is gone. After multiplying by a number like $\frac{4}{5}$, the result is still an arbitrary constant, so renaming it ($C_1$) is fine.
- **Absorbing constants.** Leftover numbers can merge into $C$, for example $x - 3 + C$ becomes $x + C$ (Example 1).
- **Reuse earlier results.** Example 10 reused $\int \ln(x)\,dx$ from Example 4. Example 3 reused the antiderivative from Example 2 and then applied the FTC.
- **Check with the derivative.** Differentiating your answer should give back the integrand, since that is the definition of an antiderivative (Definition #3).

## Formula sheet

**Integration by parts:**

$$
\int u\,dv = u \cdot v - \int v\,du
$$

**Definite integrals:**

$$
\int_a^b u(x)v'(x)\,dx = u(x)v(x)\Big|_{x=a}^{b} - \int_a^b v(x)u'(x)\,dx
$$

**Substitution:**

$$
\int f(g(x))g'(x)\,dx = \int f(u)\,du, \qquad \int_a^b f(g(x))g'(x)\,dx = \int_{g(a)}^{g(b)} f(u)\,du
$$

**Fundamental Theorem of Calculus:**

$$
\int_a^b f(x)\,dx = F(b) - F(a)
$$

**Product Rule** (the source of integration by parts):

$$
\frac{d}{dx}(uv) = u'v + uv'
$$

**Choosing $u$:**

- $\int p(x)\ln(x)\,dx$: use $u = \ln(x)$ and $dv = p(x)\,dx$.
- $\int p(x)e^{kx}\,dx$, $\int p(x)\sin(kx)\,dx$, $\int p(x)\cos(kx)\,dx$: use $u = p(x)$.
- $\int e^{kx}\sin(mx)\,dx$ or $\int e^{kx}\cos(nx)\,dx$: use the table twice, then solve for the integral.

**Results worth knowing:**

$$
\int \ln(x)\,dx = x\ln(x) - x + C
$$

$$
\int \ln^n(x)\,dx = x\ln^n(x) - n\int \ln^{n-1}(x)\,dx
$$

$$
\int e^{kx}\,dx = \frac{1}{k}e^{kx} + C, \qquad \int \sin(x)\,dx = -\cos(x) + C, \qquad \int \cos(x)\,dx = \sin(x) + C
$$
