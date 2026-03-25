## HTTP协议
#### 请求方式
进入网页根据提示，抓包后要将请求方式由GET改为CTFHUB再放包，得到flag
#### 302跳转
进入网页点击`Give me flag`后抓包，将包放到repeater模式，再发送，在响应界面最后得到flag
#### Cookie欺骗、认证、伪造
cookie是用来验证用户身份的，有网页提示可知，只有管理员才能获取到flag,先抓包，将cookie admin=0改为cookie admin=1，放包得到flag。
#### 基础认证
进入网页，点击click，随便输入用户和密码抓包，包中authentication主要是用来验证用户身份的，basic后加密的数据用base64解码后就是`刚才输入的用户:刚才输入的密码`因为提供了密码字典，可以爆破，在密码字典前每个密码前都加一个admin:   (因为大多数网站的管理员用户名是admin),再全部进行base64加密，并且，加密后末尾有=的，要将=删了（base64的加密规则是不够四个字节的用=补充，把等号删了不会有影响）把密文写在一个.txt文件中，用鼠标将basic后那一段密文高亮（接下来攻击就是替换这一段字符），再发送到intruder模块，load选择刚才的.txt文件，直接攻击，找到status是200的包，点开response最下面是flag
#### 响应包源代码
点开网站（网站的贪吃蛇挺好玩的），F12找到源码，在源码中找到flag
## 信息泄露
#### 目录遍历
进入后一级一级寻找，直到找到flag.txt点开得到flag
#### PHPINFO
PHPInfo函数信息泄露漏洞常发生一些默认的安装包，比如phpstudy等，默认安装完成后，没有及时删除这些提供环境测试的文件，比较常见的为phpinfo.php、1.php和test.php，然后通过phpinfo获取的php环境以及变量等信息，但这些信息的泄露配合一些其它漏洞将有可能导致系统被渗透和提权。
进入后搜索FLAG找到flag
#### 备份文件下载
###### 网站源码
写一个Python脚本，遍历网站源码文件名的每一种可能
```python
import requests  
url = "http://challenge-37eaa1354d995e66.sandbox.ctfhub.com:10800/"  
  
li1 = ['web','website','backup','back','www','wwwroot','temp']  
li2 = ['tar','tar.gz','zip','rar']  
for i in li1:  
    for j in li2:  
        url_final=url+"/"+i+"."+j  
        r=requests.get(url_final)  
        print(str(r)+"+"+url_final)
```

运行脚本，找到状态码为200的网址点进去下载到源码文件，找到文件中的.txt文件，用网址访问这个文件后得到flag`http://challenge-37eaa1354d995e66.sandbox.ctfhub.com:10800/flag_915214932.txt`
###### bak文件
根据提示可知源码在index.php.bak文件中，访问`http://challenge-2384fe6eaf47e42e.sandbox.ctfhub.com:10800/index.php.bak`下载文件打开源码，可以得到flag
###### vim缓存
在使用vim编辑命令时，因错误操作或系统问题强制退出，会自动生成一个swp文件（并且是隐藏的），保存修改时的所有内容，在环境url后输入`/.index.php.swp`会自动下载备份文件，用文档打开可得到flag
###### .DS_Store
在url后面输入.DS_Store,下载.DS_Store文件打开后，倒数的第二行隐藏着一个.txt文件的文件名（将中间的空格删掉），在url后面输入那个文件名得到flag
#### GIT泄露
###### log
当前大量开发人员使用git进行版本控制，对站点自动部署。如果配置不当,可能会将.git文件夹直接部署到线上环境。这就引起了git泄露漏洞。
现在控制台输入`git clone https://github.com/BugScanTeam/GitHack.git`下载githack
再`cd GitHack`,再输入`py -2 GitHack.py http://challenge-a77ad3d2a80b03b1.sandbox.ctfhub.com:10800/.git/`(即网址后加.git/)等待运行结束（时间有点长），最后显示一个文件路径，`cd 这个路径`，进来之后输入`git show`显示出flag
###### stash
前面操作同上一关，`cd 这个路径后`却无法得到flag,需要输入`git stash list`列出 stash 条目,看到一个flag,再输入`git stash pop`应用该条目，可以看到一句` deleted by us:   29664775720345.txt`,然后在刚才的路径下找到这个文件，点开后得到flag.
###### index
完全同log那一关
#### SVN泄露
当开发人员使用 SVN 进行版本控制，对站点自动部署。如果配置不当,可能会将.svn文件夹直接部署到线上环境。这就引起了 SVN 泄露漏洞。
svn泄露需要用dvcs-ripper工具，需要在kali-linux虚拟机里运行，打开虚拟机，`cd dvcs-ripper`再输入`perl rip-svn.pl -u url/.svn/`在`ls -al`发现有一个.svn文件说明有.svn泄露，再`cd .svn`再`tree`,发现下面有一个文件十分长的文件，在pristine目录下，cat一下那个文件得到flag
## HG泄露
当开发人员使用 Mercurial 进行版本控制，对站点自动部署。如果配置不当,可能会将.hg 文件夹直接部署到线上环境。这就引起了 hg 泄露漏洞。
用上题方法扫描到有.hg泄露，cd .hg 再用`grep -r flag*`看到一个flag开头的文件，用浏览器访问他，得到flag
## 文件包含
下面简单讲一下include函数的作用
你就可以理解成
假如在index.php中include了一个文件
那么不管这个文件后缀是什么 这个文件中的内容将会直接出现在index.php中
所以这道题的payload构造思路就是把题目给的shell.txt里的内容想办法放到index.php中去
根据源码构造payload：`?file=shell.txt&ctfhub('cat /flag');`这样一句话木马就包含进去了，可以利用进行命令执行
## php://input
常用到伪协议的`php://input`和`php://filter`.其中php://input要求`allow_url_include`设置为`On`查看phpinfo得知是on，可以用
- **原理**：`php://input` 作为被包含的 “文件” 时，其内容就是 POST 请求的正文，若正文是 PHP 代码，会被 `include` 执行。
- **操作步骤**：
    1. 构造 URL：`http://target.com/?file=php://input`（利用文件包含漏洞指定 `php://input`）。
    2. 发送 POST 请求，正文内容为要执行的 PHP 代码（如 `<?php system('ls /'); ?>`）。
    3. 服务器会包含 `php://input`，即执行 POST 正文中的代码，返回结果。
这里不知道为什么用hackbar不行，用bp改请求方式为post可以
![bp](/image/bp1.png)

## 命令注入
输入`127.0.0.1|ls /`看没有flag可以试一下`127.0.0.1|ls`发现一个可疑的php文件，`127.0.0.1|cat 那个文件`发现没有回显，这时看一下源码，得到flag

## 过滤目录分割符
![n](/image/1.png)
`;cd flag_is_here&&cat flag_51248814815.php`得到flag
## 过滤|
可以用`;`链接指令`;ls`  `;cat flag`
## 综合过滤练习
这一题`; | &` `cat flag ctfhub`都过滤了，其中`;`可以用%0a代替也就是换行符，cat可以用less代替，flag被过滤可以用`\`转义，空格用`${IFS}`代替，最终payloads`?ip=%0als%0acd${IFS}fl\ag_is_here%0aless${IFS}fl\ag_38081009829999.php`
flag可以用`%09`也就是tab键自动补齐代替，也可以用base64编码绕过，`%0als%0acd$IFS%09*_is_here%0abase64$IFS%09*_3425453514235.php`绕过cat,
但都一定用hackbar提交，不然网页会自动对`%0a`url加密，导致执行失败
## 读取源代码
`?file=php://filter/read=convert.base64-encode/resource=/flag`
## 远程包含
打开phpinfo发现allow_url_fopen和allow_url_include都是on，所以可以用文件包含漏洞，抓包post请求，（本来这道题想用php://filter的，但是flag被get过滤了，所以用php://input参数在post里，flag不会被过滤）
![3254](/image/3.png)
得到flag
或者用常规解法
需要用到服务器