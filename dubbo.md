# 总体分层

​		总体分为业务层`(Biz)、RPC`层`、Remote`三层。

![](image/QQ截图20211108100129.png)

![](image/QQ截图20211108100406.png)

## 总体调用过程

​		首先，服务器端`(`服务提供者`)`在框架启动时，会初始化服务实例，通过`Proxy`组件调用具体协议`(Protocol)`，把服务端要暴露的接口封装成`Invoker(`真

实类型是`AbstractProxylnvoker`，然后转换成`Exporter`，这个时候框架会打开服务端口等并记录服务实例到内存中，最后通过`Registry`把服务元数据注册到注册

中心。

![](image/QQ截图20211108101350.png)

# Hello World

```java
public interface DemoService extends EchoService {
    String sayHello(String name);
}

@DubboService
public class DemoServiceImpl implements DemoService {

    @Override
    public String sayHello(String name) {
        return "Hello " + name;
    }

    @Override
    public Object $echo(Object name) {
        return "$echo " + name;
    }
}
```

```java
public class EchoProvider {
    public static void main(String[] args) throws Exception {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ProviderConfiguration.class);
        context.start();
        System.out.println("dubbo service started");
        new CountDownLatch(1).await();
    }
    @Configuration
    @EnableDubbo(scanBasePackages = "org.apache.dubbo.samples.echo")
    static class ProviderConfiguration{
        @Bean
        public ApplicationConfig applicationConfig()
        {
            ApplicationConfig applicationConfig = new ApplicationConfig();
            applicationConfig.setName("echo-annotation-provider");
            return applicationConfig;
        }
        @Bean
        public ProviderConfig providerConfig()
        {
            ProviderConfig providerConfig = new ProviderConfig();
            providerConfig.setToken(true);
            return providerConfig;
        }
        @Bean
        public RegistryConfig registryConfig()
        {
            RegistryConfig registryConfig = new RegistryConfig();
            registryConfig.setProtocol("zookeeper");
            registryConfig.setAddress("121.5.30.71");
            registryConfig.setPort(2181);
            return registryConfig;
        }
        @Bean
        public ProtocolConfig protocolConfig()
        {
            ProtocolConfig protocolConfig = new ProtocolConfig();
            protocolConfig.setName("dubbo");
            protocolConfig.setPort(20880);
            return protocolConfig;
        }
    }
}
```

```java
public class EchoConsumer {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ConsumerConfiguration.class);
        context.start();
        DemoService demoService = context.getBean("demoServiceImpl", DemoService.class);
        System.out.println(demoService.sayHello("xiaoshanshan"));
        System.out.println(demoService.$echo("shanji"));
    }
    @Configuration
    @EnableDubbo(scanBasePackages = "org.apache.dubbo.samples.echo")
    @ComponentScan(value = {"org.apache.dubbo.samples.echo"})
    static class ConsumerConfiguration
    {
        @Bean
        public ApplicationConfig applicationConfig()
        {
            ApplicationConfig applicationConfig = new ApplicationConfig();
            applicationConfig.setName("echo-annotation-consumer");
            return applicationConfig;
        }
        @Bean
        public ConsumerConfig consumerConfig()
        {
            return new ConsumerConfig();
        }
        @Bean
        public RegistryConfig registryConfig()
        {
            RegistryConfig registryConfig = new RegistryConfig();
            registryConfig.setProtocol("zookeeper");
            registryConfig.setAddress("121.5.30.71");
            registryConfig.setPort(2181);
            return registryConfig;
        }
        @Bean
        public ProtocolConfig protocolConfig()
        {
            ProtocolConfig protocolConfig = new ProtocolConfig();
            protocolConfig.setName("dubbo");
            protocolConfig.setPort(20881);
            return protocolConfig;
        }
    }
}
```

