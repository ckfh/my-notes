# IoC容器

**容器是一种为某种特定组件的运行提供必要支持的一个软件环境**。例如，Tomcat就是一个Servlet容器，它可以为Servlet的运行提供运行环境。类似Docker这样的软件也是一个容器，它提供了必要的Linux环境以便运行一个特定的Linux进程。

通常来说，**使用容器运行组件，除了提供一个组件运行环境之外，容器还提供了许多底层服务**。例如，Servlet容器底层实现了TCP连接，解析HTTP协议等非常复杂的服务，如果没有容器来提供这些服务，我们就无法编写像Servlet这样代码简单，功能强大的组件。早期的JavaEE服务器提供的EJB容器最重要的功能就是通过声明式事务服务，使得EJB组件的开发人员不必自己编写冗长的事务处理代码，所以极大地简化了事务处理。

Spring的核心就是提供了一个**IoC容器**，**它可以管理所有轻量级的JavaBean组件，提供的底层服务包括组件的生命周期管理、配置和组装服务、AOP支持，以及建立在AOP基础上的声明式事务服务等**。

以下都是个人理解：

  1. `IoC`容器给我的最大感受就是在你在编写组件业务逻辑时，你不需要去纠结某个将被调用的实例该如何去创建，你就定义好成员字段或者方法参数假设“此刻”已经拥有了这个实例，直接用就行。
  2. 比如说我正在编写一个组件用于访问数据库，那我需要有一个`DataSource`的实例对象来帮助实现功能，我“此刻”在编写这个组件的时候，我可以选择将其定义为成员属性或者方法参数，直接用就行，至于真正的实例对象什么时候创建以及传入通通都交给容器来完成。
  3. 本质上就是把实例创建的及时必要性给打破了，你可以提前或往后去`new`这个实例，然后把这个实例交给容器，让容器帮助你注入到需要该实例的地方。
  4. 这就好比我们定义一个普通类时也都是只定义好成员类型及变量名，创建类实例后再使用相应的`setXxx()`方法给这些成员属性进行实例注入，然后调用相应的业务逻辑，但是实例注入这个步骤容器也帮助我们完成了，我们写`XML`配置文件实际上就是在写前面的过程。
  5. 把`new`的过程抽取出来，`new`是肯定要`new`的，`new`完以后直接通过字段变量或者`setXxx()`方法或者构造方法把`new`的对象传进去。
  6. 只不过我们把`new`的过程交给了容器去做，写`XML`还算是让我们把`new`以及传值的操作用非`Java`语言写一遍，但是注解就彻彻底底把这个过程都帮我们省略掉了，当然在创建第三方`bean`的时候还是写了`new`的过程，但是传值就不需要了，把`new`后的对象交给容器去注入。
  7. 你可以想象在主方法里写一大堆的`new`以及各种注入（变量、`setXxx`、构造）操作，还不如统一到配置文件里，或者说就不需要，让容器去完成就好了。

以上的理解都可以用底下文章中的一句话解决：**将组件的创建➕配置与组件的使用相分离**。

```java
// 此前开发
public class UserDaoImpl {
    // 定义成员变量的同时进行赋值:
    HikariConfig config = new HikariConfig();
    DataSource dataSource = new HikariDataSource(config);
    public void register() {
        try (Connection conn = this.dataSource.getConnection()) {
            ...
        }
    }
}

// 后来开发
public class UserDaoImpl {
    DataSource dataSource;
    // 定义好成员变量然后通过SET方法进行实例赋值:
    public void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public void register() {
        try (Connection conn = this.dataSource.getConnection()) {
            ...
        }
    }
}
public class Main {
    public static void main(String[] args) {
        // 在主方法中进行各种实例化操作以及注入操作:
        HikariConfig config = new HikariConfig();
        DataSource dataSource = new HikariDataSource(config);
        UserDaoImpl userDao = new UserDaoImpl();
        userDao.setDataSource(dataSource);
        userDao.register();
    }
}

// 借助框架开发
public class UserDaoImpl {
    DataSource dataSource;

    public void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public void register() {
        try (Connection conn = this.dataSource.getConnection()) {
            ...
        }
    }
}
public class Main {
    public static void main(String[] args) {
        // 实例化以及注入操作都交由容器来完成，我们直接获取实例，然后进行业务逻辑调用即可:
        UserDaoImpl userDao = context.getBean(UserDaoImpl.class);
        userDao.register();
    }
}
```

## IoC原理

IoC全称Inversion of Control，直译为控制反转。在理解IoC之前，我们先看看通常的Java组件是如何协作的。

```Java
// 假定一个在线书店，通过BookService获取书籍：
public class BookService {
    // 为了从数据库查询书籍，BookService持有一个DataSource。为了实例化一个HikariDataSource，又不得不实例化一个HikariConfig。
    private HikariConfig config = new HikariConfig();
    private DataSource dataSource = new HikariDataSource(config);

    public Book getBook(long bookId) {
        try (Connection conn = dataSource.getConnection()) {
            ...
            return book;
        }
    }
}
```

```Java
// 编写UserService获取用户：
public class UserService {
    // 因为UserService也需要访问数据库，因此，我们不得不也实例化一个HikariDataSource。
    private HikariConfig config = new HikariConfig();
    private DataSource dataSource = new HikariDataSource(config);

    public User getUser(long userId) {
        try (Connection conn = dataSource.getConnection()) {
            ...
            return user;
        }
    }
}
```

```Java
// 在处理用户购买的CartServlet中，我们需要实例化UserService和BookService：
public class CartServlet extends HttpServlet {
    private BookService bookService = new BookService();
    private UserService userService = new UserService();

    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        long currentUserId = getFromCookie(req);
        User currentUser = userService.getUser(currentUserId);
        Book book = bookService.getBook(req.getParameter("bookId"));
        cartService.addToCart(currentUser, book);
        ...
    }
}
```

```Java
// 在购买历史HistoryServlet中，也需要实例化UserService和BookService：
public class HistoryServlet extends HttpServlet {
    private BookService bookService = new BookService();
    private UserService userService = new UserService();
}
```

上述每个组件都采用了一种简单的**通过new创建实例并持有**的方式。仔细观察，会发现以下缺点：

  1. 实例化一个组件其实很难，例如，BookService和UserService要创建HikariDataSource，实际上需要读取配置，才能先实例化HikariConfig，再实例化HikariDataSource。
  2. 没有必要让BookService和UserService分别创建DataSource实例，完全可以共享同一个DataSource，但谁负责创建DataSource，谁负责获取其它组件已经创建的DataSource，不好处理。类似的，CartServlet和HistoryServlet也应当共享BookService实例和UserService实例，但也不好处理。
  3. 很多组件需要销毁以便释放资源，例如DataSource，但如果该组件被多个组件共享，如何确保它的使用方都已经全部被销毁。
  4. 随着更多的组件被引入，例如，书籍评论，需要共享的组件写起来会更困难，这些组件的依赖关系会越来越复杂。
  5. 测试某个组件，例如BookService，是复杂的，因为必须要在真实的数据库环境下执行。

从上面的例子可以看出，如果一个系统有大量的组件，其生命周期和相互之间的依赖关系如果**由组件自身来维护**，不但大大增加了系统的复杂度，而且会导致组件之间极为紧密的耦合，继而给测试和维护带来了极大的困难。

因此，核心问题是：**谁负责创建组件**；**谁负责根据依赖关系组装组件**；**销毁时，如何按依赖顺序正确销毁**。解决这一问题的核心方案就是IoC。

传统的应用程序中，**控制权在程序本身，程序的控制流程完全由开发者控制**，例如：CartServlet创建了BookService，在创建BookService的过程中，又创建了DataSource组件。这种模式的缺点是，**一个组件如果要使用另一个组件，必须先知道如何正确地创建它（例如创建时需要传入特定参数）**。

**在IoC模式下，控制权发生了反转，即从应用程序转移到了IoC容器，所有组件不再由应用程序自己创建和配置，而是由IoC容器负责**，这样，应用程序只需要直接使用**已经创建好并且配置好**的组件。为了能让组件在IoC容器中被“装配”出来，需要某种“注入”机制，例如，BookService自己并不会创建DataSource，而是等待外部通过setDataSource()方法来注入一个DataSource：

```Java
public class BookService {
    private DataSource dataSource;

    public void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
    }
}
```

不直接**new**一个DataSource，而是**注入**一个DataSource，这个小小的改动虽然简单，却给**编写和测试**JavaBean带来了一系列好处：

  1. BookService不再关心如何创建DataSource，因此，不必编写读取数据库配置之类的代码；
  2. DataSource实例被注入到BookService，同样也可以注入到UserService，因此，共享一个组件非常简单；
  3. 测试BookService更容易，因为注入的是DataSource，**测试时可以使用内存数据库，而不是真实的MySQL配置**。

因此，IoC又称为依赖注入（DI：Dependency Injection），它解决了一个最主要的问题：**将组件的创建➕配置与组件的使用相分离**，并且，由IoC容器负责管理组件的生命周期。

**因为IoC容器要负责实例化所有的组件，因此，有必要告诉容器如何创建组件，以及各组件的依赖关系**。一种最简单的配置是通过XML文件来实现，例如：

```XML
<beans>
    <bean id="dataSource" class="HikariDataSource" />
    <bean id="bookService" class="BookService">
        <property name="dataSource" ref="dataSource" />
    </bean>
    <bean id="userService" class="UserService">
        <property name="dataSource" ref="dataSource" />
    </bean>
</beans>
```

上述XML配置文件指示IoC容器创建3个JavaBean组件，并把id为dataSource的组件通过属性dataSource（即调用setDataSource()方法）注入到另外两个组件中。在Spring的IoC容器中，我们把所有**组件**统称为**JavaBean**，即**配置一个组件**就是**配置一个Bean**。

### 依赖注入方式

我们从上面的代码可以看到，依赖注入可以通过set()方法实现。但依赖注入也可以通过构造方法实现。很多Java类都具有带参数的构造方法，如果我们把BookService改造为通过构造方法注入，那么实现代码如下：

```Java
public class BookService {
    private DataSource dataSource;

    public BookService(DataSource dataSource) {
        this.dataSource = dataSource;
    }
}
```

Spring的IoC容器同时支持属性注入和构造方法注入，并允许混合使用。

### 无侵入容器

在设计上，Spring的IoC容器是一个高度可扩展的**无侵入容器**。所谓无侵入，是指应用程序的组件无需实现Spring的特定接口，或者说，组件根本不知道自己在Spring的容器中运行。这种无侵入的设计有以下好处：

  1. 应用程序组件既可以在Spring的IoC容器中运行，也可以自己编写代码自行组装配置；
  2. 测试的时候并不依赖Spring容器，可单独进行测试，大大提高了开发效率。

## 装配Bean

为什么要使用Spring的IoC容器，因为让容器来为我们创建并装配Bean能获得很大的好处，那么到底如何使用IoC容器？装配好的Bean又如何使用？

<img src="./image/装配bean-工程结构.jpg">

我们用Maven创建工程并引入`spring-context`依赖。

```java
// 编写一个MailService，用于在用户登录和注册成功后发送邮件通知。
public class MailService {
    private ZoneId zoneId = ZoneId.systemDefault();

    public void setZoneId(ZoneId zoneId) {
        this.zoneId = zoneId;
    }

    public String getTime() {
        return ZonedDateTime.now(this.zoneId).format(DateTimeFormatter.ISO_ZONED_DATE_TIME);
    }

    public void sendLoginMail(User user) {
        System.err.printf("Hi, %s! You are logged in at %s%n", user.getName(), this.getTime());
    }

    public void sendRegistrationMail(User user) {
        System.err.printf("Welcome, %s!%n", user.getName());
    }
}
// 编写一个UserService，实现用户注册和登录。
public class UserService {
    private MailService mailService;
    // 注意到UserService通过setMailService()注入了一个MailService。
    public void setMailService(MailService mailService) {
        this.mailService = mailService;
    }

    private List<User> users = new ArrayList<>(List.of(
            new User(1, "bob@example.com", "password", "Bob"),
            new User(2, "alice@example.com", "password", "Alice"),
            new User(3, "tom@example.com", "password", "Tom")));

    public User login(String email, String password) {
        for (User user : users) {
            if (user.getEmail().equalsIgnoreCase(email) && user.getPassword().equals(password)) {
                mailService.sendLoginMail(user);
                return user;
            }
        }
        throw new RuntimeException("login failed.");
    }

    public User getUser(long id) {
        return this.users.stream().filter(user -> user.getId() == id).findFirst().orElseThrow();
    }

    public User register(String email, String password, String name) {
        users.forEach((user) -> {
            if (user.getEmail().equalsIgnoreCase(email)) {
                throw new RuntimeException("email exist.");
            }
        });
        User user = new User(users.stream().mapToLong(User::getId).max().orElse(0L) + 1, email, password, name);
        users.add(user);
        mailService.sendRegistrationMail(user);
        return user;
    }
}
```

```xml
<!-- 编写一个特定的application.xml配置文件，告诉Spring的IoC容器应该如何创建并组装Bean。 -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">
    <!-- 告诉Spring的IoC容器创建一个userServiceBean，并将mailServiceBean组装进来。 -->
    <bean id="userService" class="com.cat.service.UserService">
        <property name="mailService" ref="mailService"/>
    </bean>
    <!-- 告诉Spring的IoC容器创建一个mailServiceBean。 -->
    <bean id="mailService" class="com.cat.service.MailService"/>

</beans>
```

- 每个`<bean ...>`都有一个`id`标识，相当于Bean的唯一ID；
- 在userServiceBean中，通过`<property name="..." ref="..." />`注入了另一个Bean；
- **Bean的顺序不重要**，Spring根据依赖关系会自动正确初始化。

```java
// 把上述XML配置文件用Java代码写出来，就像这样。
UserService userService = new UserService();
MailService mailService = new MailService();
userService.setMailService(mailService);
```

**只不过Spring容器是通过读取XML文件后使用反射完成的**。

```xml
<!-- 如果注入的不是Bean，而是boolean、int、String这样的数据类型，则通过value注入，例如，创建一个HikariDataSource。 -->
<bean id="dataSource" class="com.zaxxer.hikari.HikariDataSource">
    <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/test" />
    <property name="username" value="root" />
    <property name="password" value="password" />
    <property name="maximumPoolSize" value="10" />
    <property name="autoCommit" value="true" />
</bean>
```

```Java
// 最后一步，我们需要创建一个Spring的IoC容器实例，然后加载配置文件，让Spring容器为我们创建并装配好配置文件中指定的所有Bean，这只需要一行代码。
ApplicationContext context = new ClassPathXmlApplicationContext("application.xml");
// 接下来，我们就可以从Spring容器中“取出”装配好的Bean然后使用它。
// 获取Bean:
UserService userService = context.getBean(UserService.class);
// 正常调用:
User user = userService.login("bob@example.com", "password");
```

### ApplicationContext

可以看到，Spring容器就是ApplicationContext，它是一个接口，有很多实现类，这里我们选择`ClassPathXmlApplicationContext`，表示它会自动从classpath中查找指定的XML配置文件。

获得了ApplicationContext的实例，就获得了IoC容器的引用。从ApplicationContext中我们可以根据Bean的ID获取Bean，但更多的时候我们**根据Bean的Class类型获取Bean的引用**。

```Java
// Spring还提供另一种IoC容器叫BeanFactory，使用方式和ApplicationContext类似。
BeanFactory factory = new XmlBeanFactory(new ClassPathResource("application.xml"));
MailService mailService = factory.getBean(MailService.class);
```

BeanFactory和ApplicationContext的区别在于，BeanFactory的实现是按需创建，即第一次获取Bean时才创建这个Bean，而ApplicationContext会一次性创建所有的Bean。实际上，ApplicationContext接口是从BeanFactory接口继承而来的，并且，ApplicationContext提供了一些额外的功能，包括国际化支持、事件和通知机制等。**通常情况下，我们总是使用ApplicationContext，很少会考虑使用BeanFactory**。因为按需创建的时候，发现依赖有问题再报个错，还不如启动就报错。

## 使用Annotation配置

> Component/Autowired/Configuration/ComponentScan

使用Spring的IoC容器，实际上就是通过类似XML这样的配置文件，**把我们自己的Bean的依赖关系描述出来，然后让容器来创建并装配Bean**。一旦容器初始化完毕，我们就直接从容器中获取Bean使用它们。

使用XML配置的优点是所有的Bean都能一目了然地列出来，并通过配置注入能直观地看到每个Bean的依赖。它的缺点是写起来非常繁琐，每增加一个组件，就必须把新的Bean配置到XML中。我们可以使用Annotation配置，可以完全不需要XML，让Spring自动扫描Bean并组装它们。

```Java
// 首先，我们给MailService添加一个@Component注解：
@Component
public class MailService {
    ...
}
```

这个@Component注解就相当于定义了一个Bean，它有一个可选的名称，默认是mailService，即小写开头的类名。

```Java
// 然后，我们给UserService添加一个@Component注解和一个@Autowired注解：
@Component
public class UserService {
    @Autowired
    MailService mailService;

    ...
}
```

使用@Autowired就相当于把指定类型的Bean注入到指定的字段中。和XML配置相比，@Autowired大幅简化了注入，因为它不但可以写在set()方法上，还可以直接写在字段上，甚至可以写在构造方法中。

```Java
@Component
public class UserService {
    MailService mailService;
    // 写在构造方法中
    public UserService(@Autowired MailService mailService) {
        this.mailService = mailService;
    }
    ...
}
```

我们一般把@Autowired写在字段上，**通常使用package权限的字段，便于测试**。

```Java
// 最后，编写一个AppConfig类启动容器：
@Configuration
@ComponentScan
public class AppConfig {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        UserService userService = context.getBean(UserService.class);
        User user = userService.login("bob@example.com", "password");
        System.out.println(user.getName());
    }
}
```

AppConfig标注了@Configuration，表示它是一个配置类，因为我们创建ApplicationContext时，**使用的实现类是AnnotationConfigApplicationContext，必须传入一个标注了@Configuration的类名**。

此外，AppConfig还标注了@ComponentScan，它告诉容器，**自动搜索当前类所在的包以及子包**，把所有标注为@Component的Bean自动创建出来，并根据@Autowired进行装配。

使用Annotation配合自动扫描能大幅简化Spring的配置，我们只需要保证：每个Bean被标注为@Component并正确使用@Autowired注入；配置类被标注为@Configuration和@ComponentScan；所有Bean均在指定包以及子包内。

使用@ComponentScan非常方便，但是，我们也要特别注意包的层次结构。通常来说，**启动配置AppConfig位于自定义的顶层包，其它Bean按类别放入子包**。

## 定制Bean

> Scope/Order/Bean/PostConstruct/PreDestroy/Qualifier/Primary

- Spring默认使用Singleton创建Bean，也可指定Scope为Prototype；
- 可将相同类型的Bean注入List；
- 可用@Autowired(required=false)允许可选注入；
- 可用带@Bean标注的方法创建Bean；
- 可使用@PostConstruct和@PreDestroy对Bean进行初始化和清理；
- 相同类型的Bean只能有一个指定为@Primary，其它必须用@Quanlifier("beanName")指定别名；
- 注入时，可通过别名@Quanlifier("beanName")指定某个Bean；
- 可以定义FactoryBean来使用工厂模式创建Bean。

对于Spring容器来说，当我们把一个Bean标记为@Component后，它就会自动为我们创建一个单例（Singleton），即容器初始化时创建Bean，容器关闭前销毁Bean。在容器运行期间，我们调用getBean(Class)获取到的Bean总是同一个实例。还有一种Bean，我们每次调用getBean(Class)，容器都返回一个新的实例，这种Bean称为Prototype（原型），它的生命周期显然和Singleton不同。声明一个Prototype的Bean时，需要添加一个额外的@Scope注解。

```Java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE) // @Scope("prototype")
public class MailSession {
    ...
}
```

有些时候，我们会有一系列接口相同，不同实现类的Bean。例如，注册用户时，我们要对email、password和name这3个变量进行验证。为了便于扩展，我们先定义验证接口。然后，分别使用3个Validator对用户参数进行验证。最后，我们通过一个Validators作为入口进行验证。注意到Validators被注入了一个`List<Validator>`，Spring会自动把所有类型为Validator的Bean装配为一个List注入进来，这样一来，我们每新增一个Validator类型，就自动被Spring装配到Validators中了，非常方便。

```Java
public interface Validator {
    void validate(String email, String password, String name);
}

@Component
public class EmailValidator implements Validator {
    public void validate(String email, String password, String name) {
        if (!email.matches("^[a-z0-9]+\\@[a-z0-9]+\\.[a-z]{2,10}$")) {
            throw new IllegalArgumentException("invalid email: " + email);
        }
    }
}

@Component
public class PasswordValidator implements Validator {
    public void validate(String email, String password, String name) {
        if (!password.matches("^.{6,20}$")) {
            throw new IllegalArgumentException("invalid password");
        }
    }
}

@Component
public class NameValidator implements Validator {
    public void validate(String email, String password, String name) {
        if (name == null || name.isBlank() || name.length() > 20) {
            throw new IllegalArgumentException("invalid name: " + name);
        }
    }
}

@Component
public class Validators {
    @Autowired
    List<Validator> validators;

    public void validate(String email, String password, String name) {
        for (var validator : this.validators) {
            validator.validate(email, password, name);
        }
    }
}
```

因为Spring是通过扫描classpath获取到所有的Bean，而List是有序的，要指定List中Bean的顺序，可以加上@Order注解。

```Java
@Component
@Order(1)
public class EmailValidator implements Validator {
    ...
}

@Component
@Order(2)
public class PasswordValidator implements Validator {
    ...
}

@Component
@Order(3)
public class NameValidator implements Validator {
    ...
}
```

默认情况下，当我们标记了一个@Autowired后，Spring如果没有找到对应类型的Bean，它会抛出NoSuchBeanDefinitionException异常。可以给@Autowired增加一个required = false的参数。这个参数告诉Spring容器，如果找到一个类型为ZoneId的Bean，就注入，如果找不到，就忽略。**这种方式非常适合有定义就使用定义，没有就使用默认值的情况**。

```Java
@Component
public class MailService {
    @Autowired(required = false)
    ZoneId zoneId = ZoneId.systemDefault();
    ...
}
```

如果一个Bean不在我们自己的package管理之类，例如ZoneId，如何创建它。答案是我们自己在@Configuration类中编写一个Java方法创建并返回它，**注意给方法标记一个@Bean注解**。**Spring对标记为@Bean的方法只调用一次，因此返回的Bean仍然是单例**。

```Java
@Configuration
@ComponentScan
public class AppConfig {
    // 创建一个Bean:
    @Bean
    ZoneId createZoneId() {
        return ZoneId.of("Z");
    }
}
```

有些时候，一个Bean在注入必要的依赖后，需要进行初始化（监听消息等）。在容器关闭时，有时候还需要清理资源（关闭连接池等）。我们通常会定义一个init()方法进行初始化，定义一个shutdown()方法进行清理，然后，引入JSR-250定义的Annotation。在Bean的初始化和清理方法上标记@PostConstruct和@PreDestroy。Spring容器会对下述Bean做如下初始化流程：调用构造方法创建MailService实例；根据@Autowired进行注入；调用标记有@PostConstruct的init()方法进行初始化。而销毁（要手动调用applicationContext.close()不能直接点结束进程）时，容器会首先调用标记有@PreDestroy的shutdown()方法。Spring只根据Annotation查找**无参数**方法，对方法名不作要求。

```XML
<dependency>
    <groupId>javax.annotation</groupId>
    <artifactId>javax.annotation-api</artifactId>
    <version>1.3.2</version>
</dependency>
```

```Java
@Component
public class MailService {
    @Autowired(required = false)
    ZoneId zoneId = ZoneId.systemDefault();

    @PostConstruct
    public void init() {
        System.out.println("Init mail service with zoneId = " + this.zoneId);
    }

    @PreDestroy
    public void shutdown() {
        System.out.println("Shutdown mail service");
    }
}
```

默认情况下，对一种类型的Bean，容器只创建一个实例。但有些时候，我们需要对一种类型的Bean创建多个实例。

```Java
// 例如，同时连接多个数据库，就必须创建多个DataSource实例。
// 如果我们在@Configuration类中创建了多个同类型的Bean，Spring会报NoUniqueBeanDefinitionException异常，意思是出现了重复的Bean定义。
@Configuration
@ComponentScan
public class AppConfig {
    @Bean
    ZoneId createZoneOfZ() {
        return ZoneId.of("Z");
    }

    @Bean
    ZoneId createZoneOfUTC8() {
        return ZoneId.of("UTC+08:00");
    }
}
```

```Java
// 这个时候，需要给每个Bean添加不同的名字，可以用@Bean("name")指定别名，也可以用@Bean+@Qualifier("name")指定别名。
@Configuration
@ComponentScan
public class AppConfig {
    @Bean("z")
    ZoneId createZoneOfZ() {
        return ZoneId.of("Z");
    }

    @Bean
    @Qualifier("utc8")
    ZoneId createZoneOfUTC8() {
        return ZoneId.of("UTC+08:00");
    }
}
```

```Java
// 存在多个同类型的Bean时，注入ZoneId又会报错，意思是期待找到唯一的ZoneId类型Bean，但是找到两。
// NoUniqueBeanDefinitionException: No qualifying bean of type 'java.time.ZoneId' available: expected single matching bean but found 2
// 因此，注入时，要指定Bean的名称。
@Component
public class MailService {
    @Autowired(required = false)
    @Qualifier("z") // 指定注入名称为"z"的ZoneId
    ZoneId zoneId = ZoneId.systemDefault();
    ...
}
// 还有一种方法是把其中某个Bean指定为@Primary，这样，在注入时，如果没有指出Bean的名字，Spring会注入标记有@Primary的Bean。
@Configuration
@ComponentScan
public class AppConfig {
    @Bean
    @Primary // 指定为主要Bean
    @Qualifier("z")
    ZoneId createZoneOfZ() {
        return ZoneId.of("Z");
    }

    @Bean
    @Qualifier("utc8")
    ZoneId createZoneOfUTC8() {
        return ZoneId.of("UTC+08:00");
    }
}
// 这种方式也很常用。例如，对于主从两个数据源，通常将主数据源定义为@Primary，其它Bean默认注入的就是主数据源。如果要注入从数据源，那么只需要指定名称即可。
@Configuration
@ComponentScan
public class AppConfig {
    @Bean
    @Primary
    DataSource createMasterDataSource() {
        ...
    }

    @Bean
    @Qualifier("slave")
    DataSource createSlaveDataSource() {
        ...
    }
}
```

我们在设计模式的工厂方法中讲到，很多时候，可以通过工厂模式创建对象。Spring也提供了工厂模式，允许定义一个工厂，然后由工厂创建真正的Bean。

关于在什么时候去使用FactoryBean，可参考[掘金](https://www.jianshu.com/p/6f0a59623090)。简单来说就是设计到复杂bean的创建时，我们可以考虑使用FactoryBean。

```Java
// 用工厂模式创建Bean需要实现FactoryBean接口。
// 当一个Bean实现了FactoryBean接口后，Spring会先实例化这个工厂，然后调用getObject()创建真正的Bean。
// getObjectType()可以指定创建的Bean的类型，因为指定类型不一定与实际类型一致，可以是接口或抽象类。
// 因此，如果定义了一个FactoryBean，要注意Spring创建的Bean实际上是这个FactoryBean的getObject()方法返回的Bean。
// 为了和普通Bean区分，我们通常都以XxxFactoryBean命名。
@Component
public class ZoneIdFactoryBean implements FactoryBean<ZoneId> {

    String zone = "Z";

    @Override
    public ZoneId getObject() throws Exception {
        return ZoneId.of(zone);
    }

    @Override
    public Class<?> getObjectType() {
        return ZoneId.class;
    }
}
```

## 使用Resource

> Value

在Java程序中，我们经常会读取配置文件、资源文件等。使用Spring容器时，我们也可以把“文件”注入进来，方便程序读取。

例如，AppService需要读取logo.txt这个文件，通常情况下，我们需要写很多繁琐的代码，主要是为了定位文件，打开InputStream。

Spring提供了一个org.springframework.core.io.Resource（注意不是javax.annotation.Resource），它可以像String、int一样使用@Value注入。

```Java
import org.springframework.core.io.Resource;

@Component
public class AppService {
    @Value("classpath:/logo.txt")
    private Resource resource;
    private String logo;

    @PostConstruct
    public void init() throws IOException {
        InputStream in;
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(this.resource.getInputStream(), StandardCharsets.UTF_8))) {
            this.logo = reader.lines().collect(Collectors.joining("\n"));
        }
    }
}
```

注入Resource最常用的方式是通过classpath，即类似classpath:/logo.txt表示在classpath中搜索logo.txt文件，然后，我们直接调用Resource.getInputStream()就可以获取到输入流，**避免了自己搜索文件的代码**。

也可以直接指定文件的路径，例如：

```Java
@Value("file:/path/to/logo.txt")
private Resource resource;
```

但使用classpath是最简单的方式。使用Maven的标准目录结构，所有资源文件放入src/main/resources即可。

## 注入配置

> PropertySource

在开发应用程序时，经常需要读取配置文件。最常用的配置方法是以key=value的形式写在.properties文件中。

例如，MailService根据配置的app.zone=Asia/Shanghai来决定使用哪个时区。要读取配置文件，我们可以使用上一节讲到的Resource来读取位于classpath下的一个app.properties文件。但是，这样仍然比较繁琐。

Spring容器还提供了一个更简单的@PropertySource来自动读取配置文件。我们只需要在@Configuration配置类上再添加一个注解。

```Java
@Configuration
@ComponentScan
@PropertySource("app.properties") // 表示读取classpath的app.properties
public class AppConfig {
    @Value("${app.zone:Z}")
    String zoneId;

    @Bean
    ZoneId createZoneId() {
        return ZoneId.of(zoneId);
    }
}
```

Spring容器看到@PropertySource("app.properties")注解后，自动读取这个配置文件，然后，我们使用@Value正常注入。

```Java
@Value("${app.zone:Z}")
String zoneId;
```

注意注入的字符串语法，它的格式如下："${app.zone}"表示读取key为app.zone的value，如果key不存在，启动将报错；"${app.zone:Z}"表示读取key为app.zone的value，但如果key不存在，就使用默认值Z。这样一来，我们就可以根据app.zone的配置来创建ZoneId。

```Java
// 还可以把注入的注解写到方法参数中。
@Bean
ZoneId createZoneId(@Value("${app.zone:Z}") String zoneId) {
    return ZoneId.of(zoneId);
}
```

可见，先使用@PropertySource读取配置文件，然后通过@Value以${key:defaultValue}的形式注入，可以极大地简化读取配置的麻烦。

另一种注入配置的方式是先通过一个简单的JavaBean**持有所有的配置**。

```Java
@Component
@PropertySource("smtp.properties")
public class SmtpConfig {
    @Value("${smtp.host}")
    private String host;

    @Value("${smtp.port:25}")
    private int port;

    public String getHost() {
        return host;
    }

    public int getPort() {
        return port;
    }
}
```

然后，在需要读取的地方，使用#{smtpConfig.host}注入。

```Java
@Component
public class MailService {
    @Value("#{smtpConfig.host}")
    private String smtpHost;

    @Value("#{smtpConfig.port}")
    private int smtpPort;
}
```

注意观察#{}这种注入语法，它和${key}不同的是，#{}表示从JavaBean读取属性。"#{smtpConfig.host}"的意思是，从名称为smtpConfig的Bean读取host属性，即调用getHost()方法。一个Class名为SmtpConfig的Bean，它在Spring容器中的默认名称就是smtpConfig，除非用@Qualifier指定了名称。

使用一个独立的JavaBean持有所有属性，然后在其它Bean中以#{bean.property}注入的好处是，多个Bean都可以引用同一个Bean的某个属性。例如，如果SmtpConfig决定从数据库中读取相关配置项，那么MailService注入的@Value("#{smtpConfig.host}")仍然可以不修改正常运行。

## 使用条件装配

> Profile/Conditional

开发应用程序时，我们会使用开发环境，例如，使用内存数据库以便快速启动。而运行在生产环境时，我们会使用生产环境，例如，使用MySQL数据库。**如果应用程序可以根据自身的环境做一些适配，无疑会更加灵活**。

Spring为应用程序准备了Profile这一概念，用来表示不同的环境。例如，我们分别定义开发、测试和生产这3个环境：native；test；production。

创建某个Bean时，Spring容器可以根据注解@Profile来决定是否创建。如果当前的Profile设置为test，则Spring容器会调用createZoneIdForTest()创建ZoneId，否则，调用createZoneId()创建ZoneId。注意到@Profile("!test")表示非test环境。在运行程序时，加上JVM参数-Dspring.profiles.active=test就可以指定以test环境启动。

```Java
@Configuration
@ComponentScan
public class AppConfig {
    @Bean
    @Profile("!test")
    ZoneId createZoneId() {
        return ZoneId.systemDefault();
    }

    @Bean
    @Profile("test")
    ZoneId createZoneIdForTest() {
        return ZoneId.of("America/New_York");
    }
}
```

实际上，Spring允许指定多个Profile，例如：-Dspring.profiles.active=test,master。可以表示test环境，并使用master分支代码。要满足多个Profile条件，可以这样写。

```Java
@Bean
@Profile({ "test", "master" }) // 同时满足test和master
ZoneId createZoneId() {
    ...
}
```

除了根据@Profile条件来决定是否创建某个Bean外，Spring还可以根据@Conditional配合环境变量决定是否创建某个Bean。

```Java
// 它的意思是，如果满足OnSmtpEnvCondition的条件，才会创建SmtpMailService这个Bean。
@Component
@Conditional(OnSmtpEnvCondition.class)
public class SmtpMailService implements MailService {
    ...
}
// OnSmtpEnvCondition的条件是存在环境变量smtp，值为true。这样，我们就可以通过环境变量来控制是否创建SmtpMailService。
public class OnSmtpEnvCondition implements Condition {
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return "true".equalsIgnoreCase(System.getenv("smtp"));
    }
}
```

Spring只提供了@Conditional注解，**具体判断逻辑还需要我们自己实现**。Spring Boot提供了更多使用起来更简单的条件注解。

```Java
// 如果配置文件中存在app.smtp=true，则创建MailService：
@Component
@ConditionalOnProperty(name="app.smtp", havingValue="true")
public class MailService {
    ...
}
// 如果当前classpath中存在类javax.mail.Transport，则创建MailService：
@Component
@ConditionalOnClass(name = "javax.mail.Transport")
public class MailService {
    ...
}
// 我们以文件存储为例，假设我们需要保存用户上传的头像，并返回存储路径，在本地开发运行时，我们总是存储到文件：
@Component
@ConditionalOnProperty(name = "app.storage", havingValue = "file", matchIfMissing = true)
public class FileUploader implements Uploader {
    ...
}
// 在生产环境运行时，我们会把文件存储到类似AWS S3上：
@Component
@ConditionalOnProperty(name = "app.storage", havingValue = "s3")
public class S3Uploader implements Uploader {
    ...
}
// 其它需要存储的服务则注入Uploader：
@Component
public class UserImageService {
    @Autowired
    Uploader uploader;
}
// 当应用程序检测到配置文件存在app.storage=s3时，自动使用S3Uploader，如果存在配置app.storage=file，或者配置app.storage不存在，则使用FileUploader。
```

使用条件注解，能更灵活地装配Bean；Spring允许通过@Profile配置不同的Bean；Spring还提供了@Conditional来进行条件装配，Spring Boot在此基础上进一步提供了基于配置、Class、Bean等条件进行装配。
