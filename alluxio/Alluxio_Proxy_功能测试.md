# Alluxio Proxy的功能测试

本文档是对Alluxio的所有的功能接口的测试；文档的组织方式为按照各个接口，逐一测试，输出测试报告；

测试说明：

| File/Dir | UserMustCacheList | Load | Alluxio | HDFS |
| -------- | ----------------- | ---- | ------- | ---- |
| Path     | Yes               | -    | Yes     | No   |
| Path     | No                | No   | No      | Yes  |
| Path     | No                | Yes  | Yes     | Yes  |

说明：UserMustCacheList是可以通过client的参数`alluxio.user.mustCacheList`进行配置；

Load：是指文件从HDFS Space中load到Alluxio Space中；

Alluxio：是指文件存在在Alluxio Space中；

HDFS：是指文件存在在HDFS Space中；

## Metadata Separate

Alluxio是否从HDFS中load数据可以通过Master端的参数`alluxio.master.load.metadata.from.ufs.enable`来控制，默认值是`true`, 是社区的功能，不需要进行测试；

接下来，对其为`false`的情况进行测试；此参数为Master端参数，所以修改后`alluxio-stop.sh all -> alluxio format ->alluxio-start.sh all`;

| File/Dir | UserMustCacheList | Load | Expect Result                            | Test Result |
| -------- | ----------------- | ---- | ---------------------------------------- | ----------- |
| Path     | Yes               | -    | Metadata in Alluxio, not in HDFS         |             |
| Path     | No                | No   | Metadata not in Alluxio, while in HDFS   |             |
| Path     | No                | Yes  | Metadata both in Alluxio(with ufsPath) and HDFS |             |

测试结果观测说明：

| HDFS shell ls | Alluxio shell ls | Conclusions                            |
| ------------- | ---------------- | -------------------------------------- |
| No            | Yes              | Metadata in Alluxio, not in HDFS       |
| Yes           | No               | Metadata not in Alluxio, while in HDFS |
| Yes           | Yes              | Metadata in Alluxio and HDFS           |

针对最后一种情况，需要通过API去Alluxio Master中`getStatus`,然后打印出Ufs的信息；

## Mkdirs

1. path not Exist

|        | UserMustCacheList | Expect Result                            | Test Result |
| ------ | ----------------- | ---------------------------------------- | ----------- |
| mkdirs | Yes               | In Alluxio Space, not in HDFS space      |             |
| mkdirs | No                | Not in Alluxio space, while in HDFS space |             |

1. path exists: 

<font color = red>Expect result: throw exception</font>;

<font color = blue>Test result:  </font>

## Append

1. File exists：

|        | HDFS | Alluxio | Expect Resutl                            | Test Result |
| ------ | ---- | ------- | ---------------------------------------- | ----------- |
| Append | Yes  | No      | Through to HDFS Append and success       | Yes         |
| Append | Yes  | Yes     | Delete in Alluxio, and through to UFS append | Yes         |
| Append | No   | Yes     | throw exception (alluxio does not support append) | Yes         |

2. File not exist:

|        | UserMustCacheList | Expect Result                 | Test Result |
| ------ | ----------------- | ----------------------------- | ----------- |
| Append | Yes               | Alluxio create file and write |             |
| Append | No                | HDFS create File and write    |             |

## Create

1. path not Exist

|        | UserMustCacheList | Expect Result                            | Test Result |
| ------ | ----------------- | ---------------------------------------- | ----------- |
| create | Yes               | In Alluxio Space, not in HDFS space      |             |
| create | No                | Not in Alluxio space, while in HDFS space |             |

2. path exists (测试Alluxio Space，path 应该在UserMustCacheList中)

|        | Recursive | isDirectory | Expect Result   | Test Result |
| ------ | --------- | ----------- | --------------- | ----------- |
| create | false     | -           | throw Exception |             |
| create | true      | false       | create success  |             |
| create | true      | true        | throw Exception |             |

## Delete

1. File exists, 正常功能的测试

|        | HDFS | Alluxio | Expect Result                            | Test Result |
| ------ | ---- | ------- | ---------------------------------------- | ----------- |
| delete | Yes  | No      | Through to HDFS delete and success       |             |
| delete | Yes  | Yes     | delete in Alluxio Space first, and then in HDFS |             |
| delete | No   | Yes     | delete in Alluxio Space                  |             |

2. File not exist

|        | UserMustCacheList | Expect Result                      | Test Result |
| ------ | ----------------- | ---------------------------------- | ----------- |
| delete | Yes               | Print "Alluxio Space FileNotFound" |             |
| delete | No                | Print "HDFS space FileNotFound"    |             |

3. 异常测试

delete in Alluxio Space Failed，是否还回去Ufs中delete；

在写的时候，尝试去delete；这样可以制造出delete failed的情形；

## get

1. File exists

|      | HDFS | Alluxio | Expect Result           | Test Result |
| ---- | ---- | ------- | ----------------------- | ----------- |
| get  | Yes  | No      | get From HDFS namenode  |             |
| get  | -    | Yes     | get From Alluxio Master |             |

2. File not exist

|      | UserMustCacheList | Expect Result                          | Test Result |
| ---- | ----------------- | -------------------------------------- | ----------- |
| get  | Yes               | throw exception (alluxio space failed) |             |
| get  | No                | throw exception (hdfs space failed)    |             |

测试说明： 上面的测试用例可以用于 getFileBlockLocations、getFileStatus、listStatus；

## setOwner && setPermission

1. File exists

|      | HDFS | Alluxio | Expect Result                | Test Result |
| ---- | ---- | ------- | ---------------------------- | ----------- |
| set  | Yes  | No      | set in HDFS Space            |             |
| set  | Yes  | Yes     | set both in HDFS and Alluxio |             |
| set  | No   | Yes     | set in Alluxio space         |             |

2. File not exist

|      | UserMustCacheList |                                          |      |
| ---- | ----------------- | ---------------------------------------- | ---- |
| set  | Yes               | throw Exception(alluxio space not found) |      |
| set  | No                | throw Exception(hdfs space not found)    |      |