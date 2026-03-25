XML指可扩展标记语言，把数据结构化后传输给对方，让对方在结构化的数据进行读取。XML是一种标记型语言.
所有的XML标签都必须闭合，并且XML的标签都是没有任何意义的，可以自定义。
XML可以不做换行，都写成一行。
标签大小写敏感。
标签必须正确闭合。
在XML里，有些函数具有特殊含义。
例如`<和>`是用来闭合标签的`<man>1<2</man>`这句xml是有错误的，不可以正常显示`1<2`
如果要在内容里出现`1<2`可以把<用实体表达，
- `&lt;`表示`<`
- `&gt;`表示`>`
- `&amp;`表示`&`
- `&apos;`表示`'`单引号
- `&quot;`表示`"`双引号
# 使用php解析XML
## 使用simplexml_load_file()读取XML文档
simplexml_load_file()函数可以把一个XML文档载入对象中。

## 使用DOMDocument读取XML文档
可以用
```php
<?php
$doc=new DOMDocument();
$doc->load("student.xml");
print_r($doc->saveXML());
```
也可以用这个
```php
<?php
$xml=include("student.xml");
$doc=new DOMDocument();
$doc->loadXML($xml);
print_r($doc->saveXML());
```

这样可以访问枝叶属性
```php
<?php
$doc=new DOMDocument();
$doc->load("student.xml");
//print_r($doc->saveXML());
echo $doc->getElementsByTagName("name")->item(1)->nodeValue;
```
可以访问name属性
如果嫌刚才的代码太长，可以写简洁一点
```php
<?php
$doc=new DOMDocument();
$doc->load("student.xml");
//print_r($doc->saveXML());
$doc2=simplexml_import_dom($doc);
echo $doc2->man[1]->name;
```
# DTD声明
DTD(文档类型定义)可定义合法的XML文档构建模块。它使用一系列合法的元素来定义文档的结构。DTD可被成行的声明于XML文档中，也可以作为一个外部引用。
!DOCTYPE定义约束
!ELEMENT定义元素
!ENTITY定义实体

### 内部的DOCTYPE声明
没有DTD时可以随意定义添加标签，XMLDOCTYPE用来约束XML规则，格式是
`<!DOCTYPE 根元素 [元素声明]>`
例如
```xml
<!DOCTYPE root[<!ELEMENT root (man)><!ELEMENT man(name,age)><!ELEMENT name (#PCDATA)>]>
```
表示root下有man,man下有name,age,并且定义name只可以是文本格式。
!ENTITY定义实体，这些实体可以在XML文档中被引用。
### 外部引用DTD文件
```xml
<!DOCTYPE root SYSTEM "1.dtd">
<root>
    <man>
        <name>程咬金</name>
        <age>99</age>
    </man>
    <man>
        <name>赵云</name>
        <age>78</age>
    </man>
</root>
```
1.dtd文件写
```dtd
<!ELEMENT root (man)>
<!ELEMENT man(name,age)>
<!ELEMENT name (#PCDATA)>
```
这样就实现从外部引用。
xxe漏洞出现的原因就是外部引用DTD文件，可以读取DTD文件，也可以读取其他的文件，造成文件读取漏洞。但还得用实体给文件内容展示出来。
# 实体
使用实体可以去除一些符号的特殊含义，例如在xml中`<>`是用来闭合标签的，如果实体化后可以让他丧失这个功能。
### 字符实体
- `&lt;`表示`<`
- `&gt;`表示`>`
- `&amp;`表示`&`
- `&apos;`表示`'`单引号
- `&quot;`表示`"`双引号

### 命名实体
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE author [
  <!ELEMENT author (#PCDATA)>
  <!ENTITY writer "benben">
  <!ENTITY copyright "&#169;">
]>
<author>&writer;&copyright;</author>
```
可以显示`<author>benben©</author>`，并且这个可以实现迭代调用
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE author [
  <!ELEMENT author (#PCDATA)>
  <!ENTITY writer "benben">
  <!ENTITY copyright "&writer;&#169;">
]>
<author>&copyright;</author>
```
实现的功能于上面一样。

### 外部实体

### 参数实体
目的是能够创建替换文本的可重用部分。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE author [
  <!ENTITY % area "name,street,pincode,city,phone">
  <!ELEMENT author (%area;)>
  <!ELEMENT copyright (%area;)>
]>
<author><name>666</name></author>
```






