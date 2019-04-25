# JVM内存

* 方法区(Method Area)
* 堆(Heap)
* 栈(Stack)

***

## 方法区

> 又叫静态区，跟堆一样，被所有的线程共享。方法区包含所有的static变量, 与类的类型信. 

## 堆

> 保存所以对象本身, 每个对象所对应的类型信息保存在方法区(该class有那些操作指令方法)

## 栈

> 每个线程都有自己的栈, 互相之间不能访问.
> 栈中保存的是基本数据类型和对堆内对象的引用.
> 栈分为3部分: 基本数据类型区, 执行环境上下文, 操作指令区.

## 其他

```java
public class Bar {
    public static void main(String[] args) {
        Foo foo = new Foo("123");
        foo.getName();
    }
}

public class Foo{
    private String name;
    public Foo(String name) {
        this.name = name;
    }
    public String getName() {
        return name;
    }
}
```

> 运行`Bar.main();`执行流程: 读取Bar.class内的二进制字节流到方法区,并保存Bar类的类型信息, 执行main方法, 加载Foo.class, 堆中创建Foo对象, 栈中创建指向该对象的实例foo, 并通过对象获取该类的操作指令. 调用getName方法.

### String

```java
public class StringTest {
    public static void main(String[] args) throws InterruptedException {
        String a = "123";
        System.out.println(System.identityHashCode(a));
        new Thread(() -> {
            String a1 = "123";
            System.out.println(System.identityHashCode(a1));
        }).start();
        Thread.sleep(100);
        String b = new String("123");
        System.out.println(a == b);
        String c = "1";
        String d = c + "23";
        System.out.println(a == d);
        String e = "1" + "2" + "3";
        System.out.println(a == e);
        String f = b.intern();
        System.out.println(a == f);
    }
}
```

```java
460141958
460141958
false
false
true
true
```

> String 创建的字符串 保存在字符串常量池中,字符串常量池位于方法区.
>
> 两个相等的 460131958说明:
>
> ```java
>     /**
>      * Returns the same hash code for the given object as
>      * would be returned by the default method hashCode(),
>      * whether or not the given object's class overrides
>      * hashCode().
>      * The hash code for the null reference is zero.
>      *
>      * @param x object for which the hashCode is to be calculated
>      * @return  the hashCode
>      * @since   JDK1.1
>      */
>     public static native int identityHashCode(Object x);
> ```
>
> 该方法的意思是 执行给与对象的`hashCode()`方法, 并不受该方法被子类重写影响. 查看`Object.hashCode()`源码注释 有一句: 
>
> ```java
> This is typically implemented by converting the internal
> address of the object into an integer.
> ```
>
> 将内存地址转为Integer值. 所以当2个对象的hashCode相等时 那他们的内存地址也应该是同一个. 所以证明保存字符串常量池的地方为一个线程共享的区域.

> String b = new String("123"); 该方式声明一个String实例, 会在堆中创建一个String对象, 然后在去字符串常量池查找是否有"123"存在,无则创建.所以 b的指针指向堆中对象的地址, 而a指针指向字符串常量池中的"123".(PS. == 比较的是2个实例指向的内存地址, 如果要比较值是否相等要用 equals)

> Java编译器会将显示的常量进行拼接, 这也是为什么`a == e`的原因, 因为在加载类的字节码时 e 已经被转变为 `e = "123"`而为什么 `a == d` 为false 因为 c 是一个变量, Java编译器并不能确定 c在后面是否会改变, 所以他并不会去拼接, 而是让jvm自己去执行 拿取 c的值 拼上"23" 并存放在常量池中.

> String.intern(); 方法会返回该对象在字符串常量池中的值.

### Integer

```java
public class IntegerTest {
    public static void main(String[] args) {
        Integer a = 20;
        Integer b = 20;
        System.out.println(a == b);
        a = 200;
        b = 200;
        System.out.println(a == b);
        Integer c = new Integer(20);
        Integer d = new Integer(20);
        System.out.println(c == d);
        System.out.println(System.identityHashCode(a));
        new Thread(() -> {
            Integer a1 = 20;
            System.out.println(System.identityHashCode(a1));
        }).start();
    }
}
```

```java
true
false
false
460141958
715341055
```

> PS. == 比较的是内存地址而非值.
>
> 同样的类似与字符串常量池, int 也有自己的常量池, 但是这个常量池 并不增加 固定的为[-128, 127], 当你创建的实例为这个范围内的int 时, 会从这个常量池中获取.当超出这个范围时,则会创建新的对象存于栈中,而且不会查询这个值是否存在.`Integer c = new Integer(20);Integer d = new Integer(20);` 会在堆中创建2个对象, 所以他们的地址也不相等的. 最后2个数字不相等, 也说明栈与栈之间无法互相访问.

