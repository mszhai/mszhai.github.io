---
layout: post
title: hadoop完全分布式模式搭建和hive安装
---

# hadoop完全分布式模式搭建和hive安装

## 简介

Hadoop是用来处理大数据集合的分布式存储计算基础架构。可以使用一种简单的编程模式，通过多台计算机构成的集群，分布式处理大数据集。hadoop作为底层，其生态环境很丰富。

hadoop基础包括以下四个基本模块：
* hadoop基础功能库：支持其他hadoop模块的通用程序包。
* HDFS: 一个分布式文件系统，能够以高吞吐量访问应用的数据。
* YARN: 一个作业调度和资源管理框架。
* MapReduce: 一个基于YARN的大数据并行处理程序。

当然，hadoop相关的项目很多，包括HBase, Hive, Spark, ZooKeeper等。

## 基础准备

基础准备工作包括：安装环境的准备，ssh配置

### 安装环境的准备

首先需要3台机器（后续也可以扩展），我准备的是虚拟机中的3个ubuntu 16.04.2. 内存1G. 
创建用户：
机器名|用户名|IP|密码| Node
|-|-|-|-|-|
master| hadoop| 10.10.10.3| **| NameNode
slave1| hadoop| 10.10.10.4| **| DataNode
slave2| hadoop| 10.10.10.5| **| DataNode
```
# 更新apt
$ sudo apt-get update

# 安装vim
$ sudo apt-get install vim

# 查看IP
$ ifconfig

# 新建用户hadoop，主目录为/home/hadoop
$ useradd -d /home/hadoop -m hadoop
# 将hadoop用户加到root组
$ usermod -a -G root

# 重命名机器名
$ sudo vim /etc/hostname
```
```
# hostname
master
```

安装java
```
# 检验java是否安装
$ echo $JAVA_HOME
/usr/jdk/jdk1.8.0_121
$ java -version
```

### ssh配置

主要目的是配置ssh免密码，使3台机器可以互相访问。
```
# 检验ssh是否安装完成
$ ssh localhost

hadoop@master:~$ ssh slave1
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.8.0-49-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

257 packages can be updated.
3 updates are security updates.

*** System restart required ***
hadoop@slave1:~$ 
```
* 分别在四台机器上生成密钥对
```
# 进入hadoop用户主目录
$ cd ~
# 生成密钥对
$ ssh-keygen -t rsa
```
详细可以参考其他文章。

## hadoop安装包

hadoop下载地址（2.9.0）：<http://mirrors.hust.edu.cn/apache/hadoop/common/stable2/hadoop-2.9.0.tar.gz>

hive下载地址（0.9.0）：<http://archive.apache.org/dist/hive/hive-0.9.0/hive-0.9.0.tar.gz>
```
# 解压文件到/home/hadoop
$ cd <gzip文件目录>
$ sudo tar -zxf hadoop-2.9.0.tar.gz -C /home/hadoop
```

hadoop可以先下载到windows，配置好后sftp到ubuntu /home/hadoop上。
主要修改如下：
* core-site.xml
* hadoop-env.sh
* hdfs-site.xml
* mapred-env.sh
* mapred-site.xml.template
* masters
* slaves
* yarn-site.xml
* yarn-evn.sh
新建目录(按配置文件内容来)：
* /home/hadoop/hadoop-2.9.0/tmp
* /home/hadoop/hadoop-2.9.0/tmp/dfs/name
* /home/hadoop/hadoop-2.9.0/tmp/dfs/data

然后，上传到master. 再从主目录复制到slave1, slave2上：
```
# 如下是复制命令，也可以打包后传送
$ scp -r /home/hadoop/hadoop-2.9.0/ slave1:/home/hadoop
$ scp -r /home/hadoop/hadoop-2.9.0/ slave2:/home/hadoop
```

## hadoop安装后配置

### 环境变量设置

在/etc/profile中添加环境变量,需要在所有机器上执行
```
$ sudo vim /etc/profile
# 添加成功后执行
$ source /etc/profile
```
详细内容如下：
```
export JAVA_HOME=/usr/jdk/jdk1.8.0_121
export CLASSPATH=.:$CLASSPATH:$JAVA_HOME/lib

export HADOOP_HOME=/home/hadoop/hadoop-2.9.0
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_YARN_HOME=$HADOOP_HOME
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop

export PATH=$PATH:$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$HADOOP_HOME/lib

export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib"
export LD_LIBRARY_PATH=$HADOOP_HOME/lib/native
```

### 完成配置

在master上执行
```
# 格式化HDFS
$ hdfs namenode -format

# 格式化成功后，启动HDFS
$ start-dfs.sh
# 启动YARN
$ start-yarn.sh

# 验证
hadoop@master:~$ jps
3472 SecondaryNameNode
3255 NameNode
15307 Jps
```
可以ssh slave1, 用jps查看slave1上的情况。由于前面没有配置YARN, 所以ResourceManager服务没有启动。

另外，可以在浏览器上查看NameNode: <master:50070> 或者 <10.10.10.3:50070>.

### 报错

搭建过程中难免出错，需要多看些文章。但是最重要是，看报错的日志，分析原因。

我搭建中，主要遇到如下。第一个是用户权限的问题，如下的hive也是如此。可以用ls -lh查看权限情况。第二、三两个，主要是环境变量配置的问题，需要按教程一步步做。

* permission denied
* command not found
* jar包未发现

## hive配置

### hive安装

hive软件包下载好后，可以sftp到mater机器上。也可以直接在机器上操作。
```
# 注意需要大O
$ curl -O http://archive.apache.org/dist/hive/hive-0.9.0/hive-0.9.0.tar.gz
$ sudo tar -zxf hive-0.9.0.tar.gz -C /home/hadoop
$ cd /home/hadoop/hive-0.9.0
$ sudo mkdir warehouse
$ sudo chmod a+rwx /home/hadoop/hive-0.9.0/warehouse

# 其他命令
# 将文件夹名改为hadoop
$ sudo mv ./hadoop-2.9.0/ ./hadoop

# 删除文件夹
# -r 就是向下递归，管理有多少级目录，一并删除
# -f 就是直接强行删除，不作任何提示的意思
$ sudo rm -rf [文件夹名]
```

### hive环境变量

只需要配置master即可：
```
# /etc/profile
export HIVE_HOME=/home/hadoop/hive-0.9.0/
export PATH=$PATH:$HIVE_HOME/bin
```

### hive测试

```
$ cd $HIVE_HOME
$ bin/hive
hive> create table x (a int);
OK
Time taken: 6.267 seconds
hive> select * 
    > from x;
OK
Time taken: 0.532 seconds
hive> select * from x;
OK
Time taken: 0.082 seconds
hive> exit;
```

## 参考

[]hadoop构建数据仓库实践，王雪迎，清华大学出版社</br>
[]hive编程指南，E.C., D.W., J.R., 曹坤，人民邮电出版社</br>
[]<http://hadoop.apache.org/docs/r1.0.4/cn/quickstart.html></br>
[]<http://blog.csdn.net/wild46cat/article/details/78939284></br>
[]<http://blog.csdn.net/ab198604/article/details/8250461></br>
[]<http://blog.csdn.net/EdwinBalance/article/details/78640323>