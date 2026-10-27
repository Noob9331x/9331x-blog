# 想要精通MySQL

## 一、MySQL基础

### 1、MySQL的整体架构是什么？

核心回答：MySQL的整体架构可以分为两层：Server层和存储引擎层。Server层负责连接管理、SQL解析、权限校验、优化器、执行器、缓存等通用能力；存储引擎层负责真正的数据存储和读取，比如InnoDB、MyISAM。最常用的InnoDB还提供事务、行锁、MVCC、redo log、undo log等能力。

一句话理解：

```
server层    #负责理解和执行SQL
==========
存储引擎层    #负责真正读写数据
=========
```

整体结构：

```
 client
   ↓
 连接器
   ↓
查询缓存/解析器/预处理器
   ↓  
 优化器
   ↓ 
 执行器
   ↓ 
存储引擎
   ↓ 
磁盘数据
```

#### 1）第一层：Server层

Server层是MySOL的通用层。
不管底层用什么存储引擎，比如：InnoDB MyISAM Memory

SQL进入MySQL后，都会先经过Server层。
Server层主要包括：

```
连接器
权限校验
解析器
预处理器
优化器
执行器
binlog
函数和内置能力
```

2）连接器
客户端连接MySQL时，先到连接器。连接器负责：

```
建立连接
认证用户名密码
检查权限
维护连接状态
例如：  mysql -u root -p
```

连上后，连接器会记录这个用户有哪些权限

3）解析器

SOL进来后，解析器负责分析SQL。比如：

```
select * from user where id = 1;
```

解析器会做：词法分析和生成语法树

如果SQL写错解析器就会报语法错误。

4)预处理器

预处理器会做一些语义检查。比如：

```
表是否存在
字段是否存在
别名是否
正确权限是否满足

select age from user;
```

如果age字段不存在，就会报错。

5）优化器

优化器负责决定SQL怎么执行更高效。比如：

``` 
select * from user where age = 18 and city = 'Beijing';
```

可能有多个索引：

``` 
idx_age 
idx_city
idx_age_city
```

优化器会选择它认为成本最低的执行计划。

它会决定：

``` 
用哪个索引
表连接顺序
是否走全表扫描
如何排序
如何聚合
```

6）执行器

执行器负责真正执行SQL。
它会根据优化器生成的执行计划，调用存储引擎接口，例如：

```  
从索引中读取记录
回表查询完整数据
过滤where条件
返回结果给客户端
```

执行器不直接管理底层数据文件，而是通过存储引擎读写

#### 7）第二层：存储引擎层

存储引擎层负责真正的数据存储和读取。

常见存储引擎：

```
InnoDB     #最常用的
MyISAM
Memory
```

InnoDB支持：

``` 
事务
行级锁
MVCC
崩溃恢复
外键
redo log
undo log
```

8)Server层和InnoDB的日志

MySQL里有两个很重要的日志：

``` 
binlog
redo log
```

binlog:

``` 
Server层日志
记录逻辑操作
用于主从复制和数据恢复
```

redo log:

``` 
InnoDB日志
记录物理页修改
用于崩溃恢复
保证事务持久性
```

事务提交时，MySQL会通过两阶段提交协调binlog和redolog，保证一致。

#### 小结

MysQL整体架构可以分为Server层和存储引擎层

- Server层负责连接管理、权限校验、sQL解析、预处理、优化器、执行器以及binlog等通用功能。
- 存储引擎层负责真正的数据存储和读取，不同表可以使用不同存储引擎，最常用的是innoDB。
- SQL执行时，客户端请求先经过连接器认证和限校验，然后由解析器生成语法树，预处理器检查表和字段，优化器选择执行计划，执行器按照计划调用存储引擎接口读取或修改数据。innoDB提供事务、行锁、Mvcc、redo log、undo  log和崩溃恢复能力。

``` 
Mysql server层负责执行 sql:
InnoDB 存储引擎层负责真正存数据,管事务和恢复。
```

## 二、InnoDB和MyISAM有什么区别？