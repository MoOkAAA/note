# 无锁的后果和产生的原因

> 先来看一段代码:

```java
private static Integer count = 0;

public static void main(String[] args) throws InterruptedException {
    CountDownLatch countDownLatch = new CountDownLatch(10);
    for (int i = 0; i < 100; i++) {
        new Thread(() -> {
            try {
                count++;
            } finally {
                countDownLatch.countDown();
            }
        }).start();
    }
    countDownLatch.await();
    System.out.println(count);
}
```

> 100个线程去执行count++ 按道理输出为100 运行5次 获得一下结果
>
> 97
>
> 96
>
> 94
>
> 84
>
> 98

## 原因

### java 内存模型

![](<https://pic3.zhimg.com/80/v2-a1a75c9f7264cf78d0927663371ca9d2_hd.jpg>)

> 改变一个值 并将他写回内存中一共分为8部, 所以在无锁的情况下, 当2个线程同时 读取到值为20 的count 时, A线程将值++ 为21 写入内存后, B线程同样的操作 会使 内存中的count 在经过2次++ 之后为21.有很多种方式,来避免这种操作.加锁 是其中一种