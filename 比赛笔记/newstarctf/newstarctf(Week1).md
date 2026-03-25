## multi-headach3
根据提示应该是请求头的问题，抓包后找不到flag,但是在网址后加robots.php可以得到提示`User-agent: * Disallow: /hidden.php`然后再在原网址后加hidden.php后得到`And why you are here again and again?  
`Trust me, hidden page is not as simple as you think.`
（他急了，说明这应该是对的），然后对这个网址抓包，在响应头中找到Fl4g: flag{de17c70f-7d69-48ad-b433-955748237c67}
## strange-login
根据提示“我当然知道1=1呀”，所以可以**构造特殊输入，篡改原始 SQL 语句的逻辑**，实现无论密码是否正确，都可以登入的效果，在用户处输入`admin' or 1=1#`,密码位置随便输，登录成功，得到flag
## 宇宙的中心是php
根据给出的ip和端口访问`http://47.94.87.199:22674/`访问到一个界面，但是f12和右键都失效了，可以使用`Ctrl + Shift + I`快捷键打开，在Elements中得到提示**你还是找到了......这片黑暗的秘密**下面有一个文件名s3kret.php,访问文件`http://47.94.87.199:22674/s3kret.php`到达一个界面
```php
<?php  
highlight_file(__FILE__);  
include "flag.php";  
if(isset($_POST['newstar2025'])){    $answer = $_POST['newstar2025'];  
    if(intval($answer)!=47&&intval($answer,0)==47){  
        echo $flag;  
    }else{  
        echo "你还未参透奥秘";  
    }  
}
```
所以需要以newstar2025=....的形式发送Post请求，`intval($var, $base)`，函数若$base=0,intval会自动识别数字进制，以0x开头为十六进制，以0开头（非0x）,识别为八进制，以数字1-9开头识别为十进制，当 $base不填时则强制转换为十进制，所以需要找到一个数，强制转换十进制后!=47,自动识别进制转换为47，所以0x2f最合适，`0x` 开头被识别为十六进制，十六进制 `2f` 转十进制就是 `47`，十进制中没有 `0x` 这种表示，`intval` 会从 `0` 开始截取有效数字，最终转为 `0`（`0≠47`），满足 `!=47`。所以要发送Post请求为`newstar2025=0x2f`(或者057也可以)
## 别笑，你也过不了第二关
第一关轻松过，但第二关要求1000000分，不可能手动完成，可能要通过修改前端代码来绕过，但是修改了分数绕过不了，修改低第二关的分数要求还是过不了，就想到可能要后端绕过，但是没有发送请求，就不能抓包，所以想到用控制台，读源码可知，score参数用来储存分数，endLeval函数用来结束游戏，所以先后在控制台输入`score=1000000;`和`endLeval();`可以通关，得到flag；
## 我真得控制你了
这一关首先通过f12,右键检查，`ctrl+shift+i`和`ctrl+shift+j`都不能打开开发者模式，但是可以通过浏览器右上角三个点更多工具里的开发者工具打开，虽然打开，但他还是一直提示**开发者工具已禁用 按 ESC 键关闭此提示**，并且界面里的启动也无法点击，通过属性里的
```javascript
// 检查保护层状态 
function checkShieldStatus() { 
const shield = document.getElementById('shieldOverlay');
// 获取ID为shieldOverlay的元素（保护层）
const button = document.getElementById('accessButton');
// 获取“启动”按钮
```
所以可以通过前端删除shieldOverlay解除“启动”按钮的保护层，在属性里搜索shieldOverlay，找到类似`<div id="shieldOverlay"></div>`的代码，删除shieldOverlay，这样“启动”按钮就解锁了，狂按esc趁机点击“启动”按钮，进入到一个输入用户名和密码的界面，因为下面提示**弱口令是什么东西？**，所以就是猜，猜用户名是admin(一般来说可以)，密码就猜111111，123456，123123，abcdef,abc123等，输入111111是对的，进入一个php源码界面，根据
```php
if (isset($_GET['newstar'])) {
    $input = $_GET['newstar'];
```
所以要以?newstar=的形式发送get请求，
根据`if (is_array($input))` 禁止传入数组，只能传字符串。
根据`if (preg_match('/[^\d*\/~()\s]/', $input))`只能输入：
- \d数字（`0-9`）
- 运算符：`*`（乘）、`/`（除）
- 特殊符号：`~`（按位取反）、`()`（括号）、空格（`\s`）
- 其他字符（如`+`、`-`、字母、字母等）均被禁止。
根据`preg_match('/^[\d\s]+$/', $input`禁止纯数字输入
要利用已有字符构造出2025，因为php中`~x=-(x+1)`所以
2025=~(-2026),但因为不能有`-`但可以用`/`和`*`,`~0/1=-1`
`-2026=~(0/1)*2026`所以最终为`~(~(0/1)*2026)`,发送get请求`?newstar=~(~(0/1)*2026)`,得到flag
## 黑客小W的故事（1）
每点击一次要加16吉欧，怪兽随机出现分数直接清零，可以抓包发送到repeater，后将count改为十，这样每发送一次加160吉欧，收集成功概率大大增加。之后根据提示要发送get请求`?shipin=mogubaozi`放包，提示要用post请求与他对话，发送post请求`message=guding`但`?shipin=mogubaozi`要保留。发送DELETE请求`chongzi=clear`(同样保留饰品)，再发送post请求`message=guding`.最后根据提示抓包将user-agent改为`CycloneSlash/5.0/DashSlash/5.0`放包成功拿到flag