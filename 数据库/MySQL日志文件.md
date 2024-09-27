## MySQL的六种日志

​	重做日志，回滚日志，二进制日志，错误日志，慢查询日志，一般查询日志，中继日志。
​	其中**重做日志**和**回滚日志**与实务操作息息相关，二进制日志也与实务操作有一定的关系。

#### 重做日志redo log （redo log buffer）

**作用**：确保**事物的持久性和完整性**。防止在发生故障的时间点，尚有**脏页为写入磁盘**，重启mysql服务时，根据redo log重做。
**内容**：物理格式(存储数据库中特定的记录变更)的日志，记录的时物理数据页面的修改的信息，顺序写入。
**什么时候产生**：**事务开始之后产生redo log**，redo log的落盘不是随着事务的提交才写入，而是在事务的执行过程中，便开始写入redo log。
**什么时候释放**：当对应事务的**脏页写入到磁盘后**，redo log的使命也就完成了，重做日志占用的空间就可以重用。
**对应的物理文件**：
	innodb_log_group_home_dir 指定日志文件组所在的路径。
	innodb_log_files_in_group 指定重做文件组中文件的数量。
**关于文件的大小和数量的配置参数**：
	innodb_log_file_size 重做日志文件的大小。
	innodb_mirrored_log_groups 指定了日志镜像文件组的数量，默认1。

**其它**：
	重做日志有一个**缓存区innodb_log_buffer**，其默认大小为8m，innodb存储引擎先将重做日志写入innodb_log_buffer中。
	**三种将日志刷新到磁盘的机制**：
		Master Thread **每秒一次执行刷新**innodb_log_buffer到重做日志文件。
		事务**提交时重做日志刷新**到重做日志文件。
		重做日志缓存**可用空间少于一半时**，重做日志缓存刷新到重做日志文件。

**redo log的刷盘**

​	`innodb_flush_log_at_trx_commit = 0`，每次事务提交的时候，都只是把 redolog 留在 redolog buffer 中
​	`innodb_flush_log_at_trx_commit = 1`，每次事务提交的时候，都执行 fsync 将 redolog 直接持久化到磁盘		
​	`innodb_flush_log_at_trx_commit = 2`，每次事务提交的时候，都只执行 write 将 redolog 写到文件系统的 page cache 中

##### 事务还没提交的时候，redolog 能不能被持久化到磁盘呢？

InnoDB 有一个后台线程，**每隔 1 秒轮询一次**，具体的操作是这样的：调用 write 将 redolog buffer 中的日志写到文件系统的 page cache，然后调用 fsync 持久化到磁盘。而在**事务执行中间过程的 redolog 都是直接写在 redolog buffer 中的**，也就是说，一个**没有提交的事务的 redolog，也是有可能会被后台线程一起持久化到磁盘的**。

另外，除了后台线程每秒一次的轮询操作外，还有两种场景会让一个没有提交的事务的 redolog 写盘：

1. **innodb_flush_log_at_trx_commit 设置是 1**，这样并行的某个事务提交的时候，就会顺带将这个事务的 redolog buffer 持久化到磁盘 举个例子，假设事务 A 执行到一半，已经写了一些 redolog 到 redolog buffer 中，这时候有另外一个事务 B 提交，按照 innodb_flush_log_at_trx_commit = 1 的逻辑，事务 B 要把 redolog buffer 里的日志全部持久化到磁盘，这时候，就会带上事务 A 在 redolog buffer 里的日志一起持久化到磁盘
2. **redo log buffer 占用的空间达到 redolo buffer 大小（由参数 `innodb_log_buffer_size` 控制，默认是 8MB）一半的时候**，后台线程会主动写盘。不过由于这个事务并没有提交，所以这个写盘动作只是 write 到了文件系统的 page cache，仍然是在内存中，并没有调用 fsync 真正落盘

#### 回滚日志undo log （binlog chache）

**作用**：保存了**事务发生之前**的**数据段**而一个版本，可以用于回滚，同时可以提供**多版本并发控制下的读**(MVCC)，即非锁定读。**保证事务的原子性**。
**内容**：逻辑格式(存储事务中的一个操作)的日志，在执行undo的时候，仅仅是将数据从逻辑上恢复至事务之前的状态，而不是从物理页面上操作实现的，也一点不同于redo log。
**什么时候产生**：**事务开始之前**，将当前版本生成undo log，undo也会产生redo来保证undo log的可靠性。
**什么时候释放**：**事务提交之后**，undo log并不能立马删除，而是放入带清理链表，由purge(清除)线程判断是否由**其它事务在使用undo段中表的上一个事务之前的版本信息**，决定是否可以清理undo log的日志空间。
**对应的物理文件**：
	MySQL5.6之前，undo表空间位于**共享表空间(undo)**的回滚段，共享表空间的默认的名称时ibdata，位于数据文件目录中。
	MySQL5.6之后，undo表空间可以配置成独立的文件。

**关于文件的大小和数量的配置参数**：
	innodb_undo_directory undo独立表空间的存放目录
	innodb_undo_logs 回滚段为128k
	innodb_undo_tablespaces 多少个undo log文件
**其它**：
	undo是在事务开始之前**保存的被修改数据的一个版本**，**产生undo日志**的时候，同一会**伴随类似于保护事务持久化机制的redo log的产生**。

#### 二进制日志binlog

有statement、row、mixed三种模式，statement时记录SQL，row是记录行变化。

三种复制方式：同步复制，异步复制，半同步复制。

**作用**：**用于复制**，灾难时的数据恢复，在主从复制中，从库利用主库上的binlog进行重播，实现主从同步，用于数据库的**基于时间点的还原**。
**内容**：**逻辑格式**的日志，可以简单认为就是执行过的事务中的sql语句。还包括了执行的sql语句反向的信息。
**什么时候产生**：**事务提交的时候commit前写入**，一次性将事务中的sql语句按照一定格式记录到binlog中。(会影响到事务提交的速度)
**什么时候释放**：binlog的默认是**保持时间由参数expire_log_days配置**，也就是说对于非活动的日志文件，在生成时间**超过expire_logs_days配置的天数后，会被自动删除**。

**binlog写入磁盘的时间**：
	sync_binlog = 0 的时候，表示每次提交事务都只 write；
	sync_binlog = 1 的时候，表示每次提交事务都会 write，然后马上执行fsync；
	sync_binlog = n 的时候，表示每次提交事务都会 write，当数量达到n执行 fsync

**对应的物理文件**：路径为log_bin_basename，每个binlog日志文件，通过一个统一的index文件来组织。
**其它**：二进制日志的作用之一时还原数据库，这与redo log很类似，单本质不同
	**作用不同**：redo log时保证事务的持久性，binlog作为还原功能，是数据库层面的(也可以精确到事务层面)
	**内容不同**：redo log是物理日志，是数据页面的修改之后的物理记录，binlog是逻辑日志，可以简单认为记录就是sql语句。
	两者日志产生时间，可以释放时间，可释放的情况下清理机制，完全不同。
	恢复数据时候的效率，是基于物理日志的redo log会数据的效率要高于语句逻辑日志的binlog
	为了保证**主从数据库的一致性**，就需要保证redo log 和 binlog的一致性。
	MySQL通过**两阶段提交**过程来完成事物的一致性(redo分为prepare阶段和commit阶段)理论上是**先写redo log的prepare阶段，再写binlog，再到redo log的commit阶段来保证其一致性**。

##### 两段提交

**Prepare阶段**

**Commit阶段**：
	**flush阶段**：多个事务按进入的顺序将 binlog 从 cache 写入文件（不刷盘）；
	**sync阶段**：对 binlog 文件做 fsync 操作（多个事务的 binlog 合并一次刷盘）用于支持组提交 。
	**commit阶段**：各个事务按顺序做 InnoDB commit 操作；

MySQL推迟prepare姐u但不再让事务各自执行redo log刷盘操作，而是推迟到组提交的flush阶段。

<img src="E:\MarkDown_study\数据库\img\数据更新流程.png" alt="数据更新流程" style="zoom:200%;" />

#### 错误日志errorlong

最重要的日志之一，记录了当`mysqld.log`启动和停止时，以及服务器在运行过程中发生任何严重错误时的相关信息，当数据库出现故障无法使用时，建议先看此日志。

日志默认打开，默认存放目录`/var/log/`，默认文件名`mysqld.log`。

如果找不到，可执行`show variables like '%log_error%'`查看。

#### 慢查询日志slow query log

该日志记录了所有执行时间超过参数long_query_time，且所记录数不小于min_examined_row_limit的所有 SQL 语句。默认关闭。

1. 修改`/etc/my.cnf`文件；
2. 设置`show_query_log = 1`，1 表示开启，0 表示关闭；
3. 设置`long_query_time = 2`，未指定默认为 10 秒；
4. 设置`long_show_admin_statements = 1`，开启记录执行慢的管理语句；
5. 设置`long_queries_not_using_indexes = 1`，开启记录执行较慢且未使用索引的语句；

#### 查询日志general log

该日志记录了客户端所有的操作语句，默认关闭。

1. 修改`/etc/my.cnf`文件；
2. 设置`general_log = 1`，1 表示开启，0 表示关闭；
3. 设置日志的文件名，`general_log_file = mysql_query.log`，未指定默认为`host_name.log`。

#### 中继日志relay log

relay log日志文件具有与bin log日志文件相同的格式，从上边MySQL主从复制的流程可以看出，relay log起到一个中转的作用，slave先从主库master读取二进制日志数据，写入从库本地，后续再异步由SQL线程读取解析relay log为对应的SQL命令执行。

[一文搞懂 MySQL 日志 - fuxing． - 博客园 (cnblogs.com)](https://www.cnblogs.com/fuxing/p/18222033#2-版本链)



#### MySQL 磁盘 I/O 很高，有什么优化的方法？

binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count 参数，延迟 binlog 刷盘的时机，从而减少 binlog 的刷盘次数。