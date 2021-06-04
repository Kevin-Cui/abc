---
layout: post
title: MySQL 历史SQL语句，性能问题查找
categories: MySQL
description: MySQL 历史SQL语句，性能问题查找
keywords: MySQL history SQL
---

经常会碰到应用反馈上周出现过SQL性能问题，离上周已经过了好几天，怎么查，太难了。。。
对于MySQL了解来说排查这种问题，基本3中思路：

应用问题日志信息；
MySQL的慢日志可以，慢日志开了吗，多少秒记录指标；
MySQL的binlog应该会有记录，binlog保留多少天；
MySQL的监控指标；
以上4种都是常用的手段，除了这些方式，还能不能抓到特定的SQL语句，是否存在性能问题。 如：disk,lock,wait,join,scan,index,rows 等。

得到答案之前，先了解一下MySQL企业版怎么去解决这样的问题。


## 企业版SQL语句性能分析方式：
MySQL Enterprise Monitor(简称EM）是怎么做到sql语句的性能监控。官网提供一个月使用企业版功能，可自行下载研究。
EM对于数据库性能指标（QRTi表示查询响应时间）:
![](https://kevin-cui.github.io/mysqlstone/images/posts/mysql/20210604-01.png)

QRTi值计算例子：
如果有100次摘要/规范查询的执行，其中60次在100ms(最佳时间框架)以下完成，30次在100ms到400ms(可接受的时间框架)之间完成，其余10次超过400ms(不可接受的时间框架)，那么QRTi得分为:
```
((60 +(30 / 2) +(10*0)) / 100) = 0.75
```
备注：从这些指标计算中可以看出，MySQL官网对于性能的基本要求在在0.5秒。

![](https://kevin-cui.github.io/mysqlstone/images/posts/mysql/20210604-02.png)
![](https://kevin-cui.github.io/mysqlstone/images/posts/mysql/20210604-03.png)
![](https://kevin-cui.github.io/mysqlstone/images/posts/mysql/20210604-04.png)
2.通过开启general_log查看EM抽取源数据来源：
通过开启general_log查看EM的触发动作，
当执行EM查询分析(Query Analyzer)是通过定时抽取和相减得到这个SQL执行的当时的情况。
当执行EM查询分析(Query Analyzer)是通过定时抽取和相减得到这个SQL执行的当时的情况。
![](https://kevin-cui.github.io/mysqlstone/images/posts/mysql/20210604-05.png)

从上面记录可以理解到 MySQL的性能指标还是基于performance_schema库的,这里Statement Summary Tables用于收集当前和最近语句事件的表，并在汇总表中聚合这些信息。

3.Statement Summary Tables介绍：
先看下这张表汇总表信息：
```
mysql> SHOW TABLES LIKE '%events_statements_summary_by_digest';
+---------------------------------------------------------------------+
| Tables_in_performance_schema (%events_statements_summary_by_digest) |
+---------------------------------------------------------------------+
| events_statements_summary_by_digest                                 |
+---------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql>  SELECT * FROM performance_schema.events_statements_summary_by_digest limit 1\G
*************************** 1. row ***************************
                SCHEMA_NAME: NULL
                     DIGEST: 44e35cee979ba420eb49a8471f852bbe15b403c89742704817dfbaace0d99dbb
                DIGEST_TEXT: SELECT @@`version_comment` LIMIT ?
                 COUNT_STAR: 8
             SUM_TIMER_WAIT: 1566500000
             MIN_TIMER_WAIT: 87700000
             AVG_TIMER_WAIT: 195800000
             MAX_TIMER_WAIT: 489900000
              SUM_LOCK_TIME: 0
                 SUM_ERRORS: 0
               SUM_WARNINGS: 0
          SUM_ROWS_AFFECTED: 0
              SUM_ROWS_SENT: 8
          SUM_ROWS_EXAMINED: 8
SUM_CREATED_TMP_DISK_TABLES: 0
     SUM_CREATED_TMP_TABLES: 0
       SUM_SELECT_FULL_JOIN: 0
 SUM_SELECT_FULL_RANGE_JOIN: 0
           SUM_SELECT_RANGE: 0
     SUM_SELECT_RANGE_CHECK: 0
            SUM_SELECT_SCAN: 0
      SUM_SORT_MERGE_PASSES: 0
             SUM_SORT_RANGE: 0
              SUM_SORT_ROWS: 0
              SUM_SORT_SCAN: 0
          SUM_NO_INDEX_USED: 0
     SUM_NO_GOOD_INDEX_USED: 0
                 FIRST_SEEN: 2021-06-03 11:04:10.553945
                  LAST_SEEN: 2021-06-03 14:34:15.950372
                QUANTILE_95: 501187233
                QUANTILE_99: 501187233
               QUANTILE_999: 501187233
          QUERY_SAMPLE_TEXT: select @@version_comment limit 1
          QUERY_SAMPLE_SEEN: 2021-06-03 14:34:15.950372
    QUERY_SAMPLE_TIMER_WAIT: 489900000
1 row in set (0.00 sec)
```
字段说明：
![](https://kevin-cui.github.io/mysqlstone/images/posts/mysql/20210604-06.png)
备注：去掉变量并执行过的SQL语句性能指标

启动状态：这个表由setup_consumers决定是否激活，默认是激活
```
mysql> SELECT * FROM performance_schema.setup_consumers WHERE name = 'statements_digest';
+-------------------+---------+
| NAME              | ENABLED |
+-------------------+---------+
| statements_digest | YES     |
+-------------------+---------+
1 row in set (0.00 sec)
```
备注：所以基本MySQL安装下，操作的SQL语句是默认记录下来的。现在起码多了一种查看历史SQL语句排查思路。

语句限制参数:
分析的语句的长度不能超过下面的参数，否则截断
```sql
mysql> SHOW VARIABLES LIKE '%max_digest_leng%';
+--------------------------------------+-------+
| Variable_name                        | Value |
+--------------------------------------+-------+
| max_digest_length                    | 1024  |
| performance_schema_max_digest_length | 1024  |
+--------------------------------------+-------+
2 rows in set, 1 warning (0.01 sec)
```
备注：单位字节数（1024字节等于342个汉字）
1）第一个参数max_digest_length是session级别的，
2）第二个performance_schema_max_digest_length是语句级别的，只对performance_schema起作用。
3）如果performance_schema_max_digest_length小于max_digest_length，则相对于原始复制将被截断。

记录行长度：erformance_schema_digests_size
events_statements_summary_by_digest表中的最大行数。默认是10000，如果超过这个最大值，新的sql语句无法插入。试过可以TRUNCATE定期清理数据。
```
mysql> SHOW VARIABLES LIKE '%performance_schema_digests_size%';
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| performance_schema_digests_size | 10000 |
+---------------------------------+-------+
1 row in set, 1 warning (0.00 sec)
```
4.sys库视图
statement_analysis视图查看SQL汇总统计信息(数据来源events_statements_summary_by_digest)
视图statement_analysis查看总执行时间最长的SQL：
![](https://kevin-cui.github.io/mysqlstone/images/posts/mysql/20210604-07.png)
```
query：抽象后的SQL
Db：语句默认数据库
full_scan：全表扫描的话*，否则’’
total/max/avg/lock_latency：总时间、最大时间、平均时间、总锁等待时间
rows_sent/rows_sent_avg：总返回行数、平均返回行数
rows_examined/rows_examined_avg：总检查行数、平均检查行数
tmp_tables：创建临时表数量
exec_count：执行次数
err_count/warn_count：错误或警告次数
tmp_disk_tables：创建磁盘临时表数量
rows_sorted：排序次数
sort_merge_passes：合并排序次数
```
## 总结
events_statements_summary_by_digest表保存的是状态数据，需要间隔时间保存起来，通过前后值的减得出结果。除了慢日志，记录在这张表里SQL语句性能差的也需要优化。
问题SQL找到了，下一步就可以进行分析了。
SQL优化方式：
![](https://kevin-cui.github.io/mysqlstone/images/posts/mysql/20210604-08.png)
