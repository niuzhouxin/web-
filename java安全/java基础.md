java语言的特点就是面向对象(OOP)，而面向对象编程的三大特点就是封装，继承，多态。
## 数据类型
java的数据类型和其他语言类似，只不过还是有些特殊的地方
### 字符
表示一个符号，用`char`关键字声明，例如
```java
char c1 = 'A';
char c2 = '中';
```
char类型用单引号包裹。
### 字符串
表示字符，长度不限。用`String`关键字声明，例如
```java
String s = ""; // 空字符串，包含0个字符
String s1 = "A"; // 包含一个字符
String s2 = "ABC"; // 包含3个字符
String s3 = "中文 ABC"; // 包含6个字符，其中有一个空格
```
`String`要用双引号包裹。
### 数组
定义数组
```java
int[] ns = new int[5];
```
或者直接带着值定义，长度自动计算
```java
int[] ns = new int[] { 68, 79, 91, 85, 62 };
```
还可以进一步简写为
```java
int[] ns = { 68, 79, 91, 85, 62 };
```
### var关键字
有些时候，类型的名字太长，写起来比较麻烦。例如：
```java
StringBuilder sb = new StringBuilder();
```
这个时候，如果想省略变量类型，可以使用`var`关键字：
```java
var sb = new StringBuilder();
```
编译器会根据赋值语句自动推断出变量`sb`的类型是`StringBuilder`。对编译器来说，语句：
```java
var sb = new StringBuilder();
```
实际上会自动变成：
```java
StringBuilder sb = new StringBuilder();
```
因此，使用`var`定义变量，仅仅是少写了变量类型而已。
Java标准库提供了`Arrays.toString()`，可以快速打印数组内容：
```java
public class Main {  
    public static void main(String[] args) {  
        int[] ns = { 1, 1, 2, 3, 5, 8 };  
        System.out.println(Arrays.toString(ns));  
    }  
}
```
### for-each循环
如果想打印数组内容，不仅可以用`Arrays.toString`，还可以用`for-each`循环，
```java
public class Main {  
    public static void main(String[] args) {  
        int[] ns = { 1, 1, 2, 3, 5, 8 };  
        for(int n:ns){  
            System.out.println(n);  
        }  
    }  
}
```
## 设计模式
### 单例
保证一个类仅有一个实例，并提供一个访问它的全局访问点。
单例模式（Singleton）的目的是为了保证在一个进程中，某个类有且仅有一个实例。
因为这个类只有一个实例，因此就不可以用`new xyz()`来创建实例了，所以单例的构造方法是`private`,这样就防止了调用自己创建的实例，但是在类的内部，是可以用一个静态字段来引用唯一创建的实例的
## 方法重载
在一个类中，我们可以定义多个方法。如果有一系列方法，它们的功能都是类似的，只有参数有所不同，那么，可以把这一组方法名做成_同名_方法。例如，在`Hello`类中，定义多个`hello()`方法：
```java
class Hello {
    public void hello() {
        System.out.println("Hello, world!");
    }

    public void hello(String name) {
        System.out.println("Hello, " + name + "!");
    }

    public void hello(String name, int age) {
        if (age < 18) {
            System.out.println("Hi, " + name + "!");
        } else {
            System.out.println("Hello, " + name + "!");
        }
    }
}
```
这些方法名相同，但各自的参数不同，称为方法重载(`overload`)。
注：方法重载的返回值类型通常都是相同的
方法重载的目的是，功能类似的方法使用同一名字，更容易记住。
举个例子，`String`类提供了多个重载方法`indexOf()`，可以查找子串：
- `int indexOf(int ch)`：根据字符的Unicode码查找；
- `int indexOf(String str)`：根据字符串查找；
- `int indexOf(int ch, int fromIndex)`：根据字符查找，但指定起始位置；
- `int indexOf(String str, int fromIndex)`根据字符串查找，但指定起始位置。
例如
```java
public class Main {  
    public static void main(String[] args) throws Exception {  
        String s = "Test String";  
        int n1 = s.indexOf("t");  
        int n2 = s.indexOf("st");  
        int n3 = s.indexOf("st",4);  
        System.out.println(n1);  
        System.out.println(n2);  
        System.out.println(n3);  
    }  
}
```
输出
```
3
2
-1
```

## public,private和protected
- public的字段和方法访问没有限制。
- private的字段和方法只可以在类的内部访问，子类不可以访问。嵌套类可以访问。
- protected的字段和方法只可以在类的内部访问，子类可以访问，同一个包内其他类即使不是子类，也可以访问。
## 构造方法
和普通方法相比，构造方法没有返回值，也没有(void)，调用构造方法必须用`new`操作符。
例如
```java
// 构造方法
public class Main {
    public static void main(String[] args) {
        Person p = new Person("Xiao Ming", 15);
        System.out.println(p.getName());
        System.out.println(p.getAge());
    }
}

class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() {
        return this.name;
    }

    public int getAge() {
        return this.age;
    }
}
```
构造方法很特殊，所以构造方法的名称就是类名。即使不去定义，每个类都会有一个默认的构造方法，内容是空的。
## 继承
Java使用`extends`关键字来实现继承：
```java
class Person {
    private String name;
    private int age;

    public String getName() {...}
    public void setName(String name) {...}
    public int getAge() {...}
    public void setAge(int age) {...}
}

class Student extends Person {
    // 不要重复name和age字段/方法,
    // 只需要定义新增score字段/方法:
    private int score;

    public int getScore() { … }
    public void setScore(int score) { … }
}
```
`super`关键字表示父类（超类）。子类引用父类的字段时，可以用`super.fieldName`
但有一个例子
```java
// super
public class Main {
    public static void main(String[] args) {
        Student s = new Student("Xiao Ming", 12, 89);
    }
}

class Person {
    protected String name;
    protected int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

class Student extends Person {
    protected int score;

    public Student(String name, int age, int score) {
        this.score = score;
    }
}
```
直接编译会报错
大意是在`Student`的构造方法中，无法调用`Person`的构造方法。

这是因为在Java中，任何`class`的构造方法，第一行语句必须是调用父类的构造方法。如果没有明确地调用父类的构造方法，编译器会帮我们自动加一句`super();`，所以，`Student`类的构造方法实际上是这样：
```java
class Student extends Person {
    protected int score;

    public Student(String name, int age, int score) {
        super(); // 自动调用父类的构造方法
        this.score = score;
    }
}
```
但是因为Person类的构造方法是有参的，因此，编译失败。解决方法是调用`Person`类存在的某个构造方法。例如
```java
class Student extends Person {
    protected int score;

    public Student(String name, int age, int score) {
        super(name, age); // 调用父类的构造方法Person(String, int)
        this.score = score;
    }
}
```
这样就可以编译成功。
因此我们得出结论：如果父类没有默认的构造方法，子类就必须显式调用`super()`并给出参数以便让编译器定位到父类的一个合适的构造方法。
## 多态
在继承关系中，子类如果定义了一个与父类方法签名完全相同的方法，被称为覆写（Override）。
例如
```java
class Person {
    public void run() {
        System.out.println("Person.run");
    }
}
```
在子类student里覆写这个方法
```java
class Student extends Person {
    @Override
    public void run() {
        System.out.println("Student.run");
    }
}
```
加上`@Override`可以让编译器帮助检查是否进行了正确的覆写。希望进行覆写，但是不小心写错了方法签名，编译器会报错。但是`@Override`不是必需的。
```java
class Person {
    public void run() { … }
}

class Student extends Person {
    // 不是Override，因为参数不同:
    public void run(String s) { … }
    // 不是Override，因为返回值不同:
    public int run() { … }
}
```
Java的实例方法调用是基于运行时的实际类型的动态调用，而非变量的声明类型。
这个非常重要的特性在面向对象编程中称之为多态。
例如
```java
public class Main {  
    public static void main(String[] args) {  
        Person p = new Student();  
        p.run(); // 应该打印Person.run还是Student.run?  
    }  
}  
  
class Person {  
    public void run() {  
        System.out.println("Person.run");  
    }  
}  
  
class Student extends Person {  
    @Override  
    public void run() {  
        System.out.println("Student.run");  
    }  
}
```
这里运行输出`Student.run` 虽然`Person p = new Student();`变量p被声明为`Person`类型，但实际上实例化的是`Student`类，又因为java的多态性质，是基于运行时的实际类型的动态调用，而非变量的声明类型。所以就是输出`Student.run`了·。
在子类的覆写方法中，如果要调用父类的被覆写的方法，可以通过`super`来调用。例如：
```java
class Person {
    protected String name;
    public String hello() {
        return "Hello, " + name;
    }
}

class Student extends Person {
    @Override
    public String hello() {
        // 调用父类的hello()方法:
        return super.hello() + "!";
    }
}
```
**final**
继承可以允许子类覆写父类的方法。如果一个父类不允许子类对它的某个方法进行覆写，可以把该方法标记为`final`。用`final`修饰的方法不能被`Override`：
```java
class Person {
    protected String name;
    public final String hello() {
        return "Hello, " + name;
    }
}

class Student extends Person {
    // compile error: 不允许覆写
    @Override
    public String hello() {
    }
}
```
如果一个类不希望任何其他类继承自它，那么可以把这个类本身标记为`final`。用`final`修饰的类不能被继承：
```java
final class Person {
    protected String name;
}

// compile error: 不允许继承自Person
class Student extends Person {
}
```
对于一个类的实例字段，同样可以用`final`修饰。用`final`修饰的字段在初始化后不能被修改。例如：
```java
public class Main {  
    public static void main(String[] args) {  
        Person p = new Person();  
        p.name = "New Name";  
    }  
}  
  
  
class Person {  
    public final String name = "Unamed";  
}
```
报错`java: 无法为 final 变量 name 分配值`
所以`final`修饰符有多种作用：
- `final`修饰的方法可以阻止被覆写；
- `final`修饰的class可以阻止被继承；
- `final`修饰的field必须在创建对象时初始化，随后不可修改。
## 抽象类
由于多态的存在，每个子类都可以覆写父类的方法，如果父类的方法本身不需要实现任何功能，仅仅是为了定义方法签名，目的是让子类去覆写它，那么，可以把父类的方法声明为抽象方法：
```java
class Person {
    public abstract void run();
}
```
把一个方法声明为`abstract`，表示它是一个抽象方法，本身没有实现任何方法语句。因为这个抽象方法本身是无法执行的，所以，`Person`类也无法被实例化。编译器会告诉我们，无法编译`Person`类，因为它包含抽象方法。
```java
public class Main {  
    public static void main(String[] args) {  
        new Person();  
    }  
}  
class Person {  
    public abstract void run();  
}
```
编译报错`java: Person不是抽象的, 并且未覆盖Person中的抽象方法run()`
必须把`Person`类本身也声明为`abstract`，才能正确编译它：
```java
abstract class Person {
    public abstract void run();
}
```
如果一个`class`定义了方法，但没有具体执行代码，这个方法就是抽象方法，抽象方法用`abstract`修饰。
因为无法执行抽象方法，因此这个类也必须申明为抽象类（abstract class）。
使用`abstract`修饰的类就是抽象类。我们无法实例化一个抽象类：
```java
Person p = new Person(); // 编译错误
```
因为抽象类本身被设计成只能用于被继承，因此，抽象类可以强迫子类实现其定义的抽象方法，否则编译会报错。因此，抽象方法实际上相当于定义了“规范”。
例如，`Person`类定义了抽象方法`run()`，那么，在实现子类`Student`的时候，就必须覆写`run()`方法：
```java
public class Main {  
    public static void main(String[] args) {  
        Person p = new Student();  
        p.run();  
    }  
}  
abstract class Person {  
    public abstract void run();  
}  
  
class Student extends Person{  
    @Override  
    public void run(){  
        System.out.println("666");  
    }  
}
```
## 接口
在抽象类中，抽象方法本质上是定义接口规范：即规定高层类的接口，从而保证所有子类都有相同的接口实现，这样，多态就能发挥出威力。
如果一个抽象类没有字段，所有方法全部都是抽象方法：
```java
abstract class Person {
    public abstract void run();
    public abstract String getName();
}
```
就可以把该抽象类改写为接口：`interface`。
在Java中，使用`interface`可以声明一个接口：
```java
interface Person {
    void run();
    String getName();
}
```
所谓`interface`，就是比抽象类还要抽象的纯抽象接口，因为它连字段都不能有。因为接口定义的所有方法默认都是`public abstract`的，所以这两个修饰符不需要写出来（写不写效果都一样）。
当一个具体的`class`去实现一个`interface`时，需要使用`implements`关键字。举个例子：
```java
class Student implements Person {
    private String name;

    public Student(String name) {
        this.name = name;
    }

    @Override
    public void run() {
        System.out.println(this.name + " run");
    }

    @Override
    public String getName() {
        return this.name;
    }
}
```
我们知道，在Java中，一个类只能继承自另一个类，不能从多个类继承。但是，一个类可以实现多个`interface`，例如：
```java
class Student implements Person, Hello { // 实现了两个interface
    ...
}
```
一个`interface`可以继承自另一个`interface`。`interface`继承自`interface`使用`extends`，它相当于扩展了接口的方法。例如：
个`interface`可以继承自另一个`interface`。`interface`继承自`interface`使用`extends`，它相当于扩展了接口的方法。例如：
```java
interface Hello {
    void hello();
}

interface Person extends Hello {
    void run();
    String getName();
}
```
此时，`Person`接口继承自`Hello`接口，因此，`Person`接口现在实际上有3个抽象方法签名，其中一个来自继承的`Hello`接口。
## 静态字段和静态方法
**静态字段**
在一个`class`中定义的字段，我们称之为实例字段。实例字段的特点是，每个实例都有独立的字段，各个实例的同名字段互不影响。
还有一种字段，是用`static`修饰的字段，称为静态字段：`static field`。
实例字段在每个实例中都有自己的一个独立“空间”，但是静态字段只有一个共享“空间”，所有实例都会共享该字段。举个例子：
```java
public class Main {  
    public static void main(String[] args) {  
        Person ming = new Person("Xiao Ming", 12);  
        Person hong = new Person("Xiao Hong", 15);  
        ming.number = 88;  
        System.out.println(hong.number);  
        hong.number = 99;  
        System.out.println(ming.number);  
        System.out.println(hong.number);  
    }  
}  
  
class Person {  
    public String name;  
    public int age;  
  
    public static int number;  
  
    public Person(String name, int age) {  
        this.name = name;  
        this.age = age;  
    }  
}
```
会输出
```
88
99
99
```
对于静态字段，无论修改哪个实例的静态字段，效果都是一样的：所有实例的静态字段都被修改了，原因是静态字段并不属于实例：所有实例共享一个静态字段。
推荐用类名来访问静态字段。可以把静态字段理解为描述`class`本身的字段。对于上面的代码，更好的写法是：
```java
Person.number = 99;
System.out.println(Person.number);
```
**静态方法**
调用实例方法必须通过一个实例变量，而调用静态方法则不需要实例变量，通过类名就可以调用。静态方法类似其它编程语言的函数。例如：
```java
// static method
public class Main {
    public static void main(String[] args) {
        Person.setNumber(99);
        System.out.println(Person.number);
    }
}

class Person {
    public static int number;

    public static void setNumber(int value) {
        number = value;
    }
}
```
注意到Java程序的入口`main()`也是静态方法。
## 包
Java定义了一种名字空间，称之为包：`package`。一个类总是属于某个包，类名（比如`Person`）只是一个简写，真正的完整类名是`包名.类名`。
例如：
小明的`Person`类存放在包`ming`下面，因此，完整类名是`ming.Person`；
小红的`Person`类存放在包`hong`下面，因此，完整类名是`hong.Person`；
小军的`Arrays`类存放在包`mr.jun`下面，因此，完整类名是`mr.jun.Arrays`；
JDK的`Arrays`类存放在包`java.util`下面，因此，完整类名是`java.util.Arrays`。
在定义`class`的时候，我们需要在第一行声明这个`class`属于哪个包。
小明的`Person.java`文件：
```java
package ming; // 申明包名ming

public class Person {
}
```
小军的`Arrays.java`文件：
```java
package mr.jun; // 申明包名mr.jun

public class Arrays {
}
```
在Java虚拟机执行的时候，JVM只看完整类名，因此，只要包名不同，类就不同。
包可以是多层结构，用`.`隔开。例如：`java.util`。
**import**
在一个`class`中，我们总会引用其他的`class`。例如，小明的`ming.Person`类，如果要引用小军的`mr.jun.Arrays`类，他有三种写法：
第一种，直接写出完整类名，例如：
```java
// Person.java
package ming;

public class Person {
    public void run() {
        // 写完整类名: mr.jun.Arrays
        mr.jun.Arrays arrays = new mr.jun.Arrays();
    }
}
```
第二种写法是用`import`语句，导入小军的`Arrays`，然后写简单类名：
```java
// Person.java
package ming;

// 导入完整类名:
import mr.jun.Arrays;

public class Person {
    public void run() {
        // 写简单类名: Arrays
        Arrays arrays = new Arrays();
    }
}
```
在写`import`的时候，可以使用`*`，表示把这个包下面的所有`class`都导入进来（但不包括子包的`class`）：
```java
// Person.java
package ming;

// 导入mr.jun包的所有class:
import mr.jun.*;

public class Person {
    public void run() {
        Arrays arrays = new Arrays();
    }
}
```
因此，编写class的时候，编译器会自动帮我们做两个import动作：
- 默认自动`import`当前`package`的其他`class`；
- 默认自动`import java.lang.*`。
**编译和运行**
假设我们创建了如下目录结构
```
work
├── bin
└── src
    └── com
        └── itranswarp
            ├── sample
            │   └── Main.java
            └── world
                └── Person.java
```
其中，`bin`目录用于存放编译后的`class`文件，`src`目录按包结构存放Java源码，我们怎么一次性编译这些Java源码呢？
首先，确保当前目录是`work`目录，即存放`src`和`bin`的父目录：
```
C:\Users\Lenovo\Desktop\java>dir
 Volume in drive C is Windows-SSD
 Volume Serial Number is E4BE-48CB

 Directory of C:\Users\Lenovo\Desktop\java

2026/03/06  20:00    <DIR>          .
2026/03/05  19:38    <DIR>          ..
2026/03/06  19:59    <DIR>          bin
2026/03/06  20:00    <DIR>          src
               0 File(s)              0 bytes
               4 Dir(s)  64,928,382,976 bytes free
```
然后，编译`src`目录下的所有Java文件：
```
javac -d ./bin src/**/*.java
```
命令行`-d`指定输出的`class`文件存放`bin`目录，后面的参数`src/**/*.java`表示`src`目录下的所有`.java`文件，包括任意深度的子目录。
注意：Windows不支持`**`这种搜索全部子目录的做法，所以在Windows下编译必须依次列出所有`.java`文件：
```
javac -d bin src\com\itranswarp\sample\Main.java src\com\itranswarp\world\Person.java
```
使用Windows的PowerShell可以利用`Get-ChildItem`来列出指定目录下的所有`.java`文件：
```
PS C:\Users\Lenovo\Desktop\java> (Get-ChildItem -Path .\src -Recurse -Filter *.java).FullName
C:\Users\Lenovo\Desktop\java\src\com\itranswarp\sample\Main.java
C:\Users\Lenovo\Desktop\java\src\com\itranswarp\world\Person.java
```
因此编译命令可以写成
```
javac -d .\bin (Get-ChildItem -Path .\src -Recurse -Filter *.java).FullName
```
这个指令值适配PowerShell，不适配cmd
如果没有语法错误，就会在bin目录下发现对应的class文件。
注：在java代码里一定要写pakage声明，不然编译时无法生成bin目录下对应的目录结构。
## 作用域
在Java中，我们经常看到`public`、`protected`、`private`这些修饰符。在Java中，这些修饰符可以用来限定访问作用域。
#### public
定义为`public`的`class`、`interface`可以被其他任何类访问：
定义为`public`的`field`、`method`可以被其他类访问，前提是首先有访问`class`的权限：
上面的`hi()`方法是`public`，可以被其他类调用，前提是首先要能访问`Hello`类：
#### private
定义为`private`的`field`、`method`无法被其他类访问：
实际上，确切地说，`private`访问权限被限定在`class`的内部，而且与方法声明顺序_无关_
由于Java支持嵌套类，如果一个类内部还定义了嵌套类，那么，嵌套类拥有访问`private`的权限：
#### protected
`protected`作用于继承关系。定义为`protected`的字段和方法可以被子类访问，以及子类的子类：
上面的`protected`方法可以被继承的类访问：
#### package
最后，包作用域是指一个类允许访问同一个`package`的没有`public`、`private`修饰的`class`，以及没有`public`、`protected`、`private`修饰的字段和方法。
只要在同一个包，就可以访问`package`权限的`class`、`field`和`method`
注意，包名必须完全一致，包没有父子关系，`com.apache`和`com.apache.abc`是不同的包。
## 内部类
没看，用到再说
## classpath
`classpath`是JVM用到的一个环境变量，它用来指示JVM如何搜索`class`。
因为Java是编译型语言，源码文件是`.java`，而编译后的`.class`文件才是真正可以被JVM执行的字节码。因此，JVM需要知道，如果要加载一个`abc.xyz.Hello`的类，应该去哪搜索对应的`Hello.class`文件。
所以，`classpath`就是一组目录的集合，它设置的搜索路径与操作系统相关。例如，在Windows系统上，用`;`分隔，带空格的目录用`""`括起来，可能长这样：
```
C:\work\project1\bin;C:\shared;"D:\My Documents\project1\bin"
```
在Linux系统上，用`:`分隔，可能长这样：
```
/usr/shared:/usr/local/bin:/home/liaoxuefeng/bin
```
现在我们假设`classpath`是`.;C:\work\project1\bin;C:\shared`，当JVM在加载`abc.xyz.Hello`这个类时，会依次查找：
- <当前目录>\abc\xyz\Hello.class
- C:\work\project1\bin\abc\xyz\Hello.class
- C:\shared\abc\xyz\Hello.class
注意到`.`代表当前目录。如果JVM在某个路径下找到了对应的`class`文件，就不再往后继续搜索。如果所有路径下都没有找到，就报错。
classpath设定方式有两种
- 在系统环境变量中设置`classpath`环境变量，不推荐；会污染整个系统环境
- 在启动JVM时设置`classpath`变量，推荐。
实际上就是给`java`命令传入`-classpath`参数：
```
java -classpath .;C:\work\project1\bin;C:\shared abc.xyz.Hello
```
或者使用简写
```
java -cp .;C:\work\project1\bin;C:\shared abc.xyz.Hello
```
如果没有`-cp`，那么JVM默认就是`.`，当前目录。
在IDE中运行Java程序，IDE自动传入的`-cp`参数是当前工程的`bin`目录和引入的jar包。
更好的做法是，不要设置`classpath`！默认的当前目录`.`对于绝大多数情况都够用了。
## jar包
如果有很多`.class`文件，散落在各层目录中，肯定不便于管理。如果能把目录打一个包，变成一个文件，就方便多了。
jar包就是用来干这个事的，它可以把`package`组织的目录层级，以及各个目录下的所有文件（包括`.class`文件和其他文件）都打成一个jar文件，这样一来，无论是备份，还是发给客户，就简单多了。
jar包实际上就是一个zip格式的压缩文件，而jar包相当于目录。如果我们要执行一个jar包的`class`，就可以把jar包放到`classpath`中：
```plain
java -cp ./hello.jar abc.xyz.Hello
```
这样JVM会自动在`hello.jar`文件里去搜索某个类。
创建jar包方法：因为jar包就是zip包，所以，直接在资源管理器中，找到正确的目录，点击右键，在弹出的快捷菜单中选择“发送到”，“压缩(zipped)文件夹”，就制作了一个zip文件。然后，把后缀从`.zip`改为`.jar`，一个jar包就创建成功。但是这样的话目录结构有点问题，没成功。可以直接用指令打包
```
jar -cvf java.jar -C bin .
```
- `-c`是创建一个新的包。
- `-v`打印详细过程
- `-f java.jar`指定输出的文件名是`java.jar`
- `-C bin`切换到`bin`目录下进行打包。
- `.`打包当前目录。
如果我们要执行一个jar包的`class`，就可以把jar包放到`classpath`中
```
java -cp ./java.jar com.itranswarp.sample.Main
```
这样就可以执行`Main.class`。
jar包还可以包含一个特殊的`/META-INF/MANIFEST.MF`文件，`MANIFEST.MF`是纯文本，可以指定`Main-Class`和其它信息。JVM会自动读取这个`MANIFEST.MF`文件，如果存在`Main-Class`，我们就不必在命令行指定启动的类名，而是用更方便的命令：
```
java -jar hello.jar
```
在大型项目中，不可能手动编写`MANIFEST.MF`文件，再手动创建jar包。Java社区提供了大量的开源构建工具，例如[Maven](https://liaoxuefeng.com/books/java/maven/index.html)，可以非常方便地创建jar包。
## class版本
我们通常说的Java 8，Java 11，Java 17，是指JDK的版本，也就是JVM的版本，更确切地说，就是`java.exe`这个程序的版本：
可以通过
```
java -version
```
高版本的JDK可编译输出低版本兼容的class文件，但需注意，低版本的JDK可能不存在高版本JDK添加的类和方法，导致运行时报错。
```
javap -v bin\com\itranswarp\sample\Main.class
```
可以看到
```
Classfile /C:/Users/Lenovo/Desktop/java/bin/com/itranswarp/sample/Main.class
  Last modified 2026-3-6; size 429 bytes
  MD5 checksum d8730b89f71b6b1afb7bb28226cf94f3
  Compiled from "Main.java"
public class com.itranswarp.sample.Main
  minor version: 0
  major version: 69
```
可以看到`major version`是69
## 模块
这些`.jmod`文件每一个都是一个模块，模块名就是文件名。例如：模块`java.base`对应的文件就是`java.base.jmod`。模块之间的依赖关系已经被写入到模块内的`module-info.class`文件了。所有的模块都直接或间接地依赖`java.base`模块，只有`java.base`模块不依赖任何模块，它可以被看作是“根模块”，好比所有的类都是从`Object`直接或间接继承而来。

把一堆class封装为jar仅仅是一个打包的过程，而把一堆class封装为模块则不但需要打包，还需要写入依赖关系，并且还可以包含二进制代码（通常是JNI扩展）。此外，模块支持多版本，即在同一个模块中可以为不同的JVM提供不同的版本。
**编写模块**
首先，创建模块和原有的创建Java项目是完全一样的，以`oop-module`工程为例，它的目录结构如下
```
oop-module
├── bin
├── build.sh
└── src
    ├── com
    │   └── itranswarp
    │       └── sample
    │           ├── Greeting.java
    │           └── Main.java
    └── module-info.java
```
其中，`bin`目录存放编译后的class文件，`src`目录存放源码，按包名的目录结构存放，仅仅在`src`目录下多了一个`module-info.java`这个文件，这就是模块的描述文件。在这个模块中，它长这样：
```java
module hello.world {
	requires java.base; // 可不写，任何模块都会自动引入java.base
	requires java.xml;
}
```
其中，`module`是关键字，后面的`hello.world`是模块的名称，它的命名规范与包一致。花括号的`requires xxx;`表示这个模块需要引用的其他模块名。除了`java.base`可以被自动引入外，这里我们引入了一个`java.xml`的模块。
当我们使用模块声明了依赖关系后，才能使用引入的模块。例如，`Main.java`代码如下：
```java
package com.itranswarp.sample;

// 必须引入java.xml模块后才能使用其中的类:
import javax.xml.XMLConstants;

public class Main {
	public static void main(String[] args) {
		Greeting g = new Greeting();
		System.out.println(g.hello(XMLConstants.XML_NS_PREFIX));
	}
}
```
如果把`requires java.xml;`从`module-info.java`中去掉，编译将报错。可见，模块的重要作用就是声明依赖关系。
接下来编译
```
javac -d bin src\com\itranswarp\sample\Greeting.java src\com\itranswarp\sample\Main.java
```
如果编译成功，现在项目结构如下：

```
oop-module
├── bin
│   ├── com
│   │   └── itranswarp
│   │       └── sample
│   │           ├── Greeting.class
│   │           └── Main.class
│   └── module-info.class
└── src
    ├── com
    │   └── itranswarp
    │       └── sample
    │           ├── Greeting.java
    │           └── Main.java
    └── module-info.java
```
下一步，我们需要把bin目录下的所有class文件先打包成jar，在打包的时候，注意传入`--main-class`参数，让这个jar包能自己定位`main`方法所在的类
```plain
jar --create --file hello.jar --main-class com.itranswarp.sample.Main -C bin .
```
所以，继续使用JDK自带的`jmod`命令把一个jar包转换成模块：
```
$ jmod create --class-path hello.jar hello.jmod
```
## java核心类
#### String
在Java中，`String`是一个引用类型，它本身也是一个`class`。
```java
String s1 = "Hello";
```
实际上字符串在`String`内部是通过一个`char[]`数组表示的，因此，按下面的写法也是可以的：
```java
String s2 = new String(new char[] {'h', 'e', 'l', 'l', 'o'});
```
**字符串比较**
当我们想要比较两个字符串是否相同时，要特别注意，我们实际上是想比较字符串的内容是否相同。必须使用`equals()`方法而不能用`==`
例如
```java
public class Main {  
    public static void main(String[] args) {  
        String s1 = "Hello";  
        String s2 = "Hello";  
        System.out.println(s1 == s2);  
        System.out.println(s1.equals(s2));  
    }  
}
```
虽然这样不会报错，返回两个true，貌似可以使用`==`来比较字符串。
但实际上那只是Java编译器在编译期，会自动把所有相同的字符串当作一个对象放入常量池，自然`s1`和`s2`的引用就是相同的。
所以，这种`==`比较返回`true`纯属巧合。换一种写法，`==`比较就会失败：
```java
public class Main {  
    public static void main(String[] args) {  
        String s1 = "hello";  
        String s2 = "HELLO".toLowerCase();  
        System.out.println(s1 == s2);  
        System.out.println(s1.equals(s2));  
    }  
}
```
这样就会一个`false`一个`true`了，但是这明显是不对的。所以字符串比较必须用`equals`方法。
## StringBuilder
参考代码
```java
public class Main {  
    public static void main(String[] args) {  
        String s = "";  
        for(int i = 0;i<1000;i++){  
            s =s +","+i;  
        }  
        System.out.println(s);  
    }  
}
```
虽然可以直接拼接字符串，但是，在循环中，每次循环都会创建新的字符串对象，然后扔掉旧的字符串。这样，绝大部分字符串都是临时对象，不但浪费内存，还会影响GC效率。
为了能高效拼接字符串，Java标准库提供了`StringBuilder`，它是一个可变对象，可以预分配缓冲区，这样，往`StringBuilder`中新增字符时，不会创建新的临时对象：
```java
public class Main {  
    public static void main(String[] args) {  
        StringBuilder sb = new StringBuilder(1024);  
        for(int i = 0;i<1000;i++){  
            sb.append(",");  
            sb.append(i);  
        }  
        String s = sb.toString();  
        System.out.println(s);  
    }  
}
```
`StringBuilder`还可以进行链式操作：
```java
var sb = new StringBuilder(1024);  
sb.append("Mr ").append("Bob").append("!").insert(0, "Hello ");
```
## Maven
在了解Maven之前，我们先来看看一个Java项目需要的东西。首先，我们需要确定引入哪些依赖包。例如，如果我们需要用到[commons logging](https://commons.apache.org/proper/commons-logging/)，我们就必须把commons logging的jar包放入classpath。如果我们还需要[log4j](https://logging.apache.org/log4j/)，就需要把log4j相关的jar包都放到classpath中。这些就是依赖包的管理。

其次，我们要确定项目的目录结构。例如，`src`目录存放Java源码，`resources`目录存放配置文件，`bin`目录存放编译生成的`.class`文件。

此外，我们还需要配置环境，例如JDK的版本，编译打包的流程，当前代码的版本号。

最后，除了使用Eclipse这样的IDE进行编译外，我们还必须能通过命令行工具进行编译，才能够让项目在一个独立的服务器上编译、测试、部署。
Maven就是是专门为Java项目打造的管理和构建工具，它的主要功能有：

- 提供了一套标准化的项目结构；
- 提供了一套标准化的构建流程（编译，测试，打包，发布……）；
- 提供了一套依赖管理机制。
### Maven项目结构
```
a-maven-project
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   └── resources
│   └── test
│       ├── java
│       └── resources
└── target
```
项目的根目录`a-maven-project`是项目名，它有一个项目描述文件`pom.xml`，存放Java源码的目录是`src/main/java`，存放资源文件的目录是`src/main/resources`，存放测试源码的目录是`src/test/java`，存放测试资源的目录是`src/test/resources`，最后，所有编译、打包生成的文件都放在`target`目录里。这些就是一个Maven项目的标准目录结构。
所有的目录结构都是约定好的标准结构，我们千万不要随意修改目录结构。使用标准结构不需要做任何配置，Maven就可以正常使用。
我们再来看最关键的一个项目描述文件`pom.xml`，它的内容长得像下面：
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<project xmlns="http://maven.apache.org/POM/4.0.0"  
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">  
    <modelVersion>4.0.0</modelVersion>  
  
    <groupId>org.example</groupId>  
    <artifactId>rmi</artifactId>  
    <version>1.0-SNAPSHOT</version>  
  
    <properties>        <maven.compiler.source>25</maven.compiler.source>  
        <maven.compiler.target>25</maven.compiler.target>  
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>  
    </properties>  
</project>
```
其中，`groupId`类似于Java的包名，通常是公司或组织名称，`artifactId`类似于Java的类名，通常是项目名称，再加上`version`，一个Maven工程就是由`groupId`，`artifactId`和`version`作为唯一标识。
我们在引用其他第三方库的时候，也是通过这3个变量确定。例如，依赖`org.slfj4:slf4j-simple:2.0.16`：
```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-simple</artifactId>
    <version>2.0.16</version>
</dependency>
```
使用`<dependency>`声明一个依赖后，Maven就会自动下载这个依赖包并把它放到classpath中。
另外，注意到`<properties>`定义了一些属性，常用的属性有：

- `project.build.sourceEncoding`：表示项目源码的字符编码，通常应设定为`UTF-8`；
- `maven.compiler.release`：表示使用的JDK版本，例如`21`；
- `maven.compiler.source`：表示Java编译器读取的源码版本；
- `maven.compiler.target`：表示Java编译器编译的Class版本。

从Java 9开始，推荐使用`maven.compiler.release`属性，保证编译时输入的源码和编译输出版本一致。如果源码和输出版本不同，则应该分别设置`maven.compiler.source`和`maven.compiler.target`。

通过`<properties>`定义的属性，就可以固定JDK版本，防止同一个项目的不同的开发者各自使用不同版本的JDK。



## 参考文章
https://liaoxuefeng.com/books/java/introduction/index.html
