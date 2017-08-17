---
title: 树莓派驱动学习笔记(1)
date: 2015-07-04 13:22:17
tags:
---

以下内容来自阅读<http://elinux.org/RPi_Software>的总结

树莓派的启动：

***在上电后，竟然不是CPU首先开始工作。第一个开始工作的是GPU上的一块RISC电路***

--------------
First stage bootloader：从SD卡读取Second stage bootloader

Second stage bootloader：(bootcode.bin)读取GPU固件，启动GPU

GPU firmware (start.elf)：在这个固件中，GPU来启动CPU。

User code：最后…………CPU开始执行的用户程序

以上文件在firmware/boot/目录下，其中_cd表示cutdown,是裁剪版；_x是测试版