---
title: 走进Java中的Loom和虚拟线程
tags: [虚拟线程, 协程]
index_img: /img/project-loom-image.png
date: 2023-12-10 01:00:00
---



> 本文翻译自：[Going inside Java’s Project Loom and virtual threads](https://blogs.oracle.com/javamagazine/post/going-inside-javas-project-loom-and-virtual-threads)

## 来看虚拟线程如何将Java带回到绿色线程[^1]的时代，即Java线程不再等同于操作系统线程

先来谈谈[Loom项目](https://wiki.openjdk.java.net/display/loom/Main)（以下简称Loom），它目前正在探索Java语言的新特性、新API，以及更轻量的运行时并发 - 包含了对虚拟线程的新构思。

Java是第一个把线程融入到核心语言[^2]的主流编程平台。在线程出现之前，要在它们之间进行通信，最先进的技术就是使用多进程或者各种不太优雅的机制（UNIX共享内存，或其它某些机制？）。

在操作系统层面，线程是属于某个进程的独立调度执行单元。每个线程都有它自己的指令执行计数器和调用栈，但是会和同一个进程中的其它线程共享一个堆。

不仅如此，Java堆也是进程堆中一块单独且连续的子集，至少在HotSpot JVM中如此（其它JVM的实现可能有差别）。所以操作系统层面的线程内存模型自然而然的延续到了Java语言的领域。

线程的概念会自然地引出一个轻量级上下文切换的概念：在同一个进程的不同线程之间切换的开销要比在不同进程之间切换线程的开销更小。这主要是因为对于同一个进程下的所有线程来说，把虚拟内存地址转换到物理内存地址的映射表大部分都是相同的。

顺带一提，创建一个线程的开销也比创建一个进程来得小。当然，实际情况是否如此确定仍取决于操作系统的实现细节。

*Java语言规范*[^3]并未在操作系统线程和Java线程之间强制指定任何特殊的映射，这是假定主机操作系统恰好有类似线程的概念 - 而实际上并不总是如此。

实际上，在Java很早期的一些版本当中，JVM线程被[多路复用](https://zh.wikipedia.org/wiki/%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8)到了操作系统的线程（也被称为*平台线程*）上，即所谓的*绿色线程*，因为那些最早的JVM实现实际上仅利用到了一个平台线程。

然而，这种单平台线程的实践大约在Java1.2和1.3时代便逐渐消失（且在Sun公司的Solaris OS上要消失得更早一点）。而目前运行在主流操作系统上的现代Java版本，则遵循着一个Java线程等同于一个操作系统线程的准则。

这就意味着使用`Thread.start()`方法会调起系统的线程创建动作（就像Linux上的`clone()`方法）并创建一个真正的新的操作系统线程。

OpenJDK的Loom旨在 - 亦是其最主要的目标，重新审视这种由来已久的实现方式，并替换为新的运行时*不直接*绑定专门的操作系统线程的`Thread`对象。

换句话讲，Loom创建了一种执行模型，其代表着[执行上下文](https://en.wikipedia.org/wiki/Execution_(computing)#Context_of_execution)的对象不再必须依赖操作系统的调度。故而从某些角度来看，Loom*其实是*重回类似于绿色线程的时代。

但时过境迁，而且计算机技术方面的很多构思往往是超前的。

举个例子，你可以把EJBs（即Jakarta Enterprise Beans，以前也称作Enterprise JavaBeans）当作是一种雄心勃勃试图将环境虚拟化的受限环境形式。那它是否可被视为日后在现代PaaS系统或更小范围的Docker和Kubernetes中广受青睐的环境形式的原型呢？

所以，如果Loom（部分地）重拾绿色线程的理念，那么实现它的一种方式可能就是解决这样一个问题：环境到底发生了何种变化使得回归以往被证明未见成效的旧思路变得极具吸引力？

要稍微地探讨下这个问题，我们得先看一个例子。具体一点，我们需要创建大量的线程来使JVM崩溃：

```java
//
// Please do not actually run this code... it may crash your VM or laptop
//
public class CrashTheVM {
    private static void looper(int count) {
        var tid = Thread.currentThread().getId();
        if (count > 500) {
            return;
        }
        try {
            Thread.sleep(10);
            if (count % 100 == 0) {
                System.out.println("Thread id: "+ tid +" : "+ count);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        looper(count + 1);
    }

    public static Thread makeThread(Runnable r) {
        return new Thread(r);
    }

    public static void main(String[] args) {
        var threads = new ArrayList<Thread>();
        for (int i = 0; i < 20_000; i = i + 1) {
            var t = makeThread(() -> looper(1));
            t.start();
            threads.add(t);
            if (i % 1_000 == 0) {
                System.out.println(i + " thread started");
            }
        }
        // Join all the threads
        threads.forEach(t -> {
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
    }
}
```

这段代码会启动20,000个线程且每个线程尽可能做到最小的处理占用。实际上去运行它的话，应用很可能在远未达到稳定状态时就停止运行或者卡死。当然也有这种可能，由于机器或操作系统受到了限制，无法快速创建线程耗尽资源，使得它能够正常运行完成。

**图1**展示了我使用2019 MacBook Pro运行这段代码时，在电脑彻底失去响应之前的情形。这上面的一些计数不是很准确，比如线程数量，因为操作系统此时已处于崩溃边缘。

![java-loom-virt-figure1](/img/java-loom-virt-figure1.jpg)

**图1.** 展示了大量线程的创建：千万不要在家中尝试

很明显这不能完全代表实际生产环境中的Java应用，它展示了比如在每个连接都对应一个线程的Web服务的环境中可能会出现的一种情况。现代化的高性能Web服务能处理成千上万（或者更多）的并发连接的确已经很优秀了，但这个示例也直白的展现了这种单线程-单连接架构在这种情况下可能出现的失败。

一句话讲：现代程序需要关注比创建线程多得多的可执行上下文。

另一个关键点就是线程的开销可能远比大部分人想象的更加昂贵，并且成为了现代JVM应用的扩展瓶颈。多年来，开发者致力于解决这个问题，要么通过控制线程的开销，要么使用非线程式的执行上下文表达。

曾有过分段式[事件驱动架构（SEDA）](https://en.wikipedia.org/wiki/Staged_event-driven_architecture)的解决方案，最早出现在大概15年前。SEDA可以被描述为这样一个系统，通过多级管道把域对象从A移动到Z，期间伴随着各种各样的转换动作。在分布式系统中，可以通过消息系统或者在每个阶段使用阻塞队列以及线程池来实现SEDA。

在SEDA的每个步骤中，对域对象的处理可以用包含实现了转换逻辑代码的Java对象来描述。要保障其正常工作，代码必须确保是可终止的，不能出现无限循环。但是框架层面无法强制施加这些约束。

SEDA的方案也有一些显著的缺点，尤其对开发者的纪律性要求（很高）。接下来我们来看看一个更好的选择，即Loom。



## Loom项目介绍

[Project Loom](https://wiki.openjdk.java.net/display/loom/Main)是OpenJDK的一个项目，旨在实现”Java平台上易用的，高吞吐量的轻量级并发和新的编程模型“，并为此添加了一些新概念：

- 虚拟线程
- [定界延续](https://en.wikipedia.org/wiki/Delimited_continuation)
- [尾调用消除](https://zh.wikipedia.org/wiki/%E5%B0%BE%E8%B0%83%E7%94%A8)

这里面最关键的就是*虚拟线程*，它设计得跟普通的，编程人员所熟悉的线程类似。但是，虚拟线程由Java运行时所管理，且*不再是*操作系统线程之上的简单的，一对一的包装器。相反，它是由Java运行时在用户空间内实现的。（本文不会讨论定界延续和尾调用消除，但[这里有相关内容](https://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html)）

虚拟线程能带来的好处包括：

- 创建和阻塞的成本更低
- 可以使用Java执行调度器（线程池）
- 栈中不再存在操作系统层面的数据结构

移除虚拟线程生命周期中操作系统的参与就能消除扩展性的瓶颈。大型JVM应用需要处理百万级甚至上十亿级的对象，所以它们为何只能被限制在操作系统中区区数千个可调度对象上呢（这也是思考何为线程的一种方式）？

打破这种限制并解锁新的并发编程方式就是Loom的主要目标。

接下来我们将在实践中观察虚拟线程。下载[Project Loom beta build](https://jdk.java.net/loom/)并启动`jshell`，如下所示：

```shell
$ jshell 
|  Welcome to JShell -- Version 16-loom
|  For an introduction type: /help intro

jshell> Thread.startVirtualThread(() -> {
   ...>     System.out.println("Hello World");
   ...> });
Hello World
$1 ==> VirtualThread[<unnamed>,<no carrier thread>]

jshell>
```

从输出可以很直观的看到虚拟线程的构造。这段代码也用到了一个最新的static方法`startVirtualThread()`，它在新的执行上下文 - 即虚拟线程中，执行了一段lambda表达式。简单明了！

虚拟线程的引入必须遵循这样的原则：已有的代码库必须*完全地*按照虚拟线程出现之前的方式继续运行。这一原则决不能违背，而且我们必须作出这样一个保守的假设，所有已有的Java代码都确实需要一个常设的“基于操作系统的轻量级包装器”的线程架构，这也是到目前为止的唯一方案。

那么，虚拟线程有什么优点呢？它通过一些其他的方式打开了新的视野。目前，Java语言主要提供了两种方式来创建新线程：

- 实现`java.lang.Thread`类并调用继承的`start()`方法
- 创建`Runnable`接口的实例并将它作为参数传递给`Thread`类的构造器，然后启动生成的对象

由于线程的概念正在变化，那么重新审视创建线程的方式自然也很有必要。刚刚我们使用了一个新的静态工厂方法来创建即用即弃的虚拟线程，但是对于已存在的线程API也需要通过另一些方式进行改进。



## 线程构建器

一个新的很重要的概念就是`Thread.Builder`类，它作为`Thread`类的一个内部类被添加进来。用以下代码替换上面例子（第一个示例）中的`makeThread()`方法看看：

```java
public static Thread makeThread(Runnable r) {
    return Thread.builder().virtual().task(r).build();
}
```

这段代码会调用builder中的`virtual()`方法来显式的创建一个能够执行`Runnable`的虚拟线程。当然，我们也可以去掉对`virtual()`的调用，这样会转而创建一个传统的，由操作系统调度的线程对象。但意义何在？

如果你把`makeThread()`方法替换成虚拟线程的版本并用支持Loom的Java重新编译一遍（第一个示例的代码），然后执行生成的二进制文件。

此时，程序会顺利运行，整体的负载情况如**图2**。

![java-loom-virt-figure2](/img/java-loom-virt-figure2.jpg)

**图2.** 创建大量的虚拟线程来替代传统的Java线程

这仅仅是Loom设计理念的一个实例，唯一需要在你自己的Java应用中改动的就是创建线程处的代码。

新的线程库鼓励开发者们摆脱旧范式所使用的一种方式就是`Thread`类的子类不能是虚拟的。所以，这些子类代码仍旧会使用传统的操作系统线程来创建线程。这么做的目的是保护遗留的使用了这些子类的代码并遵循[最小意外原则](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)。

时至今日，虚拟线程变得越来越寻常，开发者们也不再那么关注操作系统线程和虚拟线程之间的差异，这会反过来阻止这种子类机制的继续使用，因为它总会创建一个由操作系统调度的线程。

要注意的是线程库的其他部分也需要升级以更好的支撑Loom。例如，`ThreadBuilder`也支持构建可以作为参数被传递到`Executors`的`ThreadFactory`实例，如下所示：

```shell
jshell> var tb = Thread.builder();
tb ==> java.lang.Thread$BuilderImpl@2e0fa5d3

jshell> var tf = tb.factory();
tf ==> java.lang.Thread$KernelThreadFactory@2e5d6d97

jshell> var vtf = tb.virtual().factory();
vtf ==> java.lang.Thread$VirtualThreadFactory@377dca04
```

当然，在某个阶段，虚拟线程必须依赖一个真正的操作系统线程来执行。这些能让虚拟线程运行在其上的操作系统线程叫做*载体线程*。纵观整个生命周期，一个虚拟线程可能会运行在多个不同的载体线程之上。这不由得让人想起了普通线程会随着时间的推移运行在不同物理CPU核心上的情况 - 二者都是执行调度的示范。

在之前的一个例子的`jshell`输出中我们已经见过载体线程了。



## 使用虚拟线程编程

虚拟线程的出现也带来了思维方式上的转变。编写了现如今的并发应用的编程人员已经习惯于（有意识或无意识地）处理伴随着传统线程而来的天生的扩展限制。

Java开发者习惯使用`Runnable`或者`Callable`接口来创建任务对象，并把他们交付给执行器 - 其背后是用于节约珍贵的线程资源的线程池。但假如所有的这一切都突然变了呢？

Loom试图通过引入比现有思路更划算的全新设计理念来解决线程的扩展性限制，且不再直接映射操作系统线程。这个新特性会看起来并且表现得同如今编程人员所熟知的线程类似。

这就意味着不用去学习一种全新的编程风格（比如[延续传递风格](https://en.wikipedia.org/wiki/Continuation-passing_style)或者[promise/future approach](https://en.wikipedia.org/wiki/Futures_and_promises)或者回调），而是需要Loom运行时在从线程过渡到虚拟线程时继续使用统一的编程模型。换句话说，虚拟线程即简单的线程，至少对编程人员而言。

虚拟线程是*抢占式的*，因为用户端代码无需显式的放弃对CPU的控制。调度点则由虚拟调度器和JDK来控制。开发者亦无需对何时会发生yields作出任何的假设，因为这纯粹是底层的实现细节了。

当然，为了理解虚拟线程的差异，也理应去了解一下操作系统调度的底层设计原理。

当操作系统调度平台线程时，会给每个线程分配CPU时间中的*时间切片*。当时间切片耗尽，会产生硬件中断且内核会重新获得（CPU）的控制权，它会移除正在执行的平台（用户）线程，并替换为另外一个线程。

这种机制保证了UNIX（或者其他各色操作系统）能够让不同任务共享处理器的时间，哪怕在几十年前计算机只有单核的时代亦如此。

虚拟线程，当然与平台线程的处理方式有所不同。现有的调度器都不会使用时间切片的方式来抢占虚拟线程。

用时间切片的方式实现虚拟线程的抢占也是可以的，JVM也有这样的能力来控制执行中的Java线程。比如，利用JVM中的*safe point*。

相反，在阻塞调用（譬如I/O）产生时，虚拟线程会自动放弃（或者*让出（yield）*）它们的载体线程。这是由库和运行时来处理的，并不会受到编程人员的显式控制。

因此，Loom允许Java开发者使用传统的线程顺序性风格来编写代码，并不会强制他们去显式的管理yielding，或者依赖复杂的非阻塞式或基于回调的操作。这样做有一些额外的好处，像调试器和分析器就能够照常运行。

框架开发者和运行时开发工程师可能需要做一些额外的工作以支持虚拟线程，但也比强行给日常的普通开发者们施加额外的认知负担要好得多。

Loom的设计者期望如此，因为虚拟线程永远不需要被池化，也永远*不应该*被池化。相反，这种模型不会限制虚拟线程的创建。为了达到这一目的，新增了一个*unbounded executor*。它可以通过新的工厂方法`Executors.newVirtualThreadExecutor()`来访问。

虚拟线程默认的调度器是自`ForkJoinPool`引入的工作窃取调度器。（有意思一点是[work-stealing aspect of fork/join](https://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html)是如何变得远比递归分解任务更重要的）

Loom的设计是以开发者们能理解其应用中不同线程带来的计算开销为基础的。

简单讲，如果有许多线程都需要持续的占用大量的CPU时间，那么应用就会处于资源紧张状态，即使再巧妙的调度策略也无用。而另一边，如果只有少量线程预计会是CPU密集型，那么就应该把它们放到独立的池且由平台线程来管理。

虚拟线程能在偶现大量的CPU密集型线程时仍运转良好。这让工作窃取调度器能平滑CPU的利用率，且代码最终会调用一个通过yield point[^4]的操作（比如阻塞I/O）。



## 一则警示“寓言”

接下来展示一个例子，当自定义调度虚拟线程时，Loom的设计也会出现一些难以预料的结果。

```java
public final class TangledLoom {
    public static void main(String[] args) {
        var scheduler = Executors.newFixedThreadPool(2);
        Runnable r = () -> {
            System.out.println(Thread.currentThread().getName() +" starting ");
            while (true) {
                int total = 0;
                for (int i = 0; i < 10; i++) {
                    total = total + hashing(i, 'X');
                }
                System.out.println(Thread.currentThread().getName() +" : "+ total);
            }
        };
        var tA = Thread.builder().virtual(scheduler).name("A").task(r).build();
        var tB = Thread.builder().virtual(scheduler).name("B").task(r).build();
        var tC = Thread.builder().virtual(scheduler).name("C").task(r).build();
        tA.start();
        tB.start();
        tC.start();
        try {
            tA.join();
            tB.join();
            tC.join();
        } catch (Throwable tx) {
            tx.printStackTrace();
        }
    }

    private static int hashing(int length, char c) {
        final StringBuilder sb = new StringBuilder();
        for (int j = 0; j < length * 1_000_000; j++) {
            sb.append(c);
        }
        final String s = sb.toString();
        return s.hashCode();
    }
}
```

运行这段代码，会得到以下输出：

```java
$ java TangledLoom
B starting 
A starting 
B : -1830232064
C starting 
C : -1830232064
B : -1830232064
C : -1830232064
B : -1830232064
C : -1830232064
B : -1830232064
C : -1830232064
```

此即所谓的*线程饥饿*，线程`A`看上去永远无法执行。

久而久之，当Loom被越来越多的Java开发者们所熟悉时，肯定会出现一系列通用的最佳实践的范式。但目前为止，大家都还处于探索新技术的早期阶段，须注意上述问题。



## Loom何时发布？[^5]

（略）。



## 延伸阅读

- [Project Loom homepage](https://wiki.openjdk.java.net/display/loom/Main)
- [Project Loom: Fibers and continuations for the Java Virtual Machine](https://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html)
- [Early access snapshot](https://www.oracle.com/java/technologies/javase/early-access-downloads.html)
- [The role of preview features in Java 14, Java 15, Java 16, and beyond](https://blogs.oracle.com/javamagazine/the-role-of-previews-in-java-14-java-15-java-16-and-beyond)
- [Webcast: Project Loom: Modern Scalable Concurrency for the Java Platform](https://www.youtube.com/watch?v=fOEPEXTpbJA)
- [Project Loom on GitHub](https://github.com/openjdk/loom)



[^1]:  绿色线程（Green Thread）：[Green vs Native Threads and Deprecated Methods in Java](https://www.geeksforgeeks.org/green-vs-native-threads-and-deprecated-methods-in-java/)
[^2]: 核心语言是编程语言本身和相关标准库的总称

[^3]: 参见Java SE 21的语言规范文档：https://docs.oracle.com/javase/specs/jls/se21/html/index.html

[^4]: yield point，类似于safe point：https://zhuanlan.zhihu.com/p/114540016
[^5]: 2023年9月19日，[Java21](https://openjdk.org/projects/jdk/21/)正式发布，带来了[Virtual Threads](https://openjdk.org/jeps/444)的特性

