# SPARK 监控 {#concept_rln_ngt_qgb .concept}

SPARK 监控概览页面包括集群 SPARK 近期告警与异常、SparkHistory 进程 JVM 指标、SparkHistory 进程的文件描述符信息和 SparkHistory 进程启停历史。

## SPARK 服务最近告警和异常 {#section_nvr_1ht_qgb .section}

![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123487/155781662538665_zh-CN.png)

默认展示该集群最近一天的告警和异常事件。

## SparkHistory 进程的 JVM 指标 {#section_s3l_3ht_qgb .section}

![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123487/155781663438666_zh-CN.png)

详细监控指标项和含义，与[HDFS 监控 NameNode 进程 JVM 指标](cn.zh-CN/监控与报警/APM 监控大盘/服务监控/HDFS 监控.md#ul_bvw_5q4_qgb)章节类似，支持自定义时间范围区间和时间粒度。

## SparkHistory 进程的文件描述符信息 {#section_pbr_3kt_qgb .section}

![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123487/155781663538667_zh-CN.png)

-   Max File Descriptor: SparkHistory 进程可以使用的最大的文件描述符数目
-   Open File Descriptor：SparkHistory 进程已经使用的文件描述符数目

## SparkHistory 进程的启停历史 {#section_rrh_llt_qgb .section}

![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/123487/155781663538668_zh-CN.png)

表格具体含义，参考[HDFS 监控 NameNode 进程启停操作历史](cn.zh-CN/监控与报警/APM 监控大盘/服务监控/HDFS 监控.md#NameNode_history)。

