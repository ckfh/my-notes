# Spring开发

Spring是一个支持快速开发Java EE应用程序的框架。它提供了一系列底层容器和基础设施，并可以和大量常用的开源框架无缝集成，可以说是开发Java EE应用程序的必备。

随着Spring越来越受欢迎，在Spring Framework基础上，又诞生了Spring Boot、Spring Cloud、Spring Data、Spring Security等一系列基于Spring Framework的**框架**。

## Spring Framework

Spring Framework主要包括几个模块：

- 支持IoC和AOP的容器；
- 支持JDBC和ORM的数据访问模块；
- 支持声明式事务的模块；
- 支持基于Servlet的MVC开发；
- 支持基于Reactive的Web开发；
- 以及集成JMS、JavaMail、JMX、缓存等其他模块。

## 理解

Spring作进一步解耦可以简单理解为它利用静态工厂方法，在其内部创建各类对象，而在需要各类对象的地方运用静态方法获取并赋值。

IoC思想基于IoC容器完成，IoC容器底层就是一个对象工厂，它利用配置解析获取bean的名称，再利用反射创建对象并返回。

工厂方法可以隐藏创建产品的细节，且不一定每次都会真正创建产品，完全可以返回缓存的产品，从而提升速度并减少内存消耗。

总是引用接口而非实现类，能允许变换子类而不影响调用方，即尽可能面向抽象编程。
