---
layout: post
title: "Redis 数据导入和导出"
date: 2019-04-22 
tag: WEB中间件
---

### 前言
    
	   前一段时间部门的Redis数据需要迁移,由于数据规模不是很大.

### 安装rvm

1.安装rvm

> 1. gpg2 --keyserver hkp://keys.gnupg.net --recv-keys D39DC0E3
> 2. curl -L get.rvm.io \| bash -s stable
> 3. find / -name rvm -print
> 4. source /usr/local/rvm/scripts/rvm
> 5. rvm install 2.3.3
> 6. rvm use 2.3.3 --default

2.安装redis-dump

> 1. gem install redis-dump -V
> 2. redis-dump -u :mypassword@localhost:6379 -d 1 >test.json
> 3. cat 6380.json \| redis-load -u IP6:6380 -d 0

### 总结

	以上记录了笔者导出和导入Redis数据的过程,简单明了.希望对读者起到帮助作用.