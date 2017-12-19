### 构建Hadoop集群（伪分布式）
#### 1. 安装Java
Hadoop运行需要先安装Java，只需要解压压缩包并配置好环境变量即可
- 解压，这里选择解压在`/opt`下
    ```
    [root@taxuezju opt]# tar -xvzf jdk-8u152-linux-x64.tar.gz
    ```
- 配置环境变量，这里直接配置系统的全局环境变量
    ```
    [root@taxuezju opt]# vim /etc/profile
    ```
    在文件末尾加上以下内容
    ``` bash
    # Java environment
    export JAVA_HOME=/opt/jdk1.8.0_152
    export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH
    export CLASSPATH=$CLASSPATH:.:$JAVA_HOME/lib:$JAVA_HOME/jre/lib
    ```
    注意`JAVA_HOME`的值根据自己主机上实际Java目录的路径变更，之后执行`source /etc/profile`使配置生效
- 验证Java环境变量配置是否正确生效
    ```
    [root@taxuezju jdk1.8.0_152]# java -version
    java version "1.8.0_152"
    Java(TM) SE Runtime Environment (build 1.8.0_152-b16)
    Java HotSpot(TM) 64-Bit Server VM (build 25.152-b16, mixed mode)
    ```
#### 2. 添加Hadoop用户组并添加新用户
这一步非必需，但是考虑安全性，并且为了区分各Hadoop进程以及同一台机器上的其他服务，最好添加独立的用户组和用户
- 添加hadoop用户组
    ```
    [root@taxuezju jdk1.8.0_152]# groupadd hadoop
    [root@taxuezju jdk1.8.0_152]# tail -n1 /etc/group
    hadoop:x:1000:
    ```
    系统给hadoop用户组默认分配的GID是1000
- 添加hadoop用户，该用户属于hadoop用户组
    ```
    [root@taxuezju jdk1.8.0_152]# useradd hadoop -g 1000
    [root@taxuezju jdk1.8.0_152]# tail -n1 /etc/passwd
    hadoop:x:1000:1000::/home/hadoop:/bin/bash
    ```
- HDFS，MapReduce，YARN服务通常作为独立的用户运行，添加hdfs，mapred和yarn用户，它们同属于同一hadoop组
    ```
    [root@taxuezju jdk1.8.0_152]# useradd -g hadoop -M hdfs
    [root@taxuezju jdk1.8.0_152]# useradd -g hadoop -M mapred
    [root@taxuezju jdk1.8.0_152]# useradd -g hadoop -M yarn
    [root@taxuezju jdk1.8.0_152]# tail -n3 /etc/passwd
    hdfs:x:1001:1000::/home/hdfs:/bin/bash
    mapred:x:1002:1000::/home/mapred:/bin/bash
    yarn:x:1003:1000::/home/yarn:/bin/bash
    ```
    这里用参数`-M`不创建对应用户的家目录
#### 3. SSH配置
Hadoop控制脚本依赖于SSH来执行针对整个集群的操作。为了允许来自集群内机器的hdfs用户和yarn用户能够无需密码即可登陆，最简单的方法是创建一个公钥/私钥对，让整个集群共享该密钥对。
键入以下指令来产生一个RSA密钥对。需要做两次，一次以hdfs用户身份，一次以yarn用户身份，这里先配置伪分布式，所以直接基于空口令生成一个新SSH密钥
```
[root@taxuezju ~]# ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
[root@taxuezju ~]# cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```
用以下命令测试是否能够连接：`ssh localhost`，成功则无需键入密码
#### 4. 安装Hadoop
在Apache Hadoop的发布页面下载hadoop二进制包，然后在某一本地目录解压缩，这里选择`/opt`
- 解压二进制包
    ```
    [root@taxuezju opt]# tar xzf hadoop-2.9.0.tar.gz
    ```
- 将Hadoop文件的拥有者改为hadoop用户和组
    ```
    [root@taxuezju opt]# chown -R hadoop:hadoop hadoop-2.9.0
    ```
- `/etc/profile` 中添加一下环境变量，便于直接在终端执行hadoop的一些脚本
    ``` bash
    #Hadoop environment
    export HADOOP_HOME=/opt/hadoop-2.9.0
    export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
    ```
#### 5. 配置Hadoop
这里先配置伪分布式，配置参照《Hadoop: The Definitive Guide》
- 配置`core-site.xml`
    ``` xml
    <configuration>
      <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost/</value>
      </property>
    </configuration>
    ```
- 配置`hdfs-site.xml`
    ``` xml
    <configuration>
      <property>
        <name>dfs.replication</name>
        <value>1</value>
      </property>
    </configuration>
    ```
- 配置`mapred-site.xml`
    ``` xml
    <configuration>
      <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
      </property>
    </configuration>
    ```
- 配置`yarn-site.xml`
    ``` xml
    <configuration>
      <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>localhost</value>
      </property>
      <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
      </property>
    </configuration>
    ```
- 在`hadoop-env.sh`中配置`JAVA_HOME`环境变量
#### 6. 运行Hadoop
- 首次使用Hadoop前。必须格式化文件系统，注意以下命令只有初次使用Hadoop时执行一次，以后不需要
    ```
    hdfs namenode -format
    ```
- 启动守护进程
    ```
    start-dfs.sh
    start-yarn.sh
    mr-jobhistory-daemon.sh start historyserver
    ```
- 检查守护进程是否正在运行
    执行`jps`即可
    ```
    [root@taxuezju hadoop]# jps
    20320 DataNode
    21121 Jps
    20740 NodeManager
    20196 NameNode
    20629 ResourceManager
    21049 JobHistoryServer
    20478 SecondaryNameNode
    ```
- 中止守护进程
    ```
    mr-jobhistory-daemon.sh stop historyserver
    stop-yarn.sh
    stop-dfs.sh
    ```
