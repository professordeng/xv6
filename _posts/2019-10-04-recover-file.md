---
title: 4. 恢复被删除的文件
---

XV6 中文件是由一个 `inode` 来表示的，删除文件及删除 `inode`，在底层实现中，会调用 [bfree()](https://github.com/professordeng/xv6-expansion/blob/master/fs.c#L80) 删除文件所占用的所有盘块，而这个函数实现很简单，只是将代表盘块的位图置 0 而已，并没有真正将磁盘中的数据删除，所以如果盘块被置位后一直没有被其他进程调用并覆盖，理论上是可以找回来的，下面我们将修改内核，添加相应代码，找回在 shell 中删除的文件。