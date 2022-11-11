---
title: Nacos不可用？服务不可用？
tags: [Nacos, SpringCloud]
index_img: /img/maxresdefault.jpg
date: 2020-11-06 07:00:00
---



最近在开发环境中，频频出现服务不可用的问题。还遇到过一次Nacos客户端连接不上服务端，但是管理平台能正常登录的问题。遂决定研究一波，记录下来。

## Nacos不可用

### 问题

部分容器在启动时一直报错，连接不上Nacos服务端，但是Nacos管理界面能正常登录查看，后来经过排查是**集群的某一个节点挂掉了**导致的。但是这时候问题来了，Nacos作为服务发现和注册中心以及配置中心，集群模式下需要保证CP，并采用Raft算法进行选举，就算挂掉一台节点也不应该出现连接不上的问题。所以到底是为什么呢？这里先来简单看看CAP和Raft。

### CAP

> 可以戳这里☞ [CAP定理](http://www.ruanyifeng.com/blog/2018/07/cap.html)

CAP定理，其实理解起来很简单，主要是针对分布式系统提出来的三个指标，而这三个指标不可能同时满足。

- **P** - Partition tolerance，网络分区容错性，客观存在的，只能接受。
- **A** - Availability，可用性，保证服务有响应，可能读取的数据不是最新的。
- **C** - Consistency，数据一致性，实例之间的数据保持一致，一台实例更新数据，另一台实例必须同步更新完数据后才能够被访问，保证读取的数据始终是最新的。

从上面可以看出**A**和**C**之间也是互相矛盾的。所以，要在它们之间寻求平衡，根据不同的场景选取合适的保证。而Nacos作为服务发现与注册中心以及配置中心使用时，必须得保证CP。但是并不是严格的CP，因为在leader节点进行写操作和同步数据时，其它节点是可以提供查询的。而且在Leader节点宕机重新选举时，集群对外无法提供服务。当然，作为一个通用的解决方案，这种情况对于绝大部分系统是可以接受的。

### Raft

作为目前比较流行的分布式一致性协议，它的实现和理解起来的难度相对较简单，像Nacos、Redis等都采用了它，先来简单了解下。

三种状态（角色）：

- Leader：负责接收所有客户端的请求，本地更新后再同步至其它节点。
- Follower：响应Leader的更新请求，同步日志文件到本地。
- Candidate：选主过程中的一个中间角色，候选者。

如果Follower检测（心跳机制：Leader周期性的发送心跳）到了Leader节点宕机，会触发选举。

- Follower递增自己的任期（term）并转换为Candidate；
- 投票给自己并且给所有其它节点发送投票请求；
- 如果在相同任期内，获得大多数的选票，则成为新的Leader，并发送心跳保持自己的角色。选票结果采取先到先得的方式；如果自己的任期小于请求中的任期，则会认为请求对应的节点为Leader，自身转换为Follower；如果自己的任期大于请求中的任期，则拒绝投票，保持Candidate；
- 如果一段时间内没有选出Leader，可能是出现了平票，则会在选举超时后重新发起（递增任期、发送投票请求）；
- 为了避免出现平票的情况，**选举的超时时间是在一个区间内随机选择的（150ms~300ms）**。也就是说，每个节点选举超时的时间是一个随机值，大大降低了一直平票的可能性。

然后就是所谓的**日志复制**，即数据同步。

- 主要是Leader节点同步日志给Follower节点；
- 如果出现日志数据不一致的情况，Leader会强制覆盖Follower的日志数据；
- Leader会维护每次同步日志的一个索引，每次同步时Follower会验证这个索引是否和自己本地的日志索引一致，不一致则Leader会在下一次推送日志时缩小这个索引，直到验证通过。

### 解决

Nacos集群在如上所述的理论支持下，就算Leader宕机，那么也应该是可以继续访问的。这点从我们能正常登录Nacos管理平台且能从控制台的节点列表看到有一台节点处于`DOWN`的状态而其它都是`UP`得以验证。那么问题可能就不是出在服务端了，应该是客户端的问题，虽然把DOWN掉的Nacos节点重启后，客户端的访问就正常了。最后，经过和运维的一番排查，发现是<u>**yml配置文件中连接Nacos的地址是集群内部访问域名地址，并不是对外的VIP（Virtual IP Address）地址**</u>。

```yaml
spring:
  cloud:
    nacos:
      config:
        namespace: dev
        # 下面这个
        server-addr: nacos-hs.nacos.svc.cluster.local:8848
        file-extension: yml
        shared-configs:
          - config-port.yml
          - config-actuator.yml
```

这里就涉及到Nacos的寻址，采用虚拟IP（VIP）的方式。集群模式下，**客户端连接时要使用这个VIP地址，当某个节点不可用时会自动转发到其它可用节点**。本质上是依靠TCP/IP的ARP协议，详细了解学习可以戳这里☞[微服务架构中基于DNS的服务发现](https://developer.aliyun.com/article/598792)。

这样一来就说的通了，客户端用拿到的IP地址（Java中DNS解析到IP后会缓存下来）一直尝试去连接DOWN掉的Nacos节点，肯定是一直失败，这也造成了一种Nacos集群不可用的假象。后来更新配置文件后，验证也没有问题了。Bingo!

## 服务不可用

### 问题

最近公司项目在搞重构，基本上是推倒重来，新的架构设计，也用上了k8s流水线，很方便我们开发与快速部署。但是问题也随即出现，在dev环境，尤其是在前后端联调（开发会把自己本地的机子注册到Nacos上去）的时候会频繁的出现服务不可用的情况，这是为什么呢？这里先整理下主要用到的开源技术。

- 网关：GateWay + Ribbon + Hystrix
- RPC：Dubbo
- 服务发现与注册中心：Nacos
- 配置中心：Nacos
- 其它：SpringCloud Stream + RocketMQ，Redis，MySQL等等

从报的错误提示”服务不可用“来搜索，发现是网关里面找不到正确的服务实例而报出来的。

```java
@RestController
public class FallbackController {

    @RequestMapping("/fallback/common")
    public Result<Object> fallbackCommon() {
        return Result.failed("服务不可用，请稍后再试！");
    }
}
```

会触发这个fallback的提示，是因为对应的路由下的服务实例不可用。所以，对上面所述的情况做下总结：

- 服务实例频繁上下线，且实例中有真正的dev容器，也有开发自己的机器；
- 采用了Ribbon作为负载均衡器；
- 采用Nacos作为服务发现与注册中心。

这里还是先来简单分析下Nacos和Ribbon的服务实例列表更新机制。

### Nacos

Nacos上的实例分为两种，持久化实例和临时实例，二者可以同时存在。

- 临时实例：默认情况下都是临时实例，在健康检查不通过的情况下，随后的一段时间内会被剔除。适合大部分场景，如弹性扩容和缩容，多余的实例会自动销毁。
- 持久化实例：在健康检查不通过的情况下，不会剔除当前实例，只会标记为不健康。适合运维场景，实时查看健康状态，便于如告警、扩容等操作。

是否临时实例由客户端`Instance`类中的`ephemeral`属性（短暂的; 瞬息的）控制，默认为true。接下来是健康检查机制：

- 临时实例：客户端会生成定时任务，**每隔5s向服务端发送心跳告知存活。服务端也存在定时检测，超过15s没有收到心跳则认为不健康，超过30s则剔除实例。**
- 持久化实例：由服务端主动检测。Server端会生成一个`HealthCheckTask`，再由`TcpSuperSenseProcessor`处理，这里利用了NIO来实现。（`SocketChannel`，`Selector.open()`）

目前我们的系统默认都是**临时实例，所以实例变更时会存在时延**。

### Ribbon

作为SpringCloud中实现客户端负载均衡的利器，Ribbon核心的一些接口如下：

- `ILoadBalancer`，这个接口下的实现`BaseLoadBalancer`是整个Ribbon实现负载均衡的核心类。
- `IRule`，负载均衡规则选取的核心接口，默认提供了轮询、随机、响应时长权重等等选取合适实例的算法。
- `IPing`，这个接口主要是负责检测Ribbon自己缓存的服务实例是否存活。

这里面重点要说的是`BaseLoadBalancer`这个类的核心逻辑，先整理下几个核心属性：

1. `allServerList`和`upServerList`：这两个集合分别存储了从客户端（这个客户端指的是当前Ribbon所在的服务实例）拉取的所有要进行负载均衡的服务实例列表和经过检测后还存活的服务实例列表。
2. `DEFAULT_RULE`：负载均衡选取实例的规则，默认的是轮询。
3. `DEFAULT_PING_STRATEGY`：默认Ping的策略，初始化`BaseLoadBalancer`时默认为null，或者为DummyPing（假Ping，永远返回true）。这个地方是一个核心关注点。
4. 还用到了两把`ReadWriteLock`：`allServerLock`和`upServerLock`，分别用来控制`allServerList`和`upServerList`的读写。

再就是核心逻辑：

1. 首先，不管使用Nacos还是Eureka作为服务发现注册中心，每台实例本地都会缓存一份依赖的服务实例列表。
2. **Ribbon会从当前所在实例的本地实例列表中拉取（<u>定时，默认30s</u>，在`BaseLoadBalancer`的实现类`DynamicServerListLoadBalancer`中）可用的实例列表**，先存放到`allServerList`。在每次服务实例列表有变更时，先去更新`allServerList`，然后依据设置的Ping策略去依次判断可用的服务列表，添加到`upServerList`中。
3. 默认的负载均衡选取规则是`RoundRobinRule`，轮询。在选取目标实例时，会判断`upServerList`是否为空，不为空则依次从`allServerList`中选取可用的实例作为目标实例。**注意，这个时候由于默认的设置，所以拿到的服务永远是可用的，即`isAlive`总为true。**

可以看出，**Ribbon在获取最新实例时也是存在时延的，且默认情况下没有开启定时Ping的任务**。

### 解决

所以，问题就很明显了。结合上面的分析，采用定时的方式进行更新，那么必定有延时，当然在实际生产环境中，基本上不可能如此频繁的变更实例，所以一定的延时是完全没有问题的。而在我们的dev环境，由于容器频繁的重新部署或者开发机器上下线，导致经常出现服务不可用的情况，这也很正常。

当然我们可以改进一下，比如**缩小Ribbon/Nacos的定时拉取/剔除实例的时间间隔，开启Ribbon中定时Ping及时感知服务下线**。但其实我们更应该从另一些方面去减少这类问题的发生，如开发联调规范化、增加多联调环境等等。