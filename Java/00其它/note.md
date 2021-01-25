# 记录

## Servlet的URL Patterns

[参考](https://help.perforce.com/hydraexpress/3.5.0/html/rwsfservletug/4-3.html)

1. 容器更喜欢精确匹配，而不是通配符匹配。
2. 容器更喜欢匹配长的URL Patterns。
3. 其它URL Patterns无法匹配的请求都有`/`来进行匹配。
4. `/`、`/*`、`/**`偏好程度依次递减。

[参考](https://help.perforce.com/hydraexpress/3.5.0/html/rwsfservletug/4-8.html#482)

1. Filter的URL Patterns通常写成`/*`、`/status/*`等形式，这是因为它按照的是**起始匹配**，即前者将过滤以`/`开头的请求，后者将过滤以`/status/*`开头的请求，因此本质上前者是包含后者的。

## 有关Spring中的@Autowired注解忽略问题

我们知道，在Spring当中一些需要自动装入组件的地方如果我们不写@Autowired注解，容器也会自动帮我们从容器当中获取组件并注入，例如某组件构造方法参数需要其它组件，配置类当中@Bean方法参数是另一个组件，即使在这些位置我们不写@Autowired注解，容器也会帮我们自动注入容器当中的组件。

个人理解，因为首先上述的两个都是要注册到容器当中的组件，既然是组件，就必须要完成实例化过程，为了完成实例化过程，容器即使没有发现@Autowired注解，它同样会从容器当中获取组件保证组件的成功实例化。

[参考](https://segmentfault.com/q/1010000019281201)

## 有关回调这个名词的解释

其实可以简单地理解为`Python`当中对函数式编程的概念：将方法作为变量传入，在方法当中调用另外一个方法。

> 函数式编程的一个特点就是，允许把函数本身作为参数传入另一个函数，还允许返回一个函数！

在`Java`当中，没法简单地像`Python`那样将一个方法作为另外一个方法的变量，我们能做的就是将前一个方法所在的类作为另外一个方法的变量。

```java
public class TaxTemplate {
    public double getTax(double total, Callable<Double> taxBase) {
        return (total - taxBase.call()) * 0.1;
    }
}
```

```java
public Object queryForObject(String sql, RowMapper mapper) {
    try (conn = connect(...)) {
        try (Statement st = conn.createStatement(sql)) {
            try (ResultSet rs = st.executeQuery()) {
                if (rs.next()) {
                    return mapper.mapRow(rs);
                }
            }
        }
    }
}
```

综合上述两端代码的演示，`getTax()`和`queryForObject()`方法就可以根据我们传入的“方法”实现不同的功能。

## 整型位数问题

Java中整型占用4个字节即32个比特位，由于最高位被当作符号位，因此其正数表示范围大小在`2^31=2147483648`。

注意这个`2147483648`是**表示范围大小**，其取值区间在`[0, 2147483647]`，因此整型最大值为`2^31-1`。

## 二进制要点

[参考](https://www.cnblogs.com/blog-cq/p/5793529.html)

[参考](https://www.cnblogs.com/lukelook/p/11274795.html)

Java中负数的二进制采用补码表示，对于可表示正负的基本类型来说，需要保留最高位作为符号位，比如`int`类型占用4个字节，但实际最大值只有`2^31`，最高位用于表示正负数。

正数的二进制表示与其原码一致，负数的二进制表示则是先对原码进行取反（除符号位外），然后加一。

另外Java会对左右移的位数取类型占用位数的模，例如对`int`类型左移33位，相当于左移1位。

`-5`的原码表示`10000000 00000000 00000000 00000101`，取反（不包括符号位）得到`11111111 11111111 11111111 11111010`，加一得到`11111111 11111111 11111111 11111011`。

`<<`左移操作就是低位补零，`>>`右移操作就是高位补符号位，可以理解为符号位不动，空缺处用符号位填充，`>>>`无符号右移操作则是高位补零。

## 对优先级队列进行迭代不保证某种顺序

优先级队列只有在使用 poll() 和 peek() 方法时保证返回优先级最高的那一个元素，而对其进行迭代时不保证特定顺序。

> Returns an iterator over the elements in this queue. The iterator does not return the elements in any particular order.

## 线程池命名

在项目中导入数据库连接池驱动时，因为其本质上是一个线程池，因此后续手动创建的线程池的默认命名不会从1开始，因为1号线程池是数据库连接池，而一般手动创建的线程池需要给定一个有意义的命名，方便回溯线程以及线程池。

## 集合和数组工具类都有相应的二分查找元素的方法

```java
Arrays.binarySearch
Collections.binarySearch
```

## 将一个整型数组转换为整型集合

```java
// Arrays.stream(arr) 返回 IntStream
// IntStream.boxed 返回一个 Stream，其元素都被包装为 Integer
// 最后转换为整型包装类集合
List<Integer> list = Arrays.stream(arr).boxed.collect(Collectors.toList());
```

## throw和return

抛出错误和返回结果都会结束当前方法的执行流程。

## 面向接口编程的威力

```java
package javax.sql;

public interface DataSource  extends CommonDataSource, Wrapper {
    ...
    Connection getConnection() throws SQLException;
    ...
}
```

```java
package com.zaxxer.hikari;

public class HikariDataSource extends HikariConfig implements DataSource, Closeable {
    ...
    @Override
    public Connection getConnection() throws SQLException
    {
        ...
    }
    ...
}
```

```java
// 我们在编码时只需要引用接口:
public Connection getConnection(DataSource ds) {
    return ds.getConnection();
}
// 从hikari连接池中获取连接:
Connection conn = getConnection(new HikariDataSource());
// 从c3p0连接池中获取连接:
Connection conn = getConnection(new c3p0());
```

## 一个不知道有没有用的笔记

ResultSet.getRow()方法它返回的不是数据表中的行号，而是当前返回的数据集中的行号，第一条记录行号为1，第二条记录行号为2，依此类推；而在JdbcTemplate中接触到的函数式接口RowMapper的mapRow(ResultSet rs, int rowNum)方法参数当中的行号是从0开始的，当然你依旧从ResultSet对象中获取行号那还是从1开始的。

## Spring容器注入报错方式

`NullPointerException`没有注入到字段，`NoSuchBeanDefinitionException`找不到实例注入字段。

## 使用数据库时报错没有合适的驱动

`No suitable driver`，请检查是否导入了对应的`mysql-connector-java`包。

## 路径映射问题

```java
// 不管后缀是什么都会被匹配
@WebFilter(urlPatterns = "/*")
// 优先匹配有具体后缀的路径，其余不匹配路径都由该路径接收
@WebFilter(urlPatterns = "/")
```

## URI

```java
// Web App名称为hello，访问/hello/index
req.getRequestURI(); // "/hello/index"
req.getContextPath(); // "/hello"
req.getRequestURI().substring(req.getContextPath().length()); // "/index"
// Web App名称为ROOT，访问/index
req.getRequestURI(); // "/index"
req.getContextPath(); // ""
req.getRequestURI().substring(req.getContextPath().length()); // "/index"
```

## TOMCAT命令行乱码

打开conf目录里的context文件，将所有的`UTF-8`替换为`GBK`保存。

## getResourceAsStream()方法探究

```java
// classes目录根路径
ClassLoader.getResourceAsStream("filename")
// classes目录中和.class文件相同路径
Class.getResourceAsStream("filename")
// classes目录根路径
Class.getResourceAsStream("/filename")
```

## 转义与不转义

在字符前加`\`表示转义，而在字符串前加`r`则表示默认不转义。

## SQL语句自动提交以及事务处理

在Spring Boot配置文件当中关闭了数据库自动提交选项，那么在所有出现SQL语句的地方都需要加上事务处理。

## 有关web应用程序中为请求和响应进行强制转码UTF-8

场景：URL携带中文参数请求一个JSON数据，JSON数据包含中文。如果直接返回字符串对象，例如`String.format("{\"name\":\"%s\"}", name)`，必须在请求头`Accept`中指定编码格式`application/json;charset=utf-8`，否则返回的响应消息中的响应头`Content-Type`会默认以`application/json;charset=ISO-8859-1`的其它编码格式在浏览器进行展示，这样即使将响应消息进行转码为`UTF-8`，但是浏览器本地还是以其它编码格式进行展示，导致中文乱码；但是使用如果不是返回字符串对象，例如`Map.of("name", name)`，那么浏览器本地默认以`UTF-8`进行编码展示。

## 格式化字符串

```Java
// 32位16进制整数（不足32位则左边补0）：
String.format("%032x", new BigInteger(1, hash)
```

## JVM、Java 编译器、Java 解释器

[CSDN 文章](https://blog.csdn.net/wangaiheng/article/details/78343260)

## 无法从阿里 Maven 源解析到依赖

大概率是网络问题，修改 dns 后再次进行尝试。我这边的实际场景是百度 dns 导致了无法解析到阿里的 Maven 源，尝试换回校园网 dns 后解析成功。

## 整数溢出

```Java
System.out.println((2147483640 + 2147483642) / 2);              // 结果错误
System.out.println(2147483640 + (2147483642 - 2147483640) / 2); // 结果正确
System.out.println((2147483640 + 2147483642) >>> 1);            // 结果正确
```

## SLF4J 和 Logback 搭配使用

使用 Maven 中央仓库提供的依赖条目导入包后，在 main 方法中进行日志打印报错，原因在于中央条目中将 logback-classic 包的 scope 设置为 test，而 main 方法自身不属于测试环境，因此需要修改该依赖的 scope 为 compile，即可进行日志打印。[参考链接](https://blog.csdn.net/wang465745776/article/details/80384210)。

## 部署 war 包

默认架构的 `Maven web` 项目在 `webapp` 文件夹中提供了 `index.jsp` 文件，当访问 `/` 路径而没有指定访问文件名时，默认使用该文件内容作为响应消息体进行返回。

场景复现：访问 `127.0.0.1:8080/hello/` ， `/hello` 是第一级目录表示 `web app` 的名字，后面的 `/` 才是需要映射的路径，于是转发到映射了该地址的一个 `servlet` 实例，该实例简单地写入了响应内容返回。但是，这个响应内容会被默认访问的 `index.jsp` 的内容覆盖，导致无论怎么修改响应内容最后显示的都是 `index.jsp` 中的内容。

解决方法：删除 `index.jsp` 文件。

## Maven 环境配置

1. 远程仓库修改、本地仓库修改、JDK 版本修改。
2. Eclipse IDE 配置：增加本地 Maven 安装目录、修改全局配置文件地址。
3. 下载 archetype-catalog 文件，增加到 IDE 的 catelog 文件目录当中。
4. 新建 Maven 项目时，选择 org.apache. Maven.archetypes 所属的 Maven-archetype-quickstart 原型进行创建。

## 值传递和引用传递

[掘金文章](https://juejin.im/post/5bce68226fb9a05ce46a0476)

对基本数据类型的数据进行操作，原始内容和副本都是存储实际值，但是两者是在不同的栈区，因此对形参的修改不会影响到原始内容。

```Java
public static void main(String[] args) {
    int x = 20;
    System.out.printf("main() x = %d\n", x); // main() x = 20
    foo(x);
    System.out.printf("main() x = %d\n", x); // main() x = 20
}
static void foo(int x) {
    System.out.printf("foo() x = %d\n", x); // foo() x = 20
    x = 10;
    System.out.printf("foo() x = %d\n", x); // foo() x = 10
}
```

Java 中任何引用类型变量的赋值内容从来都是一块堆内存的地址，所以不要妄想做出先用该变量指向一个堆内存地址，然后再指向另外一个堆内存地址，以期望对新地址的堆内存内容进行修改时能够改变旧地址指向的堆内存内容这样的做法。

```Java
public static void main(String[] args) {
    Person per = new Person();
    per.setName("cat");
    System.out.println(per); // Person [name=cat]
    // 传递地址值
    foo(per);
    System.out.println(per); // Person [name=dog]
}
static void foo(Person per) {
    System.out.println(per); // Person [name=cat]
    // 没有发生改变地址值的操作，因此修改会改变实参指向的对象内容
    per.setName("dog");
    System.out.println(per); // Person [name=dog]
}
```

```Java
public static void main(String[] args) {
    Person per = new Person();
    per.setName("cat");
    System.out.println(per); // Person [name=cat]
    // 传递地址值
    foo(per);
    System.out.println(per); // Person [name=cat]
}
static void foo(Person per) {
    System.out.println(per); // Person [name=cat]
    // 注意此时改变了实参传给形参的地址值，因此修改并不会影响原来实参指向的对象内容
    per = new Person();
    per.setName("dog");
    System.out.println(per); // Person [name=dog]
}
```
