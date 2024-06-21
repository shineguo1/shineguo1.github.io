---
layout: post
title: graphNode部署笔记
date: 2024-1-17
tags: 区块链
---

## 一. 部署

### 安装四个工具
#### rust
sudo apt install curl build-essential gcc make
curl https://sh.rustup.rs -sSf | sh

#### PostgreSQL
https://www.postgresql.org/download/linux/ubuntu/

#### ipfs
https://docs.ipfs.tech/install/command-line/#install-official-binary-distributions ##不能使用sudo snap安装，kubo已经不分发到snap上了

#### protobuf
sudo apt install protobuf-compiler

### 添加docker权限
sudo groupadd docker ##有时会说docker组已存在
sudo gpasswd -a xiao_ming docker //xiao_ming即是当前没有docker指令权限的用户名
newgrp docker  //更新

### 使用docker-compose似乎不用安装上面4个东西
git clone https://github.com/graphprotocol/graph-node.git
cd graph_node/docker
vi docker-compose.yml ##修改evm节点，sql端口、密码等
docker-compose up ##启动

## 二. 维护
1. 看日志
```js
cd Downloads/workspace/graph-node/docker/
docker-compose logs --tail=100
docker-compose logs --tail=100 | grep goerli
docker-compose logs --head=10
```

2. 停止docker服务
```js
docker ps
docker stop 54f58ca8c4c1
```