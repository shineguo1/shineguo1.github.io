---
layout: post
title: mysql8底层优化：推荐使用count(*)代替count(1)
date: 2022-05-27
tags: 计算机-1分钟知识
---
### 解释（mysql官方文档）

![](/images/rediis-select-count.png){:height="500px" style="margin:initial"}

### 结论
- InnoDB: count(*) = count(1) > count(列名)
- MyISAM: count(*) >= count(1) > count(列名)