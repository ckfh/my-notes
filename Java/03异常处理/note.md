# 异常处理

## 使用JDK Logging

使用日志最大的好处是，它自动打印了时间、调用类、调用方法等很多有用的信息。

使用Java标准库中内置的日志工具`java.util.logging`写一个简单的例子：

```java
Logger logger = Logger.getGlobal();
logger.info("start process...");
logger.warning("memory is running out...");
logger.fine("ignored...");
logger.severe("process will be terminated...");
/**
* 8月 22, 2020 10:50:51 上午 com.cat.error.Main main
* 信息: start process...
* 8月 22, 2020 10:50:52 上午 com.cat.error.Main main
* 警告: memory is running out...
* 8月 22, 2020 10:50:52 上午 com.cat.error.Main main
* 严重: process will be terminated...
*/
```

再仔细观察发现，4条日志，只打印了3条，logger.fine()没有打印。这是因为，日志的输出可以设定级别。JDK的Logging定义了7个日志级别，从严重到普通。

因为默认级别是INFO，因此，INFO级别以下的日志，不会被打印出来。使用日志级别的好处在于，调整级别，就可以屏蔽掉很多调试相关的日志输出。

Java标准库内置的Logging使用并不是非常广泛。

```java
// 使用完整ClassName创建一个Logger实例，可以将配置进行分离，不同名称package的相同名称class使用不同的Logger实例:
Logger logger = Logger.getLogger(Main.class.getName());
// logger.getName()输出和Main.class.getName()一致:
System.out.println(logger.getName());
logger.info("start process...");
try {
    "".getBytes("invalidCharsetName");
} catch (UnsupportedEncodingException e) {
    logger.severe(e.getMessage());
}
logger.info("process end...");
```

## 使用Commons Logging

和Java标准库提供的日志不同，Commons Logging是一个第三方日志库，它是由Apache创建的日志模块。

**Commons Logging的特色是，它可以挂接不同的日志系统，并通过配置文件指定挂接的日志系统。默认情况下，Commons Loggin自动搜索并使用Log4j（Log4j是另一个流行的日志系统），如果没有找到Log4j，再使用JDK Logging**。

使用Commons Logging只需要和两个类打交道，并且只有两步：第一步，通过LogFactory获取Log类的实例； 第二步，使用Log实例的方法打日志。

```Java
Log log = LogFactory.getLog(Main.class);
log.info("start...");
log.warn("end...");
/**
* 8月 22, 2020 2:30:47 下午 com.cat.error.Main main
* 信息: start...
* 8月 22, 2020 2:30:47 下午 com.cat.error.Main main
* 警告: end...
*/
```

Commons Logging定义了6个日志级别，默认级别是INFO。

使用Commons Logging时，如果在静态方法中引用Log，通常直接定义一个静态类型变量：

```Java
// 在静态方法中引用Log:
public class Main {
    static final Log log = LogFactory.getLog(Main.class);

    static void foo() {
        log.info("foo");
    }
}
```

在实例方法中引用Log，通常定义一个实例变量：

```Java
// 在实例方法中引用Log:
public class Person {
    protected final Log log = LogFactory.getLog(this.getClass());

    void foo() {
        log.info("foo");
    }
}
```

**注意到实例变量log的获取方式是`LogFactory.getLog(this.getClass())`，虽然也可以用LogFactory.getLog(Person.class)，但是前者有个非常大的好处，就是子类可以直接使用该log实例**。

由于Java类的动态特性，子类获取的log字段实际上相当于`LogFactory.getLog(Student.class)`，但却是从父类继承而来，并且无需改动代码：

```Java
// 在子类中使用父类实例化的log:
public class Student extends Person {
    void bar() {
        log.info("bar");
    }
}
```

此外，Commons Logging的日志方法，例如info()，除了标准的info(String)外，还提供了一个非常有用的重载方法：`info(String, Throwable)`，这使得记录异常更加简单：

```Java
public class Main {
    static final Log log = LogFactory.getLog(Main.class);

    public static void main(String[] args) {
        log.info("Start process...");
        try {
            "".getBytes("invalidCharsetName");
        } catch (UnsupportedEncodingException e) {
            // 在语句之后直接打印错误栈:
            log.error("got exception!", e);
        }
        log.info("Process end...");
    }
}
```

如果是`java.util.logging.logger`来实现相同效果需要两条语句：`logger.severe("got exception!");e.printStackTrace();`，因为它只提供了打印`severe(String msg)`方法。

- Commons Logging的API非常简单（指它提供了多种便捷的重载方法）；
- Commons Logging可以自动检测并使用其他日志模块（指它本身负责定义规范，具体实现方式由其它日志模块来负责）。

## 使用Log4j

> 注意Log4j的三个包是否都被导入，除了看pom.xml文件，也要看项目的导入包列表。

**Commons Logging，可以作为“日志接口”来使用。而真正的“日志实现”可以使用Log4j**。

当我们使用Log4j输出一条日志时，Log4j自动通过不同的**Appender**把同一条日志输出到不同的目的地。例如：

- console：输出到屏幕；
- file：输出到文件；
- socket：通过网络输出到远程计算机；
- jdbc：输出到数据库。

在输出日志的过程中，通过**Filter**来过滤哪些log需要被输出，哪些log不需要被输出。例如，仅输出ERROR级别的日志。

最后，通过**Layout**来格式化日志信息，例如，自动添加日期、时间、方法名称等信息。

上述结构虽然复杂，但我们在**实际使用的时候，并不需要关心Log4j的API，而是通过配置文件来配置它**。

以XML配置为例，使用Log4j的时候，我们把一个`log4j2.xml`的文件放到classpath下就可以让Log4j读取配置文件并按照我们的配置来输出日志。下面是一个配置文件的例子：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
    <Properties>
        <!-- 定义日志格式 -->
        <Property name="log.pattern">%d{MM-dd HH:mm:ss.SSS} [%t] %-5level %logger{36}%n%msg%n%n</Property>
        <!-- 定义文件名变量 -->
        <Property name="file.err.filename">log/err.log</Property>
        <Property name="file.err.pattern">log/err.%i.log.gz</Property>
    </Properties>
    <!-- 定义Appender，即目的地 -->
    <Appenders>
        <!-- 定义输出到屏幕 -->
        <Console name="console" target="SYSTEM_OUT">
            <!-- 日志格式引用上面定义的log.pattern -->
            <PatternLayout pattern="${log.pattern}"/>
        </Console>
        <!-- 定义输出到文件,文件名引用上面定义的file.err.filename -->
        <RollingFile name="err" bufferedIO="true" fileName="${file.err.filename}" filePattern="${file.err.pattern}">
            <PatternLayout pattern="${log.pattern}"/>
            <Policies>
                <!-- 根据文件大小自动切割日志 -->
                <SizeBasedTriggeringPolicy size="1 MB"/>
            </Policies>
            <!-- 保留最近10份 -->
            <DefaultRolloverStrategy max="10"/>
        </RollingFile>
    </Appenders>
    <Loggers>
        <Root level="info">
            <!-- 对info级别的日志，输出到console -->
            <AppenderRef ref="console" level="info"/>
            <!-- 对error级别的日志，输出到err，即上面定义的RollingFile -->
            <AppenderRef ref="err" level="error"/>
        </Root>
    </Loggers>
</Configuration>
```

虽然配置Log4j比较繁琐，但一旦配置完成，使用起来就非常方便。对上面的配置文件，凡是INFO级别的日志，会自动输出到屏幕，而ERROR级别的日志，不但会输出到屏幕，还会同时输出到文件。并且，一旦日志文件达到指定大小（1MB），Log4j就会自动切割新的日志文件，并最多保留10份。

要打印日志，只需要按Commons Logging的写法写，不需要改动任何代码，就可以得到Log4j的日志输出。

**在开发阶段，始终使用Commons Logging接口来写入日志，并且开发阶段无需引入Log4j。如果需要把日志写入文件， 只需要把正确的配置文件和Log4j相关的jar包放入classpath，就可以自动把日志切换成使用Log4j写入，无需修改任何代码**。

只有扩展Log4j时，才需要引用Log4j的接口（例如，将日志加密写入数据库的功能，需要自己开发）。

**JVM在执行Java程序的时候，并不是一次性把所有用到的class全部加载到内存，而是第一次需要用到class时才加载**。

动态加载class的特性对于Java程序非常重要。利用JVM动态加载class的特性，我们才能在运行期根据条件加载不同的实现类。例如，Commons Logging总是优先使用Log4j，只有当Log4j不存在时，才使用JDK的logging。利用JVM动态加载特性，大致的实现代码如下：

```java
// Commons Logging优先使用Log4j:
LogFactory factory = null;
if (isClassPresent("org.apache.logging.log4j.Logger")) {
    factory = createLog4j();
} else {
    factory = createJdkLog();
}

boolean isClassPresent(String name) {
    try {
        Class.forName(name);
        return true;
    } catch (Exception e) {
        return false;
    }
}
```

这就是为什么我们只需要把Log4j的jar包放到classpath中，Commons Logging就会自动使用Log4j的原因。

## 使用SLF4J和Logback

Commons Logging和Log4j这一对，它们一个负责充当日志API，一个负责实现日志底层，搭配使用非常便于开发。

因为对Commons Logging的**接口**不满意，有人就搞了SLF4J。因为对Log4j的**性能**不满意，有人就搞了Logback。

我们先来看看SLF4J对Commons Logging的接口有何改进。在Commons Logging中，我们要打印日志，有时候得这么写：

```java
int score = 99;
p.setScore(score);
log.info("Set score " + score + " for Person " + p.getName() + " ok.");
```

拼字符串是一个非常麻烦的事情，所以SLF4J的日志接口改进成这样了：

```java
int score = 99;
p.setScore(score);
logger.info("Set score {} for Person {} ok.", score, p.getName());
```

SLF4J的日志接口传入的是一个带占位符的字符串，用后面的变量自动替换占位符，所以看起来更加自然。

```Java
logger.info("Set score {} for Person {} ok.", score, p.getName());
```

如何使用SLF4J？它的接口实际上和Commons Logging几乎一模一样：

```Java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

class Main {
    // 对比一下Commons Logging和SLF4J的接口
    // 不同之处就是Log变成了Logger，LogFactory变成了LoggerFactory
    final Logger logger = LoggerFactory.getLogger(this.getClass());
}
```

仍然需要一个Logback的配置文件，把logback.xml放到classpath下，配置如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            <charset>utf-8</charset>
        </encoder>
        <file>log/output.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>log/output.log.%i</fileNamePattern>
        </rollingPolicy>
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <MaxFileSize>1MB</MaxFileSize>
        </triggeringPolicy>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

**始终使用SLF4J的接口写入日志，使用Logback只需要配置，不需要修改代码**。
