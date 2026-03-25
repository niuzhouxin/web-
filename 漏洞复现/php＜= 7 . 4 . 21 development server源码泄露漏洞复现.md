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
同理如果你要让他当作php解析，就随便写一个php后缀的文件名。要读取源码就用txt后缀，执行代码就用php后缀。
## 参考文章
https://www.cnblogs.com/Kawakaze777/p/17799235.html
https://projectdiscovery.io/blog/php-http-server-source-disclosure


