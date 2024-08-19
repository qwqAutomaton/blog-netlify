---
title: C++ 读入竞速
date: 2024-08-07 15:57:36
tags:
  - 摸鱼
  - C++
description: 摸鱼，爽！
---

# C++ 读入竞速

闲来无事 ~~其实是在摸鱼~~ 测测各种输入输出的方法。

## 前期准备

首先准备读入数据。

$10^7$ 个随机整数（`unsigned long long` 范围，即 $[0,2^{64})$），随机换行，文件名 `1e7_uint_64.in`：

```cpp
#include <iostream>
#include <random>
std::mt19937 rng;
int main()
{
    const int N = 100000000;
    std::ios::sync_with_stdio(false);
    for (int i = 1; i <= N; i++)
    {
        // 随机换，平均 13 个数一行
        if (rng() & 0xff < 20)
            std::cout << '\n';
        std::cout << rng() << ' ';
    }
    std::cout << std::flush; // 刷新一下输出缓冲
    return 0;
}
```

输出文件超过 $102\texttt{MB}$. 文件部分截图：

![1e7 ull file](./uint_part.png)

~~换行好…… 涩 /w\\~~

以及测试用的框架：

```cpp
#include <chrono>
typedef unsigned long long ull;
typedef ull (*func)();
double test(func read)
{
    // cin with sync
    rewind(stdin);                    // 把 C 风格读入指针移到文件头
    std::cin.seekg(0, std::ios::beg); // cin 移文件头
    auto beg = std::chrono::high_resolution_clock::now();
    read();
    auto end = std::chrono::high_resolution_clock::now();
    return (end - beg).count() / 1000000.0; // 转化成 ms
}
```

以及统一的编译命令：

```bash
g++-14 tester.cpp -o tester
./tester < 1e7_uint_64.in
```

每个方法开 `O2` 和不开 `O2`都测试 $10$ 次，取平均和最值。

开始吧！

## 传统读入

传统读入应该就是 `std::cin` 和 `scanf` 了。其中 `std::cin` 又可以有与 `stdio` 同步、解除同步、解除同步并且与 `std::cout` 解绑三种。

```cpp
ull cin_sync()
{
    ull x = 0, y;
    while (std::cin >> y)
        x ^= y;
    return x;
}
ull scanf()
{
    ull x = 0, y;
    while (scanf("%llu", &y) != EOF)
        x ^= y;
    return x;
}
ull cin_without_sync()
{
    std::ios::sync_with_stdio(false);
    ull x = 0, y;
    while (std::cin >> y)
        x ^= y;
    return x;
}
ull cin_without_sync_and_tie()
{
    std::ios::sync_with_stdio(false);
    std::cin.tie(nullptr);
    ull x = 0, y;
    while (std::cin >> y)
        x ^= y;
    return x;
}
```

测试结果：

|      |同步 `cin`|不同步 `cin`|不同步、解绑 `cin`|`scanf`|
|:----:|:-------:|:---------:|:--------------:|:-----:|
|平均用时|10808.4ms|454.994ms|421.706ms|874.959ms|
|最大用时|10926.4ms|496.551ms|464.709ms|898.52ms|
|最小用时|10680.1ms|434.524ms|403.825ms|868.402ms|
|`O2` 后平均|10441.5ms|424.523ms|398.268ms|915.556ms|
|`O2` 优化幅度|3.39%|6.70%|5.56%|-4.64%|

会发现整体的速度关系是 $\texttt{cin}(\text{unsync + untied})>\texttt{cin}(\text{unsync})>\texttt{scanf}>\texttt{cin}(\text{sync})$，O2 对 `cin` 系影响很小，甚至 `scanf` 负优化（可能是机子有波动 qaq）。

## 传统快读

这个就是用 `getchar()` （以及 `std::cin.get()`）然后手动组装成整数了。

```cpp
// getchar
ull readint()
{
    ull x = 0;
    char chr;
    for (chr = getchar(); !isdigit(chr) && chr != EOF; chr = getchar())
        ;
    if (chr == EOF)
        return -1;
    for (; isdigit(chr); chr = getchar())
        x = x * 10 + chr - '0';
    return x;
}
// getchar 卡常版
ull readint3()
{
    ull x = 0;
    static char chr;
    for (chr = getchar(); (chr < 48 || chr > 57) && (~chr); chr = getchar())
        ;
    if (chr < 0)
        return -1;
    for (; 48 <= chr && chr <= 57; chr = getchar())
        x = (x << 3) + (x << 1) + (chr & 15);
    return x;
}
// cin.get 版本只用 #define getchar std::cin.get 然后在 main 里加两句关闭同步即可
ull gcread()
{
    ull x = 0, y;
    while ((y = readint()) != ull(-1))
        x ^= y;
    return x;
}
```

同样读入这个文件，测试结果：

|      |`getchar()`|不同步、解绑 `cin.get()`|卡常 `getchar()`|卡常 `cin.get()`|
|:----:|:-------:|:---------:|:--------------:|:-----:|
|平均用时|2196.17ms|879.046ms|1894.39ms|514.504ms|
|最大用时|2230.22ms|962.985ms|2019.88ms|538.749ms|
|最小用时|2183.03ms|850.711ms|1826.91ms|509.681ms|
|`O2` 后平均|1908.83ms|493.546ms|1888.23ms|458.145ms|
|`O2` 优化幅度|13.1%|43.9%|0.33%|11.0%|

可以发现 `std::cin.get` 甚至比 `getchar` 快了一大截！翻了一下 `basic_istream` 的代码好像是因为 `cin.get` 是从缓冲区读的？不知道是不是编译器 / 系统的原因…… （homebrew gcc v14.1.0_2）

## 黑魔法

然后就是一些…… 黑魔法了。首先是 `fread` 快读：

```cpp
char fread_gc()
{
    constexpr size_t BUF_SIZ = 48 * 1024;
    static char buf[BUF_SIZ], *ptr = buf, *pEnd = buf + BUF_SIZ;
    if (ptr == pEnd)
        pEnd = buf + fread(ptr = buf, sizeof(char), BUF_SIZ, stdin);
    if (ptr == pEnd)
        return -1;
    return *ptr++;
}
ull readint()
{
#define gtchr fread_gc
    ull x = 0;
    static char chr;
    for (chr = gtchr(); (chr < 48 || chr > 57) && (~chr); chr = gtchr())
        ;
    if (chr < 0)
        return -1;
    for (; 48 <= chr && chr <= 57; chr = gtchr())
        x = (x << 3) + (x << 1) + (chr & 15);
    return x;
#undef gtchr
}
```

跑的飞快…… 用不同的缓冲区大小实验一下，测试结果：

|      |$8\texttt{KB}$|$48\texttt{KB}$|$128\texttt{KB}$|$1\texttt{MB}$|
|:----:|:-------:|:---------:|:--------------:|:-----:|
|平均用时|302.829ms|293.491ms|293.994ms|293.995ms|
|最大用时|313.579ms|305.686ms|318.804ms|315.38ms|
|最小用时|297.838ms|290.473ms|290.905ms|290.95ms|
|`O2` 后平均|278.672ms|261.635ms|252.778ms|253.628ms|
|`O2` 优化幅度|7.98%|10.9%|14.0%|13.7%|

事实证明并不能通过缩小缓冲区大小挤到 L1 Cache 里面（Apple M2 Silicon L1 $64\texttt{KB}$）得到加速…… 开小了还会因为 I/O 次数过多变慢（虽然还是比前面的快就是了）。不过缓冲区太大了卡不进 Cache 就会稍微慢一点（慢几十 ms，影响不是很大）。

考虑读入为什么会慢。我们发现读取文件是{% post_link apple-silicon-asm-1#访问控制相关 需要提权 %}的，因此会变慢！

于是努力做到减少系统调用的数量。

考虑一次 `fread` 把整个文件读入。

```cpp
constexpr size_t BUF_SIZ = 108195952; // 文件大小 + 1，开大一点
static char buf[BUF_SIZ], *ptr = buf, *pEnd = buf + fread(buf, sizeof(char), BUF_SIZ, stdin);
ull readint()
{
    ull x = 0;
    for (; (*ptr < 48 || *ptr > 57) && (ptr != pEnd); ptr++)
        ;
    if (ptr != pEnd) // 手动分支预测，把冷代码扔到后面取
    {
        for (; 48 <= *ptr && *ptr <= 57; ptr++)
            x = (x << 3) + (x << 1) + (*ptr & 15);
        return x;
    }
    else
        return -1;
}
```

测试结果：不开 `O2` 平均 276.205ms，开了之后 102.266ms 快到飞起， `O2` 优化效果达到了~~很抽象的~~ 62.97%！

一个黑魔法：`mmap` 系统调用。

```c
// sys/mann.h
void *mmap(void *addr, size_t len, int prot, int flags, int fd, off_t offset);
```

它支持将一个文件直接加载到内存里（映射到某一块空间）。具体来说是把文件描述符 `fd` 描述的文件（比如 `stdin`）从 `offset` 开始、长度为 `len` 的内容，加载到从 `addr` 开始的连续的内存页中。如果 `len` 或 `offset` 不是页大小的整数倍，则强制拓展到整数位，多出的位置置 $0$. 其中 `prot` 和 `flags` 描述了这块内存的性质。对于 `prot` 而言：

- `PROT_NONE`: 不可访问；
- `PROT_READ`: 可读取；
- `PROT_WRITE`: 可写入
- `PROT_EXEC`: 可执行

当然它们可以通过按位或 `|` 互相组合。`flags` 的参数更多。但是多是用来限制其他进程访问的。直接用 `MAP_SHARED` 表示允许其他访问即可。

用 `mmap` 写的读字符异常简洁：

```cpp
char mmap_gc()
{
    static char *buf = (char *)(mmap(NULL, 108195951, PROT_READ, MAP_SHARED, STDIN_FILENO, 0));
    return *buf++;
}
```

不过不知道为什么就是跑不起来…… QAQ

## 结论

速度大小比较。黑色为传统输入，红色为手写快读，绿的是黑魔法。

$$
\begin{aligned}
{\color{green}\texttt{fread}\text{ the whole file}}&>{\color{green}\texttt{fread}\text{, buffer fits in the cache}}&&\text{approx. }100\sim 300\texttt{ms}\\
>\texttt{cin}(\text{unsync + untied})&>\texttt{cin}(\text{unsync})>{\color{red}\texttt{cin.get}}>\texttt{scanf}&&\text{approx. }500\sim 800\texttt{ms}\\
>{\color{red}\texttt{getchar}}&>\texttt{cin}(\text{sync})&&>2000\texttt{ms}\\
&\ggg {\color{green}\texttt{mmap}}&&\text{跑不起来.jpg}
\end{aligned}
$$
