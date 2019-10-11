## Hadoop + Hbase + Zookeeper 安装记录

### 部署拓扑架构
```reStructuredText
采用完全分布式部署架构，共三个节点,系统为 centos
1. hadoop1 172.16.20.240  NameNode
2. hadoop2 172.16.20.241  DataNode
3. hadoop3 172.16.20.242  DataNode
```
### 准备环境

> 分别在各节点主机上执行,下面以命名节点 hadoop1 为例，其他节点照葫芦画瓢

1. 设置主机名称

   ```
   vim /etc/sysconfig/network
   HOSTNAME=hadoop1
   ```

2. 修改hosts文件

   ```
   vim /etc/hosts
   172.16.20.240 hadoop1
   172.16.20.241 hadoop2
   172.16.20.242 hadoop3
   ```

3. 设置主机免密登录

   ```
   # 生成密钥文件
   ssh-keygen -t rsa
   
   # 将公钥传输到远程机器
   ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop2
   ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop3
   ```

4. 禁用selinux

   ```
   vim /etc/selinux/config
   SELINUX=disabled
   ```

5. 关闭防火墙

   ```
   systemctl disable firewalld.service
   systemctl stop firewalld.service
   ```

### 需要准备的软件
1. Jdk 要求1.8及以上版本

   下载地址: https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html

   因为现在在oracle下载jdk需要登录，然后在网上找到了一个共享的账号，按需使用

   用户名：2696671285@qq.com

   密码:  Oracle123

   ```
   wget https://download.oracle.com/otn/java/jdk/8u211-b12/478a62b7d4e34b78b671c754eaaf38ab/jdk-8u211-linux-i586.rpm
   ```
   
2. Hadoop 2.8.1

   下载地址: https://hadoop.apache.org/releases.html

   ```
   wget http://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-2.8.5/hadoop-2.8.5.tar.gz
   ```

3. Hbase 2.2.0

   下载地址: http://hbase.apache.org/downloads.html

   ```
   wget http://mirrors.tuna.tsinghua.edu.cn/apache/hbase/2.2.0/hbase-2.2.0-bin.tar.gz
   ```

4. Zookeeper 3.5.5

   下载地址：https://www.apache.org/dyn/closer.cgi/zookeeper/

   ```
   wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/stable/apache-zookeeper-3.5.5-bin.tar.gz
   ```

### JDK 安装
1. 通过rpm 安装

   ```
   rpm -ivh jdk-8u211-linux-x64.rpm
   ```

2. 配置环境变量

   ```
   vim /etc/profile
   export JAVA_HOME=/usr/java/jdk1.8.0_211-amd64
   export PATH=$JAVA_HOME/bin:$PATH
   export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
   source /etc/profile
   ```

### Hadoop 安装

1. 解压hadoop

   ```
   tar -zxvf hadoop-2.8.5.tar.gz
   ```

2. 移动到指定的安装目录，我这里指定的是 /home/hadoop 下

   ```
   mv hadoop-2.8.5 /home/hadoop/hadoop
   ```

3. 配置环境变量

   ```
   vim /etc/profile
   export HADOOP_HOME=/home/hadoop/hadoop
   export HADOOP_MAPRED_HOME=$HADOOP_HOME
   export HADOOP_COMMON_HOME=$HADOOP_HOME
   export HADOOP_HDFS_HOME=$HADOOP_HOME
   export HADOOP_YARN_HOME=$HADOOP_HOME
   export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
   export PATH=$JAVA_HOME/bin:$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
   ```

4. 修改 hadoop-env.sh 设置hadoop环境变量

   ```
   vim home/hadoop/hadoop/etc/hadoop/hadoop-env.sh
   export JAVA_HOME=/usr/java/jdk1.8.0_211-amd64
   ```

5. 修改 core-site.xml

   ```
   <configuration>
     <property>
        <name>fs.default.name</name>
        <value>hdfs://hadoop1:9000</value>
     </property>
   </configuration>
   ```

6. 修改 hdfs-site.xml

   ```
   <configuration>
     <property>
               <name>dfs.namenode.name.dir</name>
               <value>/home/hadoop/hadoop/hdfs/name</value>
       </property>
   
       <property>
               <name>dfs.datanode.data.dir</name>
               <value>/home/hadoop/hadoop/hdfs/data</value>
       </property>
   
       <property>
               <name>dfs.replication</name>
               <value>2</value>
       </property>
   </configuration>
   ```

7. 设置 YARN 为 任务调度员

8. ```
   cp mapred-site.xml.template mapred-site.xml
   vim mapred-site.xml
   <configuration>
       <property>
               <name>mapreduce.framework.name</name>
               <value>yarn</value>
       </property>
   </configuration>
   ```

9. 修改 yarn-site.xml

   ```
   <configuration>
       <property>
               <name>yarn.acl.enable</name>
               <value>0</value>
       </property>
   
       <property>
               <name>yarn.resourcemanager.hostname</name>
               <value>hadoop1</value>
       </property>
   
       <property>
               <name>yarn.nodemanager.aux-services</name>
               <value>mapreduce_shuffle</value>
       </property>
   </configuration>
   ```

10. 修改 slaves 添加工作节点

   ```
   vim slaves
   hadoop2
   hadoop3
   ```

11. 因为我是在渣渣笔记本的虚拟机上搭建的集群，因此现在需要配置一下内存，修改 yarn-site.xml

    ```
    <property>
            <name>yarn.nodemanager.resource.memory-mb</name>
            <value>1536</value>
    </property>
    
    <property>
            <name>yarn.scheduler.maximum-allocation-mb</name>
            <value>1536</value>
    </property>
    
    <property>
            <name>yarn.scheduler.minimum-allocation-mb</name>
            <value>128</value>
    </property>
    
    <property>
            <name>yarn.nodemanager.vmem-check-enabled</name>
            <value>false</value>
    </property>
    ```

    修改 mapred-site.xml

    ```
    <property>
            <name>yarn.app.mapreduce.am.resource.mb</name>
            <value>512</value>
    </property>
    
    <property>
            <name>mapreduce.map.memory.mb</name>
            <value>256</value>
    </property>
    
    <property>
            <name>mapreduce.reduce.memory.mb</name>
            <value>256</value>
    </property>
    ```

12. 好像配置到这里，hadoop的相关配置就完成了，现在将这里配置在其他的节点上也同样执行，手动配置一次也行，但是我建议，直接祭出 scp 大法吧,先到 hadoop2 和 hadoop3 节点上去把ect下的hadoop干掉，然后回来 hadoop1上执行

    ```
    scp -r hadoop/ hadoop2:/home/hadoop/hadoop/etc
    scp -r hadoop/ hadoop3:/home/hadoop/hadoop/etc
    ```

13. 回到hadoop1 上执行格式化一下，奔跑吧@@大笨象！！

    ```
    hadoop namenode -format
    ```

    跑完之后,在 dfs.namenode.name.dir 所指向的路径 /home/hadoop/hadoop/hdfs/name 会记录到当前的 cuurent/xxxxx

    启动大笨象～～ ⚠️注意：这个大笨象很厉害，只需要在 命名节点上启动就好，其他数据几点会自动启动，很吊有木有！！

    ```
    sbin/start-all.sh
    jps
    4866 NodeManager
    4370 NameNode
    4899 Jps
    4648 SecondaryNameNode
    4779 ResourceManager
    4460 DataNode
    ```

### 安装 Zookeeper 

1. 修改 zoo.cfg

   ```
   cd /zookeeper/conf
   cp zoo_sample.cfg zoo.cfg
   vim zoo.cfg
   dataDir=/home/hadoop/hadoop/zookeeper/data
   
   server.1=172.16.20.240:2888:3888
   server.2=172.16.20.241:2888:3888
   server.3=172.16.20.242:2888:3888
   ```

   ⚠️这里的 server.1 server.2 server.3 并不是乱写的哦，这个是接下来相关节点的myid

2. 初始化目录并创建相对应的myid

   ```
   mkdir -p /home/hadoop/hadoop/zookeeper/data
   echo "1" > /home/hadoop/zookeeper/data/myid
   ```

3. 将当前的配置文件复制到其他的节点，并在相对应的节点上执行 第二步的 脚本

   ```
   scp zoo.cfg hadoop2:/home/hadoop/hadoop/zookeeper/conf/
   scp zoo.cfg hadoop3:/home/hadoop/hadoop/zookeeper/conf/
   ```

4. 启动吧！

   ```
   bin/zkServer.sh start
   ```

### 安装Hbase

1. 修改 hbase-env.sh 配置环境变量，这里记得要配置好JAVA_HOME，不然启动不了

   ```
   vim /home/hadoop/hbase/hbase-env.sh
   # 这里如果忘记的话，echo $JAVA_HOME 看下
   export JAVA_HOME=/usr/java/jdk1.8.0_51
   # 这里的CLASSPATH 其实就是上面hadoop的配置文件路径
   export HBASE_CLASSPATH=/home/hadoop/hadoop/etc/hadoop/
   export HBASE_MANAGES_ZK=false
   ```

2. 修改 hbase-site.xml

   ```
   vim hbase-site.xml
   <configuration>
     <property>
       <name>hbase.rootdir</name>
       <value>hdfs://hadoop1:9000/hbase</value>
     </property>
     <property>
       <name>hbase.master</name>
       <!-- 声明命名节点为master -->
       <value>hadoop1</value>
     </property>
     <property>
       <name>hbase.cluster.distributed</name>
       <value>true</value>
     </property>
     <property>
       <name>hbase.zookeeper.property.clientPort</name>
       <value>2181</value>
     </property>
     <property>
       <name>hbase.zookeeper.quorum</name>
       <!-- 这里是hadoop集群的主机名称 -->
       <value>hadoop1,hadoop2,hadoop3</value>
     </property>
     <property>
       <name>zookeeper.session.timeout</name>
       <value>60000000</value>
     </property>
     <property>
       <name>dfs.support.append</name>
       <value>true</value>
     </property>
   </configuration>
   ```

3. 修改 regionservers，声明slave们～

   ```
   vim regionservers
   hadoop2
   hadoop3
   ```

4. 貌似至此，配置已经全部完成，这个时候同样祭出 scp 大法将hase 下面整个配置文件复制到其他节点

   ```
   scp -r conf/ hadoop2:/home/hadoop/hbase/
   scp -r conf/ hadoop3:/home/hadoop/hbase/
   ```

### 启动！！！奔跑吧！！大笨象！！

```
# 启动hadoop  只需在master上启动，其他节点会被唤起
	/hadoop/sbin/start-all.sh
# 启动 zookeeper  需要在三个节点上执行
  /zookeeper/bin/zkServer.sh start
# 启动 hase  只需在master上启动，其他节点会被唤起
  /hbase/bin/start-base.sh
```

启动完成后，你就会看到大笨象跳舞了～💃

首先在master上，jps一下正常的话会看到

```
jps
63150 Jps
# hadoop进程
61287 SecondaryNameNode  
# hadoop master进程
67899 NameNode      
# hadoop进程
62367 ResourceManager    
# hbase master进程
67182 HMaster    
# zookeeper进程
67059 QuorumPeerMain     
```

在 slave上，正常会看到

```
14342 Jps
# zookeeper进程
15234 QuorumPeerMain      
# hadoop slave进程
15792 DataNode             
# hbase slave进程
15651 HRegionServer        
```


