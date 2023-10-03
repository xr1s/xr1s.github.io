+++
title = "莫比乌斯反演"
date = "2023-10-03"
type = "post"
tags = ["Mathematics"]
+++

其实就是重新整理了一下 *Introductory Combinatorics Fifth Edition* 容斥原理一章莫比乌斯反演一节，修正了书上错误的内容，加上了一些证明的内容。虽然我想说的和网上已有的资料大同小异——容斥原理、二项式反演和数论反演——不过本文打算从最一般、定义在偏序集上的 Möbius 反演开始。

本文可能过于理论化，刷题记录会扔在另一篇文章中，到时候会介绍莫比乌斯反演是如何为计算提供帮助和优化的，可能这样才更适合算法竞赛中的反演入门。

莫比乌斯反演是这么个东西，当我们有定义在偏序集 \\(\mathcal F(S,\le)\\) 上的函数 \\(f\\)，和 \\(\displaystyle F(x,y)=\sum_{x\le z\le y}f(x,z)\\)，可以通过莫比乌斯反演求得 \\(\displaystyle f(x,y)=\sum_{x\le z\le y}F(x,z)\mu(z,y)\\)，其中 \\(\mu\\) 是 \\(\mathcal F(S,\le)\\) 上的莫比乌斯函数。更一般的形式中，当 \\(x=0\\) 时候，可以简写成 \\(\displaystyle F(y)=\sum_{z\le y}f(z)\implies f(y)=\sum_{z\le y}F(z)\mu(y,z)\\)。

本质上，容斥原理、二项式反演、数论反演都是莫比乌斯反演在特定偏序集上的不同实例。

## 前置知识

### 偏序关系

偏序是一种二元关系，较为常见的偏序关系有实数域 \\(\mathbb R\\) 上的 \\(\le\\) 关系、集合之间的子集 \\(\subseteq\\) 还有整数之间的整除 \\(|\\) 关系等。

更抽象地，偏序被定义为一个集合和一个关系的二元组，如实数上的 \\(\le\\) 关系即为形如 \\((\mathbb R,\le)\\) 的二元组。

对于偏序关系 \\((S,\le)\\)，需要满足性质：

- 自反性： \\(\forall a\in S:a\le a\\)
- 传递性： \\(\forall a,b,c\in S:(a\le b\land b\le c)\implies a\le c\\)
- 反对称性： \\(\forall a,b\in S:(a\le b\land b\le a)\implies a=b\\)

反对称性中的 \\(=\\) 就表示 \\(a\\) 和 \\(b\\) 是同一个元素，其实就是说明没有两个不同的元素同时 \\(\le\\) 对方，它们要么有确定顺序的 \\(\le\\) 关系，要么没有关系。

为了方便阐述，在偏序 \\(\le\\) 的基础上定义关系，\begin{equation*}
\begin{align*}
& a\ge b: && b\le a \\\\
& a\lt b: && (a\le b)\land(a\neq b) \\\\
& a\gt b: && b\lt a
\end{align*}
\end{equation*}

需要注意 \\(\not\le\\) 和 \\(\gt\\) 的区别，\\(\not\le\\) 可以是不构成关系，而 \\(\gt\\) 是有关系的，也就是 \\(\gt\\) 组成的关系集是 \\(\not\le\\) 的子集。\\(\ge\\) 和 \\(\not\lt\\) 类似。

### 卷积

下文中只考虑定义在偏序集 \\((D,\le)\\) 上，定义域为 \\(D\times D\\) 的实值函数<!--（实际上只要值域和值域上的加法运算组成的二元组满足交换群即可）-->集合 \\(\mathcal F(D,\le)\\)——实值函数 \\(f\\) 就是说值域是实数域 \\(f:D\times D\to\mathbb R\\)——且该集合满足其中的所有函数 \\(f\in\mathcal F(D,\le)\\) 对于所有定义域中的元素 \\(\forall x,y\in D: x\not\le y\\) 有 \\(f(x,y)=0\\) 且 \\(\forall x,y\in D:x=y\implies f(x,y)\neq 0\\)。

设函数 \\(f,g\in\mathcal F(D,\le)\\)，定义卷积 \\(\*\\) 为

\begin{equation*}
\end{equation*}

\begin{equation*}
(f\*g)(x,y)=\left\\{
  \begin{align*}
    & \sum_{x\le k\le y}f(x,k)g(k,y) && \text{for }x\le y \\\\
    & 0                              && \text{otherwise} \\\\
  \end{align*}
\right.
\end{equation*}

显然有结合律 \\((f\*g)\*h=f\*(g\*h)\\)，但无交换律。

事实上，若 \\(D\\) 可数且 \\((D,\le)\\) 满足全序关系的话，将函数 \\(f\in\mathcal F(D,\le)\\) 定义域中的元素按行纵依序排列，以对应的函数值填充矩阵，这实际上就是一个上三角矩阵，函数卷积即为矩阵乘法，这也从另一方面说明了没有交换律。

### [Iverson 括号](https://en.wikipedia.org/wiki/Iverson_bracket)

定义 Iverson 括号

\begin{equation*}
[cond]=\left\\{\begin{align*}
  & 1 && \text{if }cond\text{ is true} \\\\
  & 0 && \text{otherwise} \\\\
\end{align*}\right.
\end{equation*}

### [Kronecker delta 函数](https://en.wikipedia.org/wiki/Kronecker_delta)

定义 Kronecker delta 函数 \\(\delta(x,y)=[x=y]\\)。

和积分中的克罗内克函数类似，此处定义的克罗内克函数满足对于任意 \\(f\in\mathcal F(D,\le):\delta\*f=f\*\delta=f\\)。

实际上，delta 函数即为群 \\((\mathcal F(D,\le),*)\\) 中的单位元。在可数全序集上这是一个单位矩阵。

### Zeta 函数

定义 Zeta 函数 \\(\zeta(x,y)=[x\le y]\\)。

这里的 Zeta 函数其实就是偏序集的一种表示。在可数全序集上这是一个全 1 的上三角矩阵。

## Möbius mu 函数

### 定义

对于所有 \\(f\in\mathcal F(D,\le)\\)，构造 \\(f^{-1}\\) 为

\begin{equation*}
f^{-1}(x,y)=\left\\{\begin{align*}
  &-\dfrac1{f(y,y)}\sum_{x\le k\lt y}f^{-1}(x,k)f(k,y) & \text{for }x\lt y\\\\
  &\dfrac1{f(y,y)}&\text{for }x=y\\\\
  &0&\text{for }x\not\le y
\end{align*}\right.
\end{equation*}

需要注意，虽然 \\(f^{-1}(x,y)\text{ for }x\lt y\\) 的定义依赖到了\\(f^{-1}(x,z)\text{ for }x\le z\lt y\\)，但是不存在反向的依赖，因此可递归定义。

可以证明 \\(f^{-1}\*f=\delta\\)，如下：

\begin{equation*}
\begin{align*}
(f^{-1}\*f)(x,y)=&\sum_{x\le k\le y}f^{-1}(x,k)f(k,y) \\\\
=&f^{-1}(x,y)f(y,y)+\sum_{x\le k\lt y}f^{-1}(x,k)f(k,y) \\\\
\end{align*}
\end{equation*}

当 \\(x=y\\) 的情况上式显而易见可得值为 \\(1+0=1\\)。当 \\(x\neq y\\) 时，再将 \\(f^{-1}(x,y)\\) 定义带入展开有：

\begin{equation*}
\begin{align*}
  & f^{-1}(x,y)f(y,y)+\sum_{x\le k\lt y}f^{-1}(x,k)f(k,y) \\\\
= & \left[-\dfrac1{f(y,y)}\sum_{x\le k\lt y}f^{-1}(x,k)f(k,y)\right]f(y,y)+\sum_{x\le k\lt y}f^{-1}(x,k)f(k,y) \\\\
= & -\sum_{x\le k\lt y}f^{-1}(x,k)f(k,y)+\sum_{x\le k\lt y}f^{-1}(x,k)f(k,y) \\\\
= & 0 \\\\
\end{align*}
\end{equation*}

同理可证明 \\(f*f^{-1}=\delta\\)，这里就能证明 \\(\delta\\) 是群 \\((\mathcal F(D,\le),\*)\\) 的单位元了。最后，我们在此定义 Möbius 函数 \\(\mu=\zeta^{-1}\\)。

### 性质

根据卷积定义，\\(\displaystyle\sum_{x\le z\le y}\mu(x,z)\zeta(z,y)=\delta(x,y)\\)，根据 \\(\zeta\\) 定义我们有 \\(\displaystyle\sum_{x\le z\le y}\mu(x,z)=\delta(x,y)\\)，通过将 \\(\mu(x,y)\\) 移出累加符号，这等价于

\begin{equation*}
\mu(x,y)=\left\\{\begin{align*}
& -\sum\limits_{x\le z\lt y}\mu(x,z) && \text{for }x\lt y \\\\
& 1 && \text{for }x=y \\\\
& 0 && \text{for }x\not\le y
\end{align*}\right.
\end{equation*}

### \\((\mathbb N,\le)\\)

\begin{equation*}
\mu(x,y)=\left\\{\begin{align*}
& 1 && \text{for }x=y \\\\
&-1 && \text{for }x+1=y \\\\
& 0 && \text{otherwise} \\\\
\end{align*}\right.
\end{equation*}

结论是很显然的，递推一下就得到了。

事实上这个结论对于所有可数的**全序集**而言均正确。

### \\((\mathcal P(S),\subseteq)\\)

集合的幂集上的包含关系，其 Möbius 函数为：

\begin{equation*}
\mu(X,Y)=\left\\{\begin{align*}
& (-1)^{|Y|-|X|} && \text{for } X\subseteq Y \\\\
& 0 && \text{otherwise} \\\\
\end{align*}\right.
\end{equation*}

*归纳证明：*

\begin{equation*}
\begin{align*}
\mu(X,Y)
& =-\sum_{X\subseteq Z\subset Y}\mu(X,Z) \\\\
& =-\sum_{X\subseteq Z\subset Y}(-1)^{|Z|-|X|} \\\\
& =-\sum_{\varnothing\subseteq Z\subset Y\setminus X}(-1)^{|Z|} \\\\
& =-\sum_{k=0}^{|Y|-|X|-1}\binom {|Y|-|X|}k(-1)^k \\\\
& =(-1)^{|Y|-|X|}-\sum_{k=0}^{|Y|-|X|}\binom{|Y|-|X|}k(-1)^k \\\\
& =(-1)^{|Y|-|X|}-(1-1)^{|Y|-|X|} && \triangleright\text{ 二项式定理} \\\\
& =(-1)^{|Y|-|X|}
\end{align*}
\end{equation*}

### 偏序集直积

设两个偏序集 \\((S,\le_S)\\) 和 \\((T,\le_T)\\)，直积 \\((S,\le_S)\times(T,\le_T)=(S\times T,\le)\\)，这里要求 \\(\le\\) 满足 \\(\forall (s,t),(s',t')\in X\times Y:(s,t)\le(s',t')\iff(s\le_Ss')\land(t\le_Tt')\\)。

设 \\((S,\le_S)\\) 的 Möbius 函数为 \\(\mu_S\\)，\\((T,\le_T)\\) 的 Möbius 函数为 \\(\mu_T\\)，其直积的 Möbius 函数为 \\(\mu\\)，则有：

\\[\forall s,s'\in S,t,t'\in T:\mu((s,t),(s',t'))=\mu_S(s,s')\mu_T(t,t')\\]

*还是归纳法证明：*

\begin{equation*}
\begin{aligned}
\mu((s,t),(s',t'))= & -\sum_{(s,t)\le(s'',t'')\lt(s',t')}\mu((s,t),(s'',t'')) \\\\
 = & -\sum_{(s,t)\le(s'',t'')\lt(s',t')}\mu_S(s,s'')\mu_T(t,t'') \\\\
 = & \mu_S(s,s')\mu_T(t,t')-\sum_{s\le_{\scriptstyle S}s''\le_{\scriptstyle S}s'}\mu_S(s,s'')\sum_{t\le_{\scriptstyle T}t''\le_{\scriptstyle T}t'}\mu_T(t,t'') && \triangleright 提取 (s'',t'')=(s',t') 一项 \\\\
 = & \mu_S(s,s')\mu_T(t,t') && \triangleright\sum_{i\le k\le j}\mu(i,k)=\delta(i,j)
\end{aligned}
\end{equation*}

这个结论将会在接下来的求解自然数关于整除关系形成的偏序集上的 Möbius 函数用到。

### \\((\mathbb N,|)\\)

对于任意定义域中的 \\(x,y\\)，当 \\(x|y\\)，有 \\(\mu(x,y)=\mu\left(1,\dfrac yx\right)\\)。这个结论是很显然的，证明的话归纳即可，两侧参数相等时结论显然，不等时：

\begin{equation*}
\begin{aligned}
\mu(x,y) & =-\sum_{z|\frac yx\land z\neq\frac yx}\mu(x,xz) \\\\
& =-\sum_{z|\frac yx\land z\neq\frac yx}\mu(1,z) \\\\
& =\mu\left(1,\frac yx\right)
\end{aligned}
\end{equation*}

设 \\(\mathbb P_m\\) 为 \\(m\\) 的素因数集合，则有：

\begin{equation*}
\mu(1,m)=\left\\{\begin{align*}
& 1 && \text{for }m=1 \\\\
& (-1)^{|\mathbb P_m|} && \text{for }m=\prod\limits_{p\in \mathbb P_m}p \\\\
& 0 && \text{otherwise}
\end{align*}\right.
\end{equation*}

*证明：*

设 \\(n\\) 的唯一素因数分解为 \\(\displaystyle n=\prod_{k}p_k^{\alpha_k}\\)，\\(p_k\\) 为 \\(n\\) 的第 \\(k\\) 个素因子，\\(\alpha_k\\) 是 \\(p_k\\) 对应的幂次。

对于任何可以整除 \\(n\\) 的数字 \\(s=\prod\limits_{k}p_k^{\beta_k}, t=\prod\limits_{k}p_k^{\gamma_k}\\) 来说，若 \\(s|t\\)，有 \\(\forall k:\beta_k\le\gamma_k\\)。所以 \\(n\\) 的因子组成的偏序集可以认为是 \\(k\\) 个 \\(\alpha_k+1\\) 大小的偏序集的直积。

由于每个子偏序集都是只由 \\(p_k\\) 的幂次组成，实际上这些都是全序集。前面已经证明了对于所有的全序集的 \\(\mu\\) 函数情况，此处同理：

\begin{equation*}
\mu(1,p_k^r)=\left\\{\begin{align*}
& 1  && \text{for }r=0 \\\\
& -1 && \text{for }r=1 \\\\
& 0  && \text{otherwise} \\\\
\end{align*}\right.
\end{equation*}

于是利用 Möbius 直积公式可证明结论。

当我们只考虑 Möbius 函数的第一个参数为 1 并略去时，得到的形式即为数论中的 Möbius 函数，数论 Möbius 函数满足积性 \\(m\bot n\implies\mu(mn)=\mu(m)\mu(n)\\)。前几项的值如下图所示：

![](https://upload.wikimedia.org/wikipedia/commons/c/c8/Moebius_mu.svg)

## Möbius 反演

设对于 \\(F,G\in\mathcal F(D,\le)\\) 两个函数满足： \\(\displaystyle G(x,y)=\sum_{x\le z\le y}F(x,z)\\)，则有 \\(\displaystyle F(x,y)=\sum_{x\le z\le y}G(x,z)\mu(z,y)\\)。

*证明：*

\begin{equation*}
\begin{align*}
\sum_{x\le z\le y}G(x,z)\mu(z,y)& =\sum_{x\le z\le y}\mu(z,y)\sum_{x\le w\le z}F(x,w) \\\\
& =\sum_{x\le z\le y}\mu(z,y)\sum_{w\in D}F(x,w)\zeta(w,z) \\\\
& =\sum_{w\in D}F(x,w)\sum_{w\le z\le y}\zeta(w,z)\mu(z,y) \\\\
& =\sum_{w\in D}F(x,w)\delta(w,y) \\\\
& =F(x,y)
\end{align*}
\end{equation*}

上面是一般的 Möbius 反演公式。在偏序集具有最小元（下面假设为 \\(0\\)）且函数下界取到最小元的时候，二元函数可以默认第一个参数为 \\(0\\) 简写为一元函数，也就是 *Introductory Combinatorics Fifth Edition* 上的形式：

> 当 \\(F,G\in\mathcal F(D,\le)\\) 满足 \\(G(0,x)=\sum\limits_{y\le x}F(0,y)\\) 时，有 \\(F(0,x)=\sum\limits_{y\le x}G(0,y)\mu(y,x)\\)。

在数论中，存在最小元为 \\(1\\)，Möbius 反演可以化为另一种形式：

\\[G(n)=\sum_{k|n}F(k)\iff F(n)=\sum_{k|n}\mu\left(\dfrac nk\right)G(k)\\]

### 容斥原理

设有集合 \\(A\_1,A\_2,A\_3,A\_4,\cdots,A\_n\\)，求 \\(\displaystyle\left|\bigcup_{k=1}^nA_k\right|\\)

利用容斥原理，我们知道这个的结果为 \\(\displaystyle\left|\bigcup_{k=1}^nA_k\right|=\sum_{\mathcal T\subseteq\\{A_1,A_2,\ldots,A_n\\}\land\mathcal T\neq\varnothing}(-1)^{|\mathcal T|-1}\left|\bigcap _{A\in\mathcal T}A\right|\\)

*证明：*

设集合族 \\(\mathcal A=\\{A_1,A_2,A_3,A_4,\cdots,A_n\\}\\)，全集 \\(\displaystyle U=\bigcup_{A\in\mathcal A}A\\)，\\(\mathcal S\\) 为 \\(\mathcal A\\) 的子集，构造函数

\begin{equation*}
\begin{align*}
F(\mathcal S)&=\left|\left(\bigcap_{A\in\mathcal S}A\right)\cap\left(\bigcap_{A\in\mathcal A\setminus\mathcal S}U\setminus A\right)\right| \\\\
G(\mathcal T)&=\left|\bigcup_{A\in\mathcal T}A\right| \\\\
\end{align*}
\end{equation*}

这里，\\(\displaystyle\bigcap_{A\in\mathcal S}A\\) 的意思是 \\(\mathcal S\\) 中所有集合都有的元素；\\(\displaystyle\bigcap_{A\in\mathcal A\setminus\mathcal S}U\setminus A\\) 是 \\(\mathcal S\\) 以外的集合都没有的元素，我们取它们的并集。换句话说，函数 \\(F(\mathcal S)\\) 的结果是所有 \\(\mathcal S\\) 中集合的交减去不在 \\(\mathcal S\\) 中的集合的并。因此 \\(F\\) 等价于

\begin{equation*}
\begin{align*}
& F(\mathcal S)=\left|\left(\bigcap_{A\in\mathcal S}A\right)\setminus\left(\bigcup_{A\in\mathcal A\setminus\mathcal S}A\right)\right| \\\\
\end{align*}
\end{equation*}

从元素的视角去理解函数 \\(F\\) 更简单一些，对于一个元素，当且仅当所有包含它的集合都在 \\(\mathcal S\\) 中，所有不包含它的集合都不在 \\(\mathcal S\\) 中，这个元素才对 \\(F(\mathcal S)\\) 有贡献。于是我们可以得出一个结论：对于所有 \\(\mathcal A\\) 的子集 \\(\mathcal S\\)，函数 \\(F\\) 右侧的 \\(\displaystyle \left(\bigcap_{A\in S}A\right)\setminus\left(\bigcup_{A\in\mathcal A\setminus\mathcal S}A\right)\\)，是对 \\(U\\) 的一个划分。那么自然的，\\(\displaystyle\sum_{\mathcal S\subseteq\mathcal A}F(\mathcal S)=|U|\\)。

稍作拓展，可以得到一个结论： \\(\displaystyle G(\mathcal T)=\sum_{\mathcal S\supseteq\mathcal T}F(\mathcal S)\\)。这很好理解，当 \\(\mathcal S\\) 中始终包含某些集合时，这些集合的元素总会出现在某个 \\(F(\mathcal S)\\) 中，因为它是划分。

到这里已经明确了 \\(F\\) 的性质，回过头来，这是一个关于 \\(\supseteq\\) 的偏序集。我们已经得到 \\(\subseteq\\) 偏序关系的 Möbius 函数，通过引入一个不加证明（很容易证明）的结论： \\(\mu_\le(a,b)=\mu_\ge(b,a)\\)，可以直接得到 \\(\supseteq\\) 的 Möbius 函数。

运用 Möbius 反演可以得到： \\(\displaystyle F(\mathcal S)=\sum_{\mathcal A\supseteq\mathcal T\supseteq\mathcal S}\mu(\mathcal T,\mathcal S)G(\mathcal T)=\sum_{\mathcal A\supseteq\mathcal T\supseteq\mathcal S}(-1)^{|\mathcal T|-|\mathcal S|}G(\mathcal T)\\)

接下来取 \\(\mathcal S=\varnothing\\)，有 \\(\displaystyle F(\varnothing)=\left|U\setminus\left(\bigcup_{A\in\mathcal A}A\right)\right|\\)

回代得到结论： \\(\displaystyle\left|U\setminus\left(\bigcup_{A\in\mathcal A}A\right)\right|=\sum_{\mathcal A\supseteq\mathcal T}(-1)^{|\mathcal T|}\left|\bigcap_{A\in\mathcal T}A\right|\\)

当把全集 \\(U\\) 的影响去掉之后，就得到本段开始所写的容斥原理公式。

### 二项式反演

二项式反演的公式是 \\(\displaystyle g(x)=\sum_{y=0}^x(-1)^y\binom xyf(y)\iff f(x)=\sum_{y=0}^x(-1)^y\binom xyg(y)\\)

上面从 Möbius 反演出发证明了容斥原理，本节试图从容斥原理构造特殊的集族证明二项式反演。

构造 \\(n\\) 个集合，使得 \\(\forall k\in\\{1,2,3,4,\cdots,n\\}\\) 满足任选 \\(k\\) 个集合的交均为固定值，设为 \\(g(k)\\)。这样的集族挺好构造，但是有一些特殊情况，比如这个公式对于负数也是成立的（而且大部分你构造不出来的情况，就是 \\(g\\) 或者 \\(f\\) 里存在负数），但是没法用这个证明，这里就……懒得管了。

有

\begin{equation*}
\begin{align*}
g(k)&=|A_1\cap A_2\cap A_3\cap A_4\cap\cdots\cap A_k| \\\\
&=\left|\overline{\overline{A_1}\cup\overline{A_2}\cup\overline{A_3}\cup\overline{A_4}\cup\cdots\cup\overline{A_k}}\right|
\end{align*}
\end{equation*}

设

\begin{equation*}
\begin{align*}f(k)
&=\left|\overline{A_1}\cap\overline{A_2}\cap\overline{A_3}\cap\overline{A_4}\cap\cdots\cap\overline{A_k}\right| \\\\
&=\left|\overline{A_1\cup A_2\cup A_3\cup A_4\cup\cdots\cup A_k}\right|
\end{align*}
\end{equation*}

应用容斥原理，直接可得 \\(\displaystyle g(x)=\sum_{y=0}^x(-1)^y\binom xyf(y)\\)，\\(\displaystyle f(x)=\sum_{y=0}^x(-1)^y\binom xyg(y)\\)

当然我们也可以从定义出发证明二项式反演，和证明 Möbius 反演的过程类似，只需要证明 \\(\displaystyle{\sum_{k=m}^n(-1)^{k-m}\binom nk\binom km=\delta(n,m)}\\) 即可。事实上，只要满足 \\(\displaystyle{\sum_{k=m}^n(-1)^{k-m}A(n,k)B(k,m)=\delta(n,m)}\\) 形式的数列 \\(A,B\\) 均有可反演的性质。

除了二项式系数自成一对外，还有 \\(\displaystyle\sum_{k=m}^n(-1)^{k-m}\genfrac[]{0pt}0nk\genfrac\\{\\}{0pt}0km=\delta(m,n)\\)、\\(\displaystyle\sum_{k=m}^n(-1)^{k-m}\genfrac\\{\\}{0pt}0nk\genfrac[]{0pt}0km=\delta(m,n)\\) 和 \\(\displaystyle\sum_{k=m}^n(-1)^{k-m}\genfrac\lfloor\rfloor{0pt}0nk\genfrac\lfloor\rfloor{0pt}0km=\delta(m,n)\\)。上面的式子中，\\(\genfrac[]{0pt}0{}{}\\) 表示[第一类 Stirling 数](https://en.wikipedia.org/wiki/Stirling_numbers_of_the_first_kind)，\\(\genfrac\\{\\}{0pt}0{}{}\\) 表示[第二类 Stirling 数](https://en.wikipedia.org/wiki/Stirling_numbers_of_the_second_kind)，\\(\genfrac\lfloor\rfloor{0pt}0{}{}\\) 表示 [Lah 数](https://en.wikipedia.org/wiki/Lah_number)。

### 数论反演

其实就是定义在 \\((\mathbb N,|)\\) 偏序集上的函数的 Möbius 反演。

对于数论函数 \\(f,g\\)，有 \\(\displaystyle g(n)=\sum_{d|n}f(d)\iff f(n)=\sum_{d|n}\mu(d,n)g(d)\\)

因为前面证明过 \\(\mu(a,b)=\mu\left(1,\dfrac ba\right)\\) 而且默认情况下会把 \\(\mu(1,m)\\) 表示成 \\(\mu(m)\\)，上式就可以化为网上常见的一般的形式，也就是 Dirichlet 卷积形式。
