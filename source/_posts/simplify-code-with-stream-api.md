---
title: 使用 Stream API 简化代码
tags: Java
categories: Java
date: 2018-10-07 21:11:07
---

Stream API 是 Java 8 新引入的特性，用来增强集合操作。前段时间，在开发新需求以及重构老代码的时候，我开始尝试使用Java Stream API，使写出的代码更简洁也更易维护。这篇文章便对 Java Stream API 做一个总结，也记录一下我在使用过程中的一些心得和技巧。

## 0x01 从一个简单场景讲起

先考虑这样一种可能在游戏代码里遇到的场景：每个玩家可以创建和培养众多游戏角色，现在从数据库或者其他地方获得了一批游戏角色信息，需要过滤出角色等级大于60的角色，并按照玩家ID进行归并。

这是一个常见的对数据进行过滤并归并的操作，如果在使用 JDK 7 的时候，我们的代码可能长这个样子：

<!-- more -->

```java
public Map<Long, List<Character>> groupCharacterByGamer() {
    List<Character> characters = characterDao.getCharacters();
    Map<Long, List<Character>> results = new HashMap<>();
    for (Character character : characters) {
        if (character.getLevel() > 60) {
            List<Character> gamerCharacters = results.get(character.getGamerId());
            if (gamerCharacters == null) {
                gamerCharacters = new ArrayList<>();
            }
            gamerCharacters.add(character);
            results.put(character.getGamerId(), gamerCharacters);
        }
    }
    return results;
}
```

这里列举下上面方法用到的实体类：

```java
@Data
static class Gamer {
    private Long id;
    private String name;
}

@Data
static class Character {
    private Long id;
    private Long gamerId;
    private String name;
    private Integer level;
}

public interface CharacterDao {
    List<Character> getCharacters();
}
```

下面直接看下在 JDK 8 Stream API 的帮助下，我们可以怎样让代码简单一点点：

```java
public Map<Long, List<Character>> groupCharacterByGamerWithStreamAPI() {
    List<Character> characters = characterDao.getCharacters();
    return characters
            .stream()
            .filter(character -> character.getLevel() > 60)
            .collect(Collectors.groupingBy(Character::getGamerId));
}
```

先说直观感受，对比之前的版本，使用Stream API的代码变得简单和清晰了不少。

## 0x01 Lambda 表达式 & 方法引用

在讨论Stream API之前，先看看和 Stream API 一同被引入JDK 8 的另外两个特性： Lambda 表达式和方法引用。在上面的例子中`c -> c.getLevel() > 60` 就是一个 Lambda 表达式， 而 `Character::getGamerId` 则使用了方法引用，正是在这两个特性的加持下，Stream API 才变得异常强大。

### 初探 Lambda 表达式

Java 通过引入 Lambda 表达式，为 Java 带来了函数式编程的手段，这样我们在参数传递时，不但能够传递对象，还能传递行为(函数)。在这之前，针对这种情况，一般是采用回调或者匿名内部类实现。比如，上面例子中的 `c -> c.getLevel() > 60`和下面冗长的代码实现的效果是一样的：

```java
new Predicate<Character>() {
    @Override
    public boolean test(Character character) {
        return character.getLevel() > 60;
    }
}
```

关于为什么需要 Lambda 表达式的进一步讨论，可以看看这篇文章[Why We Need Lambda Expressions in Java](https://dzone.com/articles/why-we-need-Lambda-expressions)。

从上面的例子也看到了，Lambda 表达式以 `(argument) -> {body}` 的形式呈现，但Lambda 表达式到底是什么呢？是新增的类型么？

我在学习 Lambda 表示式时，总看到别人讲其背后是函数式接口，实际上，这两者之间的关系是： Java 使用函数式接口来表示 lambda 表达式类型，即，每个 Lambda 表达式都能隐式地赋值给函数式接口。例如，我们可以通过 Lambda 表达式创建 Runnable 接口的引用：

```java
Runnable r = () -> System.out.print("Lambda");
executor.submit(r);
```

实际上在使用的时候，通常不这样赋值后使用，而是直接使用下面的方式：

```java
executor.submit(() -> System.out.print("Lambda"));
```

在未指定函数式接口类型时，编译器会根据方法的签名将对应的类型推断出来，上面例子中`submit`方法的签名为`submit(Runnable task)`，因此编译器会将该 Lambda 表达式赋值给 Runnable 接口。

### 谈谈函数式接口

什么是函数式接口？--简而言之，就是只有一个抽象方法的接口，也被称为SAM(Single Abstract Method)接口。Java 8 之后，任何满足单一抽象方法法则的接口，都会被自动视为函数接口，所以 Runnable 和 Callable 接口也是函数式接口。值得注意的是：单一抽象方法并不代表接口只有一个方法，除了唯一的抽象方法外，函数式接口中可能还有接口默认方法或者静态方法。例如在之前的 Stream API 例子中使用的`filter`方法参数是一个`Predicate<T>`接口，其含义为：“接受一个输入参数，并返回一个布尔值结果”。除`test`方法为一个抽象方法外，Predicate 接口还有`and`，`negate`，`or`三个接口默认方法以及`isEqual`这个静态方法。

接口默认方法也是Java 8 新增的特性，这一改变使接口里可以不完全是抽象的内容，也可以添加特定的具体实现。例如经常在集合类上使用的`forEach`方法，它实际上来自于 `Iterable` 接口的一个接口默认方法：

```java
default void forEach(Consumer<? super T> action) {
    Objects.requireNonNull(action);
    for (T t : this) {
        action.accept(t);
    }
}
```

除了上面提到的`Runable`等传统函数式接口，以及`Stream`的`filter`使用的`Predicate<T>`，`forEach` 使用的`Consumer<T>`外，JDK 8 还包含多个新函数接口，比如 Supplier<T>、BiConsumer<T, U> 和 BiFunction<T, U, R>，它们均是在java.util.function 包中定义的。

### 方法引用

一言以蔽之，方法引用是用于简化Lambda表达式的一种手段。对于有些功能函数的实现已经存在的情况下，我们可以直接使用方法引用来构造Lambda表达式。为此，Java 使用了一个新的操作符`::`来表达方法引用。例如：

```java
List.of("a","b","c").forEach(s -> System.out.println(s));
// 等价于
List.of("a","b","c").forEach(System.out::println);
```

方法引用通过类名和方法名来定位一个静态方法或者实例方法，其语法为`ClassName::methodName` 或者 `instanceRefence::methodName`，如果引用的方法是构造器，则方法名为`new`。方法引用比 Lambda 表达式更简洁，所以一般能直接改写成方法引用的方式就写成方法引用，且智能的IDE也会提示你进行简写。

## 0x02 Stream API 和 流式操作

了解了 Lambda 表达式和方法引用，接下来就看看Stream API 是怎么和他们结合起来使用的。

首先需要明确的是，Stream 不是集合类，它本身是不保存数据的，它更像一个更高级的迭代器。不过同传统的迭代器不一样的是，它是内部迭代，不需要显示地把数据一个一个拿出来加工，只需要传入对元素的操作(函数)，Stream 便能够在内部完成迭代。

我们在使用流的时候，可以想象成一个流水线，数据在流水线上流动，会经过一系列的加工转换，最终生成我们想要的数据。流的操作一般可以看作有三个基本步骤：获取一个数据源 -> 中间过程进行各种数据加工 -> 执行终点操作以获取想要的结果。

例如：

```java
public Map<Long, List<Character>> groupCharacterByGamerWithStreamAPI() {
    return characterDao
            .getCharacters()
            .stream() // 获取流数据源
            .filter(c -> c.getLevel() > 60) // 中间操作
            .collect(Collectors.groupingBy(Character::getGamerId)); // 终点操作
}
```

流的操作类型分为两种： 中间操作(Intermediate) 和 终点操作(Terminal)。一个流可以有N个中间操作，但是只能有一个终点操作。接下来就来看看怎么生成流，以及常见的流的中间操作和终点操作都有哪些。

### 创建 Stream

Java 提供了多种方式来创建流，比较常用的有如下几种：

+ 由集合或者数组直接生成: `Collection.stream()`, `Arrays.stream(Object[])` 等
    + `List.of("a","b","c").stream()`
    + `Arrays.stream(new String[] {"a", "b", "c"})`
+ 由流的静态方法生成： `Stream.of(T... values)`, `IntStream.range(int, int)` 等
    + `Stream.of("a", "b", "c")`
    + `IntStream.range(1,10)`
+ 从文件中获得流：`BufferedReader.lines()`, `Files.lines(Path path)` 等
    + `Files.newBufferedReader(Paths.get("/path/to/file"),StandardCharsets.UTF_8).lines()`
    + `Files.lines(Paths.get("/path/to/file"))`
+ 通过迭代或者生成器自己创建流： `Stream.iterate(Object, UnaryOperator)`，`Stream.generate(Supplier<? extends T> s)` 等
    + `Stream.iterate(1, n -> n + 2)`
    + `Stream.generate(Math::random)`

### 中间操作

常用的流的中间操作主要有：map (mapToInt, flatMap 等)、 filter、 distinct、 sorted、 peek、 limit、 skip、unordered 等。

下面简单介绍下几个最常见的中间操作，剩下的可以查官方API

#### limit/skip

`limit` 返回 Stream 的前面 n 个元素；`skip` 则是扔掉前 n 个元素。

```java
// 生成5个随机数
Stream.generate(Math::random).limit(5).forEach(System.out::println);


// 跳过第一行的表头
Files.lines(Paths.get("test.csv")).skip(1).forEach(System.out::println);
```

#### filter

`filter` 对数据进行过滤，返回的流中只包含满足断言(predicate)的数据

```java
// 统计非空字符串数量
long count = Stream.of("a", "b", "", "c").filter(s -> !s.isEmpty()).count();
```

P.S. `filter` 里面的 `s -> !s.isEmpty()` 是一个 Lambda 表达式，如果想换成方法引用的方式该怎么办呢？

可以以一个静态方法为“管道”来进行转换，该静态方法以一个方法引用为输入，再以特定的函数接口为其返回，如

```java
public static <T> Predicate<T> as(Predicate<T> predicate) {
    return predicate;
}
```

那么，上面的代码就可以改成这样了：

```java
long count = Stream.of("a", "b", "", "c").filter(as(String::isEmpty).negate()).count();
```

#### map

`map` 方法将流中的元素映射为其他的值，新的值类型可以和原来的元素类型不同：

```java
// 将字符转换成 ASCII 码
Stream.of('a', 'b', 'c').map(Object::hashCode).forEach(System.out::println);
```

`map` 方法比较简单，但是使用的频率较高，这里我再举个栗子：

```java
/**
* 从文件中按行读取字符串，并将字符串逐行转换成实体对象
*
* @param path 文件路径
* @return 转换后的实体类列表
* @throws IOException 打开文件过程中的IO异常
*/
public List<Entity> parseToEntityList(Path path) throws IOException {

    try (Stream<String> stream = Files.lines(path)) {
        return stream.filter(as(String::isEmpty).negate())
                .map(Entity::valueOf) // 省略 Entity 类定义 和 valueOf 方法
                .collect(Collectors.toList());
    }
}
```

细心的同学应该注意到了，这个例子在创建和使用流的时候使用了 `try-with-resources` 的形式，这是为啥呢？一般来讲，流不需要手动去关闭，终点操作执行完之后，流就自动关闭了。但是，对于 `Files.lines` 这种会打开外部资源的操作，操作完之后需要手动关闭，从而确保资源正确关闭，不会引起内存泄漏。`Files.lines` 的注释也写明白了：

> This method must be used within a try-with-resources statement or similar control structure to ensure that the stream's open file is closed promptly after the stream's operations have completed.

#### flatMap

`flatMap` 方法结合了 `map` 和 `flattern` 的功能，它能将映射后的流的元素全部放入一个新的流中。该方法定义如下：

```java
<R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper);
```

从`flatMap`的参数来看，mapper 函数接受一个参数并返回一个 Stream，最后 `flatMap` 方法返回的流会包含所有 mapper 返回的流的元素。简单来讲，`flatMap` 会将流中的每一个元素(常见的是集合)都转换成一个流，并将这些流合并起来生成一个新的流，个人感觉有点 “降维” 的意味。看个例子吧：

```java
 // 将多个List 合并成一个 List
List<Integer> numbers =
        Stream.of(List.of(1, 2), List.of(1, 2, 3), List.of(1, 2, 3, 4))
                .flatMap(l -> l.stream()) // 可以替换成更简洁的方法引用形式 flatMap(List::stream)
                .collect(Collectors.toList());
System.out.println(numbers); // [1, 2, 1, 2, 3, 1, 2, 3, 4]
```

拆开来写成这样会不会清楚一些呢：

```java
Function<List<Integer>, Stream<Integer>> mapper = List::stream;
List<Integer> numbers =
        Stream.of(List.of(1, 2), List.of(1, 2, 3), List.of(1, 2, 3, 4))
                .flatMap(mapper)
                .collect(Collectors.toList());
System.out.println(numbers); // [1, 2, 1, 2, 3, 1, 2, 3, 4]
```

前面我们也提到了，方法引用是用于简化生成 Lambda 表达式的一种方式，而 Lambda 表达式都可以赋值给一个函数接口，这样是不是稍微清楚一些了，也就能清楚 `flatMap` 怎么使用了呢。

注： 无论是集合类，还是 Stream 都是使用的泛型接口，清楚的理解泛型及其限定关系(super, extends, ?)十分重要。

#### sorted

对流进行排序可以通过 `sorted` 方法来实现，默认的`sorted()` 将流中的元素按照自然排序方式进行排序，也可以传入排序函数(Comparator接口)来指定排序的方式。例如：

```java
// 获取系统变量，并按key的长度排序
System.getenv()
        .keySet()
        .stream()
        .sorted((x, y) -> x.length() - y.length()) // 还可简写为 .sorted(Comparator.comparingInt(String::length))
        .forEach(System.out::println);
```

这比之前写匿名内部类的方式方便了不要太多，而且还可以通过先对流进行各种map、filter、distinct来减少元素数量后再排序，这样性能会好一点，且代码简洁清晰。

### 终点操作

当终点操作执行后，流就无法再操作了，所以终点操作是流的最后一个操作。

#### forEach

`forEach` 方法遍历流的每一个元素，执行指定的函数。前面的例子也多次用到了，比较简单，不再赘述。与其功能类似的一个中间操作是`peek`，`peek` 一般用于debug。

#### findFirst / findAny

这两个操作都是终点操作兼短路操作(short-circuiting)，即可以不用遍历完所有元素就终止操作的：

```java
Optional<String> result =
        Stream.of("one", "two", "three", "four")
                .peek(s -> System.out.println("Iterated value: " + s))
                .findFirst();
System.out.println(result.orElse("Not found"));
```

这里值得注意的是，`findFirst` 和 `findAny` 返回的都是 `Optional`，可以将它理解为一个容器，它可能含有某个值，或者不包含。我们可以使用 `Optional` 来省去大量的丑陋的判空操作并有效的防止空指针异常。

回到一开始我们的例子，如果CharacterDao可能返回 `null` 的话

```java
public interface CharacterDao {
    @Nullable
    List<Character> getCharacters();

    @Nullable
    Character findOne(Long id);
}
```

在不使用 `Optional` 的情况下，可能会加上几行判空的代码：

```java
public Map<Long, List<Character>> groupCharacterByGamerWithStreamAPI() {
    List<Character> characters = characterDao.getCharacters();
    // 判空
    if (characters == null) {
        characters = Collections.emptyList();
    }
    return characters
            .stream()
            .filter(character -> character.getLevel() > 60)
            .collect(Collectors.groupingBy(Character::getGamerId));
}
```

再看下使用 `Optional` 的话，我们可以这样写：

```java
public Map<Long, List<Character>> groupCharacterByGamerWithStreamAPI() {
    List<Character> characters = characterDao.getCharacters();
    return Optional.ofNullable(characters)
            .orElse(Collections.emptyList())
            .stream()
            .filter(character -> character.getLevel() > 60)
            .collect(Collectors.groupingBy(Character::getGamerId));
}
```

这样的话，看起来就简洁一些。再比如需要在返回结果为空的时候抛出异常，就可以这样写：

```java
public Character findCharacterById(Long id) {
    return Optional.ofNullable(characterDao.findOne(id))
            .orElseThrow(() -> new IllegalArgumentException(id + " not exists"));
}
```

#### reduce

`reduce` 这个操作主要是把流中的元素组合起来生成一个值。它提供一个初始值(种子)，然后根据运算规则(BinaryOperator)，和前面Stream的第一个、第二个、第n个元素组合。通过`reduce`我们可以实现例如字符串拼接、数值求和等功能：

```java
// 求和
int sum = IntStream.range(1,5).reduce(0, (a,b) -> a + b);
// 字符串拼接
String contact = Stream.of("a","b","c").reduce("", String::concat);
// 字符串拼接，无种子，返回值为Optional，注意与上面的区别
contact = Stream.of("a","b","c").reduce(String::concat).get();
```

#### collect

`collect` 在上面的例子也见过很多次了，这个操作是一个可变聚合(mutable reduction)操作，能将流中的元素累积到一个可变容器中。`java.util.stream.Collectors`这个辅助类来辅助进行各种reduction操作。

`Collectors` 主要包含了一些特定的收集器，如平均值`averagingInt`、最大最小值`maxBy` `minBy`、计数`counting`、分组`groupingBy`、字符串连接`joining`、分区`partitioningBy`、汇总`summarizingInt`、化简`reducing`、转换`toXXX`等。

下面举几个栗子：

__averagingInt__

```java
// 求字符串长度平均值
Double avg = Stream.of("abc", "bc", "c").collect(Collectors.averagingInt(String::length));
```

__groupingBy__

分组，在开头的例子里我们就使用到了分组，这里就不另举例子了。

__partitioningBy__

分区，其实是一种特殊的 `groupingBy`，它依照条件测试的是否两种结果来构造返回的数据结构。例如：

```java
// 按等级是否大于60分区
public Map<Boolean, List<Character>> pationCharacterByLevel60() {
    List<Character> characters = characterDao.getCharacters();
    return Optional.ofNullable(characters)
            .orElse(Collections.emptyList())
            .stream()
            .collect(Collectors.partitioningBy(character -> character.getLevel() > 60));
}
```

## 0xXX 参考资料

+ [Java 8 中的 Streams API 详解](https://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/index.html)
+ [深入浅出 Java 8 Lambda 表达式](http://blog.oneapm.com/apm-tech/226.html)
