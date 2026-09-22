# Section 7.4 - Integration of Rational Functions by Partial Fractions

## Summary

This section shows how to integrate a rational function $\frac{P(x)}{Q(x)}$ by breaking it into a sum of simpler fractions called partial fractions. Each partial fraction integrates with a known tool: a natural log, a power rule after $u$-substitution, or an arctangent after completing the square. Use it whenever the integrand is a ratio of polynomials whose denominator you can factor into linear and irreducible quadratic factors. If the fraction is improper (numerator degree at least the denominator degree), do polynomial long division first. Some integrals that are not rational at first, such as ones with $\sqrt{1+2x}$ or $e^x$, become rational after a substitution, and then partial fractions finishes them.

## Definitions and theorems

**Definition #1 (Polynomial).** Let $a_0, a_1, \ldots, a_n \in \mathbf{R}$ where $a_n \neq 0$. Then

$$
P(x) = a_0 + a_1 x + \ldots + a_n x^n
$$

is a **polynomial** of degree $n$. If $n = 1$ then $P$ is a linear polynomial.

> Example: $P(x) = 5x - 3$ is linear, and $Q(x) = x^3 - 1$ has degree 3.

**Definition #2 (Reducible Polynomial).** Let $P$ be a polynomial of degree $n$. If $u$ and $v$ are polynomials of degree less than $n$ such that

$$
P(x) = u(x) \cdot v(x)
$$

then $P$ is a **reducible polynomial**. Otherwise, $P$ is an **irreducible polynomial**.

> Example: $Q(x) = x^3 - 1 = (x-1)(x^2+x+1)$, so it is reducible. $P(x) = x^2 + 1$ is irreducible because it has no real roots.
>
> Fact from lecture: if $P$ has degree 2 or more and $P(x) = 0$ for some $x \in \mathbf{R}$, then $P$ is reducible.

**Definition #3 (Roots).** Let $P$ be a polynomial and $x_0 \in \mathbf{R}$. If $P(x_0) = 0$, then there exist a $k \in \mathbf{N}$ and a polynomial $Q$ such that

$$
P(x) = (x - x_0)^k Q(x), \quad Q(x_0) \neq 0.
$$

We say $x_0$ is a **root** of $P$ with **order** $k$. If $k = 1$ then $x_0$ is a **simple root**, and if $k > 1$ we have a **repeated root**.

> Example: $P(x) = (x^2+1)(x-5)(x-3)^7$ has a simple root at $5$ and a 7th order root at $3$.

**Definition #4 (Rational Function).** Let $P$ and $Q$ be polynomials. Then $R$ given by

$$
R(x) = \frac{P(x)}{Q(x)}
$$

is a **rational** function. If the degree of $Q$ is greater than the degree of $P$, then $R$ is a **proper rational function**.

> Any proper rational function can be written as a sum of partial fraction terms.
>
> Proper: $\dfrac{-x+2}{x^2+3x+2}$. Improper: $\dfrac{2x^2+5x+6}{x^2+3x+2}$.

**Definition #5 (Partial Fraction Decomposition).** Let $R$ be a proper rational function. Then the **partial fraction decomposition** of $R$ is

$$
R(x) = \sum_{k=1}^{m} \frac{P_k(x)}{Q_k(x)}
$$

where $P_k$ is a polynomial and $Q_k$ is a power of an irreducible polynomial. (Here the degree of $Q_k$ is greater than the degree of $P_k$ for all $k$.)

> Partial fractions: $\dfrac{3}{x+2}$ and $\dfrac{3x+1}{x^2+1}$.
>
> Not a partial fraction: $\dfrac{-x+2}{x^2+3x+2} = \dfrac{-x+2}{(x+2)(x+1)}$, because the denominator is reducible.

## Methods

### Method 1: Integrating a rational function $\frac{P(x)}{Q(x)}$ (overall procedure)

**When to use:** the integrand is a ratio of polynomials.

1. **Check if it is proper.** If $\deg P \ge \deg Q$, use polynomial long division to write
   $$\frac{P(x)}{Q(x)} = S(x) + \frac{R(x)}{Q(x)}, \quad \deg R < \deg Q.$$
   Integrate the polynomial $S(x)$ directly (Example 7).
2. **Factor the denominator** $Q(x)$ completely into linear factors and irreducible quadratic factors.
3. **Guess the form** of the decomposition (Method 2).
4. **Clear denominators** by multiplying both sides by $Q(x)$ to get a polynomial identity.
5. **Solve for the unknown constants** (Method 3).
6. **Integrate each term** (Method 4).

### Method 2: Guessing the form

Each factor of the denominator contributes terms as follows (these are the forms used in the examples):

| Factor in denominator | Terms in the decomposition | Example |
|---|---|---|
| Simple linear factor $(x-a)$ | $\dfrac{A}{x-a}$ | #1 |
| Repeated linear factor $(x-a)^2$ | $\dfrac{A}{x-a} + \dfrac{B}{(x-a)^2}$ | #3 |
| Simple irreducible quadratic $ax^2+bx+c$ | $\dfrac{Bx+C}{ax^2+bx+c}$ | #5 |
| Repeated irreducible quadratic $(ax^2+bx+c)^2$ | $\dfrac{Bx+C}{ax^2+bx+c} + \dfrac{Dx+E}{(ax^2+bx+c)^2}$ | #8, #10 |

The numerator over each factor has degree one less than that irreducible factor: a constant over a linear factor, and a linear term over a quadratic factor. A repeated factor gets one term for each power up to its order.

### Method 3: Solving for the constants

After clearing denominators you have an identity that holds for **all** $x$. Two techniques, which you can mix:

- **Plug in roots.** Setting $x$ equal to a root of a linear factor makes every term containing that factor vanish, leaving one unknown. Fastest for simple linear factors (Examples 1, 3, 7, 9).
- **Plug in any other value.** If constants remain, choose any convenient $x$ that is not a root (Example 3 uses $x = 4$; Example 5 notes $x = 0$ or $x = -1$ would also work).
- **Equate coefficients.** Expand the right side, group by powers of $x$, and match each coefficient with the left side. Include missing powers on the left with coefficient 0, for example $3x - 1 = 0x^2 + 3x - 1$ (Examples 5, 8, 10).

### Method 4: Integrating each type of term

- $\displaystyle \int \frac{1}{x-a}\,dx = \ln|x-a| + C$.
- $\displaystyle \int \frac{1}{(x-a)^2}\,dx$: let $u = x-a$ to get $\int u^{-2}\,du = -\frac{1}{u} + C$.
- $\displaystyle \int \frac{\text{linear}}{x^2+bx+c}\,dx$ with irreducible denominator (Example 6):
  1. With $u = x^2+bx+c$, $du = (2x+b)\,dx$. Rewrite the numerator as $A(2x+b) + B$, splitting it into a "clean" $u$-sub piece and a constant piece.
  2. The piece $A\dfrac{2x+b}{x^2+bx+c}$ integrates to $A\ln|x^2+bx+c|$.
  3. For the constant piece, **complete the square**, factor out the constant so the denominator looks like $w^2 + 1$, substitute, and use $\int \frac{1}{w^2+1}\,dw = \arctan w + C$.

### Method 5: Substitutions that produce a rational function

**When to use:** the integrand is not rational but becomes rational after a substitution.

- **Radical $\sqrt{1+2x}$** (Example 9): let $u^2 = 2x+1$, so $x = \frac{u^2-1}{2}$ and $dx = u\,du$.
- **Exponentials $e^x, e^{2x}, e^{3x}$** (Example 10): let $u = e^x$, so $du = e^x\,dx$ and $e^{kx} = u^k$.

Then apply Method 1 in the new variable and substitute back at the end.

## Worked examples

### Example #1 (Simple roots)

Find the partial fraction decomposition of

$$
\frac{x-3}{(x-2)(x+4)}.
$$

**Step 1: Guess the form.** The denominator has two distinct linear factors, so

$$
\frac{x-3}{(x-2)(x+4)} = \frac{A}{x-2} + \frac{B}{x+4} = \frac{A(x+4)}{(x-2)(x+4)} + \frac{B(x-2)}{(x-2)(x+4)}.
$$

**Step 2: Clear denominators.** The denominators now match, so the numerators must be equal:

$$
x - 3 = A(x+4) + B(x-2).
$$

**Step 3: Plug in $x = 2$** (kills the $B$ term):

$$
2 - 3 = A \cdot 6 + B \cdot 0 \implies -1 = 6A \implies A = -\frac{1}{6}.
$$

**Step 4: Plug in $x = -4$** (kills the $A$ term):

$$
-4 - 3 = A \cdot 0 + B(-4-2) \implies -7 = -6B \implies B = \frac{7}{6}.
$$

**Result:**

$$
\frac{x-3}{(x-2)(x+4)} = -\frac{1}{6}\cdot\frac{1}{x-2} + \frac{7}{6}\cdot\frac{1}{x+4},
$$

which is easy to integrate.

---

### Example #2

Find

$$
\int \frac{x-3}{(x-2)(x+4)}\,dx.
$$

Use the decomposition from Example 1 and linearity of the integral:

$$
\int \frac{x-3}{(x-2)(x+4)}\,dx = -\frac{1}{6}\int \frac{1}{x-2}\,dx + \frac{7}{6}\int \frac{1}{x+4}\,dx.
$$

Each piece is $\int \frac{1}{x-a}\,dx = \ln|x-a|$:

$$
= -\frac{1}{6}\ln|x-2| + \frac{7}{6}\ln|x+4| + C.
$$

---

### Example #3 (Repeated roots)

Find the partial fraction decomposition of

$$
\frac{x-4}{(x-2)(x+4)^2}.
$$

**Step 1: Guess the form.** $(x-2)$ is simple, and $(x+4)$ has order 2, so it gets one term for each power:

$$
\frac{x-4}{(x-2)(x+4)^2} = \frac{A}{x-2} + \frac{B}{x+4} + \frac{C}{(x+4)^2}.
$$

**Step 2: Clear denominators.** Multiply both sides by $(x-2)(x+4)^2$:

$$
x - 4 = A(x+4)^2 + B(x-2)(x+4) + C(x-2).
$$

**Step 3: Plug in $x = 2$.** Then $x+4 = 6$ and $x-2 = 0$:

$$
-2 = 36A + 0 + 0 \implies A = -\frac{1}{18}.
$$

**Step 4: Plug in $x = -4$.** Then $x+4 = 0$ and $x - 2 = -6$:

$$
-8 = 0 + 0 - 6C \implies C = \frac{4}{3}.
$$

**Step 5: Find $B$.** No root isolates $B$, so pick any $x$ other than $-4$ or $2$. Choose $x = 4$: the left side is $4 - 4 = 0$, $(x+4)^2 = 64$, $(x-2)(x+4) = 2 \cdot 8 = 16$, and $x - 2 = 2$:

$$
0 = 64A + 16B + 2C \implies 16B = -64A - 2C.
$$

Substitute $A = -\frac{1}{18}$ and $C = \frac{4}{3}$:

$$
B = \frac{1}{16}\left(64\cdot\frac{1}{18} - \frac{8}{3}\right) = \frac{1}{16}\left(\frac{32}{9} - \frac{24}{9}\right) = \frac{1}{16}\cdot\frac{8}{9} = \frac{1}{18}.
$$

**Result:**

$$
\frac{x-4}{(x-2)(x+4)^2} = -\frac{1}{18}\cdot\frac{1}{x-2} + \frac{1}{18}\cdot\frac{1}{x+4} + \frac{4}{3}\cdot\frac{1}{(x+4)^2},
$$

which is easy to integrate.

---

### Example #4

Find

$$
\int \frac{x-4}{(x-2)(x+4)^2}\,dx.
$$

Use the decomposition from Example 3:

$$
\int \frac{x-4}{(x-2)(x+4)^2}\,dx = -\frac{1}{18}\int\frac{1}{x-2}\,dx + \frac{1}{18}\int\frac{1}{x+4}\,dx + \frac{4}{3}\int\frac{1}{(x+4)^2}\,dx.
$$

The first two are logs. For the third, let $u = x+4$, $du = dx$:

$$
\int \frac{1}{(x+4)^2}\,dx = \int \frac{1}{u^2}\,du = -\frac{1}{u} + C = -\frac{1}{x+4} + C.
$$

So

$$
\int \frac{x-4}{(x-2)(x+4)^2}\,dx = -\frac{1}{18}\ln|x-2| + \frac{1}{18}\ln|x+4| - \frac{4}{3}\cdot\frac{1}{x+4} + C.
$$

---

### Example #5 (Simple irreducible factor)

Find the partial fraction decomposition of

$$
\frac{3x-1}{(x-1)(x^2+2x+5)}.
$$

**Step 1: Guess the form.** $x^2+2x+5$ is irreducible (completing the square gives $(x+1)^2+4 > 0$, so it has no real roots). Over an irreducible quadratic, the numerator is linear:

$$
\frac{3x-1}{(x-1)(x^2+2x+5)} = \frac{A}{x-1} + \frac{Bx+C}{x^2+2x+5}.
$$

**Step 2: Clear denominators and expand:**

$$
3x - 1 = A(x^2+2x+5) + (Bx+C)(x-1) = A(x^2+2x+5) + B(x^2-x) + C(x-1).
$$

**Step 3: Plug in $x = 1$.** Then $x^2+2x+5 = 8$ and $x - 1 = 0$:

$$
2 = 8A \implies A = \frac{1}{4}.
$$

**Step 4: Equate coefficients.** Write the left side with every power showing:

$$
0x^2 + 3x - 1 = A(x^2+2x+5) + B(x^2-x) + C(x-1).
$$

- $x^2$ terms: $0 = A + B \implies B = -\frac{1}{4}$.
- Constant terms: $-1 = 5A - C \implies C = 5A + 1 = \frac{5}{4} + 1 = \frac{9}{4}$.

(Plugging in $x = 0$ or $x = -1$ would also give equations for $B$ and $C$.)

Check with the $x$ terms: $2A - B + C = \frac{2}{4} + \frac{1}{4} + \frac{9}{4} = 3$. ✓

**Result:** Since $Bx + C = -\frac{1}{4}x + \frac{9}{4} = \frac{1}{4}(-x+9)$,

$$
\frac{3x-1}{(x-1)(x^2+2x+5)} = \frac{1}{4}\cdot\frac{1}{x-1} + \frac{1}{4}\cdot\frac{-x+9}{x^2+2x+5}.
$$

The first term is a log. The second term is harder to integrate, and Example 6 handles it.

---

### Example #6

Find

$$
\int \frac{-x+9}{x^2+2x+5}\,dx.
$$

**Step 1: Why a direct $u$-sub fails.** With $u = x^2+2x+5$, $du = (2x+2)\,dx$. The numerator is not a multiple of $2x+2$:

$$
A(2x+2) \neq -x + 9 \quad \text{for all } A \in \mathbf{R}.
$$

**Step 2: Split off the bad terms.** Write $-x + 9 = A(2x+2) + B$ for a "clean" $u$-sub. Matching $x$ coefficients: $2A = -1$, so $A = -\frac{1}{2}$. Matching constants: $2A + B = 9$, so $B = 9 + 1 = 10$. So

$$
-x + 9 = -\frac{1}{2}(2x+2) + 10,
$$

and

$$
\frac{-x+9}{x^2+2x+5} = -\frac{1}{2}\cdot\frac{2x+2}{x^2+2x+5} + \frac{10}{x^2+2x+5}.
$$

**Step 3: The $u$-sub piece.** With $u = x^2+2x+5$, $du = (2x+2)\,dx$:

$$
-\frac{1}{2}\int\frac{2x+2}{x^2+2x+5}\,dx = -\frac{1}{2}\int\frac{1}{u}\,du = -\frac{1}{2}\ln|x^2+2x+5| + C_2.
$$

**Step 4: The constant piece. Complete the square:**

$$
(x+1)^2 = x^2 + 2x + 1 \implies (x+1)^2 + 4 = x^2 + 2x + 5.
$$

Then factor out the 4 so the denominator has the form $w^2 + 1$:

$$
\int\frac{10}{x^2+2x+5}\,dx = 10\int\frac{1}{(x+1)^2+4}\,dx = \frac{10}{4}\int\frac{1}{\left(\frac{x+1}{2}\right)^2+1}\,dx.
$$

**Step 5: Substitute** $w = \frac{x+1}{2}$, so $dw = \frac{1}{2}\,dx$ and $dx = 2\,dw$:

$$
\int\frac{10}{x^2+2x+5}\,dx = \frac{10}{4}\cdot 2\int\frac{1}{w^2+1}\,dw = 5\arctan(w) + C_1 = 5\arctan\left(\frac{x+1}{2}\right) + C_1.
$$

**Step 6: Combine:**

$$
\int \frac{-x+9}{x^2+2x+5}\,dx = -\frac{1}{2}\ln|x^2+2x+5| + 5\arctan\left(\frac{x+1}{2}\right) + C.
$$

---

### Example #7 (Improper rational function)

Find

$$
\int \frac{2x^2+5x+6}{x^2+3x+2}\,dx.
$$

**Step 1: Long division.** The numerator and denominator both have degree 2, so the fraction is improper. Divide $2x^2+5x+6$ by $x^2+3x+2$:

- $x^2$ goes into $2x^2$ two times. Quotient: $2$.
- Subtract $2(x^2+3x+2) = 2x^2+6x+4$: $(2x^2+5x+6) - (2x^2+6x+4) = -x + 2$.
- The remainder $-x+2$ has degree less than 2, so stop.

This says

$$
2(x^2+3x+2) + (-x+2) = 2x^2+5x+6 \implies \frac{2x^2+5x+6}{x^2+3x+2} = 2 + \frac{-x+2}{x^2+3x+2}.
$$

**Step 2: Decompose the proper part.** Factor $x^2 + 3x + 2 = (x+1)(x+2)$:

$$
\frac{-x+2}{(x+1)(x+2)} = \frac{A}{x+1} + \frac{B}{x+2} \implies -x+2 = A(x+2) + B(x+1).
$$

- $x = -1$: $1 + 2 = A(1) \implies A = 3$.
- $x = -2$: $2 + 2 = B(-1) \implies B = -4$.

**Step 3: Integrate the proper part:**

$$
\int\frac{-x+2}{(x+1)(x+2)}\,dx = 3\int\frac{1}{x+1}\,dx - 4\int\frac{1}{x+2}\,dx = 3\ln|x+1| - 4\ln|x+2| + C.
$$

**Step 4: Put it together:**

$$
\int\frac{2x^2+5x+6}{x^2+3x+2}\,dx = \int\left(2 + \frac{-x+2}{(x+1)(x+2)}\right)dx = 2x + 3\ln|x+1| - 4\ln|x+2| + C.
$$

---

### Example #8 (Repeated irreducible factor)

Find the partial fraction decomposition of

$$
\frac{1}{(x+1)(x^2+2x+2)^2}.
$$

**Step 1: Guess the form.** $x^2+2x+2 = (x+1)^2 + 1$ is irreducible and appears squared, so it gets two terms, each with a linear numerator:

$$
\frac{1}{(x+1)(x^2+2x+2)^2} = \frac{A}{x+1} + \frac{Bx+C}{x^2+2x+2} + \frac{Dx+E}{(x^2+2x+2)^2}.
$$

**Step 2: Clear denominators:**

$$
1 = A(x^2+2x+2)^2 + (Bx+C)(x^2+2x+2)(x+1) + (Dx+E)(x+1).
$$

**Step 3: Plug in $x = -1$.** Then $x^2 + 2x + 2 = 1 - 2 + 2 = 1$ and $x + 1 = 0$:

$$
1 = A(1-2+2)^2 \implies A = 1.
$$

**Step 4: Expand.** Using $(x^2+2x+2)^2 = x^4+4x^3+8x^2+8x+4$ and $(x^2+2x+2)(x+1) = x^3+3x^2+4x+2$:

$$
\begin{aligned}
1 = {} & A(x^4+4x^3+8x^2+8x+4) \\
&+ B(x^4+3x^3+4x^2+2x) \\
&+ C(x^3+3x^2+4x+2) \\
&+ D(x^2+x) \\
&+ E(x+1).
\end{aligned}
$$

**Step 5: Equate coefficients.** The left side is $1$, so every non-constant coefficient is $0$.

- $x^4$: $0 = A + B \implies B = -1$.
- $x^3$: $0 = 4A + 3B + C = 4 - 3 + C = 1 + C \implies C = -1$.
- $x^2$: $0 = 8A + 4B + 3C + D = 8 - 4 - 3 + D \implies D = -1$.
- Constant: $1 = 4A + 2C + E \implies E = 1 - 4 - 2C = -2C - 3 = 2 - 3 = -1$.

Check with the $x$ terms: $8A + 2B + 4C + D + E = 8 - 2 - 4 - 1 - 1 = 0$. ✓

**Result:**

$$
\frac{1}{(x+1)(x^2+2x+2)^2} = \frac{1}{x+1} + \frac{-x-1}{x^2+2x+2} + \frac{-x-1}{(x^2+2x+2)^2}.
$$

---

### Example #9

Find

$$
\int \frac{1}{x\sqrt{1+2x}}\,dx.
$$

**Step 1: Rationalizing substitution.** Let $u^2 = 2x + 1$, so $u = (1+2x)^{1/2}$. Differentiating: $2u\,du = 2\,dx$, so $dx = u\,du$. Solving for $x$: $x = \frac{u^2-1}{2}$, so $\frac{1}{x} = \frac{2}{u^2-1}$.

**Step 2: Rewrite the integral:**

$$
\int\frac{1}{x(1+2x)^{1/2}}\,dx = \int\frac{2}{u^2-1}\cdot\frac{u}{u}\,du = \int\frac{2}{(u-1)(u+1)}\,du.
$$

Here $\frac{1}{u}$ comes from $\frac{1}{\sqrt{1+2x}}$ and the $u$ in the numerator comes from $dx = u\,du$.

**Step 3: Partial fractions:**

$$
\frac{2}{(u-1)(u+1)} = \frac{A}{u-1} + \frac{B}{u+1} \implies 2 = A(u+1) + B(u-1).
$$

- $u = 1$: $2 = 2A \implies A = 1$.
- $u = -1$: $2 = -2B \implies B = -1$.

**Step 4: Integrate:**

$$
\int\frac{2}{(u-1)(u+1)}\,du = \int\frac{1}{u-1}\,du - \int\frac{1}{u+1}\,du = \ln|u-1| - \ln|u+1| + C = \ln\left|\frac{u-1}{u+1}\right| + C.
$$

**Step 5: Substitute back** $u = (1+2x)^{1/2}$:

$$
\int\frac{1}{x(1+2x)^{1/2}}\,dx = \ln\left|\frac{(1+2x)^{1/2}-1}{(1+2x)^{1/2}+1}\right| + C.
$$

---

### Example #10

Find

$$
\int\frac{e^{3x}+e^{2x}+e^x}{(e^{2x}+1)^2}\,dx.
$$

**Step 1: Substitute** $u = e^x$, $du = e^x\,dx$. Then $e^{3x} = u^3$, $e^{2x} = u^2$, and $dx = \frac{du}{u}$, so

$$
(e^{3x}+e^{2x}+e^x)\,dx = (u^3+u^2+u)\,dx = (u^2+u+1)\,du.
$$

This gives

$$
\int\frac{e^{3x}+e^{2x}+e^x}{(e^{2x}+1)^2}\,dx = \int\frac{u^2+u+1}{(u^2+1)^2}\,du.
$$

**Step 2: Guess the form.** $u^2+1$ is irreducible and squared:

$$
\frac{u^2+u+1}{(u^2+1)^2} = \frac{Au+B}{u^2+1} + \frac{Cu+D}{(u^2+1)^2} \implies u^2+u+1 = (Au+B)(u^2+1) + Cu + D.
$$

**Step 3: Equate coefficients.** Expanding the right side gives $Au^3 + Bu^2 + (A+C)u + (B+D)$.

- $u^3$: the left side has no $u^3$ term, so $A = 0$. The system simplifies to $u^2+u+1 = B(u^2+1) + Cu + D$.
- $u^2$: $B = 1$.
- $u^1$: $C = 1$.
- Constant: $B + D = 1 \implies D = 0$.

So $\dfrac{u^2+u+1}{(u^2+1)^2} = \dfrac{1}{u^2+1} + \dfrac{u}{(u^2+1)^2}$.

**Step 4: Integrate.** The first piece is an arctangent. For the second, let $v = u^2+1$, $dv = 2u\,du$, so $\int\frac{u}{(u^2+1)^2}\,du = \frac{1}{2}\int v^{-2}\,dv = -\frac{1}{2}v^{-1}$:

$$
\int\frac{u^2+u+1}{(u^2+1)^2}\,du = \int\frac{1}{u^2+1}\,du + \int\frac{u}{(u^2+1)^2}\,du = \arctan(u) - \frac{1}{2}(u^2+1)^{-1} + C.
$$

**Step 5: Substitute back** $u = e^x$, so $u^2 = e^{2x}$:

$$
\int\frac{e^{3x}+e^{2x}+e^x}{(e^{2x}+1)^2}\,dx = \arctan(e^x) - \frac{1}{2}(e^{2x}+1)^{-1} + C.
$$

## Common mistakes and tips

- **Divide first if the fraction is improper.** Partial fractions only applies to proper rational functions. If $\deg P \ge \deg Q$, do long division first (Example 7).
- **Factor completely.** $\frac{-x+2}{x^2+3x+2}$ is not a partial fraction because $x^2+3x+2 = (x+1)(x+2)$ is reducible. Only powers of irreducible polynomials may appear as denominators.
- **Check for irreducibility.** A quadratic with a real root is reducible. Completing the square is a quick test: $(x+1)^2 + 4$ is never zero, so $x^2 + 2x + 5$ is irreducible.
- **Repeated factors need every power.** $(x+4)^2$ gives $\frac{B}{x+4} + \frac{C}{(x+4)^2}$, not just the squared term. The same applies to repeated quadratics (Examples 8, 10).
- **Irreducible quadratics need a linear numerator.** Write $\frac{Bx+C}{ax^2+bx+c}$, not $\frac{B}{ax^2+bx+c}$.
- **Plug in roots first.** Each root of a linear factor pins down one constant immediately. Then use another convenient $x$ value or equate coefficients for the rest.
- **Write missing powers as zero** when you equate coefficients: $3x - 1 = 0x^2 + 3x - 1$.
- **Check your constants** with an equation you have not used yet, such as a coefficient you skipped.
- **A quadratic over an irreducible quadratic is not an immediate $u$-sub.** Rewrite the numerator as $A(\text{derivative of the denominator}) + B$, then use $\ln$ for the first part and complete the square plus $\arctan$ for the second (Example 6).
- **Remember the chain factor in substitutions.** In Example 6, $w = \frac{x+1}{2}$ gives $dx = 2\,dw$, which turns $\frac{10}{4}$ into $5$.
- **Substitute back** at the end of Examples 9 and 10 so the answer is in terms of $x$.
- **Combine logs if you like:** $\ln|u-1| - \ln|u+1| = \ln\left|\frac{u-1}{u+1}\right|$.

## Formula sheet

**Decomposition forms**

$$
\frac{\cdots}{(x-a)(x-b)} = \frac{A}{x-a} + \frac{B}{x-b}
$$

$$
\frac{\cdots}{(x-a)(x-b)^2} = \frac{A}{x-a} + \frac{B}{x-b} + \frac{C}{(x-b)^2}
$$

$$
\frac{\cdots}{(x-a)(ax^2+bx+c)} = \frac{A}{x-a} + \frac{Bx+C}{ax^2+bx+c}
$$

$$
\frac{\cdots}{(x-a)(ax^2+bx+c)^2} = \frac{A}{x-a} + \frac{Bx+C}{ax^2+bx+c} + \frac{Dx+E}{(ax^2+bx+c)^2}
$$

**Improper fractions (long division)**

$$
\frac{P(x)}{Q(x)} = S(x) + \frac{R(x)}{Q(x)}, \quad \deg R < \deg Q
$$

**Basic integrals**

$$
\int\frac{1}{x-a}\,dx = \ln|x-a| + C
$$

$$
\int\frac{1}{(x-a)^2}\,dx = -\frac{1}{x-a} + C
$$

$$
\int\frac{1}{w^2+1}\,dw = \arctan(w) + C
$$

$$
\int\frac{f'(x)}{f(x)}\,dx = \ln|f(x)| + C
$$

$$
\int\frac{u}{(u^2+1)^2}\,du = -\frac{1}{2}(u^2+1)^{-1} + C
$$

**Irreducible quadratic recipe (from Example 6)**

$$
\text{numerator} = A(2x+b) + B, \qquad x^2+bx+c = \left(x+\tfrac{b}{2}\right)^2 + k^2
$$

$$
\int\frac{1}{(x+1)^2+4}\,dx = \frac{1}{2}\arctan\left(\frac{x+1}{2}\right) + C \quad \text{(via } w = \tfrac{x+1}{2},\ dx = 2\,dw\text{)}
$$

**Substitutions that produce rational functions**

$$
\sqrt{1+2x}: \quad u^2 = 2x+1,\quad x = \frac{u^2-1}{2},\quad dx = u\,du
$$

$$
e^{kx}: \quad u = e^x,\quad du = e^x\,dx,\quad e^{kx} = u^k
$$
