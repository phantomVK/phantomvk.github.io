---
layout:     post
title:      "Hadoop伪分布式搭建教程"
date:       2016-10-09
author:     "phantomVK"
header-img: "img/main_img.jpg"
catalog:    true
tags:
    - Hadoop
    - Java
---

## 注意

__运行环境：`Ubuntu 16.04 LTS AMD64(64Bit)` 、`Oracle JDK8`、`Hadoop-1.2.1`、`SSH`__


## 一、JAVA安装及配置

#### 1.1 安装Java

如果已经安装自行跳过本节。安装JDK的时候可能会报错误，解决方法参考最后一章

本站 __[Ubuntu安装Oracle JDK8教程](https://phantomvk.github.io/2016/11/23/Ubuntu_Oracle_JDK)__

## 二、 Hadoop安装及配置

### 2.1 下载Hadoop 1.2.1

先用`wget`下载Hadoop包，并把安装包解压到`/opt`。

下面用了清华大学镜像源：https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common

```bash
$ cd /opt
$ wget https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-1.2.1/hadoop-1.2.1.tar.gz
$ tar -zxvf hadoop-1.2.1.tar.gz
```


### 2.2 配置Hadoop参数

#### 2.2.0配置文件所在路径

```bash
$ cd /opt/hadoop-1.2.1/conf
```

#### 2.2.1配置hadoop-env.sh

在Hadoop中添加`JDK`路径配置信息，用你自己`JDK`的路径。

```bash
$ vim hadoop-env.sh  
 export JAVA_HOME=/usr/lib/jvm/java-9-openjdk-amd64 
```

#### 2.2.2 配置core-site.xml

```bash
$ vim core-site.xml
```

填写以下配置信息

```xml
<configuration>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/hadoop</value>
    </property>
    <property>
        <name>dfs.name.dir</name>
        <value>/hadoop/name</value>
    </property>
    <property>
        <name>fs.default.name</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

#### 2.2.3 配置hdfs-site.xml

```bash
$ vim hdfs-site.xml
```

填写以下配置信息

```xml
<configuration>
    <property>
        <name>dfs.data.dir</name>
        <value>/hadoop/data</value>
    </property>
</configuration>
```

#### 2.2.4 配置mapred-site.xml

```bash
$ vim mapred-site.xml
```

填写以下配置信息

```xml
<configuration>
    <property>
        <name>mapred.job.tracker</name>
        <value>localhost:9001</value>
    </property>
</configuration>
```

#### 2.2.5 配置/etc/profile

```bash
$ vim /etc/profile
 export HADOOP_HOME=/opt/hadoop-1.2.1   
 #还要在PATH变量中添加 $HADOOP_HOME/bin:
$ source /etc/profile
```

### 2.3 初始化namenode

运行命令后会自动开始格式化，看见NameNode被自动关闭就成功了。

```bash
$ hadoop namenode -format 
************************************************************
STARTUP_MSG: Starting NameNode
STARTUP_MSG:   host = mike-virtual-machine/127.0.1.1
STARTUP_MSG:   args = [-format]
STARTUP_MSG:   version = 1.2.1
STARTUP_MSG:   build = https://svn.apache.org/repos/asf/hadoop/common/branches/branch-1.2 -r 1503152; compiled by 'mattf' on Mon Jul 22 15:23:09 PDT 2013
STARTUP_MSG:   java = 9-internal
************************************************************/
16/06/18 11:36:16 INFO util.GSet: Computing capacity for map BlocksMap
16/06/18 11:36:16 INFO util.GSet: VM type       = 64-bit
16/06/18 11:36:16 INFO util.GSet: 2.0% max memory = 1048576000
16/06/18 11:36:16 INFO util.GSet: capacity      = 2^21 = 2097152 entries
16/06/18 11:36:16 INFO util.GSet: recommended=2097152, actual=2097152
16/06/18 11:36:17 INFO namenode.FSNamesystem: fsOwner=root
16/06/18 11:36:17 INFO namenode.FSNamesystem: supergroup=supergroup
16/06/18 11:36:17 INFO namenode.FSNamesystem: isPermissionEnabled=true
16/06/18 11:36:17 INFO namenode.FSNamesystem: dfs.block.invalidate.limit=100
16/06/18 11:36:17 INFO namenode.FSNamesystem: isAccessTokenEnabled=false accessKeyUpdateInterval=0 min(s), accessTokenLifetime=0 min(s)
16/06/18 11:36:17 INFO namenode.FSEditLog: dfs.namenode.edits.toleration.length = 0
16/06/18 11:36:17 INFO namenode.NameNode: Caching file names occuring more than 10 times 
16/06/18 11:36:17 INFO common.Storage: Image file /hadoop/dfs/name/current/fsimage of size 110 bytes saved in 0 seconds.
16/06/18 11:36:17 INFO namenode.FSEditLog: closing edit log: position=4, editlog=/hadoop/dfs/name/current/edits
16/06/18 11:36:17 INFO namenode.FSEditLog: close success: truncate to 4, editlog=/hadoop/dfs/name/current/edits
16/06/18 11:36:17 INFO common.Storage: Storage directory /hadoop/dfs/name has been successfully formatted.
16/06/18 11:36:17 INFO namenode.NameNode: SHUTDOWN_MSG: 
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at mike-virtual-machine/127.0.1.1
************************************************************/
```

## 三、 启动Hadoop服务

#### 3.1 启动并检查

进入Hadoop目录下，使用脚本启动Hadoop

```bash
$ cd /opt/hadoop-1.2.1/bin
$ ./start_all.sh
starting namenode, logging to /opt/hadoop-1.2.1/libexec/../logs/hadoop-root-namenode-mike-virtual-machine.out
localhost: Warning: Permanently added 'localhost' (ECDSA) to the list of known hosts.
root@localhost's password: 
localhost: 
localhost: starting datanode, logging to /opt/hadoop-1.2.1/libexec/../logs/hadoop-root-datanode-mike-virtual-machine.out
localhost: Warning: Permanently added 'localhost' (ECDSA) to the list of known hosts.
root@localhost's password: 
localhost: 
localhost: starting secondarynamenode, logging to /opt/hadoop-1.2.1/libexec/../logs/hadoop-root-secondarynamenode-mike-virtual-machine.out
starting jobtracker, logging to /opt/hadoop-1.2.1/libexec/../logs/hadoop-root-jobtracker-mike-virtual-machine.out
localhost: Warning: Permanently added 'localhost' (ECDSA) to the list of known hosts.
root@localhost's password: 
localhost: 
localhost: starting tasktracker, logging to /opt/hadoop-1.2.1/libexec/../logs/hadoop-root-tasktracker-mike-virtual-machine.out
```

用`jps`看看已经开启的JVM进程，看见如下服务表示Hadoop已经启动

```bash
$ jps
 2531 DataNode
 2524 NameNode
 2678 SecondaryNameNode
 2778 JobTracker
 2940 TaskTracker
 3039 sun.tools.jps.Jps
```

## 四、 疑难解答

#### 4.1 解决/etc/profile失效

在`.bashrc`添加环境变量

```bash
$ cd ~
$ ls -la
$ vim .bashrc
  export HADOOP_HOME_WARN_SUPPRESS=1
  export HADOOP_HOME=/opt/hadoop-1.2.1
  export JAVA_HOME=/usr/lib/jvm/java-9-openjdk-amd64/
  export JER_HOME=$JAVA_HOME/jre
  export CLASSPATH=$JAVA_HOME/lib:$JRE_HOME/lib:$CLASSPATH
  export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$HADOOP_HOME/bin:$PATH
$ source .bashrc
```

#### 4.2 找不到配置文件

报错信息 

```
Error: Config file not found: /usr/lib/jvm/java-9-openjdk-amd64/conf/management/management.properties
```

这是软链接文件不存在，手动创建正确软链接文件。

```bash
$ cd /usr/lib/jvm/java-9-openjdk-amd64
$ touch conf
$ ln -s lib conf
$ ls -la conf
 lrwxrwxrwx 1 root root 3 6月  16 17:24 conf -$ lib
```


#### 4.3 HADOOP_HOME报deprecated

`Warning: $HADOOP_HOME is deprecated` 

在`~/.bash_profile`里增加一个环境变量抑制错误提示:

```
export HADOOP_HOME_WARN_SUPPRESS=1
```

#### 4.4 SSH无法连接

主要是没有安装`SSH`服务

`localhost: ssh: connect to host localhost port 22: Connection refused`

下载`openssh-server`

```bash
$ ssh-agent
$ apt-get install openssh-server
```

配置`ssh_config`

```bash
$ cd /etc/ssh/ssh_config
    StrictHostKeyChecking no     # 修改为no
    UserKnownHostsFile /dev/null # 添加一行
```


配置`ssh - sshd_config`

```
$ cd /etc/ssh/sshd_config
    PermitRootLogin prohibit-password        # 改为 yes
    PasswordAuthentication prohibit-password # 取消注释
```

重启系统后开启`ssh`服务

```
$ reboot        # 重启系统
$ service sshd restart
$ ssh localhost # 检查ssh服务
```

#### 4.5 安装dpkg报错

多个窗口同时使用`apt-get`也会出现这个错误，要排除一下。如果Ubuntu安装`dpkg`出现问题:

```bash
.....
.....
E:Sub-process /usr/bin/dpkg returned an error code(1)

$ cd /var/lib/dpkg
$ mv info infobak
$ mkdir info
$ mv ./info/* ./infobak
$ rm -rf ./info 
$ mv ./infobak ./info
```


