---
layout:     post
title:      Elasticsearch snapshot & restore
subtitle:   es学习笔记
date:       2018-01-29
author:     zhiyuan.ma
header-img: img/post-bg-es.png
catalog: true
tags:
    - Elasticsearch
    - 数据处理
---

# Elasticsearch snapshot & restore

> **备份**

备份数据之前，要创建一个仓库来保存数据，仓库的类型支持共享文件系统、Amazon S3、 HDFS和Azure Cloud。 
 
Elasticsearch的一大特点就是使用简单，api也比较强大，备份也不例外。简单来说，备份分两步：

  1. 创建一个仓库
  2. 备份指定索引

## 一、创建仓库存储

共享文件系统实例如下：

```
curl -XPUT http://127.0.0.1:9200/_snapshot/EsBackup
{
    "type": "fs", 
    "settings": {
        "location": "/mount/EsDataBackupDir" 
    }
}
```

上面代码解释：

* 创建了一个名为EsBackup的存仓库
* 指定的备份方式为共享文件系统(type: fs)
* 指定共享存储的具体路径（location参数）

注意：共享存储路径，必须是所有的ES节点都可以访问的，最简单的就是nfs系统，然后每个节点都需要挂载到本地。
 
如上所示，创建存储仓库的时候，除了可以指定location参数以外，我们还可以指点**max_snapshot_bytes_per_sec**和**max_restore_bytes_per_sec**参数来限制备份和恢复时的速度，默认值都是20mb/s，假设我们有一个非常快的网络环境,我们可以增大默认值：

```
curl -XPOST http://127.0.0.1:9200/_snapshot/EsBackup
{
    "type": "fs", 
    "settings": {
        "location": "/mount/EsDataBackupDir" 
        "max_snapshot_bytes_per_sec" : "50mb", 
        "max_restore_bytes_per_sec" : "50mb"
    }
}
```

注意:这是在第一段代码的基础上来增加配置，第一段代码利用的是**PUT**请求来创建存储库，这段代码则是利用**POST**请求来更新已经存在的存储库的`settings`配置。 

**Amazon S3**存储库实例如下：

```
curl -XPUT 'http://localhost:9200/_snapshot/s3-backup' -d '{
    "type": "s3",
    "settings": {
        "bucket": "esbackup",
        "region": "cn-north-1",
        "access_key": "xxooxxooxxoo",
        "secret_key": "xxxxxxxxxooooooooooooyyyyyyyyy"
    }
}'
```

参数名词解释：

  1. Type: 仓库类型
  2. Setting: 仓库的额外信息
  3. Region: AWS Region
  4. Access_key: 访问秘钥
  5. Secret_key: 私有访问秘钥
  6. Bucket: 存储桶名称

不同的ES版本支持的region参考：[https://github.com/elastic/elasticsearch-cloud-aws#aws-cloud-plugin-for-elasticsearch](https://github.com/elastic/elasticsearch-cloud-aws#aws-cloud-plugin-for-elasticsearch)
 
使用上面的命令，创建一个仓库**（s3-backup）**，并且还创建了存储桶**（esbackup）**,返回`{"acknowledged":true}` 信息证明创建成功。
  
**确认存储桶是否创建成功：**
`curl -XPOST http://localhost:9200/_snapshot/s3-backup/_verify`
	
**查看刚创建的存储桶：**
`curl -XGET localhost:9200/_snapshot/s3-backup?pretty`

**查看所有的存储桶：**
`curl -XGET localhost:9200/_snapshot/_all?pretty`

**删除一个快照存储桶：**
`curl -XDELETE localhost:9200/_snapshot/s3-backup?pretty`
 
## 二、备份索引

创建好存储仓库之后就可以开始备份了。一个仓库可以包含多个快照（snapshots），快照可以存所有的索引或者部分索引，当然也可以存储一个单独的索引。(要注意的一点就是快照只会备份open状态的索引，close状态的不会备份)
 
**备份所有索引**
> curl -XPUT http://127.0.0.1:9200/_snapshot/EsBackup/snapshot_all

上面的代码会将所有正在运行的open状态的索引，备份到**EsBacup**仓库下一个叫**snapshot_all**的快照中。上面的api会立刻返回`{"accepted":true}`，然后备份工作在后台运行。如果你想api同步执行，可以加**wait_for_completion** 标志：
> curl -XPUT http://127.0.0.1:9200/_snapshot/EsBackup/snapshot_all?wait_for_completion=true

上面的方法会在备份完全完成后才返回，如果快照数据量大的话，会花很长时间。

**备份部分索引**

默认是备份所有open状态的索引，如果你想只备份某些或者某个索引，可以指定indices参数来完成：
> curl -XPUT 'http://localhost:9200/_snapshot/EsBackup/snapshot_12' -d '{ "indices": "index_1,index_2" }'

**查看快照信息**
查看快照信息，只需要发起GET请求就好：
> GET _snapshot/my_backup/snapshot_2

这将返回关于快照snapshot_2的详细信息：

```
{
   "snapshots": [
      {
         "snapshot": "snapshot_2",
         "indices": [
            ".marvel_2014_28_10",
            "index1",
            "index2"
         ],
         "state": "SUCCESS",
         "start_time": "2014-09-02T13:01:43.115Z",
         "start_time_in_millis": 1409662903115,
         "end_time": "2014-09-02T13:01:43.439Z",
         "end_time_in_millis": 1409662903439,
         "duration_in_millis": 324,
         "failures": [],
         "shards": {
            "total": 10,
            "failed": 0,
            "successful": 10
         }
      }
   ]
}
```

**查看所有快照信息如下：**
> GET http://127.0.0.1:9200/_snapshot/my_backup/_all

**另外还有个一api可以看到更加详细的信息：**
> GET http://127.0.0.1:9200/_snapshot/my_backup/snapshot_2/_status

更多详细内容可以到官网查看-[官方文档地址](https://www.elastic.co/guide/en/elasticsearch/guide/current/backing-up-your-cluster.html)。
 
**删除快照**
> DELETE _snapshot/my_backup/snapshot_2

重要的是使用API来删除快照,而不是其他一些机制(如手工删除,或使用自动s3清理工具)。因为快照增量,它是可能的,许多快照依靠old seaments。删除API了解最近仍在使用的数据快照,并将只删除未使用的部分。如果你手动文件删除,但是,你有可能严重破坏你的备份,因为你删除数据仍在使用,如果备份正在后台进行，也可以直接删除来取消此次备份。
 
**监控快照进展**

**wait_for_completion**标志提供了一个基本形式的监控,但没有足够的快照恢复甚至中等大小的集群。
 
另外两个api会给你更细节的状态的快照。首先你可以执行一个快照ID,就像我们早些时候得到一个特定的快照信息:
> GET _snapshot/my_backup/snapshot_3

如果当你调用这个快照还在进行,你会看到信息的时候开始,已经运行多长时间,等等。但是请注意,这个API使用相同的threadpool快照机制。如果你是快照非常大的碎片,之间的时间状态更新可以相当大,因为API是争夺相同的threadpool资源。
 
这时候有个更好的选择**_status**的api接口：
> GET _snapshot/my_backup/snapshot_3/_status

**_status** API立即返回并给出一个更详细的输出的统计:

```
{
   "snapshots": [
      {
         "snapshot": "snapshot_3",
         "repository": "my_backup",
         "state": "IN_PROGRESS", 
         "shards_stats": {
            "initializing": 0,
            "started": 1, 
            "finalizing": 0,
            "done": 4,
            "failed": 0,
            "total": 5
         },
         "stats": {
            "number_of_files": 5,
            "processed_files": 5,
            "total_size_in_bytes": 1792,
            "processed_size_in_bytes": 1792,
            "start_time_in_millis": 1409663054859,
            "time_in_millis": 64
         },
         "indices": {
            "index_3": {
               "shards_stats": {
                  "initializing": 0,
                  "started": 0,
                  "finalizing": 0,
                  "done": 5,
                  "failed": 0,
                  "total": 5
               },
               "stats": {
                  "number_of_files": 5,
                  "processed_files": 5,
                  "total_size_in_bytes": 1792,
                  "processed_size_in_bytes": 1792,
                  "start_time_in_millis": 1409663054859,
                  "time_in_millis": 64
               },
               "shards": {
                  "0": {
                     "stage": "DONE",
                     "stats": {
                        "number_of_files": 1,
                        "processed_files": 1,
                        "total_size_in_bytes": 514,
                        "processed_size_in_bytes": 514,
                        "start_time_in_millis": 1409663054862,
                        "time_in_millis": 22
                     }
                  },
                  ...
```

快照当前运行将显示IN_PROGRESS作为其状态，这个特定的快照有一个碎片仍然转移(其他四个已经完成)。
 
响应包括总体状况的快照,但还深入每和每个实例统计数据。这给你一个令人难以置信的详细视图快照是如何进展的。碎片可以以不同的方式完成:

**INITIALIZING：**集群的碎片是检查状态是否可以快照。这通常是非常快。

**STARTED：**数据被转移到存储库。

**FINALIZING：**数据传输完成;碎片现在发送快照的元数据。

**DONE：**快照完成。

**FAILED：**在快照过程中错误的出处,这碎片/索引/快照无法完成。检查你的日志以获取更多信息。

**恢复**

备份好后，恢复就更容易了，恢复**snapshot_1**里的全部索引：
> POST http://127.0.0.1:9200/_snapshot/my_backup/snapshot_1/_restore

这个api还有额外的参数：

```
POST http://127.0.0.1:9200/_snapshot/my_backup/snapshot_1/_restore
{
    "indices": "index_1", 
    "rename_pattern": "index_(.+)", 
    "rename_replacement": "restored_index_$1" 
}
```

参数`indices`设置只恢复index_1索引，参数`rename_pattern`和`rename_replacement`用来正则匹配要恢复的索引，并且重命名。和备份一样，api会立刻返回值，然后在后台执行恢复，使用**wait_for_completion** 标记强制同步执行。
 
另外可以使用下面两个api查看状态：

```
GET http://127.0.0.1:9200/_recovery/restored_index_3
GET http://127.0.0.1:9200/_recovery/
```

如果要取消恢复过程（不管是已经恢复完，还是正在恢复），直接删除索引即可：
> DELETE http://127.0.0.1:9200/restored_index_3

更多内容参考-[官方文档](https://www.elastic.co/guide/en/elasticsearch/guide/current/_restoring_from_a_snapshot.html)。
 
**参考官方文档地址：**

1、[https://www.elastic.co/guide/en/elasticsearch/guide/current/backing-up-your-cluster.html#_listing_information_about_snapshots](https://www.elastic.co/guide/en/elasticsearch/guide/current/backing-up-your-cluster.html#_listing_information_about_snapshots)
2、[https://www.elastic.co/guide/en/elasticsearch/guide/current/_restoring_from_a_snapshot.html](https://www.elastic.co/guide/en/elasticsearch/guide/current/_restoring_from_a_snapshot.html)