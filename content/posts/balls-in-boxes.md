+++
title = "小球放到盒子里的排列组合问题"
date = "2023-06-04"
type = "post"
tags = ["Mathematics"]
+++

这也算是个讲烂的话题了，网上搜到一堆。前几天刚复习到了，本想随手水一篇文章，结果又写长了。

其实这篇文章本意是想测试一下评论功能，所以一开始只有下面这张表格。但是还没开始写评论功能就已经上线了，后面就索性补充了一堆解释和证明。

相比于市面上的文章，自认为还是有所拓展的，也就是读完可能会有一些新的收获。比如说对于编号 011，提出了除插板法之外的另一种思路，并且证明两者是等价的；对于 111，也提出了除了最基础的乘法原理之外的另一种思路，并借此引入了一个第二类斯特林数的恒等式。

假设有 \\(n\\) 球 \\(m\\) 盒子：

| 编号 | 球不同 | 盒不同 | 盒可空 | 方案数 | 备注 |
|:-:|:-:|:-:|:-:|:-:|:-:|
| [000](#编号-000球相同盒相同盒非空) | ❌ | ❌ | ❌ | \\(p(n,m)\\) | \\(\displaystyle p(n)=\sum_{k=1}^np(n, k)\\) 是整数分拆函数 |
| [001](#编号-001球相同盒相同盒可空) | ❌ | ❌ | ✅ | \\(\displaystyle\sum_{k=1}^mp(n,k)\\) | |
| [010](#编号-010球相同盒不同盒非空) | ❌ | ✅ | ❌ | \\(\dbinom{n-1}{m-1}\\) | 插板法 |
| [011](#编号-011球相同盒不同盒可空) | ❌ | ✅ | ✅ | \\(\dbinom{n+m-1}{m-1}\\) | 插板法 |
| [100](#编号-100球不同盒相同盒非空) | ✅ | ❌ | ❌ | \\(\genfrac\\{\\}{0pt}0nm\\) | \\(\genfrac\\{\\}{0pt}0nm\\) 是第二类斯特林数 |
| [101](#编号-101球不同盒相同盒可空) | ✅ | ❌ | ✅ | \\(\displaystyle\sum_{k=1}^m\genfrac\\{\\}{0pt}0nk\\) | |
| [110](#编号-110球不同盒不同盒非空) | ✅ | ✅ | ❌ | \\(m!\genfrac\\{\\}{0pt}0nm\\) | |
| [111](#编号-111球不同盒不同盒可空) | ✅ | ✅ | ✅ | \\(m^n\\) | |

## 整数分拆函数

\\(p(n)\\) 是**整数分拆函数**，它描述的问题是

> 将正整数 \\(n\\) 拆成任意正整数之和的方案数。

\\(p(n, m)\\) 是对这个函数的拓展，含义是

> 将正整数 \\(n\\) 拆成刚好 \\(m\\) 个正整数之和的方案数。

定义就是[编号 000](#编号-000球相同盒相同盒非空) 问题本身，事实上两者是同一个问题的不同描述。

根据定义我们有 \begin{equation*}p(n)=\sum_{k=1}^np(n, k)\end{equation*}

\\(p(n,m)\\) 的递推公式 \begin{equation*}
    p(n,m)=\begin{cases}
        p(n-1,m-1)+p(n-m,m) & \text{if }0\lt m\lt n \\\\
        \\\\
        1 & \text{if }m=n \\\\
        \\\\
        0 & \text{otherwise}
    \end{cases}
\end{equation*}

这个递推式描述了 \\(n\\) 个球放进 \\(m\\) 个盒子的两种可能：

- 存在一个盒子只有一个球，等价于将剩下 \\(n-1\\) 个球装进 \\(m-1\\) 个盒子，有 \\(p(n-1,m-1)\\) 种方案；

- 所有盒子都至少有两个球，也就是我们先拿走 \\(m\\) 个球每个盒子装一个，问题就变成了仍然是 \\(m\\) 个盒子但是只有 \\(n-m\\) 个球，每个盒子至少有一个球，也就是 \\(p(n-m,m)\\) 的定义；

很显然这两种可能是互斥的，所以将 \\(n\\) 个球放进 \\(m\\) 个盒子里有 \\(p(n,m)=p(n-1,m-1)+p(n-m,m)\\) 种方案。边界条件还是显而易见的，不再赘述。

部分分拆数：

\begin{equation*}
\begin{array}{c|c|}
\bm{p(n)} & \bm{n\backslash m} & \bm0 & \bm1 & \bm2 & \bm3 & \bm4 & \bm5 & \bm6 & \bm7 & \bm8 & \bm9 & \bm{10} \\\\
\hline
1 & \bf0 & 1 \\\\
1 & \bf1 & 0 & 1 \\\\
2 & \bf2 & 0 & 1 & 1 \\\\
3 & \bf3 & 0 & 1 & 1 & 1 \\\\
5 & \bf4 & 0 & 1 & 2 & 1 & 1 \\\\
7 & \bf5 & 0 & 1 & 2 & 2 & 1 & 1 \\\\
11 & \bf6 & 0 & 1 & 3 & 3 & 2 & 1 & 1 \\\\
15 & \bf7 & 0 & 1 & 3 & 4 & 3 & 2 & 1 & 1 \\\\
22 & \bf8 & 0 & 1 & 4 & 5 & 5 & 3 & 2 & 1 & 1 \\\\
30 & \bf9 & 0 & 1 & 4 & 7 & 6 & 5 & 3 & 2 & 1 & 1 \\\\
42 & \bf10 & 0 & 1 & 5 & 8 & 9 & 7 & 5 & 3 & 2 & 1 & 1 \\\\
\end{array}
\end{equation*}

另外提到分拆数的算法类文章几乎肯定会提到生成函数和五边形数（因为可以优化复杂度），然而此文长度已经大大超出我的预期，这里便不延伸了。后续如果有时间可能会新开一篇文章再论（应该不会）。

## 第二类斯特林数

\\(\genfrac\\{\\}{0pt}0nm\\) 是**第二类斯特林数**，其定义就是

> 将 \\(n\\) 个两两不同的元素，划分为 \\(m\\) 个互不区分的非空子集的方案数。

所以[编号 100](#编号-100球不同盒相同盒非空) 的问题恰好可以用第二类斯特林数解决。

第二类斯特林数的递推公式 \begin{equation*}
    \genfrac\\{\\}{0pt}0nm=\begin{cases}
        \genfrac\\{\\}{0pt}0{n-1}{m-1}+m\genfrac\\{\\}{0pt}0{n-1}m & \text{if }0\lt m\lt n \\\\
        \\\\
        1 & \text{if }m=n \\\\
        \\\\
        0 & \text{otherwise}
    \end{cases}
\end{equation*}

对于递推式，它的理解方式和分拆数有所不同，它考虑的是如何从上一个状态转移到当前状态。分类讨论，当插入一个新元素时，有两类方案：

- 将新元素单独放入一个子集，则其中一个集合只有该新元素，剩下 \\(n-1\\) 个元素放到 \\(m-1\\) 个集合有 \\(\genfrac\\{\\}{0pt}0{n-1}{m-1}\\) 种可能；

- 将新元素放入一个现有的非空子集，对于 \\(n-1\\) 个元素划分为 \\(m\\) 个集合的每种情况，新元素都有 \\(m\\) 个集合可以放入，因此有 \\(m\genfrac\\{\\}{0pt}0{n-1}m\\) 种可能。

边界条件的 0 和 1 两种是显而易见的，不再赘述。

部分斯特林数：
\begin{equation*}
\begin{array}{c|}
\bm{n\backslash m} & \bf0 & \bf1 & \bf2 & \bf3 & \bf4 & \bf5 & \bf6 & \bf7 & \bf8 & \bf9 & \bf10 \\\\
\hline
\bf0 & 1 \\\\
\bf1 & 0 & 1 \\\\
\bf2 & 0 & 1 & 1 \\\\
\bf3 & 0 & 1 & 3 & 1 \\\\
\bf4 & 0 & 1 & 7 & 6 & 1 \\\\
\bf5 & 0 & 1 & 15 & 25 & 10 & 1 \\\\
\bf6 & 0 & 1 & 31 & 90 & 65 & 15 & 1 \\\\
\bf7 & 0 & 1 & 63 & 301 & 350 & 140 & 21 & 1 \\\\
\bf8 & 0 & 1 & 127 & 966 & 1701 & 1050 & 266 & 28 & 1 \\\\
\bf9 & 0 & 1 & 255 & 3025 & 7770 & 6951 & 2646 & 462 & 36 & 1 \\\\
\bf10 & 0 & 1 & 511 & 9330 & 34105 & 42525 & 22827 & 5880 & 750 & 45 & 1 \\\\
\end{array}
\end{equation*}

## 编号 000：球相同、盒相同、盒非空

参考[整数分拆函数](#整数分拆函数)的定义，方案数为 \\(p(n,m)\\)。

## 编号 001：球相同、盒相同、盒可空

和[编号 000](#编号-000球相同盒相同盒非空) 类似，但是盒子可以为空。这里分类讨论：

当有且只有 0 个盒子为空的时候，有 \\(p(n,m)\\) 种方案；

当有且只有 1 个盒子为空的时候，有 \\(p(n,m-1)\\) 种方案；

当有且只有 2 个盒子为空的时候，有 \\(p(n,m-2)\\) 种方案；

以此类推，总方案数为 \\(\displaystyle\sum_{k=1}^mp(n,k)\\)。

## 编号 010：球相同、盒不同、盒非空

根据插板法，\\(n\\) 个小球放入 \\(m\\) 个盒子，一共有 \\(\dbinom{n-1}{m-1}\\) 种方案。

这里举例解释插板法：将 \\(n\\) 个小球排在桌子上，比如下面 \\(n=5\\)：

\\[\circ\qquad\circ\qquad\circ\qquad\circ\qquad\circ\\]

中间有 4 个空位置：

\\[\circ\underset\wedge\qquad\circ\underset\wedge\qquad\circ\underset\wedge\qquad\circ\underset\wedge\qquad\circ\\]

如果我们需要将其分进 3 个盒子，即 \\(m=3\\)，相当于要在上面空位中插入两个板，将其分隔成三份，如其中一种方案：

\\[\circ\quad\mid\quad\circ\qquad\circ\quad\mid\quad\circ\qquad\circ\\]

将其分成了 3 份：1, 2, 2，分别装入三个箱子中。

那么此处一共 \\(n-1=4\\) 个空位，需要选中 \\(m-1=2\\) 个空位插板，因此一共有 \\(\dbinom 42=6\\) 种方案。

\\[
\begin{array}{c|ccccccccccc|c}
    1 & \quad & \red\circ & \quad\mid\quad & \green\circ & \quad\mid\quad & \color{royalblue}\circ & \qquad & \color{royalblue}\circ & \qquad & \color{royalblue}\circ & \quad & \quad (\red1,\green1,\textcolor{royalblue}3) \\\\
    2 & \quad & \red\circ & \quad\mid\quad & \green\circ & \qquad & \green\circ & \quad\mid\quad & \color{royalblue}\circ & \qquad & \color{royalblue}\circ & \quad & \quad (\red1,\green2,\textcolor{royalblue}2) \\\\
    3 & \quad & \red\circ & \quad\mid\quad & \green\circ & \qquad & \green\circ & \qquad & \green\circ & \quad\mid\quad & \color{royalblue}\circ & \quad & \quad (\red1,\green3,\textcolor{royalblue}1) \\\\
    4 & \quad & \red\circ & \qquad & \red\circ & \quad\mid\quad & \green\circ & \quad\mid\quad & \color{royalblue}\circ & \qquad & \color{royalblue}\circ & \quad & \quad (\red2,\green1,\textcolor{royalblue}2) \\\\
    5 & \quad & \red\circ & \qquad & \red\circ & \quad\mid\quad & \green\circ & \qquad & \green\circ & \quad\mid\quad & \color{royalblue}\circ & \quad & \quad (\red2,\green2,\textcolor{royalblue}1) \\\\
    6 & \quad & \red\circ & \qquad & \red\circ & \qquad & \red\circ & \quad\mid\quad & \green\circ & \quad\mid\quad & \color{royalblue}\circ & \quad & \quad (\red3,\green1,\textcolor{royalblue}1) \\\\
\end{array}
\\]

## 编号 011：球相同、盒不同、盒可空

最好先看[编号 010](#编号-010球相同盒不同盒非空) 的插板法解释。

因为盒子可空，而直接使用插板法只能解决非空的同类问题，因此需要做一下变换。

考虑先在旧球堆里混入 \\(m\\) 个新球，因为球是相同的，所以不需要标记哪些球是新混入的，只需要记住混入了 \\(m\\) 个球即可。

对 \\(n+m\\) 个球执行插板法，得到 \\(\dbinom{n+m-1}{m-1}\\) 个方案。这里顺带一提根据组合数的性质有 \\(\dbinom{n+m-1}{m-1}=\dbinom{n+m-1}n\\)。

已知这些方案中，每个盒子里都至少有一颗球。从每个盒子里拿走一颗球，因为球是相同的，所以不用考虑是否为新混入的球（拿走新球和拿走旧球的两个方案是等价的）。

现在就得到了 \\(m\\) 个可能为空的盒子。

也许读者看到这里可能还有一个小疑问，为什么这里不能和上面类似，对每个盒子是否为空进行讨论呢？答案是可以，但是太麻烦了。因为盒子不同，A 盒非空和 B 盒非空是两种方案，所以我们还需要对每一项乘以组合数（从全部盒子中选出置空盒子的方案数）。最后算出来答案是相同的，只是用插板法更好理解。下面简单推理一下对每个盒子是否为空进行讨论的简化过程：

\begin{equation*}
    \begin{align*}
        &\sum_{k=1}^m\binom mk\binom{n-1}{k-1} && \triangleright m\ 个盒子里选\ k\ 个非空，再将\ n\ 个球装入\ k\ 个非空盒子的方案数求和 \\\\
        =&\sum_{k=1}^m\binom m{m-k}\binom{n-1}{k-1} && \triangleright 二项式系数的对称性 \\\\
        =&\sum_{k=0}^{m-1}\binom m{m-1-k}\binom{n-1}k && \triangleright 调整了一下迭代器\ k\ 的范围 \\\\
        =&\binom {n+m-1}{m-1} && \triangleright 范德蒙恒等式
    \end{align*}
\end{equation*}

## 编号 100：球不同、盒相同、盒非空

参考[第二类斯特林数](#第二类斯特林数)的定义，方案数为 \\(\genfrac\\{\\}{0pt}0nm\\)。

## 编号 101：球不同、盒相同、盒可空

和[编号 100](#编号-100球不同盒相同盒非空) 类似，但是盒子可以为空。因为盒子相同，所以这里也可以进行分类讨论：

考虑没有盒子为空的情况，则等于 \\(\genfrac\\{\\}{0pt}0nm\\)。

考虑只有一个盒子为空的情况，等于 \\(\genfrac\\{\\}{0pt}0n{m-1}\\)。

考虑只有两个盒子为空的情况，等于 \\(\genfrac\\{\\}{0pt}0n{m-2}\\)。

依此类推，总方案数为 \\(\displaystyle\sum_{k=1}^{m}\genfrac\\{\\}{0pt}0nk\\)。

## 编号 110：球不同、盒不同、盒非空

首先理解[第二类斯特林数](#第二类斯特林数) \\(\genfrac\\{\\}{0pt}0nm\\) 是[编号 100](#编号-100球不同盒相同盒非空) 情况的解。

那么对于盒子不同的情况，相当于将[编号 100](#编号-100球不同盒相同盒非空) 的所有方案，每个盒子编号后重新排列，每个排列就是本问题的一种方案。

排列共有 \\(m!\\) 种，所以共有 \\(m!\genfrac\\{\\}{0pt}0nm\\) 种方案。

## 编号 111：球不同、盒不同、盒可空

因为盒子是不相同的，所以对于每颗球，都有 \\(m\\) 个盒子可以放，因此根据乘法原理，共有 \\(m^n\\) 种方案。

当然和[编号 110](#编号-110球不同盒不同盒非空) 将[编号 100](#编号-100球不同盒相同盒非空) 的盒子重排列得到答案的逻辑类似。本问题也可以通过[编号 101](#编号-101球不同盒相同盒可空) 的结论重排列的方式推导过来，但是两者存在一个差异，那就是**两个均为空的盒子是不能参与重排列的**，因为在一个排列中将它们交换得到的两个方案并无不同。

注意到在 110 和 100 中盒子均非空，而球和盒均各不相同。所以在一个排列中，任何两个盒子 A 和 B 中均有球，记为球 1 和球 2，而球在 A 盒或 B 盒肯定是两种方案 A1B2 和 A2B1，所以不论哪种方案都有 \\(m!\\) 种排列。

所以这里存在着一些差异，那就是我们在从 \\(m\\) 个盒子中选定 \\(k\\) 个非空盒子的时候，还需要对其进行排列，也就是 \\(m^{\underline k}\\) 种。这里 \\(m^{\underline k}\\) 是递降阶乘，定义为 \\(m^{\underline k}=\dfrac{m!}{(m-k)!}=m(m-1)(m-2)\cdots(m-k+1)\\)（就是排列数 \\(m\\) 取 \\(k\\)）。

从这里可以从逻辑上推导出另一个关于[第二类斯特林数](#第二类斯特林数)的恒等式：

\begin{equation*}
\sum_{k=1}^mm^{\underline k}\genfrac\\{\\}{0pt}0nk=m^n
\end{equation*}

如果你打开下面的参考资料第二类斯特林数，你会发现维基百科中写的是 \\(\displaystyle\sum_{k=1}^n\genfrac\\{\\}{0pt}0nk(x)_k=x^n\\)，和上面的唯一区别是 \\(\displaystyle\sum\\) 迭代器 \\(k\\) 是到 \\(n\\) 为止的。不要紧，只要 \\(k\gt n\\) 或者 \\(k\gt m\\) 满足任意一个，结果都是 \\(0\\)，所以两者是等价的，这里其实应该是 \\(\min(n,m)\\)。

## 参考资料

- <a href="https://oi-wiki.org/math/combinatorics/partition/" target="_blank">分拆数 - OI Wiki</a>

- <a href="https://en.wikipedia.org/wiki/Stirling_numbers_of_the_second_kind" target="_blank">（英文）第二类斯特林数 - 维基百科</a>

- <a href="https://en.wikipedia.org/wiki/Partition_function_(number_theory)" target="_blank">（英文）分拆数（数论） - 维基百科</a>

- <a href="https://en.wikipedia.org/wiki/Binomial_coefficient" target="_blank">（英文）二项式系数 - 维基百科</a>
