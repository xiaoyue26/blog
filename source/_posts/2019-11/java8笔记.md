---
title: java8笔记
date: 2019-11-03 20:56:11
tags: java
categories: 
- java

---


# lambda语法
`i->{return "xxx";}`
本质上是借用匿名类来实现,并没有超脱原来的java的解题思路。

这种类似于传一个函数指针的模式，底层是`@FunctionalInterface`注解的接口，因此我们可以自己声明一个接口作为方法的行参:
```java
 @FunctionalInterface
    public interface GetIdataFunction {
        int getI(int i);

        default String getInfo() {
            return "找下标为i的数据";
        }

    }

    public boolean searchMatrix(int[][] matrix, int target) {
        if (matrix == null || matrix.length == 0 || matrix[0].length == 0 || target < matrix[0][0]) {
            return false;
        }

        int lowerBound = getLowerBound(target, 0, matrix.length - 1, i -> matrix[i][0]);
        if (lowerBound >= 0) {
            return true;
        }
        final int finalLowerBound = -lowerBound - 2;
        int x = getLowerBound(target, 0, matrix[0].length - 1, i -> matrix[finalLowerBound][i]);
        return x >= 0;
    }

    private int getLowerBound(int target, int left, int right, GetIdataFunction helper) {
        int mid;
        while (left <= right) {
            mid = (left + right) >>> 1;
            int midVal = helper.getI(mid);
            if (target < midVal) {
                right = mid - 1;
            } else if (target > midVal) {
                left = mid + 1;
            } else {
                return mid;
            }
        }
        return -left - 1;
    }
```

# 现成的Function声明汇总
java8给了一些统一的描述术语(DSL)来分类这些函数接口:

| 接口               | lambda         | 含义                               |
|--------------------|----------------|------------------------------------|
| Predicate< T>      | T->boolean     | 类似于filter，接收T类型返回boolean |
| Consumer< T>       | T->void        | 类似于foreach，接收T类型返回void   |
| Function< T,R>     | T->R           | 类似于Map，接收T类型返回R类型      |
| Supplier< T>       | ()->T          |                                    |
| UnaryOperator< T>  | T->T           |                                    |
| BinaryOperator< T> | (T,T)->T       |                                    |
| BiPredicate< L,R>  | (L,R)->boolean |                                    |
| BiConsumer< T,U>   | (T,U)->void    |                                    |
| BiFunction< T,U,R> | (T,U)->R       |                                    |

常见的几种函数指针都可以用这些现成的声明，如果要抛异常，或者比较特殊的入参、出参，就自己用`@FunctionalInterface`注解声明个接口和相应方法即可。

此外，上述接口的T,R泛型都需要类、引用，因此如果要节省装箱开销，可以用相应的原始类型声明，比如一个`int->int`的变换，相应的接口就是`IntUnaryOperator`。
因此上一节中的代码可以简化成:
```java
public boolean searchMatrix(int[][] matrix, int target) {
        if (matrix == null || matrix.length == 0 || matrix[0].length == 0 || target < matrix[0][0]) {
            return false;
        }

        int lowerBound = getLowerBound(target, 0, matrix.length - 1, i -> matrix[i][0]);// 闭包捕获matrix
        if (lowerBound >= 0) {
            return true;
        }
        final int finalLowerBound = -lowerBound - 2;
        int x = getLowerBound(target, 0, matrix[0].length - 1, i -> matrix[finalLowerBound][i]);// 闭包捕获matrix和finalLowerBound
        return x >= 0;
    }

    private int getLowerBound(int target, int left, int right, IntUnaryOperator helper) {
        int mid;
        while (left <= right) {
            mid = (left + right) >>> 1;
            int midVal = helper.applyAsInt(mid);
            if (target < midVal) {
                right = mid - 1;
            } else if (target > midVal) {
                left = mid + 1;
            } else {
                return mid;
            }
        }
        return -left - 1;
    }
```


其他的原始类型(免装箱)的现成有的声明如下:（要用的时候可以查表）
```java
// 1. Predicate<T> T->boolean :
IntPredicate,LongPredicate, DoublePredicate
// 2. Consumer<T> T->void :
IntConsumer,LongConsumer, DoubleConsumer
// 3. Function<T,R> T->R :
IntFunction<R>,
IntToDoubleFunction,
IntToLongFunction,
LongFunction<R>,
LongToDoubleFunction,
LongToIntFunction,
DoubleFunction<R>,
ToIntFunction<T>,
ToDoubleFunction<T>,
ToLongFunction<T> 
// 4. Supplier<T> ()->T :
BooleanSupplier,IntSupplier,LongSupplier,DoubleSupplier
// 5. UnaryOperator<T> T->T :
IntUnaryOperator,
LongUnaryOperator,
DoubleUnaryOperator
// 6. BinaryOperator<T> (T,T)->T :
IntBinaryOperator,
LongBinaryOperator,
DoubleBinaryOperator
// 7. BiConsumer<T,U> (T,U)->void :
ObjIntConsumer<T>,
ObjLongConsumer<T>,
ObjDoubleConsumer<T>
// 8. BiFunction<T,U,R> (T,U)->R :
ToIntBiFunction<T,U>,
ToLongBiFunction<T,U>,
ToDoubleBiFunction<T,U> 
```

# 无状态
默认无状态的操作:  map,filter,collect; // 假如不使用外部变量(闭包)
有状态的操作: reduce,sum,max,sorted,distinct,skip,limit; // 有内部状态、初始值

// flatmap? 

#  原始类型流
原始类型流如: `IntStream`
```java
IntStream intStream = menu.stream().mapToInt(Dish::getCalories);
Stream< Integer> stream = intStream.boxed();
```

原始类型的Optional:`OptionalInt`
```java
OptionalInt maxCalories = menu.stream().mapToInt(Dish::getCalories).max();
```

# java中的蜂巢结构
d3中有个蜂巢结构,java8中也可以有:
```java
Map<Dish.Type, Map<CaloricLevel, List<Dish>>> dishesByTypeCaloricLevel = menu.stream().collect(
groupingBy(Dish::getType,
   groupingBy(dish -> {
        if (dish.getCalories() <= 400) return CaloricLevel.DIET;
        else if (dish.getCalories() <= 700) return CaloricLevel.NORMAL;
        else return CaloricLevel.FAT;
    });
));
```

还可以分组计数:
```java
Map<Dish.Type, Long> typesCount = menu.stream().collect(
                    groupingBy(Dish::getType, counting()));
```


# collectors中的characteristics
返回枚举值的list:
```
UNORDERED: 结果不受顺序影响;
CONCURRENT: 可以并行运行;
IDENTITY_FINISH: 累加器和最后的结果类型是一样的,不需要再转换。
```
一般是是返回后两者,因为一般都会希望维持顺序(如`toList`):
```java
@Override
public Set<Characteristics> characteristics() {
    return Collections.unmodifiableSet(EnumSet.of(
    IDENTITY_FINISH, CONCURRENT));
}
```
如果是生成质数这种,实现上依赖之前已经生成的质数,有状态,则不能并行。只返回一个`IDENTITY_FINISH`.



# Spliterator
Spliterator的特性:
```
ORDERED 元素有既定的顺序（例如List），因此Spliterator在遍历和划分时也会遵循这一顺序
DISTINCT 对于任意一对遍历过的元素x和y，x.equals(y)返回false
SORTED 遍历的元素按照一个预定义的顺序排序
SIZED 该Spliterator由一个已知大小的源建立（例如Set），因此estimatedSize()返回的是准确值
NONNULL 保证遍历的元素不会为null
IMMUTABLE Spliterator的数据源不能修改。这意味着在遍历时不能添加、删除或修改任何元素
CONCURRENT 该Spliterator的数据源可以被其他线程同时修改而无需同步
SUBSIZED 该Spliterator和所有从它拆分出来的Spliterator都是SIZED
```
DISTINCT和SORTED比较难达到;
一般是:
```java
  @Override
  public int characteristics() {
      return ORDERED + SIZED + SUBSIZED + NONNULL + IMMUTABLE;
  }
```

# 设计模式
策略模式: 策略发生变化，只修改用到的函数。(例如输入校验策略)
模版方法: 修改一大段代码中其中一个函数的调用实现。(传入函数指针)
观察者模式: 例如GUI场景下，注册一大堆监听事件。(传入函数指针)
## 责任链模式: 
每个处理者可以增加后继处理节点,形成一个链表，每次处理的时候，调用下一个处理者。用lamda中的addThen的话:
```java
UnaryOperator<String> headerProcessing =
 (String text) -> "From Raoul, Mario and Alan: " + text;
UnaryOperator<String> spellCheckerProcessing =
 (String text) -> text.replaceAll("labda", "lambda");
Function<String, String> pipeline =
 headerProcessing.andThen(spellCheckerProcessing);
String result = pipeline.apply("Aren't labdas really sexy?!!")
```
## 工厂模式
可以把构造函数放map里:
```java
final static Map<String, Supplier<Product>> map = new HashMap<>();
static {
 map.put("loan", Loan::new);
 map.put("stock", Stock::new);
 map.put("bond", Bond::new);
} 
```

# lamda方法的单元测试
1. 存成成员变量(函数指针);
2. 匿名方法粒度太小了，应该测试上层方法;

# lambda的副作用
1. 出错信息模糊: 错误栈缺失准确的信息；

## 调试技巧
使用peek函数。
```java
List<Integer> result =
 numbers.stream()
 .peek(x -> System.out.println("from stream: " + x))
 .map(x -> x + 17)
 .peek(x -> System.out.println("after map: " + x))
 .filter(x -> x % 2 == 0)
 .peek(x -> System.out.println("after filter: " + x))
 .limit(3) // 只取3个. 
 .peek(x -> System.out.println("after limit: " + x))
 .collect(toList());
```

# 3种兼容性
二进制级兼容： 现有二进制执行文件能无缝链接；
源码级兼容：   现有程序能够成功编译通过；
函数行为级兼容：程序接受同样的输入能得到同样的结果。

接口中加入新的方法：
二进制级兼容：兼容;(不会被调用)
源码级兼容：不兼容;(不符合语法)
函数行为级兼容: 兼容;(不影响原有函数)

抽象类和有默认方法的接口区别：抽象类可以有成员、不能多继承。


默认方法的接口多继承优先级问题：
1. 类中的实现> 接口中的;
2. 子接口中实现>父接口中的;
3. 上述两种无法判断时,必须显式调用,声明调用的是哪个实现:
```java
// class C implement B,A
B.super.hello(); // 显式调用B接口中的方法;
```

# Option
每次新建一个类的时候，所有引用类成员都尽量搞成`Option`.
如果可能为null,就搞成`Option`;
如果不可能为null,就维持朴素类型。// 比如构造时就自动填充确定值的final

相当于领域中将是否可选这个属性也封装了进去，而不是靠注释来弱约束。
```java
// 初始化可以这样:
Optional<Car> optCar = Optional.empty();
// 根据空或非空值创建: 
Optional<Car> optCar = Optional.ofNullable(car); // 推荐初始化方式
```

用了Optional以后,后续用Stream api处理(疯狂调用`flatMap`),整个过程就变成函数式,再也不需要null关键字。


对于可选成员(`Option<T>`): 调用`flatMap`;
对于必选成员(`final String`): 调用`map`;

Option的缺点: 无法序列化。
如果需要可以序列化的类,只能隐藏Option变量:
```java
public Optional<Car> getCarAsOptional() {
             return Optional.ofNullable(car);
}
```

Optional不能滥用(不能序列化):(java Lambda（JSR-335）专家组考虑并拒绝了它)
> 尽量避免将Optional用于类属性、方法参数及集合元素中，因为以上三种情况，完全可以使用null值来代替Optional，没有必要必须使用Optional，另外Optional本身为引用类型，大量使用Optional会出现类似(这样描述不完全准确)封箱、拆箱的操作，在一定程度上会影响JVM的堆内存及内存回收。
相关tip:
https://segmentfault.com/a/1190000018936877?utm_source=tag-newest

- 不要放进Map里，因为Optional不是作为值类型设计的，它的`equals、hashcode、==`方法都不能保证正确性(和唯一性有关的方法)。

最后: POJO还是保持纯粹，这样也有向前先后兼容性。


# 异步编程

无法从线程失败中恢复的版本:
```java
public Future<Double> getPriceAsync(String product) {     
        CompletableFuture<Double> futurePrice = new CompletableFuture<>();          
        new Thread( () -> {
           double price = calculatePrice(product);
           futurePrice.complete(price);// here
        }).start();
       return futurePrice;
}
```

加上try catch:
```java
try {
    double price = calculatePrice(product);
    futurePrice.complete(price);
} catch (Exception ex) {
    futurePrice.completeExceptionally(ex);
}   
```

精简后:
```java
public Future<Double> getPriceAsync(String product) {
    return CompletableFuture.supplyAsync(() -> calculatePrice(product));
}
```

## 并行查询(无状态)
无状态:
```java
shops.parallelStream().map(shop->String.format("%s price is %.2f"
,shop.getName(), shop.getPrice(product)))).collect(ToList());
```

定制Executor:
```java
private final Executor executor =
Executors.newFixedThreadPool(Math.min(shops.size(), 100),
    new ThreadFactory() {
     public Thread newThread(Runnable r) {
       Thread t = new Thread(r);
        t.setDaemon(true);// 守护线程,主线程结束后关闭
        return t;
    }
});

```


计算密集,无IO: 使用Stream接口; // 不加线程数
IO密集,计算简单: 使用`CompletableFuture`,自定义线程池; // 可以疯狂加线程

# 时间api

## LocalDate和LocalTime
```java
LocalDate date = LocalDate.of(2014, 3, 18);
LocalDate today = LocalDate.now();
LocalTime time = LocalTime.of(13, 45, 20);
// 字符串:
LocalDate date = LocalDate.parse("2014-03-18");
LocalTime time = LocalTime.parse("13:45:20");
```
## 复合后: LocalDateTime
```java
LocalDateTime dt1 = LocalDateTime.of(2014, Month.MARCH, 18, 13, 45, 20); LocalDateTime dt2 = LocalDateTime.of(date, time);// 复合localDate和localTime
```

## unix时间: `java.time.Instant`
```java
Instant.ofEpochSecond(3);// 3
```

## 时间间隔: Duration
```java
Duration d1 = Duration.between(time1, time2);
Duration d1 = Duration.between(dateTime1, dateTime2);
Duration d2 = Duration.between(instant1, instant2);
Duration threeMinutes = Duration.ofMinutes(3);
```

## 天间隔: Period
```java
Period tenDays = Period.between(LocalDate.of(2014, 3, 8),
                        LocalDate.of(2014, 3, 18));
Period tenDays = Period.ofDays(10);
```

上述类都是不可变的。// 线程安全

修改直接返回新变量:
```java
LocalDate date2 = date1.withYear(2011);
LocalDate date2 = date1.plusWeeks(1);
```

## 个性化操作: TemporalAdjuster
下个周日、下个工作日、本月最后一天:
```java
import static java.time.temporal.TemporalAdjusters.*;
LocalDate date1 = LocalDate.of(2014, 3, 18);
LocalDate date2 = date1.with(nextOrSame(DayOfWeek.SUNDAY));
LocalDate date3 = date2.with(lastDayOfMonth());
```

## 格式化、格式转换
### localData<=>String
```java
LocalDate date = LocalDate.of(2014, 3, 18);
String s1 = date.format(DateTimeFormatter.BASIC_ISO_DATE); // 20140318
String s2 = date.format(DateTimeFormatter.ISO_LOCAL_DATE); // 2014-03-18

LocalDate date1 = LocalDate.parse("20140318",
                                 DateTimeFormatter.BASIC_ISO_DATE);
LocalDate date2 = LocalDate.parse("2014-03-18",
                                 DateTimeFormatter.ISO_LOCAL_DATE);
```

定制格式:
```java
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
LocalDate date1 = LocalDate.of(2014, 3, 18);
String formattedDate = date1.format(formatter);
LocalDate date2 = LocalDate.parse(formattedDate, formatter);
```

## 时区
`java.time.ZoneId`和`ZonedDateTime`:
从LocalData转换到ZonedDateTime:(其他几个也一样)
```java
ZoneId romeZone = ZoneId.of("Europe/Rome");
// 或者: ZoneId zoneId = TimeZone.getDefault().toZoneId();
LocalDate date = LocalDate.of(2014, Month.MARCH, 18); 6
ZonedDateTime zdt1 = date.atStartOfDay(romeZone);

```

{% img /images/2019-11/zonedDateTime.png 800 1200 zonedDateTime %}

时区差: `ZoneOffset`

其他日历系统: `ThaiBuddhistDate`,`MinguoDate`,`JapaneseDate`,`HijrahDate`



# 其他tips
ConcurrentHashMap类：
用`mappingCount`方法代替`size`方法;
`mappingCount`: 返回long;
`size`: 返回int; (实际大小超过int时无效)

重复遍历流: 可以使用StreamForker,缺点本质上是存了内存。

# lambda底层实现
## 匿名类的实现
匿名内部类底层是生成一个`class$1`的类。(外层类名+数字)
缺点： 每个类都需要加载、验证的操作，如果类太多，开销太大；

## 查看字节码和常量池
使用命令:
```shell
javap -c -v ClassName 
```


## lambda的实现： InvokeDynamic
本质上是在所在类里新增了一个方法。将Lambda表达式的代码体填入到运行时动态创建的`静态方法`，就完成了Lambda表达式的字节码转换。
动态链接方法，然后用`InvokeDynamic`调用:
```java
public class Lambda {
    Function<Object, String> f = [dynamic invocation of lambda$1]
     static String lambda$1(Object obj) {
        return obj.toString();
     }
} 
```
优点:
1. lamda相关字节码的生成推迟到使用的时候; (懒加载)
2. 如果不是闭包：有类似于static final变量的缓存效果;(闭包的话,因为可能捕获不同的局部变量，不可以缓存)
3. 没有额外的开销;
如果lambda是个闭包，也就是捕获了其他作用域内的变量(当然编译时会要求是final或者实质上是final)，也很简单，就是给这个函数加一个参数即可。





