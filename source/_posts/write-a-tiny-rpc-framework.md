---
title: 造轮子： 写一个简单的RPC框架
date: 2018-12-15 21:31:32
tags: 
    - rpc
categories: Middleware
---

这两天在看[《Netty实战》](https://book.douban.com/subject/27038538/)这本书，本着“纸上得来终觉浅，绝知此事要躬行”的态度，决定使用Netty实现些小东西以加深理解。思来想去，决定用netty来实现一个简单的RPC框架，于是便有了此篇文章。源码见[https://github.com/tomoyadeng/trpc](https://github.com/tomoyadeng/trpc)

<!-- more -->

## 0x00 RPC要素以及技术选型

RPC(远程过程调用)，简单来讲就是要实现：调用远程计算机上的服务，就像调用本地服务一样。

{% asset_img rpc.png %}

一个简单的RPC主要包含如下的几个要素：

1. 注册中心(Registry): 提供服务注册与服务发现的功能
2. 服务提供方-Provider(Server): 服务提供方
3. 服务调用方-Consumer(Client): 服务调用方

一次RPC调用的流程如下图（忽略掉注册和发现的过程）：

{% asset_img rpc-call.png %}

根据RPC的技术需求，我使用如下技术选型:

|技术关键点|说明|技术选型|选型说明|
|---|---|---|---|
|注册中心|要有注册中心来提供服务注册和发现功能|Etcd|主要考虑到在K8s中部署Etcd比Zookeeper更容易|
|网络框架|实现远程调用需要将Client的请求发送到Server并接收调用结果|Netty|公认的高性能网络框架|
|序列化|编程时是面向对象的，传输是面向字节的，需要在网络传输时将对象和字节互相转化|Protostuff|基于 Protobuf 序列化框架，面向 POJO，无需编写 .proto 文件|
|动态代理|动态代理客户端使客户端不感知远程调用的过程|CGLib|强大的、高性能的代码生成库|

接下来就看看如何实现各个层次的逻辑吧。

## 0x01 服务注册 & 服务发现

首先定义好注册中心的接口

```java
public interface Registry {
    void register(EndPoint endPoint, String serviceName) throws Exception;

    List<EndPoint> discovery(String serviceName) throws Exception;
}
```

```java
@Data
@AllArgsConstructor
public class EndPoint {
    private String host;
    private int port;

    public String toString() {
        return host + ":" + port;
    }
}
```

我这里采用Etcd作为注册中心，因此实现了一个[EtcdRegistry](https://github.com/tomoyadeng/trpc/blob/master/trpc-core/src/main/java/com/tomoyadeng/trpc/core/registry/EtcdRegistry.java)

## 0x02 序列化/反序列化 & Netty编码/解码

RPC要把一次本地方法调用变成能够被网络传输的字节流，那么就需要考虑要进行序列化的对象是什么以及采用哪种序列化协议。

一次方法调用涉及到的对象实体就两个：请求和响应

```java
@Data
public class TRpcRequest {
    private long id;
    private String className;
    private String methodName;
    private Class<?>[] paramTypes;
    private Object[] params;
}
```

```java
@Data
public class TRpcResponse {
    private long id;
    private Throwable exception;
    private Object result;

    public boolean hasException() {
        return exception != null;
    }
}
```

确定了需要序列化的对象实体，接下来就是确定序列化的协议，并实现序列化和反序列化方法。

高级的RPC框架一般都会抽象出序列化器/反序列化器的接口，并提供不同的实现，以供使用者进行选择。简单起见，我这儿直接写一个静态类来实现protobuf协议的序列化和反序列化。

```java
@UtilityClass
public class SerializationUtil {

    public static <T> byte[] serialize(T o) {
        Schema schema = RuntimeSchema.getSchema(o.getClass());
        return ProtostuffIOUtil.toByteArray(o, schema, LinkedBuffer.allocate());
    }

    public static <T> T desrialize(byte[] bytes, Class<T> clazz) {
        Schema<T> schema = RuntimeSchema.createFrom(clazz);
        T message = schema.newMessage();
        ProtostuffIOUtil.mergeFrom(bytes, message, schema);
        return message;
    }
}
```

搞定了序列化对象以及协议，接下来就是实现Netty的编解码器来完成序列化和反序列化的逻辑，有了编解码器，在Netty中就可以Speak to POJO了。

Netty实现编码器和解码器比较简单，直接继承MessageToByteEncoder和ByteToMessageDecoder并实现其抽象方法即可。

```java
public class TRpcEncoder extends MessageToByteEncoder {
    private Class<?> targetClazz;

    public TRpcEncoder(Class<?> targetClazz) {
        this.targetClazz = targetClazz;
    }

    @Override
    protected void encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) throws Exception {
        if (targetClazz.isInstance(msg)) {
            byte[] data = SerializationUtil.serialize(msg);
            out.writeInt(data.length);
            out.writeBytes(data);
        }
    }
}
```

```java
public class TRpcDecoder extends ByteToMessageDecoder {
    private Class<?> targetClazz;

    public TRpcDecoder(Class<?> targetClazz) {
        this.targetClazz = targetClazz;
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        if (in.readableBytes() < 4) {
            return;
        }
        in.markReaderIndex();
        int dataLen = in.readInt();
        if (dataLen < 0) {
            ctx.close();
        }
        if (in.readableBytes() < dataLen) {
            in.resetReaderIndex();
            return;
        }
        byte[] data = new byte[dataLen];
        in.readBytes(data);
        Object obj = SerializationUtil.desrialize(data, targetClazz);
        out.add(obj);
    }
}
```

## 0x03 Server端

Server端需要完成如下几个任务：

1. 处理调用：处理客户端的调用，完成对应的本地方法调用，并调用结果返回给客户端
2. 监听端口：启动一个Server，并监听一个端口，以接收客户端的请求
3. 服务注册：服务端提供哪些服务的调用，需要在服务启动时注册到注册中心上

接下来便来一步步实现上面任务的关键逻辑。

### ServerHandler

Netty的处理流程是基于pipeline的，pipeline中包含一些完成具体操作的handler，前面已经实现的编码器和解码器也是handler，现在就要实现接收消息并完成调用的handler。

```java
@Slf4j
@ChannelHandler.Sharable
public class ServerHandler extends SimpleChannelInboundHandler<TRpcRequest> {
    private final Map<String, Object> clazzMap;

    public ServerHandler(Map<String, Object> clazzMap) {
        this.clazzMap = clazzMap;
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, TRpcRequest msg) throws Exception {
        log.info("receive a request, id={}", msg.getId());
        TRpcResponse response = getResponse(msg);

        ctx.writeAndFlush(response).addListener((GenericFutureListener<ChannelFuture>) future -> {
            if (!future.isSuccess()) {
                log.error("exception", future.cause());
            }
        });
    }

    private TRpcResponse getResponse(TRpcRequest request) {
        TRpcResponse response = new TRpcResponse();
        response.setId(request.getId());

        try {
            Object target = clazzMap.get(request.getClassName());
            if (target == null) {
                throw new IllegalArgumentException(request.getClassName() + " not registered");
            }

            Class<?> clazz = Class.forName(request.getClassName());
            FastClass fastClass = FastClass.create(clazz);
            FastMethod fastMethod = fastClass.getMethod(request.getMethodName(), request.getParamTypes());
            Object result = fastMethod.invoke(target, request.getParams());
            response.setResult(result);

        } catch (Exception e) {
            response.setException(e);
            log.error("invoke exception", e);
        }
        return response;
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        if (cause instanceof IOException) {
            log.info("exception", cause);
        }
        super.exceptionCaught(ctx, cause);
    }
}
```

上面代码的逻辑就是：接收到Request后通过CGLib反射调用目标对象对应的方法，并将调用返回的结果包装成Response返回

### ServerBootstrap

接下来就要启动一个Server来监听一个端口，为了方便扩展，抽象一个Server接口，通过调用Server的start的方法来完成Server的启动，监听目标端口以响应客户端调用。

```java
public interface Server {
    void start();
}
```

在Netty中是通过ServerBootstrap来引导一个Server，这里实现了一个SimpleServer来完成引导过程。

```java
@Slf4j
public class SimpleServer implements Server {
    private EndPoint endPoint;
    private final Map<String, Object> clazzMap;

    public SimpleServer(EndPoint endPoint, Map<String, Object> clazzMap) {
        this.endPoint = endPoint;
        this.clazzMap = clazzMap;
    }

    public void start() {
        EventLoopGroup boosGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(boosGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline()
                                    .addLast(new TRpcDecoder(TRpcRequest.class))
                                    .addLast(new TRpcEncoder(TRpcResponse.class))
                                    .addLast(new ServerHandler(clazzMap));
                        }
                    })
                    .option(ChannelOption.SO_BACKLOG, 1024)
                    .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                    .childOption(ChannelOption.SO_KEEPALIVE, true)
                    .childOption(ChannelOption.TCP_NODELAY, true);

            log.info("Server start at {}", endPoint);
            ChannelFuture future = bootstrap.bind(endPoint.getHost(), endPoint.getPort()).sync();

            future.channel().closeFuture().sync();
        } catch (Exception e) {
            log.error("exception", e);
        } finally {
            workerGroup.shutdownGracefully();
            boosGroup.shutdownGracefully();
        }
    }
}
```

### 服务注册

服务注册就是要将服务提供方所能提供的服务注册到注册中心上。为了使代码更小的侵入性，这里通过一个自定义的注解来标记要注册的服务实现其对应的接口：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
public @interface RpcService {
    Class<?> value();
}
```

接下来实现在服务启动的时候就去扫描指定路径，完成服务注册。这里使用开源的Reflections来完成包扫描，并拿到所有带RpcService注解的实现类，解析出所有的服务名称后调用注册接口完成注册。

```java
Reflections reflections = new Reflections(packagePath);
Set<Class<?>> classes = reflections.getTypesAnnotatedWith(RpcService.class);

Map<String, Object> serviceMap = new ConcurrentHashMap<>();
classes.forEach(clazz -> {
    RpcService rpcService = clazz.getAnnotation(RpcService.class);
    Class<?> rpcClazz = rpcService.value();
    String clazzName = rpcClazz.getName();

    try {
        Object obj = clazz.getConstructor().newInstance();
        serviceMap.put(clazzName, obj);
        registry.register(endPoint, clazzName);
    } catch (Exception e) {
        log.error("exception in register class " + clazzName, e);
    }
});
```

## 0x04 Client端

Client 端要完成如下几个任务：

1. 连接到服务端： 客服端需要连接到具体的服务端，才能进行远程调用(网络传输)
2. 服务发现： 在调用远程服务时，需要到注册中心上查找具体有哪些服务端能提供对应的服务
3. 动态代理： 需要动态生成代理对象来封装远程调用的网络过程

### Handler

首先要实现Handler来处理客服端发送请求和接收响应，由于Netty的handler是异步的，这里用简单的`wait/notify`来完成异步转同步

```java
@Override
protected void channelRead0(ChannelHandlerContext ctx, TRpcResponse msg) throws Exception {
    this.response = msg;
    log.info("receive rsp id={}, result={}", msg.getId(), msg.getResult());
    synchronized (obj) {
        obj.notifyAll();
    }
}

public TRpcResponse send(TRpcRequest request) throws Exception {
    if (channelFuture == null) {
        connect();
    }

    channelFuture.channel().writeAndFlush(request).sync();

    synchronized (obj) {
        obj.wait();
    }

    return response;
}
```

### Bootstrap

客户端的引导过程实际上就是连接到指定的远端地址

```java
public void connect() throws InterruptedException {
    Bootstrap bootstrap = new Bootstrap();
    bootstrap.group(group)
            .channel(NioSocketChannel.class)
            .handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel channel) throws Exception {
                    channel.pipeline()
                            .addLast(new TRpcEncoder(TRpcRequest.class))
                            .addLast(new TRpcDecoder(TRpcResponse.class))
                            .addLast(SimpleClient.this);
                }
            })
            .option(ChannelOption.SO_KEEPALIVE, true);

    channelFuture = bootstrap.connect(endPoint.getHost(), endPoint.getPort()).sync();
}
```

### 服务发现

服务发现就是要到注册中心上查找目标服务的服务提供方。为了使代码更小的侵入性，这里通过一个自定义的注解来标记要进行服务发现的接口。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
public @interface RpcApi {
}
```

接下来就是实现在应用启动时，去扫描指定路径下的RPC接口，发现服务提供方的地址，将结果缓存下来并定时进行刷新。

```java
 @Override
public void init() {
    Reflections reflections = new Reflections(packagePath);
    Set<Class<?>> classes = reflections.getTypesAnnotatedWith(RpcApi.class);

    ScheduledExecutorService executorService = new ScheduledThreadPoolExecutor(2);
    classes.forEach(clazz -> executorService.scheduleAtFixedRate(() -> {
        try {
            if (clazz.isInterface()) {
                String className = clazz.getName();
                List<EndPoint> endPoints = registry.discovery(className);
                log.info("discovery {} endPoints for {}", endPoints.size(), className);
                serviceMap.put(className, endPoints);
                endPoints.forEach(endPoint -> {
                    if (clientPool.get(endPoint) == null) {
                        Client client = getClient(endPoint, this.group);
                        clientPool.put(endPoint, client);
                    }
                });
            }
        } catch (Exception e) {
            log.error("exception in register clazz " + clazz.getName(), e);
        }
    }, 0, 60, TimeUnit.SECONDS));
}
```

### 动态代理实现

通过CGLib来实现客户端的动态代理，首先实现一个MethodInterceptor来完成如下任务：

1. 构造request
2. 获取client完成网络调用
3. 解析response中的调用结果

```java
public class ClientInterceptor implements MethodInterceptor {
    private ClientFactory clientFactory;

    public ClientInterceptor(ClientFactory clientFactory) {
        this.clientFactory = clientFactory;
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        TRpcRequest request = new TRpcRequest();
        Class clazz = method.getDeclaringClass();
        String clazzName = clazz.getName();

        request.setId(System.currentTimeMillis());
        request.setClassName(clazzName);
        request.setMethodName(method.getName());
        request.setParamTypes(method.getParameterTypes());
        request.setParams(args);

        Client client = clientFactory.getClient(clazzName);

        TRpcResponse response = client.send(request);
        if (response.hasException()) {
            throw response.getException();
        }
        return response.getResult();
    }
}
```

随后提供一个代理工厂类来完成代理对象的创建

```java
@Slf4j
public class ClientProxy {
    public static <T> T create(Class<T> clazz, ClientFactory clientFactory) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(clazz);
        enhancer.setCallback(new ClientInterceptor(clientFactory));
        return (T) enhancer.create();
    }
}
```

## 0x05 测试

接下就是使用刚刚写好的RPC框架来实现一个简单的RPC调用流程

### Etcd

如何启动一个本地的Etcd作为注册中心这里就不赘述了，因为在Windows上启动一个Etcd比较简单，属于开箱即用的那种。

### API

首先定义好一个要进行RPC调用的接口，并将API打包成jar发布。

```java
@RpcApi
public interface HelloService {
    String hello(String hello);
}
```

### Server端实现

新建一个 Spring boot 工程来实现Server端。

首先要添加API的依赖

```groovy
compile group: 'com.tomoyadeng', name: 'trpc-sample-api', version: '1.0'
compile('org.springframework.boot:spring-boot-starter-web')
```

编写API的实现类

```java
@Slf4j
@RpcService(value = HelloService.class)
public class HelloServiceImpl implements HelloService {
    @Override
    public String hello(String s) {

        return "[" + getHostName() + "]: " + s;
    }

    private String getHostName() {
        try {
            return Inet4Address.getLocalHost().toString();
        } catch (UnknownHostException e) {
            return "localhost";
        }
    }
}
```

编写启动类：启动类中包含了一些配置的处理

```java
public class ServerBootstrap {

    @Value("${app.trpc.providerHost:#{null}}")
    private String providerHost;

    @Value("${app.trpc.providerPort:#{null}}")
    private String providerPort;

    @Value("${app.trpc.etcdRegistryAddr:#{null}}")
    private String etcdResgitryAddress;

    @PostConstruct
    public void init() {
        Configuration configuration = new Configuration();
        if (providerHost != null) {
            configuration.setProviderHost(providerHost);
        }

        if (providerPort != null) {
            configuration.setProviderPort(Integer.parseInt(providerPort));
        }

        Registry registry = etcdResgitryAddress == null ? new EtcdRegistry() : new EtcdRegistry(etcdResgitryAddress);
        start(configuration, registry);
    }

    private void start(Configuration configuration, Registry registry) {
        try {
            ServerStarter starter = new ServerStarter(configuration, registry, "com.tomoyadeng.trpc.sample.server.impl");
            Executors.newSingleThreadExecutor().execute(starter::start);
        } catch (Exception e) {
            log.error("exception", e);
        }
    }
}
```

启动应用：

```java
@SpringBootApplication
public class TRpcSampleServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(TRpcSampleServerApplication.class, args);
    }
}
```

### Client端实现

新建一个 Spring boot 工程来实现客户端，并提供一个Rest调用的接口来触发RPC的调用。

首先还是添加 API的依赖，以及 Spring boot web 的依赖

```groovy
compile group: 'com.tomoyadeng', name: 'trpc-sample-api', version: '1.0'
compile('org.springframework.boot:spring-boot-starter-web')
```

添加配置类，用以实现动态代理对象的自动装配

```java
@org.springframework.context.annotation.Configuration
public class ClientConfiguration {

    @Bean
    public ClientFactory clientFactory() {
        Configuration configuration = new Configuration();
        Registry registry = registry(configuration);
        ClientFactory clientFactory = new DefaultClientFactory(configuration, registry, "com.tomoyadeng.trpc.sample.api");
        clientFactory.init();
        return clientFactory;
    }

    @Bean
    public Registry registry(Configuration configuration) {
        return new EtcdRegistry();
    }

    @Bean
    @Scope(scopeName = "prototype")
    public HelloService helloService(ClientFactory clientFactory) {
        return ClientProxy.create(HelloService.class, clientFactory);
    }
}
```

提供一个RestController

```java
@RestController
@RequestMapping(path = "/api/v1/trpcsample/hello")
public class HelloController {

    @Autowired
    private HelloService helloService;

    @GetMapping("")
    public String hello() {
        return helloService.hello("hello");
    }
}
```

启动应用

```java
@SpringBootApplication
public class TRpcSampleClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(TRpcSampleClientApplication.class, args);
    }
}
```

增加 Spring boot 的配置文件`application.yaml`

```yaml
server:
  port: 80
```

因为我的服务端也启动在本机的，spring boot web 内嵌的tomcat服务器的默认端口是8080，刚才起服务端的时候已经占用了这个端口，客户端需要换一个端口。

启动OK后，直接通过浏览器访问`http://localhost//api/v1/trpcsample/hello`进行测试
