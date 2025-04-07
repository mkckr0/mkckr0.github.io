---
title: Newton-Cotes 求积公式性质的证明
date: 2025-04-03 10:50
---

Newton-Cotes 求积公式是对定积分的数值计算。它利用 Lagrange 插值多项式近似替代函数 $f(x)$ 进行定积分计算。于是一个复杂的、甚至不可求的定积分计算，就被转化为多项式的定积分计算。

$$
\begin{gather*}
\int_{a}^{b} f(x) dx \approx \int_{a}^{b} p_n(x) dx = (b-a) \sum_{i=0}^{n} C_{i}^{(n)} f(x_{i}) \\

C_{i}^{( n )} = \frac{1}{n} \frac{ (-1) ^{n-i} }{ i! (n-i)!} \int_{a}^{b} \prod_{\substack{j=0 \\ j \neq i}}^{n} (t-j) dt

\end{gather*}
$$

Newton-Cotes 求积公式可以理解为定积分数值近似等于插值节点的加权平均值乘以区间长度，而 $C_{i}^{(n)}$ 就是各个插值节点的权重系数，称为 Cotes 系数。Newton-Cotes 求积公式的核心就是 Cotes 系数，它与 $f(x)$ 无关。所以可以制作一张 n 阶 Cotes 系数表，在计算特定 $f(x)$ 的定积分时可以直接代入计算。

Cotes 系数有两条性质：

- 对称性：$C_{i}^{(n)} = C_{n-i}^{(n)}$
- 规范性：$\displaystyle \sum_{i=0}^{n} C_{i}^{(n)} = 1$

可以利用这两条性质简化 Cotes 系数的计算。下面对这两条性质进行证明。

## 证明对称性

观察 $C_{i}^{(n)}$ 可以发现 $i!(n-i)!$ 具有对称性，所以只需要证明 $\displaystyle (-1) ^{n-i} \int_{a}^{b} \prod_{\substack{j=0 \\\\ j \neq i}}^{n} (t-j) dt = (-1) ^{i} \int_{a}^{b} \prod_{\substack{j=0 \\\\ j \neq n-i}}^{n} (t-j) dt$。

可以看到被积函数 $\displaystyle \prod_{\substack{j=0 \\\\ j \neq i}}^{n} (t-j)$ 和 $\displaystyle \prod_{\substack{j=0 \\\\ j \neq n-i}}^{n} (t-j)$ 既不相等，也不相差 $(-1)^{k}$。无法通过定积分线性变换得出积分值相等。

观察 4 阶 Cotes 系数 $C_{1}^{(4)}$ 和 $C_{3}^{(4)}$ 的 $\displaystyle \prod_{\substack{j=0 \\\\ j \neq i}}^{n} (t-j)$

$$
\begin{align}
\prod_{\substack{j=0 \\ j \neq 1}}^{4} (t-j) &= t\bcancel{(t-1)}(t-2)(t-3)(t-4) \\
\prod_{\substack{j=0 \\ j \neq 3}}^{4} (t-j) &= t(t-1)(t-2)\bcancel{(t-3)}(t-4) \\
\end{align}
$$

二者具有一定的对称性。将 $(2)$ 式进行逆序。

$$
\begin{align}
\prod_{\substack{j=0 \\ j \neq 3}}^{4} (t-j) &= (t-4)\bcancel{(t-3)}(t-2)(t-1)t \\
\end{align}
$$

这样被删除的项和 $(1)$ 式位置相同。再将每一项都乘以 $-1$。

$$
\begin{align}
(-1)^{4} \prod_{\substack{j=0 \\ j \neq 3}}^{4} (t-j) &= (4-t)\bcancel{(3-t)}(2-t)(1-t)(-t) \\
\end{align}
$$

再观察 $(1)$ $(4)$。发现它们都是公差为 $1$ 的递减数列。此时就可以进行替换，$4-t$ 替换为 $s$，也就是 $t= 4-s$。

$$
\begin{align}
(-1)^{4} \prod_{\substack{j=0 \\ j \neq 3}}^{4} ((4-s)-j) &= s\bcancel{(s-1)}(s-2)(s-3)(s-4) \\
\end{align}
$$

最后 $(1)$ $(5)$ 是完全等价的，它们的定积分也相等。在这个变换过程中，核心的步骤就是作 $t=4-s$ 替换。推广到 n 阶的情况，就是 $t=n-s$。

经过上述分析，可以作以下变换

$$
\begin{align*}

C_{n-i}^{( n )}

&= \frac{1}{n} \frac{ (-1) ^{n - (n-i)} }{ (n-i)! (n-(n-i))!} \int_{a}^{b} \prod_{\substack{j=0 \\ j \neq n-i}}^{n} (t-j) dt \\

&= \frac{1}{n} \frac{ (-1) ^{i} }{ i! (n-i)! } \int_{b}^{a} \prod_{\substack{j=0 \\ j \neq n-i}}^{n} ((n-s)-j) d(n-s) \\

&= \frac{1}{n} \frac{ (-1) ^{i} }{ i! (n-i)! } \int_{a}^{b} \prod_{\substack{j=0 \\ j \neq n-i}}^{n} ((n-j)-s) ds \\

&= \frac{1}{n} \frac{ (-1) ^{i} }{ i! (n-i)! } \int_{a}^{b} \prod_{\substack{k=n \\ k \neq i}}^{0} (k-s) ds \\

&= \frac{1}{n} \frac{ (-1) ^{i} }{ i! (n-i)! } \int_{a}^{b} \prod_{\substack{k=0 \\ k \neq i}}^{n} (k-s) ds \\

&= \frac{1}{n} \frac{ (-1) ^{i} }{ i! (n-i)! } \int_{a}^{b} \prod_{\substack{j=0 \\ j \neq i}}^{n} (j-t) dt \\

&= \frac{1}{n} \frac{ (-1) ^{i+n} }{ i! (n-i)! } \int_{a}^{b} \prod_{\substack{j=0 \\ j \neq i}}^{n} (t- j) dt \\

&= \frac{1}{n} \frac{ (-1) ^{n-i} }{ i! (n-i)! } \int_{a}^{b} \prod_{\substack{j=0 \\ j \neq i}}^{n} (t- j) dt \\

&= C_{i}^{( n )}

\end{align*}
$$

即 Cotes 系数的对称性得证。

## 证明规范性

因为 Cotes 系数 $C_{i}^{(n)}$ 与 $f(x)$ 无关，所以不妨令它为常函数 $f(x) \equiv 1$。代入 Newton-Cotes 求积公式的定义，得到

$$
\begin{align*}
& \int_{a}^{b} 1 dx \approx \int_{a}^{b} p_n(x) dx = (b-a) \sum_{i=0}^{n} C_{i}^{(n)} \\

\implies & b-a \approx \int_{a}^{b} p_n(x) dx = (b-a) \sum_{i=0}^{n} C_{i}^{(n)} \\

\end{align*}
$$

所以只需要证明 $\displaystyle b-a = \int_{a}^{b} p_n(x) dx$，也就是将近似相等变成严格相等，就可以得到 $\displaystyle \sum_{i=0}^{n} C_{i}^{(n)} = 1$。

常函数 $f(x) \equiv 1$ 本身也是一个插值多项式，因为 $\forall x_{i} : f(x_{i}) = 1$ 成立。根据插值多项式的性质，对于特定插值条件的插值多项式存在且唯一。所以 $p_n(x) = f(x) \equiv 1$。所以 Cotes 系数的规范性得证。

经过这个证明过程，可以得出结论：多项式函数（包括常函数）的 Lagrange 插值多项式就是其本身。