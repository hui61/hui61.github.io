---
title: 基于Docker的Hadoop和Spark环境搭建
date: 2023-08-21T16:47:12+08:00
tags: ["Docker", "大数据", "Hadoop", "Spark"]
series: ["大数据环境搭建"]
featured: true
---

本文主要介绍了如何在docker中快速搭建Hadoop和Spark环境。如果想入门大数据，那么一个Docker环境下的大数据平台是最佳学习方式。

<!--more-->

## 环境

`环境：MacOS Ventura 13.5`

`机型：MacBook Pro (M1, 2021)`

## 准备工作
- [hadoop-3.3.1-aarch64](https://dlcdn.apache.org/hadoop/common/hadoop-3.3.1/hadoop-3.3.1-aarch64.tar.gz)
- [JDK1.8-aarch64](https://gitee.com/Bric666/java/attach_files/803375/download/jdk-8u301-linux-aarch64.tar.gz)
- [scala-2.12.14](https://downloads.lightbend.com/scala/2.12.14/scala-2.12.14.tgz)
- [spark-3.2.1-bin-hadoop3.2](https://dlcdn.apache.org/spark/spark-3.2.1/spark-3.2.1-bin-hadoop3.2.tgz)
- [pyspark-3.4.1](https://files.pythonhosted.org/packages/0c/66/3cf748ba7cd7c6a4a46ffcc8d062f11ddc24b786c5b82936c857dc13b7bd/pyspark-3.4.1.tar.gz)

Move `hadoop-3.3.1-aarch64.tar.gz`、`jdk-8u301-linux-aarch64.tar.gz`、`scala-2.12.14.tgz`、`spark-3.2.1-bin-hadoop3.2.tgz` and `pyspark-3.4.1.tar.gz` to `resources` folder

## 构建

### build Dockerfile
```
docker build -f Dockerfile -t puppets/hadoop:1.1 .
```

### create hadoop network

```
sudo docker network create --driver=bridge hadoop
```

### start container

```
sudo ./start-container.sh
```

**output:**

```
start hadoop-master container...
start hadoop-slave1 container...
start hadoop-slave2 container...
```

### Start

```
docker exec -it hadoop-master bash
./start-hadoop.sh
```
因为yarn配置在hadoop-slave2节点，所以还需要去hadoop-slave2启动
```
docker exec -it hadoop-slave2 bash
./start-hadoop.sh
```

- HDFS UI -> http://localhost:9870/
  {{< figure src="/images/blog/hadoop-spark-docker/hdfs-ui.png">}}
- SPARK UI -> http://localhost:8088/
  {{< figure src="/images/blog/hadoop-spark-docker/spark-ui.png">}}

## 测试
在master节点运行任务
```
./run-wordcount.sh 3.3.1
```

**output**

```
input file1.txt:
Hello Hadoop

input file2.txt:
Hello Docker

wordcount output:
Docker    1
Hadoop    1
Hello    2
```

