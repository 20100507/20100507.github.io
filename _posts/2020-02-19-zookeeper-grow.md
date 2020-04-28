---
layout: post
title: "Zookeeper不丢失Quorum扩容"
date: 2020-02-19
tag: 大数据技术
---

### 前言
    
	   笔者平台搭建时,Zookeeper仅仅部署了三个节点,三个节点对于大型集群是比较少的.笔者对zk作了扩容.从3个节点扩容到了5个节点
	同时保证在不丢失已有的Quorum情况下,下面是笔者的操作的流程
	   
		  
### 部署3台zk集群

   ```
   mkdir ~/zook
cd ~/zook
wget http://apache.claz.org/zookeeper/zookeeper-3.4.5/zookeeper-3.4.5.tar.gz # You may wish to choose a closer mirror
tar xzf zookeeper-3.4.5.tar.gz
for i in `seq 5` ; do mkdir conf$i ; echo $i > conf$i/myid ; done

tickTime=2000
dataDir=/home/vagrant/zook/conf1
clientPort=2181
initLimit=50
syncLimit=200
server.1=localhost:2881:3881
server.2=localhost:2882:3882
server.3=localhost:2883:3883

zookeeper-3.4.5/bin/zkServer.sh start-foreground conf1/zoo.cfg


tickTime=2000
dataDir=/home/vagrant/zook/conf2
clientPort=2182
initLimit=50
syncLimit=200
server.1=localhost:2881:3881
server.2=localhost:2882:3882
server.3=localhost:2883:3883

zookeeper-3.4.5/bin/zkServer.sh start-foreground conf2/zoo.cfg


tickTime=2000
dataDir=/home/vagrant/zook/conf3
clientPort=2183
initLimit=50
syncLimit=200
server.1=localhost:2881:3881
server.2=localhost:2882:3882
server.3=localhost:2883:3883

zookeeper-3.4.5/bin/zkServer.sh start-foreground conf3/zoo.cfg
   ```

### 加入两个新的节点

* 修改配置&启动

```
tickTime=2000
dataDir=/home/vagrant/zook/conf4
clientPort=2184
initLimit=50
syncLimit=200
server.1=localhost:2881:3881
server.2=localhost:2882:3882
server.3=localhost:2883:3883
server.4=localhost:2884:3884
server.5=localhost:2885:3885
zookeeper-3.4.5/bin/zkServer.sh start-foreground conf4/zoo.cfg
tickTime=2000
dataDir=/home/vagrant/zook/conf5
clientPort=2185
initLimit=50
syncLimit=200
server.1=localhost:2881:3881
server.2=localhost:2882:3882
server.3=localhost:2883:3883
server.4=localhost:2884:3884
server.5=localhost:2885:3885
zookeeper-3.4.5/bin/zkServer.sh start-foreground conf5/zoo.cfg
```

启动服务后,查看节点是否加入到集群中,snapshot数据是否被激活.然后执行如下步骤


### 修改旧的节点

```
for i in `seq 3` ;do vim conf$i/zoo.cfg ; done
server.4=localhost:2884:3884
server.5=localhost:2885:3885
```

### 重启就节点的follower节点

```
Ctrl+C 停止,然后执行如下命令再运行
zookeeper-3.4.5/bin/zkServer.sh start-foreground conf2/zoo.cfg
```

* 确保上述重启的节点已经加入到集群中然后重启另外一个节点

```
# Ctrl+C
zookeeper-3.4.5/bin/zkServer.sh start-foreground conf3/zoo.cfg
```

### 重启旧节点的leader节点

```
# Ctrl+C
zookeeper-3.4.5/bin/zkServer.sh start-foreground conf1/zoo.cfg

```


### 注意事项

	笔者在zookeeper3.4.x做了测试,其他尚未测试

### 总结

	以上记录Zookeeper扩容过程.希望本文对读者起到帮助作用.
