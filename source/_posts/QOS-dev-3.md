---
title: QOS 开发 3
date: 2024-07-24 11:12:53
tags:
  - 操作系统
  - 汇编
description: 终于可以和硬盘交互了！暗搓搓写一个 bug 进去（？？？
---
# QOS 开发日记 3: 硬盘、内核和加载器

前情提要：写了个简陋的 MBR，然后通过显存写了字符串。

然而用显存写太麻烦了，于是后来又改回用系统中断写了。修改之后：

```x86asm
; 设置光标位置
; ah: 功能号 0x02
; bx: 光标所在页
; dx: 目标光标位置
    mov ax, 0x0200
    mov bx, 0x0000
    mov dx, 0x0000 ; 光标坐标 dh 行 dl 列
    int 0x10 ; 设定光标 (0, 0)
    mov ax, mbr_init
    mov bp, ax
    mov ax, 0x1300
    mov bx, 0x000f
    mov cx, 0x002d
    mov dx, 0x0000
    int 0x10 ; 输出字符串
    mov ax, 0x0200
    mov bx, 0x0000
    mov dx, 0x0100
    int 0x10 ; 光标放下一行
    jmp $
    ; 13 和 10 是 \r \n
    mbr_init db "QOS[MBR ]: Initialising master boot record.", 13, 10, "$"
```

众所周知，程序是储存在硬盘里的一段指令，而内核也是一种程序。那么为了运行真正的内核，我们需要先从硬盘加载它。因此我们先来看硬盘。

## 硬盘

> 根据 ATA 规范，所有符合 ATA 的驱动器必须始终支持 PIO 模式作为默认的数据传输机制。

现在比较流行的 SATA 其实就是一种 ATA，所以肯定也支持 PIO 啦~

嗯不过现在系统为了效率，通常会使用更复杂的读写模式，甚至走 PCIe 直接和 CPU 连起来。这里就不管那么多了……

### 端口问题

![硬盘读写端口](hd-port.png)

这边对这些端口做一些解释。

#### 读

`0x1f1` 返回错误信息，同样是按 bit 存储。结构大概是：

```plaintext
   0     1     2     3     4     5     6     7
+-----+-----+-----+-----+-----+-----+-----+-----+
| AMNF|TKZNF| ABRT| MCR | IDNF|  MC | UNC | BBK |
+-----+-----+-----+-----+-----+-----+-----+-----+
 未找到 未找到  命令   变更  未找到  内容 不可纠正 检测到
  地址  零磁道  终止   请求   ID    变更 数据错误  坏块
```

`0x1f6` 也是按 bit 存储的：

```plaintext
   0     1     2     3     4     5     6     7
+-----+-----+-----+-----+-----+-----+-----+-----+
|CHS柱头位或LBR地址24-27位| DRV |  1  | 寻址 |  1  |
+-----+-----+-----+-----+-----+-----+-----+-----+
                        表示选择      0: CHS
                        主盘/从盘     1: LBA
```

`0x1f7` 是状态寄存器端口，同样按 bit：

```plaintext
   0     1     2     3     4     5       6     7
+-----+-----+-----+-----+-----+-----+-------+-----+
| ERR |    无 效   | DAT |    无效    | DRDY | BSY |
+-----+-----+-----+-----+-----+-----+-------+-----+
  出错             硬盘数据            硬盘检测   表示
                   准备好了             正常   硬盘正忙
```

#### 写

此时有一些寄存器有别的功能。

`0x1f1` 是参数端口，传入写硬盘的参数。

`0x1f7` 是指令端口。这里主要用到的指令是：

- `0xEC`: 硬盘识别
- `0x20`: 读扇区
- `0x30`: 写扇区

### 具体操作

和硬件交互的方法，除了上次直接写到内存的某个指定区域（统一编址），还可以用 `in` 和 `out` 命令：

```x86asm
; 从 dx 输入数据
; 选择 al 还是 ax 取决于 dx 指代的寄存器是 8b 还是 16b
in al, dx
in ax, dx

; 输出数据到 dx
; 同样根据 dx 选择 al 或 ax
out dx, al
out dx, ax
```

而在与硬盘的交互中，如果写入了 `Command` 寄存器（即 `0x1f7` 端口），那么硬盘就开始执行命令了。所以应当保证 `Command` 最后写入。

所以~ 给出一个大概得操作顺序（primary 通道为例）：

1. 往 `0x1f2` 写入操作的扇区数量；
2. 往 `0x1f3` 到 `0x1f5` 写入操作的 LBA 地址的低 24 位；
3. 往 `0x1f6` 中 0-3 位写入 LBA 的高 4 位，设置第 6 位为 1 表示使用 LBA 模式，再设置第 4 位选择主盘或从盘；
4. 最后写入 `0x1f7` 表示命令；
5. 然后读取 `0x1f7` 的 `Status` 寄存器，判断硬盘工作是否完成；
6. 如果之前是在读硬盘，转到 7；否则 `ret` 走人；
7. 从 `0x1f0` 读数据（2B）。

## 加载 Loader

为了运行内核，我们需要先从硬盘加载它。~~这句话好像哪里见过~~

因此我们需要一个 Loader 加载它。

但是这个 Loader 也需要从硬盘里加载... ~~套娃呢~~

Loader 的功能：

- 加载内核
- 检查并确认内核完整性
- 设置环境
- 启动内核

不过这个 Loader 总得放在哪里吧...

众所周知在实模式下有两块可用的区域：`0x07e00` 到 `0x9fbff`（总共 622080B 可用）和 `0x00500` 到 `0x07bff`（总共 30464B 可用）。随便挑一块，就 `0x8000` 开始的这一块吧。

以及 MBR 占据了硬盘的第 0 扇区（LBA，如果是 CHS 的话就是第 1 扇区），那么 Loader 就放在第 1 扇区好了。

于是先在 `src/boot.inc` 里加一些宏定义：

```x86asm
LOADER_ADDR equ 0x08000 ; loader 加载地址
LOADER_SECT equ 0x01 ; loader 所在扇区编号
DISK_SECT_DIV2 equ 0x0100 ; 一个扇区需要读取的次数（大小 / 2，因为一次读取 2B）
```

然后重写一下 `src/mbr.s`:

```x86asm
; src/mbr.s
%include "boot.inc"
SECTION MBR vstart=0x7c00 ; 表示起始地址为 0x7c00
; 初始化/
    mov ax, cs ; 此时 cs 是 0x00，把它当零寄存器用用
    mov dx, ax
    mov es, ax
    mov ss, ax
    mov fs, ax ; 清空这些寄存器
    mov ax, 0xb800
    mov gs, ax ; 显存段地址
    mov sp, 0x7c00 ; 当前栈指针指向 0x7c00
    ; 清屏并输出字符串
    mov ax, 0x0600
    mov bx, 0x0700
    mov cx, 0x0000
    mov dx, 0x184f
    int 0x10
    ; 获取当前光标位置
    mov ax, 0x0200
    mov bx, 0x0000
    mov dx, 0x0000
    int 0x10
    ; 输出字符串。。。
    mov ax, str1
    mov bp, ax
    mov ax, 0x1301
    mov bx, 0x000f
    mov cx, 0x0018
    int 0x10
    mov ax, 0x0300
    mov bx, 0x0000
    int 0x10
    mov ax, str2
    mov bp, ax
    mov ax, 0x1301
    mov bx, 0x000f
    mov cx, 0x0017
    int 0x10
    ; 从硬盘读取 loader
    mov eax, LOADER_SECT
    mov bx, LOADER_ADDR
    mov cx, 1
    ; 读取硬盘
    mov esi, eax
    mov di, cx
    ; 设置扇区数量
    mov dx, 0x1f2
    mov al, cl
    out dx, al
    mov eax, esi
    ; 设置 LBA 地址 0-23b
    ; 循环右移然后用 al 取低 8 位
    mov dx, 0x1f3
    out dx, al
    mov cl, 8
    shr eax, cl
    mov dx, 0x1f4
    out dx, al
    shr eax, cl
    mov dx, 0x1f5
    out dx, al
    ; 设置 0x1f6 一堆东西
    shr eax, cl
    and al, 0x0f
    or al, 0xe0
    mov dx, 0x1f6
    out dx, al
    ; 设置 0x1f7 为读入（0x20）
    mov dx, 0x1f7
    mov al, 0x20
    out dx, al
.not_ready: ; 这里一直循环到硬盘准备好为止
    nop ; 相当于 sleep 一个时钟周期
    in al, dx ; 读入 Status 寄存器（0x1f7）
    and al, 0x88 ; 和 0b10001000 与一下，取第 3 和 7 位
    cmp al, 0x08 ; 第三位为 1（准备好）且第七位为 0（不忙）
    jnz .not_ready ; 如果不相等，跳回去
    ; 准备好读入了
    mov ax, di
    mov dx, DISK_SECT_DIV2
    mul dx ; 计算要读的次数。相当于 ax *= dx
    mov cx, ax ; cx 是 loop 的计数器
    mov dx, 0x1f0 ; 移动到 data 寄存器
.read_disk:
    in ax, dx ; 读入
    mov [bx], ax ; 写到内存里
    add bx, 2 ; 移动内存，一次 2B
    loop .read_disk
    ; 再输出一堆字符串。。。
    mov ax, 0x0300
    mov bx, 0x0000
    int 0x10
    mov ax, str4
    mov bp, ax
    mov ax, 0x1301
    mov bx, 0x000f
    mov cx, 0x0019
    int 0x10
    mov ax, 0x0300
    mov bx, 0x0000
    int 0x10
    mov ax, str3
    mov bp, ax
    mov ax, 0x1301
    mov bx, 0x000f
    mov cx, 0x0018
    int 0x10
    jmp LOADER_ADDR ; 跳到 loader 地址运行
    str1 db "QOS[MBR]: MBR running.", 0x0d, 0x0a, "$"
    str2 db "QOS[MBR]: Loading KL.", 0x0d, 0x0a, "$"
    str3 db "QOS[MBR]: Starting KL.", 0x0d, 0x0a, "$"
    str4 db "QOS[MBR]: MBR finished.", 0x0d, 0x0a, "$"
; 把剩下的填充为 0
    times 510 - ($ - $$) db 0 ; $$ 表示当前节的开始位置，那么 $-$$ 就是已经写了多少
    db 0x55, 0xaa ; 填充最后两个字节。注意端序问题
```

当然我们还得写 loader。

```x86asm
; src/loader.s
%include "boot.inc"
SECTION LOADER vstart=LOADER_ADDR ; 起始地址是 0x8000
    mov ax, 0x0300 ; 查光标位置，0x03
    mov bx, 0x0000
    int 0x10
    ; 输出字符串
    mov ax, str11
    mov bp, ax
    mov ax, 0x1301 ; 输出字符串
    mov bx, 0x000f
    mov cx, 0x0017
    int 0x10
    jmp $ ; 反复跳转这里

    str11 db "QOS[KL ]: KL running.", 0x0d, 0x0a, "$"
```

那么汇编命令也需要修改。

```bash
# 汇编，-I 表示包括这个目录
nasm -I src/ src/mbr.s -o bin/mbr.bin
nasm -I src/ src/loader.s -o bin/loader.bin
# 写到模拟硬盘里，seek=1 代表跳过一个块从第 1 号块里开始写（编号从零开始）
dd if=bin/mbr.bin of=masterdisk.img count=1 bs=512 conv=notrunc
dd if=bin/loader.bin of=masterdisk.img count=1 bs=512 seek=1 conv=notrunc
```

运行，成功。
