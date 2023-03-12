---
title: Dubbo版本升级踩坑记录
tags: [Dubbo, Nacos]
index_img: /img/dubbo_20230312155430.png
date: 2021-03-28 20:00:00
---



## 背景

最近我们部门负责基础组件的[**大佬**](https://github.com/yangyang0507)准备升级下Dubbo（v2.7.3 -> v2.7.8），本以为是一次简单的升级，我们各个服务配合刷新下依赖即可，结果却闹出了一系列的问题：

- RPC中枚举序列化问题：这个我们讨论过，应该要**禁止在RPC调用中直接使用枚举作为字段类型**
- Dubbo泛化调用问题：2.7.8版本在泛化调用无参方法时，由于没有对types做非空校验，导致NPE，详情戳☞ [#6840](https://github.com/apache/dubbo/issues/6840)
- 项目启动时找不到provider接口提供者：2.7.8版本创建的provider都会带有Group标识，但是之前的版本没有这个标识，导致provider匹配不上报No Provider错误
- <u>**与Nacos集成引起系统持续不断的创建大量 nacos.naming 线程，导致系统负载持续增加出现崩溃的迹象：这是本次重点记录分析的问题**</u>

## 问题发现

升级上线一小段时间之后，运维通过监控工具`prometheus`发现各个服务实例创建了大量的线程，最少都有500+，最多的单个实例更是达到了4000+，明显太不正常了。持续关注了一段时间后发现线程数一直在增加，并没有减少的迹象，遂开始排查问题。

## 问题解决

首先由运维暂时性的定时重启实例来保证服务的正常。

然后利用[Arthas](https://arthas.gitee.io/index.html)的`thread`命令：查找最忙的N个线程、阻塞其他线程的线程、指定状态（WAITTING、TIMED_WAITTING等）的线程等等，观察发现有大量的`nacos.naming`线程。遂去GitHub Nacos的issues搜索有没有相关的问题描述，果然[#4491](https://github.com/alibaba/nacos/issues/4491)这个issue跟我们的问题很相似，观察到的线程状态跟他的截图也差不多。

之后定位到的是Dubbo issue[#6988](https://github.com/apache/dubbo/issues/6988)和 [#6568](https://github.com/apache/dubbo/issues/6568)，源自2.7.7版本的一个bug。之后按照Dubbo 2.7.9版本的解决方式编译打包了一个新版本来解决这个问题，注意并没有直接升级Dubbo最新版本，是因为怕再出现一些其它问题。

## 问题分析

由于issue上已经把问题代码指出来并进行了修复，那我们现在根据观察到的线程结合源码来反向追踪到问题代码。

利用[Arthas](https://arthas.gitee.io/index.html)的`thread`命令查找到的线程，发现大部分的都是`nacos.naming`相关线程，譬如：

- `nacos.client.naming.updater`：`HostReactor`实例的周期任务线程，用来更新本地缓存的服务实例列表的定时任务。
- `nacos.client.naming.client.listener`：`EventDispatcher`实例的周期任务线程，定时监听服务实例变更的消息（从`HostReactor`处得知）并分发`NamingEvent`事件给订阅者。
- `nacos.client.naming.push.receiver`：`PushReceiver`实例的周期任务线程，开启UDP端口，接收Naocs服务端主动推送的实例节点变动信息，调用`HostReactor`的相关方法来更新服务实例列表，再做ack响应。

查看相关源码发现，这些线程都跟`NacosNamingService`这个类有关系：

```java
// 实现了NamingService接口
public class NacosNamingService implements NamingService {
    ...
    private void init(Properties properties) {
        namespace = InitUtils.initNamespaceForNaming(properties);
        initServerAddr(properties);
        InitUtils.initWebRootContext();
        initCacheDir();
        initLogName(properties);

        eventDispatcher = new EventDispatcher();
  		// 代理对象，跟注册中心Server相关的请求都走它
        serverProxy = new NamingProxy(namespace, endpoint, serverList, properties);
  		// 客户端心跳
        beatReactor = new BeatReactor(serverProxy, initClientBeatThreadCount(properties));
  		// 客户端实例刷新
        hostReactor = new HostReactor(eventDispatcher, serverProxy, cacheDir, isLoadCacheAtStart(properties),
            initPollingThreadCount(properties));
    }
    ...
```

再继续追溯，发现其实例都是由`NamingFactory`创建：

```java

public class NamingFactory {

    public static NamingService createNamingService(String serverList) throws NacosException {
        try {
            Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.naming.NacosNamingService");
            Constructor constructor = driverImplClass.getConstructor(String.class);
            NamingService vendorImpl = (NamingService)constructor.newInstance(serverList);
            return vendorImpl;
        } catch (Throwable e) {
            throw new NacosException(NacosException.CLIENT_INVALID_PARAM, e);
        }
    }
    // 最终是这个方法被调用
    public static NamingService createNamingService(Properties properties) throws NacosException {
        try {
            Class<?> driverImplClass = Class.forName("com.alibaba.nacos.client.naming.NacosNamingService");
            Constructor constructor = driverImplClass.getConstructor(Properties.class);
            NamingService vendorImpl = (NamingService)constructor.newInstance(properties);
            return vendorImpl;
        } catch (Throwable e) {
            throw new NacosException(NacosException.CLIENT_INVALID_PARAM, e);
        }
    }
}

public class NacosFactory {
    /**
     * Create naming service
     *
     * @param properties init param
     * @return Naming
     * @throws NacosException Exception
     */
    // 此方法被外部所调用
    public static NamingService createNamingService(Properties properties) throws NacosException {
        return NamingFactory.createNamingService(properties);
    }
}
```

上面这个方法被两个地方调用到，一个是`NacosDiscoveryProperties`，一个是`NacosNamingServiceUtils`。但由于此次问题是由于Dubbo升级导致的，所以最后的调用方可以不用关心前者，它是Nacos注册发现相关的调用。

继续查看`NacosNamingServiceUtils`的调用链，最后定位到了问题代码`NacosRegistryFactory#createRegistryCacheKey`方法：

```java
// 创建NacosRegistry对象的工厂，NacosRegistry就是Dubbo集成Nacos的注册类，通过它从Nacos上拉取、注册、移除服务实例，
// 最终是通过NacosNamingService来实现的，所以每一个NacosRegistry对象都会持有一个NamingService的对象。
public class NacosRegistryFactory extends AbstractRegistryFactory {

  	// 这个方法就是产生问题的方法代码
    @Override
    protected String createRegistryCacheKey(URL url) {
      	/** 问题代码中是没有这一段注释代码的，直接返回了 url.toFullString()
        String namespace = url.getParameter(CONFIG_NAMESPACE_KEY);
        // 重点一：这一个url.toServiceStringWithoutResolving()方法是没有拼接parameter的
        // 也就是说新的url指向的是一个基础protocol+host+port的字符串，相当于移除了多余的参数（传入的URL中是包含其它参数的）。
        url = URL.valueOf(url.toServiceStringWithoutResolving());
        if (StringUtils.isNotEmpty(namespace)) {
            // 如果有namespace则添加为新url的参数
            url = url.addParameter(CONFIG_NAMESPACE_KEY, namespace);
        }
      	*/
				
      	// 这个方法拼接了URL的完整路径：protocol(+username+password)+host+port+servicekey+其它parameters
      	// 上面这段修复代码加上了之后，这个时候parameters其实里面只有namespace一个参数了。
        return url.toFullString();
    }

    @Override
    protected Registry createRegistry(URL url) {
      	// 这个地方调用到了创建NamingService实例的方法，可以看出是跟NacosRegistry实例关联的
        return new NacosRegistry(url, createNamingService(url));
    }
}

// 父类 AbstractRegistryFactory
public abstract class AbstractRegistryFactory implements RegistryFactory {
  
    ...
  
    // 重点二：这个url中包含了其他参数，其中引发bug的就是含有一个timestamp参数，导致缓存未命中，重复大量创建了Registry对象，
    // 进而出现很多的NamingService实例，以及其创建的周期性任务线程。
    @Override
    public Registry getRegistry(URL url) {
      if (destroyed.get()) {
        LOGGER.warn("All registry instances have been destroyed, failed to fetch any instance. " +
                    "Usually, this means no need to try to do unnecessary redundant resource clearance, all registries has been taken care of.");
        return DEFAULT_NOP_REGISTRY;
      }

      url = URLBuilder.from(url)
        .setPath(RegistryService.class.getName())
        .addParameter(INTERFACE_KEY, RegistryService.class.getName())
        .removeParameters(EXPORT_KEY, REFER_KEY)
        .build();
      // 这儿创建一个缓存map的key，避免重复创建 Registry 实例，期望同一个NameSpaceId下的Registry实例只会创建一个（单例）。
      // 但是由于这个方法里面没有将url中的多余参数timestamp移除，导致缓存key未命中重复创建大量的Registry实例。
      String key = createRegistryCacheKey(url);
      // Lock the registry access process to ensure a single instance of the registry
      LOCK.lock();
      try {
        Registry registry = REGISTRIES.get(key);
        if (registry != null) {
          return registry;
        }
        //create registry by spi/ioc
        registry = createRegistry(url);
        if (registry == null) {
          throw new IllegalStateException("Can not create registry " + url);
        }
        REGISTRIES.put(key, registry);
        return registry;
      } finally {
        // Release the lock
        LOCK.unlock();
      }
    }
    ...
}
```

通过Arthas的`getstatic`命令查看 `AbstractRegistryFactory`的`REGISTRIES`缓存map中的key，这是我在测试环境获取到的示例：

```
[arthas@1]$ getstatic org.apache.dubbo.registry.support.AbstractRegistryFactory REGISTRIES
field: REGISTRIES
@HashMap[
    @String[nacos://nacos-cs.nacos.svc.cluster.local:8848/DEFAULT_GROUP/org.apache.dubbo.registry.RegistryService?namespace=test-rpc]:@NacosRegistry[nacos://nacos-cs.nacos.svc.cluster.local:8848/org.apache.dubbo.registry.RegistryService?application=auth-center&dubbo=2.0.2&group=DEFAULT_GROUP&id=org.apache.dubbo.config.RegistryConfig#0&interface=org.apache.dubbo.registry.RegistryService&namespace=test-rpc&pid=1&qos.enable=false&release=2.7.9&timestamp=1616754580092]
]
```

正确key的组成：

`nacos://nacos-cs.nacos.svc.cluster.local:8848/DEFAULT_GROUP/org.apache.dubbo.registry.RegistryService?namespace=test-rpc`

源码分析时也参考了这个issue [#6568](https://github.com/apache/dubbo/issues/6568)的debug截图，基本搞清楚了整个问题产生的原因以及解决方式的逻辑。

## 小结

升级开源基础组件一定要慎重，对于待升级版本的评审还是很重要的，可以预先通过官网、GitHub或者StackOverflow等等调研下相关版本的问题再动手也不迟 ~