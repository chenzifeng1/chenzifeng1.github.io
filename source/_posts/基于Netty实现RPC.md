---
title: 基于Netty实现RPC
date: 2020-08-18 16:00:04
tags: RPC，Netty
---

## RPC进阶之路（三）

## Netty协议

源码阅读

- Netty-4.1.x :  [http://docs.52im.net/extend/docs/src/netty4_1/](http://docs.52im.net/extend/docs/src/netty4_1/)
- Netty-4.0.x:  [http://docs.52im.net/extend/docs/src/netty4/](http://docs.52im.net/extend/docs/src/netty4/)
- Netty-3.x   :  [http://docs.52im.net/extend/docs/src/netty3/](http://docs.52im.net/extend/docs/src/netty3/)

Netty在线API文档

- Netty-4.1.x API文档(在线版)：[http://docs.52im.net/extend/docs/api/netty4_1/](http://docs.52im.net/extend/docs/api/netty4_1/)
- Netty-4.0.x API文档(在线版)：[http://docs.52im.net/extend/docs/api/netty4/](http://docs.52im.net/extend/docs/api/netty4/)
- Netty-3.x API文档(在线版)：[http://docs.52im.net/extend/docs/api/netty3/](http://docs.52im.net/extend/docs/api/netty3/)

服务器在处理响应的设计模式有两种：

- 线程驱动：同步阻塞就是线程驱动模式，即一个线程处理一个客户端建立的socket
- 事件驱动：I/O复用模型应该时事件驱动的。参考观察者模式，建立一个事件池，通过单线程监听事件池是否有待处理的事件，如果有的话，线程将事件取出进行处理。

注意区分Java的NIO包与非阻塞I/O模型之间的区别。NIO包是java自带的基于异步非阻塞I/O模型(I/O多路复用模型)所实现的一个工具包；而NIO模型则是非阻塞I/O模型，实在传统阻塞I/O模型的基础上将socket设为非阻塞实现的。

## Netty核心组件

我们可以查看Netty依赖包的项目树来查看Netty都包括那些组件

![Netty包结构](https://github.com/chenzifeng1/chenzifeng1.github.io/blob/master/picture/Netty-%E5%8C%85%E7%BB%93%E6%9E%84.png?raw=true)

- **Bootstrap和ServerBootstrap:** Netty应用程序通过设置Bootstrap的引导类来完成，该类提供了一个用于应用程序网络层配置的容器，服务端使用的是ServerBootstrap，客户端使用的是Bootstrap。

    ![Netty启动器类](https://github.com/chenzifeng1/chenzifeng1.github.io/blob/master/picture/Netty-%E5%90%AF%E5%8A%A8%E7%B1%BB.png?raw=true)

- **Channel:**Netty 中的接口 Channel 定义了与 socket 丰富交互的操作集：bind, close, config, connect, isActive, isOpen, isWritable, read, write 等等。
- **ChannelHandler:**ChannelHandler 支持很多协议，并且提供用于数据处理的容器，ChannelHandler由特定事件触发， 常用的一个接口是ChannelInboundHandler，该类型处理入站读数据（socket读事件）
- **ChannelPipeline:**按照我的理解，Netty由于采用了I/O复用模型，使单线程更好的处理Socket的操作，它会将多个ChannelHandler组成一个链式结构，每当一个channel中有Socket的事件需要处理时，就从ChannelHandler的处理链中选取一个ChannelHandler进行处理。为了更好的管理ChannelHandler链，我们需要ChannelPipiline提供一个容器在管理ChannelHandler，容器可以根据ChannelHandler的类型（处理Socket的读、写）分类管理。同时提供API来管理沿着链入站和出站事件的流动。每个 Channel 都有自己的ChannelPipeline，当 Channel 创建时自动创建的

    ![ChannelPipeline模型](https://github.com/chenzifeng1/chenzifeng1.github.io/blob/master/picture/Netty-ChannelPipeline.PNG?raw=true)

- **EventLoop:**处理channel的I/O操作，一个EventLoop会处理多个Channel中的I/O操作，一个EventLoopGroup通常会包括多个EventLoop，并且会提供一个迭代来检索。
- **ChannelFuture:**由于Netty的所有I/O操作都是异步的，因此可能无法立即返回结果，为了获取I/O操作的结果，Netty定义了ChannelFuture来获取结果。

## Netty实现RPC

### 协议处理

在这一部分，我们需要定义三部分：

1. 序列化接口及实现
2. 编解码器
3. 请求/响应参数体
- **序列化接口及实现**

    在网络层传输中，我们的数据是以字节形式进行传输的。

    因此在接口中我们需要定义两种方法：一个是字节转Byte的方法，一个是Byte转字节的方法。

    ```java
    public interface Serializer {

        /**
         * 将对象序列化成字节数组
         * @param object
         * @return
         * @throws IOException
         */
        byte[] serialize(Object object) throws IOException;

        /**
         * 将字节数组反序列化成对象
         * @param bytes
         * @param <T>
         * @return
         * @throws IOException
         */
        <T> T deserialize(Class<T> clazz,byte[] bytes) throws IOException;
        
    }
    ```

    定义好接口之后，我们需要定义其实现类,实现类的方式可以多种，这里我们选择JSON自带的对象转byte和 byte转对象的方法。之所以定义接口，是因为如果我们想更换序列化方法，只需要重写一个实现序列化接口的类即可，不需要更改太多代码。面向接口编程，开放封闭原则。

    ```java
    public class SerializerImpl implements Serializer {
        @Override
        public byte[] serialize(Object object) {
            return JSON.toJSONBytes(object);
        }

        @Override
        public <T> T deserialize(Class<T> clazz,byte[] bytes)  {
            return JSON.parseObject(bytes,clazz);
        }
    }
    ```

- **编解码器**

    当我们定义好序列化接口之后就可以将信息进行编码与解码了，在这里我们可以规定信息传输的协议：在编码器中，我们如何将拼接信息，信息头包括什么、信息体如何、校验码加在哪里等等；在解码器中，我们根据协议将接收的信息解码，获取信息体，信息头中的数据，可以对校验码进行校验，查看数据在传输过程中是否有丢包等问题。还可以通过策略模式调用不同的信息编解码策略，这些是可以做的，但是由于本篇内容仅作入门使用，故只做对象转byte与byte转对象。

    编码器：

    ```java
    public class RpcEncoder extends MessageToByteEncoder {
        private Class<?> clazz;
        private Serializer serializer;

        public RpcEncoder(Class<?> clazz, Serializer serializer) {
            this.clazz = clazz;
            this.serializer = serializer;
        }

        @Override
        protected void encode(ChannelHandlerContext channelHandlerContext, Object o, ByteBuf byteBuf) throws Exception {
            //首先判断一下初始化的编码器的信息类型是否一致
            if (clazz!=null && clazz.isInstance(o) ){
                //将信息编码
                byte[] bytes = serializer.serialize(o);
                //字节流中写入字节长度
                byteBuf.writeInt(bytes.length);
                //写入信息
                byteBuf.writeBytes(bytes);
            }
        }
    }
    ```

    解码器：

    ```java

    public class RpcDecoder extends ByteToMessageDecoder {
        private Class clazz;
        private Serializer serializer;

        public RpcDecoder(Class clazz, Serializer serializer) {
            this.clazz = clazz;
            this.serializer = serializer;
        }

        @Override
        protected void decode(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf, List<Object> list) throws Exception {
            //因为编码时写入了一个int类型的数据，因此byteBuf里面的可读字节一定大于4，如果小于4则是无数据
            if (byteBuf.readableBytes()<4){
                return;
            }
            //标记当前读的位置
            byteBuf.markReaderIndex();
            int dataLength = byteBuf.readInt();
            //如果当前可读字节数小于数据长度，重置当前读的位置
            if (byteBuf.readableBytes()<dataLength){
                byteBuf.resetReaderIndex();
                return;
            }
            //将byteBuf中的数据读到data中
            byte[] data = new byte[dataLength];
            byteBuf.readBytes(data);
            list.add(data);
        }
    }
    ```

- **请求/响应参数体**

    我们通过定义请求、响应的参数体来对我们要传输的数据进行包装。

    请求体：

    ```java
    @Data
    @ToString
    public class RPCRequest {
        /**
         * 请求对象Id
         */
        private String requestId;
        /**
         * 请求对象的类名
         */
        private String className;
        /**
         * 请求方法
         */
        private String method;
        /**
         * 参数类型
         */
        private Class<?>[] parameterType;
        /**
         * 参数
         */
        private Object[] parameters;

    }
    ```

    响应体：

    ```java
    @Data
    @ToString
    public class RPCResponse {
        /**
         * 响应id ：请求Id和响应Id要保持一致，因为客户端和服务端只有通过一致的Id才能确定调用和
    		 * 响应的是同一个方法
         */
        private String requestId;
        /**
         * 错误信息
         */
        private String errorInfo;
        /**
         * 响应结果
         */
        private Object result;
    }
    ```

    请求与响应体的定义成这样是为了之后利用反射更灵活的获得对象实体。回归框架目的，我们使用Netty开发一个RPC框架，是为了完成远程过程调用。那么**如何传参，如何定位到服务端的服务方法**是框架必须考虑的事情**。**于是客户端和服务端需要制定共同遵守的协议，这样客户端向服务端发起过程方法调用请求时，服务端根据协议解析客户端的数据，就能知道具体调用哪个方法。其中**RPCRequest**的**requestId**，即请求对象ID是为了**验证服务器请求和响应是否匹配。**

### 客户端

Netty客户端的实现注意以下几点：

- 编写启动方法，指定传输使用Channel
- 指定ChannelHandler，对网络传输中的数据进行读写处理
- 添加编解码器
- 添加失败重试机制
- 添加发送请求消息的方法

代码实现：

```java
@Slf4j
public class NettyClient {
    /**
		 * 事件处理器集合：每个I/O操作都交给EventLoop来处理，
		 * 而EventLoopGroup里面通常包括大于一个的EventLoop
		 */
    private EventLoopGroup eventLoopGroup;
    //定义了与Socket进行交互的操作集
    private Channel channel;
    //客户端处理器
    private ClientHandler clientHandler;
    //ip地址
    private String host;
    //端口
    private Integer port;
    //最大重试次数
    private static final int MAX_RETRY = 5;

    public NettyClient(String host, Integer port) {
        this.host = host;
        this.port = port;
    }

    public void connect() {
				//ClientHandler为Channel链提供了容器，以及驱动事件沿Channel链进出的API
        clientHandler = new ClientHandler();
        eventLoopGroup = new NioEventLoopGroup();
        //启动类
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(eventLoopGroup)
                //指定传输用的Channel
                .channel(NioSocketChannel.class)
                .option(ChannelOption.SO_KEEPALIVE, true)
                .option(ChannelOption.TCP_NODELAY, true)
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 500)
                .handler(new ChannelInitializer<SocketChannel>() {
	                    @Override
	                    protected void initChannel(SocketChannel socketChannel) throws Exception {
	                        ChannelPipeline channelPipeline = socketChannel.pipeline();
	                        //添加编码器
	                        channelPipeline.addLast(new RpcEncoder(RPCRequest.class, new SerializerImpl()));
	                        //添加解码器
	                        channelPipeline.addLast(new RpcDecoder(RPCResponse.class, new SerializerImpl()));
	                        //请求处理类
	                        channelPipeline.addLast(clientHandler);
	                    }
	                });
        connect(bootstrap, host, port, MAX_RETRY);
    }

    /**
     * 失败重连机制
     *
     * @param bootstrap
     * @param host
     * @param port
     * @param retry
     */
    private void connect(Bootstrap bootstrap, String host, int port, int retry) {
        ChannelFuture channelFuture = bootstrap.connect(host, port).addListener(future -> {
            if (future.isSuccess()) {
                log.info("连接成功");
            } else if (retry == 0) {
                log.error("重连失败");
            } else {
                //重连次数
                int order = (MAX_RETRY - retry) + 1;
                //本次重连间隔
                int delay = 1 << order;
                log.error("第 {} 次重连失败，时间：{}",order,new Date());
                //定时任务
                bootstrap.config().group().schedule( ()->connect(bootstrap,host,port,retry-1),delay, TimeUnit.SECONDS);
            }
        });
        channel = channelFuture.channel();
    }

    /**
     * 发送数据：使用channel将请求发送之后进入等待状态，如果响应来了会被唤醒，唤醒时说明ClientHandler响应对象中已经有请求对应的响应了，使用请求Id获取该响应即可。
     * @param rpcRequest
     * @return
     */
    public RPCResponse send(final RPCRequest rpcRequest){
       try {
           channel.writeAndFlush(rpcRequest).await();
       }catch (InterruptedException e){
           log.error("发送数据失败",e);
       }
       return clientHandler.getRPCResponse(rpcRequest.getRequestId());
    }

    @PreDestroy
    public void close(){
        //关闭通道，同步不可中断
        eventLoopGroup.shutdownGracefully();
        channel.closeFuture().syncUninterruptibly();

    }
}
```

上面的代码主要实现客户端连接服务端的相关操作，我们可以看出客户端在连接服务端时做了几个操作：

1. 创建并设置客户端的启动类`Bootstrap`，设置包括I/O事件处理集`EventLoopGroup`、操作、指定传输用的channel以及I/O处理器，初始化`channel`。
2. 连接服务端，设置失败重连机制，在一次连接失败时进行下次连接尝试

客户端的代码更像是在配置Netty通讯的客户端环境，具体通信细节交给了代理来做，在客户端代码中，我们定义了一个`ClientHandler`的对象，该类继承了`ChannelDuplexHandler`(ChannelHandler的一个实现类)。在`ClientHandler`类里面，我们定义了关于客户端数据读写的方法，只不过在`NettyClient`里面我们只在ChannelPipeLine将`ClientHandler`设置为处理器。具体调用的位置是在代理类。

```java
public class ClientHandler extends ChannelDuplexHandler {

    /**
     * Map维护请求对象ID和响应结果的映射关系
     */
    private final Map<String ,DefaultFuture> futureMap = new ConcurrentHashMap<>();

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        /* 猜测：此处应该是建立响应结果与请求对象Id的映射关系，传过来的两个参数：
        * ChannelHandlerContext是一个接口
        * msg是传过来的响应数据
        * 验证猜测：猜测错误，此处是将传过来的msg封装为DefaultFuture的处理过程
        */
        if (msg instanceof RPCResponse){
            RPCResponse rpcResponse = (RPCResponse) msg;
            //Map已经维护好了对应的映射关系
            DefaultFuture defaultFuture = futureMap.get(rpcResponse.getResponseId());
            // 异步响应？
            defaultFuture.setRpcResponse(rpcResponse);
        }

        super.channelRead(ctx,msg);
    }

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        if (msg instanceof RPCRequest){
            RPCRequest rpcRequest = (RPCRequest) msg;
            //在发送请求之前，首先在客户端维护一个请求的Id,并关联一个DefaultFuture的实例
            futureMap.put(rpcRequest.getRequestId(),new DefaultFuture());
        }
        super.write(ctx,msg, promise);
    }

    /**
     * 获取响应数据
     * @param responseId
     * @return
     */
    public RPCResponse getRPCResponse(String responseId){
        try {
            DefaultFuture defaultFuture = futureMap.get(responseId);
            return defaultFuture.getRpcResponse(10);
        }finally {
            //获取响应之后移除对应关系
            futureMap.remove(responseId);
        }
    }
}
```

在这个类中，我们定义了一个Map对象来维护请求对象Id和响应数据的映射关系，map的key是请求对象的Id,value是响应对象的接收类。

- 当我们请求服务时，也就是进行**写操作时**，我们会将请求对象的Id以及一个新的DefaultFuture对象作为一个键值对存入`futureMap`中，然后调用父类的write方法，将内容写到Socket中去。
- 当我们得到服务端的响应时，也就是进行**读操作时**，我们会将消息转成`RPCResponse`的对象，再封装到DefaultFuture中。

我们可以看一下`DefaultFuture`这个响应对象的接收类如何定义的

```java
public class DefaultFuture {
    private RPCResponse rpcResponse;
    private volatile Boolean isSuccess = false;
    private final Object object = new Object();

    /**
     * 通过wait()和notify()方法来实现异步调用
     * @param timeOut
     * @return
     */
    public RPCResponse getRpcResponse(int timeOut) {
        synchronized (object){
            while(!isSuccess){
                try {
                    object.wait(timeOut);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
        return rpcResponse;
    }

    public void setRpcResponse(RPCResponse rpcResponse) {
        if (isSuccess){
            return;
        }
        synchronized (object){
            this.rpcResponse = rpcResponse;
            this.isSuccess = true;
            object.notify();
        }

    }
}
```

DefaultFuture类中有一个标识是否接收到数据的属性 `private volatile Boolean isSuccess = false;` 还有一个作为锁对象的属性  `private final Object object = new Object();` 。

由于Netty的I/O模型是异步非阻塞的，所以不管是在服务端还是客户端，工作线程都不会一直阻塞着等待socket的读写操作。在这里，我们通过对object加同步锁，来保证每次只有一个线程可以对同一个响应对象进行操作。标志属性`isSuccess` 一开始设为`false`，这样就算线程尝试读取响应对象，会被阻塞。当响应数据被set的时候，我们会将标志属性改为`true` ，这样当线程再次尝试获取响应对象的时候就可以进入循环。

我们假设有两个线程，一个进行读操作，一个进行写操作：写操作的线程先拿到object的同步锁，那读操作的线程进入循环，被阻塞。写线程写完数据之后，设置好标志属性，调用`notify()` 方法通知被阻塞的读线程，由于此时`isSuccess`是true,因此循环结束，返回数据。由此看Netty的异步也不是纯粹的异步，也有线程被阻塞等待的过程。（等待验证）

### 服务端

Netty服务端的实现跟客户端的实现差不多，只不过要注意的是，当对请求进行解码过后，需要通过代理的方式调用本地函数。

```java
@Slf4j
public class NettyServer implements InitializingBean {
    /**
     * 负责处理客户端连接的线程池
     */
    private EventLoopGroup boss = null;
    /**
     * 负责处理读写操作的线程池
     */
    private EventLoopGroup worker = null;
    /**
     * 服务处理器
     */
    @Autowired
    private ServerHandler serverHandler;

    @Override
    public void afterPropertiesSet() throws Exception {
        //可以使用Eureka或者Zookeeper做注册中心，暂不处理
//        ServiceRegistry registry  =new ZKServiceRegistry();\
        ArrayList<Class<?>> categories = new ArrayList<>();
        ServiceRegistry registry = new ServiceRegistry(categories.iterator());
        start(registry);
    }

    public void start(ServiceRegistry registry) throws Exception {
        //建立负责客户端连接的线程池
        boss = new NioEventLoopGroup();
        //建立负责读写操作的线程池
        worker = new NioEventLoopGroup();
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap.group(boss, worker);
        serverBootstrap.channel(NioServerSocketChannel.class);
        serverBootstrap.option(NioChannelOption.SO_BACKLOG, 1024);
        serverBootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel socketChannel) throws Exception {
                ChannelPipeline channelPipeline = socketChannel.pipeline();
                //设置编码器
                channelPipeline.addLast(new RpcEncoder(RPCRequest.class, new SerializerImpl()));
                //设置解码器
                channelPipeline.addLast(new RpcDecoder(RPCResponse.class, new SerializerImpl()));
                //设置处理器
                channelPipeline.addLast(serverHandler);
            }
        });
        bind(serverBootstrap, 8888);
    }

    /**
     * @param serverBootstrap
     * @param port
     */
    public void bind(final ServerBootstrap serverBootstrap, int port) {
        serverBootstrap.bind(port).addListener(future -> {
            if (future.isSuccess()) {
                log.info("端口绑定成功 {}", port);
            } else {
                log.error("端口绑定失败{}", port);
                bind(serverBootstrap, port + 1);
            }
        });
    }

    @PreDestroy
    public void destory() throws InterruptedException{
        boss.shutdownGracefully().sync();
        worker.shutdownGracefully().sync();
        log.info("关闭Netty");
    }
}
```

之前在介绍组件的时候就介绍过`EventLoopGroup` ，这个组件相当于线程池，存放`EevntLoop` ,`EevntLoop` 可以认为是负责处理socket读写操作的线程。但是在I/O复用模型中，只有一个线程来负责监听所有的I/Osocket的事件，这里有点不理解，还需要在学习和验证。另外服务端的启动类是`ServerBootstrap`，启动方法中初始化了负责处理客户端连接的线程池`boss`和负责处理I/O的线程池`worker` ，并且配置了启动类`ServerBootstrap` 。

`EevntLoopGroup`  是一个接口，定义了注册方法以及提供用于遍历的API.

```java
public interface EventLoopGroup extends EventExecutorGroup {
    EventLoop next();

    ChannelFuture register(Channel var1);

    ChannelFuture register(ChannelPromise var1);

    /** @deprecated */
    @Deprecated
    ChannelFuture register(Channel var1, ChannelPromise var2);
}
```

与客户端代码相同的是，Server端代码一样需要一个`ChannelHandler` ，不过两者的ChannelHandler所做的工作不一样而已。下面看看Server端的ChannelHandler：

```java
@Slf4j
public class ServerHandler extends SimpleChannelInboundHandler<RPCRequest> implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    protected void channelRead0(ChannelHandlerContext channelHandlerContext, RPCRequest rpcRequest) throws Exception {
        RPCResponse rpcResponse = new RPCResponse();
        rpcResponse.setRequestId(rpcRequest.getRequestId());
        try {
            //handler是服务端通过反射根据RPCRequest对象中的信息调用响应的方法。
            Object handler = handler(rpcRequest);
            log.info("返回结果：{}",handler);
            rpcResponse.setResult(handler);
        }catch (Throwable throwable){
            rpcResponse.setErrorInfo(throwable.getMessage());
            throwable.printStackTrace();
        }
        channelHandlerContext.writeAndFlush(rpcResponse);
    }

    /**
     * 服务端使用代理处理请求数据
     * 因为不知道请求数据的类型和属性
     * RPCRequest里面封装了关于请求数据Class类型和属性及方法的信息
     * @param request
     * @return
     */
    private Object handler(RPCRequest request) throws ClassNotFoundException,InvocationTargetException{
        //利用反射获取请求数据
        Class<?> clazz = Class.forName(request.getClassName());
        Object serverBean = applicationContext.getBean(clazz);
        log.info("ServerBean: {}",serverBean);
        Class<?> serverClass = serverBean.getClass();
        log.info("ServerClass：{}",serverClass);
        String methodName = request.getMethod();

        Class<?>[] parametersTypes = request.getParameterType();
        Object[] parameters = request.getParameters();

        //使用CGLIB Reflect
        FastClass fastClass = FastClass.create(serverClass);
        FastMethod fastMethod = fastClass.getMethod(methodName,parametersTypes);
        log.info("开始调用CGLIB动态代理执行服务端方法...");
        //调用方法
        return fastMethod.invoke(serverBean, parameters);
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

}
```

服务器的ChannelHandler负责把客户端传过来的RPCRequest对象利用反射拿到对应的类对象，并调用响应的方法。RPCRequest对象的属性包括需要调用的类名，方法名，参数名和参数类型，服务端`ChannelHandler`的`handler` 方法就是根据这些属性来完成方法调用的，返回对象是object,这样是不是能做到客户端不需要持有服务端相应的类或接口了？（需要验证）

 `channelHandlerContext.writeAndFlush(rpcResponse);` 将调用方法的返回值写回Channel中。

### 代理

这里我们把具体的通信细节放到代理类中实现，完成客户端和服务端的代码的解耦。代理部分分为一个代理类工厂和具体的代理类。

先看代理工厂：

```java
public class ProxyFactory {
    public static <T> T create(Class<T> interfaceClass) throws Exception {
        return (T) Proxy.newProxyInstance(interfaceClass.getClassLoader(), new Class<?>[]{interfaceClass}, new RpcClientDynamicProxy<T>(interfaceClass));
    }
}
```

代理工厂创建代理实例的过程 以后有时间会了解并进行补充。

查看具体的代理类：

```java
@Slf4j
public class RpcClientDynamicProxy<T> implements InvocationHandler {
    /**
     * 服务端地址
     */
    private static final String SERVER_HOST = "127.0.0.1";
    /**
     * 服务端端口
     */
    private static final int SERVER_PORT = 8888;

    private Class<T> clzz;

    public RpcClientDynamicProxy(Class<T> clzz) throws Exception{
        this.clzz = clzz;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        RPCRequest rpcRequest = new RPCRequest();
        String requestId = UUID.randomUUID().toString();
        String className = method.getDeclaringClass().getName();
        String methodName = method.getName();

        Class<?>[] parameterType =  method.getParameterTypes();

        rpcRequest.setRequestId(requestId);
        rpcRequest.setClassName(className);
        rpcRequest.setMethod(methodName);
        rpcRequest.setParameterType(parameterType);
        rpcRequest.setParameters(args);

        log.info("请求内容 {}",rpcRequest);

        //开启Netty服务端，直连
        NettyClient client = new NettyClient(SERVER_HOST,SERVER_PORT);
        log.info("开始连接服务器 {}", new Date());
        client.connect();
        RPCResponse rpcResponse = client.send(rpcRequest);
        log.info("请求返回结果 {}",rpcResponse);
        return rpcResponse.getResult();
    }
}
```

在具体的代理类中的`invoke`方法中，设置了RPCRequest对象的属性，并且调用了NettyClient的connect方法。

参考文章[《基于Netty实现RPC框架》](https://juejin.im/post/6844903957622423560#heading-10)