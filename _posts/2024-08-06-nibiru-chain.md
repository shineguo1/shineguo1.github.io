---
layout: post
title: nibiru部署链笔记
date: 2024-8-6
tags: 区块链
---

## evm-localnet
- 参考[官方文档 - 快速启动](https://nibiru.fi/docs/dev/evm/quickstart.html)

### 踩坑
1.  go需要安装更高的版本，不然标准库会缺少maps和slices库
```bash
## 截至2024-08-06官方文档叙述要安装1.18.2版本。我更新成1.21.12版本后项目编译成功
wget https://golang.org/dl/go1.18.2.linux-amd64.tar.gz
```
2. 按官方文档操作默认只允许本地访问。跨域访问需要额外修改配置文件。
```bash
## just localnet命令会在 /root/.nibid/config/ 文件夹内生成一系列文件。此命令启动不会持久化数据，每次启动都会初始化配置文件和创世文件，区块从0开始。
cd /root/.nibid/config/
## config.toml是非evm节点配置。 app.toml是evm客户端配置。
## 需要把 (http)127.0.0.1:8545, (ws)127.0.0.1:8546 的host改成 `0.0.0.0`
nano app.toml
## nibid是区块链节点启动命令。中止进程后，再启动是从上次停止的区块延续。
## 后台启动, 日志存在./logs/localnet.log
cd ~/nibiru
nohup nibid start > ./logs/localnet.log 2>&1 &
```