---
title: Netty学习(一)
date: 2018-08-28 19:25:06
categories: 
- Netty
tags:
- Netty
---

> 之前有做过消息推送相关的应用，使用的Netty框架，一直对这个框架非常感兴趣，也学习了一些它的原理，但感觉还是不够，所以想从今天开始对Netty框架写一个系列的使用及原理学习的博客，提升自己，也希望对看到这篇博客的朋友有所帮助，欢迎大家一起讨论。

> 我一直从事Android开发岗位，后台知识是自学，没有真正参加一个后台项目，所以在文中后台开发比较简单，如有问题欢迎指出，共同学习。

今天写第一篇博客，还是先从Netty框架的使用开始，我自己做了一个easyIM的简单Demo，可以实现简单的聊天功能，使用Protocol Buffer传输数据，以后会继续完善它的功能。

服务端代码地址 [Github/easyImServer](https://github.com/David1840/easyimServer)

客户端代码地址 [Github/easyIm](https://github.com/David1840/easyim)

## 一、服务端

使用SpringBoot搭建的后台服务，比较简单。


#### 创建服务端主逻辑

```
fun start() {
        val boss = NioEventLoopGroup() //用于处理服务器端接收客户端连接
        val worker = NioEventLoopGroup() //进行网络通信（读写）

        try {
            val port = nettyConfig.port  //配置文件中配置端口
            val bootStrap = ServerBootstrap()   //辅助工具类，用于服务器通道配置
            bootStrap.group(boss, worker)      //绑定两个线程组
                    .channel(NioServerSocketChannel::class.java) //指定NIO的模式
                    .childHandler(ProtocolPipeline())            //配置具体的数据处理方式
                    .option(ChannelOption.SO_BACKLOG, 1024)          //设置TCP缓冲区
                    .option(ChannelOption.SO_SNDBUF, 32 * 1024) //设置发送数据缓冲大小
                    .option(ChannelOption.SO_RCVBUF, 32 * 1024) //设置接受数据缓冲大小
                    .childOption(ChannelOption.SO_KEEPALIVE, true)  //保持连接
                    .childOption(ChannelOption.TCP_NODELAY, true)   //禁用Nagle算法，降低延迟

            val future = bootStrap.bind(port).sync() 
            logger.info("server start finish,the port is $port")

            future.channel().closeFuture().sync()

        } catch (e: InterruptedException) {
            logger.error("server start error ${e.message.toString()}")
        } finally {
            boss.shutdownGracefully()
            worker.shutdownGracefully()
        }
    }
```
#### ProtocolPipeline数据处理

```
class ProtocolPipeline : ChannelInitializer<SocketChannel>() {
    override fun initChannel(ch: SocketChannel) {
        val pipeline = ch.pipeline()

        pipeline.addLast("send heartbeat", IdleStateHandler(60, 0, 0, TimeUnit.SECONDS)) //心跳机制，读空闲，60S

        // 使用Protobuf，客户端和服务端必须保持一致
        pipeline.addLast(ProtobufVarint32FrameDecoder())
        pipeline.addLast("proto decoder", ProtobufDecoder(IMessage.Protocol.getDefaultInstance()))
        pipeline.addLast(ProtobufVarint32LengthFieldPrepender())
        pipeline.addLast("proto encoder", ProtobufEncoder())
        pipeline.addLast(ServerHandler()) //接收到数据后的处理逻辑
    }
}
```

#### 传输数据

数据使用protobuf，

```
syntax = "proto2";

message Protocol {
    optional ContentType contentType = 1;  //类型
    optional bytes content = 2;  //内容
}

//数据类型
enum ContentType {
    Register_INFO = 0;
    Register_UUID = 1;
    Message_INFO = 2;
    HEART_BEAT = 3;
}

//发送给所有人还是发给一个人
enum MessageType {
    ALL = 0;
    ONE = 1;
}

//注册，客户端发给服务端
message Register {
    optional string name = 1;
}

//注册返回，服务端发给客户端
message RegisterUUID {
    optional string name = 1;
    optional string UUID = 2;
}

// 消息类
message Message {
    optional MessageType type = 1; //个人还是全部
    required string uuid = 2;    //如果发送给个人，此项必填
    optional string message = 3; //消息具体内容
}

//客户端发送给服务端心跳包
message HeartBeat_Ping{
    required string time = 1;
    required string uuid = 2;
}

//服务端返回客户端心跳包
message HeartBeat_Pong{
    required string time = 1;
    required string uuid = 2;
}
```

#### ServerHandler处理逻辑

```

class ServerHandler : ChannelInboundHandlerAdapter() {

    private val logger = LoggerFactory.getLogger(ServerHandler::class.java)

    // 心跳丢失计数器
    private var counter: Int = 0

    @Throws(Exception::class)
    override fun channelActive(ctx: ChannelHandlerContext) {
        logger.info("有人加入了！")
    }

    override fun channelInactive(ctx: ChannelHandlerContext) {
        logger.info("有人退出")
        super.channelInactive(ctx)
        ChannelMapController.removeByChannle(ctx.channel())
    }

    override fun userEventTriggered(ctx: ChannelHandlerContext, evt: Any?) {
        if (evt is IdleStateEvent) {
            if (counter >= 3) {
                // 连续丢失3个心跳包 (断开连接)
                ctx.channel()?.close()?.sync()
                ChannelMapController.removeByChannle(ctx.channel())
                logger.info("已与Client断开连接")
            } else {
                counter++
                logger.info("丢失了第 $counter 个心跳包")
            }
        }
    }

    override fun channelRead(ctx: ChannelHandlerContext, msg: Any?) {
        val protoMsg = msg as IMessage.Protocol  //解析Protocol
        val contentType = protoMsg.contentType
        if (contentType == IMessage.ContentType.HEART_BEAT) {
            counter = 0
            logger.info("收到心跳包")
        } else {
            handlerMessage(ctx, msg)
        }
    }

    private fun handlerMessage(ctx: ChannelHandlerContext, msg: IMessage.Protocol) {
        counter = 0
        val contentType = msg.contentType
        when (contentType) {
            IMessage.ContentType.Message_INFO -> {
                val message: IMessage.Message = IMessage.Message.parseFrom(msg.content)
                if (message.type == IMessage.MessageType.ALL) {
                    logger.info("收到全员广播消息: ${message.message}")
                    ChannelMapController.sendMsgToAll(ProtocolFactory.getMessage(message.message, IMessage.MessageType.ONE, ""), ctx.channel())
                } else if (message.type == IMessage.MessageType.ONE) {
                    logger.info("收到个人消息: ${message.message}")
                }
            }
            IMessage.ContentType.Register_INFO -> {
                logger.info("收到注册消息")
                val register: IMessage.Register = IMessage.Register.parseFrom(msg.content)
                val uuid = UUIDGenerator.getUUID()
                ChannelMapController.put(uuid, ctx.channel())
                ctx.writeAndFlush(ProtocolFactory.getUUIDProto(register.name, uuid))
            }

            else -> {

            }
        }
    }
}

```


## 二、客户端

#### 创建连接

```
fun start() {
        mGroup = NioEventLoopGroup()
        try {
            val b = Bootstrap()
            b.group(mGroup)
                    .channel(NioSocketChannel::class.java)
                    .remoteAddress(InetSocketAddress("172.18.157.43", 1088))
                    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
                    .handler(ProtocolPipeline())

            mChannelFuture = b.connect().awaitUninterruptibly()
            mChannelFuture!!.channel().closeFuture().sync()
        } finally {
            mGroup!!.shutdownGracefully().sync()
        }
    }
```

#### ProtocolPipeline数据处理

```
class ProtocolPipeline : ChannelInitializer<SocketChannel>() {
    override fun initChannel(ch: SocketChannel) {
        val pipeline = ch.pipeline()

        pipeline.addLast("send heartbeat", IdleStateHandler(0, 30, 0, TimeUnit.SECONDS)) //写延时30秒，表示30秒没有写操作就会触发心跳机制
        // 和服务端保持一致
        pipeline.addLast(ProtobufVarint32FrameDecoder())
        pipeline.addLast("proto decoder", ProtobufDecoder(IMessage.Protocol.getDefaultInstance()))
        pipeline.addLast(ProtobufVarint32LengthFieldPrepender())
        pipeline.addLast("proto encoder", ProtobufEncoder())
        pipeline.addLast(ClientHandler())
    }
}
```

#### ClientHandler处理逻辑

```
class ClientHandler : SimpleChannelInboundHandler<IMessage.Protocol>() {
    private val TAG = "ClientHandler"

    override fun channelActive(ctx: ChannelHandlerContext) {
        SendMsgController.setChannelHandler(ctx) // 将channel保存在一个单例中
    }

    override fun channelRead0(p0: ChannelHandlerContext?, message: IMessage.Protocol) {
        Log.e(TAG, "get form server $message")
        val contentType = message.contentType
        when (contentType) {
            IMessage.ContentType.HEART_BEAT -> {
            }
            IMessage.ContentType.Message_INFO -> {
            }

            IMessage.ContentType.Register_UUID -> {
            }
            else -> {

            }
        }
    }


    override fun userEventTriggered(ctx: ChannelHandlerContext?, evt: Any?) {
        if (evt is IdleStateEvent) {
            if (evt.state() == IdleState.WRITER_IDLE) {
                Log.d(TAG, "send heartbeat!")
                ctx?.writeAndFlush(ProtocolFactory.getHeartBeat())
            } else {
                Log.d(TAG, "其他超时：${evt.state()}")
            }
        }

        super.userEventTriggered(ctx, evt)
    }

}
```

#### 单例SendMsgController

```
object SendMsgController {

    val TAG = SendMsgController::class.java.simpleName
    var channelHandlerContext: ChannelHandlerContext? = null

    fun setChannelHandler(channelHandlerContext: ChannelHandlerContext) {
        this.channelHandlerContext = channelHandlerContext
    }

    fun sendMsg(msg: IMessage.Protocol) {
        if (channelHandlerContext != null) {
            channelHandlerContext!!.writeAndFlush(msg)
        } else {
            Log.e(TAG, "channelHandlerContext is null")
        }

    }

    fun sendMsg(msg: IMessage.Protocol, future: ChannelFutureListener) {
        if (channelHandlerContext != null) {
            channelHandlerContext!!.writeAndFlush(msg).addListener(future)
        } else {
            Log.e(TAG, "channelHandlerContext is null")
        }
    }

    fun close() {

    }
}
```

在连接建立后就将channel保存在一个单例中，之后所有channel相关的操作都可以使用这个单例。

