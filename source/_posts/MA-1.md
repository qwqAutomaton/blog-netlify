---
title: 数学分析学习笔记 (I)
date: 2024-08-20 16:54:19
tags:
  - 数学
  - 学习笔记
description: 不可以摸鱼了！于是开始看一点数分然后乱口胡 qwq
---

# 数学分析笔记 (1)


## 写在前面 & 记号约定

书用的是华东师范编的第五版（高等教育出版社）。

然后约定一下记号！

- $\cap,\cup,\backslash$ 分别表示集合的交集、并集和差集；

- $\mathbb N=\{0,1,2,\cdots\}$ 表示**自然数集**（有包括 $0$ 的！）。

- $\mathbb Z,\mathbb Q,\mathbb Q^c,\mathbb R$ 分别表示整数集、有理数集、无理数集、实数集。

- $\mathbb Z_+,\mathbb Z_-,\mathbb Z^\times$ 分别表示 $\mathbb Z$ 中正、负、非零的部分（即正整数、负整数、非零整数）。其他集合同理。

- 在小数部分的上面加一个点表示循环小数，如
  
  $$
  \begin{aligned}
  1.{\color{red}\dot 2}&=1.\color{red}222\cdots\\
  1.2{\color{red}\dot 3}&=1.2\color{red}333\cdots\\
  1.{\color{red}\dot2\dot3}&=1.\color{red}232323\cdots\\
  1.2{\color{red}\dot34\dot 5}&=1.2\color{red}345343545\cdots
  \end{aligned}
  $$

## 实数

### 前置：有理数的表示等等

数分研究的是定义在实数集上的函数，因此有必要先研究实数本身的性质。

有理数的定义是明确的。**有理数**可以表示为分数形式 $\dfrac pq$，其中 $p\in \mathbb Z,q\in \mathbb Z^*$；也可以表示为**有限十进小数**或**无限十进循环小数**的形式。那么对应过来，**无理数**就是**无限十进不循环小数**。于是无理数和有理数就统称为**实数**。

然而后面关于实数的讨论，因为涉及了无理数，没办法兼容有理数的分数表示和有限十进制表示。因此这里需要先把有限小数表示成无限小数的形式，这样的话所有实数都可以用无限小数的形式处理了。

对于正的有理数 $x$，如果它是有限小数（当然也包括整数），也就是说

$$
x=a_0.a_1a_2\cdots a_n=a_0+a_1\times 10^{-1}+a_2\times 10^{-2}+\cdots+a_n\times 10^{-n}
$$

其中 $a_n\neq 0,a_0\in \mathbb N$，且对于所有的 $i=1,2,\cdots,n$ 有 $a_i\in[0,9]\cap\mathbb N$（好吧其实就是把这个小数用十进制形式写出来）。那么我们把它表示成

$$
x=a_0.a_1a_2\cdots (a_n-1)9999\cdots
$$

特别地，当 $x=a_0$ 为一个整数时，将其表示为

$$
x=(a_0-1).9999\cdots
$$

而对于负有理数 $y$，则先将 $(-y)>0$ 表示成无限小数的形式，然后在所得的无限小数前面加一个负号。

对于零而言，规定将其表示成

$$
0=0.000\cdots=0.\dot 0
$$

举个例子：

- $5=4.999\cdots=4.\dot 9$

- $1.132=1.131999\cdots=1.131\dot 9$

- $-3=-2.999\cdots =-2.\dot 9$

- $-0.234=-0.233999\cdots=-0.233\dot 9$

这样一来，所有的实数都可以用一个**确定**（唯一）的无限小数表示了。这对后面是很有好处滴\~

### 实数的大小比较

有理数怎么比大小，这个应该是已经知道的 ~~小学二年级，请~~。

那么下面给出两个实数 $x,y$ 之间大小比较的方法。我们把这两个实数用无限小数的形式表示。

先定义两个非负实数之间的大小关系。对于两个实数

$$
\begin{aligned}
x&=a_0.a_1a_2\cdots a_n\cdots\\
y&=b_0.b_1b_2\cdots b_n\cdots
\end{aligned}
$$

其中 $a_0,b_0\in\mathbb N$ 是非负整数，$a_k,b_k(k\in\mathbb Z_+)$ 都是 $0$ 到 $9$ 的整数。下面定义 $x$ 和 $y$ 的大小关系。

1. 我们称 $x$ 和 $y$ 相等（记作 $x=y$），如果 $a_k=b_k$ 对于所有非负整数 $k\in\mathbb N$ 成立

2. 我们称 $x$ 大于 $y$（记作 $x>y$），如果 $a_0>b_0$ 或存在某一非负整数 $l$ 使得
   
   $$
   a_k=b_k(k=0,1,\cdots,l);a_{l+1}>b_{l+1}
   $$
   
   即前 $l$ 位相同，且第 $l+1$ 位满足 $a_{l+1}>b_{l+1}$.

3. 我们称 $x$ 小于 $y$（记作 $x<y$），如果 $y>x$. （通过大于对称地定于小于）

然后定义负实数之间的大小关系。对于两个负实数 $x,y$，如果按照上面的规定，分别有 $-x=-y,-x>-y$，则分别称 $x=y,x<y$（或 $y>x$）。另外，我们规定任意非负实数都大于任意负实数。这样就定义了所有实数之间的大小比较。

### 近似

然而直接一位一位比较太麻烦了…… 考虑通过比较和原数有关的有限小数比较原数的大小。因此我们定义**过剩近似**和**不足近似**。

- 设非负实数 $x=a_0.a_1a_2\cdots a_n\cdots$，称它的 **$n$ 位不足近似**为有理数
  
  $$
  x_n=a_0.a_1a_2\cdots a_n
  $$
  
  而对于负实数 $x'=-a_0.a_1a_2\cdots a_n\cdots$，它的 $n$ 位不足近似为
  
  $$
  x'_n=-a_0.a_1a_2\cdots a_n-\dfrac 1{10^n}
  $$

- 而非负实数 $x$ 的 **$n$ 位过剩近似**为有理数
  
  $$
  \overline{x_n}=x_n+\dfrac 1{10^n}=a_0.a_1a_2\cdots a_n+\dfrac 1{10^n}
  $$

  类似地，对于负实数 $x'$，它的 $n$ 位过剩近似为

  $$
  \overline{x'_n}=x'_n+\dfrac 1{10^n}=-a_0.a_1a_2\cdots a_n
  $$

我们不难看出这两个近似的“单调性”：当 $n$ 增大时，$x$ 的不足近似 $x_n$ 不减，且总不大于 $x$，即 $x_0\le x_1 \le x_2 \le \cdots\le x$; 类似地，$x$ 的过剩近似 $\overline{x_n}$ 不增，且总不小于 $x$，即 $\overline{x_0}\ge \overline{x_1}\ge \overline {x_2}\ge \cdots \ge x$.

### 实数大小比较的等价条件和应用

利用上面的近似，可以得出两个实数之间大小比较的一个**等价条件**：

> **命题 1.1** 设 $x=a_0.a_1a_2\cdots$ 和 $y=b_0.b_1b_2\cdots$ 是两个实数。那么 $x>y$ 的一个等价条件是：存在一个非负整数 $n\in\mathbb N$，使得

$$
x_n>\overline{y_n}
$$

<details>
<summary>证明</summary>

考虑分两部分证。

由于 $x>0>y$ 时是显然的，而 $0>x>y$ 的情况很容易转化为两个正实数之间的大小比较（加负号反过来即可）。下面仅证明正实数的情况。

$\impliedby$: 根据上面近似的“单调性”，有

$$
x\ge x_n>\overline{y_n}\ge y
$$

即 $x>y$.

$\implies$: 根据大于的定义，存在 $n\in\mathbb N$ 使得

$$
a_i=b_i(i=0,1,\cdots,k-1),a_k>b_k
$$

因此，取 $n=k+1$，则有

$$
\begin{aligned}
x_n&=a_0.a_1a_2\cdots a_ka_{k+1}\\
\overline{y_n}&=b_0.b_1b_2\cdots b_kb_{k+1}+\dfrac 1{10^{k+1}}
\end{aligned}
$$

两式相减，得到

$$
\begin{aligned}
x_n-\overline{y_n}&=\quad a_k\times10^{-k}+a_{k+1}\times 10^{-k-1}\\
&\quad-b_k\times 10^{-k}-b_{k+1}\times 10^{-k-1}-10^{-k-1}\\
&=(a_k-b_k)\times 10^{-k}+(a_{k+1}-b_{k+1}-1)\times 10^{-k-1}
\end{aligned}
$$

由于 $a_k>b_k\implies a_k\ge b_k+1$（注意 $a_i,b_i\in \mathbb N$），且 $a_{k+1}\ge 0,b_{k+1}\le 9$，得到

$$
x_n-\overline{y_n}\ge 10^{-k}+10\times 10^{-k-1}=0\qquad \qquad(*)
$$

取等号当且仅当 $a_k=b_k+1,a_{k+1}=0,b_{k+1}=9$. 接下来证明这种情况能够回归到其他普通的、已经证明了的状态。

考虑接下来从 $n$ 开始逐个检查它后面的数。假设当前检查到了 $n'>n$. 那么显然 $(*)$ 式能取等当且仅当 $a_i\equiv 0,b_i\equiv 9$ 对于所有的 $n\le i\le n'$. 根据实数的无限小数表示，这种情况时不存在的。因此能够回归到普通的状态（即递归能够返回）。

~~事实上严格的证明还得用 Dedekind 分割…… 也就是得把整个实数定义一遍 qaq~~

</details>

根据这个命题，可以证明有理数在实数当中也是稠密的，即：

> **命题 1.2** 设实数 $x, y$ 满足 $x<y$. 证明：存在有理数 $r$ 满足
>
> $$
> x<r<y
> $$

<details>
<summary>证明</summary>

因为 $x<y$，所以存在非负整数 $n$ 使得 $\overline{x_n}<y_n$.

考虑有理数

$$
r=\dfrac {\overline{x_n}+y_n}2
$$

那么有

$$
r<\dfrac {y_n+y_n}2=y_n\le y
$$

同理，

$$
r>\dfrac{\overline{x_n}+\overline{x_n}}2=\overline{x_n}\ge x
$$

即证。

</details>

### 实数集及其性质

方便起见，我们记

$$
\mathbb R=\{x\mid x\text{ is real}\}
$$

是由全体实数构成的集合。那么实数集有这些性质：

1. $\mathbb R$ 对四则运算（$+,-,\times,\div$，其中除数 $\neq 0$）封闭。即对于任意两个实数 $x,y$ 满足
  
  $$
  x+y,x-y,xy,\dfrac xy(y\neq 0)\in\mathbb R
  $$

2. 实数集有序，即对于任意两实数 $x,y$，以下三者有且仅有一个成立：$x>y,x<y,x=y$.
3. 实数的序具有传递性，即若 $x<y,y<z$ 则 $x<z$. 因此将 $x<y,y<z$ 简记为 $x<y<z$.
4. 实数有 Archimedes 性，即对于任意 $y>x>0$，存在**正整数** $n$ 使得 $na>b$.
5. 实数是稠密的，即对于任意两个实数 $x<y$，存在实数 $r$ 满足 $x<r<y$，且 $r$ 既可能是有理数，也可能是无理数。
6. 存在实数轴：在一条直线上取一点 $O$ 作为原点，指定一个方向为正向，并规定单位长度，那么这条直线称为**数轴**。那么存在一个从数轴上的点到实数集的双射（一一对应）。

因为之前没有显式地定义实数的四则运算等等，性质 1 比较难证 qaq. 并且阿基米德性的证明好像也得依赖 Dedekind 分割…… 那就证一下其他几个叭 qwq

> **命题 1.3** （实数集有序）求证：对于任意两个实数 $x,y$，下面三个命题有且仅有一个成立：$x>y,x<y,x=y$.

<details>
<summary>证明</summary>

根据大于、小于和等于的定义，容易知道这三个命题互相矛盾。因此只需要证明它们当中一定有一个成立即可。

反证。如果存在一对

$$
\begin{aligned}
x&=a_0.a_1a_2\cdots\\
y&=b_0.b_1b_2\cdots
\end{aligned}
$$

使得这三个命题均不成立。如果 $a_0\neq b_0$，根据整数的有序性，可知要么 $a_0>b_0$，要么 $a_0<b_0$. 根据之前的定义，此时要么 $x<y$，要么 $x>y$. 与假设矛盾。

如果 $a_0=b_0$，根据假设，$x\neq y$，因此存在正整数 $n$ 使得

$$
a_i=b_i(i=0,1,\cdots,n-1), a_n\neq b_n
$$

那么类似上面，同样讨论 $a_n$ 和 $b_n$ 的大小关系，可以推出矛盾。

因此假设不成立，原命题成立，即实数集是有序的。

</details>

> **命题 1.4** （实数的序的传递性）若三个实数 $x,y,z$ 满足 $x<y$ 且 $y<z$，那么 $x<z$.

<details>
<summary>证明</summary>

根据实数大小比较的等价条件（命题 1.1），存在整数 $n,m$ 使得 $\overline{x_n}<y_n,\overline{y_m}<z_m$.

取 $t=\max\{n,m\}$，根据近似的单调性，容易证明 $\overline{x_n}\ge \overline{x_t},y_n\le y_t\le y\le \overline{y_t}\le \overline{y_m},z_m\le z_t$.

那么把这一串连起来，就有

$$
\overline{x_t}\le \overline{x_n}<y_n\le y_t\le y\le \overline{y_t}\le \overline{y_m}<z_m\le z_t
$$

即 $\overline{x_t}<z_t$. 同样根据命题 1.1，得到 $x<z$.

</details>

这篇先到这里叭

去摸鱼了 qwq
