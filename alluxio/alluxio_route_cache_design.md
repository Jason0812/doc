# Alluxio Route and Cache feature design

Alluxio的路由功能已经开发完毕，现在处于测试阶段；功能已经调通；@201705015完成所有的功能测试；

将Alluxio和Ufs的分离功能分析设计完成，预计在@201705015完成开发工作；

本篇文档是在上述两个功能实现之后的，统一Alluxio作为Route和Cache的功能；预计在@20170519日完成开发和功能测试工作；

## 对Cache使用场景的限定

Alluxio的使用场景是：一份数据多次使用，将具备这样特性的数据cache在内存中，从而提升计算时的IO性能；

有些数据是暂时的文件(过期删除)，同时此类文件数量较多，如History log，可以将此类文件仅Cache在Alluxio Space中；

1. 针对需要频繁Append的数据，不允许将此类数据Cache在Alluxio Space中；
2. 目前Append 方法用的最多的是Flume，和Flume Sync，从而确定是否将此类数据放到Alluxio Space 中；
3. 在create的时候，只是支持将UserMustCacheList中的路径写入到Alluxio Space中，其他的都写入的HDFS中；
4. 对于热数据，都是通过手动load的方法去cache数据；所以要为此类cache添加一个方法；



## 各个功能的详细设计

1. 参数的设计：在Client端增加两个参数，alluxio.user.route.enabled = true(default); alluxio.user.cache.enabled = true(default); 默认参数设置的含义：Alluxio 在我们的环境中，一定是具备Route功能的；
2. 将client的调用分发到Alluxio Server和HDFS server中执行，是通过Proxy这层去做的；
3. 如果只打开Route功能，proxy将所有的操作转发给UFS；
4. 路由缓存功能采取的是最终一致性模型，以HDFS的数据为准；
5. 接下来的分析设计是针对同时打开Route和cache功能的；

### Append

1. 由于Append这个操作在Alluxio Space中是不支持的；所以proxy只能将Append的操作，转发到HDFS中；

NOTE：

1. 如果Append的文件同时存在在Alluxio Space和HDFS space中，在Append之后会造成数据的不一致性；
2. 由于HDFS在Append的过程中，其他的读的进程，会读取到部分Block的数据，但是不会读取到正在Append的这个block的数据；HDFS在Append的时候，是最终一致性模型；
3. *解决办法*： Proxy层在Hdfs append之后，主动的去free(还是delete)掉该文件在Alluxio Space中的数据；在free之后，在下次读取之时，将此数据加载(还是创建)到Alluxio Space中去(如何加载)，用户不能通过Alluxio去加载ufs的数据，只能通过我们自己去加载数据；在free和delete的时候的并发问题；
4. 通过使用场景来进行限定；

### Create

create的时候，根据WriteType(Through)和userMustCacheList可以分为以下几种情况：

| WriteType         | isUserMustCacheList | Alluxio Space | HDFS Space |
| ----------------- | ------------------- | ------------- | ---------- |
| Through (default) | Yes                 | Yes           | No         |
| Through (default) | No                  | No            | Yes        |
| Must_cache        | --                  | Yes           | No         |
| Cache_Through     | Yes                 | Yes           | No         |
| Cache_Through     | No                  | No            | Yes        |

通过可以总结为如下几种情况：

1. 当path在UserMustCacheList中或者WriteType为Must_Cache，proxy仅会将create转发到Alluxio Space中；
2. 当path不在UserMustCacheList中，WriteType为Through和Cache_through的时候，Proxy仅会将Create转发到HDFS Space中；

NOTE：当我们将Ufs中的数据加载到Alluxio Space中的时候，不能通过这个create 方法来实现；可以为Alluxio添加一个loadDataFromUfs(应当用Alluxio的Worker直接去load数据，同社区版本的Alluxio load的实现，worker的选取机制);

### Delete

因为最终是以HDFS的数据为准，所以在delete的时候，先去Alluxio Space中delete数据；成功后，再去HDFS中delete数据；

NOTE：

1. 当Alluxio Space中的数据delete不成功，则throw exception，终止程序执行；
2. 当Alluxio Space delete成功之后，HDFS中的delete失败之后的数据一致性问题，则交给了HDFS，跟当前的HDFS的做法一样；

### getFileBlockLocations && getFileStatus && listStatus

上述几个操作都是获取File和Block信息的查询接口；对于查询接口，Proxy会先去Alluxio Master中查询当前需要查询的路径是否在Alluxio Master中，如果在Alluxio Space中，则转发给Alluxio Space中；

如果数据不在Alluxio Space中，proxy 则直接转发给HDFS；去HDFS namenode中查询数据；

### mkdir

同create操作一样，考虑writeType和UserMustCacheList；

### setOwner && setGroup && .etc.

采取的一致性模型是最终是以HDFS为准，所以此类操作，先修改Alluxio Space的元数据，然后再去HDFS中修改元数据；

如果Alluxio Space中的元数据修改失败，则throw exception，程序终止执行；

如果Alluxio Space中的元数据修改成功，但HDFS中的元数据修改失败，此时应当查询HDFS中的元数据信息，将Alluxio中的元数据进行重新设定；然后报错，返回；

NOTE：当HDFS修改之后，是否要删除缓存，然后重新加载；

### Open

在open的时候，Proxy层会根据readtype和isUserMustCacheList去转发open操作；

1. readtype在open的时候不起作用；因为alluxio Space中的数据是通过手动加载进去的；
2. 不支持readCache操作，即将远程的Alluxio Space的数据，读到本地的Alluxio Space然后缓存起来；因为这样会造成Alluxio Space空间的使用不可控；
3. 当open的路径是UserMustCacheList路径，proxy则不转发到HDFS 中；
4. open先去查询要读的路径是否在Alluxio Space中，如果在Alluxio Space中，Proxy则将open转发到Alluxio Space中，否则转发到Hdfs Space中；