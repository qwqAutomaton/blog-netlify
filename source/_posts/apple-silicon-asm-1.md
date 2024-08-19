---
title: 苹果 M2 汇编 1
date: 2024-07-20 11:04:51
tags:
  - 汇编
  - 学习笔记
description: ARM 汇编好玩—— >w<
---

# 苹果 M2 芯片汇编学习笔记 (1)

找了好久终于找到 m2 的教程了（泪）于是记录笔记！

## 环境说明

芯片：Apple M2

操作系统：Sonoma 14.5

内核版本：Darwin 23.5.0

XNU 版本：10063.121

架构：arm64 (AArch 64)

<details>
<summary>查看方式</summary>

```bash
uname [-amnoprsv]
```

各选项效果：
|选项|效果|
|----|----|
|`-a`|全部信息|
|`-m`|硬件架构|
|`-n`|网络主机名|
|`-o`|内核名称，效果等于 `-s`|
|`-p`|处理器类型|
|`-r`|内核版本|
|`-s`|内核名称|
|`-v`|操作系统版本|
</details>

汇编器：as

链接器：ld

编译器：clang 15.0 / gcc 14.1 (homebrew gcc)

## 硬件相关知识

### 寄存器相关

#### 通用寄存器

CPU 采用 arm 架构，因此有 31 个通用寄存器。它们都是 64 位的寄存器，称为 `r0` 到 `r30`。

其中，`ri` 寄存器的低 32 位部分通过 `wi` 读写；64 位部分则通过 `xi` 读写。如：寄存器 `r0` 的低 32 位为 `w0`，64 位为 `x0`。

#### 特殊寄存器

特殊寄存器有：零寄存器，`sp` 寄存器，`pc` 寄存器。

**零寄存器**：`xzr` 和 `wzr`。

写入无效，读取时永远为 0。不过零寄存器不一定是真实的物理寄存器，可以通过一些小的逻辑判断模拟。

这样可以简化操作逻辑，加快运行速度。

**`sp` 寄存器**：储存栈顶位置（Stack Position）。

**`pc` 寄存器**：储存下一条指令的位置（Program Counter）。

### 数据储存相关

#### 端序（Endianness）

就是说数的低位和内存的低位是否一致的问题。把一个值为 `0b00100101` 的 `char`（一字节）存入内存有两种方法：

```plaintext
     Memory
 1 2 3 4 5 6 7 8
+-+-+-+-+-+-+-+-+
|0|0|1|0|0|1|0|1|   --> 大端序（Big endian）
+-+-+-+-+-+-+-+-+
|1|0|1|0|0|1|0|0|   --> 小端序（Little endian）
+-+-+-+-+-+-+-+-+
```

M2 使用小端序。

小端序的好处：

- 进行大转小的类型转换（如 `long long` 转 `int`）时不需要移动内存，直接丢弃高位
- CPU在做数值运算时，‌依次从低到高取数运算，更高效 [存疑]

#### 整数

以二进制补码形式存储。

原码：取最高位为符号位。先取整数的绝对值的二进制表示，如果是正数则将符号为置 0，否则置 1。

反码：正数的反码就是原码；负数的反码是其绝对值的原码取反。

补码：整数的补码就是原码；负数的补码是反码 +1。

使用补码可以将减法转化为无符号加法。

#### 浮点数

根据 [IEEE754](https://standards.ieee.org/ieee/754/6210/) 标准，浮点数储存采用符号位（S）、指数位（E）、分数位（N）的结构。

以 32 位浮点 `float` 为例，它在内存中的储存方式是：

```plaintext
               Memory
 1 2 .... 9 10 ...  ... ...  ... 32
+-+--------+-----------------------+
|S| E (8b) |       N   (23b)       |
+-+--------+-----------------------+
```

表示的值为

$$
\underbrace{(-1)^S}_{\text{符号}}\times\underbrace{(1+N\times 2^{-23})}_{\text{分数}}\times \underbrace{2^{E-127}}_{\text{指数}}
$$

如果把这个浮点数对应的内存看作一个 32 位二进制数，并从高到低记作 $(b_{31}b_{31}\dots b_2b_0)_2$，那么这个浮点数就是（下标 2 表示二进制）

$$
{\color{maroon}(-1)^{b_{31}}}\times {\color{darkblue}(1.b_{22}b_{21}\dots b_{0})_2}\times {\color{darkgreen}2^{(b_{30}b_{29}\dots b_{23})_2-127}}
$$

注意浮点数的比较应当考虑误差（$\varepsilon$ 是一个小正数，一般可取 $10^{-5}$ 等），如：

$\sout{f_1=f_2}\qquad -\varepsilon\le f_1-f_2\le\varepsilon$

#### 溢出

运算时如果在最高位产生进位，最高位将被丢弃。

### 操作系统相关

MacOS 建立在 Darwin 的基础之上，以 Aqua 为 GUI；而 Darwin 的内核是 XNU。

XNU 是混合型内核，有三个重要的部分：IO kit，Mach 和 BSD。

MacOS 的可执行文件一般都是 Mach-O 格式的文件。可以用 `file` 命令检查：

```bash
file [filename]
```

Mach-O 文件分成三个部分：头（header）、装载指令（load command）和数据（data）组成。其中，数据部分包含了很多个段（segment），每个段又包含了很多个节（section）。比如，`__TEXT` 段包含了字符串、汇编代码等，`__DATA` 段包含了初始化的全局变量（非常量）。每个段的大小应当是一页（4096 B）的倍数，这样可以方便加载程序。

### 访问控制相关

ARM 下，特权级别也被称为异常级别（exception level）。AArch64 下有四个 EL：

- EL0：普通应用

- EL1；操作系统内核和相关的函数（syscall 调用的就是这些）

- EL2：虚拟机监视器（Hypervisor）

- EL3：安全监视器（Secure monitor）

低 EL 的程序不能调用高 EL 的函数。

但是如果有的时候真的要调用系统内核的函数呢？通过发起异常进行提权，这也是 EL 得名的原因。

## 参考资料与注释

- [在 Apple Silicon Mac 上入门汇编语言](https://evian-zhang.github.io/learn-assembly-on-Apple-Silicon-Mac/index.html)
- [数字编码 - Hello Algo](https://www.hello-algo.com/chapter_data_structure/number_encoding/)
- [RISC-V RV32I中零寄存器有什么用](https://www.zhihu.com/question/308314026/answer/573831395)
