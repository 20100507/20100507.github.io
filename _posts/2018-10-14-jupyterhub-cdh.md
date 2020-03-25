---
layout: post
title: "JupyterHub&CDH"
date: 2018-10-14   
tag: 机器学习
---

### 前言
    
	为搭建大数据python集群算法环境,在CDH中安装 python on spark,版本要求CDH5.7,aNAconda-3-6.2.0

### 安装jupyterhub

**执行如下命令**

>1.  bash Anaconda2-2018.12-Linux-x86_64.sh
>2.  pip install jupyterhub notebook -i https://pypi.douban.com/simple/
>3.  //如果没有nodejs需要安装
>4.  npm install -g configurable-http-proxy
>5.  pip install pyspark==2.1.2 -i https://pypi.tuna.tsinghua.edu.cn/simple
>6.  pip install jupyterthemes -i https://pypi.douban.com/simple/
>7.  pip install  jupyter_client -i https://pypi.douban.com/simple/
>8.  pip install ipykernel -i https://pypi.douban.com/simple/
>9.  mkdir /jupyter_notebook
>10. yum -y install nginx
  
**修改环境变量**

`export SPARK_HOME=/opt/cloudera/parcels/SPARK2/lib/spark2`<br/>
`export PYSPARK_SUBMIT_ARGS='--master yarn --deploy-mode client --num-executors 3 --executor-cores 1 --executor-memory 2G  pyspark-shell'`<br/>
`export HADOOP_CONF_DIR=/etc/hadoop/conf.cloudera.yarn/`<br/>
`export YARN_CONF_DIR=/etc/hadoop/conf.cloudera.yarn/`<br/>
`export PYTHONPATH=$SPARK_HOME/python:$SPARK_HOME/python/lib/py4j-0.10.4-src.zip:$SPARK_HOME/python/lib/pyspark.zip:$PYTHONPATH`<br/>
 
**修改配置文件**

> 修改 /etc/jupyterhub/jupyterhub_config.py
> 

<br/>
<br/>

  ```
  c.JupyterHub.ip = '10.10.21.7'
  c.JupyterHub.port = 12443
  c.Spawner.ip = '10.10.21.7'
  c.PAMAuthenticator.encoding = 'utf8'
  #c.Authenticator.whitelist = {'hadoop'}  #默认不能使用root登录，需要修改配置
  c.Authenticator.whitelist = {'dev', 'bianqi'} 
  c.NotebookApp.allow_root = True
  c.LocalAuthenticator.create_system_users = True
  c.Authenticator.admin_users = {'hadoop'}
  #c.JupyterHub.authenticator_class = 'dummyauthenticator.DummyAuthenticator'
  #c.DummyAuthenticator.password = "bitnei_123"
  c.NotebookApp.contents_manager_class = "jupytext.TextFileContentsManager"
  c.JupyterHub.statsd_prefix = 'jupyterhub'
  #c.NotebookApp.notebook_dir = '/volume1/study/python/' #jupyter 自定义目录使用
  c.Spawner.notebook_dir = '/jupyter_notebook' #jupyterhub自定义目录
  c.JupyterHub.statsd_prefix = 'jupyterhub'
  ```

> 修改nginx配置文件
> 

<br/>
<br/>

```
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user root;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;
    server {
        listen       80;
	server_name  labs.bitnei.cn;
	rewrite ^(.*) https://$server_name$1 permanent; #自动跳转到https
    }

    server {
        listen       443 ssl http2;
        listen       [::]:443 ssl http2;
        server_name labs.bitnei.cn;
        ssl_certificate "/etc/nginx/bitnei_cn.pem";
        ssl_certificate_key "/etc/nginx/bitnei_cn.key";
        add_header Strict-Transport-Security "max-age=15768000; includeSubdomains; preload" always;
        ssl_session_timeout 5m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #按照这个协议配置
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;#按照这个套件配置
        ssl_prefer_server_ciphers on;
        location / {
	    proxy_pass http://10.10.21.7:12443; #转发到本机项目端口
            proxy_set_header Host $host;
            proxy_set_header X-Real-Scheme $scheme;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            # WebSocket support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_read_timeout 120s;
            proxy_next_upstream error; 
        }
    }
}
```

### 启动程序


**启动nginx**

> systemctl start nginx

**启动jupyterhub**

> nohup /opt/anaconda/bin/python /opt/anaconda/bin/jupyterhub --config=/etc/jupyterhub/jupyterhub_config.py --no-ssl > /dev/null 2>&1 &

### 访问

**访问localhost如下图**

<div align="left">
<img src="/images/posts/jupyterhub/screen.png" height="540" width="1180" />
</div>

**书写代码注意事项**

>  必须加入如下环境变量配置
`export HADOOP_CONF_DIR=/etc/hadoop/conf.cloudera.yarn/`
`export YARN_CONF_DIR=/etc/hadoop/conf.cloudera.yarn/` 

### 附录

**切换主题**

>  pip install jupyterthemes
>  jt -t monokai -T -N -altp -fs 13 -nfs 13 -tfs 13 -ofs 13

### 总结
  
  裸机跑jupyterhub有宕机的风险,考虑要修改为jupyterhub on kubernetes方式,代替裸机资源利用率更高,安全性更强.

