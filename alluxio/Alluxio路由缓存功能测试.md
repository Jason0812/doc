# Alluxio 路由缓存功能测试

## HA测试

分别在namenode1上启动Alluxio master和proxy；

```shell
[alluxio@namenode1 ~]$ alluxio-start.sh master
[alluxio@namenode1 ~]$ alluxio-start.sh proxy
[alluxio@namenode2 ~]$ alluxio-start.sh master
[alluxio@namenode2 ~]$ alluxio-start.sh proxy
```

在Client端配置`core-site.xml`

```xml
<property>
        <name>fs.defaultFS</name>
        <value>alluxio-ft://namenode1:19998</value>
</property>
```

1. 在客户端执行如下命令：

```shell
[bigdata@viewfsclient hadoop]$ hdfs dfs -ls /guoyj
Alluxio client (version 1.4.1-SNAPSHOT) is trying to connect with FileSystemMasterClient master @ namenode1/10.19.221.119:19998
```

此时client连接是namenode1;

2. 停掉namenode1上的master；

```shell
[alluxio@namenode1 target]$ alluxio-stop.sh master
Killed 1 processes on namenode1

##此时会去连接namenode2；
[bigdata@viewfsclient hadoop]$ hdfs dfs -ls /guoyj
Alluxio client (version 1.4.1-SNAPSHOT) is trying to connect with FileSystemMasterClient master @ namenode2/10.19.221.120:19998
```

NOTE：当Master关掉之后，立即执行命令，会报错；

```shell
17/05/22 14:59:00 INFO logger.type: Alluxio client (version 1.4.1-SNAPSHOT) is trying to connect with FileSystemMasterClient master @ namenode1/10.19.221.119:19998
17/05/22 14:59:00 ERROR logger.type: Failed to connect (0) to FileSystemMasterClient master @ namenode1/10.19.221.119:19998 : java.net.ConnectException: Connection refused
17/05/22 14:59:00 INFO logger.type: Master addresses: [namenode2:19998, master:19998, namenode1:19998]
```

**测试结果**：

当其中一个Master 挂掉之后，Standy 的Master会在*一个时间窗口内*启动，和Client进行交互；

<font color = red> 需要对此时间窗口进行研究和优化 </font>

## Mount测试

在测试之前对Alluxio进行format；在HDFS中创建

```shell
[bigdata@namenode1 hadoop]$ hdfs dfs -mkdir /user/guoyj
[bigdata@namenode1 hadoop]$ hdfs dfs -mkdir /tmp
```

在Alluxio中对目录进行挂载: 

1. 挂载的文件在HDFS中存在；

```shell
[alluxio@namenode1 target]$ alluxio fs mount /guoyj hdfs://CnSuningHadoop/user/guoyj
Mounted hdfs://CnSuningHadoop/user/guoyj at /guoyj
[alluxio@namenode1 target]$ alluxio fs mount /tmp hdfs://CnSuningHadoop/tmp
Mounted hdfs://CnSuningHadoop/tmp at /tmp
```

2. 挂载的文件在HDFS中不存在；

```shell
[alluxio@namenode1 target]$ alluxio fs mount /test hdfs://CnSuningHadoop/test4444
ThriftIOException(message:Ufs path /test4444 does not exist)
```

3. 挂载的是一个文件；

```shell
[bigdata@namenode1 hadoop]$ hdfs dfs -touchz /testFile
[alluxio@namenode1 target]$ alluxio fs mount /testFile hdfs://CnSuningHadoop/testFile
ThriftIOException(message:Ufs path /testFile does not exist)
```

4. 将HDFS的路径重复挂载到其他Alluxio路径上；

```shell
[alluxio@namenode1 target]$ alluxio fs mount /guoyj1 hdfs://CnSuningHadoop/user/guoyj
Mount point hdfs://CnSuningHadoop/user/guoyj is a prefix of hdfs://CnSuningHadoop/user/guoyj
```

5. 对Alluxio已挂载的点进行多次挂载；

```shell
[alluxio@namenode1 target]$ alluxio fs mount /guoyj hdfs://CnSuningHadoop/alluxio
Mount point /guoyj already exists
```

6. 对挂载点的子目录进行挂载；

```shell
[alluxio@namenode1 target]$ alluxio fs mount /guoyj/alluxio hdfs://CnSuningHadoop/alluxio
Mount point /guoyj is a prefix of /guoyj/alluxio
```

**测试结果**: 通过上面测试可知：

1. Alluxio的路径是挂载点的时候，不能对其进行再次挂载；也不能将UFS中的路径挂载到其子目录中；
2. HDFS的路径只能挂载一次；
3. 不支持对HDFS中不存在的路径进行挂载，不支持对HDFS中的文件进行挂载；

### ListMountPoint测试

```shell
[alluxio@namenode1 target]$ alluxio fs listMountPoint
/tmp -> hdfs://CnSuningHadoop/tmp
/ -> /home/alluxio/software/alluxio/underFSStorage
/guoyj -> hdfs://CnSuningHadoop/user/guoyj
```

通过测试结果可知：listMountPoint符合预期；

### Load功能测试







## RC - API测试

API的测试分为两部分，第一部分是作为路由功能的时候的API测试，第二部分是作为路由缓存功能的API测试；

此处主要是对RC功能的API测试的记录文档；

说明在此测试文档中，

Alluxio shell：使用Alluxio的shell命令去操作；`[alluxio@namenode1 target]`

HDFS shell：使用HDFS的shell，jar包为HDFS社区的jar；`[bigdata@namenode1 hadoop]`

Proxy shell： 使用HDFS的shell命令，jar是开发的proxy的jar；`[bigdata@viewfsclient lib]`

在Proxy shell的这个客户端中添加配置：

```shell
alluxio.user.mode.cache.enabled=true
alluxio.user.mustCacheList=/tmp
```

### Mkdir

在mkdir的时候，应当根据路径是否在UserMustCacheList去测试；

1. mkdir 的路径在userMustCacheList中；(测试结果符合预期)

```shell
[bigdata@viewfsclient lib]$ hdfs dfs -mkdir /tmp/testDir

[bigdata@viewfsclient lib]$ hdfs dfs -ls /tmp
drwxrwxrwx   - bigdata bigdata          0 2017-05-22 14:32 /tmp/testDir

[alluxio@namenode1 target]$ alluxio fs ls /tmp
drwxrwxrwx     bigdata        bigdata        0.00B     05-22-2017 14:32:15:810  Directory      /tmp/testDir

[bigdata@namenode1 hadoop]$ hdfs dfs -ls /tmp
```

NOTE: 在测试中报错如下：

```shell
17/05/22 14:32:17 ERROR logger.type: FileDoesNotExistException. path: alluxio://namenode1:19998/tmp/testDir
```

2. mkdir的路径不在UserMustCacheList中；(测试结果符合预期)

```shell
[bigdata@viewfsclient lib]$ hdfs dfs -mkdir /guoyj/testDir

[bigdata@viewfsclient lib]$ hdfs dfs -ls /guoyj/
drwxr-xr-x   - bigdata supergroup          0 2017-05-22 14:35 /guoyj/testDir

[alluxio@namenode1 target]$ alluxio fs ls /guoyj

[bigdata@namenode1 hadoop]$ hdfs dfs -ls /user/guoyj
Found 1 items
drwxr-xr-x   - bigdata supergroup          0 2017-05-22 14:35 /user/guoyj/testDir
```

3. 当创建的路径存在的时候；

```shell
[bigdata@viewfsclient lib]$hdfs dfs -mkdir /guoyj/testDir
mkdir: `/guoyj/testDir': File exists
[bigdata@viewfsclient lib]$ hdfs dfs -mkdir /tmp/testDir
mkdir: `/tmp/testDir': File exists
```

**测试结果**: 

1. 当创建的路径在UserMustCacheList中，文件只会存在在Alluxio Space中，不会through到UFS中；元数据只在Alluxio中；
2. 当创建的路径不在UserMustCacheList中， 文件只会存在HDFS Space中，不会在Alluxio中存在；元数据只在HDFS中；
3. 创建的文件已存在时，会提示文件已存在；

### Create

在create的时候，应当根据路径是否在UserMustCacheList去测试；

1. 文件在UserMustCacheList中；

```shell
[bigdata@viewfsclient lib]$ hdfs dfs -touchz /tmp/testFile

[bigdata@viewfsclient lib]$ hdfs dfs -ls /tmp
-rw-r--r--   3 bigdata bigdata          0 2017-05-22 14:45 /tmp/testFile

[alluxio@namenode1 target]$ alluxio fs ls /tmp
-rw-r--r--     bigdata        bigdata        0.00B     05-22-2017 14:45:24:394  In Memory      /tmp/testFile

[bigdata@namenode1 hadoop]$ hdfs dfs -ls /tmp
```

2. 文件不在UserMustCacheList中；

```shell
[bigdata@viewfsclient lib]$ hdfs dfs -touchz /guoyj/testFile
[bigdata@viewfsclient lib]$ hdfs dfs -ls /guoyj
-rw-r--r--   3 bigdata supergroup          0 2017-05-22 14:47 /guoyj/testFile

[alluxio@namenode1 target]$ alluxio fs ls /guoyj

[bigdata@namenode1 hadoop]$ hdfs dfs -ls /user/guoyj
-rw-r--r--   3 bigdata supergroup          0 2017-05-22 14:47 /user/guoyj/testFile
```

3. 文件已存在

```shell
##这类测试应当去写API去测试，设置overwrite进行测试；
```

4. 创建的文件是路径且已存在

```shell
[bigdata@viewfsclient lib]$ hdfs dfs -touchz /tmp/testDir
touchz: `/tmp/testDir': Is a directory

[bigdata@viewfsclient lib]$ hdfs dfs -touchz /guoyj/testDir
touchz: `/guoyj/testDir': Is a directory
```





