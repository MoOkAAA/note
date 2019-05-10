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

