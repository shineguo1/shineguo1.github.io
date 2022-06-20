---
layout: post
title: flink环境搭建
date: 2022-06-17
tags: 计算机基础
---
### 一、官网安装

[flink官网](https://nightlies.apache.org/flink/flink-docs-release-1.12/zh/try-flink/local_installation.html) 
1. 下载flink.tgz
2. 解压
3. 启动本地集群
```
$ ./bin/start-cluster.sh
```
4. 报错:
```
[ERROR] The execution result is empty. 
[ERROR] Could not get JVM parameters and dynamic configurations properly. 
[ERROR] Raw output from BashJavaUtils: /d/Program Files/flink/flink-1.12.7/bin/config.sh: line 485: D:\Program: No such file or directory
```
5. 解决: 在`/conf/flink-conf.yaml`加上自定义jdk配置，如：`env.java.home: "D:\Program Files\Java\jdk1.8.0_211"`.

### 二、IDEA配置
1. 编译发生错误 `Error:(7, 28) object xxxx is not a member of package .... ` <br/>
解决：idea菜单栏找到 idea build  -->  rebuild project
