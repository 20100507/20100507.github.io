---
layout: post
title: "Hadoop源码分析-02-Configuration"
date: 2020-01-25
tag: Hadoop源码分析
---

### 前言

	   在Hadoop启动的过程中,如果我们修改了配置后,会加载hdfs-site.xml core-site.xml yarn-site.xml 等配置文件,如果你没有修改配置文件,Hadoop默认会加载hdfs-default.xml文件.相对来说Configuration是比较结构简单,本文笔者分析一下关于Hadoop Configuration(配置类的结构),进一步加深读者对hadoop配置文件加载的理解.


​		  
### 配置文件类介绍

**源码类解释**

<div align="left">
<img src="/images/posts/hadoop-source-02/Hadoop01.png" height="920" width="1180" />
</div>
> 1.  ConfigRedactor  存储修改敏感信息
> 2.  Configurable  可配置的顶层接口
> 3.  Configuration 核心引导配置类
> 4.  ConfigurationWithLogging 继承自Configuration  携带日志的配置类
> 5.  Configured 实现 Configurable 接口 同样为顶层的基类
> 6.  ConfServlet 配置文件的http Servlet配置类 也就是我们通常在web-ui看到的配置
> 7.  Reconfigurable 可重新配置的顶层接口继承Configurable ,对Configurable 扩展
> 8.  ReconfigurableBase 抽象类继承自Configured实现Reconfigurable
> 9.  ReconfigurationServlet  可重新配置文件的http Servlet配置类
> 10.  ReconfigurationTaskStatus 重新配置后的任务状态
> 11.  ReconfigurationUtil 重新配置工具类[包含获取改变的属性和解析改变的属性]



**核心类图结构如下**


<div align="left">
<img src="/images/posts/hadoop-source-02/Hadoop02.png" height="920" width="1180" />
</div>


如上图：顶层接口为Configurable ,Configured继承自Configurable，ReConfigurable继承自Configurable，ReconfigurableBase 继承Configurable  并且实现了ReConfigurable. Configured与Configuration均有关联.



### 核心类内部方法

**核心类方法介绍**



***Configurable***

```
public interface Configurable {
  void setConf(Configuration conf);
  Configuration getConf();
}
```



***Configured***

```
public class Configured implements Configurable {

  private Configuration conf;
  public Configured() {
    this(null);
  }
  public Configured(Configuration conf) {
    setConf(conf);
  }
  @Override
  public void setConf(Configuration conf) {
    this.conf = conf;
  }
  @Override
  public Configuration getConf() {
    return conf;
  }
}
```



​     Configured实现了Configurable,Configuration作为Configured类的成员变量,Configured控制着整个Configuration生命周期,包括设置Configuration该对象,获取该对象.

***Configuration***

   由于`Configuration`方法代码比较多,笔者选其中一个方法作为解析

<div align="left">
<img src="/images/posts/hadoop-source-02/Hadoop03.png" height="920" width="1180" />
</div>


 如上图 Resource作为内部final类，资源可以表示加载hadoop本身已经有默认的配置文件或者用户自定义的配置文件.

 addDefaultResource 方法，如果默认的资源中不包含改名字的键值对,就添加到默认的资源中,然后循环遍历如果

判断是否开启加载默认的值，如果开启则重新刷新配置.

```
  public static synchronized void addDefaultResource(String name) {
    if(!defaultResources.contains(name)) {
      defaultResources.add(name);
      for(Configuration conf : REGISTRY.keySet()) {
        if(conf.loadDefaults) {
          conf.reloadConfiguration();
        }
      }
    }
  }
```

### 总结

	以上为笔者简单分析的`Configuration`配置结构希望读者可以下载源代码继续查看该部分,Hadoop发展十几年内部代码结构逻辑非常复杂.本文只能起到抛砖引玉的作用.
	希望本文对读者起到帮助作用.
