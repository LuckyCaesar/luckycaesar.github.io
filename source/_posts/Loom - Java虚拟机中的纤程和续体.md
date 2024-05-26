---
title: Loom - Java虚拟机中的纤程和续体
tags: [Loom, 纤程, 协程, 续体, 虚拟线程]
index_img: /img/project-loom-image.png
date: 2024-05-26 23:00:00
---



> 本文翻译自：[Project Loom: Fibers and continuations for the Java Virtual Machine](https://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html)

# 概要

Loom项目（以下简称Loom）的使命是构建一个易于书写、调试、描述和维护的更能满足当下需求的并发应用。线程，自Java诞生的第一天就随之出现，是一种天然且实用的并发结构（关于线程的讨论先搁置一旁，可作为一个单独的议题），但也因其不够灵便的抽象正在被取代。它当前基于操作系统内核线程的实现方式无法充分地满足现代化的需求，而且也会对云上尤其珍贵的计算资源产生浪费。

Loom将引入纤程 - 由Java虚拟机所管理的一种轻量、高效的线程，让开发者可以使用同样简单的抽象却具备更高的性能和更低的占用。我们要做的就是让并发再次变得（更）简单！一个纤程由两部分组成 - 续体和调度器。由于Java已经有了`ForkJoinPool`这种设计得很优秀的调度器，那么只要把续体引入到JVM中来即可实现纤程。

# 初衷

许多为Java虚拟机编写的应用都是并发的 - 这就意味着，类似服务器和数据库这样的程序，必须能应对大量的请求、突现的并发以及对计算资源的竞争。Loom要做的就是大幅降低编写高效的并发应用的难度，或者更准确的说，消除编写并发程序时在简单和高效之间所做的权衡取舍。

自二十多年前Java首次发布以来，它最重要的一个贡献就是使得线程和同步原语变得易用。Java线程（不管是直接使用，或是间接接触过，譬如处理HTTP请求的Java servlets）为并发应用的编写提供了一个相对简单的抽象。然而，截至目前，要编写能满足当下需求的并发应用的一个最主要难点在于，由运行时所提供的软件并发单元（线程）无法同域并发单元的规模相匹配，无论是用户、事务还是单个操作。尽管应用层面的并发单元比较粗糙 - 比如用一个socket连接表示的session，一台服务器仍能够处理高达百万量级的活跃的socket，然而利用操作系统线程实现了Java线程的Java运行时，却无法高效地处理区区几千个连接。这种多个数量级的悬殊会产生很大的影响。

编程人员不得不做出抉择，是直接把域并发单元建模成一个线程，大大降低单个服务所能支撑的并发规模。还是使用其它的结构实现比线程（任务）更细粒度的并发单元，再通过编写不会阻塞线程运行的*异步*代码来支持并发。

近些年来Java生态已经引入了很多异步API，从JDK中的异步NIO、异步Servlets，到许多第三方异步库。这些API被设计出来不是因为它们易于编写和理解 - 实际很难上手，也不是因为它们易于调试和描述 - 这更困难（它们连真正意义上的堆栈追踪都无法提供），也不是因为它们构建得比同步API更加优雅 - 相反并不那么优雅，更不是因为它们能更好地适配其他语言或者整合已有代码 - 其实它们适配得很糟糕。只是因为Java中软件并发单元的实现 - 线程，在占用和性能方面做的不够好。很不幸，这个良好且自然的抽象正在被抛弃，取而代之的则是那些从许多方面来看都更糟糕的不够自然的抽象，仅仅是想要它运行时的性能特征。

当然基于内核线程实现的Java线程也有一些优点，主要是内核能支持所有的本地（native）代码，所以运行在线程里的Java代码也能调用本地API。但上文提到的缺点仍大到无法忽视，造成的结果就是要么编码困难、维护成本高，要么会浪费相当多的计算资源，当代码运行在云环境时也将尤其昂贵。实际上，一些语言及它们的运行时已经成功实现了轻量级的线程，最知名的就是Erlang和Go，其展示出来的特性非常实用也广受欢迎。

Loom最主要的目标就是新增这样一个轻量级线程结构，我们叫它纤程，由Java运行时所管理。当然也允许有选择地使用已有的由操作系统提供的重量级线程。纤程在内存占用上比内核线程要低得多，而且任务切换的开销接近于0。单个JVM实例能派生百万量级的纤程，编程人员可以毫不犹豫的发起同步阻塞的调用，因为这些阻塞实际上是没有成本的。除了能让并发应用变得更简单且/或更具弹性外，这也能让框架库的作者们过得更轻松一些，因为不用再为权衡简单和性能而提供同步和异步两套API。简单将至而无需权衡。

正如我们所看到的，线程不再是一个原子结构，而是由两个关键部分组成 - 调度器和*续体*。我们目前打算拆分这两部分，然后基于这两个构建块，在其顶层实现Java的纤程。纤程虽然是Loom成立最主要的动机，但引入续体作为一个可面向用户的抽象也占据了一部分原因，因为续体还有其他用途（如：[Python's generators](https://wiki.python.org/moin/Generators)）。

# 目标和范围

纤程能提供一个低级别的原语，可以基于它实现一些有趣的编程范式，比如管道、actor[^1]和数据流[^2]。虽然这些用途也会被考虑在内，但Loom的目标*不是*设计这些更高级别的结构，也并不建议新的编程风格或者推荐一些在纤程之间进行数据交换的模式（如：共享内存 vs 消息传递）。由于限制线程访问内存的问题是其他OpenJDK项目的主题，也由于此问题（指当前这种线程抽象模型存在的缺点）存在于所有基于线程抽象的实现，不管是轻量的还是重量的，所以Loom可能会和其他项目有所交叉。

Loom的目标是添加一个轻量的线程结构 - 纤程 - 到Java平台上。它会以何种形式呈现在用户面前将在后面进行讨论。首要目的就是让*大部分*Java代码（意思是Java类文件中的代码，不一定是使用Java语言编写的）无需修改或者以最小的修改就能运行在纤程之中。让Java代码调用的本地代码能运行在纤程之中*不是必需*的，虽然在某些环境下*可能*会出现这种情况。让运行在纤程中的*每一段*代码都能享有性能优势也不是最终目的。实际上，一些不适合轻量级线程的代码在性能上反而会有所损失。

Loom的另一个目标是添加一个公共的*限定续体*[^3]（或*协程*[^4]）到Java平台上。当然，相对于纤程（它需要续体，但这些续体无需暴露为公共的API，接下来会有说明）而言它是较次要的。

Loom还有一个目标是尝试为纤程提供各式的*调度器*，但*并不会*深入研究调度器的设计，因为我们相信`ForkJoinPool`会是一个非常棒的纤程调度器。

正如同把操纵调用栈[^5]的能力添加到JVM中是必须的一样，引入一个能把栈展开[^6]到某个点并使用给定参数执行某个方法的更轻量的结构体也同样是Loom的目标之一（总的来说就是高效尾调用[^7]的泛化）。我们把这个特性叫做 *unwind-and-invoke*，或者叫UAI。当然，这并不是说要新增一个尾调用的自动优化到JVM当中。

 Loom可能会涉及到Java平台上一些不同的组件，会依照各自的特性来进行划分：

- 续体和UAI会在JVM中得到实现并对外提供非常简单的Java API。
- 纤程主要在JDK中使用Java来实现，但也需要一些JVM的支持。
- JDK中使用到的会使得线程阻塞的本地代码可能需要做一些适配以便于能在纤程中也正常运行。特别注意，这意味着要修改 `java.io` 类。
- JDK中使用到的一些低级别的线程同步器（尤其是`LockSupport`类），比如`java.util.concurrent`，需要做一些适配来支持纤程，但是工作量取决于纤程API的设计。当然无论如何，预期改动还是比较小的（因为纤程会提供一个跟线程很相似的API）。
- 调试器、分析器及其他一些服务性质的功能也需要感知到纤程的存在以便于提供更好的用户体验。这意味着JFR和JVMTI需要考虑到纤程，而且相关平台的MBeans也需要添加进来。
- 目前为止，我们不认为需要对Java语言本身做改动。

目前还处于项目的早期阶段，所以一切 - 包括其范围，都有可能调整。

# 术语

由于内核线程和轻量级线程只是同一种抽象的不同实现，那么很多术语上的混淆就会接踵而至。此文档会约定以下协议，且每一处通信均会遵循：

- 词语*线程*只代表抽象本身（不久将进行探讨），不会再指代任何特定的实现。所以 *线程* 可能指代抽象的任何实现，不管是基于操作系统还是运行时。
- 当想要指代某个特定的实现时，可以用*重量级线程* 、*内核线程*或者*操作系统线程*这样的措辞来表示基于操作系统内核实现的线程。用*轻量级线程*、*用户态线程*以及*纤程*这样的字眼来表示由语言运行时 - 在Java平台上即JVM和JDK，实现的线程。且这些词语都不会用来描述特定的Java类（至少在目前API设计尚不明了的早期阶段不会）。
- 首字母大写的单词`Thread`和`Fiber`会特指具体的Java类，主要被用于API设计的讨论，而非实现。

# 何为线程

*线程*是一系列有序执行的计算机指令的集合。当我们在处理一些除计算外，还有IO、时停和同步的操作 - 即那些会导致计算流去等待某些额外事件的指令时，线程需具备这样的能力，能*挂起*它自己，并且当这些等待的事件出现时也能*自动恢复*运行。在线程等待期间，它应该让出CPU的使用权，使得其它线程能够正常运行。

这些能力主要涉及到两个不同方面的内容。*续体*是一系列有序执行的指令集，而且可以自我挂起（有关续体的更深入的讨论会在接下来的[续体](#续体)这一章节进行）。*调度器*会把续体分配到CPU核心上，用准备好的替换那些暂停的，并且要保证那些即将恢复执行的续体能最终被分配到CPU上继续运行。故而一个线程需要包含两个结构：续体和调度器，不过它们应该没有必要拆分出来暴露为APIs。

另一方面，线程表示一个最基础的抽象 - 至少在当前上下文中，且并未实现任何的编程范式。尤其要说明的一点是，他们仅指代那些编程人员编写的可运行和暂停的顺序性代码的抽象，而不会涉及到任何在线程之间进行信息共享的机制，比如共享内存或者消息传递。

由于涉及到两个独立的部分，我们可以选择不一样的实现。目前，Java平台提供的线程结构是`Thread`类，是基于内核线程实现的。且续体和调度器的实现都依赖于操作系统。

由Java平台暴露出来的续体结构可以和现有的Java调度器进行结合 - 如`ForkJoinPool`、`ThreadPoolExecutor`或者第三方的实现，也可以和为了实现纤程而特殊优化的调度器进行结合。

当然也可以在运行时和操作系统之间把这两个线程的构建块实现进行拆分。比如，在Google（[视频](https://www.youtube.com/watch?v=KXuZi9aeGTw)，[幻灯片](http://www.linuxplumbersconf.org/2013/ocw/system/presentations/1653/original/LPC - User Threading.pdf)）对Linux内核的修改中，允许用户态代码来接管调度内核线程，因此从本质上来说续体的实现仅依赖于操作系统，同时让一些库来操纵调度。由用户态调度的同时仍允许本地代码运行在这种线程实现上确实带来了一些好处，但也存在占用相对而言较高以及栈无法调整大小的弊端，而且截至目前尚不可用。拆分实现的另外一种方式 - 由操作系统进行调度而运行时来实现续体 - 看起来似乎没有任何好处，因为此方案综合了二者的缺点。

但是，为何用户态线程在各个方面都比内核线程更好，为何它们能称得上*轻量*？同样，（用户态线程）让续体和调度器这两个组件分开考虑时也变得很方便。

要暂停一次计算，续体必须能够存储完整的调用栈上下文，或者简单地说，能存储栈。为了支持本地语言，存储栈的内存空间必须是连续的且要始终保持处于同一片内存地址。虽然虚拟内存能提供一些灵活性，但是这种内核续体（即栈）在轻量性和灵活度方面仍旧存在着一些限制。理想情况下，我们期望栈是可以根据使用情况来扩展或收缩的。由于语言运行时实现的线程是无需支持任何本地代码的，所以我们可以在如何存储续体方面获得更大的灵活度，还能够减少空间的占用。

基于操作系统实现的线程在调度器上存在着很多问题。比如，操作系统调度器会运行在内核模式下，因此每当一个线程阻塞且控制权回到调度器时，必然会产生不小的内核态/用户态切换的开销。再比如，操作系统调度器的设计是偏通用的且能调度各种各样的程序线程。但是，一个视频编码线程的行为必然同一个处理网络IO请求的线程的行为有很大差别，同一个调度算法肯定无法同时成为二者的最佳选择。在服务端上处理事务的线程倾向于表现出某些特定的行为模式，这也会考验通用操作系统调度器的调度能力。举个例子，对于一个处理事务的线程`A`来说，在一次请求上执行一些动作然后再把数据传递给另一个线程`B`再做一些更多的处理，是一种很常见的行为模式。那么此时需要在两个线程之间进行切换时引入一些同步机制，像锁或者消息队列，但是仍保持同样的行为：`A`操作一些数据`x`，唤醒`B`并把数据传递给它，然后`A`进入阻塞直到被来自网络或者另一个线程的新请求唤醒。这种模式极其普遍，所以我们可以假定`A`在唤醒`B`之后很快进入阻塞，那么此时将`B`调度到和`A`一样的核心上会有很多好处，因为`x`已经存在于该核心的缓存之中。此外，把`B`添加到核心的本地队列中也无需任何额外的同步争用开销。实际上，正是有了`ForkJoinPool`这样的工作窃取调度器才使得这种假设能如此精确，因为它会将运行中的任务所调度的任务添加到本地队列当中。然而，对于操作系统内核来说，却无法作出此种假设。在内核看来，线程`A`在唤醒`B`之后可能仍需持续运行一段时间，那么它就会把刚刚非阻塞状态的线程`B`调度到不同的核心上，因此两个线程都需要做一些同步保证，而且一旦`B`访问`x`就会引发缓存故障。

# 纤程

所以，纤程就是我们所说的Java中计划的用户态线程。本章节会列出纤程的需求并探讨一些设计上的问题和各种选项。当然，这并不是说要写得详尽无遗，相反只会呈现一些设计空间的轮廓并营造一种参与挑战的氛围。

从基础能力方面来看，纤程必须能够运行在任意的Java代码片段之中，能和其他线程（轻量级或重量级）并发，且允许用户等待它们终止运行，即所谓的join。很明显，必须有某种机制来保证纤程的挂起和恢复，类似于`LockSupports`的`park/unpark`方法。当然，我们也要能拿到纤程的堆栈追踪信息，用于监控/调试的同时也能跟踪它们的状态（挂起/运行中）等等。简而言之，由于纤程仍是一种线程，那么它就跟`Thread`类所代表的重量级线程有着非常相似的API。再考虑到Java的内存模型，那么纤程会表现得跟目前的`Thread`一模一样。由于纤程将会使用JVM管理的续体来实现，所以我们也得考虑让它们也跟操作系统的续体保持兼容，类似Google的用户调度内核线程的实现。

另一个相对重要的设计决策聚焦在线程本地变量。目前，线程本地变量数据是由（`Inheritable`）`ThreadLocal `类来标识。那么我们在纤程中如何处理线程本地变量呢？关键在于，`ThreadLocal`中有两个非常不一样的用法。一个是使用线程上下文来关联数据，纤程可能也需要有这样的能力。另一个是利用条带化[^8]来减少并发数据结构中的争用。其实这种用法把`ThreadLocal`作为处理器本地（更准确的说是，CPU核心的本地）近似结构（an approximation of a processor-local[^9] construct）而滥用了。在纤程里，这两种不同的用法可能需要明确地拆分开来，因为如今可能跨越百万量级线程（指纤程）的线程本地变量已经不再是一个良好的处理器本地数据近似了。将线程作为上下文与将线程作为处理器近似从而进行更精确处理的要求不仅限于实际的`ThreadLocal`类，还包括将线程实例映射到数据以实现条带化的任何类。如果纤程由`Thread`s来表示，那么针对这种条带化数据结构得做一些调整。不管怎样，预计都会随着纤程的新增而添加一个显式API来访问处理器标识，不管是精确的还是近似的。

内核线程很重要的一个特性是基于时间分片的抢占（这里也可以简单地称之为，强力地或强制地抢占）。一个计算了一段时间且没有阻塞IO或者同步的内核线程会在一定时间后被强制抢占。虽然这乍一看对纤程来说也是一个很重要的设计和实现难点 - 当然，我们也确实决定要支持它，利用JVM safepoints 可以简单地实现，但是其实这个能力并不重要，即便加上之后也没有多少变化（所以最好抛开它）。原因如下：不同于内核线程，纤程的数量会很多（几十上百万量级）。如果*大量*纤程都需要很多的CPU时间，会导致它们*频繁*地被强制抢占，那么当线程数量超过核心数量几个量级时，应用就会由于这种数量级的差距而供不应求，任何调度策略都无济于事。如果需要*大量*纤程*低频*运行长耗时的计算任务，那么一个好的调度器会把纤程合理地分配到可用核心（即工作内核线程）上。如果需要*少量*纤程*高频*运行长耗时的计算任务，那么最好使用重量级线程。因此不同的线程实现虽然提供了同样的抽象，但有时仍会出现某种实现好于另一种的情况。对纤程来说也不必做到在任何情况下都得比内核线程更优。

然而，真正实现上的挑战可能在于如何协调好纤程与能阻塞内核线程的内置JVM代码的关系。示例包括了隐形代码，譬如把类从硬盘加载到面向用户的功能模块当中，又或者像`synchronized`和`Object.wait`。由于纤程调度器会以多路复用的方式将大量纤程调度到少量工作内核线程之上，那么阻塞内核线程可能会占用调度器的很大一部分可用资源，当然是需要避免的。

考虑一个极端，上述每种情况都需要对纤程友好，即：如果是由纤程触发的阻塞，那么只会阻塞纤程而不是底层的内核线程。而另一个极端，所有情况下都忽略纤程继续去阻塞底层的内核线程。在这二者之间，我们可以使得一部分构造成为纤程阻塞而另一部分仍然保持内核线程阻塞。有诸多例子能给出合理的理由来维持原状，即保持内核线程阻塞。比如，类加载只会在启动期间频繁触发，而在启动后很少触发。因此，正如上述所说，纤程调度器围绕着这种阻塞可以简单地调度处理。而许多对`synchronized`的使用是为了内存访问的保护并且只会阻塞非常短的时间 - 完全可以忽略它。所以我们甚至可能决定不去改动`synchronized`，然后鼓励那些使用了`synchronized`来做IO访问同步及需要频繁阻塞的代码改为使用`j.u.c`里面的同步结构（它们会是纤程友好的），如果这些代码想要运行在纤程中的话。类似的，对于`Object.wait`的使用，虽然在现代代码中并不常见，但是仍然建议（至少目前为止我们建议）改为使用`j.u.c`。

不管怎样，一个纤程阻塞了其所在的底层内核线程就会触发一些能被JFR/Mbeans监控到的系统事件。

虽然纤程鼓励使用寻常、简单且自然的同步阻塞代码，但其实适配现有的异步APIs也很简单，会把它们转换成纤程阻塞的模式。假定有这么一个库提供了下面这样一个为了某些长耗时操作而设计的异步API - `foo`，然后返回了一个`String`：

```java
interface AsyncFoo {
   public void asyncFoo(FooCompletion callback);
}
```
其中的回调或者完结处理器`FooCompletion`有如下定义：
```java
interface FooCompletion {
  void success(String result);
  void failure(FooException exception);
}
```
我们会提供一个异步转纤程阻塞的结构，如下所示：
```java
 
abstract class _AsyncToBlocking<T, E extends Throwable> {
    private _Fiber f;
    private T result;
    private E exception;
  
    protected void _complete(T result) {
        this.result = result;
        unpark f
    }
  
    protected void _fail(E exception) { 
        this.exception = exception;
        unpark f
    }
  
    public T run() throws E { 
        this.f = current fiber
        register();
        park
        if (exception != null)
           throw exception;
        return result;
    }
  
    public T run(_timeout) throws E, TimeoutException { ... }
  
    abstract void register();
}
```
然后我们可以通过定义以下的类来创建一个阻塞版本的API：
```java
abstract class AsyncFooToBlocking extends _AsyncToBlocking<String, FooException> 
     implements FooCompletion {
  @Override
  public void success(String result) {
    _complete(result);
  }
  @Override
  public void failure(FooException exception) {
    _fail(exception);
  }
}
```
再然后我们就可以把异步API封装成同步版本：
```java
class SyncFoo {
    AsyncFoo foo = get instance;
  
    String syncFoo() throws FooException {
        new AsyncFooToBlocking() {
          @Override protected void register() { foo.asyncFoo(this); }
        }.run();
    }
}
```
可以用这种方式把一些通用的异步类都涵盖进来，比如`CompletableFuture`。

# 续体

向Java平台中添加续体的初始动机是实现纤程，但续体也有一些其他有趣的用途，这也是把续体作为公共API暴露出来的次要目的。当然，这些其他用途带来的好处预期肯定是远不如纤程的。实际上，续体不会在纤程之上增加表达力[^10]（即，续体可以在纤程之上去实现）。

在本文及Loom项目的任何地方，*续体*这个词都指代*定界续体*（有时候也称之为*协程*[^11]）。这里我们把定界续体想成是一段可以挂起（它自己）和（由它的调用者）恢复执行的串行代码。一些人可能对另一种说法更熟悉一点，把续体认为是一组对象（通常是子程序），它们代表了一次计算的“余下”或者“未来”部分。这二者描述的其实是一件事情：一个挂起的续体是一个对象，当它被恢复或者“被调用”时，会执行一些剩下的计算。
一个定界续体是一段带有入口点（entry point）的串行子程序（就像线程一样），这里我们简单地称其为*入口点*（在Scheme中，这个叫做*重置点*（reset point）），它可以在某些时候挂起或者让出执行，所以我们也可以叫做*挂起点*（suspension point）或者*让出点*（yield point）。当一个定界续体挂起时，控制权会转移到续体之外。而当它恢复时，控制权会回到上次的让出点，那么执行上下文会完好无损地到达入口点。有多种方式可以表达定界续体，但对于Java编程人员来说，以下粗略的伪代码能最好地诠释：

```java
foo() { // (2)
  ... 
  bar()
  ...
}

bar() {
  ...
  suspend // (3)
  ... // (5)
}

main() {
  c = continuation(foo) // (0)
  c.continue() // (1)
  c.continue() // (4)
}
```
在(0)处创建了一个续体，入口点是`foo`。然后执行(1)把控制权传递给续体的入口点(2)处。再然后继续执行直至遇到子程序`bar`内部的挂起点(3)，此时会返回并重新指向(1)处的调用。当在(4)这个地方再次调用续体时，控制权会返回到后续的让出点(5)这一行。

此处讨论的续体是“堆叠的”，因为续体可能会阻塞在调用栈的任意嵌套深度之内（在上面的例子中，是阻塞在被`foo` - 即入口点，调用的函数`bar`内部）。作为对比，非堆叠的续体可能只会在与入口点相同的子程序中挂起。当然，此处的续体是不可重入的，这就意味着对续体的任何调用都可能会改变“当前”的挂起点。换句话说，续体对象是有状态的。

实现续体最主要的技术任务 - 实际上，也是整个项目最主要的任务，是向HotSpot中添加捕获、存储和恢复调用栈的能力，而不是作为内核线程的一部分。且JNI栈帧很大可能也*不再支持*。

续体是纤程的基础，如果续体要作为一个公共API暴露出来，那么就需要我们支持嵌套续体。这就意味着在续体内部运行的代码不仅需要能够挂起续体本身，还要能挂起封装它的代码（比如，挂起封装了它的纤程）。举例，续体常见的用途是generators的实现。一个generator会提供一个迭代器，每当generator让出执行时，运行在generator中的代码会给这个迭代器生成另一个值。因此可以用以下代码来表示：
```java
new _Fiber(() -> {
  for (Object x : new _Generator(() -> {
      produce 1
      fiber sleep 100ms
      produce 2
      fiber sleep 100ms
      produce 3
  })) {
      System.out.println("Next: " + x);
  }
})
```
在一些参考资料中，表现出此种行为的嵌套续体有时也被叫做“带多具名提示信息的定界续体”（delimited continuations with multiple named prompts），但是我们还是把它称作*作用域续体*（scoped continuations）。可以看看[这篇博客](https://blog.paralleluniverse.co/2015/08/07/scoped-continuations)对于作用域续体的理论表达力地讨论（对于那些感兴趣的人来说，续体是一种“一般效应”（general effect），可以用来达成任意效应 - 如赋值，即使在没有任何副作用的纯语言中。这就是为何续体从某种意义上来说是命令式编程中最基础的抽象）。

运行在续体中的代码不会持有对续体的引用，并且其作用域通常都有一些固定的名称（因此挂起作用域`A`也会挂起其最内层的续体）。当然，让出点也提供了一种机制，可以在代码和续体实例之间来回传递信息。当一个续体挂起时，无法触发封装了让出点的`try/finally`块（即，运行在续体中的代码无法检测到正处于挂起过程中）。

把续体作为纤程中的一个独立结构（无论他们是否会暴露为公共API）来实现的原因之一是明确的关注点分离。因此，续体不是线程安全的且他们的任何操作都不会产生跨线程的 happens-before 关系。纤程实现时需要负责建立当续体从一个内核线程迁移到另一个内核线程时必要的内存可见性保证。

可能会新增的API大致如下所示。续体是非常低级别的原语，可能只会被一些库的作者用来构建更高级别的组件（就像`java.util.Stream`的实现利用了`Spliterator`一样）。我们期望用到了续体的类会持有一个续体类的私有实例，或者更有可能就是续体的子类，这样续体的实例就不会直接暴露给这些组件的消费者。
```java
class _Continuation {
    public _Continuation(_Scope scope, Runnable target) 
    public boolean run()
    public static _Continuation suspend(_Scope scope, Consumer<_Continuation> ccc)
    
    public ? getStackTrace()
}
```
当续体终止时，`run`方法会返回`true`。挂起时，会返回`false`。`suspend`方法允许在让出点传递数据给续体（使用`ccc`回调可以向给定的续体实例中注入数据），然后续体也可以返回数据到挂起点（使用返回值，即续体实例本身，可以从中查询一些信息）。

为了演示通过续体来实现纤程的简易性，这里提供如下示例，部分地、简单地实现了一个代表纤程的`_Fiber`类。我们可以注意到，大部分的代码都是在维护纤程的状态，以保证它不会被多个并发所调度：
```java
class _Fiber {
    private final _Continuation cont;
    private final Executor scheduler;
    private volatile State state;
    private final Runnable task;

    private enum State { NEW, LEASED, RUNNABLE, PAUSED, DONE; }
  
    public _Fiber(Runnable target, Executor scheduler) {
        this.scheduler = scheduler;
        this.cont = new _Continuation(_FIBER_SCOPE, target);
      
        this.state = State.NEW;
        this.task = () -> {
              while (!cont.run()) {
                  if (park0())
                     return; // parking; otherwise, had lease -- continue
              }
              state = State.DONE;
        };
    }
  
    public void start() {
        if (!casState(State.NEW, State.RUNNABLE))
            throw new IllegalStateException();
        scheduler.execute(task);
    }
  
    public static void park() {
        _Continuation.suspend(_FIBER_SCOPE, null);
    }
  
    private boolean park0() {
        State st, nst;
        do {
            st = state;
            switch (st) {
              case LEASED:   nst = State.RUNNABLE; break;
              case RUNNABLE: nst = State.PAUSED;   break;
              default:       throw new IllegalStateException();
            }
        } while (!casState(st, nst));
        return nst == State.PAUSED;
    }
  
    public void unpark() {
        State st, nst;
        do {
            State st = state;
            switch (st) {
              case LEASED: 
              case RUNNABLE: nst = State.LEASED;   break;
              case PAUSED:   nst = State.RUNNABLE; break;
              default:       throw new IllegalStateException();
            }
        } while (!casState(st, nst));
        if (nst == State.RUNNABLE)
            scheduler.execute(task);
    }
  
    private boolean casState(State oldState, State newState) { ... }  
}
```

# 调度器

如上所述，类似`ForkJoinPools`的工作窃取调度器特别适合用来调度那些经常阻塞或者频繁地与IO及其他线程交互的线程。当然，纤程也会有一些插件化的调度器，或者用户可以自定义实现（调度器的SPI同`Executor`接口一样简单方便）。从过往经验来看，预计异步模式下的`ForkJoinPool`可以在绝大部分场景下很好的支撑起纤程的调度，但是我们可能还会去调研一两种其他更简单的设计，如pinned-shceduler，一种总是会把给定的纤程固定调度到指定内核线程上的调度器（假设是固定到处理器）。

# 展开调用

与续体不同，展开的栈帧的内容不会被保留，且无需在任何对象中具象化此结构。

待续。

# 附加挑战

虽然实现此目标的主要动机是让并发更容易/更具可扩展性，但由Java运行时实现的线程以及那些运行时可以对其进行更多控制的线程，也有其他优势。比如，这种线程可以在一台机器上暂停并且序列化，然后在另一台机器上被反序列化后再恢复执行。这个特性在分布式系统中很有用，代码可以重新定位到更靠近其访问数据的位置，从而受益。或者在一个[函数即服务](https://en.wikipedia.org/wiki/Function_as_a_Service)的云平台上，一台运行用户代码的机器实例可以在其等待一些额外事件时终止线程，随后在另外一台实例上恢复执行，可能已经是不同的物理机器了。这样的话，可以更好的进行资源利用，且对于服务端和客户端来说也减少了耗时。当然，纤程也会有类似的方法，`parkAndSerialize`和`deserializeAndUnpark`。

由于纤程预期是可被序列化的，那么续体当然也应该可以。一旦它们是可序列化的，那么它们也应该是可被克隆的，因为克隆续体的能力实际上是增加表达力（它允许返回到一个之前的挂起点）。然而，要使得续体的克隆有足够的能力应用于这些场景面临着很大的挑战，因为Java代码在栈外还存储了大量的信息，克隆在某些定制化场景下就会很“深”。

# 其他实现

其他的可以替代纤程来解决并发简易性和性能之间的冲突的方案中，最有名的是aysnc/await。它已经在C#和Node.js中实现，而且大概率也会被标准的JavaScript所采纳。续体和纤程在aysnc/await中占据主导地位，因为它们很容易通过续体来实现（实际上，它可以被一种叫做非堆叠续体的弱形式的限定续体所实现，这种续体不会捕获整个调用栈，只会捕获单个子程序的本地上下文），但反之则不然。

虽然实现aysnc/await要比完整形态的续体和纤程更简单，但是此方案还远远不能解决问题。虽然async/await使得代码更加简单，并使其看起来像正常的顺序性代码。但与异步代码一样，它仍然需要对现有代码进行大量的改造，且需要在库中提供明确的支持，还不能很好地与同步代码互通。换句话说，它并未真正解决著名的["colored function" problem](http://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)。



[^1]: [https://en.wikipedia.org/wiki/Actor_model](https://en.wikipedia.org/wiki/Actor_model)
[^2]: [https://en.wikipedia.org/wiki/Dataflow_programming](https://en.wikipedia.org/wiki/Dataflow_programming)
[^3]: [https://en.wikipedia.org/wiki/Delimited_continuation](https://en.wikipedia.org/wiki/Delimited_continuation)
[^4]: [https://en.wikipedia.org/wiki/Coroutine](https://en.wikipedia.org/wiki/Coroutine)
[^5]: [https://en.wikipedia.org/wiki/Call_stack](https://en.wikipedia.org/wiki/Call_stack)
[^6]: [https://en.wikipedia.org/wiki/Call_stack#Unwinding](https://en.wikipedia.org/wiki/Call_stack#Unwinding)
[^7]: [https://en.wikipedia.org/wiki/Tail_call](https://en.wikipedia.org/wiki/Tail_call)
[^8]: [https://en.wikipedia.org/wiki/Data_striping](https://en.wikipedia.org/wiki/Data_striping)

[^9]: GPT的回答：一个approximation of a processor-local是指一种在计算机系统中轻松分配计算资源给各个处理器的方式。通常，处理器是计算机系统中执行计算的单个组件。一个approximation的processor-local是一种近似方法，它使得计算任务在多个处理器之间分配，从而实现更高效的计算能力和扩展性。通常，processor-local approximations可以通过以下方式实现：1. 数据划分：根据数据的特点将数据分为多个部分，并将这些部分分配给不同的处理器。2. 通信协议：通过多种协议，允许处理器之间共享数据，以实现分布式计算。3. 并行计算库：使用已有的并行计算库，可以帮助程序员更轻松地实现并行化。总之，一个processor-local approximation允许您根据计算任务和计算资源来分配任务和资源。这种近似方法有助于提高计算效率和扩展性。

[^10]: [https://en.wikipedia.org/wiki/Expressive_power_(computer_science)](https://en.wikipedia.org/wiki/Expressive_power_(computer_science))
[^11]: 是否将其称为续体或协程仍待定 — 它们的含义有所不同，但命名目前没有完全标准化，续体可能是更通用的术语
