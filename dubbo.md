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



# 注册中心

​		`Dubbo`通过注册中心实现了分布式 环境中各服务之间的注册与发现，是各个分布式节点之间的纽带。作用：

​				1、动态加入：一个服务提供者通过注册中心可以动态地把自己暴露给其他消费者，无须消费者逐个去更新配置文件。

​				2、动态发现：一个消费者可以动态地感知新的配置、路由规则和新的服务提供者，无须重启服务使之生效。

​				3、动态调整：注册中心支持参数的动态调整，新参数自动更新到所有相关服务节点。

​				4、统一配置：避免了本地配置导致每个服务的配置不一致问题。

​		`ZooKeeper`是官方推荐的注册中心，在生产环境中有过实际使用，具体的实现在`Dubbo`源码的`dubbo-registry-zookeeper`模块中。阿里内部并没有使用

`Redis`作为注册中心，`Redis`注册中心并没有经过长时间运行的可靠性验证，其稳定性依赖于`Redis`本身。`Multicast`模式则不需要启动任何注册中心，只要通过

广播地址，就可以互相发现。服务提供者启动时，会广播自己的地址。消费者启动时，会广播订阅请求，服务提供者收到订阅请求，会根据配置广播或单播给订阅

者。不建议在生产环境使用。

​		`Dubbo`拥有良好的扩展性，如果以上注册中心都不能满足需求，那么可以基于`RegistryFactory`和`Registry`自行扩展。



## 工作流程

​		1、服务提供者启动时，会向注册中心写入自己的元数据信息，同时会订阅配置元数据信息。

​		2、消费者启动时，也会向注册中心写入自己的元数据信息，并订阅服务提供者、路由和配置元数据信息。

​		3、服务治理中心启动时，会同时订阅所有消费者、服务提供者、路由和配置元数据信息。

​		4、当有服务提供者离开或有新的服务提供者加入时，注册中心服务提供者目录会发生变化，变化信息会动态通知给消费者、服务治理中心。

​		5、当消费方发起服务调用时，会异步将调用、统计信息等上报给监控中心。

![](image/QQ截图20211109092410.png)



## 原理概述

​		`ZooKeeper`是树形结构的注册中心，每个节点的类型分为持久节点、持久顺序节点、临时节点和临时顺序节点。

​				持久节点：服务注册后保证节点不会丢失，注册中心重启也会存在。

​				持久顺序节点：在持久节点特性的基础上增加了节点先后顺序的能力。

​				临时节点：服务注册后连接丢失或`session`超时，注册的节点会自动被移除。

​				临时顺序节点：在临时节点特性的基础上增加了节点先后顺序的能力。

​		`Dubbo`使用`ZooKeeper`作为注册中心时，只会创建持久节点和临时节点两种，对创建的顺序并没有要求。

​		`ZooKeeper`的树形结构，分为四层：`root(`根节点`)`、`service(`接口名称`)`、四种服务目录`(providers、consumers、routers、configurations)`、服务目录

下具体的`Dubbo`服务`URL`。

![](image/QQ截图20211109093437.png)

​				1、树的根节点是注册中心分组，下面有多个服务接口，分组值来自用户配置`<dubbo:registry>`中的`group`属性，默认是`/dubbo`。

​				2、服务接口下包含`4`类子目录，分别是`providers、consumers、routers、configurators`，这个路径是持久节点。

​				3、服务提供者目录`(/dubbo/service/providers)`下面包含的接口有多个服务者`URL`元数据信息。

​				4、服务消费者目录`(/dubbo/service/consumers)`下面包含的接口有多个消费者`URL`元数据信息。

​				5、路由配置目录`(/dubbo/service/routers)`下面包含多个用于消费者路由策略`URL`元数据信息。

​				6、动态配置目录`(/dubbo/service/configurators)`下面包含多个用于服务者动态配置`URL`元数据信息。

![](image/QQ截图20211109093827.png)

​		在`Dubbo`框架启动时，会根据用户配置的服务，在注册中心中创建`4`个目录，在`providers`和`consumers`目录中分别存储服务提供方、消费方元数据信息，

主要包括`IP`、端口、权重和应用名等数据。

​		在`Dubbo`框架进行服务调用时，用户可以通过服务治理平台下发路由配置。如果要在运行时改变服务参数，则用户可以通过服务治理平台下发动态配置。服

务器端会通过订阅机制收到属性变更，并重新更新已经暴露的服务。 



​		`Redis`注册中心也沿用了`Dubbo`抽象的`Root、Service、Type、URL`四层结构。但是由于`Redis`属于`NoSQL`数据库，数据都是以键值对的形式保存的，并不能

像`ZooKeeper`一样直接实现树形目录结构。因此，`Redis`使用了`key/Map`结构实现了这个需求，`Root、Service、Type`组合成`Redis`的`key`，`Redis`的`value`是一

个`Map`结构，`URL`作为`Map`的`key`，超时时间作为`Map`的`value`。

![](image/QQ截图20211109100600.png)

```java
public void doRegister(URL url) {
        String key = toCategoryPath(url); // 生成 redis 的 key
        String value = url.toFullString(); // 生成 URL
        String expire = String.valueOf(System.currentTimeMillis() + expirePeriod); // 超时时间
        try {
            redisClient.hset(key, value, expire); // 注册到 redis 注册中心
            redisClient.publish(key, REGISTER);
        } catch (Throwable t) {
            throw new RpcException("Failed to register service to redis registry. registry: " + url.getAddress() + ", service: " + url + ", cause: " + t.getMessage(), t);
        }
    }
```





## 订阅/发布

​		当一个已有服务提供者节点下线，或者一个新的服务提供者节点加入微服务环境时，订阅对应接口的消费者和服务治理中心都能及时收到注册中心的通知，并

更新本地的配置信息。如此一来，后续的服务调用就能避免调用已经下线的节点，或者能调用到新的节点。整个过程都是自动完成的，不需要人工参与。

​		服务提供者和消费者都需要把自己注册到注册中心。服务提供者的注册是为了让消费者感知服务的存在，从而发起远程调用；也让服务治理中心感知有新的服

务提供者上线。消费者的发布是为了让服务治理中心可以发现自己。

```java
zkClient.create(toUrlPath(url), url.getParameter(DYNAMIC_KEY, true)); // 发布，即调用 zookeeper 客户端在注册中心上创建一个目录
zkClient.delete(toUrlPath(url)); // 取消发布，即调用 zookeeper 客户端在注册中心上删除对应的目录
```



​		订阅通常有`pull`和`push`两种方式，一种是客户端定时轮询注册中心拉取配置，另一种是注册中心主动推送数据给客户端。这两种方式各有利弊，目前

`Dubbo`采用的是第一次启动拉取方式，后续接收事件重新拉取数据。 在服务暴露时，服务端会订阅`configurators`用于监听动态配置，在消费端启动时，消费端会

订阅`providers、routers`和`configuratops`这三个目录，分别对应服务提供者、路由和动态配置变更通知。

​		可以在`<dubbo:registry>`的`client`属性中设置`curator、zkclient`来使用不同的客户端实现库，如果不设置则默认使用`Curator`作为实现。

​		`ZooKeeper`注册中心采用的是事件通知`+`客户端拉取的方式，客户端在第一次连接上注册中心时，会获取对应目录下全量的数据。并在订阅的节点上注册一

个`watcher`，客户端与注册中心之间保持`TCP`长连接，后续每个节点有任何数据变化的时候，注册中心会根据`watcher`的回调主动通知客户端，客户端接到通知

后，会把对应节点下的全量数据都拉取过来。全量拉取有一个局限，当微服务节点较多时会对网络造成很大的压力。

​		`ZooKeeper`的每个节点都有一个版本号，当某个节点的数据发生变化`(`即事务操作`)`时， 该节点对应的版本号就会发生变化，并触发`watcher`事件，推送数

据给订阅方。版本号强调的是变更次数，即使该节点的值没有变化，只要有更新操作，依然会使版本号变化。

​		事务操作：客户端任何新增、删除、修改、会话创建和失效操作，都会被认为是事务操作，会由`ZooKeeper`集群中的`leader`执行。即使客户端连接的是非

`leader`节点，请求也会被转发给`leader`执行，以此来保证所有事物操作的全局时序性。由于每个节点都有一个版本号，因此可以通过`CAS`操作比较版本号来保证

该节点数据操作的原子性。

​		客户端第一次连上注册中心，订阅时会获取全量的数据，后续则通过监听器事件进行更新。 服务治理中心会处理所有`service`层的订阅，`service`被设置成

特殊值`*`。此外，服务治理中心除了订阅当前节点，还会订阅这个节点下的所有子节点：

```java
if (ANY_VALUE.equals(url.getServiceInterface())) {
		String root = toRootPath();
         boolean check = url.getParameter(CHECK_KEY, false);
         ConcurrentMap<NotifyListener,ChildListener> listeners = zkListeners.computeIfAbsent(url,k -> new ConcurrentHashMap<>()); //订阅所有数据
         ChildListener zkListener = listeners.computeIfAbsent(listener, k -> (parentPath, currentChilds) -> {
         		for (String child : currentChilds) { // 子节点有变化
                 	child = URL.decode(child);
                 	if (!anyServices.contains(child)) { // 当前子节点未被订阅，即是新节点，立即订阅
                    	anyServices.add(child);
                        subscribe(url.setPath(child).addParameters(INTERFACE_KEY, child,Constants.CHECK_KEY, String.valueOf(check)), k);
                    }
         		}
         });
		zkClient.create(root, false); // 创建持久节点
		List<String> services = zkClient.addChildListener(root, zkListener);
		if (CollectionUtils.isNotEmpty(services)) {
				for (String service : services) { // 遍历所有子节点进行订阅
					service = URL.decode(service);
					anyServices.add(service);
					subscribe(url.setPath(service).addParameters(INTERFACE_KEY, service,Constants.CHECK_KEY, String.valueOf(check)), listener);
				}
		}
}
```

​		普通消费者首先根据`URL`的类别得到一组需要订阅的路径。如果类别是`*`，则会订阅四种类型的路径`(providers、routers、consumers、configurators)`, 否

则只订阅`providers`路径：

```java
List<URL> urls = new ArrayList<>();
for (String path : toCategoriesPath(url)) { // 根据 URL 类别获取一组需要订阅的路径
	ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.computeIfAbsent(url, k -> new ConcurrentHashMap<>());
	ChildListener zkListener = listeners.computeIfAbsent(listener, k -> new RegistryChildListenerImpl(url, path, k, latch));
	if (zkListener instanceof RegistryChildListenerImpl) {
		((RegistryChildListenerImpl) zkListener).setLatch(latch);
	}
	zkClient.create(path, false);
	List<String> children = zkClient.addChildListener(path, zkListener); // 订阅，返回该节点下的子路径并缓存
	if (children != null) {
		urls.addAll(toUrlsWithEmpty(url, path, children));
	}
}
notify(url, listener, urls); // I回调 NotifyListener,更新本地缓存信息
```

​		此处会根据`URL`中的`category`属性值获取具体的类别：`providers、routers、 consumers、configurators`，然后拉取直接子节点的数据进行通知`(notify)`。

如果是`providers`类别的数据，则订阅方会更新本地`Directory`管理的`Invoker`服务列表；如果是`routers`分类，则订阅方会更新本地路由规则列表；如果是

`configuators`类别，则订阅方会更新或覆盖本地动态参数列表。





​		`Redis`订阅发布使用的是过期机制和`publish/subscribe`通道。服务提供者发布服务，首先会在`Redis`中创建一个`key`，然后在通道中发布一条`register`事

件消息。 但服务的`key`写入`Redis`后，发布者需要周期性地刷新`key`过期时间，在`RedisRegistry`构造方法中会启动一个`expireExecutor`定时调度线程池，不断

调用`deferExpired()`方法去延续`key`的超时时间。如果服务提供者服务宕机，没有续期，则`key`会因为超时而被`Redis`删除，服务也就会被认定为下线。

​		订阅方首次连接上注册中心，会获取全量数据并缓存在本地内存中。后续的服务列表变化则通过`publish/subscribe`通道广播，当有服务提供者主动下线的时

候，会在通道中广播一条`unregister`事件消息，订阅方收到后则从注册中心拉取数据，更新本地缓存的服务列表。新服务提供者上线也是通过通道事件触发更新

的。

​		如果使用`Redis`作为服务注册中心，会依赖于服务治理中心。如果服务治理中心定时调度， 则还会触发清理逻辑：获取`Redis`上所有的`key`并进行遍历，如

果发现`key`已经超时，则删除`Redis`上对应的`key`。清除完后，还会在通道中发起对应`key`的`unregister`事件，其他消费者监听到取消注册事件后会删除本地对

应服务的数据，从而保证数据的最终一致。

```java
private void deferExpired() {
	for (URL url : new HashSet<>(getRegistered())) { // 获取所有已注册的 key ，进行遍历
		if (url.getParameter(DYNAMIC_KEY, true)) {
			String key = toCategoryPath(url);
			if (redisClient.hset(key, url.toFullString(), String.valueOf(System.currentTimeMillis() + expirePeriod)) == 1) {
				redisClient.publish(key, REGISTER); // 如果返回 1 ，说明 key 已经被删除，重新发布，在通道中广播
			}
		}
	}

	if (doExpire) {
		for (Map.Entry<URL, Long> expireEntry : expireCache.entrySet()) {
			if (expireEntry.getValue() < System.currentTimeMillis()) {
				doNotify(toCategoryPath(expireEntry.getKey()));
			}
		}
	}

	if (admin) { // 如果是服务治理中心，则还要清理过期的 key
		clean();
	}
}
```

![](image/QQ截图20211109111140.png)

​		`Redis`客户端初始化的时候，需要先初始化`Redis`的连接池`jedisPools`，此时如果配置注册中心的集群模式为`<dubbo:registry cluster="replicate" />`，

则服务提供者在发布服务的时候，需要同时向`Redis`集群中所有的节点都写入，是多写的方式。但读取还是从一个节点中读取。 在这种模式下，`Redis`集群可以

不配置数据同步，一致性由客户端的多写来保证。 

​		如果设置为`failover`或不设置，则只会读取和写入任意一个`Redis`节点，失败的话再尝试下一个`Redis`节点。这种模式需要`Redis`自行配置数据同步。 

​		另外，在初始化阶段，还会初始化一个定时调度线程池`expireExecutor`，它主要的任务是延长`key`的过期时间和删除过期的`key`线程池调度的时间间隔是超

时时间的一半。

​		发布：服务提供者和消费者都会使用注册功能。



​		在订阅时，如果是首次订阅，则会先创建一个`Notifier`内部类，这是一个线程类，在启动时会异步进行通道的订阅。在启动`Notifier`线程的同时，主线程会

继续往下执行，全量拉一次注册中心上所有的服务信息。后续注册中心上的信息变更则通过`Notifier`线程订阅的通道推送事件来实现。

```java
if (service.endsWith(ANY_VALUE)) { // 以 * 结尾，订阅所有服务
	if (first) { // 不是第一次，则获取所有服务的 key ，并更新本地缓存
		first = false;
		Set<String> keys = redisClient.scan(service);
		if (CollectionUtils.isNotEmpty(keys)) {
			for (String s : keys) {
				doNotify(s);
			}
		}
		resetSkip(); // 重置计数器
	}
	redisClient.psubscribe(new NotifySub(), service);
}
else {
	if (first) { // 不以 * 结尾，且不是第一次，则代表已订阅过
		first = false;
		doNotify(service); // 出发通知，更新本地缓存，重置失败计数器
		resetSkip();
	}
	redisClient.psubscribe(new NotifySub(), service + PATH_SEPARATOR + ANY_VALUE); // 订阅一个或多个符合给定模式的频道
}
```



## 缓存

​		缓存的存在就是用空间换取时间，如果每次远程调用都要先从注册中心获取一次可调用的服务列表，则会让注册中心承受巨大的流量压力。另外，每次额外的

网络请求也会让整个系统的性能下降。因此`Dubbo`的注册中心实现了通用的缓存机制。

![](image/QQ截图20211109113806.png)

​		消费者或服务治理中心获取注册信息后会做本地缓存。内存中会有一份，保存在`Properties`对象里，磁盘上也会持久化一份文件，通过`file`对象引用：

```java
private final Properties properties = new Properties();
private File file; // 磁盘文件服务缓存对象
private final ConcurrentMap<URL, Map<String, List<URL>>> notified = new ConcurrentHashMap<>(); // 内存中的服务缓存对象
```

​		内存中的缓存`notified`是`ConcurrentHashMap`里面又嵌套了一个`Map`，外层`Map`的`key`是消费者的`URL`，内层`Map`的`key`是分类，包含`providers、`

`consumers、routes、configurators`四种。`value`则是对应的服务列表，对于没有服务提供者提供服务的`URL`，它会以特殊的`empty://`前缀开头。





​		在服务初始化的时候，`AbstractRegistry`构造函数里会从本地磁盘文件中把持久化的注册数据读到`Properties`对象里，并加载到内存缓存中：

```java
	private void loadProperties() {
        if (file != null && file.exists()) {
            InputStream in = null;
            try {
                in = new FileInputStream(file); // 读取本地缓存文件
                properties.load(in);
                if (logger.isInfoEnabled()) {
                    logger.info("Loaded registry cache file " + file);
                }
            } catch (Throwable e) {
                logger.warn("Failed to load registry cache file " + file, e);
            } finally {
                if (in != null) {
                    try {
                        in.close();
                    } catch (IOException e) {
                        logger.warn(e.getMessage(), e);
                    }
                }
            }
        }
    }
```

​		`Properties`保存了所有服务提供者的`URL`，使用`URL#serviceKey()`作为`key`，提供者列表、路由规则列表、配置规则列表等作为`value`。由于`value`是列

表，当存在多个的时候使用空格隔开。还有一个特殊的`key.registies`，保存所有的注册中心的地址。如果应用在启动过程中注册中心无法连接或宕机，则`Dubbo`

框架会自动通过本地缓存加载`Invokers`。



​		缓存的保存有同步和异步两种方式。异步会使用线程池异步保存，如果线程在执行过程中出现异常，则会再次调用线程池不断重试：

```java
	private void saveProperties(URL url) {
        ...
            if (syncSaveFile) {
                doSaveProperties(version); // 同步保存
            } else {
                registryCacheExecutor.execute(new SaveProperties(version)); // 异步保存
            }
        ...
    }
```

​		`AbstractRegistry#notify`方法中封装了更新内存缓存和更新文件缓存的逻辑。当客户端第一次订阅获取全量数据，或者后续由于订阅得到新数据时，都会调

用该方法进行保存。



## 重试

​		`FailbackRegistry`继承了`AbstractRegistry`，并在此基础上增加了失败重试机制作为抽象能力。`ZookeeperRegistry`和`RedisRegistry`继承该抽象方法后，

直接使用即可。

​		`FailbackRegistry`抽象类中定义了一个`ScheduledExecutorService`，每经过固定间隔`(`默认为`5`秒`)`调用`FailbackRegistry#retry()`方法。另外，该抽象类

中还有五个比较重要的集合：

![](image/QQ截图20211109115809.png)

​		在定时器中调用`retry`方法的时候，会把这五个集合分别遍历和重试，重试成功则从集合中移除。`FailbackRegistry`实现了`subscribe、unsubscribe`等通用

方法，里面调用了未实现的模板方法，会由子类实现。通用方法会调用这些模板方法，如果捕获到异常，则会把`URL`添加到对应的重试集合中，以供定时器去重

试。

