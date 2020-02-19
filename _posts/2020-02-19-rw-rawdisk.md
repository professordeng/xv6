---
title: 2. 磁盘裸设备的读写
date: 2019-11-03
---

1. 准备磁盘镜像文件 `rawdisk.img`，大小为 `RAWSPACE_SIZE=32MB`

2. 修改 `Makefile` 的 `qemu` 仿真命令选项 `QEMUOPTS`， 加入 

   ```bash
   -driver file=rawdisk.img,index=2,media=disk,format=raw
   ```

3. 系统启动时对 `rawdisk` 盘的初始化

4. 提供系统调用对 `rawdisk` 的读写

5. 编写应用程序检验读写结果，检查 `rawdisk.img` 内容是否符合所写结果。

