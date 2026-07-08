---
title: "capability权限认知"
date: 2025-10-28 09:23:10
categories: [知识仓库]
tags: [capability]
---

### capability权限是什么

linux为细化root权限，将root权限细化为了40多个具体的权限，通过setcap，getcap来调用，作用于可执行文件，只在二进制程序运行时才查询相应权限


__linux本身的shell并不能直接调用这些细化的权限，只有一些能调用内核的语言才能靠自己的模块来调用，如C语言。python等高级语言能实现类似效果大多也是因为封装的C。__


python如果在文件有CAP_SETUID的前提下，想要调用并提权root shell使用下面的代码

```python
    import os 
    os.setuid(0)
    os.system("/bin/bash")    
```
### capability后面的ep是什么

__e是有效集，p是允许集，e内的权限必须p内也有。内核执行二进制程序时，会检查有效集e内的capability权限并执行。__

__C 语言可以通过 cap_set_proc() 这类系统调用，把 Permitted 里的能力手动添加到 Effective 集合完成激活；但是 Python、Shell、PHP 这类高级语言没有提供操作进程能力集合的封装接口，没法写代码去激活p里的权限，所以只设置+p几乎无法利用提权，必须带上+e，直接将capability权限放在有效集__

在为文件添加权限时可以只有p，但是只有e会报错。


![mm](/assets/img/0602cap.jpg)


### 一些常用的capability权限：

1. __CAP_SETUID__
：允许进程调用setuid()/seteuid()随意修改自身用户 ID，可以直接把 UID 改成 0（root），不需要 SUID 权限位。
运维场景：需要程序在多个普通用户之间切换执行任务

2. __CAP_SETGID__
：可以随意修改进程的用户组 GID，切换任意用户组权限

3. __CAP_DAC_OVERRIDE__
：直接绕过系统所有文件、文件夹的读写执行权限校验。哪怕文件是000权限、属于 root，也能直接读、写、删除。

4. __CAP_DAC_READ_SEARCH__
：只能绕过文件读权限、文件夹浏览权限，不能写文件

5. __CAP_CHOWN__
：可以使用chown命令，修改系统任意文件的属主、属组，哪怕不是文件所有者。

6. __CAP_FOWNER__
：绕过文件属主校验，可以修改不属于自己文件的权限、时间戳、ACL 权限、特殊属性（比如 immutable 锁定文件）。

7. __CAP_NET_BIND_SERVICE__
：普通用户进程可以绑定1024 以下的特权端口（80、443、22、3306 这类端口默认只有 root 能监听）。

8. __CAP_NET_ADMIN__
：完整网络管理权限：修改网卡 IP、路由表、防火墙规则、网卡混杂模式、VPN 配置。

9. __CAP_SYS_ADMIN__:（Linux 里的 “超级权限大包”）
几乎包含绝大多数系统管理员操作：
挂载 / 卸载磁盘、修改 swap 分区、配置磁盘配额
调用 chroot 切换根目录、配置内核参数
大部分需要 root 的系统操作都依赖这个权限
提权方式：挂载 root 分区、chroot 逃逸、写入内核配置拿到最高权限

10. __CAP_SYS_MODULE__
：允许加载、卸载 Linux 内核驱动模块。
高危利用：上传恶意内核模块，直接拿到系统最高持久化 root 权限，很难被查杀。

11. __CAP_SYS_PTRACE__
：可以附加跟踪系统内任意进程（包括 root 进程），读取进程内存、注入代码、劫持 root 进程执行命令。
经典利用：注入 sshd、sudo 这类 root 进程，窃取密钥或执行反弹 Shell。

12. __CAP_SYS_BOOT__
：可以调用系统 API 直接重启、关闭服务器。

13. __CAP_SYS_TIME__
：修改系统时间、硬件时钟，常用于审计日志篡改、绕过时间类安全策略

14. __CAP_KILL__：可以给系统内任意进程发送杀死信号，能杀掉 root 运行的进程。

15. __CAP_MKNOD__：可以创建设备文件（比如 /dev/null、磁盘设备），可通过创建设备文件提权。

16. __CAP_SETFCAP__：可以给其他文件赋予 / 移除 Capability 权限，拿到后可以给自己常用命令绑定CAP_SETUID永久提权。
