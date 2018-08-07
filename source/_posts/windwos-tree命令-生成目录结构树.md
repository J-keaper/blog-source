title: windwos tree命令 生成目录结构树
author: Keaper
date: 2018-07-15 16:49:52
tags:
---
tree命令的格式是
```dos
tree [drive:][path] [/F] [/A]

   /F   显示每个文件夹中文件的名称。
   /A   使用 ASCII 字符，而不使用扩展字符。
```

可以重定向输出到文件中：
例如：
```dos
tree . /F > tree.txt
```