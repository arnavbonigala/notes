# Section 7.8 - Improper Integrals

## Summary

A definite integral $\int_a^b f(x)\,dx$ normally needs a finite interval and a function that is defined everywhere on it. Improper integrals handle the two cases where one of these fails. Type I is an infinite interval, such as $[a,\infty)$, $(-\infty,b]$, or $\mathbf{R}$. Type II is a function that blows up to $\pm\infty$ at an endpoint or at an interior point. In both cases you integrate over a region where everything is fine, use the FTC, and then take a limit. The integral **converges** if the limit is a real number and **diverges** otherwise. If you cannot find an antiderivative, the **Comparison Test** lets you decide convergence or divergence by bounding the integrand with a simpler function whose integral you already know.

## Definitions and theorems

**Remark (when an ordinary integral exists).** For $\int_a^b f(x)\,dx$ to exist, we require:

1. $[a,b]$ to have finite length.
2. $f$ is defined everywhere on $[a,b]$, but $f$ need not be continuous on $[a,b]$.

**Remark (from §2.6, limits you will use constantly).** If $n>0$,

$$
\lim_{x\to\infty}\frac{1}{x^n}=0,
$$

and $n<0$ implies

$$
\lim_{x\to\infty}\frac{1}{x^n}=\infty.
$$

Also,

$$
\lim_{x\to\infty}e^x=\infty \quad\text{and}\quad \lim_{x\to\infty}e^{-x}=\lim_{x\to\infty}\frac{1}{e^x}=0.
$$

**Definition #1 (Improper Integral I).** If $\int_a^t f(x)\,dx$ exists for all $t\ge a$, then

$$
\int_a^\infty f(x)\,dx=\lim_{t\to\infty}\int_a^t f(x)\,dx
$$

is the **improper integral** of $f$ over $[a,\infty)$.

**Definition #2 (Improper Integral I).** If $\int_t^b f(x)\,dx$ exists for all $t\le b$, then

$$
\int_{-\infty}^b f(x)\,dx=\lim_{t\to-\infty}\int_t^b f(x)\,dx
$$

is the **improper integral** of $f$ over $(-\infty,b]$.

**Definition #3 (convergence and divergence).** Suppose

$$
I=\int_a^\infty f(x)\,dx\in\mathbf{R}.
$$

Then the improper integral **converges**. If $I\notin\mathbf{R}$, then the improper integral **diverges**. So if the limit is $\pm\infty$ or does not exist, the integral diverges.

**Warning.** We can have

$$
\lim_{x\to\infty}f(x)=0 \quad\text{and}\quad \int_0^\infty f(x)\,dx=\infty.
$$

(Example #3 shows this with $f(x)=1/x$.)

**Definition #4 (Improper Integral I, whole line).** If

$$
I_1=\int_{-\infty}^a f(x)\,dx \quad\text{and}\quad I_2=\int_a^\infty f(x)\,dx
$$

converge, then

$$
\int_{\mathbf{R}}f(x)\,dx=\int_{-\infty}^\infty f(x)\,dx=I_1+I_2.
$$

**Warning.** The terms

$$
\int_{\mathbf{R}}f(x)\,dx \quad\text{and}\quad \lim_{t\to\infty}\int_{-t}^t f(x)\,dx
$$

need **not** be equal. (Example #4 shows this with $\sin x$.)

**Note (from Example #4).** If $\int_{\mathbf{R}}f(x)\,dx$ converges, then

$$
\int_{\mathbf{R}}f(x)\,dx=\lim_{t\to\infty}\int_{-t}^t f(x)\,dx,
$$

since $\int_{\mathbf{R}}f(x)\,dx$ converges for all "paths" to $\infty$. The symmetric limit is safe to use only after you know the integral converges.

**Definition #5 (Improper Integral II, blow-up at the left endpoint).** Let $f$ be a continuous function on $(a,b]$ with

$$
\lim_{x\to a^+}f(x)=\pm\infty.
$$

Then

$$
\int_a^b f(x)\,dx=\lim_{t\to a^+}\int_t^b f(x)\,dx
$$

is the **improper integral** of $f$ over $[a,b]$.

**Definition #6 (Improper Integral II, blow-up at the right endpoint).** Let $f$ be a continuous function on $[a,b)$ with

$$
\lim_{x\to b^-}f(x)=\pm\infty.
$$

Then

$$
\int_a^b f(x)\,dx=\lim_{t\to b^-}\int_a^t f(x)\,dx
$$

is the **improper integral** of $f$ over $[a,b]$.

**Definition #7 (Improper Integral II, blow-up at an interior point).** Let $f$ be a continuous function on $[a,c)\cup(c,b]$ with $f$ unbounded at $c\in(a,b)$. If

$$
I_1=\int_a^c f(x)\,dx \quad\text{and}\quad I_2=\int_c^b f(x)\,dx
$$

converge, then the **improper integral**

$$
\int_a^b f(x)\,dx=I_1+I_2
$$

**converges**.

**Theorem #1 (Comparison Test).** Suppose $f$ and $g$ are continuous on the interval $[a,\infty)$ where

$$
0\le g(x)\le f(x)
$$

for all $x\ge a$. Then:

1. If $\int_a^\infty f(x)\,dx$ converges, then $\int_a^\infty g(x)\,dx$ converges.
2. If $\int_a^\infty g(x)\,dx$ diverges, then $\int_a^\infty f(x)\,dx$ diverges.

The functions must be nonnegative ($0\le g(x)\le f(x)$). If the smaller integral diverges, so must the larger integral. If the bigger integral converges, so must the smaller integral.

**Remark (standard integrals, which you need to prove on the exams).**

1. $\int_1^\infty 1/x^p\,dx$ converges if and only if $p>1$.
2. $\int_0^1 1/x^p\,dx$ converges if and only if $p<1$.
3. $\int_0^\infty 1/b^x\,dx$ converges for $b>1$.
4. $\int_{-\infty}^0 b^x\,dx$ converges for $b>1$.

Proofs are in the Methods section below.

## Methods

### Method 1: Infinite interval (Type I)

**When to use:** a limit of integration is $\infty$ or $-\infty$.

The idea from lecture: a proper integral needs a bounded interval, so "cheat" by using the FTC on $[a,t]$ and then let the interval grow.

1. Replace the infinite bound with a variable $t$:
   - $\int_a^\infty f\,dx \to \lim_{t\to\infty}\int_a^t f\,dx$
   - $\int_{-\infty}^b f\,dx \to \lim_{t\to-\infty}\int_t^b f\,dx$
2. Find an antiderivative $F$ using any technique (substitution, etc.).
3. Evaluate with the FTC: $\int_a^t f\,dx=F(t)-F(a)$.
4. Take the limit. A real number means it converges. $\pm\infty$ or no limit means it diverges.

### Method 2: Whole real line

**When to use:** $\int_{-\infty}^\infty f\,dx$ or $\int_{\mathbf{R}}f\,dx$.

1. Pick any split point $a$ (often $0$).
2. Evaluate $\int_{-\infty}^a f\,dx$ and $\int_a^\infty f\,dx$ separately with Method 1.
3. If **both** converge, the answer is their sum. If **either** diverges, the whole integral diverges.
4. Do not replace this with $\lim_{t\to\infty}\int_{-t}^t f\,dx$ unless you already know the integral converges.

### Method 3: Unbounded integrand (Type II)

**When to use:** $f$ blows up to $\pm\infty$ somewhere in $[a,b]$. Check where denominators vanish, where square roots of zero appear in denominators, and where $\tan$, $\ln$, and similar functions have asymptotes.

The idea from lecture: a proper integral needs a bounded function, so integrate up to a point near the blow-up and take a one-sided limit.

1. Locate the bad point.
2. If it is at the left endpoint $a$: $\lim_{t\to a^+}\int_t^b f\,dx=\lim_{t\to a^+}(F(b)-F(t))$.
3. If it is at the right endpoint $b$: $\lim_{t\to b^-}\int_a^t f\,dx=\lim_{t\to b^-}(F(t)-F(a))$.
4. If it is at an interior point $c$: split into $\int_a^c+\int_c^b$, handle each piece with steps 2 and 3, and require **both** pieces to converge.

### Method 4: Mixed types

**When to use:** the interval is infinite **and** the function blows up somewhere (for example $\int_{\mathbf{R}}1/x^2\,dx$).

1. Split the integral so that each piece has exactly one problem (one infinite bound or one blow-up point).
2. Evaluate each piece with its own limit.
3. It converges only if every piece converges. As soon as one piece diverges, you can stop.

### Method 5: Comparison Test

**When to use:** you cannot find (or do not need) an antiderivative, and you only want to know whether the integral converges.

1. Check that the integrand is nonnegative on the interval.
2. To show **convergence**, find a **bigger** $f$ with $0\le g\le f$ whose integral converges. Standard choices are $c/x^p$ with $p>1$.
3. To show **divergence**, find a **smaller** $g\ge 0$ with $g\le f$ whose integral diverges. A standard choice is $1/x$.
4. The inequality only needs to hold from some point on. If it holds for $x\ge c$, compare on $[c,\infty)$. The leftover piece on $[a,c]$ is a proper integral, so it is finite and does not change convergence or divergence.

### Proofs of the standard integrals (required on exams)

**1. $\int_1^\infty x^{-p}\,dx$ converges if and only if $p>1$.**

If $p=1$:

$$
\lim_{t\to\infty}\int_1^t\frac{1}{x}\,dx=\lim_{t\to\infty}(\ln t-\ln 1)=\infty,
$$

so it diverges (Example #3).

If $p\ne 1$:

$$
\int_1^t x^{-p}\,dx=\left[\frac{x^{1-p}}{1-p}\right]_{x=1}^{t}=\frac{t^{1-p}-1}{1-p}.
$$

If $p>1$, then $1-p<0$, so $t^{1-p}=1/t^{p-1}\to 0$ by the §2.6 remark. The limit is $\frac{-1}{1-p}=\frac{1}{p-1}$, so the integral converges.

If $p<1$, then $1-p>0$, so $t^{1-p}\to\infty$ and the integral diverges.

**2. $\int_0^1 x^{-p}\,dx$ converges if and only if $p<1$.**

For $p>0$, $x^{-p}$ blows up at $0^+$. (For $p\le 0$ the integral is proper and finite, which matches the claim.)

If $p=1$:

$$
\lim_{t\to 0^+}\int_t^1\frac{1}{x}\,dx=\lim_{t\to0^+}(\ln 1-\ln t)=\lim_{t\to0^+}(-\ln t)=\infty,
$$

so it diverges.

If $p\ne1$:

$$
\int_t^1 x^{-p}\,dx=\left[\frac{x^{1-p}}{1-p}\right]_{x=t}^{1}=\frac{1-t^{1-p}}{1-p}.
$$

If $p<1$, then $1-p>0$ and $t^{1-p}\to 0$ as $t\to0^+$. The limit is $\frac{1}{1-p}$, so the integral converges.

If $p>1$, then $t^{1-p}=1/t^{p-1}\to\infty$ as $t\to0^+$, so the integral diverges.

**3. $\int_0^\infty b^{-x}\,dx$ converges for $b>1$.**

Since $\frac{d}{dx}b^{-x}=-\ln(b)\,b^{-x}$,

$$
\int_0^t b^{-x}\,dx=\left[-\frac{b^{-x}}{\ln b}\right]_{x=0}^{t}=\frac{1-b^{-t}}{\ln b}.
$$

For $b>1$, $\ln b>0$ and $b^{-t}\to 0$ as $t\to\infty$, so the limit is $\frac{1}{\ln b}$. The integral converges.

**4. $\int_{-\infty}^0 b^x\,dx$ converges for $b>1$.**

$$
\int_t^0 b^x\,dx=\left[\frac{b^x}{\ln b}\right]_{x=t}^{0}=\frac{1-b^t}{\ln b}.
$$

For $b>1$, $b^t\to 0$ as $t\to-\infty$, so the limit is $\frac{1}{\ln b}$. The integral converges.

## Worked examples

### Example #1

Find $\displaystyle\int_0^\infty\frac{1}{x^2+1}\,dx$.

**Step 1: identify the type.** The upper bound is $\infty$, so this is Type I (Definition #1). The integrand is continuous everywhere, so $\int_0^t$ exists for every $t\ge 0$.

**Step 2: antiderivative.**

$$
\int\frac{1}{1+x^2}\,dx=\arctan(x)+C.
$$

**Step 3: write the limit and apply the FTC.**

$$
\lim_{t\to\infty}\int_0^t\frac{1}{1+x^2}\,dx=\lim_{t\to\infty}\big(\arctan(t)-\arctan(0)\big).
$$

**Step 4: evaluate the limit.** $\tan x$ has vertical asymptotes at $x=\pm\pi/2$, so its inverse $\arctan x$ has horizontal asymptotes at $y=\pm\pi/2$. So

$$
\lim_{t\to\infty}\arctan(t)=\frac{\pi}{2}\quad\text{and}\quad\arctan(0)=0.
$$

**Answer.**

$$
\int_0^\infty\frac{1}{1+x^2}\,dx=\frac{\pi}{2}.
$$

It converges.

### Example #2

Find $\displaystyle\int_2^\infty\frac{1}{x\ln(x)}\,dx$.

**Step 1: identify the type.** Type I (upper bound $\infty$). On $[2,\infty)$, $\ln x\ge\ln 2>0$, so the integrand is continuous there.

**Step 2: substitute.** Let $u=\ln(x)$ and $du=\frac{1}{x}\,dx$. Then

$$
\int\frac{1}{x\ln(x)}\,dx=\int\frac{1}{u}\,du=\ln(u)+C=\ln(\ln(x))+C.
$$

**Step 3: limit and FTC.**

$$
\begin{aligned}
\int_2^\infty\frac{1}{x\ln(x)}\,dx&=\lim_{t\to\infty}\int_2^t\frac{1}{x\ln(x)}\,dx\\
&=\lim_{t\to\infty}\ln(\ln(x))\Big|_{x=2}^{t}\\
&=\lim_{t\to\infty}\big(\ln(\ln(t))-\ln(\ln(2))\big)\\
&=\infty,
\end{aligned}
$$

**Step 4: justify the limit.** $\lim_{t\to\infty}\ln(t)=\infty$, so $\ln(\ln t)$ is $\ln$ of something going to $\infty$, which also goes to $\infty$. The term $\ln(\ln 2)$ is a fixed constant.

**Answer.** The integral diverges (to $\infty$).

### Example #3

Find $\displaystyle\int_1^\infty\frac{1}{x}\,dx$.

**Step 1: identify the type.** Type I (Definition #1).

**Step 2: limit and FTC.**

$$
\begin{aligned}
\int_1^\infty\frac{1}{x}\,dx&=\lim_{t\to\infty}\int_1^t\frac{1}{x}\,dx\\
&=\lim_{t\to\infty}\ln(x)\Big|_{x=1}^{t}\\
&=\lim_{t\to\infty}\big(\ln(t)-\ln(1)\big)\\
&=\infty,
\end{aligned}
$$

since $\ln(1)=0$ and $\lim_{t\to\infty}\ln(t)=\infty$.

**Answer.**

$$
\int_1^\infty\frac{1}{x}\,dx=\infty,
$$

so it diverges. However,

$$
\lim_{x\to\infty}\frac{1}{x}=0.
$$

This is the example behind the Warning: the integrand going to $0$ does **not** guarantee convergence.

### Example #4

Find $\displaystyle\int_{\mathbf{R}}\sin(x)\,dx$.

**Step 1: identify the type.** Whole real line, so by Definition #4 both halves must converge.

**Step 2: test the right half.**

$$
\lim_{t\to\infty}\int_0^t\sin(x)\,dx=\lim_{t\to\infty}\big(-\cos(x)\big)\Big|_{x=0}^{t}=\lim_{t\to\infty}\big(-\cos(t)+\cos(0)\big)=\lim_{t\to\infty}\big(1-\cos(t)\big). \quad (*)
$$

This diverges since $\lim_{t\to\infty}\cos(t)$ does not exist ($\cos t$ keeps oscillating between $-1$ and $1$).

**Step 3: the tempting symmetric limit.** Sine is odd, so for all $t>0$,

$$
\int_{-t}^t\sin(x)\,dx=0 \quad\Rightarrow\quad \lim_{t\to\infty}\int_{-t}^t\sin(x)\,dx=0.
$$

(Explicitly: $\int_{-t}^t\sin x\,dx=-\cos t+\cos(-t)=-\cos t+\cos t=0$.)

**Step 4: conclude.** But $\int_0^\infty\sin(x)\,dx$ diverges, so

$$
\int_{\mathbf{R}}\sin(x)\,dx \text{ diverges}
$$

by $(*)$. The symmetric limit gives $0$, the real improper integral diverges, and this is the Warning after Definition #4 in action.

**Note.** If $\int_{\mathbf{R}}f(x)\,dx$ converges, then $\int_{\mathbf{R}}f(x)\,dx=\lim_{t\to\infty}\int_{-t}^tf(x)\,dx$, since a convergent integral converges for all "paths" to $\infty$.

### Example #5

Find $\displaystyle\int_{-1/2}^{1}\frac{1}{\sqrt{2x+1}}\,dx$.

**Step 1: identify the type.** $f(x)=\frac{1}{\sqrt{2x+1}}$ is undefined at $x=-1/2$ (the denominator is $\sqrt 0=0$) and

$$
\lim_{x\to-1/2^+}f(x)=\infty.
$$

So this is Type II with the problem at the left endpoint (Definition #5).

**Step 2: antiderivative.**

$$
\int(2x+1)^{-1/2}\,dx=(2x+1)^{1/2}+C.
$$

Check: $\frac{d}{dx}(2x+1)^{1/2}=\frac12(2x+1)^{-1/2}\cdot 2=(2x+1)^{-1/2}$. (Equivalently, substitute $u=2x+1$, $du=2\,dx$: $\frac12\int u^{-1/2}du=u^{1/2}+C$.)

**Step 3: limit and FTC.**

$$
\lim_{t\to-1/2^+}\int_t^1 f(x)\,dx=\lim_{t\to-1/2^+}\Big((2+1)^{1/2}-(2t+1)^{1/2}\Big).
$$

**Step 4: evaluate.** As $t\to-1/2^+$, $2t+1\to 0^+$, so

$$
\lim_{t\to-1/2^+}(2t+1)^{1/2}=0.
$$

**Answer.**

$$
\int_{-1/2}^{1}(2x+1)^{-1/2}\,dx=\sqrt3-0=\sqrt3.
$$

It converges.

### Example #6

Find $\displaystyle\int_0^{\pi/2}\tan(t)\,dt$.

**Step 1: identify the type.** $\tan x\to\infty$ as $x\to\pi/2^-$, so this is Type II with the problem at the right endpoint (Definition #6).

**Step 2: antiderivative.** With $u=\cos(x)$, $du=-\sin(x)\,dx$:

$$
\begin{aligned}
\int\tan(x)\,dx&=\int\frac{\sin(x)}{\cos(x)}\,dx\\
&=\int-\frac{1}{u}\,du\\
&=-\ln|\cos(x)|+C\\
&=\ln|\sec(x)|+C.
\end{aligned}
$$

(The last step uses $-\ln|\cos x|=\ln|\cos x|^{-1}=\ln|\sec x|$.)

**Step 3: limit and FTC.**

$$
\lim_{t\to\pi/2^-}\int_0^t f(x)\,dx=\lim_{t\to\pi/2^-}\Big(-\ln|\cos(t)|-\big(-\ln|\cos(0)|\big)\Big).
$$

The second term is $-\ln|\cos 0|=-\ln 1=0$.

**Step 4: evaluate.**

$$
\lim_{t\to\pi/2^-}\cos(t)=0 \quad\text{and}\quad \lim_{x\to0}\ln|x|=-\infty
\quad\Rightarrow\quad \lim_{t\to\pi/2^-}-\ln|\cos(t)|=\infty.
$$

**Answer.**

$$
\int_0^{\pi/2}\tan(t)\,dt=\infty.
$$

It diverges.

### Example #7 (Two Types)

Find $\displaystyle\int_{\mathbf{R}}\frac{1}{x^2}\,dx$.

**Step 1: identify the problems.** The interval is infinite in both directions (Type I), and $1/x^2$ blows up at $x=0$ (Type II). Split so each piece has one problem, for example $\int_{-\infty}^0+\int_0^1+\int_1^\infty$. If any piece diverges, the whole integral diverges.

**Step 2: antiderivative.**

$$
\int x^{-2}\,dx=-\frac{1}{x}+C.
$$

**Step 3: the piece $[1,\infty)$.**

$$
\lim_{t\to\infty}\int_1^t f(x)\,dx=\lim_{t\to\infty}\left(-\frac1t-\left(-\frac11\right)\right)=\lim_{t\to\infty}\left(-\frac1t+1\right)=1
$$

since $\lim_{x\to\infty}-\frac1x=0$. So

$$
\int_1^\infty x^{-2}\,dx=1.
$$

This piece converges.

**Step 4: the piece $(0,1]$.**

$$
\lim_{t\to0^+}\int_t^1 f(x)\,dx=\lim_{t\to0^+}\left(-\frac11-\left(-\frac1t\right)\right)=\lim_{t\to0^+}\left(-1+\frac1t\right).
$$

But $\lim_{x\to0^+}\frac1x=\infty$, so

$$
\int_0^1\frac{1}{x^2}\,dx=\infty.
$$

**Step 5: conclude.** Hence

$$
\int_0^\infty\frac{1}{x^2}\,dx=\int_0^1\frac{1}{x^2}\,dx+\int_1^\infty\frac{1}{x^2}\,dx=\infty+1=\infty
\quad\Rightarrow\quad \int_{\mathbf{R}}\frac{1}{x^2}\,dx \text{ diverges}.
$$

There is no need to check $(-\infty,0]$: one divergent piece is enough.

### Example #8

Find $\displaystyle\int_1^\infty\frac{\ln^2(x)}{x}\,dx$.

**Solution A: Comparison Test.**

Let

$$
f(x)=\frac{\ln^2(x)}{x},\qquad g(x)=\frac1x.
$$

Then

$$
0<g(x)\le f(x)\quad\text{for } x\ge e,
$$

since $\ln(x)\ge1$ when $x\ge e$. This gives $\ln^2(x)\ge 1$, and dividing by $x>0$ gives $\frac{\ln^2 x}{x}\ge\frac1x$.

We have

$$
\int_e^\infty\frac1x\,dx=\infty\quad\Rightarrow\quad\int_e^\infty\frac{\ln^2(x)}{x}\,dx=\infty
$$

by the Comparison Test (part 2: the smaller integral diverges, so the larger one does). The first integral diverges because $\lim_{t\to\infty}(\ln t-\ln e)=\lim_{t\to\infty}(\ln t-1)=\infty$.

Since $\int_e^\infty\frac{\ln^2(x)}{x}\,dx=\infty$, it follows that

$$
\int_1^\infty\frac{\ln^2(x)}{x}\,dx=\infty.
$$

The reason: $\int_1^\infty=\int_1^e+\int_e^\infty$, and $\int_1^e$ is a proper integral of a continuous function on a finite interval, so it is a finite number. Finite plus $\infty$ is $\infty$.

**Solution B: directly.** For $\frac{\ln^2(x)}{x}$, use $u=\ln(x)$, $du=\frac1x\,dx$:

$$
\int\frac{\ln^2(x)}{x}\,dx=\int u^2\,du=\frac13u^3+C=\frac13\ln^3(x)+C.
$$

Since

$$
\lim_{x\to\infty}\frac13\ln^3(x)=\infty,
$$

we have

$$
\int_1^\infty\frac{\ln^2(x)}{x}\,dx=\lim_{t\to\infty}\int_1^t\frac{\ln^2(x)}{x}\,dx=\lim_{t\to\infty}\left(\frac13\ln^3(t)-\frac13\ln^3(1)\right)=\infty,
$$

using $\ln 1=0$.

**Answer.** The integral diverges.

### Example #9

Find $\displaystyle\int_1^\infty\frac{\sin(x)+2}{x^2}\,dx$.

**Step 1: bound the integrand.** Since $-1\le\sin x\le1$,

$$
0<\sin(x)+2\le1+2=3.
$$

(The lower bound holds because $\sin x+2\ge-1+2=1>0$.) Dividing by $x^2>0$:

$$
\Rightarrow\quad 0<\frac{\sin(x)+2}{x^2}\le\frac{3}{x^2}.
$$

**Step 2: the bigger integral converges.** From Example #7,

$$
\int_1^\infty x^{-2}\,dx=1\quad\Rightarrow\quad\int_1^\infty\frac{3}{x^2}\,dx=3.
$$

**Step 3: conclude.**

$$
\Rightarrow\quad\int_1^\infty\frac{\sin(x)+2}{x^2}\,dx\ \text{converges}
$$

by the Comparison Test (part 1: the bigger integral converges, so the smaller one does). The test tells you that it converges, and it does not give you the value.

## Common mistakes and tips

- **Always write the limit.** Writing $F(\infty)$ or plugging in the bad point directly is not a valid step. Replace the problem spot with $t$ and take a limit.
- **Use one-sided limits for Type II.** Use $t\to a^+$ at a left-endpoint blow-up and $t\to b^-$ at a right-endpoint blow-up.
- **The integrand going to $0$ does not mean convergence.** $\lim_{x\to\infty}1/x=0$, but $\int_1^\infty 1/x\,dx=\infty$.
- **Do not use the symmetric limit for $\int_{\mathbf{R}}$.** $\lim_{t\to\infty}\int_{-t}^t\sin x\,dx=0$, but $\int_{\mathbf{R}}\sin x\,dx$ diverges. Split at a point and require both halves to converge. The symmetric limit is only valid once convergence is known.
- **Watch for hidden blow-ups inside the interval.** $\int_{\mathbf{R}}1/x^2\,dx$ looks like a Type I problem, but $x=0$ is also a Type II problem. If you ignore it, you will get a wrong finite answer.
- **Every piece must converge.** With Definition #4 or #7, a single divergent piece makes the whole integral diverge, so you can stop as soon as you find one.
- **The Comparison Test requires nonnegative functions.** Check $0\le g\le f$ before you use it.
- **Use the right direction of comparison.** Bigger converges means smaller converges. Smaller diverges means bigger diverges. "Smaller than something divergent" and "bigger than something convergent" tell you nothing.
- **Comparing from some point onward is fine.** In Example #8 the inequality only holds for $x\ge e$. The piece on $[1,e]$ is a proper integral and is finite, so it does not affect the conclusion.
- **"Diverges" includes "does not exist".** $\lim\cos t$ has no limit, which counts as divergence the same way $\infty$ does.
- **Be ready to prove the four standard results on exams** (the Remark at the end of the slides).

## Formula sheet

**Type I (infinite interval):**

$$
\int_a^\infty f\,dx=\lim_{t\to\infty}\int_a^t f\,dx,\qquad
\int_{-\infty}^b f\,dx=\lim_{t\to-\infty}\int_t^b f\,dx
$$

$$
\int_{-\infty}^\infty f\,dx=\int_{-\infty}^a f\,dx+\int_a^\infty f\,dx\quad\text{(both must converge)}
$$

**Type II (unbounded integrand):**

$$
\text{blow-up at } a:\ \int_a^b f\,dx=\lim_{t\to a^+}\int_t^b f\,dx,\qquad
\text{blow-up at } b:\ \int_a^b f\,dx=\lim_{t\to b^-}\int_a^t f\,dx
$$

$$
\text{blow-up at } c\in(a,b):\ \int_a^b f\,dx=\int_a^c f\,dx+\int_c^b f\,dx\quad\text{(both must converge)}
$$

**Comparison Test** ($0\le g\le f$ on $[a,\infty)$):

$$
\int_a^\infty f \text{ converges}\Rightarrow\int_a^\infty g\text{ converges},\qquad
\int_a^\infty g\text{ diverges}\Rightarrow\int_a^\infty f\text{ diverges}
$$

**Standard integrals:**

$$
\int_1^\infty\frac{1}{x^p}\,dx\ \text{converges}\iff p>1\quad\left(\text{value }\tfrac{1}{p-1}\right)
$$

$$
\int_0^1\frac{1}{x^p}\,dx\ \text{converges}\iff p<1\quad\left(\text{value }\tfrac{1}{1-p}\right)
$$

$$
\int_0^\infty\frac{1}{b^x}\,dx=\frac{1}{\ln b},\qquad\int_{-\infty}^0 b^x\,dx=\frac{1}{\ln b}\qquad(b>1)
$$

**Limits to know:**

$$
\lim_{x\to\infty}\frac{1}{x^n}=0\ (n>0),\quad \lim_{x\to\infty}e^{-x}=0,\quad \lim_{t\to\infty}\arctan t=\frac{\pi}{2},\quad \lim_{t\to\infty}\ln t=\infty,\quad \lim_{x\to0^+}\ln x=-\infty
$$

**Antiderivatives used in this section:**

$$
\int\frac{dx}{1+x^2}=\arctan x+C,\qquad \int\frac{dx}{x\ln x}=\ln(\ln x)+C,\qquad \int\tan x\,dx=\ln|\sec x|+C
$$

$$
\int(2x+1)^{-1/2}dx=(2x+1)^{1/2}+C,\qquad \int\frac{\ln^2x}{x}\,dx=\frac13\ln^3x+C
$$

**Key results from the examples:**

$$
\int_0^\infty\frac{dx}{1+x^2}=\frac{\pi}{2},\quad \int_1^\infty\frac{dx}{x}=\infty,\quad \int_1^\infty\frac{dx}{x^2}=1,\quad \int_0^1\frac{dx}{x^2}=\infty
$$
