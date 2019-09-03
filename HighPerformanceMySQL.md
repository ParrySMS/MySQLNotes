# 高性能MySQL

## C1 架构历史

  | ---  连接/线程处理 ---|
查询缓存       <-----      解析器
             SQL优化器

​             存储引擎

- 每个连接单独一个线程执行
- 解析器会创建解析树进行优化
- Select 会先查询缓存 （Query Cache）
- 并发控制：服务器层，存储引擎层
- 并发读或写：共享锁，排他锁（共享读，写上锁）
- 锁粒度应该尽量小
- 锁策略：在所得开销和数据安全之间权衡
  - 表锁：开销最小。ALTER table 时服务器会自动上表锁
  - 行锁：最大程度支持并发处理，但是锁开销大，在存储引擎层实现
- 事务：一组原子性查询
- ACID:  atomicity, consistency, isolation, durability (原子,一致,隔离,持久)
- 隔离级别
  - 未提交读 read uncommotted ：事务的未提交修改对其他可见， 可读取未提交的数据（Dirty Read)，不建议。
  - 提交读 read committed ：修改不可见，直至提交。同个事务中读取结果两次可能不同,又名 Nonrepeatable read
  - 可重复读 repeatable read : 同个事务中读取结果两次相同，产生幻行（Phantom Row），其他事务可能插入数据。**(MySQL 默认)**
  - 可串行化 serializable ：强制加锁使其串行记录
- 设置隔离级别，可在下一个事务开始时生效 
`set session transaction isolation level read committed;`
`set session transaction isolation level serializable;`
- 死锁：锁的行为和顺序和存储引擎相关
  - 死锁检测：InnoDB将持有最少行级的排他锁的事务进行回滚
  - 死锁超时机制
- 事务日志：预写式 (Write-Ahead Logging)，追加差分记录
- 事务型存储引擎：InnoDB ，NDB Cluster
- 非事务：MyISAM, 内存表
- 自动提交 autocommit
  - 如果不是显示开始一个事务，每个查询会自动 commit 。可通过  autocommit变量设置。`set autocommit = 1`  启动1/OB，关闭0/OFF
  - 关闭之后需要手动执行 commit，所有查询处于一个未提交的事务中。
  - ddl导致大量数据改变或者有锁表的语句，会使其自动提交。


