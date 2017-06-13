# 运行环境配置

1. 将Alluxio和HDFS部署好之后；
2. 将以下的HDFS路径Mount到Alluxio相应的文档中；

```shell
#将/tmp目录挂载上来，同时check /tmp目录的权限
#YARN将jar和.xml的文件上传到/tmp/hadoop-yarn/staging/bigdata/.staging/xx
#YARN会将log写到/tmp/logs下面
#Hive会将结果先写到/tmp/hive-hive下，然后rename到用户自己的目录/user/xx/hive/warehouse/xx下；
#MustCacheList不能配置为/tmp, MUST_CACHE list的路径应当配的粒度小点；e.g /tmp/logs
alluxio fs mount /tmp hdfs://xx/tmp

#将/history目录挂载上来，同时check /history目录的权限
alluxio fs mount /history hdfs://xx/history

#将各个用户的目录挂载到Alluxio的User下面:
alluxio fs mkdir /user
alluxio fs mount /user/bigdata hdfs://xx/user/bigdata
alluxio fs mount /user/** hdfs://xx/user/**
```

3. check Alluxio的配置信息

```shell
#主要的配置参数(以下参数在Master端配置，$ALLUXIO_HOME/alluxio-site.properties)
#用于将Alluxio的master和HDFS断开；
#1.Master不会主动去NameNode加载元数据；将此参数传递给了LoadMetaOptions的一个参数；
#2.Master不会去UFS去操作，比如：rename, delete
alluxio.master.load.metadata.from.ufs.enable=false


```

