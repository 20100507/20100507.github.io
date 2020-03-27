---
layout: post
title: "NFS install"
date: 2019-01-20 
tag: NFS
---

### 前言
    
	   由于笔者所在的公司旧服务器和新的服务器磁盘插槽不一样,为了在新的机器上使用旧的机器上的磁盘最终选择采用NFS挂载磁盘.

### 安装

1.目标挂载的服务器上安装nfs-utils(旧服务器)

> 1. `yum -y install nfs-utils`

2.新建需要挂载的目录(新服务器)

> 1. `mkdir /data/volumes -pv`

3.修改如下文件(旧服务器)

> 1. `vim /etc/exports`
> 2. `/data/volumes   10.10.26.0/16(rw,no_root_squash)`
> 3. `exportfs -arv`

4.目标挂载的服务器上启动NFS(旧服务器)

> 1. `service rpcbind restart`
> 2. `service nfs start`

5.挂载网络磁盘(新服务器)

> 1. `mount -t nfs 10.10.26.17:/data/volumes /mnt`

6.检查是否载入磁盘(新服务器)

> 1. `showmount -e`

## 总结

	以上记录了笔者安装NFS过程,简单明了.希望对读者起到帮助作用.