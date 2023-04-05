# mysql 性能分析


[服务监控系列文章](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU3NjY5MjY2Ng==&action=getalbum&album_id=2810766531256156162#wechat_redirect)


[服务监控系列视频](https://www.bilibili.com/video/BV1vL4y1P7fj/?vd_source=2ab2b434a3dfee1cf437b88820cc8e46)


## 全局监控
对mysql的架构组件有了了解，才知道应该监控哪些指标。

### 指标信息
cpu,内存，iops，buffer pool 命中率,脏页比率


### 日志信息
**慢查询**
```shell
mysql> show variables like 'slow_query%';
+---------------------------+----------------------------------+
| Variable_name             | Value                            |
+---------------------------+----------------------------------+
| slow_query_log            | OFF                              |
| slow_query_log_file       | /mysql/data/localhost-slow.log   |
+---------------------------+----------------------------------+

mysql> show variables like 'long_query_time';
+-----------------+-----------+
| Variable_name   | Value     |
+-----------------+-----------+
| long_query_time | 10.000000 |
+-----------------+-----------+
```
**死锁日志**
查看最近一次死锁日志
```shell
// 找到LATEST DETECTED DEADLOCK
SHOW ENGINE INNODB STATUS;
```
保存死锁日志
```shell
show variables like 'innodb_print_all_deadlocks';
```
将 innodb_print_all_deadlocks 参数设置为 1 ，这样每次发生死锁后，系统会自动将死锁信息输出到错误日志中

#### 死锁日志分析
```shell
2020-06-30 11:52:49 0x7fb5531fa700  ###死锁发生的时间

*** (1) TRANSACTION:   ##事务一

TRANSACTION 11435982, ##事务编号 ACTIVE 38 ##活跃时长以s计算  sec starting index read ##事务正在进行的状态，本案例显示正在读取索引数据

mysql tables in use 1, locked 1 ##当前事务有一个表上有锁，并且是一把锁

LOCK WAIT 2 lock struct(s), ##表示锁等待 size 1136, ##事务执行中分配给锁的内存大小，1 row lock(s)##当前事务持有锁的个数

MySQL thread id 52, ##表示MySQL的进程ID

OS thread handle 140416759858944, 

query id 1983 ###表示SQL的id 

localhost root updating ##表示root@'localhost'执行的 update操作

delete from t2 where c1='asd'  ###表示事务中正在执行(等待)的SQL

##通常情况下通过show engine innodb 不能拿到完整的事务的信息，这个给DBA排查问题增加了不少麻烦，需要跟研发沟通结合SQL审计找出此事务的全部SQL

*** (1) WAITING FOR THIS LOCK TO BE GRANTED: ##等待后去这个锁

RECORD LOCKS #记录锁 space id 181 page no 4 ##记录space id 为181,page序号为4(一般跟事务2上的hold lock信息相互对应)

 n bits 80 index idx_c1 of table `test`.`t2` trx id 11435982 lock_mode X waiting

###说明等待test库t2表上的idx_c1索引的X 锁

Record lock, heap no 3 PHYSICAL RECORD: n_fields 2; compact format; info bits 32 ##表示记录锁的物理记录信息

 0: len 3; hex 617364; asc asd;;

 1: len 4; hex 80000002; asc     ;;

##表示记录锁的字段信息 ##对应表中字段的值，存储为16进制，可以解析为十进制来对应表中锁的数据



*** (2) TRANSACTION:

TRANSACTION 11435977, ACTIVE 51 sec inserting

mysql tables in use 1, locked 1

5 lock struct(s), heap size 1136, 4 row lock(s), undo log entries 2

MySQL thread id 51, OS thread handle 140416760391424, query id 1984 localhost root update

##与上面分析类似

insert into t2 (c1,c2) values ('aaa','mmm')

*** (2) HOLDS THE LOCK(S): ##事务2已经获取的锁信息、与上面分析类同

RECORD LOCKS space id 181 page no 4 n bits 80 index idx_c1 of table `test`.`t2` trx id 11435977 lock_mode X

###表示目前拥有这行的数据记录锁

Record lock, heap no 3 PHYSICAL RECORD: n_fields 2; compact format; info bits 32

 0: len 3; hex 617364; asc asd;; 

 1: len 4; hex 80000002; asc     ;;



###与事务一类同



*** (2) WAITING FOR THIS LOCK TO BE GRANTED: ##事务2等待获取的锁信息

RECORD LOCKS space id 181 page no 4 n bits 80 index idx_c1 of table `test`.`t2` trx id 11435977 lock_mode X locks gap before rec insert intention waiting ###表示等待获取gap锁，在RR模式下GAP锁，insert数据过程中会有lock x锁+ next key锁的组合操作

Record lock, heap no 3 PHYSICAL RECORD: n_fields 2; compact format; info bits 32

 0: len 3; hex 617364; asc asd;;

 1: len 4; hex 80000002; asc     ;;



*** WE ROLL BACK TRANSACTION (1)

##结果回滚了事务1

```

**错误日志**
```shell
SHOW VARIABLES LIKE 'log_error';
```
**系统错误日志**
dmesg
/var/log/messages

## 线上问题如何排查

### 查看当前线程执行情况

可以看到

```shell
show full processlist
```
or
```shell
select id, db, user, host, command, time, state, info
from information_schema.processlist
where command != 'Sleep'
order by time desc 
```
id - 线程ID，可以用：kill id; 杀死一个线程，很有用
db - 数据库
user - 用户
host - 连库的主机IP
command - 当前执行的命令，比如最常见的：Sleep，Query，Connect 等
time - 消耗时间，单位秒
state - 执行状态，比如：Sending data，Sorting for group，Creating tmp table，Locked等等
info - 执行的SQL语句

### 借助于performace_shema性能分析
需要 开启performance_schema配置 ，可能会有百分之10的性能损耗，但我认为是值得的，监控然后发现问题比这点性能损失更加重要，很多时候，我们不应该把压力放到数据库这一层。

```shell
show variables  like  'performance_schema';
```
![image.png](https://s2.loli.net/2023/03/07/KakDn78GUjm3Vbv.png)

#### 锁问题，查看锁等待信息

**查看当前事务锁等待**

可以得到 等待与被等待的sql语句与锁类型，与等待时间。
```shell
mysql> select * from sys.innodb_lock_waits\G
*************************** 1. row ***************************
                wait_started: 2020-09-16 11:21:42           #发生锁等待的开始时间
                    wait_age: 00:00:06                      #锁已经等待了多久，该值是一个时间格式值
               wait_age_secs: 6                             #锁已经等待了几秒钟，该值是一个整型值，该字段是MySQL5.7.9中新增的
                locked_table: `employees`.`dept_emp`        #锁等待的表名称
                locked_index: PRIMARY                       #锁等待的索引名称
                 locked_type: RECORD                        #锁等待的锁类型
              waiting_trx_id: 8325                          #锁等待的事物ID
         waiting_trx_started: 2020-09-16 11:21:42           #发生锁等待的事务开始时间
             waiting_trx_age: 00:00:06                      #发生锁等待的事务总的锁等待时间，该值是一个时间格式
     waiting_trx_rows_locked: 1                             #发生锁等待的事务已经锁定的行数（如果是复杂事务会累计）
   waiting_trx_rows_modified: 0                             #发生锁等待的事务已经修改的行数（如果是复杂事务会累计）
                 waiting_pid: 128861                        #发生锁等待的事务的processlist_id
               waiting_query: update employees.dept_emp set  ... 99-01-02' where emp_no='10001'  #发生锁等待的事务的SQL语句文本
             waiting_lock_id: 8325:60:6:2                   #发生锁等待的锁id
           waiting_lock_mode: X                             #发生锁等待的锁模式
             blocking_trx_id: 8324                          #持有锁的事务id
                blocking_pid: 128819                        #持有锁的事务的processlist_id
              blocking_query: NULL                          #持有锁的事务的SQL语句文本
            blocking_lock_id: 8324:60:6:2                   #持有锁的锁id
          blocking_lock_mode: X                             #持有锁的锁模式
        blocking_trx_started: 2020-09-16 11:20:53           #持有锁的事务的开始时间 
            blocking_trx_age: 00:00:55                      #持有锁的事务已执行了多长时间，该值为时间格式值
    blocking_trx_rows_locked: 332334                        #持有锁的事务的锁定行数
  blocking_trx_rows_modified: 0                             #持有锁的事务需要修改的行数
     sql_kill_blocking_query: KILL QUERY 128819             #执行KILL语句来杀死持有锁的查询语句（而不是终止会话）。该字段是MySQL5.7.9中新增的
sql_kill_blocking_connection: KILL 128819                   #执行KILL语句以终止持有锁的语句的会话。该字段是MySQL5.7.9中新增的
1 row in set, 3 warnings (0.17 sec)

mysql>
```
**查看MDL锁等待**

MDL锁即元数据锁，元数据指描述数据的数据，当dml（update insert）语句与ddl（比如改列名等）语句
操作同一个元数据时，就会用到mdl锁。

```shell
mysql> select * from sys.schema_table_lock_waits\G
*************************** 1. row ***************************
               object_schema: employees              #发生MDL锁等待的schema名称
                 object_name: dept_emp               #正在等待MDL锁的表名称
           waiting_thread_id: 130764                 #正在等待MDL锁的线程ID
                 waiting_pid: 130739                 #正在等待MDL锁的processlist_id
             waiting_account: root@localhost         #正在等待MDL锁与线程关联的account名称
           waiting_lock_type: EXCLUSIVE              #被阻塞的线程正在等待的MDL锁类型
       waiting_lock_duration: TRANSACTION            #该字段来自元数据锁子系统中的锁定时间，有效值为：STATEMENT、TRANSACTION、EXPLICIT、STATMENT和TRANSACTION值分别表示在语句或事务结束时会释放的锁。EXPLICIT值表示可以在语句或事务结束时会被保留，需要显示释放的锁，例如：使用FLUSH TABLES WITH READ LOCK获取的全局锁
               waiting_query: ALTER TABLE `employees`.`dept_ ... te` DATE NOT NULL COMMENT '时间'     #正在等待MDL锁的线程对应的语句文本
          waiting_query_secs: 7                      #正在等待MDL锁的语句已经等待了多长时间（秒）
 waiting_query_rows_affected: 0                      #受正在等待MDL锁的语句影响的数据行数（该字段来自performance_schema.events_statement_current表，该表中记录的是语句事件，如果语句是多表联结查询语句，则该语句可能已经执行了一部分DML语句，所以即使改语句当前被其他线程阻塞了，被阻塞线程的这个字段也可能出现大于0的值）
 waiting_query_rows_examined: 0                      #正在等待MDL锁的语句从存储引擎检查的数据行数（同理，该字段来自performance_schema.events_statement_current）
          blocking_thread_id: 130763                 #持有MDL锁的线程id
                blocking_pid: 130738                 #持有MDL锁的processlist_id
            blocking_account: root@localhost         #持有MDL锁的与线程关联的account名称
          blocking_lock_type: SHARED_WRITE           #持有MDL锁的锁类型
      blocking_lock_duration: TRANSACTION            #与waiting_lock_duration字段的解释相同，只是该值与持有MDL锁的线程相关
     sql_kill_blocking_query: KILL QUERY 130738      #生成的KILL持有MDL锁的查询语句
sql_kill_blocking_connection: KILL 130738            #生成的KILL持有MDL锁的对应会话的语句
mysql>
```


#### 查询效率问题  临时表，文件排序？

**使用了文件排序的sql**
```shell
mysql> select * from sys.statements_with_sorting\G
*************************** 1. row ***************************
            query: SELECT IF ( ( `locate` ( ? , ` ...  . `COMPRESSED_SIZE` ) ) DESC   #经过标准化转换的语句字符串
               db: sys                       #语句对应的默认数据库，如果没有默认数据库，则该字段值为null     
       exec_count: 52                        #语句执行的总次数
    total_latency: 825.71 ms                 #语句执行的总延迟时间（执行时间）
sort_merge_passes: 22                        #语句执行发生的语句排序合并的总次数
  avg_sort_merges: 0                         #针对发生排序合并的语句，每条语句的平均排序合并次数（见视图查询语句文本中的SUM_SORT_MERGE_PASSES/COUNT_STAR）
sorts_using_scans: 8                         #语句排序执行全表扫描的总次数
 sort_using_range: 0                         #语句排序执行范围扫描的总次数
      rows_sorted: 28670                     #语句执行发生排序的总数据行数
  avg_rows_sorted: 551                       #针对发生排序的语句，每条语句的平均排序数据行数（见视图查询语句文本中的SUM_SORT_ROWS/COUNT_STAR）
       first_seen: 2020-09-14 17:45:48       #该语句第一次出现的时间
        last_seen: 2020-09-18 14:03:19       #该语句最近一次出现的时间
           digest: 410a247bd55c869c709912d4cb6ec9c4  #语句摘要计算的MD5 hash值
```

**查看使用了临时表的sql**
```shell
mysql> select * from sys.statements_with_temp_tables limit 1\G
*************************** 1. row ***************************
                   query: SELECT IF ( ( `locate` ( ? , ` ...  . `COMPRESSED_SIZE` ) ) DESC   #经过标准化转换的语句字符串
                      db: sys                                 #语句对应的默认数据库，如果没有默认数据库，则该字段值为null  
              exec_count: 3                                   #语句执行的总次数
           total_latency: 583.97 ms                           #语句执行的总延迟时间（执行时间）
       memory_tmp_tables: 24                                  #语句执行时创建的内部内存临时表的总数量
         disk_tmp_tables: 15                                  #语句执行时创建的内部磁盘临时表的总数量
avg_tmp_tables_per_query: 8                                   #对于使用了内存临时表的语句，每条语句使用内存临时表的平均数量（见视图查询语句文本中的SUM_CREATED_TMP_TABLES/COUNT_STAR）
  tmp_tables_to_disk_pct: 63                                  #内存临时表的总数量与磁盘临时表的总数量百分比，表示磁盘临时表的转换率（见视图查询语句文本中的SUM_CREATED_TMP_DISK_TABLES/SUM_CREATED_TMP_TABLES）
              first_seen: 2020-09-18 13:19:55                 #该语句第一次出现的时间
               last_seen: 2020-09-18 14:03:59                 #该语句最近一次出现的时间
                  digest: cdc49c85aa2a0790a8ac38cd6e117ff2    #语句摘要计算的MD5 hash值
1 row in set (0.01 sec)

mysql>
```

#### 查询效率 索引选择策略，实际sql执行耗时分析

**explain**

通过explain去分析语句可能使用到的索引，以及使用索引的type，预计扫描行数和是否用到文件排序等信息。
Extra 表字段存储是否使用排序，临时表等。
一般情况下，临时表和排序操作都是我们应该避免的。

explain 返回结果type字段解析

```shell
ALL：Full Table Scan， MySQL将遍历全表以找到匹配的行

index: Full Index Scan，index与ALL区别为index类型只遍历索引树

range:只检索给定范围的行，使用一个索引来选择行

ref: 使用了索引，但是有回表的操作。

eq_ref: 类似ref，区别就在使用的索引是唯一索引，对于每个索引键值，表中只有一条记录匹配，简单来说，就是多表连接中使用primary key或者 unique key作为关联条件

const、system: 针对主键或唯一索引的等值查询扫描, 最多只返回一行数据. const 查询速度非常快, 因为它仅仅读取一次即可,system是const类型的特例，当查询的表只有一行的情况下，使用system
```


```shell
mysql> explain select * from servers;
+----+-------------+---------+------+---------------+------+---------+------+------+-------+
| id | select_type | table   | type | possible_keys | key  | key_len | ref  | rows | Extra |
+----+-------------+---------+------+---------------+------+---------+------+------+-------+
|  1 | SIMPLE      | servers | ALL  | NULL          | NULL | NULL    | NULL |    1 | NULL  |
+----+-------------+---------+------+---------------+------+---------+------+------+-------+
1 row in set (0.03 sec)
```
##### 执行顺序
id相同时，执行顺序由上到下
id不同时，id越大越先执行。

explain 方法有局限性
第一  explain 是预估的执行计划和可能使用的索引，当多个possible keys 时，不知道数据库最终采用哪些索引去执行查询。
第二 explain无法更精细的查看sql各个阶段的耗时情况，比如锁等待耗时。
所以有以下两种方法来更精细准确的分析sql查询。

**trace**
能够看到sql是如何解析，索引如何被选择，执行的。

调大trace的容量
```shell
set session optimizer_trace_max_mem_size = 10485760;
```

开启optimizer_trace
```shell
set session optimizer_trace="enabled=on";
```

执行sql...

查看trace
```shell
select TRACE from INFORMATION_SCHEMA.OPTIMIZER_TRACE\G
```



**show profiles**
更加精细的查看sql执行耗时。

查看profiling系统变量
```shell
mysql> show variables like '%profil%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| profiling              | OFF   |   --开启SQL语句剖析功能。0或OFF表示关闭(默认模式)。1或ON表示开启
| profiling_history_size | 15    |   --设置保留profiling的数目，缺省为15(即使用show profiles;命令展示的默认是最近执行的15条sql的记录)，范围为0至100，为0时将禁用profiling。这个参数在MySQL 5.6.8过期，在后期版本中会被移除。
+------------------------+-------+
 
```
开启会话级别的profile
```shell
set profiling  =1 ;
```
执行慢sql ...


查看当前session所有已产生的profile
```shell
mysql> show profiles;
+----------+------------+-------------------------------------------------------+
| Query_ID | Duration   | Query                                                                                                                                                                                                                                                                                                        |
--------------------------------------------------------------------------------+
|        1 | 0.00037700 | show variables like '%profil%'                                                                                                                                                                                                                                                                               |
|        2 | 0.00029500 | show databases                                                                                                                                                                                                                                                                                               |
|        3 | 0.00030800 | show tables                                                                                                                                                                                                                                                                                                  |
|        4 | 0.00135700 | select count(*) from emp
+----------+------------+-------------------------------------------------------+
```

展示特定查询的profile
```shell
show profile all for query 9;
+--------------------+----------+----------+------------+-------------------+---------------------+--------------+---------------+---------------+-------------------+-------------------+-------------------+-------+------------------+---------------+-------------+
| Status             | Duration | CPU_user | CPU_system | Context_voluntary | Context_involuntary | Block_ops_in | Block_ops_out | Messages_sent | Messages_received | Page_faults_major | Page_faults_minor | Swaps | Source_function  | Source_file   | Source_line |
+--------------------+----------+----------+------------+-------------------+---------------------+--------------+---------------+---------------+-------------------+-------------------+-------------------+-------+------------------+---------------+-------------+
| starting           | 0.000997 | 0.000000 |   0.000000 |                 0 |                   1 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | NULL             | NULL          |        NULL |
| Opening tables     | 0.000015 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_base.cc   |        4622 |
| System lock        | 0.000003 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | lock.cc       |         260 |
| Table lock         | 0.000515 | 0.001000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | lock.cc       |         271 |
| optimizing         | 0.000093 | 0.000000 |   0.000000 |                 0 |                   1 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_select.cc |         835 |
| statistics         | 0.151343 | 0.124981 |   0.023996 |                 1 |                 155 |            0 |             0 |             0 |                 0 |                 0 |                39 |     0 | unknown function | sql_select.cc |        1026 |
| preparing          | 0.000082 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_select.cc |        1048 |
| executing          | 0.000027 | 0.000000 |   0.000000 |                 0 |                   1 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_select.cc |        1789 |
| Sending data       | 0.081436 | 0.072989 |   0.006999 |                 0 |                  82 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_select.cc |        2347 |
| init               | 0.000024 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_select.cc |        2537 |
| optimizing         | 0.000006 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_select.cc |         835 |
| statistics         | 0.000006 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_select.cc |        1026 |
| preparing          | 0.000007 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_select.cc |        1048 |
| executing          | 0.000005 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_select.cc |        1789 |
| Sending data       | 0.000046 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_select.cc |        2347 |
| end                | 0.000004 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_select.cc |        2583 |
| query end          | 0.000003 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_parse.cc  |        5123 |
| freeing items      | 0.000076 | 0.001000 |   0.000000 |                 1 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_parse.cc  |        6147 |
| removing tmp table | 0.000006 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_select.cc |       10929 |
| closing tables     | 0.000004 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_select.cc |       10954 |
| logging slow query | 0.000002 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_parse.cc  |        1735 |
| cleaning up        | 0.000004 | 0.000000 |   0.000000 |                 0 |                   0 |            0 |             0 |             0 |                 0 |                 0 |                 0 |     0 | unknown function | sql_parse.cc  |        1703 |
+--------------------+----------+----------+------------+-------------------+---------------------+--------------+---------------+---------------+-------------------+-------------------+-------------------+-------+------------------+---------------+-------------+
 
```




