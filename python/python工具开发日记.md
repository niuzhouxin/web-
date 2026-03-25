如果要开发一些用于做ctf的web题的工具，requests模块十分重要。
## request模块
先写一个可以发送get请求，并获取响应码，请求头，响应头的脚本。
```python
import requests  
  
def header_checker(url):  
    if not url.startswith(("http://", "https://")): # 防止输入网址时忘记写协议头  
        url = "https://" + url  
    try:  
        response = requests.get(url,allow_redirects=False,timeout=5) # 避免重定向导致header缺失  
        headers = {k.lower(): v for k,v in response.headers.items()} # 响应头的键统一为小写  
        print("响应码:"+str(response.status_code))  
        print("响应头:"+str(headers))#先转换成字符串类型才可以进行拼接  
        print("请求头:"+str(response.request.headers))  
  
    except requests.exceptions.ConnectionError as e:  
        print(e)  
  
if __name__ == '__main__':  
    header_checker("www.baidu.com")
```
输出类似
```
响应码:200
响应头:{'cache-control': 'private, no-cache, no-store, proxy-revalidate, no-transform', 'content-encoding': 'gzip', 'content-length': '1145', 'content-type': 'text/html', 'pragma': 'no-cache', 'server': 'bfe', 'set-cookie': 'BDORZ=27315; max-age=86400; domain=.baidu.com; path=/', 'date': 'Wed, 11 Mar 2026 09:04:44 GMT'}
请求头:{'User-Agent': 'python-requests/2.32.5', 'Accept-Encoding': 'gzip, deflate, zstd', 'Accept': '*/*', 'Connection': 'keep-alive'}
```
## 扫描
如果要做一个类似`dirsearch`那样的路径扫描工具，就需要导入字典，实现批量扫描，批量发送请求。
所以就要写循环了
```python
import requests  
import re  
  
def request_sender(url,path): # 指定url和字典路径  
    if not url.startswith(("http://", "https://")):  
        url = "https://" + url  
    with open(path,"r",encoding="utf-8") as f:  
        while True:  
            line = f.readline() # 逐行读取字典内容  
            if not line: # 如果遇到空行，说明字典结束了，就break  
                break  
            print(line.strip()) # 用strip()去掉每行末尾的\n 
    try:  
        response = requests.get(url,allow_redirects=False,timeout=5)  
    except requests.exceptions.ConnectionError as e:  
        print(e)  
  
request_sender("https://www.baidu.com","./dicts.txt")
```
输出类似
```
!.gitignore
!.htaccess
!.htpasswd
%2e%2e//google.com
%2e%2e;/test
%3f/
%C0%AE%C0%AE%C0%AF
```
接下来只要把这些路径依次拼接到url后面，再依次发送请求，再捕获响应码就可以了。
```python
import requests  
import re  
  
def request_sender(url,path): # 指定url和字典路径  
    if not url.startswith(("http://", "https://")):  
        url = "https://" + url  
    with open(path,"r",encoding="utf-8") as f:  
        while True:  
            line = f.readline() # 逐行读取字典内容  
            if not line: # 如果遇到空行，说明字典结束了，就break  
                break  
            if re.search(r"/$",url): # 优化输入，因为有的url有最后的/,有的没有。  
                urls = url + line.strip()  
            else:  
                urls = url + "/" + line.strip()  
            try:  
                response = requests.get(urls, allow_redirects=False, timeout=5)  
                print(response.status_code)  
            except requests.exceptions.ConnectionError as e:  
                print(e)  
  
  
request_sender("https://www.baidu.com/","./dicts.txt")
```
理论上讲，这样已经完全可以进行扫描了，输出类似这样
```
404
404
404
404
404
404
404
404
...
```
但是有一个致命缺陷就是太慢了，一个字典大概又一万行，这样一个一个发包，得扫到猴年马月呀。
接下来就要考虑使用多线程，多进程或多协程这样的并发技术来提高效率。
## 用多进程进行优化
因为cpu有多个核，所以可以充分利用这多个核，同时开多个进程，来提高效率。
原来的速度
```python
import requests  
import re  
import time  
  
def request_sender(url, path):  # 指定url和字典路径  
    count = 0  
    start = time.time()  
    if not url.startswith(("http://", "https://")):  
        url = "https://" + url  
    with open(path, "r", encoding="utf-8") as f:  
        while True:  
            count += 1  
  
            line = f.readline()  # 逐行读取字典内容  
            if not line:  # 如果遇到空行，说明字典结束了，就break  
                break  
            if re.search(r"/$", url):  # 优化输入，因为有的url有最后的/,有的没有。  
                urls = url + line.strip()  
            else:  
                urls = url + "/" + line.strip()  
            try:  
                response = requests.get(urls, allow_redirects=False, timeout=5)  
                print(response.status_code)  
                if count % 100 == 0:  
                    print(time.time() - start)  
            except requests.exceptions.ConnectionError as e:  
                print(e)  
  
  
request_sender("https://www.baidu.com/", "./dicts.txt")
```
优化前的脚本发送100个请求花费了120秒，十分慢。优化后
```python
from os import cpu_count  
  
import requests  
import re  
from multiprocessing import Pool  
import time  
  
  
  
def request_sender(url,path): # 指定url和字典路径  
    count = 0  
    start = time.time()  
    if not url.startswith(("http://", "https://")):  
        url = "https://" + url  
    with open(path,"r",encoding="utf-8") as f:  
        while True:  
            count += 1  
  
            line = f.readline() # 逐行读取字典内容  
            if not line: # 如果遇到空行，说明字典结束了，就break  
                break  
            if re.search(r"/$",url): # 优化输入，因为有的url有最后的/,有的没有。  
                urls = url + line.strip()  
            else:  
                urls = url + "/" + line.strip()  
            try:  
                response = requests.get(urls, allow_redirects=False, timeout=5)  
                print(response.status_code)  
                if count % 100 == 0:  
                    print(count)  
                    print(time.time() - start)  
            except requests.exceptions.ConnectionError as e:  
                print(e)  
if __name__ == "__main__":  
    pool = Pool(processes=cpu_count())  
    p = Pool(cpu_count()*4)  # cpu核心数的四倍，可以调试，找到最佳的倍数
    p.apply_async(request_sender, args=("http://www.baidu.com","./dicts.txt"))  
    p.close()  
    p.join()
```
经过测试这个发送100次请求大概要花费7秒。相比很快了
回看时🥲🥲这里突然发现一个问题，就是我实际只写了
```python
p = Pool(cpu_count() * 4)  
p.apply_async(request_sender, args=("http://www.baidu.com", "./dicts.txt"))
```
实际上只启动了一个任务，虽然有很多进程，但是只有一个进程在工作。等于还是单进程，再改一下,正确做法是要把字典分成好多份，交给不同的进程取跑
```python
from os import cpu_count  
  
import requests  
import re  
from multiprocessing import Pool  
import time  
  
  
def request_sender(args):  # 指定url和字典路径  
    url, chunk = args # Pool.map每次只可以传入一个参数，所以把url和chunk打包成一个元组传入  
    count = 0  
    start = time.time()  
    if not url.startswith(("http://", "https://")):  
        url = "https://" + url  
    for line in chunk:  
        count += 1  
        urls = url.rstrip("/") + '/' + line # 直接去掉url最后的/，统一加上/，简化了一下代码  
        try:  
            response = requests.get(urls, allow_redirects=False, timeout=5)  
            print(response.status_code)  
            if count % 100 == 0: # 加的测试速度的功能，方便调试  
                print(count)  
                print(time.time() - start)  
        except requests.exceptions.ConnectionError as e:  
            print(e)  
  
  
if __name__ == "__main__":  
    n = cpu_count() * 4  
    with open("./dicts.txt","r",encoding="utf-8") as f:  
        lines = [line.strip() for line in f if line.strip()]  
        chunks = [lines[i::n] for i in range(n)]  
        with Pool(n) as p: # 可以自动关闭进程池，可以不用手写p.close() 和 p.join()了  
            p.map(request_sender,[("https://www.baidu.com",chunk) for chunk in chunks])  
            #map把列表里的每个元素依次分配给进程池里的进程
```
这样再测试一下，发现有一个问题
```
404
('Connection aborted.', ConnectionAbortedError(10053, '你的主机中的软件中止了一个已建立的连接。', None, 10053, None))
('Connection aborted.', ConnectionResetError(10054, '远程主机强迫关闭了一个现有的连接。', None, 10054, None))
('Connection aborted.', ConnectionAbortedError(10053, '你的主机中的软件中止了一个已建立的连接。', None, 10053, None))
404
404
404
```
执行会输出这个，这个好像是发送太快了被ban了，或者校园网的问题。把校园网拔了，再加个User-Agent试一下。
```python
from os import cpu_count  
  
import requests  
import re  
from multiprocessing import Pool  
import time  
  
  
def request_sender(args):  # 指定url和字典路径  
    url, chunk = args # Pool.map每次只可以传入一个参数，所以把url和chunk打包成一个元组传入  
    count = 0  
    start = time.time()  
    if not url.startswith(("http://", "https://")):  
        url = "https://" + url  
    for line in chunk:  
        count += 1  
        urls = url.rstrip("/") + '/' + line # 直接去掉url最后的/，统一加上/，简化了一下代码  
        headers = {  
            "User-Agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.169 Safari/537.36"  
        }  
        try:  
            response = requests.get(urls, headers = headers,allow_redirects=False, timeout=5)  
            print(response.status_code)  
            if count % 100 == 0: # 加的测试速度的功能，方便调试  
                print(count)  
                print(time.time() - start)  
        except requests.exceptions.ConnectionError as e:  
            print(e)  
  
  
if __name__ == "__main__":  
    n = cpu_count() * 4  
    with open("./dicts.txt","r",encoding="utf-8") as f:  
        lines = [line.strip() for line in f if line.strip()]  
        chunks = [lines[i::n] for i in range(n)]  
        with Pool(n) as p: # 可以自动关闭进程池，可以不用手写p.close() 和 p.join()了  
            p.map(request_sender,[("http://121.89.81.39/",chunk) for chunk in chunks])  
            #map把列表里的每个元素依次分配给进程池里的进程
```
扫描`www.baidu.com`还是不行，应该是有反爬机制，暂时不知道怎么办，以后再解决，我扫了一下，我的博客，没有任何限制，发现速度快多了。

但是这还是不够，想一下，字典里有一万行，还是太慢了。
再优化的化就要用到多线程了。
## 用多线程进行优化
想一下，多进程优化是通过充分利用cpu资源来提高效率的，而我们的目录扫描是要不断发送网络请求的，其中90%以上时间都是要花费在网络延迟上，而不是cpu上，只优化cpu资源利用效果是有限的。
像这样的IO密集型任务，利用多线程优化是最好的。
```python
  
  
import requests  
import re  
from concurrent.futures import ThreadPoolExecutor  
import time  
  
  
def request_sender(args):  # 指定url和字典路径  
    url, chunk = args # Pool.map每次只可以传入一个参数，所以把url和chunk打包成一个元组传入  
    count = 0  
    start = time.time()  
    if not url.startswith(("http://", "https://")):  
        url = "https://" + url  
    for line in chunk:  
        count += 1  
        urls = url.rstrip("/") + '/' + line # 直接去掉url最后的/，统一加上/，简化了一下代码  
        headers = {  
            "User-Agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.169 Safari/537.36"  
        }  
        try:  
            response = requests.get(urls, headers = headers,allow_redirects=False, timeout=5)  
            print(response.status_code)  
            if count % 100 == 0: # 加的测试速度的功能，方便调试  
                print(count)  
                print(time.time() - start)  
        except requests.exceptions.ConnectionError as e:  
            print(e)  
  
  
if __name__ == "__main__":  
    n = 100  
    with open("./dicts.txt","r",encoding="utf-8") as f:  
        lines = [line.strip() for line in f if line.strip()]  
        chunks = [lines[i::n] for i in range(n)]  
        start = time.time()  
        with ThreadPoolExecutor(max_workers=n) as executor:  
            executor.map(request_sender, [("http://www.baidu.com",chunk) for chunk in chunks])  
        print("总耗时："+ str(time.time()-start))
```
这样优化速度相对很快了，单还是有可以优化的点。
这个脚本每次都会建立新的TCP连接，`requests.get()` 会：
- TCP三次握手
- TLS握手
- HTTP请求
- 关闭连接
这些操作如果重复一万次的话也是很耗时间的。
要解决的话就要用的`Session`复用连接。
```python
import requests  
import re  
from concurrent.futures import ThreadPoolExecutor  
import time  
  
headers = {  
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.169 Safari/537.36"  
}  
def request_sender(args):  # 指定url和字典路径  
    url, chunk = args # Pool.map每次只可以传入一个参数，所以把url和chunk打包成一个元组传入  
    session = requests.Session()  
    if not url.startswith(("http://", "https://")):  
        url = "https://" + url  
    for line in chunk:  
        urls = url.rstrip("/") + '/' + line # 直接去掉url最后的/，统一加上/，简化了一下代码  
        try:  
            response = session.get(urls, headers = headers,allow_redirects=False, timeout=5)  
            if response.status_code != 404: # 不输出404了，避免终端输出过多  
                print(response.status_code)  
        except requests.exceptions.ConnectionError as e:  
            print(e)  
  
  
if __name__ == "__main__":  
    n = 100 # 线程数  
    with open("./dicts.txt","r",encoding="utf-8") as f:  
        lines = [line.strip() for line in f if line.strip()]  
    chunks = [lines[i::n] for i in range(n)]  
    start = time.time()  
    with ThreadPoolExecutor(max_workers=n) as executor:  
        executor.map(request_sender, [("http://www.baidu.com",chunk) for chunk in chunks])  
    print("总耗时："+ str(time.time()-start))
```
## 使用优化
前面的没有进度条，输出也有问题，用户用起来很难受。
```python
import requests  
import re  
from concurrent.futures import ThreadPoolExecutor  
import time  
from tqdm import tqdm  
  
headers = {  
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.169 Safari/537.36"  
}  
progress = None  
def request_sender(args):  # 指定url和字典路径  
    url, chunk = args # Pool.map每次只可以传入一个参数，所以把url和chunk打包成一个元组传入  
    session = requests.Session()  
    if not url.startswith(("http://", "https://")):  
        url = "https://" + url  
    for line in chunk:  
        urls = url.rstrip("/") + '/' + line # 直接去掉url最后的/，统一加上/，简化了一下代码  
        try:  
            response = session.get(urls, headers = headers,allow_redirects=False, timeout=2)  
            if response.status_code != 404: # 不输出404了，避免终端输出过多  
                print(str(response.status_code) + " " + line )  
        except requests.RequestException as e:  
            pass  
        finally:  
            progress.update(1)# 每个请求完自动更新  
  
if __name__ == "__main__":  
    n = 100 # 线程数  
    with open("./dicts.txt","r",encoding="utf-8") as f:  
        lines = [line.strip() for line in f if line.strip()]  
    chunks = [lines[i::n] for i in range(n)]  
    start = time.time()  
    processes = tqdm(total = len(lines) ,desc = "扫描进度",unit = "个")#进度条  
    with ThreadPoolExecutor(max_workers=n) as executor:  
        executor.map(request_sender, [("https://www.baidu.com",chunk) for chunk in chunks])  
    print("总耗时："+ str(time.time()-start))
```
但这个进度条有问题，因为多个线程，同时更新进度条，会导致进度条一直为0。加个锁就行了。
```python
  
from threading import Lock  
import requests  
import re  
from concurrent.futures import ThreadPoolExecutor  
import time  
from tqdm import tqdm  
  
headers = {  
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.169 Safari/537.36"  
}  
progress = None  
progress_lock = Lock()  
def request_sender(args):  # 指定url和字典路径  
    url, chunk = args # Pool.map每次只可以传入一个参数，所以把url和chunk打包成一个元组传入  
    session = requests.Session()  
    if not url.startswith(("http://", "https://")):  
        url = "https://" + url  
    for line in chunk:  
        urls = url.rstrip("/") + '/' + line # 直接去掉url最后的/，统一加上/，简化了一下代码  
        try:  
            response = session.get(urls, headers = headers,allow_redirects=False, timeout=2)  
            if response.status_code != 404: # 不输出404了，避免终端输出过多  
                print(str(response.status_code) + " " + line )  
        except requests.RequestException as e:  
            pass  
        finally:  
            with progress_lock:#加锁，确保只有一个线程更新进度条  
                progress.update(1)# 每个请求完自动更新  
  
if __name__ == "__main__":  
    n = 100 # 线程数  
    with open("./dicts.txt","r",encoding="utf-8") as f:  
        lines = [line.strip() for line in f if line.strip()]  
    chunks = [lines[i::n] for i in range(n)]  
    start = time.time()  
    progress = tqdm(total = len(lines) ,desc = "扫描进度",unit = "个",position=0, leave=True)#进度条 position=0 确保进度条固定，leave=True 保证执行后进度条不消失  
    with ThreadPoolExecutor(max_workers=n) as executor:  
        executor.map(request_sender, [("https://www.baidu.com",chunk) for chunk in chunks])  
    print("总耗时："+ str(time.time()-start))
```
这样进度条就差不多了。但是这样还是纯源码，没有进行任何封装。

## 后续封装
如果要后续修改的话，就要改成可以在cmd窗口直接操作，各种参数，例如timeout，线程数，都可以直接在cmd窗口中直接修改。
这样就可以用 `argparse` 模块：
```python
from email.policy import default  
from threading import Lock  
import requests  
import re  
from concurrent.futures import ThreadPoolExecutor  
import time  
from tqdm import tqdm  
import argparse  
  
headers = {  
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",  
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",  
    "Accept-Language": "zh-CN,zh;q=0.9,en;q=0.8",  
    "Accept-Encoding": "gzip, deflate",  
    "Connection": "keep-alive",  
}  
progress = None  
progress_lock = Lock()  
def request_sender(args):  # 指定url和字典路径  
    url, chunk = args # Pool.map每次只可以传入一个参数，所以把url和chunk打包成一个元组传入  
    session = requests.Session()  
    if not url.startswith(("http://", "https://")):  
        url = "https://" + url  
    for line in chunk:  
        urls = url.rstrip("/") + '/' + line # 直接去掉url最后的/，统一加上/，简化了一下代码  
        try:  
            response = session.get(urls, headers = headers,allow_redirects=False, timeout=2)  
            if response.status_code != 404: # 不输出404了，避免终端输出过多  
                progress.write(str(response.status_code) + " " + line )  
        except requests.RequestException as e:  
            pass  
        finally:  
            with progress_lock:#加锁，确保只有一个线程更新进度条  
                progress.update(1)# 每个请求完自动更新  
  
if __name__ == "__main__":  
    parser = argparse.ArgumentParser(description="目录扫描工具")  
    parser.add_argument("-u", "--url",required=True ,help="目标url")  
    parser.add_argument("-d" ,"--dict" , default="./dicts.txt", help="字典路径，默认 ./dicts.txt")  
    parser.add_argument("-n", "--threads", default=40, type=int, help="线程数，默认 40")  
    args = parser.parse_args()  
    n = 40 # 线程数  
    with open(args.dict,"r",encoding="utf-8") as f:  
        lines = [line.strip() for line in f if line.strip()]  
    chunks = [lines[i::args.threads] for i in range(args.threads)]  
    start = time.time()  
    progress = tqdm(total = len(lines) ,desc = "扫描进度",unit = "个",position=0, leave=True)#进度条 position=0 确保进度条固定，leave=True 保证执行后进度条不消失  
    with ThreadPoolExecutor(max_workers=args.threads) as executor:  
        executor.map(request_sender, [(args.url,chunk) for chunk in chunks])  
    print("总耗时："+ str(time.time()-start))
```
线程多了环境承受不住，少了太慢，没招了。🥲，但也学到了一些工具开发时用到的一些编程手法。
