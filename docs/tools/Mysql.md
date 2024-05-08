# Mysql Notes

## Install on Ubuntu22.04
使用如下命令可以直接安装latest的Mysql-server，安装完毕后使用`sudo systemctl status mysql`查看mysql服务的运行状态，默认会自动启动
```bash
    sudo apt update
    sudo apt install mysql-server
```

## 解决启动后默认访问不了问题
1. 修改`/etc/mysql/mysql.conf.d/mysqld.cnf`文件，在`[mysqld]`下添加`skip-grant-tables`，意为跳过权限检查，并重启Mysql服务
2. 登录后修改mysql.user表中的`plugin`和`authentication_string`字段，解决直接alter table引发的socket报错
```sql
    mysql -u root -h localhost
    use mysql;
    update user set plugin='mysql_native_password', authentication_string='Your password' where User='root';
    flush privileges;
```
3. 修改`/etc/mysql/mysql.conf.d/mysqld.cnf`文件，将`skip-grant-tables`注释掉，重启Mysql服务

## Mysql配置远程连接
配置远程连接需要修改两个地方，如果是服务器，记得查看是否配置了防火墙端口规则
1. 修改`/etc/mysql/mysql.conf.d/mysqld.cnf`文件，将`bind-address`更改为`0.0.0.0`，意为可以从任何ip访问数据库
2. 修改`mysql`数据库中的`user`表，将希望连接用户的`host`字段更改为`%`，意为可以从任何ip访问数据库
```sql
    mysql -u root -h localhost
    use mysql;
    update user set host='%' where User='Your user'
    flush privileges
```

## 使用笔记

written by *squarehuang*

>  For mysql 5.x

>注：安装mysql8.x参考：[Ubuntu20.04安装mysql8.x - 猎手家园 - 博客园 (cnblogs.com)](https://www.cnblogs.com/hunttown/p/17119331.html#:~:text=8.0%E4%B9%8B%E5%90%8E%E7%9A%84mysql%E4%B8%8D%E6%94%AF%E6%8C%81%20%E6%8E%88%E6%9D%83%E7%9A%84%E6%97%B6%E5%80%99%E5%B0%B1%E8%BF%9B%E8%A1%8C%E7%94%A8%E6%88%B7%E5%88%9B%E5%BB%BA%EF%BC%8C%E6%89%80%E4%BB%A5%E5%88%9B%E5%BB%BA%20%E4%B9%8B%E5%90%8E%E6%89%8D%E8%83%BD%E6%8E%88%E6%9D%83%EF%BC%9B%20%23%E5%88%9B%E5%BB%BA%E7%94%A8%E6%88%B7%20mysql%3E%20create%20user%20%27abcd%27%40%27localhost%27,mysql%3E%20grant%20all%20privileges%20on%20%2A.%2A%20to%20%27abcd%27%40%27localhost%27%3B)

### 1. 基本命令

+ 服务启动与关闭：net start mysql | net stop mysql
+ 重新启动：`service mysqld restarts`
+ 登录用户：mysql -h 端口IP -u username -p
+ 退出: quit(QUIT)，exit
+ 清除等待内容：\c, ctrl+c, clear
+ 清屏：system clear(Linux)
+ 查看Mysql版本信息：status，
  + 版本号+日期：SELECT VERSION(), CURRENT_DATE;
+ 修改提示符 PROMPT xxx
  + \\D 日期， \\d 数据库 ,\\h 服务器名称 , \\u 当前用户
+ Show命令：
  + 当前数据库: SHOW DATABASES;
  + 当前数据库中的表：SHOW TABLES;
  + 表的属性信息：SHOW COLUMNS FROM table_name
  + 表的索引信息：SHOW INDEX FROM table_name
  + 显示数据库中所有表的信息: SHOW TABLE STATUS FROM database_name (LIKE 'runroob%'，表名以runroob开头的表的信息)

| SELECT VERSION( )  | 服务器版本信息            |
| ------------------ | ------------------------- |
| SELECT DATABASE( ) | 当前数据库名 (或者返回空) |
| SELECT USER( )     | 当前用户名                |
| SHOW STATUS        | 服务器状态                |
| SHOW VARIABLES     | 服务器配置变量            |

补：

+ Mysql命令的**大小写**问题

 >默认Windows下不区分大小写，Linux下区分大小写，实际上由全局参数`lower_case_file_system`和`lower_case_table_names`控制，Linux下修改my.cnf，Windows下修改my.ini，
 >详情：
 >`lower_case_file_system`：ON，OFF(不敏感)
 >`lower_case_table_names`：0(敏感)，1，2
 >查看命令：show global variables like '%lower_case%';
 >
 >要求：表名、字段名必须使用小写字母或数字 ， 禁止出现数字开头，禁止两个下划线中间只 出现数字。

+ 规范

> 关键字与函数名称全部大写，数据库名称，表名称，字段名称全部小写，语句以分号结尾

### 2.用户权限管理

+ 权限分布：

<img src="../assets/image-20230407113321962.png" alt="image-20230407113321962" style="zoom:50%;" />

+ 权限检查顺序：

<img src="../assets/image-20230407113258432.png" alt="image-20230407113258432" style="zoom:50%;" />

+ 用户操作：

  + 创建用户以及密码

  ```mysql
  CREATE USER 'username'@'ip' IDENTIFIED BY 'password';
  #eg. @表示任何主机都可以通过该用户连接
  CREATE USER 'lisi'@'%' IDENTIFIED BY 'password';
  ```

  + 删除用户：`DROP user username@ip;`

  + 修改用户密码：

    ```mysql
    ALTER USER usrname@ip IDENTIFIED BY 'newpassword';
    SET PASSWORD FOR username@ip = PASSWORD('newpassword');
    #外部shell中
    shell> mysqladmin -u user_name -h host_name password "new_password"
    ```

  + 设置密码过期：

    ```mysql
    #修改my.cnf
    default_password_lifetime=xxx
    #给用户设置
    ALTER USER username@ip PASSWORD EXPIRE INTERVAL 90 DAY/ NEVER / DEFAULT
    ```

  + 锁定用户：`ALTER USER username@ip account unlock/lock`

  + 用户重命名：`rename user username@ip to new_username@ip`

+ 权限操作：

  + 常见grant权限：

    + select, insert, update, delete, create, alter, drop, refrerences, index, create view, show view, create routine, alter routine, execute
    + 也可以接查询语句设置针对列的权限

  + 授权：`GRANT (priv_type) ON (priv_level) TO 'username'@'IP'`

    ```mysql
    grant select, insert, update, delete on testdb.* to common_user@'%' #示例
    
    #全部数据库
    grant all on *.* to dba@'localhost' 
    #某个数据库的某个表
    grant all privileges on testdb.testtable to dba@'localhost'
    #允许该用户继续给别的用户赋权
    grant all privileges on *.* to 'zhangsan'@'192.168.1.%' with grant option;
    #存储过程
    grant execute on procedure testdb.pr_add to 'dba'@'localhos'
    #函数
    grant execute on function testdb.fn_add to 'dba'@'localhost'
    ```

  + 查看权限：`show grants (for username@localhost);`

  + 撤销权限：`revoke all on *.* from dba@localhost;`

    + 语法和grant相同，将to为from

  + 创建完后刷新：`FLUSH PRIVILEGES;`

### 3.数据库数据操作

##### 3.1数据库

+ 创建数据库： CREATE DATABASE DATABASE_NAME
  + 不登录版本：mysqladmin -u root -p create RUNOOB
+ 删除数据库：DROP DATABASE DATABASE_NAME
  + 不登录版本：mysqladmin -u root -p drop RUNOOB
+ 选择数据库：use database_name

##### 3.2数据表

+ 创建数据表：CREATE TABLE table_name(column_name column_type)

  ```sql
  CREATE TABLE IF NOT EXISTS runoob_tbl(
     runoob_id INT UNSIGNED AUTO_INCREMENT,
     runoob_title VARCHAR(100) character set utf8 NOT NULL,
     runoob_author VARCHAR(40) NOT NULL,
     submission_date DATE,
     PRIMARY KEY ( runoob_id )
  )ENGINE=InnoDB DEFAULT CHARSET=utf8;
  ```

  +  PRIMARY指定主键，CHARSET设置编码，ENGINE指定存储引擎

  - AUTO_INCREMENT定义列为自增的属性，一般用于主键，数值会自动加1。
  - 如果你不想字段为 **NULL** 可以设置字段的属性为 **NOT NULL**， 在操作数据库时如果输入该字段的数据为**NULL** ，就会报错。
  - `character set utf8`设置字符集防止中文乱码

+ 删除数据表： DROP TABLE table_name;

+ 修改数据表：利用ALTER命令

  + 修改字段：

    + ADD(添加列)，DROP(删除列)，MODIFY、CHANGE(修改字段类型名称)

    + ```mysql
      ALTER TABLE testalter_tbl DROP i;
      ALTER TABLE testalter_tbl ADD i INT FIRST;
      ALTER TABLE testalter_tbl ADD i INT AFTER c;
      ALTER TABLE testalter_tbl MODIFY c CHAR(10);
      ALTER TABLE testalter_tbl CHANGE i j BIGINT;
      ```

    + 默认值的处理：

      + 如果你不设置默认值，MySQL会自动设置该字段默认为 NULL。

        ```mysql
        ALTER TABLE testalter_tbl 
        	  MODIFY j BIGINT NOT NULL DEFAULT 100;
        ```

  + 修改表名：ALTER TABLE testalter_tbl RENAME TO alter_tbl;

  + ALTER的其他用途：

    + 修改存储引擎：`ALTER TABLE table_name ENGINE=engine_name `;
    + 修改子段相对位置：`alter table tableName modify name1 type1 first(after name2);` 放在最前面或name2后面

+ 查看数据表：

  + `SHOW CREATE TABLE table_name\G`
  + `Describe table_name`
  + `desc table_name`
  + `select * from table_name`



##### 3.3数据

+ 插入数据：字符型要用引号

  ```mysql
  INSERT INTO table_name ( field1, field2,...fieldN )
                         VALUES
                         ( value1, value2,...valueN );
  #示例
  mysql> INSERT INTO runoob_tbl
      -> (runoob_title, runoob_author, submission_date)
      -> VALUES
      -> ("学习 MySQL", "菜鸟教程", NOW());
  ```

+ 查询数据：

  ```mysql
  SELECT column_name,column_name
  FROM table_name
  [WHERE Clause]
  group by column_nmae
  having xxx
  order by
  [LIMIT N][ OFFSET M];
  ```

  + 执行顺序：

    + FROM, including JOINs
    + WHERE
    + GROUP BY
    + HAVING
    + WINDOW functions
    + SELECT
    + DISTINCT
    + UNION
      + UNION要保证两个表的列的数目和列的类型完全一致，可以用NULL去测试类型
    + ORDER BY
    + LIMIT and OFFSET

  + WHERE子句：操作符有=, <>(!=), >, <, >=, <=

    ```mysql
    SELECT field1, field2,...fieldN FROM table_name1, table_name2...
    [WHERE condition1 [AND [OR]] condition2.....
    ```

  + Like子句：%匹配any str, _匹配any char

    ```mysql
    SELECT field1, field2,...fieldN 
    FROM table_name
    WHERE field1 LIKE condition1 [AND [OR]] filed2 = 'somevalue'
    ```

  + ORDER BY子句：ASC(升序，默认)，DESC(降序)，可设定多个字段

    ```mysql
    SELECT field1, field2,...fieldN FROM table_name1, table_name2...
    ORDER BY field1 [ASC [DESC][默认 ASC]], [field2...] [ASC [DESC][默认 ASC]]
    ```

  + GROUP BY子句：function在列上操作

    ```mysql
    SELECT column_name, function(column_name)
    FROM table_name
    WHERE column_name operator value
    GROUP BY column_name;
    (WITH ROLLUP)
    ```

    + 最后可以加上，WITH ROLLUP，此函数是对聚合函数进行求和，注意 with rollup是对 group by 后的第一个字段，进行分组求和，不能和order by一起使用。

  + JOIN子句：

    + **INNER JOIN（内连接,或等值连,自然连接）**：获取两个表中字段匹配关系的记录。

    + **LEFT JOIN（左连接）：**获取左表所有记录，即使右表没有对应匹配的记录。

    + **RIGHT JOIN（右连接）：** 与 LEFT JOIN 相反，用于获取右表所有记录，即使左表没有对应匹配的记录。

      ```mysql
      SELECT a.runoob_id, a.runoob_author, b.runoob_count 
      FROM runoob_tbl a INNER JOIN tcount_tbl b 
      ON a.runoob_author = b.runoob_author;
      ```

+ 修改、更新数据：

  ```mysql
  UPDATE table_name SET field1=new-value1, field2=new-value2
  [WHERE Clause]
  ```

+ 删除数据：

  ```mysql
  DELETE FROM table_name [WHERE Clause]
  ```

+ 其他操作符：

  + UNION：ALL(返回所有结果，包含重复数据), DISTINCT(删除重复数据，默认)

    ```mysql
    SELECT expression1, expression2, ... expression_n
    FROM tables
    [WHERE conditions]
    UNION [ALL | DISTINCT]
    SELECT expression1, expression2, ... expression_n
    FROM tables
    [WHERE conditions];
    ```

  + 正则匹配：REGEXP / RLIKE（坑）

### 4.事务操作

+ 事务：一系列数据库操作语句构成一个事务
  + 满足：原子性（**A**tomicity，或称不可分割性）、一致性（**C**onsistency）、隔离性（**I**solation，又称独立性）、持久性（**D**urability）。
  + MySQL 命令行的默认设置下，事务都是自动提交。可以选择用BEGIN/START TRANSACTION，或者SET AUTOCOMMIT=0
+ 事务控制语句：
  + BEGIN/ START TRANSACTION: 开启一个事务
  + COMMIT/COMMIT WORK: 提交事务进行修改
  + ROLLBACK/ ROLLBACK WORK: 结束事务，撤销未提交的修改
  + SAVEPOINT identifier: 创建一个保存点
  + RELEASE SAVEPOINT identifier: 删除一个保存点
  + ROLLBACK TO identifier: 回滚到保存点
  + SET TRANSACTION: 设置隔离级别，四种READ UNCOMMITTED、READ COMMITTED、REPEATABLE READ 和 SERIALIZABLE。

### 5.索引和约束操作

+ INDEX

  + 创建索引：CREATE INDEX indexName ON table_name (column_name)

    + ALTER table tableName ADD INDEX indexName(columnName)

    + 如果是CHAR，VARCHAR类型，length可以小于字段实际长度；如果是BLOB和TEXT类型，必须指定 length

    + 创建表时指定：

      + ```mysql
        CREATE TABLE mytable(   
        ID INT NOT NULL,   
        username VARCHAR(16) NOT NULL,   
        INDEX [indexName] (username(length))  
        );  
        ```

  + 添加索引：ALTER table tableName ADD INDEX indexName(columnName)

  + 删除索引：DROP INDEX [indexName] ON mytable; 

  + 唯一索引：索引列的值必须唯一，允许有空。组合索引的组合必须唯一

    + 第一种把INDEX换为UNIQUE INDEX，第二中把ADD INDEX 换为 ADD UNIQUE
    + 创建表时把INDEX换位UNIQUE

  + 显示索引信息：

    + SHOW INDEX FROM table_n/ SHOW INDEX IN table_name


+ CONSTRAINT:

  + ALTER命令添加约束：

    - `ALTER TABLE tbl_name ADD PRIMARY KEY (column_list): `该语句添加一个主键，这意味着索引值必须是唯一的，且不能为NULL。

    - `ALTER TABLE testalter_tbl DROP PRIMARY KEY`: 删除主键

    - `ALTER TABLE tbl_name ADD UNIQUE index_name (column_list):`这条语句创建索引的值必须是唯一的（除了NULL外，NULL可能会出现多次）。

    - `ALTER TABLE tbl_name ADD INDEX index_name (column_list):` 添加普通索引，索引值可出现多次。

    - `ALTER TABLE tbl_name ADD FULLTEXT index_name (column_list)`:该语句指定了索引为 FULLTEXT ，用于全文索引。

    - `ALTER table_name ADD CONSTRAINT (constraint_name) constraint`增加约束

      ```mysql
      ALTER TABLE book ADD CONSTRAINT PRIMARY KEY(bno);
      ```

  + ALTER命令删除约束

    + `AUTO_INCREMENT`约束：`ALTER TABLE table_name change i i int `
    + `FOREIGN KEY`: `ALTER TABLE table_name DROP FOREIGN KEY constraint_name`

  + CHECK操作的实现：利用enum字段

    + 特别注意：enum的下标从1开始

      ```mysql
      create table table_name(
      	priority ENUM('low','middle', 'high') NOT NULL
      );
      ```

    + 更改ENUM值：先新增新类型值，将原有的要被替换掉的类型值的数据变更为新的类型值，最后再次变更表中的Enum类型值，删去要替换掉的类型值

      ```mysql
      alter table servers modify status enum ('initializing','initialized','error','initialize_failed') default 'initializing';
      update servers set status = 'initialize_failed' where status = 'error';
      alter table servers modify status enum ('initializing','initialized','initialize_failed') default 'initializing';
      ```

      


### 6.数据导出\入

+ 数据导出

  ```mysql
  SELECT * FROM passwd INTO OUTFILE '/tmp/runoob.txt'
  FIELDS TERMINATED BY ',' ENCLOSED BY '"'
  LINES TERMINATED BY '\r\n'; #csv格式
  ```

  + mysqldump命令导出SQL格式
    + mysqldump -u root -p RUNOOB runoob_tbl > dump.txt
    + mysqldump -u root -p --all-databases > database_dump.txt
    + mysqldump -u root -p RUNOOB runoob_tbl > dump.txt
  + mysqldump导出SQL脚本：
    + mysqldump -u root -p --no-create-info \
          --tab=/tmp RUNOOB runoob_tbl

+ 数据导入: local代表客户主机，否则默认为服务器

  ```mysql
  LOAD DATA LOCAL INFILE 'dump.txt' INTO TABLE mytbl
  FIELDS TERMINATED BY ':'
  LINES TERMINATED BY '\r\n';
  ```

  + mysql -u root -p 123456 < runoob.sql

  + source /home/abc/abc.sql，登陆后创建数据库设置编码后导入


### 7.视图操作

**视图是什么**

+ 解释：其实就是虚拟表，行和列来自于自定义视图的查询所引用基本表，并在具体引用视图时动态生成

+ 特性：
  + 视图的建立和删除不影响基本表
  + 对视图的内容的更新(增、删、该)直接影响基本表
  + 视图来自多个基本表时，不允许添加和删除数据

**视图操作**

+ 建立视图：CREATE VIEW view_name as 查询语句

  + 一般以视图名以**v_**或者**view_**开头
  + 多表：CREATE[OR REPLACE] VIEW viewname[columnlist]AS SELECT statement;

+ 查询视图：SELECT * FROM view_name

  + 描述视图：desc view_name
  + show tables
  + show create view view_nmae

+ 修改视图：

  + ALTER VIEW viewname[columnlist]AS SELECT statement;

+ 删除视图：drop view view_name

+ 更新视图数据

  + insert into view_name values(xxx)

  + update delete和一般语法相同

  + 不能更新的情况：

    + 视图中包含SUM()、COUNT()、MAX()和MIN()等函数

    + 视图中包含UNION、UNION ALL、DISTINCT、GROUP BY和HAVING等关键字

    + 视图对应的表存在没有默认值的列，而且该列没有包含在视图里

    + 包含子查询的视图


### 8. Trigger

+ 定义：触发器是与表有关的数据库对象，在满足定义条件时触发，并执行触发器中定义的语句集合。

  ```mysql
  CREATE TRIGGER trigger_name BEFORE/AFTER UPDATE/DELETE/INSERT
  ON table_name FOR EACH ROW
  BEGIN
  xxx #不同语句用分号隔开
  END;
  ```
<img src="../assets//image-20230402221556084.png" alt="image-20230402221556084" style="zoom:50%;" />

  `FOR EACH ROW`表示任何一条记录上的操作满足触发事件后都会触发该触发器

  更改结束符以免突然结束语句

  ```mysql
  DELIMITER ||
  xxx
  DELIMITER ;
  ```

  处理语句：

  + 抛出异常

    ```mysql
    SIGNAL sqlstate '45000' #用户未定义的操作
    SET MESSAGE_TEXT = 'xxx';
    ```

    

  ```mysql
  DECLARE var_name var_type [DEFAULT value];
  SET var_name = value;
  select xxx into var_name;
  NEW.columnname #新增行的某列数据
  OLD.columnname #删除行的某列数据
  IF xxx THEN
  	xxx
  ELSE 
  	xxx
  END IF;
  #完整示例
  DELIMITER ||
  CREATE TRIGGER check_book_inventory BEFORE
  INSERT ON borrow FOR EACH ROW
  BEGIN 
  DECLARE STOCK1 INT;
  SELECT stock into STOCK1 from book WHERE book.bno = NEW.bno;
  -- SELECT stock from book WHERE book.bno = NEW.bno;
  IF (STOCK1 > 0) THEN
  	UPDATE book SET book.stock=STOCK1-1 WHERE book.bno = NEW.bno;
  ELSE
  	SIGNAL sqlstate '45000'
      SET MESSAGE_TEXT = 'Out of STOCK';
  END IF;
  END
  ||
  DELIMITER ;
  ```

+ 查看触发器：`SHOW TRIGGERS`

+ 删除触发器：`DROP TRIGGER trigger_n ame; `

### 

### 0.小知识

+ 临时表的创建与删除: CREATE(DROP) TEMPORARY TABLE SalesSummary

+ 完整复制表:

  + CREATE TABLE targetTable LIKE sourceTable;
    INSERT INTO targetTable SELECT * FROM sourceTable;

+ AUTO_INCREMENT： 只能给一个字段使用，必须有NOT NULL属性，只能约束整数类型。当定义AUTO_INCREMENT后，就无需用户输入数据。

  ```mysql
  #示例
  mysql> CREATE TABLE insect
      -> (
      -> id INT UNSIGNED NOT NULL AUTO_INCREMENT,
      -> PRIMARY KEY (id),
      -> name VARCHAR(30) NOT NULL, 
      -> date DATE NOT NULL,
      -> origin VARCHAR(30) NOT NULL
  )engine=innodb auto_increment=100 charset=utf8;
  
  ALTER TABLE t AUTO_INCREMENT = 100;
  ```

+ 处理重复：

  + 防止出现重复数据

    + 设置主键，注意主键默认值设置为NOT NULL
    + INSERT IGNORE INTO插入数据，忽略数据库中已经存在的数据，如果数据库没有数据，就插入新的数据，如果有数据的话就跳过这条数据

  + 统计重复数据：

    ```mysql
    SELECT COUNT(*) as repetitions, last_name, first_name
    FROM person
    GROUP BY last_name, first_name
    HAVING repetitions > 1;
    ```

  + 过滤重复数据：

    + SELECT DISTINCT last_name FROM person
    + SELECT last_name FROM person GROUP BY last_name

  + 删除重复数据：

    ```mysql
    CREATE TABLE tmp SELECT last_name, first_name, sex FROM person_tbl  GROUP BY (last_name, first_name, sex);
    DROP TABLE person_tbl;
    ALTER TABLE tmp RENAME TO person_tbl;
    ```

    

### x. 基础知识

##### x.1 数据类型：

+ 数值类型：INTEGER, SMALLINT, DECIMAL(NUMERIC), FLOAT, REAL, DOUBLE

  + 注意当decimal类型长度小于14的时候,向decimal类型字段中插入数据时，小数位无效的0会被自动去掉，这时候页面显示需要进行小数位格式化

  <img src="../assets//Pasted image 20230320135025.png" alt="Pasted image 20230320135025.png" style="zoom:50%;" />

+ 日期和时间类型：DATETIME、DATE、TIMESTAMP、TIME、YEAR，当指定不合法的Mysql不能表示的值使用零值

  + 插入date类型数据要加上单引号

<img src="../assets//Pasted image 20230320135735.png" alt="Pasted image 20230320135735.png" style="zoom:50%;" />


+ 字符串类型：CHAR、VARCHAR、BINARY、VARBINARY、BLOB、TEXT、ENUM和SET  


<img src="../assets//Pasted image 20230320135953.png" alt="Pasted image 20230320135953.png" style="zoom:50%;" />


##### x.2 NULL值处理

- **IS NULL:** 当列的值是 NULL,此运算符返回 true。
- **IS NOT NULL:** 当列的值不为 NULL, 运算符返回 true。
- **<=>:** 比较操作符（不同于 = 运算符），当比较的的两个值相等或者都为 NULL 时返回 true。

```mysql
select * , columnName1+ifnull(columnName2,0) from tableName;
SELECT * FROM runoob_test_tbl WHERE runoob_count IS NULL;
```

columnName1，columnName2 为 int 型，当 columnName2 中，有值为 null 时，columnName1+columnName2=null， ifnull(columnName2,0) 把 columnName2 中 null 值转为 0。

##### x.3 主键和外键问题

+ 主键

  + 每个表只能定义一个主键，主键值必须唯一标识表中的每一行，不能为NULL

  + 一个字段名只能在联合主键字段表中出现一次, 联合主键不能包含不必要的多余字段

  + 不同表的主键名不能相同
+ 外键：

  + desc命令会显示为MUL，表示在属性上创建了非唯一索引，并且外键列可以出现相同的值(是的，自己想想)
  + 外键可以为NULL
  + 子表的外键列不能设置为NOT NULL

+ 更改父表中的数据时，要确保子表中有相应的数据

##### x.4 grant问题

+ `granted by current_role`是指使用和当前会话相关联的角色而不是当前用户来进行授权行为。若使用role来grant，即使授予权限的用户将来被撤销权限，只要这个role仍然具有相关权限，那么被授予权限的用户将会继续拥有，避免了不必要的cascade回收

  而如果使用当前用户来授权，那么该用户被撤销权限时，被授予权限的用户也会被撤销相应权限。

Fibonacci heap是在Binomial heap基础上进行改进的一种数据结构，Push操作的均摊代价为O(1)，Pop\_min操作的均摊代价仍为O(logn)，但是Decrease\_key的均摊代价被降低到了O(1)。核心在于前者舍弃了在Binomial heap中Push和decrease\_key中的merge操作，将其转移到了Pop\_min中。总体时间复杂度仍然是O(ElogV)