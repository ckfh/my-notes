# 设计模式

- 开闭原则：软件应该对扩展开放，而对修改关闭。这里的意思是在增加新功能的时候，能不改代码就尽量不要改，如果只增加代码就完成了新功能，那是最好的。
- 里氏替换原则：这是一种面向对象的设计原则，即如果我们调用一个父类的方法可以成功，那么替换成子类调用也应该完全可以运行。
- 学习设计模式，关键是学习设计思想，不能简单地生搬硬套，也不能为了使用设计模式而过度设计，要合理平衡设计的复杂度和灵活性，并意识到设计模式也并不是万能的。

## 创建型模式

- 创建型模式关注点是如何创建对象，其核心思想是要把**对象的创建和使用相分离**，这样使得两者能相对独立地变换。

### 工厂方法

> 定义一个用于创建对象的接口，让子类决定实例化哪一个类。Factory Method使一个类的实例化延迟到其子类。  

- 工厂方法的目的是使得创建对象和使用对象是分离的，并且客户端总是引用抽象工厂和抽象产品。

    ```Java
    public class Main {
        public static void main(String[] args) {
            // 在客户端中，我们只需要和工厂接口NumberFactory和抽象产品Number打交道。
            // 调用方可以完全忽略真正的工厂NumberFactoryImpl和实际的产品BigDecimal。
            // 好处是允许创建产品的代码独立地变换，而不会影响到调用方。
            // 例如将实际产品替换为Double，此时调用方依旧能够正常进行工作。
            NumberFactory factory = NumberFactory.getFactory();
            Number result = factory.parse("123.456");
            System.out.println(result);
        }
    }
    // 工厂接口NumberFactory
    interface NumberFactory {
        // 产品接口Number
        Number parse(String s);

        static NumberFactory getFactory() {
            return impl;
        }

        NumberFactory impl = new NumberFactoryImpl();
    }
    // 实际工厂NumberFactoryImpl
    class NumberFactoryImpl implements NumberFactory {
        @Override
        public Number parse(String s) {
            // 实际产品BigDecimal
            return new BigDecimal(s);
        }
    }
    ```

- 有的童鞋会问：一个简单的parse()需要写这么复杂的工厂吗。实际上大多数情况下我们并不需要**抽象工厂**，而是通过静态方法直接返回产品。

    ```Java
    public class Main {
        public static void main(String[] args) {
            Number result = NumberFactory.parse("123.456");
            System.out.println(result);
        }
    }

    class NumberFactory {
        public static Number parse(String s) {
            return new BigDecimal(s);
        }
    }
    ```

- 这种简化地使用静态方法创建产品的方式称为静态工厂方法（Static Factory Method）。

    ```Java
    // 通过Integer的静态方法来获得Integer对象。
    // Integer既是产品又是静态工厂。它提供了静态方法valueOf()来创建Integer。
    Integer n = Integer.valueOf(100);
    ```

    ```Java
    public final class Integer {
        // 比起直接写new Integer(100)，它的好处在于，valueOf()内部可能会使用new创建一个新的Integer实例，但也可能直接返回一个缓存的Integer实例。
        // 对于调用方来说，没必要知道Integer创建的细节。如果调用方直接使用Integer n = new Integer(100)，那么就失去了使用缓存优化的可能性。
        public static Integer valueOf(int i) {
            if (i >= IntegerCache.low && i <= IntegerCache.high)
                return IntegerCache.cache[i + (-IntegerCache.low)];
            return new Integer(i);
        }
        ...
    }
    ```

- **工厂方法可以隐藏创建产品的细节，且不一定每次都会真正创建产品，完全可以返回缓存的产品，从而提升速度并减少内存消耗**。
- 经常使用的另一个静态工厂方法是List.of()，这个静态工厂方法接收可变参数，然后返回List接口。需要注意的是，调用方获取的产品总是List接口，而且并不关心它的实际类型。即使调用方知道List产品的实际类型是java.util.ImmutableCollections$ListN，也不要去强制转型为子类，因为静态工厂方法List.of()保证返回List，但也完全可以修改为返回java.util.ArrayList。这就是里氏替换原则：返回实现接口的任意子类都可以满足该方法的要求，且不影响调用方。
- **总是引用接口而非实现类，能允许变换子类而不影响调用方，即尽可能面向抽象编程**。
- 我们使用MessageDigest时，为了创建某个摘要算法，总是使用静态工厂方法getInstance(String)，调用方通过产品名称获得产品实例，不但调用简单，而且获得的引用仍然是MessageDigest这个抽象类。

    ```Java
    MessageDigest md5 = MessageDigest.getInstance("MD5");
    MessageDigest sha1 = MessageDigest.getInstance("SHA-1");
    ```

- 使用静态工厂方法实现一个类似20200202的整数转换为LocalDate。

    ```Java
    class LocalDateFactory {
        public static LocalDate fromInt(int yyyyMMdd) {
            return LocalDate.parse(String.valueOf(yyyyMMdd), DateTimeFormatter.BASIC_ISO_DATE);
        }
    }
    ```

> 工厂方法是指定义工厂接口和产品接口，但如何创建实际工厂和实际产品被推迟到子类实现，从而使调用方只和抽象工厂与抽象产品打交道。  
> 实际更常用的是更简单的静态工厂方法，它允许工厂内部对创建产品进行优化。  
> 调用方尽量持有接口或抽象类，避免持有具体类型的子类，以便工厂方法能随时切换不同的子类返回，却不影响调用方代码。  
