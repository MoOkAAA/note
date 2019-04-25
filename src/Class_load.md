# 类加载过程
* 加载
* 验证
* 准备
* 解析 
* 初始化
* 实例化
* 卸载
> 类加载 和 链接(验证 准备 解析) 阶段是 交织进行 只保证各个阶段的开始顺序按照流程来
***
### 加载
> 类加载器可在预见类可能被使用之前加载他,若类的.class文件丢失或存在错误,将在加载时报错,未加载不报错,类加载器**将类的.class文件中的二进制数据读入到运行时数据区的方法区内**,然后**在堆区创建java.lang.Class对象**,用来封装类在方法区内的数据结构,Class对象封装了类在方法区内的数据结构,并且提供了访问方法区内的数据结构的接口.(PS.数据结构指类的变量,方法等)

> 类的加载途径:
> * 从本地系统中直接加载
> * 通过网络下载.class文件
> * 从zip，jar等归档文件中加载.class文件
> * 从专有数据库中提取.class文件
> * 将Java源文件动态编译为.class文件 

## 验证
> 验证是连接阶段的第一步 验证字节流是否符合虚拟机规范要求 不会危害虚拟机安全,主要包含4个阶段检验动作:
> * 文件格式验证：验证字节流是否符合Class文件格式的规范；例如：是否以 0xCAFEBABE开头、主次版本号是否在当前虚拟机的处理范围之内、常量池中的常量是否有不被支持的类型。
> * 元数据验证：对字节码描述的信息进行语义分析（注意：对比javac编译阶段的语义分析），以保证其描述的信息符合Java语言规范的要求；例如：这个类是否有父类，除了 java.lang.Class之外。
> * 字节码验证：通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。
> * 符号引用验证：确保解析动作能正确执行。

## 准备
> 为类的 静态变量分配内存，并将其初始化为默认值
> 不包括实例变量，实例变量会在对象实例化时随着对象一块分配在Java堆中。
> ` public static int value = 3;`
> a 的值在准备阶段为0,赋值操作存放于类构造器<clinit>() 中,将在初始化阶段执行
> ` public static final int value = 3;`
> 此时 a在准备阶段的值为 3

## 解析
> 把类中的符号引用转换为直接引用
> 符号引用就是一组符号来描述目标，可以是任何字面量。
> 直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。
>
> 解析动作主要针对
> * 类
> * 接口
> * 字段
> * 类方法
> * 接口方法
> * 方法类型
> * 方法句柄
> * 调用点限定符

## 初始化

> 对类变量进行初始值设定:
>
> * **声明类变量是指定初始值**
> * **使用静态代码块为类变量指定初始值**
>
> JVM(Java 虚拟机)初始化步骤:
>
> - 假如这个类还没有被加载和连接，则程序先加载并连接该类
> - 假如该类的直接父类还没有被初始化，则先初始化其直接父类
> - 假如类中有初始化语句，则系统依次执行这些初始化语句
>
> 类初始化时机:
>
> * new
> * 访问类或接口的静态变量,调用静态方法
> * 反射(`Class.forName();`)
> * 初始化子类,父类会先于子类初始化
> * Java虚拟机启动时被标明为启动类的类（ `JavaTest`），直接使用 `java.exe`命令来运行某个主类

# 类加载器

```java
package com.neo.classloader;
public class ClassLoaderTest {
    public static void main(String[] args) {
        ClassLoader loader = Thread.currentThread().getContextClassLoader();
        System.out.println(loader);
        System.out.println(loader.getParent());
        System.out.println(loader.getParent().getParent());
    }
} 
```
```java
sun.misc.Launcher$AppClassLoader@64fef26a
sun.misc.Launcher$ExtClassLoader@1ddd40f3
null
```

![](<http://mmbiz.qpic.cn/mmbiz_png/PgqYrEEtEnokxXiapDdvntH8PGwa0zGXM7qXKib1ibsib1BuyLxjoP1sgorwib78yTD4896N5r1AibdhDXTHZ7z9VyBg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1>)

> PS. 此处**非继承关系**,而为**组合关系**

> 由代码执行结果可以得知,无法获得 因为由C语言实现的Bootstrap ClassLoader,无法确定返回的方式,所以为null

> * **启动类加载器**： `BootstrapClassLoader` 加载存放于**`JDK\jre\lib`**下, 或被`-Xbootclasspath`参数指定的路径中的类,所以**`java.`**开头的类均被此加载器加载, Java 程序无法直接引用(PS. 原因如上)
> * **扩展类加载器**： `ExtensionClassLoader`加载 **`JDK\jre\lib\ext`**目录中，或者由 **`java.ext.dirs`**系统变量指定的路径中的所有类库（如`javax.`开头的类），开发者可以直接使用扩展类加载器。
> * **应用程序类加载器**： `ApplicationClassLoader`加载用户类路径（`ClassPath`）所指定的类，开发者可以直接使用该类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

> **JVM类加载机制**
>
> - **全盘负责**，当一个类加载器负责加载某个Class时，该Class所依赖的和引用的其他Class也将由该类加载器负责载入，除非显示使用另外一个类加载器来载入
> - **父类委托**，先让父类加载器试图加载该类，只有在父类加载器无法加载该类时才尝试从自己的类路径中加载该类(PS.双亲委派模型)
> - **缓存机制**，缓存机制将会保证所有加载过的Class都会被缓存，当程序中需要使用某个Class时，类加载器先从缓存区寻找该Class，只有缓存区不存在，系统才会读取该类对应的二进制数据，并将其转换成Class对象，存入缓存区。这就是为什么修改了Class后，必须重启JVM，程序的修改才会生效

# 其他

## 类加载方式

>* 命令行启动应用时候由JVM初始化加载
>* 通过`Class.forName()`方法动态加载
>* 通过`ClassLoader.loadClass()`方法动态加载

## Class.forName()和ClassLoader.loadClass()区别

> - `Class.forName()`：将类的.class文件加载到jvm中之外，还会对类进行解释，执行类中的static块；
> - `ClassLoader.loadClass()`：只干一件事情，就是将.class文件加载到jvm中，不会执行static中的内容,只有在newInstance才会去执行static块。
> - `Class.forName(name,initialize,loader)`带参函数也可控制是否加载static块。并且只有调用了newInstance()方法采用调用构造函数，创建类的对象 。