---
title: "gopher执行redis命令有返回但超时错误"
date: 2025-11-01 10:13:10
categories: [错误及解决]
tags: [redis,gopher,协议]
---

__使用gopher协议远程使用redis命令时，虽然有数据传回但依旧超时__

![mm](/assets/img/1101gopher01.jpg)


__原因__：出现这个问题的原因在于，gopher返回结果的前提是收到完整报文，而redis那边在执行完命令后并不会直接断开连接而是继续等待后续命令的输入；

gopher就一直等它返回完整报文，两边互等，最后超过gopher的超时时长，就会报超时错误


__解决__：让redis在执行完命令后再添加一条断开连接的命令，QUIT就行

![mm](/assets/img/1101gopher02.jpg)

*ps:中间的报错是因为redis没有往指定目录写入的权限*