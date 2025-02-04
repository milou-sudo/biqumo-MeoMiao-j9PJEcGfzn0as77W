
## 前言


下文介绍的自定义协议仅作为学习示例，纯粹是玩具项目，没有实际可用性。无需过度关注和讨论其合理性


### 进行通信的双方是谁？


常见的模型


客户端\-服务器，例如HTTP协议，浏览器\<\=\>Web服务器。


中转站模型，如MQTT协议，应用服务\<\=\>中转站\<\=\>硬件客户端


对等模型，例如Thrift协议，应用服务\<\=\>应用服务。


### 通用协议如此丰富，还需要自定义协议吗？


需要。许多中间件服务在构建集群时，服务节点之间需要进行高效的内部通信。


在这种场景下，自定义协议能发挥巨大的作用：


* **去除冗余字段**：自定义协议能够减少无用字段，最大化优化通信吞吐量
* **灵活性**：自定义协议可以根据需求进行灵活扩展，支持注入优先级控制，解压缩控制等特点。


自定义协议可以减少无用字段，最大限度地优化通信吞吐量；也更加灵活，可以进行优先级控制。


例如，**Kafka** 就使用了自定义协议来满足高效的消息传递需求。


## 自定义协议设计


所谓网络协议，就是传输的报文格式，以及收发双方处理报文的规则。


报文格式做如下设计：


* 固定头部（4字节）
	+ 字节1：消息类型
		- 1\=req，2\=res, 3\=pub, 4\=sub, 5\=msg
		- 用一个字节来表示类型有点浪费了。
	+ 字节2\~字节4：消息体长度
		- 这三个字节能够表示最大值为 16777215，即最大消息体长度为 **16MB**。
* 消息体（可变长度）


规则：


1\.服务端收到req包，需返回res包


2\.服务端收到sub包，需更新订阅情况


3\.服务端收到pub包，需根据订阅情况发送msg包


## 粘拆包问题


在设计网络协议时，不可不谈粘拆包问题。


### **什么是粘包和拆包？**


这两个都是接收端在接收数据时遇到的问题，其中


* 粘包：多个数据包合并成一个包接收
* 拆包：一个数据包被拆分成多个包接收


### **为什么会出现粘包与拆包？**


根本原因就是传输层的TCP协议，是面向字节流的，它不知道数据边界。


此外，TCP根据网络情况（如最大传输单元MTU）动态调整报文大小，导致数据包的分段与合并。


从而产生粘包和拆包问题


**传输流程：**


1\.发送缓冲区：当应用层产生数据后，这些数据会首先进入Socket连接的发送缓冲区


2\.数据拆分：网卡根据缓冲区中的数据内容，将数据拆分成多个小的TCP数据报进行发送


3\.接收与重组：接收端的TCP栈会将接收到的多个TCP数据包重新组装成完整的字节流（Socket接收缓冲区）


### 案例场景


一个常见的场景是，客户端连续发送多个消息（如 100 个字符串），而服务端接收到的数据可能并不完全是 100 条。


要复现这种问题也很简单，只要客户端连续发100个字符串，检查服务端收到的数据条数。


客户端代码：连接建立后，连续发送100次字符串




```
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
//        ctx.writeAndFlush(Unpooled.copiedBuffer("Netty rocks!", CharsetUtil.UTF_8));
        new Thread(() -> {
            for (int i = 0; i < 100; i++) {
                ctx.writeAndFlush(Unpooled.copiedBuffer("Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!", CharsetUtil.UTF_8));
            }
        }).start();
    }
```


服务端代码：每收到一个包，就打印一次。




```
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf in = (ByteBuf) msg;
        System.out.println("Server Receive:"+in.toString(CharsetUtil.UTF_8));
        ctx.write(in);
    }
```


结果：仅收到两个包，同时存在粘包和拆包问题。一个Siuuuu被截断了




```
Server Receive:Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuu
Server Receive:uuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!Siuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuuu!
```


### 如何处理粘包和拆包？


处理方式由消息格式决定


* 固定长度：每条消息的长度固定，不足部分使用填充
* 特殊分隔符：每条消息的末尾添加特定的分隔符
* 消息头\+消息体：消息头长度固定，包含消息体长度信息


由于我们采用的时第三种方式，也是最复杂的一种。


处理的核心在于**消息头**，因为它携带了消息体的长度信息，是判断消息边界的关键。


**粘包的处理**


步骤如下：


1. 提取消息头：首先提取消息头，从中获取消息体的长度信息
2. 读取完整消息：根据消息体的长度，从数据流中读取完整的消息内容
3. 重复执行：重复步骤1和步骤2，直到没有更多的数据，或当前数据不足以构成完整的消息


**拆包的处理**


拆包的处理方式与粘包类似：


1. 缓存数据：如果接收到的数据不足一条完整消息，则将数据存入缓冲区。
2. 合并新数据：在接收到新数据时，判断缓冲区和新数据是否可以组成完整消息，直到消息完整为止。
3. 继续缓存：剩下的数据如果不足，则继续缓存


**代码案例**


1）客户端


根据上面的协议格式，构建消息。（这里的消息体内容是随机字符串，实际应用中通常是序列化后的POJO对象。）


连接建立后连续发送200条随机长度的消息。




```
public class EchoClientHandler extends SimpleChannelInboundHandler {


    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        super.channelInactive(ctx);
        System.out.println("断开连接");
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        new Thread(() -> {
            //连续发送200条消息
            for (int i = 0; i < 200; i++) {
                try {
                    ctx.writeAndFlush(Unpooled.copiedBuffer(buildRandomMsg()));
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }).start();

    }

    protected void channelRead0(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf) throws Exception {
        System.out.println("Client receive:"+byteBuf.toString(CharsetUtil.UTF_8));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }

    //构建消息，其中body内容为随机长度的随机字符串
    public static byte[] buildRandomMsg() throws IOException {
        int length = RandomUtil.randomInt(100, 200);
        String body = RandomUtil.randomString(length);

        System.out.println("长度："+length+"||内容："+body);

        byte type = 1;
        byte[] lengthBytes = new byte[3];
        lengthBytes[0] = (byte) (length >> 16);
        lengthBytes[1] = (byte) (length >> 8);
        lengthBytes[2] = (byte) length;
        byte[] bodyBytes = body.getBytes(CharsetUtil.UTF_8);
        return concatByteArrays(new byte[]{type}, lengthBytes, bodyBytes);
    }

    //拼接字节数组
    public static byte[] concatByteArrays(byte[]... byteArrays) throws IOException {
        // 使用 ByteArrayOutputStream 来拼接字节数组
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();

        for (byte[] array : byteArrays) {
            byteArrayOutputStream.write(array);
        }

        // 返回拼接后的字节数组
        return byteArrayOutputStream.toByteArray();
    }
}
```


2）服务端


在看代码前，先说明一下channelRead的调用流程


* Socket接收到TCP报文，将数据写入内核缓冲区
* NIO线程检测到此Socket有可读消息
* NIO线程从内核缓冲区读取消息，得到ByteBuf msg
* NIO线程调用channelRead


得到两个信息


1. msg是从缓冲区读取的，它可能包含多条完整消息 \+ 一条残缺消息。
2. msg已经从缓冲区读出，缓冲区数据已清空。对于不完整的消息需要自行缓存


下面代码是直接实现的，主要用来介绍完整的处理逻辑。


实际应用中推荐继承Netty提供的ByteToMessageDecoder，它帮你实现了缓存管理。




```
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    private static final int HEADER_LENGTH = 4; //消息头部长度
    private ByteBuf buffer = Unpooled.buffer(1024); //缓存残缺消息

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ByteBuf income = (ByteBuf) msg;

        //上一次有缓存存在，则本数据包不是消息头开头，
        if(buffer.readableBytes() > 0) {            buffer.ensureWritable(income.readableBytes()); //进行必要的扩容
            income.readBytes(buffer, income.readableBytes());

            readMsgFromBuffer(buffer);

            //剩下一点残缺消息
            if(buffer.readableBytes() > 0) {
                //保留剩下的数据，重置读索引为0
                System.out.println("缓存剩余字节："+buffer.readableBytes());
                buffer.discardReadBytes();
            } else { //刚刚好，则清空数据
                buffer.clear();
            }
        } else {
            readMsgFromBuffer(income);

            //剩下的数据全部写入缓存
            if (income.readableBytes() >0) {
                System.out.println("剩余字节:"+income.readableBytes());
                income.readBytes(buffer, income.readableBytes());
            }
        }

    }

    //从字节数组中读取完整的消息
    private void readMsgFromBuffer(ByteBuf byteBuf) {
        //剩余可读消息是否包含一个消息头
        while(byteBuf.readableBytes() >= HEADER_LENGTH) {
            byteBuf.markReaderIndex(); //由于可能读不到完整的消息，所以读之前先标记索引位置，方便重置
            //读取消息头
            byte[] headerBytes = new byte[4];
            byteBuf.readBytes(headerBytes);
            //获取类型
            int type = headerBytes[0] & 0xFF;
            //获取消息体长度
            int bodyLength = ((headerBytes[1] & 0xFF) << 16) |
                    ((headerBytes[2] & 0xFF) << 8) |
                    (headerBytes[3] & 0xFF);

            //不包含请求体
            if (byteBuf.readableBytes() < bodyLength) {
                byteBuf.resetReaderIndex(); //重置读索引到当前消息头位置
                break;
            }

            // 完整消息体已经接收，处理消息
            byte[] body = new byte[bodyLength];
            byteBuf.readBytes(body);
            System.out.println("type:"+type+"||length:"+bodyLength+"||body:"+new String(body, CharsetUtil.UTF_8));
        }
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
//        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER).addListener(ChannelFutureListener.CLOSE);
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}
```


服务端输出：服务端逐行打印出消息类型，长度，消息体。




```
...
type:1||length:175||body:0cDDAkum0F9DNwF511AKitTe2zRoSc27IjBYwgoODkXxx78xp0cowcDDNWTZ6xjCZyn6wmI2UxXLYB25TjUnOG9ZyjiZ9Jge3kbxabRjZAo0qsCYFfKMyzxApp953z1N7uDbP9rmlxeyYbYiif3y3ybtnnaAkuKFcspje6SLRnY69Nz
```


## 消息体编解码（序列化）


在经过前面粘包和拆包处理后，我们已经能够成功地从数据流中分离并组装出完整的消息。然而，在实际应用中，消息体通常需要进一步转换为对象，才能提交给上层的业务逻辑。


这是传输层的关键职责之一。


### 常见序列化方法


常见的POJO对象序列化方式包括：


**Java序列化（Serializable）**


优点：内置，无需额外依赖。


缺点：


* 性能较差，序列化和反序列化速度较慢。
* 无法跨语言使用，限制了不同语言（如Java服务端和C\+\+客户端）之间的数据交换。


**JSON**


优点：可读性好，方便调试，支持各种语言


缺点：相较于二进制格式，JSON的键（key）通常占用较多空间，大规模数据传输时，带宽开销大。


**Protocol Buffers（ProtoBuf）**优势：


* 高效的二进制序列化，体积小，序列化和反序列化速度快。
* 支持跨语言使用，适用于不同编程语言之间的通信。


### 代码案例


这里我们使用ProtoBuf。


**构建消息类**


写一个.proto文件，定义消息格式。


hello\_request.proto




```
option java_multiple_files = true;
option java_package = "protocol";
option java_outer_classname = "Request";

message HelloRequest {
    required string requestId = 1;
    optional string content = 2;
}
```


下载ProtoBuf编译工具包，protoc\-{version}\-win64\.zip




```
https://github.com/protocolbuffers/protobuf/releases
```


编译，得到Java文件




```
protoc -I=$SRC_DIR --java_out=$DST_DIR $SRC_DIR/hello_request.proto
```


引入对应版本的Jar包。（jar包版本要和protoc版本一致，否则报错）




```
https://mvnrepository.com/artifact/com.google.protobuf/protobuf-java
```


接着就可以使用类构建POJO对象和对象的编解码了。


**客户端**


其他地方不变，使用上面生成好的HelloRequest类，构建对象。通过setter塞入数据，然后通过toByteArray()得到序列化后的二进制数据。


注意：现在的length应该是整个消息体的字节数，不再是随机字符串的长度。




```
public static byte[] buildRandomMsg() throws IOException {
        int randomStrLength = RandomUtil.randomInt(100, 200);
        String msgId = UUID.randomUUID().toString();
        String content = RandomUtil.randomString(randomStrLength);
        HelloRequest request = HelloRequest.newBuilder()
                .setRequestId(msgId)
                .setContent(content)
                .build();
        byte[] bodyBytes = request.toByteArray();
        int length = bodyBytes.length;

        System.out.println("发送消息："+request.toString());

        byte type = 1;
        byte[] lengthBytes = new byte[3];
        lengthBytes[0] = (byte) (length >> 16);
        lengthBytes[1] = (byte) (length >> 8);
        lengthBytes[2] = (byte) length;

        return concatByteArrays(new byte[]{type}, lengthBytes, bodyBytes);
    }
```


**服务端**


其他地方不变，解析body的时候，使用HelloRequest.parseFrom(byte\[] bytes)进行解码，得到HellpRequest对象。




```
   //System.out.println("type:"+type+"||length:"+bodyLength+"||body:"+new String(body, CharsetUtil.UTF_8));
   if(type == 1) {
        try {
              HelloRequest request = HelloRequest.parseFrom(body);
              System.out.println("收到消息:"+request.toString());
        } catch (Exception e) {
              System.out.println("解析失败："+new String(body, CharsetUtil.UTF_8));
        }
    } else {
              System.out.println("消息类型未知："+type);
    }
```


**结果**


客户端输出




```
...
发送消息：requestId: "ca9b3e07-0662-467c-9bed-843b519c2480"
content: "q82EuHvGgMhwbHl1t0qfv4M2NCJLikxahpEc8q9ezpCWUbU9M1Oh6U6zfIOnBC50ex5BweYfZ2JB0NoLmP4hgIsNzZ8mtfFPayi8KlDWRQw3gj7ENRgxjbm4HxJgrdDNobuguc8EPQ3SccWXGTsZytLEeOHJXskiGlH4oEf"
```


服务端输出




```
....
```


```
收到消息:requestId: "ca9b3e07-0662-467c-9bed-843b519c2480"
content: "q82EuHvGgMhwbHl1t0qfv4M2NCJLikxahpEc8q9ezpCWUbU9M1Oh6U6zfIOnBC50ex5BweYfZ2JB0NoLmP4hgIsNzZ8mtfFPayi8KlDWRQw3gj7ENRgxjbm4HxJgrdDNobuguc8EPQ3SccWXGTsZytLEeOHJXskiGlH4oEf"
```


## 实现异步请求


### 结构设计


底层Socket是天然支持异步的，因为发送和接收是可以同时进行的，不会互相影响。


要实现异步请求的效果，上层API只要做到以下几点：


1. 请求发送后，不会阻塞当前执行线程
2. 响应到达后可以触发回调
3. 超时（指定时间内没有收到响应）也可以触发回调


实现方式


1. 请求接口发送请求后返回Future对象，可选择同步等待
2. 客户端保留请求和对应的callback
3. 服务端响应的时候返回请求ID
4. 客户端根据ID获取关联请求，执行callback。


![](https://img2024.cnblogs.com/blog/1313132/202501/1313132-20250106121717750-94160340.png)


首先，项目结构图如下:


1\.划线部分是废弃类


2\.【变更】解码方式修改，新增通用的MessageDecoder可供双方解码，其继承于ByteToMessageDecoder。


3\.【新增】新增HelloResponse


4\.【新增】新增通用MessageEncoder，继承于MessageToByteEncoder


![](https://img2024.cnblogs.com/blog/1313132/202501/1313132-20250107154844848-19601071.png)


### 代码实现


1\. MessageDecoder.java


相比前面直接实现的，这里不用去管理缓存。另外，这里解析好的消息会写入List，但它其实是逐个传给下一个Handler。




```
public class MessageDecoder extends ByteToMessageDecoder {
    private static final int HEADER_LENGTH = 4; //消息头部长度

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List out) throws Exception {
        // 检查是否足够的字节来读取一个消息头
        while (in.readableBytes() >= HEADER_LENGTH) {
            in.markReaderIndex(); // 标记当前读取位置，便于重置

            // 读取消息头部
            byte[] headerBytes = new byte[4];
            in.readBytes(headerBytes);

            // 获取类型
            int type = headerBytes[0] & 0xFF;
            // 获取消息体长度
            int bodyLength = ((headerBytes[1] & 0xFF) << 16) |
                    ((headerBytes[2] & 0xFF) << 8) |
                    (headerBytes[3] & 0xFF);

            // 检查缓冲区中的数据是否足够读取整个消息体
            if (in.readableBytes() < bodyLength) {
                in.resetReaderIndex(); // 重置读指针，等待更多数据
                break;
            }

            // 读取消息体
            byte[] body = new byte[bodyLength];
            in.readBytes(body);

            // 处理消息，根据消息头中的类型，解析成不同的对象
            try {
                Object msg = null;
                if(type == 1) {
                    msg = HelloRequest.parseFrom(body);
                } else if(type == 2) {
                    msg = HelloResponse.parseFrom(body);
                } else {
                    System.out.println("未知消息："+new String(body, CharsetUtil.UTF_8));
                }
                if(Objects.nonNull(msg)) {
                    out.add(msg);
                }

            } catch (Exception e) {
                System.out.println("解析失败: " + new String(body, CharsetUtil.UTF_8));
            }
        }
    }
}
```


2\.MessageEncoder.java


ProtoBuf生成的类可以调用toByteArray()序列化成字节数组。这样消息体的二进制数据就有了。


而消息头则根据消息类型和消息体长度进行构建




```
public class MessageEncoder extends MessageToByteEncoder {
    @Override
    protected void encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) throws Exception {
        if(!(msg instanceof GeneratedMessage)) {
            System.out.println("未知类型："+msg.getClass());
            return;
        }
        int type = 0;
        if(msg instanceof HelloRequest) {
            type = 1;
        } else if(msg instanceof HelloResponse) {
            type = 2;
        }
        byte[] bodyBytes = ((GeneratedMessage) msg).toByteArray();
        int length = bodyBytes.length;
        byte[] lengthBytes = new byte[3];
        lengthBytes[0] = (byte) (length >> 16);
        lengthBytes[1] = (byte) (length >> 8);
        lengthBytes[2] = (byte) length;
        out.writeByte(type);
        out.writeBytes(lengthBytes);
        out.writeBytes(bodyBytes);
    }
}
```


3\.ServerMessageHandler


服务器得到Decoder解析好的消息后，间隔一段时间（1\-4秒）发回响应内容。




```
public class ServerMessageHandler extends SimpleChannelInboundHandler {
    //异步线程
    private ScheduledExecutorService mockRequestHandler = Executors.newSingleThreadScheduledExecutor();

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
        if(msg instanceof HelloRequest) {
            System.out.println("收到消息："+msg);
            HelloRequest request = (HelloRequest) msg;

            //使用处理线程，不阻塞NIO线程
            //模拟处理请求，处理时间随机1~4秒
            mockRequestHandler.schedule(() -> {
                ctx.writeAndFlush(HelloResponse.newBuilder()
                        .setRequestId(request.getRequestId()) 
                        .setStatus(200)
                        .setData("Handled:"+request.getContent()) //增加一个前缀，表示服务器已处理
                        .build()
                );
            }, RandomUtil.randomInt(1, 4), TimeUnit.SECONDS);

        }
    }
}
```


4\.EchoServer


服务端启动类，配置Handler，启动端口监听。




```
public class EchoServer {
    private final int port;

    public EchoServer(int port) {
        this.port = port;
    }

    public void start() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(); //处理CONNECT的线程
        EventLoopGroup workerGroup = new NioEventLoopGroup(4); //Worker线程
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .localAddress(port)
                    .childHandler(new ChannelInitializer() {
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            socketChannel.pipeline()
                                    .addLast(new MessageDecoder())
                                    .addLast(new MessageEncoder())
                                    .addLast(new ServerMessageHandler());
                        }
                    });
            ChannelFuture f = b.bind().sync(); //开始监听
            System.out.println("启动监听："+port);
            f.channel().closeFuture().sync(); //阻塞直到程序退出
        } finally {
            bossGroup.shutdownGracefully().sync();
            workerGroup.shutdownGracefully().sync();
        }
    }

    public static void main(String[] args) throws Exception {
        new EchoServer(9090).start();
    }
}
```


5\.ClientMessageHandler


对接ClientApi


1\.连接建立后告知ClientApi


2\.收到响应后提交给ClientApi




```
public class ClientMessageHandler extends SimpleChannelInboundHandler {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
        if(msg instanceof HelloResponse) {
//            System.out.println("收到消息："+msg);
            //收到响应内容，则触发回调
            ClientApi clientApi = Container.getClientApi();
            if(Objects.nonNull(clientApi)) {
                clientApi.onResponse((HelloResponse) msg);
            }
        } else {
            System.out.println("未知消息："+msg);
        }
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        super.channelActive(ctx);
        
        //连接成功，告知ClientApi
        ClientApi clientApi = Container.getClientApi();
        if(Objects.nonNull(clientApi)) {
            clientApi.onConnected(ctx);
        } 

    }
    
}
```


6\.EchoClient


客户端启动类，配置编解码类和消息处理类，最后连接到目标地址。


注意：这里没有main入口，入口在Test类。




```
public class EchoClient {
    private final String host;
    private final int port;

    public EchoClient(String host, int port) {
        this.host = host;
        this.port = port;
    }

    public void start() throws Exception {
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
                    .channel(NioSocketChannel.class)
                    .remoteAddress(new InetSocketAddress(host, port))
                    .handler(new ChannelInitializer() {
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            socketChannel.pipeline()
                                    .addLast(new MessageDecoder())
                                    .addLast(new MessageEncoder())
                                    .addLast(new ClientMessageHandler());
                        }
                    });
            ChannelFuture f = b.connect().sync();
            System.out.println("开始连接");
            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully().sync();
        }
    }

}
```

 

7\.Container


很简单，就是一个静态类。用来存放ClientApi的引用




```
public class Container {
    public static ClientApi clientApi;

    public static void setClientApi(ClientApi clientApi) {
        Container.clientApi = clientApi;
    }

    public static ClientApi getClientApi() {
        return clientApi;
    }
}
```


8\.ClientApi


ClientApi负责建立连接，发送请求，回调响应。支持同步和异步两种请求方式。




```
public class ClientApi {
    private final String host;
    private final int port;

    private final Map> waitingRequests = new HashMap<>();
    private final ScheduledExecutorService timer = Executors.newSingleThreadScheduledExecutor();
    private final ReentrantLock lock = new ReentrantLock();
    private ChannelHandlerContext ctx = null;
    private CompletableFuture waitConnectionFuture;

    ClientApi(String host, int port) {
        this.host = host;
        this.port = port;
        //初始化后注册到Container中，方便其他类引用
        Container.setClientApi(this);
    }

    //连接建立后，回调Context
    public void onConnected(ChannelHandlerContext ctx) {
        lock.lock();
        try {
            if(waitConnectionFuture != null) {
                waitConnectionFuture.complete(ctx);
                waitConnectionFuture = null;
            }
        } finally {
            lock.unlock();
        }
    }

    //获取连接
    public ChannelHandlerContext getConnection() throws Exception {
        lock.lock();
        try {
            //连接已存在，直接发
            if(ctx != null) {
                return ctx;
            }
            //连接不存在，建立连接
            waitConnectionFuture = new CompletableFuture<>();
            new Thread(()-> {
                try {
                    new EchoClient(host, port).start(); //这个会阻塞当前线程，所以另启线程
                } catch (Exception e) {
                    e.printStackTrace();
                    //连接断开，也触发回调
                    if(waitConnectionFuture != null) {
                        waitConnectionFuture.completeExceptionally(e);
                    }
                }
            }).start();
        } finally {
            lock.unlock();
        }

        //get()等待之前，需要释放锁
        ctx = waitConnectionFuture.get();
        return ctx;
    }

    public CompletableFuture baseRequest(HelloRequest request) {
        //先注册回调
        CompletableFuture future = new CompletableFuture<>();
        addToMap(request.getRequestId(), future);
        //再发送请求
        try {
            getConnection().writeAndFlush(request);
            System.out.println("发出消息："+request);
        } catch (Exception e) {
            removeFromMap(request.getRequestId());
            throw new RuntimeException("请求错误："+e);
        }
        //添加超时，防止服务器没响应，造成泄露
        timer.schedule(() -> timeout(request.getRequestId()), 5, TimeUnit.SECONDS);
        return future;
    }

    //同步请求
    public HelloResponse sendRequest(HelloRequest request) throws Exception {
        return this.baseRequest(request).get();
    }

    //异步请求
    public void sendRequestAsync(HelloRequest request, Function callback) {
        this.baseRequest(request).thenApply(callback);
    }

    private void addToMap(String requestId, CompletableFuture future) {
        lock.lock();
        try {
            waitingRequests.put(requestId, future);
        } finally {
            lock.unlock();
        }
    }

    private void removeFromMap(String requestId) {
        lock.lock();
        try {
            waitingRequests.remove(requestId);
        } finally {
            lock.unlock();
        }
    }

    public void timeout(String requestId) {
        lock.lock();
        try {
            CompletableFuture future1 = waitingRequests.get(requestId);
            if(Objects.nonNull(future1)) {
                future1.completeExceptionally(new RuntimeException("请求超时"));
            }
        } finally {
            lock.unlock();
        }
    }

    public void onResponse(HelloResponse response) {
        lock.lock();
        try {
            //收到响应后，根据请求ID获取回调。
            CompletableFuture future1 = waitingRequests.get(response.getRequestId());
            if(Objects.nonNull(future1)) {
                future1.complete(response);
            }
        } finally {
            lock.unlock();
        }
    }

    public void close() {
        if(ctx != null) {
            ctx.close();
        }
    }

}
```


9\.测试类


测试类，使用ClientApi，发送请求。


下面代码分别是同步发送5个请求和异步发送5个请求




```
public class Test {
    public static void main(String[] args) {
        ClientApi clientApi = new ClientApi("127.0.0.1", 9090);
        for (int i = 0; i < 5; i++) {
            try {
                HelloRequest request = buildHelloRequest();

                //同步请求，收到响应后才会发下一个请求
                HelloResponse response = clientApi.sendRequest(request);
                System.out.println("同步收到：" + response);
                //异步请求，发送完成即可发送下一个请求
//                clientApi.sendRequestAsync(request, response2 -> {
//                    System.out.println("异步收到："+response2);
//                    return true;
//                });

            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    //构建请求
    public static HelloRequest buildHelloRequest() {
        int randomStrLength = RandomUtil.randomInt(100, 200);
        String msgId = UUID.randomUUID().toString();
        String content = RandomUtil.randomString(randomStrLength);
        HelloRequest request = HelloRequest.newBuilder()
                .setRequestId(msgId)
                .setContent(content)
                .build();
        return request;
    }
}
```


### 运行效果


**1）同步请求**


可以看到，只有收到前一个请求的响应后，才会发下一个请求




```
开始连接
发出消息：requestId: "7ac9008f-8532-4740-bb1f-f5fe2d60fd62"
content: "zYU1oWKiJdLX2K87J0306Xdeq0BmlkijpD1p6t831A3b2fEP0JCZT9QqaT7oWnCpoKMIpZRZ2gJWbSvEIhxs56m8Zr0YhsvaPdQj1x8f4Q1HuLh"

同步收到：requestId: "7ac9008f-8532-4740-bb1f-f5fe2d60fd62"
status: 200
data: "Handled:zYU1oWKiJdLX2K87J0306Xdeq0BmlkijpD1p6t831A3b2fEP0JCZT9QqaT7oWnCpoKMIpZRZ2gJWbSvEIhxs56m8Zr0YhsvaPdQj1x8f4Q1HuLh"

发出消息：requestId: "d74a8a83-28dd-4e40-9956-dfdd3d890bda"
content: "83LzAdgxQ8MYz1CmzRXfEn3ibz9WqiJHcQCRtkE4dCEZUnW44UNGfKtHR0nBNE7al7PdvdexEDDTth3Aoy6mati8TVOP54xRUT26MAaV0DP0UhU7V7QWaMfiKwV2oVpxMPqg2thwNdd5WCG53"

同步收到：requestId: "d74a8a83-28dd-4e40-9956-dfdd3d890bda"
status: 200
data: "Handled:83LzAdgxQ8MYz1CmzRXfEn3ibz9WqiJHcQCRtkE4dCEZUnW44UNGfKtHR0nBNE7al7PdvdexEDDTth3Aoy6mati8TVOP54xRUT26MAaV0DP0UhU7V7QWaMfiKwV2oVpxMPqg2thwNdd5WCG53"

发出消息：requestId: "e47f4135-dc92-4a25-9fed-ca7b4ced41f5"
content: "1St23ktz7nhXcICb0Yqo3QoLNRoeKJ4V4jKcaO8psUZRXnhXtIUG2WZC7d0TCbS221pUxqTlTDojSbtQvCzjIcL3JTgPkBg46rk8uRIpX3yvx0RHmZhwjBkpPoCTtWfTzk6r5SK0SK7g0QAjQacULIDXS5K1Z1U9q"

同步收到：requestId: "e47f4135-dc92-4a25-9fed-ca7b4ced41f5"
status: 200
data: "Handled:1St23ktz7nhXcICb0Yqo3QoLNRoeKJ4V4jKcaO8psUZRXnhXtIUG2WZC7d0TCbS221pUxqTlTDojSbtQvCzjIcL3JTgPkBg46rk8uRIpX3yvx0RHmZhwjBkpPoCTtWfTzk6r5SK0SK7g0QAjQacULIDXS5K1Z1U9q"

发出消息：requestId: "5122e929-be13-488e-b3ea-6e5acf7ebbbc"
content: "pj9sWOqhFGzplbUNieLOOzMKzSUEumgd2rMzR1cO4GwrmcXHb5vma32LbuHVQl8tkigKMHk9HCKM9xnUAdbCqopeTzbo0ixQkGzclud78hVFTV4PM2qYZDeWMBRDMrUXOJS0sCIxgyGFudz7XUGfJNSuJjio8dch8JPDRmHkZsABRxobZeafxiqGT"

同步收到：requestId: "5122e929-be13-488e-b3ea-6e5acf7ebbbc"
status: 200
data: "Handled:pj9sWOqhFGzplbUNieLOOzMKzSUEumgd2rMzR1cO4GwrmcXHb5vma32LbuHVQl8tkigKMHk9HCKM9xnUAdbCqopeTzbo0ixQkGzclud78hVFTV4PM2qYZDeWMBRDMrUXOJS0sCIxgyGFudz7XUGfJNSuJjio8dch8JPDRmHkZsABRxobZeafxiqGT"

发出消息：requestId: "0cd23413-303d-4414-8cf1-20bd46a691d2"
content: "YWnQxVh0Z4yLPQeM6q3aiz7JYD6fEqZHFiE45KgebiZlwW7DlYnhZTZ7sG4rZqrvsHXQ65PCoN569kfJMHuJFp9kqnlBKeJ1iawYBFQfI5EqspxsaB7vkMuC1vA5ula2jwagoQoU6Yk0gi0EKEX1fpLIYvtYdMqTWjAfLFqc5s8yjPr0G"

同步收到：requestId: "0cd23413-303d-4414-8cf1-20bd46a691d2"
status: 200
data: "Handled:YWnQxVh0Z4yLPQeM6q3aiz7JYD6fEqZHFiE45KgebiZlwW7DlYnhZTZ7sG4rZqrvsHXQ65PCoN569kfJMHuJFp9kqnlBKeJ1iawYBFQfI5EqspxsaB7vkMuC1vA5ula2jwagoQoU6Yk0gi0EKEX1fpLIYvtYdMqTWjAfLFqc5s8yjPr0G"
```


**2）异步请求**


可以看到5个请求会直接发出，不会等待响应。响应顺序也跟请求顺序不一样。




```
开始连接
发出消息：requestId: "2096b54f-825c-4fdf-817e-97b3a4b99fb2"
content: "ckp1dXcnYItdXafrURU6gJ9b5qW19rqPKLB22qlA2sHRfkSZEpmT4qi0TOAaDbM43v62svI1K6IccnlJtCjlpcu8RAdpfuO5hNBWsXpOaSGUgY4loLNlNFIDE5o7juhfCD2skV2"

发出消息：requestId: "e8c501e0-a4e2-4972-a6be-f92539141252"
content: "ZpOz1YK6e9VOrX69xqNovUXfapY6Ito7z6LlsM6o1Vzeo1hibzvOcxAzYD8hIsOFvGAqk024XbL7yidlgPk4F9GId6ydRxzjjNdg8csxG9FdBXzzr6xuESJ"

发出消息：requestId: "2c56fc02-5fd8-4c17-bca5-ab0def66493c"
content: "FzoUaLvHxA0Tm7eU4GL9bIE6mEMNRIUSZILPLiREPXGfhcgoasYd1W5jEfAooE697LQr2DMw6fBdwEqHunQcl6doxrnxSQAZorHztHvyKXAFmbnF3aDkYgO82HaHGXuC"

发出消息：requestId: "477cbb82-77c0-4270-b161-555da7b6a5e2"
content: "Ow81y5qPraVcobOiZ6sCH72jJGNC0784ox5crQYP5fZ6CXoWphRdC9WW4NKSiChbci6aGutnWJbO1HlpR0FV4m9qahbWGkFI0Zr2uvMbuaj8SPpH6X"

发出消息：requestId: "98db5068-1a85-455f-8fea-4b6c8a562776"
content: "nWnNIaJFt1otie04SWoaoN08f2BOuTMyRbtFuEhj0LiYilRjeKswzqrbKlze30ZBFNIuvEz6P97rP9lM5bkuDYLv1QuKOd1wctfeF9K2RbKh6hvOfgHE5wl2xUk0B6nBFK5fI1sdj3hhoiPLApQZjGzFaSHZGVtLdM4yPBC6BhmsNCPkAo2AxcQ0iZuVEHkihs"

异步收到：requestId: "2096b54f-825c-4fdf-817e-97b3a4b99fb2"
status: 200
data: "Handled:ckp1dXcnYItdXafrURU6gJ9b5qW19rqPKLB22qlA2sHRfkSZEpmT4qi0TOAaDbM43v62svI1K6IccnlJtCjlpcu8RAdpfuO5hNBWsXpOaSGUgY4loLNlNFIDE5o7juhfCD2skV2"

异步收到：requestId: "e8c501e0-a4e2-4972-a6be-f92539141252"
status: 200
data: "Handled:ZpOz1YK6e9VOrX69xqNovUXfapY6Ito7z6LlsM6o1Vzeo1hibzvOcxAzYD8hIsOFvGAqk024XbL7yidlgPk4F9GId6ydRxzjjNdg8csxG9FdBXzzr6xuESJ"

异步收到：requestId: "477cbb82-77c0-4270-b161-555da7b6a5e2"
status: 200
data: "Handled:Ow81y5qPraVcobOiZ6sCH72jJGNC0784ox5crQYP5fZ6CXoWphRdC9WW4NKSiChbci6aGutnWJbO1HlpR0FV4m9qahbWGkFI0Zr2uvMbuaj8SPpH6X"

异步收到：requestId: "2c56fc02-5fd8-4c17-bca5-ab0def66493c"
status: 200
data: "Handled:FzoUaLvHxA0Tm7eU4GL9bIE6mEMNRIUSZILPLiREPXGfhcgoasYd1W5jEfAooE697LQr2DMw6fBdwEqHunQcl6doxrnxSQAZorHztHvyKXAFmbnF3aDkYgO82HaHGXuC"

异步收到：requestId: "98db5068-1a85-455f-8fea-4b6c8a562776"
status: 200
data: "Handled:nWnNIaJFt1otie04SWoaoN08f2BOuTMyRbtFuEhj0LiYilRjeKswzqrbKlze30ZBFNIuvEz6P97rP9lM5bkuDYLv1QuKOd1wctfeF9K2RbKh6hvOfgHE5wl2xUk0B6nBFK5fI1sdj3hhoiPLApQZjGzFaSHZGVtLdM4yPBC6BhmsNCPkAo2AxcQ0iZuVEHkihs"
```


 


## 实现订阅发布


//TBD


 


 本博客参考[slowerssr加速器](https://slowerss.com)。转载请注明出处！
