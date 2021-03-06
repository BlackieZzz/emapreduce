# Tranquility {#concept_t2g_jqd_z2b .concept}

本文以 Kafka 为例，介绍在 E-MapReduce 中如何使用 Tranquility 从 Kafka 集群采集数据，并实时推送至 E-MapReduce Druid 集群。

Tranquility 是一个以 push 方式向 E-MapReduce Druid 实时发送数据的应用。它替用户解决了分区、多副本、服务发现、防止数据丢失等多个问题，简化了用户使用 E-MapReduce Druid 的难度。它支持多种数据来源，包括 Samza、Spark、Storm、Kafka、Flink 等等。

## 与 Kafka 集群交互 {#section_xbp_tsd_z2b .section}

首先是 E-MapReduce Druid 集群与 Kafka 集群的交互。两个集群交互的配置方式大体和 Hadoop 集群类似，均需要设置连通性、hosts 等。对于非安全 Kafka 集群，请按照以下步骤操作：

1.  确保集群间能够通信（两个集群在一个安全组下，或两个集群在不同安全组，但两个安全组之间配置了访问规则）。
2.  将 Kafka 集群的 hosts 写入到 E-MapReduce Druid 集群每一个节点的 hosts 列表中，注意 Kafka 集群的 hostname 应采用长名形式，如 emr-header-1.cluster-xxxxxxxx。

对于安全的 Kafka 集群，您需要执行下列操作（前两步与非安全 Kafka 集群相同）：

1.  确保集群间能够通信（两个集群在一个安全组下，或两个集群在不同安全组，但两个安全组之间配置了访问规则）。
2.  将 Kafka 集群的 hosts 写入到 E-MapReduce Druid 集群每一个节点的 hosts 列表中，注意 Kafka 集群的 hostname 应采用长名形式，如 emr-header-1.cluster-xxxxxxxx。
3.  设置两个集群间的 Kerberos 跨域互信（详情参考[跨域互信](intl.zh-CN/开源组件介绍/Kerberos认证/跨域互信.md#)），且最好做双向互信。
4.  准备一个客户端安全配置文件：

    ``` {#codeblock_gak_99v_yja}
    KafkaClient {
          com.sun.security.auth.module.Krb5LoginModule required
          useKeyTab=true
          storeKey=true
          keyTab="/etc/ecm/druid-conf/druid.keytab"
          principal="druid@EMR.1234.COM";
      };
    ```

    之后将该配置文件同步到 E-MapReduce Druid 集群的所有节点上，放置于某一个目录下面（例如/tmp/kafka/kafka\_client\_jaas.conf）。

5.  在 E-MapReduce Druid 配置页面的 overlord.jvm 里新增如下选项：

    ``` {#codeblock_cjk_cm2_7hm}
    Djava.security.auth.login.config=/tmp/kafka/kafka_client_jaas.conf
    ```

    。

6.  在 E-MapReduce Druid 配置页面的 middleManager.runtime 里配置`druid.indexer.runner.javaOpts=-Djava.security.auth.login.confi=/tmp/kafka/kafka_client_jaas.conf`和其他 JVM 启动参数。
7.  重启Druid服务。

## Druid使用Tranquility Kafka {#section_q4h_ctd_z2b .section}

由于 Tranquility 是一个服务，它对于 Kafka 来说是消费者，对于 E-MapReduce Druid 来说是客户端。您可以使用中立的机器来运行 Tranquility，只要这台机器能够同时连通 Kafka 集群和 E-MapReduce Druid 集群即可。

1.  Kafka 端创建一个名为 pageViews 的 topic。

    ``` {#codeblock_ajl_v65_3fi}
    -- 如果开启了 kafka 高安全：
     export KAFKA_OPTS="-Djava.security.auth.login.config=/etc/ecm/kafka-conf/kafka_client_jaas.conf"
     --
     ./bin/kafka-topics.sh --create --zookeeper emr-header-1:2181,emr-header-2:2181,emr-header-3:2181/kafka-1.0.1 --partitions 1 --replication-factor 1 --topic pageViews
    ```

2.  下载 Tranquility 安装包，并解压至某一路径下。
3.  配置 datasource。

    这里假设您的 topic name 为 pageViews，并且每条 topic 都是如下形式的 json 文件：

    ``` {#codeblock_r12_9w8_ovl}
    {"time": "2018-05-23T11:59:43Z", "url": "/foo/bar", "user": "alice", "latencyMs": 32}
     {"time": "2018-05-23T11:59:44Z", "url": "/", "user": "bob", "latencyMs": 11}
     {"time": "2018-05-23T11:59:45Z", "url": "/foo/bar", "user": "bob", "latencyMs": 45}
    ```

    对应的 dataSrouce 的配置如下：

    ``` {#codeblock_gz0_6rl_zkw}
    {
       "dataSources" : {
         "pageViews-kafka" : {
           "spec" : {
             "dataSchema" : {
               "dataSource" : "pageViews-kafka",
               "parser" : {
                 "type" : "string",
                 "parseSpec" : {
                   "timestampSpec" : {
                     "column" : "time",
                     "format" : "auto"
                   },
                   "dimensionsSpec" : {
                     "dimensions" : ["url", "user"],
                     "dimensionExclusions" : [
                       "timestamp",
                       "value"
                     ]
                   },
                   "format" : "json"
                 }
               },
               "granularitySpec" : {
                 "type" : "uniform",
                 "segmentGranularity" : "hour",
                 "queryGranularity" : "none"
               },
               "metricsSpec" : [
                 {"name": "views", "type": "count"},
                 {"name": "latencyMs", "type": "doubleSum", "fieldName": "latencyMs"}
               ]
             },
             "ioConfig" : {
               "type" : "realtime"
             },
             "tuningConfig" : {
               "type" : "realtime",
               "maxRowsInMemory" : "100000",
               "intermediatePersistPeriod" : "PT10M",
               "windowPeriod" : "PT10M"
             }
           },
           "properties" : {
             "task.partitions" : "1",
             "task.replicants" : "1",
             "topicPattern" : "pageViews"
           }
         }
       },
       "properties" : {
         "zookeeper.connect" : "localhost",
         "druid.discovery.curator.path" : "/druid/discovery",
         "druid.selectors.indexing.serviceName" : "druid/overlord",
         "commit.periodMillis" : "15000",
         "consumer.numThreads" : "2",
         "kafka.zookeeper.connect" : "emr-header-1.cluster-500148518:2181,emr-header-2.cluster-500148518:2181,   emr-header-3.cluster-500148518:2181/kafka-1.0.1",
         "kafka.group.id" : "tranquility-kafka",
       }
     }
    ```

4.  运行如下命令启动 Tranquility。

    ``` {#codeblock_uyo_dgy_q90}
    ./bin/tranquility kafka -configFile 
    ```

5.  在 Kafka 端启动 producer 并发送一些数据。

    ``` {#codeblock_i4q_d1i_vq7}
    ./bin/kafka-console-producer.sh --broker-list emr-worker-1:9092,emr-worker-2:9092,emr-worker-3:9092 --topic pageViews
    ```

    输入：

    ``` {#codeblock_fkc_ylj_gse}
    {"time": "2018-05-24T09:26:12Z", "url": "/foo/bar", "user": "alice", "latencyMs": 32}
     {"time": "2018-05-24T09:26:13Z", "url": "/", "user": "bob", "latencyMs": 11}
     {"time": "2018-05-24T09:26:14Z", "url": "/foo/bar", "user": "bob", "latencyMs": 45}
    ```

    在 Tranquility 日志中查看相应的消息，在 E-MapReduce Druid 端则可以看到启动了相应的实时索引 task。


