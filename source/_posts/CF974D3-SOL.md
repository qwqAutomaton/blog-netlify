---
title: Codeforces Round 974 (Div. 3) 题解
date: 2024-09-22 14:57:58
tags:
  - 题解
description: 复健失败…… 码力为 0 qaq
---

# CF974 Div3 题解

决定复健一下，结果发现码力爆炸了……

[传送门](https://codeforces.com/contest/2014)

## A. Robin Helps

> 题意简述：有 $n$ 个人，第 $i$ 个人有 $a_i$ 块钱。给定一个阈值 $k$，如果 $a_i\ge k$，那么 Robin 会把这个人的钱全部抢走；如果 $a_i=0$ 且 Robin 手里有钱，就会给他一块钱。Robin 开始没有钱。问 Robin 要给几个人钱。
>
> $t$ 组数据，满足 $t\le 10^4,n\le 50$.

直接 $O(n)$ 模拟。

```cpp
#include <iostream>
void _()
{
    int n, k;
    std::cin >> n >> k;
    int tot = 0, cnt = 0;
    for (int i = 1, a; i <= n; i++)
    {
        std::cin >> a;
        if (a >= k)
            tot += a;
        else if (a == 0 && tot)
            tot--, cnt++;
    }
    std::cout << cnt << std::endl;
}
int main()
{
    std::ios::sync_with_stdio(false);
    int t;
    std::cin >> t;
    while (t--)
        _();
    return 0;
}
```

## B. Robin Hood and the Major Oak

> 题意简述：给定 $n,k$，判断和式 $\displaystyle{\sum_{i=n-k+1}^ni^i}$ 的奇偶性。$t$ 组数据，$t\le 10^4,n\le 10^9$.

很显然 $i^i$ 为偶数当且仅当 $i$ 为偶数。因此等价于判断 $\sum_{i=n-k+1}^ni=k(2n-k+1)/2=nk-\dfrac {k(k+1)}2$ 的奇偶性。不过当时思路有点奇怪……

```cpp
#include <iostream>
const char* rsp[2] = {"YES\n", "NO\n"};
void _()
{
    int n, k;
    std::cin >> n >> k;
    int tot = k / 2;
    if (k & 1 && n & 1) tot++;
    std::cout << rsp[tot & 1];
}
int main()
{
    std::ios::sync_with_stdio(false);
    int t;
    std::cin >> t;
    while (t--)
        _();
    return 0;
}
```

## C. Robin Hood in Town

> 题意简述：给定一个序列 $a_n$，求使得下列条件成立的最小非负整数 $x$：将序列中的最大值加上 $x$ 后，严格小于均值的一半的数的个数大于总数的一半，即 $\left|\{i\mid a_i<\dfrac {\bar a}2\}\right| > \dfrac n2$.

题目要求等价于中位数（如果 $n$ 为奇数）或「正中的下标向上取整」（如果 $n$ 为偶数）小于操作后的均值。因此先升序排序。假设排序后这一项下标为 $m=\left\lfloor\dfrac n2\right\rfloor+1$，那么题意就等价于找到 $x$ 使得

$$
a_m<\dfrac 12\bar a+x=\dfrac {\sum a}{2n}+x
$$

因此可知最小的 $x=2na_m-\sum a+1$. 同时如果序列已经满足这个条件，那么 $x=0$；如果求出来的这个 $m\ge n$，那么这样的 $x$ 不存在。

```cpp
#include <iostream>
#include <algorithm>
int a[200010];
void _()
{
    int n;
    long long sum = 0;
    std::cin >> n;
    for (int i = 1; i <= n; i++)
    {
        std::cin >> a[i];
        sum += a[i];
    }
    std::sort(a + 1, a + 1 + n);
    int mid = (n + 2) / 2;
    if (mid >= n)
        std::cout << "-1\n";
    else
    {
        long long d = a[mid];
        if (2 * d * n < sum)
            std::cout << "0\n";
        else
            std::cout << 2 * d * n - sum + 1 << "\n";
    }
}
int main()
{
    std::ios::sync_with_stdio(false);
    int t;
    std::cin >> t;
    while (t--)
        _();
    return 0;
}
```

## D. Robert Hood and Mrs Hood

> 题意简述：给定 $k$ 个区间和整数 $d$. 求所有长度为 $d$ 且包含在 $[1,n]$ 中的区间中，和给定区间相交的个数最多和最少的区间的位置。多组数据，数据组数 $t\le 10^4,d,k\le n\le 10^5$. 保证所有数据中 $n$ 的和不超过 $2\times 10^5$.

考虑滑动窗口（？）。依次考虑区间 $[1,n-d+1]$ 中的每一个点作为区间起点，那么在移动过程中，如果右端点有新的区间，则当前区间数 $+1$；如果左端点有区间结束，那么当前区间数 $-1$. 可以开两个数组记录一下。

```cpp
#include <iostream>
#include <cstring>
const int MAXN = 100010;
int l[MAXN], r[MAXN];
void _()
{
    int n, d, k;
    std::cin >> n >> d >> k;
    memset(l, 0, sizeof(l));
    memset(r, 0, sizeof(r));
    for (int i = 1; i <= k; i++)
    {
        int x, y;
        std::cin >> x >> y;
        l[x]++, r[y]++;
    }
    int i = 1, j = 1;
    int tot = 0;
    for (; j <= d; j++)
        tot += l[j];
    int mx = tot, mn = tot;
    int p = 1, q = 1;
    for (; j <= n; j++, i++)
    {
        tot += l[j];
        tot -= r[i];
        if (tot > mx)
        {
            mx = tot;
            p = i + 1;
        }
        if (tot < mn)
        {
            mn = tot;
            q = i + 1;
        }
    }
    std::cout << p << ' ' << q << '\n';
}
int main()
{
    std::ios::sync_with_stdio(false);
    int t;
    std::cin >> t;
    while (t--)
        _();
    return 0;
}
```

## E. Rendez-vous de Marian et Robin

> 题意简述：给定一张无向有边权的图 $G(V,E)$，图中某些点上有马，边需要一定时间通过。两人从 $1,n$ 两点分别出发，在有马的点处可以瞬间骑马。骑马时通过边的时间是走路的一半。两人最终需要在某个顶点处相遇。问相遇需要的最小时间。$t\le 10^4,|V|\le 2\times 10^5,|E|\le 2\times 10^5$. 保证所有数据中点数之和小于 $2\times 10^5$.

考虑将图中的一个点拆成两个 $u,u'$，其中 $u$ 表示人走路在的点，$u'$ 表示骑马走的路。那么开始时预先连 $|V|$ 条 $u'\to u$ 的有向边，长度为 $0$，表示从骑马到走路不用时间。然后对于所有有马的点 $v$，连一条 $v\to v'$、长度为 $0$ 的有向边，表示上马不耗时。然后对于每一条边 $(u,v)$，在 $(u',v')$ 之间连一条长度为原边一般的新无向边。然后从 $1$ 和 $n$ 分别跑一遍单源最短路即可。可以把 $i$ 号点拆成 $2i,2i+1$ 两个点。

哦然后还有输入输出量比较大，以及用了挺多次的 STL，手动开个 O3. 还有就是每次都 `memset` 会 T...

~~赛时脑子抽了调 dij 调了一个多小时…… 甚至最后还 fst 了~~

```cpp
#pragma GCC optimize(3)
#include <iostream>
#include <cstring>
#include <queue>
using ll = long long;
const int MAXV = 400000 + 10, MAXE = 1200010;
int to[MAXE], nxt[MAXE];
ll d[MAXE];
int head[MAXV], cnt = 0;
ll dis[2][MAXV];
bool vis[2][MAXV];
void ae(int u, int v, ll w)
{
    nxt[++cnt] = head[u];
    head[u] = cnt;
    to[cnt] = v;
    d[cnt] = w;
}
void dij(int s, int _)
{
#define D dis[_]
#define V vis[_]
    struct dat
    {
        ll dis;
        int to;
        bool operator<(const dat &d) const { return dis > d.dis; }
    };
    std::priority_queue<dat> q;
    D[s] = 0;
    q.push({D[s], s});
    while (!q.empty())
    {
        int u = q.top().to;
        q.pop();
        if (V[u])
            continue;
        V[u] = 1;
        for (int i = head[u]; ~i; i = nxt[i])
        {
            int v = to[i];
            ll w = d[i];
            if (D[v] > D[u] + w)
            {
                D[v] = D[u] + w;
                q.push({D[v], v});
            }
        }
    }
#undef V
#undef D
}
#define mxt(i) std::max(dis[0][i], dis[1][i])
#define INF 0x3f3f3f3f3f3f3f3fLL
void _()
{
    int n, m, h;
    // clear
    // memset(head, -1, sizeof(head));
    // memset(dis, 0x3f, sizeof(dis));
    // memset(vis, 0, sizeof(vis));
    std::cin >> n >> m >> h;
    for (int i = 1; i <= 2 * n + 1; i++)
    {
        head[i] = -1;
        dis[0][i] = dis[1][i] = INF;
        vis[0][i] = vis[1][i] = 0;
    }
    cnt = 0;
    // input
    for (int i = 1; i <= n; i++)
        ae(2 * i + 1, 2 * i, 0);
    for (int i = 1, x; i <= h; i++)
    {
        std::cin >> x;
        ae(2 * x, 2 * x + 1, 0);
    }
    for (int i = 1; i <= m; i++)
    {
        int p, q;
        ll r;
        std::cin >> p >> q >> r;
        ae(2 * p, 2 * q, r);
        ae(2 * q, 2 * p, r);
        ae(2 * p + 1, 2 * q + 1, r / 2);
        ae(2 * q + 1, 2 * p + 1, r / 2);
    }
    dij(2 * 1, 0);
    dij(2 * n, 1);
    int minp = -1;
    for (int i = 2; i <= n << 1; i += 2)
    {
        if (dis[0][i] == INF || dis[1][i] == INF)
            continue;
        if (minp == -1 || mxt(i) < mxt(minp))
            minp = i;
    }
    if (~minp)
        std::cout << mxt(minp) << "\n";
    else
        std::cout << "-1\n";
}

int main()
{
    std::ios::sync_with_stdio(false);
    std::cin.tie(0);
    int t;
    std::cin >> t;
    while (t--)
        _();
    return 0;
}
```

## F. Sheriff's Defence

> 题意简述：给定一棵 $n$ 个结点的无根树，$i$ 号结点上有 $a_i$ 个金币。对每个结点，可以选择加固或不加固。加固会使和它相邻的所有结点中的金币数量都减去 $c$，而不加固什么都不会发生。允许某些结点的金币是负数。求所有加固过的点中金币数量之和的最大值。

简单树形 DP. 设 $f[i][0/1]$ 表示以 $i$ 为根的子树，不加固 / 加固结点 $i$ 得到的金币之和的最大值。方程（用 $S(i)$ 表示 $i$ 的儿子）：

$$
\begin{aligned}
f[i][0] &= \sum_{j\in S(i)}\max\{f[j][0], f[j][1]\}\\
f[i][1] &= \sum_{j\in S(i)}\max\{f[j][0], f[j][1] - 2c\}
\end{aligned}
$$

这里 $f[i][1]$ 当中减去 $2c$ 的原因：$j$ 加固耗费 $i$ 结点的 $c$ 个金币（没算在 $f[j][0/1]$ 里）；$i$ 加固再耗费 $j$ 节点 $c$ 个金币。

```cpp
// O2 一时爽，一直 O2 一直爽
#pragma GCC optimize(2)
#include <iostream>
#include <vector>
#define int long long
void _();
signed main()
{
    std::ios::sync_with_stdio(false);
    signed t;
    std::cin >> t;
    while (t--)
        _();
    return 0;
}
const int MAXN = 200010;
std::vector<unsigned> to[MAXN];
int f[MAXN][2], a[MAXN];
unsigned n, c;
void dfs(unsigned rt, unsigned pre = 0)
{
    // std::cout << "dfs(" << rt << "). pre=" << pre << "\n";
    f[rt][0] = 0;
    f[rt][1] = a[rt];
    for (auto i : to[rt])
    {
        if (i == pre)
            continue;
        dfs(i, rt);
        f[rt][0] += std::max(f[i][1], f[i][0]);
        f[rt][1] += std::max(f[i][1] - c * 2, f[i][0]);
    }
    // std::cout << "f[" << rt << "][0/1]=(" << f[rt][0] << "," << f[rt][1] << ".\n";
}
void _()
{
    std::cin >> n >> c;
    for (unsigned i = 1; i <= n; i++)
    {
        std::cin >> a[i];
        std::vector<unsigned>().swap(to[i]);
    }
    for (unsigned i = 1, u, v; i < n; i++)
    {
        std::cin >> u >> v;
        to[u].push_back(v);
        to[v].push_back(u);
    }
    dfs(1);
    std::cout << std::max(f[1][0], f[1][1]) << std::endl;
}
```

## G. Milky Days

> 题意简述：有 $n$ 次送牛奶，第 $i$ 次在第 $d_i$ 天，送了 $a_i$ 品脱牛奶。有人每天喝 $m$ 品脱，并且每次喝都优先喝最新鲜的。牛奶送到 $k$ 天后会过期。如果能够喝 $m$ 品脱，那么这一天称为「牛奶满足日」；否则奶照喝，但是不满足。问总共有多少天是牛奶满足日？

~~赛时想了一个很诡异的思路…… 大概是把所有喝完的奶用并查集合并起来（类似第二分块那样？）但是没时间打……~~

考虑类似单调栈的想法。把所有送来的牛奶压到栈里，先把栈底过期的弹掉，然后从栈顶开始喝。可以在所有数据之后添加一个 $d_{n+1}=+\infty,a_{n+1}=0$ 的数据。处理过期的细节比较恶心。

```cpp
#include <iostream>
#include <queue>
void _();
int main()
{
    std::ios::sync_with_stdio(false);
    int t;
    std::cin >> t;
    while (t--)
        _();
    return 0;
}
const int MAXN = 100010;
int d[MAXN], a[MAXN];
struct milk
{
    int amount;
    int date;
} q[MAXN];
int fr = 0, tl = 0;
void _()
{
    int n, m, k;
    std::cin >> n >> m >> k;
    for (int i = 1; i <= n; i++)
        std::cin >> d[i] >> a[i];
    d[n + 1] = 1e9;
    a[n + 1] = 0;
    int cur = 1;
    int cnt = 0;
    int got = 0;
    fr = tl = 0;
    for (int i = 1; i <= n + 1; i++)
    {
        while (tl != fr && cur < d[i])
        {
            auto [am, dt] = q[--tl];
            if (dt + k - 1 < cur)
                continue; // expired
            if (dt > cur) // drink all
                cur = dt, got = 0;
            if (m - got > am)
                got += am; // not enough milk
            else // enough milk, drink it
            {
                int tmp = std::min(cur + (am - m + got) / m + 1,
                    std::min(dt + k, d[i])); // total milk day
                int tmp2 = am - (tmp - cur) * m + got; // remaining milk
                if (tmp2) q[tl++] = {tmp2, dt};
                cnt += tmp - cur; // accumulate days
                got = 0; // remaining milk has counted
                cur = tmp; // next date
            }
        }
        q[tl++] = {a[i], d[i]};
    }
    std::cout << cnt << std::endl;
}
```

## H. Robin Hood Archery

> 题意简述：给定序列 $a_n$，游戏中每轮伦理剧任取一个还没取过的数，Robin 先，Sheriff 后。所有数都取完后，取出的数之和大的那个胜利；若一样，则平局。两人每次都取最优决策。现有 $q$ 个询问，每次询问一个区间 $[l,r]$，表示限定在 $a_l\sim a_r$ 中取，问 Sherrif 是否可能不输（平或赢）。

考虑这样的事情：由于 Robin 先取，可以每次取剩下的当中最大的那个。假设 Robin 第 $i$ 次取 $R_i$，Sheriff 第 $i$ 次取 $S_i$，容易证明这么取的情况下 $R_i\ge S_i$，因此 Sheriff 不可能赢；并且平局需要这 $n$ 个不等式的等号同时取到。因此平局等价于区间内每个数出现的次数是偶数（这样就可以一个对一个取到等号）。

问题转化为：给定序列和询问，求区间内每个数出现的次数是否为偶数。

### 法 1：莫队

注意到对于一个区间 $[l,r]$，如果维护区间中**出现次数是奇数的数的个数**，可以在 $O(1)$ 内完成端点差 $1$ 的区间转移：

```cpp
int tot; // 当前区间奇数的数的总数
void move(int x)
{
    odd[a[x]] ^= 1;
    tot += odd[a[x]] ? 1 : -1; 
}
// [l, r] -> [l-1, r]
move(--l);
// [l, r] -> [l+1, r]
move(l++);
// [l, r] -> [l, r-1]
move(r--);
// [l, r] -> [l, r+1]
move(++r);
```

于是考虑离线做莫队。复杂度 $O((n+q)\sqrt n)$.

```cpp
#include <iostream>
#include <cstring>
#include <algorithm>
void _();
int main()
{
    std::ios::sync_with_stdio(false);
    int t;
    std::cin >> t;
    while (t--)
        _();
    return 0;
}
const int MAXN = 2e5 + 10;
struct query
{
    int l, r;
    int i, ans;
} q[MAXN];
bool odd[1000010];
int a[MAXN];
const char* OPT[2] = {"YES\n", "NO\n"};
int tot;
void move(int x)
{
    odd[a[x]] ^= 1;
    tot += odd[a[x]] ? 1 : -1;
}
void _()
{
    int n, m;
    std::cin >> n >> m;
    for (int i = 1; i <= n; i++)
        std::cin >> a[i];
    for (int i = 1; i <= m; i++)
    {
        std::cin >> q[i].l >> q[i].r;
        q[i].i = i;
    }
    std::sort(q + 1, q + 1 + m,
        [&](const query &p, const query &q) -> bool
              { return p.l / 450 == q.l / 450 
                        ? p.r < q.r 
                        : p.l < q.l; });
    tot = 0;
    memset(odd, 0, sizeof(odd));
    int l = 1, r = 0;
    while (r < q[1].r)
        move(++r);
    while (l < q[1].l)
        move(l++);
    q[1].ans = tot;
    for (int i = 2; i <= m; i++)
    {
        // move to q[i]
        while (r < q[i].r)
            move(++r);
        while (l > q[i].l)
            move(--l);
        while (l < q[i].l)
            move(l++);
        while (r > q[i].r)
            move(r--);
        q[i].ans = tot;
    }
    std::sort(q + 1, q + 1 + m,
        [&](const query &p, const query &q) -> bool
              { return p.i < q.i; });
    for (int i = 1; i <= m; i++)
        std::cout << OPT[!!q[i].ans];
}
```

### 法 2：异或哈希

考虑将一个区间看做两段前缀之差：$[l,r]=[1,r]\setminus [1,l-1]$. 那么 $[l,r]$ 中数字都成对出现等价于这两个前缀当中各个数出现的奇偶性相同。

而考虑到异或的神奇性质：$a\oplus a=0$，是否可以通过求出前缀的异或和，然后判断 $[l,r]$ 这一段的异或和是否为 0？

不一定。很容易找到一组数使得他们两两不同，但异或和为 $0$. 这可以理解为某种意义上的哈希冲突（将整个序列映射到它的异或和）。

既然是「哈希」冲突，那么我们类比真正哈希当中避免冲突的方法。一个最简单的方法就是：不求原数的异或和，而是先给每个数随机分配一个「代表值」，再求异或和。也就是找到一个映射 $\varphi:x\mapsto x'$，然后求 $\bigoplus_{i=l}^r\varphi(x_i)$. 为了让冲突尽量减少，我们需要让这些代表值在二进制下的 $1$ 更「均匀」一些，因此使用梅森旋转的随机数。

```cpp
#include <iostream>
#include <ctime>
#include <random>
void _();
int main()
{
    std::ios::sync_with_stdio(false);
    int t;
    std::cin >> t;
    while (t--)
        _();
    return 0;
}
const int MAXN = 2e5 + 10;
std::mt19937_64 rng(time(nullptr));
typedef unsigned long long ull;
ull code[1000010];
int a[MAXN];
ull xors[MAXN];
void _()
{
    int n, q;
    std::cin >> n >> q;
    for (int i = 1; i <= n; i++)
    {
        std::cin >> a[i];
        code[a[i]] = rng();
    }
    xors[0] = 0;
    for (int i = 1; i <= n; i++)
        xors[i] = xors[i - 1] ^ code[a[i]];
    for (int i = 1, l, r; i <= q; i++)
    {
        std::cin >> l >> r;
        if (xors[r] ^ xors[l - 1])
            std::cout << "NO\n";
        else
            std::cout << "YES\n";
    }
}
```

当然如果还担心容易被 hack 的话，可以考虑做双重哈希（每个数有两个 `code` 分别取异或和）。不过 editorial 里面只用了一个。
