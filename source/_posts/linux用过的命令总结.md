---
title: linux用过的命令总结
date: 2019-12-14 21:54:03
tags: linux
description: 作个记录，方便自己和他人
---

##### ssh-add ，ssh-agent

- github 可以用 ssh 来进行仓库的沟通，这里用非对称加密。服务器上放公钥，本地放私钥。两个钥匙都有一个邮箱用来做标记。
- [ssh-agent](https://wangchujiang.com/linux-command/c/ssh-agent.html) 是密钥管理器，配合其它程序完成认证
- [ssh-add](https://wangchujiang.com/linux-command/c/ssh-add.html) 是个跑腿的, 进行密钥的增删改查
