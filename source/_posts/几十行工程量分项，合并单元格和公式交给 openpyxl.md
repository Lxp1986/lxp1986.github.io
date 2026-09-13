---
title: 几十行工程量分项，合并单元格和公式交给 openpyxl
date: 2026-09-13 22:35:00
categories: [办公自动化]
tags: [Python, openpyxl, Excel, 工程量, 表格打印, 报价]
description: 工程量计算表用 openpyxl 生成 xlsx 的完整写法：标题行、表头、序号列三种合并方式，数值与单位为什么必须分列，number_format 和列宽怎么设，公式写进格子不等于算过（data_only 读回是 None，实测），以及打印前要加的冻结行与重复标题行。附带可直接跑通的脚本。
---

工程量计算表这活，麻烦不在算，在排版。一个项目分三大类、十几个分项，每个大类底下挂几个子项，序号列要竖着合并，单位单独一列，报出去的数还得统一两位小数。手工做的时候，复核阶段淤厚从 12 cm 改成 15 cm，工程量全变，合并区域跟着错位，等于重做一遍。

现在这张表是脚本出的：分项和工程量在 Python 里算完，openpyxl 负责落成 xlsx。参数改完重跑一条命令，表连格式一起重出。

## 先确认版本对得上

macOS 自带的 `/usr/bin/python3` 里就带 openpyxl（我这台是 3.1.5），先看一眼：

```bash
python3 -c "import openpyxl; print(openpyxl.__version__)"
```

没有就装：

```bash
python3 -m pip install -U openpyxl
```

写 `python3 -m pip` 而不是直接 `pip`，能保证装进当前这个解释器。如果你脚本里用的是 Homebrew 的 python，命令里的 python3 得换成同一个路径，否则跑起来报 ModuleNotFoundError。

## 数字和单位分开放

后面所有格式能不能生效，就看这一步：

```python
rows = [
    ("一", "人工清淤", "m³", 492.60, 60.00),
    ("", "土渠清淤（1:1）", "m³", 2916.13, 60.00),
    ("", "溪河清淤（1:1.25）", "m³", 295.75, 60.00),
    ("二", "除草清障", "m²", 1180.00, 6.50),
]
```

工程量存 float，单位存字符串，别拼成一格写成 `"492.60m³"`。拼起来这格就变成文本，甲方把表导进他自己的汇总表时这列一个数都加不起来——被这么问过一回，对方原话是"你这列怎么全是字"。

## 三种合并写法

```python
ws.merge_cells("A1:F1")                                              # 标题横跨整行
ws.merge_cells(start_row=3, start_column=1, end_row=5, end_column=1) # 序号列纵合并
```

记住一条：**值只存在左上角那一格**。A3:A5 合并后 A3 是"一"，A4、A5 读出来是 None；往 A4 写值会直接抛异常（`'MergedCell' object attribute 'value' is read-only`）。所以顺序是先写值、后合并。

![合并单元格只认左上角那一格](/img/openpyxl-merge-rules.svg)

## 数字格式与列宽

```python
ws.cell(row=r, column=4, value=qty).number_format = "0.00"      # 工程量两位小数
ws.cell(row=r, column=5).number_format = "#,##0.00"             # 单价带千分位
ws.column_dimensions["B"].width = 24
```

`number_format` 只管显示，格子里存的还是原数，不影响计算。规则写一次，所有分项一起生效，这是比手工点格式刷省事的地方。

## 公式写进去不等于算过了

合价那列写的是公式：

```python
ws.cell(row=r, column=6, value="=D{0}*E{0}".format(r))
ws.cell(row=total, column=6, value="=SUM(F3:F{0})".format(total - 1))
```

openpyxl 不计算公式，它只是把 `=D3*E3` 这个字符串塞进单元格，等你用表格软件打开时才算。刚生成的表里，合价格子其实是空的。

读回来更要小心：`load_workbook(path, data_only=True)` 读的是缓存值，脚本刚生成的文件没有缓存，读到的是 None（我实测确认）。要是下一步还要拿合价去干别的活——汇总到总表、或者塞进报价书文档——别去读公式格，在 Python 里算好写数值；或者先让 LibreOffice 无头转一遍，把缓存算出来：

```bash
soffice --headless --convert-to xlsx --outdir /tmp /tmp/工程量计算表.xlsx
```

转完再 `data_only` 读一次，合价格子就从 None 变成数了（我这边 F3 读出来是 29556，对应 492.60 × 60）。

## 打印前加两行设置

工程量表最后是要打出来签字的，这两行不加，打出来就是废纸：

```python
ws.freeze_panes = "A3"            # 冻结标题行，滚动不错行
ws.print_title_rows = "1:2"       # 每页重复打印标题
ws.page_setup.orientation = "landscape"
ws.page_setup.paperSize = ws.PAPERSIZE_A4
```

竖向 A4 装不下六列，右边两列会被切到第二页。改横向、加上标题行重复，一页一页摊开看，每页顶上都有"序号/分项名称/单位/工程量…"，签完字归档也认得出来。

## 完整脚本

```python
from openpyxl import Workbook
from openpyxl.styles import Font, Alignment, Border, Side, PatternFill

rows = [("一", "人工清淤", "m³", 492.60, 60.00),
        ("", "土渠清淤（1:1）", "m³", 2916.13, 60.00),
        ("", "溪河清淤（1:1.25）", "m³", 295.75, 60.00),
        ("二", "除草清障", "m²", 1180.00, 6.50)]

wb = Workbook(); ws = wb.active; ws.title = "工程量"
thin = Side(style="thin", color="808080")
bd = Border(left=thin, right=thin, top=thin, bottom=thin)
font = Font(name="宋体", size=10)

ws.merge_cells("A1:F1")
ws["A1"] = "渠道清淤工程量计算表"
ws["A1"].font = Font(name="宋体", size=14, bold=True)
ws["A1"].alignment = Alignment(horizontal="center", vertical="center")

for i, t in enumerate(["序号", "分项名称", "单位", "工程量", "单价（元）", "合价（元）"], 1):
    c = ws.cell(row=2, column=i, value=t)
    c.font = Font(name="宋体", size=10, bold=True); c.border = bd
    c.fill = PatternFill("solid", fgColor="DCE6F1")
    c.alignment = Alignment(horizontal="center", vertical="center")

for i, (no, name, unit, qty, price) in enumerate(rows):
    r = 3 + i
    ws.cell(row=r, column=1, value=no)
    ws.cell(row=r, column=2, value=name)
    ws.cell(row=r, column=3, value=unit)
    ws.cell(row=r, column=4, value=qty).number_format = "0.00"
    ws.cell(row=r, column=5, value=price).number_format = "#,##0.00"
    ws.cell(row=r, column=6, value="=D{0}*E{0}".format(r)).number_format = "#,##0.00"
    for col in range(1, 7):
        cell = ws.cell(row=r, column=col); cell.border = bd; cell.font = font
        cell.alignment = Alignment(horizontal="center" if col in (1, 3) else "right",
                                   vertical="center")

ws.merge_cells(start_row=3, start_column=1, end_row=5, end_column=1)   # 写完值再合并

total = 3 + len(rows)
ws.merge_cells(start_row=total, start_column=1, end_row=total, end_column=5)
c = ws.cell(row=total, column=1, value="合计")
c.alignment = Alignment(horizontal="right", vertical="center")
t = ws.cell(row=total, column=6, value="=SUM(F3:F{0})".format(total - 1))
t.number_format = "#,##0.00"; t.font = Font(name="宋体", size=10, bold=True)
for col in range(1, 7):
    ws.cell(row=total, column=col).border = bd

for col, w in zip("ABCDEF", (8, 24, 6, 12, 12, 14)):
    ws.column_dimensions[col].width = w
ws.freeze_panes = "A3"
ws.print_title_rows = "1:2"
ws.page_setup.orientation = "landscape"
ws.page_setup.paperSize = ws.PAPERSIZE_A4
wb.save("工程量计算表.xlsx")
```

跑完用 `openpyxl.load_workbook` 读回来核一下合并区域，上面这段脚本出来的是 `['A1:F1', 'A3:A5', 'A7:E7']`，对得上就没问题。

什么时候别用脚本：表结构还在天天变、列要临时加减的阶段，改脚本比手改格子还慢。脚本适合结构定了、要反复重出的表——工程量计算表、清单表都是这一类。我这套定型之后，同一个项目的表重出过四遍，改的都是参数，没再动过格子。
