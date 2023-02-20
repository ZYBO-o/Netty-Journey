# 一. 概述

Future与Promise在异步处理时，经常用到这两个接口。

首先要说明 netty 中的 Future 与 jdk 中的 Future 同名，但是是两个接口，**netty 的 Future 继承自 jdk 的 Future，而 Promise 又对 netty Future 进行了扩展** 。

-  **jdk Future** ：只能同步等待任务结束（或成功、或失败）才能得到结果；
-  **netty Future** ：可以同步等待任务结束得到结果，也可以异步方式得到结果，但都是要等任务结束；
-  **netty Promise** ：不仅有 netty Future 的功能，而且脱离了任务独立存在， **只作为两个线程间传递结果的容器** 。

| 功能/名称    | jdk Future                     | netty Future                                                 | Promise      |
| ------------ | :----------------------------- | ------------------------------------------------------------ | ------------ |
| cancel       | 取消任务                       | -                                                            | -            |
| isCanceled   | 任务是否取消                   | -                                                            | -            |
| isDone       | 任务是否完成，不能区分成功失败 | -                                                            | -            |
| get          | 获取任务结果，阻塞等待         | -                                                            | -            |
| getNow       | -                              | 获取任务结果，非阻塞，还未产生结果时返回 null                | -            |
| await        | -                              | 等待任务结束，如果任务失败，不会抛异常，而是通过 isSuccess 判断 | -            |
| sync         | -                              | 等待任务结束，如果任务失败，抛出异常                         | -            |
| isSuccess    | -                              | 判断任务是否成功                                             | -            |
| cause        | -                              | 获取失败信息，非阻塞，如果没有失败，返回null                 | -            |
| addLinstener | -                              | 添加回调，异步接收结果                                       | -            |
| setSuccess   | -                              | -                                                            | 设置成功结果 |
| setFailure   | -                              | -                                                            | 设置失败结果 |

# 二. 使用

## 1. JDK Future

```java
public class JDKFuture {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        // 1. 创建线程池
        ExecutorService service = Executors.newFixedThreadPool(2);
        // 2. 提交任务
        Future<Integer> future = service.submit(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                System.out.println(Thread.currentThread() + " : 计算结果中..");
                Thread.sleep(1000);
                return 50;
            }
        });

        System.out.println(Thread.currentThread() + " : 等待结果..");
        //3. 主线程通过future获取结果,阻塞等待结果返回
        Integer integer = future.get();
        System.out.println(Thread.currentThread() + " : 得到结果 = " + integer);
    }
}
```

运行结果：

```luna
Thread[main,5,main] : 等待结果..
Thread[pool-1-thread-1,5,main] : 计算结果中..
Thread[main,5,main] : 得到结果 = 50
```

## 2. Netty Future

```java
public class NettyFuture {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        NioEventLoopGroup group = new NioEventLoopGroup();

        EventLoop eventLoop = group.next();

        Future<Integer> future = eventLoop.submit(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                System.out.println(Thread.currentThread() + " : 计算结果中..");
                Thread.sleep(1000);
                return 70;
            }
        });

        //同步方法，主线程中获取结果
        //System.out.println(Thread.currentThread() + " : 等待结果..");
        //Integer integer = future.get();
        //System.out.println(Thread.currentThread() + " : 得到结果 = " + integer);

        //异步方法
        future.addListener(new GenericFutureListener<Future<? super Integer>>() {
            @Override
            public void operationComplete(Future<? super Integer> future) throws Exception {
                System.out.println(Thread.currentThread() + " : 得到结果 = " + future.getNow());
            }
        });
    }
}
```

运行结果：

```java
Thread[nioEventLoopGroup-2-1,10,main] : 计算结果中..
Thread[nioEventLoopGroup-2-1,10,main] : 得到结果 = 70
```

Netty 中的 Future 对象，可以通过 EventLoop 的 `sumbit()` 方法得到

- 可以通过 Future 对象的 **`get()`方法** ，阻塞地获取返回结果
- 也可以通过 **`getNow()`方法** ，获取结果，若还没有结果，则返回 null ，该方法是非阻塞的
- 还可以通过 **`future.addListener()`方法** ，在 `Callable()` 方法执行的线程中，异步获取返回结果

## 3. Netty Promise

```java
public class NettyPromise {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        //1.准备 EventLoop 对象
        NioEventLoopGroup group = new NioEventLoopGroup();
        EventLoop loop = group.next();

        //2.可以主动创建 promise, 结果容器
        DefaultPromise<Integer> promise = new DefaultPromise<>(loop);

        new Thread(() -> {
            //3.任意一个线程执行计算，计算完毕后向 promise 填充结果
            System.out.println(Thread.currentThread() + " : 开始计算...");
            try {
                Thread.sleep(5000);
                promise.setSuccess(80);
            } catch (InterruptedException e) {
                e.printStackTrace();
                promise.setFailure(e);
            }
        }).start();

        //4.接收结果的线程
        System.out.println(Thread.currentThread() + " : 接收结果...");
        System.out.println(Thread.currentThread() + " : 结果 = " + promise.get());
    }
}
```

结果：

```luna
Thread[main,5,main] : 接收结果...
Thread[Thread-0,5,main] : 开始计算...
Thread[main,5,main] : 结果 = 80
```

