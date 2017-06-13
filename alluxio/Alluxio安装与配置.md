# Alluxio安装与配置

## 包分发

1. 创建用户alluxio，详细信息如下：

| 用户名     | 组名         | 所在机器 | 用途                      |
| ------- | ---------- | ---- | ----------------------- |
| alluxio | supergroup | all  | Alluxio(Master, Worker) |

2. 创建命令：

```shell
#useradd -g supergroup alluxio
```

NOTE: `#`表示`root`执行， `$`表示`alluxio`用户执行

3. 免密码登录

`master1` 和`master2`机器之间互登，及`master1, master2`和其他的`datanode`的免密登录；执行命令如下：

```shell
##root create temp passwd for alluxio
echo alluxio:alluxio | chpasswd
## Master1
ssh-keygen
$ssh-copy-id master1

##copy ~/.ssh to master2
scp -r ~/.ssh master2:~/.ssh

## master1 to workers
ssh-copy-id worker1
## for many workerlist
#for i in `cat workerlist`; do echo $i; sshpass -p 'alluxio' ssh-copy-id $i; done

##root delete temp passwd
echo alluxio: | chpasswd

## verify setting, execute in master1 and master2
for i in `cat workerlist`; do ssh -o "StrictHostKeyChecking no" $i 'hostname'; done
```

4. 包分发

在用户的主目录`/home/alluxio`中创建`software`目录；解压`JDK`和`Alluxio`包到software下，并创建相应的软链接；

```shell
ln -s /home/alluxio/software/alluxio-xxx alluxio
```

`/home/alluxio/software`的结构如下

```shell&#39;
lrwxrwxrwx  1 alluxio bigdata   37 Jun  7 12:03 alluxio -> /home/alluxio/software/alluxio-1.4.0/
drwxr-xr-x 20 alluxio bigdata 4096 Jun  7 11:32 alluxio-1.4.0
lrwxrwxrwx  1 alluxio bigdata   35 Jun  7 12:03 java -> /home/alluxio/software/jdk1.7.0_60/
drwxr-xr-x  8 alluxio bigdata 4096 Apr 17  2014 jdk1.7.0_60
```

## 环境配置

在`alluxio`用户的`~/.bashrc`文件中添加如下内容。

```shell
export JAVA_HOME=/home/alluxio/software/java
export ALLUXIO_HOME=/home/alluxio/software/alluxio
export PATH=$JAVA_HOME/bin/:$ALLUXIO_HOME/bin:$PATH
```

## Alluxio配置

`master`端在`alluxio-env.sh`中添加如下配置:

```shell
export ALLUXIO_HOME=/home/alluxio/software/alluxio
export ALLUXIO_LOGS_DIR={$ALLUXIO_HOME}/logs
##不同的master节点，需要改相应的配置
export ALLUXIO_MASTER_HOSTNAME=alluxio01
export ALLUXIO_RAM_FOLDER=/mnt/ramdisk
export ALLUXIO_WORKER_MEMORY_SIZE=2GB
export ALLUXIO_MASTER_JAVA_OPTS+="-Xms512m -Xmx512M -XX:NewRatio=7 -XX:SurvivorRatio=4 -XX:MaxPermSize=200M -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:MaxDirectMemorySize=128M -XX:CMSInitiatingOccupancyFraction=70 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/home/alluxio/software/alluxio/logs/master.gc.log -Dalluxio.zookeeper.enabled=true -Dalluxio.zookeeper.address=alluxio01:2181,alluxio02:2181,alluxio03:2181,alluxio04:2181,alluxio05:2181 -Dalluxio.master.journal.folder=hdfs://CnSuningHadoop/alluxio/journal"
export ALLUXIO_WORKER_JAVA_OPTS="-Xms1G -Xmx1G -XX:NewRatio=7 -XX:SurvivorRatio=4 -XX:MaxPermSize=200M -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled -XX:MaxDirectMemorySize=128M -XX:CMSInitiatingOccupancyFraction=70 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/home/alluxio/software/alluxio/logs/worker.gc.log -Dalluxio.zookeeper.enabled=true -Dalluxio.zookeeper.address=alluxio01:2181,alluxio02:2181,alluxio03:2181,alluxio04:2181,alluxio05:2181"
export ALLUXIO_USER_JAVA_OPTS="  -Dalluxio.zookeeper.enabled=true -Dalluxio.zookeeper.address=alluxio01:2181,alluxio02:2181,alluxio03:2181,alluxio04:2181,alluxio05:2181"
```

在`alluxio-site.properites`中添加如下配置：

```shell
# Common properties
alluxio.debug=false
alluxio.test.mode=false
alluxio.web.threads=1

# Integeration properties
alluxio.integration.master.resource.cpu=1
alluxio.integration.master.resource.mem=1024MB
alluxio.integration.worker.resource.cpu=1
alluxio.integration.worker.resource.mem=1024MB

# Security properties
alluxio.security.authentication.type=SIMPLE
alluxio.security.authorization.permission.umask=000
alluxio.security.authorization.permission.enabled=true
alluxio.security.authorization.permission.supergroup=supergroup
alluxio.security.group.mapping.class=alluxio.security.group.provider.ShellBasedUnixGroupsMapping

# Master properties
alluxio.master.bind.host=0.0.0.0
alluxio.master.port=19998
alluxio.master.retry=29
alluxio.master.web.bind.host=0.0.0.0
alluxio.master.web.port=19999
## Default value: /
alluxio.master.whitelist=/
alluxio.master.worker.threads.max=2048
alluxio.master.worker.threads.min=512
alluxio.master.worker.timeout.ms=300000
alluxio.master.tieredstore.global.levels=3
alluxio.master.tieredstore.global.level0.alias=MEM
alluxio.master.tieredstore.global.level1.alias=SSD
alluxio.master.tieredstore.global.level2.alias=HDD
alluxio.master.journal.folder=hdfs://CnSuningHadoop/alluxio/journal
# Worker Properties
alluxio.worker.bind.host=0.0.0.0
alluxio.worker.block.threads.max=2048
alluxio.worker.block.threads.min=256
alluxio.worker.data.bind.host=0.0.0.0
alluxio.worker.data.folder=/alluxioworker/
alluxio.worker.data.port=29999
alluxio.worker.file.persist.pool.size=2
alluxio.worker.memory.size=1000MB
alluxio.worker.port=29998
alluxio.worker.session.timeout.ms=10000
alluxio.worker.tieredstore.reserver.enabled=true
alluxio.worker.tieredstore.levels=1
alluxio.worker.tieredstore.level0.alias=MEM
alluxio.worker.tieredstore.level0.dirs.path=/mnt/ramdisk
alluxio.worker.tieredstore.level0.dirs.quota=${alluxio.worker.memory.size}
alluxio.worker.tieredstore.level0.reserved.ratio=0.1
#alluxio.worker.tieredstore.level1.alias=HDD
#alluxio.worker.tieredstore.level1.dirs.path=/data/alluxio/data1,/data/alluxio/data2
#alluxio.worker.tieredstore.level1.dirs.quota=1GB,2GB
#alluxio.worker.tieredstore.level1.reserved.ratio=0.2
alluxio.worker.web.bind.host=0.0.0.0
alluxio.worker.web.port=30000
alluxio.worker.block.heartbeat.timeout.ms=300000


# User Properties
alluxio.user.block.master.client.threads=10
alluxio.user.block.worker.client.threads=128
alluxio.user.block.remote.read.buffer.size.bytes=8MB
alluxio.user.block.size.bytes.default=1024
alluxio.user.failed.space.request.limits=3
alluxio.user.file.buffer.bytes=1MB
alluxio.user.file.cache.partially.read.block=true
alluxio.user.file.master.client.threads=10
alluxio.user.file.readtype.default=NO_CACHE
alluxio.user.file.write.location.policy.class=alluxio.client.file.policy.LocalFirstPolicy
alluxio.user.file.writetype.default=CACHE_THROUGH
alluxio.user.lineage.enabled=false
alluxio.user.lineage.master.client.threads=10
alluxio.user.ufs.delegation.enabled=true
#alluxio.user.mustCacheList=""
alluxio.master.load.metadata.from.ufs.enable=false
#alluxio.user.mode.route.enabled=true
#alluxio.user.mode.cache.enabled=true
```

在`works`中添加Alluxio worker的hostname;

*`alluxio.user.mustCacheList`*应当根据实际使用情况分析和配置；

需要将`journal log`和挂载点所在的HDFS集群的`core-site.xml` 和`hdfs-site.xml`复制到`./conf`下面；

## 启动Alluxio

在启动`Alluxio`之前，先确保`zookeeper`集群和`HDFS`集群已经启动成功；

在`HDFS`集群建立`alluxio.master.journal.folder`的目录并赋予权限；

```shell
drwxr-xr-x   - alluxio supergroup          0 2017-06-07 15:35 /alluxio
```

Alluxio集群的启动步骤如下：

```shell
# Master1
alluxio format
alluxio-start.sh master
alluxio-start.sh proxy
# Master2 
alluxio-start.sh master
alluxio-start.sh proxy

# worker
alluxio-start.sh worker
```

NOTE:

1. 在启动worker的时候，会让你输入mount的ramfs的密码；通过修改`/etc/sudoers` 解决这个问题：

```shell
alluxio ALL=(ALL) NOPASSWD:ALL
```

2. 在Alluxio启动成功之后，应当将HDFS上面的相关的路径全部mount到Alluxio中；

```shell
alluxio fs mkdir /tmp
alluxio fs mount /tmp hdfs://xxSuningHadoop/tmp(挂到下级目录上面去)
alluxio fs mkdir /user
alluxio fs mount /user/bigdata hdfs://xxSuningHadoop/user/bigdata
```

3. *当路径没有挂载直接使用，此时会报错；*

## YARN运行在Alluxio上的配置

在用YARN用户提交任务的时候，需要将HDFS的/user/yarn挂载到Alluxio中，同时将/history的目录挂载到Alluxio中：

```shell
##HDFS
hdfs dfs -mkdir -p /tmp/logs
hdfs dfs -chown yarn:yarn /tmp/logs
hdfs dfs -chmod -R 777 /tmp/logs
hdfs dfs -mkdir /history
hdfs dfs -chown bigdata:bigdata /history
hdfs dfs -chmod 777 /history
hdfs dfs -mkdir /tmp/hadoop-yarn/
hdfs dfs -chown yarn:yarn /tmp/hadoop-yarn/
hdfs dfs -chmod -R 777 /tmp/hadoop-yarn/
alluxio fs mount /history hdfs://xxSuningHadoop/history
alluxio fs mount /tmp/logs hdfs:/xx/tmp/logs


hdfs dfs -mkdir /user/yarn
hdfs dfs -chown yarn:yarn /user/yarn
alluxio fs mount /user/yarn hdfs://xxSuningHadoop/user/yarn
alluxio fs mount /tmp/hadoop-yarn hdfs://xx/tmp/hadoop-yarn
```

在`core-site.xml`中添加如下配置：

```xml
<property>
        <name>fs.defaultFS</name>
        <value>alluxio-ft://alluxio01:19998</value>
        </property>
<property>
  <property>
        <name>alluxio.zookeeper.address</name>
        <value>alluxio01:2015,alluxio02:2015,alluxio03:2015</value>
</property>
<property>
        <name>fs.alluxio.impl</name>
        <value>alluxio.hadoop.FileSystem</value>
</property>
<property>
        <name>fs.alluxio-ft.impl</name>
        <value>alluxio.hadoop.FaultTolerantFileSystem</value>
</property>
<property>
         <name>fs.AbstractFileSystem.alluxio.impl</name>
         <value>alluxio.hadoop.AlluxioFileSystem</value>
</property>
<property>
         <name>fs.AbstractFileSystem.alluxio-ft.impl</name>
         <value>alluxio.hadoop.FaultTolerantAlluxioFileSystem</value>
</property>
<property>
         <name>alluxio.user.mode.cache.enabled</name>
         <value>true</value>
</property>
<property>
         <name>alluxio.client.cache.enable</name>
         <value>false</value>
</property>
```

NOTE: *`alluxio.client.cache.enabled`*的default值是true，该值只在长进程中配置为false；

将alluxio 的client jar包copy到yarn的`./share/hadoop/common/lib`;

在`hadoop-env.sh`中添加：

```shell
export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:$HADOOP_HOME/share/hadoop/common/hadoop-lzo-0.4.20-SNAPSHOT.jar:$HADOOP_HOME/share/hadoop/common/lib/alluxio-core-client-1.4.1-SNAPSHOT-jar-with-dependencies.jar
```

至此可以执行Map-Reduce任务：

```shell
yarn jar ~/software/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.4.0.jar wordcount /user/yarn/nodelist /user/yarn/out
```

## Hive运行在Alluxio上的配置



## HBase运行在Alluxio上的配置



## Spark运行在Alluxio上的配置

在spark的conf/core-site.xml中增加和修改如下配置：

```xml
<property>
        <name>fs.defaultFS</name>
        <value>alluxio-ft://alluxio01:19998</value>
</property>
<property>
        <name>alluxio.zookeeper.address</name>
        <value>alluxio01:2015,alluxio02:2015,alluxio03:2015</value>
</property>
<property>
        <name>fs.alluxio.impl</name>
        <value>alluxio.hadoop.FileSystem</value>
</property>
<property>
        <name>fs.alluxio-ft.impl</name>
        <value>alluxio.hadoop.FaultTolerantFileSystem</value>
</property>
<property>
         <name>fs.AbstractFileSystem.alluxio.impl</name>
         <value>alluxio.hadoop.AlluxioFileSystem</value>
</property>
<property>
         <name>fs.AbstractFileSystem.alluxio-ft.impl</name>
         <value>alluxio.hadoop.FaultTolerantAlluxioFileSystem</value>
</property>
<property>
         <name>alluxio.user.mode.cache.enabled</name>
         <value>true</value>
</property>
<property>
         <name>alluxio.client.cache.enable</name>
         <value>true</value>
</property>
```

在spark的conf/spark-defaults.conf中修改配置：

```shell
spark.yarn.jars alluxio://xxxx:port/user/bigdata/spark/lib/*.jar,alluxio://xxxx:prot/user/bigdata/spark/ext/*.jar
```

然后将jar包put到此路径下；

将alluxio client的jar包放入到相应的路径后，然后在spark的conf/spark-env.sh中添加：

```shell
export SPARK_CLASSPATH=/home/bigdata/software/hadoop/lib/alluxio-core-client-1.4.1-SNAPSHOT-jar-with-dependencies.jar:$SPARK_CLASSPATH
```

至此执行spark的任务通过；

```shell
bin/spark-submit --class org.apache.spark.examples.JavaWordCount --master yarn --deploy-mode client ~/software/spark/examples/spark-examples_2.10-1.6.2.jar /user/proxyTest/spark/LICENSE
```

## CREATE NEW USER

```shell
##在OS层面添加用户
useradd proxyTest
##在HDFS中添加相应的用户可访问的路径
[bigdata@alluxio01 ~]$ hdfs dfs -mkdir -p /user/proxyTest/
[bigdata@alluxio01 ~]$ hdfs dfs -chown proxyTest:proxyTest /user/proxyTest
```

在core-site.xml中添加：

```xml
<property>
        <name>fs.defaultFS</name>
        <value>alluxio-ft://alluxio01:19998</value>
        </property>
<property>
  <property>
        <name>alluxio.zookeeper.address</name>
        <value>alluxio01:2015,alluxio02:2015,alluxio03:2015</value>
</property>
<property>
        <name>fs.alluxio.impl</name>
        <value>alluxio.hadoop.FileSystem</value>
</property>
<property>
        <name>fs.alluxio-ft.impl</name>
        <value>alluxio.hadoop.FaultTolerantFileSystem</value>
</property>
<property>
         <name>fs.AbstractFileSystem.alluxio.impl</name>
         <value>alluxio.hadoop.AlluxioFileSystem</value>
</property>
<property>
         <name>fs.AbstractFileSystem.alluxio-ft.impl</name>
         <value>alluxio.hadoop.FaultTolerantAlluxioFileSystem</value>
</property>
<property>
         <name>alluxio.user.mode.cache.enabled</name>
         <value>true</value>
</property>
<property>
         <name>alluxio.client.cache.enable</name>
         <value>true</value>
</property>
```

在配置完~/.bashrc之后，需要将以下路径挂载之后，才可以访问HDFS；

```shell
alluxio fs mount /user/proxyTest hdfs://CnSuningHadoop/user/proxyTest
```

**执行MR任务**

接下来测试此用户去跑MR的任务：通过

```shell
yarn jar ~/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.4.0.jar wordcount  /user/proxyTest/exclude /user/proxyTest/out
```

**部署spark**

参考上面spark的部署，可以执行任务；

**部署Hive**

**部署Hbase**



## 上线前置条件

hive路径问题hive-6727

Hbase路径问题： 15129

```xml
<property>
  <name>alluxio.user.ufs.delegation.enabled</name>
  <value>false</value>
</property>
<!--这个参数需要仔细研究-->
```

