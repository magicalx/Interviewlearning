# 一条语句是如何执行的

mysql的逻辑架构图![逻辑架构图](/数据库/img/mysql-1.png)  
mysql 可以分为Server层和存储引擎层两部分。  
Server层包括连接器、查询缓存、分析器、优化器、执行器。涵盖MySQL的大多数核心服务功能，以及所有内置函数(日期、时间、数学和加密函数等)，所有跨存储引擎的功能：存储过程、触发器、视图等。  
存储引擎负责数据的存储和提取，插件式的，支持Innodb、MyISAM、Memory等多个存储引擎。默认是Innodb。   
## 连接器
连接器负责跟客户端建立连接、获取权限、维持和管理连接。  
* 输入用户名或密码不对，收到Access Denied for user
* 用户密码认证通过，连接器会到权限表里查出user拥有的权限，之后的权限判断都依赖此时读到的权限。
* 客户端连接长时间没动静，连接器会自动断开，参数wait_timeout控制，默认是8小时。

由于数据库建立连接过程比较复杂，我们应该尽量减少连接的动作，尽量使用长连接。 使用长连接后会发现MySQL占用内存涨的特别快。  
解决方案：  
1.定期断开连接，使用一段时间或者程序里面判断执行过一个占用内存的大查询后，断开连接，之后查询重连接。  
2.使用mysql5.7+，每次执行一个比较大的操作后，通过mysql_reset_connnection来重新初始化连接资源。

## 查询缓存
mysql拿到查询语句后，会去查询缓存，缓存是以key-value形式，key查询语句，value查询结果，直接缓存在内存中；查询缓存失效非常频繁，只要表一更新，所有的查询缓存都被清空。mysql提供了按需使用的方式，设置query_cache_type设置DEMAND，使用SQL_CACHE显示指定。mysql8.0直接将查询缓存的整块功能删掉了。

## 分析器
需要对SQL语句进行解析，分析器会先做词法分析，输入的一条SQL语句，mysql需要识别出里面的字符串分别是什么、代表什么。  
做完词法分析，接着做语法分析，根据语法分析，判断SQL语句是否满足mysql语法。  

## 优化器
优化器是在表里面有多个索引的时候，决定使用哪个索引；或者语句有多表关联(join)的时候，决定各个表的连接顺序。 

## 执行器
通过分析器知道了要做什么，优化器知道了该怎么做，进入执行阶段。  
执行的时候，首先判断user对这个表有不有执行查询的权限，如果没有，机会返回权限错误。

## 问题
如果表中没有字段k,执行语句`select * from T where k = 1`,报错误是在哪个阶段？   
优化器阶段，语法分析完后，优化器进行选择索引啥的，会检查表有不有这个字段，不需要打开表，不是执行器