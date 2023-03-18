---
title: 函数式编程增强库Vavr初探（一）
tags: [函数式编程, vavr]
index_img: /img/vavr00.jpeg
date: 2022-10-30 15:00:00
---



[Vavr](https://github.com/vavr-io/vavr)是对Java8（及以上版本）的函数式编程体验的增强库，官方介绍如下：

> Vavr is an object-functional language extension to Java 8, which aims to reduce the lines of code and increase code quality. It provides persistent collections, functional abstractions for error handling, concurrent programming, pattern matching and much more.
>
> Vavr是对Java8的对象-函数式语言的扩展，目的是减少代码行数并提高代码质量。提供了持久化集合，针对错误处理的函数式抽象，并发编程，模式匹配以及其他更丰富的内容。

本文主要是翻译至Vavr官方文档中的[使用指南](https://docs.vavr.io/#_usage_guide)部分，会出现一些术语，主要参考维基百科。



# 3. 使用指南

Vavr带来的是对一些最基础类型 — 很明显在Java中是缺失的或者（实现地）比较简陋的，精心设计的表征：`Tuple`，`Value`和`λ`。

在Vavr中，一切（实现）都是基于这三个最基本的构建块（building blocks）：

![](/img/vavr01.jpeg)



## 3.1. 多元组[^1]

Java里面没有一个针对多元组的整体概念。一个Tuple可以将固定数量的元素组合在一起作为一个整体进行传递。不同于Array或者List，它可以持有不同类型的对象，但同时也是不可变的。

多元组的类型有Tuple1，Tuple2，Tuple3等等，目前最多支持8个元素。对于一个多元组 `t`，可以通过`t._1`方法访问它的第一个元素，`t._2`方法访问第二个，以此类推。

### 3.1.1. 创建多元组

接下来通过一个示例来演示如何创建一个持有String和Integer两种类型的元素的多元组：

```java
// (Java，8)
Tuple2<String, Integer> java8 = Tuple.of("Java", 8); ❶

// "Java"
String s = java8._1; ❷

// 8
Integer i = java8._2; ❸
```

❶ 通过静态工厂方法`Tuple.of()`来创建一个多元组。

❷ 获取这个多元组的第一个元素。

❸ 获取这个多元组的第二个元素。

### 3.1.2. 组件映射

组件映射就是为多元组的每一个元素执行一个函数，然后返回一个新的多元组：

```java
// (vavr, 1)
Tuple2<String, Integer> that = java8.map(
		s -> s.substring(2) + "vr",
    	i -> i / 8
);
```

### 3.1.3. 单映射器映射

当然，也可以通过一个映射函数来映射一个多元组（中的所有元素）：

```java
// (vavr, 1)
Tuple2<String, Integer> that = java8.map(
		(s, i) -> Tuple.of(s.substring(2) + "vr", i / 8)
);
```

### 3.1.4. 多元组变换

根据多元组的内容，变换操作可以得到一个新的类型的值：

```java
// "vavr 1"
String that = java8.apply(
		(s, i) -> s.substring(2) + "vr " + i / 8
);
```



## 3.2. 函数

函数式编程的所有工作都与值有关并能使用函数对值进行变换。Java8仅提供了一个接收单个参数的`Function`和一个接收两个参数的`BiFunction`，Vavr则把这个参数上限提高到了8个。这些函数式接口被命名为`Function0,Function1,Function2,Function3`等以此类推。如果需要能抛出检查异常的函数，则可以使用`CheckedFunction1,CheckedFunction2`等等。

下面的lambda表达式创建了一个两数相加的函数：

```java
// sum.apply(1, 2) = 3
Function2<Integer, Integer, Integer> sum = (a,b) -> a + b;
```

这种实现是对下面这种匿名类实现的简写：

```java
Function2<Integer, Integer, Integer> sum = new Function2<Integer, Integer, Integer>() {
    @Override
    public Integer apply(Integer a, Integer b) {
        return a + b;
    }
};
```

当然也可以使用静态工厂方法`Function3.of(...)`接收任意的方法引用来创建函数：

```java
Function3<String, String, String, String> function3 =
        Function3.of(this::methodWhichAccepts3Parameters);
```

实际上，Vavr的函数式接口是针对Java8函数式接口的“类固醇”（意思是增强），也提供了更多的特性如：

- 组合（Composition）
- 提升（Lifting）
- 柯里化（Currying）
- 记忆化（Memoization）

### 3.2.1. 组合

对函数可以进行组合。在数学中，函数组合是指把一个函数应用到另一个函数的结果当中从而产生第三个函数。举个例子，函数`f: X → Y`和函数`g: Y → Z`可以组合生成一个新的函数`h:g(f(X))`就能得到`X → Z`的映射。

（要实现组合）可以使用`andThen`方法：

```java
Function1<Integer, Integer> plusOne = a -> a + 1;
Function1<Integer, Integer> multiplyByTwo = a -> a * 2;

Function1<Integer, Integer> add1AndMultiplyBy2 = plusOne.andThen(multiplyByTwo);

then(add1AndMultiplyBy2.apply(2)).isEqualTo(6);
```

或者`compose`方法：

```java
Function1<Integer, Integer> add1AndMultiplyBy2 = multiplyByTwo.compose(plusOne);

then(add1AndMultiplyBy2.apply(2)).isEqualTo(6);
```

### 3.2.2. 提升

我们可以把一个偏函数[^2]提升为一个返回`Option`结果的全函数[^2]。*偏函数* 是数学中的术语，一个从X到Y的偏函数就是函数 f: X' → Y，X'是X的某个子集。它通过不强制函数f将X中的每一个元素都映射到Y中对应的元素来概括一个函数 f: X → Y的概念。这就意味着偏函数仅适用于某些输入值，如果有非法的输入值调用了函数，那么通常会抛出异常。

下面的方法`divide`是一个只接收非零除数的偏函数：

```java
Function2<Integer, Integer, Integer> divide = (a, b) -> a / b;
```

我们可以使用`lift`方法把`divide`变成一个可以接受所有输入的全函数：

```java
Function2<Integer, Integer, Option<Integer>> safeDivide = Function2.lift(divide);

// = None
Option<Integer> i1 = safeDivide.apply(1, 0); ❶

// = Some(2)
Option<Integer> i2 = safeDivide.apply(4, 2); ❷
```

❶ 如果使用非法的输入值执行函数，被提升的函数会返回`None`来替代抛出异常。

❷ 如果使用合法的输入值执行函数，被提升的函数会返回`Some`。

下面的方法`sum`是一个只接收正数输入的偏函数：

```java
int sum(int first, int second) {
    if (first < 0 || second < 0) {
        throw new IllegalArgumentException("Only positive integers are allowed"); ❶
    }
    return first + second;
}
```

❶ 传入负数，函数`sum`会抛出`IllegalArgumentException`。

我们也可以通过方法引用来提升`sum`方法：

```java
Function2<Integer, Integer, Option<Integer>> sum = Function2.lift(this::sum);

// = None
Option<Integer> optionalResult = sum.apply(-1, 2); ❶
```

❶ 被提升的函数会捕捉到`IllegalArgumentException`并映射成`None`进行返回。

### 3.2.3. 偏函数应用（Partial application）

偏函数应用允许你用一个已有的函数通过固定某些值（指参数值）来派生出一个新的函数。你可以固定一个或者多个参数，而且要固定的参数数量会决定新函数的参数数量，这样`新参数数量 = （原参数数量 - 要固定的参数数量）`。参数都是从左向右依次绑定的。

```java
Function2<Integer, Integer, Integer> sum = (a, b) -> a + b;
Function1<Integer, Integer> add2 = sum.apply(2); ❶

then(add2.apply(4)).isEqualTo(6);
```

❶ 第一个参数`a`的值被固定为2。

接下来展示的是一个固定了前3个参数的`Function5`，返回的是一个`Function2`：

```java
Function5<Integer, Integer, Integer, Integer, Integer, Integer> sum = (a, b, c, d, e) -> a + b + c + d + e;
Function2<Integer, Integer, Integer> add6 = sum.apply(2, 3, 1); ❶

then(add6.apply(4, 3)).isEqualTo(13);
```

❶ 参数`a`，`b`，`c`的值被分别固定成了2，3，1。

偏函数应用不同于柯里化，会在接下来的章节中进行讨论。

### 3.2.4. 柯里化

柯里化是一种通过固定其中一个参数的值来部分的应用一个函数的技术，从而得到一个返回`Function1`的`Function1`函数。

当函数`Function2`被*柯里化*后，得到的结果和对`Function2`进行*偏函数应用*的结果很难区别，因为两者都是一个一元函数。

```java
Function2<Integer, Integer, Integer> sum = (a, b) -> a + b;
Function1<Integer, Integer> add2 = sum.curried().apply(2); ❶

then(add2.apply(4)).isEqualTo(6);
```

❶ 第一个参数的值被固定为2。

这时你可能注意到了，除了使用了`.curried()`方法以外，这部分代码和[偏函数应用](#3.2.3.-偏函数应用（Partial-application）)章节中给出的二元函数的例子是一模一样的。但是，在更多元的函数中，它们之间的区别会变得越来越明显。

```java
Function3<Integer, Integer, Integer, Integer> sum = (a, b, c) -> a + b + c;
final Function1<Integer, Function1<Integer, Integer>> add2 = sum.curried().apply(2); ❶

then(add2.apply(4).apply(3)).isEqualTo(9); ❷
```

❶ 注意参数中存在的额外的函数。

❷ 除了最后一次调用以外，对`apply`的进一步调用会返回不同的函数`Function1`。

### 3.2.5. 记忆化

记忆化是缓存的一种形式。一个有记忆的函数只会执行一次然后（之后的执行）会从缓存中取值进行返回。在下面的示例中，第一次执行会计算得到一个随机数然后在第二次执行时会返回缓存的这个数。

```java
Function0<Double> hashCache =
    Function0.of(Math::random).memoized();

double randomValue1 = hashCache.apply();
double randomValue2 = hashCache.apply();

then(randomValue1).isEqualTo(randomValue2);
```



## 3.3. 值

在函数式环境中，我们把值认作是一种[范式](https://en.wikipedia.org/wiki/Normal_form_(abstract_rewriting))（normal form），一个无法进一步求值的表达式。在Java中我们通过把一个对象的状态设置为final并称它为[不可变对象](https://en.wikipedia.org/wiki/Immutable_object)来表达这种含义。

Vavr中的函数值则对不可变对象进行了抽象，通过在实例之间共享不可变内存实现了高效的写操作。我们就"免费"实现了线程安全！（共享不可变内存这里我理解为Java中final或static final的变量，天然具备线程安全的特性，无需花费额外的成本。）

### 3.3.1. Option

Option是一个单子[^3]容器类型，代表着一个可选值。Option的实例要么是`Some`的实例，要么是`None`的实例。

```java
// optional *value*, no more nulls
Option<T> option = Option.of(...);
```

如果你在使用了Java中的`Optional`类后再接触Vavr（中的`Option`），会发现一个很关键的不同点。在`Optional`里面，调用`.map`方法然后使其返回null会得到一个空的`Optional`。而在Vavr里面，会得到一个`Some(null)`然后导致`NullPointerException`。

使用`Optional`，在下面的场景中是合理的。

```java
Optional<String> maybeFoo = Optional.of("foo"); ❶
then(maybeFoo.get()).isEqualTo("foo");
Optional<String> maybeFooBar = maybeFoo.map(s -> (String)null) ❷
                                       .map(s -> s.toUpperCase() + "bar");
then(maybeFooBar.isPresent()).isFalse();
```

❶ 可选项是`Some("foo")`。

❷ 结果选项在这里就变成了空。

而使用了Vavr的`Option`，在同样的场景下会返回`NullPointerException`。

```java
Option<String> maybeFoo = Option.of("foo"); ❶
then(maybeFoo.get()).isEqualTo("foo");
try {
    maybeFoo.map(s -> (String)null) ❷
            .map(s -> s.toUpperCase() + "bar"); ❸
    Assert.fail();
} catch (NullPointerException e) {
    // this is clearly not the correct approach
}
```

❶ 可选项是`Some("foo")`。

❷ 结果选项在这里会是`Some(null)`。

❸ 在`null`上去执行`s.toUpperCase()`的调用（会抛错）。

看起来Vavr的实现好像违背了一些准则，但实际上并没有，它始终遵循着单子在调用`.map`时维护计算上下文的要求。对于一个`Option`而言，这就意味着在`Some`上调用`.map`会返回`Some`，在`None`上调用`.map`会返回`None`。而在上面的Java的`Optional`例子中，上下文从`Some`变成了`None`。

这样看起来`Option`好像没什么用，但实际上它会强制你关注可能会出现`null`的场景并能合理的处理它们而不是在不知不觉中接受了它们，而处理`null`的合理的方式就是使用`flatMap`。

```java
Option<String> maybeFoo = Option.of("foo"); ❶
then(maybeFoo.get()).isEqualTo("foo");
Option<String> maybeFooBar = maybeFoo.map(s -> (String) null) ❷
                                .flatMap(s -> Option.of(s) ❸
                                    .map(t -> t.toUpperCase() + "bar"));
then(maybeFooBar.isEmpty()).isTrue();
```

❶ 可选项是`Some("foo")`。

❷ 结果选项在这里会是`Some(null)`。

❸ `s`，在这里它的值是`null`，会变成`None`。

或者，可以把`.flatMap`直接放到有可能出现`null`值的地方。

```java
Option<String> maybeFoo = Option.of("foo"); ❶
then(maybeFoo.get()).isEqualTo("foo");
Option<String> maybeFooBar = maybeFoo.flatMap(s -> Option.of((String)null)) ❷
                                     .map(s -> s.toUpperCase() + "bar");
then(maybeFooBar.isEmpty()).isTrue();
```

❶ 可选项是`Some("foo")`。

❷ 结果选项是`None`。

[Vavr博客](http://blog.vavr.io/the-agonizing-death-of-an-astronaut/)中对此进行了更详细的探讨。

### 3.3.2. Try

Try是一个单子容器类型，表示一次要么返回异常，要么返回计算成功的值的计算。和`Either`有点类似，但在语义上又有所区别。Try的实例都是`Success`或者`Failure`的实例。

```java
// no need to handle exceptions
Try.of(() -> bunchOfWork()).getOrElse(other);
```

```java
import static io.vavr.API.*;        // $, Case, Match
import static io.vavr.Predicates.*; // instanceOf

A result = Try.of(this::bunchOfWork)
    .recover(x -> Match(x).of(
        Case($(instanceOf(Exception_1.class)), t -> somethingWithException(t)),
        Case($(instanceOf(Exception_2.class)), t -> somethingWithException(t)),
        Case($(instanceOf(Exception_n.class)), t -> somethingWithException(t))
    ))
    .getOrElse(other);
```

### 3.3.3. Lazy

Lazy是一个单子容器类型，表示一个惰性计算的值。同Supplier相比，Lazy是有记忆的，即它只会执行一次并由此具有了[引用透明性](https://en.wikipedia.org/wiki/Referential_transparency)（一个表达式在程序中可以被它等价的值替换，而不影响结果）。

```java
Lazy<Double> lazy = Lazy.of(Math::random);
lazy.isEvaluated(); // = false
lazy.get();         // = 0.123 (random generated)
lazy.isEvaluated(); // = true
lazy.get();         // = 0.123 (memoized)
```

你也可以（通过Lazy）创建一个真正的惰性值（只适用于接口（interfaces））：

```java
CharSequence chars = Lazy.val(() -> "Yay!", CharSequence.class); // 第二个参数是一个interface
```

### 3.3.4. Either

Either表示的是两种可能类型的值。一个Either只会是Left或者Right。如果一个给定的Either是Right，然后把它投射给Left，那么对Left的操作不会对Right的值有任何影响。同样的，如果一个给定的Either是Left，然后把它投射给Right，那么对Right的操作也不会对Left的值有任何影响。如果Left投射给Left或者Right投射给Right，那么这些操作就会互相产生影响。

例：一个compute()函数，返回了一个Integer值（成功的情况下）或者返回一个String类型的错误信息（失败的情况下）。按惯例，成功的情况下返回Right，失败的情况下返回Left。

```java
Either<String,Integer> value = compute().right().map(i -> i * 2).toEither();
```

如果compute()的结果是Right(1)，value的值就是Right(2)。

如果compute()的结果是Left("error")，value的值就是Left("error")。

### 3.3.5. Future

Future是一个在某个不确定的时刻才会变得可用的计算结果。它提供的所有操作都是非阻塞的，底层的ExecutorService通常被用做执行异步处理程序，比如onComplete(...)。

一个Future有两种状态：等待中和已完成。

**等待中：** 计算正在进行当中，只有一个处于等待中的future才能被标记为已完成或者已撤销。

**已完成：** 计算完成后，返回结果就是成功，返回异常或者被撤销则是失败。

回调可以在任意时间点上被注册到Future。当Future完成时，这些（回调）动作会被执行。注册到已完成的Future上的动作会被立即执行，它可能会运行在一个独立的线程中，这取决于底层的ExecutorService。而那些注册到被撤销的Future上的动作会带着失败的结果被执行。

```java
// future *value*, result of an async calculation
Future<T> future = Future.of(...);
```

### 3.3.6. Validation

Validation控件是一个*应用式函子*[^4]，有助于累积错误。在我们尝试组合单子时，会在第一次遇到错误时短路。但是'Validation'可以继续这个组合过程，并累积所有错误。这在对多个字段做校验时尤其有用，比方说一个web表单，你肯定是想一次请求拿到所有可能遇见的错误，而不是一次一个。

例：在一个web表单中，有'name'和'age'两个字段，希望（提交后）要么创建一个有效的Person实例，要么返回校验的错误列表。

```java
PersonValidator personValidator = new PersonValidator();

// Valid(Person(John Dow, 30))
Validation<Seq<String>, Person> valid = personValidator.validatePerson("John Doe", 30);

// Invalid(List(Name contains invalid characters: '!4?', Age must be greater than 0))
Validation<Seq<String>, Person> invalid = personValidator.validatePerson("John? Doe!4", -1);
```

`Validation.Valid`实例包含了一个有效的值，而`Validation.Invalid`实例则包含了一组校验的错误列表。

下面的这个校验器就是用来把不同的校验结果合并成一个`Validation`实例：

```java
class PersonValidator {

    private static final String VALID_NAME_CHARS = "[a-zA-Z ]";
    private static final int MIN_AGE = 0;

    public Validation<Seq<String>, Person> validatePerson(String name, int age) {
        return Validation.combine(validateName(name), validateAge(age)).ap(Person::new);
    }

    private Validation<String, String> validateName(String name) {
        return CharSeq.of(name).replaceAll(VALID_NAME_CHARS, "").transform(seq -> seq.isEmpty()
                ? Validation.valid(name)
                : Validation.invalid("Name contains invalid characters: '"
                + seq.distinct().sorted() + "'"));
    }

    private Validation<String, Integer> validateAge(int age) {
        return age < MIN_AGE
                ? Validation.invalid("Age must be at least " + MIN_AGE)
                : Validation.valid(age);
    }

}
```

如果校验成功，即输入的数据是有效的，那么一个`Person`实例会根据给定的字段`name`和`age`被创建。

```java
class Person {

    public final String name;
    public final int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person(" + name + ", " + age + ")";
    }

}
```



## 3.4 集合

为了满足函数式编程的要求，即所谓不变性，我们做了大量的工作来为Java设计一个全新的集合库。

Java中的Stream会将计算拔高到一个不同的层次，然后在另一个显式步骤中把它关联到某个特定的集合。而在Vavr中我们抛弃了所有这些额外的公式化的东西。

新的集合是基于[java.lang.Iterable](http://docs.oracle.com/javase/8/docs/api/java/lang/Iterable.html)（来实现的），所以它们也利用了迭代风格的语法糖。

```java
// 1000 random numbers
for (double random : Stream.continually(Math::random).take(1000)) {
    ...
}
```

`TraversableOnce`拥有大量实用的函数来操作集合，它的API和[java.util.stream.Stream](http://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)类似，但却更加成熟。

### 3.4.1. List

Vavr中的`List`是一个不可变的链表。（对List的）变种（Mutations）会创建新的实例。大多数操作会在线性时间内执行，产生结果的操作将依次执行。

#### Java8

```java
Arrays.asList(1, 2, 3).stream().reduce((i, j) -> i + j);
```

```java
IntStream.of(1, 2, 3).sum();
```

#### Vavr

```java
// io.vavr.collection.List
List.of(1, 2, 3).sum();
```

### 3.4.2. Stream

`io.vavr.collection.Stream`的（底层）实现是一个惰性的链表，只有在需要时才会对值进行计算。由于它惰性的特性，大多数操作都会在常数时间内执行。操作通常会处于中间且会在一次遍历中被执行。

streams最妙的是我们可以用它们来表示（理论上）无限长的序列。

```java
// 2, 4, 6, ...
Stream.from(1).filter(i -> i % 2 == 0);
```

### 3.4.3. 性能特点

表1. 序列操作的时间复杂度

|               | head()     | tail()     | get(int)   | update(int, T) | prepend(T)  | append(T)   |
| :------------ | :--------- | :--------- | :--------- | :------------- | :---------- | ----------- |
| Array         | const      | linear     | const      | const          | linear      | linear      |
| CharSeq       | const      | linear     | const      | linear         | linear      | linear      |
| Iterator      | const      | const      | —          | —              | —           | —           |
| List          | const      | const      | linear     | linear         | const       | linear      |
| Queue         | const      | const^a^   | linear     | linear         | const       | const       |
| PriorityQueue | log        | log        | —          | —              | log         | log         |
| Stream        | const      | const      | linear     | linear         | const^lazy^ | const^lazy^ |
| Vector        | const^eff^ | const^eff^ | const^eff^ | const^eff^     | const^eff^  | const^eff^  |


表2. Map/Set操作的时间复杂度

|               | contains/Key | add/put    | remove     | min    |
| :------------ | :----------- | :--------- | :--------- | :----- |
| HashMap       | const^eff^   | const^eff^ | const^eff^ | linear |
| HashSet       | const^eff^   | const^eff^ | const^eff^ | linear |
| LinkedHashMap | const^eff^   | linear     | linear     | linear |
| LinkedHashSet | const^eff^   | linear     | linear     | linear |
| Tree          | log          | log        | log        | log    |
| TreeMap       | log          | log        | log        | log    |
| TreeSet       | log          | log        | log        | log    |

说明：

- const — 常数时间
- const^a^ — 摊还常数时间，少数操作可能开销大一点
- const^eff^ — 有效常数时间，取决于一些假定比如hash键的分布
- const^lazy^ — 惰性常数时间，操作是递延的
- log — 对数时间
- linear — 线性时间



## 3.5 性质检查[^5]

性质检查（也被称作[性质检验](https://en.wikipedia.org/wiki/Property_testing)）是一个很强有力的手段，可以通过函数式的方式帮助我们测试代码的性质。它基于生成的随机数据，再把这些数据传递给用户定义的检查函数。

Vavr在其`io.vavr:vavr-test`模块中有对性质检查的支持，所以在测试中要使用它的话得确保引入了这个模块。

```java
Arbitrary<Integer> ints = Arbitrary.integer();

// square(int) >= 0: OK, passed 1000 tests.
Property.def("square(int) >= 0")
        .forAll(ints)
        .suchThat(i -> i * i >= 0)
        .check()
        .assertIsSatisfied();
```

复杂数据结构的生成器都是由一些简单生成器组成的。



## 3.6. 模式匹配

Scala天然拥有模式匹配的特性，相对于*朴素*的Java来说这是一个优点。其基础语法和Java的switch类似：

```scala
val s = i match {
  case 1 => "one"
  case 2 => "two"
  case _ => "?"
}
```

注意*match*是一个表达式，它会产出一个结果。此外，它还提供了：

- 具名参数 `case i: Int ⇒ "Int " + i`
- 对象解构 `case Some(i) ⇒ i`
- 卫[^7] `case Some(i) if i > 0 ⇒ "positive " + i`
- 多重条件 `case "-h" | "--help" ⇒ displayHelp`
- 编译期*穷尽*检查

模式匹配是一个很棒的特性。它让我们从一堆if-then-else分支中解放了出来，减少了代码量的同时，也能专注于更有意义的部分。

### 3.6.1. Java匹配的基础知识

Vavr提供了一个和Scala的match相似的macth API，添加如下的引用可以启用它：

```java
import static io.vavr.API.*;
```

它提供了静态方法*Match*，*Case*，还有所谓的 *atomic patterns*：

- `$()` - 通配符模式
- `$(value)` - 等价模式
- `$(predicate)` - 条件模式

在（一定的/合理的）范围内，最初Scala的示例可以用如下的方式进行实现：

```java
String s = Match(i).of(
    Case($(1), "one"),
    Case($(2), "two"),
    Case($(), "?")
);
```

⚡ 我们的方法名（Case）统一使用了大写开头，因为Java中的'case'是一个关键字，这让这个API有点特殊。

#### 穷尽

最后那个通配符模式`$()`能帮我们从抛出的匹配错误中解放出来，如果没有任何分支匹配上的话。由于我们无法像Scala编译器那样执行穷尽检查，所以我们提供了可选结果的返回：

```java
Option<String> s = Match(i).option(
    Case($(0), "zero")
);
```

#### 语法糖

如上所述，`Case`允许匹配条件模式。

```java
Case($(predicate), ...)
```

Vavr提供了一组默认的断言。

```java
import static io.vavr.Predicates.*;
```

它可以用来实现最初的那个Scala的例子：

```java
String s = Match(i).of(
    Case($(is(1)), "one"),
    Case($(is(2)), "two"),
    Case($(), "?")
);
```

**多重条件**

可以用`isIn`的断言来检查多重条件：

```java
Case($(isIn("-h", "--help")), ...)
```

**处理副作用[^6]**

Match就像一个表达式，它会返回一个值。为了处理副作用我们要用到一个返回`Void`的辅助函数`run`：

```java
Match(arg).of(
    Case($(isIn("-h", "--help")), o -> run(this::displayHelp)),
    Case($(isIn("-v", "--version")), o -> run(this::displayVersion)),
    Case($(), o -> run(() -> {
        throw new IllegalArgumentException(arg);
    }))
);
```

⚡ `run`用于消除歧义，因为`void`在Java中不是一个有效的返回值。

**注意：**`run`不能当作直接返回值，也就是不能放到lambda表达式的外面：

```java
// Wrong!
Case($(isIn("-h", "--help")), run(this::displayHelp))
```

否则，分支就会在模式匹配命中*之前*执行，这会导致整个匹配表达式的中断。相反我们应该把它放在lambda表达式里面：

```java
// Ok
Case($(isIn("-h", "--help")), o -> run(this::displayHelp))
```

总之，`run`的使用不当很容易造成错误，一定要小心。我们正考虑可能在之后的某个release版本中把它废弃掉，然后会提供一个更好的API来处理函数副作用。

#### 具名参数

Vavr使用了lambda的方式为匹配的值提供了具名参数。

```java
Number plusOne = Match(obj).of(
    Case($(instanceOf(Integer.class)), i -> i + 1),
    Case($(instanceOf(Double.class)), d -> d + 1),
    Case($(), o -> { throw new NumberFormatException(); })
);
```

截止目前，我们都是采用原子模式来做值匹配。如果一个原子模式匹配成功，那么匹配对象的真正类型则是从模式的下文中推断出来的。

接下来，我们来看看能够匹配（理论上）任意深度的对象图的递归模式。

#### 对象分解

Java中使用构造器来实例化类。我们可以把*对象分解*理解成把对象分解为它的各个部分。

构造器是一个可以施加参数然后返回新实例的函数，那相应的解构器就是一个接收实例（参数）然后返回某部分的函数。这时我们就说一个对象*unapplied*。

对象解构不一定是一个唯一操作。比方说，LocalDate能够被分解成：

- 年、月、日组件
- 代表对应某个时刻的纪元毫秒的long值
- ...

### 3.6.2. 模式

在Vavr中我们使用模式来描述某个特定类型的实例是如何被解构的。这些模式可以和Match API结合在一起使用。

#### 预定义的模式

针对Vavr中的很多类型都已经有对应的匹配模式了，可以通过如下方式引入它们：

```java
import static io.vavr.Patterns.*;
```

比如说我们现在需要匹配一个Try的结果：

```java
Match(_try).of(
    Case($Success($()), value -> ...),
    Case($Failure($()), x -> ...)
);
```

⚡ Vavr中第一个雏形Match API是允许从匹配模式中提取用户自定义的对象选择。但是如果没有合适的编译器的支持，这肯定是行不通的，因为生成的方法数量会呈指数级增长。目前的API做出了妥协，即所有模式都匹配，但只有根模式会被*分解*。

```java
Match(_try).of(
    Case($Success($Tuple2($("a"), $())), tuple2 -> ...),
    Case($Failure($(instanceOf(Error.class))), error -> ...)
);
```

可以看到这里的Success和Failure是根模式，它们被分解成了Tuple2和Error，拥有了正确的泛型类型。

⚡ 深度嵌套的类型是根据匹配参数而不是匹配模式来推断的。

#### 用户自定义模式

能够unapply（即[对象分解](#对象分解)提到的概念）任意对象，包括不可变类的实例，都是至关重要的。Vavr提供了编译期注解`@Patterns`和`@Unapply`，以声明式的方式实现了这一点。

要启用annotation processor，需要项目中依赖[vavr-match](http://search.maven.org/#search|ga|1|vavr-match)包。

⚡ 注：当然我们也可以不通过代码生成器而是直接实现这些模式。想了解有关更多的信息，可以查看生成的源码。

```java
import io.vavr.match.annotation.*;

@Patterns
class My {

    @Unapply
    static <T> Tuple1<T> Optional(java.util.Optional<T> optional) {
        return Tuple.of(optional.orElse(null));
    }
}
```

annotation processor放了一个文件MyPatterns在同一个包中（默认是在target/generated-sources下面），也支持内部类。

特殊情况：如果类名是$，那么生成的类名就只是个不带前缀的Patterns。

#### 卫[^7]

现在我们用*卫*来实现对Optionals的匹配。

```java
Match(optional).of(
    Case($Optional($(v -> v != null)), "defined"),
    Case($Optional($(v -> v == null)), "empty")
);
```

可以通过实现`isNull`和`isNotNull`来简化断言。

⚡ 是的，把null提取出来看起来很奇怪对不对。那么来尝试一下Vavr中的Option吧，换掉Java中的Optional！

```java
Match(option).of(
    Case($Some($()), "defined"),
    Case($None(), "empty")
);
```



[^1]: [多元组](https://zh.wikipedia.org/wiki/%E5%A4%9A%E5%85%83%E7%BB%84)，也称为顺序组（英语：Tuple），泛指有限个元素所组成的序列。在数学及计算机科学分别有其特殊的意义。
[^2]:偏函数和全函数的定义百科中都对应一个词条：https://en.wikipedia.org/wiki/Partial_function
[^3]:在函数式编程中，[单子](https://zh.wikipedia.org/wiki/单子_(函数式编程))（monad）是一种抽象，它允许以泛型方式构造程序。支持它的语言可以使用单子来抽象出程序逻辑需要的[样板代码](https://zh.wikipedia.org/w/index.php?title=样板代码&action=edit&redlink=1)。
[^4]:在函数式编程中，[应用式函子](https://zh.wikipedia.org/wiki/%E5%BA%94%E7%94%A8%E5%BC%8F%E5%87%BD%E5%AD%90)，或简称应用式（applicative），是在[函子](https://zh.wikipedia.org/wiki/函子_(函数式编程))和[单子](https://zh.wikipedia.org/wiki/单子_(函数式编程))之间的中间结构。应用式函子允许函子式计算成为序列（不同于平常函子），但是不允许使用前面计算的结果于后续计算的定义之中（不同于单子）。
[^5]:Property Checking。
[^6]:在计算机科学中，[函数副作用](https://zh.wikipedia.org/wiki/%E5%89%AF%E4%BD%9C%E7%94%A8_(%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6))指当调用函数时，除了返回可能的函数值之外，还对主调用函数产生附加的影响。
[^7]:在计算机程序设计中，[卫](https://zh.wikipedia.org/wiki/%E5%8D%AB%E8%AF%AD%E5%8F%A5)（guard）是布尔表达式，其结果必须为真，程序才能执行下去。卫语句（guard code或guard clause）用于检查[先决条件](https://zh.wikipedia.org/wiki/先决条件)。