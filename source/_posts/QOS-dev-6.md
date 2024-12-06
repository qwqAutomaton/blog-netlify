---
title: QOS 开发 6
date: 2024-10-10 16:47:59
tags:
  - 操作系统
  - 汇编
  - c
description: 第六篇惹！继续和内存玩~ 然后也是时候结束 loader 让她正式加载内核了 owo
---

# QOS 开发日记 6：内存分页和简单内核

众所周知，保护模式很重要的一个功能就是保护内存。然而仅仅使用 GDT 并不能保证完全的内存安全（其实主要是会降低内存的利用率 qwq）。因此需要引入分页：把一个进程使用的内存限制在一页里，实现某种程度的内存隔离。

我们考虑对内存的引用方式（内存地址的表示方式）：

- 在实模式下我们用 20 位表示地址，这 20 位就代表了物理地址（`0x00000` \~ `0xfffff`，总共 $4\texttt {GB}$）。

- 在仅使用 GDT 的保护模式下，用选择子先在 GDT 中选择描述符，然后再通过偏移量和描述符得到最终的物理地址。

上面两种表示方式都可以统一为“段 : 偏移”的形式（实模式中的第 16-19 位就看做段）。通过段和偏移得出最终地址（20 位）的流程称为“段部件”。实模式和仅用 GDT 的保护模式下段部件的输出就是真实的物理地址；而在引入分页之后，它则会输出“线性地址”，而线性地址会通过页部件（各级页表）转换成最终的物理地址。

## 页表

和储存内存段的 GDT 类似，我们也需要一个表来存储页的信息（即页表）。一般来说，一级页表中的页的大小为 $4\texttt{KB}$，那么机子的 $4\texttt{GB}$ 内存就被拆分成 $1048576$ 个（即 $1\texttt M$ 个）页。页表中的每个项占 $4\texttt B$（后面说）用来表示这个页的各种信息（物理基址、权限等等）。那么如何安排这个页表？

一个朴素的想法是直接开 $4\texttt B\times 1048576=4\texttt{MB}$ 这么巨大的线性表存在内存里，然后要调用的时候直接在这个表里取某一项即可。比如段部件输出了地址 `0x1234_5678`，那么先取高 20 位（第 12-31 位）作为页表的偏移，即找到页表从头开始的第 `0x12345` 项；然后取出这一项对应的物理地址基址（比如是 `0x9876_5000`），再加上低 12 位（第 0-11 位）作为偏移就得到了真实的物理地址，也就是 `0x9876_5678`.

当然，就像 GDT，我们也需要保存页表的开始地址。这里使用的是特殊寄存器 `CR3`. 因此操作流程就是：

- 通过 `CR3` 找到页表。
- 取高 20 位，乘以 4 后（页表一项的大小为 4 字节）就是所需的页表项的偏移。
- 把这个偏移加上页表基址找到所需项地址。
- 通过这个地址找到这一项对应的物理地址，然后再加上低 12 位。

也就是：

```x86asm
; 用虚拟地址获取物理地址
; 参数 eax: 虚拟地址
; 返回 eax: 物理地址
get_physical_addr:
    mov ebx, eax
    mov ecx, eax
    rsh ebx, 12  ; 取高 20 位
    lsh ebx, 2   ; 计算页表偏移
    add ebx, cr3 ; 页表地址
    mov eax, [ebx]
    and ecx, 0xfff ; 取低 12 位
    add eax, ecx
    ret
```

然而一条内存条贵得要死…… 这么浪费怎么行？而且后面如果要升级成 64 位的系统，储存的表的数量还得翻一个平方，指定存不下（悲

于是考虑分治的想法，把页表分级。一级页表中存放 1024 个索引，常驻内存；一级页表中的索引指向二级页表，一些二级页表常驻内存（如给内核分配的内存），另一些等待用户进程创建。这样就能省下暂时不用的内存的空间啦~

具体来说，每个二级页表对应了 $4\texttt{MB}$ 的物理内存（$4\texttt{GB}/1024=4\texttt{MB}$），总共包含 $4\texttt {MB}/4\texttt{KB}=1024$ 个页（每个页对应 $4\texttt{KB}$ 的物理内存）。

那么这时寻址过程就变成了这样 owo（假设需要找到虚拟地址 `0x0123_4567` 对应的物理地址）

- 把这个地址分割成高 10 位、中间 10 位、低 12 位三个部分，分别对应页目录项、页表项、实际偏移。此处拆成了 $4 (\texttt{0000\_0001\_00b})$、$564 (\texttt{10\_0011\_0100b})$ 和 $1383 (\texttt{0101\_0110\_0111b,0x567})$.
- 在 PDT 中寻找第 $4$ 项（即偏移为 $4\times 4\texttt B=16\texttt B$ 的项），假设这个 PDE 对应的页表地址为 `0xabcd_e000`;
- 在对应的页表（即 `0xabcd_e000`）处寻找第 $564$ 项，假设这一项对应的物理地址为 `0x9876_5000`
- 用页表对应的地址加上偏移，就得到它的真实地址为 `0x98876_5567`.

写成 C 大概是：

```c
typedef unsigned addr_t; // 地址是 32 位无符号数
extern addr_t CR3; // 存放 PDT 地址
addr_t get_physical_address(addr_t vadd)
{
    unsigned hi10 = vadd >> 22,
      mid10 = (vadd << 10) >> (10 + 12),
      low12 = (vadd << 20) >> 20;
    addr_t page_table = CR3 + 4 * hi10;
    addr_t page = *page_table + 4 * mid10;
    return *page | low12;
}
```

不过实际上页表当中不止存有地址，还有一些控制位（但是基本思路差不多）。

然后就是页目录项（一级）和页表项（二级）的结构：

![页目录和页表](page.png)

可以注意到这里描述地址只用了 20 位。这是因为每个内存页的大小都是固定的 $4\texttt {KB}$，页的基址一定是类似 `0x____000` 的形式（最后 12 位为 0），所以就直接舍弃掉低 12 位，只保存高位。剩下还有一些控制位：

- `AVL`：Available 可用位。
- `G`：Global 全局位。若为 1，则该页会在缓存（TLB）中一直保存（快表）。顺便这里加个知识点：清空 TLB 有两种方式，一种是 `invlpg` 指令针对单独虚拟地址条目进行清理，还有一种是修改 `CR3` 寄存器，这将直接清空 TLB.
- `PAT`：Page Attribute Table，用于指定页的属性。这里直接置 0.
- `D`：Dirty 脏位，指示当前页是否被修改过（被弄脏了 qaq）。
- `A`：Access，访问位。指示当前页是否被访问过（由 CPU 赋值）。
- `PCD`：Page-level Cache Disable，表示是否禁用页级缓存。同样置 0.
- `PWD`：Page-Level Write Through，页级通写位。表示该页不仅在内存，还存在在高速缓存。同样置 0.
- `US`：User / Supervisor，为 $1$ 表示 User，任意特权级别（$0\sim 3$）都可以使用此页；为 $0$ 表示 Supervisor，特权级别 3 不可访问，而 $0\sim 2$ 可以访问此页。
- `RW`：Read / Write，读写位，指示是否可读写。
- `P`：存在位，和段描述符里面的一样，为 1 表示存在。

那么~ 如果要启用页机制，只需要把这些页表放在内存里，然后让 cpu 知道页表在哪里就可以了。具体来说就是：

- 生成页表，放在内存中某个位置。
- 将这个地址放入 `CR3` 中。
- 将 `CR0` 的 `PG` 位置 $1$.

由于 `CR3` 寄存器被用来存储页表的位置，因此也被称为 PDBR (Page Directory Base Register).

然后 `CR3` 的结构是这样的：

```plaintext
 31               12 11      5   4     3   2  0
+-------------------+---------+-----+-----+----+
| 页表基址的 31-12 位 |         | PCD | PWT |    |
+-------------------+---------+-----+-----+----+
```

这里出现的 `PCD` 和 `PWT` 和之前页表中的这几个控制位的作用是一样的，即 是否启用页级缓存 和 页级通写位，也一样置 0.

## 实操 owo

~~拖了好久的说 qwq~~

首先是一些常量定义：

```x86asm
; in file boot.inc

; ... ;

;------- Page constant ------;
PDT_ADDR equ 0x10000 ; 页目录表地址

PGE_P_OK equ 1b   ; 存在内存中
PGE_RWOK equ 00b  ; 可读写
PGE_RWNO equ 10b  ; 不可读写
PGE_USER equ 100b ; 用户级
PGE_SPVS equ 000b ; 特权级
;------- Page constant ------;
```

那么很显然需要先把存放一级页表的空间开辟出来，然后将其清零：

```x86asm
; in file loader.s, after entering PE

.clear_pdt_space:
    mov ecx, 4096 ; 4K items
    mov esi, 0    ; bias
.clear_pdt_space_loop:
    mov byte [PDT_ADDR + esi], 0
    inc esi
    loop .clear_pdt_space_loop
```

清零之后就要开始放内容了！将虚拟内存的 `0x`（即高 $1\texttt G$）分给内核，剩下的 $3\texttt G$ 留给用户。然后这高 $1\texttt G$ 的虚拟地址对应的物理地址则是低 $1\texttt G$（因为各种内存映射 / MBR / 内核加载器都是在最低的 $1\texttt M$ 里面）。那么 PDT 中第一个页表就应该对应物理地址 `0x0000_0000` 到 `0x003f_ffff`. 而这个页表紧邻 PDT，即基址为 `0x0010_1000`（PDT 基址 `0x0010_0000` 再加上 PDT 的大小 $4\texttt K$ 也就是 `0x1000`）。那么这个页表对应的 PDE 就应该是：

- 由于舍弃基址的低 12 位，这个 PDE 的高 31-12 位为 `0x0010_0`.
- `US` 置 $1$ 表示用户内存，所有特权级皆可访问。
- `RW` 置 $1$ 表示内存可读写。
- `P` 置 $0$ 表示内存存在。
- 剩下的就全部置 $0$ 不管他辣！

因此这一项 PDE 的值就是 `0x0010_0006`.

```x86asm
.insert_pde: ; 插入这一项表
    mov eax, PDT_ADDR
    add eax, 0x1000 ; 表的位置（紧接在 PDT 末尾）
    mov ebx, eax ; 先存一下，后面用
    or eax, PGE_USER | PGE_RWOK | PGE_P_OK ; 控制位
    ; 插入 PDT！总共要插 2 次
    mov [PDT_ADDR + 0x0000], eax ; PDT 第一项
    mov [PDT_ADDR + 0x0c00], eax ; PDT 的 0x300 项，对应虚拟内存中的 0xc000_0000
```

这里把两个地方的虚拟地址都映射到了同一个页表当中。当然也要插入 PDT 本身的地址（放在最后一个位置）：

```x86asm
    sub eax, 0x1000 ; 减完了就是 PDT 位置加上控制位
    mov [PDT_ADDR + 4092], eax ; 4092 处就是最后一个了
```

然后就是插入第一个页表了。这个页表的每一项（每个内存页）直接按照顺序排下去就行了。

```x86asm
; 虚拟地址：0x0 ~ 0xfffff 以及 0xc0000000 ~ 0xc00fffff
.insert_pte:
    mov ecx, 256 ; 1 个页表 1M，一页 4K，总共 256 个页
    mov esi, 0 ; 偏移（下标）
    mov edx, PGE_USER | PGE_RWOK | PGE_P_OK ; 控制位
.insert_pte_loop:
    ; *4 是因为一项 4B
    mov [ebx + esi * 4], edx
    inc esi
    add edx, 4096 ; += 4K，下一页的物理地址
    loop .insert_pte_loop
```

然后就是把给内核预留的 $1\texttt G$ 的空间对应的页目录元素（也就是 `0xc000_0000` 到 `0xffff_ffff`）插到 PDT 里面。

```x86asm
create_kernel_pde:
    mov eax, PDT_ADDR
    add eax, 0x2000 ; 接下来的页表都紧跟着第一个页表
    or eax, PGE_USER | PGE_RWOK | PGE_P_OK
    mov esi, 769 ; 偏移量，0xc000_0000 对应的就是第 769 项
    mov ecx, 254 ; 总共需要插入的项数（减掉最末尾指向 PDT 本身的）
.create_kernel_pde_loop:
    mov [PDT_ADDR + esi * 4], eax
    inc esi
    add eax, 0x1000 ; 下一个表
    loop .create_kernel_pde_loop
```

这样子整个表就建好了。接下来只要把 GDT 更新一下（因为之前储存的并不是虚拟地址，需要转化之后才能用），然后把 PDT 的位置放到 `CR3` 里，再把 `CR0` 的对应控制位（也就是第 31 位，最高位）置 1 即可。

更新 GDT：

```x86asm
    sgdt [gdt_ptr] : 把原来 gdtr 的内容暂时存放到 gdt_ptr 指向的内存
    mov ebx, [gdt_ptr + 2] ; gdt_ptr 指向第一项，往后移 2B 就是 gdt 的地址了
    ; GDT 地址往后移动 3 项（一项 8B）就是视频段的选择子
    ; 再取选择子第 4B（基址）加上 0xc000_0000 即可
    or dword [ebx + 0x18 + 4], 0xc000_0000
    ; 同样把 GDT 的地址加上 0xc000_0000
    add dword [gdt_ptr + 2], 0xc000_0000
    ; esp 栈指针的地址也要加上 0xc000_0000 变成虚拟地址
    add esp, 0xc000_0000
```

以及其他：

```x86asm
    mov eax, PDT_ADDR
    mov cr3, eax ; 设置 CR3 的值为 PDT 的位置

    mov eax, cr0 ; 将 CR0 的页表启用
    or eax, 0x8000_0000
    mov cr0, eax

    lgdt [gdt_ptr] ; 加载 gdt
```

~~于是就搞定啦~~

搞定什么搞定！我们在 `0xc00` 到 `0xfff` 创建了总共 $255$ 个页目录，但是只创建了紧随其后的一个页表。为什么这样不会出问题呢？

我们虽然只建立了一个二级页表，却形成了事实上的 $5$ 段内存映射。它们是（下面的地址如果不加说明，都是指虚拟地址）：

- `0x0000_0000` 到 `0x000f_ffff`，共 $1\texttt{MB}$：这一段指向了我们创建的第一个页表，也就是物理地址的 `0x0000_0000` 到 `0x000f_ffff`.
- `0xc000_0000` 到 `0xc00f_ffff`，共 $1\texttt{MB}$：这一段也指向第一个页表，对应物理地址的 `0x0000_0000` 到 `0x000f_ffff`.
- `0xffc0_0000` 到 `0xffc0_0fff`，共 $4\texttt{KB}$：这一段比较怪。手动模拟一下取物理地址的流程，这一段的高 10 位为 `1111'1111'11b`，因此取到的是 PDT 的最后一项，而这一项指向 PDT 本身。那么实际上在第二步取二级页表对应项时是将 PDT 本身看做一个二级页表（注意页目录表项和页表项格式的相似性）。此时第二步取得的中间 10 位为 `00'0000'0000b`，也就是页表的第一项（对应 PDT 的第一项），指向物理地址 `PDT_ADDR+0x1000=0x101000` 也就是第一项页表所在的位置。因此这段地址对应物理地址 `0x0010_1000` 到 `0x0010_1fff`.
- `0xfff0_0000` 到 `0xfff0_0fff`，共 $4\texttt{KB}$：同上分析高 10 位为 `1111'1111'11b`，对应页表最后一项，指向 PDT 本身；中 10 位为 `11'0000'0000` 也就是 `0x300`，对应 PDT 的第 `0x300` 项（偏移 `0xc00` 处）。而此处也是指向第一个页表所在的位置（即 `0x0010_1000`），因此这一段地址对应的物理地址也是 `0x0010_1000` 到 `0x0010_1fff`.
- `0xffff_f000` 到 `0xffff_ffff`：同样，高 10 位为 `1111'1111'11b` 指向 PDT 的最后一项，中 10 位为 `11'1111'1111b` 也指向最后一项。因此这段内存就是 PDT 占据的空间，即它对应的物理地址为 `0x0010_0000` 到 `0x0010_0fff`.

实际上我们是通过把 PDT 本身也看做二级页表，减少了需要额外创建并占用空间的二级页表的数量。

总结一下，实模式下可访问的 $1\texttt{MB}$ 空间（物理地址 `0x0_0000` 到 `0xf_ffff`）的虚拟地址为 `0x0000_0000` 至 `0x000f_ffff`（即与物理地址相等）或 `0xc000_0000` 至 `0xc00f_ffff`（经过页表的映射），因此 GDT 中视频段的段描述符只要把基址加上一个 `0xc000_0000` 就可以变成虚拟地址了；访问 PDT 自身则是通过访问虚拟地址 `0xffff_f000` 到 `0xffff_ffff` 也就是最高的 $4\texttt{KB}$ 内存。

实际上上面的推算内容可以直接在 bochs 中用 `info tab` 命令得到：

```bash
<bochs> info tab
cr3: 0x000000100000 # PDT 的位置
# 格式：虚拟内存 -> 物理内存
0x0000000000000000-0x00000000000fffff -> 0x000000000000-0x0000000fffff
0x00000000c0000000-0x00000000c00fffff -> 0x000000000000-0x0000000fffff
0x00000000ffc00000-0x00000000ffc00fff -> 0x000000101000-0x000000101fff
0x00000000fff00000-0x00000000ffffefff -> 0x000000101000-0x0000001fffff
0x00000000fffff000-0x00000000ffffffff -> 0x000000100000-0x000000100fff
```

## 内！核！

~~话说已经快忘记现在在写的是什么东西力（悲）~~

不管怎么说，「内核加载器」总得加载一下内核。那么接下来就是一个最简单的内核咯~

写内核直接用汇编手搓也不是不行，但是这样实在是太痛苦了 QAQ

所以决定用 C：

```c
// in file src/kernel/main.c
int main()
{
    while (1);
    return 0;
}
```

没错就是把 `jmp $` 用 C 写出来（逃

不过显然我们不可以直接把它用 `gcc` 一步到位搓成可执行文件。我们实际上需要先将其编译成对象文件（用选项 `-c`）；然后再用链接器给其中的符号定位、确定入口后链接成一个 ELF 文件；最后再把这个 ELF 文件写到硬盘里就好了。

### ELF 文件的格式

ELF 全称 Executable and Linkable Format（可执行且可链接格式）。下面先给出 ELF 文件的格式（部分翻译自 [Executable and Linkable Format - Wikipedia](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)）：

文件由一个 ELF 头（ELF Header）和后面的文件数据组成。其中，数据可以包含以下三项内容：

- 程序头表（Program header table）：用于描述 $0$ 个或几个内存段。
- 节头表（Section header table）：用于描述 $0$ 个或几个节。
- “真正的”数据：由上面的 pht 和 sht 引用而来。

这张图（引自 wiki）描述了 ELF 文件的结构，即一个 ELF 文件即可以（从程序头表的角度）看做数个程序（每个程序拥有一些段），也可以（从节头表的角度）看做一系列节（section）的组合。

![ELF 文件的结构](elf-layout.png)

其中，这些段包含了这个 ELF 运行所需的各种数据，而节储存了和链接、重定位相关的信息。这个文件中的每一个字节都至多属于一个节（section）；当某一个字节不属于任何一个节的时候，被称为“孤儿字节”（orphan byte）。

### ELF 头

ELF 头决定了整个 ELF 文件的基本信息，比如架构、位数等等。对于一个 32 位的 ELF 文件，她的 ELF 头的长度为 $52\texttt B$. 她位于文件的开始位置。

这个头可以用结构体来表示（在文件中的位置和结构体中变量的顺序一致）：

```c
typedef uint16_t Elf32_Half;
typedef uint32_t Elf32_Word;
typedef uint32_t Elf32_Addr;
typedef uint32_t Elf32_Off;
typedef struct
{
    unsigned char e_ident[16]; /* Magic number and other info */
    Elf32_Half e_type;         /* Object file type */
    Elf32_Half e_machine;      /* Architecture */
    Elf32_Word e_version;      /* Object file version */
    Elf32_Addr e_entry;        /* Entry point virtual address */
    Elf32_Off e_phoff;         /* Program header table file offset */
    Elf32_Off e_shoff;         /* Section header table file offset */
    Elf32_Word e_flags;        /* Processor-specific flags */
    Elf32_Half e_ehsize;       /* ELF header size in bytes */
    Elf32_Half e_phentsize;    /* Program header table entry size */
    Elf32_Half e_phnum;        /* Program header table entry count */
    Elf32_Half e_shentsize;    /* Section header table entry size */
    Elf32_Half e_shnum;        /* Section header table entry count */
    Elf32_Half e_shstrndx;     /* Section header string table index */
} Elf32_Ehdr;
```

可以直接把 ELF 读到一个 `char` 数组中之后，直接用一个结构体的指针来读取，即

```c
char buf[10000];
// ...
Elf32_Ehdr *ptr = (Elf32_Ehdr *)buf; // 直接取最开始的位置即可
// ptr->e_ident[0] ...
```

<未完待续>
