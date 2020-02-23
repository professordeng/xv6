---
title: 8. 通用代码
---

## 1. string.c

由于系统比较简单，并没在系统中提供单独的 C 语言库，因此一些字符串处理功能需要 XV6 自己提供，实现细节在 [string.c](https://github.com/professordeng/xv6-expansion/blob/master/string.c)。

1. `memset()` 用于将一段内存空间用指定的数值（8 bit）填充。
2. `memcpy()` 将 n 个字节从一个内存地址 `v1` 拷贝到另一个内存地址 `v2` 处。
3. `memmove()` 用于将 `src` 地址开始的 n 个字节移动到 `dst` 地址的位置（前后区间可能有重叠，需要区分处理），`memcpy()` 本质上也是 `memmove()`。
4. `strncmp()` 用于比较两个字符串，如果不相同则返回第一个不同字符的位置。
5. `strncpy()` 和 `safestrcpy()` 用于拷贝 n 字节长的字符串。
6. `strlen()` 用于返回一个字符串的长度。 