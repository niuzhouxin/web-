## 一.F12
按键盘上的F12可以打开开发者工具，或者右键网页选择检查。`Ctrl + Shift + I` 和`Ctrl + Shift + j`或者点击浏览器右上角三个点->更多工具->开发者工具
## 抓包
抓包就是把客户端对服务端的请求拦截，并且捕获服务端到客户端的响应。
**把get包改为post**:右键包，选择修改请求方式
**post传参的格式**:在post包的最下面隔一行写，格式为`变量名=参数`
pretty:美化呈现响应包的内容
raw:原始未美化的响应包
## http巩固
客户端就是发起请求的那一端，服务端就是接受请求返回数据的那一端
请求包包含客户端的请求信息，响应包包含服务端的回复信息
**请求头**：`User-Agent`表明了是哪个客户端发起请求，如(chorme浏览器)
`cotent-type`请求体的数据格式（如json,图片等）
`cookie`携带客户端的身份信息，如登录状态，会话ID
`host`要访问的服务端域名，如`www.baidu.com`
referer：可以识别请求的来源，比如你从百度搜索结果点进某网站，该网站的服务器会收到 Referer 为 “[www.baidu.com](https://www.baidu.com/)” 的信息。
**请求体**:主要有三种格式json格式{"username":"jg","ga":"agd"}
表单格式username=xxx&password=12423
二进制格式，图片，文件的原始数据
**响应头**：Status-code,响应码如200成功,404不存在
Content-Type：说明响应体的数据格式（如 HTML、JSON、图片）。 Cache-Control：指定数据缓存规则（如是否缓存、缓存时长）。
 Set-Cookie：服务端向客户端设置 Cookie（如登录后的身份凭证）。
 **响应体**：html格式的网页源码，json格式的和二进制格式的
## robots.
 robots.txt 是网站主动提供的 “爬虫访问协议”，明确告知爬虫 “允许抓取的范围、禁止抓取的目录、抓取频率限制” 等规则。1. 合规爬虫访问网站时，会先请求 `网站域名/robots.txt`（如 `www.baidu.com/robots.txt`），获取规则。但是君子协议，可以不遵守

## GET请求POST请求
GET请求有长度限制，限长2048，POST请求无长度限制
 


