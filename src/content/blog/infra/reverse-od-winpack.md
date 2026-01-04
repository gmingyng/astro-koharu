---
title: Reverse入门——用OD手脱Winpack0.39final壳
link: reverse-od-winpack
catalog: true
date: 2024-01-18 06:53:00
description: 用OD手脱Winpack0.39final壳
tags:
  - infra
  - reverse
  - winpack
categories:
  - Infra
---

-------------------------------------------------------------------
目标程序："C:\Windows\SysWOW64\notepad.exe"

操作环境：Windows11

使用工具：Winpack0.39final, 32dbg, lordpe
-------------------------------------------------------------------
<h2>预处理</h2>

 首先，在 C:\Windows\SysWOW64 目录下可以找到记事本的 32 位版本 notepad.exe，用 winpack 对他加壳得到 notepad_upx1.exe。
 
 先用 lordpe 分析目标程序的 pe 头，
[![](http://121.40.138.116/wp-blog/wp-content/uploads/2024/01/c055919af603d9903ca378fd8e53efb.png)](http://121.40.138.116/wp-blog/wp-content/uploads/2024/01/c055919af603d9903ca378fd8e53efb.png)
可以看到加壳后文件被分为两个区，.Upack 里是空的，这是因为 upack 把原本分区数据压缩后放入.rsrc 区中，运行时解压缩后才会把数据放回.Upack 区里，脱壳就是找到程序的 OEP。

 用 x32dbg 打开 notepad_upx1.exe,[![](http://121.40.138.116/wp-blog/wp-content/uploads/2024/01/7dd8b693039c55f3ed268b3412fb22b.png)](http://121.40.138.116/wp-blog/wp-content/uploads/2024/01/7dd8b693039c55f3ed268b3412fb22b.png)
可以看到程序的 EP 是 00D10B67，在第二节区。由于 OD 把数据也识别成代码的关系，往上一翻会发现 pushad 消失了，但是不影响代码执行。这里用两种方法找到 OEP。

<h2>寻找OEP</h2>
<h3>根据跨段指令寻找</h3>

 因为最后代码会被解压缩到第一个节区，所以必然有一个大的 jmp 或者 call eax。ctrl+b 输入 E9 查找 jmp 指令,
[![](http://121.40.138.116/wp-blog/wp-content/uploads/2024/01/a458378ab030c304d322c8d32b21fc6.png)](http://121.40.138.116/wp-blog/wp-content/uploads/2024/01/a458378ab030c304d322c8d32b21fc6.png)
最后后一个 jmp 到 00CB7190 位于.Upack 区，直接锁定,
[![](http://121.40.138.116/wp-blog/wp-content/uploads/2024/01/deec71a2ff7053edfc702dbd231e54c.png)](http://121.40.138.116/wp-blog/wp-content/uploads/2024/01/deec71a2ff7053edfc702dbd231e54c.png)
F4 到对应位置跳转过去,
[![](http://121.40.138.116/wp-blog/wp-content/uploads/2024/01/0098fa3990dcd4edc3c3de4d47041ac.png)](http://121.40.138.116/wp-blog/wp-content/uploads/2024/01/0098fa3990dcd4edc3c3de4d47041ac.png)

接下来调用 v32dbg 自带的 scylla 修复，把 00CB7190 设为 OEP 自动寻找 IAT 并导入(出错的导入模块可以先直接删除，之后再看能不能运行)。
[![](http://121.40.138.116/wp-blog/wp-content/uploads/2024/01/4d0ec5b0ddc0ef9e5854d24fe262d38.png)](http://121.40.138.116/wp-blog/wp-content/uploads/2024/01/4d0ec5b0ddc0ef9e5854d24fe262d38.png)
dump 并 fix dump 分别得到 filename_dump.exe 和 filename_dump_SCY.exe 文件，后者就是我们要的脱壳程序，运行成功。

<h3>根据堆栈平衡原理寻找</h3>

 在程序去到 OEP 的前一刻大概率要恢复寄存器环境，可以对程序在 EP 的 ESP 所存地址下硬件断点，多 F9 几次即可找到 OEP。



