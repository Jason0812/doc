# Proxy功能测试点

1. load的时候，HDFS的时间和Alluxio中的时间点保持一致；

```shell
#Test Result
##HDFS Space
-rw-r--r--   3 bigdata supergroup          0 2017-06-11 08:57 /user/bigdata/testMetadataTime

##Alluxio Space
-rw-r--r--     bigdata        supergroup     0.00B     06-11-2017 08:57:56:316  In Memory      /user/bigdata/testMetadataTime

#Parent Directory
##HDFS Space
drwxrwxrwx   - bigdata supergroup          0 2017-06-11 08:57 /user/bigdata

##Alluxio Space
drwxrwxrwx     bigdata        supergroup     2.00B     06-09-2017 16:31:19:139  Directory      /user/bigdata
```

测试结论：Alluxio中文件的时间戳保持一致，但是上层路径的时间没有保存一致；<这个bug，是否需要fix>

NOTE: Alluxio自己在文件夹下创建新的路径的时候，也不会去`update`父目录的时间；

2. append的时候，采取的方法是先删除在append的方法；

NOTE：

1. 在append过程中，读取是从HDFS读取的；
2. append之后，目前不支持将文件主动的load到Alluxio space中来；*从功能完善的角度，需要将之前在Alluxio Space中的文件主动的load进来*
3. append的另一种设计方法将附录-1

*测试结果参考Alluxio API 测试结果*  需要增加HDFS shell的测试结果；



3. Create， 在create的时候，如果overwrite为true，会先删除在Alluxio space中的文件，但是不会主动的将文件reload到Alluxio space中；*从功能完善的角度，需要将之前在Alluxio Space中的文件主动的load进来，方案的设计参考append的第二种方法设计*，

*测试结果参考Alluxio API测试*， 需要增加HDFS shell的测试结果；

4. delete；*测试结果参考Alluxio API测试*， 需要增加HDFS shell的测试结果；

NOTE：delete的测试应当确保是在各自的space中delete，而不会去ufs中delete；

5. getFileBlockLocations：当文件在AlluxioSpace中的时候，是从Alluxio中获取，当文件不在的时候，是从UFS中获取；

NOTE：当文件没有改变的时候，Alluxio Space中的数据如果读取不到的时候回去Ufs读取

当读取错误的时候，也要求其去Ufs中读取数据；*添加测试：当文件读取异常的时候，会不会去UFSSpace中读取；*如果在读取过程出现错误不去ufs中读取的话，应当在返回BlockLocation的时候同时返回ufs的BlockLocations，从而让yarn做下一步的调度和数据的读取；*存在的问题： 下一次调度的时候，任然会get当前的block的数据；*

yarn是否存在判断副本的location和任务的location；如何相同，直接从本地读副本；

Alluxio 如何支持直接从本地Ufs中读取block；

当文件有改变的时候，新的block并不在Alluxio Space中，同时新加的block信息也不会Alluxio的元数据中，所以此时是拿不到数据的；*此功能需要添加 -> 如何让元数据进入到Alluxio Space中；*

6. getFileStatus；

文件在AlluxioSpace中的数据，从Alluxio中从master拿取元数据；不在Alluxio中，在HDFS中的时候，从HDFS的namenode拿去元数据；*第一版本的设计是文件在MustCacheList的时候才会去Alluxio中去拿元数据，否则都是去Ufs中去拿取元数据；此步骤会影响性能，因为我们是热数据进Alluxio space，对热数据的访问会非常频繁，这样会加大对HDFS的namenode的压力，加速的性能仅仅来自于从memory和从disk读取数据* ；

采取当前的方案，会从拿取元数据的过程中就开始加速；此功能需要测试；

7. listStatus

listStatus的设计和getFileStatus不同，当path在mustcachelist的时候，才会去Alluxio 的master中去访问元数据，当不在的时候，直接去Ufs中访问元数据；因为proxy的最终的一致性是以HDFS为准的；

不采用getFileStatus的方案是因为，当某个文件夹之后部分在Alluxio space中的时候，此时master中只有部分元数据；从而导致元数据的不一致性；此时应当直接从HDFS中获取元数据；

8. mkdirs
9. open
10. rename
11. load
12. 多副本的控制策略

## 附录

1. append的第二种方案，不删除Alluxio space中的文件，直接多HDFS的文件进行Append；

NOTE：

1. append的过程中，文件的读取是从Alluxio Space中读取；
2. 通知master端，此文件需要做consistency check，由master端定期check去保证append之后，文件的最终一致性；
3. 此方法的latency较大，对一个block来说，latency等于：append过程中完成一个block写的时间 + Alluxio Master check consistency的时间 + load meta的时间；　