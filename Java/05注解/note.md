# 注解

## 使用注解

注解是放在Java源码的类、方法、字段、参数前的一种特殊“注释”。

注释会被编译器直接忽略，**注解则可以被编译器打包进入class文件**，因此，注解是一种用作标注的“元数据”。

从JVM的角度看，注解本身对代码逻辑没有任何影响，如何使用注解完全由工具决定。

第一类是由编译器使用的注解，例如：@Override：让编译器检查该方法是否正确地实现了覆写；@SuppressWarnings：告诉编译器忽略此处代码产生的警告。**这类注解不会被编译进入.class文件，它们在编译后就被编译器扔掉了**。

第二类是由工具处理.class文件使用的注解，比如有些工具会在加载class的时候，对class做动态修改，实现一些特殊的功能。这类注解会被编译进入.class文件，但加载结束后并不会存在于内存中。**这类注解只被一些底层库使用，一般我们不必自己处理**。

第三类是在程序运行期能够读取的注解，**它们在加载后一直存在于JVM中，这也是最常用的注解**。例如，一个配置了@PostConstruct的方法会在调用构造方法后自动被调用（这是Java代码读取该注解实现的功能，JVM并不会识别该注解）。

定义一个注解时，还可以定义配置参数。配置参数可以包括：所有基本类型；String；枚举类型；基本类型、String以及枚举的数组。因为配置参数必须是**常量**，所以，上述限制保证了注解在定义时就已经确定了每个参数的值。注解的配置参数可以有**默认值**，缺少某个配置参数时将使用默认值。此外，大部分注解会有一个名为value的配置参数，对此参数赋值，可以只写常量，相当于省略了value参数。如果只写注解，相当于全部使用默认值。

```Java
public class Hello {
    @Check(min=0, max=100, value=55)
    public int n;

    @Check(value=99)
    public int p;

    @Check(99) // @Check(value=99)
    public int x;

    @Check     // 所有参数都使用默认值
    public int y;
}
```

## 定义注解

```Java
// 使用@interface语法来定义注解（Annotation）
public @interface Report {
    // 注解的参数类似无参数方法。
    // 可以用default设定一个默认值（强烈推荐）。最常用的参数应当命名为value。
    int type() default 0;
    String level() default "info";
    String value() default "";
}
```

有一些注解可以修饰其它注解，这些注解就称为元注解（meta annotation）。Java标准库已经定义了一些元注解，我们只需要使用元注解，通常不需要自己去编写元注解。

最常用的元注解是`@Target`。使用@Target可以定义Annotation能够被应用于源码的哪些位置：类或接口：ElementType.TYPE；字段：ElementType.FIELD；方法：ElementType.METHOD；构造方法：ElementType.CONSTRUCTOR；方法参数：ElementType.PARAMETER。

```Java
// 实际上@Target定义的value是ElementType[]数组，只有一个元素时，可以省略数组的写法
// 定义注解@Report可用在方法上
@Target(ElementType.METHOD)
public @interface Report {
    int type() default 0;
    String level() default "info";
    String value() default "";
}
// 定义注解@Report可用在方法或字段上，可以把@Target注解参数变为数组
@Target({
    ElementType.METHOD,
    ElementType.FIELD
})
public @interface Report {
    ...
}
```

另一个重要的元注解`@Retention`定义了Annotation的生命周期：仅编译期：RetentionPolicy.SOURCE；仅class文件：RetentionPolicy.CLASS；运行期：RetentionPolicy.RUNTIME。

```Java
// 如果@Retention不存在，则该Annotation默认为CLASS
// 因为通常我们自定义的Annotation都是RUNTIME，所以，务必要加上@Retention(RetentionPolicy.RUNTIME)这个元注解
@Retention(RetentionPolicy.RUNTIME)
public @interface Report {
    int type() default 0;
    String level() default "info";
    String value() default "";
}
```

使用`@Repeatable`这个元注解可以定义Annotation是否可重复。这个注解应用不是特别广泛。

```Java
@Repeatable(Reports.class)
@Target(ElementType.TYPE)
public @interface Report {
    int type() default 0;
    String level() default "info";
    String value() default "";
}

@Target(ElementType.TYPE)
public @interface Reports {
    Report[] value();
}
// 经过@Repeatable修饰后，在某个类型声明处，就可以添加多个@Report注解
@Report(type=1, level="debug")
@Report(type=2, level="warning")
public class Hello {
}
```

使用`@Inherited`定义子类是否可继承父类定义的Annotation。@Inherited仅针对@Target(ElementType.TYPE)类型的annotation有效，**并且仅针对class的继承，对interface的继承无效**。

```Java
@Inherited
@Target(ElementType.TYPE)
public @interface Report {
    int type() default 0;
    String level() default "info";
    String value() default "";
}
```

必须设置@Target来指定Annotation可以应用的范围；应当设置@Retention(RetentionPolicy.RUNTIME)便于运行期读取该Annotation。

## 处理注解

如何使用注解完全由工具决定。SOURCE类型的注解主要由编译器使用，因此我们一般只使用，不编写。CLASS类型的注解主要由底层工具库使用，涉及到class的加载，一般我们很少用到。只有RUNTIME类型的注解不但要使用，还经常需要编写。

因为注解定义后也是一种class，所有的注解都继承自java.lang.annotation.Annotation，因此，读取注解，需要使用反射API。Java提供的使用反射API读取Annotation的方法包括：**判断某个注解是否存在于Class、Field、Method或Constructor**：Class.isAnnotationPresent(Class)；Field.isAnnotationPresent(Class)；Method.isAnnotationPresent(Class)；Constructor.isAnnotationPresent(Class)；**使用反射API读取Annotation**：Class.getAnnotation(Class)；Field.getAnnotation(Class)；Method.getAnnotation(Class)；Constructor.getAnnotation(Class)。

```Java
// 判断@Report是否存在于Person类:
Person.class.isAnnotationPresent(Report.class);
// 获取Person定义的@Report注解:
Report report = Person.class.getAnnotation(Report.class);
int type = report.type();
String level = report.level();

// 使用反射API读取Annotation有两种方法。
// 方法一是先判断Annotation是否存在，如果存在，就直接读取。
Class cls = Person.class;
if (cls.isAnnotationPresent(Report.class)) {
    Report report = cls.getAnnotation(Report.class);
    ...
}
// 第二种方法是直接读取Annotation，如果Annotation不存在，将返回null。
Class cls = Person.class;
Report report = cls.getAnnotation(Report.class);
if (report != null) {
...
}
```

读取方法（Method）、字段（Field）和构造方法（Constructor）的Annotation和Class类似。但要读取**方法参数**的Annotation就比较麻烦一点，因为方法参数本身可以看成一个数组，而每个参数又可以定义多个注解，所以，一次获取方法参数的所有注解就必须用一个二维数组来表示。

```Java
public void hello(@NotNull @Range(max=5) String name, @NotNull String prefix) {
}

// 获取Method实例:
Method m = ...
// 获取所有参数的Annotation:
Annotation[][] annos = m.getParameterAnnotations();
// 第一个参数（索引为0）的所有Annotation:
Annotation[] annosOfName = annos[0];
for (Annotation anno : annosOfName) {
    if (anno instanceof Range) { // @Range注解
        Range r = (Range) anno;
    }
    if (anno instanceof NotNull) { // @NotNull注解
        NotNull n = (NotNull) anno;
    }
}
```

注解如何使用，**完全由程序自己决定**。例如，JUnit是一个测试框架，它会自动运行所有标记为@Test的方法。

```Java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Range {
    int min() default 0;

    int max() default 255;
}
// 定义了注解，本身对程序逻辑没有任何影响。我们必须自己编写代码来使用注解。
public class Person {
    @Range(min = 1, max = 20)
    public String name;
    @Range(max = 10)
    public String city;
}
// 编写一个Person实例的检查方法，它可以检查Person实例的String字段长度是否满足@Range的定义。
void check(Person person) throws IllegalAccessException {
    for (Field field : person.getClass().getFields()) {
        Range range = field.getAnnotation(Range.class);
        if (range != null) {
            Object value = field.get(person);
            if (value instanceof String) {
                String s = (String) value;
                // 我们自己编写代码获取注解参数并进行逻辑判断
                if (s.length() < range.min() || s.length() > range.max()) {
                    throw new IllegalArgumentException("Invalid field " + field.getName());
                }
            }
        }
    }
}
```

这样一来，我们通过@Range注解，配合check()方法，就可以完成Person实例的检查。**注意检查逻辑完全是我们自己编写的，JVM不会自动给注解添加任何额外的逻辑**。

可以在运行期通过反射读取RUNTIME类型的注解，注意千万不要漏写@Retention(RetentionPolicy.RUNTIME)，否则运行期无法读取到该注解。

可以通过程序处理注解来实现相应的功能：对JavaBean的属性值按规则进行检查；JUnit会自动运行@Test标记的测试方法。
