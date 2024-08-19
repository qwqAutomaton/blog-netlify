---
title: LaTeX、Typst 和 Markdown 使用体验
date: 2024-08-11 14:13:51
tags:
  - 摸鱼
description: 用了几天的 Typst 和 LaTeX，感觉 Typst 有点像高配 Markdown？但是不太理解为啥 Typst 里要加那么多编程式的东西 qnq
---
# LaTeX、Typst 和 Markdown 使用体验

之前从 OI-wiki 的 check 里偶然看见了某个叫 Typst 的东西，于是就去搜索了一下。[Typst 官网指路](typst.app)

稍微阅读了一下文档，感觉挺好的！于是直接 `brew install` 开始用 Typst 写东西了 owo

使用起来感觉还行，而且 VS Code 里的插件（用的是 [Tinymist Typst](https://marketplace.visualstudio.com/items?itemName=myriad-dreamin.tinymist)）支持不错。可以实时预览修改（这个的话 Markdown 也可以做到）并且导出成 PDF，而且渲染的速度很快（和 $\LaTeX$ 相比 qwq）。预览里还能直接点击跳转到源文件的对应部分！~~并且这个预览确实比 Markdown 插件的预览好看一点 qwq~~

插件缺点的话大概是预览里选中文字之后不知道为什么没办法取消选择... 以及有的时候会不小心点到哪里然后跳转回去导致又要往下翻好久 QAQ

Typst 优点大概是能支持比 Markdown 更多的排版选择，以及对内嵌公式的整个翻新。Typst 的公式里打特殊字符和函数什么的不需要像 Markdown（里的 $\LaTeX$）一样用宏堆起来（然后就是满屏的 `\` qaq）。比如说打这一组公式（从{% post_link treap-complexity-proof %}里搞过来的 owo）：

$$
\begin{aligned}
\mathbb E(\text{dep}(x))&< 2\sum_{j=2}^n\dfrac 1j\\
&<2\sum_{j=2}^n\int_{j-1}^j\dfrac 1x\text dx\\
&=2\int_1^n\dfrac 1x\text dx\\
&=2\ln n=O(\log n)
\end{aligned}
$$

Markdown（实际上是 $\LaTeX$）：

```latex
$$
\begin{aligned}
  \mathbb E(\text{dep}(x)) &< 2 \sum_{j=2}^n \dfrac 1j\\
  &< 2 \sum_{j=2}^n \int_{j-1}^j\ dfrac 1x \text dx\\
  &= 2 \int_1^n \dfrac 1x \text dx\\
  &= 2 \ln n = O(\log n)
\end{aligned}
$$
```

Typst（`highlight.js` 甚至不支持 Typst…… 只能先用纯文本代替了 qaq）：

```typst
// Typst 公式用单个 $ 包裹即可
// $ 两边加上空格就是 display mode
// 或者换行掉也可以
// 换行是单个反斜杠
$
EE("dep"(x)) &< 2 sum_(j=2)^n 1/j\
&< 2 sum_(j=2)^n integral_(j-1)^j 1/x dif x\
&= 2 integral_1^n 1/x dif x\
&= 2 ln n = O(log n)
$
```

超喜欢它把 `a/b` 直接转化成大分数的形式！而且里面对特殊符号的处理基本都是自然语言，所有就不会出现像 $\LaTeX$ 里那么巨大多的 `\` 了！。

以及 Typst 的行内公式也可以做到指定语言然后高亮！

不过缺点就是 Typst 公式里的符号系统有点麻烦…… 比如打出这一串箭头：

$$
\leftarrow \rightarrow \leftrightarrow \iff
$$

$\LaTeX$：

```latex
\leftarrow \rightarrow \leftrightarrow \iff
```

Typst（不过这些 `.` 是不考虑顺序的）：

```plaintext
arrorw.l arrow.r arrow.l.r arrow.l.r.double.long
```

还有一个就是不能像 $\LaTeX$ 那样设置公式内部某一段的颜色之类的 qaq（倒是可以全局设置）

LaTeX 给我的感觉就是重！但是很好用 owo

而且 `tikz` 宏包可以很方便地（内嵌）画图，特别是画一些函数图像、图论的图很好用。

总的来看，重量：Markdown < Typst < LaTeX，使用难度 Markdown < LaTeX < Typst（主要原因是网上 Typst 相关的博客 / 文档太少了 qaq），渲染出的 pdf 的正式程度 Markdown < Typst ≈ LaTeX。

嗯。
