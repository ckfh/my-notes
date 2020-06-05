# 函数式编程

- 函数式编程（Functional Programming）是把函数作为基本运算单元，函数可以作为变量，可以接收函数，还可以返回函数。历史上研究函数式编程的理论是Lambda演算，所以我们经常把支持函数式编程的编码风格称为Lambda表达式。

## Lambda基础

- Java的方法分为实例方法以及静态方法，无论是实例方法，还是静态方法，本质上都相当于过程式语言的函数。
- **只不过Java的实例方法隐含地传入了一个this变量，即实例方法总是有一个隐含参数this。**

### Lambda表达式

- 可以用Lambda表达式替换单方法接口。

```Java
/*
 * 1.新建一个 Comparator 子类，实现 compare 方法，实例化一个对象并作为实参传入。
 * 2.匿名内部类，直接在参数位置 new Comparator 并实现 compare 方法。
 * 3.lambda 表达式，只要写出方法定义，参数类型可省略和返回值由编译器自动推断。
 */
Person[] array = new Person[] { new Person("cat", 22), new Person("dog", 23) };
Arrays.sort(array, (p1, p2) -> p2.getAge() - p1.getAge()); // 按照年龄对数组进行降序排序
```

### FunctionalInterface

- 把只定义了单方法的接口称之为FunctionalInterface，用注解@FunctionalInterface标记。

```Java
@FunctionalInterface
public interface Callable<V>
```

## 方法引用

- 所谓方法引用，是指如果某个**方法签名**和接口恰好一致，就可以直接传入方法引用。
- **在这里，方法签名只看参数类型和返回类型，不看方法名称，也不看类的继承关系。**

```Java
/*
 * Comparator<Person> -> int compare(Person, Person) 和静态方法 int cmp(Person, Person) 相比，
 * 除了方法名外，参数类型和返回类型相同，两者方法签名一致，直接把方法名当作 lambda 表达式传入。
 */
public static void main(String[] args) {
    Person[] array = new Person[] { new Person("cat", 22), new Person("dog", 23) };
    Arrays.sort(array, LambdaTest::cmp);
}
static int cmp(Person p1, Person p2) {
    return p2.getAge() - p1.getAge();
}
/*
 * Comparator<Person> -> int compare(Person, Person) 和实例方法 int compareAge(Person) 相比，
 * 后者签名只有一个参数，但由于实例方法有一个隐含的 this 参数，在实际调用的时候，第一个隐含参数总是传入 this，
 * 相当于静态方法 public static int compareAge(this, Person other)，因此也可作为方法引用传入。
 */
Arrays.sort(array, Person::compareAge);
// 来自 Person 类的实例方法 compareAge
public int compareAge(Person other) {
    return other.getAge() - this.getAge();
}
```

### 构造方法引用

```Java
public Person(String name) {
    super();
    this.name = name;
}
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}
/*
 * map() 需要传入的 FunctionalInterface 定义如上所示，将泛型对应之后得到方法签名 Person apply(String)，
 * 即传入参数 String，返回类型 Person。而 Person 类的构造方法恰好满足这个条件，虽然构造方法没有 return 语句，
 * 但实际上隐式地返回了 this 实例，类型就是 Person。
 */
List<String> names = new ArrayList<String>();
names.add("Bob");
names.add("Alice");
names.add("Tim");
List<Person> persons = names.stream().map(Person::new).collect(Collectors.toList());
```

### 小结

- FunctionalInterface允许传入：接口的实现类（传统写法，代码较繁琐）；Lambda表达式（只需列出参数名，由编译器推断类型）；符合方法签名的静态方法；符合方法签名的实例方法（实例类型被看做第一个参数类型）；符合方法签名的构造方法（实例类型被看做返回类型）。
- FunctionalInterface不强制继承关系，不需要方法名称相同，只要求方法参数（类型和数量）与方法返回类型相同，即认为方法签名相同。

## 使用Stream

- Stream API的基本用法就是：创建一个Stream，然后做若干次转换，最后调用一个求值方法获取真正计算的结果。
- Stream API的特点是：Stream API提供了一套新的流式处理的抽象序列；Stream API支持函数式编程和链式操作；Stream可以表示无限序列，并且大多数情况下是惰性求值的。

### 创建Stream

```Java
Stream<String> stream = Stream.of("A", "B", "C", "D"); // 指定元素创建
stream.forEach(System.out::println);
```

```Java
Stream<String> stream1 = Arrays.stream(new String[] { "A", "B", "C" }); // 指定数组创建
Stream<String> stream2 = Arrays.asList("X", "Y", "Z").stream(); // 指定 Collection 创建
```

```Java
Stream<Integer> natual = Stream.generate(new Supplier<Integer>() {
    private AtomicInteger ai = new AtomicInteger(0);

    @Override
    public Integer get() {
        return this.ai.incrementAndGet();
    }
}); // 基于 Supplier 创建，这种 Stream 保存的不是元素，而是算法，可用来表示无限序列
natual.limit(20).forEach(System.out::println); // 无限序列必须变成有限序列再调用最终求值操作
```

```Java
// 通过一些 API 提供的接口，直接获得 Stream。

// Files 类的 lines() 方法把一个文件变成一个 Stream，每个元素代表文件的一行内容。此方法对于按行遍历文本文件十分有用。
try (Stream<String> lines = Files.lines(Paths.get(".\\static\\file.txt"))) {
    lines.forEach(System.out::println);
}
// Pattern 对象有一个 splitAsStream() 方法，可以直接把一个长字符串分割成 Stream 序列而不是数组
Pattern p = Pattern.compile("\\s+");
Stream<String> s = p.splitAsStream("The quick brown fox jumps over the lazy dog");
s.forEach(System.out::println);
```

```Java
// 为了保存 int，只能使用 Stream<Integer>，但这样会产生频繁的装箱、拆箱操作。
IntStream is = Arrays.stream(new int[] { 1, 2, 3 });
// mapToLong() 调用引用方法将每个元素作为参数传入得到 Long 类型的返回值，最后返回一个 LongStream 序列。
LongStream ls = Arrays.asList("1", "2", "3").stream().mapToLong(Long::parseLong);
```

```Java
// 编写一个能输出斐波那契数列的 LongStream。
LongStream fiboStream = LongStream.generate(new LongSupplier() {
    private long t1 = 0;
    private long t2 = 1;
    private long nextTerm;

    @Override
    public long getAsLong() {
        this.nextTerm = t1 + t2;
        this.t1 = t2;
        this.t2 = nextTerm;
        return this.t1;
    }
});
fiboStream.limit(10).forEach(System.out::println);
```
