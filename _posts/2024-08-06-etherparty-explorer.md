---
layout: post
title: 搭建etherparty的简单区块浏览器
date: 2024-8-6
tags: 区块链
---

## 介绍
- `https://github.com/etherparty/explorer/tree/master` 是一个nodejs项目。利用json-rpc接口，把接口查询结果可视化。并没有做合约解析和区块链数据统计。

## 搭建过程
```bash
# 前置：安装npm命令，略过
# 拉取git项目
git clone https://github.com/etherparty/explorer.git
# 进入项目目录
cd explorer
# 安装 node_modules
npm install
## 修改配置 - eth_node_url地址 
nano app.js
## 修改配置 - angular,jquery,bootstrap的css、js资源网址是外网，替换成国内资源，或下载到项目目录使用相对路径引用。 
nano index.html

## 后台运行
sudo nohup npm start > explorer.log 2>&1 &
```
