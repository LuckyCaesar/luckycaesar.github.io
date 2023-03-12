---
title: 再遇Spring循环依赖
tags: [Spring, 循环依赖, 动态代理]
index_img: /img/a25d229007-mobiwusi.jpg
date: 2023-03-12 13:00:00
---



## 闲话

说起Spring中的循环依赖，相信所有的Spring Javaer们都耳熟能详，不论是作为面试八股文死记硬背过，还是在实际项目中踩过坑。我也一样，本以为有Spring来解决这个问题就万事大吉了。然而，这次遇到的却不大一样。

那先来简单回顾下Spring是如何解决循环依赖的，以及，能否解决所有情况的循环依赖？

### 提前暴露和三级缓存

Spring创建bean实例的核心逻辑位于`AbstractAutowireCapableBeanFactory.doCreateBean`方法里面，简单讲主要是这么几个步骤：

- 实例化：`createBeanInstance`，通过构造器实例化，在JVM中创建一个空对象。
- 依赖填充：`populateBean`，`@Autowire`、`@Resource`等注解扫描，依赖属性填充。还有就是执行实现了`InstantiationAwareBeanPostProcessor`接口的增强，譬如我们熟悉的`@PostConstructor`注解就是在这一步被解析的。
- 初始化：`initializeBean`，通常所说的初始化，一些Aware接口、BeanPostProcessor接口的增强，还有我们熟悉的`InitializingBean.afterPropertiesSet`方法、`@Bean`或者xml中bean标签声明的`initMethod`方法等的执行。

而循环依赖的问题，就是在上面的步骤中处理的。Spring巧妙的利用了三级缓存以及提前暴露构造器实例化好的空对象来解决。

```java
/** Cache of singleton objects: bean name --> bean instance */
// 一级缓存
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** Cache of singleton factories: bean name --> ObjectFactory */
// 三级缓存
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

/** Cache of early singleton objects: bean name --> bean instance */
// 二级缓存
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```

更多的原理和源码分析，网上找了一篇文章讲得很详细：[讲一讲Spring中的循环依赖](https://developer.aliyun.com/article/766880)



### 解决了哪些情况下的循环依赖

贴一下文中的表格：

| 依赖情况               | 依赖注入方式                                       | 循环依赖是否被解决 |
| :--------------------- | :------------------------------------------------- | :----------------- |
| AB相互依赖（循环依赖） | 均采用setter方法注入                               | 是                 |
| AB相互依赖（循环依赖） | 均采用构造器注入                                   | 否                 |
| AB相互依赖（循环依赖） | A中注入B的方式为setter方法，B中注入A的方式为构造器 | 是                 |
| AB相互依赖（循环依赖） | B中注入A的方式为setter方法，A中注入B的方式为构造器 | 否                 |

在了解了bean的创建过程，以及Spring解决循环依赖的方式，就能回答上面那篇文章最后提出的思考题。提前暴露和第三级缓存有一个大前提，那就是构造器实例化阶段已经完成，所以：

1. 如果A、B均采用构造器注入，那不管是A还是B先创建，在构造器实例化阶段就已经产生了循环依赖，提前暴露也就失效了，无法解决循环依赖。

2. 最后一种情况，B中注入A的方式为setter方法，A中注入B的方式为构造器。根据Spring默认创建的顺序，A先创建，此时会去找B的实例，再在B中去注入A，然而此时A正处于构造器实例化的过程中，无法提前暴露到第三级缓存中，所以也无法解决。

3. 那反过来，A中注入B的方式为setter方法，B中注入A的方式为构造器。按照A、B的创建先后顺序，A在实例化后被提前暴露进入到第三级缓存当中，而B在实例化注入A时，也能顺利找到提前暴露的A的空对象，所以也就不会报错。

其实除了上面这两种情况，还有一种无法解决的循环依赖，非单例bean的情况。这是自然的，prototype的情况下，Spring并不会管理这些bean的生命周期，也就不存在所谓的循环依赖解决了。

```java
if (isPrototypeCurrentlyInCreation(beanName)) {
    throw new BeanCurrentlyInCreationException(beanName);
}
```



## 报错

接下来就看看报错。先描述场景，*在同一个项目的不同分支上启动服务，存在循环依赖的两个Service的业务代码完全一样，但是结果却大相径庭*。这里使用Aaa和Bbb两个类来做示例，它们之间的循环依赖属于普通的注解注入，而非构造器注入：

```java
@Service
public class Aaa implements AaaIface {
    @Resource
    private Bbb bbb;
    
    @Override
    // 这个@Transactional注解，是问题的核心点之一
    @Transactional
    public void testA() {
        System.out.println(bbb);
    }
}

@Service
public class Bbb implements BbbIface {
    @Resource
    private Aaa aaa;

    @Override
    public void testB() {
        System.out.println(aaa);
    }
}
```

前一节所说的A与B之间无法解决的循环依赖通常遇到的报错日志都是类似这样的：

```tex
Requested bean is currently in creation: Is there an unresolvable circular reference?

...

The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  aaa defined in file [Aaa.class]
↑     ↓
|  bbb defined in file [Bbb.class]
└─────┘
```

而此次遇到的报错却是这样的：

```java
org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'aaa': 
Bean with name 'aaa' has been injected into other beans [bbb] in its raw version as part of a circular reference, but has eventually been wrapped.
This means that said other beans do not use the final version of the bean.
This is often the result of over-eager type matching - consider using 'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.
```

这里面有一些关键字**wrapped、final version**等，猜测跟代理有关。那循环依赖时如果存在被包装代理的增强bean，会有什么不一样？

顺藤摸瓜，找到了报错的源码。这个地方的逻辑是在`populateBean()`和`initializeBean()`之后，也就是完成了`aaa`的初始化后才会触发这些判断：

```java
// 正在创建的aaa的原始bean
Object bean = instanceWrapper.getWrappedInstance();

// 是否触发提前暴露：这里三个条件都是true。aaa是单例 && allowCircularReferences默认为true && aaa正在创建中
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
...
// 依赖填充和初始化
populateBean(beanName, mbd, instanceWrapper);
exposedObject = initializeBean(beanName, exposedObject, mbd);
...
if (earlySingletonExposure) {
    // 针对提前暴露的情况再进行判断，earlySingletonReference肯定是不会为空的，从二级缓存中会取到
    Object earlySingletonReference = getSingleton(beanName, false);
    if (earlySingletonReference != null) {
        // 这个地方exposedObject居然不等于bean？
        if (exposedObject == bean) {
            exposedObject = earlySingletonReference;
        }
        else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
            for (String dependentBean : dependentBeans) {
                if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                    actualDependentBeans.add(dependentBean);
                }
            }
            // else if中前面的逻辑先不看，最后触发了这个异常
            if (!actualDependentBeans.isEmpty()) {
                throw new BeanCurrentlyInCreationException(beanName,
                        "Bean with name '" + beanName + "' has been injected into other beans [" +
                        StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                        "] in its raw version as part of a circular reference, but has eventually been " +
                        "wrapped. This means that said other beans do not use the final version of the " +
                        "bean. This is often the result of over-eager type matching - consider using " +
                        "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
            }
        }
    }
}
```

看了这段代码，感觉有点晕。先不追究根本原因，来找找解决方案吧。



## 解决

这个报错的解决方案，网上一搜就有，一般都是使用`@Lazy`注解：

```java
@Resource
@Lazy
private Bbb bbb;
```

Spring创建`aaa`时发现被`@Lazy`标识的依赖`bbb`，就会对其进行处理，实际上创建的是一个代理对象，而不是真正的`bbb`实例：

```java
protected Object buildLazyResourceProxy(final LookupElement element,
                                        final @Nullable String requestingBeanName) {
    TargetSource ts = new TargetSource() {
        @Override
        public Class<?> getTargetClass() {
            return element.lookupType;
        }
        @Override
        public boolean isStatic() {
            return false;
        }
        @Override
        public Object getTarget() {
            return getResource(element, requestingBeanName);
        }
        @Override
        public void releaseTarget(Object target) {
        }
    };
    ProxyFactory pf = new ProxyFactory();
    pf.setTargetSource(ts);
    if (element.lookupType.isInterface()) {
        pf.addInterface(element.lookupType);
    }
    ClassLoader classLoader = (this.beanFactory instanceof ConfigurableBeanFactory ?
                               ((ConfigurableBeanFactory) this.beanFactory).getBeanClassLoader() : null);
    return pf.getProxy(classLoader);
}
```

所以`aaa`创建时，触发填充`bbb`的依赖，实际上获取到的是上面这个代理对象，没有执行`bbb`的真正创建逻辑，也就是压根儿不会触发循环依赖相关的判断和报错。*在启动完成，`aaa`用到`bbb`时，会触发上面这个代理对象的`getTarget`逻辑，获取真正的`bbb`实例，而此时`bbb`中依赖的`aaa`对象可以直接在Spring容器中获取到，不管是代理对象还是原始对象，自然也不会有问题*。这就是所谓的Lazy的含义吧。

报错是解决了，但是为什么会出现这种错误呢？



## 追因

### AOP版本

有了解决方案，心里就安心多了。在问题分支上开启debug调试，发现走到上面报错源码的地方：*`exposedObject != bean`，exposedObject最终变成了一个CGLIB的代理对象，而bean仍然是实例化好的原始对象*。那肯定不会相等了，所以为什么exposedObject 会变成一个代理对象呢？

最终定位到了这里：`org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#postProcessAfterInitialization()`。*注意这个方法名称，说明是bean初始化阶段`initializeBean`里面某个后置处理器的逻辑*。这也跟上面报错的源码顺序对得上，在`initializeBean`之后。

经过一番追查，发现这里的确是一个重点：*spring-aop在某个版本中对于提前暴露的bean和当前正在创建的原始bean的比对逻辑有变动*。

```java
// 核心逻辑所在类
org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator

// 5.0.13.RELEASE之前
private final Set<Object> earlyProxyReferences = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
...
@Override
public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    if (!this.earlyProxyReferences.contains(cacheKey)) {
        this.earlyProxyReferences.add(cacheKey);
    }
    return wrapIfNecessary(bean, beanName, cacheKey);
}
...
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        // 关键就是下面这个if判断，以前相当于只判断beanName。beanName相同，则认为earlyProxyReferences中的对象和bean是相同的，直接返回bean。
        if (!this.earlyProxyReferences.contains(cacheKey)) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}

//======================================================================================

// 5.0.13.RELEASE之后
private final Map<Object, Object> earlyProxyReferences = new ConcurrentHashMap<>(16);
...
@Override
public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    this.earlyProxyReferences.put(cacheKey, bean);
    return wrapIfNecessary(bean, beanName, cacheKey);
}
...
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        // 但是新版本还增加了bean对象是否相等的判断，如果不等，说明earlyProxyReferences中的对象是被提前代理过的，则当前bean肯定也是需要被代理的，触发执行wrapIfNecessary方法，返回代理后的对象。如果相等，则说明bean无需代理。
        // 当然这里旧版本的并不一定就是有问题的，肯定是有其它的逻辑变动了才会进行调整。
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```

exposedObject最终就指向上面这个方法返回的bean，所以如果被代理了，那exposedObject肯定是不等于正在创建的原始bean的。

到这里我以为找到问题原因了，是版本不一致导致的。然而仔细比对了两个分支的版本后，发现都是使用的*5.0.19.RELEASE*版本，也就是说执行的都是新逻辑。问题分支出现的旧版本并不是当前业务类所在module依赖的。

这就有点奇怪了，于是再到没有报错的分支上继续调试。这一调试果然就发现了不一样的地方。



### AspectJ增强器和Advice增强器

两个分支都出现了一个一样的后置处理`AnnotationAwareAspectJAutoProxyCreator`，但是它并没有触发代理。而在出现问题的分支上，*多执行了一个`DefaultAdvisorAutoProxyCreator`的后置处理器，就是它执行完后会进行代理操作*。它们都继承了`AbstractAutoProxyCreator`类，所以在进行后置增强时都会走到上面的`postProcessAfterInitialization`逻辑中去。关于这两个类的部分注释如下。

DefaultAdvisorAutoProxyCreator：

```java
{@code BeanPostProcessor} implementation that creates AOP proxies based on all
candidate {@code Advisor}s in the current {@code BeanFactory}. This class is
completely generic; it contains no special code to handle any particular aspects,
such as pooling aspects.
```

AnnotationAwareAspectJAutoProxyCreator：

```
{@link AspectJAwareAdvisorAutoProxyCreator} subclass that processes all AspectJ
annotation aspects in the current application context, as well as Spring Advisors.
```

可以看出分别是处理基于Spring原生的Advice接口的增强和基于AspectJ注解的增强。而在当前场景下，两个类并没有AspectJ的注解，也就没有需要`AnnotationAwareAspectJAutoProxyCreator`来处理的增强了。

找到具体的类就好办了，全局搜索后发现是业务框架层权限jar包里面自动装配的bean：

```java
@Bean
@DependsOn("lifecycleBeanPostProcessor")
public DefaultAdvisorAutoProxyCreator getDefaultAdvisorAutoProxyCreator() {
    DefaultAdvisorAutoProxyCreator creator = new DefaultAdvisorAutoProxyCreator();
    creator.setProxyTargetClass(true);
    return creator;
}
```

我们的权限框架使用的是Shiro，在外面封装了一层，使之更适配本地业务。那么加这样的自动装配肯定就跟Shiro的一些增强有关，很明显装配这个bean跟它依赖的`lifecycleBeanPostProcessor`生命周期增强bean息息相关。但是这里不再去深究Shiro相关的代码和配置，只需要知道确实是这个配置引发了问题即可。



### 事务增强

走到这里，我们了解了是`DefaultAdvisorAutoProxyCreator`触发了对`aaa`的代理。那么就还剩下最后一步，到底进行了什么样的增强？

答案就是*事务增强*。类Aaa的public方法上有`@Transactional`注解，那肯定会触发Spring的扫描，对其进行事务增强。贴一下debug截图佐证：

![transactionInterceptor](/img/transactionInterceptor.png)



### 根因分析

最后的最后，再完整的分析一下报错处源码的逻辑：

```java
if (earlySingletonExposure) {
    // 此时提前暴露的aaa已经被提升到了二级缓存，取出来的就是一个事务增强的代理对象
    Object earlySingletonReference = getSingleton(beanName, false);
    if (earlySingletonReference != null) {
        // 这里exposedObject是经过initializeBean之后包装增强的代理对象，自然不会等于原始bean
        if (exposedObject == bean) {
            exposedObject = earlySingletonReference;
        }
        // allowRawInjectionDespiteWrapping默认等于false
        // hasDependentBean(beanName)：aaa当前肯定有依赖的bean，这里也就是bbb
        else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
            // 取bbb
            String[] dependentBeans = getDependentBeans(beanName);
            Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
            for (String dependentBean : dependentBeans) {
                // 这里removeSingletonIfCreatedForTypeCheckOnly返回false，所以会把bbb放入actualDependentBeans
                if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                    actualDependentBeans.add(dependentBean);
                }
            }
            if (!actualDependentBeans.isEmpty()) {
                // 不为空，触发报错
                throw new BeanCurrentlyInCreationException(beanName, ...);
            }
        }
    }
}

// 判断bbb是否已经完整的被创建，这里肯定是返回false。因为aaa依赖填充创建bbb时，由于提前暴露，bbb可以正常的完成创建的流程。
protected boolean removeSingletonIfCreatedForTypeCheckOnly(String beanName) {
    if (!this.alreadyCreated.contains(beanName)) {
        // 如果没有完成创建，则会从多级缓存中移除bbb
        removeSingleton(beanName);
        return true;
    }
    else {
        // bbb已经完成了创建，alreadyCreated中已经存在bbb
        return false;
    }
}
```

此处报错的逻辑已然梳理通顺了，能看出来其实在`allowRawInjectionDespiteWrapping=false`的情况下，后面的报错必然会触发。因为`aaa`肯定依赖`bbb`且此时`bbb`必定已经创建完成。从这个报错来看，Spring认为循环依赖中如果出现代理是有问题的。那为什么呢？先看看关于`allowRawInjectionDespiteWrapping`属性的注释：

```java

/**
  * Set whether to allow the raw injection of a bean instance into some other
  * bean's property, despite the injected bean eventually getting wrapped
  * (for example, through AOP auto-proxying).
  * <p>This will only be used as a last resort in case of a circular reference
  * that cannot be resolved otherwise: essentially, preferring a raw instance
  * getting injected over a failure of the entire bean wiring process.
  * <p>Default is "false", as of Spring 2.0. Turn this on to allow for non-wrapped
  * raw beans injected into some of your references, which was Spring 1.2's
  * (arguably unclean) default behavior.
  * <p><b>NOTE:</b> It is generally recommended to not rely on circular references
  * between your beans, in particular with auto-proxying involved.
  * @see #setAllowCircularReferences
*/
public void setAllowRawInjectionDespiteWrapping(boolean allowRawInjectionDespiteWrapping) {
    this.allowRawInjectionDespiteWrapping = allowRawInjectionDespiteWrapping;
}
```

可以看出Spring也特别提醒（NOTE）了涉及自动代理的情况，不要出现循环依赖，但也没有详细说明，类似那种example代码的。那就只能我们自己来找了，一番debug下来，其实原因也很简单，看下截图就明白了：

![bbb中依赖的aaa对象](/img/b_a20230312112052.png)

![aaa创建时的exposedObjec和earlySingletonReference](/img/a_exo_20230312112104.png)

从截图可以很明显的看出来，*`bbb`中依赖的`aaa`的代理对象ab7178dd@6910（这个代理对象和被提升到二级缓存中的earlySingletonReference的对象是同一个）和最终`aaa`初始化后的代理对象2dd8f255@6967，即exposedObject，是不一样的。因为`aaa`提前暴露触发的代理和初始化触发的代理是分开的两次触发，得到的对象当然不一样。也就是说此时如果不抛出错误，那么Spring容器中会出现两个不同的`aaa`的代理对象*。这显然是错误的。

至此，总算是完完整整的梳理出了这个问题的根因。画了一张简单的整体流程图解：

![Spring循环依赖和动态代理（点击查看大图）](/img/Spring循环依赖.jpg)



## 拓展

梳理完整个流程后，也不得不感叹Spring设计之精妙，当然也非常复杂。最后还可以扩展出一些知识点：

- Spring AOP与AspectJ的比较 [Comparing Spring AOP and AspectJ](https://www.baeldung.com/spring-aop-vs-aspectj)
- JDK动态代理：[Dynamic Proxies in Java](https://www.baeldung.com/java-dynamic-proxies)
- CGLIB动态代理：[Introduction to cglib](https://www.baeldung.com/cglib)
- 事务：[Introduction to Transactions in Java and Spring](https://www.baeldung.com/java-transactions)
