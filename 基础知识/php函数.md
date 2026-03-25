- `strstr()`:用于在一个字符串中查找另一个字符串的首次出现，并返回从该位置到原字符串末尾的子字符串（包括匹配的部分）。如果未找到，则返回 `false`。
- `stristr()`:功能与 `strstr()` 基本一致，**唯一区别是它在搜索时不区分大小写**（大小写不敏感）。
- `json_decode()` 是 PHP 中的一个函数，用于将 JSON 格式的字符串转换为 PHP 变量（如对象或数组）。
### 基本用法：

```php
$jsonString = '{"name":"Tom", "age":20}';
$phpVar = json_decode($jsonString);

// 此时 $phpVar 是一个对象
echo $phpVar->name; // 输出：Tom
```

### 常用参数：

- 第二个参数为 `true` 时，会返回数组而不是对象：

  ```php
   $phpArray = json_decode($jsonString, true);
   echo $phpArray['age']; // 输出：20
   ```
- `mysql_real_escape_string`是 PHP 中一个用于处理 MySQL 数据库查询字符串安全的函数，主要作用是**对字符串中的特殊字符进行转义**，以防止 SQL 注入攻击。
- `preg_replace('/AND/i',"", $id);`将参数$id里的AND不区分大小写的替换为空，也就是直接删了。
- `intval($var, $base)`，函数若$base=0,intval会自动识别数字进制，以0x开头为十六进制，以0开头（非0x）,识别为八进制，以数字1-9开头识别为十进制，当 $base不填时则强制转换为十进制
- `strchr()` 是 PHP 中的一个字符串处理函数，用于查找字符串在另一个字符串中首次出现的位置，并返回从该位置到字符串结尾的子串（包括匹配的字符）。若 `$needle` 未在 `$haystack` 中找到，返回 `false`。
### 一、文件包含与路径处理函数

1. **`include` / `require` 系列**
    
    - 漏洞场景：文件包含漏洞（LFI/RFI）。若包含的文件名可控，可通过 `../` 目录遍历读取敏感文件，或包含远程恶意脚本（需 `allow_url_include=On`）。
    - 示例：
        
        php
        
        ```php
        // 危险代码：$file可控
        include $_GET['file']; 
        // 利用：?file=../../etc/passwd 读取系统文件
        ```
        
2. **`file_get_contents($filename)` / `file_put_contents($filename, $data)`**
    
    - 漏洞场景：任意文件读取 / 写入。若 `$filename` 或 `$data` 可控，可读取服务器文件（如 `/flag`）或写入 webshell。
    - 示例：
        
        php
        
        ```php
        // 读取漏洞：?file=/flag
        echo file_get_contents($_GET['file']);
        // 写入漏洞：?data=<?php eval($_POST['cmd']);?>
        file_put_contents('shell.php', $_GET['data']);
        ```
        
3. **`fopen($file, $mode)` / `fread($handle, $length)`**
    
    - 漏洞场景：同 `file_get_contents`，可通过可控路径读取 / 写入文件。

### **二、命令执行与代码注入函数**

1. **`exec()` / `shell_exec()` / `system()` / `passthru()`**
    
    - 漏洞场景：命令注入。若函数参数可控，可拼接恶意命令（如 `;`、`|`、`&&` 分隔）。
    - 示例：
        
        php
        
        ```php
        // 危险代码：$ip可控
        system("ping " . $_GET['ip']); 
        // 利用：?ip=127.0.0.1;cat /flag 执行命令
        ```
        
2. **`eval()` / `assert()`**
    
    - 漏洞场景：代码注入。若传入字符串可控，可执行任意 PHP 代码。
    - 示例：
        
        php
        
        ```php
        // 危险代码：$code可控
        eval($_GET['code']); 
        // 利用：?code=phpinfo(); 或 ?code=system('cat /flag');
        ```
        
3. **`call_user_func($func, $param)` / `call_user_func_array()`**
    
    - 漏洞场景：函数调用注入。若 `$func` 可控，可调用危险函数（如 `system`、`eval`）。
    - 示例：
        
        php
        
        ```php
        call_user_func($_GET['func'], $_GET['param']); 
        // 利用：?func=system&param=cat /flag
        ```
        

### **三、字符串与序列化函数**

1. **`unserialize($str)`**
    
    - 漏洞场景：反序列化漏洞。若 `$str` 可控，且类中存在魔术方法（`__construct`、`__destruct` 等），可构造恶意序列化字符串执行代码。
    - 示例：
        
        php
        
        ```php
        class Test {
            function __destruct() {
                system($this->cmd);
            }
        }
        unserialize($_GET['data']); // 可控data可注入cmd
        ```
        
2. **`preg_replace('/pattern/e', $replacement, $str)`**
    
    - 漏洞场景：正则代码注入。`e` 修饰符会将 `$replacement` 作为 PHP 代码执行，若 `$replacement` 可控则危险。
    - 示例：
        
        php
        
        ```php
        preg_replace('/(.*)/e', $_GET['rep'], 'test'); 
        // 利用：?rep=system('cat /flag') 执行命令
        ```
        
3. **`str_replace()` / `preg_replace()`**
    
    - 漏洞场景：过滤绕过。若用于过滤危险字符（如 `<?php`、`system`），可能因规则不严谨被绕过（如大小写、编码、特殊字符拼接）。

### **四、其他高危函数与场景**

1. **`extract($_GET)`**
    
    - 漏洞场景：变量覆盖。将数组键值转为变量，若用户可控数组，可覆盖程序原有变量（如覆盖登录验证的 `$isAdmin`）。
    - 示例：
        
        php
        
        ```php
        extract($_GET); // 传入?isAdmin=1 覆盖原有$isAdmin
        if ($isAdmin) { echo $flag; }
        ```
        
2. **`parse_str($str)`**
    
    - 漏洞场景：变量覆盖。类似 `extract`，解析字符串为变量，可控 `$str` 可覆盖变量。
3. **`session_start()` 与 `session_id()`**
    
    - 漏洞场景：会话固定 / 劫持。若 `session_id` 可控，可伪造会话 ID 欺骗认证。
4. **`highlight_file()` / `show_source()`**
    
    - 场景：读取文件源码。常用于泄露当前脚本代码（如题目给出 `?file=index.php` 显示源码找漏洞）。
5. **`md5()` / `sha1()`**
    
    - 场景：哈希碰撞。若用哈希值比较验证（如 `md5($a) == md5($b)`），可利用数组或特殊字符串（如 `ffifdyop` 生成 `'or'6\xc9]\x99\xe9!r,\xf9\xedb\x1c` 绕过）。

### **五、防御绕过相关函数**

- **`base64_encode()` / `base64_decode()`**：常被用于编码绕过过滤（如将恶意代码 base64 编码后传输，解码后执行）。
- **`urldecode()` / `rawurldecode()`**：URL 解码可能绕过基于 URL 编码的过滤（如二次解码）。
- **`chr()`**：通过 ASCII 码拼接字符绕过关键字过滤（如 `chr(115).chr(121).chr(115).chr(116).chr(101).chr(109)` 拼接 `system`）。
- `preg_quote()` 是 PHP 中用于处理正则表达式模式的函数，主要作用是**对字符串中的正则特殊字符进行转义**，避免这些字符被正则引擎解释为特殊语法（如量词、分组等），确保它们仅作为普通字符被匹配。

### **函数语法**

php

```php
preg_quote(string $str, ?string $delimiter = null): string
```

- **参数**：
    - `$str`：需要转义的字符串。
    - `$delimiter`（可选）：正则表达式的分隔符（如 `/`、`#`、`~` 等），若指定，该分隔符也会被转义（避免与正则模式的分隔符冲突）。
- **返回值**：转义后的字符串。

### **转义的特殊字符**

`preg_quote()` 会对以下正则特殊字符前添加反斜杠 `\` 进行转义：

`. \ + * ? [ ^ ] $ ( ) { } = ! < > | : -`

`addslashes()` 是 PHP 中用于字符串转义的内置函数，主要用于在特殊字符前添加反斜杠 `\`，以防止这些字符被误解析为代码语法（如 SQL 语句、HTML 或命令中的特殊符号）。
`mysqli_real_escape_string($con, $str)`与`addslashes()`几乎相同
- `mysqli_multi_query()` 是 PHP 中 MySQLi 扩展的函数，用于**在一次数据库连接中执行多条 SQL 语句**（以分号 `;` 分隔）。
- `preg_match('/PHP/', $str)`检查$str中是否有PHP
- `json_decode`
- ```php
$jsonStr = '{"name":"张三","age":25,"hobbies":["编程","运动"]}';

// 默认返回对象
$obj = json_decode($jsonStr);
echo $obj->name; // 输出：张三

// 返回关联数组
$arr = json_decode($jsonStr, true);
echo $arr['age']; // 输出：25
echo $arr['hobbies'][0]; // 输出：编程
```
- `substr()`提取字符串的函数，如`substr($a,0,5);`表示从第一个字符串开始往后提取五个字符串`substr($a,-5)`表示从后开始提取五个字符串
- `strpos($file, '..')` 用于查找 `'..'` 在 `$file` 中首次出现的位置。如果找到，返回对应的索引值（整数）；如果没找到，返回 `false`。
- `=== false` 是严格比较，确保只有当 `strpos` 确实返回 `false`（即 `$file` 中完全没有 `'..'`）时，整个条件才为 `true`。
- `ord();`,输出ASCII编码，如`ord('A');`输出65
- `file_exists()`在 PHP 中，`file_exists()` 是一个用于检查文件或目录是否存在的函数。它接受一个文件路径作为参数，如果该路径对应的文件或目录存在，则返回 `true`，否则返回 `false`。
- `is_dir()`在 PHP 中，`is_dir()` 是一个用于判断指定路径是否为目录的函数。它接受一个路径参数，如果该路径存在且是一个目录，则返回 `true`，否则返回 `false`。
- `file_put_contents($filename, base64_decode($_GET['data']));`这段代码的功能是将通过 GET 参数`data`传递的 Base64 编码内容解码后，写入到`$filename`指定的文件中。
- `stripos( <content>, '<?')`：在文件内容中查找 `'<?'` 子串（PHP 开始标签），`stripos` 是大小写不敏感的查找，返回 **位置（0-based）** 或 `false`。
- `ereg()`其功能与 `preg_match` 类似，但存在一些关键差异。`ereg` 的模式**不需要分隔符**（如 `/`），直接写正则内容即可；而 `preg_match` 必须用分隔符包裹。`ereg` 仅支持**POSIX 基本正则表达式**，不支持 Perl 风格的扩展语法（如 `\d` 表示数字、`\w` 表示单词字符等），需用 `[0-9]` 代替 `\d`，`[a-zA-Z0-9_]` 代替 `\w`。`ereg` 默认大小写敏感，若需忽略大小写，需使用 `eregi`；而 `preg_match` 可通过修饰符 `i` 实现。
- `is_number()`检查变量是否为**数字**或**可被解释为数字的字符串**（包括整数、浮点数、科学计数法表示的字符串等）。如果是数字或数字字符串，返回 `true`；否则返回 `false`。用来过滤非数字，如`123a` ,如果对输入的字长有限制，可以用科学计数法压缩字长，如`1000000`可以写成`1e6`,长度压缩为三，但依然可以被`is_number()`识别为数字
- `ctype_space()` 是 PHP 中用于判断字符串是否仅包含**空白字符**的内置函数，检查输入字符串是否由空白字符组成，包括空格、制表符、换行符等，空字符串返回 `false`。全为空白字符返回 `true`，否则返回 `false`。若传入非字符串（如整数、数组），会先尝试转换为字符串。例如 `ctype_space(0)` 会转为字符串 `"0"`，返回 `false`。
- `trim()` 是 PHP 中用于移除字符串**首尾空白字符（或指定字符）** 的内置函数，支持自定义需要移除的字符集。如`trim($str, "#");`移除首尾的空白字符，默认不填的话，一处空白字符
- `call_user_func()` 是 PHP 中用于**动态调用函数或方法**的内置函数，支持以字符串形式指定函数名，或通过数组指定对象方法，如
```php
function add($a,$b){
	return $a+$b;
}
$result=call_user_func('add',2,3)
echo $result;//输出5
```
同时也可以调用匿名函数和类的静态方法
- `file_get_contents()` 函数，它是 PHP 中用于读取文件内容（或网络资源）的常用函数，能将整个文件内容一次性读入字符串,如
`$content = file_get_contents('./test.txt');`读取本地文件，`$html = file_get_contents('https://example.com');`读取网页内容（需开启 allow_url_fopen 配置）,
`$content = file_get_contents('./data.txt', false, null, 5, 10);`从第 5 个字节开始，读取 10 个字节（仅本地文件有效）,如果为负数，则是从后往前读
`shell_exec()` 是 PHP 中用于执行系统命令的函数，它会将命令的输出作为字符串返回
- `readfile();`输出文件的字节数，例如`echo readfile("text.php");`就会输出文件字节数143，可以用相对路径，可以用绝对路径。