在 Burp Suite（BP）中，若要将抓到的 GET 请求包重放并修改为 POST 类型，可按以下步骤操作：

1. 先在 Burp 的 “Proxy” 模块的 “History” 中找到目标 GET 请求，右键选择 “Send to Repeater”，将其发送到 “Repeater” 模块。
    
2. 在 “Repeater” 界面，找到请求行（通常以 “GET” 开头，如`GET /path HTTP/1.1`），直接将开头的 “GET” 改为 “POST”。
    
3. 修改请求方法后，需要添加 POST 请求所需的 “Content - Length” 头（用于指定请求体长度），还可根据实际需求添加 “Content - Type” 头（如表单提交常用`application/x-www-form-urlencoded`）。
    
4. 在请求头下方的空白区域（即请求体部分），输入 POST 请求需要提交的数据（如`key=value&param=123`）。
    
5. 完成修改后，点击 “Send” 按钮即可发送修改后的 POST 请求，并查看响应结果。