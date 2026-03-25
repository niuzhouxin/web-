## addr
源码
```python
from ipaddress import ip_address  
from flask import Flask, request, render_template, redirect, url_for, session, flash  
import subprocess  
import platform  
  
app = Flask(__name__)  
app.secret_key = 'secret_key_changed_in_container'  
  
@app.route('/')  
def index():  
    current_user = session.get('user')  
    return render_template('index.html', current_user=current_user)  
  
@app.route('/ping', methods=['POST'])  
def ping():  
    target = request.form.get('target', '')  
    current_user = session.get('user')  
    if current_user and current_user.upper() != 'ADMIN':  
        return render_template(  
            'index.html',   
            ping_result="只有管理员可以使用此工具。",   
            current_user=current_user  
        )  
    if not current_user:  
         return render_template(  
            'index.html',   
            ping_result="只有管理员可以使用此工具。",   
            current_user=None  
        )  
      
    if not target:  
        return render_template('index.html', ping_result="请输入目标地址", current_user=current_user)  
    try:  
        target = ip_address(target).compressed  
    except Exception:  
        return render_template('index.html', ping_result="ip地址非法", current_user=current_user)  
  
    param = '-n' if platform.system().lower() == 'windows' else '-c'  
    try:  
        command = f'ping {param} 4 {target}'  
        result = subprocess.run(  
            command,   
            shell=True,  
            capture_output=True,   
            text=True,   
            timeout=10  
        )  
        output = result.stdout if result.returncode == 0 else result.stderr  
        if not output:  
             output = "Ping 失败或无法解析主机。"  
  
    except subprocess.TimeoutExpired:  
        output = "请求超时。"  
    except Exception as e:  
        output = f"执行错误: {str(e)}"  
  
    return render_template('index.html', ping_result=output, current_user=current_user)  
  
@app.route('/set_user_session', methods=['POST'])  
def set_user_session():  
    username = request.form.get('username', '').strip()  
  
    if username.lower() == 'admin':  
        flash("禁止操作：不允许设置 'admin' 用户名！")  
        return redirect(url_for('index'))  
  
    session['user'] = username  
    flash(f"用户名已更新为: {username}")  
    return redirect(url_for('index'))  
  
@app.route('/logout')  
def logout():  
    session.pop('user', None)  
    flash("已退出登录。")  
    return redirect(url_for('index'))  
  
if __name__ == '__main__':  
    app.run(host='0.0.0.0', port=5000, debug=True)
```
本来想用伪造session，但是key貌似是假的。
但是观察到
```
if current_user and current_user.upper() != 'ADMIN': 
if username.lower() == 'admin':
```
用到了upper和lower函数，这两个函数用的是unicode规则，不是asciii规则，
unicode里有两个不同的i :
`i`和`ı`
在python中，
```
'ı'.upper() == 'I'      
'ı'.lower() == 'ı'      
```
所以就这样绕过了。
获取了admin权限。
接下来就命令执行了。
通过这一句
```python
target = ip_address(target).compressed
```
所以字符在ip语法层是非法的。
但是我输入`::1%;ls`就可以执行。
因为`ip_address("::1%;ls")`检测你是不是ipv4/ipv6地址。
在ipv6规范里，
```
IPv6地址 % zone_id
```
是合法语法。
所以`::1%;ls`在python看来，`::1`是合法的ipv6地址，后面就不会检测了。
然后就会交给`command = f'ping {param} 4 {target}' subprocess.run(command, shell=True)`执行了。
于是执行语句是
```
ping -c 4 ::1%;ls
```
`;`是命令分割符，他会先执行`ping -c 4 ::1%`再执行`ls`，从而命令执行。
但是如果`::1%;ls /`就会报错，应该是不可以有`/`，在 IP 地址中，`/` 用于表示 CIDR 子网掩码。`ip_address()` 函数一看到 `/` 就会抛出异常
所以就一步步走到根目录
```
::1%;cd ..;cd ..;cd ..;cat flag
```
得到flag。

