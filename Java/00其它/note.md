# 记录

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
