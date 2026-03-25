## 基本知识
JavaScript 是一种轻量级的编程语言。
JavaScript 是可插入 HTML 页面的编程代码。
JavaScript 插入 HTML 页面后，可由所有的现代浏览器执行。
HTML 中的 Javascript 脚本代码必须位于 `<script> 与 </script>` 标签之间。
Javascript 脚本代码可被放置在 HTML 页面的 `<body> 和 <head>` 部分中。
可以把js写在html代码里
```html
<!DOCTYPE html>
<html>
    <head>
        <title>Test Page</title>
    </head>
    <body>
        <script>
            document.write("<h1>this is a text!</h1>");
            document.write("<p>this is a paragraph!</p>");
        </script>
    </body>
</html>
```
这个会在页面加载时执行。也可以在某个事件发生时执行代码，例如用户点击按钮时。
```html
<!DOCTYPE html>
<html>
    <head>
        <script>
            function myFunction(){
                document.getElementById("demo").innerHTML ="这是我的第一个JavaScript函数";
            }
        </script>
    </head>
    <body>
        <h1>我的web页面</h1>
        <p id="demo">一个段落</p>
        <button type="button" onclick="myFunction()">点一下</button>
    </body>
</html>
```
点击按钮，自动执行函数。在界面中出现这句话。
也可以把脚本保存到外部文件里，如需使用要放在`<script>`标签里，例如`<script src="myscript.js"></script>`,外部脚本不包含`<script>`标签。
浏览器console(控制台)窗口可以执行js代码，例如输入`console.log('666')`会在控制台回显666。可以在浏览器的源码界面导入js文件。
javascript的注释是双斜杠//,多行注释以`/*`开始，以`*/`结束。
javascript语句是发给浏览器的命令，告诉浏览器要做的事情。
javascript代码由分号分割;
一般使用var关键字声明变量，例如`var a=1; var b="niu"`

## 输出
输出有几种方式
- `window.alert()`弹出警告框。
- `document.write()`将内容写到HTML文档里。
- `innerHTML`写入到HTML元素
- `console.log()`写入浏览器的控制台
例如
```html
<!DOCTYPE html>
<html>
    <head>
        <script>
            function myFunction(){
                window.alert("Hello World!");
            }
        </script>
    </head>
    <body>
        <h1>我的web页面</h1>
        <p id="demo">一个段落</p>
        <button type="button" onclick="myFunction()">点一下</button>
    </body>
</html>
```
就可以实现点击按钮，跳出弹窗。
document.ElementByld("demo")是使用id属性来查找HTML元素的代码。innerHTML="666"是用于修改元素的HTML的js代码
例如
```html
<!DOCTYPE html>
<html>
    <head>
        <script>
            function myFunction(){
                document.getElementById("demo").innerHTML = "你点击了按钮"
            }
        </script>
    </head>
    <body>
        <h1>我的web页面</h1>
        <p id="demo">一个段落</p>
        <button type="button" onclick="myFunction()">点一下</button>
    </body>
</html>
```
可以实现点击按钮，修改标题内容。
## 变量
一般使用var关键字声明变量，例如`var a=1; var b="niu"`
但也可以使用let关键字声明变量
const用于定义常量，即一旦赋值后，变量的值不能再修改。
例如
```js
let city="北京";var age=30;const name="666";console.log(city,name,age);
```
## 数据类型
变量的数据类型可以用typeof查看，例如
```js
console.log(typeof "666");console.log(typeof 666);
```
返回string number
## 对象
```js
var person={firstname:"John", lastname:"Doe", id:5566};
```
对象的寻址，`person.id`或`person['id']`  javaScript对象是属性和方法的容器。
对象方法可以这样创建
```js
methodName : function(){

	}
```
可以这样访问对象方法
```js
objectName.methodName()
```
## 实战
如果想写一个在页面显示当前时间的函数
```js
// 获取显示时间的DOM元素 
const timeElement = document.getElementById('currentTime'); 
// 格式化时间函数：补零（如 9 → 09） 
function formatNum(num) { return num < 10 ? `0${num}` : num; } 
// 获取并更新当前时间的函数 
function updateCurrentTime() { 
// 创建日期对象 
	const now = new Date(); 
	// 提取年、月、日、时、分、秒、星期 
	const year = now.getFullYear(); const month =formatNum(now.getMonth() + 1); 
	// 月份从0开始，需+1 
	const day = formatNum(now.getDate()); 
	const hour = formatNum(now.getHours()); 
	const minute = formatNum(now.getMinutes()); 
	const second = formatNum(now.getSeconds()); 
	const weekList = ['日', '一', '二', '三', '四', '五', '六']; 
	const week = weekList[now.getDay()]; 
	// getDay()返回0-6（0代表周日） 
	// 拼接时间字符串 
	const timeStr = `${year}年${month}月${day}日 星期${week} ${hour}:${minute}:${second}`; 
	// 更新DOM显示 
	timeElement.textContent = timeStr; } 
	// 初始化：立即执行一次，避免页面加载后延迟显示 
	updateCurrentTime(); // 每秒（1000毫秒）更新一次时间 
	setInterval(updateCurrentTime, 1000);
```
详细解释。
- `const timeElement = document.getElementById('currentTime');`找到页面中 `id="currentTime"` 的元素（比如 `<div id="currentTime"></div>`），并把这个元素对象赋值给变量 `timeElement`。
- **核心知识点**：`document.getElementById('XXX');`是DOM操作的核心方法，根据元素id精确获取页面元素
```js
function formatNum(num){
    return num<10 ? `0${num}`:num
}
```
时间格式规范函数，如果时间小于10，自动在前面补0，如09,使显示更美观。
这里用到了三元运算符`条件?满足时执行:不满足时执行` 在这里反引号包裹`0${num}`是ES6模板字符串，比传统的`'0'+num`更直观，
接下来解释核心函数updateCurrentTime
- `const now = new Date();` `new Date()` 会创建一个「当前系统时间」的 Date 实例，包含年、月、日、时、分、秒、星期等所有时间信息。
- 下面的函数都是提取系统当前时间，并用formatNum函数处理，再用ES6模板拼接在一起
- `timeElement.textContent = timeStr;` 把拼接好的时间字符串写入之前获取的 `timeElement` 元素中，页面就会显示这个时间。
- 这里用的是textContent,获取元素的纯文本内容，因为innerHTML会解析HTML标签，这里的时间字符串也没有html标签。
- `updateCurrentTime();`页面加载后，立即执行一次 `updateCurrentTime()` 函数。如果只靠下面的 `setInterval`，页面加载后会等待 1 秒才第一次显示时间，导致前 1 秒内 `currentTime` 元素是空的，体验差。
- `setInterval(updateCurrentTime, 1000);` 周期性执行函数，每隔一秒（1000毫秒）执行依次函数

## 