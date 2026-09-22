# Section 7.2 - Trigonometric Integrals

## Summary

This section covers integrals built from powers of trigonometric functions, such as $\int \sin^m(x)\cos^n(x)\,dx$ and $\int \tan^m(x)\sec^n(x)\,dx$. You rewrite the integrand with a trig identity (Pythagorean or power reduction) until a plain $u$-substitution works. Which identity to use depends on whether the powers are odd or even. Reach for these methods when the integrand is a product of powers of $\sin$ and $\cos$, or of $\tan$ and $\sec$, and no direct antiderivative is available.

## Definitions and theorems

**Definition #1.** The functions tangent and secant are given by

$$
\tan(x) = \frac{\sin(x)}{\cos(x)} \quad \text{and} \quad \sec(x) = \frac{1}{\cos(x)}
$$

for $x \neq k\pi + \pi/2$ where $k \in \mathbf{Z}$.

**Theorem #1.** We have the Pythagorean identities

$$
\cos^2(x) + \sin^2(x) = 1 \quad \text{and} \quad \tan^2(x) = \sec^2(x) - 1,
$$

the power reduction/half-angle identities

$$
\sin^2(x) = \frac{1 - \cos(2x)}{2} \quad \text{and} \quad \cos^2(x) = \frac{1 + \cos(2x)}{2}
$$

and

$$
\sin(x)\cos(x) = \frac{1}{2}\sin(2x).
$$

**Theorem #2.** We have

$$
\frac{d}{dx}\sin(x) = \cos(x) \quad \text{and} \quad \frac{d}{dx}\cos(x) = -\sin(x).
$$

Also

$$
\int \sin(x)\,dx = -\cos(x) + C \quad \text{and} \quad \int \cos(x)\,dx = \sin(x) + C
$$

where $C \in \mathbf{R}$.

**Theorem #3.** We have

$$
\frac{d}{dx}\tan(x) = \sec^2(x) \quad \text{and} \quad \frac{d}{dx}\sec(x) = \sec(x)\cdot\tan(x).
$$

Also

$$
\int \tan(x)\,dx = \ln|\sec(x)| + C
$$

and

$$
\int \sec(x)\,dx = \ln|\sec(x) + \tan(x)| + C
$$

where $C \in \mathbf{R}$.

## Methods

### Method A: $\int \sin^m(x)\cos^n(x)\,dx$

1. **$m$ odd (odd power of sine).**
   - Split off one sine: $\sin^m(x) = \sin^{m-1}(x)\sin(x)$. Since $m-1$ is even, $\sin^{m-1}(x)$ is a power of $\sin^2(x)$.
   - Replace every $\sin^2(x)$ with $1 - \cos^2(x)$.
   - Substitute $u = \cos(x)$, $du = -\sin(x)\,dx$. The saved $\sin(x)\,dx$ becomes $-du$.
   - Integrate the polynomial in $u$, then back substitute.

2. **$n$ odd (odd power of cosine).**
   - Split off one cosine: $\cos^n(x) = \cos^{n-1}(x)\cos(x)$.
   - Replace every $\cos^2(x)$ with $1 - \sin^2(x)$.
   - Substitute $u = \sin(x)$, $du = \cos(x)\,dx$.
   - Integrate the polynomial in $u$, then back substitute.

3. **$m$ and $n$ both even.**
   - Use the power reduction identities $\sin^2(x) = \frac{1 - \cos(2x)}{2}$ and $\cos^2(x) = \frac{1 + \cos(2x)}{2}$.
   - Repeat if a squared cosine such as $\cos^2(2x)$ shows up.
   - Integrate terms like $\cos(kx)$ with the substitution $u = kx$.

### Method B: $\int \tan^m(x)\sec^n(x)\,dx$

1. **$m$ odd (odd power of tangent).** Use $\tan^2(x) = \sec^2(x) - 1$ and "save" a $\sec(x)\tan(x)$, which is the derivative of $\sec(x)$. In Example #5 ($n = 0$) this became splitting $\tan^3(x)$ into $\tan(x)\sec^2(x) - \tan(x)$ and handling each piece separately.
2. **$n$ even (even power of secant).** Use $\sec^2(x) = 1 + \tan^2(x)$ and "save" a $\sec^2(x)$. Then substitute $u = \tan(x)$, $du = \sec^2(x)\,dx$.
3. **Otherwise, experiment.** Rewrite in terms of $\sin$ and $\cos$, try identities, and look for a function whose derivative also appears in the integrand.

### Method C: Quotients such as $\tan$ and $\cot$

Write the function as a quotient of $\sin$ and $\cos$. When the numerator is (up to sign) the derivative of the denominator, substitute $u =$ denominator to get $\int \frac{1}{u}\,du = \ln|u| + C$.

## Worked examples

### Example #1

Find $\displaystyle\int \cos^2(x)\,dx$.

Both powers are even ($m = 0$, $n = 2$), so use power reduction:

$$
\cos^2(x) = \frac{1 + \cos(2x)}{2}.
$$

Then

$$
\begin{aligned}
\int \cos^2(x)\,dx &= \int \frac{1 + \cos(2x)}{2}\,dx \\
&= \int \frac{1}{2} + \frac{1}{2}\cos(2x)\,dx \\
&= \frac{x}{2} + \frac{1}{4}\sin(2x) + C.
\end{aligned}
$$

The second term uses the $u$-substitution $u = 2x$, $du = 2\,dx$, so $dx = \frac{1}{2}\,du$:

$$
\int \frac{1}{2}\cos(2x)\,dx = \int \frac{1}{4}\cos(u)\,du = \frac{1}{4}\sin(u) + C = \frac{1}{4}\sin(2x) + C.
$$

$$
\boxed{\int \cos^2(x)\,dx = \frac{x}{2} + \frac{1}{4}\sin(2x) + C}
$$

### Example #2

Find $\displaystyle\int \cos^5(x)\,dx$.

The power of cosine is odd, so save one $\cos(x)$ and use $\cos^2(x) = 1 - \sin^2(x)$ on the rest:

$$
\begin{aligned}
\cos^5(x) &= \left(\cos^2(x)\right)^2\cos(x) \\
&= \left(1 - \sin^2(x)\right)^2\cos(x).
\end{aligned}
$$

So

$$
\int \cos^5(x)\,dx = \int \left(1 - \sin^2(x)\right)^2\cos(x)\,dx.
$$

Substitute $u = \sin(x)$, $du = \cos(x)\,dx$:

$$
\begin{aligned}
\int \left(1 - \sin^2(x)\right)^2\cos(x)\,dx &= \int (1 - u^2)^2\,du \\
&= \int 1 - 2u^2 + u^4\,du \\
&= u - \frac{2}{3}u^3 + \frac{1}{5}u^5 + C.
\end{aligned}
$$

The expansion step is $(1 - u^2)^2 = 1 - 2u^2 + u^4$. Back substitute $u = \sin(x)$:

$$
\boxed{\int \cos^5(x)\,dx = \sin(x) - \frac{2}{3}\sin^3(x) + \frac{1}{5}\sin^5(x) + C}
$$

### Example #3

Find $\displaystyle\int \sin^3(x)\cos^5(x)\,dx$.

The power of sine is odd, so save one $\sin(x)$ and use $\sin^2(x) = 1 - \cos^2(x)$:

$$
\begin{aligned}
\sin^3(x)\cos^5(x) &= \sin^2(x)\cos^5(x)\sin(x) \\
&= \left(1 - \cos^2(x)\right)\cos^5(x)\sin(x) \\
&= \left(\cos^5(x) - \cos^7(x)\right)\sin(x).
\end{aligned}
$$

Let $u = \cos(x)$, $du = -\sin(x)\,dx$, so $\sin(x)\,dx = -du$. Then

$$
\begin{aligned}
\int \sin^3(x)\cos^5(x)\,dx &= \int \left(\cos^5(x) - \cos^7(x)\right)\sin(x)\,dx \\
&= -\int u^5 - u^7\,du \\
&= \int u^7 - u^5\,du \\
&= \frac{1}{8}u^8 - \frac{1}{6}u^6 + C \\
&= \frac{1}{8}\cos^8(x) - \frac{1}{6}\cos^6(x) + C.
\end{aligned}
$$

The minus sign from $du = -\sin(x)\,dx$ gets absorbed by flipping $u^5 - u^7$ to $u^7 - u^5$.

$$
\boxed{\int \sin^3(x)\cos^5(x)\,dx = \frac{1}{8}\cos^8(x) - \frac{1}{6}\cos^6(x) + C}
$$

### Example #4

Find $\displaystyle\int \sin^2(x)\cos^2(x)\,dx$.

Both powers are even, so use the power reduction identities

$$
\sin^2(x) = \frac{1 - \cos(2x)}{2}, \qquad \cos^2(x) = \frac{1 + \cos(2x)}{2}.
$$

Multiply them, using the difference of squares $(1 - a)(1 + a) = 1 - a^2$:

$$
\sin^2(x)\cos^2(x) = \frac{(1 - \cos(2x))(1 + \cos(2x))}{4} = \frac{1}{4}\left(1 - \cos^2(2x)\right).
$$

A second even power, $\cos^2(2x)$, appears. Apply power reduction again with $2x$ in place of $x$:

$$
\cos^2(2x) = \frac{1 + \cos(4x)}{2}.
$$

So

$$
\begin{aligned}
\sin^2(x)\cos^2(x) &= \frac{1}{4}\left(1 - \frac{1}{2} - \frac{1}{2}\cos(4x)\right) \\
&= \frac{1}{4}\left(\frac{1}{2} - \frac{1}{2}\cos(4x)\right).
\end{aligned}
$$

Integrate:

$$
\begin{aligned}
\int \sin^2(x)\cos^2(x)\,dx &= \frac{1}{4}\int \frac{1}{2} - \frac{1}{2}\cos(4x)\,dx \\
&= \frac{1}{4}\left(\frac{1}{2}x - \frac{1}{8}\sin(4x)\right) + C.
\end{aligned}
$$

For the cosine term, $u = 4x$, $dx = \frac{1}{4}\,du$ gives $\int \frac{1}{2}\cos(4x)\,dx = \frac{1}{8}\sin(4x) + C$. Distributing the $\frac{1}{4}$:

$$
\boxed{\int \sin^2(x)\cos^2(x)\,dx = \frac{x}{8} - \frac{1}{32}\sin(4x) + C}
$$

### Example #5

Find $\displaystyle\int \tan^3(x)\,dx$.

Use $\tan^2(x) = \sec^2(x) - 1$:

$$
\begin{aligned}
\tan^3(x) &= \tan(x)\left(\sec^2(x) - 1\right) \\
&= \tan(x)\sec^2(x) - \tan(x),
\end{aligned}
$$

so

$$
\int \tan^3(x)\,dx = \int \tan(x)\sec^2(x)\,dx - \int \tan(x)\,dx.
$$

**First integral.** Let $u = \tan(x)$, $du = \sec^2(x)\,dx$:

$$
\int \tan(x)\sec^2(x)\,dx = \int u\,du = \frac{1}{2}\tan^2(x) + C_1.
$$

**Second integral.** Write $\tan(x) = \frac{\sin(x)}{\cos(x)}$ and let $w = \cos(x)$, $dw = -\sin(x)\,dx$:

$$
\begin{aligned}
\int \tan(x)\,dx &= \int \frac{\sin(x)}{\cos(x)}\,dx \\
&= -\int \frac{1}{w}\,dw \\
&= -\ln|w| + C_2 \\
&= -\ln|\cos(x)| + C_2 \\
&= \ln\left|\frac{1}{\cos(x)}\right| + C_2 \\
&= \ln|\sec(x)| + C_2.
\end{aligned}
$$

The step $-\ln|\cos(x)| = \ln\left|\frac{1}{\cos(x)}\right|$ uses $-\ln a = \ln\frac{1}{a}$. This derivation proves the formula for $\int \tan(x)\,dx$ in Theorem #3.

Combining, with $C = C_1 - C_2$:

$$
\boxed{\int \tan^3(x)\,dx = \frac{1}{2}\tan^2(x) - \ln|\sec(x)| + C}
$$

**Does WolframAlpha give a different result?** WolframAlpha returns

$$
\int \tan^3(x)\,dx = \frac{\sec^2(x)}{2} + \log(\cos(x)) + \text{constant}.
$$

This looks different, but the two answers agree up to a constant. Using $\tan^2(x) = \sec^2(x) - 1$:

$$
\frac{1}{2}\tan^2(x) = \frac{1}{2}\sec^2(x) - \frac{1}{2},
$$

and $-\ln|\sec(x)| = \ln|\cos(x)|$. So

$$
\frac{1}{2}\tan^2(x) - \ln|\sec(x)| = \frac{\sec^2(x)}{2} + \ln|\cos(x)| - \frac{1}{2}.
$$

The extra $-\frac{1}{2}$ gets absorbed into the arbitrary constant, so both antiderivatives are correct.

### Example #6

Find $\displaystyle\int \sec^4(x)\,dx$.

**Method 1 (save a $\sec^2(x)$).** The power of secant is even, so use $\sec^2(x) = 1 + \tan^2(x)$:

$$
\begin{aligned}
\sec^4(x) &= \sec^2(x)\sec^2(x) \\
&= \left(1 + \tan^2(x)\right)\sec^2(x).
\end{aligned}
$$

So

$$
\int \sec^4(x)\,dx = \int \left(1 + \tan^2(x)\right)\sec^2(x)\,dx.
$$

Substitute $u = \tan(x)$, $du = \sec^2(x)\,dx$:

$$
\begin{aligned}
\int \left(1 + \tan^2(x)\right)\sec^2(x)\,dx &= \int 1 + u^2\,du \\
&= u + \frac{1}{3}u^3 + C.
\end{aligned}
$$

Back substitute:

$$
\boxed{\int \sec^4(x)\,dx = \tan(x) + \frac{1}{3}\tan^3(x) + C}
$$

**Method 2 (rewrite in sine and cosine).** Start from $\sec^4(x) = \frac{1}{\cos^4(x)}$ and write the numerator $1$ as $\cos^2(x) + \sin^2(x)$:

$$
\begin{aligned}
\sec^4(x) &= \frac{1}{\cos^4(x)} \\
&= \frac{\cos^2(x) + \sin^2(x)}{\cos^4(x)} \\
&= \left(1 + \frac{\sin^2(x)}{\cos^2(x)}\right)\frac{1}{\cos^2(x)}.
\end{aligned}
$$

The last step splits the fraction: $\frac{\cos^2(x)}{\cos^4(x)} + \frac{\sin^2(x)}{\cos^4(x)} = \frac{1}{\cos^2(x)}\left(1 + \frac{\sin^2(x)}{\cos^2(x)}\right)$.

Substitute $u = \frac{\sin(x)}{\cos(x)}$, $du = \frac{1}{\cos^2(x)}\,dx$ (this is $u = \tan(x)$, $du = \sec^2(x)\,dx$ in disguise). Then $\frac{\sin^2(x)}{\cos^2(x)} = u^2$ and

$$
\begin{aligned}
\int \frac{1}{\cos^4(x)}\,dx &= \int 1 + u^2\,du \\
&= u + \frac{1}{3}u^3 + C.
\end{aligned}
$$

Back substitute to get the same answer:

$$
\int \sec^4(x)\,dx = \tan(x) + \frac{1}{3}\tan^3(x) + C.
$$

### Example #7

Find $\displaystyle\int \cot(x)\,dx$.

Write

$$
\cot(x) = \frac{\cos(x)}{\sin(x)}.
$$

Let $u = \sin(x)$, $du = \cos(x)\,dx$. Then

$$
\begin{aligned}
\int \cot(x)\,dx &= \int \frac{\cos(x)}{\sin(x)}\,dx \\
&= \int \frac{1}{u}\,du \\
&= \ln|u| + C \\
&= \ln|\sin(x)| + C.
\end{aligned}
$$

$$
\boxed{\int \cot(x)\,dx = \ln|\sin(x)| + C}
$$

## Common mistakes and tips

- **Pick the substitution from the saved factor.** If you save a $\sin(x)$, substitute $u = \cos(x)$. If you save a $\cos(x)$, substitute $u = \sin(x)$. If you save a $\sec^2(x)$, substitute $u = \tan(x)$. The saved factor has to be the $du$.
- **Watch the sign with $u = \cos(x)$.** Since $du = -\sin(x)\,dx$, a minus sign appears (Example #3, and $\int \tan(x)\,dx$ in Example #5). Forgetting it flips the sign of the whole answer.
- **Don't drop the chain rule factor.** $\int \cos(2x)\,dx = \frac{1}{2}\sin(2x) + C$, not $\sin(2x) + C$. Likewise, $\int \cos(4x)\,dx = \frac{1}{4}\sin(4x) + C$.
- **Power reduction may need repeating.** In Example #4, reducing $\sin^2(x)\cos^2(x)$ produced $\cos^2(2x)$, which needed a second pass with $\cos^2(2x) = \frac{1 + \cos(4x)}{2}$.
- **Check the odd power first.** Only use power reduction when both powers of $\sin$ and $\cos$ are even. If either power is odd, the save-one-factor method is faster.
- **Different-looking answers can both be right.** Trig antiderivatives can differ by a constant, as with $\frac{1}{2}\tan^2(x)$ versus $\frac{1}{2}\sec^2(x)$ in Example #5. Before deciding an answer is wrong, test whether the difference is constant using an identity.
- **Log identities clean up answers.** $-\ln|\cos(x)| = \ln|\sec(x)|$ because $-\ln a = \ln\frac{1}{a}$.
- **When stuck, rewrite in $\sin$ and $\cos$.** Example #6 Method 2 and Example #7 both work by writing everything as quotients of $\sin$ and $\cos$ and spotting a derivative.
- **Keep the absolute values** inside logarithms: $\ln|\sec(x)|$, $\ln|\sin(x)|$.

## Formula sheet

**Definitions**

$$
\tan(x) = \frac{\sin(x)}{\cos(x)}, \qquad \sec(x) = \frac{1}{\cos(x)}, \qquad \cot(x) = \frac{\cos(x)}{\sin(x)}
$$

**Pythagorean identities**

$$
\cos^2(x) + \sin^2(x) = 1, \qquad \tan^2(x) = \sec^2(x) - 1
$$

**Power reduction / half-angle**

$$
\sin^2(x) = \frac{1 - \cos(2x)}{2}, \qquad \cos^2(x) = \frac{1 + \cos(2x)}{2}, \qquad \sin(x)\cos(x) = \frac{1}{2}\sin(2x)
$$

**Derivatives**

$$
\frac{d}{dx}\sin(x) = \cos(x), \quad \frac{d}{dx}\cos(x) = -\sin(x), \quad \frac{d}{dx}\tan(x) = \sec^2(x), \quad \frac{d}{dx}\sec(x) = \sec(x)\tan(x)
$$

**Integrals**

$$
\int \sin(x)\,dx = -\cos(x) + C, \qquad \int \cos(x)\,dx = \sin(x) + C
$$

$$
\int \tan(x)\,dx = \ln|\sec(x)| + C, \qquad \int \sec(x)\,dx = \ln|\sec(x) + \tan(x)| + C
$$

$$
\int \cot(x)\,dx = \ln|\sin(x)| + C
$$

**Strategy for $\int \sin^m(x)\cos^n(x)\,dx$**

| Case | Rewrite | Substitute |
|---|---|---|
| $m$ odd | $\sin^m(x) = \sin^{m-1}(x)\sin(x)$, use $\sin^2(x) = 1 - \cos^2(x)$ | $u = \cos(x)$ |
| $n$ odd | $\cos^n(x) = \cos^{n-1}(x)\cos(x)$, use $\cos^2(x) = 1 - \sin^2(x)$ | $u = \sin(x)$ |
| $m, n$ both even | $\sin^2(x) = \frac{1 - \cos(2x)}{2}$, $\cos^2(x) = \frac{1 + \cos(2x)}{2}$ | $u = kx$ for $\cos(kx)$ terms |

**Strategy for $\int \tan^m(x)\sec^n(x)\,dx$**

| Case | Rewrite | Save |
|---|---|---|
| $m$ odd | $\tan^2(x) = \sec^2(x) - 1$ | $\sec(x)\tan(x)$ |
| $n$ even | $\sec^2(x) = 1 + \tan^2(x)$ | $\sec^2(x)$, then $u = \tan(x)$ |
| otherwise | experiment | |

**Results worth knowing**

$$
\int \cos^2(x)\,dx = \frac{x}{2} + \frac{1}{4}\sin(2x) + C, \qquad \int \sec^4(x)\,dx = \tan(x) + \frac{1}{3}\tan^3(x) + C
$$

$$
\int \tan^3(x)\,dx = \frac{1}{2}\tan^2(x) - \ln|\sec(x)| + C
$$
