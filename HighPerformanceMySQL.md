# 高性能MySQL

## C1 架构历史

  | ---  连接/线程处理 ---|

查询缓存       <-----      解析器

​             SQL优化器

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



- - **事务 transaction** ：一组原子性查询
- ACID:  atomicity, consistency, isolation, durability (原子,一致,隔离,持久)



- - **隔离级别**
  - 未提交读 read uncommotted ：事务的未提交修改对其他可见， 可读取未提交的数据（Dirty Read)，不建议。
  - 提交读 read committed ：修改不可见，直至提交。同个事务中读取结果两次可能不同,又名 Nonrepeatable read
  - **可重复读 repeatable read : 同个事务中读取结果两次相同，产生幻行（Phantom Row），其他事务可能插入数据。(MySQL 默认)**
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
  - **如果不是显式开始一个事务，每个查询会自动 commit** 。可通过  autocommit变量设置。`set autocommit = 1`  启动1/OB，关闭0/OFF
  - 关闭之后需要手动执行 commit，所有查询处于一个未提交的事务中。
  - ddl导致大量数据改变或者有锁表的语句，会使其自动提交。



- 事务中混合使用引擎
  - MySQL 不负责事务，由存储引擎实现。
  - 假设同时使用了 InnoDB (事务) 和 MyISAM (非事务)，事务回滚时，**非事务的变更无法撤销**。



- InnoDB 的两阶段锁
  - 隐式：执行过程随时能加锁，执行 commit 或 rollback 时释放 	    
  - InnoDB 会根据隔离级别在需要时自动加锁
  - 显式：
    - select ... lock in share mode
    - select ... for update
  - 锁表服务层实现，例如 MyISAM 会使用 lock/unlock tables 



- **建议：不要显式执行 lock tables，除非事务禁用了 autocommit 只能手动 lock**



- 多版本并发控制 MVCC
  - 只在 [提交读 read committed] 和 [可重复读 repeatable read] 隔离级别下可行  
  - 非阻塞的读操作，写操作只锁定必要行
  - 通过报错时间点快照实现
  - 乐观锁，悲观锁 （Refer: [MySQL 乐观锁与悲观锁](https://www.jianshu.com/p/f5ff017db62a)）
  - InnoDB 的 MVCC 具体实现
    - 每行记录后面两个隐藏列（创建版本号，删除版本号）
    - Select 
      -  where ( 创建版本< 当前 ) and ( 删除版本 > 当前 ||删除版本未定义）
    - Insert 
      - 创建版本 = 当前
    - Delete
      - 删除版本 = 当前
    - Update  
      - insert new one and delete original one 



- MySQL <u>Storage</u> Engine:  working for SQL
  - default : InnoDB
  - every schema saves as a  `schemaname/`
  - table defination saves as `tablename.frm`
    
    -  schema and table saving related to Server
  - data saving related to Storage Engine 	 
  - case sensitivity related to sys because using file path
    - win : case insensitive =  CASE INSENSITIVE
    - unix :  case sensitive 
    - **just use lowercase and`_`**
  
  - using `BINARY ` to make field name case sensitive 
  
    ```sql
    CREATE TABLE 表名( 
        字段名 VARCHAR(10) BINARY 
    );
    ```
    
  
- 查看表属性  `show table status like 'xxxx'`

```
| Name| Engine | Version | Row_format | Rows | Avg_row_length | Data_length | Max_data_length | Index_length | Data_free | Auto_increment | Create_time | Update_time | Check_time | Collation          | Checksum | Create_options | Comment |

notes: length mean the size of data with unit Byte.
```

  - Row_format 
    - dynamic 包含可变的行长度，如用了 varchar, blob  
    - fixed 固定
    - compressed 存在于压缩表
    
  - Rows 行数 : MyISAM 精确值，InnoDB 估计值 

  - Data_free 空闲空间：已分配但未使用，空间内可能包含一些之前删除的行

  - Auto_increment ：下一个插入的数值



- InnoDB --- 默认的事务引擎
  - 设计用来处理大量的短期事务 short-lived, 较少回滚  
  - 性能好，自动崩溃恢复
  - MySQL 5.5 之前 是 旧引擎 + InnoDB plugin
  - 利用排序创建索引
  - 用 Sphinx 组合起来可以支持全文索引
  
  
  
- 数据存储: InnoDB 数据存储在表空间 （tablespace），由引擎管理，采用 MVCC 支持高并发，实现了四个标准的隔离级别（read uncommotted, read committed, repeatable read, serializable）。默认可重复读repeatable read 。

- 聚簇索引主键来建表，二级索引（非主键索引）中会包含主键列，若索引较多，应当使用小主键Refer。

- [参考：Mysql聚簇索引和非聚簇索引原理](https://blog.csdn.net/lisuyibmd/article/details/53004848)

  

- InnoDB 引擎的架构重点：**InnoDB Locking and Transaction Model**
  - [MySQL 5.7 Reference Manual---InnoDB Locking and Transaction Model](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking-transaction-model.html)
  - [MySQL 8.0 Reference Manual---InnoDB Locking and Transaction Model](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-transaction-model.html)
  
  
  
- InnoDB 的存储格式跨平台

- 很多内部优化：预测读，自适应内存哈希索引 adaptive hash index，插入缓冲区 inset buffer

- 通过机制和工具支持真正的热备份



- MyISAM --- 5.1之前曾经的默认引擎
  - Sun 收购 MySQL后首个版本，5.1版本2008年冬发布
  - 不支持事务和行级锁
  - 崩溃后无法安全恢复
  - 数据文件 `xxx.MYD`
  - 索引文件 `xxx.MYI`
  - 通过修改表的 `MAX_ROWS` 和 `AVG_ROW_LENGTH` 可以修改表大小的上限（取决于表指针长度位数），修改后会重建整个表以及所有索引（耗时）。
  - 特性：
    - 整张表加锁，读共享锁，写排它锁。读取时允许其他插入（并发插入 concurrent insert）
    - 设计简单，数据以紧密格式存储，但是表锁可能带来性能问题
    - 长字段 BLOB, TEXT 可以基于前500字符创建索引
    - **支持全文索引**（基于分词创建的）
    - 延迟更新索引键（Delayed Key Write）
      - 可指定 DELAY_KEY_WRITE 选项，可全局可各表
      - 修改的索引数据先写入缓冲区（in-mem key buffer）
      - 缓冲区清空或表关闭时写入磁盘
      - 优点是提升写入性能，缺点是崩溃导致索引损坏则需要执行修复
    - 压缩表
      - 使用 `myisampack` 进行打包压缩
      - 适合只读表，支持索引
      - 压缩时表中的记录是独立压缩的，所以读行只是解压行 



- 一些其他引擎：

  - **Archive** 高速插入和压缩，全表扫描

  - **CSV**  转化逗号分隔值的csv文件为表，不支持索引，作为数据交换中介

  - MariaDB 的 **FederatedX** 访问其他数据库服务的代理

  - **NDB** 集群引擎

  - **Memory**  （HEAP表） 快速访问，无修改，不在意重启数据丢失

    - 支持 hash 索引，表级锁，固定长度列（ varchar 会转成 char ）

    - MySQL 内部保存中间结果所使用的临时表优先用 Memory 表，其次用 MyISAM  

    - 用于映射表（mapping ）

    - 缓存 （periodically aggregated data）

    - 数据分析的中间数据

      

- 第三方引擎
  - OLTP 类引擎
  - 面向列的存储引擎 
  - 社区引擎 OQGraph 支持图



- 插入表 --- 日志型应用
  - 大量 insert
  - 需要做分析时的性能解决方案
    - 复制到备库，然后执行分析
    - 记录表中加入日期分表，对没有插入操作的历史表来操作




- 在性能测试环境中，进行断电的模拟崩溃测试，以获得崩溃恢复的时间以及可靠性。

- 只读表(类目清单)可以考虑用 MyISAM,但是性能并不会比 InnoDB 好很多。InnoDB依靠聚类索引性能并不差。
- 订单处理类的业务需要用支持事务的数据库。

- 改用更高读写的数据存储引擎不一定带来性能提升，有可能一些查询对存储引擎不友好
- 数据仓库：数据量10T以上级别
- 转换表的存储引擎
  - `alter table mytable ENGINE = xxxx`
  
  - 执行时间长：逐行复制数据并且在原表加锁
  
  - 转换后将失去特性，即使转回也无法恢复
  
  - 全量复制，执行过程可以加锁，创建一个同数据的新引擎表
  
    - `create table inno_table LIKE myisam_table`
  
    - `alter table inno_table ENGINE=InnoDB`
  
    - `insert into inno_table select * from myisam_table`
  
    - 数据量大的时候可以分批处理
  
      ```mysql
      start transaction;
      insert into inno_table 
          (select * from myisam_table where id between x and y);
      commit;    
      ```

- 导入导出工具 **mysqldump**

  

## C2 benchmark 基准测试

- 尽量简单直接，结果容易相互比较，成本低且易执行
- 策略
  - 集成式 full-stack
  - 单组件式 single-component 
- 只测试 MySQL 
  - 比较不同schema或查询的性能
  - 针对某个具体问题
  - 测试迭代 
- 明确测试目标，以确认指标
  - 吞吐量：单位时间事务处理数
  - 响应时间或延迟：百分比响应时间（95%都在5ms内）
  - 并发
    - web无状态，应该测并发请求
    - 数据库可以有大量连接，但可能同时只有少数连接在执行查询
    - 同时工作的线程或者连接数
    - 一般用于测试不同并发下的性能
  - 可扩展性
    - 增加资源或者增加负载
  - 测试应注意避开低级错误
    - 数据集：要全集，非均匀数据分布，均匀分布参数。
    - 多用户和单用户不同，分布式测试需要分布式服务器。
    - 要和真实用户行为匹配，不一定是连续请求。
    - 反复执行同一个查询（缓存利用）
    - 检查错误日志，某个查询变快了可能是语法错误导致未执行。
    - 系统预热需要时间
  - 基准测试应该运行足够长的时间，以测试文档状态的性能
    - 不能只做一系列短期测试（eg. 60s）来评估系统性能
  - 测试需要收集被测试系统的信息，系统状态以及各性能指标
  - 数据图形能发现一些问题
  - 集成测试工具
    - 