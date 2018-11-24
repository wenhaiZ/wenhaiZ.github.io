---
layout: post
title: "Java 8 中的 Optional"
date: 2018-11-20 16:51:00 +0800
tags: [Code,Java,Kotlin]
subtitle: "以及与 Kotlin 的可空类型对比"
---
>本文主要介绍了 Java 8 中的可空类型 Optional 的使用及源码分析，并和 Kotlin 中的可空类型进行了对比。

## NullPointerException
相信每个使用过 Java 的人都曾遇到过 `NullPointerException` 这个异常，这是一个很常见的异常，大家都知道怎么回事。  

处理它的时侯，我们可以从代码逻辑上保证某个变量一定不会为 `null`，但更多时候，我们需要通过 `if` 语句来判断即将操作的变量是否为空，如果不为空，再继续操作它，就像下面这样：  

```java
if(something!=null){
    something.doSomething();
    //.....
}
```  

如果逻辑简单，只涉及一两个变量，那么这段代码看起来并不是那么糟糕，但如果是涉及到多个变量的嵌套逻辑，对于出现的每个变量，都需要进行一次非空检查，于是代码就变成了这样：
```java
String city = "UNKNOWN";
if(programmer != null){
    GirlFriend girlFriend = programmer.getGirlFriend();
    if(girlFriend != null){
        Address address = girlFriend.getAddress();
        if(address != null){
            city = address.getCity();
        }
    }
}
```
代码会变得很臃肿，可读性很差，而且，一旦忘记检查某个变量，那么就有可能出现 `NullPointerException`。  

在我看来，Java 的变量声明有一点自相矛盾的地方。 

比如声明一个类型为 Student 的变量 s, `Student s` 这句代码的意思是 **s 是一个 Student 类型的变量，它可能为空**，也就是说，Student 这个类型本身就有着可能为空的含义，但是，`null instanceof Student` 这个表达式的值却永远为`false`，这就和声明变量时的含义产生了歧义。

所以我认为，虽然 Java 类型本身包含了可空的含义，却没有从语法上对可空类型进行支持，它需要程序员时刻记着变量可能为空，稍不留神，就可能得到`NullPointerException`。  

时刻提心吊胆的想着变量可能为空，这太痛苦了，在这样的前提下，写出能安全运行的代码是一件多么累的事情啊。

好消息是，在 Java 8 中，可以通过 `java.util.Optional` 这个类更加清楚的说明某个变量可能为空，从而更好的处理“值缺失”的情况，让代码更简洁也更安全。

## Optional 使用
Optional 的中文意思是可选的，也就是说，这个值可能有也可能没有。  
简单来说，它就是个容器，里面包裹了一个可能为`null`的值。  

Optional 支持泛型，比如想表示一个程序员，他可能有女朋友，也可能没有，就可以使用 `Optional` 来表示：
```java
class Programmer{
    private Optional<GirlFriend> girlFriend;
}
```
如果在 IDEA 里写下上面的代码，会得到一个 warning:
>*Optional was designed to provide a limited mechanism for library method return types where there needed to be a clear way to represent "no result". Using a field with type java.util.Optional is also problematic if the class needs to be Serializable, which java.util.Optional is not.*  

也就是说， **Optional 最好用于标记库方法的返回值，而不是类字段**，因为可能会面临序列化的问题。  

所以，我们可以把上面的代码改成下面这样，就比较符合 Optional 的设计理念：
```java
public class Programmer {

    private GirlFriend girlFriend;

    public Optional<GirlFriend> getGirlFriend() {
        return Optional.of(girlFriend);
}
```

上面代码中的 `Optional.of()` 方法就是创建 `Optional` 实例的一个方法，`Optional` 类提供了三个静态方法用于创建 `Optional` 实例，如下表所示：  

|方法名|作用|
|:----:|:----:|:----:|
|Optional.empty()|创建一个空的 `Optional` 对象，其中的值为 `null`|
|Optional.of(T value)|创建一个值为 `value` 的 `Optional` 对象。如果 `value` 为 `null`，则会抛出 `NullPointerException`|
|Optional.ofNullable(T value)|创建一个值为 `value` 的 `Optional` 对象。如果 `value` 为 `null`,会调用 `empty()` 创建一个空的 `Optional` 对象|
  

`Optional` 提供了一个 `isPresent`方法来表示其是否含有值，如果有值，该方法会返回 `true`，然后就可以通过 `get()` 方法来获取该值：

```java
Programmer p = new Programmer();
Optional<GirlFriend> girlFriend = p.getGirlFriend();
if (girlFriend.isPresent()){
    System.out.println(girlFriend.get().getName());
}
```
但是这样使用和进行非空判断没有什么区别，官方也不建议这么使用 `Optional`。   

`Optional` 类提供了一系列方法，来帮助我们更好的使用它来替换掉原来的 `null` 检查。

### 使用 `orElse` 方法提供默认值  

如果需要在值为 `null` 时返回一个默认值，通常可以这么做：
```java
GirlFriend girlFriend = p.getGirlFriend()!=null ? p.getGirlFriend() : new GirlFriend("Lisa");
```
借助 Optioal 的 `orElse` 方法，上面代码可以更简单：
```java
GirlFriend girlFriend = p.getGirlFriend().orElse(new GirlFriend("Lisa"));
```  

`Optional` 还提供了一个 `orElseThrow` 方法，用于在值缺失的情况下接抛出一个异常：
```java
GirlFriend girlFriend = p.getGirlFriend().orElseThrow(IllegalArgumentException::new);
```
>在上面的代码中，`IllegalArgumentException::new` 是 Java 8 引入的[方法引用](http://www.runoob.com/java/java8-method-references.html)，代表执行 `IllegalArgumentException` 的构造方法来创建一个异常实例。   

### 使用 `filter` 方法进行值的筛选
如果需要在获取值的同时，对该对象的某些属性值进行判断，符合条件时才使用。在 Java 8 之前，我们可能需要这么写：
```java
GirlFriend girl = ...;
if(girl != null && "Alice".equals(girl.getName())){
    System.out.println("ok");
}
```
`Optional` 提供了一个 `filter` 方法，借助 Java 8 新引入的 [Lambda 表达式](https://www.oracle.com/technetwork/articles/java/architect-lambdas-part1-2080972.html)，可以很简单的完成这样的操作：
```java
Optional<GirlFriend> maybeGirlFriend = ...;
maybeGirlFriend.filter(girl -> "Alice".equals(girl.getName())
            .ifPresent(() -> System.out.println("ok"));
```
`filter` 方法中，传入了一个 `Lambda` 表达式来进行条件判断。`filter` 方法同样会返回一个 `Optional` 对象，之后调用 `ifPresent` 方法，当存在符合条件的值时，进行输出。

### 使用 `map` 方法进行值提取或者值变换
如果需要从对象中提取一些值，比如从 `Programmer` 对象中获取 `GrilFriend` 对象，并检查年龄，代码如下：
```java
if(programmer != null){
    GirlFriend girlFriend = programmer.getGirlFriend();
    if(girlFriend != null && girlFriend.getAge() > 20){
        System.out.println("ok");
    }
}
```
借助 Optional 提供的 map 方法，可以将获取 `GirlFriend` 对象的代码简化成如下形式：
```java
Optional<Programmer> maybeProgrammer = ....;
Optional<GirlFriend> girlFriend = maybeProgrammer.map(Programmer::getGirlFriend);
```
Java 8 为 [`Stream`](https://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/index.html) 类也提供了一个 `map` 方法，它的作用是对于流中的每个元素应用 `map` 方法中传入的操作。    

`Optional` 的 `map` 方法与之类似，如果 `Optional` 包含的对象不是空的，那么对其应用 `map` 方法传入的操作，否则就什么也不做。

结合 `map` 和 `filter` 两个方法，可以将上面未使用 `Optional` 的代码更改如下：
```java
//需要注意，在这个例子中，Programmer#getGirlFriend 方法返回值为 GirlFriend，而不是 Optional<GirlFriend>  
maybeProgrammer.map(Programmer::getGirlFriend) //使用 map 获取 GirlFriend 对象，返回值为 Optional<GirlFriend>
      .filter(girl -> girlFriend.getAge() > 20)  //使用 filter 过滤
      .ifPresent(() -> System.out.println("ok"));  //如果存在则进行输出
```
### 使用 `flatMap` 方法处理 `Optional` 嵌套
对于下面的代码：
```java
String city = programmer.getGirlFriend().getAddress().getCity();
```
如果使用 `map` 方法来写，参照上面的代码，可以写成这样：
```java
/**
* 在这个例子中，Programmer#getGirlFriend 返回值类型为 Optional<GirlFriend>
* GirlFriend#getAddress 方法返回值类型为 Optional<Address>
* 而 Address#getCity 方法返回值为 String
*/
String city = programmer.map(Programmer::getGirlFriend)
                  .map(GirlFriend::getAddress)
                  .map(Address::getCity)
                  .orElse("UNKNOWN");
```
但是，这段代码会编译失败。   

原因就是 `programmer` 的类型是 `Optional<Programmer>`，可以通过 `map` 方法执行 `getGirlFriend` 方法，而 `getGirlFriend` 方法返回值类型是 `Optional<GirlFriend>`，这样一来，`programmer.map(Programmer::getGirlFriend)`  返回的类型就是 `Optional<Optional<GirlFriend>>` ，因此就不能通过 `map` 方法来调用 `GirlFriend` 的 `getAddress` 方法。（原因会在后面源码解析的部分讲）

那如何解决 `Optional` 的嵌套问题呢？    

答案是使用 `flatMap` 方法。  

Java 8 中同样为 `Stream` 提供了 `flatMap` 方法，它可以把嵌套的流整合成一个流，比如像下面的代码这样：
```java
//创建嵌套的流对象
Stream<Stream<Integer>> nestStream = Stream.of(Stream.of(1, 2, 3), Stream.of(4, 5, 6), Stream.of(7, 8, 9));

//应用 flatMap 方法
Stream<Integer> flatStream = nestStream.flatMap(integerStream -> integerStream); 

//嵌套流变成了普通流，输出为 1,2,3,4,5,6,7,8,9
flatStream.forEach(System.out::println); 
```
使用 `Optional` 的 `flatMap` 方法就可以处理 `Optional` 嵌套的情况，上面有问题的代码可以改成下面这样：
```java
String city = programmer.flatMap(Programmer::getGirlFriend)
                    .flatMap(GirlFriend::getAddress)
                    .map(Address::getCity)
                    .orElse("UNKNOWN");
```
在上面的代码中，`programmer.flatMap` 方法，返回的类型为 `Optional<GirlFriend>`，再次调用 `flatMap` ，返回的类型就是 `Optional<Address>`， 最后可以使用 `map` 方法的原因是 `getCity` 方法返回类型是 `String`，而不是 `Optional<String>`。


这是使用 `Optional` 的时候比较容易疏忽的问题，刚开始使用 `map` 和 `flapMap` 可能会比较费解，看完下面的源码可能会好理解一些。

## Optional 源码简析  

`Optional` 只是一个包装类，源码也不过 300 多行，没有什么复杂的逻辑，但是会涉及 Java 8 中几个新增的函数式接口(`FunctionalInterface`)，下面就对 `Optional` 的几个主要方法进行简单的源码分析。  

先来看两个静态方法。  

### `Optional.of`
```java
public static <T> Optional<T> of(T value) {
    return new Optional<>(value);
}
```
`Optional.of` 方法调用的 Optional 的构造方法，这个构造方法是 `private` 的，逻辑也很简单：
```java
private Optional(T value) {
    this.value = Objects.requireNonNull(value);
}
```
`Objects.requireNonNull` 方法会检查传入参数是否为 `null`，如果为 `null` ，则会抛出 `NullPointerException` 异常：
```java
//Ojbects.java
public static <T> T requireNonNull(T obj) {
    if (obj == null)
        throw new NullPointerException();
    return obj;
}
```
### `Optional.ofNullable`
```java
public static <T> Optional<T> ofNullable(T value) {
    return value == null ? empty() : of(value);
}
```
`Optional.ofNullable` 方法在参数为 `null` 时会调用 `Optional.empty` 方法返回一个空的 `Optional` 对象，不会抛出异常。而 `Optional.empty` 方法的逻辑十分简单，就是将一个静态的常量转换成所需要的类型：  

```java
private static final Optional<?> EMPTY = new Optional<>();

public static<T> Optional<T> empty() {
    Optional<T> t = (Optional<T>) EMPTY;
    return t;
}
```

`Optional` 的实例方法涉及 Java 8 中的几个函数式接口，先来看 `ifPresent` 方法。

### `Optional#ifPresent`
```java
public void ifPresent(Consumer<? super T> consumer) {
    if (value != null)
        consumer.accept(value);
}
```
`ifPresent` 方法接受一个 `Consumer` 类型的参数，`Consumer` 是一个接口，包含一个无返回值的 `accept` 方法，用于对传入的值进行操作:
```java
@FunctionalInterface
public interface Consumer<T> {

    /**
     * Performs this operation on the given argument.
     *
     * @param t the input argument
     */
    void accept(T t);
}
```
值得注意的是，`Consumer` 接口使用 `@FunctionalInterface` 进行标注，在 Java 8 中，此类接口都可以使用 `Lambda` 来替换匿名内部类来进行实例化。

### `Optional#orElseGet`
```java
public T orElseGet(Supplier<? extends T> other) {
    return value != null ? value : other.get();
}
```
`orElseGet` 方法接收一个类型为 `Supplier` 的参数，`Supplier` 同样是一个函数式接口，它的定义如下：
```java
@FunctionalInterface
public interface Supplier<T> {

    /**
     * Gets a result.
     *
     * @return a result
     */
    T get();
}
```
可以看到它与 `Consumer` 恰好相反，`Supplier` 提供了 `get` 方法用于生成一个特定类型的对象。

### `Optional#filter`  

`filter` 方法用于对数据进行筛选，它的源码如下：
```java
public Optional<T> filter(Predicate<? super T> predicate) {
    Objects.requireNonNull(predicate);
    if (!isPresent())
        return this;
    else
        return predicate.test(value) ? this : empty();
}
```

它通过一个名为 `Predicate` 的接口来对值进行筛选，如果 `Predicate` 的 test 方法返回 `true` ，就返回当前 `Optional` 对象，否则就返回一个空的`Optional` 对象。   

`Predicate` 接口的定义如下：
```java
@FunctionalInterface
public interface Predicate<T> {

    /**
     * Evaluates this predicate on the given argument.
     *
     * @param t the input argument
     * @return {@code true} if the input argument matches the predicate,
     * otherwise {@code false}
     */
    boolean test(T t);
}

// 该接口还要几个默认方法，这里不作展示
```
它通过 `test` 方法来判断值是否符合条件。

接下来是两个比较有意思的方法：`map` 和 `flatMap`，先来看 `map`方法。

### `Optional#map`
```java
public <U> Optional<U> map(Function<? super T, ? extends U> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent())
        return empty();
    else {
        // 可能出现 Optional 嵌套
        return Optional.ofNullable(mapper.apply(value));
    }
}
```
`map` 方法接收一个 `Function` 接口实例，`Function` 接口定义如下:
```java
@FunctionalInterface
public interface Function<T, R> {

    /**
     * Applies this function to the given argument.
     *
     * @param t the function argument
     * @return the function result
     */
    R apply(T t);
}
```
可以看出 Function 就是一个“转换”接口，它提供了一个 `apply` 方法，通过类型为 `T` 的对象来生成一个类型为 `R` 的对象。   

需要注意的是，`map` 方法的返回值类型为 `Optional(U)`，并且传入接口泛型定义为 `Function<? super T, ? extends U>`，在最后的 `return` 语句中，使用了 `Optional.ofNullable` 方法创建一个 `Optional` 对象。  

这个时候，如果 `U` 本身就是一个 `Optional<?>` 类型的对象，那么返回值就会为 `Optional<Optional<?>>`，这就是前面连续调用 `map` 方法所出现的问题，也就是出现 `Optional` 嵌套的原因。

那 `flatMap` 为什么就不会出现 `Optional` 嵌套呢？看了代码就知道：

### Optional#flatMap()
```java
public <U> Optional<U> flatMap(Function<? super T, Optional<U>> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent())
        return empty();
    else {
        //只返回 apply 的返回值
        return Objects.requireNonNull(mapper.apply(value));
    }
}
```
`flatMap` 同样接收一个 `Function` 对象，**不同的是**它对其泛型定义为 `Function<? super T, Optional<U>>`，并且，在最后的 `return` 语句中，直接返回了 `Function` 的 `apply` 方法的返回值（加入了 `null` 判断），也就是说，通过 `Function` 转换后的类型与返回值类型是一样的，不会存在 `Optional` 嵌套问题，**除非 `apply` 方法返回值本身就是 Optional 的嵌套**。

以上就是 Optional 类的主要方法的源码。  

通过源码，我们可以看到，Java 8 通过引入 `Optional` 类来对普通对象进行包装，并借助函数式接口和 `Lambda` 来简化操作，以此实现对可空类型的支持。但这样使用起来不免让人觉得别扭，特别是 Optional 的嵌套对于初次使用的人来说有点不太好理解，另外，在对象外包装了一层，可能还会面临性能问题。 

下面来看看 `Java` 的“小老弟” `Kotlin` 是怎么做的。

## [Kotlin](https://kotlinlang.org/) 中的可空类型
`Kotlin` 的设计哲学之一就是“安全”，它以更小的成本来提供比 Java 更高的安全级别。  

Kotlin 的安全主要体现为类型安全，例如，为了避免很多意外的`NullPointerException`，Kotlin 引入了可空类型，但与 Java 8 的 Optional 采用的“容器式”处理不同，可空类型在 Kotlin 中得到了语法层面的支持。

在 Kotlin 中，如果某个变量是可空的，可以通过下面的方式声明：
```java
//声明一个可空类型的字符串
var nullableString: String? = "A string can be null."

nullableString = null  //可以被赋值为 null
```
>值得注意的是，在 Kotlin 中，`null is String?` 这句表达式的值为 `true`，这是 Kotlin 比 Java 更合理的地方，也是 Java 值得怀疑的原因。 

对于不会为空的变量，声明时去掉 `?` 即可：
```java
//声明一个不可以为空的字符串变量
var nonNullString: String = "A string can't be null."
//不能赋值为 null,会报错
nonNullString = null 
```
调用方法时，对于非空变量，可以直接调用：
```
nonNullString.toUpperCase()
```
而对于可空类型的变量，调用方法时就不能直接通过 `.` 来调用了，而是通过 `?.`:
```java
var nullableString: String? = "A string can be null."
val upper = nullableString.substring(1) //编译不通过
val upper = nullableString?.substring(1) // 编译可以通过
```
在上面的代码中，如果 `nullableString` 为空，则 `substring` 方法不会被调用，而是直接返回 `null`，因此就不会抛出 `NullPointerException`。

如果想在变量为空时提供一个默认值，还可以借助`?:`操作符：
```java
val upper = nullableString?.substring(1)?:"NULL"
```
这样，当 `nullableString` 为 null 时，upper 的值为 `"NULL"`。 


当然，如果可以确定某个可空类型的变量在某些情况下一定是非空的，但编译器又不能进行智能转换时，可以使用`!!.`进行非空断言：  

```java
nullableString!!.toUpperCase() 
```
对于上面的情况，如果 `nullableString` 为空，就会抛出 `NullPointerException`。  

那 Kotlin 是如何实现对可空类型的支持的呢？  

我们可以通过将字节码反编译一下来看到背后的秘密。  

比如，对于下面的代码：
```java
var programmer: Programmer? = Programmer()
val name = programmer?.girlFriend?.name
```
将字节码反编译之后的 Java 代码为：
```java
Programmer programmer = new Programmer();
GirlFriend var10000 = programmer.getGirlFriend();
//加入了 null 检查
String name = var10000 != null ? var10000.getName() : null;
```
可以看到，编译之后的代码加入了 null 检查，并提供了默认值 `null`。

而对于使用 `!!` 的代码：
```java
var programmer: Programmer? = Programmer()
val name = programmer?.girlFriend!!.name
```
将字节码反编译之后的 Java 代码为：
```java
Programmer programmer = new Programmer();
GirlFriend var10000 = programmer.getGirlFriend();
if (var10000 == null) {
    //用于抛出 NullPointerException
    Intrinsics.throwNpe();
}
String name = var10000.getName();
```
> 查看 Kotlin 的字节码方法：IDEA:Tools->Kotlin->Show Kotlin ByteCode.   
在字节码面板点击 `Decompile` 可以将字节码反编译成 Java 代码。   

通过上面的例子可以看到，Kotlin 对可空类型的支持是**在编译时为使用可空类型的代码加入 `null` 检查**，在保证安全的同时提高编码效率，减少冗余代码。

## 总结  
在我看来，可空类型的意义并不在于减少判空（它只是换了一种判空的方式而已），而是明确告诉程序员某个变量可能是空的，必须在编码时进行处理后才能通过编译，这样一来，就可以将 `NullPointerException` 这个运行时错误转换成一个编译期错误，有效减少那些编码过程不易察觉的 `NullPointerException`。  

Java 8 通过引入包装类 `Optional` 并配合函数式接口来支持可空类型，而 Kotlin 则直接在类型系统中增加了可空类型，并在编译时生成 `null` 检查代码。 

虽然 Java 对可空类型的支持显得不太彻底，但我们也应该为 Java 面向现代化的进步感到高兴。[Java 9](https://www.ibm.com/developerworks/cn/java/the-new-features-of-Java-9/index.html )还为 `Optional` 增加了`ifPresentOrElse`、`or` 和 `stream` 等方法，使得 `Optional` 的操作更加灵活了。

而可空类型作为 Kotlin 的语言特性，使用起来就比 Java 就方便多了。   

随着 [Kotlin 1.3 的发布](https://blog.jetbrains.com/kotlin/2018/10/kotlin-1-3/)，Kotlin 也变得越来越成熟。作为一种年轻、高效、安全和简洁的编程语言，Kotlin 还有很多的地方值得我们去学习和探索。

## 参考
- [Tired of Null Pointer Exceptions? Consider Using Java SE 8's Optional!](https://www.oracle.com/technetwork/articles/java/java8-optional-2175753.html)
- [Kotlin in Action](https://www.manning.com/books/kotlin-in-action) Chapter 6