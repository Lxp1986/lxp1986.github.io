---
title: 用 qpdf 合并扫描件交资料，顺手解开权限锁和打开密码
date: 2026-09-15 22:38:00
categories: [数码工具]
tags: [qpdf, PDF, 命令行, 扫描件, macOS]
description: 收方资料要合成一份 PDF 交出去：按页序合并、只挑几页、去掉禁打印禁复制的权限锁、解开打开密码、提交前线性化，全部用 qpdf 一条命令完成。
---

上周交收方资料，手上是手机拍的七张签证单、监理发来的两份"禁止打印"PDF，还有一份甲方给的加密图纸（打开要输密码）。以前我的做法是打开预览，一张张往侧边栏里拖，再导出成一份——页数一多，拖到第四十张手就开始抖，碰上打不开的还得先截图再拼。这类活现在全交给 qpdf，一个 brew 装的小工具。

先说清楚它管什么：qpdf 只动 PDF 的壳——合并、拆页、权限、加密、线性化，不重新编码里面的图片。所以手机拍的扫描件合完，清晰度还是原来那个。

## 装

```bash
brew install qpdf
qpdf --version
# qpdf version 12.4.1
```

## 合并：把散着的 PDF 拼成一份

文件按顺序命名成 01、02、03，然后：

```bash
qpdf --empty --pages 01.pdf 02.pdf 03.pdf -- 收方资料.pdf
```

三个位置要记牢：`--empty` 表示从零开始、不读任何输入文件；`--pages` 后面列的是"要哪几页"；`--` 之后跟的才是输出文件名。**整条命令里最后那个位置参数永远是输出**。

这就是最容易敲错的地方。写成这样：

```bash
qpdf 收方资料.pdf --pages 01.pdf 02.pdf --
# qpdf: an output file name is required; use - for standard output
```

报错是因为 `收方资料.pdf` 被当成了输入文件，qpdf 到最后也没找到输出名，退出码 2。文件多的时候可以用通配符：

```bash
qpdf --empty --pages *.pdf -- 全部资料.pdf
```

## 只挑几页出来

整包里只要第 2 到第 4 页：

```bash
qpdf 全部.pdf --pages . 2-4 -- 部分.pdf
# 结果 3 页
```

那个 `.` 表示"前面这个文件自己"。页码写法：全部是 `1-z`，一段是 `2-4`，单页直接写 `3`。要拆成单页文件：

```bash
qpdf --split-pages 图纸.pdf 第%d页.pdf
# 实测生成 第1页.pdf … 第4页.pdf
```

## 权限锁：能打开，但点不了打印和复制

先看它锁了什么：

```bash
qpdf --show-encryption 锁定.pdf
```

输出里出现 `print high resolution: not allowed`、`extract for any purpose: not allowed`，就是禁止打印、禁止复制文字。

这种锁是写在文件里的一组权限标记，靠阅读器自觉遵守，不是加密——文件内容本来就是明文。对方设的"打开密码"是空的，所以一条命令就能出一份干净副本：

```bash
qpdf --decrypt 锁定.pdf 可打印.pdf
```

出来的文件里没有任何加密字典，能打印、能选字、能合并进别的 PDF。怎么判断是不是这一类：`--show-encryption` 输出里 `User password` 那行是空的，就是只有权限锁、没有打开密码。

## 连打开都要密码的

```bash
qpdf --password=1234 --decrypt 加密图纸.pdf 图纸.pdf
```

要是还想按页合并，密码必须挨着那个输入文件写：

```bash
qpdf 加密图纸.pdf --password=1234 --pages . -- 图纸.pdf
```

实测写成 `qpdf --empty --password=1234 --pages 加密图纸.pdf -- out.pdf` 会报 `invalid password`：`--empty` 没有输入文件，密码被它吃掉了，轮到真读文件时手上已经没密码了。

## 交出去之前线性化

```bash
qpdf --linearize 收方资料.pdf 收方资料-提交.pdf
```

把交叉引用表挪到文件开头，对方用浏览器或网盘预览时能先把第一页画出来。代价是文件略大：实测 22834 字节变成 23442 字节，涨 2.7%。

## 打不开的 PDF，先体检

```bash
qpdf --check 别人发来的.pdf
```

干净的文件返回 0，输出 `No syntax or stream encoding errors found`。文件真坏（比如网盘传输只下了一半）会一边刷 warning 一边重建交叉引用表，最后给退出码 3。**3 不等于失败**，文件通常还能救出来，只是个别数据流（比如某页的图）可能已经空了，得翻一遍确认。

## 合并会不会把图压糊

不会。qpdf 只是把页搬进同一个文件，不重新采样。实测两份分别 11610 字节和 11598 字节的 PDF 合起来是 22583 字节，比两份之和还小一点——重复的字体和颜色空间被合并去重了。要压体积那是 Ghostscript 的活，用途不一样。

![qpdf 处理扫描件的三条常用路线](/img/qpdf-pdf-flow.svg)

现在我的固定流程是：手机拍完按 01、02、03 命名，`qpdf --empty --pages ... -- 收方资料.pdf` 合成，再 linearize 出一份提交版，翻一遍确认页序和方向没错就发出去。带锁的文件在合并前单独解密一份留着，因为 qpdf 能读得动空打开密码的锁文件，但对方阅读器不一定肯给打印——顺手输出一份无锁版，少一轮来回。当然，动手前确认文件是自己的、或者对方允许处理。
