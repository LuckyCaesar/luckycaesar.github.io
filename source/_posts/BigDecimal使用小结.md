---
title: BigDecimal使用小结
tags: [Java, BigDecimal]
index_img: /img/384bbbdb-money.jpeg
date: 2023-02-04 12:00:00
---



这些年接触过很多不同业务类型的系统，其中对数字敏感的系统也有很多，在Java中当仁不让的会大量使用BigDecimal，过程中不乏很多踩坑之处，因此做个简单的整理。



## 实例化

一般我们都会使用`new BigDecimal()`构造器和`BigDecimal.valueOf()`方法来实例化一个BigDecimal对象。这里特别要注意构造器入参为`double`类型时：

```java
// doubleDecimal = 0.1000000000000000055511151231257827021181583404541015625
BigDecimal doubleDecimal = new BigDecimal(0.1d);
// doubleDecimal1 = 1
BigDecimal doubleDecimal1 = new BigDecimal(1.0d);

// stringDecimal = 1.234
BigDecimal stringDecimal = new BigDecimal("1.234");

// doubleValueDecimal = 1.234，等同于 new BigDecimal("1.234")
BigDecimal doubleValueDecimal = BigDecimal.valueOf(1.234d);
```

可以看到`double`类型的入参会有所谓的”*精度丢失*“问题，所以在new时推荐使用字符串或其他几种类型参数的构造器，或者使用`.valueOf()`方法，避免出现这种问题。

对于这种”*精度丢失*“问题，在入参为`double`类型的构造器上有这样一段注释：

```tex
* The results of this constructor can be somewhat unpredictable.
* One might assume that writing {@code new BigDecimal(0.1)} in
* Java creates a {@code BigDecimal} which is exactly equal to
* 0.1 (an unscaled value of 1, with a scale of 1), but it is
* actually equal to
* 0.1000000000000000055511151231257827021181583404541015625.
* This is because 0.1 cannot be represented exactly as a
* {@code double} (or, for that matter, as a binary fraction of
* any finite length).  Thus, the value that is being passed
* <em>in</em> to the constructor is not exactly equal to 0.1,
* appearances notwithstanding.
```

简单来说就是代码中所传入的 0.1d 并不能真正的表示一个 0.1 的double数，实际上在内存中：

```tex
0.1d = 0.1000000000000000055511151231257827021181583404541015625
```

所以并不是精度真的丢失了，而是BigDecimal原原本本的返回了0.1 double数的真正值，只不过在程序代码逻辑里，我们要的是这个效果：`0.1d = 0.1`。

可以反过来看看这个例子，`.valueOf()`和`.toString()`会对 `d` 这个double数字进行近似运算，认为这个数字就是等于 0.1，在程序中会看起来更合理：

```java
double d = 0.1000000000000000055511151231257827021181583404541015625;

// decimal = 0.1000000000000000055511151231257827021181583404541015625
BigDecimal decimal = new BigDecimal(d);

// strValue = 0.1
String strValue = String.valueOf(d);

// doubleValue = 0.1
Double doubleValue = Double.valueOf(d);

// doubleValue1 = 0.1
String doubleValue1 = Double.toString(d);
```



除了用以上的方式进行实例化，BigDecimal也把0 - 10的数字初始化为一个缓存常量池，并暴露出几个常用数值，其余的数字会在调用一些实例化方法时判断，如果匹配就直接返回池中初始化好的对象：

```java
// Cache of common small BigDecimal values.
private static final BigDecimal ZERO_THROUGH_TEN[] = {
    new BigDecimal(BigInteger.ZERO,       0,  0, 1),
    new BigDecimal(BigInteger.ONE,        1,  0, 1),
    new BigDecimal(BigInteger.TWO,        2,  0, 1),
    new BigDecimal(BigInteger.valueOf(3), 3,  0, 1),
    new BigDecimal(BigInteger.valueOf(4), 4,  0, 1),
    new BigDecimal(BigInteger.valueOf(5), 5,  0, 1),
    new BigDecimal(BigInteger.valueOf(6), 6,  0, 1),
    new BigDecimal(BigInteger.valueOf(7), 7,  0, 1),
    new BigDecimal(BigInteger.valueOf(8), 8,  0, 1),
    new BigDecimal(BigInteger.valueOf(9), 9,  0, 1),
    new BigDecimal(BigInteger.TEN,        10, 0, 2),
};

// The value 0, with a scale of 0.
public static final BigDecimal ZERO = ZERO_THROUGH_TEN[0];

// The value 1, with a scale of 0.
public static final BigDecimal ONE = ZERO_THROUGH_TEN[1];

// The value 10, with a scale of 0.
public static final BigDecimal TEN = ZERO_THROUGH_TEN[10];

// The value 0.1, with a scale of 1.
private static final BigDecimal ONE_TENTH = valueOf(1L, 1);

// The value 0.5, with a scale of 1.
private static final BigDecimal ONE_HALF = valueOf(5L, 1);
```



## 舍入模式



#### RoundingMode.UP(BigDecimal.ROUND_UP)

向远离0的方向舍入，非0数字生效。正数变大，负数变小。

```java
BigDecimal roundingUp1 = new BigDecimal("1.2344").setScale(3, RoundingMode.UP);// = 1.235

BigDecimal roundingUp2 = new BigDecimal("1.2345").setScale(3, RoundingMode.UP);// = 1.235

BigDecimal roundingUp3 = new BigDecimal("1.2346").setScale(3, RoundingMode.UP);// = 1.235

BigDecimal roundingUp4 = new BigDecimal("-1.2344").setScale(3, RoundingMode.UP);// = -1.235

BigDecimal roundingUp5 = new BigDecimal("-1.2345").setScale(3, RoundingMode.UP);// = -1.235

BigDecimal roundingUp6 = new BigDecimal("-1.2346").setScale(3, RoundingMode.UP);// = -1.235

BigDecimal roundingUp7 = new BigDecimal("-1.23401").setScale(3, RoundingMode.UP);// = -1.235

BigDecimal roundingUp8 = new BigDecimal("1.23401").setScale(3, RoundingMode.UP);// = 1.235

BigDecimal roundingUp9 = new BigDecimal("-1.2340").setScale(3, RoundingMode.UP);// = -1.234

BigDecimal roundingUp10 = new BigDecimal("1.2340").setScale(3, RoundingMode.UP);// = 1.234
```



#### RoundingMode.DOWN(BigDecimal.ROUND_DOWN)

向靠近0的方向舍入，非0数字生效。正数变小，负数变大。

```java
BigDecimal roundingDown1 = new BigDecimal("1.2344").setScale(3, RoundingMode.DOWN);// = 1.234

BigDecimal roundingDown2 = new BigDecimal("1.2345").setScale(3, RoundingMode.DOWN);// = 1.234

BigDecimal roundingDown3 = new BigDecimal("1.2346").setScale(3, RoundingMode.DOWN);// = 1.234

BigDecimal roundingDown4 = new BigDecimal("-1.2344").setScale(3, RoundingMode.DOWN);// = -1.234

BigDecimal roundingDown5 = new BigDecimal("-1.2345").setScale(3, RoundingMode.DOWN);// = -1.234

BigDecimal roundingDown6 = new BigDecimal("-1.2346").setScale(3, RoundingMode.DOWN);// = -1.234

BigDecimal roundingDown7 = new BigDecimal("-1.23401").setScale(3, RoundingMode.DOWN);// = -1.234

BigDecimal roundingDown8 = new BigDecimal("1.23401").setScale(3, RoundingMode.DOWN);// = 1.234

BigDecimal roundingDown9 = new BigDecimal("-1.2340").setScale(3, RoundingMode.DOWN);// = -1.234

BigDecimal roundingDown10 = new BigDecimal("1.2340").setScale(3, RoundingMode.DOWN);// = 1.234
```



#### RoundingMode.CEILING(BigDecimal.ROUND_CEILING)

向正无穷方向舍入，非0数字生效。正负数都会变大。

```java
BigDecimal roundingCeiling1 = new BigDecimal("1.2344").setScale(3, RoundingMode.CEILING);// = 1.235

BigDecimal roundingCeiling2 = new BigDecimal("1.2345").setScale(3, RoundingMode.CEILING);// = 1.235

BigDecimal roundingCeiling3 = new BigDecimal("1.2346").setScale(3, RoundingMode.CEILING);// = 1.235

BigDecimal roundingCeiling4 = new BigDecimal("-1.2344").setScale(3, RoundingMode.CEILING);// = -1.234

BigDecimal roundingCeiling5 = new BigDecimal("-1.2345").setScale(3, RoundingMode.CEILING);// = -1.234

BigDecimal roundingCeiling6 = new BigDecimal("-1.2346").setScale(3, RoundingMode.CEILING);// = -1.234

BigDecimal roundingCeiling7 = new BigDecimal("-1.23401").setScale(3, RoundingMode.CEILING);// = -1.234

BigDecimal roundingCeiling8 = new BigDecimal("1.23401").setScale(3, RoundingMode.CEILING);// = 1.235

BigDecimal roundingCeiling9 = new BigDecimal("-1.2340").setScale(3, RoundingMode.CEILING);// = -1.234

BigDecimal roundingCeiling10 = new BigDecimal("1.2340").setScale(3, RoundingMode.CEILING);// = 1.234
```



#### RoundingMode.FLOOR(BigDecimal.ROUND_FLOOR)

向负无穷方向舍入，非0数字生效。正负数都会变小。

```java
BigDecimal roundingFloor1 = new BigDecimal("1.2344").setScale(3, RoundingMode.FLOOR);// = 1.234

BigDecimal roundingFloor2 = new BigDecimal("1.2345").setScale(3, RoundingMode.FLOOR);// = 1.234

BigDecimal roundingFloor3 = new BigDecimal("1.2346").setScale(3, RoundingMode.FLOOR);// = 1.234

BigDecimal roundingFloor4 = new BigDecimal("-1.2344").setScale(3, RoundingMode.FLOOR);// = -1.235

BigDecimal roundingFloor5 = new BigDecimal("-1.2345").setScale(3, RoundingMode.FLOOR);// = -1.235

BigDecimal roundingFloor6 = new BigDecimal("-1.2346").setScale(3, RoundingMode.FLOOR);// = -1.235

BigDecimal roundingFloor7 = new BigDecimal("-1.23401").setScale(3, RoundingMode.FLOOR);// = -1.235

BigDecimal roundingFloor8 = new BigDecimal("1.23401").setScale(3, RoundingMode.FLOOR);// = 1.234

BigDecimal roundingFloor9 = new BigDecimal("-1.2340").setScale(3, RoundingMode.FLOOR);// = -1.234

BigDecimal roundingFloor10 = new BigDecimal("1.2340").setScale(3, RoundingMode.FLOOR);// = 1.234
```



#### RoundingMode.HALF_UP(BigDecimal.ROUND_HALF_UP)

四舍五入。

```java
BigDecimal roundingHalfUp1 = new BigDecimal("1.2344").setScale(3, RoundingMode.HALF_UP);// = 1.234

BigDecimal roundingHalfUp2 = new BigDecimal("1.2345").setScale(3, RoundingMode.HALF_UP);// = 1.235

BigDecimal roundingHalfUp3 = new BigDecimal("1.2346").setScale(3, RoundingMode.HALF_UP);// = 1.235

BigDecimal roundingHalfUp4 = new BigDecimal("-1.2344").setScale(3, RoundingMode.HALF_UP);// = -1.234

BigDecimal roundingHalfUp5 = new BigDecimal("-1.2345").setScale(3, RoundingMode.HALF_UP);// = -1.235

BigDecimal roundingHalfUp6 = new BigDecimal("-1.2346").setScale(3, RoundingMode.HALF_UP);// = -1.235

BigDecimal roundingHalfUp7 = new BigDecimal("-1.2340").setScale(3, RoundingMode.HALF_UP);// = -1.234

BigDecimal roundingHalfUp8 = new BigDecimal("1.2340").setScale(3, RoundingMode.HALF_UP);// = 1.234
```



#### RoundingMode.HALF_DOWN(BigDecimal.ROUND_HALF_DOWN)

五舍六入。

```java
BigDecimal roundingHalfDown1 = new BigDecimal("1.2344").setScale(3, RoundingMode.HALF_DOWN);// = 1.234

BigDecimal roundingHalfDown2 = new BigDecimal("1.2345").setScale(3, RoundingMode.HALF_DOWN);// = 1.234

BigDecimal roundingHalfDown3 = new BigDecimal("1.2346").setScale(3, RoundingMode.HALF_DOWN);// = 1.235

BigDecimal roundingHalfDown4 = new BigDecimal("-1.2344").setScale(3, RoundingMode.HALF_DOWN);// = -1.234

BigDecimal roundingHalfDown5 = new BigDecimal("-1.2345").setScale(3, RoundingMode.HALF_DOWN);// = -1.234

BigDecimal roundingHalfDown6 = new BigDecimal("-1.2346").setScale(3, RoundingMode.HALF_DOWN);// = -1.235

BigDecimal roundingHalfDown7 = new BigDecimal("-1.2340").setScale(3, RoundingMode.HALF_DOWN);// = -1.234

BigDecimal roundingHalfDown8 = new BigDecimal("1.2340").setScale(3, RoundingMode.HALF_DOWN);// = 1.234
```



#### RoundingMode.HALF_EVEN(BigDecimal.ROUND_HALF_EVEN)

银行家舍入（四舍六入五取偶法）。四舍六入，五分两种情况，如果前一位为奇数，则入，为偶数，则舍。

```java

BigDecimal roundingHalfEven1 = new BigDecimal("1.2344").setScale(3, RoundingMode.HALF_EVEN);// = 1.234

BigDecimal roundingHalfEven2 = new BigDecimal("1.2345").setScale(3, RoundingMode.HALF_EVEN);// = 1.234

BigDecimal roundingHalfEven3 = new BigDecimal("1.2315").setScale(3, RoundingMode.HALF_EVEN);// = 1.232

BigDecimal roundingHalfEven4 = new BigDecimal("1.2346").setScale(3, RoundingMode.HALF_EVEN);// = 1.235

BigDecimal roundingHalfEven5 = new BigDecimal("-1.2344").setScale(3, RoundingMode.HALF_EVEN);// = -1.234

BigDecimal roundingHalfEven6 = new BigDecimal("-1.2345").setScale(3, RoundingMode.HALF_EVEN);// = -1.234

BigDecimal roundingHalfEven7 = new BigDecimal("-1.2315").setScale(3, RoundingMode.HALF_EVEN);// = -1.232

BigDecimal roundingHalfEven8 = new BigDecimal("-1.2346").setScale(3, RoundingMode.HALF_EVEN);// = -1.235

BigDecimal roundingHalfEven9 = new BigDecimal("-1.2340").setScale(3, RoundingMode.HALF_EVEN);// = -1.234

BigDecimal roundingHalfEven10 = new BigDecimal("1.2340").setScale(3, RoundingMode.HALF_EVEN);// = 1.234
```



#### RoundingMode.UNNECESSARY(BigDecimal.ROUND_UNNECESSARY)

断言传入的数字已经是指定长度精确的，否则抛出异常。

```java
BigDecimal rounding1 = new BigDecimal("1.234").setScale(3, RoundingMode.UNNECESSARY);// = 1.234

BigDecimal rounding2 = new BigDecimal("1.2340").setScale(3, RoundingMode.UNNECESSARY);// = 1.234

// "java.lang.ArithmeticException: Rounding necessary"
BigDecimal rounding3 = new BigDecimal("1.2341").setScale(3, RoundingMode.UNNECESSARY);
```



## 比较

在比较时，使用`BigDecimal.ZERO`等常量进行`equals`比较时，一定要注意精度。还有就是使用`equals`方法和`compareTo`方法进行比较可能会得到不同的结果：

```java
// b1 = false，要注意这个，BigDecimal.ZERO = 0
boolean b1 = BigDecimal.ZERO.equals(new BigDecimal("0.00"));
// b2 = true
boolean b2 = BigDecimal.ZERO.equals(new BigDecimal("0"));

// b3 = true
boolean b3 = BigDecimal.ZERO.compareTo(new BigDecimal("0")) == 0;
// b4 = true
boolean b4 = BigDecimal.ZERO.compareTo(new BigDecimal("0.00")) == 0;

// b5 = true
boolean b5 = new BigDecimal("0.00").compareTo(new BigDecimal("0.00")) == 0;
// b6 = true
boolean b6 = new BigDecimal("0.00").equals(new BigDecimal("0.00"));

// b7 = true
boolean b7 = new BigDecimal("0.0").compareTo(new BigDecimal("0.00")) == 0;
// b8 = false
boolean b8 = new BigDecimal("0.0").equals(new BigDecimal("0.00"));
```

总结两点：

- `equals`会对精度进行比较，所以同样的值的精度不同，会认为不相等。如 0.0 和 0.00
- `compareTo`方法更符合宏观比较算法，同样值的两个对象，精度一不一样都认为是相等的

推荐使用`compareTo`方法，尤其是在进行一些业务逻辑计算之后的比较，使用`equals`方法容易忽略精度问题（踩过坑 T＿T）。



## 小数点移位

通常要进行小数点移位，想到的都是乘以10、100或者除以10、100来实现，但是在BigDecimal里面，使用乘除法进行移位有很大的风险。

使用除法进行左移位操作，舍入模式和移位后需要保留的位数必须精确指定，否则就不是简单的移位了，会造成移位后精度损失。同样的，使用乘法进行右移位操作，也涉及到精度问题：

```java
// = 1.235
BigDecimal divide = new BigDecimal("123.456").divide(new BigDecimal("100"), RoundingMode.HALF_EVEN);

// = 1.23456
BigDecimal divide1 = new BigDecimal("123.456").divide(new BigDecimal("100"), 5, RoundingMode.UNNECESSARY);


// = 12345.61200
BigDecimal multiply = new BigDecimal("123.45612").multiply(new BigDecimal("100"));

// = 12345.61
BigDecimal multiply1 = new BigDecimal("123.45612").multiply(new BigDecimal("100"), MathContext.DECIMAL32);

// = 12345.61200
BigDecimal multiply2 = new BigDecimal("123.45612").multiply(new BigDecimal("100"), MathContext.DECIMAL64);

// = 12345.61200
BigDecimal multiply3 = new BigDecimal("123.45612").multiply(new BigDecimal("100"), MathContext.DECIMAL128);
```

推荐使用`movePointLeft`和`movePointRight`方法，简单方便不会出错：

```java
// = 1.23456
BigDecimal movePointLeft = new BigDecimal("123.456").movePointLeft(2);

// = 12345.612
BigDecimal movePointRight = new BigDecimal("123.45612").movePointRight(2);
```



## 乘除

两个点：

- **除以0！除以0！除以0！**

- 在计算完后必须使用`.setScale`方法来统一进行最终结果的精度约束。如果只有除法，那必须使用提供了scale和roundingMode参数的`.divide`方法，因为除法很容易除不尽，会产生无限循环小数。

