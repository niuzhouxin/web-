[参考文章]([020_Web安全攻防实战：HTTP请求走私原理、高级攻击技术与全面防御策略深度指南-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2589360))
## HTTP关键头部字段
| 头部字段              | 作用               | 攻击相关性  |
| ----------------- | ---------------- | ------ |
| Content-Length    | 指示消息体的长度         | 核心攻击目标 |
| Transfer-Encoding | 指定消息体的传输编码       | 核心攻击目标 |
| Host              | 指定目标主机           | 路由相关攻击 |
| Connection        | 控制连接行为           | 连接处理攻击 |
| Transfer-Encoding | 指定传输编码（如chunked） | 格式混淆攻击 |
|                   |                  |        |
## 内容长度与传输编码
HTTP协议提供了两种主要方式来指定消息体的结束位置：Content-Length头部和Transfer-Encoding头部。这两种机制之间的交互是请求走私攻击的核心。
### Content-Length
- 通过数值指定消息体的字节数。
- 格式：`Content-Length: <number>`
### Transfer-Encoding
- 支持分块传输编码
- 格式：`Transfer-Encoding: chunked`
- 块格式：`[chunk-size][\r\n][chunk-data][\r\n]`
- 结束块：`0[\r\n][\r\n]`
## 优先级规则
根据HTTP规范，当同时存在Content-Length和Transfer-Encoding头部时，Transfer-Encoding应该优先。但不同服务器的实现可能存在差异，这正是攻击的切入点。
## HTTP请求走私本质
HTTP请求走私攻击利用了多层HTTP服务器（如前端代理和后端服务器）之间对HTTP请求边界理解的差异，导致请求被错误地解析和处理。
攻击者精心构造的请求在前端服务器和后端服务器中被不同地解析，导致：
- 前端服务器将一个请求视为两个独立请求
- 后端服务器将两个请求合并为一个
- 或产生其他边界混淆
## 检测漏洞类型的方法
![](/image/214.png)
## CL.TE攻击（Content-Length优先于Transfer-Encoding）
这种攻击利用了前端服务器使用Content-Length而后端服务器使用Transfer-Encoding的解析差异。
- 前端服务器仅处理Content-Length头部，将请求体视为指定长度的数据
- 后端服务器优先处理Transfer-Encoding头部，将请求体解析为分块传输的数据
- 攻击者构造的请求体中包含另一个完整的HTTP请求，被后端服务器视为单独的请求
例如：
```
POST / HTTP/1.1 
Host: example.com 
Content-Length: 13 
Transfer-Encoding: chunked 

0 

SMUGGLED REQ
```
- 前端服务器：使用Content-Length: 13，认为请求体结束于第13个字节（“0\r\n\r\nSMU”）
- 后端服务器：使用Transfer-Encoding: chunked，解析到"0\r\n\r\n"后认为第一个请求结束，将"GGLED REQ"视为新的请求
### 例题
对所给网页抓包发现他默认是HTTP/2![](/image/196.png)，HTTP/2是无法请求走私的，所以要降级到HTTP/1.1![](/image/197.png)再把请求方式改为post，并且可以把一些无关的内容移除掉，只留下`Host Content-Type Content-Length和第一行`  ，最后把设置里自动更新`Content-Length`关掉。最后还要启动显示不可打印字符![](/image/199.png)这在计算`Content-Length`时有帮助，因为`\r\n`本身占两个字节。接下来测试前后端分别是用哪种方式，![](/image/200.png)加上`Transfer-Encoding: chunked`
输入内容，修改`Content-Langth`为6，这样会截取到`3\r\nabc`这六个字节，后面的忽略。发送请求，等了好几秒猜反应，500，服务超时。是因为前端发送请求时丢弃了X，而当后端收到请求时它会查找X，下一个数据块原本应该在X的位置，但由于缺少数据块，后端会保持连接一段时间，等待数据块到达，当数据块最终没有到达时，后端会终止请求，报错500超时。这就判断了前端服务器使用Content-Length而后端服务器使用Transfer-Encoding。
接下来开始污染，可以先发送![](/image/201.png)这是攻击请求，以0加两个换行符提示分块消息的结束，这样后端就会被`0\r\n\r\n`之后的内容污染，再找一个一样的请求来跟进攻击请求，该请求将附加到之前污染后端的前缀之后，`foo=bar`后面要有一个换行符，发送![](/image/202.png)可以看到Unrecognized method GPOST，这样问题就解决了。
### 通过差异响应确认 CL.TE 漏洞
构造![](/image/215.png)其中`X-Ignore: X`后面一定不可以加换行符，因为后续发送的GET请求，会直接拼接到后面，如果有换行符，会识别成两个GET请求，![](/image/216.png),这样再找一个原始的GET包，直接发送会显示404,完成实验。因为后端用的是`Transfer-Encoding: chunked`所以`0\r\n\r\n`后的内容会自动和下一个请求拼接。

## TE.CL攻击（Transfer-Encoding优先于Content-Length）
这种攻击利用了前端服务器使用Transfer-Encoding而后端服务器使用Content-Length的解析差异。
1. 前端服务器处理Transfer-Encoding头部，将请求体解析为分块数据
2. 后端服务器忽略Transfer-Encoding（可能由于格式问题），转而使用Content-Length
3. 攻击者在块数据中隐藏额外的请求，被后端服务器错误处理
例如
```
POST / HTTP/1.1 
Host: example.com 
Transfer-Encoding: chunked 
Content-Length: 4 

5 
Hello 
0

```
- 前端服务器：使用Transfer-Encoding，解析到"0\r\n\r\n"后认为请求结束
- 后端服务器：忽略Transfer-Encoding（可能由于空格或其他格式问题），使用Content-Length: 4，将前4个字节视为请求体（“5\r\nH”），剩余部分可能影响后续请求
### 例题
同上先检测前后端用的分别是什么，构造一个请求![](/image/203.png)发送发现400，Invalid request。Content-Type截取第一个数据块，第二个数据块期待的是一个十六进制数，但是我写了一个X，直接拒绝请求。如果改一下payload，加个结束符，最后的X不加换行。![](/image/204.png)发送后会超时，这是因为前端用的是`Transfer-Encoding: chunked`以`0\r\n\r\n`结束，X就被丢弃了，再把请求体转发到后端服务器，如果后端用的是`Content-Length`，长度用的是6，但是发送到后端的只有`0\r\n\r\n`长度只有5，后端服务器会一直等待六个字节的到达，直到超时。这就通过计时技术来判断出了。
这时要用一个另外的包，修改一下，。发送后可以得到200 ok，![](/image/205.png)再找到原来的包修改一下![](/image/206.png)发送得到200 ok，在找到第二个包，直接发送，可以看到![](/image/207.png)`Unrecognized method G0POST`，这个一定要按顺序来。
因为前端用的是`Transfer-Encoding: chunked`，以`0\r\n\r\n`表示分块消息的结束，所以没有内容被丢弃，全被发送到了后端服务器，后端用的是`Content-Length`长度写的是3，他只会截到`1\r\n`，后面的`G0`就污染了请求体。所以就形成了`G0POST`。
接下来要完成实验，把第一个包构造成![](/image/208.png)放包，再对第二个包放包就完成了实验。![](/image/209.png)
**解释**：
之所以是56，是因为选中部分的长度的十六进制是0x56，上面一部分的`Content-Length`是因为`56\r\n`长度是4，上面一部分的`Content-Length`是因为`0\r\n\r\n`长度是五，我们至少将他设置为6（但是这也是有上限的，上限就第二个包的字节数加上5）。这样至少会有一个字节的内容添加到污染的后端服务器的前缀中![](/image/210.png)，下面的只留三行就行了。
### 通过差异响应确认 TE.CL 漏洞
发送pyalod![](/image/217.png)结果time out了，说明是TE.CL漏洞。![](/image/218.png)接下来构造pyload，9f的长度不包括x=1后面的换行符，发送请求，再找一个包改为POST请求发送，会404。
## TE.TE
如果前端后端都是`Transfer-Encoding: chunked`但可以通过某种方式混淆该标头，使其中一个服务器不处理它。
### 题目
构造payload检测到前端用的是`Transfer-Encoding: chunked`![](/image/211.png)
他期待的是一个十六进制数，不是就报错。再试一下另一个payload![](/image/212.png)发现后端用的也是`Transfer-Encoding: chunked`，如果用的是`Content-Type`他会延时报错。
这就要用到混淆了，在payload加一行`Transfer-Encoding: X`这是无效的，再发送就会超时，这是因为前端处理更宽松，接受了格式错误的表头并处理了无用的正文，而后端会更严格，拒绝该标头，改用`Content-Length`处理。造成漏洞的原因可能是前端用的是`Nginx`后端用的不同，或者用的不同版本。
接下来操作基本同上了，payload![](/image/213.png)
## 防御
使用 HTTP/2 端到端协议的网站本质上可以抵御请求走私攻击。由于 HTTP/2 规范引入了一种单一且强大的机制来指定请求长度，攻击者无法人为制造所需的歧义。
## 利用漏洞
### 绕过前端安全控制
#### CL.TE 漏洞
通过构造payload可知是CL.TE漏洞，构造payload再发送![](/image/219.png)再构造另一个请求发送会发现Invalid request![](/image/220.png)
这是因为拼接后会有两个方法![](/image/221.png)所以我们只需要在后面加一个`X-Ingore: X`后面没有换行符，![](/image/222.png)这这样就解决了，再发送会访问`/fgagra`但是他是不存在的,会404，改为`/admin`后发送，就会返回`HTTP/1.1 401 Unauthorized  Admin interface only available to local users`
所以要修改Host![](/image/223.png)再发送会发现`"error":"Duplicate header names are not allowed"`这是因为存在两个Host字段，修改一下，最终payload是![](/image/224.png)发送就可以了，这是因为后面发送的请求会拼接到x=后面，成为x的值，host也就丧失了原有的含义，但是这样请求就没有`Content-Type和Content-Length`了，因为这俩哥们被当做x的值了，请求头不完整，就需要把这两个补全，因为x=长2，所以`Content-Length: 3`，不知道为什么用127.0.0.1不可以。再![](/image/225.png)就可以删除密码了。
#### TE.CL 漏洞
根据验证可知是TE.CL漏洞，所以构造payload![](/image/226.png)发送再发送![](/image/227.png)会回显404，说明成功了。再改为/admin会回显401，要加host为localhost，再放包就可以成功执行。最终payload是![](/image/228.png)再找到删除的路径，再发一次请求就可以了。（记得每一次改请求的时候记得改长度）
### 揭示前端请求重写
这一关直接访问/admin会显示`Admin interface only available if logged in as an administrator, or if requested from 127.0.0.1`如果抓包加上`X-Forward-For: 127.0.0.1`无法访问，这有可能是前端覆盖了。
经测试发现这是CL.TE漏洞。
构造payload![](/image/229.png)发送后404了，说明可以了，改为admin，再加一句`X-Forwarded-For: 127.0.0.1`![](/image/230.png)但是这样发送后还是401，可以用那个搜索框来找到确切的`X-Forwarded-For`，搜索是发送POST请求，`search=...`，所以修改pa   yload为![](/image/231.png)发送请求，再发送![](/image/232.png)这样就可以看到服务器用的不是`X-Forwarded-For`而是`X-nHyllS-Ip`，图片圈错重点了。原理是第二个POST请求被后端当成新的独立请求时，后端正常走了一次请求处理逻辑，自动附加了这个内部头 `X-nHyllS-Ip`，我就在响应中看到了本不该给外部用户的内部头，最终payload![](/image/233.png)
### 捕获其他用户的请求
经测试这是CL.TE漏洞，用payload![](/image/234.png)发送，再发送另一个请求，回显404。
因为要劫持会话，所以发送一个评论抓包，把抓到的包放到第一个包后面，把`comment`参数放到最后，这是因为我们希望正常请求作为评论附加在请求体末尾，这样管理员浏览网页发送GET请求时，请求的所有请求头都会以注释的形式显示出来，显示在评论区。
接下来构造![](/image/235.png)之所以`Content-Length: 900`是因为为了可以容下一个完整的GET请求头，如果不够可以再加。我们发送一个构造的请求，再发送一次正常的请求，知道返回200 ok，这说明评论发布成功，如果返回302，就再重新试一次。其实正常请求的长度要长，如果正常请求的长度不大于900的话就会time out，所以要用换行符凑数，让长度大于900，不然后端会一直等那缺失的字节，导致time out ，接下来操作就行了，成功后会显示![](/image/236.png)接下来就劫持，这个长度最好适当一些，太长了好像不行。可以逐步调整，直到显示完整对session-cookie，得到后把session用hackbar改一下，再刷新，就完成了。
### 利用 HTTP 请求走私攻击反射型 XSS
经检验试CL.TE漏洞。再随便进一个帖子。发现user-agent字段存在XSS漏洞，把user-agent改为![](/image/237.png)发现利用成功。
接下来只需要构造![](/image/238.png)再发送一个正常请求。就可以利用成功了。如果要在浏览器显示响应，需要用![](/image/239.png)把链接复制到bp内嵌浏览器中。
### 将站内重定向转换为开放重定向
### Web缓存投毒
是CL.TE漏洞。
这里有一个重定向，编号7的帖子会被重定向到8，如果把Host改一下，就可以站外重定向。![](/image/240.png)![](/image/241.png)还可以找到一个包，就是![](/image/242.png)这里的`Cache-Control`是缓存时间，`Age`是已经过了多长时间。只要在缓存快过期了，发送一个攻击请求，用来污染后端，后端会重定向到我的服务器，![](/image/243.png)把host改为`exploit-0a75003703a5f4cc8187f78601f100e3.exploit-server.net` 这样在缓存进行到大约27秒时，发送攻击请求，再发送`/resources/js/tracking.js`这样就会拼接到攻击请求后面。实现重定向。![](/image/244.png)
这样再访问靶场界面就完成了。
### Web缓存欺骗
这是CL-TE漏洞，有一个登录界面，登陆后会显示APIkey，如果我们构造攻击请求![p](/image/245.png)在这样在缓存进行到大约27秒时，发送攻击请求，多发送几次，再发送`/resources/js/tracking.js`这样就会得到apikey。一旦有用户访问，就会触发。网速太慢了，操作不来，先鸽了。
## Bypassing access controls via HTTP/2 request tunnelling
这一关用的是