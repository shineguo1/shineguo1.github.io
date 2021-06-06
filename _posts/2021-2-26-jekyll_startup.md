---
layout: post
title: jekyll环境搭建（windows）
date: 2021-02-26
tags: jekyll
---

### 介绍

 　 [Jekyll](http://jekyllcn.com/) —— 将纯文本转换为静态博客网站。

### Jekyll 环境配置
1.官网下载安装 Ruby+Devkit 2.7.2-1 (x64) ，安装路径建议英文

2.命令行安装 jekyll

```     
$ gem install jekyll     
```

3.命令行安装bundle

```
$ bundle install
```
3.1 如果出现 【Could not locate Gemfile】-，使用下面命令
```
$ bundle init


```

4.命令行更新bundle :

```
$ bundle update --bundler

```

5.进入项目根目录，启动本地服务

```
$ bundle exec jekyll serve
```

本地调试地址： [http://localhost:4000](http://localhost:4000)。

![](/images/posts/jekyll/image1.png)


### 目录
　
[官网文档](http://jekyll.com.cn/docs/structure/)







