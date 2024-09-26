---
title: 北航 OJ P11 题解
date: 2024-09-26 16:24:28
tags:
  - 题解
description: 很玄学的博弈论ww
---

# Let's play a game 题解

[题目 link](https://accoding.buaa.edu.cn/problem/11/index)

实质上就是 Euclid Game.

## 思路 -1 gcd

考虑题目第一行的欧几里得，朴素地考虑辗转相除，即每次取的这个倍数 $k$ 都取它能取到的最大值（即 $\left\lfloor\dfrac mn\right\rfloor$，其中 $m>n$）：

```c
int gcd(int a, int b)
{
    if (a < b) return gcd(b, a);
    if (b == 0) return 0;
    return gcd(b, a % b) + 1;
}
```

过了样例，但是 WA 光光了，，，

~~不过这怎么想都不是最优解叭！！~~

## 思路 0 dfs

考虑暴力 dfs. 设 $\text {dfs}(n,m)$ 表示对于两个整数 $n,m$ 的游戏中先手必胜与否。那么

$$
\text {dfs}(n,m)=\begin{cases}
\text{dfs}(m,n)&&n<m\\
0&&m=0\\
\prod_{k=1}^{n/m}\text{dfs}(n-km,m)&&\text{otherwise}
\end{cases}
$$

这个应当是很显然的。然而直接这么做铁 TLE qaq

## 正解（？）

于是就来到正解（或者至少能过的解）。不妨设此时两个数 $n\ge m$. 

两种必败 / 必胜的平凡情况就不细说了（$m\mid n,m=0$ 等）。考虑前面 $\text{otherwise}$ 里的情况，这个倍数 $k$ 的情况。

一种朴素的分类：$k=1$ 和 $k\neq 1$. 前者表示当前玩家没有选择，只能减去一个 $m$，因此可以直接返回 $\text {dfs}(m,n-m)$. 那么后面的情况呢？

注意到后面的情况如果必败（返回 $0$），当且仅当对于所有可行的倍数 $k$ 都有 $\text{dfs}(n-km,m)=1$ 即减去所有的倍数后都必胜（此时转换了角色）。

然而这个必败的条件是非常苛刻的，因此猜测这种情况不存在。实际上可以这么证明（不知道够不够严谨 qaq）：

对弈当前的局面 $(n,m)$，先手肯定能够预先算出（因为都取最优解）尽量减掉之后的局面 $(m, n\bmod m)$ 对于对手而言是否必胜。如果不是（也就是必败），那就取成这种情况；否则，先手将其取为 $(n\bmod m+m,m)$ 的情况，那么此时对手只有一种情况，且这种情况对对手而言是必败的。因此不论怎样，先手都必胜。

那么正解就不难写咯~

```c
// sg(n, m) = 1 表示先手 (Nova) 必胜
int sg(int n, int m)
{
    if (n < m)
        return sg(m, n);
    if (m == 0)
        return 0;
    if (n % m == 0)
        return 1;
    if (n / m >= 2)
        return 1;
    return sg(m, n - m) ^ 1; // 交换先后手
}
```
