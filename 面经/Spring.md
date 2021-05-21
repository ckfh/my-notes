## Spring Boot 自动装配原理

从启动类注解向上找，找到 `@Import(AutoConfigurationImportSelector.class)` 注解。

借助 `ClassLoader.getResources(String name)` 从所有的 `META-INF/spring.factories` 文件中读取一系列的组件名称导入到容器当中，其中最主要的是位于 `spring-boot-autoconfigure-x.x.x.jar` 包下的 `spring.factories` 文件，它包含了所有 Spring Boot 帮助我们进行自动配置的自动配置类全限定类名，这些自动配置类都是由该包自己提供的。

**即规定属性文件名，然后在其中写死所有的自动配置类名称。**

虽然会读到将近 100 多个的自动配置类，并且这些自动配置类中包含了许多组件，但是得益于 `@Conditional` 注解所提供的**条件装配**功能，可以非常方便地执行**按需开启自动配置**，即相当多的自动配置类，只有在容器当中存在指定组件，或不存在指定组件，或类路径下没有指定类的定义时才会导入到容器当中，包括定义在这些自动配置类当中的各种组件也有各式各样的条件装配注解。

## IoC 思想

它是一种设计思想，将手动创建对象的控制权交给框架来管理，**将组件的创建、配置与组件的使用相分离**。

当我们需要创建一个对象的时候，只需要配置好配置文件或者注解即可，完全不用考虑对象是如何被创建出来的。

## AOP 思想

本质就是一个代理模式。

对于带有接口的被代理类将采用 JDK 动态代理的方式进行 AOP 实现。

对于没有接口的被代理类将采用 CGLIB 第三方库在运行期实现动态织入。

## MVC 工作原理

<img src="http://my-blog-to-use.oss-cn-beijing.aliyuncs.com/18-10-11/49790288.jpg">

## Bean 的生命周期

<img src="http://my-blog-to-use.oss-cn-beijing.aliyuncs.com/18-9-17/5496407.jpg">

## 事务支持以及事务的传播级别

框架通过 `AOP` 代理，在执行业务逻辑之前开启一个事务，一旦执行过程中出现 `RuntimeException` 异常，将直接回滚数据。

事务的默认传播级别是 `REQUIRED`，它的意思是如果当前没有事务，则创建一个事务，如果当前有事务，则加入当前事务。

还有比较常见的是 `SUPPORTS`，它是如果有事务就加入，没事务也不开启事务执行。`REQUIRES_NEW` 表示不管当前有没有事务，都将开启一个新的事务执行，如果当前有事务，则当前事务会被挂起。

## 一个事务方法是如何知道当前是否存在事务

Spring 使用声明式事务，最终也是借助 `JDBC` 事务来实现，答案是使用 `ThreadLocal`，Spring 总是把 `JDBC` 相关的 `Connection(native)` 和 `TransactionStatus(Spring)` 实例绑定到 `ThreadLocal`。如果一个事务方法从 `ThreadLocal` 未取到事务，那么它会打开一个新的 `JDBC` 连接，同时开启一个新的事务，否则，它就直接使用从 `ThreadLocal` 获取的 `JDBC` 连接以及 `TransactionStatus`。

**因此，事务正确传播的一个前提是，方法调用在一个线程内执行，换句话说，事务只能在当前线程传播，无法跨线程传播。**

