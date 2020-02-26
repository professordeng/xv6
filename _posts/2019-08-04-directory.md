---
title: 4. 目录
---

前面讨论的内容，要不就是盘块操作，要不就是已经获得文件的索引节点情况下的操作，它们都不需要文件名作为输入信息。现在来讨论如何根据文件名查找文件的索引节点，以及文件路径名构成的树形目录结构。

## 1. 目录查找

根据文件名，利用 [dirlookup()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L523) 在目录文件中查找目录项。字符串比较是通过 [namecmp](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L514) 完成的（实际上就是简单封装了 `strncmp()`）。 

在文件目录的查找过程中，偏移量每次移动 `sizof(de)`。

[dirlink()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L551) 负责创建一个链接文件，首先确认没有重名文件，其次找到一个空闲目录项，然后填写内容。

[skipelm](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L581) 用于逐级分解路径名。