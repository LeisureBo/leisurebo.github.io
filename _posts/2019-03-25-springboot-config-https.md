---
layout: post
title: springboot2.0配置Https访问
category: springboot
---

## 生成安全证书

```
e:\keystore>keytool -genkeypair -alias localhost -keyalg RSA -keystore E:\keystore\localhost.keystore -storetype pkcs12
输入密钥库口令:
再次输入新口令:
您的名字与姓氏是什么?
  [Unknown]:  localhost
您的组织单位名称是什么?
  [Unknown]:  xinwei
您的组织名称是什么?
  [Unknown]:  xinwei
您所在的城市或区域名称是什么?
  [Unknown]:  bj
您所在的省/市/自治区名称是什么?
  [Unknown]:  bj
该单位的双字母国家/地区代码是什么?
  [Unknown]:  cn
CN=localhost, OU=xinwei, O=xinwei, L=bj, ST=bj, C=cn是否正确?
  [否]:  y
```

## 配置HTTPS

* 将生成的`localhost.keystore`证书库文件复制到resource路径下
* 配置application.properties

```
# server port
server.port=1024
# SSL config
server.ssl.key-store-type=PKCS12
server.ssl.key-store=classpath:hellowood.p12
server.ssl.key-store-password=123456
server.ssl.key-alias=hellowood
server.ssl.enabled=true
```
* 测试接口

```
@RestController
public class Controller0325 {
	
	@RequestMapping("/hello")
	public String test01(@RequestParam String param) {
		return "Hello " + param;
	}
}
```

* 启动应用，打印日志提示已启用https端口

```
2019-03-25 17:04:20.126  INFO 9612 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 1024 (https) with context path ''
2019-03-25 17:04:20.129  INFO 9612 --- [           main] com.bo.BootTestApplication               : Started BootTestApplication in 2.351 seconds (JVM running for 2.853)
```

* 访问`http://localhost:1024/hello?param=world`，会提示需要使用https协议进行访问

```
Bad Request
This combination of host and port requires TLS.
```

* 访问`https://localhost:1024/hello?param=world`，成功返回`hello world`

## 配置同时支持http和https访问

* 官方文档说明

> Using configuration such as the preceding example means the application no longer supports a plain HTTP connector at port 8080. 
Spring Boot does not support the configuration of both an HTTP connector and an HTTPS connector through application.properties. 
If you want to have both, you need to configure one of them programmatically. We recommend using application.properties to configure 
HTTPS, as the HTTP connector is the easier of the two to configure programmatically. See the spring-boot-sample-tomcat-multi-connectors 
sample project for an example.

spring boot无法在配置文件中同时配置俩个端口，只能一个通过配置文件、一个通过编码来配置，http的编码原比https简单所以官方推荐编码实现http配置。

下面直接copy官方的示例配置，创建了一个同时支持http访问的配置类，端口设置为8080。

```
@Configuration
public class HttpsConfig {

	@Value("${server.http-port:8080}")
	private int httpPort;

	@Bean
	public ServletWebServerFactory servletContainer() {
		TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory();
		tomcat.addAdditionalTomcatConnectors(createStandardConnector());
		return tomcat;
	}

	private Connector createStandardConnector() {
		Connector connector = new Connector(Http11NioProtocol.class.getName());
		connector.setPort(httpPort);
		return connector;
	}
}
```

* 启动应用，日志如下

```
2019-03-25 17:21:51.240  INFO 4312 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 1024 (https) 8080 (http) with context path ''
2019-03-25 17:21:51.243  INFO 4312 --- [           main] com.bo.BootTestApplication               : Started BootTestApplication in 2.27 seconds (JVM running for 2.738)
```

* 访问`http://localhost:8080/hello?param=world`和`https://localhost:1024/hello?param=world`，均成功返回`hello world`

## 配置http重定向https

* 在配置文件中添加http端口

```
server.http-port:8080
server.port:1024
```

* 添加重定向配置

```
@Configuration
public class HttpsConfig {

	@Value("${server.http-port:8080}")
	private int httpPort;

	@Value("${server.port:8443}")
	private int httpsPort;

	@Bean
	public ServletWebServerFactory servletWebServerFactory() {
		TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory() {
			@Override
			protected void postProcessContext(Context context) {
				SecurityConstraint securityConstraint = new SecurityConstraint();
				securityConstraint.setUserConstraint("CONFIDENTIAL");
				SecurityCollection securityCollection = new SecurityCollection();
				securityCollection.addPattern("/*");
				securityConstraint.addCollection(securityCollection);
				context.addConstraint(securityConstraint);
			}
		};
		factory.addAdditionalTomcatConnectors(redirectConnector());
		return factory;
	}

	private Connector redirectConnector() {
		Connector connector = new Connector(Http11NioProtocol.class.getName());
		connector.setScheme("http");
		connector.setPort(httpPort);
        connector.setSecure(false);
        connector.setRedirectPort(httpsPort);
		return connector;
	}
  
}
```

* 再次启动应用，看到日志中有 HTTP 和 HTTPS 的端口信息

```
2019-03-25 17:51:55.658  INFO 5820 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 1024 (https) 8080 (http) with context path ''
2019-03-25 17:51:55.661  INFO 5820 --- [           main] com.bo.BootTestApplication               : Started BootTestApplication in 2.369 seconds (JVM running for 2.862)
```

* 访问`http://localhost:8080/hello?param=world`，将会被重定向到`https://localhost:1024/hello?param=world`

## 添加wss协议支持

* 配置类中添加bean

```
@Bean
public TomcatContextCustomizer tomcatContextCustomizer() {
    return new TomcatContextCustomizer() {
        @Override
        public void customize(Context context) {
            context.addServletContainerInitializer(new WsSci(), null);
        }
    };
}
```

## 参考资料

- [Spring Boot 配置https访问](https://blog.csdn.net/u013360850/article/details/85493764)
- [spring boot 2.0 配置同时支持http和https](https://blog.csdn.net/qq_34459487/article/details/80885690)
- [spring-boot#Configure SSL](https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/#howto-configure-ssl)
- [spring-boot-sample-tomcat-multi-connectors](https://github.com/spring-projects/spring-boot/tree/v2.0.0.RELEASE/spring-boot-samples/spring-boot-sample-tomcat-multi-connectors) 


