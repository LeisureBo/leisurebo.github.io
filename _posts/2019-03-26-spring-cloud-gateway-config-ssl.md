---
layout: post
title: spring cloud gateway配置Https访问
category: springboot
tags: [https]
excerpt: gateway https 路由采坑记录
---

## 配置SSL安全证书

由于spring-cloud-gateway依赖springboot，SSL配置可参考上一篇博客：[springboot配置https访问](https://www.itbo.top/springboot/2019/03/25/springboot-config-https.html)

需要注意的是，spring-cloud-gateway是基于spring-webflux的，与spring-boot-starter-web依赖有冲突，所以只能以配置文件形式来配置https。

SSL配置如下:

```
server:  
  port: 9005
  ssl:
    enabled: true
    key-alias: localhost
    key-store-password: 123456
    key-store: classpath:localhost.keystore
    key-store-type: PKCS12
```

## https路由问题

原始路由配置如下：

```
spring:
  cloud:
    gateway:
      routes:
      #---------------------------sockjs ws 端点配置----------------------------
      - id: msg-push-1
        uri: lb://msg-push
        predicates:
        - Path=/msgpush/**
        filters:
        - StripPrefix=1
        - name: Retry
          args:
            retries: 3
            series:
            - SERVER_ERROR
            methods:
            - GET
            - POST
      #---------------------------原生 ws 端点配置------------------------------
      - id: msg-push-2
        uri: lb:ws://msg-push
        predicates:
        - Path=/msgpush-ws/**
        filters:
        - StripPrefix=1
        - name: Retry
          args:
            retries: 3
            series:
            - SERVER_ERROR
            methods:
            - GET
            - POST
```

上面配置了spring-websocket的两个路由节点，一个是基于http的stomp sockjs端点，另一个是基于ws的原生websocket端点。

由于spring-cloud-gateway是注册到eureka中心的，所以这里使用LoadBalancerClientFilter的"lb"前缀，将所有到达gateway的请求转发到微服务节点进行处理。

但是当gateway升级为https后会存在一个问题，默认情况下，gateway将请求转发到微服务节点时也会使用https，这就需要接收请求的微服务也需要支持https。

否则通过https网关访问不支持https的微服务会抛出如下异常：

```
io.netty.handler.ssl.NotSslRecordException: not an SSL/TLS record: 485454502f312e3120343030200d0a5472616e736665722d456e636f64696e673a206368756e6b65640d0a446174653a205475652c203236204d617220323031392031323a35333a343120474d540d0a436f6e6e656374696f6e3a20636c6f73650d0a0d0a300d0a0d0a
	at io.netty.handler.ssl.SslHandler.decodeJdkCompatible(SslHandler.java:1156)
	at io.netty.handler.ssl.SslHandler.decode(SslHandler.java:1221)
	at io.netty.handler.codec.ByteToMessageDecoder.decodeRemovalReentryProtection(ByteToMessageDecoder.java:489)
	at io.netty.handler.codec.ByteToMessageDecoder.callDecode(ByteToMessageDecoder.java:428)
	at io.netty.handler.codec.ByteToMessageDecoder.channelRead(ByteToMessageDecoder.java:265)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:362)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:348)
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:340)
	at io.netty.channel.DefaultChannelPipeline$HeadContext.channelRead(DefaultChannelPipeline.java:1434)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:362)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:348)
	at io.netty.channel.DefaultChannelPipeline.fireChannelRead(DefaultChannelPipeline.java:965)
	at io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:163)
	at io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:647)
	at io.netty.channel.nio.NioEventLoop.processSelectedKeysOptimized(NioEventLoop.java:582)
	at io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:499)
	at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:461)
	at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:884)
	at java.lang.Thread.run(Thread.java:748)
```

如果配置每个微服务支持https，需要修改一下微服务的注册配置，添加hostname并且关闭prefer-ip-address，否则会抛出异常：

> No subject alternative names matching IP address XXX found

原因：在发送https的时候会检查ip的主机名，由于设置了使用ip注册所以获取不到主机名，导致该异常出现，去掉就可以使用了，前提是机器间使用主机名可以ping通。

具体配置如下：

```
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:8761/eureka/
  instance:
      # 声明主机名
      hostname: localhost
      # 关闭使用ip地址注册(默认false)
      prefer-ip-address: false
```

## 配置gateway对内转发http

通过分析LoadBalancerClientFilter源码发现，gateway对内转发请求时可以覆盖默认的scheme前缀，源码如下：

```
public class LoadBalancerClientFilter implements GlobalFilter, Ordered {

    private static final Log log = LogFactory.getLog(LoadBalancerClientFilter.class);
    public static final int LOAD_BALANCER_CLIENT_FILTER_ORDER = 10100;

    private final LoadBalancerClient loadBalancer;

    public LoadBalancerClientFilter(LoadBalancerClient loadBalancer) {
        this.loadBalancer = loadBalancer;
    }

    @Override
    public int getOrder() {
        return LOAD_BALANCER_CLIENT_FILTER_ORDER;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 获取路由请求的url，即配置的uri加上请求的端口及参数，例：lb://msg-push:9005/stomp
        URI url = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);
        // 获取lb前缀后携带的scheme前缀，即'lb:<scheme>'中尖括号部分，如果没配置则该值为null
        String schemePrefix = exchange.getAttribute(GATEWAY_SCHEME_PREFIX_ATTR);
        if (url == null || (!"lb".equals(url.getScheme()) && !"lb".equals(schemePrefix))) {
            return chain.filter(exchange);
        }
        //preserve the original url
        addOriginalRequestUrl(exchange, url);

        log.trace("LoadBalancerClientFilter url before: " + url);

        final ServiceInstance instance = loadBalancer.choose(url.getHost());

        if (instance == null) {
            throw new NotFoundException("Unable to find instance for " + url.getHost());
        }

        URI uri = exchange.getRequest().getURI();

        // if the `lb:<scheme>` mechanism was used, use `<scheme>` as the default,
        // if the loadbalancer doesn't provide one.
        // 重点配置参数
        String overrideScheme = null;
        if (schemePrefix != null) {
            overrideScheme = url.getScheme();
        }
        // 这里将路由请求转换为具体的微服务的请求url，包括协议前缀部分，overrideScheme会替代网关请求的scheme前缀
        URI requestUrl = loadBalancer.reconstructURI(new DelegatingServiceInstance(instance, overrideScheme), uri);

        log.trace("LoadBalancerClientFilter url chosen: " + requestUrl);
        exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, requestUrl);
        return chain.filter(exchange);
    }
```

结合源码和官方文档说明，我发现在配置lb路由时添加scheme前缀可指定转发到微服务时使用的协议，例如：`uri: lb:http://msg-push`。

由于msg-push-2的路由配置已经添加了ws前缀，无需更改，所以这里只需将msg-push-1的路由配置添加http前缀即可，最终配置如下：

```
spring:
  cloud:
    gateway:
      routes:
      #---------------------------sockjs ws 端点配置----------------------------
      - id: msg-push-1
        uri: lb:http://msg-push
        predicates:
        - Path=/msgpush/**
        filters:
        - StripPrefix=1
        - name: Retry
          args:
            retries: 3
            series:
            - SERVER_ERROR
            methods:
            - GET
            - POST
      #---------------------------原生 ws 端点配置------------------------------
      - id: msg-push-2
        uri: lb:ws://msg-push
        predicates:
        - Path=/msgpush-ws/**
        filters:
        - StripPrefix=1
        - name: Retry
          args:
            retries: 3
            series:
            - SERVER_ERROR
            methods:
            - GET
            - POST
```

可替代的解决方案可使用参考资料中的第二篇博文，不过个人感觉这样配置侵入性太强，不推荐。


## 参考资料

- [spring-cloud-gateway 使用https注意事项1---设置证书和需要注意的问题](https://www.jianshu.com/p/e5ca9a0953fe)
- [spring-cloud-gateway 使用https注意事项2---如何在转发后端服务的时候使用http](https://www.jianshu.com/p/5a36129399f2)
- [spring-cloud-gateway 官方文档](https://github.com/spring-cloud/spring-cloud-gateway/blob/master/docs/src/main/asciidoc/spring-cloud-gateway.adoc)
