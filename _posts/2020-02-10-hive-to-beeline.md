---
layout: post
title: "禁用HiveCli,执行hive命令默认连接HiveServer2"
date: 2020-01-15
tag: 大数据技术
---

### 前言
    
    目前我们使用Hive的大多数时候采用的HiveCli模式,但是HiveCli模式有一些短板,比如Ranger无法控制权限,所执行的相关脚本无法
    被Atlas捕获记录数据血缘关系,在交互方面HiveCli交互体验效果不是太好等等,Hive社区逐渐地抛弃HiveCli.
    生产环境通常使用诸如Zeus,Hera,Easyscheduler调度中心执行HQL,默认直接执行`hive -e "show databases;"`或者执行文件
    `hive -f show.hive`,由于生产环境已经使用了将近两年的`hive`命令.目前我们打算切换到`beeline`,但是如果一个一个去修改调度中
    中心的脚本文件几乎不太可能.最终我们采用二次修改开发`hive`本身命令的内容,达到了无缝切换`beeline`方式.

### 脚本介绍

 * 首先切换到hive安装目录下,`cd /opt/hive/bin`我们可以看到如下命令,hive命令需要二次开发修改脚本
 
<div align="left">
<img src="/images/posts/hive02/hive01.png" height="140" width="1440" />  
</div>

 * 切换到ext目录,其中beeline.sh需要二次修改的
    
<div align="left">
<img src="/images/posts/hive02/hive02.png" height="140" width="1440" />  
</div>

### 修改脚本内容

 * 修改hive脚本,在hive文件最前面添加判断是否传入参数(此段脚本可有可无),目的禁用命令行交互
 
 ```
   if [ $# -eq 0 ]
   then
   echo "you must input  -f to execute hive file or -e to query hive sql"
   exit -1;
   fi
 ```

  * 修改hive脚本,在hive脚本中判断如果没有输入`--service`情况下采用`beeline`方式连接hive

```
if [ "$SERVICE" = "" ] ; then
  if [ "$HELP" = "_help" ] ; then
    SERVICE="help"
  else
    SERVICE="beeline"
  fi
fi
```

  * 修改beeline.sh文件,添加默认的连接`hiveserver2`服务地址,采用`${USER}`获取当前执行shell的用户,`-p`输入密码,`$@`为其他命令行参数
 
```
 THISSERVICE=beeline
export SERVICE_LIST="${SERVICE_LIST}${THISSERVICE} "

beeline () {
  CLASS=org.apache.hive.beeline.BeeLine;

  # include only the beeline client jar and its dependencies
  beelineJarPath=`ls ${HIVE_LIB}/hive-beeline-*.jar`
  superCsvJarPath=`ls ${HIVE_LIB}/super-csv-*.jar`
  jlineJarPath=`ls ${HIVE_LIB}/jline-*.jar`
  hadoopClasspath=""
  if [[ -n "${HADOOP_CLASSPATH}" ]]
  then
    hadoopClasspath="${HADOOP_CLASSPATH}:"
  fi
  export HADOOP_CLASSPATH="${hadoopClasspath}${HIVE_CONF_DIR}:${beelineJarPath}:${superCsvJarPath}:${jlineJarPath}"
  export HADOOP_CLIENT_OPTS="$HADOOP_CLIENT_OPTS -Dlog4j.configurationFile=beeline-log4j2.properties "

  #exec $HADOOP jar ${beelineJarPath} $CLASS $HIVE_OPTS "$@"
  exec $HADOOP jar ${beelineJarPath} $CLASS $HIVE_OPTS -u "jdbc:hive2://ip:10000" -n${USER} -p123456789 "$@"
}

beeline_help () {
  beeline "--help"
}

```

### 测试

* 测试执行`hive -e "show databases"`命令(原理执行hive脚本的用户执行beeline连接hiveserver2)
 
<div align="left">
<img src="/images/posts/hive02/hive03.png" height="440" width="1440" />  
</div>

<br/>

* 测试执行`hive -f "data.hive"`命令

<div align="left">
<img src="/images/posts/hive02/hive04.png" height="440" width="1440" />  
</div>  

<br/>

* 以上都测试成功,可无缝切换hivecli到hiveserver2.

### 总结

    以上是笔者总结的hivecli到hiveserver2无缝切换,希望对读者起到一定帮助作用.
