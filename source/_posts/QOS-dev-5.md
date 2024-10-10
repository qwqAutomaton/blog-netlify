---
title: QOS 开发 5
date: 2024-08-23 09:43:26
tags:
  - 操作系统
  - 汇编
description: 补充一下之前进入保护模式的一些细节 qwq 以及怎么查询机子的内存容量
---

# QOS 开发日记 5: 流水线和内存

(upd @ 20240911 补完后面的内存查询w)

## 流水线

进入保护模式之前需要先 `jmp` 一次还记得嘛 qwq

```x86asm
jmp dword SEL_CODE:pe_start
```

这里的 `dword` 应该比较好理解，意思是后面的这个地址的大小是双字（$4\texttt B = 32\texttt{bit}$）。后面地址这里也应该比较清楚：由于前面已经开启了保护模式（`CR0` 的 `PE` 设置成 1 了），所以需要用选择子找到代码段的位置（`SEL_CODE`），然后再跳到对应的 label. 可是为什么要这么做呢？

考虑底层的一些东西。众所周知，CPU 为了加快运行效率，采用了流水线执行：

![3 级流水线的例子](pipeline.png)

那么这个就是流水线了。

然而有些指令可能执行时间很长，比如

```x86asm
mov eax, [0x1111] ; 从内存 0x1111 取值
add ecx, ebx ; 加法
```

取内存的指令执行时间很长，可能要好几个周期；然而它和后面做加法的指令没有逻辑 / 数据上的冲突。因此我们可以把加法指令提出来先做，也就是**乱序执行**。

虽然 x86 最初支持的指令集是 CISC（复杂指令集），但是发展到现在，它的内核已经变成 RISC（精简指令集）了，把大的操作分成了很多小的操作。而那些小的操作基本上逻辑关联不大，因此可以利用乱序执行优化效率。

不过既然要做这种乱序执行，肯定要先计算出到底是什么个「乱」法（这个活是编译器 / 汇编器在干的）；然后就需要把这些顺序加入到流水线里进行真正的执行了。

还有一个就是分支预测。也就是 CPU 会预测各种 `jmp` 指令跳转的概率，并把（看起来）概率更大的那个预先加载到流水线里面。可以类比 C++ 里面的

```cpp
if (__builtin_expect(!!(/* Condition */), 1))
    { /* Expected to be more possible */ }
else { /* Expected to be less possible */ }
```

那么缺点就是如果遇到了和预测不匹配的地方就要把之前预测的流水线清空，开销比较大。不过其实也不会出现什么致命性的问题就是了。

然而在实模式下的指令和保护模式下的指令并不相同，直接执行流水线里面的值很可能会有错误。因此就要通过一个远跳（far jump）清空流水线，也就是代码里的

```x86asm
jmp     dword       SEL_CODE : pe_start
;       ^~~~~       ^~~~~~~~   ^~~~~~~~
;   表示地址是 32 位  选择子（段）    标记
; jmp <段>:<地址> 就是远跳啦
```

那么清空了实模式下的流水线，接下来就正式进入保护模式了（彻底抹除了实模式的痕迹 qaq）。

## 内存查询

那么如何查询具体的内存大小？实际上有 3 种方法。不过这些都需要在进入保护模式之前使用 BIOS 中断来实现（因为进入之后就不能用中断了 qaq）。

我们注意到可以使用 `detect_memory` 函数进行内存检测：

```c
int detect_memory();
```

~~对就是这么简单~~ （好像是在 linux 的 `setup.bin` 里面？）

然而这个函数底层是通过调用 `0x15` 号中断实现的。那么子功能号当然就是根据 `eax` 寄存器决定了。大概是：

|`eax`|功能|
|:---:|:-:|
|`0xe820`|遍历主机上所有内存|
|`0xe801`|分别检测低于 $15\texttt{MB}$ 和 $16\texttt{M}\sim 4\texttt{GB}$ 内存。最大支持 $4\texttt{GB}$|
|`0x8800`(实际上是 `ah=0x88`)|最多检测 $64\texttt{MB}$ 内存，实际内存超过此容量也按照 $64\texttt{MB}$ 返回|

这三个功能中最强大的是 `0xe820`，因为它返回的信息是最丰富的。因此就用这种方式查内存啦~

实际上 `0xe820` 子功能返回的是一个“结构体”，也就是**地址范围描述符结构体**（Addreses range descriptor structure, ARDS）。格式：

![ARDS 格式](ARDS.jpg)

可以理解为这个 `struct`：

```c
typedef struct
{   // sizeof(unsigned) = 4
    unsigned int BaseAddrLow, BaseAddrHigh;
    unsigned int LengthLow, LengthHigh;
    unsigned int Type;
    // 或者直接 unsigned long long BaseAddr; 也行...
} ARDS;
```

其它四个都比较好理解（基址、长度），下面是这个 `Type` 的格式：

![ARDS type 格式](ARDS-type.jpg)

那么什么内存会被判定为保留？主要是：

- 系统 ROM.
- 设备内存映射（比如实模式下的 `0xb80000 - 0xbffff` 映射到文本模式的 VGA）。
- 由于一些奇怪的原因这块内存不可用。

不过由于现在做的是 $32$ 位的系统，我们只需要考虑 `BaseAddrLow` 之类的低 $4\texttt B$ 的部分。

嗯不过用 `0xe820` 这个子功能还有别的参数：

![0xe820 parameters part 1](e820-param.jpg)

![0xe820 parameters part 2](e820-param-2.jpg)

有一个很有意思的点：`ecx` 指的是缓冲区的大小（也就是预留的空间），然后 `es:di` 是缓冲区的地址。实际上在使用的时候，调用者应当在 `ecx` 写入希望获得的大小，而结果则是获取的实际大小。

很显然实际查询的时候需要一段一段已知循环地去查，`ebx` 等于 $0$ 时停止。当然也要考虑 `CF` 为 $1$，也就是出错的情况：

```c
ebx = 0x0000; // 清空寄存器
edx = 0x534D4150;
// es 已经在 mbr 里赋 0，这里就不需要再管了
di = &adrs;   // 设置地址
do
{
    eax = 0xe820;    // 设置子功能
    ecx = 20;        // 期望返回的大小
    // 上面这两个都是会变的，因此每次循环都要设置一次
    interrupt(0x15); // 调用中断
    if (CF == 1)
    {
        // ... 错误处理
    }
    di += ecx;       // 增大写入地址
} while (ebx != 0)
```

写成汇编就是这一坨 qaq：

```x86asm
    ; 检查内存
    ; 在 ards 处开辟一块存 ARDS 的空间
    ; ards_cnt 处存放 ARDS 数量（2B）
    xor ebx, ebx
    mov edx, 0x534d4150
    mov di, ards
.check_mem:
    mov eax, 0x0000_e820
    mov ecx, 20
    int 0x15
    jc .check_mem_fail
    inc word [ards_cnt]
    add di, cx
    cmp ebx, 0
    jnz .check_mem
    jmp .check_mem_finished
.check_mem_fail:
    ; 总之是输出失败之类的 w
.check_mem_finished:
    ; 输出查询结束
```

由于它是检验内存的，我们将其放在 Loader 加载 GDT 的部分之前。同时在开头 GDT 后面增加一块用于储存 ARDS 的区域。用 C 理解大概是：

```c
struct ARDS ards[25]; // 暂时开 25 个
unsigned short ards_cnt; // 再额外开 2B 用于计数
// ...
// check memory
```

为了方便在内存中找到这块 `ards` 内存，可以在这块内存之前存一个魔法数字（比如 `dq 0x1145_1419_1981_0AAA`）。在 bochs 中，查询内存使用命令 `x` 或 `xp`.

那么经过测试，有 $6$ 块 ARDS，其中每块的内容为：

|ARDS 编号|基址 `BaseAddrLow`|长度 `LangthLow`|类型 `Type`|
|:------:|:----------------:|:-------------:|:--------:|
|$0$|`0x00000000`|`0x0009f000`|`0x01` 可用内存|
|$1$|`0x0009f000`|`0x00001000`|`0x02` 保留内存|
|$0$|`0x000e8000`|`0x00018000`|`0x02` 保留内存|
|$0$|`0x00100000`|`0x01ef0000`|`0x01` 可用内存|
|$0$|`0x01ff0000`|`0x00010000`|`0x03` 保留内存|
|$0$|`0xfffc0000`|`0x00040000`|`0x02` 保留内存|

这和第一篇当中实模式下的内存分配是相同的。
