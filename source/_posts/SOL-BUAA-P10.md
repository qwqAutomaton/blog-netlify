---
title: 北航 OJ P10 题解
date: 2024-09-19 17:49:39
tags:
  - 题解
description: C 写矩阵快速幂好难受……
---

# 零崎的人间冒险II 题解

[题目 link](https://accoding.buaa.edu.cn/problem/10/index)

> Tag: 矩阵快速幂；递推

## 解题思路

考虑递推。设 $f[i][0/1]$ 表示进行 $i$ 次舍牌时，最后一次为刚（$1$）或怂（$0$）的方案数。那么根据题意，应当容易得出递推式：

$$
\begin{cases}
f[i][0]=f[i-1][0]+f[i-1][1]\\\\
f[i][1]=f[i-1][0]
\end{cases}
$$

那么总方案数就是 $f[i][0] + f[i][1]$.

然后观察初始条件，很显然 $f[1][0]=f[1][1]=1,f[2][0]=2,f[2][1]=1$. 直接按照这个式子暴力递推，复杂度 $O(n)$ 并不能接受（因为 $n\le\texttt{INT\_MAX}=2147483647$）。

考虑优化。直接将上下两式相加，得到

$$
f[i][0]+f[i][1]=f[i-1][0]+f[i-1][1]+f[i-1][0]
$$

将最后一个 $f[i-1][0]$ 再次代入（此时我们假定 $i$ 足够大）得到

$$
f[i][0]+f[i][1]=(f[i-1][0]+f[i-1][1])+(f[i-2][0]+f[i-2][1])
$$

如果令 $f[i]=f[i][0]+f[i][1]$，容易发现 $f[i]=f[i-1]+f[i-2]$；然后又有 $f[1]=2,f[2]=3$，可以看出 $f[i]=\text{Fib}[i+2]$ 是斐波那契数列的第 $i+2$ 项。

然后就可以做矩阵快速幂，这个思路可以参考 [OI-wiki 中对矩阵加速递推的文章](https://oi-wiki.org/math/linear-algebra/matrix/#%E7%9F%A9%E9%98%B5%E5%8A%A0%E9%80%9F%E9%80%92%E6%8E%A8)。不过需要注意的是此时幂次直接用 $n$ 即可，不需要 $n-2$.

一些注意事项：由于模运算的优美性质（(#\`O′) 喂！），可以在做矩阵乘法的时候边乘边模；同时建议开 `long long`，不然中间结果可能会乘爆；以及模数是 $100007$！~~为什么不是 1e9+7 qaq~~

## AC代码

~~c 的结构体好丑…… 还我 cpp o(>m<)o~~

~~里面挖了一点坑，注意别 CE 噜~~

```c
/* 
 Author: qwq automaton
 Result: AC	Submission_id: 6362077
 Created at: Thu Sep 19 2024 17:24:36 GMT+0800 (China Standard Time)
 Problem_id: 10	Time: 1	Memory: 1648
*/

#include <stdio.h>
#define int long long // 懒得换了，define 一下w
#define MOD 100007 // 模数
// 手写一个矩阵的复制
#define cpy(m1, m2) m1.a = m2.a; m1.b = m2.b; m1.c = m2.c; m1.d = m2.d
struct mat22
{
    int a, b;
    int c, d;
};
// 乘法，相当于 a *= b
void mul(struct mat22 *a, struct mat22 *b)
{
    struct mat22 res;
    res.a = (a->a * b->a + a->b * b->c) % MOD;
    res.b = (a->a * b->b + a->b * b->d) % MOD;
    res.c = (a->c * b->a + a->d * b->c) % MOD;
    res.d = (a->c * b->b + a->d * b->d) % MOD;
    cpy((*a), res);
}
// 快速幂，但是直接返回所求的值
int qpow(struct mat22 a, int n)
{
    struct mat22 res;
    res.a = res.b = 1; // 递推结果矩阵
    res.c = res.d = 0;
    while (n)
    {
        if (n % 2 == 1)
            mul(&res, &a);
        mul(&a, &a);
        n /= 2;
    }
    return res.a;
}
struct mat22 it;
int main()
{
    int n;
    while (scanf("%lld", &n) != EOF)
    {
        it.a = 1;
        it.b = 1;
        it.c = 1;
        it.d = 0;
        printf("%lld\n", qpow(it, n));
    }
    return 0;
}

```

