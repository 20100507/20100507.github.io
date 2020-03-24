---
layout: post
title: "Hbase降低版本&集成Ranger"
date: 2018-08-15   
tag: 大数据Ranger
---

### 前言
    
	为了适应ranger1.2对应的hbase1.3.1版本，只能降低hbase1.4.2版本。根本原因在于hbase1.4.2 的协处理器和1.3.2 协处理器的差别很大。ranger无法兼容hbase，修改ranger源代码
	的工作量比较大，因此离线降低hbase版本和Phoenix版本，同时需要整合ranger1.2和降低后的hbase1.3.2版本。

### 操作

**拷贝 HDFS 上/hbase目录下的数据备份hbase 用于回滚**

> `hdfs dfs -cp /hbase /hbase-back`

**停止hbase**
  
> `ansible-playbook -i bit-bdp-production playbooks/hbase.yml --tags stop --become-method=sudo`
 
**重新安装hbase1.3**

***调整hbase ansible脚本 以及hbase 对应的Phoenix版本，执行如下命令重新安装hbase***

> 1. 拷贝Hbase安装包        `ansible-playbook -i bit-bdp-production playbooks/hbase.yml --tags instsall_hbase --become-method=sudo`
> 2. 初始化相关目录         `ansible-playbook -i bit-bdp-production playbooks/hbase.yml --tags init_directory --become-method=sudo`
> 3. 更新hbase环境变量      `ansible-playbook -i bit-bdp-production playbooks/hbase.yml --tags hbase_env --become-method=sudo`
> 4. 更新hbase配置          `ansible-playbook -i bit-bdp-production playbooks/hbase.yml --tags update_config --become-method=sudo`
> 5. 将hbase修改为系统服务  `ansible-playbook -i bit-bdp-production playbooks/hbase.yml --tags services --become-method=sudo`
> 6. 安装phoenix            `ansible-playbook -i bit-bdp-production playbooks/hbase.yml --tags install_phoenix --become-method=sudo`

**删除zookeeper数据和HDFS数据**

> 1. `hdfs dfs -rmr /hbase`
> 2. `zkCli.sh  rmr /hbase`

**重新启动hbase**

> 1. `ansible-playbook -i bit-bdp-production playbooks/hbase.yml --tags restart --become-method=sudo`

**观察hbase-ui正常启动**

<div align="left">
<img src="/images/posts/hbase-ranger/hbase.png" height="540" width="1140" />  
</div>

**重新拷贝备份后的hbase的目录表数据到新的hbase的目录**

***注意不可以拷贝系统表的数据目录***

> 1. `hdfs dfs -cp /hbase-back/data/`对应的namespace空间/对应的表
> 2. `hdfs dfs -cp /hbase-back/data/default/`包含Phoenix的元数据表【以及自己的创建的没有namespace的表】

**重新部署Phoenix客户端**

> 1. `ansible-playbook -i bit-bdp-production playbooks/phoenix.yml  --become-method=sudo`

**执行如下命令从hdfs读取表的结构信息，根据regioninfo生成meta表，分配region到regionserver上**

> 1. `hbase hbck -fixMeta`
> 2. `hbase hbck -fixAssignments`

**安装协处理器**

地图的相关表需要协处理器支持，必须拷贝协处理器相关的jar到hbase的classpath路径下：否则地图关联的表一直处于RIT状态

> 1. ansible-playbook -i bit-bdp-production playbooks/hbase.yml --tags install_geomesa --become-method=sudo

重新启动hbase 不再出现RIT并且所有的分区没有离线的分区和切分的分区已经其他分区同时在命令行创建hbase表成功，插入数据，删除数据，disable表，
删除表都正常说明。证明 hbase已经恢复。

如果在执行`hbase hbck -fixAssignments`命令行出现了人为中断，或者regionserver 宕机下线，命令执行失败一直挂起，此时如果手动kill掉正在运行的
`hbase hbck -fixAssignments`命令。会导致 hbase的自动 rebalance，split, merge 一直处于关闭状态,即使hbase不再有`RIT`了，必须手动执行如下命令
才会开启hbase自动平衡和自动切分自动合并功能：

> 1. `balance_switch true` //开启自动配合
> 2. `splitormerge_switch 'SPLIT', true` //开启自动切分
> 3. `splitormerge_switch 'MERGE', true` //开启自动合并

**执行Phoenix查询表**
> 1. `python2.7 /opt/phoenix/bin/sqlline.py 10.10.21.14:2181`

<div align="left">
<img src="/images/posts/hbase-ranger/phoenix.png" height="540" width="1140" />  
</div>
<div align="left">
<img src="/images/posts/hbase-ranger/phoenix-select.png" height="540" width="1140" />  
</div>

**整合ranger&hbase 执行如下脚本  [原理协处理器拦截请求]**

> 1. `ansible-playbook -i bit-bdp-production playbooks/hbase.yml --tags install_ranger --become-method=sudo`

重启hbase观察web-ui,看到协处理器

<div align="left">
<img src="/images/posts/hbase-ranger/hbase-coprocessor.png" height="50" width="1140" />  
</div>

**登录ranger配置权限**

<div align="left">
<img src="/images/posts/hbase-ranger/ranger.png" height="430" width="1140" />  
</div>

### 后记

    采用社区版方案部署大数据集群最大的障碍是版本问题。从hbase1.3到1.4 小版本的跨越 hbase存储数据的结构没有发生变化，小版本降低对于hbase数据安全影
    响不大，但是停机降低hbase代价比较大，后期需要调研零停机降低hbase&phoenix版本。