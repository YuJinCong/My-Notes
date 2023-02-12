# MySQL 概念

---

## MySQL 数据库

MySQL 是一种关系型数据库。开源免费，并且方便扩展。在 Java 开发中常用于保存和管理数据。默认端口号 3306。

MySQL 数据库主要分为 Server 和存储引擎两部分，现在最常用的存储引擎是 InnoDB。

---

## 指令执行过程

MySQL 数据库接收到用户指令后，首先由 Server 负责对数据操作的分析、处理和优化，再交给存储引擎执行数据存取操作。

### 连接器

连接器负责用户登录数据库时的身份认证，校验账户密码。校验通过后连接器会连接到权限表，并读取该用户的所有权限。如果连接未断开，即使该用户权限被管理员修改也不受影响。

### ~~查询缓存~~

缓存 SELECT 语句以及返回的结果。收到查询语句会首先和缓存比对，如果相同就直接从查询缓存里返回数据。

更新表后，这个表上的所有的查询缓存都会被清空。这导致实际使用场景中查询缓存的作用非常少，在 MySQL 8.0 版本后移除。

### 分析器

如果查询语句未命中缓存，或者是更新语句，那么将由分析器负责分析 SQL 语句的用途。

1. 词法分析：提取关键字，提取 SQL 语句的关键元素，明确 SQL 语句的功能。

2. 语法分析：判断 SQL 语句是否正确，是否符合 MySQL 的语法。如果不符合语法则返回错误信息。

### 优化器

明确 SQL 语句功能后，由优化器负责选择尽可能最优的执行方案。比如多个索引的时候选择索引，多表查询的时候选择关联顺序。

### 执行器

确定执行方案后，由执行器负责校验该用户有没有权限，并交由存储引擎执行语句，然后从存储引擎返回数据。

---

## 存储引擎

实际执行对数据库数据的存取。目前 MySQL 默认使用 InnoDB 引擎。相比于过去使用 MyISAM 引擎，有以下几个优势：

1. 索引：数据文件本身是主索引。
2. 外键：支持外键。
3. 事务：添加本地日志，支持安全恢复；支持行级锁，提高并发度。
4. 并发：支持多版本并发控制，提升性能。

### 索引

#### 存储结构

MySQL 数据库使用以下两种数据结构存储和查找数据：

1. **B+ 树**：（默认）适用于连续查询多条数据。
2. **哈希表**：适用于查询单条数据。

#### 索引类型

索引名称|索引类型|字段类型|备注
-|-|-
PRIMARY KEY|主索引|主键|字段值不能重复，也不能为空。
INDEX|普通索引|自定义字段|无，效率低。
UNIQUE|唯一索引|自定义字段|字段值不能重复，效率高。
FULLTEXT|文本索引|自定义字段|无，用于文本检索。

- **主索引**

在 InnoDB 存储引擎中数据文件本身就是主索引（聚簇索引）：数据以 B+ 树形式存储，根据主键值进行排序。

我们可以为其他字段建立辅助索引（非聚簇索引），以提高对字段的查询速度，但同时会降低表的更新速度。在辅助索引中记录主键值而不是字段地址：根据辅助索引查找后，仍需要根据主键值在主索引中查询数据。

- **组合索引**

索引内可以包含多个字段，N 个字段的组合索引实际建立了 N 个索引。

对 a/b/c 三个字段建立的组合索引，实际会先在 a 索引中查找，再到 a/b 索引中查找，最后在 a/b/c 索引中查找。


### 视图

视图是一个虚拟表，不实际存储数据。其内容会通过查询其他表得到，在引用视图时动态生成。

1. 权限管理：表的权限管理不能限制到具体的行和列，但通过视图则可以限制用户能得到的结果集。
2. 数据独立：表的结构发生变化，不会对用户使用视图查询到的数据产生影响。


### 外键

从表通过外键关联到主表的主键，建立数据表之间的关系。

- 优点：保障数据的一致性和完整性。
- 缺点：增加数据之间的耦合度，难以集群。**因此不推荐使用外键。**

#### 删除策略

对主表的数据进行 UPDATE/DELETE 操作时，将会会影响到关联的从表。

外键模式|删除策略
-|-
RESTRICT|（默认）从表有相关数据时，主表不能更新/删除。
CASCADE|  主表记录更新/删除时，从表相关记录也会被更新/删除。
SET NULL| 主表数据更新/删除时，从表相关记录的外键值被设为 NULL。
NO ACTION| 啥也不做



### 日志

当数据库数据发生更改时，用日志记录数据库操作。当发生错误或者冲突时，可以进行回滚。保证数据的一致性。

#### bin log 归档日志

最开始 MySQL 并没与 InnoDB 引擎，其他存储引擎只有通用的 bin 日志用来归档（位于 server 层）。

InnoDB 引擎完成主存数据更新后向执行器提交，由 bin 日志记录操作。如果主存数据已更新，且 bin 日志没有被写入时数据库崩溃，后续进行机器备份的时候就会丢失原有数据。这导致数据没有安全恢复的能力：一旦数据库发生异常重启，之前提交的记录都会丢失。

#### redolog 重做日志

MySQL 引入 InnoDB 引擎后，自带了 redo 日志。用于数据库发生异常重启时系统记录的恢复。

1. InnoDB 引擎完成主存数据更新但还未提交时，由 redo 日志记录操作并进入 prepare 状态。
2. InnoDB 引擎向执行器提交时，由 bin 日志记录操作。
3. 提交完成后执行器通知 InnoDB 引擎，redo 日志进入 commit 状态。

如果 bin 日志没有被写入时数据库崩溃，后续进行机器备份的时候就会按照 redo 日志恢复数据。

如果 bin 日志已经写完但 redo 日志还处于 prepare 状态时数据库崩溃。MySQL 会判断 redo 日志是否完整，如果完整就立即提交。否则再判断 bin 日志是否完整，如果完整就提交 redo 日志，不完整就回滚事务。这样就解决了数据一致性的问题。

---

## 事务

事务是逻辑上的一组操作，要么都执行，要么都不执行。保障数据之间的同步。

### 事务特性 ACID

- **原子性**： 事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不起作用；
- **一致性**： 执行事务前后，数据保持一致，多个事务对同一个数据读取的结果是相同的；
- **隔离性**： 并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间数据库是独立的；
- **持久性**： 一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。

### 并发事务潜在问题

- **丢失修改**

事务（T1）修改数据的过程中，另一个并发事务（T2）也修改了该数据。导致事务（T1）对数据的修改丢失。

- **脏读**

事务（T1）修改数据但还未写入数据库时，另一个并发事务（T2）使用了该数据。导致事务（T2）读取数据可能是不正确的。

- **不可重复读**

事务（T1）两次读取数据的过程中，另一个并发事务（T2）修改了该数据。导致事务（T1）两次读取数据的结果不同。

- **幻读**

事务（T1）两次读取数据集合的过程中，另一个并发事务（T2）插入或删除了部分数据。导致事务（T1）两次读取数据的结果不同。

### 数据锁

存储引擎通过给数据加锁来保障事务性。MyISAM 引擎只支持表级锁，而 InnoDB 存储引擎支持行级锁和表级锁，默认为行级锁。

- **表级锁**：对当前操作的整张表加锁，实现简单，资源消耗也比较少，加锁快，不会出现死锁。但触发锁冲突的概率最高，并发度低。

- **行级锁**：只针对当前操作的数据行加锁。大大减少数据库操作的冲突，并发度高。但加锁的开销也最大，可能会出现死锁。

InnoDB支持三种行锁定方式：

- **间隙锁**：锁定索引的记录间隙，确保索引记录的间隙不变。

Next-Key Lock 是行级锁和间隙锁的组合使用。当 InnoDB 扫描索引记录的时候，会首先对索引记录加上行锁（Record Lock），再对索引记录两边的间隙加上间隙锁（Gap Lock）。其他事务就不能在这个间隙修改或者插入记录。间隙锁是针对事务隔离级别为可重复读或以上级别，可以有效防止幻读的发生。

### 事务隔离级别

- **READ-UNCOMMITTED(RU) 读取未提交** 

事务进行读操作时允许其他事务访问，事务进行写操作将会禁止其他事务写。

- **READ-COMMITTED(RC) 读取已提交**

事务进行读操作时允许其他事务访问，事务进行写操作将会禁止其他事务读写。

- **REPEATABLE-READ(RR) 可重复读**

事务进行读操作时会禁止其他事务写，事务进行写操作将会禁止其他事务读写。

- **SERIALIZABLE 可串行化**

事务进行读写操作时，都会禁止其他事务读写。

隔离级别 | 丢失修改 | 脏读 |  不可重复读 |幻读
-|-|-|-|-
READ-UNCOMMITTED|×|√|√|√
READ-COMMITTED|×|×|√|√
REPEATABLE-READ|×|×|×|√
SERIALIZABLE|×|×|×|×

1. InnoDB 存储引擎默认支持的隔离级别是 REPEATABLE-READ(RR) ，且 InnoDB 在该事务隔离级别下使用 Next-Key Lock 锁算法，可以避免幻读。
2. InnoDB 存储引擎在分布式事务情况下一般会用到 SERIALIZABLE 隔离级别。


---

## MVCC

### MVCC 概念

MVCC 即多版本并发控制，维持一个数据的多个版本，使得读写操作没有冲突。从而提高数据库并发性能，做到即使有读写冲突时，也能不加锁非阻塞并发读。

是乐观锁的一整实现方式，就是每行都有版本号，保存时根据版本号决定是否成功。

### 读类型

- 当前读

 读取的是记录最新版本，同时会对读取的记录进行加锁保证其他并发事务不能修改
select lock in share mode(共享锁), select for update ; update, insert ,delete


- 快照读
  

可能读到的是数据之前的历史版本.在很多情况下，避免了加锁操作，降低了开销；

像不加锁的select操作就是快照。但如果隔离级别是最高级串行化，快照读会退化成当前读。

### MVCC 实现

在 InnoDB 存储引擎中，每行记录除了我们自定义的字段外，还会隐式记录：

1. 最近修改：记录创建这条记录/最后一次修改该记录的事务ID
2. 回滚指针：指向这条记录的上一个版本，存储于 rollback segment 里。
3. 隐藏主键：如果数据表没有主键，InnoDB 会自动产生一个自增的聚簇索引。
4. 删除标记：标记该记录是否已被删除。


事务进行快照读操作的时候生产的读视图(Read View)，在该事务执行的快照读的那一刻，会生成数据库系统当前的一个快照，记录并维护系统当前活跃事务的ID(当每个事务开启时，都会被分配一个ID, 这个ID是递增的，所以最新的事务，ID值越大)。

InnoDB 会根据读取事务 ID 判断应该都什么时间段的数据。

在RC隔离级别下，是每个快照读都会生成并获取最新的Read View；而在RR隔离级别下，则是同一个事务中的第一个快照读才会创建Read View, 之后的快照读获取的都是同一个Read View。


---

## 数据类型

### 字符串类型

常用的字符串类型有定长字符串 CHAR 和变长字符串 VARCHAR 两种，必须用数字注明可容纳的字符数。

- CHAR(n) 表示固定容纳 n 个字符，当少于 n 个字符时，会使用空格填充。
- VARCHAR(n) 表示最多容纳 n 个字符，当少于 n 个字符时不会补空。起始位和结束位需要额外 3 字节。

> 英文字母单个字符占 1 字节。汉字单个字符 UTF-8 编码占 3 字节，GBK 编码占 2 字节。

对于长字符串数据可以用 TEXT 或 BLOB 类型存储，固定占用 65535 字节。其中 TEXT 保存文本格式，BLOB 保存二进制格式。如果需要存储更短或更长的字符串类型数据，可以使用 TEXT/BLOB 的扩充类型：如 TINYTEXT、MEDIUMTEXT 和 LONGTEXT。

类型名称|大小（字节）|数据库类型|JAVA 类型
-|-|-|-	 
CHAR(n)|N(0-255)|CHAR|String		 	 	 	 
VARCHAR(n)|N+3(0-65535)|VARCHAR|String
TEXT|65535|VARCHAR|String	
BLOB|65535|BLOB|byte[]

CHAR 类型最大只能容纳 255 字节数据，已不推荐使用。目前 VARCHAR 支持容纳最大 65535 字节数据，且长度不固定能有效节省数据库空间，推荐尽量使用 VARCHAR 数据类型取代 TEXT。

TEXT/BLOB 等大数据类型不支持 MySQL 内存临时表，进行排序等操作必须在磁盘中进行。尽量不要使用，一定要用建议单独建表。


### 整型 & 浮点型

#### 布尔型

布尔型数据用 BIT 表示。

类型名称|大小（字节）|取值|JAVA 类型
-|-|-|-	 
BIT|1|0 or 1|Boolean

#### 整型

整形数据一般用 INT/INTEGER 表示，固定占用 4 字节。如果需要存储更短或更长的整型数据，可以使用 INT 的扩充类型：如 TINYINT、SMALLTEXT 和 MEDIUMTEXT 和 BIGINT。

在声明整型数据时也可以注明显示位宽，如 int(n)。在整形数据不足 n 位时会自动补零，几乎没有任何用处。

类型名称|大小（字节）|表示范围|JAVA 类型
-|-|-|-	 
TINYINT| 1 |	(-128，127)	(0，255)| Integer
SMALLINT|2 |	(-32 768，32 767)(0，65 535)|	Integer
MEDIUMINT|3 |(-8 388 608，8 388 607)(0，16 777 215)	|Integer
INT/INTEGER|	4 |	(-2 147 483 648，2 147 483 647)	(0，4 294 967 295)|	Integer
BIGINT|	8 |非常大|	BigInteger

#### 浮点型

常用的浮点型数据类型有 FLOAT/DOUBLE ，FLOAT 类型固定占用 4 字节，DOUBLE 类型固定占用 8 字节。但 FLOAT/DOUBLE 只是近似存储，在数据库中我们常用 DECIMAL 类型记录金额，在数据库中实际以字符串形式存储，以确保不会产生任何误差。

- DECIMAL(M,D) M 表示最大位数，D 表示小数点右侧的位数。如 DECIMAL(5,2) ，小数点前 3 位，小数点后 2 位。

类型名称|大小（字节）|表示范围|JAVA 类型
-|-|-|-	 
FLOAT| 4 |	| Float
DOUBLE|8 |	| Double
DECIMAL(M,D)|M + 2 | | BigDecimal	

### 日期类型

java.sql 包内有专用 Java 类型匹配，注意数据类型必须是 `java.sql.Date`，而不是 `java.util.Date` 。

类型|大小（字节）|格式|表示范围|JAVA 类型
-|-|-|-|-	
YEAR|1|`YYYY`|`1901/2155`|Date
DATE|3|`YYYY-MM-DD`|`1000-01-01/9999-12-31`|Date
TIME|3|`HH:MM:SS`|`-838:59:59'/'838:59:59`|Time
TIMESTAMP|4|`YYYY-MM-DD HH:MM:SS`| `1970-01-01 00:00:00/2038-1-19 11:14:07`|Timestamp
DATETIME|8|`YYYY-MM-DD HH:MM:SS`|`1000-01-01 00:00:00/9999-12-31 23:59:59`|Timestamp


### 枚举 & 集合

```sql
create table tab(  
   gender enum('male','remale','secret')            -- gender 属性为枚举类型
);  

insert into tab values ('remale');  

 select * from tab where gender=2;                  -- 两者等价
  select * from tab where gender= 'remale';
```

记录字符串，但底层数据实际以2个字节的整型(smallint)保存。

在已知的值中进行单选。最大数量为65535.按保存的位置顺序，从1开始逐一递增。

NULL值的索引是NULL。空字符串错误值的索引值是0。

```sql
create table tab(  
   gender set('male','remale','secret')            -- gender 属性为集合类型
);  

insert into tab values ('male', 'remale');  
```

记录字符串，但底层数据实际以8个字节的整型(bigint)保存。

在已知的值中进行多选。最多有 64 个成员。


```sql
-- 查询 flag 字段包含 a,b 的字段
mysql -> select * from table_name where FIND_IN_SET('a,d', flag);
```

# MySQL 指令

---

## 基本概念

### SQL 指令

SQL 指令是用于访问和处理数据库的标准的计算机语言。对于 MySQL 等常用数据库都可以通过使用 SQL 访问和处理数据系统中的数据。

### 注意事项

1. SQL 对大小写不敏感。
2. 标识符应避免与关键字重名！可用反引号（`）为标识符包裹。
3. 注释
   - 单行注释： `# 注释内容`
   - 多行注释： `/* 注释内容 */`
   - 单行注释： `-- 注释内容` 
4. 模式通配符
   - 匹配任意单个字符： `_`
   - 匹配任意数量字符，包括 0 个：`%`
   - 单引号需要进行转义： `\'` 
5. 清除已有语句：`\c`

---

## 服务指令

### 启动/终止服务

```bash
net start mysql           # 启动本机 MySQL 运行
net stop mysql            # 终止本机 MySQL 运行
```

### 连接/断开服务

MySQL 服务运行时，输入连接指令即可连接 MySQL 数据库。

需要输入的属性分别为 (h)IP 地址、(P)端口号、(u)用户名、(p)密码。 端口号若为 3306 可省略，密码可空缺。

```bash
# 本地连接
mysql -h localhost -u root -p 

# 远程连接
mysql -h 10.0.0.51 -P 3306 -u root -p 123456

# 断开连接
mysql> exit
mysql> quit
mysql> /p
```

---

## 管理指令

### 用户管理

MySQL 数据库的全部用户信息保存在 `mysql 库 / user 表`内，用户含有以下属性：

- **user 属性**：用户名
- **host 属性**：允许用户登入的网络
- **authentication_string 属性**：密码

#### 增删改查

能够对用户进行增删改查操作，需要当前用户拥有非常高的数据库权限。

```sql
-- 增加用户(CREATE)
mysql> CREATE USER 'boy'@'localhost' IDENTIFIED BY '';                -- 创建用户 boy 允许从本地网络登录
mysql> CREATE USER 'girl'@'10.0.0.%' IDENTIFIED BY '123456';          -- 创建用户 girl 允许从特定网络登录

-- 删除用户(DROP)
mysql> DROP USER 'girl'@'10.0.0.%';

-- 修改用户(ALTER)
mysql> ALTER USER 'boy'@'localhost' IDENTIFIED BY '123456';

-- 重命名用户(RENAME)
mysql> RENAME USER 'boy'@'localhost' TO 'man'@'localhost';

-- 设置密码
mysql> SET PASSWORD = PASSWORD('123456');                              -- 为当前用户设置密码
mysql> SET PASSWORD FOR 'boy'@'localhost' = PASSWORD('123456');        -- 为指定用户设置密码

-- 查询全部用户信息(DESC/SELECT)
mysql> DESC mysql.user;                                            
mysql> SELECT user,host,authentication_string FROM mysql.user     
```


#### 权限管理

用户权限分为非常多种，包括全局权限、库权限、表权限、列权限等。

```sql
-- 赋予权限(GRANT)
mysql> GRANT SELECT,INSERT ON *.*             -- 赋予用户选择插入权限（所有库的所有表）
    -> TO 'boy'@'localhost'                   -- 不存在将新建用户
    -> IDENTIFIED BY '123456'                 
    -> WITH GRANT OPTION;                     -- （可选）允许用户转授权限

-- 撤消权限(REVOKE)
mysql> REVOKE INSERT ON *.*
    -> FROM 'boy'@'localhost';

-- 查看权限
mysql> SELECT Host,User,Select_priv,Grant_priv
    -> FROM mysql.user
    -> WHERE User='testUser';
```



### 数据库管理

MySQL 内划分为多个互相独立的数据存储区域，调用数据库指令时必须提前声明要使用的数据库。

- **数据库选项信息**

| 属性          | 含义     | 备注                      |
| ------------- | -------- | ------------------------- |
| CHARACTER SET | 编码方式 | 默认为 utf8mb4            |
| COLLATE       | 校对规则 | 默认为 utf8mb4_general_ci |

```sql
-- 查看所有数据库
mysql> SHOW DATABASES;

-- 进入/切换数据库
mysql> USE mydb;

-- 查看当前数据库
mysql> SELECT DATABASE();

-- 创建数据库
mysql> CREATE DATABASE [IF NOT EXISTS] mydb;
mysql> CREATE DATABASE [IF NOT EXISTS] mydb CHARACTER SET utf8mb4;

-- 删除数据库
mysql> DROP DATABASE [IF EXISTS] mydb;

-- 查看数据库选项信息
mysql> SHOW CREATE DATABASE mydb;

-- 修改数据库选项信息
mysql> ALTER DATABASE mydb CHARACTER SET utf8;
```

### 表管理

- **表属性**

| 属性            | 含义         | 备注                 |
| --------------- | ------------ | -------------------- |
| CHARSET         | 字符集       | 默认使用数据库字符集 |
| ENGINE          | 存储引擎     | 默认为 InnoDB        |
| DATA DIRECTORY  | 数据文件目录 |                      |
| INDEX DIRECTORY | 索引文件目录 |                      |
| COMMENT         | 表注释       |                      |

*如果表标记为 TEMPORARY 则为临时表，在连接断开时表会消失。*

- **列属性**

| 属性           | 含义     | 备注                                                         |
| -------------- | -------- | ------------------------------------------------------------ |
| PRIMARY KEY    | 主键     | 标识记录的字段。可以为字段组合，不能为空且不能重复。         |
| INDEX          | 普通索引 | 可以为字段组合，建立普通索引。                               |
| UNIQUE         | 唯一索引 | 可以为字段组合，不能重复，建立唯一索引。                     |
| NOT NULL       | 非空     | （推荐）不允许字段值为空。                                   |
| DEFAULT        | 默认值   | 设置当前字段的默认值。                                       |
| AUTO_INCREMENt | 自动增长 | 字段无需赋值，从指定值（默认 1）开始自动增长。表内只能存在一个且必须为索引。 |
| COMMENT        | 注释     | 字段备注信息。                                               |
| FOREIGN KEY    | 外键     | 该字段关联到其他表的主键。默认建立普通索引。                 |

#### 表操作

```sql
-- 查看所有表
mysql> SHOW TABLES;

-- 创建表
mysql> CREATE [TEMPORARY] TABLE [IF NOT EXISTS] student
       (
           id INT(8) PRIMARY KEY AUTO_INCREMENT=20190001,
           name VARCHAR(50) NOT NULL,
           sex INT COMMENT 'Male 1，Female 0',
           access_time DATE DEFAULT GETDATE(),
           major_id INT FOREIGN KEY REFERENCES major(id) 
       )ENGINE=InnoDB;

mysql> CREATE TABLE grade
       (
           student_id INT,
           course_id INT,
           grade INT,
           PRIMARY KEY (student_id,course_id),
           CONSTRAINT fk_grade_student FOREIGN KEY (student_id) REFERENCES student(id),
           CONSTRAINT fk_grade_course FOREIGN KEY (course_id) REFERENCES course(id)
       );

-- 删除表
mysql> DROP TABLE [IF EXISTS] student;

-- 清空表数据（直接删除表，再重新创建）
mysql> TRUNCATE [TABLE] student;

-- 查看表结构
mysql> SHOW CREATE TABLE student;
mysql> DESC student;

-- 修改表属性
mysql> ALTER TABLE student ENGINE=MYISAM;

-- 重命名表
mysql> RENAME TABLE student TO new_student;
mysql> RENAME TABLE student TO mydb.new_student;      

-- 复制表
mysql> CREATE TABLE new_student LIKE student;                  -- 复制表结构
mysql> CREATE TABLE new_student [AS] SELECT * FROM student;    -- 复制表结构和数据


-- 检查表是否有错误
mysql> CHECK TABLE tbl_name [, tbl_name] ... [option] ...
-- 优化表
mysql> OPTIMIZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tbl_name [, tbl_name] ...
-- 修复表
mysql> REPAIR [LOCAL | NO_WRITE_TO_BINLOG] TABLE tbl_name [, tbl_name] ... [QUICK] [EXTENDED] [USE_FRM]
-- 分析表
mysql> ANALYZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tbl_name [, tbl_name] ...
```

#### 列操作

```sql
-- 添加字段
mysql> ALTER TABLE student ADD [COLUMN] age INT;               -- 默认添加在最后一行
mysql> ALTER TABLE student ADD [COLUMN] age INT AFTER sex;     -- 添加在指定字段后
mysql> ALTER TABLE student ADD [COLUMN] age INT FIRST;         -- 添加在第一行

--修改字段
mysql> ALTER TABLE student MODIFY [COLUMN] id SMALLINT;        -- 修改字段属性
mysql> ALTER TABLE student CHANGE [COLUMN] id new_id INT;      -- 修改字段名

-- 删除字段
mysql> ALTER TABLE student DROP [COLUMN] age;   

-- 编辑主键
mysql> ALTER TABLE student ADD PRIMARY KEY(id,age);           
mysql> ALTER TABLE student DROP PRIMARY KEY;                 

-- 编辑外键
mysql> ALTER TABLE student ADD CONSTRAINT fk_student_class FOREIGN KEY(cid) REFERENCES class(id);
mysql> ALTER TABLE student DROP FOREIGN KEY fk_student_class;  
```

---

## 数据指令

### 增删改查

**插入数据**，如果已有主键值则插入数据失败。

```sql
mysql> INSERT INTO student
    -> (ID,name,grade)
    -> VALUES(755,'王东浩',80);
```

**插入并替换数据**，如果已有主键值则先删除再插入。

```sql
mysql> REPLACE INTO student
    -> (ID,name,grade)
    -> VALUES(755,'王东浩',80);
```

**更新数据**

```sql
mysql> UPDATE student
    -> SET name='孙鹏',grade=60
    -> WHERE id=753;
```

**删除数据**

```sql
mysql> DELETE FROM student
    -> WHERE id=754;
```

**查询数据**

```sql
mysql> SELECT id,name FROM student               -- 按条件查询数据
    -> WHERE id BETWEEN 753 and 755;

mysql> SELECT * FROM student;                    -- 查询全部数据
```

#### 条件语句

- DISTINCT 关键字用于对查询结果去重，必须放于所有字段前。只有多个字段全部相等才会被去重。

```sql
mysql> SELECT DINTINCE age,sex FROM student;     -- 查询数据并去重
```

- WHERE 语句用于指定 更新/删除/查询 的操作范围，如果不设定范围将对全部数据进行操作。

```sql
mysql> SELECT * FROM student WHERE id = 100;
mysql> SELECT * FROM student WHERE id != 100;
mysql> SELECT * FROM student WHERE id [NOT] BETWEEN 30 AND 50;
mysql> SELECT * FROM student WHERE id [NOT] IN (30, 35 ,50);
mysql> SELECT * FROM student WHERE grade IS [NOT] NULL;
```

- LIKE 语句用于对字符串进行模糊匹配：`%`代表任意多个字符 `_`代表一个字符 `/`代表转义

```sql
mysql> SELECT * FROM student WHERE name LIKE 'Tom%';
```


### 分组排序

#### 数据分组

| 分组函数     | 功能           |
| ------------ | -------------- |
| count        | 个数           |
| sum          | 总和           |
| max          | 最大值         |
| min          | 最小值         |
| avg          | 求平均值       |
| group_concat | 组内字符串拼接 |

1. GROUP 语句指定数据的分组方式，如果不含则默认把全部数据合并为一条数据。（本质是生成临时表）
2. AS 关键字为表或者列起别名，可省略。
3. HAVING 语句对分组后的结果进行筛选。

```sql
-- 查询班级总数
mysql> SELECT COUNT(*) FROM class;                    -- 全部合并

-- 查询各年级人数
mysql> SELECT grade, SUM(class.student_num) AS nums 
    -> FROM class 
    -> GROUP BY grade                                 -- 各班数据按年级合并
    -> HAVING SUM(class.student_num) > 200;           -- 筛选人数大于 200 的年级
```

#### 数据排序

- ORDER 语句指定数据显示顺序，ASC 为升序 / DESC 为降序。
- LIMIT 语句对排序后的数据进行筛选，指定起始序号和总数量。

```sql
-- 查询学生信息
mysql> SELECT * 
    -> FROM student 
    -> ORDER BY grade DESC, ID ASC                   -- 按成绩降序排列，若相同按学号升序排列
    -> LIMIT 10,20;                                  -- 筛选第 11 - 30 名
```


### 多表查询

#### 嵌套查询

1. FROM 型：子语句返回一个表，且必须给子查询结果取别名。
2. WHERE 型：子语句返回一个值，不能用于 UPDATE。

```sql
-- FROM 型
mysql> SELECT * 
    -> FROM (SELECT * FROM tb WHERE id > 0) AS subfrom 
    -> WHERE id > 1;

-- WHERE 型
mysql> SELECT * 
    -> FROM tb
    -> WHERE money = (SELECT max(money) FROM tb);
```


#### 合并查询

1. 默认为 DISTINCT 形式，不同表查询到的相同数据只展示一个。
2. 设置为 ALL 则不同表查询到的相同结果重复展示。

```sql
-- DISTINCT 形式
mysql> (SELECT * FROM student WHERE id < 10) 
    -> UNION
    -> (SELECT * FROM student WHERE id > 20);

-- ALL 形式
mysql> (SELECT * FROM student1) 
    -> UNION ALL 
    -> (SELECT * FROM student2);
```

#### 连表查询

- **内连接 INNER JOIN**：（默认）未指定连接条件时，自动查找相同字段名匹配连接条件。

```sql
mysql> SELECT s.id,s.name,c.name
    -> FROM student s JOIN class c
    -> ON e.cid = c.id;

mysql> SELECT * 
    -> FROM student s, class c 
    -> WHERE s.id = c.id; 
```

- **交叉连接 CROSS JOIN**：未指定连接条件时，视为无连接条件。

```sql
mysql> SELECT *
    -> FROM boy CROSS JOIN girl;                 -- 显示所有交配可能


mysql> SELECT *
-> FROM boy, girl;                               -- 等价写法
```

- **外连接 OUTER JOIN**：如果数据不存在，也会出现在连接结果中。

  - **LEFT JOIN**：左表数据一定显示，没有匹配右表数据用 null 填充。
  - **RIGHT JOIN**：右表数据一定显示，没有匹配左表数据用 null 填充。
  - **FULL JOIN**：两表数据一定显示，没有匹配数据用 null 填充。

```sql
mysql> SELECT s.id,s.name,c.name                   -- 显示学生的班级信息
    -> FROM student s LEFT JOIN class c            -- 没有班级的学生也会显示
    -> ON s.cid = c.id;

-- 先筛选再连接（效率等价，但如果有大量重复值提前筛选可以提高效率）
mysql> SELECT s.id,s.name,c.name    
    -> FROM student s LEFT JOIN (SELECT DINTINCT id, name FROM class) c       
    -> ON s.cid = c.id;
```



---

## 高级指令

### 索引

- **索引类型**

| 索引名称    | 索引类型 | 字段类型   | 备注                         |
| ----------- | -------- | ---------- | ---------------------------- |
| PRIMARY KEY | 主索引   | 主键       | 字段值不能重复，也不能为空。 |
| INDEX       | 普通索引 | 自定义字段 | 无，效率低。                 |
| UNIQUE      | 唯一索引 | 自定义字段 | 字段值不能重复，效率高。     |
| FULLTEXT    | 文本索引 | 自定义字段 | 无，用于文本检索。           |

```sql
-- 查询索引
mysql> SHOW INDEX FROM student;

-- 创建索引
mysql> CREATE [UNIQUE|FULLTEXT] INDEX idx_student_age 
    -> [USING BTREE]                                           -- 指定索引类型，默认 B+ 树
    -> ON student(age);                                        -- 指定索引属性

mysql> ALTER TABLE student ADD INDEX [idx_student_age](id,age);   
mysql> ALTER TABLE student ADD UNIQUE [uniq_student_age](age);         
mysql> ALTER TABLE student ADD FULLTEXE [ft_student_age](age);  

-- 删除索引
mysql> DROP INDEX idx_student_age ON student;

mysql> ALTER TABLE student DROP INDEX idx_student_age;                 
```


### 视图

**视图算法**

| 算法      | 名称         | 含义                                         |
| --------- | ------------ | -------------------------------------------- |
| UNDEFINED | 未定义(默认) | MySQL 自主选择相应的算法。                   |
| MERGE     | 合并         | 视图的查询语句，与外部查询需要先合并再执行。 |
| TEMPTABLE | 临时表       | 将视图执行完毕后形成临时表，再做外层查询.    |


**更新选项**

| 算法    | 名称       | 含义                               |
| ------- | ---------- | ---------------------------------- |
| CACADED | 级联(默认) | 满足所有视图条件才能进行数据更新。 |
| LOCAL   | 本地       | 满足本视图条件就能进行数据更新。   |

```sql
-- 创建视图
mysql> CREATE VIEW view_student
    -> AS (SELECT * FROM student);

mysql> CREATE ALGORITHM = MERGE
    -> VIEW view_student
    -> AS (SELECT * FROM student)
    -> WITH LOCAL CHECK OPTION;        

-- 查看结构
mysql> SHOW CREATE VIEW view_student;

-- 删除视图
mysql> DROP VIEW [IF EXISTS] view_student;

-- 修改视图结构（慎用）
mysql> ALTER VIEW view_student
    -> AS (SELECT * FROM student);
```


### 事务

开启事务后，所有输入的 SQL 语句将被认作一个不可分割的整体，在提交时统一执行。

如果在输入过程中出现问题，可以手动进行回滚。在输入过程中可以设置保存点。

```sql
-- 事务开启
mysql> START TRANSACTION;
mysql> BEGIN;
-- 事务提交
mysql> COMMIT;
-- 事务回滚
mysql> ROLLBACK;

-- 保存点
mysql> SAVEPOINT mypoint;                     -- 设置保存点
mysql> ROLLBACK TO SAVEPOINT mypoint;         -- 回滚到保存点
mysql> RELEASE SAVEPOINT mypoint;             -- 删除保存点
```


InnoDB 存储引擎支持关闭自动提交，强制开启事务：任何操作都必须要 COMMIT 提交后才能持久化数据，否则对其他客户端不可见。

```sql
mysql> SET AUTOCOMMIT = 0|1;             -- 0 表示关闭自动提交，1 表示开启自动提交。
```

### 锁定

MySQL 可以手动对表/行锁定，防止其它客户端进行不正当地读取和写入。

```sql
-- 锁定
mysql> LOCK TABLES student [AS alias];          
-- 解锁
mysql> UNLOCK TABLES;
```

### 触发器

触发程序是与表有关的数据库对象，监听记录的增加、修改、删除。当出现特定事件时，将激活该对象执行 SQL 语句。

1. MySQL 数据库只支持**行级触发器**：如果一条 INSERT 语句插入 N 行数据，语句级触发器只执行一次，行级触发器要执行 N 次。

2. 在触发器中，可以使用 `OLD` 和 `NEW` 表示该行的新旧数据。删除操作只有 `OLD`，增加操作只有 `NEW` 。

```sql
-- 查看触发器
mysql> SHOW TRIGGERS;

-- 创建触发器
mysql> CREATE TRIGGER my_trigger 
    -> BEFORE INSERT                    -- 触发时间 BEFORE/AFTER 触发条件 INSERT/UPDATE/DELETE
    -> ON student                       -- 监听表必须是永久性表
    -> FOR EACH ROW                     -- 行级触发器
    -> BEGIN
    -> INSERT INTO student_logs(id,op,op_time，op_id) VALUES(null,'insert',now(),new.id)
    -> END;

-- 删除触发器
mysql> DROP TRIGGER [schema_name.]trigger_name;
```

# MySQL 优化

---

## 分表设计

1. MySQL 数据库限制每个表最多存储 4096 列，并且每一行数据大小不能超过 65535 字节。
2. 数据量到达百万级以上时，会导致修改表结构、备份、恢复都有非常大的困难。
3. 数据量越大，装载进内存缓冲池时所占用的内存也就越大，缓冲池无法一次性装载时就会频繁进行磁盘 IO ，大大降低查询速率。

因此当表数据量过大时，就要进行分表操作：

- 水平分表：数据项分开存储。
- 垂直分表：按字段拆分。

### 分表原则

1. 经常一起使用的数据放到一个表中，避免更多的关联操作。

2. 尽量做到冷热数据分离，减小表的宽度，减少磁盘 IO，保证热数据的内存缓存命中率（表越宽，）；更有效的利用缓存，，避免读入无用的冷数据；

### 范式

对表进行逻辑拆分的准则。范式级别越高，存储数据冗余越小，但相应表之间的关系也越复杂导致难以维护。一般使用第三范式。

- **1NF** 第一范式 关系数据库一定符合条件

- **2NF** 第二范式 不能产生部分依赖（非主键字段不能被主键中部分字段唯一确定）

- **3NF** 第三范式 不能存在传递依赖（非主键字段不能被其它非主键字段唯一确定）

---

## 语句设计

### 语句查询

- **查询语句执行次数**

```sql
-- 查询数据库各类型语句执行次数
mysql> SHOW [SESSION] STATUS;                      -- 当前连接
mysql> SHOW GLOBAL STATUS;                         -- 数据库开启后

mysql> SHOW SESSION STATUS LIKE 'Com_insert%';     -- 查询插入语句执行次数
mysql> SHOW GLOBAL STATUS LIKE 'Innodb_rows_%';    -- Innodb 专用，查询影响行数
```

- **查询当前执行语句**

```sql
mysql> SHOW PROCESSLIST;            -- 查看数据库所有连接信息，包含正在执行的 SQL 语句


mysql> SHOW PROFILES;                           -- 查看当前连接执行的所有指令：ID 和 执行时间
mysql> SHOW PROFILE FOR QUERY 5;                -- 显示第 5 条指令执行的具体信息                
mysql> SHOW PROFILE CPU FOR QUERY 5;            -- 显示第 5 条指令各步骤的执行时间
```

- **解释语句执行方式**

```sql
mysql> EXPLAIN 具体语句;                 -- 解释语句执行的状况（重要）
```

### 语句优化


1. fileSort 排序，没有索引时利用文件系统排序，效率低。
2. index 排序，如果通过索引能直接返回数据，效率高。（只能返回有索引的字段）


对于 fileSort 排序，增大排序区大小满足排序需求，可以提高排序效率。

对语句的优化，主要就是对于索引的运用。





1. 避免使用子查询，可以把子查询优化为 join 操作。子查询不能利用索引。

2. 对应同一列进行 or 判断时，使用 in 代替 or。in 的值不要超过 500 个，in 操作可以更有效的利用索引，or 大多数情况下很少能利用到索引。


分组时，默认先排序后分组。会生成临时表。


1. 通过 ORDER BY null 不排序直接分组。
2. 上索引，有索引不临时。


#### LIMIT 分页查询

使用连表查询

```sql
SELECT * FROM student ORDER BY id LIMIT 2000000, 10;        # 浪费时间，排序 2000000 条后筛选
SELECT * FROM student s, (SELECT id FROM student ORDER BY id LIMIT 2000000, 10) t WHERE s.id = t.id;
```


## 优化方式

### 插入操作

#### 批量插入

注意：txt文件各个字段间，要用一个"table"键的距离隔开。一行只写一条数据。批量执行文本中的 SQL 语句。

```
mysql ->load data infile 'E:/student.txt' into table student;
```


1. 按主键顺序插入更高效！生成有序 txt 更好。
2. 关闭唯一性校验：`SET UNIQUE CHECK = 0` ，导入完成后记得开启。
3. 事务提交，手动提交事务：`SET AUTOCOMMIt = 0` ，导入完成后记得开启。









---

## 批量执行

### 导入导出


1
mysqldump -uroot -pMyPassword databaseName tableName1 tableName2 > /home/foo.sql
mysqldump -u 用户名 -p 数据库名 数据表名 > 导出的文件名和路径 

导出整个数据库

1
mysqldump -u root -p databaseName > /home/test.sql   (输入后会让你输入进入MySQL的密码)
mysql导出数据库一个表，包括表结构和数据
mysqldump -u 用户名 -p 数据库名 表名> 导出的文件名和路径

1
mysqldump -u root -p databaseName tableName1 > /home/table1.sql
如果需要导出数据中多张表的结构及数据时，表名用空格隔开

1
mysqldump -u root -p databaseName tableName01 tableName02 > /home/table.sql
仅导出数据库结构

1
mysqldump -uroot -pPassWord -d databaseName > /home/database.sql
仅导出表结构

1
mysqldump -uroot -pPassWord -d databaseName tableName > /home/table.sql
将语句查询出来的结果导出为.txt文件
1
mysql -uroot -pPassword database1 -e "select * from table1" > /home/data.txt
　　

数据导入
常用source 命令 
进入mysql数据库控制台，mysql -u root -p 
mysql>use 数据库 
使用source命令，后面参数为脚本文件(.sql) 

1
mysql>source /home/table.sql

### 存储过程

将大批量、经常重复执行的 SQL 语句集合预存储在数据库里，外部程序可以直接调用，减少了不必要的网络通信代价。

---

## SQL 安全

### SQL 注入


服务器向数据库发送的 SQL 语句往往包含用户输入：

```sql
-- 登录验证

SELECT * FROM users WHERE id = ' 用户输入1 ' and password = ' 用户输入2 ';
```

如果攻击者在用户输入中插入 `'`、`or`、`#` ，就可以改变 SQL 语句的功能。达到想要的目的。

```sql
-- 返回 alice 用户信息，登录 alice 账号
SELECT * FROM users WHERE id = ' alice'# ' and password = ' 用户输入2 ';
-- 返回全部用户信息，通常会默认登录首个账号（管理员）
SELECT * FROM users WHERE id = ' ' or 1# ' and password = ' 用户输入2 ';
SELECT * FROM users WHERE id = ' ' or '1 ' and password = ' 用户输入2 ';     -- AND 优先级高，先执行 AND 再执行 OR
 -- 直接删库
SELECT * FROM users WHERE id = ' '; DROP TABLE users;#' ' and password = ' 用户输入2 ';
```


在执行攻击时，攻击者可能需要知道数据库表信息，譬如表名，列名等。

1. 通过错误信息发现（输入错误语句获取）：在进行开发的时候尽量不要把出错信息打印到页面上，使用专门的错误页。
2. 通过盲注发现：

比如在用户输入中插入以下字符（如果表名首字母 ASCII 大于 97 则休眠 5s），就可以根据数据返回时间判断数据库表信息。

```sql
SELECT * FROM users WHERE id = ' alice' and if((select ascii(substr((select table_name from information_schema.tables where table_schema= database() limit 0,1),1,1)))>97,sleep(5),1) # ' and password = ' 用户输入2 ';
```


### SQL 注入防御

(prepare_statement) 为避免 SQL 注入攻击，新版本后端语言(PHP/Java) 都支持对输入 SQL 语句预处理：会自动检查用户输入并对单引号用`\`做强制转义， MySQL 数据库收到转义后的单引号也会用 setString 方法做转义处理。

学习网站开发的人就再也不用担心sql注入的威胁了。

---


### 内存优化

InnoDB 用一块内存区做缓存池，既缓存数据也缓存索引。

linux mysql 配置文件 user-my.cnf

innodb_buffer_pool_size = 512M  （默认128m）

innodb_log_buffer_size  日志缓存大小，过于小会频繁写入磁盘

### 日志管理

---







