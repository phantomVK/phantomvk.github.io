---
layout:     post
title:      "Hadoop伪分布式搭建"
date:       2016-10-09
author:     "phantomVK"
catalog:    true
tags:
    - Tools
---

## 一、环境及配置

运行环境：__Ubuntu 16.04 LTS AMD64__、__OpenJDK 9__、__Hadoop-1.2.1__、__OpenSSH__

Hadoop依赖JDK，请确认系统已安装JDK，具体设置请参考文章:  __[Ubuntu安装Oracle JDK8](/2016/11/23/Ubuntu_Install_JDK/)__ 

## 二、 Hadoop安装及配置

### 2.1 下载Hadoop

通过wget在[清华大学镜像源](https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common)下载Hadoop并解压到`/opt`

```bash
$ cd /opt
$ wget https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-1.2.1/hadoop-1.2.1.tar.gz
$ tar -zxvf hadoop-1.2.1.tar.gz
```


### 2.2 配置Hadoop参数

#### 2.2.1 配置文件路径

```bash
$ cd /opt/hadoop-1.2.1/conf
```

#### 2.2.2 配置hadoop-env.sh

在文件中添加安装`JDK`的路径，这里用的是`OpenJDK`

```bash
$ vim hadoop-env.sh  

 export JAVA_HOME=/usr/lib/jvm/java-9-openjdk-amd64 
```

#### 2.2.3 配置core-site.xml

```bash
$ vim core-site.xml
```

配置内容如下：

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

#### 2.2.4 配置hdfs-site.xml

```bash
$ vim hdfs-site.xml
```

配置内容如下：

```xml
<configuration>
    <property>
        <name>dfs.data.dir</name>
        <value>/hadoop/data</value>
    </property>
</configuration>
```

#### 2.2.5 配置mapred-site.xml

```bash
$ vim mapred-site.xml
```

配置内容如下：

```xml
<configuration>
    <property>
        <name>mapred.job.tracker</name>
        <value>localhost:9001</value>
    </property>
</configuration>
```

#### 2.2.6 配置/etc/profile

```bash
$ vim /etc/profile
```

增加以下配置：

```
 export HADOOP_HOME=/opt/hadoop-1.2.1 
```

相同文件`PATH`变量中追加参数`$HADOOP_HOME/bin:`，保存、退出并刷新`profile`

```bash
$ source /etc/profile
```

### 2.3 初始化NameNode

运行命令后会自动开始格式化，最后NameNode自动关闭

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

进入Hadoop目录，通过脚本启动Hadoop

```bash
$ cd /opt/hadoop-1.2.1/bin
$ ./start_all.sh
```

运行结果

```
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

用`jps`查看可知Hadoop已启动

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

#### 4.1 解决 /etc/profile 失效

在`~/.bashrc`文件添加环境变量

```bash
$ cd ~
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

这个错误可能在使用OpenJDK时出现

```
Error: Config file not found: /usr/lib/jvm/java-9-openjdk-amd64/conf/management/management.properties
```

原因是软链接不存在，手动创建正确软链接即可

```bash
$ cd /usr/lib/jvm/java-9-openjdk-amd64
$ touch conf
$ ln -s lib conf
$ ls -la conf
 lrwxrwxrwx 1 root root 3 6月  16 17:24 conf -$ lib
```

#### 4.3 HADOOP_HOME is deprecated

```bash
Warning: $HADOOP_HOME is deprecated
```

在`~/.bash_profile`里增加一个环境变量抑制错误提示

```
export HADOOP_HOME_WARN_SUPPRESS=1
```

#### 4.4 SSH无法连接

没有安装`SSH`服务导致

```
localhost: ssh: connect to host localhost port 22: Connection refused
```

下载`openssh-server`

```bash
$ ssh-agent
$ apt-get install openssh-server
```

配置`ssh_config`

```bash
$ cd /etc/ssh/ssh_config
    StrictHostKeyChecking no     # 修改为no
    UserKnownHostsFile /dev/null # 添加这行
```


配置`sshd_config`

```bash
$ cd /etc/ssh/sshd_config
    PermitRootLogin prohibit-password        # 改为 yes
    PasswordAuthentication prohibit-password # 取消注释
```

重启系统，开启`ssh`服务

```bash
$ service sshd restart
$ ssh localhost # 检查ssh服务
```

#### 4.5 安装dpkg报错

多个窗口同时使用`apt-get`会出现此错误，避免同时使用多个窗口

```bash
.....
.....
E:Sub-process /usr/bin/dpkg returned an error code(1)
```

如果问题无法解决，可以尝试：

```bash
$ cd /var/lib/dpkg
$ mv info infobak
$ mkdir info
$ mv ./info/* ./infobak
$ rm -rf ./info 
$ mv ./infobak ./info
```


