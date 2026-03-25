## Hello,World
新建一个文件hello.java
```java
public class hello{
	public static void main(String[] args){
		System.out.print("Hello World");
	}
}
```
要运行代码需要进入cmd，首先进行编译
```bash
javac hello.java
```
执行成功后就可以看到文件里多了一个`hello.class`的文件，这就是编译成功了。
接下来就要运行了
```bash
java hello
```
这样就会输出`Hello World`，它本质上是运行的hello.class这个文件。
当然如果有IDEA的话就不用这么麻烦了，直接RUN就行了。
**注意**
- 文件名和类名要一致
- 因为类名不可以以数字开头，也不可以是java关键字，所以命名文件时要避开。
在IDEA直接输入`psvm`回车就可以直接出现
```java
public static void main(String[] args) {  
    
}
```
## 注释
`//`单行注释
`/**/`多行注释
