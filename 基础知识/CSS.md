CSS 指层叠样式表 (**C**ascading **S**tyle **S**heets)，样式定义**如何显示** HTML 元素， 样式通常存储在**样式表**中，一个HTML文档可以显示不同的样式，样式解决了一个很大的问题，HTML 标签原本被设计为用于定义文档内容，如下实例：
<h1>这是一个标题</h1>
<p>这是一个段落。</p>
样式表定义如何显示 HTML 元素，就像 HTML 中的字体标签和颜色属性所起的作用那样。样式通常保存在外部的 .css 文件中。我们只需要编辑一个简单的 CSS 文档就可以改变所有页面的布局和外观。
CSS 必须依附于 HTML 存在，所以得先写一个html文件引入css文件,
```html
<!DOCTYPE html>
<html>
    <head>
        <meta charset="UTF-8">
        <title>我的CSS练习</title>
        <link rel="stylesheet" href="666.css">
    </head>
    <body>
        <div class="card">
            <h1>Hello CSS!!</h1>
            <p>这是我写的第一个样式卡片</p>
            <button>点击我</button>
        </div>
    </body>
</html>
```

```css
body{
    background-color:aliceblue;/*背景颜色*/
    font-family:'Franklin Gothic Medium', 'Arial Narrow', Arial, sans-serif;/*字体*/
    display:flex;/*弹性布局,轻松实现「居中、均分、垂直排列、自适应间距」等效果*/
    justify-content:center;/*水平居中*/
    align-items:center;/*垂直居中*/
    height:100vh;/*沾满屏幕高度*/
    margin:0;/*边距为0 清除浏览器默认的外边距（比如 body 自带的 8px 外边距）*/
}
/*设计卡片样式*/
.card{
    background-color:white;/*背景颜色*/
    width:300px;/*宽度*/
    padding:30px;/*内边距 （文字与边框的距离）*/
    border-radius:15px;/*圆角*/
    box-shadow:0 4px 15px rgba(0,0,0,0.1);/*阴影*/
    text-align:center;/*文字居中*/
}
/*标题颜色*/
h1{
    color:black;
}
/*按钮样式*/
button{
    background-color:deepskyblue;/*背景颜色*/;
    color:white;/*按钮文字颜色*/
    border:none;/*无边框*/
    padding:10px 20px;/*内边距(控制按钮大小)*/
    border-radius:5px;/*圆角*/
    cursor:pointer;/*鼠标悬浮时显示手型光标*/
    transition: background 0.3s;/*动画过渡*/
}
/*鼠标悬停在按钮上的效果*/
button:hover{
    background-color:darkcyan;/*给按钮添加「鼠标悬停时变色」的反馈效果，让用户明确感知 “这个按钮可点击”*/
}
```
这样可以实现一个卡片，效果是这样![](/image/123.png)
如果需要让某个元素固定在某个位置，比如右上角，可以用绝对定位
```css
position:absolute;
top:20px;/*距离顶部20px*/
right:20px;/*距离右侧20px*/
```
如果要给元素加边框，
```css
border: 1px solid #ccc;/*给元素添加**1 像素宽、实线样式、浅灰色**的边框*/
```
如果边框要是圆角
```css
border-radius: 8px;
```
8px是圆角半径大小