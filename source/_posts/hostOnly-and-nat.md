---
title: VirtualBox 虚拟机的 HostOnly 与 NAT 模式的原理
date: 2019-11-28 15:04:49
tags: 网络协议
---

Homestead 是一个 Ubuntu 虚拟机，由 `Laravel` 官方出品，内置了开箱及用的开发环境。因近期在研究网络协议，所以想看看 `Homestead` 是如何上网的，多年前因搞不懂
`Host only` 和 `桥联` 和 `NAT` 折腾了很久，才搞定虚拟机上网的问题，当然纯靠 Google 的，至于原理是不懂的。趁着网络协议这股劲来研究下。

#### 先上 Homestead 的网卡信息
- eth0:  用来上网的网卡
- eth1:  用来与主机交互的网卡

![](https://static.tt12t.com/blog/homestead_adapter.png)

#### 再上 windows 的网卡

VirtualBox Host-Only 的IP是： 192.168.10.1

![](https://static.tt12t.com/blog/windows_adapter.png)

#### 他们的工作关系图
![](https://static.tt12t.com/host-only-nat-adapter.png)