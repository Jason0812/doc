# Alluxio HA

Alluxio的容错通过多master实现。同一时刻，有多个master在运行，但是只有一个master被选举为leader，作为worker和client的通信首选；其余的master进入备用状态，和leader共享日志；以确保和leader维护这同样的文件系统的元数据，从而在leader失效时快速接管leader的工作；

## 环境要求

1. zookeeper
2. 共享journal的底层文件系统（选取HDFS）

Alluxio使用zookeeper实现容错和leader选举；Alluxio Master使用zookeeper来选举leader，client使用zookeeper查询当前的leader的地址和IP；

## 配置Alluxio

当zookeeper和共享文件系统都正常运行时，需要在每个主机上配置`Alluxio-env.sh`；

实现Alluxio的容错；需要为master，worker，client添加额外的配置在`Alluxio-env.sh`中；

```
-Dalluxio.zookeeper.enabled=true
-Dalluxio.zookeeper.address=[zookeeper_hostname]:2181, [zookeeper_hostname]:2181
```

### Master 配置

除了以上的配置外，Alluxio Master需要额外的配置：

```Shell
export ALLUXIO_MASTER_HOSTNAME = ""
```

需要指定正确的文件夹；需要在`ALLUXIO_JAVA_OPTS`中设置`alluxio.master.journal.folder`

```shell
-Dalluxio.master.journal.folder=hdfs://xxx/user/alluxio/journal
```

所有Alluxio Master以这种方式启动后，都可以实现Alluxio Master的容错；其中一个成为leader，其余重播日志，直到当前的master失效；

### Worker配置

只要以上配置正确，worker就可以查询zookeeper，获取当前应当链接的master。所以Worker无需设置`ALLUXIO_MASTER_HOSTNAME`;

*NOTE: 注意: 当在容错模式下运行Alluxio, worker的默认心跳超时时间可能太短。 为了能在master进行故障转移时正确处理master的状态，建议将worker的默认心跳超时时间设置的长点。 增加worker上的默认超时时间，可以通过修改`conf/alluxio-site.properties`下的配置参数 `alluxio.worker.block.heartbeat.timeout.ms` 至一个大些的值（至少几分钟）*

### Client配置

client端需要配置以下两个参数

```
-Dalluxio.zookeeper.enabled=true
-Dalluxio.zookeeper.address=[zookeeper_hostname]:2181
```

client在以上配置正确的时，会去zookeeper获取当前的lead master；

## HA的测试

在使用shell的时候，在`bin/hdfs`中添加Java执行的opts

```shell
-Dalluxio.zookeeper.enabled=true -Daluxio.zookeeper.address=[zookeeper_hostname1]:2181,[zookeeper_hostname2]:2181
```

在`etc/hadoop/core-site.xml`中配置

```xml
<property>
  <name>fs.defaultFS</name>
  <value>alluxio-ft://namenode1:19998</value>
</property>
```

**NOTE: Alluxio 的HA机制有待研究**

 