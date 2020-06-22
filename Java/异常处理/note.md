# 异常处理

## 使用JDK Logging

- 使用日志最大的好处是，它自动打印了时间、调用类、调用方法等很多有用的信息。

    ```Java
    Logger logger = Logger.getGlobal();
    logger.info("start process...");
    logger.warning("memory is running out...");
    logger.fine("ignored...");
    logger.severe("process will be terminated...");
    /*
    * 六月 22, 2020 10:08:48 上午 com.cat.logging.Main main
    * 信息: start process...
    * 六月 22, 2020 10:08:48 上午 com.cat.logging.Main main
    * 警告: memory is running out...
    * 六月 22, 2020 10:08:48 上午 com.cat.logging.Main main
    * 严重: process will be terminated...
    */
    ```

- 日志的输出可以设定级别。JDK的Logging定义了7个日志级别，从严重到普通。
- 因为默认级别是INFO，因此，INFO级别以下的日志，不会被打印出来。使用日志级别的好处在于，调整级别，就可以屏蔽掉很多调试相关的日志输出。
- Java标准库内置的Logging使用并不是非常广泛。

## 使用Commons Logging

- Commons Logging的特色是，它可以挂接不同的日志系统，并通过配置文件指定挂接的日志系统。默认情况下，Commons Loggin自动搜索并使用Log4j（Log4j是另一个流行的日志系统），如果没有找到Log4j，再使用JDK Logging。
- 使用Commons Logging只需要和两个类打交道，并且只有两步：第一步，通过LogFactory获取Log类的实例； 第二步，使用Log实例的方法打日志。

    ```Java
    Log log = LogFactory.getLog(Main.class);
    log.info("start...");
    log.warn("end...");
    /*
    * 六月 22, 2020 10:24:14 上午 com.cat.logging.Main main
    * 信息: start...
    * 六月 22, 2020 10:24:14 上午 com.cat.logging.Main main
    * 警告: end...
    */
    ```

- Commons Logging定义了6个日志级别，默认级别是INFO。
- 使用Commons Logging时，如果在静态方法中引用Log，通常直接定义一个静态类型变量。

    ```Java
    // 在静态方法中引用Log:
    public class Main {
        static final Log log = LogFactory.getLog(Main.class);

        static void foo() {
            log.info("foo");
        }
    }
    ```

- 在实例方法中引用Log，通常定义一个实例变量。

    ```Java
    // 在实例方法中引用Log:
    public class Person {
        protected final Log log = LogFactory.getLog(getClass());

        void foo() {
            log.info("foo");
        }
    }
    ```

- 注意到实例变量log的获取方式是LogFactory.getLog(getClass())，虽然也可以用LogFactory.getLog(Person.class)，但是前一种方式有个非常大的好处，就是子类可以直接使用该log实例。
- 由于Java类的动态特性，子类获取的log字段实际上相当于LogFactory.getLog(Student.class)，但却是从父类继承而来，并且无需改动代码。

    ```Java
    // 在子类中使用父类实例化的log:
    public class Student extends Person {
        void bar() {
            log.info("bar");
        }
    }
    ```

- 此外，Commons Logging的日志方法，例如info()，除了标准的info(String)外，还提供了一个非常有用的重载方法：info(String, Throwable)，这使得记录异常更加简单。

    ```Java
    public class Main {
        static final Log log = LogFactory.getLog(Main.class);

        public static void main(String[] args) {
            log.info("Start process...");
            try {
                "".getBytes("invalidCharsetName");
            } catch (UnsupportedEncodingException e) {
                log.error("got exception!", e);
            }
            log.info("Process end...");
        }
    }
    ```

- 个人理解：首先在当前类中根据自身Class类获得一个Log实例，之后在类方法中使用该Log实例进行日志打印。

## 使用Log4j

- Commons Logging，可以作为“日志接口”来使用。而真正的“日志实现”可以使用Log4j。
- 当我们使用Log4j输出一条日志时，Log4j自动通过不同的**Appender**把同一条日志输出到不同的目的地。
- 在输出日志的过程中，通过**Filter**来过滤哪些log需要被输出，哪些log不需要被输出。
- 最后，通过**Layout**来格式化日志信息。
- 上述结构虽然复杂，但我们在实际使用的时候，并不需要关心Log4j的API，而是通过配置文件来配置它。
- 以XML配置为例，使用Log4j的时候，我们把一个log4j2.xml的文件放到classpath下就可以让Log4j读取配置文件并按照我们的配置来输出日志。
- 要打印日志，只需要按Commons Logging的写法写，不需要改动任何代码，就可以得到Log4j的日志输出。
- **在开发阶段，始终使用Commons Logging接口来写入日志，并且开发阶段无需引入Log4j。如果需要把日志写入文件， 只需要把正确的配置文件和Log4j相关的jar包放入classpath，就可以自动把日志切换成使用Log4j写入，无需修改任何代码**。
- 如果要更换Log4j，只需要移除log4j2.xml和相关jar；只有扩展Log4j时，才需要引用Log4j的接口（例如，将日志加密写入数据库的功能，需要自己开发）。

## 使用SLF4J和Logback

- Commons Logging和Log4j这一对，它们一个负责充当日志API，一个负责实现日志底层，搭配使用非常便于开发。
- 因为对Commons Logging的接口不满意，有人就搞了SLF4J。因为对Log4j的性能不满意，有人就搞了Logback。
- SLF4J的日志接口传入的是一个带占位符的字符串，用后面的变量自动替换占位符，所以看起来更加自然。

    ```Java
    logger.info("Set score {} for Person {} ok.", score, p.getName());
    ```

    ```Java
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;

    class Main {
        // 对比一下Commons Logging和SLF4J的接口
        // 不同之处就是Log变成了Logger，LogFactory变成了LoggerFactory
        final Logger logger = LoggerFactory.getLogger(getClass());
    }
    ```
