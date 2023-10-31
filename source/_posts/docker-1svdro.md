---
title: docker
date: '2023-10-24 18:17:58'
updated: '2023-10-27 17:04:49'
categories: [技术]
tags: ['运维']
comments: true
toc: true
---

# docker

## 批量删除images

```bash
docker rmi $(docker images | grep "aeron" | awk '{print $3}') -f
# -f强制删除防止有重复的
```

## 安装nginx

```bash
docker run \
-p 52468:80 \
--name nginx2 \
-v /home/xxxxx/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
-v /home/xxxxx/nginx/conf/conf.d:/etc/nginx/conf.d \
-v /home/xxxxx/nginx/log:/var/log/nginx \
-v /home/xxxxx/filepath:/usr/share/nginx/html \
-d nginx:latest

# 配置文件在执行之前就需要存在，这样在启动时不会因为找不到配置文件出错
```
<!-- more -->

## 导出oracle最新的镜像

### 制作镜像

```bash
docker commit -a "yuliang" -m "oraclebus" 73f39306b50c yul/oracle:v2

#-a:指定的是镜像作者
#-m:镜像的注释
#73f39306b50c  原容器ID
#yul/oracle:v2 新容器的名称:版本
```

### 导出/导入镜像

```bash
# 导出并压缩
docker save  yul/oracle:v2 |gzip > yuloracle_v2.tar.gz
# 解压导入
gunzip -c yuloracle_v2.tar.gz | docker load
```

### 执行启动

```bash
sqlplus / as sysdba
startup
```

### 监听

```bash
lsnrctl status
lsnrctl start
```

## Dockerfile启动springboot

```shell
FROM java:8
MAINTAINER yl
# RUN useradd -ms /bin/bash algo
# USER algo
RUN mkdir webapp
RUN mkdir dist
RUN mkdir evaluate
RUN mkdir quote
RUN mkdir quote_bak
RUN mkdir balance
RUN mkdir rocketmq
WORKDIR /root/
COPY webapp/  webapp/
COPY dist/  dist/
ADD rocketmq-a.tar.gz /root/
COPY broker.conf  /root/rocketmq-a/conf/broker.conf
COPY entrypoint.sh entrypoint.sh 
RUN chmod +x /root/entrypoint.sh
ENTRYPOINT ["./entrypoint.sh"]
```

### FROM

引用基础镜像

```shell
FROM <image>
FROM <image>:<tag>
FROM <image>:<digest> 
三种写法，其中<tag>和<digest> 是可选项，如果没有选择，那么默认值为latest
```

### MAINTAINER

指定作者

```shell
MAINTAINER yl
```

### ADD

把文件复制到镜像中，相当于scp，不过不需要密码和ip
