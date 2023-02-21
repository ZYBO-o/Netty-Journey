# 一. 粘包现象

**服务器代码：**

```java
@Slf4j
public class HelloServer {
    void start(){
        NioEventLoopGroup boss = new NioEventLoopGroup(1);
        NioEventLoopGroup worker = new NioEventLoopGroup();

        try {
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.channel(NioServerSocketChannel.class);
//            serverBootstrap.option(ChannelOption.SO_RCVBUF,10);
            serverBootstrap.group(boss,worker);
            serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new LoggingHandler(LogLevel.DEBUG));
                    ch.pipeline().addLast(new ChannelInboundHandlerAdapter(){
                        // 连接建立时会执行该方法
                        @Override
                        public void channelActive(ChannelHandlerContext ctx) throws Exception {
                            log.debug("connected{}",ctx.channel());
                            super.channelActive(ctx);
                        }
                        // 连接断开时会执行该方法
                        @Override
                        public void channelInactive(ChannelHandlerContext ctx) throws Exception {
                            log.debug("disconnect {}", ctx.channel());
                            super.channelInactive(ctx);
                        }
                    });
                }
            });
            ChannelFuture channelFuture = serverBootstrap.bind(8080);
            log.debug("{} binding...",channelFuture.channel());
            channelFuture.sync();
            log.debug("{} bound...",channelFuture.channel());
            channelFuture.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            log.error("server error", e);
        }finally {
            boss.shutdownGracefully();
            worker.shutdownGracefully();
            log.debug("stoped");
        }

    }

    public static void main(String[] args) {
        new HelloServer().start();
    }
}
```

**客户端代码：**

```java
@Slf4j
public class HelloWorldClient {
    public static void main(String[] args) {
        NioEventLoopGroup worker = new NioEventLoopGroup();

        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.channel(NioSocketChannel.class);
            bootstrap.group(worker);
            bootstrap.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                    log.debug("connetted...");
                    ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                        //会在连接channel建立连接后，触发active事件
                        @Override
                        public void channelActive(ChannelHandlerContext ctx) throws Exception {
                            log.debug("sending...");
                       
                            for (int i = 0; i < 10; i++) {
                                ByteBuf buffer = ctx.alloc().buffer();
                                buffer.writeBytes(new byte[]{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15});
                                ctx.writeAndFlush(buffer);
                            }
                        }
                    });
                }
            });
            ChannelFuture channelFuture = bootstrap.connect("127.0.0.1", 8080).sync();
            channelFuture.channel().closeFuture().sync();

        } catch (InterruptedException e) {
            log.error("client error", e);
        } finally {
            worker.shutdownGracefully();
        }

    }
}
```

**运行结果：**

```java
22:51:18 [DEBUG] [main] c.l.n.p.HelloWorldServer - [id: 0x10e3b18e, L:/0:0:0:0:0:0:0:0:8080] binding...
22:51:18 [DEBUG] [main] c.l.n.p.HelloWorldServer - [id: 0x10e3b18e, L:/0:0:0:0:0:0:0:0:8080] bound...
22:51:25 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x193974e2, L:/127.0.0.1:8080 - R:/127.0.0.1:61906] REGISTERED
22:51:25 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x193974e2, L:/127.0.0.1:8080 - R:/127.0.0.1:61906] ACTIVE
22:51:25 [DEBUG] [nioEventLoopGroup-3-1] c.l.n.p.HelloWorldServer - connected[id: 0x193974e2, L:/127.0.0.1:8080 - R:/127.0.0.1:61906]
22:51:25 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x193974e2, L:/127.0.0.1:8080 - R:/127.0.0.1:61906] READ: 160B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f |................|
|00000010| 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f |................|
|00000020| 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f |................|
|00000030| 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f |................|
|00000040| 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f |................|
|00000050| 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f |................|
|00000060| 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f |................|
|00000070| 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f |................|
|00000080| 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f |................|
|00000090| 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f |................|
+--------+-------------------------------------------------+----------------+
22:51:25 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x193974e2, L:/127.0.0.1:8080 - R:/127.0.0.1:61906] READ COMPLETE
```

可见虽然客户端是分别以16字节为单位，通过channel向服务器发送了10次数据，可是 **服务器端却只接收了一次，接收数据的大小为160B，即客户端发送的数据总大小，这就是粘包现象** 。

# 二. 半包现象

将客户端-服务器之间的channel容量进行调整,将服务端注释掉的代码打开

- **服务端代码**

```java
// 调整channel的容量
serverBootstrap.option(ChannelOption.SO_RCVBUF, 10);
```

**注意**

>  `serverBootstrap.option(ChannelOption.SO_RCVBUF, 10)`  影响的底层接收缓冲区（即滑动窗口）大小，仅决定了 netty 读取的最小单位，**netty 实际每次读取的一般是它的整数倍**

**运行结果**

```javascript
22:56:15 [DEBUG] [main] c.l.n.p.HelloWorldServer - [id: 0x3185dbfc, L:/0:0:0:0:0:0:0:0:8080] binding...
22:56:15 [DEBUG] [main] c.l.n.p.HelloWorldServer - [id: 0x3185dbfc, L:/0:0:0:0:0:0:0:0:8080] bound...
22:56:24 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x206018a7, L:/127.0.0.1:8080 - R:/127.0.0.1:61994] REGISTERED
22:56:24 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x206018a7, L:/127.0.0.1:8080 - R:/127.0.0.1:61994] ACTIVE
22:56:24 [DEBUG] [nioEventLoopGroup-3-1] c.l.n.p.HelloWorldServer - connected[id: 0x206018a7, L:/127.0.0.1:8080 - R:/127.0.0.1:61994]
22:56:24 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x206018a7, L:/127.0.0.1:8080 - R:/127.0.0.1:61994] READ: 36B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f |................|
|00000010| 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f |................|
|00000020| 00 01 02 03                                     |....            |
+--------+-------------------------------------------------+----------------+
22:56:24 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x206018a7, L:/127.0.0.1:8080 - R:/127.0.0.1:61994] READ COMPLETE
22:56:24 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x206018a7, L:/127.0.0.1:8080 - R:/127.0.0.1:61994] READ: 40B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 00 01 02 03 |................|
|00000010| 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 00 01 02 03 |................|
|00000020| 04 05 06 07 08 09 0a 0b                         |........        |
+--------+-------------------------------------------------+----------------+
22:56:24 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x206018a7, L:/127.0.0.1:8080 - R:/127.0.0.1:61994] READ COMPLETE
22:56:24 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x206018a7, L:/127.0.0.1:8080 - R:/127.0.0.1:61994] READ: 40B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 0c 0d 0e 0f 00 01 02 03 04 05 06 07 08 09 0a 0b |................|
|00000010| 0c 0d 0e 0f 00 01 02 03 04 05 06 07 08 09 0a 0b |................|
|00000020| 0c 0d 0e 0f 00 01 02 03                         |........        |
+--------+-------------------------------------------------+----------------+
22:56:24 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x206018a7, L:/127.0.0.1:8080 - R:/127.0.0.1:61994] READ COMPLETE
22:56:24 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x206018a7, L:/127.0.0.1:8080 - R:/127.0.0.1:61994] READ: 40B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 00 01 02 03 |................|
|00000010| 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f 00 01 02 03 |................|
|00000020| 04 05 06 07 08 09 0a 0b                         |........        |
+--------+-------------------------------------------------+----------------+
22:56:24 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x206018a7, L:/127.0.0.1:8080 - R:/127.0.0.1:61994] READ COMPLETE
22:56:24 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x206018a7, L:/127.0.0.1:8080 - R:/127.0.0.1:61994] READ: 4B
         +-------------------------------------------------+
         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |
+--------+-------------------------------------------------+----------------+
|00000000| 0c 0d 0e 0f                                     |....            |
+--------+-------------------------------------------------+----------------+
22:56:24 [DEBUG] [nioEventLoopGroup-3-1] i.n.h.l.LoggingHandler - [id: 0x206018a7, L:/127.0.0.1:8080 - R:/127.0.0.1:61994] READ COMPLETE
```

从结果可知客户端每次发送的数据， **因channel容量不足，无法将发送的数据一次性接收** ，便产生了半包现象。

# 三. 现象分析

## 1. 粘包

- 现象，发送 abc def，接收 abcdef
- 原因
  - 应用层：接收方 ByteBuf 设置太大（Netty 默认 1024）
  - 滑动窗口：假设发送方 256 bytes 表示一个完整报文，但由于接收方处理不及时且窗口大小足够大，这 256 bytes 字节就会缓冲在接收方的滑动窗口中，当滑动窗口中缓冲了多个报文就会粘包
  - Nagle 算法：会造成粘包

## 2. 半包

- 现象，发送 abcdef，接收 abc def
- 原因
  - 应用层：接收方 ByteBuf 小于实际发送数据量
  - 滑动窗口：假设接收方的窗口只剩了 128 bytes，发送方的报文大小是 256 bytes，这时放不下了，只能先发送前 128 bytes，等待 ack 后才能发送剩余部分，这就造成了半包
  - MSS 限制：当发送的数据超过 MSS 限制后，会将数据切分发送，就会造成半包

## 3. 滑动窗口



## 4. MSS 限制





## 5. Nagle 算法

- 即使发送一个字节，也需要加入 tcp 头和 ip 头，也就是总字节数会使用 41 bytes，非常不经济。因此为了提高网络利用率，tcp 希望尽可能发送足够大的数据，这就是 Nagle 算法产生的缘由
- 该算法是指发送端即使还有应该发送的数据，但如果这部分数据很少的话，则进行延迟发送
  - 如果 SO_SNDBUF 的数据达到 MSS，则需要发送
  - 如果 SO_SNDBUF 中含有 FIN（表示需要连接关闭）这时将剩余数据发送，再关闭
  - 如果 TCP_NODELAY = true，则需要发送
  - 已发送的数据都收到 ack 时，则需要发送
  - 上述条件不满足，但发生超时（一般为 200ms）则需要发送
  - 除上述情况，延迟发送
