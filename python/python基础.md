## 切片
例如
```python
L = ['Michael', 'Sarah', 'Tracy', 'Bob', 'Jack']  
print(L[0:3])
#['Michael', 'Sarah', 'Tracy']
```
这里的`L[0:3]`表示取索引值从0到3的内容，当然不包括3。这里的0可以省略，写成`L[:3]`也可以。
python还支持倒数切片，
```python
L = ['Michael', 'Sarah', 'Tracy', 'Bob', 'Jack']  
print(L[-2:])
#['Bob', 'Jack']
#['Bob']
```
这里的`L[-2:]`表示取从倒数第二个到最后。`L[-2:-1]`表示取倒数第二个到倒数第一个，不包括，最后一个。
切片还支持隔几个取一个数
```python
L = list(range(100))  
print(L[10:20:2])
#[10, 12, 14, 16, 18]
```
这就是隔两个取一个数。
如果只写`L[:]`就表示原封不动复制这个列表。
同样的，元组，字符串都支持切片操作。
## 迭代
主要是通过for循环遍历可迭代对象，其中`list`，`dict`，`String`都可以迭代，整数不可以迭代。
```python
for i in list(range(3)):  
    print(i) 
#0
#1
#2
d = {'a': 1, 'b': 2, 'c': 3}  
for key in d:  
   print(key)
a
b
c
for key in "ABC":  
   print(key)
A
B
C
```
这里字典的遍历是默认遍历键的，如果想遍历值，需要。
```python
d = {'a': 1, 'b': 2, 'c': 3}  
for key in d.values():  
   print(key)
1
2
3
```
Python内置的`enumerate`函数可以把一个`list`变成索引-元素对，这样就可以在`for`循环中同时迭代索引和元素本身：
```python
d = {'a': 1, 'b': 2, 'c': 3}  
for key,value in enumerate(d):  
   print(key,value)
0 a
1 b
2 c
```
python的for循环里同时引用两个变量是很常见的。例如
```python
for i,j in [(1,1),(2,4),(3,9)]:  
   print(i,j)
1 1
2 4
3 9
```
## 列表生成式
```python
L=[]  
for i in range(1,10):  
    x = i*i  
    L.append(x)  
print(L)
#[1, 4, 9, 16, 25, 36, 49, 64, 81]
```
这样写很麻烦，有更简介的写法。
```python
print([x*x for x in range(1,10)])
```
这样一行搞定。
如果要输出偶数平方，可以加if语句。
```python
print([x*x for x in range(1,10) if x%2==0])
```
还可以有两层循环
```python
print([m+n for m in "ABC" for n in "XYZ"])
```
这样可以输出全排列。
实际应用
```python
import os
print([x for x in os.listdir(".")])
```
这样可以以列表的形式列出文件，更美观。
**if-else**用法
```python
print([x if x%2==0 else -x for x in range(1,10)])
#[-1, 2, -3, 4, -5, 6, -7, 8, -9]
```
## 生成器
如果列表很大，很占用空间，如果列表元素可以按照某种算法推算出来，那我们是否可以在循环的过程中不断推算出后续的元素呢？这样就不必创建完整的list，从而节省大量的空间。在Python中，这种一边循环一边计算的机制，称为生成器：generator。
要创建一个generator，有很多种方法。第一种方法很简单，只要把一个列表生成式的`[]`改成`()`，就创建了一个generator：
```python
g = (x*x for x in range(10))  
print(g)
#<generator object <genexpr> at 0x000001A6C36A6420>
```
这里的g就是生成器。
如果要打印`generator`的元素，需要用到`next()`
```python
print(next(g))  
print(next(g))  
print(next(g))  
print(next(g))  
print(next(g))  
print(next(g))  
print(next(g))  
print(next(g))  
print(next(g))
0
1
4
9
16
25
36
49
64
```
generator保存的是算法，每次调用`next(g)`，就计算出`g`的下一个元素的值，直到计算到最后一个元素，没有更多的元素时，抛出`StopIteration`的错误。
当然不可以一个一个来，因为`generator`也是可迭代对象，就可以`for`循环。
```python
for n in g:  
    print(n)
0
1
4
9
16
25
36
49
64
81
```
这是一个输出斐波那契数列的函数
```python
def fib(max):
    n, a, b = 0, 0, 1 #a=0 b=1
    while n < max: 
        print(b)
        a, b = b, a + b #
        n = n + 1
    return 'done'
```
其中`a, b = b, a + b`等价于
```python
t = (b, a + b) # t是一个tuple
a = t[0]
b = t[1]
```
这是python的同步赋值（元组解包）。
python执行`a, b = b, a + b`时会分两步执行，
第 1 步：先把右边全部计算完毕，打包成一个元组
```python
a, b = b, a + b
#      ^^^^^^^^
#      右边先变成 (b, a+b) 这个元组，值已经固定了
```
第 2 步：再把元组的值依次赋给左边的变量
```python
a = b        # 拿元组第一个值
b = a + b    # 拿元组第二个值（此时 a+b 已经是旧值算好的结果）
```
这样就不用显示写出临时变量就可以赋值。
这样的话直接`a,b = b,a`可以直接交换两个变量的值，不必设置临时变量。
这样逻辑上就很类似生成器了，要把`fib`函数变成generator函数，只需要把`print(b)`改为`yield b`就可以了：
```python
def fib(max):  
    n, a, b = 0, 0, 1  
    while n < max:  
        yield b  
        a, b = b, a + b  
        n = n + 1  
    return 'done'  
print(fib(100))
#<generator object fib at 0x000002757FCC6420>
```
如果要输出就用for循环
```python
for n in fib(100):  
    print(n)
```
一个生成杨辉三角的生成器函数。
```python
def triangles():  
    n = [1]  
    while True:  
        yield n  
        n = [1]+[n[i]+n[i+1] for i in range(len(n) - 1) ]+[1]
```
## 多进程
Unix/Linux操作系统提供了一个`fork()`系统调用，它非常特殊。普通的函数调用，调用一次，返回一次，但是`fork()`调用一次，返回两次，因为操作系统自动把当前进程（称为父进程）复制了一份（称为子进程），然后，分别在父进程和子进程内返回。

子进程永远返回`0`，而父进程返回子进程的ID。这样做的理由是，一个父进程可以fork出很多子进程，所以，父进程要记下每个子进程的ID，而子进程只需要调用`getppid()`就可以拿到父进程的ID。

Python的`os`模块封装了常见的系统调用，其中就包括`fork`，可以在Python程序中轻松创建子进程：
```python
import os  
  
print("process (%s) start ..." % os.getpid())
```
这样可以打印进程id。
输出类似
```
process (396) start ...
I (396) just creat a child process (397)
I' am a child process (397) and my parent process is 396
```
由于Windows没有`fork`调用，上面的代码在Windows上无法运行。
虽然如此，但是在windows上依然可以多进程服务，`multiprocessing`模块就是跨平台版本的多进程模块。
`multiprocessing`模块提供了一个`Process`类来代表一个进程对象，下面的例子演示了启动一个子进程并等待其结束：
```python
from multiprocessing import Process  
import os  
  
# 子进程要执行的代码  
def run_proc(name):  
    print('Run child process %s (%s)...' % (name, os.getpid()))  
  
if __name__=='__main__':  
    print('Parent process %s.' % os.getpid())  
    p = Process(target=run_proc, args=('test',))  
    print('Child process will start.')  
    p.start()#启动子进程，开始执行run_proc  
    p.join() #  
    print('Child process end.')
```
运行实例
```
Parent process 28636.
Child process will start.
Run child process test (38472)...
Child process end.
```
创建子进程时，只需要传入一个执行函数和函数的参数，创建一个`Process`实例，用`start()`方法启动，这样创建进程比`fork()`还要简单。

`join()`方法可以等待子进程结束后再继续往下运行，通常用于进程间的同步。
**Pool**
如果要启动大量的子进程，可以用进程池的方式批量创建子进程：
```python
from multiprocessing import Pool  
import os,time,random  
  
def long_time_task(name):  
    print("run task %s (%s)" % (name,os.getpid()))  
    start = time.time()  
    time.sleep(random.random() * 3)  
    end = time.time()  
    print("task %s run %0.2f seconds" % (name,end-start))  
if __name__ == '__main__':  
    print("parent process is %s" % os.getpid())  
    p = Pool(4)  
    for i in range(5):  
        p.apply_async(long_time_task,args=(i,))  
    print('Waiting for all subprocesses done...')  
    p.close()  
    p.join()  
    print('All subprocesses done!')
```
运行结果示例
```
parent process is 46112
Waiting for all subprocesses done...
run task 0 (29728)
run task 1 (48052)
run task 2 (33376)
run task 3 (44252)
task 3 run 0.27 seconds
run task 4 (44252)
task 4 run 0.05 seconds
task 2 run 0.45 seconds
task 1 run 0.86 seconds
task 0 run 1.66 seconds
All subprocesses done!
```
对`Pool`对象调用`join()`方法会等待所有子进程执行完毕，调用`join()`之前必须先调用`close()`，调用`close()`之后就不能继续添加新的`Process`了。

请注意输出的结果，task `0`，`1`，`2`，`3`是立刻执行的，而task `4`要等待前面某个task完成后才执行，这是因为`Pool`的默认大小在我的电脑上是4，因此，最多同时执行4个进程。这是`Pool`有意设计的限制，并不是操作系统的限制。如果改成：

```python
p = Pool(5)
```

就可以同时跑5个进程。

由于`Pool`的默认大小是CPU的核数，如果你不幸拥有8核CPU，你要提交至少9个子进程才能看到上面的等待效果。
**子进程**
很多时候，子进程并不是自身，而是一个外部进程。我们创建了子进程后，还需要控制子进程的输入和输出。

`subprocess`模块可以让我们非常方便地启动一个子进程，然后控制其输入和输出。

下面的例子演示了如何在Python代码中运行命令`nslookup www.python.org`，这和命令行直接运行的效果是一样的：
```python
from multiprocessing import Pool  
import os  
import subprocess  
  
print("$ nslookup www.python.org")  
r = subprocess.call(['nslookup','www.python.org'])  #以列表形式传入要执行的外部命令的命令和参数，交给subprocess.call来执行
print('exit code:',r) # r就是退出码
```
输出结果
```
$ nslookup www.python.org
Non-authoritative answer:
Server:  ns.sc.cninfo.net
Address:  61.139.2.69

Name:    dualstack.python.map.fastly.net
Addresses:  2a04:4e42:400::223
	  2a04:4e42:600::223
	  2a04:4e42::223
	  2a04:4e42:200::223
	  151.101.64.223
	  151.101.128.223
	  151.101.192.223
	  151.101.0.223
Aliases:  www.python.org

exit code: 0
```
如果子进程还需要输入，则可以通过`communicate()`方法输入：
```python
import subprocess  
  
print("$ nslookup")  
p = subprocess.Popen(['nslookup'], stdout=subprocess.PIPE,stderr=subprocess.PIPE,stdin=subprocess.PIPE)  
output,err = p.communicate(b'set q=mx\npython.org\nexit\n')  
print(output.decode('utf-8'))  
print('exit code:',p.returncode)
```
上面的代码相当于在命令行执行命令`nslookup`，然后手动输入：
```
set q=mx
python.org
exit
```
运行结果
```
$ nslookup
Default Server:  ns.sc.cninfo.net
Address:  61.139.2.69

> > Server:  ns.sc.cninfo.net
Address:  61.139.2.69

python.org	MX preference = 50, mail exchanger = mail.python.org
> 
exit code: 0
```
**进程间通信**
`Process`之间肯定是需要通信的，操作系统提供了很多机制来实现进程间的通信。Python的`multiprocessing`模块包装了底层的机制，提供了`Queue`、`Pipes`等多种方式来交换数据。

我们以`Queue`为例，在父进程中创建两个子进程，一个往`Queue`里写数据，一个从`Queue`里读数据：
```python
from multiprocessing import Process,Queue  
import os,time,random  
  
def write(q):  
    print('process to write %s' % os.getpid())  
    for i in ['A','B','C']:  
        print('put %s to queue' % i)  
        q.put(i)  
        time.sleep(random.random())  
def read(q):  
    print('process to read %s' % os.getpid())  
    while True:  
        value = q.get(True)#这里的True表示阻塞模式，队列有数据就直接取出，队列无数据就等待，直到有数据为止  
        print('get %s from queue' % value)  
if __name__ == '__main__':  
    # 父进程创建Queue,并传给各个子进程  
    q = Queue()  
    pw = Process(target=write, args=(q,))  
    pr = Process(target=read, args=(q,))  
    # 启动子进程pw，写入  
    pw.start()  
    # 启动子进程pr，读取  
    pr.start()  
    #等待pw结束  
    pw.join()  
    # pr进程里是死循环，无法等待其结束，只能强行终止:  
    pr.terminate()
```
运行结果如下
```
process to write 43112
put A to queue
process to read 41976
get A from queue
put B to queue
get B from queue
put C to queue
get C from queue
```
**为什么需要多进程**
1. 绕过GIL限制 ：Python的全局解释器锁（GIL）限制多线程并行，多进程可真正实现并行计算
2. 充分利用多核CPU：现代CPU多为多核，多进程可充分发挥硬件性能
3. 提高程序稳定性：进程间内存隔离，一个进程崩溃不会影响其他进程 适合CPU密集型任务：科学计算、数据处理、图像处理等
## 多线程
多任务可以由多个进程完成，也可以由一个进程内的多个线程运行。
进程是由若干个线程组成的。
Python的标准库提供了两个模块：`_thread`和`threading`，`_thread`是低级模块，`threading`是高级模块，对`_thread`进行了封装。绝大多数情况下，我们只需要使用`threading`这个高级模块。
启动一个线程就是把一个函数传入并创建`Thread`实例，然后调用`start()`开始执行：
```python
import time,threading  
  
def loop():  
    print('thread %s is running' % threading.current_thread().name)  
    # threading.current_thread() 获取当前线程对象  
    # .name 获取线程名字（创建时指定的 'LoopThread'）  
    n = 0  
    while n < 5:  
        n = n + 1  
        print('thread %s >>> %s' % (threading.current_thread().name,n))  
        time.sleep(1)  
    print('thread %s is done' % threading.current_thread().name)  
  
if __name__ == '__main__':  
    print('thread %s is running...' % threading.current_thread().name)  
    t = threading.Thread(target=loop,name = 'LoopThread')  
    t.start()  
    t.join()  
    print('thread %s ended.' % threading.current_thread().name)
```
执行结果如下
```python
thread MainThread is running...
thread LoopThread is running
thread LoopThread >>> 1
thread LoopThread >>> 2
thread LoopThread >>> 3
thread LoopThread >>> 4
thread LoopThread >>> 5
thread LoopThread is done
thread MainThread ended.
```
其中`MainThread`就是主线程。
由于任何进程默认就会启动一个线程，我们把该线程称为主线程，主线程又可以启动新的线程，Python的`threading`模块有个`current_thread()`函数，它永远返回当前线程的实例。主线程实例的名字叫`MainThread`，子线程的名字在创建时指定，我们用`LoopThread`命名子线程。名字仅仅在打印时用来显示，完全没有其他意义，如果不起名字Python就自动给线程命名为`Thread-1`，`Thread-2`……
**Lock**
多线程和多进程最大的不同在于，多进程中，同一个变量，各自有一份拷贝存在于每个进程中，互不影响，而多线程中，所有变量都由所有线程共享，所以，任何一个变量都可以被任何一个线程修改，因此，线程之间共享数据最大的危险在于多个线程同时改一个变量，把内容给改乱了。

来看看多个线程同时操作一个变量怎么把内容给改乱了：
```python
# multithread
import time, threading

# 假定这是你的银行存款:
balance = 0

def change_it(n):
    # 先存后取，结果应该为0:
    global balance # 因为要修改balance变量，如果不声明global，它可能会认为这是局部变量
    balance = balance + int(n)
    balance = balance - int(n)

def run_thread(n):
    for i in range(10000000):
        change_it(n)

t1 = threading.Thread(target=run_thread, args=(5,))
t2 = threading.Thread(target=run_thread, args=(8,))
t1.start()
t2.start()
t1.join()
t2.join()
print(balance)
```
我们定义了一个共享变量`balance`，初始值为`0`，并且启动两个线程，先存后取，理论上结果应该为`0`，但是，由于线程的调度是由操作系统决定的，当`t1`、`t2`交替执行时，只要循环次数足够多，`balance`的结果就不一定是`0`了。

原因是因为高级语言的一条语句在CPU执行时是若干条语句，即使一个简单的计算：
```
balance = balance + n
```
也分两步：

1. 计算`balance + n`，存入临时变量中；
2. 将临时变量的值赋给`balance`。
究其原因，是因为修改`balance`需要多条语句，而执行这几条语句时，线程可能中断，从而导致多个线程把同一个对象的内容改乱了。

两个线程同时一存一取，就可能导致余额不对，你肯定不希望你的银行存款莫名其妙地变成了负数，所以，我们必须确保一个线程在修改`balance`的时候，别的线程一定不能改。
如果我们要确保`balance`计算正确，就要给`change_it()`上一把锁，当某个线程开始执行`change_it()`时，我们说，该线程因为获得了锁，因此其他线程不能同时执行`change_it()`，只能等待，直到锁被释放后，获得该锁以后才能改。由于锁只有一个，无论多少线程，同一时刻最多只有一个线程持有该锁，所以，不会造成修改的冲突。创建一个锁就是通过`threading.Lock()`来实现：
```python
# 假定这是你的银行存款:  
balance = 0  
lock = threading.Lock()  
  
def change_it(n):  
    # 先存后取，结果应该为0:  
    global balance  
    balance = balance + n  
    balance = balance - n  
def run_thread(n):  
    for i in range(1000000):  
        lock.acquire()  # 要先获得锁  
        try:  
            change_it(n)  
        finally:  
            lock.release() # 改完了释放锁
```
获得锁的线程用完后一定要释放锁，否则那些苦苦等待锁的线程将永远等待下去，成为死线程。所以我们用`try...finally`来确保锁一定会被释放。
```python
print(multiprocessing.cpu_count())
# 20
```
可以看到电脑cpu有二十个核。
## ThreadLocal
在多线程环境下，每个线程都有自己的数据。一个线程使用自己的局部变量比使用全局变量好，因为局部变量只有线程自己能看见，不会影响其他线程，而全局变量的修改必须加锁。

但是局部变量也有问题，就是在函数调用的时候，传递起来很麻烦：
```python
def process_student(name):
    std = Student(name)
    # std是局部变量，但是每个函数都要用它，因此必须传进去：
    do_task_1(std)
    do_task_2(std)

def do_task_1(std):
    do_subtask_1(std)
    do_subtask_2(std)

def do_task_2(std):
    do_subtask_2(std)
    do_subtask_2(std)
```
每个函数一层一层调用都这么传参数那还得了？用全局变量？也不行，因为每个线程处理不同的`Student`对象，不能共享。
这时候就可以用`ThreadLocal`
## 正则表达式
字符串是编程时涉及到的最多的一种数据结构，对字符串进行操作的需求几乎无处不在。比如判断一个字符串是否是合法的Email地址，虽然可以编程提取`@`前后的子串，再分别判断是否是单词和域名，但这样做不但麻烦，而且代码难以复用。

正则表达式是一种用来匹配字符串的强有力的武器。它的设计思想是用一种描述性的语言来给字符串定义一个规则，凡是符合规则的字符串，我们就认为它“匹配”了，否则，该字符串就是不合法的。
所以我们判断一个字符串是否是合法的Email的方法是：
1. 创建一个匹配Email的正则表达式；
2. 用该正则表达式去匹配用户的输入来判断是否合法。
在正则表达式中，如果直接给出字符，就是精确匹配，用`\d`可以匹配一个数字，`\w`可以匹配一个字母或数字。
- `'00\d'`可以匹配`'007'`，但无法匹配`'00A'`；
- `'\d\d\d'`可以匹配`'010'`；
- `'\w\w\d'`可以匹配`'py3'`；
`.`可以匹配任意字符，所以`py.`可以匹配`pyc`，`pyo`，`py!`
要匹配变长的字符，在正则表达式中`*`表示任意字符（包括0个），用`+`表示至少一个字符，用`?`表示0个或1个字符，用`{n}`表示n个字符，用`{n,m}`表示n-m个字符。
例如`\d{3}\s+\d{3,8}`
- `\d{3}`表示匹配三个数字，例如`090`
- `\s`表示匹配一个空格（也包括Tab等空白字符），`\s+`表示至少一个空格,` `或`  `等。
- `\d{3,8}`表示匹配3-8个字符，例如`123456`
如果要匹配`010-1423`这样的号码，由于在正则中`-`有特殊含义，需要用`\`转义，所以上面的正则是`\d{3}\-\d{3,8}`
匹配需要用到re模块的match
```python
import re  
  
print(re.match(r'\d{3}\-\d{3,8}','010-4325'))
#<re.Match object; span=(0, 8), match='010-4325'>
```
如果要做更进阶的匹配，可以用`[]`表示范围：
- `[0-9a-zA-Z\_]`可以匹配一个字母，数字或者下划线。
- `[0-9a-zA-Z\_]+`可以至少匹配一个字母，数字，下划线组成的字符串，比如`0fds_`
- `[a-zA-Z\_][0-9a-zA-Z\_]*`可以匹配由字母或下划线开头，后接任意个由一个数字、字母或者下划线组成的字符串，也就是Python合法的变量名；
- `[a-zA-Z\_][0-9a-zA-Z\_]{0,19}`这样就可以限制匹配长度1-20
- `A|B`表示匹配A或者B，所以`(P|p)ython`可以匹配`'Python'`或者`'python'`。
- `^`表示行的开头，`^\d`表示行必须以数字开头。


## 参考
https://liaoxuefeng.com/books/python/introduction/index.html