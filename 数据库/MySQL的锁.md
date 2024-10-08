## MySQL的锁

**全局锁：**FTWRL

**表级锁：**
	**表锁**：对整个表进行加锁。
	**元数据锁**：保证当用户对执行CRUD操作时，防止其他线程对这个表结构做了变更。
	**意向锁**：加速判断表里是否有记录被加锁。
	**AUTO-INC锁**：主键的自增值，是通过其实现，在执行插入语句后就会立即释放。
	当 **innodb_autoinc_lock_mode = 2** 时，并且 **binlog_format = row**，既能提升并发性，又不会出现数据一致性问题。

**行级锁：**
	**Record Lock**：记录锁，仅仅把一条记录锁上。
	**Gap Lock**：间隙锁，锁定一个范围，但不包含记录本身。间隙锁之间相互不互斥，**防止插入幻影记录而提出的**。
	**Next-key Lock**：Record Lock 和 Gap Lock的组合，锁定一个范围，并且锁定记录本身。
	**插入意向锁**：属于间隙锁，在插入时检查是否有间隙锁 。

**什么是SQL语句会加锁？**

~~~sql
// 对读取的记录加共享锁(S型锁)
select ... lock in share mode;
// 对读取的记录加独占锁(X型锁)
select ... for update;
// 对操作的记录加独占锁(x型锁)
update table ... where id = 1;
// 对操作的记录加独占锁(x型锁)
delete from table where id = 1;
~~~

#### MySQL是怎么加行级锁？

**加锁的对象是索引，加锁的基本单位是 next-key lock**，它是由记录锁和间隙锁组合而成的，**next-key lock 是前开后闭区间，而间隙锁是前开后开区间**。
在**能使用记录锁或者间隙锁就能避免幻读现象的场景下**，next-key lock 就会退化成记录锁或间隙锁。

**唯一索引等值查询**：
	当查询的记录是「存在」的， next-key lock 会**退化成「记录锁」**
	查询的记录是「不存在」的， next-key lock 会**退化成「间隙锁」**

**非唯一索引等值查询**：
	当查询的记录「存在」时，在扫描的过程中，对扫描到的二级索引记录加的是next-key锁，而对于**第一个不符合条件**的二级索引记录，该**二级索引的next-key锁会退化成间隙锁**。
	当查询的记录「不存在」时，扫描到**第一条不符合条件**的二级索引记录，该**二级索引的 next-key 锁会退化成间隙锁**。因为不存在满足查询条件的记录，所以不会对主键索引加锁。

**非唯一索引和主键索引的范围查询的加锁规则不同之处在于**：
	唯一索引在满足一些条件的时候，索引的 next-key lock 退化为间隙锁或者记录锁。
	非唯一索引范围查询，索引的 next-key lock 不会退化为间隙锁和记录锁

#### update没加索引会所全表

where 会根据条件选择不同的索引，当没有合适的索引时，可能会遍历全表，导致全表上锁的情况。

#### MySQL记录锁+间隙锁可以防止删除操作而导致的幻读问题吗？

MySQL 记录锁+间隙锁**可以防止**删除操作而导致的幻读问题

#### MySQL 死锁了，怎么办？

如果两个操作对同一区间加了间隙锁和记录锁，那么这两个事务内对数据的操作，会因为相互的锁而导致死锁问题。

思索地四个必要条件：互斥、占有且等待、不可抢占，循环等待。
接触死锁：
	①设置事务等待锁的超时时间。②开启主动死锁检测。