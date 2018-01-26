---
layout: post
title: mongo-connector使用
category: 存储系统
tags: mongodb
---

> mongo-connector creates a pipeline from a MongoDB cluster to one or more target systems, such as Solr, Elasticsearch, or another MongoDB cluster. It synchronizes data in MongoDB to the target then tails the MongoDB oplog, keeping up with operations in MongoDB in real-time.

一个从Mongodb导出数据的工具，类似于阿里的DataX，由于是大批量消息的传递，一般都是通过日志的方式，与mysql的同步很像。有一点疑问的，项目的说明上说：

> mongo-connector replicates operations from the MongoDB oplog, so a replica set must be running before startup. 

如果没有replica set就没有这oplog么？看了下官网上对oplog的介绍：

> The oplog (operations log) is a special capped collection that keeps a rolling record of all operations that modify the data stored in your databases. MongoDB applies database operations on the primary and then records the operations on the primary’s oplog. The secondary members then copy and apply these operations in an asynchronous process.

Mysql数据库，即使没有slave数据库，也会有binlog的，这binlog有两个作用，一是可以从master数据库保持同步，二是在数据库损坏的情况下可以根据binlog恢复数据库，而且我看到的资料，binlog只对性能有1%的损耗。看上面的对Mongodb的oplog的介绍，一点也没说从错误中恢复的事情，看来这oplog就是没打算用在这方面了，因为加了这oplog按照Mysql的数据，对性能损耗也大不了多少，关键是了解的系统如Hbase、ES等都有个log的。虽然这oplog没有这功能，但是，Mongodb其实是有个单独的机制的，叫[journaling](https://docs.mongodb.com/manual/core/journaling/)。那为什么没有像Mysql这样，用同一个log呢？待研究。

### Convert a Standalone to a Replica Set ###

上面说的，要把Mongodb搞个Replica set，可以参考[Convert a Standalone to a Replica Set](https://docs.mongodb.com/manual/tutorial/convert-standalone-to-replica-set/)，先再/home/mongodb/data下创建两个文件夹作为数据库存储的路径db0和db1，然后后台启动一个

```
nohup mongod --port 27017 --dbpath /home/mongodb/data/db0 --replSet rs0 &
```

然后进入控制台，初始化：

```
rs.initiate()
```

当然没那么顺利，code码93报错信息：
> FailedToParse: Empty host component parsing HostAndPort from \":27017\" for member:{ _id: 0, host: \":27017\" }

Mongodb的配置问题，ip配置的问题，Mongodb默认也是没有配置文件的，需要手动加个，直接放在etc下吧，源码安装的，是有个默认的配置的，名字叫mongod.conf可以find下拷贝到etc下,然后编辑下，直接把那行“bindIp: 127.0.0.1”加上个本机ip，改成“bindIp: 10.xx.xx.xx,127.0.0.1”,kill掉在启动

```
nohup mongod --port 27017 --dbpath /home/mongodb/data/db0 --config /etc/mongod.conf --replSet rs0 &
```

继续报错，“child process failed, exited with error number 1”，因为日志目录不存在，新建日志目录，修改配置文件，运行，还是报这个错，不过这次有日志文件了，说pid目录不存在，手动创建下，发现有fork的方式，那么可以不用nohup后台启动了，直接：
```
mongod --port 27017 --dbpath /home/mongodb/data/db0 --config /etc/mongod.conf --replSet rs0 --fork
```

终于可以了“child process started successfully, parent exiting”。然后继续初始化：

```
rs.initiate()
{
        "info2" : "no configuration specified. Using a default configuration for the set",
        "me" : "10.45.157.55:27017",
        "ok" : 1,
        "operationTime" : Timestamp(1510983316, 1),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1510983316, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}

```

说明成功了，看下默认的配置到底啥样：

```
rs.config()
{
        "_id" : "rs0",
        "version" : 1,
        "protocolVersion" : NumberLong(1),
        "members" : [
                {
                        "_id" : 0,
                        "host" : "10.45.157.55:27017",
                        "arbiterOnly" : false,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,
                        "tags" : {
                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                }
        ],
        "settings" : {
                "chainingAllowed" : true,
                "heartbeatIntervalMillis" : 2000,
                "heartbeatTimeoutSecs" : 10,
                "electionTimeoutMillis" : 10000,
                "catchUpTimeoutMillis" : -1,
                "catchUpTakeoverDelayMillis" : 30000,
                "getLastErrorModes" : {
                },
                "getLastErrorDefaults" : {
                        "w" : 1,
                        "wtimeout" : 0
                },
                "replicaSetId" : ObjectId("5a0fc69408b20c997aedcd96")
        }
}

```

可以看下rs的状态：

```
rs.status()
{
        "set" : "rs0",
        "date" : ISODate("2017-11-18T05:41:45.383Z"),
        "myState" : 1,
        "term" : NumberLong(1),
        "heartbeatIntervalMillis" : NumberLong(2000),
        "optimes" : {
                "lastCommittedOpTime" : {
                        "ts" : Timestamp(1510983698, 1),
                        "t" : NumberLong(1)
                },
                "readConcernMajorityOpTime" : {
                        "ts" : Timestamp(1510983698, 1),
                        "t" : NumberLong(1)
                },
                "appliedOpTime" : {
                        "ts" : Timestamp(1510983698, 1),
                        "t" : NumberLong(1)
                },
                "durableOpTime" : {
                        "ts" : Timestamp(1510983698, 1),
                        "t" : NumberLong(1)
                }
        },
        "members" : [
                {
                        "_id" : 0,
                        "name" : "10.45.157.55:27017",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 399,
                        "optime" : {
                                "ts" : Timestamp(1510983698, 1),
                                "t" : NumberLong(1)
                        },
                        "optimeDate" : ISODate("2017-11-18T05:41:38Z"),
                        "electionTime" : Timestamp(1510983316, 2),
                        "electionDate" : ISODate("2017-11-18T05:35:16Z"),
                        "configVersion" : 1,
                        "self" : true
                }
        ],
        "ok" : 1,
        "operationTime" : Timestamp(1510983698, 1),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1510983698, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}
```

### 使用mongo-connector ###

安装的过程很顺利，直接:

```
pip install mongo-connector
```

然后mongo-connector -h就可以看到支持的参数了，mongo-connector项目上的简单介绍：

```
mongo-connector -m <mongodb server hostname>:<replica set port> 
                -t <replication endpoint URL, e.g. http://localhost:8983/solr> 
                -d <name of doc manager, e.g., solr_doc_manager>
```

多加个自动提交的时间，为了测试快点，运行：

```
mongo-connector --auto-commit-interval=0 -m 127.0.0.1:27017 -t 10.72.76.140:9200 -d elastic2_doc_manager
```

然后查看es的信息
```
http://10.72.76.140:9200/wzy/col/_search
```

已经有对应的内容了，在mongo-connector服务不可用期间Mongodb新增的内容，在mongo-connector恢复后可以继续同步。当然生产环境，--auto-commit-interval肯定不能是0了，否则性能会严重下降。

与Mongodb一样，生产环境需要个配置文件，pip安装的也没找到，直接从github上找到默认的配置文件，然后修改下放到etc下，下次运行就可以直接使用带一个配置文件的命令了：

```
mongo-connector -c /etc/mongo-connector.conf
```

配置文件内容如下：

```
{
    "__comment__": "Configuration options starting with '__' are disabled",
    "__comment__": "To enable them, remove the preceding '__'",

    "mainAddress": "localhost:27017",
    "oplogFile": "/var/log/mongo-connector/oplog.timestamp",
    "noDump": false,
    "batchSize": -1,
    "verbosity": 0,
    "continueOnError": false,

    "logging": {
        "type": "file",
        "filename": "/var/log/mongo-connector/mongo-connector.log",
        "__format": "%(asctime)s [%(levelname)s] %(name)s:%(lineno)d - %(message)s",
        "__rotationWhen": "D",
        "__rotationInterval": 1,
        "__rotationBackups": 10,

        "__type": "syslog",
        "__host": "localhost:514"
    },

    "authentication": {
        "__adminUsername": "username",
        "__password": "password",
        "__passwordFile": "mongo-connector.pwd"
    },

    "__comment__": "For more information about SSL with MongoDB, please see http://docs.mongodb.org/manual/tutorial/configure-ssl-clients/",
    "__ssl": {
        "__sslCertfile": "Path to certificate to identify the local connection against MongoDB",
        "__sslKeyfile": "Path to the private key for sslCertfile. Not necessary if already included in sslCertfile.",
        "__sslCACerts": "Path to concatenated set of certificate authority certificates to validate the other side of the connection",
        "__sslCertificatePolicy": "Policy for validating SSL certificates provided from the other end of the connection. Possible values are 'required' (require and validate certificates), 'optional' (validate but don't require a certificate), and 'ignored' (ignore certificates)."
    },

    "__fields": ["field1", "field2", "field3"],

    "__namespaces": {
        "excluded.collection": false,
        "excluded_wildcard.*": false,
        "*.exclude_collection_from_every_database": false,
        "included.collection1": true,
        "included.collection2": {},
        "included.collection4": {
            "includeFields": ["included_field", "included.nested.field"]
        },
        "included.collection5": {
            "rename": "included.new_collection5_name",
            "includeFields": ["included_field", "included.nested.field"]
        },
        "included.collection6": {
            "excludeFields": ["excluded_field", "excluded.nested.field"]
        },
        "included.collection7": {
            "rename": "included.new_collection7_name",
            "excludeFields": ["excluded_field", "excluded.nested.field"]
        },
        "included_wildcard1.*": true,
        "included_wildcard2.*": true,
        "renamed.collection1": "something.else1",
        "renamed.collection2": {
            "rename": "something.else2"
        },
        "renamed_wildcard.*": {
            "rename": "new_name.*"
        },
        "gridfs.collection": {
            "gridfs": true
        },
        "gridfs_wildcard.*": {
            "gridfs": true
        }
    },

    "docManagers": [
        {
            "docManager": "elastic2_doc_manager",
            "targetURL": "10.72.76.140:9200",
            "__bulkSize": 1000,
            "__uniqueKey": "_id",
            "__autoCommitInterval": null
        }
    ]
}
```

### 使用中遇到的问题 ###

使用monggo-connector从mongo同步到es中遇到两个问题：

#### 1. 同步到ES的过程中，monggo-connector报错： ####

> 2018-01-09 21:30:41,808 [ERROR] mongo_connector.oplog_manager:202 - OplogThread: Last entry no longer in oplog cannot recover! Collection(Database(MongoClient(host=[u'localhost:27017'], document_class=dict, tz_aware=False, connect=True, replicaset=u'rs0'), u'local'), u'oplog.rs')  
> 2018-01-09 21:30:42,165 [ERROR] mongo_connector.connector:398 - MongoConnector: OplogThread <OplogThread(Thread-3, stopped 140514745640704)unexpectedly stopped! Shutting down

#### 2. 对第一个问题按照mongo的git的wiki上的解答[Resyncing the Connector](https://github.com/mongodb-labs/mongo-connector/wiki/Resyncing-the-Connector)处理后，发现monggo-connector中记录同步进度的文件(oplog process file)里面一直没有数据 ####

下面分别对下面两个问题进行分析：

### 问题分析:同步到ES的过程中，monggo-connector报错 ###
参考[Resyncing the Connector](https://github.com/mongodb-labs/mongo-connector/wiki/Resyncing-the-Connector)，引起问题的原因是mongo的oplog是个[capped collection](https://docs.mongodb.com/manual/core/capped-collections/)，一个固定大小的集合，超过这个大小就会覆盖以前的记录。当oplog相对较小，后者mongo的插入速度一直大于从mongo到es的同步速度的时候，就会发生这个问题。解决办法么，就是给个相对较大的oplog的size。

另外，如果要重新同步数据，git的wiki中的步骤是需要先删除es中的数据，重新的index一遍。删除es中的数据后，把monggo-connectorde的oplog process file删除。这样monggo-connector会发现进度文件没有，会新创建文件，并进行一次collection-dump操作，把整张表dump到es中，然后从dump后的时间根据mongo的oplog继续同步。

如果只是有indert和update操作，没有delete操作，也可以不用删除es中的数据，直接删除oplog process file，如果有删除操作，collection-dump操作并不知道哪些数据在之前es中的被删除掉了，这样collection-dump后，es中仍然后之前被删除掉的数据。

### 问题分析：oplog process file里面一直没有数据 ###
这不是一个问题。按照[Collection Dumps and the Oplog Progress File](https://github.com/mongodb-labs/mongo-connector/wiki/Oplog-Progress-File#collection-dumps-and-the-oplog-progress-file)中的描述：

> When the oplog progress file cannot be found, or if it is empty, Mongo Connector will begin pulling data from all MongoDB collections (or the ones given in --namespace-set) in the "collection dump" phase. The oplog progress file is then updated with the most recent timestamp from before the dump happened. 

由于要保持数据的同步，而oplog大小有限，对于一个长时间运行的系统，oplog一般是不全的，所以操作如下，记录下当前时间t1，然后全collection拷贝，拷贝完后，从t1时间的oplog进行后面的操作同步。monggo-connector的代码如下：

```
def upsert_all(dm):
    try:
        for namespace in dump_set:
            from_coll = self.get_collection(namespace)
            total_docs = retry_until_ok(from_coll.count)
            mapped_ns = self.namespace_config.map_namespace(
                    namespace)
            LOG.debug("Bulk upserting approximately %d docs from "
                     "collection '%s'",
                     total_docs, namespace)
            dm.bulk_upsert(docs_to_dump(from_coll),
                           mapped_ns, long_ts)
    except Exception:
        if self.continue_on_error:
            LOG.exception("OplogThread: caught exception"
                          " during bulk upsert, re-upserting"
                          " documents serially")
            upsert_each(dm)
        else:
            raise
```

上面的dm参数是docManager，即使写成debug的级别的日志，能看到一批批的数据插入到es，但是没有进度信息。而问题发现的时候有3亿条数据，ES的插入速度又比较慢，过了1天多的时间Oplog Progress File才有数据。

### 参考 ###
[mongo-connector的Readme](https://github.com/mongodb-labs/mongo-connector)