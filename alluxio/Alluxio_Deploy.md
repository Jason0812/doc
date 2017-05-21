# Alluxio Deploy的文档

## 单一nameservice的部署

### Client端的配置部署

1. 将HDFS的`core-site.xml`中的`fs.defaultFS`配置如下：

```xml
<Property>
  <name>fs.defaultFS</name>
  <value>alluxio-ft://namenode1:19998</value>
</Property>
```

1. 然后将`hdfs-site.xml`copy到client端；
2. `alluxio.user.mode.cache.enabled` 设置成true，表示打开cache功能；
3. `alluxio.user.mode.route.enabled`设置成true，表示打开route功能；
4. `alluxio.user.mustCacheList`设置需要Must_cache的配置，以逗号隔开；

### alluxio Master端的部署配置

将HDFS中的`core-site.xml`和`hdfs-site.xml`复制到`alluxio/conf/`中即可；(需要向所有master中复制这两个文件)；

在master端`alluxio.master.load.metadata.from.ufs.enable`设置为true；表示Master不去HDFS中load元数据；

## 多nameservice的部署

### client端的部署

1. 将HDFS的`core-site.xml`中的`fs.defaultFS`配置如下：

```xml
<Property>
  <name>fs.defaultFS</name>
  <value>alluxio-ft://namenode1:19998</value>
</Property>
```

1. 在`hdfs-site.xml`中配置如下：

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

1. `alluxio.user.mode.cache.enabled` 设置成true，表示打开cache功能；
2. `alluxio.user.mode.route.enabled`设置成true，表示打开route功能；
3. `alluxio.user.mustCacheList`设置需要Must_cache的配置，以逗号隔开；

### Alluxio Master端的部署配置

1. 将client配置步骤中的`core-site.xml`和`hdfs-site.xml` copy 到所有的Alluxio Master的`alluxio/conf`下
2. 在master端`alluxio.master.load.metadata.from.ufs.enable`设置为true；表示Master不去HDFS中load元数据；