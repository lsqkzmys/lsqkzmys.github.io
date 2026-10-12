---
title: "网络接口ACL和DHCP以及NAT"
date: 2025-03-05 
categories: [知识仓库,网络]
tags: [ACL,DHCP,NAT]
---


## **<span style="color:#3498db">ACL</span>**

**ACL**中文全称访问控制列表，简单易懂，用于控制访问

只要启用了acl，默认拒绝所有。匹配从上至下，匹配到一条就退出

这里用在网络接口，即控制进出网络接口的流量。

思科的acl分三类

- **标准**  仅源ip地址进行过滤 1-99  1300-1999    简单访问控制
- **扩展**  源/目的ip，端口，协议 100-999  2000-2699  精细化访问控制
- **命名**  源/目的ip，端口，协议，自定义名称  可读性更好，可以说简单和精细化

现在最常用的是命名acl

```ini
# 标准acl
access-list 10 permit/deny  192.168.10.0  0.0.0.255

#应用到指定端口
interface gigabitEthernet 0/0
ip access-group 10 in   --这个in

# 命名acl
ip access-list extend server-access
permit tcp 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 eq 80
permit tcp any any established --允许已建立连接的返回流量   访问不允许，但响应建立连接后可以
deny ip any any log  --所有被拒绝的ip都存在日志里

#应用到指定端口
interface gigabitEthernet 0/0
ip access-group server-access out  
```

### 端口ACL的in/out

- **<span style="color:#e74c3c">in</span>**:对接口而言的内部，即从设备发往外面的方向
- **<span style="color:#e74c3c">in</span>**:对接口而言的外部，即从外面发往设备的方向

## **<span style="color:#3498db">DHCP</span>**

**DHCP：**即自动分发ip的服务。需要一个网关来绑定网段。之后再设置地址池，排除地址等

```ini
interface vlan 10 
ip address 192.168.10.1 255.255.255.0
no shutdown

#三层交换机开启路由功能
ip routing

#排除地址
ip dhcp excluded-address 192.168.10.1 192.168.10.10

#建立地址池
ip dhcp pool vlan10
network 192.168.10.0 255.255.255.0
default router 192.168.10.1
dns-server 223.5.5.5
exit

#开启dhcp服务
service dhcp
```

## **<span style="color:#3498db">NAT</span>**

**<span style="color:#27ae60">一般内网可以有成百上千的内网地址，而外网地址往往只有屈指可数的几个，那么外网的数据包应该怎么知道该把数据传给内网的哪个IP呢？这里就需要用到NAT了</span>**

**一个主机一般有1-65535个可用端口，那么只要将端口和对应的ip映射起来，出去的的时候源地址是外网ip加对应的端口，如2.235.1.24:12345，那么外网回来的数据包目的地址也只要指向2.235.1.24:12345就可以了**

**内网ip出外网的SNAT映射，一般都是动态的：看主机有哪些空闲的非知名端口(1024-65535)，就随机映射一个端口。而如果是内网暴露给外网的服务，则一般用静态地址映射，避免外网访问业务不能有个固定端口**

```ini
#内网主机出网动态映射
#指定所有内网网段并成一条ACL规则
ip access-list extended NAT_ALLOW
permit ip 192.168.10.0 0.0.0.255 any
permit ip 192.168.20.0 0.0.0.255 any
permit ip 192.168.30.0 0.0.0.255 any
exit

#映射为外网口的ip（外网口 interface GigabitEthernet0/0）
ip nat inside source list NAT_ALLOW interface GigabitEthernet0/0 overload

#内网服务静态映射到外网
ip nat inside source static tcp 192.168.1.20 3389 202.1.1.10 3389 extendable

#如果ip是动态获取的（PPPoE），可以把公网 IP 替换成`interface 外网接口名称`
ip nat inside source static tcp 192.168.1.10 80 interface GigabitEthernet0/1 8080 extendable

```
