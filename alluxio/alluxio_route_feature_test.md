# Alluxio作为路由功能的测试

## 功能测试

功能测试主要包括：append, create, mkdir, open, setAttribute, rename, delete

API的测试采用的是Hdfs的shell进行测试；

**单一nameservice的部署见附录**

*单一的nameservice*  PASS，采用的是hdfs的shell，`hdfs dfs -ls /guoyj`

**多nameservice的部署见附录**

*多nameservice的API的测试*：PASS，功能测试结果符合预期

*API 的性能测试*： Continue，输出性能测试报告；

## 性能测试

*dfs-perf工具的使用和部署见附录*

性能测试包括*： @20170515，输出性能测试报告；

1. 直接操作HDFS的时候，dfs-perf的QPS；
2. 通过Alluxio作为Proxy之后，dfs-perf的QPS；



## HA测试

1. Alluxio HA机制的配置参考文档 `alluxio-HA.md`
2. 在配置成功之后，修改`FaultTolerantFileSystem`，让其继承`AbstractFileSystem`；

通过测试打印的日志可知，Alluxio在namenode1挂了之后，会转换到namenode2上操作；

*HA的功能测试*： PASS

*性能测试* continue

## 附录

### 单一nameservice的部署

#### Client端的配置部署

1. 将HDFS的`core-site.xml`中的`fs.defaultFS`配置如下：

```xml
<Property>
  <name>fs.defaultFS</name>
  <value>alluxio-ft://namenode1:19998</value>
</Property>
```

2. 然后将`hdfs-site.xml`copy到client端；

#### alluxio Master端的部署配置

将HDFS中的`core-site.xml`和`hdfs-site.xml`复制到`alluxio/conf/`中即可；(需要向所有master中复制这两个文件)

### 多nameservice的部署

#### client端的部署

1. 将HDFS的`core-site.xml`中的`fs.defaultFS`配置如下：

```xml
<Property>
  <name>fs.defaultFS</name>
  <value>alluxio-ft://namenode1:19998</value>
</Property>
```

2. 在`hdfs-site.xml`中配置如下：

```xml
<Property>
  <name>dfs.nameservices</name>
  <value>hadoop1,hadoop2</value>
</Property>
<!--配置hadoop1的相关配置-->
<Property>
  <name>dfs.ha.namenodes.hadoop1</name>
  <value>nn1,nn2</value>
</Property>
<Property>
  <name>dfs.namenode.rpc-address.hadoop1.nn1</name>
  <value>hostname/ip:9000</value>
</Property>
<Property>
  <name>dfs.namenode.rpc-address.hadoop1.nn2</name>
  <value>hostname/ip:9000</value>
</Property>
<Property>
  <name>dfs.namenode.http-address.hadoop.nn1</name>
  <value>hostname/ip:50070</value>
</Property>
<Property>
  <name>dfs.namenode.http-address.hadoop.nn2</name>
  <value>hostname/ip:50070</value>
</Property>
<Property>
  <name>dfs.client.failover.proxy.provider.hadoop1</name>
  <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</Property>
<!--配置hadoop2的相关配置-->
<Property>
  <name>dfs.ha.namenodes.hadoop2</name>
  <value>nn1,nn2</value>
</Property>
<Property>
  <name>dfs.namenode.rpc-address.hadoop2.nn1</name>
  <value>hostname/ip:9000</value>
</Property>
<Property>
  <name>dfs.namenode.rpc-address.hadoop2.nn2</name>
  <value>hostname/ip:9000</value>
</Property>
<Property>
  <name>dfs.namenode.http-address.hadoop2.nn1</name>
  <value>hostname/ip:50070</value>
</Property>
<Property>
  <name>dfs.namenode.http-address.hadoop2.nn2</name>
  <value>hostname/ip:50070</value>
</Property>
<Property>
  <name>dfs.client.failover.proxy.provider.hadoop2</name>
  <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
</Property>
```

以上为需要添加到`hdfs-site.xml`文件中，其他的配置保持不变；

#### Alluxio Master端的部署配置

1. 将client配置步骤中的`core-site.xml`和`hdfs-site.xml` copy 到所有的Alluxio Master的`alluxio/conf`下；

### Dfs-perf的使用

#### On Hdfs的使用和配置

1. 在`conf/dfs-perf-env.sh`中配置如下参数

```shell
export DFS_PERF_DFS_ADRESS="hdfs://nameservice"
export DFS_PERF_DFS_OPTS="-Dpasalab.dfs.perf.hdfs.impl=org.apache.hadoop.hdfs.DistributedFileSystem"
```

2. 配置测试相关的参数



3. usages

```shell
bin/dfs-perf-clean
bin/dfs-perf Metadata
bin/dfs-perf-collect Metadata
```

## 测试问题汇总

```shell
hdfs dfs -rm /guoyj/file1
```

在这种情形下，会把/guoyj/file1这个文件move到/user/bigdata/.Trash下面去，此时/user/bigdata必须在Alluxio中挂载；