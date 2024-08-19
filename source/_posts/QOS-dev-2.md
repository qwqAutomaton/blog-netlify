---
title: QOS 开发 2
date: 2024-07-22 07:20:12
tags:
  - 操作系统
  - 汇编
description: 不过这样还是太麻烦了 QAQ
---
# QOS 开发日记 2: 寄存器、显存和其它

## 寄存器

在实模式下有几个可用的寄存器：

|寄存器|助记名称|用途|
|------|--------|----|
|`ax`|累加器（accumulator）|常用于各种运算以及保存和外设输入输出的数据|
|`bx`|基址寄存器（base）|常用于储存地址|
|`cx`|计数器（counter）|常用于统计循环次数|
|`dx`|数据寄存器（data）|通常情况下只用于保存外设控制器的端口号地址|
|`si`|源变址寄存器（source index）|常用于字符串中的源地址|
|`di`|目的变址寄存器（destination index）|常用于字符串中的目标地址|
|`sp`|栈指针（stack pointer）|段基址为 `ss`，指向栈顶。`push` 和 `pop` 会对它造成修改|
|`bp`|基址指针（base pointer）|用于访问栈中间的数据。可通过 `ss:bp` 访问。不随 `push` 等改变|
|`flags`|标志寄存器|用于存放好多标记|

总结一下：寄存器 `ax` 到 `dx` 都是通用寄存器，啥都能存。

后面带 `i` 的一般是地址，带 `p` 的一般是指针，带 `s` 的一般是段地址。

关于这个 `flags` 寄存器，它里面的数据是按位读取的，如下：

```plaintext
                              F  L  A  G  S
 31  ...  21 20   19 18 17 16 15 14  13-12  11 10  9  8  7  6 5  4 3  2 1  0 
+--+-----+--+---+---+--+--+--+--+--+-------+--+--+--+--+--+--+-+--+-+--+-+--+
|  | ... |ID|VIP|VIF|AC|VM|RF|  |NT|  IOPL |OF|DF|IF|TF|SF|ZF| |AF| |PF| |CF|
+--+-----+--+---+---+--+--+--+--+--+-------+--+--+--+--+--+--+-+--+-+--+-+--+
```

其中，比较重要的标志位是：

- 计算标志：
  - `CF`（Carry Flag）进位标志：反映运算最高位是否进位或借位。
  - `PF`（Parity Flag）奇偶标志：反映运算结果低 8 位中 `1` 的个数的奇偶性。若有偶数个 `1` 则置 `1`，否则置 `0`。
  - `AF`（Auxiliary Carry Flag） 辅助进位标志：在字节操作时低半字节向高半字节进位或借位、字操作时低字节向高字节进位或借位，置 `1`，否则置 `0`（类似 `CF`）。
  - `ZF`（Zero Flag）零标志：用于判断结果是否为 `0`。
  - `SF`（Sign Flag）符号标志：用于反映运算结果的符号，运算结果为负则置 `1`，否则置 `0`。由于有符号数用补码表示，`SF` 与运算结果的最高位相同。
  - `OF`（Overflow Flag）溢出标志：反映有符号数加减运算是否溢出。
- 控制标志：
  - `TF`（Trap Flag）陷阱标志：当 `TF` 被设置为 `1` 时，CPU 进入单步模式，每执行一步指令后都产生一个单步中断,主要用于程序的调试。8086 / 8088 中没有专门用来置位和清零 `TF` 的命令，需要用其他办法。
  - `IF`（Interrupt Flag）中断标志：决定 CPU 是否响应外部可屏蔽中断请求。
  - `DF`（Direction Flag）方向标志：决定串操作指令执行时有关指针寄存器调整方向。当 `DF` 为 `1` 时，串操作指令按递减方式改变有关存储器指针值，每次操作后使 `SI`、`DI` 递减。

## 更新 MBR

这次我们直接操作显存来输出。

根据上一次的表格，我们可以直接往 `0xb8000` 到 `0xb8fff` 写数据达到输出的效果。

显示器上的每个字符用两个字节表示，如图：

```plaintext
31   ......   16 15 .... 1
[ 背景色 ]
+-+-+-+-+-+-+-+-+---------+
|K|R|G|B|I|R|G|B|  ASCII  |
+-+-+-+-+-+-+-+-+---------+
        [ 前景色 ]
```

其中 `K` 表示是否闪烁，`I` 表示前景亮度。

由此写出新的代码：

```x86asm
; src/mbr.s
; 主引导记录
;*-----------------------*;
SECTION MBR vstart=0x7c00 ; 表示起始地址为 0x7c00
; 初始化
    mov ax, cs ; 此时 cs 是 0x00，把它当零寄存器用用
    mov dx, ax
    mov es, ax
    mov ss, ax
    mov fs, ax ; 清空这些寄存器
    mov ax, 0xb800
    mov gs, ax ; 显存段地址
    mov sp, 0x7c00 ; 当前栈指针指向 0x7c00

; 清屏
    mov ax, 0x0600 ; 0x06, 0x00
    mov bx, 0x0700 ; 0x07, 0x00
    mov cx, 0x0000 ; 左上角 (0, 0) = (0x00, 0x00)
    mov dx, 0x184f ; 右下角 (79, 24) = (0x4f, 0x18)
    int 0x10 ; 调用中断

; 用显存打印字符串
    mov byte [gs:0x00], 'Q'
    mov byte [gs:0x01], 0x0a
    mov byte [gs:0x02], 'O'
    mov byte [gs:0x03], 0x0a
    mov byte [gs:0x04], 'S'
    mov byte [gs:0x05], 0x0a
    mov byte [gs:0x06], ':'
    mov byte [gs:0x07], 0x0a
    mov byte [gs:0x08], ' '
    mov byte [gs:0x09], 0x0a
    mov byte [gs:0x0a], 'H'
    mov byte [gs:0x0b], 0x8a
    mov byte [gs:0x0c], 'i'
    mov byte [gs:0x0d], 0x8a
    mov byte [gs:0x0e], '.'
    mov byte [gs:0x0f], 0x8a

; 无限循环，等待
    jmp $ ; $ 表示当前指令位置

; 把剩下的填充为 0
    times 510 - ($ - $$) db 0 ; $$ 表示当前节的开始位置，那么 $-$$ 就是已经写了多少
    db 0x55, 0xaa ; 填充最后两个字节。注意端序问题
```

此时再运行汇编、写入、模拟就可以了。

方便起见，写个脚本：

```bash
# make.sh
nasm src/mbr.s -o bin/mbr.bin
dd if=bin/mbr.bin of=harddisk.img count=1 bs=512 conv=notrunc
```

## 关于启动 bochs 后终端 <kbd>backspace</kbd> 等键映射被改成 <kbd>^?</kbd> 的解决方案

用 `screen` 跑就行了：

```bash
screen bochs -f .bochsrc
```

注意需要在连接到串口之前输入 `c`，否则可能会有奇怪的问题！
