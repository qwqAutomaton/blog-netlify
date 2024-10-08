---
title: QOS 开发 4
date: 2024-08-14 15:40:07
tags:
  - 操作系统
  - 汇编
description: 实地址太危险了一不小心就 jmp 错地方爆了 /ll
---
# QOS 开发日记 4: 保护模式

我们需要保护模式！

~~不是什么时候能用 C 啊我不想再 mov 来 mov 去了（大吵大闹~~

这篇写了好久…… 代码量也开始有点多了，扔 github 上存着：[QOS.git](https://github.com/qwqAutomaton/QOS)

## re: 寄存器

保护模式下的寄存器又有变化。。。

实模式下寄存器只用了 16 位，但是事实上 x86 的通用寄存器有 64 位。

和之前一样，寄存器也是分成通用寄存器（`ax`, `bx`, `cx`, `dx`, `di`, `si`, `bp`, `sp`）、段寄存器（`cs`, `ds`, `es`, `fs`, `gs`, `ss`）、变址寄存器和指针寄存器（`si`, `di`, `sp`, `bp`）以及两个特殊的寄存器 `flags` 和 `ip`。

### 通用寄存器

```plaintext
63               31       15   7   0
+----------------+--------+----+----+
|                |        | ah | al |
+----------------+--------+----+----+
|                |        |<-- ax ->|
|                |<---    eax   --->|
|<----------     rax     ---------->|
```

可以把寄存器名字理解为「前缀+名+后缀」的形式（这对除了段寄存器之外的其他寄存器也适用）：

|前后缀|含义|
|-----|---|
|`r**`|64 位|
|`e**`|32 位|
|`**`（直接用名称）|16 位|
|`*h`（名称第一个字符加 `h`）|16 位中的高 8 位|
|`*l`（名称第一个字符加 `l`）|16 位中的低 8 位|

然后用途的话也是和{% post_link QOS-dev-2#寄存器 之前的 %}一样。

### 段寄存器

直接扔一个表格 qwq

|寄存器|全称|用途|
|-----|---|----|
|`cs`|代码段 code segment|代码基础位置|
|`ds`|数据段 data segment|变量等数据所在位置|
|`es`, `fs`, `gs`|附加段 extra segment|附加段的位置|
|`ss`|栈段 stack segment|栈的位置，栈顶通过 `ss:sp` 访问|

实模式里面段寄存器存的是段开头的地址，但是在保护模式中储存的不是段开头地址了，而是一个叫做“选择子”（selector）的东西（这个后面说算了）。不过在写的时候寻址还是按照 `_s:_p` 的形式（段+地址）寻址。

### 特殊寄存器

`ip` 是指令指针，指向下一条指令。也就是说，当前执行完之后的下一条指令的位置是 `cs:ip`。

`flags` 是标志位寄存器，内容参见{% post_link QOS-dev-2#寄存器 之前的文章%}。

## 保护模式

~~终于到保护模式了……前置怎么这么多（恼）~~

那么之前写的 MBR 和内核加载器都是在实模式下运行的。但是实模式有两个很致命的缺点：一个是内存只能用 $1 \texttt{MB}$ 太小；还有一个就是它里面的内存使用都是通过绝对地址实现的，那这样的话不同的程序跑起来就很容易出现冲突出锅了（悲）。因此 x86 架构提供了保护模式这么一个概念。

为了解决内存太小的问题，寄存器肯定就要扩展了，扩展的位数取决于操作系统的位数（这里就选 32 位啦~ 这样好写一点）。当然寄存器的位数实际上取决于 CPU 的硬件（即硬件上堆了几个 latch），实模式下只是用了硬件寄存器的低 16 位而已。那么要使用 32 位就在寄存器名字前面加个 `e` 就行了，64 位加 `r`。

~~听说现在有个 AVX 是 256 位的寄存器？有钱人就是可怕.jpg~~

不过像上面讲的，段寄存器不能再前面加 `e` 或者加 `r` 来扩展，它们在保护模式下还是 16 位的。但是理论来说变成 32 位 $2^{32}\texttt{B}=4\texttt{GB}$ 的地址之后，段寄存器应该也变成 32 位才对啊？因为保护模式为了突出它和实模式的区别（？），维护了一个全局描述符表（global descriptor table, GDT）来管理内存，段寄存器需要在 GDT 上登记一下才能使用。

那么先说一下保护模式的功能：

- 提供了内存分段和分页机制，允许操作系统将内存划分为多个段和页面，并为每个段和页面设置访问权限，从而实现对内存的保护。这样可以防止用户程序越界访问内存、修改操作系统数据结构或者恶意篡改其他程序的数据（然后报 `segmentation fault` 或者 `bus error` 之类的 RE）。
- 引入了特权级别的概念，限制用户程序对关键资源的访问，确保系统的稳定性和安全性。
- 提供了对 IO 设备的保护机制，可以限制用户程序对设备的直接访问，确保设备的安全和可靠性。
- 更强大的异常和中断处理能力，操作系统可以利用这些机制来响应硬件事件、处理错误情况以及进行多任务调度。

总之就是内存保护、IO 保护、权限控制和其他东西 owo

### 段描述符

段描述符描述了一个段的属性。 ~~什么废话~~

它是一个 64 位的值，结构大概是这样的：

```plaintext
 63     56  55  54  53  52 51      48  47 46  45  44 43    40 39     32
+---------+---+---+---+---+----------+---+------+---+--------+---------+
|段基址高8位| G |D/B| L |AVL|段界限高4位| P |  DPL | S |  TYPE  |段基址中8位|  高 32 位
+---------+---+---+---+---+----------+---+------+---+--------+---------+

 31                                16 15                              0
+------------------------------------+---------------------------------+
|           段基址低 16 位             |           段界限低 16 位          |  低 32 位
+------------------------------------+---------------------------------+
```

- 段基址：就是实模式中的那个段开头的地址。因为是 32 位操作系统所以是 32 位的。
- G：等于 0 表示段界限的单位是 B；等于 1 则表示段界限的单位是 4K。
- 段界限：表示段扩展边界的最值，即最大扩展多少（代码段、数据段等）或最小扩展多少（栈），简单地说就是这个段的空间大小。它需要配合 G 决定单位。比如说段界限的值是 $15$，G = 0 则说明段的大小是 $15\texttt B$；G = 1 则说明段的大小是 $15\times 4\texttt{KB}=60\texttt{KB}$.
- AVL：保留字段，无特殊用途
- S：描述这个段是系统段（0）还是数据段（1）。
- L：描述这个段是 32 位代码段（0）还是 64 位代码段（1）。因为写的是 32 位的操作系统，所以 L = 0。
- D/B：对于代码段，描述这个段的指令中操作数和指令地址是 16 位（0）还是 32 位（1）。由于相同的原因，D/B = 1，因此表示指令指针时应当用 `EIP`。
- DPL：该描述符的特权级别 (Descriptor Privilege Level)，指本内存段的特权级，这两位能表示 0~3 共 4 种特权级，数字越小，特权级越大。CPU 由实模式进入保护模式后，特权级自动为 0。用户程序通常处于 3 级。
- P：表示段是否存在于内存中。这个主要是让 CPU 检查的。如果 P=0，则会抛出异常，然后转到异常处理程序。
- TYPE：表示内存段或门的类型。需要根据上面的 S 属性决定定义。

其中 TYPE 字段具体含义如图。
![段描述符 TYPE 字段](sd-type.png)
非系统段需要展开讲讲。

- `X`: 执行（eXcute）
- `R`: 读取（Read）
- `W`: 写入（Write）
- `A`: 访问（Access）。这个是由 CPU 设置的。CPU 每次访问过这一段后都会将它设置为 1。所以新建一个描述符的时候这一位要变成 0。
- `C`: 一致性（Conforming）。一致性代码是指：特权级别设计为允许相同或更低特权等级的代码访问，但不允许提升特权级别进行写入的代码段。可以理解为是内核（高特权级）开放给用户程序（低特权级）的代码。
- `E`: 扩展方向（Extension），这个段向上（0）或向下（1）扩展。

### GDT 和选择子

全局描述符表其实就是一个数组…… 里面存的是很多描述内存段的信息（段描述符，后面再说）。

既然是储存了整个内存的段的情况，那它自然很巨大。所以机子专门维护了一个寄存器（全局描述符表寄存器，GDTR）指向这个表的地址。

![GDT 结构](gdt.png)

而 GDTR 是一个 48 位的寄存器。结构：

```plaintext
47                 15        0
+------------------+---------+
|  GDT 内存起始地址  | GDT 界限 |
+------------------+---------+
```

可以通过指令 `lgdt` 给 GDTR 赋值（但是不能直接 `mov lgdt, xxx`）。

这里你会发现描述 GDT 界限只用了 16 位，因此 GDT 最大的大小就是 $2^{16}=65536\texttt B$. 而一个段描述符需要 $64\texttt{b}=8\texttt B$ 的空间，因此这个 GDT 最多能够定义 $65536/8=8192$ 个段描述符。

那么这样一来，段寄存器里就只用存段描述符的索引就可以了（类比数组下标），也就是选择子。选择子是一个 16 位的数据，结构：

```plaintext
15                 3    2     0
+------------------+----+-------+
|    描述符索引值    | TI |  RPL  |
+------------------+----+-------+
```

这里描述符索引值用了 13 位，因此最多能够索引 $2^{13}=8192$ 个描述符。这和上面算出来 GDT 最多能够存储的描述符数量是相同的 ~~所以没算错~~。

然后是 TI，也就是表指示器（Table Indicator），表示这个选择子指向的是 GDT 中的描述符（1）还是 LDT 中的描述符（0）。

RPL 则表示了请求特权级（Requested Privilege Level），表示当前这个选择子希望请求的特权级别。

然后呢在进入保护模式之前，需要先构建 GDT，然后先往里面插入 4 个描述符：空描述符（第 0 个）、代码段（给内核用）、数据段和视频段（也就是 VGA 绑定的那块内存，实模式下是 `0xb8000` 到 `0xbffff` 的这一块）。

## 进入保护模式

好了！讲了这么多，可以开始敲键盘咯 >w<

### 打开 A20 地址线

计算机底层寻址是需要通过地址总线实现的。在实模式下，只有 20 条地址线可以用（A0 ~ A20），因此寻址的范围只有 $1\texttt{MB}$. 如果强行寻址超过 $1\texttt{MB}$ 的位置，系统会自动将其对 $1\texttt{MB}$ 取模，然后按余数寻址（实际上是因为只有 20 条线可用，因此丢弃了第 20 位以及它上面的所有位）。

然而现在我们需要进入保护模式，要操作更多的内存了。因此我们需要启用第 21 根（A20）地址线。具体做法是把 `0x92` 端口的第 1 位设置为 $1$ 就行了。

```x86asm
mov dx, 0x92
in al, dx ; 读入原来的值
or al, 0x02 ; 第 1 位设为 1
out dx, al ; 写入
```

> 8086 使用 20 条地址线，因此仅支持 $1\texttt{MB}$ 的内存，即地址范围为 `0x00000` 到 `0xfffff`。如果强行求高于 `0xfffff` 的位置，则会直接放弃所有高位，相当于对 $2^{20}$ 取模（其实是高位的地方根本没有地址线，所以就取不到那几位的值了）。这种过程叫做地址回绕。
>
> 然而到了 80286（第一个具有保护模式的 CPU）之后，地址线增加到 24 条。理论来说，如果直接访问超过 `0xffff` 的地址（不过也要小于 `0x10ffef`）并不会发生地址回绕。但是为了兼容性，它的实模式也应当和之前的 20 条地址线表现相同，也就是要求在实模式下发生回绕，保护模式下不回绕。为了解决这个问题，IBM 在键盘上加了一些输出线控制第 21 根地址线（A20）的可用性，也被称为 A20 Gate.
>
> 显然，A20 Gate = 1，则打开第 21 条地址线；否则，和 8086 一样，采用地址回绕。
>
> 因此在进入保护模式之前需要先打开 A20；而由于上面的历史遗留问题，打开 A20 是通过类似硬件操作的方式完成的。

### 加载 GDT 描述符

于是我们需要加载一下 GDT 里面的描述符。

现在 `boot.inc` 里给出一堆定义：

```x86asm
;------- GDT --------;
; GDT 中段界限单位为 4K
GDT_hi_G_4K equ 00000000_10000000_00000000_00000000b
; 有效地址，操作数 32 位
GDT_hi_D_32 equ 00000000_01000000_00000000_00000000b
GDT_hi_L_32 equ 00000000_00000000_00000000_00000000b
; 保留位 ; 代码段 32 位
GDT_hi_AVAL equ 00000000_00000000_00000000_00000000b
; 存在位
GDT_hi_P_OK equ 00000000_00000000_10000000_00000000b
; 特权级 0
GDT_hi_DPL0 equ 00000000_00000000_00000000_00000000b
; 特权级 1
GDT_hi_DPL1 equ 00000000_00000000_00100000_00000000b
; 特权级 2
GDT_hi_DPL2 equ 00000000_00000000_01000000_00000000b
; 特权级 3
GDT_hi_DPL3 equ 00000000_00000000_01100000_00000000b
; 是数据段（不是系统段）
GDT_hi_SGDT equ 00000000_00000000_00010000_00000000b
; 是代码段（不是系统段）
GDT_hi_SGCD equ 00000000_00000000_00010000_00000000b
; 是系统段
GDT_hi_SGSY equ 00000000_00000000_00000000_00000000b
; 代码段可执行，不可读，非一致性，已访问位 a 清 0 (x=1, r=0, c=0, a=0)
GDT_hi_TPCD equ 00000000_00000000_00001000_00000000b
; 数据段不可执行，向上扩展，可写，已访问位 a 清 0 (x=0, e=0, w=1, a=0)
GDT_hi_TPDT equ 00000000_00000000_00000010_00000000b
; 内核代码段描述符高 32 位
GDT_hi_CODE equ 00000000_1_1_0_0_1111_1_00_1_1000_00000000b
; 内核数据段描述符高 32 位
GDT_hi_DATA equ 00000000_1_1_0_0_1111_1_00_1_0010_00000000b
; 内核视频段描述符高 32 位。段基址中 8 位: 0x0b = 00001011b
GDT_hi_VDEO equ 00000000_1_1_0_0_0000_1_00_1_0010_00001011b
;------- GDT --------;
;----- Selector -----;
SEL_RPL0 equ 00b  ; 请求特权级 0
SEL_RPL1 equ 01b  ; 请求特权级 1
SEL_RPL2 equ 10b  ; 请求特权级 2
SEL_RPL3 equ 11b  ; 请求特权级 3
SEL_TI_G equ 000b ; 表指示器：GDT
SEL_TI_G equ 100b ; 表指示器：LDT
;----- Selector -----;
```

然后修改 `loader.s`！

有一些前置需要了解。x86 是有一套字（word）系统：

|名称|大小|定义命令|
|:-:|:----------:|:---:|
|字节 byte|$1\texttt B$|`db`|
|字 word|$2\texttt B$|`dw`|
|双字 double word|$4\texttt B$|`dd`|
|四字 quad word|$8\texttt B$|`dq`|

首先是建立 GDT 并插入 4 个描述符（实际上是直接分配了这么多的空间）：

```x86asm
SECTION LOADER vstart=LOADER_ADDR
    jmp loader_start ; 直接跳到后面执行，
GDT_BASE: ; 空描述符，防越界; 也是 GDT 基址
    dq 0
GDT_CODE_DESC: ; 代码段描述符
    dd 0x0000ffff ; 代码段的基址低 8 位为 0x0000，界限低 8 位为 0xffff
    dd GDT_hi_CODE ; 代码段高 32 位
GDT_DATA_DESC: ; 数据段描述符
    dd 0x0000ffff
    dd GDT_hi_DATA
GDT_DATA_VDEO: ; 视频段描述符
    dd 0x80000007 ; 视频段大小 0xbffff-0xb8000 +1 = 0x8000, 单位为 4K 则界限为 0x8000 / 4K - 1 = 0x07（注意要减 1）
    dd GDT_hi_VDEO
GDT_SIZ equ $ - GDT_BASE ; 计算当前 GDT 大小
GDT_LIM equ GDT_SIZ - 1  ; GDT 界限
times 60 dq 0 ; 填充 60 个空描述符
```

然后需要设置 GDTR 寄存器，使它指向 GDT 的表头（也就是 `GDT_BASE`）。先得出 GDTR 应该等于多少：

```x86asm
gdt_ptr dw GDT_LIM  ; 低  8 位是界限
        dq GDT_BASE ; 高 16 位是 GDT 地址
```

最后再预设一下代码段、数据段和视频段的选择子：

```x86asm
; 代码段的选择子
SEL_CODE equ (0x01 << 3) + SEL_TI_G + SEL_RPL0
; 数据段的选择子
SEL_DATA equ (0x02 << 3) + SEL_TI_G + SEL_RPL0
; 视频段的选择子
SEL_VDEO equ (0x03 << 3) + SEL_TI_G + SEL_RPL0
```

然后在 `jmp` 之后的地方加载 GDTR：

```x86asm
lgdt [gdt_ptr]
```

### 开启保护模式

好！内存地址问题解决了之后就可以进入保护模式了！那么要通过哪个开关打开保护模式呢？

实际上 CPU 又搞了一个寄存器让我们控制处理器的工作模式，叫做 `CR0`（control register 0）。结构（摘自 [CPU register x86 - OSdev wiki](https://wiki.osdev.org/CPU_Registers_x86#CR0)）：

| Bit |Label|Description|
|:---:|:---:|:---------:|
|`0`|PE|保护模式 Protected Mode Enable|
|`1`|MP|监视协处理器 Monitor co-processor|
|`2`|EM|协处理器硬件仿真 x87 FPU Emulation|
|`3`|TS|经历任务切换 Task switched|
|`4`|ET|支持一些扩展特性 Extension type|
|`5`|NE|启用数字错误等 Numeric error|
|`16`|WP|只读页是否可写入 Write protect|
|`18`|AM|内存对齐检查 Alignment mask|
|`19`|NW|是否禁用换从写透传（write-through）Not-write through|
|`30`|CD|禁用缓存 Cache disable|
|`31`|PG|启用分页机制 Paging|

那么很显然，如果要打开保护模式，就要将 `CR0` 中的 `PE` 位设置为 1.

```x86asm
mov eax, cr0
or eax, 0x01
mov cr0, eax
```

然后接下来就可以直接在保护模式下跑代码了吗？还要干一点活...

由于现在在保护模式下，调用任何一块内存都需要经过选择子 $\rightarrow$ GDT $\rightarrow$ 裸机地址实现，而众所周知的是代码也是被加载到内存里的。因此接下来我们需要刷新一下段描述符缓冲寄存器（不然就还是原先实模式下的值，是不正确的）。

以及，由于从 $8\texttt B$ 的实模式进入了 $32\texttt B$ 保护模式的，我们需要告诉汇编器接下来的代码要汇编成 $32$ 位的机器码。

```x86asm
    jmp dword SEL_CODE:pe_start ; 刷新流水线，后面讲
[bits 32] ; 伪指令，告诉 nasm 接下来汇编成 32 位的代码（会在指令前面加上 0x66 之类的前缀）
pe_start:
    mov ax, SEL_CODE
    mov ds, ax ; 将段描述符缓冲寄存器更新为选择子
    mov es, ax
    mov esp, LOADER_ADDR ; 设置栈顶指针
```

然后就可以在保护模式下写东西了！不过需要注意的是，在保护模式下就不能用 BIOS 中断了 qaq 所以只能用显存输出。

这里用字体颜色区分保护模式和实模式（实模式下用白色 `0x0f`，保护模式下用绿色 `0x02`）。

```x86asm
    mov ax, SEL_VDEO
    mov gs, ax ; 获取显存段
    mov byte [gs:0x0500], 'Q'
    mov byte [gs:0x0501], 0x02
```

然而显存输出是众所周知的麻烦。因此干脆写一个脚本算了。需要注意的是 $32$ 位下不能 `mov qword`！因此只能两个两个字符输出。

```python
s = 'QOS[KL ]: Activated Protected Environment.'
d = 0x0500 # 起始地址
f = 0x0200 # flag
while s != '':
    t = s[:2]
    s = s[2:]
    t = list(map(lambda x: f | ord(x), t))
    val = 0
    for i in t[::-1]:
        val = (val << 16) | i
    print(f'mov dword [gs:{hex(d)}], {hex(val)}')
    d += 4

# usage: python3 gen.py > tmp
```

### 运行失败？

然而直接用之前的 `make.sh` 并不能完整的运行：进入 Loader 后完全没有反应……

可以 `ls` 一下二进制文件夹：

```bash
QOS % ls -la bin/loader.bin
-rw-r--r-- 1040  8 16 16:56 loader.bin
```

加了 GDT 的 Loader 的二进制文件的大小已经超过了一个扇区（$512\texttt B$）。因此需要修改 `mbr.s` 中加载的扇区数量。为了方便，这里就直接加载 16 个扇区。

```x86asm
    ; ...
    mov eax, LOADER_SECT
    mov bx, LOADER_ADDR
    mov cx, 16
    ; 读取硬盘
    ; ...
```

然后还有自动汇编脚本也要改一改（其实就是写入磁盘这里改一下）：

```bash
nasm -I src/ src/mbr.s -o bin/mbr.bin
nasm -I src/ src/loader.s -o bin/loader.bin

dd if=bin/mbr.bin of=masterdisk.img count=1 bs=512 conv=notrunc
dd if=bin/loader.bin of=masterdisk.img count=16 bs=512 seek=1 conv=notrunc # here
```

这样子运行就成功了。
