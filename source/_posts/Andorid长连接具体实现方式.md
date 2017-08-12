---
title: 使用Netty框架实现Andorid长连接
date: 2016-10-20 22:07:22
tags: Android
---
> NIO，即Non Blocking IO，非阻塞IO，在JAVA中NIO的核心就是Selector机制。简单而言，创建一个Socket Channel，并将其注册到一个Selector上（多路复用器），这个Selector将会“关注”Channel上发生的IO读写事件，并在事件发生（数据就绪）后执行相关的处理逻辑。对于阻塞IO，它需要在read()、write()操作上阻塞而直到数据操作完毕，但是NIO则不需要，只有当Selector检测到此Channel上有事件时才会触发调用read、write操作。

## 需要实现的功能

- 创建后台服务，在服务中和服务器建立长连接
- 定时发送心跳包
- 断线重连机制

<!-- more -->

## 具体思路：

- 使用IdleStateHandler来检测读写操作的空闲时间
- 客户端write空闲5s后向服务端发送一个心跳包
- 在发送下一个心跳包之前，未收到服务端应答，失败计数器加一
- 收到服务端应答，失败计数器清零
- 超过没有收到应答最大次数后断开连接
- 重新和服务器建立连接，如果连接失败，会在固定时间间隔之后重新连接。

## 具体实现

- 引入包

```java
dependencies {
      compile 'io.netty:netty-all:5.0.0.Alpha2'
}
```

- 初始化Clinet服务，和后台建立长连接

```java
    public void startServer() {
        eventLoopGroup = new NioEventLoopGroup();
        // Client服务启动器 3.x的ClientBootstrap
        // 改为Bootstrap，且构造函数变化很大，这里用无参构造。
        bootstrap = new Bootstrap();
        try {
            // 指定channel类型
            bootstrap.channel(NioSocketChannel.class);
            //设置TCP协议的属性
            bootstrap.option(ChannelOption.SO_KEEPALIVE, true);
            // 指定EventLoopGroup
            bootstrap.group(eventLoopGroup);
            //指定地址
            bootstrap.remoteAddress(ConstsNet.host, ConstsNet.port);
            // 指定Handler
            bootstrap.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel socketChannel) throws Exception {
                    //回行换车解码器
                    socketChannel.pipeline().addLast(new LineBasedFrameDecoder(2048));
                  	//自定义数据编解码器
                    socketChannel.pipeline().addLast(new DataDecoder());
                    socketChannel.pipeline().addLast(new DataEncoder());
                    // 自己的逻辑Handler
                    socketChannel.pipeline().addLast(new KeepAliveServerInitializer());
                }
            });
            //同步创建连接
            ChannelFuture future = bootstrap.connect(ConstsNet.host, ConstsNet.port).sync();
            if (future.isSuccess()) {
                //Channel中所有的操作均是异步的，IO操作都会返回一个ChannelFuture实例：
                socketChannel = (SocketChannel) future.channel();
                //发送心跳包
                socketChannel.writeAndFlush({code:'login'});
            }
        } catch (Exception e) {
            e.printStackTrace();
            Utils.showToast(this, getResources().getString(R.string.action_networke));
            if (!isStop) {
              	//出错之后开启断线重连
                connServer();
            }
        }
    }
```

- 创建延迟任务，以固定的时间间隔重新连接服务器

```java
    private void connServer() {
        try {
            isStop = false;
            if (executorService != null) {
                executorService.shutdown();
            }
            executorService = Executors.newScheduledThreadPool(1);
            //本次任务执行完成后，需要延迟设定的延迟时间，才会执行新的任务
            executorService.scheduleWithFixedDelay(new Runnable() {

                boolean isConnSucc = true;

                @Override
                public void run() {
                    try {
                        // 重置计数器
                        unRecPongTimes = 0;
                        // 连接服务端
                        if (socketChannel != null && socketChannel.isOpen()) {
                            socketChannel.close();
                        }
                        ChannelFuture future = bootstrap.connect(ConstsNet.host, ConstsNet.port).sync();
                        if (future.isSuccess()) {
                            socketChannel = (SocketChannel) future.channel();
                            socketChannel.writeAndFlush({code:'login'});
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                        isConnSucc = false;
                    } finally {
                        if (isConnSucc) {
                            if (executorService != null) {
                                //如果连接成功 关闭延时任务
                                executorService.shutdown();
                            }
                        }
                    }
                }
            }, RE_CONN_WAIT_SECONDS, RE_CONN_WAIT_SECONDS, TimeUnit.SECONDS);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

- 设置心跳包发送间隔和心跳包Handler

```java
private class KeepAliveServerInitializer extends ChannelInitializer<SocketChannel> {

        @Override
        protected void initChannel(SocketChannel ch) throws Exception {
            ch.pipeline().addLast(new LineBasedFrameDecoder(2048));
            //检测读写操作的空闲时间
            ch.pipeline().addLast(new IdleStateHandler(0, WRITE_WAIT_SECONDS, 0));
            ch.pipeline().addLast(new DataDecoder());
            ch.pipeline().addLast(new DataEncoder());
            //设置心跳处理器
            ch.pipeline().addLast(new Heartbeat());
        }
    }
```

- 心跳包处理器

```java
 private class Heartbeat extends SimpleChannelInboundHandler<String> {

        JSONObject data;

        /**
         * 每一次收到消息时触发。
         * @param ctx
         * @param msg
         * @throws Exception
         */
        @Override
        protected void messageReceived(ChannelHandlerContext ctx, String msg) throws Exception {
            if (msg == null) {
                return;
            }
            data = new JSONObject(msg);
            if (!data.has("code")) {
                return;
            }
            // 失败计数器清零
            unRecPongTimes = 0;
           //处理从服务端接收到的消息 TODO
        }

        //如果在指定的时间内容没有write事件，就会触发该方法
        public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
            if (evt instanceof IdleStateEvent) {
                IdleStateEvent event = (IdleStateEvent) evt;
                if (event.state() == IdleState.READER_IDLE) {
                    Logger.i("读超时)");
                } else if (event.state() == IdleState.WRITER_IDLE) {
                    if (unRecPongTimes < MAX_UN_REC_PONG_TIMES) {
                        //发送下一个心跳包，检测服务端是否还存活
                        ctx.channel().writeAndFlush("{code:'ping'}");
                        unRecPongTimes++;
                    } else {
                        ctx.channel().close();
                    }
                } else if (event.state() == IdleState.ALL_IDLE) {
                    /*总超时*/
                    Logger.i("总超时)");
                }
            }
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            Logger.i("错误原因：" + cause.getMessage());
            if (data != null) {
            }
            ctx.channel().close();
        }

        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            super.channelActive(ctx);
        }

        /**
         * 连接断开
         * @param ctx
         * @throws Exception
         */
        @Override
        public void channelInactive(ChannelHandlerContext ctx) throws Exception {
            ctx.close();
             /*
             * 重连
	         */
            if (!isStop) {
                connServer();
            }
            Logger.i("客户端失效");
        }
    }
```