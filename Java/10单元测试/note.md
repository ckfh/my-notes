# 单元测试

## 编写JUnit测试

单元测试就是针对最小的功能单元编写测试代码。Java程序最小的功能单元是方法，因此，**对Java程序进行单元测试就是针对单个Java方法的测试**。

使用浮点数时，由于浮点数无法精确地进行比较，因此，我们需要调用assertEquals(double expected, double actual, double delta)这个重载方法，指定一个误差值：

```Java
assertEquals(0.1, Math.abs(1 - 9 / 10.0), 0.0000001);
```

**在编写单元测试的时候，我们要遵循一定的规范：一是单元测试代码本身必须非常简单，能一下看明白，决不能再为测试代码编写测试；二是每个单元测试应当互相独立，不依赖运行的顺序；三是测试时不但要覆盖常用测试用例，还要特别注意测试边界条件，例如输入为0，null，空字符串""等情况**。

## 使用Fixture

JUnit提供了编写测试前准备、测试后清理的固定代码，我们称之为Fixture。

标记为@BeforeEach和@AfterEach的方法，它们会在运行每个@Test方法前后自动运行。

还有一些资源初始化和清理可能更加繁琐，而且会耗费较长的时间，例如初始化数据库。JUnit还提供了@BeforeAll和@AfterAll，它们在运行所有@Test前后运行。

因为@BeforeAll和@AfterAll在所有@Test方法运行前后仅运行一次，因此，它们只能初始化静态变量，事实上，@BeforeAll和@AfterAll也只能标注在静态方法上。

编写Fixture的套路如下：

  1. 对于实例变量，在@BeforeEach中初始化，在@AfterEach中清理，它们在各个@Test方法中互不影响，因为是不同的实例；
  2. 对于静态变量，在@BeforeAll中初始化，在@AfterAll中清理，它们在各个@Test方法中均是唯一实例，会影响各个@Test方法。

**大多数情况下，使用@BeforeEach和@AfterEach就足够了。只有某些测试资源初始化耗费时间太长，以至于我们不得不尽量“复用”时才会用到@BeforeAll和@AfterAll**。

最后，注意到每次运行一个@Test方法前，JUnit首先创建一个XxxTest实例，因此，每个@Test方法内部的成员变量都是独立的，**不能也无法把成员变量的状态从一个@Test方法带到另一个@Test方法**。

## 异常测试

JUnit提供assertThrows()来期望捕获一个指定的异常。第二个参数Executable封装了我们要执行的会产生异常的代码。

```Java
public class Factorial {
    public static long fact(long n) {
        if (n < 0) {
            throw new IllegalArgumentException();
        }
        if (n > 20) {
            throw new ArithmeticException();
        }
        long r = 1;
        for (long i = 1; i <= n; i++)
            r = r * i;
        return r;
    }
}
public class FactorialTest {
    @Test
    void testNegative() {
        assertThrows(IllegalArgumentException.class, () -> {
            Factorial.fact(-1);
        });
    }

    @Test
    void testPositive() {
        assertThrows(ArithmeticException.class, () -> {
            Factorial.fact(21);
        });
    }
}
```

## 条件测试

在运行测试的时候，有些时候，我们需要排除某些@Test方法，不要让它运行，这时，我们就可以给它标记一个@Disabled

```Java
public class Config {
    public static String getConfigFile(String filename) {
        String os = System.getProperty("os.name").toLowerCase();
        if (os.contains("win")) {
            return "C:\\" + filename;
        }
        if (os.contains("mac") || os.contains("linux") || os.contains("unix")) {
            return "/usr/local/" + filename;
        }
        throw new UnsupportedOperationException();
    }
}
public class ConfigTest {
    @Test
    @EnabledOnOs(OS.WINDOWS) // 只在Windows系统上运行
    void testWindows() {
        assertEquals("C:\\test.ini", Config.getConfigFile("test.ini"));
    }

    @Test
    @EnabledOnOs({OS.LINUX, OS.MAC}) // 只在Mac/Linux系统上运行
    void testLinuxAndMac() {
        assertEquals("/usr/local/test.cfg", Config.getConfigFile("test.cfg"));
    }
}
```

常用的条件测试：

- 不在Windows平台执行的测试，可以加上@DisabledOnOs(OS.WINDOWS)；
- 只能在Java 9或更高版本执行的测试，可以加上@DisabledOnJre(JRE.JAVA_8);
- 只能在64位操作系统上执行的测试，可以用@EnabledIfSystemProperty判断;
- 需要传入环境变量DEBUG=true才能执行的测试，可以用@EnabledIfEnvironmentVariable;
- 万能的@EnableIf可以执行任意Java语句并根据返回的boolean决定是否执行测试。

## 参数化测试

参数化测试和普通测试稍微不同的地方在于，一个测试方法需要接收至少一个参数，然后，传入一组参数反复运行。

```Java
@ParameterizedTest
@ValueSource(ints = {0, 1, 5, 100})
void testAbs(int x) {
    assertEquals(x, Math.abs(x));
}
@ParameterizedTest
@ValueSource(ints = {-1, -5, -100})
void testAbsNegative(int x) {
    assertEquals(-x, Math.abs(x));
}
```

```Java
public class StringUtils {
    public static String capitalize(String s) {
        if (s.length() == 0)
            return s;
        return Character.toUpperCase(s.charAt(0)) + s.substring(1).toLowerCase();
    }
}

public class StringUtilsTest {
    @ParameterizedTest
    @MethodSource
    void testCapitalize(String input, String result) {
        assertEquals(result, StringUtils.capitalize(input));
    }
    // 通过@MethodSource注解并编写一个同名的静态方法来提供测试参数：
    static List<Arguments> testCapitalize() {
        return new ArrayList<Arguments>() {
            {
                add(Arguments.arguments("abc", "Abc"));
                add(Arguments.arguments("APPLE", "Apple"));
                add(Arguments.arguments("gooD", "Good"));
            }
        };
    }
}

public class StringUtilsTest {
    @ParameterizedTest
    @CsvSource({"abc, Abc", "APPLE, Apple", "gooD, Good"})
    void testCapitalize(String input, String result) {
        assertEquals(result, StringUtils.capitalize(input));
    }

    @ParameterizedTest
    @CsvFileSource(resources = {"/test-capitalize.csv"})
    void testCapitalize(String input, String result) {
        assertEquals(result, StringUtils.capitalize(input));
    }
}
```
