---
layout: post
title: 0.0.0.0、localhost、127.0.0.1 区别
date: 2022-06-16
tags: 计算机-1分钟知识
---

#### 0.0.0.0
- 表示本机所有ip地址，比如tomcat配置了0.0.0.0:8080, 所有本机ip地址包括192.168.1.3, 127.0.0.1, 那么192.168.1.3:8080, 127.0.0.1:8080都能访问。内外网ip地址都能访问。
- 在路由表里，它表示的是这样一个集合：所有未知的主机和目的网络。这里的“未知”是指在本机的路由表里没有特定条目指明如何到达。

#### localhost
- 本地的一个域名，一般指向127.0.0.1，可以在配置文件里修改。

#### 127.0.0.1
- 回环地址。本地的虚拟接口，所以对本机来说不会宕掉（本机宕了就没办法了）。同理，本机以外无法访问。
