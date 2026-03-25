```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<title>我的html练习</title>
</head>
<body>
<h1>我的第一个标题</h1>
<p>我的第一个段落</p>
</body>
</html>
```
- `<!DOCTYPE html>` 声明为 HTML5 文档
- `<html>` 元素是 HTML 页面的根元素
- `<head>` 元素包含了文档的元（meta）数据，如 `<meta charset="utf-8">` 定义网页编码格式为 **utf-8**。
- `<title>` 元素描述了文档的标题
- `<body> `元素包含了可见的页面内容
- `<h1>` 元素定义一个大标题
- `<p>` 元素定义一个段落
- 只有 `<body>` 区域 才会在浏览器中显示。
- 目前在大部分浏览器中，直接输出中文会出现中文乱码的情况，这时候我们就需要在头部将字符声明为 UTF-8 或 GBK。
id：为元素指定唯一的标识符。
<div id="header">This is the header</div>
class：为元素指定一个或多个类名，用于 CSS 或 JavaScript 选择。
<p class="text highlight">This is a highlighted text.</p>
style：用于直接在元素上应用 CSS 样式。
<p style="color: blue; font-size: 14px;">This is a styled paragraph.</p>
title：为元素提供额外的提示信息，通常在鼠标悬停时显示。
<abbr title="HyperText Markup Language">HTML</abbr>
data-*：用于存储自定义数据，通常通过 JavaScript 访问。
<div data-user-id="12345">User Info</div>
某些属性仅适用于特定的 HTML 元素。

**`href`**（用于 `<a>` 和 `<link>` 元素）：指定链接的目标 URL。

<a href="https://www.example.com">Visit Example</a>

**`src`**（用于 `<img>`, `<script>`, `<iframe>` 等元素）：指定外部资源的 URL。

<img src="image.jpg" alt="An example image">

alt（用于 `<img>` 元素）：为图像提供替代文本，当图像无法显示时显示。

<img src="image.jpg" alt="An example image">

**`type`**（用于 `<input>` 和 `<button>` 元素）：指定输入控件的类型。

<input type="text" placeholder="Enter your name">

**`value`**（用于 `<input>`, `<button>`, `<option>` 等元素）：指定元素的初始值。

<input type="text" value="Default Value">

disabled（用于表单元素）：禁用元素，使其不可交互。

<input type="text" disabled>

**`checked`**（用于 `<input type="checkbox">` 和 `<input type="radio">`）：指定复选框或单选按钮是否被选中。

<input type="checkbox" checked>

**`placeholder`**（用于 `<input>` 和 `<textarea>` 元素）：在输入框中显示提示文本。

<input type="text" placeholder="Enter your email">

**`target`**（用于 `<a>` 和 `<form>` 元素）：指定链接或表单提交的目标窗口或框架。
`target="_blank"`：
- 控制链接的打开方式，`_blank` 表示 “在新的浏览器标签页中打开链接”。
<a href="https://www.example.com" target="_blank">Open in new tab</a>
布尔属性是指不需要值的属性，它们的存在即表示 true，不存在则表示 false。

disabled：禁用元素。

<input type="text" disabled>

readonly：使输入框只读。

<input type="text" readonly>

required：指定输入字段为必填项。

<input type="text" required>

**`autoplay`**（用于 `<audio>` 和 `<video>` 元素）：自动播放媒体。

<video src="video.mp4" autoplay></video>


