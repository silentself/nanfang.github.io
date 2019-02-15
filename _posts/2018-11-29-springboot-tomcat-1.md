---
layout: post
title:  springboot内嵌tomcat优化<1>
excerpt: "springboot内嵌的tomcat的优化有短板，在application*.yml配置文件上貌似设置不了最大连接数,所以需要在java类中设置..."
categories: [java]
tags: [springboot]
comments: true
---

#### 1、java类

```java
@Configuration
public class TomcatConfig {

    @Value("${server.port}")
    private String port;
    @Value("${server.acceptorThreadCount}")
    private String acceptorThreadCount;
    @Value("${server.minSpareThreads}")
    private String minSpareThreads;
    @Value("${server.maxSpareThreads}")
    private String maxSpareThreads;
    @Value("${server.maxThreads}")
    private String maxThreads;
    @Value("${server.maxConnections}")
    private String maxConnections;
    @Value("${server.protocol}")
    private String protocol;
    @Value("${server.redirectPort}")
    private String redirectPort;
    // @Value("${server.compression}")
    // private String compression;
    @Value("${server.connectionTimeout}")
    private String connectionTimeout;

    @Value("${server.MaxFileSize}")
    private String MaxFileSize;
    @Value("${server.MaxRequestSize}")
    private String MaxRequestSize;

    @Bean
    public ServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory();
        tomcat.addConnectorCustomizers(new VarysTomcatConnectionCustomizer());
        return tomcat;
    }

    @Bean
    public MultipartConfigElement multipartConfigElement() {
        MultipartConfigFactory factory = new MultipartConfigFactory();
        // 单个数据大小
        factory.setMaxFileSize(MaxFileSize); // KB,MB
        /// 总上传数据大小
        factory.setMaxRequestSize(MaxRequestSize);
        return factory.createMultipartConfig();
    }

    public class VarysTomcatConnectionCustomizer implements TomcatConnectorCustomizer {

        public VarysTomcatConnectionCustomizer() {
        }

        @Override
        public void customize(Connector connector) {
            /*connector.setPort(Integer.valueOf(port));
			connector.setAttribute("connectionTimeout", connectionTimeout);
			connector.setAttribute("acceptorThreadCount", acceptorThreadCount);
			connector.setAttribute("minSpareThreads", minSpareThreads);
			connector.setAttribute("maxSpareThreads", maxSpareThreads);
			connector.setAttribute("maxThreads", maxThreads);
			connector.setAttribute("maxConnections", maxConnections);
			connector.setAttribute("protocol", protocol);
			connector.setAttribute("redirectPort", "redirectPort");*/
            // connector.setAttribute("compression", "compression");
            Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();
            protocol.setPort(Integer.valueOf(port));
            protocol.setMaxConnections(Integer.valueOf(maxConnections));
            protocol.setConnectionTimeout(Integer.valueOf(connectionTimeout));
            protocol.setAcceptorThreadCount(Integer.valueOf(acceptorThreadCount));
            protocol.setMinSpareThreads(Integer.valueOf(minSpareThreads));
            protocol.setMaxThreads(Integer.valueOf(maxThreads));

        }
    }

}
```

#### 2、yaml文件中设置参数

```yaml
server:
  port: 8089
  acceptorThreadCount: 4
  minSpareThreads: 50
  maxSpareThreads: 50
  maxThreads: 1000
  maxConnections: 10000
  connectionTimeout: 3600000
  protocol: org.apache.coyote.http11.Http11Nio2Protocol
  redirectPort: 443
  compression:
    enabled: true
  MaxFileSize: 300MB
  MaxRequestSize: 500MB
```

