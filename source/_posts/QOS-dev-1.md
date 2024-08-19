---
title: QOS 开发 1
date: 2024-07-21 09:38:26
tags:
  - 操作系统
  - 汇编
description: 应该算是操作系统开发的 Hello World 了吧？
---
# QOS 开发日记 1: 环境配置与简易 MBR

## 环境配置

选择 bochs 作为 x86 模拟器。安装：

```bash
brew install sdl2 # 依赖
brew install bochs
```

选择 nasm 作为汇编器。安装：

```bash
brew install nasm
```

项目文件结构：

```plaintext
QOS/
+---src/          源码
+---bin/          输出的二进制文件
+---harddisk.img  模拟硬盘文件
+---.bochsrc      bochs 配置文件
```

## bochs 配置

### 配置启动盘

用 bochs 自带的 `bximage` 即可。根据它的提示输入选项。

硬盘种类选择 `flat`，页大小（size of sector）选 512，总大小 60M。

名字就叫 `harddisk.img` 好了。

### 配置启动文件

这里有很多坑。。。。直接贴 .bochsrc 罢

```yml
# 设置内存为 32 MB
megs: 32

# 设置机器 BIOS
romimage: file=/opt/homebrew/Cellar/bochs/2.8/share/bochs/BIOS-bochs-latest
# 设置 VGA BIOS
vgaromimage: file=/opt/homebrew/Cellar/bochs/2.8/share/bochs/VGABIOS-lgpl-latest

# 设置启动盘为硬盘（不要软盘 qwq）
boot: disk

# 设置日志输出
log: bochs_out.log

# 禁用鼠标
mouse: enabled=0

# 启用键盘并设置映射
# 这里应当要选 sdl2-pc-us.map
# 注意这样设置之后会导致当前终端的输入出锅，需要重启终端
# 典型问题：vim 中输入 <backspace> 被识别为 ^?
keyboard: keymap=/opt/homebrew/Cellar/bochs/2.8/share/bochs/keymaps/sdl2-pc-us.map

# 设置硬盘相关
# 启用 ata0 控制器，指定其 IO 端口地址（主要 0x1f0，次要 0x3f0），终端请求线为 14
ata0: enabled=1, ioaddr1=0x1f0, ioaddr2=0x3f0, irq=14
# ata0 可以挂载两个盘，这里先只挂一个主盘
ata0-master: type=disk, path=/Users/macbookair/Desktop/QOS/harddisk.img, mode=flat
```

保存退出，然后运行

```bash
bochs -f .bochsrc
```

使用配置文件启动模拟器。模拟器启动之后会告诉你串口号，直接 `screen` 过去即可。

这时进入的是调试器，输入 `c` 表示 `continue`，继续运行。

当然现在运行是报错的：`no bootable device`。

## MBR 编写

主引导记录（master boot record）是机器启动之后运行的第二块代码（第一块是 BIOS）。

### 前置芝士

启动操作系统的时候，机器会先运行**ROM**里的 BIOS，做一些和硬件相关的检查，并建立中断向量表（IVT）。同时，刚开机的时候，x86 架构进入的是实模式，也就是用绝对地址的模式。此时由于地址线只有 20 条，按字节寻址只能包含 1 MB 的地址（从 `0x00000` 到 `0xfffff`）。这 1M 的空间分布是这样的：

|起始地址|结束地址|总大小|用途描述|
|--------|--------|------|--------|
|`0xfffef`|`0xfffff`|16B|BIOS 入口|
|`0xf0000`|`0xfffef`|65520B|系统 BIOS|
|`0xc8000`|`0xeffff`|163840B|映射硬件适配器的 ROM 或内存映射式 IO|
|`0xc0000`|`0xc7fff`|32767B|显示适配器 BIOS|
|`0xb8000`|`0xbffff`|32768B|文本显示适配器|
|`0xb0000`|`0xb7fff`|32768B|黑白显示适配器|
|`0xa0000`|`0xaffff`|65536B|彩色显示适配器|
|`0x9fc00`|`0x9ffff`|1024B|扩展 BIOS 数据区（EBDA）|
|`0x07e00`|`0x9fbff`|622080B|可用|
|`0x07c00`|`0x07dff`|512B|MBR 加载地址|
|`0x00500`|`0x07bff`|30464B|可用|
|`0x00400`|`0x004ff`|256B|BIOS 数据区|
|`0x00000`|`0x003ff`|1024B|中断向量表|

系统使用 16 位的 `cs` 和 `ip` 寄存器，利用分段机制进行寻址（一般来说是 `cs:ip`）。但是此时寻址都是 20 位的，需要一些处理才能用两个 16 位寄存器表示。处理方式：`cs` 左移 4 位再加上 `ip`，写成 `C++` 就是：

```cpp
(cs << 4) + ip
```

在启动时，`cs:ip` 强制初始化为 `0xf000:0xfff0`，那么初始的地址就是 `0xffff0`。事实上这个位置就是一句 `jmp`。

然后在 BIOS 执行的最后，它会检查磁盘的某个特定扇区（0 盘 0 道 1 扇区），如果这个扇区末尾两个字节是 `0x55 0xaa`，那么它就认为这个扇区是活动的，然后把它加载到 `0x07c00`。我们刚刚设置了扇区大小为 512B。最后的最后 BIOS 执行指令 `jmp 0x0000:0x7c00` 跳转到那个位置，开始执行。

（由于 x86 平台采用小端序，第 511 字节应当是 `0xaa`，512 才是 `0x55`。）

### 简易 MBR

我们现在只用做到清屏、输出字符串、反复 `jmp` 实现等待即可。

关于系统中断：向寄存器 `ah` 存入功能号，然后 `int 中断号`。注意是向 `a` 的高位传值。

清屏：功能号 `0x06`，中断号 `0x10`。

输出：功能号 `0x13`，中断号 `0x10`。

获取光标位置：中断号 `0x03`。

于是我们直接写代码即可。

```x86asm
; src/mbr.s
; 主引导记录
;*-----------------------*;
SECTION MBR vstart=0x7c00 ; 表示起始地址为 0x7c00
    mov ax, cs ; 此时 cs 是 0x00，把它当零寄存器用用
    mov dx, ax
    mov es, ax
    mov ss, ax
    mov fs, ax ; 清空这些寄存器
    mov sp, 0x7c00 ; 当前栈指针指向 0x7c00

; 清屏。实际上是上卷，上卷所有行就是清屏了
; ah: 功能号 0x06
; al: 上卷行数，如果为 0 就是全部
; bh：上卷行属性
; (cl, ch) 窗口左上角 (x, y)
; (dl, dh) 窗口右下角 (x, y)
; 无返回值
    mov ax, 0x0600 ; 0x06, 0x00
    mov bx, 0x0700 ; 0x07, 0x00
    mov cx, 0x0000 ; 左上角 (0, 0) = (0x00, 0x00)
    mov dx, 0x184f ; 右下角 (79, 24) = (0x4f, 0x18)。VGA 文本模式一行只有 80 字符
    int 0x10 ; 调用中断

; 获取光标位置
; ah: 功能号 0x03
; bh: 待获取的光标页号
; 返回：
; ch: 光标开始行
; cl: 光标结束行
; dh: 光标所在行
; dl: 光标所在页
    mov ax, 0x0300 ; 0x03, 0x00
    mov bx, 0x0000 ; 0x00, 0x00
    int 0x10

; 打印字符串
; ah: 功能号 0x13
; al: 打印模式：0 显示字符串，光标返回起始位置；1 显示字符串，光标跟随；2 显示字符串和属性，光标返回起始位置；3 显示字符串和属性，光标跟随
; es:bp: 字符串地址。由于已经初始化 es = cs，所以不用管了
; bh: 储存要显示的页号
; bl: 字符属性，0x02 表示黑底绿字
; cx: 字符串长度
; (dl, dh): (光标所在页, 光标所在行)
    mov ax, message ; 输入字符串地址
    mov bp, ax ; 挪到 bp 里面
    mov cx, 0x0016 ; 字符串长度为 22=0x16（不算行末 \0）
    ; dx 寄存器直接使用上面获得的光标位置
    mov bx, 0x0002 ; 当前页为第 0 页，属性 0x02
    mov ax, 0x1300 ; 0x13, 0x00
    int 0x10

; 无限循环，等待
    jmp $ ; $ 表示当前指令位置；$$ 

; 数据
    message db "QOS: test print string"

; 把剩下的填充为 0
    times 510 - ($ - $$) db 0 ; $$ 表示当前节的开始位置，那么 $-$$ 就是已经写了多少
    db 0x55, 0xaa ; 填充最后两个字节。注意端序问题
```

保存然后汇编：

```bash
nasm src/mbr.s -o bin/mbr.bin
```

然后可以用 `xxd` 观察 `mbr.bin`，发现最后两字节确实是 `55aa`：

```bash
QOS % xxd bin/mbr.bin
00000000: 8cc8 89c2 8ec0 8ed0 8ee0 bc00 7cb8 0006  ............|...
00000010: bb00 07b9 0000 ba4f 18cd 10b8 0003 bb00  .......O........
00000020: 00cd 10b8 357c 89c5 b916 00bb 0200 b800  ....5|..........
00000030: 13cd 10eb fe51 4f53 3a20 7465 7374 2070  .....QOS: test p
00000040: 7269 6e74 2073 7472 696e 6700 0000 0000  rint string.....
00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000060: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000070: 0000 0000 0000 0000 0000 0000 0000 0000  ................
[ 以下省略好多行。。。 ]
000001d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001f0: 0000 0000 0000 0000 0000 0000 0000 55aa  ..............U.
```

同样通过 `ls` 也可以看出 `mbr.bin` 大小正好是 512B。

汇编之后就是写入 ROM 了。使用 `dd` 命令：

```bash
dd if=bin/mbr.bin of=harddisk.img count=1 bs=512 counv=notrunc
```

`dd` 命令格式：

- of=FILE: 指定被写入的文件为 FILE
- if=FILE: 指定输入文件为 FILE
- bs=BLOCKS 指定块的大小
- count=指定块数
- seek=BLOCKS 指定写入到文件时要跳过几个块
- conv=CONVS 指定转换方式，notrunc 表示不截断文件

写入之后使用同样的命令运行，成功！