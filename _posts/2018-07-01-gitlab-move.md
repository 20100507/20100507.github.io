---
layout: post
title: "GitLab迁移"
date: 2018-07-08   
tag: 数据迁移 
---

### 前言
    
	GitLab由于某些原因,公司运维部要求迁移本部门的gitlab所有的代码。
	迁移gitlab涉及到所有的提交记录。

### 拉取

** 执行克隆命令 **

如下为执行命令:

> 1. git clone --bare git@192.168.6.161:bit_data/weather_city.git
> 2. cd weather_city

### 上传

** 执行上传命令 **

如下为执行命令:

> 1. git push --mirror git@192.168.1.20:datagroup/bit_data/weather_city.git
> 2. cd .. 

<br>

转载请注明：[边琪的博客](https://www.bianqi.info) » [点击阅读原文](https://www.bianqi.info/2018/07/gitlab-move/)     
