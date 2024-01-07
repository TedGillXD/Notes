# MySQL 基础知识

## SQL的分类
* **DDL(Data Definition Language)**: ``CREATE`` | ``ALTER`` | ``DROP`` | ``RENAME`` | ``TRUNCATE``
* **DML(Data Manipulation Language)**: ``INSERT`` | ``DELETE`` | ``UPDATE`` | ``SELECT``
* **DCL(Data Control Language)**: ``COMMIT`` | ``ROLLBACK`` | ``SAVEPOINT`` | ``GRANT`` | ``REVOKE``

### DDL的使用
DDL的操作一旦执行，就不可回滚
1. ``CREATE``: 创建数据库，表等信息
    ```sql
    CREATE DATABASE database_name # 创建数据库
    CREATE TABLE table_name(field1 field1type, field2 field2type, ...) # 创建数据表
    ```
2. ``ALTER``:
3. ``DROP``:
4. ``RENAME``:
5. ``TRUNCATE``: 删除指定表中的所有数据，保留表结构，数据不可以使用``ROLLBACK``回滚
    ```sql
    TURNCATE TABLE table_name;
    ```


### DML的使用
DML在默认情况下，一旦执行，也是不能回滚的，想要实现回滚功能，需要在执行DML前执行
```sql
SET autocommit = FALSE;
```
1. **DELETE**: 删除指定表中的数据，就算清空所有数据，也能保留表结构，可以回滚
    ```sql
    DELETE FROM table_name (WHERE condition);
    ```

### DCL的使用
1. **COMMIT**: 提交数据，一旦执行``COMMIT``，则数据就被永久保存在数据库中了，意味着不可以被回滚
2. **ROLLBACK**: 回滚数据，一旦执行``ROLLBACK``，则可以实现数据的回滚

## MySQL 数据类型

## MySQL 高级特性

### MySQL的配置
#### 1. 默认字符集
在MySQL8.0中，默认字符集是utf8mb4，意味着数据库中能直接存储中文等复杂字符信息。而在MySQL5.7中，默认字符集是latin1，这个字符集不包含中文，假如我们尝试插入中文信息，在MySQL5.7中会出现错误。
为了解决MySQL5.7默认不支持中文的问题，第一个方法，我们可以在后端将中文信息使用**Base64编码**将信息转换为只用latin1中的字符就可以表示的形式再存入数据库。
第二个方法，我们可以修改默认字符集
```sh
vim /etc/my.conf #进入mysql在linux中的配置文件
```
在文件的最后加上
```sh
character_set_server=utf8 #配置字符集
```
最后重启MySQL服务
如果要修改现有的数据表的字符集，可以使用以下指令
```sql
ALTER TABLE table_name CONVERT TO CHARACTER SET 'utf8'; # 修改现有数据表的字符集
```

#### 2. 各个级别的字符集
在MySQL上有4个级别的字符集和比较规则，分别是：
* 服务器级别(*_server): 一个mysql服务器会有一个默认的字符集设置，可通过上面提到的方法进行修改
* 数据库级别(*_database): 每一个database都可以设置自己的字符集，当创建数据库时没有指定字符集，这个数据库的字符集会跟随服务器级别的字符集
* 表级别: 每一张数据表都可以有自己的字符集，当我们在创建表的时候没有指定字符集，这个表的字符集就会跟随数据库级别的字符集
* 列级别: 我们可以为数据的每一列都单独设置一个字符集，当我们在创建列的时候没有指定字符集，这个列的字符集会跟随表级别的字符集

查看所有级别的字符集，我们可以通过以下的sql命令实现
```sql
SHOW VARIABLES LIKE 'character%'
```
![mysql character set](MySQL_images/show_character_set.png)

#### 3. 请求到响应过程中字符集的变化
![Alt text](MySQL_images/procedure_of_charset_during_request_in_mysql.png)
需要注意的是，发送请求的客户端的字符集需要和MySQL Server的``character_set_client``保持一致，而接受响应的客户端的字符集需要和MySQL Server中的``character_set_results``保持一致，否则会出现乱码。

#### 4. SQL的大小写规范
在SQL中，分为case sensitive和case insensitive两种，第一种表示大小写会影响到实际的查询，而第二种则相反，举个例子，假设我们有个数据库名字叫dbtest，在case sensitive的情况下：
```sql
USE dbtest; # 正常访问dbtest database
USE dBtest; # 报错，无法访问
```
而在case insensitive的情况下：
```sql
USE dbtest; # 正常访问dbtest database
USE dBtest; # 同样能访问dbtest database
```
默认情况下，`Windows系统是case insensitive`，而`Linux系统下是case sensitive`，通过使用以下SQL我们可以查看这一参数：
```sql
SHOW VARIABLES LIKE '%lower_case_table_names%'; # 检查当前SQL是否大小写敏感
```
`此参数默认为0，表示大小写敏感`，当参数为1时，表示大小写不敏感

#### 5. sql_mode 设置
查看当前的sql_mode:
```sql
SELECT @@session.sql_mode;  # 查看当前MySQL的sql_mode
```
**宽松模式**: 在这个模式下，即使我们在插入数据的时候给了一个错误的数据，也可能会被接受，比如我有一个表：
```sql
CREATE TABLE users(username, char(10));
```
我们可以看到，username这个field的长度应该是10，但是在宽松模式下，如果我们插入例如`0123456789abcd`，已经超过了10这个长度，SQL并不会报错，而是会截取前10个字符存入数据库中。
```sql
INSERT INTO users (username) VALUES ('0123456789abcd'); # 不会报错
```
**严格模式**: 对于上面这种情况，在严格模式下则是会报错
```sql
INSERT INTO users (username) VALUES ('0123456789abcd'); # 会报错
```

#### 6. MySQL的数据目录（数据都是存档在Linux的哪个目录下？）
##### 6.1 MySQL数据库的存放路径
`/var/lib/mysql/`: 这个目录下存放的是通过`CREATE DATABASE database_name`创建的数据库(database)文件夹
实际上，这个文件夹的路径可以通过SQL语句查询查看：
```sql
SHOW VARIABLES LIKE 'datadir'; # 查看数据库文件路径
```
`/usr/bin/`和`/usr/sbin/`: 这个目录下存放的是所有Linux的可执行指令，包括MySQL的指令的可执行文件
`/usr/share/mysql-8.0/`: 这个目录下存放的是MySQL命令及配置文件
`/etc/`: 用来存放`my.cnf`配置文件
##### 6.2 MySQL系统自带的数据库
* `mysql`
    MySQL系统自带的核心数据库，存储了MySQL的**用户账户和权限信息，一些存储过程、事件的定义信息，一些运行过程中产生的日志文件，一些帮助信息以及时区信息等**
* `information_schema`
    MySQL系统自带的数据库，保存着MySQL服务器**维护的所有其他数据库的信息**，比如有哪些表，哪些视图，哪些触发器，哪些列，哪些索引。在这个数据库中还包括一些以`innodb_sys`开头的表，用于表示内部系统表
* `performance_schema`
    MySQL系统自带的数据库，保存着MySQL服务器运行过程中的一些状态信息，可以用来**监控MySQL服务的各类性能指标**。包括统计最近执行了哪些与具有，在执行过过程中的每一个阶段都花费了多长时间，内存的使用情况等
* `sys`
    MySQL系统自带的数据库，这个数据库主要是通过**视图**的形式把`information_schema`和`performance_schema`结合起来，帮助系统管理员和开发人员监控MySQL的技术性能

##### 6.3 数据库在文件系统中的表示（Innodb）
正如我上面所述，所有的数据库文件夹是存放在`/var/lib/mysql/`这个路径下的，我们可以在这文件加下找到诸如`mysql`，`sys`，`performance_schema`等文件夹，也包括我们自己创建的database文件夹。
![Alt text](MySQL_images/var.lib.mysql.png)
我们可以进入数据库文件夹，在MySQL5.7版本下，我们会发现每一个数据表对应着两个文件，`table_name.frm`和`table_name.ibd`，其中frm文件是负责存放表结构的，而ibd则是负责存放对应的数据，以及一个额外的`db.opt`文件，这个文件负责存放该数据表的一些默认设置，如字符集设置、比较规则等。
![Alt text](MySQL_images/datatable.png)
而在MySQL8.0版本下，我们会发现一个数据表只对应了一个`table_name.ibd`文件，同时也没有了`db.opt`，实际上是所有的5.7版本多出来的文件所存储的信息都整合到了.ibd文件当中

##### 6.4 数据库在文件系统中的表示（MYISAM）
和Innodb引擎一样，数据库文件夹也是存放在`/var/lib/mysql/`这个路径下的，和Innobd相比，在MySQL5.7版本下，除了.frm文件外，另外的两个文件是`table_name.MYD`和`table_name.MYI`，实际上这两个文件存储的信息就等于在Innodb中的.ibd文件
![Alt text](MySQL_images/myisam5.7.png)
而在MySQL8.0版本下，数据表依旧是分开存储的，.MYD和.MYI保持不变，而.frm文件则变为了.sdi

#### 7. Mysql的配置文件

配置文件格式
在MySQL的配置文件中，启动选项被划分为若干组，每一个组有一个组名，用中括号`[]`括起来，如下所示
```properties
[server]
# some properties

[mysqld]
# some properties

[mysqld_safe]
# some properties

[client]
# some properties

[mysql]
# some properties

[mysqladmin]
# some properties
```
每一个propertiy有两种格式，分别是
```properties
[server]
option1             # only key
option2=value2      # key-value format
```
#### 8.MySQL的用户与角色管理

### MySQL 架构

#### 1. Mysql的整体架构
MySQL是典型的CS架构，客户端向服务器进程发送一段文本（SQL语句），服务器处理后再向客户端发送一段文本（处理结果），我们以查询请求为例，具体流程如下图所示：
![Alt text](MySQL_images/mysql_process.png)
下面这张图能很好的展示展开后的MySQL5.7的架构
![Alt text](MySQL_images/MySQL_Architecture.png)
* `Connectors`: 表示MySQL服务器以外的客户端程序（与各个语言相关，图JDBC是Java用来链接服务器的库）
* `Services & Utilities`: 基础服务组件
* `Connection Pool`: 提供了多个用于客户端与服务器端进行交互的线程
* `SQL Interface`: 接受SQL指令并且返回查询结果的
* `Parser`: 用于解析SQL Interface传过来的SQL指令，包括：语法解析、语义解析，最终生成语法树
* `Optimizer`: **核心组件**，对SQL进行优化，并生成一个`执行计划`
* `Cache`: 以Key-Value的形式缓存查询结果 **MySQL8.0已废除**
* `Pluggable Storage Engines`: 与操作系统的文件系统进行交互，真正在文件系统中进行数据读取的实际上是Storage Engines
* `File System`: 操作系统提供的文件系统

整体上，MySQL分为**连接层**，**服务层**和**引擎层**，连接层包括Connection Pool，服务层包括SQL Interface、Parser、Optimizer和Cache，引擎层指的是Pluggable Storage Engines。

一个SQL语句的大致的执行顺序是怎么样的呢？
`1.在客户端使用如JDBC等库发送SQL语句到MySQL服务器中` -> `2.在Connection Pool中创建线程，建立连接` -> `3.调用相关的SQL Interface` -> `4.在Cache中查询是否有已经存在的一摸一样的查询结果（deprecated in Mysql8.0）` -> `5.Parser解析SQL语句，生成语法树` -> `6.在Optimizer对SQL进行优化（逻辑上与物理上（使用索引））` -> `7.调用Storage Engine的API进行查找` -> `8.在文件系统进行查找` -> `9.将查询结果缓存在Cache中（deprecated in Mysql8.0）` -> `10.将数据返回到客户端`

#### 2. SQL的执行流程

### MySQL 索引及调优

### MySQL 事务(Transaction)

### MySQL 日志与备份